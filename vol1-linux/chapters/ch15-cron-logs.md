---
title: "第 15 章 定时任务与系统日志"
---

# 第 15 章 定时任务与系统日志

> 第 14 章你学会了把重复劳动写成脚本，但脚本不会自己跑——总得有人记得在凌晨两点打开终端。第 15 章上半场解决「谁半夜来按按钮」：cron 与 systemd timer，让时间成为触发器。下半场解决「出事之后怎么破案」：日志。一台没人看的系统和一台没人写日志的系统，故障时的命运是一样的——靠猜。这一章结束，你的系统第一次具备「白天黑夜都自动运转、出了事能复盘」的形态，这也是第 17 章《综合实战：从零部署并加固一个 Web 服务》里健康检查和备份的地基。

> **本章目标**
> - 能写正确的 crontab 五段式（含 `*`、`*/5`、`1-5`、`a,b` 列表与周字段 0/7），说清 PATH、`%` 转义、输出重定向三个坑，并能用 `crontab -e -l -r` 安全管理任务。
> - 能用 `journalctl`（`-e -f -u -b --since/--until -p`）和 `/var/log` 三剑客回答「这个服务昨晚发生了什么」，并用 `logger`、`logrotate` 给自己的脚本接上日志与轮转。
> - 能按「curl → ss → systemctl status → journalctl --since」的时间线独立完成一次网站故障定位，并说出 cron 与 systemd timer 的分工与选择依据。

<br>

## 14.1 cron 是什么：让时间当触发器

先建立分工概念，这两种自动化在日常生活中天天在跑，但机制完全不同：

- **systemd 服务（service）**：第 13 章《systemd 与服务管理》的主角。它管的是「常驻进程和开机启动」——开机时起来、崩溃了拉起来、依赖就绪才启动。**事件与依赖驱动**。
- **cron**：本章上半场。它只管一件事——**到点执行命令**。时间驱动，不关心开机顺序，也不负责「进程挂了帮我拉起来」。

一个形象的比喻：systemd 像物业（负责小区长期运转、谁家的水电出问题都归它维护），cron 像闹钟（只负责在指定的时间叫你，叫醒之后你怎么起床它不管）。实际场景里两者经常搭档：cron 每天凌晨触发一次备份脚本（脚本是第 14 章的 `backup-home.sh`），而备份脚本要依赖的数据库服务由 systemd 保证常驻。

Debian 13 的默认安装里 cron 守护进程（daemon）通常已经就位，先验证：

```bash
command -v crontab && systemctl status cron | head -5
```

```text
/usr/bin/crontab
● cron.service - Regular background program processing daemon
     Loaded: loaded (/usr/lib/systemd/system/cron.service; enabled)
     Active: active (running) …
```

（没有 crontab 就 `apt install cron`，安装方法详见第 10 章《软件包管理：apt 与 dpkg》；服务状态看不懂就回第 13 章复习 `systemctl status` 的三行信息。）cron 的工作方式：`cron` 守护进程常驻内存，每分钟醒一次，翻一遍所有 crontab，「现在该执行的」就启动对应的用户 shell 跑命令，跑完把结果（如果有输出）尝试寄一封邮件给任务主人。

<br>

## 14.2 crontab 五段式：逐字段拆开看

crontab（cron table，cron 的任务表）里一行就是一个任务，格式是**五段时间 + 命令**：

```text
分    时    日    月    周      命令
*     *     *     *     *      command
0-59  0-23  1-31  1-12  0-7
```

列一张对照表，逐字段讲清边界：

| 字段 | 取值范围 | 记忆法 | 例子 |
| --- | --- | --- | --- |
| 分（minute） | 0-59 | 一小时的 1/60 | `*/5` 每 5 分钟；`30` 第 30 分 |
| 时（hour） | 0-23 | 24 小时制，没有上午下午 | `2` 凌晨 2 点；`*/2` 每 2 小时 |
| 日（day of month） | 1-31 | 一个月里的第几天 | `1` 每月 1 号；`*/10` 每 10 天 |
| 月（month） | 1-12 | 数字即可，也认 jan…dec 英文缩写 | `1` 一月；`1,7` 一月和七月 |
| 周（day of week） | **0 和 7 都是周日**，1 是周一…6 是周六 | 周末是 0/6 或 7/6 | `0` 周日；`1-5` 周一到周五（工作日） |

**周字段的 0 和 7 相等**这是规定，没有为什么，记住别踩。其余字段 0 都是合法值，只有周是例外从 0（周日）开始。

### 14.2.1 五种写法：*、*/n、a-b、a,b 和单值

时间字段的五种原子写法：

```text
*      每个值都命中（每分钟、每小时……）
*/5    每 5 个单位（步长 step）——第 0、5、10…分钟
1-5    范围（range）——周一到周五
1,3,5  列表（list）——周一、周三、周五
30     单值——第 30 分钟
```

组合出真实世界的例子，逐条念一遍就懂了：

```text
# 每天凌晨 2:00 备份
0 2 * * *   /data/home/scripts/backup-home.sh

# 每 5 分钟一次健康检查
*/5 * * * *  /data/home/scripts/check.sh >> /data/home/logs/check.log 2>&1

# 工作日（周一到周五）早上 8:30 发日报
30 8 * * 1-5   /usr/local/bin/daily-report.sh

# 每月 1 号和 15 号的中午 12 点做账
0 12 1,15 * *  /usr/local/bin/billing.sh

# 每周日凌晨 3:15 清理临时文件
15 3 * * 0   find /data/home/tmp -type f -mtime +7 -delete
```

注意最后一条的 `find ... -delete`：**第 5 章《文件与目录操作》说过 `-delete` 之前先 `-print` 预演**——写进 cron 的删除命令尤其如此，cron 不会跟你确认，跑偏一个月，证据都凉了。

五段之外别忘了**第六段的灵魂：命令本身**。命令要么绝对路径（`/data/home/scripts/...`），要么接受 14.6 的 PATH 坑——没有当前目录概念，别写 `./script.sh`。

