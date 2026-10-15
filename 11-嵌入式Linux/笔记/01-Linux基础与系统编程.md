# 01 · Linux 基础与系统编程

> 目标：命令行生存能力 + 应用层编程全部家当（文件IO/进程线程/信号/socket）。这是 Linux 方向的"手艺基础"。

**先说人话**：这一篇解决"上了 Linux，没有 Keil 没有图形界面，我怎么干活"的问题。把 Linux 想成一栋大楼：命令行是前台服务台，文件是房间，进程/线程是楼里干活的人，socket 是楼对外的电话线——本篇教你**住进这栋楼并自如活动**。好消息是：你学 FreeRTOS 时建立的概念（任务/队列/信号量/中断纪律）全部可以平移过来，只是换了名字。

## 1. 命令行生存包（20 个命令打天下）

**这是什么/解决什么问题**：Linux 开发 90% 的操作在命令行完成——编译、查日志、管进程、看网络全靠它们。先混个脸熟，不必死记（用 `man 命令名` 随时可查）。

```bash
文件：ls -l / cd / pwd / cp -r / mv / rm -rf(慎) / mkdir -p / find . -name "*.c"
查看：cat / less(翻页) / head -20 / tail -f xxx.log(实时追日志) / grep -rn "main" .
权限：chmod +x run.sh / sudo（管理员）
进程：ps aux | grep app / top / kill -9 PID(强杀) / jobs & fg
网络：ping / ifconfig / netstat -tlnp(谁在监听)
工具：tar -zxvf xx.tar.gz / df -h(磁盘) / uname -a / man 2 open(第2卷=系统调用手册！)
```

**命令逐条白话**（首次见面专用，用到再回来查）：

| 命令 | 白话解释 |
|------|---------|
| `ls -l` | 列出当前目录的文件清单（-l=详细模式，能看到权限/大小/时间） |
| `cd` / `pwd` | 换房间 / 报告"我现在在哪个房间"（cd .. 退上一层，cd 回家） |
| `cp -r` | 复制（-r=连文件夹里的东西一起搬） |
| `mv` | 移动或重命名（同一本书换个书架，或换个书名） |
| `rm -rf` | 递归+强制删除，**没有回收站**——敲回车前默念三遍路径 |
| `mkdir -p` | 建文件夹（-p=中间层不存在就顺路全建好） |
| `find . -name "*.c"` | 从当前目录往下翻箱倒柜找所有 .c 文件 |
| `cat` / `less` | 一次性倒出小文件内容 / 翻页看大文件（q 退出，/关键词 搜索） |
| `head -20` | 只看文件前 20 行 |
| `tail -f` | 盯着文件尾部实时刷新——程序一边写日志你一边看到，追日志神器 |
| `grep -rn "main"` | 在文件内容里搜关键词（-r=连子目录一起搜，-n=带行号） |
| `chmod +x` | 给文件发"可执行许可证"（否则 ./run.sh 提示权限不足） |
| `sudo` | 以管理员身份执行（改系统文件/装软件都要它） |
| `ps aux \| grep app` | 给当前所有进程拍快照，再从中筛出 app（\|=管道：把前一个命令的输出递给后一个） |
| `top` | 实时任务管理器：谁在吃 CPU/内存一目了然（q 退出） |
| `kill -9 PID` | 给进程发"立即枪毙"信号（PID=进程身份证号，ps 里查到） |
| `jobs` / `fg` | 看有哪些后台任务 / 把后台任务调回前台 |
| `ping` | 向对方喊一声"在吗"，测网络通不通 |
| `ifconfig` | 查本机网卡 IP（看自家门牌号） |
| `netstat -tlnp` | 查哪些端口正被哪些程序监听（"谁开着门"） |
| `tar -zxvf` | 解压 .tar.gz（z=解 gzip、x=解开、v=显示过程、f=对文件操作） |
| `df -h` | 看磁盘还剩多少空间（-h=按人类习惯的单位显示） |
| `uname -a` | 报内核版本等信息（确认编译目标对不对时用） |

**man 手册分卷**：1=命令、2=系统调用、3=库函数——`man 2 open` 看内核门、`man 3 fopen` 看库包装。**会查 man = 不用搜索引擎也能开发**。

## 2. 一切皆文件：文件描述符

**这是什么/解决什么问题**：Linux 把文件/socket/串口/管道统一抽象为**文件**，操作它们=拿整数句柄（fd）调系统调用——好处是**只学一套 open/read/write/close，就能操作一切 I/O**。

类比：**fd=号码牌**。Linux 是个超级食堂，所有窗口（文件、串口、socket）都凭号码牌取餐——`open` 一次发一块牌，之后 `read/write` 全凭牌号，`close` 把牌还回去。0、1、2 号牌是每个进程出生就自带的（标准输入/输出/错误），你申请到的第一块牌通常从 3 开始：

