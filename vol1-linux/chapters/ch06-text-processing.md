# 第 6 章 文本处理：grep / sed / awk

> 上一章我们守好了家门：谁、能对哪些文件、做什么。可开机之后，一个 SRE 真正耗时的地方不一定在"权限"，而在"读字"——凌晨三点告警来了，你要从几万行日志里捞出那几行关键信息；要给机房所有机器统一改一个配置项；要在容器编排的 YAML 里批量关闭某个开关。这些活的共同点是：对象全是纯文本。这一章把 Linux 最锋利的三把刀交给你：`grep` 负责找，`sed` 负责改，`awk` 负责算。

> **本章目标**
> - 能独立写出「筛选 → 切列 → 排序去重 → 统计」四级管道，从一份访问日志里取出 Top 10、状态码分布等结论。
> - 能读懂并编写常见的 BRE / ERE 正则（正则表达式，Regular Expression），说清 `grep` 与 `grep -E` 的差别与各自适用场合。
> - 能用 `sed` 完成行寻址、替换、增删改，并安全地使用 `-i`（原地修改）与其备份后缀；能用 `awk` 对字段化文本做计数与求和。

---

## 6.1 为什么文本处理是 SRE 的命门

先看一个典型的工作日切片。

早上九点，业务方说「昨天下午有个接口偶发超时」。你手上有什么？有 Nginx 访问日志、容器标准输出、K8s 的事件（Event）记录、Pod 里吐出来的 JSON。全是文本。

中午，安全同学说「全集群排查一下有没有把 `privileged: true` 打开的工作负载」。你要在几十份 YAML 里扫一个字符串。文本。

傍晚，你要交日报：昨天有多少 5xx、流量前十里来自哪些 IP。你要对数万行日志做分组计数。还是文本。

Linux 设计哲学里有一条：**一切皆文件，而文件大多是文本**。所以它给了你一套极其克制的工具：每个工具只做一件事（找、切、排、算），然后靠管道（Pipe）把它们串起来。你不需要学一个庞大的"日志分析平台"，几十个字符的命令就能得到答案。反过来，如果你只会用鼠标点图形界面，上面三件事一件都做不快。

还有个工程现实：服务器上很多东西压根没有图形界面。容器镜像为了瘦身，常常连编辑器都不带（详见第 7 章 vim）。SRE 的第一生产力就是命令行，第二生产力是把命令拼成管道。

> **SRE 贴士**：真正值钱的不是记住某条命令，而是把「问题」翻译成「管道」的能力。训练方法：每次排障完后，把当时敲的那串命令连同输入示例一起存进自己的命令笔记（第 0 章说的笔记），攒够两百条，你就有了自己的排障手册。公司里 Senior 和 Junior 的差距，很大一部分体现在这段翻译速度上。

---

## 6.2 重定向与管道回顾

第 3 章我们跑过第一条命令，也见过 `>`。这里系统讲一遍，因为后面每一步都要用。

每个进程默认打开三个文件描述符（File Descriptor）：

| 编号 | 名字 | 默认去向 |
| --- | --- | --- |
| 0 | 标准输入（stdin） | 键盘 |
| 1 | 标准输出（stdout） | 终端屏幕 |
| 2 | 标准错误（stderr） | 终端屏幕 |

重定向（Redirection）就是改这些去向：

```bash
ls -l /etc > out.txt        # 标准输出写入 out.txt（覆盖）
ls -l /etc >> out.txt       # 追加到末尾，不覆盖
ls -l /no/such/dir 2> err.txt   # 只把错误信息写进 err.txt
ls -l /etc > all.txt 2>&1       # 标准输出与标准错误都进同一个文件
ls -l /etc &> all.txt           # 上一行的等价简写（bash 支持）
ls -l /etc > /dev/null 2>&1     # 什么都不要，只想看退出码
cat < /etc/hostname             # 从文件读标准输入，等价于 cat /etc/hostname
```

注意顺序：`2>&1` 必须放在 `>` 之后。写成 `2>&1 > all.txt` 的意思是"先把错误指向当前屏幕，再把标准输出改到文件"，错误就漏到屏幕上了——这是新手最常踩的坑。

管道 `|` 则是把前一个命令的标准输出，直接当成后一个命令的标准输入：

```bash
cat /etc/passwd | wc -l
```

一条管道的精华在于：每一步只处理一种关注点，最后自然得到结论。本章 6.13 会集中练这个。

> **警告**：`>` 会**先清空目标文件**再写入。写错路径的后果是数据直接消失且不可恢复（没有回收站）。两条铁律：一，输出重定向到重要文件前，先 `ls -l` 确认目标；二，日志类、要反复写的场合一律用 `>>`。在生产上执行任何带 `>` 的清理脚本前，先在测试路径上跑一遍。

> **生产 vs 实验室**：实验室里你可以随时 `rm -rf` 重来（云电脑本身就是这个设计）。生产上 `>` 覆盖的是业务日志、是 /etc 下的配置、是别人正在读的文件。十年前的老运维会告诉你：改配置前 `cp nginx.conf nginx.conf.$(date +%F)` 这一秒钟，救过很多人的周末。

---

## 6.3 cat / tac / nl：看清文件全貌

