---
title: "第 5 章 文件与目录操作"
---

# 第 5 章 文件与目录操作

> 上一章我们认识了 Linux 的文件系统树，学会了用 `pwd` 看自己在哪、用 `cd` 走来走去、用 `ls` 张望——那都是"看"。这一章开始"动手"：建目录、建文件、拷贝、移动、删除、做链接。这套操作占了运维日常敲命令的相当大比例，也藏着最多的事故：删错一个目录、`rm -rf` 后面的变量是空的、以为清空了文件其实只改了个时间戳。学完本章，你对文件的操作会从"敢做"变成"会做，且知道哪里会咬手"。

> **本章目标**
> - 能独立完成建目录（`mkdir -p`）、建文件、拷贝、移动、删除的日常操作，并说清每个命令最常用参数的含义。
> - 能用实验说清硬链接与软链接（符号链接）的核心差别：inode、跨文件系统限制、删源文件后的表现，并用 `ls -l` 第二列解释硬链接数。
> - 能用 `find` 按名字、类型、大小、时间找出目标文件并交给 `-exec` 处理；能解释隐藏文件规则与 Linux 文件名大小写敏感带来的坑。

---

## 4.1 建目录：mkdir 和那个救命的 -p

说人话：**`mkdir`（make directory）就是建一个空目录。** 它的关键几乎全在一个参数上：`-p`。

```bash
mkdir 目录名...
```

逐参数：

| 参数 | 含义 |
| --- | --- |
| `-p` | parents，连同缺失的上级目录一起建；且当目录**已存在时不报错**（幂等，可重复执行不出错） |
| `-v` | verbose，每建一个目录都打印一行（脚本调试时有用） |
| `-m 模式` | 建目录的同时指定权限（权限模型详见第 6 章《用户、群组与权限管理》） |

来，先看不用 `-p` 会怎样：

```bash
mkdir /data/home/project/web
```

```text
mkdir: cannot create directory '/data/home/project/web': No such file or directory
```

这句话的意思是：`project` 这一层不存在，`web` 没地方挂。加上 `-p`：

```bash
mkdir -p /data/home/project/web /data/home/project/api
```

没输出——又印证了那条老规矩：**Linux 命令没消息，多半就是好消息**。一条命令把两层目录一起建好了，也可以一次列多个目标，空格分隔。

常见错误/坑：

- 忘了 `-p`，报 `No such file or directory`——新手第一大坑，见到这句先问自己"上级目录在吗"。
- 目录已存在时不带 `-p`，会报 `mkdir: cannot create directory 'api': File exists`。写脚本批量建目录务必带 `-p`，这样脚本重复执行永远是安全的。
- 路径末尾多打一个斜杠（`mkdir -p a/b/`）无妨，Linux 不在乎；但心里要清楚建的是 `a/b`，不是 `a/b/` 这个东西本身。

> **SRE 贴士**：凡是"创建目录"的自动化（部署脚本、CI 流水线、初始化容器），`mkdir -p` 是唯一正确姿势——它让脚本幂等：跑一遍和跑十遍结果相同、都不报错。运维里"可重复执行不出错"是一个很高尚的品质，`cd` + 裸 `mkdir` 的组合则是最常见的把部署脚本搞挂的写法（第二次执行红着一片报错）。

## 4.2 建空文件与改时间：touch

说人话：**`touch`（触摸）有两张脸——文件不存在就建一个空文件；文件已存在，就把它的"修改时间"更新为当前时刻。**

```bash
touch 文件名...
```

```bash
cd /data/home
touch note.txt
ls -l note.txt
```

```text
-rw-r--r-- 1 root root 0 Nov 13 09:30 note.txt
```

大小是 0——建的就是一个空文件。再摸一次已有文件：

```bash
echo "部署脚本 v1" > note.txt
ls -l note.txt
touch note.txt
ls -l note.txt
```

```text
-rw-r--r-- 1 root root 16 Nov 13 09:30 note.txt
-rw-r--r-- 1 root root 16 Nov 13 09:35 note.txt
```

注意第二次 `ls -l`：时间戳变成了 `09:35`，**但内容一个字没动**。

常见错误/坑——这是本节最值得立的一块牌子：**`touch` 不是"清空文件"，不是"覆盖写入"！** 很多人以为"我 touch 过了，文件就是干净的了"，结果脚本往"干净"文件后面追加内容，新旧数据糊成一锅粥。要清空，用 `> 文件名`（重定向语义，第 7 章细讲）或 `truncate -s 0 文件名`；`touch` 只碰时间。

`touch` 的常用参数：

| 参数 | 含义 |
| --- | --- |
| `-c` | 文件不存在时**不创建**（--no-create） |
| `-t 时间戳` | 把时间设为指定值，格式 `[[CC]YY]MMDDhhmm[.ss]`，例如 `touch -t 202601011200 old.txt` 设为 2026-01-01 12:00 |

`-t` 在测试里好用：想验证"`find` 按时间查找"或"`make` 按时间戳判断是否重新编译"时，`touch -t` 让你手工捏出一个"很久没动过的文件"，不用真的等半年。

## 4.3 拷贝：cp

说人话：**`cp`（copy）把文件或目录复制一份，源不动、目标多出一份。** 目录必须加 `-r`，这是铁律；其余参数全是"拷贝时要不要连属性一起带走"的变体。