不确定某段写得对不对？第 4 章说的帮助三件套在这里的终极形态是 `man 5 crontab`——**注意是 5 不是 1**，这是「文件格式手册」的节号（manual section 5），`man 1 crontab` 讲的是命令行工具本身（-e/-l/-r），两个都要看，别只看一个就以为读完了。

<br>

## 14.3 crontab 命令与编辑器那点事

管理自己的任务表用 `crontab` 命令，三个子命令：

| 命令 | 作用 |
| --- | --- |
| `crontab -e` | 编辑（edit）当前用户的任务表 |
| `crontab -l` | 列出（list）当前任务，只读 |
| `crontab -r` | **删除（remove）整个任务表**，没有任何确认 |
| `crontab -u bookuser -l` | 以管理员身份查看别人（root 专属） |

### 14.3.1 crontab -e 会打开一个你可能不认识的编辑器

`crontab -e` 首次运行，常常直接给你一个没见过全屏编辑器——不是没装 nano/vim，是它用了系统默认编辑器（Debian 上常常是 nano 或 vim.tiny，取决于有没有装 vim）。两种解法：

```bash
EDITOR=nano crontab -e        # 一次性指定本次用 nano
echo 'export EDITOR=nano' >> ~/.bashrc   # 或者写进 bashrc，永久生效（第 14 章说过环境变量）
```

nano 的用法很简单：方向键移动，直接打字，底部一排 `^O`（Ctrl-O 保存）、`^X`（Ctrl-X 退出）、`^K` 剪行）。如果是 vim：`i` 进插入模式，写完 `Esc` 再 `:wq` 回车（全套指法见第 8 章《vim 编辑器》）。

**crontab -e 保存退出时会自动做语法检查**，某行写错了会提示 `errors in crontab file, can't install`，这时候选「重新编辑」而不是强行写入。写完之后必须 `crontab -l` 复核一遍——屏幕上的和落盘的常常有出入，这是所有「我以为我加了任务」事故的第一现场。

### 14.3.2 crontab -r：一键清空，没有后悔药

> **警告**：`crontab -r` **删除整个 crontab 文件，不弹确认、不进回收站**。误敲的经典现场：本来想 `-l` 看列表，手指一滑打成 `-r`，多年积累的几十条定时任务连同注释一秒清零，且 cron 不保留任何历史版本。安全替代做法：一、只想删某几条任务，用 `crontab -e` 进去手动注释或删除对应行，绝不用 `-r`；二、定期 `crontab -l > /data/home/backup/crontab-$(date +%F).txt` 留档（备份习惯详见第 16 章）；三、已经把 crontab 当「基础设施代码」管理的团队，直接在版本库里维护、用 `crontab file` 从文件装载，改任务走代码评审，和改生产代码一个待遇；四、真误删了也别慌，先停手别再写新任务，看看有没有最近一次 `-l` 导出的留档或监控系统里的任务记录可回溯。

<br>

## 14.4 系统级 cron：/etc/crontab 与 /etc/cron.d/

用户级 crontab（`crontab -e` 维护的）一人一份，而系统级任务有另外两处合法住所，**格式是六字段——多出一列「以哪个用户身份运行」**：

```bash
cat /etc/crontab
```

```text
SHELL=/bin/sh
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# m h dom mon dow user  command
17 *    * * *   root    cd / && run-parts --report /etc/cron.hourly
25 6    * * *   root    test -x /usr/sbin/anacron || run-parts --report /etc/cron.daily
47 6    * * 7   root    test -x /usr/sbin/anacron || run-parts --report /etc/cron.weekly
52 6    1 * *   root    test -x /usr/sbin/anacron || run-parts --report /etc/cron.monthly
```

逐列念第一行 `17 * * * * root cd / && run-parts --report /etc/cron.hourly`：每小时的第 17 分钟，以 root 执行 `run-parts`，把 `/etc/cron.hourly` 目录里的可执行脚本挨个跑一遍。

- **文件头两行是变量**：`SHELL=` 指定这些任务用什么 shell；`PATH=` 显式给了完整路径——**这就是 14.6 第一个坑的官方答案：cron 不继承你登录环境里的 PATH，官方文件都老老实实自己写全**。你自己写 crontab 时也可以照抄这两行。
- **run-parts 机制**：`/etc/cron.hourly`、`.daily`、`.weekly`、`.monthly` 是四个「脚本投放目录」，往里丢一个可执行脚本，到点自动跑，不需要写任何时间表达式。logrotate 的每日执行、anacron 的每日巡检都走这里。
- `anacron` 那一串是另一套补跑机制：机器不是一直开机的（笔记本、偶尔 suspend 的云电脑），cron 到点没开机就永远错过了；anacron 记录「上次运行时间」，下次开机把错过的 daily/weekly/monthly 补上。你的电脑大概率装了（`command -v anacron`），一行 `test -x` 就是「有 anacron 就不必每日补刀」。

`/etc/cron.d/` 是给软件包用的「投放口」：装个备份代理、监控 agent，它把 `包名` 文件往这一丢，里面照样是六字段格式。自己往这写文件时注意两条纪律：**文件名不能带小数点**（`my.job` 会被 run-parts 风格的工具忽略）、文件权限 644、属主 root。它比 `/etc/crontab` 干净在哪：一文件一任务，卸载软件时删文件就行，不会和你的手写任务缠在一起。

<br>

## 14.5 @快捷写法：六个特殊关键字

五段式记不住没关系，cron 内置了六个 `@` 开头的快捷方式：

| 写法 | 等价五段式 | 含义 |
| --- | --- | --- |
| `@reboot` | 无等价 | 每次开机（cron 启动时）执行一次 |
| `@hourly` | `0 * * * *` | 每小时整点 |
| `@daily` | `0 0 * * *` | 每天 00:00（等价于 `@midnight`） |
| `@weekly` | `0 0 * * 0` | 每周日 00:00 |
| `@monthly` | `0 0 1 * *` | 每月 1 号 00:00 |
| `@yearly` | `0 0 1 1 *` | 每年 1 月 1 日 00:00 |