`cat`（concatenate，拼接）的本意是把多个文件拼起来输出，最常用却是"看整个文件"。

```bash
cat /etc/hostname          # 看一个文件
cat a.txt b.txt > all.txt  # 拼接两个文件
cat -n /etc/hostname       # 带行号输出，注意是 stdout 层面加号，不改文件
cat -A /etc/hostname       # 显示不可见字符：行尾是 $，制表符是 ^I
cat -s file                # 压缩连续空行为一个空行
```

`cat -A` 是排查"脚本跑不起来"的神器：`^M$` 说明文件是 Windows 的 CRLF 换行，`^I` 说明有 Tab——这两种都会让脚本和 YAML 解析器跟你急。

`nl` 是专职加行号的工具，比 `cat -n` 更讲究：

```bash
nl -ba /etc/hostname       # 给"所有"行编号，空行也编
nl -bt /etc/hostname       # 只给非空行编号
```

`-ba`（body all）与 `-bt`（body non-empty）这两个参数值得记牢：`grep -n` 只给匹配行编号，而 `nl -ba` 给你全篇行号——报错信息说"第 37 行有问题"时，你需要的正是全篇行号。

`tac` 是 `cat` 倒过来拼，从最后一行往前输出：

```bash
tac /var/log/dpkg.log | head -5
```

日志是"越新越重要"，`tac` 配 `head` 就是"最新 N 条"的经典姿势。不过日志文件更大的时候，我们更常用 `tail`。

---

## 6.4 head / tail：一头一尾与实时跟踪

```bash
head /etc/passwd            # 默认前 10 行
head -n 5 /etc/passwd       # 前 5 行（-n 5 等价写法 head -5 也支持）
head -c 200 access.log      # 前 200 字节（Byte），适合看二进制或一行超长的日志
tail -n 20 access.log       # 末尾 20 行
tail -n +10 access.log      # 从第 10 行开始到末尾（+ 号是"起点的意思）
tail -f access.log          # 跟踪（follow）：文件有新内容就持续打印
tail -F access.log          # 更强的跟踪：文件被轮转（logrotate）后能自动跟上新文件
```

`-f` 与 `-F` 的区别值得单独说：日志轮转（详见第 14 章 logrotate）会在凌晨改名 `access.log` 为 `access.log.1` 再建新文件，普通 `-f` 跟的是旧的文件句柄，从此再也不动；`-F` 会重新打开新文件。排障过夜时必须用 `-F`，或者干脆两个都够——直接 `tail -F`，没有副作用。

```text
root@book:~# tail -n 3 /data/home/logs/access.log
192.168.31.23 - - [13/Nov/2026:10:02:12 +0800] "GET /docs/ HTTP/1.1" 200 20480 "https://book.example.com/docs" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 Chrome/142.0 Safari/537.36"
192.168.31.21 - - [13/Nov/2026:10:02:20 +0800] "GET /api/list HTTP/1.1" 500 1326 "https://book.example.com/" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 Chrome/142.0 Safari/537.36"
198.51.100.9 - - [13/Nov/2026:10:02:25 +0800] "GET /phpmyadmin/ HTTP/1.1" 404 209 "-" "Go-http-client/1.1"
```

**`tail -f` 与 `instabox exec` 的矛盾怎么破**

`instabox exec book -- tail -f /data/home/logs/access.log` 是个陷阱：exec 单条命令上限 10 分钟，而且**输出跑完才返回**。`tail -f` 永远不"跑完"，于是你会盯着黑屏等 10 分钟，最后等到退出码 `124`（超时），什么也没看着。

正确姿势是"把流落到磁盘，再离线看"：

```bash
# 第一次：起了个后台任务，输出落到文件，exec 立刻返回
instabox exec book -- sh -c 'nohup /data/home/scripts/longjob.sh > /data/home/logs/longjob.log 2>&1 & echo started'
# 之后：随时看增量，exec 都是秒回
instabox exec book -- tail -n 50 /data/home/logs/longjob.log
```

`nohup` 让任务忽略挂断信号，`&` 丢到后台，`>` 把输出接到文件——这三件套我们会在第 8 章《进程管理与系统监控》正式讲。Windows  readers 还要注意：终端里回车有时不产生换行，`tail -f` 打印到一半看起来像卡住，其实只是刷新时机问题，Ctrl-C 退出即可。

> **SRE 贴士**：云电脑空闲 30 分钟自动暂停，所以后台任务日志一定要写到 `/data/home`（持久盘）。写在 `/tmp` 或系统盘，机器一暂停 resumed 后你可能就找不到它了。写盘位置的选择，和命令本身一样重要。

---

## 6.5 sort 与 uniq：排序、去重与计数

`sort` 把行排序，`uniq` 处理**相邻**的重复行——所以 99% 的场景都是 `sort | uniq` 连用。

```bash
sort file                # 按字典序（实际上是按 locale 的语言排序规则）升序
sort -r file             # 逆序（reverse）
sort -n file             # 按数值（numeric）比较，而不是字符串比较
sort -k 2 file           # 以第 2 列为排序关键字
sort -k 3 -t: -n /etc/passwd   # 定界符为冒号，按第 3 列（UID）数值排序
sort -u file             # 排序并去重（unique），等价于 sort | uniq 的效果
sort -f file             # 忽略大小写（fold case）
```