```
0=stdin  1=stdout  2=stderr（每个进程天生自带这三个）
```

```c
#include <fcntl.h>
#include <unistd.h>

int fd = open("/dev/led", O_WRONLY);        /* 打开（设备文件也这么开！） */
if (fd < 0) { perror("open"); return -1; }  /* 错误三件套：判-1 + perror + 退出 */

write(fd, "1", 1);                          /* 写 1 字节 */
close(fd);                                  /* 不关=资源泄漏（fd 上限 1024 个） */
```

**两套 API 的关系**：`fopen/fread`（C 标准库，带缓冲）包着 `open/read`（系统调用，直达内核）——嵌入式偏好系统调用层（少一层缓冲，时序可控）。系统调用的白话：**按下"进内核"的按钮**，用户态程序自己碰不到硬件，只能请内核代劳。

## 3. 进程：fork/exec/wait 三部曲

**这是什么/解决什么问题**：Linux 里一个程序想"一边干 A 一边干 B"（比如主程序 + 独立的守护程序），就靠 fork 复制进程、exec 换程序、wait 收尸——这是 Linux 应用管理的三块基石。

进程 = 程序运行的实例（独立地址空间，MMU 隔离）。类比：**程序是菜谱，进程是照着菜谱正在炒的那锅菜**——同一份菜谱可以同时炒好几锅，互不干扰。

```c
pid_t pid = fork();                         /* 分身术：一调两返回 */
if (pid == 0) {
    execl("/usr/bin/uart_daemon", "uart_daemon", NULL);  /* 子进程变身别的程序 */
    _exit(127);                             /* exec 失败才会走到这 */
} else if (pid > 0) {
    int status;
    waitpid(pid, &status, 0);               /* 父进程收尸（不 wait = 僵尸进程） */
} else {
    perror("fork");
}
```

- **fork 后子进程是父进程的克隆**（代码从 fork 处继续跑，返回值区分身份）；
- **exec 家族=换灵魂**：进程壳不变，程序整个换掉；
- **僵尸进程**=子死父不收尸——`wait/waitpid` 或父进程忽略 SIGCHLD。

**fork 双返回·时间线分步看**（初学最容易懵的点，逐帧回放）：

1. **第 0 帧**：父进程执行到 `fork()` 这一行——此时世上只有一个进程；
2. **第 1 帧**：内核收到 fork 请求，复制出一个子进程（现代内核用"写时复制"：父子先共享内存，谁先写谁再真拷，省内存又快）；
3. **第 2 帧**：fork 在**父子两个进程里各返回一次**——父进程拿到子进程的 PID（>0），子进程拿到 0；
4. **第 3 帧**：两个进程从 fork 的下一行**各自继续跑**，靠 `if (pid == 0)` 分流——子进程走 execl 变身，父进程走 waitpid 等孩子；
5. **第 4 帧**：子进程 exec 换灵魂成为 uart_daemon；父进程 waitpid 挂起等待，孩子退出后给它"收尸"。

一句话：**fork 只调用一次，却返回两次**——不是"一个函数返回了两个值"，而是"两个进程各自拿到自己的那份返回值"。

**三个系统调用白话**：`fork`=复印一份自己（克隆进程）；`execl`=连人带魂换成另一个程序（但 PID 不变）；`waitpid`=给孩子**收尸**——子进程退出后会留一份"死亡记录"等父进程来读，父进程一直不读，它就变成占着户口本的**僵尸进程**（ps 里状态 Z）。

## 4. 线程：pthread 与同步四件套

**这是什么/解决什么问题**：上一节进程是"克隆一个自己出去单过"（各自有厨房），线程是"一个家里雇几个帮工"——**全家用同一个厨房（共享内存），各干各的活（各自执行流）**。线程比进程轻（创建/切换开销小），通信也简单（直接读写共享变量，但要防竞争）。

线程 = 同进程内共享地址空间的执行流（对应 FreeRTOS 任务！）：

**两个 pthread 函数白话**：`pthread_create`=招聘一名帮工，让它从 worker 函数开始干活；`pthread_join`=等帮工下班并回收"工牌"（不 join 就撤退=主线程先跑了全队散伙）。

```c
void *worker(void *arg) {                   /* 线程函数原型（像不像 FreeRTOS 任务？） */
    for (;;) { printf("working\n"); sleep(1); }
    return NULL;
}

pthread_t tid;
pthread_create(&tid, NULL, worker, NULL);
pthread_join(tid, NULL);                    /* 等线程结束（FreeRTOS 的 vTaskDelete 对照） */
```

