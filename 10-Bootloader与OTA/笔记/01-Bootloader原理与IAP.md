# 01 · Bootloader 原理与 IAP

> 目标：吃透"上电第一段代码在干什么"，完成 Flash 分区、跳转 APP、串口 IAP 三大件。

## 1. Bootloader 是什么、为什么必须有

这一节回答"上电之后、main 之前，CPU 到底在跑谁的代码，这段代码凭什么存在"。

没有 OTA 的产品：升级 = 拆壳接 ST-Link（量产不可接受）。有 Bootloader 的产品：一根串口/网线/4G 就能换固件。

**上电流程全景**：

```
上电 → 硬件取 SP/PC（07-STM32 启动四步）→ 复位中断 → 启动文件 → main
      └ Bootloader 常驻 Flash 头部（0x08000000），它先跑：
          检查升级标志？ ──是──▶ 收新固件写入 APP 区 → 校验 → 跳转
                └──否──▶ 直接跳转 APP
```

类比：**酒店入住**——Bootloader 是前台（永远第一个到岗）：查预订（升级标志）→ 安排房间（写 Flash）→ 送客进门（跳转 APP）；APP 是住客，住下后前台不管（直到下次复位）。对应关系钉牢：

- 前台的登记本 = 参数区的升级标志（掉电也不能丢的"纸面记录"）；
- 安排房间 = 把新固件写进 APP 区（客人还没进门，房间先收拾好）；
- 送客进门 = 跳转三步（本篇核心：验身份、清场地、换门牌）；
- 住客入住后前台退场 = APP 接管全部中断向量与外设，前台只在下次复位后重新出现。

**铁律：Bootloader 自身不升级自己**——升级中掉电 = 整机变砖（只有 Bootloader 还活着才有机会救）。工程方案：Bootloader 只升级 APP 区。

## 2. Flash 分区规划（第一步，纸面工作）

这一节回答"512KB Flash 怎么切、每块干什么"。分区是升级系统的地基——**先在纸上画对，代码才有的放矢**。

以 STM32F103ZE（512KB）为例：

```
0x0800_0000 ┌──────────────────┐
            │ Bootloader 32KB   │  ← 上电必跑，永不升级
0x0800_8000 ├──────────────────┤
            │ APP_A 224KB       │  ← 当前运行固件
0x0804_0000 ├──────────────────┤
            │ APP_B 224KB       │  ← 升级下载区/备用区
0x0807_8000 ├──────────────────┤
            │ 参数区 32KB       │  ← 版本号/校验/升级标志/回滚计数
0x0807_FFFF └──────────────────┘
```

**分区三原则**：Bootloader 够用就好（小 = 出错面小）；APP 区两块等大（双分区轮流写）；参数区独立（升级标志绝不能被固件擦掉）。F1 每 2KB 一页（擦除单位），分区边界对齐页。

对照酒店理解分区：前台（Bootloader）永远在岗，所以放在门口（0x08000000，上电必经之路）；A/B 两间客房**等大**——客人随时可能从 A 换到 B 或从 B 换到 A，房间不一样大就住不下；登记本（参数区）锁在前台保险柜里（独立分区）——绝不能放客房里，否则客人退房（固件擦写）把登记本一起撕了，回滚就没了依据。

## 3. 跳转三步（本篇核心代码，背下来）

这一节是本篇核心：Bootloader 把 CPU **彻底移交**给 APP 的全部动作。为什么不是一句函数调用？因为 APP 不是普通函数——它自带独立的向量表、栈和中断环境，直接跳过去等于"换了城市还沿用旧门牌"，必须完成一整套交接仪式。

跳转前建立画面——**前台送客三步**：①验房（魔数校验：APP 区像不像一个合法固件）；②清场（时钟/外设/中断全部恢复出厂）；③换门牌+交钥匙（向量表指向 APP、栈指针切到 APP 的栈顶）。

