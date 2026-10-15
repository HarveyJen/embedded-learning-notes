# 03 · U-Boot、内核与根文件系统

> 目标：打通"裸板 → 跑起 Linux"的完整启动链，用 Buildroot 一键出整套系统，掌握烧卡启动与 U-Boot 常用操作——Linux 方向最大的一道坎。

## 1. 启动链全景：四棒接力

**这是什么/解决什么问题**：Linux 板子上电后**不会直接跑你的程序**——要像接力赛一样跑四棒（BootROM→SPL→U-Boot→内核），每棒只做一件事，再把控制权交给下一棒。搞清每一棒干什么，后面所有"起不来"的排障都有地图可循。

**时间线分步叙述**（上电后发生了什么，逐帧回放）：

- **T0 上电**：CPU 手里只有片内 SRAM（DDR 还没"通电激活"），出厂固化的 BootROM 从启动介质（SD/EMMC）头部读 1KB 加载执行——**这就是烧卡必须把 U-Boot 写在 1KB 偏移处的原因**：不写这儿，BootROM 找不到；
- **T1 SPL**：小到能塞进片内 SRAM（几百 KB），专职"点亮 DDR"——DDR 初始化时序复杂，是裸机程序里最大的事；DDR 一好，才有地方放大家伙；
- **T2 U-Boot**：搬进 DDR 后如鱼得水，串口、网络、存储全可用——它把 zImage（内核）+dtb（设备树）读进内存，喊一声 `bootz` 交棒；
- **T3 内核**：解压自举，按设备树逐个初始化驱动，挂上 rootfs（`root=` 指定的分区），启动 1 号进程 init；
- **T4 init/systemd**：按 /etc 配置拉起各服务，最后给你 shell——你的应用从这登台。

**与 STM32 启动的对照**（08 篇 Bootloader 思想放大版）：STM32 一段代码直达 APP；Linux 四棒接力，每棒职责单一、可独立更换——**复杂度换灵活性**。

类比：**剧团巡演**——BootROM=剧场大门（永远开放）、SPL=搭台工人（搭好舞台即走）、U-Boot=报幕员（介绍主角上场）、Kernel=舞台本身、rootfs=后台化妆间（演员们的家）、你的应用=正式演出。

## 2. U-Boot：报幕员十八般武艺

**这是什么/解决什么问题**：U-Boot 是**裸机小系统**（前面学的寄存器操作、串口、网络协议这里全是真的），正式职责是把内核请上台；副业是给你一个命令行——系统还没起来时，你靠它搬文件、改参数、救砖。

### 2.1 常用命令（串口进 U-Boot 命令行）

```bash
help / 版本信息                # 迷路先 help
printenv                       # 看环境变量（bootcmd/bootargs 是灵魂）
setenv bootargs 'console=ttymxc0,115200 root=/dev/mmcblk1p2 rootwait rw'
setenv bootcmd 'mmc dev 1; fatload mmc 1:1 0x80800000 zImage; fatload mmc 1:1 0x83000000 imx6ull.dtb; bootz 0x80800000 - 0x83000000'
saveenv                        # 写入 EMMC 持久化
tftp 0x80800000 zImage         # 网络拉内核（开发神器：免反复插卡）
dhcp                           # 自动拿 IP
mmc dev/mmc info               # 存储设备操作
reset                          # 重启
```

**命令白话**：`printenv`=念一遍当前台词本（环境变量）；`setenv`=改台词（引号内整段替换，改完只存在内存里）；`saveenv`=把改好的台词写进 EMMC 持久化（**不 saveenv 断电就忘**）；`tftp`=从局域网服务器拉文件到内存（开发神器：改了内核不用反复插卡烧写）；`dhcp`=自动拿 IP；`mmc dev/info`=切换/查看存储设备；`reset`=重启重来。

### 2.2 两个灵魂变量（必须吃透）

- **bootargs**：传给内核的"开场白"——控制台接哪个串口、根文件系统在哪（`root=`）、读写模式；
- **bootcmd**：自动启动脚本——从哪、怎么把内核和设备树搬进内存、用什么命令起跳。