```bash
cp [选项] 源... 目标目录/目标文件
```

注意语法形状：源可以有多个（空格分隔），目标可以是一个目录（所有源拷进去）或一个文件（只能有一个源）。

逐参数：

| 参数 | 含义 |
| --- | --- |
| `-r` / `-R` | recursive，递归拷贝整个目录树。**拷目录忘加它，一定报错** |
| `-i` | interactive，目标已存在时先问一声 `overwrite?` 再动手 |
| `-p` | preserve，保留权限、属主、时间戳等属性（root 拷贝时属主也能带走） |
| `-a` | archive，归档模式，等于 `-dR --preserve=all`：递归 + 保留一切属性 + 符号链接按原样拷（不解引用）。**做备份和迁移，无脑选 `-a`** |
| `-u` | update，只在"源比目标新"或"目标不存在"时才拷，跳过无需更新的文件 |
| `-v` | 拷贝时打印每个动作，进度心里有数 |

真实走一遍：

```bash
cd /data/home
mkdir -p dst
echo "hello sre" > a.txt
cp a.txt b.txt
cp a.txt dst/               # dst 是已存在的目录，a.txt 被拷进去变成 dst/a.txt
cp -r project project.bak   # 拷贝整个目录树，-r 不能省
```

```bash
cp -i a.txt b.txt
```

```text
cp: overwrite 'b.txt'? y
```

输 `y` 回车才覆盖，输 `n` 保住原文件。这是手滑防线，虽然默认不开（Debian 的 root 默认没有 `alias cp='cp -i'`，别指望它），但敲危险覆盖前自己带上。

拷目录忘加 `-r` 的报错必须认得：

```bash
cp project project.copy
```

```text
cp: -r not specified; omitting directory 'project'
```

`-a` 的价值看对比：

```bash
cp project/a.txt /tmp/a.plain.txt
cp -a project/a.txt /tmp/a.keep.txt
ls -l --time-style=full-iso /data/home/project/a.txt /tmp/a.plain.txt /tmp/a.keep.txt
```

```text
-rw-r--r-- 1 root root 21 2026-11-13 09:35:12.000000000 +0800 /data/home/project/a.txt
-rw-r--r-- 1 root root 21 2026-11-13 09:41:07.000000000 +0800 /tmp/a.plain.txt
-rw-r--r-- 1 root root 21 2026-11-13 09:35:12.000000000 +0800 /tmp/a.keep.txt
```

普通 `cp` 出来的文件时间戳是"刚才"；`-a` 出来的跟源一模一样。备份要的就是这个——**恢复出来的文件应该和备份时刻一模一样**，时间戳变了，"哪些文件需要增量更新"的判断就全乱了（第 16 章《防火墙与数据备份》讲 rsync 时还会回来算这笔账）。

常见错误/坑：

- `cp a.txt project` 当 `project` 是**已存在的目录**，是把 `a.txt` 拷进目录变成 `project/a.txt`；若 `project` 不存在，则直接生成一个叫 `project` 的文件副本——一字之差，两种结果。拷完 `ls -l` 确认一眼是个好习惯。
- `cp -p` 只有 root 才能把属主也带走，普通用户拷贝后属主是自己（不是 bug，是安全设计，第 6 章讲 chown 时细说）。
- 往正在被服务写的文件上 `cp` 覆盖（比如覆盖一个正在运行的网站配置），可能得到一个半新半旧的混合体。生产上这类"热覆盖"要走"写临时文件 + mv 原子替换"的路数——`mv` 在同一文件系统内是改个名字，瞬间完成，读者要么看到旧的、要么看到新的，没有中间态。这是本章 4.4 与本书后面反复用到的工程手法。

> **SRE 贴士**：真正的备份场景里，`cp -a` 很快会让位给 `rsync -a`——每次全量拷贝太慢，rsync 只传差异、断点可续、可 Dry Run（先看不删）。`cp -a` 的战场是"一次性的整目录搬迁/归档"，两者的取舍详见第 16 章《防火墙与数据备份》。但请务必先会用 `cp -a`，因为 rsync 出问题时（参数打歪、选项忘带），最后救场的往往就是一条朴素的 `cp -a`。

## 4.4 移动与重命名：mv

说人话：**`mv`（move）把文件/目录换个位置或换个名字。** 同一台机器的同一个文件系统内（比如都放在 `/data` 持久盘内），它几乎瞬间完成——内核只是把"文件名 → inode"的目录条目改了改，数据一动没动；跨文件系统（比如从系统盘 `/etc` 挪到持久盘 `/data`）时，它退化为"拷贝 + 删源"，慢且中途不可打断。

```bash
mv [选项] 源... 目标目录/新名字
```

逐参数：

| 参数 | 含义 |
| --- | --- |
| `-i` | 目标已存在先问 `overwrite?` |
| `-f` | force，不询问直接覆盖（`-i` 的相反数） |
| `-n` | no-clobber，绝不覆盖已存在的目标 |
| `-v` | 打印过程 |

日常用法：

```bash
cd /data/home
mv a.txt renamed.txt              # 原地改名，内容不变
mv renamed.txt project/           # 移进目录
mv project/renamed.txt /data/home/demo-moved.txt   # 边移边改名
```