```c
#define APP_ADDR  0x08008000UL

typedef void (*pFunction)(void);

void JumpToApp(void)
{
    uint32_t appSp  = *(volatile uint32_t*)APP_ADDR;            /* ① APP 区首字=栈顶指针 */
    uint32_t appPc  = *(volatile uint32_t*)(APP_ADDR + 4);      /*   第二字=复位向量 */

    if ((appSp & 0x2FFE0000) != 0x20000000) return;             /* 魔数校验：不像合法栈顶 → APP 无效，留在 Bootloader */

    HAL_RCC_DeInit();                                           /* ② 反初始化：时钟回默认、关外设、关中断 */
    HAL_DeInit();
    for (int i = 0; i < 8; i++) NVIC->ICER[i] = 0xFFFFFFFF;     /* 清所有中断使能 */
    for (int i = 0; i < 8; i++) NVIC->ICPR[i] = 0xFFFFFFFF;
    SysTick->CTRL = 0;                                          /* 停滴答（不关会跳转瞬间进孤儿中断） */

    SCB->VTOR = APP_ADDR;                                       /* ③ 向量表指到 APP——APP 的中断才能正确分发 */
    __set_MSP(appSp);                                           /* 设主栈指针 */
    pFunction appEntry = (pFunction)appPc;
    __enable_irq();
    appEntry();                                                 /* 起飞，永不返回 */
}
```

**逐行拆解**（对照代码，一行一行过）：

- `appSp = *(volatile uint32_t*)APP_ADDR;`——读 APP 区**第一个字**：Cortex-M 启动文件的铁律是"偏移 0 放初始栈顶指针"，这是新房的"户口本第一页"；
- `appPc = *(volatile uint32_t*)(APP_ADDR + 4);`——**第二个字**是复位向量（APP 的 Reset_Handler 地址），跳转的最终目的地；
- `if ((appSp & 0x2FFE0000) != 0x20000000) return;`——**①验房**：把读到的值与 RAM 地址模式比对，合法栈顶必落在 0x2000_xxxx 区间；APP 没烧/烧错区时读到的 0xFFFFFFFF 或代码字都不符合 → 直接 return 留在 Bootloader（防变砖第一道闸）；
- `HAL_RCC_DeInit(); HAL_DeInit();`——**②清场**：时钟恢复复位默认、外设全部反初始化。为什么？APP 的初始化代码假设自己面对的是"刚上电"的芯片；带着 Bootloader 配好的时钟直接进，串口波特率、I2C 时序都可能算错——"跳过去波特率乱"的玄学就是这么来的；
- `NVIC->ICER[i] = 0xFFFFFFFF; / ICPR[i] = ...`——把所有中断**关门谢客**（禁能）并清挂起。不清的话，跳转瞬间来了个挂起中断，CPU 拿着旧向量表（或刚换的新表）去找处理函数 → 跳野地址 → HardFault（"孤儿中断"事故）；
- `SysTick->CTRL = 0;`——单独停滴答：SysTick 是 HAL 时基的心跳，跳转瞬间它还在响 = 必然孤儿中断，所以单独立一条（为什么上面 ICER 关不掉它：SysTick 属于内核系统异常，不走 NVIC 的中断使能位，必须自己单独停）；
- `SCB->VTOR = APP_ADDR;`——**③换门牌（最灵魂的一行）**：向量表基址从 0x08000000 改指 APP 区，从此所有中断按 APP 的门牌找处理函数；
- `__set_MSP(appSp);`——**交新钥匙**：主栈指针切到 APP 的栈顶，从此函数调用、局部变量都在 APP 自己的栈上进行，Bootloader 的栈正式退休；
- `appEntry();`——起飞，永不返回。它不是普通函数调用（没有返回地址在等你），是一次有去无回的移交。

**APP 侧配套改动**（CubeMX 里改两处）：

```c
/* ① system_stm32f1xx.c：向量表偏移 */
#define VECT_TAB_OFFSET  0x8000U            /* 与 APP 起始地址一致 */

/* ② 链接脚本/Keil IROM 起始改 0x08008000，大小 224K —— 告诉编译器"你住在 A 区" */
```

这两处为什么必须改：VECT_TAB_OFFSET 让 APP 在自己的初始化里**再次确认**向量表位置（与 Bootloader 里设 VTOR 形成双保险）；链接地址不改的话，编译器默认按 0x08000000 排布代码——跟 Bootloader 的地盘重叠，跳过去执行的也是错位的字节。