**U-Boot 启动失败的排障序**：串口有没有输出 → 卡在哪个阶段（SPL 无输出=DDR/镜像头问题；U-Boot 进了但 bootz 失败=镜像/dtb/地址错）→ bootargs 的 root= 对不对（起不来 kernel panic 常客）。

## 3. 设备树：硬件说明书与代码分离（04 篇深入）

**这是什么**：本节先记住分工——内核是"通用演员"，dtb 是"这个剧场的道具清单"，U-Boot 负责把两者凑一起。语法与驱动细节 04 篇展开。

U-Boot 把 **zImage（内核）+ .dtb（设备树）** 一起交给内核——内核靠 dtb 知道"这块板子有什么硬件、接在哪个地址"。设计动机：**同一份内核配不同 dtb 就能跑不同板子**（硬件变了改说明书，不重编内核）。

## 4. 根文件系统 rootfs：应用的"家"

**这是什么/解决什么问题**：内核只会"跑"，不会"干活"——真正干活的全在 rootfs 里（命令、配置、库、你的应用）。内核起来后第一件事就是挂载根文件系统（`root=` 指定），然后找 **/sbin/init** 开跑。类比：内核是会呼吸的身体，rootfs 是它的行李箱——没有行李箱，它连 ls 这样的命令都变不出来。

```
/bin /sbin   命令（busybox 一人分饰千角）
/etc         配置（inittab/init.d 自启脚本/网络配置）
/lib         共享库（glibc/musl——02 篇 GLIBC 匹配问题的源头；所有命令都要靠它们才能跑，缺了或不配连 ls 都报 not found）
/dev         设备文件（驱动入口，mdev/udev 动态生成）
/root /home  用户空间
```

**busybox**：把几百个 Unix 命令塞进一个 1MB 可执行文件（软链接到它）——嵌入式 rootfs 的标配地基。类比：一人分饰千角的**万能群演**——`ls`、`cp`、`ps` 全是它的马甲（软链接），喊哪个名字它演哪个角色。

## 5. Buildroot：一键构建整套系统

**这是什么/解决什么问题**：手工编 U-Boot/内核/busybox/库/应用再拼 rootfs，练一次懂原理，但日常这么干既慢又不可复现。**Buildroot=给整个系统用的自动流水线**——你勾选要什么（目标架构/工具链/软件包），它自动下载源码→交叉编译→拼装 rootfs→产出全套镜像。配置文件可进 git，换台电脑一键重建一模一样的系统——这就是"可复现"的工程价值。

手工编 U-Boot/内核/busybox/库/应用再拼 rootfs，练一次懂原理，日常用 **Buildroot**（menuconfig 勾选 → 自动下载编译拼装出全镜像）：

```bash
make menuconfig                 # 图形配置：
#  Target Options  → ARM Cortex-A7（i.MX6ULL）
#  Toolchain       → 自定义交叉工具链（或 Buildroot 自建）
#  System config   → 主机名、root 密码、init 系统（busybox init 或 systemd）
#  Kernel          → 用自己的 zImage 配置
#  Target packages → 勾应用：dropbear(ssh) / libmbedtls / 你的程序包
#  Filesystem images → ext4 / tar
make -j8                        # 一键出：u-boot、zImage、dtb、rootfs.ext4 全套
ls output/images/               # 成果全在这
```

**命令白话**：`make menuconfig`=打开图形化勾选菜单（上下键移动、空格选中、ESC 退出）——相当于给流水线下订单；`make -j8`=流水线开工（8 线程并行，首次构建要几十分钟，属正常）。

**产物烧卡**：

```bash
# SD 卡分区（板厂脚本 or 手动 fdisk）：p1=FAT（放 zImage+dtb+u-boot 引导脚本；选 FAT 是因为 U-Boot 的 fatload 命令读它最省事） p2=ext4（rootfs）
dd if=u-boot-dtb.imx of=/dev/sdb bs=1k seek=1 conv=fsync   # U-Boot 写在卡的第 1KB 偏移（BootROM 约定！）
# 拷 zImage/dtb 到 p1，解包 rootfs 到 p2 → 插卡启动 → 串口看到 shell
```

