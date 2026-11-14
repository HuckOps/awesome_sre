---
title: "第 8 章 进程管理与系统监控"
---

# 第 8 章 进程管理与系统监控

> 前面几章我们一直在和「文件」打交道：查看、创建、删除、赋权。但服务器真正干活的是「进程（Process）」——nginx 在响应请求、sshd 在等待连接、你的 shell 命令在执行。这一章把镜头从文件转向进程：怎么看到它们、怎么控制它们、怎么在系统变慢时知道是谁在作怪。监控与排障，是 SRE 日常里最高频的功课。

> **本章目标**
> - 能解释程序、进程、PID/PPID、fork/exec 的关系，并用 `ps aux`、`ps -ef` 读懂进程的每一列。
> - 能用 `top` 判断系统是否真的忙（重点是 load average 的正确读法），并用信号和作业控制管理前台/后台/挂起的进程。
> - 能把一个长任务安全地丢到后台持久运行（nohup / tmux），并用 `free`、`vmstat` 等一組命令说清机器的健康狀況。

## 8.1 程序与进程：从一份代码到一个活着的实体

### 8.1.1 程序（Program）vs 进程（Process）

**程序是躺着的，进程是跑着的。**

- 程序（Program）是磁盘上的一个可执行文件，比如 `/usr/bin/nginx`。它是静态的代码与数据，不占 CPU、不占内存，就是一堆字节。
- 进程（Process）是程序被加载进内存、由内核调度运行的**活体实例**。同一个程序可以同时跑出多个进程：比如你开了三个终端窗口跑三个 `sleep 100`，磁盘上只有一个 `sleep` 程序，内存里却是三个进程，各有各的身份编号。

用人话说：程序是菜谱，进程是按菜谱实际炒出来的那一盘菜。菜谱可以抄无数份，菜可以同时炒很多盘。

### 8.1.2 PID 与 PPID

内核给每个进程发一个唯一身份证，叫 **PID（Process ID，进程标识符）**，从 1 开始递增（达到上限后回收复用）。1 号进程是系统里所有进程的祖宗——在 Debian 13 上就是 systemd（详见第 12 章《systemd 与服务管理》）。

每个进程还有 **PPID（Parent PID，父进程标识符）**，记录它是谁生出来的。父子关系构成一棵倒挂的树：

```text
systemd(1)
├─ sshd(812)
│   └─ bash(901)          ← 你 instabox ssh book 进来后的交互 shell
│       └─ sleep(1234)    ← 你敲的命令
├─ nginx(650)
│   ├─ nginx(651)         ← worker 进程，master 生出来的
│   └─ nginx(652)
└─ containerd(540)
```

想看家谱，一条命令就够：

```bash
ps -ef --forest | head -20
```

`--forest` 用 ASCII 线条画出父子关系，哪个进程派生谁一目了然。

### 8.1.3 fork 与 exec：新进程是怎么诞生的

Linux 创建新进程的经典两步是 **fork + exec**：

1. **fork（分叉）**：一个已有进程调用 fork，内核复制出一个与自己几乎一模一样的子进程。此时子进程的代码和父进程相同，只是拿到了新的 PID。
2. **exec（执行）**：子进程调用 exec，把**自己替换成**另一个程序——磁盘上的新程序被加载进内存，从它的入口开始跑。PID 不变，但内核里的「人」已经换了一个。

shell 执行 `sleep 100` 的完整过程：

```text
 你的 shell (PID 901)
      │  你敲下：sleep 100 并回车
      ▼
 fork() ──► 内核复制出一个子进程 (PID 1234，暂时还是 shell 的拷贝)
      │
      │  子进程 exec("/usr/bin/sleep", "100")
      ▼
 子进程「变身」为 sleep (PID 1234)，数 100 秒后退出
      │
      ▼  父进程 wait() 回收子进程，拿到退出码，回到提示符
```

> **SRE 贴士**：写运维脚本时，`$$` 是当前 shell 的 PID，`$PPID` 是父进程的 PID（详见第 13 章《Shell 脚本入门》）。理解了 fork/exec 就能理解两件事：为什么 systemd 是 1 号进程且是万进程之祖；为什么孤儿进程会被 1 号进程「收养」并回收——父进程死了，子进程的 PPID 就变成 1，不会没人管。

<br>

## 8.2 ps：进程的一张快照

**ps（Process Status）告诉你「此刻」有哪些进程。** 它是快照工具，不是监控工具——看实时变化要用 top（下一节）。

ps 有两种完全不同的参数风格，这是新手最容易懵的地方，一次讲透。

### 8.2.1 BSD 风格：ps aux

**先说人话**：`ps aux` 列出系统全部进程，每个进程一行，附带 CPU、内存、状态等关键指标。这是运维排障最常用的一条命令，背下来。

```bash
ps aux
```

逐参数解释：

- `a`：显示所有用户的进程（不只自己的），BSD 风格不带横线。
- `u`：以「用户导向」格式输出，额外显示 USER、%CPU、%MEM、VSZ、RSS、START 这些列。
- `x`：连没有控制终端（TTY 列是 `?`）的进程也显示，也就是显示守护进程（daemon）们。

代表性真实输出（节选）：

