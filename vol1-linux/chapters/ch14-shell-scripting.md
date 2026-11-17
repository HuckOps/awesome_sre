---
title: "第 14 章 Shell 脚本入门"
---

# 第 14 章 Shell 脚本入门

> 前面十二章，你一直在做同一件事：把一条一条命令敲得又快又准。这一章换个身份——从「命令的使用者」变成「命令的编排者」。脚本（script）就是把一批命令按顺序写进一个文件，让机器自己照着跑。它带来三样东西：可重复（同样的活今天做和明年做，动作一模一样）、可复盘（出了问题能顺着脚本读出它做了什么）、可交接（同事读脚本就知道这套系统是怎么维护的）。不会写脚本的运维是「人肉定时器」，会写脚本的运维才拥有一台「会自动干活的系统」。

> **本章目标**
> - 能写出结构正确的 bash 脚本（shebang、`set -euo pipefail`、变量、函数），并说清 `./script.sh` 与 `bash script.sh` 的区别、单双引号的差异、`export` 对子进程的影响。
> - 能独立运用退出码、条件判断、循环和函数，把一个重复性运维动作（/data/home 备份）写成带日期戳、带自检、带正确退出码的完整脚本。
> - 能用 `bash -x`、`bash -n` 和自定义 `PS4` 调试脚本，并识别初学者最高频的十个错误中的至少五个。

<br />

## 13.1 为什么把重复劳动脚本化

先讲一个真实得有点伤人的场景。你要每天半夜备份一次 `/data/home`。手工做法是：登录、敲 `tar`、敲 `date` 拼文件名、等它跑完、确认、退出。三件事决定了手工一定出事：**人会忘**（假期、团建、通宵排障后的第二天早上）；**人会错**（某天手滑把目标目录写错，或者漏了 `--exclude` 把备份自己备份进备份里）；**人会变**（三个月后你换了个参数，上个月的备份和这个月的标准不一样，追溯时对不上）。

脚本化之后，这三件事一次性解决：动作固定（不会忘不会错）、标准固定（每次跑的是同一份代码）、历史固定（脚本文件在 git 里，谁改的、改成什么样，第 16 章《防火墙与数据备份》会讲的「变更记录」直接从版本库来）。

哪些活适合脚本化？判断标准很朴素：**同一个动作你要做第二遍的时候，就该考虑写进脚本**。SRE 日常里最常见的脚本化对象：备份与恢复演练、系统巡检（磁盘、内存、端口、证书有效期）、日志清理与轮转配合、发布前后的健康检查、故障时的「取证脚本」（一次性把故障现场的关键信息打包保存，这类脚本平时就要备好，出事时你手是抖的，别指望现敲）。

> **SRE 贴士（生产 vs 实验室）**：实验室里你写脚本是为了「省事」，生产上脚本是「命」。差别在三处：生产的脚本必须可审计（改脚本走代码评审，和改代码一个待遇）；必须幂等（同一个脚本跑两遍不会把事情搞坏，比如备份脚本要能安全重入）；必须考虑并发（两个实例同时跑同一个备份脚本，日期戳要精确到秒甚至加锁，第 9 章《进程管理与系统监控》讲进程时留的坑在这补上）。所以本章的目标不是「把脚本写出来」，而是「按能上生产的方式把脚本写出来」——先健壮、后功能，这个优先级和开发恰恰相反。

<br />

## 13.2 第一个脚本：解释器与运行方式

打开云电脑，我们按纪律把练习文件放 `/data/home`（持久盘，第 11 章说的分层）：

```bash
mkdir -p /data/home/scripts && cd /data/home/scripts
```

用第 8 章《vim 编辑器》的 vim，或者先用一个不依赖编辑器的方式创建第一个脚本：

```bash
cat > hello.sh <<'EOF'
#!/bin/bash
# hello.sh —— 第一个脚本：报告「我是谁、在在哪、现在几点」
echo "你好，$(whoami)！今天是 $(date '+%F %T')，主机名 $(hostname)"
EOF
```

逐行拆解：

- **第一行 `#!/bin/bash` 叫 shebang（解释器声明行）**。它不是注释，是给内核看的：当这个文件被「当作程序执行」时，内核读 shebang，启动它指定的解释器，把文件路径作为参数传进去。写成 `#!/bin/bash`，意思就是「用 bash 这个解释器来执行我」。
- **`#` 开头的注释**：shebang 之外的所有 `#` 行都是注释。写注释是脚本的基本功——三个星期后你还得读懂它。
- **`echo` 输出 + 命令替换 `$(命令)`**：双引号里的 `$(date ...)` 会先执行命令、再把输出嵌进字符串。这是第 13.3 节的重点，这里先眼熟。

### 13.2.1 #!/usr/bin/env bash 与 #!/bin/bash 之争

你还会经常看到另一种写法：`#!/usr/bin/env bash`。区别要说透：

| 写法 | 含义 | 适用场景 |
| --- | --- | --- |
| `#!/bin/bash` | 直接用绝对路径 `/bin` 下的 bash | Linux 服务器。Debian 经过 usrmerge（详见第 4 章《命令行入门与文件系统树》，`/bin` 是 `/usr/bin` 的符号链接），`/bin/bash` 一定存在 |
| `#!/usr/bin/env bash` | 用 `env` 命令去 PATH 环境变量里「找第一个 bash」再执行 | 脚本要在多种机器间移植（比如在 macOS 上用 Homebrew 装的 bash 优先于系统自带的旧版 bash） |

一句话结论：**Debian 云电脑上写 `#!/bin/bash` 就好，直接、明确、没有搜索开销**；`/usr/bin/env bash` 是「跨平台通用派」，你的脚本如果打算开源分享、或者团队里混着 macOS 开发机，用它更稳。cron 章节（第 15 章）还有一个细节和这个选择有关，到时候回看。

### 13.2.2 chmod +x、./script.sh 与 bash script.sh

现在运行它。第一次运行大概率吃闭门羹：

```bash
./hello.sh
```

```text
bash: ./hello.sh: Permission denied
```

原因：新建的文件默认权限是 644（`rw-r--r--`，第 6 章《用户、群组与权限》讲过），**没有执行权限（x 位）**，系统就不允许它「被当作程序运行」。补上：

```bash
chmod +x hello.sh
ls -l hello.sh
```

```text
-rwxr-xr-x 1 root root 137 11-13 09:35 hello.sh
```

```bash
./hello.sh
```

```text
你好，root！今天是 2026-11-13 09:35:12，主机名 book
```

**为什么前面必须写 `./`？** 因为 shell 只在 `PATH` 环境变量列出的目录里找「可执行文件」，当前目录通常不在其中（这是安全设计：防止你在一个目录里放一个叫 `ls` 的恶意程序冒充系统命令）。所以运行当前目录的脚本要显式写路径：`./hello.sh` 意思是「就在当前目录，别去 PATH 找」。