用法就是拿它替代五段：`@daily /data/home/scripts/backup-home.sh`。

`@reboot` 值得多说一句：它跑在 cron 守护进程启动的时刻，**适合「开机一次性」的准备任务**（挂载一个目录、清理上次崩溃的残留、给监控注册本机）。但它替代不了 systemd 服务——它不管依赖顺序（网络没好它就跑）、不管进程守护（跑完就完）、系统里也没有它的运行状态可查。要「开机常驻」就用第 13 章的 `systemctl enable`，`@reboot` 只是「开机清爽地跑一次」。

<br>

## 14.6 cron 的三个坑：PATH、% 与没人收的邮件

### 14.6.1 坑一：PATH 极简

你在交互 shell 里跑得好好的命令，进了 crontab 就 `command not found`。原因：cron 启动的 shell 是个「干净环境」——不读你的 `.bashrc`、不继承你终端里的 PATH，用户 crontab 的默认 PATH 往往只有 `/usr/bin:/bin` 一类。你自己装的 `python3` 在 `/usr/local/bin`？自写脚本在 `/data/home/scripts`？统统不在默认搜索范围。

**三种解法，按推荐顺序**：

1. crontab 顶部显式声明，照抄系统文件的写法：`PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin`。
2. 脚本里所有命令用绝对路径（`/usr/bin/curl`、`/bin/date`）。怎么查绝对路径？第 14 章学过的 `command -v curl`。
3. 脚本开头自检依赖：`command -v tar >/dev/null || { echo "FATAL: 找不到 tar"; exit 1; }`——第 14 章 13.10 的 `check_tools` 函数就是干这个的。

### 14.6.2 坑二：% 必须转义

crontab 里命令是「一行文本」，而 **% 是 crontab 的保留字符**——它把「命令行」和「送进命令的标准输入」截断。也就是说，命令里裸写的 `%` 会被 cron 吃掉、理解成别的意思。想传字面量百分号，写 `\%`：

```bash
# 想每小时给日志文件追加一行 "磁盘 90% 正常" —— 错误写法
0 * * * * echo "磁盘 90% 正常" >> /data/home/disk.log

# 正确：% 转义
0 * * * * echo "磁盘 90\% 正常" >> /data/home/disk.log
```

最常踩的场景是 `date` 的格式串：`date '+%F'` 里的 `%F` 会让 cron 把命令从 `+` 后面截断，剩下的 `+%F` 变成「标准输入」喂给 `echo`，行为完全错乱。防御写法两条：crontab 里所有 `date` 的格式串写 `\%F`；或者更省心——**把复杂逻辑挪进脚本文件，crontab 里只留一行「执行某脚本」**，%、引号、管道全在脚本里解决，第 14 章的备份脚本就是这么设计的。

### 14.6.3 坑三：输出进了一个不存在的邮箱

cron 的默认行为：任务有任何标准输出/标准错误，就尝试**发一封邮件**给任务主人。问题是这台机器没有邮件服务（MTA，Mail Transfer Agent）——本地没有、也不会发出去。于是：要么输出丢失（你看不到），要么卡在本地邮件队列里越攒越多，你以为任务「跑得很安静」，其实它的哭喊都进了黑洞。

**正解：每个 cron 任务都把输出重定向到文件**，这是铁律：

```bash
*/5 * * * * /data/home/scripts/check.sh >> /data/home/logs/check.log 2>&1
```

`>>` 追加写（别用 `>`，会覆盖昨天的记录）、`2>&1` 把错误也接进同一个文件——第 7 章《文本处理：grep / sed / awk》的重定向一节，在 cron 场合价值翻倍。想「只在出错时留证据」还有个进阶写法：`/path/script.sh > /dev/null 2>>/data/home/logs/err.log`（正常输出丢弃，错误单独留档）。顺带一句：真要在服务器上收 cron 邮件，得装 MTA（postfix 一类）并做发件域配置，那是中级课题，本书不展开——**学习阶段，重定向到文件就是完整答案**。

> **SRE 贴士（生产 vs 实验室）**：实验室里 cron 任务失败的代价是你的一次困惑；生产上三条坑各有各的死法：PATH 坑让备份脚本静默不干活，一个月后磁盘故障才发现「上次成功备份是两个月前」；% 坑让日志里出现 `+` 和半行乱码；邮件坑最阴——监控系统本身没配好输出，故障时想查 cron 干了什么，什么都没有。生产上的对应纪律：任务的每一行输出都必须有去处（重定向到受管日志文件，再由 logrotate 接管，见 14.11）；PATH 显式声明；任务写进版本库并走评审。另外别忘了第 1 章的事实：这台云电脑空闲 30 分钟自动暂停。如实说明 suspend 期间的定时任务行为——suspend 时文件、内存、进程整体冻结保留，机器不运行，cron 也就不会执行；恢复之后，cron 按「恢复时刻的当前时间」重新排班，**冻结期间错过的时间点不补跑**（这是 cron 的通用行为）。唯一有补跑希望的是 anacron 接管的 daily/weekly/monthly（14.4 讲过），但它靠开机触发，suspend/resume 不重启就不触发，别当万能补丁。至于「恢复后 cron 守护进程是否原样活着、有没有延迟」这类细节依平台实现而异，别信网上的二手结论——练习 1 就是你的观察实验：挂上每分钟任务，suspend 十分钟再 resume，看日志断在哪、从几点接着记，用自己的眼睛下结论。生产上同理：凡是「错过要补、依赖要守」的任务，答案在 14.7 的 timer 加 `Persistent=true`。

<br>

## 14.7 systemd timer：另一种「到点执行」

cron 之外，systemd 还提供官方定时器：**timer unit 到点触发一个 service unit**。它是第 13 章 unit 体系的自然延伸，先看一个最小实例，再和 cron 对比。

创建服务文件（timer 触发它）：