```text
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.2 168452 13268 ?        Ss   09:01   0:01 /sbin/init
root       452  0.0  0.1  15792  6144 ?        Ss   09:01   0:00 /lib/systemd/systemd-journald
root       901  0.0  0.3  21404 11264 pts/0    Ss   09:03   0:00 -bash
root       912  0.0  0.1  22672  5644 ?        Ss   09:03   0:00 sshd: bookuser [priv]
bookusr   1234  0.0  0.0   2616  1408 pts/1    R+   09:15   0:00 ps aux
bookusr   1201  2.3  0.4 871224 33520 ?        Sl   09:04   0:05 node server.js
root      1180  0.0  0.1  10816  4212 ?        Ss   09:05   0:00 nginx: master process /usr/sbin/nginx
root      1181  0.1  0.3  11332  7684 ?        S    09:05   0:00 nginx: worker process
root      1105  0.0  0.2  99804 16820 ?        Ssl  09:02   0:00 /usr/bin/dockerd
debian     960  0.0  0.1  24568  9024 ?        Ss   09:01   0:00 /usr/bin/python3 /usr/bin/supervisord
```

逐列解读：

| 列名 | 含义 |
| --- | --- |
| USER | 进程属主。看谁在跑，是排障第一眼。 |
| PID | 进程标识符，kill/观察就找它。 |
| %CPU | 进程自启动以来的**平均** CPU 占用率，不是瞬时的。刚跑了 3 秒的进程显示 2.3%，不代表此刻在吃 CPU。 |
| %MEM | 占物理内存的百分比，按 RSS 算的。 |
| VSZ | 虚拟内存（Virtual Memory Size），进程「认为自己有」的地址空间大小，单位 KB。申请了但没碰的内存也算数，所以这个数通常很大，别被吓到。 |
| RSS | 常驻内存（Resident Set Size），**实际占用的物理内存**，单位 KB。判断谁吃内存看这列。 |
| TTY | 进程的控制终端。`?` 表示没有终端（守护进程/内核线程），`pts/0` 表示挂在某个伪终端上。 |
| STAT | 进程状态码，下面专门列一张表。 |
| START | 启动时间。当天启动显示时刻（如 09:03），跨天显示月日（如 Nov10）。 |
| TIME | 累计占用的 CPU 时间（不是运行时长！），格式 `分:秒`。一个跑了 3 天只占 1 分 CPU 的进程，这里显示 1:00。 |
| COMMAND | 完整命令行。方括号 `[kworker/0:1]` 是内核线程的固定写法。 |

### 8.2.2 System V 风格：ps -ef

```bash
ps -ef
```

逐参数解释：

- `-e`： every process，所有进程（等价于 BSD 的 `ax` 组合语义）。
- `-f`：full 全量格式，输出 PPID、STIME 等列。

输出（节选）：

```text
UID        PID  PPID  C STIME TTY      STAT   TIME CMD
root         1     0  0 09:01 ?        Ss     0:01 /sbin/init
root       452     1  0 09:01 ?        Ss     0:00 /lib/systemd/systemd-journald
root       901   812  0 09:03 ?        Ss     0:00 sshd: bookuser@pts/0
bookusr    912   901  0 09:03 pts/0    Ss     0:00 -bash
bookusr   1234   912  0 09:15 pts/0    R+     0:00 ps -ef
root      1105     1  0 09:02 ?        Ssl    0:00 /usr/bin/dockerd
root      1180     1  0 09:05 ?        Ss     0:00 nginx: master process /usr/sbin/nginx
```

和 `ps aux` 相比，`-ef` 的两大独有优势：

1. 有 **PPID** 列：能直接看出进程父子关系，不用 `--forest` 猜。
2. 有 **C** 列：最近一次调度到的 CPU 使用率概览，以及 UID（属主数字 ID）。

### 8.2.3 两种风格速查对比

| 对比项 | ps aux（BSD） | ps -ef（System V） |
| --- | --- | --- |
| 参数带横线 | 不带横线，直接 `aux` | 带横线，`-ef` |
| 显示全部进程 | a + x 组合 | -e 一步到位 |
| 特色列 | %CPU、%MEM、VSZ、RSS | PPID、C、UID |
| 典型用途 | 看谁吃 CPU/内存（排障首选） | 看父子关系、追查进程来源 |
| 实践建议 | 两个都要会，配合 `grep` 单点确认 | |

> **SRE 贴士**：排障三步走——先用 `ps aux | sort -k3 -rn | head` 找 CPU 大户，再用 `ps -ef | grep 进程名` 确认它的父进程是谁（父进程往往告诉你「这东西是谁拉起来的」：是 sshd 拉的就是你，是 1 号拉起的通常是服务或容器）。两个风格没有优劣，只有场景。

### 8.2.4 STAT 状态码速查表

STAT 列不止一个字母，主状态 + 附加标志拼在一起。主状态：

| 码 | 名称 | 含义 |
| --- | --- | --- |
| R | Running/runnable | 正在 CPU 上跑，或在排队等 CPU。 |
| S | Sleeping（可中断睡眠） | 绝大多数进程的常态：等事件（输入、网络、定时器），来了就能被唤醒。 |
| D | Uninterruptible sleep（不可中断睡眠） | 卡在磁盘/网络 I/O 的系统调用里，不理会大部分信号。**看到一堆 D 状态进程，通常是存储出问题**，这是最危险的状态之一。 |
| T | Stopped | 被挂起（Ctrl-Z 或 SIGSTOP），不运行也不退出。 |
| t | Traced | 被调试器（gdb/ptrace）跟踪。 |
| Z | Zombie（僵尸） | 已退出但父进程还没回收，占着进程表的坑。少量僵尸无妨，持续增长要查父进程。 |
| X | Dead | 已死，即将被回收，瞬间状态，基本抓不到。 |

附加标志（跟在主状态字母后面）：