第二种运行方式是**把脚本文件当作参数传给 bash**：

```bash
bash hello.sh
```

```text
你好，root！今天是 2026-11-13 09:35:30，主机名 book
```

两种方式的本质差异：

| 方式 | 依赖 shebang 吗 | 依赖执行权限吗 | 本质 |
| --- | --- | --- | --- |
| `./hello.sh` | 依赖 | 依赖（chmod +x） | 系统「执行一个程序」，解释器由 shebang 决定 |
| `bash hello.sh` | 不依赖 | 不依赖 | bash「读入一个文本文件并逐行执行」，shebang 被当成注释忽略 |

推论都很实用：忘了 `chmod +x` 时 `bash hello.sh` 照样能跑（调试期省事）；shebang 写错时 `bash script.sh` 不报错、`./script.sh` 报 `bad interpreter`；`sh hello.sh` 是第三种方式——**Debian 上 `sh` 是 dash（一个更严格的 POSIX shell）**，不是 bash！用 `sh` 跑 bash 语法的脚本（比如下面要讲的 `[[ ]]`），会得到一串 `syntax error`。所以：**bash 脚本就用 `bash` 执行，别用 `sh`。**

<br />

## 13.3 变量与引用

### 13.3.1 定义与命名规则

Shell 变量（variable）就是「给一串文字起的名字」。规则四条，和大多数编程语言一致：

1. **只能由字母、数字、下划线组成，且不能以数字开头**：`name1` 合法，`1name` 非法。
2. **区分大小写**：`File` 和 `file` 是两个变量。
3. **等号两侧不能有空格**：`name=HuckOps` 合法，`name = HuckOps` 会被 shell 当成「用命令 `name`，参数 `=` 和 `HuckOps`」——这是新手遇到的第一个诡异报错。
4. 惯例上变量名用小写、常量（readonly）和环境变量用大写——全大写不是语法要求，是「一眼看出它会不会变」的纪律。

```bash
myname=HuckOps
echo $myname
```

```text
HuckOps
```

引用变量用 `$`。**大部分场合推荐 `${myname}` 的花括号形式**，因为它把变量名的边界明确圈出来了：

```bash
day=Friday
echo "今天是$day,good"     # 危险：shell 找的是变量 day，恰好对
echo "今天是${day},good"   # 正确
```

```text
今天是Friday,good
今天是Friday,good
```

上面这例子 `$day,` 恰好能解析，但假如变量名是 `d`：`echo $d,good` 就被解析成「变量 `d,good`」。花括号不是风格问题，是正确性问题：**只要变量名后面紧跟其他字符，一律用 `${}`**。

### 13.3.2 单引号、双引号与无引号

同一个变量值，三种引用方式三种结果，这是 shell 最经典的「入门考」：

```bash
msg="hello   world"
echo "双引号: $msg"
echo '单引号: $msg'
echo 无引号: $msg
```

```text
双引号: hello   world
单引号: $msg
无引号: hello world
```

规律总结成一张表：

| 写法 | `$` 变量替换 | 命令替换 `$( )` | 空格处理 | 通配符 `* ? []` |
| --- | --- | --- | --- | --- |
| 双引号 `"..."` | 展开 | 展开 | 保留（整体是一个词） | 不展开 |
| 单引号 `'...'` | **不展开** | **不展开** | 保留（整体是一个词） | 不展开 |
| 无引号 | 展开 | 展开 | **按空格拆成多个词** | **会展开成文件名** |

三条使用纪律：写死一段文本（正则、配置片段）用**单引号**；文本里要嵌变量用**双引号**；无引号只在你确实知道「这个词里没有空格、没有星号」时用。**「变量不加引号」是脚本第一大死因**：假设 `file=""`，`rm $file` 变成 `rm`（没事），但 `dirname="my docs"`，`cd $dirname` 会拆成两个词直接报错——正确写法永远是 `cd "$dirname"`。养成肌肉记忆：**引用变量必带双引号**。

### 13.3.3 readonly：只读变量

```bash
readonly BACKUP_ROOT=/data/home/backup
BACKUP_ROOT=/tmp        # 试图修改
```

```text
bash: BACKUP_ROOT: readonly variable
```

语义和它的名字一样直白：这个变量从此刻起只读，任何赋值都直接报错退出。适合用在脚本里锁死「不允许改的核心配置」，防止后面的代码（或未来的你）手滑覆盖。

### 13.3.4 export：变量与子进程的可见性

shell 变量分两种：**普通变量**只在当前 shell 里活着；**环境变量（environment variable）**会被「传递」给这个 shell 启动的所有子进程。转换开关是 `export`：

```bash
GREETING="hello from parent"
unset CHILD_VAR
echo "父 shell 里: $GREETING"
env | grep GREETING       # 父 shell 的环境里有没有它？
```

```text
父 shell 里: hello from parent
```

```bash
bash -c 'echo "子进程里: [$GREETING]"'
```

```text
子进程里: []
```

子进程里是空的——**普通变量不进环境，子进程看不见**。export 之后再试：

```bash
export GREETING
bash -c 'echo "子进程里: [$GREETING]"'
env | grep GREETING
```

```text
子进程里: [hello from parent]
GREETING=hello from parent
```

子进程读到了，`env`（列出全部环境变量）里也出现了。一次写成 `export GREETING="..."` 是等价的。记住这条链路：**export 只影响「往后由本 shell 启动的子进程」，不会影响父 shell，也不会回溯影响已经启动的进程。** 这就是第 15 章 cron 环境极简问题的根源：cron 启动的脚本环境里没有你交互 shell 里的那些变量，不 export 的、登录时才加载的变量，脚本里一个都没有。

`unset 变量名` 删除变量（不能删 readonly 的）。

<br />

## 13.4 特殊变量与参数展开

bash 有一批「名字由系统分配」的特殊变量（special variable），脚本里天天见：

| 变量 | 含义 | 典型用途 |
| --- | --- | --- |
| `$?` | **上一条命令的退出码**，0 成功、非 0 失败 | 判断成败、函数返回值，脚本的灵魂变量 |
| `$$` | 当前脚本/ shell 的 **PID**（进程号） | 加锁文件名、日志里标记「这是谁写的」 |
| `$0` | 当前脚本自己的名字/路径 | 日志和错误信息里标明身份 |
| `$#` | 位置参数的**个数** | 「用法不对就退出」的第一道检查 |
| `$1 $2 ...` | 第 1、第 2 个位置参数 | 脚本的「命令行选项」 |
| `$@` | 全部位置参数，**各自独立成词** | `for f in "$@"` 遍历，带空格的文件名不炸 |
| `$*` | 全部位置参数，**拼成一个词** | `echo "$*"` 之类「合成一句话」的场景 |