三条都不吭声，静悄悄完成。危险在于最后那个形态：目标位置已有同名文件时，**`mv` 默认直接覆盖，不问**。防它两条路：自己带 `-i`，或者养成"目标存在就先 ls"的习惯。

> **警告**：`mv` 覆盖没有回收站、没有二次确认（不带 `-i` 时）。把一周的日志 `mv` 进一个已有文件的目录、把新配置 `mv` 到 `nginx.conf` 上，都是一秒完成、无法撤销。**安全替代做法**：跨目录移动先 `ls 目标路径/同名文件` 确认不存在，或一律带 `-n`（不覆盖）；批量操作前 `mv -vn ...` 干跑一遍看它想干什么，确认无误再去掉 `-n`。

## 4.5 删除：rm 与不可逆的后果

说人话：**`rm`（remove）删文件、删目录（配 `-r`）。** 它是本章风险最高的命令，没有之一——Linux 没有回收站、没有"撤销删除"、`rm` 掉的瞬间文件就真的没了。

```bash
rm [选项] 文件/目录...
```

逐参数：

| 参数 | 含义 |
| --- | --- |
| `-r` / `-R` | recursive，连目录带内容一起删。**删目录必须带** |
| `-i` | interactive，删每个文件前逐个问 `remove regular file 'x'?` |
| `-f` | force，不询问、不存在的文件也不报错。`-f` 会压过 `-i` |
| `--` | 告诉 rm"参数到这儿为止"，后面全是文件名。防 `-abc.txt` 这种被当成选项 |

安全的交互式删除长这样：

```bash
cd /data/home/project
rm -ri api
```

```text
rm: remove directory 'api'? y
rm: remove regular empty file 'api/old-api.key'? y
```

一个个问、一个个答，删错一个文件都还有机会反悔（这个例子里的 `.key` 文件你多半不想删——看，交互的价值就在这）。日常手删建议默认 `rm -i`；脚本里批量删则万万不能带 `-i`（没有观众来按 y），改用下面"先 ls 再 rm"的两步法。

> **警告**：**`rm -rf` 是不可逆的终极操作：没有确认、不问类型、不返回数据。** 命令落下的那一刻，目录连同里面几年的日志、代码、备份，从此从这台机器上消失——系统盘只有 16 GiB、持久盘 20 GiB，删完了连"翻垃圾桶"的余地都没有。生产服务器上误 `rm -rf` 是教科书级事故，恢复靠备份（没备份就等于永久丢失），本书第 16 章会教你用备份兜底。三条铁律请刻进肌肉记忆：
> 1. **先 `ls`，再 `rm`。** `rm -rf api/` 之前先 `ls api/`，看清这一层下面躺的是什么；删前多花三秒看一眼，胜过事后三天重建。
> 2. **`rm -rf $变量/` 是事故之王。** 脚本里 `DIR=/data/home/old`，若因为拼写错误、上游命令失败、环境没加载，导致 `$DIR` 实际为空，`rm -rf $DIR/` 就展开成 `rm -rf /`——**从根开始删**。防护手段有三：用 `set -u` 让未定义变量直接让脚本报错退出；用 `rm -rf -- "${DIR:?DIR 未设置，拒绝执行}"` 让空变量触发报错而非静默展开；删关键路径前让脚本先打印将要执行的命令（echo 一遍再真删）。现代 GNU rm 还有 `--preserve-root` 默认拒绝直接删 `/`，但它救不了你删 `/data`、删 `/etc`。
> 3. **不要在"当前目录没确认过"的 shell 里敲 `rm -rf *`。** 通配符在按下回车前才展开，你以为在家目录，其实 cd 失败后人还在 `/`；此时一条 `rm -rf *` 就是全场屠杀。公共机器上想删"当前目录的所有文件"，先 `pwd`、再 `ls`，或者干脆写全路径。
>
> **安全替代做法**：日常练习环境可以给自己修一个"假回收站"——删除前先 `mv 目标 /data/home/trash/`，确认确实不要了再清空 trash；重要目录删除前 `tar czf` 一份归档。把"不可逆"改造成"两步走"，误操作就有回头路。

常见错误/坑：

- `rm file1 dir1`：删了文件，但 `dir1` 不带 `-r` 删不掉，报 `rm: cannot remove 'dir1': Is a directory`。
- `rm -f *.log` 遇到一个不存在的匹配时，因为 `-f` 在，不报错也不尴尬；但没有 `-f` 时 `*.log` 没匹配到东西，shell 会把字面量 `*.log` 传给 rm，报 `No such file or directory`——这是 glob 与命令的交互，第 4 章讲过通配符，这里见到了它的副作用。
- `rm -rf .*` 是另一个老陷阱：`.*` 会匹配到 `.`（当前目录）和 `..`（上级目录）！虽然现代 rm 会拒绝删 `.` 和 `..`，但这条命令的意图几乎总是错的。要删隐藏文件，用 `rm ./-name` 明确点名，或 `find . -maxdepth 1 -name '.*' -type f` 精准挑选。

## 4.6 链接：ln，硬链接 vs 软链接

说人话：**`ln`（link）给一个已存在的文件再起一个名字。** 这个"再起一个名字"有两种实现，性格完全不同：