为什么一定要 `-n`？因为字符串比较下 `"10" < "9"`（`1` 比 `9` 小）。凡是数字统计，不加 `-n` 全是错的结果。

```text
root@book:~# sort -t: -k3 -n /etc/passwd | head -6
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
```

`-t:` 指定定界符，`-k3` 取第 3 个字段（UID），`-n` 按数值排。这是看懂一台机器上"用户们"的最快方式。

`uniq` 的参数少得可怜但个个常用：

```bash
uniq file            # 去掉相邻重复行（只保留一个）
uniq -c file         # 计数（count）：每行前面打印出现次数
uniq -d file         # 只打印有重复的行
uniq -u file         # 只打印唯一（unique，无重复）的行
```

```text
root@book:~# awk '{print $9}' /data/home/logs/access.log | sort | uniq -c
      7 200
      4 404
      2 500
      1 302
      1 304
```

数一数：7+4+2+1+1=15，正好是我们样本日志的行数，说明一个都没漏。**做统计后顺手核对总数**，这是防止管道中途丢行的好习惯。

> **警告**：`uniq` 只看相邻行。没排序直接 `uniq -c`，得到的是"每段连续重复"的计数，看起来有数，其实是错的。看到 `uniq` 前面没有 `sort`，先提高警惕。

---

## 6.6 cut / paste / pr：按列切分与拼接

`cut` 是文本里的"取列器"，对 CSV、`/etc/passwd`、表格化输出极其顺手。

```bash
cut -d: -f1,3 /etc/passwd      # 以冒号为定界符，取第 1 和第 3 列
cut -d: -f3- /etc/passwd       # 取第 3 列到末尾（范围）
cut -d: -f1-3 --complement del ... # 反选（--complement 取"之外"的列）
cut -c1-8 file                 # 不加 -d 时按字符（character）位置截取
cut -b1-8 file                 # 按字节（byte）截取，处理二进制时小心用
```

`-f` 支持用逗号组合任意列，缺省定界符是 Tab。

```text
root@book:~# cut -d: -f1,3 /etc/passwd | head -6
root:0
daemon:1
bin:2
sys:3
sync:4
games:5
```

`paste` 把多个文件按行左右拼接（默认用 Tab 隔开，`-d` 可换成别的定界符），`pr` 则为打印做分页排版，加页码页眉。这两个日常用得少，知道存在即可：`paste -d, a.txt b.txt` 把两列合成 CSV，需要打印或生成纸质工单时 `pr -2 file`。

> **SRE 贴士**：`cut -d` 的坑在于它不懂引号。CSV 里有 `a,"b,c",d` 这种带引号的字段，`cut -d,` 会切错列。真要解析复杂 CSV 或 JSON，交给 Python（云电脑模板自带 Python + uv）写十几行脚本，比用 shell 硬拗更可靠。shell 的价值在"快"，不在"全能"。

---

## 6.7 wc 与 tr：计数与字符整形

`wc`（word count）统计行、词、字节：

```bash
wc /etc/passwd        # 一次输出 行数 词数 字节数 文件名
wc -l /etc/passwd     # 只要行数（lines）
wc -w /etc/passwd     # 只要词数（words）
wc -c /etc/passwd     # 只要字节数（bytes）
wc -m /etc/passwd     # 按字符数（characters），与 -c 的差别在中文等多字节字符
```

中文文件里 `wc -c` 与 `wc -m` 结果会不一样，因为 UTF-8 下一个汉字占 3 字节。这是入门时最爱搞混的一对。

```text
root@book:~# wc -l /data/home/logs/access.log
15 /data/home/logs/access.log
```

`tr`（translate）做字符级的替换、删除、压缩：

```bash
tr -d '\r' < win.txt > unix.txt      # 删除所有回车字符（Windows 文件转 Unix）
tr -s ' ' < messy.txt               # 压缩（squeeze）连续空格为一个
tr 'a-z' 'A-Z' < lower.txt          # 小写转大写
tr -dc '0-9\n' < mixed.txt          # 只保留数字和换行，其余全删
```

修 CRLF 是 `tr -d '\r'` 的成名技。根因是 Windows 用 `\r\n` 两个字符表示换行，Unix 只认 `\n`，多出来的 `\r` 会让 shell 脚本第一行的 shebang 变成 `#!/bin/bash\r`，解释器找不到 `bash\r` 这个命令，于是报一个看起来毫无关系的错：

```text
/bin/sh: 1: script.sh: not found
```

见到这种莫名报错，先 `cat -A script.sh | head -1`，看到 `^M$` 就实锤了。

---

## 6.8 正则表达式速成

前面所有命令都还是"找固定的词"。想找"所有 5 开头的状态码"，就得靠正则（正则表达式，Regular Expression）。

先把两类口音的差别钉死，这是最常见的困惑来源：

| 特性 | BRE（基本正则） | ERE（扩展正则） |
| --- | --- | --- |
| 谁在用 | `grep`、`sed` 默认 | `grep -E`、`sed -E`、`awk` |
| `+` `?` `|` `{}` `()` 的含义 | **普通字符**，要写成 `\+` `\?` `\|` `\{` `\}` `\(` `\)` 才是特殊含义 | **本身就是特殊含义**，直接写 |
| 反向 | `\+` 才表示"一次或多次" | `+` 就表示"一次或多次" |

