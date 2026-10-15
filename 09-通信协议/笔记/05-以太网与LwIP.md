# 05 · 以太网与 LwIP：给 MCU 插上网线

> 目标：理解 TCP/IP 各层在 MCU 里的实体，跑通 LwIP 三个台阶（ping → TCP echo → HTTP），掌握 Socket API 与"中断/任务"协作模型。四大件《计算机网络》理论在此全面落地。

## 1. MCU 联网的两条硬件路线

这是什么：让 MCU 联网有两条路——在 MCU 里跑完整的 TCP/IP 软件栈（累但懂原理），或买一颗把 TCP/IP 硬件化了的芯片（W5500），MCU 只用 SPI 发命令（省心但黑盒）。选择的本质：**学习深度 vs 交付速度**。

| 路线 | 代表芯片 | MCU 干什么 | 适合 |
|------|----------|-----------|------|
| **软件协议栈** | STM32F407/F107 内置 MAC + 外接 PHY（LAN8720） | LwIP 跑在 MCU 上，CPU 管收发包 | 学习首选/大量包处理 |
| **硬件协议栈** | **W5500**（BOM 篇的老朋友） | 芯片内置 TCP/IP，MCU 只发 SPI 命令 | 51/低配 MCU 快速联网 |

类比：软件栈=自己开车去机场（灵活但累）；W5500=直接打车（简单省心，按"socket 号"交互）。**学习用软件栈（理解原理），产品求省事用 W5500**。

**PHY 是什么**：把 MAC 的数字帧变成网线上的模拟信号的"调制解调器"，MCU 与它之间走 RMII 接口（精简媒体独立接口——MAC 与 PHY 之间的一束标准数字信号线，"精简"在引脚比 MII 减半、百兆够用）+ MDIO 配置总线（读 PHY 状态寄存器判断插线/速率）。

类比：MAC 是"打包车间"，PHY 是"货车队+高速公路"——车间只管打包，路上跑是车队的事；MDIO 则是车间给车队打电话查路况的专线（插没插网线、跑百兆还是千兆，问它）。

## 2. TCP/IP 分层在 MCU 里的实体（对照计算机网络的地图）

这是什么：《计算机网络》里的 TCP/IP 分层图，在 PC 上藏在操作系统内核里看不见；在 MCU 上，每一层都能指认出一个具体的 .c 文件或外设——**抽象理论第一次变成摸得着的代码**。这张表就是"理论→实体"的对照词典。

| 层 | 协议 | MCU/LwIP 里的实体 |
|----|------|-------------------|
| 链路层 | 以太网帧、ARP | ETH 外设 + DMA 描述符收发环 |
| 网络层 | IP、ICMP、ARP | LwIP ip4.c / ping 用的 ICMP |
| 传输层 | TCP/UDP | LwIP tcp.c/udp.c（TCP 三次握手在这层，四大件理论兑现） |
| 应用层 | HTTP/MQTT/DNS | 你的代码或中间件 |

**记忆锚点**：ping 用 ICMP（网络层，无端口）；TCP/UDP 才有端口；socket = 应用层握住传输层的把手。

## 3. LwIP：嵌入式世界的标准协议栈

这是什么：把几十万行的 Linux 网络栈砍到几百 KB、还能跑在几十 KB RAM 的 MCU 上——这就是 LwIP。它提供三套"点菜档次"不同的 API：同一套协议内核，三副面孔。

LwIP（Lightweight IP）= 给小内存设备用的开源 TCP/IP 栈，FreeRTOS 生态标配。

### 3.1 三种 API（选型必知）

类比：raw = 自带食材现炒（性能最高，但锅碗瓢盆自己刷=回调必须快进快出）；netconn = 净菜半成品（RTOS 封装，省事但隔着一层）；socket = 下馆子点菜（与 Linux 同构、最省心，学习首选）。

| API | 模型 | 性能 | 适用 |
|-----|------|------|------|
| raw | 回调函数（无 OS 也能跑） | 最高 | 裸机、极致性能（回调地狱） |
| netconn | RTOS 专用封装 | 中 | 中间件内部 |
| **socket（netdb）** | BSD 套接字（与 Linux 同构） | 中 | **学习与应用首选**（四大件直接对接） |

### 3.2 CubeMX + FreeRTOS 配置要点

```
① Connectivity → ETH：RMII 模式（PHY LAN8720 注意 PHY 地址，常为 0 或 1——
   板子上 ETH_PHY_AD 引脚接法决定，错了 ping 不通的头号原因）
② Middleware → LwIP：启用、平台选 FreeRTOS（自动起 tcpip_thread）
③ 勾 DHCP 或手动静态 IP（先静态 IP 学习，免得 DHCP 失败一头雾水）
④ 内存：MEM_SIZE 建议 8~16KB 起步；pbuf 池按需
   （pbuf=packet buffer，LwIP 的"包裹"结构，网络数据在栈里全程以它流转，§5 的 TX pbuf 就是发件包裹）
```