- **硬链接（hard link）**：同一个 inode（索引节点，文件系统里记录"数据块在哪、属主是谁、大小多少"的那本账簿）的第二个目录项。两个名字平起平坐，指向同一份数据。
- **软链接（ symbolic link / soft link，也叫符号链接）**：一个独立的小文件，内容只有一行字——"我的真身是某某路径"。像 Windows 的快捷方式。

```bash
ln 目标 链接名        # 建硬链接
ln -s 目标 链接名     # 建软链接（-s = symbolic）
```

光讲不练假把式，来一次全记录实验。注意所有动作都在 `/data/home`（持久盘）内做：

```bash
mkdir -p /data/home/lab && cd /data/home/lab
echo "hello sre" > src.txt
ln src.txt hard.txt
ln -s src.txt soft.txt
ls -l
ls -li src.txt hard.txt soft.txt
```

```text
-rw-r--r-- 2 root root 11 Nov 13 09:42 hard.txt
lrwxrwxrwx 1 root root  7 Nov 13 09:42 soft.txt -> src.txt
-rw-r--r-- 1 root root 11 Nov 13 09:42 src.txt
```

```text
101 -rw-r--r-- 2 root root 11 Nov 13 09:42 hard.txt
102 lrwxrwxrwx 1 root root  7 Nov 13 09:42 soft.txt -> src.txt
101 -rw-r--r-- 1 root root 11 Nov 13 09:42 src.txt
```

逐行解读，三个细节全在台面上：

1. **inode 号**：`hard.txt` 和 `src.txt` 都是 `101`——同一本账，两个名字。`soft.txt` 是 `102`，它是独立的另一个文件。
2. **`ls -l` 的第二列（链接数）**：`hard.txt` 和 `src.txt` 都是 `2`——两个名字指向同一份数据；`soft.txt` 是 `1`，软链接自己不构成对源 inode 的引用。这一列就是"硬链接数"，日常判断"这文件还有没有别的名字"就看它。
3. **软链接长什么样**：文件类型位是 `l`（link），权限永远是 `rwxrwxrwx`（那只是摆设，真实权限看真身），名字后带箭头 `-> src.txt`，大小 7 字节——正好是字符串 `src.txt` 的长度。

现在做那个决定性实验——**把源文件删掉，看两位选手的表现**：

```bash
rm src.txt
cat hard.txt
```

```text
hello sre
```

硬链接波澜不惊：内容还在。为什么？`rm` 做的是"摘掉一个目录项"；这个 inode 还有 `hard.txt` 一个名字挂着，内核判断"仍有人引用"，数据一块不动。直到最后一个名字被摘掉、链接数归零，数据块才真正释放。

再看软链接：

```bash
cat soft.txt
ls -l soft.txt
```

```text
cat: soft.txt: No such file or directory
lrwxrwxrwx 1 root root 7 Nov 13 09:42 soft.txt -> src.txt
```

真身没了，软链接还在，但它指向的路径已经不存在——这叫**悬空链接（dangling symlink）**，`cat` 直接报 `No such file or directory`。`ls -l` 里那个箭头依然优雅地指向 `src.txt`，像一块风化的墓碑。

两种链接的性格对比表，建议截图收藏：

| 维度 | 硬链接 | 软链接（符号链接） |
| --- | --- | --- |
| inode | 与源**相同** | 独立的新 inode |
| `ls -l` 第二列链接数 | 源计数 +1 | 不变 |
| 跨文件系统 | **不可以**（报 `Invalid cross-device link`） | 可以 |
| 链接目录 | **不可以** | 可以 |
| 删掉源文件后 | 数据无恙，换个名字活着 | 悬空，访问即 `No such file or directory` |
| 占用空间 | 不占（就是同一份数据） | 占一点点（路径字符串的 inode） |
| 典型用途 | 同一文件多个规范入口（日志归档、异地双挂） | 路径兼容、版本切换（`current` → `v2.3`）、跨盘引用 |

跨文件系统这条限制，在咱们的云电脑上是**天天能撞见的活例子**：`/data` 是 20 GiB 持久盘，系统盘 16 GiB，它俩是两个**不同的文件系统（file system）**。试试把 `/data` 里的文件硬链到 `/root`：

```bash
ln /data/home/lab/hard.txt /root/hard.txt
```

```text
ln: failed to create hard link '/root/hard.txt': Invalid cross-device link
```

`Invalid cross-device link` 翻译：**跨设备，不行。** 原因很实在：硬链接共享的是"inode 号"，而 inode 号只在同一个文件系统内唯一——A 文件系统的 101 号账和 B 文件系统的 101 号账是两本完全不同的账，指过去就指错了。软链接没有这个病，它存的是"字符串路径"，指哪都行（前提是目标路径真实存在、且访问者有一路上各级目录的权限，第 6 章会讲为什么"一路上的权限"是个关键短语）。

目录的规矩两条：**目录不能被硬链接**（对目录执行 `ln` 会得到 `ln: 目录名: hard link not allowed for directory`——文件系统要是敢给目录开硬链，`..` 的环能把 fsck 逼疯）；**目录可以被软链接**，而且这是全 Linux 最常用的手法之一。看一个现成的：

```bash
ls -l /bin
```

```text
lrwxrwxrwx 1 root root 7 Nov 13 08:00 /bin -> usr/bin
```

第 4 章说过 usrmerge——`/bin` 早已不是真目录，它是指向 `usr/bin` 的软链接。软链接让"路径永远有效、底下实际版本随便换"成为可能：今天真身在 `v1/`，明天换成 `v2/`，所有引用 `current` 这个名字的脚本一行不改。