一句话记忆：**BRE 用反斜杠"开光"，ERE 用反斜杠"去光"。** 所以 `grep -E '(a|b)+'` 能跑，`grep '(a|b)\+'` 才能跑。什么时候必须 `-E`？当你要用 `|` 表示"或者"、用 `+`/`?` 表示重复、用 `|` 组合多个分支、用 `{}` 指定次数，且不想写一串反斜杠时。图省事也可以全程 `grep -E`，本书推荐：**日常首选 ERE，只有要写进最小化环境的脚本（比如 busybox 环境）才用 BRE。**

正则基础零件：

```text
字符       含义
.          任意单个字符（不含换行）
[abc]      字符类：a、b、c 任意一个
[^abc]     否定字符类：不是 a、b、c
[a-z0-9]   范围：小写字母或数字
\d \w      只在部分工具里成立（Perl 风格），Linux 基础工具不要依赖
^          行首锚点
$          行尾锚点
*          前一个元素重复 0 次或多次
+          前一个元素重复 1 次或多次（ERE）
?          前一个元素重复 0 次或 1 次（ERE）
{n,m}      重复 n 到 m 次（ERE 直接写；BRE 写 \{n,m\}）
\|         或（BRE）；ERE 直接写 |
\( \)      分组与捕获（BRE）；ERE 直接写 ( )
\          转义，把特殊字符变普通，或把普通字符变特殊
```

注意"量词只作用于紧挨着它的那一个元素"：

```bash
grep -E 'ab+c'     # 匹配 abc、abbc、abbbc
grep -E 'a(bc)+'   # 匹配 abc、abcbc
grep -E '^[0-9]{3}$'  # 整行恰好是三位数字
```

一个高频陷阱：`.` 匹配任意字符，所以 `grep 'a.c'` 也能匹配到 `abc.txt` 里的 `a.c` 部分之外的字符串。想表示真正的点号，必须写 `\.`。

---

## 6.9 grep 全家桶

`grep`（global regular expression print）的职责：按模式筛行。

```bash
grep PATTERN FILE
grep -E PATTERN FILE         # 用扩展正则
grep -F PATTERN FILE         # 固定字符串（fixed），里面的 * . [ 都不特殊，适合搜 IP、路径
grep -v PATTERN FILE         # 反选（invert），输出"不匹配"的行
grep -i PATTERN FILE         # 忽略大小写
grep -n PATTERN FILE         # 带行号
grep -c PATTERN FILE         # 只打印匹配行数
grep -o PATTERN FILE         # 只打印匹配到的那段文本，不是整行
grep -r PATTERN DIR         # 递归（recursive）搜目录
grep -l PATTERN DIR         # 只列出包含匹配的文件名（files-with-matches）
grep -w PATTERN FILE        # 全词匹配（word），避免 cat 匹配到 catalog
grep --color=auto PATTERN   # 高亮匹配部分（Debian 已为 root 默认开启）
```

拿访问日志做例子（样本文件见 6.13，共 15 行）：

```text
root@book:~# grep -n ' 500 ' /data/home/logs/access.log
8:192.168.31.21 - - [13/Nov/2026:10:01:40 +0800] "GET /api/list HTTP/1.1" 500 1326 "https://book.example.com/" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 Chrome/142.0 Safari/537.36"
14:192.168.31.21 - - [13/Nov/2026:10:02:20 +0800] "GET /api/list HTTP/1.1" 500 1326 "https://book.example.com/" "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 Chrome/142.0 Safari/537.36"

root@book:~# grep -E '(500|502|503|504)' /data/home/logs/access.log | wc -l
2
root@book:~# grep -oE '[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+' /data/home/logs/access.log | sort -u | wc -l
7
```

第二条用 `-E` 和 `|` 一次圈出所有服务端错误码；第三条 `-o` 只留 IP 段，`sort -u` 数出有 7 个不同 IP。

**grep 的退出码，脚本里的关键**

`grep` 的返回值本身就是信息：

| 退出码 | 含义 |
| --- | --- |
| 0 | 至少匹配到一行 |
| 1 | 一行都没匹配（注意：这不是错误！） |
| 2 | 真出错了，比如文件不存在 |

于是"判断某配置是否存在"可以完全不看输出：

```bash
if grep -q '^PermitRootLogin' /etc/ssh/sshd_config; then
    echo "该项已配置"
else
    echo "未配置，需要加一行"
fi
```

`-q`（quiet）直接静默，只留退出码。`grep file 不存在的文件` 退出码是 2 且向 stderr 打 `No such file or directory`——所以写进脚本时，`grep` 没找到东西别急着报错，先看看退出码到底是 1 还是 2。配合 `set -euo pipefail` 的完整写法详见第 13 章《Shell 脚本入门》。

> **SRE 贴士**：`grep -v grep` 这个经典组合（`ps aux | grep nginx | grep -v grep`）是为了滤掉 grep 自己那一行。但现在很多环境的 `ps` 输出里 grep 行是 `grep --color=auto nginx`，用 `grep -v 'grep'` 依然有效；更稳妥的做法是用 `pgrep -a nginx` 或 `ps aux | grep '[n]ginx'`——后面这种用字符类躲开自身的技巧，值得抄进小本本。