`"$@"` 和 `"$*"` 加不加引号是两个世界，演示给你看：

```bash
set -- "备份 目录" "报告.txt"     # set -- 手动设置位置参数，两个「文件名」（第一个带空格）
printf '[%s]\n' "$@"             # 每个参数一行
echo "-----"
printf '[%s]\n' "$*"             # 全部拼成一个词
```

```text
[备份 目录]
[报告.txt]
-----
[备份 目录 报告.txt]
```

**带空格的参数遍历必须用 `"$@"` 或 `"$*"`**：不加引号的 `$*` 会split，用 `"$@"` 每个参数原样保留。写 `for f in "$@"` 是备份脚本处理文件清单的标准姿势。

### 13.4.1 参数展开：默认值、长度、掐头去尾

三个最常用的展开语法，麻雀虽小威力巨大：

```bash
echo "未定义变量的默认值: ${UNDEFINED_VAR:-使用默认值}"
word="hello"
echo "字符串长度: ${#word}"          # ${#var} 统计字符数
file="backup-2026.tar.gz"
echo "掐掉后缀: ${file%.tar.gz}"     # % 从右边删掉最短匹配
echo "掐掉前缀: ${file#backup-}"     # # 从左边删掉最短匹配
```

```text
未定义变量的默认值: 使用默认值
字符串长度: 5
掐掉后缀: backup-2026
掐掉前缀: 2026.tar.gz
```

- `${var:-默认}`：变量不存在或为空时用默认值，**原变量不变**。适合「配置项可缺省」。
- `${#var}`：字符串长度，写循环、校验输入常用。
- `${var%后缀}`、`${var#前缀}`：从右边/左边删掉匹配的模式——处理文件名（去扩展名、去目录前缀）比调用 `sed` 轻得多。更复杂的场景交给第 7 章《文本处理：grep / sed / awk》的工具，但这两个小语法在脚本里值得形成条件反射。

还有两个同门兄弟留下一句话印象：`${var:=默认}`（不只是替换，还把默认值**写回变量**）、`${var:?提示}`（变量为空就报错退出，`set -u` 之外的第二道保险）。

<br />

## 13.5 read 交互与位置参数

### 13.5.1 read：从键盘读一行

`read`（读入）从标准输入读一行，按空格切成若干段，依次赋给变量：

```bash
read -p "请输入备份目录（回车用默认 /data/home/backup）: " dir
echo "收到: $dir"
```

```text
请输入备份目录（回车用默认 /data/home/backup）: 
收到: 
```

回车不输入，`dir` 就是空字符串。这就是 13.4.1 的用武之地：`dir=${dir:-/data/home/backup}`。`read` 的常用选项：

| 选项 | 作用 |
| --- | --- |
| `-p "提示"` | 先打印提示语再读（不带换行），免去单独写一行 echo |
| `-s` | 静默：输入不回显，读密码用（`read -s -p "Password: " pw`） |
| `-t 秒数` | 超时：N 秒内没有输入就放弃（自动化里防「卡脚本等人」） |
| `-r` | 不把反斜杠当转义符——读路径、读配置行的默认推荐姿势 |

### 13.5.2 位置参数：脚本的「命令行」

脚本可以接收参数，运行姿势是 `./backup.sh 源 目标`，脚本内用 `$1 $2` 取：

```bash
cat > check-args.sh <<'EOF'
#!/bin/bash
# check-args.sh —— 检查「用法」：参数个数不对直接教用户怎么用
if [ $# -ne 2 ]; then
    echo "用法: $0 <源目录> <目标目录>" >&2
    exit 1
fi
echo "源: $1"
echo "目标: $2"
EOF
bash check-args.sh
```

```text
用法: check-args.sh <源目录> <目标目录>
```

不带参数运行，`$#` 是 0，脚本打印用法并以退出码 1 退出——**这就是 `$#` 加 `exit 1` 的标准组合：脚本的第一件事永远是校验用法**，参数都不对，跑下去就是错上加错。带上两个参数再运行一次，得到「源」和「目标」两行，退出码 0。`shift` 命令可以「消费」掉一个位置参数（原来的 `$2` 变成 `$1`），处理多个同质参数（比如 `backup.sh dir1 dir2 dir3`）时好用。

<br />

## 13.6 算术运算：$(( ))、let 与 bc

bash 的算术用 `$(( ))`，只支持整数：

```bash
echo "1+2=$((1+2))"                    # 加减直接写
echo "2 的 10 次方=$((2**10))"         # ** 是幂
echo "7/2=$((7/2))"                    # 整数除法：3，小数部分直接砍掉
echo "7%3=$((7%3))"                    # 取余：1
a=10; echo "a*3+1=$((a*3+1))"          # 变量免加 $，直接参与运算
```

```text
1+2=3
2 的 10 方=1024
7/2=3
7%3=1
a*3+1=31
```

（`**` 在 bash 的算术展开里是幂运算，`2**10=1024`。）三个易错点先立好：**除法是整数除法**（`7/2` 得 3，想要小数见下面的 bc）；**变量参与运算不用写 `$`**（写 `$a` 也没错，但不必要）；**空格可有可无**（`$(( 1 + 2 ))` 完全等价）。

`let` 是等价的旧写法（`let x=1+2`），还有 `(( ))` 形式（`((x=1+2))`）。后者有一个**经典坑**必须点名：

```bash
count=0
(( count++ )); rc=$?
echo "count 现在 = $count，但 (( count++ )) 这条命令的退出码 = $rc"
result=$(( count - count ))
(( result )); echo "对值为 0 的 result 做 (( result ))，退出码 = $?"
```

```text
count 现在 = 1，但 (( count++ )) 这条命令的退出码 = 1
对值为 0 的 result 做 (( result ))，退出码 = 1
```

规则：`(( 表达式 ))` 的**退出码**取决于「表达式结果是否为 0」——结果 0 时退出码是 1（失败）。上面第一行的坑在于 `count++` 求的是**旧值 0**，于是整行变成「失败」，哪怕 count 实际已经加到 1；第二行更直白：表达式值是 0，退出码就是 1。所以 `(( 0 ))`、`(( total=0 ))` 这类命令，在后面接 `if`、`&&`、或遇到 `set -e` 时表现极其反直觉。防御写法：只用 `$(( ))` 求值，把 `(( ))` 留给明确的布尔场合（如 `if (( a > b ))`，这是它的舒适区）。

**浮点和高精度运算用 `bc`**（一个独立的计算器程序，Debian 大概率没预装，`apt install bc`，详见第 10 章）：

```bash
echo "scale=2; 10/3" | bc        # scale=2 指定小数位数
echo "scale=4; 4*a(1)" | bc -l   # -l 加载数学库，a(1) 是 arctan(1)=π/4
```

```text
3.33
3.1415
```