```bash
vim /etc/systemd/system/hello-tick.service
```

```text
[Unit]
Description=每分钟记一次时间的演示任务

[Service]
Type=oneshot
ExecStart=/bin/sh -c '/usr/bin/date "+%F %T" >> /data/home/logs/tick.log'
```

（为什么套一层 `/bin/sh -c`？systemd 的 `ExecStart` 不经过 shell 解释，`>>` 这种重定向语法只有 shell 认得——凡是带引号、管道、重定向的命令都这么包一层，这正是第 13 章「ExecStart 要绝对路径、复杂命令套 sh -c」的复现。另外记得先 `mkdir -p /data/home/logs`，目录不在，脚本写得再对也写不进文件。）

再创建配套的 timer 文件（**文件名必须 .timer，且引用的服务名必须真实存在**）：

```bash
vim /etc/systemd/system/hello-tick.timer
```

```text
[Unit]
Description=每分钟触发一次 hello-tick.service

[Timer]
OnCalendar=*:0/1
AccuracySec=1s

[Install]
WantedBy=timers.target
```

```bash
systemctl daemon-reload                 # 改了 unit 文件必须重载
systemctl enable --now hello-tick.timer # 启动并设置为开机启用
systemctl list-timers --all             # 看系统里所有计时器和下次触发时间
```

```text
NEXT                        LEFT     UNIT              ACTIVATES
Fri 2026-11-13 09:50:00 CST 12s      hello-tick.timer  hello-tick.service
```

（还有一类基于「开机后过了多久」的写法：`OnBootSec=5min`、`OnUnitActiveSec=1h`，前者开机 5 分钟后触发，后者「上次触发后每小时再来一次」；再加 `Persistent=true`，错过的触发在下次开机补办——这正是 cron 没有而 systemd timer 有的能力。）`OnCalendar` 的语法速查：

| OnCalendar | 含义 |
| --- | --- |
| `*-*-* 02:00:00` | 每天 02:00 |
| `Mon *-*-* 09:00:00` | 每周一 09:00 |
| `*:0/5` | 每 5 分钟（小时的 0、5、10…分） |
| `*-*-* *:00:00` | 每小时整点 |
| `Sat,Sun 12:00:00` | 周六和周日中午 |

timer 的两个独门优势：**触发时间可以被 `systemctl list-timers` 统一审视**（不用翻 N 个 crontab）；**错过会补**（`Persistent=true`，机器 suspend 过、宕机过的场景是刚需）。那 cron 什么时候还值得用？**要交给「没有 systemd 的环境」用、或维护既有的大量 cron 老任务、以及「系统级脚本投放目录」场景（cron.daily）**——大多数 Debian 桌面/服务器老脚本仍在 cron 手里。务实的分工建议：

| 维度 | cron | systemd timer |
| --- | --- | --- |
| 触发时间直观性 | 五段式紧凑，熟练工写得飞快 | OnCalendar 啰嗦但可读性好 |
| 环境控制 | PATH 极简，三个坑要挨个防 | 可用 `Environment=` 精确指定 |
| 日志 | 自己重定向到文件 | 直接进 journal，`journalctl -u` 可查 |
| 错过补跑 | 基本没有（anacron 补 daily） | `Persistent=true` 原生支持 |
| 依赖管理 | 无 | 可服务间 `After=` 依赖 |
| 关机期间任务 | 不补 | 可补 |
| 生态现状 | 所有发行版都有，通用 | systemd 发行版原生 |

结论一句话：**新写的部署级定时任务优先 timer（补跑、依赖、日志三样都是刚需）；需要兼容非 systemd 环境、或管理别人留下的 cron 生态时，老实把 cron 的三个坑防好**。本章练习会让你两种都写一遍，自己感受。

<br>

## 14.8 日志体系：从 syslog 到 journald

### 14.8.1 一段简史与「实话实说」

Linux 世界的日志经历过一次换代。早年所有进程都往一个叫 **syslog（系统日志）** 的通道写：应用程序调用 `syslog()` 函数（或写 `/dev/log` 这个 Unix 套接字），由一个常驻的 syslog daemon（Debian 上是 **rsyslog**）收下，按「谁写的、多严重」分流写进 `/var/log/` 下的各个文件。这套机制活了三十年，文件+文本的形式是无数运维吃饭的本事。

systemd 崛起之后，换代发生了：**systemd-journald（日志守护）成了新的日志收集者**，日志进「日志（journal）」——一个二进制的、带索引、带元数据（哪个 unit、哪个 PID、什么优先级）的存储。**Debian 13 的实话是：默认安装只带 journald，不再默认安装 rsyslog**。这意味着：

```bash
ls /var/log/
```

```text
apt  dmesg  dpkg.log  fsck  journal  private  wtmp
```

（你机器上可能略有差别，以实际为准。）看清现实：**`/var/log/syslog`、`/var/log/auth.log` 这些传统文件默认不存在**——它们只有装了 rsyslog 之后才会出现。想看认证日志怎么办？不装 rsyslog 也看得到，用第 13 章预告的姿势：`journalctl -u ssh`（ssh 服务的认证记录同样在 journal 里）。如果团队有「盯着 /var/log/auth.log 干活」的老习惯，`apt install rsyslog` 就回到传统世界，毫不冲突——两套收集者可以并存，journald 收 journal、rsyslog 继续写它的文件（还多一条「把 journal 内容也转发给 rsyslog」的管道）。

### 14.8.2 facility 与 level：日志的两个维度

syslog 世界给每条消息打两个标签，理解这两个概念，`logger`、`journalctl -p`、防火墙和 fail2ban 的配置全都能读懂：

**facility（类别）——谁写的**：

| facility | 来源 |
| --- | --- |
| `auth` / `authpriv` | 认证、授权（登录、sudo） |
| `kern` | 内核 |
| `user` | 默认类别，普通程序 |
| `daemon` | 守护进程 |
| `mail` / `cron` / `ftp` / `lpr` | 邮件 / 定时任务 / 文件传输 / 打印 |
| `local0`~`local7` | 留给系统管理员自定义的八个「自留地」 |