---

## 6.10 sed：流编辑器

`sed`（stream editor）一行行读入、按指令处理、一行行送出，默认不改原文件。

```bash
sed [选项] '指令' 文件
sed -n '1,10p' file        # -n 静默模式；p 打印；只输出第 1~10 行
sed 's/old/new/' file      # 每行第一个 old 替换为 new
sed 's/old/new/g' file     # g = global，一行内全部替换
sed 's#a/b#c#d#' file      # s 命令的定界符可以用 #，路径里斜杠多时救命
sed -i 's/old/new/g' file  # 原地修改（in-place）
sed -i.bak 's/old/new/g' file  # 原地修改并留扩展名为 bak 的备份
```

**行地址**决定指令作用在哪几行：

```bash
sed -n '5p' file            # 第 5 行
sed -n '1,10p' file         # 第 1~10 行
sed -n '10,$p' file         # 第 10 行到末行
sed -n '/^root/p' /etc/passwd    # 地址也可以是正则：匹配行才处理
sed -n '1,10{/nologin/p}' file   # 双重限定：1~10 行里含 nologin 的（花括号块写法进阶）
```

**替换与标志**：

```bash
sed 's/nologin/DISABLED/' file         # 每行替换第一处
sed 's/nologin/DISABLED/g' file        # 每行替换所有处
sed 's/nologin/DISABLED/2' file        # 每行第 2 处
sed 's/nologin/DISABLED/gp' file       # 替换后还把这行打出来（配合 -n 用）
```

**增删改**：

```bash
sed '2d' file          # 删除（delete）第 2 行
sed '/^#/d' file       # 删除所有注释行——清洗配置文件的经典一招
sed '2a 新行内容' file  # 在第 2 行之后追加（append）
sed '2i 新行内容' file  # 在第 2 行之前插入（insert）
sed '2c 新行内容' file  # 把第 2 行整行改成新内容（change）
```

一次真实操作示范：

```text
root@book:~# sed -n '1,5p' /etc/passwd
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync

root@book:~# sed 's#/usr/sbin/nologin#DISABLED#' /etc/passwd | sed -n '2,4p'
daemon:x:1:1:daemon:/usr/sbin:/DISABLED
bin:x:2:2:bin:/bin:/DISABLED
sys:x:3:3:sys:/dev:/DISABLED
```

第二条命令用 `#` 作定界符，因为替换内容里有 `/`。如果用 `/` 就得写成 `s/\/usr\/sbin\/nologin/DISABLED/`，一堆反斜杠，谁看谁头大。**规则：当 old 或 new 里出现 `/`，立刻换成 `#`（或其他不在内容里的符号）。**

**分组捕获**用 `\1`、`\2` 引用括号里的内容，做"调换两列"这种事很方便：

```bash
sed -E 's#^([^:]+):x:([0-9]+):.*#用户 \1 的 UID 是 \2#' /etc/passwd
```

输出形如 `用户 root 的 UID 是 0`。

> **警告**：`sed -i` 直接改写原文件，写错模式就是永久损失，而且它**不问你一声**。三条安全纪律：一，改前 `cp file file.bak`，或者直接用 `sed -i.bak` 让 sed 自己留备份；二，**先去掉 `-i` 跑一遍看输出**，确认无误再补 `-i`（这个习惯能挡掉九成事故）；三，别对 `/etc` 下的系统文件裸奔 `sed -i`，第 7 章我们会用 vim 带备份地改系统配置。

> **生产 vs 实验室**：实验室里 `sed -i` 改崩了，重装就是。生产上你面对的是几十台机器、几百个配置，而且没有"重装"这个选项。正确姿势是：先在一台灰度机上验证，把命令连同预期输出写进变更单，再用配置管理工具批量下发，每次保留 diff 与回滚脚本。"能一键改一万台"和"敢按这个按钮"是两门功课。

---

## 6.11 awk：字段化处理利器

`sed` 按行处理，`awk` 按**字段**处理，而且自带编程能力（变量、条件、循环、函数）。

核心概念只有几个：

- **记录（Record）**：默认一行就是一条记录。
- **字段（Field）**：行内按空白切成 `$1 $2 $3 ...`，`$0` 是整行，`$NF` 是最后一个字段（NF = 字段数）。
- **NR**：已读入的记录数（行号）。**FNR**：当前文件内的行号（多文件时有用）。
- **FS**：输入字段分隔符，默认是"连续空白"。**OFS**：输出字段分隔符，默认空格。
- **BEGIN / END**：读入前 / 读入后各执行一次的块，适合放初始化和汇总。

```bash
awk '{print $1}' access.log                    # 每行打印第 1 字段
awk -F: '{print $1, $3}' /etc/passwd           # -F 指定分隔符；$3 是 UID
awk '$9 == 500 {print $7}' access.log          # 带条件：只处理状态码 500 的行
awk '{n++} END {print n}' access.log           # 统计行数（同 wc -l）
awk 'BEGIN {print "报表开始"} {sum+=$10} END {printf "总流量 %d 字节\n", sum}' access.log
```