<br />

## 13.7 条件测试：test 与 [ ]、[[ ]]

shell 里没有「布尔类型」，判断靠一个叫 `test` 的命令（以及它的两个别名 `[ ]` 和 `[[ ]]`）。**测试的本质是执行一条命令，看它的退出码**：0 表示「条件成立」，1 表示「不成立」（还记得 13.4 的 `$?` 吗，串起来了）。

### 13.7.1 文件与字符串测试

```bash
[ -e /etc/hostname ] && echo "/etc/hostname 存在"
[ -d /data/home ] && echo "/data/home 是目录"
[ -f /data/home/hello.sh ] && echo "hello.sh 是普通文件"
[ -r /etc/shadow ] && echo "root 可读 /etc/shadow"
[ -w /etc ] && echo "/etc 可写（root 能写）"
[ -x /bin/ls ] && echo "/bin/ls 可执行"
[ -s /data/home/hello.sh ] && echo "hello.sh 非空"
f=""; [ -z "$f" ] && echo "f 是空字符串"
[ -n "abc" ] && echo "abc 是非空字符串"
```

```text
/etc/hostname 存在
/data/home 是目录
hello.sh 是普通文件
root 可读 /etc/shadow
/etc 可写（root 能写）
/bin/ls 可执行
hello.sh 非空
f 是空字符串
abc 是非空字符串
```

速查表，按记住优先级排：

| 类别 | 选项 | 成立条件 |
| --- | --- | --- |
| 文件存在性 | `-e path` | 存在（文件、目录、链接都算） |
| | `-f path` | 存在**且是普通文件** |
| | `-d path` | 存在**且是目录** |
| | `-s path` | 存在且**非空** |
| 权限位 | `-r` / `-w` / `-x` | 当前用户可读 / 可写 / 可执行 |
| 字符串 | `-z s` | 长度为 0（zero） |
| | `-n s` | 长度非 0（nonzero） |
| 字符串比较 | `[ "a" = "a" ]`、`[ "a" != "b" ]` | 相等 / 不相等 |

「当前用户可读」这点值得停一秒：`-r` 看的是**真实权限判定**而不是权限位。普通用户测 root 私有的文件，`-r` 就为假——条件判断天然带上了第 6 章的权限模型。

**新手第一死法献给字符串测试**：`[ $s = "abc" ]` 在 `s` 为**空**时会展开成 `[ = "abc" ]`，报 `unary operator expected`；在 `s` 含空格时更糟。防御写死：**`[ ]` 里所有变量都加双引号**（`[ "$s" = "abc" ]`），或者干脆用下面的 `[[ ]]`。

### 13.7.2 数值比较：-eq 家族

数值比较**不能**用 `=`，要用 `-eq` 系列：

```bash
x=10
[ "$x" -eq 10 ] && echo "x 等于 10"
[ "$x" -gt 5 ] && echo "x 大于 5"
[ "$x" -ne 9 ] && echo "x 不等于 9"
```

```text
x 等于 10
x 大于 5
x 不等于 9
```

`-eq`（equal）、`-ne`（not equal）、`-gt`（greater than）、`-ge`（greater or equal）、`-lt`（less than）、`-le`（less or equal）。**注意 `-gt` 两边也必须有引号或确定是数字**：`[ x -gt 5 ]`（x 没美元符号）会被当成字符串「x」，报 `integer expression expected`。

文件新旧比较也归 `test` 管，写增量备份、构建缓存天天用：`[ file1 -nt file2 ]`（file1 比 file2 **新**才成立）、`[ file1 -ot file2 ]`（比它旧）。

### 13.7.3 [[ ]]：bash 增强版

`[[ ]]` 是 bash 的内置增强语法（不是外部命令，POSIX 的 sh 没有它，所以 `sh` 跑会报错——13.2.2 的回旋镖来了）。四个关键增强：

```bash
s=""
[[ $s = "abc" ]] || echo "s 不等于 abc（注意：s 是空的，也没报错）"
f="app.log"
[[ $f == *.log ]] && echo "f 以 .log 结尾（模式匹配，右边不是普通字符串！）"
a=5; b=10
[[ $a -lt $b && $a -gt 0 ]] && echo "变量没加引号也安全，&& || 直接写在里面"
[[ "b" > "a" ]] && echo "[[ ]] 里可以直接做字典序比较"
```

```text
s 不等于 abc（注意：s 是空的，也没报错）
f 以 .log 结尾（模式匹配，右边不是普通字符串！）
变量没加引号也安全，&& || 直接写在里面
[[ ]] 里可以直接做字典序比较
```

1. **变量不引也安全**：空变量、带空格、带星号都不会炸（它在语法层就不做分词和通配展开），治好了 13.7.1 的第一死法。
2. **`==` 右边是模式**（pattern）：`*.log`、`backup-20??.tar.gz` 这种 glob 直接匹配，等价比要写 `case`。注意这时左边仍不加引号（引号会让模式失效）。
3. **`&&` 和 `||` 能写进 `[[ ]]` 内部**，替代 `[ ] && [ ]` 的长链，少起进程、少出错。
4. **`<` `>` 是字典序比较**（`[ ]` 里它们是重定向符号，会直接闯祸）。

纪律：**bash 脚本一律优先 `[[ ]]`**；`[ ]` 只在「要兼容 POSIX sh」时用（比如给 dash 写的系统脚本）。本节后面所有例子默认 `[[ ]]`。

<br />

## 13.8 退出码与 &&、||、!

把 13.7 开头那句话再钉一遍：**shell 世界里「成功」长这样——退出码 0；失败是 1 到 255 的任何非零值。** 这反常识（0 竟然代表成功），但整个 shell 的逻辑都建立在这上面：`if`、`while`、`&&`、`||`、`set -e` 全是退出码的下游。

常用退出码速认（遇到时能马上知道「出了什么事」）：

| 退出码 | 含义 |
| --- | --- |
| 0 | 成功 |
| 1 | 一般性失败（命令自己报的错、测试不成立） |
| 2 | shell 语法错误（用法写错） |
| 126 | 找到了文件但不能执行（多半是漏了 chmod +x） |
| 127 | 命令不存在（PATH 里找不到，拼写错） |
| 128+n | 被第 n 号信号杀死（常见 130=Ctrl-C，137=SIGKILL） |
| 124 | 超时被杀（第 1 章说的 `instabox exec` 单条 10 分钟超时就是这个码） |

组合技三个符号：

```bash
true  && echo "成功才执行"      # && ：左边成功，才执行右边
false || echo "失败才执行"      # || ：左边失败，才执行右边
[ 1 -gt 2 ] || echo "实测: 1 不大于 2"
! [ 1 -gt 2 ] && echo "取反后成立"   # ! ：把条件反过来
```

