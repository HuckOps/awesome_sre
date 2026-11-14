---
title: "第 3 章 命令行入门与文件系统树"
---

# 第 3 章 命令行入门与文件系统树

> 前面两章我们搭好了 InstaBox 云电脑并登了进来，拿到了一个 `root@book:~#` 提示符。这一章把镜头对准「命令本身」：怎么跑、怎么查帮助、怎么在文件树里走，以及 Linux 把一切都当成文件的哲学。命令行是 SRE 的第一生产资料，把它练熟，后面每一章都事半功倍。

> **本章目标**
> - 能用命令格式、Tab 补全、history 高效操作 shell，并读懂 `man` 与 `--help`。
> - 能在 FHS 文件系统树里迅速定位常见目录，并用 `pwd/cd/ls` 完整导航。
> - 能用通配符、绝对/相对路径精准指定文件，并用 `whoami/id/hostname/uptime` 确认当前环境。

## 3.1 Shell 是什么：你和操作系统的对话窗口

**Shell（外壳）** 是 Linux 用户与内核之间的中间层。内核负责调度 CPU、内存、硬盘，但不直接跟人说话；Shell 负责读取你敲进去的每一行，解析成内核能懂的指令，再把结果打印到终端上。

Debian 13 默认的交互 Shell 是 **bash（Bourne Again Shell）**，但系统里还有另一个 Shell 叫 **dash**。你可以用下面这条命令看到区别：

```bash
ls -l /bin/sh
```

```text
lrwxrwxrwx 1 root root 4 Feb 14 09:12 /bin/sh -> dash
```

`/bin/sh` 是一个符号链接，指向 `dash`——这是 Debian 家族的 **usrmerge** 带来的现状：传统上 `/bin` 是独立目录，现在 `/bin`、`/sbin`、`/lib` 都分别符号链接到了 `/usr/bin`、`/usr/sbin`、`/usr/lib`。对你写交互命令没影响，但写脚本时，`#!/bin/sh` 和 `#!/bin/bash` 的行为会有细微差别（详见第 13 章《Shell 脚本入门》）。

## 3.2 命令的通用格式：`命令 [-选项] [参数]`

几乎所有 Linux 命令长这样：

```text
命令 [-选项] [参数]
```

用 `ls` 举例，先裸跑：

```bash
ls
```

```text
Desktop  Documents  Downloads  Music  Pictures  Public  Templates  Videos
```

再加选项 `-l`（long，长格式）和参数 `~`（家目录）：

```bash
ls -l ~
```

选项通常以短横线开头，多个选项可以合并写，比如 `ls -lah` 等价于 `ls -l -a -h`。选项前的短横线不可省略；参数则通常是文件名、目录名或其他对象。

## 3.3 Tab 补全与 history：少打字，少出错

在终端里敲命令时，按 **Tab** 键会自动补全命令名、文件名或目录名。如果有多个匹配，按两次 Tab 会列出所有候选。

实验：输入 `cd /et` 然后按两次 Tab，你会看到 `/etc` 被补全并列出相关项。

把常用命令敲腻了之后，查历史最快的办法是：

```bash
history
```

它会列出带编号的命令清单。再结合 **Ctrl-R**，可以在历史里反向搜索：按下 Ctrl-R，输入几个字符（比如 `ls`），shell 会自动填出最近一次匹配的命令，回车即可执行。

还有两个历史速记：
- `!!`：重复执行上一条命令（常用于“忘了加 sudo”的时候，敲 `sudo !!`）。
- `!n`：执行 history 里第 n 条命令。

## 3.4 必会快捷键：手别离开键盘

下面这些快捷键在 bash 里通用，建议肌肉记忆化：

| 按键 | 作用 |
| --- | --- |
| Ctrl-A | 光标跳到行首 |
| Ctrl-E | 光标跳到行尾 |
| Ctrl-U | 删除从光标到行首的所有字符 |
| Ctrl-W | 删除光标前一个单词 |
| Ctrl-K | 删除从光标到行尾的所有字符 |
| Ctrl-L | 清屏（等价于 `clear` 命令） |
| Ctrl-R | 在命令历史里反向搜索 |
| Ctrl-C | 中断当前正在运行的命令（发送 SIGINT） |
| Ctrl-D | 关闭当前终端 / 发送 EOF |
| Ctrl-Z | 把当前前台进程挂到后台（详见第 8 章《进程管理与系统监控》） |
| 上下箭头 | 翻阅历史命令 |
| Tab | 自动补全命令名、文件名、目录名 |

## 3.5 帮助三件套：不会用就别瞎猜

不会一个命令时，有三条路，按优先级排：