| 码 | 含义 |
| --- | --- |
| s | 会话领导（session leader），进程组的头。 |
| + | 位于前台进程组，正占着终端键盘。 |
| `<` | 高优先级（nice 值为负），内核更关照它。 |
| `N` | 低优先级（nice 值为正），主动让贤。 |
| `l` | 多线程进程（如 dockerd 显示的 `Ssl`）。 |

> **警告**：见到**大量 D 状态进程**不要上来就 kill。D 状态是在内核里等 I/O，普通信号杀不动，`kill -9` 也得等内核调用返回——正确做法是查存储：`dmesg | tail` 看有没有磁盘报错、用 `iostat -x 1` 看 %util 和 await 是否打满。杀了 D 进程往往只是把现场弄得更乱。

## 8.3 top：全屏实时监控

**先说人话**：`ps` 是一张照片，`top` 是一台监视器——它每秒刷新，动态展示谁在吃 CPU、谁在吃内存、系统负载到哪了。

```bash
top
```

进入全屏界面。先看**头部三行**，它们比进程列表更重要，按 `q` 退出。

```text
top - 09:20:33 up 19 min,  1 user,  load average: 0.36, 0.41, 0.40
Tasks: 102 total,   1 running, 101 sleeping,   0 stopped,   0 zombie
%Cpu(s):  1.8 us,  0.9 sy,  0.0 ni, 97.1 id,  0.0 wa,  0.0 hi,  0.2 si,  0.0 st
MiB Mem :   7876.0 total,   2210.4 free,   2860.8 used,   2804.8 buff/cache
MiB Swap:      0.0 total,      0.0 free,      0.0 used.   4670.9 avail Mem
```

**第一行（任务行）**：

- `09:20:33`：当前时间；`up 19 min`：开机已 19 分钟（和 `uptime` 的 load 同源）。
- `1 user`：当前登录用户数（1 个就是你）。
- `load average: 0.36, 0.41, 0.40`：**本节重点**，见下。

**第二行（Tasks 行）**：进程总数快照——`102 total, 1 running, 101 sleeping`。想看僵尸进程数量，盯这里。

**第三行（%Cpu(s) 行）**，逐一念：

| 字段 | 含义 |
| --- | --- |
| us | user，用户态 CPU 占比（你的程序在算数）。 |
| sy | system，内核态 CPU 占比（内核在干活：调度、系统调用、I/O 处理）。 |
| ni | nice，被降优先级进程占的 CPU。 |
| id | idle，空闲。**真正「CPU 闲」看这数**。 |
| wa | iowait，等磁盘 I/O 的占比。wa 高不是 CPU 忙，是存储跟不上。 |
| hi / si | 硬中断 / 软中断处理占比，网络包处理多时会涨。 |
| st | steal，被 hypervisor（宿主）偷走的时间——云电脑上这个值反映邻居吵闹程度。 |

**load average 三数的正确读法**（面试高频，也是运维基本功）：

`load average: 0.36, 0.41, 0.40` 分别是**过去 1 分钟、5 分钟、15 分钟**的平均负载。它数的是「想跑但拿不到 CPU 的进程数」，即**Running 状态 + D 状态**进程的总数（Linux 把不可中断睡眠也算进负载，这是很多人不知道的细节）。判断标准只有一条：**和 CPU 核数比**。你的云电脑 4 核：

- 4.00 是满分线：load ≈ 核数，说明每个核都有一个活干，接近饱和但还没堵。
- load > 4 持续不降：真的忙不过来，进程在排队，用户体验开始变差。
- load < 4 且 id 很高：系统很闲，别自己吓自己。

> **SRE 贴士**：**load 高不等于 CPU 高**。等待磁盘的 D 进程也会抬高 load。看到 load 15 但 %Cpu 的 id 还有 60%，先查 I/O（wa、D 状态、iostat），别急着扩容 CPU。反过来，load 1.0 在 4 核机器上不高，但如果某个业务延迟飙升，也可能是单线程瓶颈——load 是平均数，不代表每个核都忙。

**进程列表区**：默认按 %CPU 降序，列有 PID、USER、PR、NI、VIRT、RES、SHR、S、%CPU、%MEM、TIME+、COMMAND。两个关键内存列：

- **VIRT**：虚拟内存，同 ps 的 VSZ，含申请未用的部分，虚胖。
- **RES**：实际物理内存，同 ps 的 RSS，认这个。
- **SHR**：Shared Memory，可共享的内存（共享库等），不是独占的。

**常用快捷键**：

- `P`（大写）：按 CPU 排序，找吃 CPU 的进程（默认）。
- `M`（大写）：按内存排序，找吃内存的进程。
- `1`（数字一）：展开显示**每颗核心**的 %Cpu 行，看负载是压在一个核上还是铺满四核——排查单线程程序必用。
- `E`：切换内存单位（KiB/MiB/GiB）。
- `k`：交互式 kill，输入 PID 再选信号（先 15 后 9，见下节）。
- `r`：交互式 renice，调整优先级。
- `c`：切换显示完整命令行还是进程名。
- `h` / `?`：帮助。

### htop：更现代的同类工具

Debian 默认不带 htop，一条命令安装：

```bash
apt install -y htop
```

`htop` 用色块和树形展示，支持鼠标点选、横向滚动、一键按 F9 发信号，比 top 友好得多。但**生产服务器上它未必装着**，紧急排障时你面对的是裸 top——所以本书以 top 为主，htop 知道有这回事、会基础操作即可。

> **SRE 贴士**：**生产 vs 实验室**——你的云电脑可以 `apt install htop` 图舒服，但生产服务器装任何东西都是变更：要走流程、要考虑攻击面、要考虑版本兼容。最小安装原则下的真实世界里，你手里的牌常常只有 top、ps、/proc。这就是为什么本章花大力气讲 top 而不是教大家「装个 htop 就行了」。详见第 9 章《软件包管理》的最小安装原则。