## 4. 三个台阶（实操主线）

这是什么：网络调试忌讳"一口吃成胖子"——三个台阶从下往上逐层点亮：ping 通证明 1~2 层好了，TCP echo 证明传输层会用了，HTTP GET 证明应用层会写了。**每上一阶前先确认下阶是通的，排查范围就减半**。

### 台阶①：ping 通（链路+网络层就绪）

做什么：先证明"最底下两层是通的"——网线、PHY、RMII、IP 栈全部就绪，这是网络世界的"通电测试"。

```
PC: ping 192.168.1.100  → Reply from ... （通了！）
```
ping 通 = 网线/PHY/RMII/IP 栈全部 OK。**不通时排查序**：看 LED（PHY 链路灯）→ 查 PHY 地址 → 抓包看 ARP 有没有应答 → 查 IP 是否同网段。

### 台阶②：TCP echo 服务器（Socket API 第一课）

做什么：写一个"鹦鹉学舌"服务器——你发什么它回什么。它是网络编程的 Hello World，socket 五个调用各就各位，跑通它就掌握了服务器的骨架。

```c
void EchoTask(void *arg)                        /* 独立任务（08-FreeRTOS 架构再+1） */
{
    char buf[256];
    int srv = socket(AF_INET, SOCK_STREAM, 0);  /* 1. 造socket（AF_INET=IPv4, STREAM=TCP） */

    struct sockaddr_in addr = {
        .sin_family = AF_INET,
        .sin_port   = htons(8888),              /* 2. 绑定端口——htons：小端MCU转网络大端序（字节序坑！） */
        .sin_addr.s_addr = INADDR_ANY,          /*    监听本机所有 IP */
    };
    bind(srv, (struct sockaddr*)&addr, sizeof(addr));
    listen(srv, 1);                             /* 3. 开门迎客，挂起 1 个等待队列 */

    for (;;) {                                  /* 4. 服务循环：接一个客人聊完再接下一个 */
        int cli = accept(srv, NULL, NULL);      /*    阻塞等连接（三次握手由栈完成） */
        int n;
        while ((n = recv(cli, buf, sizeof(buf), 0)) > 0) {
            send(cli, buf, n, 0);               /*    收多少回多少：echo */
        }
        closesocket(cli);                       /*    recv=0 对端正常关闭 → 挂断 */
    }
}
```

电脑验证：`ncat 192.168.1.100 8888`（或网络调试助手）发什么回什么。**四个系统调用的心智**：socket 造话筒 → bind 报门牌 → listen 开店 → accept 迎客。

按"开店"比喻逐行拆（每行一句话：谁在做什么）：
- `socket()`：申请一间空的店面（拿到句柄，此刻没地址没业务）；
- `bind()`：挂上门牌"8888 号"——告诉系统，寄到这个端口的包裹都归我这间店；
- `listen()`：开门挂"营业中"，门口摆 1 把等位椅（等待队列）；
- `accept()`：坐门口等客人上门（三次握手由协议栈悄悄完成），客人到了才返回一个"专属服务员"（cli）；
- `recv/send` 循环：服务员与客人一对一对话，收多少回多少；`recv` 返回 0 = 客人结账离店（对端关闭）；
- `closesocket(cli)`：送客后回 `accept` 接下一位——注意 srv 从未关闭，店还开着。

### 台阶③：HTTP GET（应用层初体验）

做什么：当一次 TCP 客户端向服务器要数据——你会发现所谓 HTTP，就是"按格式写一段文本发过去"。这是亲手发 HTTP 报文的"祛魅时刻"。

```c
/* 向 http://192.168.1.5/api/data 发 GET（TCP 客户端角色） */
int s = socket(AF_INET, SOCK_STREAM, 0);
struct sockaddr_in srv = { .sin_family = AF_INET, .sin_port = htons(80) };
inet_aton("192.168.1.5", &srv.sin_addr);        /* 点分IP → 二进制 */
connect(s, (struct sockaddr*)&srv, sizeof(srv));  /* 主动握手 */

char req[] = "GET /api/data HTTP/1.1\r\nHost: 192.168.1.5\r\nConnection: close\r\n\r\n";
send(s, req, strlen(req), 0);                   /* HTTP 就是文本协议！发进去就完事 */
int n = recv(s, rsp, sizeof(rsp), 0);           /* 收响应：状态行+头+正文 */
closesocket(s);
```

**恍然大悟点**：HTTP 请求就是一段格式化文本（`\r\n` 换行），TCP 只负责可靠搬运——应用层协议"在 TCP 之上谈文本"的模式，也是 07 篇 MQTT 的前奏。

## 5. 收发数据流：DMA 描述符环与 tcpip_thread（理解异步）

这是什么：你的任务调用 `recv` 时明明"睡着了"，数据来了它是怎么醒的？本节回答"一帧数据从网线到你的变量，中间经过了谁的手"。核心认知：**你的任务只跟 socket 打交道，快递分拣全是 tcpip_thread 在干**。

**收包时间线**（每步一句话：谁在做什么）：