```text
root@book:~# awk -F: '{printf "%-10s %s\n", $1, $7}' /etc/passwd | head -5
root       /bin/bash
daemon     /usr/sbin/nologin
bin        /usr/sbin/nologin
sys        /usr/sbin/nologin
sync       /bin/sync

root@book:~# awk '$9 == 500 {print $7}' /data/home/logs/access.log
/api/list
/api/list

root@book:~# awk '{sum+=$10} END {printf "total=%d avg=%.2f KiB\n", sum, sum/NR/1024}' /data/home/logs/access.log
total=135025 avg=8.79 KiB
```

逐段解释最后一条：`{sum+=$10}` 对每行把第 10 字段（响应字节数）累加进变量 `sum`；`END {printf ...}` 在全处理完之后执行一次，`printf` 的格式串和 C 语言一样，`%d` 打整数、`%.2f` 保留两位小数、`\n` 换行。注意 awk 变量不用声明、不用 `$` 取用，初值 0——这些"隐式约定"是 awk 简洁的来源，也是它容易写错的原因。

**实战：统计访问日志里 IP 出现频次 Top 10**

这是运维面试与实战都高频的一道题。我们把 6.5 到 6.9 的零件拼起来：

```bash
awk '{print $1}' /data/home/logs/access.log | sort | uniq -c | sort -rn | head -10
```

逐段拆解：

| 片段 | 职责 |
| --- | --- |
| `awk '{print $1}'` | 只留第 1 字段，也就是客户端 IP，其余全丢掉 |
| `sort` | 把相同 IP 排到一起——`uniq` 的前置条件 |
| `uniq -c` | 每个 IP 输出一行「次数 + IP」 |
| `sort -rn` | 按次数数值逆序排，频次高的在前 |
| `head -10` | 只取前 10 名 |

```text
root@book:~# awk '{print $1}' /data/home/logs/access.log | sort | uniq -c | sort -rn | head -10
      4 192.168.31.20
      3 192.168.31.21
      2 10.0.0.7
      2 198.51.100.9
      2 203.0.113.44
      1 192.168.31.22
      1 192.168.31.23
```

管道的妙处在这：五段各自都能单独测试。怀疑 `awk` 取错列？单独跑第一段。怀疑 `sort -rn` 没生效？单独跑第三段的输出看一眼。**排错时把管道一节一节拆开跑**，比盯着整条命令猜要快得多。

> **警告**：awk 的重定向是"shell 先创建文件"语义。`awk '{print $1 > "/data/home/logs/out.txt"}' access.log` 会在命令开始前就把 `out.txt` 截断为 0 字节——如果 `out.txt` 恰好是你想读的原始日志，数据当场就没了。规则：awk / sed 的输出文件**绝不要**和输入文件同名；要覆盖写就用 `>` 由 shell 控制并三思，要追加才考虑 `>>`。

> **SRE 贴士**：`awk` 在性能敏感的场合很能打：`grep + sed + awk` 的组合通常比启一个 Python 解释器快几倍，而且不依赖运行环境。日志采样、容器启动探针、配置生成这类"小活儿"，shell 加 awk 往往是最合适的工具。等逻辑复杂到要维护（超过两重循环、要读多个文件互相关联），就该换 Python 了——工具选型的临界点，心里要有数。

---

## 6.12 diff -u 与 patch：精确描述"改了什么"

`diff` 逐行比较两个文件，`-u` 输出统一格式（unified format），带上下文，人和工具都好读：

```bash
diff file1 file2        # 普通格式
diff -u file1 file2     # 统一格式，最常用
diff -ru dir1 dir2      # 递归比较两个目录树
diff -q dir1 dir2       # 只报告"哪些文件不同"，不看内容
```

```text
root@book:~# diff -u a.conf b.conf
--- a.conf	2026-11-13 10:30:11.000000000 +0800
+++ b.conf	2026-11-13 10:31:02.000000000 +0800
@@ -1,4 +1,4 @@
 server {
-    listen 80;
+    listen 8080;
     server_name book.example.com;
 }
```

读法：`-` 开头的行是"旧文件有、新文件没有"，`+` 开头的行是"新的改法"，`@@ -1,4 +1,4 @@` 说明这个改动块在旧文件第 1 行起共 4 行、新文件第 1 行起共 4 行的范围内。

有了这份 diff，`patch` 就能把它套用到第三个文件上：

```bash
diff -u a.conf b.conf > listen8080.patch
patch a.conf < listen8080.patch     # 应用补丁
patch -R a.conf < listen8080.patch  # -R 反向应用，等于撤销
```

Git 的打补丁工作流（`git diff > x.patch` → `git apply` / `am`）、内核社区提 patch、以及把配置改动打包发给同事评审，底层都是这一套。**记住这个能力，它让你"用文本精确描述一次变更"**，这正是变更管理的雏形（完整实践详见第 16 章《综合实战：从零部署并加固一个 Web 服务》）。

---

## 6.13 管道组合实战

先造一份练习数据（约 15 行，够看出规律又不刷屏）：