```text
成功才执行
失败才执行
实测: 1 不大于 2
取反后成立
```

配合 `$?` 立刻用上：

```bash
grep -q root /etc/passwd; echo "grep 的退出码: $?"
```

```text
grep 的退出码: 0
```

（`grep -q` 是安静模式：只返回退出码不打印内容，脚本里判断「文件里有没有某行」的标准姿势，参数详见第 7 章。）`cmd1 && cmd2 || cmd3` 是「成功做 B，失败做 C」的紧凑写法，但别嵌套超过一层，复杂分支老老实实写 `if`——评审脚本时没人愿意解析你的符号迷宫。

<br />

## 13.9 流程控制：if、case、for、while、until

### 13.9.1 if / elif / else

```bash
cat > check-disk.sh <<'EOF'
#!/bin/bash
# check-disk.sh —— 检查 /data 磁盘使用率，超过阈值给警告
threshold=80
usage=$(df -h /data | awk 'NR==2 {print $5}' | tr -dc '0-9')
if [[ -z "$usage" ]]; then
    echo "取不到使用率，放弃" >&2
    exit 2
elif [ "$usage" -ge "$threshold" ]; then
    echo "警告: /data 使用率 ${usage}%，超过 ${threshold}%"
    exit 1
else
    echo "正常: /data 使用率 ${usage}%"
    exit 0
fi
EOF
bash check-disk.sh
```

```text
正常: /data 使用率 13%
```

结构逐段说：`if 命令; then ...; elif 命令; then ...; else ...; fi`——**注意 if 后面跟的是「命令」不是「表达式」**，上面这个例子里是 `[[ ... ]]` / `[ ... ]`，但 `if grep -q x file; then` 同样成立。分号在同行书写时不能省；`then`/`else`/`fi` 是「反着拼的关键词」家族（if/fi、case/esac、do/done），bash 的老传统。`>&2` 是把错误信息送去标准错误（stderr），错误信息进错误流、正常输出进标准输出——这样 `2> /dev/null` 才能单独屏蔽错误，这个习惯第 7 章的重定向一节讲过，脚本里同样适用。

### 13.9.2 case：多路分支

`if/elif` 超过三路就该换 `case` 了。**两种语法细节必须记牢：每个分支以 `;;` 结尾，写 `*)` 兜底**：

```bash
cmd=${1:-status}      # 第一个参数缺省当作 status
case "$cmd" in
    start|restart)   # | 表示「或」，start 和 restart 一个分支
        echo "执行: $cmd"
        ;;
    stop)
        echo "执行: stop"
        ;;
    status|"")
        echo "执行: status（或空参数）"
        ;;
    *)               # * 兜底：以上都不匹配时走这里
        echo "未知命令: $cmd" >&2
        exit 2
        ;;
esac
echo "case 执行完毕，退出码 $?"
```

```text
执行: status（或空参数）
case 执行完毕，退出码 0
```

（把第一个参数换成 `start`、`stop`、`reboot` 各跑一遍，看四个分支的走向。）`;;` 的含义是「这个分支到此结束，跳出 case」——漏写它，bash 会继续往下执行下一个分支，是最常见的 case 事故。`*)` 兜底分支强烈建议永远写，它是「未知输入」的防坠网，生产经验里大量事故源于「以为用户只会传 A 或 B，结果传了 C」。

### 13.9.3 for：遍历列表与计数循环

```bash
for s in /data/home /var/log /etc; do
    echo "== $s =="
    df -h "$s" | awk 'NR==2 {print "  使用率", $5}'
done
```

```text
== /data/home ==
  使用率 13%
== /var/log ==
  使用率 41%
== /etc ==
  使用率 41%
```

（`in` 后面是**由空格分隔的一串词**，变量 `s` 依次取每个值。）bash 专有的 C 风格计数循环，三段表达式（初始化; 条件; 步进）：

```bash
sum=0
for ((i = 1; i <= 10; i++)); do
    sum=$((sum + i))
done
echo "1 加到 10 = $sum"
```

```text
1 加到 10 = 55
```

还有个偷懒写法 `for i in {1..10}`（花括号展开，第 4 章），一行生成 1 到 10 的列表，但**做不了动态边界**（`{1..$n}` 不展开）——步进要控制时（`{1..10..2}` 步长 2）可以，动态场景回到 `for ((;;))`。

### 13.9.4 while、until 与 read line：逐行处理文件

```bash
count=0
while read -r line; do
    count=$((count + 1))
done < /etc/passwd
echo "/etc/passwd 共 $count 行"
```

```text
/etc/passwd 共 26 行
```

`while read -r line` 是「逐行读文件」的万能句式：`-r` 关掉反斜杠转义（读路径、配置行的默认姿势），标准输入**从 `< 文件` 来**。同族的 `until [[ 条件 ]]; do ... done` 是「条件不成立才循环」（比如反复 `curl` 直到服务起来了再继续）。

**下一个是本节最重要的坑：管道会把 while 送进子 shell。**

```bash
count=0
echo "管道版（有坑）:"
cat /etc/passwd | while read -r line; do
    count=$((count + 1))
done
echo "  count=$count  ← 循环明明跑完了，父 shell 里却纹丝不动"

count=0
echo "重定向版（正确姿势）:"
while read -r line; do
    count=$((count + 1))
done < /etc/passwd
echo "  count=$count"
```

```text
管道版（有坑）:
  count=0  ← 循环明明跑完了，父 shell 里却纹丝不动
重定向版（正确姿势）:
  count=26
```

管道右侧的 `while` 在 bash 里跑在**子 shell**里：循环里对 `count` 的修改发生在子 shell 的变量空间，循环一结束子 shell 退出，父 shell 里的 `count` 原封不动。**遍历结束时值带不回来——这是 `cat file | while read line` 最大的暗坑**，写统计逻辑时折磨过无数人。两个正解：一是本例的 `done < file`（重定向，无管道、无子 shell）；二是一定要用管道来源（比如处理命令输出）时，改用进程替换：`while read -r line; do ... done < <(some command)`，或者 `shopt -s lastpipe`（让管道最后一段留在当前 shell，仅非交互 shell 生效）。三选一，选你读得懂的那个。

> **SRE 贴士**：这个子 shell 陷阱在真实生产里的作案手法极其隐蔽——夜里跑的巡检脚本 `curl ... | grep -c ... | while read n; do ... done`，白天看日志一切正常，隔天发现昨晚该发的磁盘告警没人处理。故障复盘时人盯着 `cat | while` 看半小时都看不出问题，因为每一行单独看都对。根治就一招：**循环体内要改循环体外的变量，就不能用管道喂它**——改用 `done < 文件` 或 `done < <(命令)`。

`break` 立刻跳出循环、`continue` 跳过本轮进入下一轮，和 C 系列语言一致。还值得一提的 `select`（生成带编号的数字菜单），知道 bash 有这玩意即可，本书不展开。