> **SRE 贴士**：硬链接最经典的运维用法是"**增量快照式备份**"：比如今晚要保存日志目录的全貌——先 `cp -a` 把目录结构复制一份（目录本身不能硬链接，能硬链接的是里面的文件），再把新目录里每个文件 `ln` 回对应的源文件，同一份数据不占额外空间；明天再做一轮，删掉的只是上一轮快照里"多出来的名字"。`rsnapshot` 这类老牌备份工具的核心正是这套硬链接把戏，它的现代替代品是文件系统快照和 `rsync --link-dest`。另一个反直觉的坑要记住：**很多编辑器（vim 默认行为）保存文件时是"写新文件 + 改名替换 inode"，硬链接会因此在保存后断掉一条**。你辛苦维护的两个名字，被一次 `:wq` 拆散——维护硬链接请用 `echo x >> hard.txt`（真追加）而不是编辑器整体另存。

实验做完了，把刚删掉的 `src.txt` 恢复回来，后面几节还要用它当素材：

```bash
echo "hello sre" > src.txt
ls -l
```

```text
-rw-r--r-- 1 root root 11 Nov 13 10:02 src.txt
```

注意它的链接数从 2 变回了 1——刚才 `hard.txt` 还和它共享 inode，现在源文件是新建的，账本重新开。

## 4.7 file：Linux 不靠扩展名，靠魔法字节

说人话：**`file` 命令看一眼文件的开头几个字节，告诉你是谁。** 为什么需要它？因为 Linux 世界"扩展名"只是个装饰——`report.jpg` 完全可以是一个 shell 脚本，`.log` 里可能塞着 gzip 压缩包。真正标识身份的是文件头几个字节的固定签名，术语叫**魔法字节（magic bytes）**。

```bash
file [选项] 文件...
```

常用参数：`-b`（brief，只输出类型一句话，不带文件名，适合脚本）；`-i`（输出 MIME 类型，如 `text/plain; charset=us-ascii`）。

```bash
cd /data/home/lab
file src.txt hard.txt soft.txt /bin/bash /data/home/lab
```

```text
src.txt:                 ASCII text
hard.txt:                 ASCII text
soft.txt:                 symbolic link to src.txt
/bin/bash:                ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/ld-linux-x86-64.so.2, BuildID[sha1]=a1b2c3d4e5f6..., for GNU/Linux 3.2.0, stripped
/data/home/lab:           directory
```

解读：`ASCII text` 是纯英文文本；`ELF ... executable` 是编译出来的二进制程序；`symbolic link to src.txt` 它一眼看穿软链接并报出目标；`directory` 就是目录。如果文件是空文件或数据类型不明显，可能报 `empty` 或 `data`——都是真实答案，`file` 不装。

常见错误/坑：`file` 只读开头一小段（默认 buffersize 内），一个 PNG 图片尾部藏恶意代码它不管；`file` 判断 `.jpg` 里的 gzip 流也可能只报第一种匹配。安全场景的完整检测（病毒扫描、深度解析）是另一门专业，`file` 是廉价的第一道筛查。

> **生产 vs 实验室**：你以后做的 Web 服务如果允许用户上传头像，**永远不要只校验扩展名**——攻击者把 webshell 命名成 `avatar.jpg` 就绕过前端检查了，服务端必须看魔法字节（或用现成的 MIME 嗅探库）。反过来，在实验室里这一眼能帮你认出"这到底是脚本还是二进制"，`file` 一分钟省掉一次 `vim` 打开乱码的尴尬。这个"别信表面标识、看内容签名"的思路，在日志、附件、镜像、容器层各层都会复现。

## 4.8 查看内容：cat / head / tail / less / tac / nl

文件建好了，总得看看里面。这一族命令按"取多少"分工。

### cat：一口吐完

说人话：`cat`（concatenate，本意"串接"）把一个或多个文件从头到尾倒出来。

```bash
cat [选项] 文件...
```

- `-n`：给每行编号输出。
- `-A`：显示所有不可见字符——行尾 `$`、制表符 `^I`。排查"为什么这行匹配不上"（行尾多了回车/空格）时好用到哭。

```bash
cat -n note.txt
```

```text
     1  部署脚本 v1
     2  第二行
```

常见错误/坑：**别对大文件 `cat`**。一个 2 GiB 的日志 `cat` 下去，终端刷屏几万行、你按 `Ctrl-C` 还可能把终端字符编码搞乱（恢复用 `reset` 命令，附录 B 有快捷键表）。大文件该用 `head`/`tail`/`less`。

### head / tail：只要头尾

```bash
head [-n 行数] 文件
tail [-n 行数] 文件
```

不给 `-n` 默认都是 10 行。`-n 20`、`-n -20` 的含义分别是"取前 20 行"和"**除了最后 20 行都取**"（tail 的负数是排除尾部，head 的 `-n -20` 是排除头部——同一符号在两个命令里意思不同，别记混）。

```bash
head -3 /etc/passwd
tail -3 /etc/passwd
```

```text
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
```

```text
_apt:x:100:65534::/nonexistent:/usr/sbin/nologin
systemd-network:x:998:998:systemd Network Management:/:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
```

### tail -f 与 -F：日志跟踪的两兄弟