**同步互斥对照表**（FreeRTOS 篇的内核对象全面升级）。

怎么读这张表：左列是你学过的 FreeRTOS 内核对象，中列是 Linux 的"同款零件"，右列是使用要点——**横向逐行对照**，等于把旧知识翻译成新 API 名字，不用从零学：

| FreeRTOS | pthread | 备注 |
|----------|---------|------|
| 互斥量（优先级继承） | `pthread_mutex_t` | 用法几乎一样，谁拿谁还 |
| 二值信号量 | `sem_t` | sem_wait/sem_post |
| 计数信号量 | `sem_t`（初值>1） | 同上 |
| 事件组 | `pthread_cond_t`（条件变量） | cond_wait 必须配 mutex（假唤醒！） |
| 队列 | 环形缓冲+mutex+cond 自组 | 或消息队列 mq |

**必背坑**：竞争条件靠 mutex；条件变量用 `while(条件不满足) pthread_cond_wait()`（**while 不是 if**——防假唤醒）。

假唤醒白话：条件没满足、wait 也可能"无故"返回——OS 允许这种实现，且别的线程可能在你醒来到重新拿锁之间抢先改了条件；所以醒来必须重查条件，不成立就接着睡。

## 5. 信号：异步"快递通知"

**这是什么/解决什么问题**：信号是内核或别的进程往你进程**门缝里塞的紧急纸条**——不用你主动去查，塞到时无论你在干啥都会被打断去处理。Ctrl+C 能杀程序，就是因为终端给进程塞了一张 SIGINT 纸条。

常用信号白话：`SIGINT`=Ctrl+C（请求中断）；`SIGTERM`=礼貌版"请你退出"（systemd 停服务就发它，**务必捕获做善后**）；`SIGKILL`/`SIGSTOP`=系统留的最后一手，纸条塞死、**无法捕获**。

```c
void handler(int sig) { flag = 1; }         /* 处理函数：越短越好（只置标志！与 ISR 同纪律） */

signal(SIGINT, handler);                    /* Ctrl+C 不再直接杀 */
signal(SIGTERM, graceful_exit);             /* systemd 停服务发这个——捕获它做善后 */
/* SIGKILL/SIGSTOP 捕获不了（系统留的最后一手） */
```

**工程套路**：信号处理函数只置 volatile 标志，主循环/单独线程处理——**与 FreeRTOS ISR 纪律一模一样**（"中断里只喊话"思想跨系统通用）。

## 6. socket：网络编程（与 09-通信协议 05 篇同构）

**这是什么/解决什么问题**：socket 是进程间"打电话的听筒"——两个进程（可以不在同一台机器）各持一个 socket，通过 IP+端口拨号接通后互发数据。你写网关、写采集服务、连云端全靠它。

07 篇已在 LwIP 上写过 socket 四部曲——**Linux 版 API 完全一样**（LwIP 就是仿 BSD socket 设计的）：

**四部曲白话**：`socket`=买一个听筒；`bind`=给听筒登记一个号码（端口）；`listen`+`accept`=开机等来电、接起；`recv/send`=通话收发。

```c
int srv = socket(AF_INET, SOCK_STREAM, 0);
bind/listen/accept/recv/send                  /* 一模一样，略 */
```

**Linux 新增杀器：epoll**（高并发必备）。它是内核提供的"监控台账"——把一堆 fd 登记进去，谁有事内核记谁一笔，你只处理"有事名单"。

**epoll 事件循环·时间线分步看**（把"一个线程盯一千个 socket"拆成逐帧动作）：

1. **第 1 步**：`epoll_create1` ——向内核租一个"监控台账"（ep 实例）；
2. **第 2 步**：`epoll_ctl(ADD)` ——把要盯的 fd（如监听 socket）登记进台账，注明"等它可读"；
3. **第 3 步**：`epoll_wait` ——线程在门口**睡着**（不占 CPU，这正是与死循环轮询的本质区别）；
4. **第 4 步**：某 socket 来了数据/新连接——内核把它标为就绪，写进台账的"就绪清单"，并叫醒线程；
5. **第 5 步**：`epoll_wait` 返回就绪清单——线程只处理名单里那几个 fd，处理完回到第 3 步继续睡。

对比 select/poll：它们每次调用都要把**全部** fd 抄给内核、回来再挨个查（千个连接=每次点名千次）；epoll 的登记是一次性的，醒来只给"有事的人"名单——这就是千级连接也扛得住的原因。

```c
int ep = epoll_create1(0);
struct epoll_event ev = { .events = EPOLLIN, .data.fd = srv };
epoll_ctl(ep, EPOLL_CTL_ADD, srv, &ev);     /* 关注 srv 可读 */

struct epoll_event out[16];
int n = epoll_wait(ep, out, 16, -1);        /* 一个线程盯 N 个 fd，谁醒处理谁 */
for (int i = 0; i < n; i++) handle(out[i].data.fd);
```