<br />

## 13.10 函数：把脚本攒成积木

```bash
log() {                       # 函数定义，圆括号不能带参数
    echo "[$(date '+%F %T')] $*" | tee -a /data/home/scripts/run.log
}

die() {
    log "FATAL: $*"
    exit 1
}

check_tools() {
    local missing=()          # local：函数内变量。详见下文
    for t in tar df awk; do
        command -v "$t" >/dev/null || missing+=("$t")
    done
    if [ ${#missing[@]} -gt 0 ]; then
        die "缺少命令: ${missing[*]}"
    fi
}

log "脚本开始，PID=$$，参数个数=$#"
check_tools
log "工具检查通过，继续"
```

```text
[2026-11-13 09:41:00] 脚本开始，PID=1781，参数个数=0
[2026-11-13 09:41:00] 工具检查通过，继续
```

要点逐个过：

- **定义**：`名字() { 命令; }` 或 `function 名字 { ...; }`，现代脚本用前者。**定义必须写在调用之前**，所以函数统一放脚本开头。
- **位置参数独立**：函数内 `$1 $2 $# $@` 指的是**传给函数的参数**，和脚本的位置参数互不干扰。上面 `log` 用 `$*` 把全部参数拼成一句话输出。
- **return vs $()**：`return N` 只能返回 0~255 的**退出码**（和命令的成功失败一个语义），不是「返回值」。函数想交出**数据**，用 `echo` 打印 + 命令替换 `$(函数名)` 接收。分不清这两者的脚本，逻辑一定乱。
- **local 是纪律不是装饰**：函数里不加 `local` 的变量**全部是全局变量**，改一次污染整个脚本，而且污染会静默传递。写函数的第一行就规划好哪些变量要 `local`。

<br />

## 13.11 健壮性：set -euo pipefail、IFS 与 trap

「能跑的脚本」和「能在凌晨三点无人值守跑的脚本」之间隔着这一节。

### 13.11.1 set -euo pipefail 逐项拆

放在 shebang 之后，一行管全脚本：

```bash
set -euo pipefail
```

| 选项 | 作用 | 防的是哪类事故 |
| --- | --- | --- |
| `set -e`（errexit） | 任意命令返回非零退出码，**脚本立刻退出** | 「第 3 行 tar 失败了，第 4 行照样 rm 源文件」——错误不扩散，不带着伤继续跑 |
| `set -u`（nounset） | 引用**未定义**的变量直接报错退出 | 变量名拼错（`$backupdri` 而非 `$backupdir`）时静默变成空字符串，引发更深的灾难 |
| `set -o pipefail` | 管道（`a | b | c`）任一环非零，**整条管道算失败** | 默认 bash 只看管道最后一段的退出码：`tar ... | gzip` 中 tar 炸了、gzip 成功，退出码仍是 0，备份坏成空包还提示「成功」 |

三者是叠加关系，合起来是「fail fast（快速失败）」三件套。**但 `set -e` 绝不是「加上就安全」的护身符**，它的盲区必须补课：

> **警告**：`set -e` 的例外清单比你想的长——以下场景命令失败了脚本**不会**退出：`if` / `while` 的条件部分；`&&` / `||` 链的非末段（`false && true` 不触发）；被 `!` 取反的命令；函数在条件语境里被调用。更反直觉的是管道里中间段、以及 `$( )` 命令替换里的失败默认传播得很弱。真实事故现场长这样：`set -euo pipefail` 的脚本里 `cp file1 dir || true` 之后，运维以为「反正继续跑了」，结果是本节想防的那类事故换了个马甲回来——`|| true` 把「cp 失败」吃成了「成功」，而后面没人再检查。安全替代做法有三条，按优先级排列：一、**逐命令显式判断**而不是靠 `set -e` 兜底：关键动作写 `if ! cp a b; then log "FATAL"; exit 1; fi`，让失败处理成为代码的主干而不是暗坑；二、确实「允许失败」的命令（`grep -q` 找不到、`useradd` 已存在）用 `|| true` 且**旁边写注释说明为什么允许**，评审时优先审这些豁免；三、调试期用 `bash -x`（13.12）跟一遍真实退出码，别凭想象推测 `set -e` 的行为。一句话纪律：**`set -e` 是兜底的安全网，不是设计本身；每条「允许失败」的命令都必须有名字写着的理由。**

### 13.11.2 IFS：安全分隔

IFS（Internal Field Separator，内部字段分隔符）决定 bash 把一行文本按什么字符切成词，默认是「空格 + 制表符 + 换行」。`read a b <<< "x y"` 能切开、`for f in $(ls)` 翻车，都是它在幕后操作。**脚本开头复位成一个干净值**是通行做法：

```bash
IFS=$' \t\n'      # 显式声明默认三件套（\t 是制表符，\n 是换行）
```

什么时候动它：处理路径/文件清单、读 CSV。**持久覆盖 IFS 之前想清楚作用域**——它影响整个脚本的所有分词行为，改完不记得恢复，是下一段代码莫名拆词的幽灵。惯例是「想特殊处理时在函数内 `local IFS=$'\n'`，出了函数自动还原」。

### 13.11.3 trap：被中断时也要收拾干净

脚本跑到一半被 Ctrl-C（或系统杀进程），会留下一地垃圾：半个备份文件、没删的临时目录、写了一半的状态文件。`trap` 给信号绑一个「回调」：

```bash
cleanup() {
    rm -rf "$TMPDIR"          # 收拾临时文件
    log "收到中断信号或脚本退出，清理完成"
}
TMPDIR=$(mktemp -d /data/home/tmp.XXXXXX)
trap cleanup EXIT             # 脚本退出时（无论正常与否）
trap 'echo "收到 SIGINT，中止中"; exit 130' INT   # 捕获 Ctrl-C
trap 'echo "收到 SIGTERM，中止中"; exit 143' TERM # 捕获 systemctl stop / kill
log "临时目录: $TMPDIR"
```

```text
[2026-11-13 09:43:12] 临时目录: /data/home/tmp.Ab3xY9
[2026-11-13 09:43:12] 收到中断信号或脚本退出，清理完成
```

（第二行来自脚本自然退出时 EXIT 触发；跑着按 Ctrl-C 则会先看到 SIGINT 那行。）三件事值得讲清：**EXIT 信号**在脚本以任何方式结束时都触发，是「清理函数」的标准锚点；`trap ... INT TERM` 让你在被杀前有机会体面收尾——生产上 `systemctl stop`（第 13 章）发的就是 SIGTERM，能不能优雅退出全靠它；`mktemp -d` 生成「随机名临时目录」，比手写 `mkdir /tmp/xxx` 安全（不会撞名、不会被别人预创建攻击）。

<br />