**level（严重级别）——多严重**，从重到轻：`emerg`（系统要完蛋）、`alert`、`crit`、`err`（错误）、`warning`（警告）、`notice`、`info`（信息）、`debug`（调试）。

写日志、筛日志都用这两个坐标：`logger -p cron.warning "任务超时"` 的意思就是「以 cron 类别、warning 级别写一条」。

### 14.8.3 /var/log 三剑客：不走 syslog 的那几位

虽然 syslog 文件默认缺席，`/var/log` 里这几位是包管理工具和内核自己写的、不依赖 syslog，一直在你盘上：

```bash
tail -5 /var/log/dpkg.log          # dpkg 的流水账：每个包的安装/升级/删除
ls /var/log/apt/                   # APT 的历史与终端输出
tail -3 /var/log/apt/history.log
```

```text
2026-11-13 09:12:04 install curl:amd64 <none> 8.x.y
2026-11-13 09:12:04 status half-installed curl:amd64 8.x.y
2026-11-13 09:12:04 configure curl:amd64 8.x.y <none>
```

```text
Start-Date: 2026-11-13  09:11:58
Commandline: apt install curl
Install: curl:amd64 (8.x.y), libcurl4t64:amd64 (…)
```

三剑客的排障价值：`dpkg.log` 逐包记录装了什么、什么时候、哪个版本——「这台机器上的 nginx 谁装的、哪天的」用它回答；`apt/history.log` 记录每条 apt 命令和结果——「上礼拜谁升级了系统、升了什么包」用它回答；`/var/log/fsck`、`dmesg` 老两位留给启动和内核事件（内核消息现在也能用 `journalctl -k` 看，见下一节）。**「/var/log 下什么都没有」的排查惯性要改掉了：Debian 13 的第一现场是 journal，文件在其次**——这是新一代运维和十年前教材的最大差别。

<br>

## 14.9 journalctl 全家桶：二进制日志的正确打开方式

journald 把日志收进 `/var/log/journal`（持久化存储，机器重启还在；拿不到持久盘时退回 `/run/log/journal`，重启即失）。读它用 `journalctl`，参数逐个过：

| 参数 | 作用 | 使用场景 |
| --- | --- | --- |
| 无参数 | 从最早一条开始翻（进 less，`q` 退出，`/` 搜索，`G` 跳末尾） | 通览一段历史 |
| `-e` | 直接跳到**末尾**最新处 | 「现在最新出了什么事」 |
| `-f` | **跟随**（follow）实时滚动，Ctrl-C 退出 | 看着日志复现故障，和 `tail -f` 一个手感 |
| `-u 服务名` | 只看某个 unit 的日志 | 排障主力：`-u nginx` |
| `-b` | 本次开机以来 | 重启过就要先 `-b` 再谈别的 |
| `-b -1` | **上一次**开机的日志 | 崩溃重启后查「上一次死前」 |
| `-k` | 只看内核消息（等价于 dmesg 的 journal 版） | 硬件、驱动、OOM |
| `--since` / `--until` | 时间窗，`--since "2026-11-13 09:00"` 也认 `-1h`、`today` | 「故障发生在 10 点前后」 |
| `-p 级别` | 只看某级别及以上，`-p err` 只留错误 | `-p err` 看错误、`-p warning` 看警告 |
| `-n 50` | 只看最近 N 条 | 不想翻页 |
| `--disk-usage` | journal 占用了多少磁盘 | 磁盘满了先查它 |
| `--vacuum-time=7d` | 清理 7 天前的归档日志 | 见下方警告 |
| `--no-pager` | 不进翻页器直接打印 | 管道给别的命令 |

挑几个组合出真实姿势：

```bash
journalctl -e                      # 最新日志，末尾
journalctl -u nginx -f             # 盯着 nginx 的日志实时看
journalctl -b -1 -p err            # 上次开机以来的所有错误
journalctl --since "2026-11-13 09:00" --until "2026-11-13 09:30" -u cron
journalctl -k | grep -i "out of memory"     # 内核日志里翻 OOM（grep 详见第 7 章）
journalctl -u ssh --no-pager | tail -20    # ssh 认证最近 20 条
```

```text
-- Journal begins at Thu 2026-11-13 08:47:12 CST, ends at Fri 2026-11-13 09:51:33 CST. --
Nov 13 09:51:12 book systemd[1]: Started Session 3 of user root.
Nov 13 09:51:33 book sshd[812]: Accepted publickey for root from 10.0.0.5 port 55412 ssh2
```

时间戳精确到秒（要亚秒精度用 `-o short-precise`）、每行带主机名、进程名和 PID（`sshd[812]` 就是），这就是二进制日志比纯文本多出来的元数据。`-b -1` 的诚实提醒：**如果这台机器从开机到现在还没重启过（或 suspend 恢复不算重启），上一次启动的日志可能不存在或为空**，`-b -1` 报 `Journal prefix not found` 不是命令写错了，是真的没有。

> **警告**：`journalctl --vacuum-time=7d`（以及 `--vacuum-size=500M`）清理的是**归档的旧日志文件**，它不碰「当前活跃」的日志文件——所以清理后 `--disk-usage` 常常降得没有预期多，这不是命令没用，是设计如此。更要注意的是：vacuum 是**不可逆删除**，故障复盘没做完就 vacuum，等于亲手销毁证据。安全替代做法：一、先 `journalctl --disk-usage` 量一下实际占用再决定动不动手（默认配置下 journal 上限一般是所在文件系统的 10%，以你量到的数为准）；二、要「当场腾空间」但复盘还没做完，先用 `journalctl -u nginx --no-pager > /data/home/nginx.log` 把要留的证据导出到持久盘，再 vacuum；三、根治靠改 `/etc/systemd/journald.conf`（如 `SystemMaxUse=500M`）并 `systemctl restart systemd-journald`，改配置比事后 vacuum 体面——后者只是删证据，前者从源头控总量。