这是全书最值得记住的一组命令之一，SRE 值班看日志的基本功：

| 参数 | 全称 | 行为 |
| --- | --- | --- |
| `-f` | --follow[=descriptor] | 一直跟着文件**追加**的内容输出，文件在原地长大它就一直吐。按 `Ctrl-C` 结束 |
| `-F` | --follow=name --retry | 跟着**文件名**：文件被轮转（改名/删除）后，它会重新打开同名新文件继续跟。**生产看日志优先用它** |

差别什么时候见真章？Linux 服务都有日志轮转（logrotate，第 15 章《定时任务与系统日志》讲）：半夜 `app.log` 被改名为 `app.log.1`、新的 `app.log` 重新开张。你用 `-f` 跟的是老描述符，日志从此消失；用 `-F` 跟的是名字，自动切到新文件，跟踪不断。第 15 章之后你会经常 `tail -F` 加 `grep` 一起用。

```bash
tail -n 3 /data/home/note.txt
```

```text
部署脚本 v1
第二行
第三行，到此结束
```

### less：分页浏览，Linux 的"阅读器"

说人话：`less`（"less is more than more"）让长文件一页一页看，前后都能翻，还能搜索。

```bash
less /etc/passwd
```

进去之后：空格/`f` 下一页，`b` 上一页，`/关键字` 回车向下搜索（`n` 下一个命中，`N` 上一个），`G` 跳到文件尾，`1G` 跳到文件头，`q` 退出。它不会把全文灌进终端，多大的文件都是秒开——**看大文件、翻手册（`man` 底层就是它），用 `less`**。完整快捷键表见附录 B《常用快捷键与 vim 技巧》。老的 `more` 只能往下翻、不能往上翻，还会自动一行行滚，了解一下即可，日常用 `less` 代替。

### tac 与 nl：两个小工具

`tac`（cat 倒着拼）把文件**从最后一行倒着输出**——看日志时"最近的错误在最前面"比正着翻到结尾快得多：

```bash
tac /data/home/note.txt
```

```text
第三行，到此结束
第二行
部署脚本 v1
```

`nl`（number lines）输出时加行号，和 `cat -n` 像但不同：`nl` 默认**空行不编号**，`cat -n` 所有行都编。给文档加行号、给代码做引用（"第 42 行"）时 `nl` 更清爽：

```bash
nl /data/home/note.txt
```

```text
     1  部署脚本 v1
     2  第二行

     3  第三行，到此结束
```

## 4.9 find：按条件翻地皮找文件

说人话：**`find` 在一个目录树里按你给的条件（名字、类型、大小、时间）逐个过筛，把命中的文件揪出来。** 运维找问题的标准工具："这目录里最近谁动过？""超过 30 天的临时文件清一清""全盘有多少个超过 1 GiB 的大文件？"

```bash
find [路径...] [表达式]
```

最常用的条件，逐个配上真实例子。先造一点实验素材：

```bash
mkdir -p /data/home/findtest/sub
cd /data/home/findtest
dd if=/dev/zero of=big.img bs=1M count=2 status=none
echo "近" > fresh.log
echo "老" > old.log
touch -t 202501010000 old.log
echo "目录里" > sub/keep.txt
```

按名字找：

```bash
find /data/home/findtest -name "*.log"
```

```text
/data/home/findtest/fresh.log
/data/home/findtest/old.log
```

`sub/keep.txt` 不在其中——`find` 递归下钻到了子目录，但 `*.log` 没匹配上它。现在记住 find 最大的坑：**`-name` 的模式必须加引号**。不加引号，shell 会先在当前目录展开 `*.log`再把结果传给 find，find 收到的可能是一堆具体路径而不是模式，浅目录里能碰巧对、深目录里直接失灵。加引号把模式原样交给 find，这是唯一正确姿势。顺带：`-iname` 是大小写不敏感版（本章末尾你就知道为什么需要它）。

按类型找：

```bash
find /data/home/findtest -type f -name "*.log"
find /data/home/findtest -type d
```

```text
/data/home/findtest/sub
/data/home/findtest
```

`-type f` 只要普通文件（还有 `d` 目录、`l` 符号链接等），常用来把"目录本身"和"里面的东西"分开处理——比如 5.10 节我们用 `find -type d` 和 `find -type f` 分别给目录和文件设不同权限。

按大小找：

```bash
find /data/home/findtest -type f -size +1M
```

```text
/data/home/findtest/big.img
```

`-size +1M`：大于 1 MiB（`k` 是 KiB、`M` 是 MiB、`G` 是 GiB；`-` 号是小于）。找谁吃满了磁盘第一条就敲它，精确版是 `du`（第 11 章《磁盘与存储管理》的主角）。

按修改时间找：

```bash
find /data/home/findtest -name "*.log" -mtime -1
```

```text
/data/home/findtest/fresh.log
```

`-mtime -1`：最近 1 天内修改过（`-n` 是 n 天内；`+n` 是 n 天以前，**清理老文件用它俩的分工记牢**）。还有个更细的 `-mmin -60`（最近 60 分钟）。`-newer 某文件` 更妙：找出"比这个参照文件还新"的文件，适合做变更审计——"配置文件 3 号被改过，找出那天之后被动过的一切"。