## 8.4 信号与 kill：进程之间的礼貌与强制

### 8.4.1 什么是信号

**信号（Signal）是进程间通知的标准化方式**：内核或某个进程给目标进程发一个编号，目标进程按约定做出反应——或者走人，或者停下，或者重读配置。进程可以对「大部分」信号安装自己的处理函数，但有三个信号（9 KILL、19 STOP、18 CONT 的默认行为）不能捕获、不能忽略，这是内核的底线。

常用信号表：

| 编号 | 名称 | 默认行为 | 典型用途 |
| --- | --- | --- | --- |
| 1 | HUP（HangUP） | 终止（但可被捕获） | 重载配置：nginx -s reload、sshd 重读配置都走它 |
| 2 | INT（Interrupt） | 终止 | 键盘 Ctrl-C 发的就是它 |
| 15 | TERM（Terminate） | 终止 | **kill 的默认值，优雅退出**：程序可捕获它来保存状态、关闭句柄 |
| 9 | KILL | 立即终止 | 强杀，不可捕获、不可忽略、不可等待 |
| 18 | CONT（Continue） | 继续运行 | 唤醒被 STOP 的进程 |
| 19 | STOP | 暂停 | 冻结进程，不可捕获、不可忽略 |
| 3 | QUIT | 终止并 core dump | Ctrl-\ 触发，用于调试崩溃 |

### 8.4.2 kill 命令家族

```bash
kill -TERM 1234      # 发 15 号信号给 PID 1234（-TERM 可省略，默认就 15）
kill -9 1234         # 强杀，不可捕获
kill -HUP 1180       # 通知 nginx master 重载配置
killall nginx        # 按进程名杀，杀死所有 nginx（等价于对每个 nginx 进程 kill）
pkill -f "node server.js"   # -f 按完整命令行模式匹配，一行命令干翻一类
pkill -u bookusr     # 杀某个用户的全部进程（谨慎！）
```

逐参数解释：

- `kill`：默认发 15（TERM）。信号名大小写均可（`-TERM` / `-term` / `-15` 等价）。
- `killall 进程名`：精确按可执行文件名杀，比 `pkill` 严谨（不会误伤命令行里带同名子的进程）。
- `pkill -f 模式`：`-f` 匹配**完整命令行**而非进程名，`pkill -f "node server.js"` 能杀掉 `node server.js`，而 `pkill node` 只匹配进程名是 node 的。**-f 威力大也危险**，模式写松了会误杀（比如 `-f "python"` 可能命中一堆无关脚本）。

**运维礼仪：先 TERM 后 KILL。** `kill -9` 是掀桌子：进程没有机会 flush 缓冲区、关闭数据库连接、清理临时文件——被杀的一瞬间，可能留下半写的日志、半落库的数据、泄漏的锁。正确姿势永远是：

```bash
kill 1234          # 先礼：给 5~10 秒让它自己体面退出
sleep 5
ps -p 1234         # 还活着，查一下状态
kill -9 1234       # 后兵：确认它不听话再强杀
```

**killall / pkill 的常见坑**：`killall firefox` 杀不掉名叫 `firefox-bin` 的进程；`pkill -f java` 可能一次干掉三个不同业务的 JVM。执行这类命令前，**先 `pgrep -a 模式` 干跑一遍确认匹配清单**（`pgrep` 只列不杀，是 pkill 的安全预演版），再动手。

> **警告**：`kill -9` 前先问自己三个问题：它是什么服务（systemd 管的应该用 systemctl restart 而不是杀）？杀了会不会丢数据？有没有更温柔的方式（TERM、HUP、systemctl reload）？**生产环境不要以 root 身份随手 kill**——杀错一个进程的代价，比你多等五秒的代价大得多。最小权限原则详见第 5 章《用户、群组与权限》。

### 8.4.3 一个完整例子

nginx 改了配置要生效，规范做法是发 HUP 给 master 进程（worker 收到后优雅重启，不断连接）：

```bash
ps -ef | grep nginx | grep -v grep     # 先找到 master 的 PID 1180
kill -HUP 1180                          # 通知重载
```

要停掉一个跑飞的 node 进程（PID 1201）：

```bash
kill 1201            # 先 TERM
sleep 3
ps -p 1201           # 确认是否退出，输出为空即已退出
```

若它赖着不走，再 `kill -9 1201`。生产上由 systemd 托管的服务（详见第 12 章）应统一走 `systemctl restart`，让 unit 文件说了算，而不是手动杀进程。

## 8.5 前台、后台与「甩不掉」的长任务

### 8.5.1 前台、挂起与后台：一份终端的时间管理

默认情况下，命令占着你的终端直到跑完，这叫**前台（foreground）**执行。三个基本操作：

- `命令 &`：启动即丢**后台（background）**，终端立刻回到提示符，可以继续敲别的命令。
- `Ctrl-Z`：把**正在前台运行**的进程**挂起（suspend）**——它变成 Stopped（STAT 里显示 T），不运行也不退出，像按了暂停键。
- `jobs`：查看当前 shell 管理的后台/挂起作业列表，`%1`、`%2` 是作业号。
- `fg %1`：把作业 1 调回前台（Ctrl-Z 后想恢复运行用它）。
- `bg %1`：让挂起的作业 1 在后台继续跑起来。

完整演示：