<br>

## 14.10 logger：给自己的脚本接上日志通道

脚本和系统工具一样有权写日志，用 `logger`（syslog 家族的命令行客户端，journald 环境里直接进 journal）：

```bash
logger "backup-home.sh 手动跑一次"
logger -t backup -p user.notice "备份完成，大小 3.4M"   # -t 打标签，-p 指定类别.级别
logger -t backup -p user.err "备份失败：磁盘只读"
journalctl -t backup -n 3 --no-pager                    # -t 按标签反查
```

```text
Nov 13 09:52:41 book backup[1903]: 备份完成，大小 3.4M
Nov 13 09:52:44 book backup[1903]: 备份失败：磁盘只读
```

三个参数值回票价：`-t 标签` 相当于给消息盖章（反查时 `journalctl -t 备份` 一把捞出）；`-p 类别.级别` 决定它落在 14.8.2 坐标系哪里；`-i` 连 PID 一起记（`[1903]` 就是）。**和第 14 章「脚本自己 `echo` 到日志文件」并不冲突**，分工是：脚本的业务流水（备份了哪些文件、多大）写自己的日志文件（`tee -a`，13.13 的做法）；**状态和故障事件（开始、失败、完成）同时 logger 一条进 journal**，这样 `journalctl` 一个入口就能看到全系统的健康事件，不必记住每个脚本的日志路径。生产脚本两边都写，就是这个原因。

> **SRE 贴士**：判断「一个任务昨晚有没有正常跑」，别急着翻脚本自己的日志文件——先 `journalctl -t backup --since yesterday` 一眼看状态事件（成功/失败的开场和收场），再决定要不要钻进流水明细。另一个少有人用的好习惯：把 `local0`~`local7` 这八个自定义 facility 分门别类（比如 `local1` 给备份、`local2` 给监控采集），`logger -p local1.info` 一打，`journalctl SYSLOG_FACILITY=1` 就能按类回捞——多脚本系统里，这比每个脚本各起一个日志文件清爽得多。

<br>

## 14.11 logrotate：日志的「保质期管理」

日志不轮转，磁盘必满——第 11 章「磁盘满排查」的头号嫌疑人就是没人管的日志文件。**logrotate 是 Debian 上的日志管家**，每天被 `/etc/cron.daily` 叫醒一次，按配置把日志「切一刀、压起来、留几份、扔旧的」。

```bash
dpkg -l logrotate >/dev/null 2>&1 || echo "没装的话: apt install logrotate"
cat /etc/logrotate.conf | grep -v '^#' | grep -v '^$'
cat /etc/logrotate.d/nginx 2>/dev/null || echo "nginx 还没装，先看通用配置"
```

```text
weekly
rotate 4
create
compress
include /etc/logrotate.d
```

配置结构一句话：**「对哪些日志文件，按什么节奏，施加哪些动作」**。上面的全局配置：每周轮转、保留 4 份、轮转后新建空原文件、压缩归档、并 include `/etc/logrotate.d/` 下所有细分配置。逐行认一遍常用指令（全表在 `man 8 logrotate`）：

| 指令 | 含义 | 备注 |
| --- | --- | --- |
| `daily` / `weekly` / `monthly` | 轮转节奏 | 按文件名看不到日期，就是这回事 |
| `rotate N` | 保留 N 份旧日志 | 之外的在下次轮转时删除 |
| `compress` | 旧日志用 gzip 压缩 | 小文件 10 倍节省 |
| `delaycompress` | **最近一份先不压** | 进程可能还握着它，压了会丢 |
| `missingok` | 日志不存在不报错 | 软件卸了、任务还没首跑 |
| `notifempty` | 空了就不轮转 | 避免一堆 `access.log.1.gz` 空壳 |
| `create mode owner group` | 切完按指定权限属主新建空文件 | 不改属主，进程写新文件会失败 |
| `copytruncate` | **先复制内容再清空原文件** | 见下方警告 |
| `su user group` | 切换身份后操作 | 多用户目录树必需 |
| `dateext` | 旧文件名带日期 | `access.log-20261113.gz` 比 `.1.gz` 直观 |

给 nginx 写一个标准案例（日志文件写在哪、多频繁，一切按第 17 章《综合实战：从零部署并加固一个 Web 服务》的目录来）：

```bash
vim /etc/logrotate.d/nginx
```

```text
/data/home/logs/nginx/*.log {
    daily
    rotate 14
    compress
    delaycompress
    missingok
    notifempty
    create 0644 www-data adm
    sharedscripts
    postrotate
        [ -f /run/nginx.pid ] && kill -USR1 $(cat /run/nginx.pid)
    endscript
}
```

逐行说：`daily` 每天切；`rotate 14` 留 14 天；`compress + delaycompress` 压缩但**昨天切下来的那份先不压**（nginx 还要往里写最后一小会儿）；`notifempty` 空了不切，免得攒一堆空壳；`create 0644 www-data adm` 切完按原权限新建空 `access.log`，不给 nginx「写不进新文件」的机会；`sharedscripts + postrotate` 整组切完后发 `USR1` 信号让 nginx 重开日志文件（优雅重载的信号机制见第 9 章《进程管理与系统监控》）。

> **警告**：`copytruncate`（先 `cp` 再 `> file` 清空）是「不通知进程就能改文件」的通用解，但有写丢窗口：复制完成到清空之间进程新写的内容，会被清空动作抹掉——高写入量的日志几分钟就能攒出可观的空洞。安全替代做法：一、**优先用进程自己的信号机制**（nginx 的 USR1、多数服务的 `reopen` 或 `SIGHUP`），配合 `create` 模式，零丢失；二、只有进程**完全没有**重开文件的机制时（比如某个只会追加写、不会响应信号的遗留脚本），才上 `copytruncate`，并且接受这个窗口存在；三、别把它当万金油，日志量大的服务用了 copytruncate，故障复盘时会看到「时间线断片」，那正是丢数据的刀口。