只找一层：`-maxdepth 1` 限制不下钻子目录，全盘扫描配它，否则 find 会把 `/proc`、`/sys` 这些虚拟文件系统也翻个底朝天（各种权限错误刷屏，记得 `2>/dev/null` 把噪声咽了）。

### -exec：找到之后，顺手执行

说人话：`find` 揪出来不算完，`-exec` 让每个命中项直接执行一条命令，find 一行搞定"筛选 + 处理"。

```bash
find /data/home/findtest -name "*.log" -exec ls -l {} \;
```

```text
-rw-r--r-- 1 root root 5 Jan  1  2026 /data/home/findtest/old.log
-rw-r--r-- 1 root root 5 Nov 13 09:50 /data/home/findtest/fresh.log
```

语法规矩只有一条：**结尾必须是 `\;`（反斜杠 + 分号）**，`{}` 会被替换成每个命中文件的路径。分号（`;`）是 shell 的特殊字符，不加反斜杠会被提前解释掉，这是 find 新手第二大坑。`{}` 后面跟不跟空格都能跑（GNU 风格不跟空格更稳）。

和批量执行相关的另一个工具叫 `xargs`（把标准输入转成命令行参数），配合管道用，如 `find ... | xargs rm`。它的坑比 `-exec` 多（文件名带空格会拆串、管道失败不被发现），日常首选 `-exec ... \;`；要压榨性能记住 `-exec ... {} +`（加号结尾，把一批文件名合成一条命令执行，比逐个 exec 快得多）。选型细节不展开，`man find` 随时翻。

> **SRE 贴士**：生产中 `find` + `-exec` 是定时清理脚本的顶梁柱，而合格的用法永远是**两步走**：第一步 `-exec echo {} \;`（或 `-exec ls {} \;`）看它想动谁，第二步才把 `ls` 换成 `rm`。跳脱这一步直接 `find / -name "*.log" -mtime +30 -exec rm {} \;` 的人，早晚会删到一个他不该删的地方。顺序不能反，这是"破坏性命令的执行礼仪"，配第 14 章学完 `set -euo pipefail` 之后，你会把这种"干跑 → 确认 → 真跑"写成脚本的标准结构。

## 4.10 隐藏文件与大小写敏感：两个容易忘的设定

### 隐藏文件：一个点，两种世界

Linux 没有"隐藏属性"这种东西，规矩朴素得可爱：**文件名以 `.` 开头，就是隐藏文件**。`ls` 默认不显示，`ls -a`（all）连它们带 `.`、`..` 一起亮出来。`*` 通配符不匹配以点开头的名字——这也是第 4 章说过"`.` 与 `..`"之后顺理成章的设计。

```bash
cd /data/home/lab
touch .config
ls
ls -a
```

```text
src.txt hard.txt soft.txt ...
```

```text
.  ..  .config  hard.txt  src.txt  soft.txt
```

Linux 的"配置放点文件"文化就建在这条规矩上：`~/.bashrc`、`~/.ssh/`、`~/.config/`——家目录里一排点文件，`ls` 一眼干净，`ls -a` 才见全貌。历史上隐藏文件机制本来就是个忙里偷情的意外（早期 `ls` 把 `.` 和 `..` 特殊处理，有人就把文件起名成点开头蹭"不显示"），谁知成了全行业惯例。

运维场景里最常和它打交道的地方：**部署时漏拷贝 `.env`、`.gitignore` 之类的点文件**；用 `cp -r dir/* dst/` 拷贝后点文件集体缺席，正确姿势是 `cp -a dir/. dst/`（把 `.` 作为源，连同隐藏东西一起带过去）。另一个坑前面提过：`.*` 通配会咬到 `.` 和 `..`，处理隐藏文件时用 `find -maxdepth 1 -name '.*'` 或明确点名，别裸奔 `rm .*`。

### 大小写敏感：FILE.txt 和 file.txt 是两个文件

**Linux 的文件名严格区分大小写。** `FILE.txt`、`File.txt`、`file.txt` 放在同一目录，是三个互不相干的文件：

```bash
cd /data/home/lab
touch File.txt case-demo.txt
ls
ls file.txt
```

```text
File.txt case-demo.txt hard.txt src.txt ...
```

```text
ls: cannot access 'file.txt': No such file or directory
```

我们常用的 ext4（第 11 章会正式认识它）就是一个大小写敏感的文件系统。这意味着几件事：

- `cat Config.yaml` 和 `cat config.yaml` 结果可能完全不同——**配置文件引用的大小写必须逐字母对上**。
- 从 Windows/macOS（默认大小写不敏感）迁移项目到 Linux 服务器，`include "Config.PHP"`、`require './Utils/Helper.php'` 这类代码可能全线报"文件不存在"，而代码在你自己电脑上跑得好好的。**迁移第一课：核对所有路径引用的大小写。**
- shell 命令也敏感：`CD`、`Ls` 都不存在，只有小写的 `cd`、`ls`。

> **生产 vs 实验室**：实验室里你一个人操作，大小写错了当场 `No such file or directory`，代价是尴尬一分钟。生产上这是"凌晨告警、多人甩锅"的经典联赛项目：代码本地（macOS/Windows）跑得欢，一到 Linux 容器就 `FileNotFound`，而两个文件名只差一个字母，肉眼极难分辨。**跨平台协作的规范：文件一律小写加连字符/下划线（`my-app.conf`），别跟大小写敏感性赌视力。** 顺手再说一条相关的真实约束：ext4 文件名的字节上限约 255，可以用空格、中文、几乎一切字符（除 `/` 和空字符），但运维共识是"名字里别有空格"——一个空格能让 90% 的老脚本翻车，详见的第 14 章学完 quoting 你会更深地体会为什么。