1. **`--help`**：大多数外部命令（如 `ls`、`cp`、`tar`）都支持，输出简洁的用法摘要。
2. **`help`**：针对 shell 内建命令（如 `cd`、`echo`、`export`），用 `help cd` 查看。
3. **`man`**：manual 手册，最权威、最详细。用 `man ls` 查看，里面的操作是：
   - `/关键词`：向下搜索关键词
   - `n`：跳到下一个匹配
   - `N`：跳到上一个匹配
   - `q`：退出 man 页

以 `ls` 为例：

```bash
ls --help | head -20
```

```text
Usage: ls [OPTION]... [FILE]...
List information about the FILEs (the current directory by default).
Sort entries alphabetically if none of -cftuvSUX nor --sort is specified.

Mandatory arguments to long options are mandatory for short options too.
  -a, --all                  do not ignore entries starting with .
  -A, --almost-all           do not list implied . and ..
  ...
```

再看 man 页的首屏（部分截取）：

```bash
man ls
```

里面会有 NAME / SYNOPSIS / DESCRIPTION / OPTIONS / EXAMPLES 等段落，每个选项都有英文详解。英文看不懂没关系，先抓住选项名和对应的 `-` 缩写，配合实验慢慢积累。

## 3.6 FHS 目录树：Linux 把什么都当成文件

Linux 遵循 **FHS（Filesystem Hierarchy Standard，文件系统层次标准）**，根目录 `/` 下面每个目录都有固定职责。在你的 InstaBox 云电脑上执行：

```bash
ls -F /
```

```text
bin/  boot/  dev/  etc/  home/  lib/  lib64/  media/  mnt/  opt/  proc/  root/  run/  sbin/  srv/  sys/  tmp/  usr/  var/
```

下面逐目录讲，配合 `ls -l` 的实际输出：

| 目录 | 职责 | 备注 |
| --- | --- | --- |
| `/` | 根目录，一切起点 | 所有文件系统最终挂载在此 |
| `/bin` | 必需的用户命令（`ls`、`cp`、`mv` 等） | Debian 13 里 `/bin -> /usr/bin`（符号链接） |
| `/boot` | 内核、引导加载程序文件 | 云电脑通常不直接动 |
| `/dev` | 设备文件（硬盘、终端、空设备 `/dev/null`） | “一切皆文件”的体现 |
| `/etc` | 系统配置文件（`/etc/passwd`、`/etc/fstab` 等） | 运维高频阅读区 |
| `/home` | 普通用户家目录的父目录 | 云电脑里普通用户家目录在 `/data/home` |
| `/lib` / `/lib64` | 共享库文件 | 通常也是符号链接到 `/usr/lib*` |
| `/mnt` / `/media` | 临时挂载点（U 盘、光盘、额外磁盘） | 云电脑较少用到 |
| `/opt` | 第三方可选软件安装目录 | 比如手动解压的 JDK、Go |
| `/proc` | 内核与进程的虚拟文件系统 | 不占真实磁盘，读它等于查内核状态 |
| `/root` | root 用户的家目录 | 你当前登录的身份，`~` 就是这里 |
| `/run` | 运行时数据（PID 文件、锁文件） | 重启后清空 |
| `/sbin` | 系统管理命令（`iptables`、`reboot` 等） | 通常也指向 `/usr/sbin` |
| `/srv` | 系统提供的服务数据（网站根目录等） | 第 16 章会用到 `/srv/www` |
| `/sys` | 内核参数与设备模型的虚拟文件系统 | 配合 `/proc` 看系统底层状态 |
| `/tmp` | 临时文件，所有用户可写 | 带 Sticky 位（详见第 5 章） |
| `/usr` | 用户程序、库、文档、源码的次级层级 | `/usr/bin` 存放绝大多数用户命令 |
| `/var` | 可变数据（日志 `/var/log`、缓存 `/var/cache`） | 磁盘满常见元凶 |

`/usr` 下面还有一层：`/usr/bin`（用户命令）、`/usr/sbin`（系统管理命令）、`/usr/lib`（库）、`/usr/share`（跨架构共享数据）、`/usr/local`（本地手动安装软件）。书里后面提到 `/usr/bin/xxx`，就是指这个层级。

你的云电脑有一个特殊目录 **`/data`**：它是 20 GiB 持久盘，家目录 `/data/home` 就在这个盘里。系统盘只有 16 GiB，**用户自己的练习文件务必放在 `/data/home`**，别往系统盘堆东西（详见第 10 章《磁盘与存储管理》）。

## 3.7 导航命令：pwd 与 cd

**`pwd`（Print Working Directory）** 打印当前所在的绝对路径：

```bash
pwd
```

```text
/root
```

**`cd`（Change Directory）** 切换目录，用法极其频繁：