| 模型 | 场景 |
|------|------|
| 线程 per 连接 | 十几个连接，最直观 |
| select/poll | 几百连接，老 API |
| **epoll** | 千级连接，事件驱动（nginx 同款思路） |

## 7. 终极实验：多线程 TCP 采集服务

**这是什么**：把本篇零件拼成整机——主线程 accept 新连接，每来一个客人开一个线程专门伺候，客人数据落日志。它是 05 篇智能网关 Web 模块的**胚胎版**，先在 PC 上跑通，交叉编译后原样上板。

```c
/* 网关雏形：accept 每个客户端 → 交给线程处理 → 收数据写日志 */
for (;;) {
    int cli = accept(srv, NULL, NULL);
    int *pfd = malloc(sizeof(int)); *pfd = cli;
    pthread_create(&tid, NULL, client_thread, pfd);   /* 每客一线程 */
}
void *client_thread(void *arg) {
    int fd = *(int*)arg; free(arg);
    char buf[256]; int n;
    while ((n = recv(fd, buf, sizeof(buf), 0)) > 0)
        Log_Write(buf, n);                     /* 收到就记（生产者消费者雏形） */
    close(fd); return NULL;
}
```

为什么每客多此一举 `malloc` 一小块内存传 fd？`pthread_create` 是异步的——若直接传 `&cli`，主循环立刻又 accept 覆盖 `cli`，线程里读到的可能已是别人的连接号；每客一份独立内存才互不踩脚。

## 8. 常见坑

怎么读这张表：排查程序"行为不对"时，拿现象来对号入座——左列是出错原因，右列是你看到的症状（**从现象反查原因**）。

| 坑 | 现象 |
|----|------|
| 忘 close fd | 资源耗尽，打开文件失败 |
| fork 后父子共用 fd | 缓冲双写/意外干扰（fork 前刷 fflush） |
| 忘 wait 子进程 | 僵尸进程越积越多 |
| 线程里 return 忘 pthread_exit/join | 主线程退出全队散伙 |
| cond_wait 用 if | 假唤醒导致逻辑错（用 while） |
| 信号处理里 printf/malloc | 非异步安全函数 → 死锁（只置标志） |
| 忽略 EINTR | 系统调用被信号打断返回 -1，误判为错误（重试） |

## 9. 章节小结

- 命令行 20 命令+man 分卷 = 生存能力；一切皆文件，fd 是通用句柄
- fork/exec/wait 进程三件套；pthread 同步四件套与 FreeRTOS 内核对象一一对应
- 信号=异步通知，处理函数只置标志；socket 与 LwIP 同构，epoll 是高并发正解
- **本篇所有能力=08-FreeRTOS 思维的"进程版翻译"**——概念已在你脑中，只是换了 API 名字

## 工程实操任务

- [ ] 在 Ubuntu/WSL 完成全部代码实验，写一个"多线程 TCP echo + 日志线程"小服务
- [ ] 用 `ps/top/netstat` 观察自己程序：线程数、fd 数、内存——建立"运行时体检"习惯
- [ ] 复现僵尸进程（子退出父不 wait），再用 waitpid 修复
- [ ] epoll 版 echo（100 连接压测 vs 线程版内存占用对比）
- [ ] SIGTERM 优雅退出：捕获信号 → 通知线程收尾 → 清理 fd → 退出

## 课后思考题

1. 为什么 Linux 驱动开发前必须先学系统编程？（应用是驱动的"用户"——不知道 open/read 怎么被调，就不知道驱动的 file_operations 为谁而写；测试程序=驱动的验收标准）
2. "一切皆文件"给嵌入式带来什么红利？（串口/LED/传感器统一 open/read/write——应用代码与底层设备解耦；同一套 echo 程序稍改路径就能读传感器——**抽象即复用**）
3. epoll 与 FreeRTOS 队列集（Queue Set）的思想共性？（都是"一个等待点监控多个事件源，谁就绪处理谁"——I/O 多路复用是跨 OS 的通用设计模式，面试能讲出这层=加分）

## 深入资料（想学得更细再看）

- **视频课**：B站搜索「韩顺平 Linux」（入门向，命令行与系统操作讲得细，给第 1 节打底够用）
- **书籍**：《鸟哥的Linux私房菜》基础学习篇（命令行/权限/Shell 入门第一书，对应第 1 节）；《Linux/UNIX系统编程手册》TLPI（系统编程圣经——本篇全部主题都有对应章，进程/线程/信号/socket/epoll 按目录主题检索着读）