```bash
mkdir -p /data/home/logs /data/home/demo
cat > /data/home/logs/access.log <<'EOF'
192.168.31.20 - - [13/Nov/2026:10:01:12 +0800] "GET /static/app.js HTTP/1.1" 200 51234 "https://book.example.com/" "Mozilla/5.0 (X11; Linux x86_64)"
10.0.0.7 - - [13/Nov/2026:10:01:13 +0800] "GET /api/health HTTP/1.1" 200 15 "-" "curl/8.13.0"
192.168.31.20 - - [13/Nov/2026:10:01:15 +0800] "POST /api/login HTTP/1.1" 302 0 "https://book.example.com/login" "Mozilla/5.0 (X11; Linux x86_64)"
203.0.113.44 - - [13/Nov/2026:10:01:18 +0800] "GET /admin HTTP/1.1" 404 209 "-" "python-requests/2.32"
192.168.31.21 - - [13/Nov/2026:10:01:22 +0800] "GET / HTTP/1.1" 200 8241 "https://www.google.com/" "Mozilla/5.0 (X11; Linux x86_64)"
192.168.31.20 - - [13/Nov/2026:10:01:30 +0800] "GET /static/app.js HTTP/1.1" 304 0 "https://book.example.com/" "Mozilla/5.0 (X11; Linux x86_64)"
198.51.100.9 - - [13/Nov/2026:10:01:33 +0800] "GET /.env HTTP/1.1" 404 209 "-" "Go-http-client/1.1"
192.168.31.21 - - [13/Nov/2026:10:01:40 +0800] "GET /api/list HTTP/1.1" 500 1326 "https://book.example.com/" "Mozilla/5.0 (X11; Linux x86_64)"
192.168.31.22 - - [13/Nov/2026:10:01:44 +0800] "GET /favicon.ico HTTP/1.1" 200 318 "https://book.example.com/" "Mozilla/5.0 (X11; Linux x86_64)"
192.168.31.20 - - [13/Nov/2026:10:02:01 +0800] "GET /static/app.js HTTP/1.1" 200 51234 "https://book.example.com/" "Mozilla/5.0 (X11; Linux x86_64)"
10.0.0.7 - - [13/Nov/2026:10:02:05 +0800] "GET /api/health HTTP/1.1" 200 15 "-" "curl/8.13.0"
203.0.113.44 - - [13/Nov/2026:10:02:10 +0800] "GET /wp-login.php HTTP/1.1" 404 209 "-" "python-requests/2.32"
192.168.31.23 - - [13/Nov/2026:10:02:12 +0800] "GET /docs/ HTTP/1.1" 200 20480 "https://book.example.com/docs" "Mozilla/5.0 (X11; Linux x86_64)"
192.168.31.21 - - [13/Nov/2026:10:02:20 +0800] "GET /api/list HTTP/3.1" 500 1326 "https://book.example.com/" "Mozilla/5.0 (X11; Linux x86_64)"
198.51.100.9 - - [13/Nov/2026:10:02:25 +0800] "GET /phpmyadmin/ HTTP/1.1" 404 209 "-" "Go-http-client/1.1"
EOF
```

这里用到了 here-document（`<<'EOF'`）：`cat` 从紧随的一串文本读入，直到单独一行的 `EOF` 为止；`'EOF'` 加引号表示内容照抄、不做变量展开。写多行配置的顺手办法，脚本里的完整用法详见第 13 章。

**实战一：状态码分布 + 是否有服务端错误**

```bash
awk '{print $9}' /data/home/logs/access.log | sort | uniq -c | sort -rn
```

```text
root@book:~# awk '{print $9}' /data/home/logs/access.log | sort | uniq -c | sort -rn
      7 200
      4 404
      2 500
      1 302
      1 304
```

7 个 200、4 个 404，还有 2 个 500——服务端确实有错，接下来就去查 500 打在了哪个接口（`awk '$9==500 {print $7}'`）。

**实战二：流量前十里，谁在勤快地扫你**

```bash
grep -oE '[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+' /data/home/logs/access.log | sort | uniq -c | sort -rn | head
```

```text
root@book:~# grep -oE '[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+' /data/home/logs/access.log | sort | uniq -c | sort -rn | head
      4 192.168.31.20
      3 192.168.31.21
      2 10.0.0.7
      2 198.51.100.9
      2 203.0.113.44
      1 192.168.31.22
      1 192.168.31.23
```

配合上一节"404 集中在 `/wp-login.php`、`/phpmyadmin/`、`/.env`"一起看，就能下结论：`203.0.113.44` 和 `198.51.100.9` 是扫描器，`192.168.31.20` 是正常用户。**先统计、再定性**，别凭一行日志下判断。

**实战三：一行命令找出最占空间的日志**

```bash
du -sh /var/log/* 2>/dev/null | sort -rh | head -5
```

```text
root@book:~# du -sh /var/log/* 2>/dev/null | sort -rh | head -5
4.0K	/var/log/alternatives.log
0	/var/log/btmp
...
```

`2>/dev/null` 用来丢掉 `du: cannot read directory ... Permission denied` 这类噪音（在第 5 章之后，你终于明白这些 Permission denied 是谁在拦你）。注意 `sort -rh` 的 `-h` 表示"按人类可读的单位比大小"（4.0K < 1.2M），这是 Debian 版 GNU coreutils 支持的参数。