```bash
ping -c 100 223.5.5.5 &
# [1] 2341        ← shell 报告：这是 1 号作业，PID 2341

sleep 300
# 跑了一会儿你后悔了，按 Ctrl-Z
# [2]+  Stopped                 sleep 300

jobs
# [1]-  Running                 ping -c 100 223.5.5.5 &
# [2]+  Stopped                 sleep 300

bg %2          # 让 sleep 300 去后台继续数数
fg %1          # 把 ping 调回前台盯着它，Ctrl-C 随时可杀
```

关键概念：**作业（job）是 shell 的概念，进程是内核的概念**。`%1` 是作业号，`2341` 是 PID；作业号只在当前 shell 内有效，PID 全局有效。

> **SRE 贴士**：直接 `&` 丢后台的任务有个隐蔽坑——它的 stdout/stderr 仍然接着你的终端。你在终端里做别的操作，它的输出会时不时插进来刷屏；更糟的是，终端一关（SSH 断开），内核会给这个进程组的每个进程发 SIGHUP，默认行为是**终止**。忙活了半小时的后台任务，断线一秒归零。

### 8.5.2 让任务活过终端：nohup、setsid、disown

SSH 断开会发 SIGHUP，这是后台任务夭折的根因。三个应对武器，一次讲清区别：

| 工具 | 机制 | 典型场景 |
| --- | --- | --- |
| `nohup 命令 &` | 让进程**忽略** SIGHUP，同时把输出重定向到 `nohup.out`（或你指定的文件） | 一次性长任务：数据导出、批量脚本 |
| `setsid 命令` | 让进程**脱离当前会话**，成为新会话的首领，压根收不到终端的 SIGHUP | 需要彻底独立于终端的常驻任务 |
| `disown` | 把已启动的作业从 shell 作业表里除名，shell 退出时不再给它发 SIGHUP | 任务已经 & 起来了才想起要保命 |

三者常配合使用：`nohup setsid 命令 > 日志 2>&1 &` 是最稳的组合拳。

> **SRE 贴士**：**输出一定要落到文件。** `nohup` 不指定输出会写进当前目录的 `nohup.out`，家目录里莫名其妙多出一个文件就是这么来的；正式用法永远是显式重定向：`> 任务日志 2>&1`，把标准输出和错误输出都收进日志文件。事后排障全靠它。

### 8.5.3 InstaBox 平台限制：为什么本章对你尤其重要

你的云电脑执行环境有个硬约束：**`instabox exec` 单条命令上限 10 分钟，输出跑完才整段返回，不流式**（详见第 1 章《搭建实验环境》）。这意味着：

1. 任何可能超过 10 分钟的任务，都不能放在 `instabox exec` 里干等——超时直接砍掉（退出码 124）。
2. 即使不超时，干等 10 分钟也看不到中间输出，无法判断进度。
3. 标准姿势：**长任务丢后台 + 写日志文件 + 事后 tail 查看**，三步走。

完整可复现实例——用 nohup 跑一个耗时循环，然后查看日志：

```bash
nohup bash -c 'for i in $(seq 1 60); do echo "$(date "+%F %T") 第 $i 次心跳"; sleep 5; done' > /data/home/heartbeat.log 2>&1 &
# [1] 2468   ← shell 汇报作业号与 PID；若看到 nohup: ignoring input 属正常提示
```

这条命令的含义拆开看：`nohup` 保证它不怕 SIGHUP；`bash -c '...'` 里是一个 60 次、每次间隔 5 秒的循环（总时长约 5 分钟，用来模拟「确定会超过耐心、但在 exec 里也能跑完」的任务）；输出全部写进 `/data/home/heartbeat.log`（持久盘，机器重启不丢——数据放持久盘是铁律，详见第 2 章）。

启动后立刻回到提示符，你可以断开、去干别的。想看进度，随时：

```bash
tail -f /data/home/heartbeat.log
```

`tail -f` 会持续跟踪文件新增内容并刷出来（Ctrl-C 退出查看，不影响后台任务）。看几眼进度、确认它在正常跑，Ctrl-C 退出即可。如果嫌 `-f` 刷屏，`tail -n 20 文件` 看最后 20 行也行。

任务跑完后，日志文件里有完整时间线——这就是「事后审计」的依据：几点开始的、每次间隔是否均匀、有没有报错。

```bash
wc -l /data/home/heartbeat.log        # 60 行即全部跑完
grep -c 心跳 /data/home/heartbeat.log  # 统计条数，验证没有丢
```

> **SRE 贴士**：把这套「nohup + 日志 + tail -f」变成肌肉记忆，它是在 InstaBox 上开发的所有长任务的默认形态。真实生产环境里，比 nohup 更规范的载体是 systemd 服务（详见第 12 章）或容器——但掌握「任务与终端解耦、输出与文件绑定」这个核心思想，比记住具体命令更重要。

### 8.5.4 tmux：比 nohup 更香的终端 multiplexer

nohup 解决了「任务不死」，但任务对你是个黑盒：想看实时输出只能 tail，想中途给它发个输入没门。**tmux（Terminal MUltipleXer）换个思路：让你的终端会话本身活在后端服务里**，SSH 断了会话还在，重连回去接着用——任务、输出、输入，全须全尾地保存。

在 tmux 的世界里有三级结构：**会话（session）→ 窗口（window）→ 窗格（pane）**。一个会话是一个持久工作现场；一个会话可以开多个窗口（像浏览器的标签页）；一个窗口可以切分成多个窗格（同屏并列）。所有操作的魔法前缀是 **Ctrl-b**（同时按下 Ctrl 和 b，松开，再按下一个键）。

必会命令表：