**命令白话**：`dd`=按字节原样搬运/写入的"克隆工"（if=从哪读、of=写到哪、conv=fsync=确保真正落盘才返回）；`bs=1k seek=1`=从第 1KB 偏移开始写（BootROM 到这儿找引导，写错位置=串口全静默）。

**首次成功标志**：串口滚出内核启动日志 → `Welcome to Buildroot` → `login:` → root 进入 → `ls /` 看到你的家。

## 6. 常见坑

怎么读这张表：启动类问题的排查心法是**按启动链逐棒定位**——左列原因、右列现象；现象是"串口全静默"就查前两棒（SPL/DDR/偏移），是"kernel panic"就查后两棒（镜像/dtb/root=）。

| 坑 | 现象 |
|----|------|
| U-Boot 偏移没写在 1KB 处 | BootROM 找不到引导，串口全静默 |
| SPL/DDR 参数与板不匹配 | 上电无任何输出（连 U-Boot 都不进） |
| dtb 与内核不配套 | 起到一半 panic/外设全失灵 |
| root= 写错分区 | kernel panic - not syncing: VFS unable to mount |
| 工具链 glibc 与 rootfs 不匹配 | 进 shell 前一路 not found |
| zImage 没编 dtb support | 起不来现代内核（CONFIG_ 忘开） |
| 串口只出不进 | console 参数与实际 tty 名不符 |

## 7. 章节小结

- 启动四棒：BootROM→SPL(DDR)→U-Boot(引导)→Kernel(挂 rootfs 起 init)——每环职责单一
- U-Boot 灵魂变量 bootargs（告诉内核 root 在哪）/bootcmd（怎么起跳）；tftp 是开发神器
- rootfs=busybox+库+配置+你的应用；**Buildroot=可复现的一键全家桶**
- 烧卡三件：U-Boot 写 1KB 偏移、zImage+dtb 进 FAT、rootfs 进 ext4
- **"起不来"永远按启动链逐棒排查**——静默查 SPL/DDR，起跳查镜像，挂载查 root=

## 工程实操任务

- [ ] 手工路线走一遍：单独编译 U-Boot → 用 tftp 启动厂商内核（吃透 bootcmd/bootargs）
- [ ] Buildroot 全流程：menuconfig 定制（加 dropbear+你的 hello 服务）→ 烧卡 → ssh 进入
- [ ] U-Boot 网络升级实验：tftp 拉新 zImage 现场替换启动，体会"免烧写迭代"
- [ ] 故障复现：改错 root= 分区、换错 dtb——记录 panic 现象并修复（排障手册素材）
- [ ] 把"烧卡启动"整理成一页 SOP（作品集工程能力佐证）

## 课后思考题

1. 为什么 SPL 要单独存在，不让 U-Boot 一步到位？（片内 SRAM 只有几百 KB 放不下完整 U-Boot，且 DDR 需要先初始化——SPL 小巧专职"点亮 DDR"，是**资源约束下的两段式加载**，与 STM32 芯片可直接全速跑形成对比（Cortex-A 的 RAM 不在片内））
2. 设备树为什么不直接编进内核，而要随启动单独传？（一套内核配多板（换 dtb 即换硬件视图）；内核升级与硬件描述解耦——**数据与逻辑分离**的经典工程决策）
3. 对比 STM32 OTA（08 篇）与 Linux 系统升级（替换 zImage/rootfs），失败保护思想有何异同？（同：都不能"升级正在跑的自己"（A/B 分区思想一致）；异：Linux 有文件系统天然支持双 rootfs 挂载切换+U-Boot 判断——**不变砖哲学跨平台通用**）

## 深入资料（想学得更细再看）

- **视频课**：B站搜索「正点原子 i.MX6ULL 嵌入式Linux」（野火同名系列同价——移植与烧卡全流程跟做）
- **移植文档**：正点原子 i.MX6ULL 移植文档（配套资料里的 U-Boot/内核移植手册，第 1~2 节卡住时按图索骥）
- **官方文档**：buildroot.org 手册（第 5 节 Buildroot 的权威出处，menuconfig 每个选项都有解释）