## 本章小结

本章把"对文件和目录的增删改查"整套手艺过了一遍。按使用频率给你一张收尾表：

| 命令 | 干什么 | 最常用参数/注意 |
| --- | --- | --- |
| `mkdir` | 建目录 | `-p` 建多层且幂等，脚本必备 |
| `touch` | 建空文件/改时间戳 | 不清空内容！`-t` 手工捏时间 |
| `cp` | 拷贝 | 目录带 `-r`；备份迁移用 `-a`；覆盖前想 `-i` |
| `mv` | 移动/改名 | 同文件系统内瞬间完成；覆盖无提示，`-n` 防误伤 |
| `rm` | 删除 | `-r` 删目录；不可逆，先 `ls` 再 `rm`；防 `$变量` 展开为空 |
| `ln` | 链接 | `-s` 软链（跨盘、可链目录、悬空风险）；硬链同 inode 不可跨盘 |
| `file` | 判断真实类型 | 看魔法字节，不信扩展名 |
| `cat` `head` `tail` | 看内容 | 大文件别 `cat`；`tail -F` 跟日志扛轮转 |
| `less` | 分页浏览 | `man` 的底层，大文件秒开 |
| `find` | 按条件查找 | `-name` 加引号；`-exec ... \;`；先干跑再真删 |
| 隐藏文件 | 点开头 | `ls -a` 显形；`cp -a dir/.` 连隐藏一起拷 |
| 大小写 | 严格敏感 | 跨平台迁移先核对大小写 |

生产高频中的高频：`tail -F` 看日志、`find + -exec` 做清理、`cp -a`/`mv` 做部署、`ln -s` 做版本切换。而风险序列TOP1永远是 `rm`——记住那个原则：**删除类操作，先 `ls`，再 `rm`；带变量的，先想"它会是空吗"。**

## 动手练习

以下全部可在云电脑上完成，命令默认在 `/data/home` 下操作（第 3 章的铁律：数据放持久盘）。

1. 用一条 `mkdir -p` 命令创建 `/data/home/lab/a/b/c` 三层目录，然后不带 `-p` 再建一次，把报错原文抄下来并用一句话解释它。
2. 在 `/data/home/lab` 下 `echo "hello sre" > src.txt`，分别建硬链接 `hard.txt` 和软链接 `soft.txt`。用 `ls -li` 记录三者的 inode 和链接数；然后 `rm src.txt`，分别 `cat hard.txt` 和 `cat soft.txt`，记录两次的输出并解释差异。
3. 把 `/data/home/lab` 用 `cp -a` 备份为 `/data/home/lab.bak`，用 `ls -l` 对比两个目录的权限和时间戳是否一致，并解释为什么备份要这个特性。
4. 安全删除演练：先 `ls` 再 `rm -ri` 删掉练习 3 建的 `lab.bak`。然后写出：一条带 `${DIR:?}` 防护的删除骨架命令，并解释变量为空时会发生什么。
5. 在 `/data/home/findtest` 下造几个 `.log` 文件（其中一个 `touch -t 202501010000` 捏成老文件），用 `find` 分别按名字、`-type f`、`-mtime -1`、`-size +1M` 找出目标，最后用 `-exec ls {} \;` 列出结果。解释 `-name "*.log"` 引号为什么必须加。
6. 综合排查：用 `file` 判断 `/bin/bash`、一个空文件、一个 `.log` 各是什么类型；再 `touch File.txt` 和 `file.txt` 两个文件，验证它们是两个文件并 `rm` 其中之一验证另一个无恙。最后用 `tac` 倒序查看你练习 2 里用过的某个文件。

> **参考答案**：
> 1. 第一次成功；第二次报 `mkdir: cannot create directory '/data/home/lab/a/b/c': File exists` 或对不存在的上级报 `No such file or directory`——`mkdir` 不会替你把缺失的上级目录补出来。
> 2. `hard.txt` 与 `src.txt` inode 相同、链接数都是 2（`soft.txt` inode 独立、链接数 1）。删 `src.txt` 后：`hard.txt` 内容照旧（inode 仍有名字挂着），`soft.txt` 变成悬空链接，`cat` 报 `No such file or directory`。
> 3. 权限、属主、时间戳全部一致；时间戳保留才能让"增量备份"判断文件有没有变，否则每次备份都以为全变了。
> 4. `rm -ri` 逐个确认；骨架：`DIR=/data/home/xxx; rm -rf -- "${DIR:?DIR 未设置，拒绝执行}"`——`${DIR:?}` 在变量为空/未定义时直接让 shell 报错退出，命令根本不执行。
> 5. `-name "*.log"` 必须加引号，否则 shell 先在当前目录展开 `*.log`，find 拿到的是具体路径而非模式，深层目录失灵。
> 6. `file` 对二进制报 `ELF 64-bit LSB pie executable ...`、空文件报 `empty`、`ASCII text` 等；`File.txt` 与 `file.txt` 是两个独立文件，删一个另一个不受影响。