```
① PHY：电信号进站，还原成以太网帧
② ETH MAC：验帧、过滤"不是寄给我的"
③ DMA：不吵 CPU，默默把帧写进 RX 描述符环指着的内存块
④ 以太网中断：只贴通知单，不搬货
⑤ tcpip_thread：从描述符环取货 → 逐层拆包（以太网头→IP头→TCP头）→ 数据放进对应 socket 的接收缓冲
⑥ 你的任务：recv() 从阻塞中醒来，抱着数据返回
```

**发送时间线**：

```
① 你的任务：send() 把数据交给 LwIP（拷进 TX pbuf）
② LwIP：包上 TCP 头/IP 头/以太网头，挂到 TX 描述符
③ DMA：把帧吐给 MAC → PHY → 网线
```

```
网线 → PHY → ETH MAC → DMA 写入 RX 描述符环的内存块 → 以太网中断通知
     → LwIP 的 tcpip_thread（CubeMX 自动创建）解析分发
     → 唤醒你的任务（accept/recv 从阻塞中醒来）
发送：你的 send() → 拷进 TX pbuf → 挂 TX 描述符 → DMA 吐给 MAC → 网线
```

（原图保留作速记版。）

**关键认知**：你的任务只跟 socket 打交道，脏活（DMA/中断/协议解析）全在 tcpip_thread——**"中断委托给任务"（FreeRTOS 篇）思想在整个协议栈里放大重演**。

## 6. 常见坑

网络问题排障的基本盘：**先分层定位**（链路通没通？网络层呢？传输层呢？），再查本层配置——下表按此逻辑归类，从上往下对症状。

| 坑 | 现象 |
|----|------|
| PHY 地址错 | ping 不通（最常见的"硬件对了也不通"） |
| 忘 htons/htonl | 端口 8888 变成 47138，连接不上 |
| MEM_SIZE 太小 | 大包丢/卡死（抓包只见请求不见响应） |
| 每个 recv 丢进 while 无退出条件 | 对端断开后死循环（recv 返回 0/-1 要处理） |
| LWIP_SOCKET 没开 | socket 头文件不存在 |
| 阻塞 socket 塞死任务 | recv 永远等（设 SO_RCVTIMEO 或非阻塞轮询） |
| IP 冲突/同网段外 | ping 通了但跨网段不通（网关/掩码配置） |

## 7. 章节小结

- 两条硬件路线：软件栈（F407+PHY+LwIP）vs 硬件栈（W5500，SPI 一把梭）
- LwIP 三 API，学 socket（与 Linux 同构）；tcpip_thread 是全栈的"总调度任务"
- 三台阶：ping（网络层通）→ TCP echo（传输层通）→ HTTP GET（应用层通）
- 字节序 htons/htonl 是网络编程第一坑；HTTP=文本协议开眼界

## 工程实操任务

- [ ] F407+LAN8720（或 W5500 完成同等三台阶）：ping → TCP echo → HTTP GET 全通
- [ ] echo 任务加 SO_RCVTIMEO，实现"60 秒无活动断开"，验证对端拔线
- [ ] Wireshark 抓包：找到 ARP 请求/应答、TCP 三次握手、你的 echo 数据帧——对照四大件教材截图存档
- [ ] echo 升级为"命令服务器"：收 `led on` 点灯回 `OK`（自定义应用层协议雏形）
- [ ] （选做）DHCP 获取 IP 后用串口打印 ipaddr（netif 回调）

## 课后思考题

1. 为什么 MCU 常用"阻塞 socket+独立任务"而不是裸机 raw 回调？（裸机 raw=回调里必须极速返回否则饿死协议栈，逻辑散；任务+阻塞=线性思维、互斥好管理（FreeRTOS 队列衔接）——用 2KB 栈买回可维护性，与 RTOS 选型逻辑一脉相承）
2. TCP 断线（网线拔了）你的 echo 任务怎么知道？（两种途径：recv 返回 0/-1（对端关或 RST）；网线物理断开由 PHY 中断/netif 状态回调（link_down）感知——健壮程序两者都处理，并实现自动重连）
3. W5500 有 8 个 socket，硬件栈方案里"并发连接"怎么设计？（每 socket 一个状态机/任务（FreeRTOS 版）；轮询 socket 寄存器收发——硬件栈的"并发"受 socket 数硬限制，软件栈受内存与任务数限制——**并发上限是方案选型硬指标**）

## 深入资料（想学得更细再看）

- **官方规范/文档**：野火《LwIP应用开发实战指南》（GitHub 搜"LwIP应用开发实战指南"有开源电子版，中文 LwIP 教程事实标准）+ LwIP 官方 wiki（savannah.nongnu.org，配置项与源码说明）
- **书籍**：《计算机网络：自顶向下方法》TCP/UDP/应用层章节——三次握手/端口/字节序的理论源头
- **工具**：Wireshark（wireshark.org，抓 ARP/三次握手/echo 数据帧——本篇实操任务主角）