磁盘空间紧张的排查路径：`df -h` 看哪个分区满了 → `du -sh /var/log/* | sort -rh` 看谁大 → 决定是清理、轮转还是搬去持久盘（详见第 10 章《磁盘与存储管理》）。

> **SRE 贴士**：上面三条命令有个共同骨架——**取样 → 选列 → 分组计数 → 排序取头**。把 `awk '{print $1}'` / `$9` / `cut -d: -f1` 换个字段，就能换成统计 UA 版本、统计 HTTP 方法、统计登录失败来源。骨架不变，只换零件，这是文本处理真正的复用点。

---

## 本章小结

这一章的主线是"用命令行处理文本"，可以分成四层来记：

| 层次 | 工具 | 一句话职责 |
| --- | --- | --- |
| 看 | `cat -A` / `nl -ba` / `head` / `tail -F` | 把内容看全、看头尾、看不可见字符 |
| 筛 | `grep`（含 `-E`/`-F`/`-v`/`-o`/`-r`） | 按正则或固定串留下想要的行 |
| 整 | `sort` / `uniq -c` / `cut` / `tr` / `wc` | 排序、去重计数、切列、字符整形 |
| 算 | `sed` / `awk` | 按行改写、按字段统计与编程 |

生产环境里最高频的三条，建议形成肌肉记忆：

```bash
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -10   # Top N 统计
grep -rn 'keyword' /etc/somedir                                      # 配置排查
sed -i.bak 's/old/new/g' file                                        # 带备份地批量改配置
```

正则的两套口音（BRE/ERE）、`grep` 的三个退出码、`sed` 的寻址与备份、`awk` 的 `$0/$1/$NF/NR/NF/FS/BEGIN/END`——这四组概念是本章考试重点，也是后面写脚本（第 13 章）、看日志（第 14 章）的地基。最后提醒一句：所有工具都遵守"只读优先"原则，**先看再改，改前备份**。

---

## 动手练习

以下题目都可以在 `book` 云电脑上完成，涉及的文件都在 `/data/home`，搞坏了删掉重来即可。

1. 用 `nl -ba` 给 `/etc/passwd` 全篇加行号，找出你自己的 `bookuser` 在第几行；再用 `grep -n` 做同样的事，比较两者输出差异并说明为什么。
2. 写一条管道：统计 `/data/home/logs/access.log` 中每个请求路径（第 7 字段）的访问次数，取前 5 名，并用 `uniq -c` 的输出反推总数是否等于 15。
3. 用 `grep -E` 从 `/data/home/logs/access.log` 里挑出所有被当作"扫描"的请求行（提示：UA 是 `python-requests` 或 `Go-http-client`），只输出 IP 和路径两个字段。
4. 用 `awk` 统计 `access.log` 里响应字节数（第 10 字段）的总和与平均值，并用 `wc -l` 核对行数；再写一条等价的 `wc -c` 思路，说明为什么两者数值不可能相等。
5. 造两个文件 `a.conf`、`b.conf`（内容差一行，如 `listen 80;` 改成 `listen 8080;`），用 `diff -u` 生成补丁，用 `patch` 应用到 `a.conf`，再用 `patch -R` 撤销，最后用 `diff a.conf b.conf` 确认状态。
6. 综合题：把上面第 2 题的 Top 5 结果重定向到 `/data/home/logs/top5.txt`（用 `>>` 追加一行时间戳说明），再用 `sed -i.bak` 把文件里的空格分隔改成 Tab，最后 `cat -A` 验证分隔符真的变了。

> **参考答案**：
>
> 1. `grep -n 'bookuser' /etc/passwd` 只给匹配行编号（如 `34:bookuser:...`）；`nl -ba` 给全文编号，所以你要在输出里找那一行，且空行也占号。两者编号对同一行是一致的。
> 2. `awk '{print $7}' /data/home/logs/access.log | sort | uniq -c | sort -rn | head -5`。得到的计数加起来应等于 15；不等就说明有行的字段结构不同（如某行没有引号包裹的请求），回头用 `sed -n 'Np'` 看那一行。
> 3. `grep -E 'python-requests|Go-http-client' access.log | awk '{print $1, $7}'`。
> 4. `awk '{s+=$10} END {printf "%d %.2f\n", s, s/NR}' access.log`。`wc -c access.log` 数的是整个文件的全部字节（含时间戳、引号、UA 文本），当然远大于纯响应体大小的合计，两者口径不同，没有可比性。
> 5. `diff -u a.conf b.conf > change.patch`，`patch a.conf < change.patch`（输出 `patching file a.conf`），`patch -R a.conf < change.patch`；撤销后 `diff a.conf b.conf` 应当重新显示差异。
> 6. 关键是两步：`(echo "# $(date) 的统计"; awk '{print $7}' access.log | sort | uniq -c | sort -rn | head -5) >> /data/home/logs/top5.txt`；然后 `sed -i.bak 's/[[:space:]]\+/\t/' /data/home/logs/top5.txt`（`[[:space:]]` 是 POSIX 字符类，`\+` 是 BRE 的"一次或多次"）；`cat -A` 看到 `^I` 即 Tab 生效，`.bak` 是原始备份。