| 操作 | 命令 / 按键 |
| --- | --- |
| 新建会话 | `tmux new -s work`（-s 起名叫 work） |
| 列出会话 | `tmux ls` |
| 接回会话 | `tmux attach -t work`（-t 指定名字） |
| 分离会话（任务继续跑） | `Ctrl-b d`（detach） |
| 杀会话 | `tmux kill-session -t work` |
| 新建窗口 | `Ctrl-b c`（create window） |
| 切窗口 | `Ctrl-b n`（next）/ `Ctrl-b p`（previous）/ `Ctrl-b 0~9` |
| 关窗口 | `Ctrl-b &`（确认后关闭） |
| 横向切窗格 | `Ctrl-b "` |
| 纵向切窗格 | `Ctrl-b %` |
| 窗格间移动 | `Ctrl-b 方向键` |
| 重命名会话 | `tmux rename-session -t work newname` |

典型工作流：SSH 进云电脑 → `tmux new -s work` → 在里面跑长任务 → 按 `Ctrl-b d` 分离（detach）走人，或直接关掉 SSH 窗口 → 任务在 tmux 会话里继续跑 → 下次 SSH 进来 `tmux attach -t work`，现场和你离开时一模一样，滚动历史都在。

**和 InstaBox 共存的注意点**：空闲 30 分钟自动暂停（免费档锁死，不可调）。tmux 能保会话，但保不住**被暂停的机器**——暂停时整个 VM 冻结，tmux 会话随之冻结，恢复后原地继续。所以：长时间挂机前，要么保持机器有活动（比如任务本身在跑、有日志在写），要么接受「回来时看到机器刚被唤醒」这个事实。suspend/resume 的细节详见第 1 章。

> **SRE 贴士**：**生产 vs 实验室**——真实生产环境并不鼓励「手动 tmux 挂着跑业务」，常驻进程应该由 systemd 托管（崩溃自动重启、开机自启、日志进 journal，详见第 12 章）。tmux 在生产里的正确岗位是：**你的运维操作台**——现场排障时多层窗格同屏看日志、改配置、盯进程，断线重连不丢现场。在云电脑上它一身兼两任，但你心里要分清这两个角色。

## 8.6 系统监控命令群像

top 之外，还有一组轻量命令各管一摊。下面按「内存 → 综合 → I/O → 内核视角」过一遍。

### 8.6.1 free -h：内存到底还剩多少

```bash
free -h
```

`-h`（human）自动换算成 KiB/MiB/GiB 人类可读单位。输出：

```text
               total        used        free      shared     buff/cache   available
Mem:           7.7Gi       1.1Gi       4.9Gi        12Mi       2.2Gi       6.3Gi
Swap:             0B          0B          0B
```

逐列讲透：

| 列 | 含义 |
| --- | --- |
| total | 物理内存总量（8 GiB 机器显示 7.7Gi，被内核和固件占一点点，正常）。 |
| used | **进程实占**的内存（RSS 之和，扣除可回收部分）。 |
| free | 完全没人碰的内存。 |
| shared | 多进程共享的内存（tmpfs、共享内存段），通常很小。 |
| buff/cache | 块缓存 + 页缓存：读过的文件内容、待写盘的缓冲。**这不是浪费，是加速器**——内存闲着也是闲着，不如拿来做缓存，应用要时立刻吐出来，行话叫「取之不尽」。 |
| available | **应用程序实际还能申请到多少内存**，估算值 = free + 大部分 buff/cache。 |

**判断内存够不够，只看 available 这一列。** 经典的误判是看到 `free` 小就喊「内存不够了要加内存」——实际上 buff/cache 随时可以让出来。反过来，used 持续贴着 total、available 趋近 0，才是真紧张，接下来就是 OOM Killer 出场（内核挑个倒霉进程杀掉，详见第 12 章排障）。

第 2 章验货时就用过它，Swap 为 0 是 InstaBox 模板的现状——第 10 章《磁盘与存储管理》会教你按需加 swapfile。

### 8.6.2 vmstat 1 5：五分钟健康体检

```bash
vmstat 1 5
```

第一个参数是**采样间隔（秒）**，第二个是**采样次数**——每 1 秒打一行，共 5 行。第一行是开机至今的平均值，看趋势以后面几行为准。

```text
procs -----------memory---------- ---swap-- -----io---- -system-- -------cpu-------
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 2  0      0 5132496 168412 2215800    0    0    42    18   65  120  2  1 97  0  0
 1  0      0 5132000 168360 2215820    0    0     0    24   58  110  1  1 98  0  0
 0  0      0 5131900 168380 2215840    0    0    12     8   60  108  1  1 98  0  0
```

重点列：

- `r`（running）：等 CPU 的进程数，**长期 > CPU 核数就是 CPU 瓶颈**。
- `b`（blocked）：不可中断睡眠（D 状态）进程数，持续 > 0 多半在等 I/O。
- `swpd / si / so`：swap 使用量与换入换出速率。si/so 持续非 0 说明内存在换页，性能已经在受损。
- `bi / bo`：块设备读入/写出的块数（KB/s 口径以版本为准），磁盘 I/O 压力参考。
- `in / cs`：每秒中断数 / 上下文切换数。cs 飙高往往意味着进程/线程切换过于频繁。
- `us / sy / id / wa / st`：CPU 五大项，和 top 第三行同一套口径。

vmstat 的定位：**一条命令同时看 CPU、内存、I/O、中断**，SSH 进一台陌生机器先跑它，30 秒建立第一印象。

### 8.6.3 iostat -x 1：磁盘视角（需要 sysstat）

```bash
apt install -y sysstat
iostat -x 1
```