**跳转失败三查**：魔数校验不过（APP 没烧/烧错区）→ HardFault（VTOR 没设/中断没清）→ 玄学（时钟没 DeInit，串口波特率乱）。

## 4. Flash 写入：解锁→擦除→烧写→上锁

这一节回答"新固件怎么落进 Flash"。Flash 不是内存，写入有物理规矩：**只能把 1 改成 0，想把 0 改回 1 必须整页擦除**——所以流程固定四步，一步不能省。

类比在**刻碑**上改字：先解锁（不然刻不动），改字得整块磨平重刻（擦除），然后一笔一划刻新字（烧写），完工落锁、拍照留底（上锁+回读校验）。

```c
HAL_FLASH_Unlock();                                        /* 1. 解锁（防误写，两个魔数钥匙） */
FLASH_EraseInitTypeDef e = { FLASH_TYPEERASE_PAGES, 0x08008000, 112 }; /* 2. 擦 A 区 112 页 */
uint32_t err;
HAL_FLASHEx_Erase(&e, &err);                               /*    擦完全 0xFF */

for (uint32_t i = 0; i < len; i += 4)                      /* 3. 按 32 位字烧写（F1 最小单位） */
    HAL_FLASH_Program(FLASH_TYPEPROGRAM_WORD, dst + i, *(uint32_t*)(src + i));

HAL_FLASH_Lock();                                          /* 4. 上锁 */
/* 5. 回读比对 —— 写完必校验！ */
```

关键行拆解：

- `HAL_FLASHEx_Erase(&e, &err)`——112 页 × 2KB = 224KB，正好擦平整个 A 区，擦完全区变 0xFF（"磨平重刻"）；
- `for (...; i += 4)`——F1 烧写最小单位是 32 位字：按 4 字节步进、地址必须对齐，跨行错位会静默失败；
- 回读比对不可省：写 Flash 有失败率，且"写入静默失败"（读回还是旧值）是本篇坑表第一名。

**三纪律**：写前必擦（Flash 只能 1→0，不擦写不进去）；按对齐宽度写；写完回读 CRC 比对才算数。

## 5. 串口 IAP：Ymodem 收固件全流程

这一节把前面所有零件装成整机：标志判断 → 收固件 → 校验 → 写 Flash → 跳转。先看时间线再看代码。

**一次完整 IAP 升级的时间线**：

- 第 1 步：设备复位，Bootloader 先跑（它住在 0x08000000，上电必经），读参数区升级标志——上一轮 APP 运行中收到升级命令，写好了 FLAG_UPGRADE；
- 第 2 步：标志有效 → Bootloader 打印"等待 Ymodem"，PC 端串口助手把 .bin 一帧一帧发来（每帧带序号+CRC16），Bootloader 逐包校验后**边收边写**进 A 区预留扇区——不能整包缓存在 RAM：像 F103ZE 只有 64KB RAM，装不下 224KB 的 APP，必须分包落 Flash；
- 第 3 步：收完 → 回读 A 区整包，比对 CRC 与大小合法性 → 全部一致才算写成功；
- 第 4 步：全部通过 → 写 FLAG_OK → JumpToApp() 跳进新固件——用户视角就是"设备重启后换了新功能"；
- 第 5 步（失败路径）：任何一步失败 → 写 FLAG_FAIL → 重启重试——Bootloader 自己从没被动过，重试永远有得重来；
- 对照路径：标志无效 → 跳过 2~4 直接 JumpToApp——正常上电只多花几十毫秒，这就是"永不拆壳升级"的全部成本。

```c
void IAP_Main(void)                                  /* Bootloader 主流程 */
{
    uint32_t flag = ReadUpgradeFlag();               /* 参数区读标志（备份寄存器更佳，见 02 篇） */
    if (flag == FLAG_UPGRADE) {
        Shell_Print("等待 Ymodem 发送固件...\r\n");
        /* Ymodem_Receive 逐包收：每包 CRC 过关就调 WritePacket 落 Flash（边收边写），
           绝不能整包缓存在 RAM——64KB RAM 的 F103ZE 装不下 224KB 的 APP */
        if (Ymodem_Receive(WritePacket) == OK) {     /* PC 串口助手发 .bin */
            if (Flash_VerifyApp()) {                 /* 收完回读整包：CRC/大小合法性 */
                WriteUpgradeFlag(FLAG_OK);
            }
        } else WriteUpgradeFlag(FLAG_FAIL);          /* 失败标记 → 重启重试 */
    }
    JumpToApp();                                     /* 无论升级与否，最终都进 APP */
}
```