## 13.12 调试：bash -x、PS4 与常见报错解读

### 13.12.1 bash -x：逐行看执行

```bash
cat > bug.sh <<'EOF'
#!/bin/bash
set -euo pipefail
name="sre book"
echo "hello $name"
echo "count: $((3 / 0))"
echo "这行还会执行吗？"
EOF
bash -x bug.sh
```

```text
+ set -euo pipefail
+ name='sre book'
+ echo hello 'sre' book
+ 3 / 0
bash: bug.sh: line 5: 3 / 0: division by 0
```

`bash -x` 把每条命令**展开后**、执行前打印出来（默认前缀 `+ `）。三个信息量：你看到的是「真正执行的命令」而不是「你以为写的命令」——变量替换、引号剥离全部现形；最后一行 `division by 0` 直接点名了行号 **line 5**；`set -e` 生效，脚本当场死在第 5 行，末行 `echo` 没有出现。调试三板斧的分工：`bash -n 脚本` **只查语法不执行**（先过一遍拼写）；`bash -x 脚本` **逐行跟执行**（看运行到哪、变量是什么）；`set -x ... set +x` **只调试脚本中间一段**（别被 `set -x` 的全量输出淹没）。

### 13.12.2 PS4：给 trace 加行号

`-x` 默认只打印 `+`，行多了不知道是谁家孩子。自定义 `PS4` 环境变量，把行号和函数名带上：

```bash
PS4='+[$0:$LINENO]${FUNCNAME:+(${FUNCNAME})} ' bash -x bug.sh 2>&1 | head -5
```

```text
+[bug.sh:2]() set -euo pipefail
+[bug.sh:3]() name='sre book'
+[bug.sh:4]() echo hello 'sre' book
+[bug.sh:5]() echo count: 0
+[bug.sh:5]() 3 / 0
```

`$LINENO` 是 bash 内置的当前行号变量，`${FUNCNAME:+...}` 表示「在函数里才加函数名」。**生产脚本排障时这行咒语值得背下来**：几百行脚本的 trace 输出，有了文件名和行号才是可读的。

### 13.12.3 初学者十大报错速查

| 报错 | 含义 | 处方 |
| --- | --- | --- |
| `Permission denied`（执行脚本时） | 少了执行位 | `chmod +x script.sh` |
| `bad interpreter: /bin/bash^M` | 从 Windows 编辑器带出了 CR 回车符 | `sed -i 's/\r$//' script.sh` 洗掉，或存成 Unix 换行 |
| `command not found` | 拼错、或 PATH 里没有（cron 场景高发，见第 15 章） | 用绝对路径；`command -v cmd` 验证 |
| `syntax error near unexpected token` | `if`/`fi` 不配对、`[[` 用了 sh 跑等 | `bash -n` 定位行号；确认用 bash 执行 |
| `unary operator expected` | `[ ]` 里变量为空且没加引号 | 换 `[[ ]]`，或给变量加双引号 |
| `integer expression expected` | 拿字符串和 `-gt` 比 | 只对数字用 `-eq` 系列；确认变量是数字 |
| `division by 0` | `$(( ))` 除零 | 先判断除数非零 |
| `too many arguments` | `$*` 没加引号被拆成一片 | 遍历永远用 `"$@"` |
| `readonly variable` | 动了 readonly 变量 | 删掉赋值，或重新起 shell |
| 脚本「一切正常」但退出码是 1 | `set -e` + 最后一条命令失败；或函数 `return` 了非零 | 脚本末尾显式 `exit 0`（成功后主动声明） |

<br />

## 13.13 实战：/data/home 备份脚本

现在把本章所有积木攒成一个能上生产姿态的脚本。**需求**：把 `/data/home` 打包成 tar.gz 存到 `/data/home/backup`，文件名带日期戳，备份完自检包的完整性，全程留日志，成功失败都有正确退出码。这个脚本是第 16 章《防火墙与数据备份》的底稿，第 15 章将让它每天自动跑。

```bash
vim /data/home/scripts/backup-home.sh
```

```bash
#!/bin/bash
# backup-home.sh —— 把 /data/home 打包备份到 /data/home/backup
# 用法: ./backup-home.sh
set -euo pipefail
IFS=$' \t\n'

SRC="/data"                           # 备份源：整个 /data（tar 用 -C 切入）
SRC_DIR="home"                        # 实际备份 /data/home（持久盘家目录）
DEST="/data/home/backup"              # 备份目标目录
STAMP="$(date '+%Y%m%d-%H%M%S')"      # 日期戳，精确到秒，绝不覆盖历史
FILE="${DEST}/home-${STAMP}.tar.gz"
LOG="${DEST}/backup-home.log"

log() { echo "[$(date '+%F %T')] $*" | tee -a "$LOG"; }
die() { log "FATAL: $*"; exit 1; }

cleanup() {
    log "backup-home.sh 退出，退出码=$?"
}
trap cleanup EXIT

# 1. 环境自检
[ -d "/data/${SRC_DIR}" ] || die "源目录不存在"
command -v tar >/dev/null || die "找不到 tar 命令"
mkdir -p "$DEST"

# 2. 打包（排除备份目录本身，否则备份会「自己套自己」疯长）
log "开始备份 -> ${FILE}"
tar -czf "$FILE" -C "$SRC" --exclude="${SRC_DIR}/backup" "$SRC_DIR"

# 3. 完整性自检：没验证过的备份等于没备份（恢复演练详见第 16 章《防火墙与数据备份》）
if ! tar -tzf "$FILE" > /dev/null 2>&1; then
    rm -f "$FILE"
    die "备份文件损坏，已删除 $FILE"
fi
log "备份完成，大小 $(du -h "$FILE" | cut -f1)"
exit 0
```

跑起来：

```bash
chmod +x /data/home/scripts/backup-home.sh
/data/home/scripts/backup-home.sh
```

```text
[2026-11-13 09:45:01] backup-home.sh 退出，退出码=0
```

日志在 `/data/home/backup/backup-home.log` 里看全程，产物看一眼：

```bash
cat /data/home/backup/backup-home.log
ls -lh /data/home/backup/
```

```text
[2026-11-13 09:45:01] 开始备份 -> /data/home/backup/home-20261113-094501.tar.gz
[2026-11-13 09:45:01] 备份完成，大小 3.4M
[2026-11-13 09:45:01] backup-home.sh 退出，退出码=0
total 4.1M
-rw-r--r-- 1 root root 3.4M 11-13 09:45 home-20261113-094501.tar.gz
-rw-r--r-- 1 root root  247 11-13 09:45 backup-home.log
```