调试 logrotate 有个「只演习不动手」的模式：

```bash
logrotate -d /etc/logrotate.conf        # -d = debug：完整走一遍逻辑，但不真正切
logrotate -f /etc/logrotate.d/nginx     # -f = force：不等排期，立刻强制切一次
man 8 logrotate                         # 完整手册在手
```

（`-d` 会打印它「将要做什么」的一大段，验证配置正确性的标准姿势。）两个补充事实：`/var/lib/logrotate/status`（state 文件）记录「每个日志文件上次轮转是哪一天」，logrotate 靠它决定今天该不该切——**误删它，所有日志会被当成「从没切过」，下次全部重切**；其二，Debian 上的每日触发不是 cron 五段式而是 `/etc/cron.daily/logrotate` 脚本（第 14.4 讲过的投放目录），所以「我的日志怎么每天半夜被切」——去那看。

<br>

## 14.12 实战排障案例：网站挂了，从 curl 到 journalctl

现在把本章所有工具串成一次完整的故障定位。背景：第 17 章《综合实战：从零部署并加固一个 Web 服务》会正式部署的官网，提前从第 10 章先装个 nginx 做案例主角。某天早上 9:47，收到「网站打不开」的告警。**排障纪律：全程只读诊断优先、一次只动一个变量、每步记录发现**，时间线如下：

| 时间 | 动作 | 命令 | 发现 |
| --- | --- | --- | --- |
| 09:47 | 收到告警，先验证现象：本地访问 health 接口 | `curl -sS -m 5 http://127.0.0.1/health.json; echo "退出码 $?"` | `curl: (7) Failed to connect`，退出码 7——连都连不上，不是内容问题 |
| 09:48 | 本地都不通 → 看端口有没有在监听 | `ss -tulnp | grep 80` | 无输出——**nginx 根本没在 80 上听** |
| 09:49 | 服务状态：是故意停的还是崩了 | `systemctl status nginx` | `Active: failed (Result: …)` 且 `Main PID` 已消失，`Process: 1147 ExecStart=… (code=exited, status=1…)`——**崩了，不是没启动** |
| 09:50 | 查死亡原因，锁定故障时间窗 | `journalctl -u nginx --since "2026-11-13 09:40" --until "09:47" --no-pager` | 09:44 有一条 `nginx: [emerg] open() "/etc/nginx/sites-enabled/book.conf" failed (2: No such file or directory)`，随后主进程退出——**配置文件没了** |
| 09:51 | 根因假设：配置被误删（比对 git/备份留档） | `ls /etc/nginx/sites-enabled/` + 翻 `/data/home/backup` 的上次留档 | 符号链接不在了；恢复留档里的配置 |
| 09:55 | 修复并验证 | `systemctl start nginx && curl -sS http://127.0.0.1/health.json` | `{"status":"ok"}`——本地恢复 |
| 09:56 | 确认监听与对外可访问 | `ss -tulnp | grep 80` + 本机 `curl` HTTPS 链接（正式对外形态见第 17 章《综合实战：从零部署并加固一个 Web 服务》） | 80 在听，外部通 |
| 09:58 | 复盘留档：把证据归档 | `journalctl -u nginx --since today --no-pager > /data/home/logs/incident-20261113.log` | 复盘写清楚时间线：谁动了配置、为什么挂了 4 分钟、如何防止 |

复盘三个要点，比修好它更重要：**发现阶段只读**（curl/ss/status/journal 全是读操作，看不懂不动手，避免小故障被修成大故障）；**时间窗意识**（`--since/--until` 一上，日志从几千行缩到十条，定位到哪个 unit 也是一刀）；**复盘制度化**（故障归档 + 一条变更纪律：配置进 git、下线前先备份）。这套时间线就是第 17 章「故障排查表」的原型。

> **SRE 贴士（生产 vs 实验室）**：实验室里「恢复留档」可以是从 `/data/home/backup` 拷回文件，五分钟搞定；生产上同样一套流程要升级：变更用「金丝雀发布、逐步放量」、配置变更走评审进 git、监控告警在故障发生 30 秒内就到你手机（而不是早上 9:47）。但唯一不变的是顺序——**curl → ss → systemctl status → journalctl --since**，这套「从现象到根因」的下钻路径，在一台机器和生产成百上千台集群上完全同构。卷 5 的监控体系会把这套人工流程变成自动化的告警聚合，现在练熟的是底层手感。

<br>

## 14.13 集中日志展望：卷 5 见

本章走到的是「单机日志」的尽头：一台机器上 journal + `/var/log` + logrotate 自给自足。生产的真相是：**你管理的不是一台机器**——故障发生时，问题往往横跨 web 层、应用层、数据库层三台以上机器的日志，靠 SSH 逐台 `tail` 是石器时代。

**集中式日志（centralized logging）** 是答案：每台机器上的 agent 把日志实时送到中央存储（Elasticsearch/Loki/ClickHouse 一类），一个查询界面搜全集群，按时间线还原跨服务故障。第 17 章的 health check、fail2ban、nginx 都会留下日志痕迹，卷 5《可观测性》会把它们汇成查询、仪表盘与告警——本章你只需要知道：**journalctl 练熟的「按 unit、按时间窗、按级别」三件套，正是集中日志平台的查询语言**；`logger` 打点、logrotate 管本地的习惯，就是将来 agent 转发的地基。别急，先把一台机器看明白。

<br>

## 本章小结

这一章你让系统「到点自己干活」，并学会了「事后查案」。两条主线：