Ymodem 帧结构（理解即可，库现成）：SOH(1B 数据 128)/STX(1KB 数据)+序号+数据+CRC16——**又是"帧头+序号+CRC"老三样**（09-通信协议 01 篇模板第三次重演，协议设计万变不离其宗）。

## 6. 常见坑

| 坑 | 现象 |
|----|------|
| APP 没改 VECT_TAB_OFFSET | 进 APP 后中断全乱（进 main 前或一进中断就死） |
| 链接地址没改 | APP 用 Bootloader 的地址跑，跳转 HardFault |
| 跳转前不清中断/SysTick | 跳转瞬间孤儿中断 → HardFault |
| 写 Flash 忘解锁/擦除 | 写入静默失败（读回还是旧值） |
| 参数区放 APP 区里 | 升级把标志擦了，回滚逻辑失效 |
| Bootloader 也用 HAL 全家桶 | 体积超标 32KB（HAL 只保留用的外设，或改 LL/寄存器） |
| 升级标志放普通 Flash 但 APP 直接写参数区 | 竞态损坏（统一由 Bootloader 写参数区） |

## 7. 章节小结

- Bootloader=上电第一段代码：查标志→升级→跳转；**永不升级自己**
- 分区三块：Bootloader/APP（双区）/参数区；边界对齐擦除页
- 跳转三步：校验栈顶魔数 → DeInit+清中断 → 设 VTOR+MSP 起飞；APP 侧同步改 VTOR 偏移与链接地址
- Flash 写入四步：解锁→擦→烧→上锁+回读校验；Ymodem=帧头+序号+CRC 的又一次重演

## 工程实操任务

- [ ] 纸面画出你板子的分区地图（含每区起始地址与大小）
- [ ] 裸跳转实验：ST-Link 分别烧 Bootloader（A 区写死一个 LED 版 APP）验证跳转三步
- [ ] 串口 IAP：串口助手 Ymodem 发送 .bin 完成"LED 呼吸灯 → 跑马灯"在线切换
- [ ] 故障复现三连：不改 VTOR、不清中断、烧错区——记录现象并修复（排障经验入库）
- [ ] CubeProgrammer 回读两区对比，确认写入与校验逻辑正确

## 课后思考题

1. 为什么校验栈顶魔数 `(sp & 0x2FFE0000) == 0x20000000` 能判断 APP 有效？（合法 MSP 必在 RAM 区间 0x2000xxxx；APP 区若为空/损坏读到 0xFFFFFFFF 或代码字，与 RAM 地址模式不符——廉价而有效的"像不像栈指针"启发式检查）
2. 为什么升级标志更建议放 RTC 备份寄存器而非 Flash 参数区？（备份寄存器单字原子写不怕掉电半写；Flash 写半程掉电可能标志损坏——电池在（BOM 篇 CR1620 的用途之一）时备份域是天然"可靠小存储"）
3. 若产品 Flash 只有 128KB，装不下双分区怎么办？（方案：单分区+片外存储（SPI Flash W25Q64 当暂存区，Bootloader 从外存搬运）；或差分包压缩；或 Bootloader+压缩 APP 解压运行——**空间不够，外部补/压缩凑**，设计永远有余地）

## 深入资料（想学得更细再看）

- **官方文档**：ST 应用笔记 AN2557（F1 串口 IAP 官方方案——思路参考，其他芯片找同系列 AN）；《Cortex-M3权威指南》向量表章节（跳转三步与向量表偏移的底层依据）
- **书籍/专栏**：野火《STM32 HAL 库开发实战指南》IAP 章节；RT-Thread 文档站 OTA 手册（系统性概念好读物，文档站 rtthread.org）