几个没来得及展开的设计决策，评审脚本时它们会被一个个问：**日期戳到秒**——每天跑两次也不会互相覆盖，历史留痕；`--exclude` 排除备份目录——否则今天的备份会把昨天的备份也打进去，体积指数膨胀，这是真实运维里发生过无数次的事故；`tar -tzf` 只列表不解包——低成本验证压缩包结构完整，比自检步骤里说的「没验证过的备份等于没备份」更日常；`trap ... EXIT` 记退出码——凌晨的定时任务没人盯着，日志里必须留下「它是怎么结束的」；显式 `exit 0`——不让 `set -e` 时代码末尾的偶然返回值决定脚本成败。

> **SRE 贴士（生产 vs 实验室）**：实验室里备份在 `/data/home` 本地目录就完事；生产的三二一原则是「三份副本、两种介质、一份异地」（详见第 16 章），脚本的生命周期管理也更严：谁改的（git 提交记录）、改完怎么验证（跑一次 dry-run / 恢复演练）、什么时候失效（定期复盘）。另外别忘了第 15 章预告的那个坑——这个脚本现在手动跑，明天交给 cron，环境会变得「极简」：不 source 你的 `.bashrc`、PATH 只有 `/usr/bin:/bin`，脚本里所有命令要么绝对路径、要么 `command -v` 自检，这两件事本章已经替你做了。

<br />

## 本章小结

这一章你从「敲命令的人」变成了「写脚本的人」。主线可以压成一张地图：

```text
脚本结构：shebang(#! /bin/bash) → set -euo pipefail → IFS → 常量/变量 → 函数 → 主流程 → trap EXIT → 显式 exit
变量引用：命名(字母数字下划线,不能数字开头) | 双引号(可展开) vs 单引号(纯字面) | 变量必须加引号
特殊变量：$? 退出码  $$ PID  $0 脚本名  $#/$@/$*  ${var:-默认}  ${#var}  ${var%后缀}
运算：    $(( )) 整数(除法取整) | (( )) 有退出码陷阱 | bc 做浮点
测试：    [[ ]] 优先：-e -f -d -r -w -x、-z -n、= !=、-eq -ne -gt/-lt、-nt -ot、模式 == *.log
控制流：  if/elif/else/fi、case(;; 与 *) 、for in / for ((;;))、while read < 文件(警惕管道子 shell)、until、break/continue
函数：    定义在调用前 | $1/$# 独立 | local 防全局污染 | return 退出码、echo/$() 传数据
调试：    bash -n 查语法 → bash -x 跟执行 → PS4 带行号
```

要带走的是三个习惯：**引用变量必带引号**（防分词的第一道墙）；**每条「允许失败」的命令都要写得出理由**（`set -e` 时代码评审的第一课）；**脚本的结束方式要显式**（`exit 0` / `die()`，别让运气决定退出码）。下一章，让这些脚本按时自己跑起来，并学会看懂系统的「黑匣子」。

## 动手练习

1. 写出你的第一个脚本 `whoami.sh`：shebang 用 `/bin/bash`，输出当前用户、主机名、内核版本（`uname -r`）和 `/data` 使用率（参考 13.9.1 的 `df` + `awk` 写法）。`chmod +x` 后分别用 `./whoami.sh`、`bash whoami.sh`、`sh whoami.sh` 三种方式执行，记录三者结果的差异并解释原因。
2. 变量实验：定义 `msg="sre book"`，用双引号、单引号、无引号各 `echo` 一次；再定义 `version=1.10`，分别 `echo $version.0` 和 `echo ${version}.0` 解释差别；最后 `export` 一个变量并用 `bash -c` 验证子进程可见性，`env | grep` 确认它进了环境。
3. 写 `check-file.sh`：要求恰好一个位置参数，否则打印用法到 stderr 并以退出码 1 退出；参数合法时依次报告「路径是否存在、是文件还是目录、是否可读、大小」，每一项有明确中文结论；自己用「存在的文件、不存在的路径、不带参数」三种输入各测一次，记录三种退出码。
4. 循环与作用域：用 `for ((i=1;i<=100;i++))` 求 1 到 100 的和；再写「管道版」和「重定向版」两个 `while read -r line` 统计 `/etc/passwd` 行数，确认管道版的计数为什么带不回来，并写下你选择的修复方式。
5. 健壮性改造：拿第 4 题里没加 `set -euo pipefail` 的脚本，补上三件套和 `trap cleanup EXIT`；故意在循环里触发一次除零或访问一个不存在的变量（`set -u` 抓未定义变量），用 `bash -x` 观察脚本死在第几行、报什么错。
6. 综合题：完善 13.13 的 `backup-home.sh`——给它加上「保留最近 5 份、自动删除更早」的逻辑（提示：`ls -1t` 排序后 `tail -n +6` 取旧文件），跑一次验证备份+自检+清理全链路，然后把备份包 `tar -tzf` 列表的前 10 行贴进笔记；把这个脚本交给第 15 章练习 2 的 cron 任务。

> **参考答案**：
> 1. `sh` 是 dash：`$(...)`、`uname -r` 这些 POSIX 语法它能跑，所以三者输出可能「几乎一样」；但只要脚本出现 bash 专有语法（`[[ ]]`、`(( ))`、数组），`sh` 立刻报 `syntax error`——差异在「语法集」不在「这次输出」。写 bash 脚本就用 `bash` 或 `./`（靠 shebang）执行。
> 2. `$version.0` 被解析成变量 `version` 加字面量 `.0`，正好「侥幸正确」；真正会翻车的是 `$ver.0` 找不到变量 `ver.0` 的场合——所以一律 `${}`。export 后 `bash -c 'echo "$var"'` 能读到，`env | grep` 出现该变量；普通变量两者都读不到。
> 3. 关键三行：`[ $# -ne 1 ] && { echo "usage: $0 <path>" >&2; exit 1; }`；存在用 `-e`，`-d`/`-f` 区分目录与文件，`-r` 测可读，`du -h`/`stat` 看大小；三种退出码应为 1（缺参）、0（存在文件）、0 或 1（不存在时给出「不存在」结论后主动 exit 0/1，你的设计要能自洽）。
> 4. `count=$((count+1))` 循环 100 次；管道版 count 不变的原因：管道右侧在子 shell 运行；修复三选一：`done < /etc/passwd`、`done < <(cat /etc/passwd)`、`shopt -s lastpipe`。
> 5. 除零会死在 `arithmetic` 相关行并带行号；未定义变量被 `set -u` 抓时报 `unbound variable` 并点名行号；`bash -x` 输出里每条命令前有 `+`，最后一行是致命错误的原文。
> 6. 保留逻辑参考：`ls -1t "$DEST"/home-*.tar.gz | tail -n +6 | xargs -r rm -f`；`-t` 按时间排序（新的在前），`tail -n +6` 从第 6 行开始即「前 5 份之外」的旧文件，`xargs -r` 保证没有旧文件时不执行 rm。跑完后 `ls -lh` 确认只剩 5 份 + 日志。