Debian 默认没有 iostat，装 sysstat 包（Debian 13 的模板镜像里可能已随基础工具带入，以你的机器为准；没有就装）。`-x` 输出扩展统计：

```text
Linux  (book)  11/13/2026  _x86_64_  (4 CPU)

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           1.90    0.00    0.90    0.10    0.00   97.10

Device            r/s     rkB/s   w/s     wkB/s   dqm/d  ... %util  await
vda              3.20     84.10   6.40    110.20  0.00   ...  1.20   0.85
```

粗读两条就够入门：

- **%util**：设备饱和度，100% 表示磁盘一直在干活，没有余量。云电脑的持久盘是网络存储，%util 高 + await 高说明存储到瓶颈了。
- **await**：每个 I/O 请求的平均等待时间（毫秒），值越大，应用「等磁盘等得心慌」的感觉越强。

**si/so 和 %util 要放一起看**：vmstat 的 si/so 说「内存在换页」，iostat 说「是不是存储的锅」，两个一对，I/O 瓶颈的原凶就跑不掉。

### 8.6.4 /proc：内核的档案室

`/proc` 是个伪文件系统（概念详见第 3 章），文件内容全是内核实时吐出的数据。不依赖任何工具，cat 就能看：

```bash
cat /proc/loadavg    # 0.36 0.41 0.40 1/102 12345 —— 前三个数同 load average；1/102 是 running/总进程数；末位是最后运行的 PID
cat /proc/meminfo    # 几百行内存明细：MemTotal、MemFree、MemAvailable、Buffers、Cached、SwapTotal……
cat /proc/cpuinfo    # 每颗核心一段：model name、flags、cpu cores。nproc 的信息源头就在这
```

排障三剑客用法：监控脚本采样 load（`cat /proc/loadavg | awk '{print $1}'`），精确到某项内存指标（`grep MemAvailable /proc/meminfo`），或者确认 CPU 特性。

### 8.6.5 dmesg 与 uptime

```bash
dmesg | tail -20     # 最近 20 条内核消息：磁盘错误、OOM、USB/网络事件
```

dmesg（display message）读的是内核环形缓冲区——系统发生的「硬件级大事」都记在这。I/O 报错、网卡掉线、内存不足杀进程（`Out of memory: Killed process 1234 (node)`），第一时间来这里翻。**普通容器里未必有权限**，云电脑 root 身份随便看（这也是最小权限原则的另一面：权限大看得多，也错得多）。

```bash
uptime
```

```text
 09:31:02 up 32 min,  1 user,  load average: 0.36, 0.41, 0.40
```

uptime 就是把 top 第一行单独搬出来，三个数同样是 1/5/15 分钟负载。写监控脚本时它是采样 load 最轻量的方式（更轻的是读 `/proc/loadavg`）。

## 8.7 实战：定位一个吃 CPU 的进程

**场景**：监控告警说云电脑负载飙到 8（4 核机器，满载是 4），你 SSH 进去时机器卡顿，敲命令都反应慢。完整流程：

**第一步：top 定位嫌疑进程**（30 秒）

```bash
top
```

看头部：load average 是不是真的高（比如 `3.85, 5.20, 6.01`——三数递增说明负载还在涨）；%Cpu 的 id 是不是快归零。按 `P` 确认按 CPU 排序，看进程列表前几行是谁。

```text
top - 09:40:11 up 51 min,  1 user,  load average: 3.85, 5.20, 6.01
Tasks: 105 total,   3 running, 102 sleeping,   0 stopped,   0 zombie
%Cpu(s): 88.5 us,  6.2 sy,  0.0 ni,  4.1 id,  1.0 wa,  0.0 hi,  0.2 si,  0.0 st
MiB Mem :   7876.0 total,   4102.6 free,   2850.8 used,    922.4 buff/cache
MiB Swap:      0.0 total,      0.0 free,      0.0 used.   4660.4 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
 2468 bookusr   20   0   12368   3072   2600 R  99.3   0.0   4:51.06 bash
 1201 bookusr   20   0  871224  33520  12800 S   2.3   0.4   0:05.12 node
```

一分钟内锁定：PID 2468 的 bash，%CPU 99.3，跑了 4 分多钟——就是它。load 6 > 4 证实系统在排队。

**第二步：ps 交叉验证，确认身份与来源**（30 秒）

```bash
ps -fp 2468
```

```text
UID        PID  PPID  C STIME TTY      STAT   TIME CMD
bookusr   2468     1  0 09:35 ?        R     4:51 bash -c for i in $(seq 1 99999999); do echo $i; done
```

PPID 是 1：它的原始终端已经没了（多半是之前 nohup 丢后台、终端断开后任务被 1 号进程收养），是个跑死循环的脚本。

**第三步：决策与处置**（1 分钟）

三个选项按优先级评估：

1. 这是什么任务？我认识它吗？如果是自己的实验脚本（本例就是 8.5.3 的同款循环，写崩了），可以直接杀。
2. 如果是业务进程——**先 TERM 后 KILL**：`kill 2468`，等 5 秒确认；杀不掉再 `kill -9 2468`。
3. 如果是 systemd 托管的服务，正确动作是 `systemctl restart 服务名`（详见第 12 章），而不是裸杀进程——杀了它，systemd 会按 unit 配置重启或放弃，现场不可控。

本例处置：

```bash
kill 2468         # 先 TERM；没有捕获TERM习惯的业务程序给它 5 秒；确认不退再 kill -9
sleep 3
ps -fp 2468       # 输出为空 = 已退出
top              # 再看头部：load 开始往下走
```