```bash
# 定时任务
crontab -e -l -r        # 用户级；-r 是核武器，删单行走 -e
*/5 * * * * cmd         # 五段式：分 时 日 月 周（周 0/7 都是周日）；* */n a-b a,b
@daily /path/script.sh  # 六个 @ 快捷；@reboot 只跑一次，别当服务用
PATH=... 开头顶格声明    # 坑一：环境极简，命令要么绝对路径要么 command -v 自检
命令里的 % 一律写 \%     # 坑二：% 是 crontab 保留字符
>> /data/home/logs/x.log 2>&1   # 坑三：没 MTA，输出必须重定向到文件
systemctl enable --now x.timer   # timer 路线：OnCalendar + Persistent=true 补跑

# 日志
journalctl -e -f -u nginx -b -1 -k --since "..." -p err   # journal 全家桶
logger -t tag -p user.notice "..."   # 脚本主动写 journal
/var/log/dpkg.log /var/log/apt/history.log   # 不走 syslog 的三剑客
logrotate -d /etc/logrotate.conf    # 只演习不动手；status 在 /var/lib/logrotate/status
```

排障章法也固定下来了：**curl（验证现象）→ ss（端口在不在）→ systemctl status（进程死没死、怎么死的）→ journalctl --since（死亡时间线）→ 修复 → 验证 → 归档复盘**。下一章第 16 章《防火墙与数据备份》，给系统拉上「谁能进」的闸门，并把第 14 章的备份脚本升级成有恢复演练的完整备份体系。

## 动手练习

1. 查看并整理你的 cron：`crontab -l` 看现状（空的正常），然后用 `crontab -e` 加一条**每 5 分钟**向 `/data/home/logs/cron-test.log` 追加一行「时间戳 + 磁盘使用率」的任务（提示：`date` 直接放脚本里，避开 `%` 坑），等 10 分钟后 `cat` 日志确认跑了起来。练完用 `crontab -e` 注释掉它。
2. 照 14.6.1 写一个「防翻车」版 cron 任务：顶部声明 `PATH=`，命令用绝对路径执行第 14 章的 `backup-home.sh`，输出重定向到 `/data/home/logs/backup.log`，`2>&1` 不许省；跑一次验证日志里有「开始/完成/退出码」三行。然后故意把脚本里的 `tar` 改成 `tarr`，观察下一次运行日志里记录了什么错误——这就是 14.6.1 说的「必须留输出」的价值。
3. 对照 14.7 写一个自己的 systemd timer：`tick.service` + `tick.timer`，每分钟用 `date` 往 `/data/home/logs/tick.log` 追加一行；`systemctl enable --now`，`systemctl list-timers` 看到自己的 timer，`journalctl -u tick.service` 能看到执行记录。和练习 1 的 cron 任务对比一次，写下你更愿意在生产上选哪个、为什么。
4. 日志三剑客与 journal：`tail /var/log/dpkg.log` 找出你最近装的一个包；`journalctl -u ssh --since today` 翻今天的 ssh 记录；`journalctl -b -p err` 看本次启动以来的错误。再用 `logger -t drill -p user.warning "排障演练 2026-11-13"` 打一条自己的日志，然后 `journalctl -t drill` 把它捞出来。
5. logrotate 实验：给 `/data/home/logs/nginx/access.log`（自己 `mkdir -p` + 灌几行假数据造出来）写一个 `/etc/logrotate.d/drill`，要求 daily、rotate 3、compress、missingok、notifempty；先 `logrotate -d` 演习，再 `logrotate -f /etc/logrotate.d/drill` 强制切一次，`ls -lh` 看切割与压缩结果；跑第二次强制切，确认旧文件从 `.1.gz` 顺延为 `.2.gz`、`/var/lib/logrotate/status` 里出现了轮转日期记录。
6. 综合排障演练：模仿 14.12 做一次完整定位——装好 nginx 后正常访问一次 `curl http://127.0.0.1/`，然后 `systemctl stop nginx` 制造故障，按「curl → ss → systemctl status → journalctl --since」四步把现象和证据记进 `/data/home/logs/drill-report.md`，最后 `systemctl start nginx` 恢复并验证。复盘时回答：每一步命令各自的「判据」是什么。

> **参考答案**：
> 1. crontab 行参考 `*/5 * * * * /data/home/scripts/cron-test.sh`，脚本里 `echo "$(date '+%F %T') disk=$(df -h /data | awk 'NR==2 {print $5}')" >> /data/home/logs/cron-test.log`；等两轮确认两条以上记录；注释后 cron 仍会保留该文件，只是不再执行。
> 2. 三行（开始/完成/退出码）来自 13.13 的 `log()` + `trap cleanup EXIT`；改成 `tarr` 后，`set -e` + `command not found` 会记下失败和退出码 127——若不做重定向，这些证据会进黑洞（14.6.3）。
> 3. timer 文件照 14.7；`systemctl list-timers` 里 NEXT 列刷新正常、`journalctl -u tick.service` 有执行记录即为成功。选择上生产优先 timer 的理由：日志直接进 journal、有 `Persistent=true` 补跑、时间可统一审视；cron 胜在通用和五段式紧凑。
> 4. `tail /var/log/dpkg.log` 按时间戳找最近一条 install/configure；`journalctl -u ssh --since today` 能看到 `Accepted publickey` 一类记录（云电脑上 ssh 走平台通道，看不到记录属正常，如实记录即可）；`logger` 一条后 `journalctl -t drill -n 3` 应能按标签捞出。
> 5. 配置里只有 `compress` 没有 `delaycompress`，所以第一次 `-f` 后直接出现 `access.log.1.gz`；第二次后旧文件顺延成 `access.log.2.gz`、新切下的变成 `.1.gz`；`rotate 3` 意味着 `.4.gz` 出现时 `.1.gz` 会被删。drill 配置没写 `create`，所以切割后原文件不重建——这正是生产配置要写 `create` 的原因（14.11 案例就有）。`-d` 输出里应有 `considering log ... log needs rotating`；status 文件 `/var/lib/logrotate/status` 里出现该日志文件的轮转日期。
> 6. 四步判据：curl 退出码 7（连不上）→ `ss` 无 80 监听 → `systemctl status` 显示 `inactive (dead)`（这是 stop，不是 failed，注意区别）→ `journalctl --since` 有 `Stopped nginx` 记录；start 后 `curl http://127.0.0.1/` 返回欢迎页即闭环。