```bash
cd /etc          # 切到绝对路径 /etc
cd ..            # 切到上级目录
cd               # 不带参数，切回家目录（root 就是 /root）
cd -             # 切回上一次所在的目录（“后退”键）
cd ~             # 切到当前用户家目录
cd ~bookuser     # 切到用户 bookuser 的家目录（需权限）
```

`.` 表示当前目录，`..` 表示父目录，`~` 表示家目录。这三个是路径里的“速记符号”。

## 3.8 ls：目录里到底有什么

`ls` 是你最常用的命令之一，不带参数只列文件名。下面逐个拆解常用选项，每个都配一次实际输出：

### 3.8.1 `-l`：长格式，看清每一列

```bash
ls -l /etc/hostname
```

```text
-rw-r--r-- 1 root root 11 Nov 13 09:11 /etc/hostname
```

每一列从左到右的含义：
1. **文件类型与权限**：`-rw-r--r--`，第一位 `-` 表示普通文件；后面三组 rwx 分别对应属主、属组、其他人（详见第 5 章）。
2. **硬链接数**：`1`，这个文件有几个硬链接指向它（详见第 4 章）。
3. **属主**：`root`。
4. **属组**：`root`。
5. **文件大小**：`11` 字节。
6. **修改时间**：`Nov 13 09:11`。
7. **文件名**：`/etc/hostname`。

### 3.8.2 `-a`：显示隐藏文件

Linux 里以点 `.` 开头的文件是隐藏文件，`ls` 默认不显示：

```bash
ls -a ~
```

```text
.  ..  .bashrc  .profile
```

`.` 是当前目录，`..` 是父目录，它们永远存在。

### 3.8.3 `-h`：人类可读的大小

配合 `-l`，把字节换算成 K/M/G：

```bash
ls -lh /var/log/syslog 2>/dev/null || ls -lh /var/log/auth.log
```

```text
-rw-r----- 1 root root 12K Nov 13 09:12 /var/log/auth.log
```

（实际文件名以你的机器为准，这里展示格式即可。）

### 3.8.4 `-R`：递归列出子目录

```bash
ls -R /etc/ssh | head -10
```

会把目录下所有层级的文件都列出来，配合 `head` 限制行数（`head` 的用法详见第 4 章）。

### 3.8.5 `-d`：只显示目录本身，不进去看

```bash
ls -ld /etc
```

```text
drwxr-xr-x 1 root root 4096 Nov 13 09:11 /etc
```

不加 `-d`，`ls /etc` 会列出 `/etc` 里面的文件；加了 `-d`，只显示 `/etc` 这个目录自身的属性。

### 3.8.6 `-t` / `-S`：按时间或大小排序

```bash
ls -lt /var/log        # 按修改时间，最新在前
ls -lSh /var/log       # 按大小，最大在前，人类可读
```

### 3.8.7 `-i`：显示 inode 编号

```bash
ls -li /etc/hostname /etc/hosts
```

```text
123456 -rw-r--r-- 1 root root 11 Nov 13 09:11 /etc/hostname
789012 -rw-r--r-- 1 root root 187 Nov 13 09:11 /etc/hosts
```

第一列数字就是 inode 编号，两个文件 inode 相同就表示它们是同一文件的硬链接（详见第 4 章）。

## 3.9 路径：绝对、相对与通配符

**绝对路径** 从根目录 `/` 开头，如 `/etc/hostname`，无论你在哪个目录都能用。**相对路径** 从当前目录算起，如 `../bookuser` 或 `./notes.txt`。

通配符在路径里能极大减少敲键盘：
- `*` 匹配任意字符串：`ls /etc/*.conf` 列出所有以 `.conf` 结尾的文件。
- `?` 匹配任意单个字符：`ls file?.txt` 匹配 `file1.txt`、`fileA.txt` 但不匹配 `file10.txt`。
- `[]` 匹配字符集：`ls [abc]*.txt` 匹配以 a、b 或 c 开头的 `.txt` 文件。

> **SRE 贴士**：通配符展开是 shell 本身干的，不是命令。`ls *.log` 执行前，shell 已经把 `*.log` 替换成当前目录所有 `.log` 文件的完整列表再传给 `ls`。如果目录里 `.log` 文件成千上万，可能会碰到“参数列表过长”的错误，这时候改用 `find`（详见第 4 章）。

## 3.10 日常查看命令：date / cal / whoami / id / hostname / uptime

这六个命令看似简单，是排障时第一排确认环境的“体温计”。

### 3.10.1 date：当前日期时间

```bash
date
```

```text
Sat Nov 14 09:12:34 CST 2026
```

排障时先看时间对不对，日志里的时间戳与本机一致才好对齐。