**第四步：复盘记录**（2 分钟）。kill 完不算完，SRE 的闭环是「写下来」：什么进程、谁拉的、为什么跑飞、怎么处置的、下次怎么防（改成有退出条件的脚本、丢 tmux 里盯着跑、或者交给 systemd 托管）。没有这条，下次同样的事还会发生。

> **SRE 贴士**：**生产 vs 实验室**——上面的流程你可以在云电脑上随便演练；真实生产中，「kill 一个陌生进程」前还要多问几句：它是别的同事或别的服务的吗（查 `ps -ef --forest` 看父进程与用户）？现在是不是业务高峰（kill 会不会引发故障，详见第 12 章 systemd 的托管和保护）？处置要不要先报备（变更管理）？云电脑弄坏就重建，生产里一个手滑就是工单和问责。练手可以放开，上生产先收着。

## 本章小结

这一章把镜头从文件转向进程，四件事请带走：

1. **进程是跑着的程序**，靠 PID 认人、靠 PPID 找爹；fork + exec 两步诞生新进程；`ps aux`（BSD 风格，看 CPU/内存）和 `ps -ef`（System V 风格，看父子关系）是两大快照工具，STAT 状态码表（R/S/D/T/Z + s/+/<、N）要会念。
2. **top 的读法**：头部三行（负载/任务/CPU）比进程列表更重要；load average 要和 CPU 核数比，load 高不等于 CPU 忙，先排 I/O；P/M/1 三个快捷键覆盖日常。
3. **信号的礼仪**：先 TERM（15）后 KILL（9），HUP 用于优雅重载；killall 按名杀、pkill 按模式杀，杀前先 `pgrep` 预演；生产上 systemd 托管的服务用 systemctl 管理。
4. **任务与终端解耦**：`&`、Ctrl-Z、`jobs`/`fg`/`bg` 是入门，`nohup`/`setsid`/`disown` 保命，tmux 提供完整的工作现场；叠加 InstaBox 的 exec 10 分钟上限，「后台 + 日志 + tail」是本书环境的标准长任务姿势。free -h 只看 available、vmstat 1 5 看 r/b/si/so、iostat 看 %util/await、dmesg 看内核大事——这套组合够你应对九成日常巡检。

## 动手练习

1. **认进程**：在云电脑上运行 `ps aux`，挑出 PID 最小的 3 个进程和 1 号进程，分别说明它们的 USER 和 COMMAND；再用 `ps -ef --forest` 确认 1 号进程是谁。
2. **读 top**：运行 `top`，记录此刻的 load average 三个数、%Cpu(s) 各字段值；按 `1` 展开每核，看看 CPU 负载铺在几个核上；按 `M` 找出吃内存最多的进程，记下它的 RES 值。退出后画一张「头部三行速查卡」。
3. **状态码**：用 `sleep 1000 &` 丢一个后台任务，`jobs` 看它的状态；Ctrl-Z 挂起它，看 `ps` 里 STAT 变什么；再用 `bg %1` 让它复活，`kill %1` 收尾。
4. **长任务实战**：按 8.5.3 的写法，用 nohup 跑一个向 `/data/home/ex08.log` 写 30 行带时间戳的循环（每次间隔 3 秒），启动后立刻断开 SSH 重连，用 `tail -f` 看 10 秒，Ctrl-C 退出查看；任务跑完后用 `wc -l` 验证行数。
5. **tmux 入门**：`tmux new -s book8`，在里面跑一个 `sleep 600`；Ctrl-b d 分离，`tmux ls` 确认会话还在；退出 SSH 再重连，`tmux attach -t book8` 接回现场；最后 Ctrl-b d 分离后 `tmux kill-session -t book8` 清理。
6. **综合排障演练**：故意用 `bash -c 'while :; do :; done' &` 造一个吃满一核的进程，top 确认它，用 8.7 节四步法定位、处置、并在 `/data/home/ex08-report.txt` 写下复盘（进程名、PID、来源判断、处置方式、防再发）。

> **参考答案**：
> 1. PID 1 在 Debian 13 是 systemd（`/sbin/init` 软链或 systemd 本体）；小 PID 进程多为 systemd 拉起的系统服务（journald、dbus 等），USER 多为 root 或系统账户。`ps -ef --forest` 中 PID 1 位于最顶端。
> 2. 4 核机器 load 3 个数都 < 4 即健康；id 高说明空闲；按 `M` 的 %MEM 列找内存大户（多半是 node 或 dockerd），RES 是它的实际物理内存。
> 3. `sleep 1000 &` 默认 STAT 是 S（Sleeping）；Ctrl-Z 后变 T（Stopped）；`bg %1` 后回到 Running 但仍在后台（ps 显示 S/R，jobs 显示 Running）；`kill %1` 用默认 TERM 收掉。
> 4. 关键点：日志路径落在 `/data/home`（持久盘，别用系统盘）；`tail -f` 的 Ctrl-C 只退出查看不杀任务；`wc -l` 应为 30。
> 5. `tmux ls` 显示 `book8: 1 windows (created ...)`；detach 后 sleep 600 继续在跑；attach 后现场原样；最后用 kill-session 收尸，别让僵尸会话越攒越多。
> 6. `while :; do :; done` 是经典烧核循环，%CPU 接近 100；四步法：top 定位 → `ps -fp PID` 确认（PPID 与 CMD）→ 决策（本例是实验进程，TERM 即可，bash 默认不吃捕获信号，观察不到退出迹象再升级 -9）→ 复盘落盘。体会「先礼后兵」和「systemd 服务别裸杀」两个原则。