### 3.10.2 cal：日历

```bash
cal
```

```text
    November 2026
Su Mo Tu We Th Fr Sa
 1  2  3  4  5  6  7
 8  9 10 11 12 13 14
15 16 17 18 19 20 21
22 23 24 25 26 27 28
29 30
```

### 3.10.3 whoami / id：我是谁

```bash
whoami
```

```text
root
```

```bash
id
```

```text
uid=0(root) gid=0(root) groups=0(root)
```

`uid=0` 就是 root 身份。第 5 章会详细讲 `id` 的每一字段。

### 3.10.4 hostname：机器名

```bash
hostname
```

```text
book
```

更详细的 `hostnamectl` 详见第 11 章《网络配置与远程管理》。

### 3.10.5 uptime：系统已运行多久

```bash
uptime
```

```text
 09:12:34 up 1:23,  1 user,  load average: 0.00, 0.01, 0.05
```

最后三个数是 **load average（负载均值）**，分别代表最近 1 分钟、5 分钟、15 分钟的平均活跃进程数。数值远大于 CPU 核数（本机 4 核，详见第 1 章验货）时，系统就很忙了（详见第 8 章）。

## 3.11 本章小结

| 知识点 | 关键命令/概念 |
| --- | --- |
| Shell | bash（交互）、dash（/bin/sh），`#!/bin/sh` vs `#!/bin/bash` 的差异 |
| 命令格式 | `命令 [-选项] [参数]`，多选可合并 |
| 补全与历史 | Tab、双击 Tab、history、`!!`、`!n`、Ctrl-R |
| 快捷键 | Ctrl-A/E/U/W/K/L/R/C/D/Z |
| 帮助 | `--help`、`help`、`man`（`/ ? n N q`） |
| FHS 目录树 | `/`、`/bin`、`/etc`、`/home`、`/var`、`/proc`、`/sys`、`/usr`、`/data`（云电脑持久盘） |
| 导航 | `pwd`、`cd`（`..`、`.`、`~`、`-`） |
| ls | `-l -a -h -R -d -t -S -i` 逐参数含义 |
| 路径 | 绝对/相对，通配符 `* ? []` |
| 体温计命令 | `date`、`cal`、`whoami`、`id`、`hostname`、`uptime` |

> **生产 vs 实验室**：生产服务器上尽量少用 root 直接敲命令，优先用普通用户 + `sudo`（详见第 5 章）；在这台云电脑上你默认就是 root，练习方便，但养成“先看再敲”的习惯，哪天登上生产机才不会手滑。

## 动手练习

1. 在云电脑上执行 `ls -lah /`，列出根目录所有文件（含隐藏文件），并指出哪些是目录、哪些是符号链接。
2. 进入 `/etc`，用 `ls -l passwd` 看 `/etc/passwd` 的权限与大小，再用 `ls -ld /etc` 只看 `/etc` 目录本身。
3. 执行 `cd /var/log`，然后 `cd -` 回到原来的目录，说明 `cd -` 到底回到了哪里。
4. 用 Tab 补全输入 `ls /usr/bin/zip`（不需要真有 zip 命令，观察补全行为），再用 `history` 看刚才输入的命令是否被记录。
5. 执行 `uptime`，读出三个 load average 数值，结合第 1 章验货的 4 核 CPU，判断机器当前是否繁忙。

> **参考答案**：
> 1. 输出会以 `d` 开头的行表示目录（如 `bin/`、`etc/`），以 `l` 开头的表示符号链接（如 `bin -> usr/bin`），具体列表随机器而异，重点在认出 `d`、`-`、`l` 三种首字符。
> 2. `ls -l /etc/passwd` 输出类似 `-rw-r--r-- 1 root root 2xxx Nov ... /etc/passwd`；`ls -ld /etc` 输出类似 `drwxr-xr-x ... /etc`。
> 3. 执行 `cd /var/log` 后当前目录变为 `/var/log`；再执行 `cd -` 回到执行第一条 `cd` 之前的目录（通常是 `~`，即 `/root`），shell 会把切回后的目录打印出来。
> 4. 按 Tab 时 shell 会尝试补全 `/usr/bin/zip`；若存在则补全为完整路径，不存在则提示无匹配。`history` 的最后几行应包含你刚才输入的命令。
> 5. `uptime` 输出最后三个数分别是 1/5/15 分钟负载。机器有 4 个核，若三个数都远小于 4（如 0.xx），说明当前很空闲；若接近或超过 4，说明有进程在竞争 CPU。

下一章（第 4 章《文件与目录操作》）将教你怎么真正动手创建、复制、移动、删除文件，以及硬链接与软链接的区别。
