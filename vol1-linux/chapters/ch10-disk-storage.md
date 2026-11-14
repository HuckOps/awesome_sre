---
title: "第 10 章 磁盘与存储管理"
---

# 第 10 章 磁盘与存储管理

> 前面几章我们操作文件、进程、软件，默认了一个事实：文件总有地方放。这一章回答「地方」的问题——磁盘怎么分区、怎么挂载、怎么写文件系统、空间怎么就满了。SRE 的值班夜里，最高频的两类告警一个是「磁盘满了」一个是「服务连不上」，而前者几乎每晚都会在不同的机房上演。把空间管理这门课学扎实，你就率先拿到了半夜少接几个告警的资格证。

> **本章目标**
> - 能画出存储层次图，并用 `df -h`、`df -i`、`du` 三个命令说清「空间去哪了」（包括 inode 耗尽这类不看 `df -i` 就漏掉的案例）。
> - 能解释文件系统、inode、块设备、挂载的概念，并用 `lsblk`、`blkid`、`findmnt` 读懂一台机器的存储布局。
> - 能用「镜像文件 + loop 设备」在云电脑上无痛完成一次完整实验（建盘、格式化、挂载、写入、卸载），并会安全地修改 `/etc/fstab`、管理 swap。

<br>

## 10.1 存储的层次：金字塔与量级感

计算机的存储不是铁板一块，而是一座金字塔。越快的越贵、越小的越靠近 CPU，越慢的越便宜、越大的越靠外：

```text
        ▲ 快 / 贵 / 容量小 / 断电没
        │
   CPU 寄存器、缓存（Cache）      ns 级         KB~MB
        │
   ┌────┴────────────────┐
   │  内存（RAM）          │  百 ns 级        GB（你的云电脑 8 GiB）
   ├─────────────────────┤
   │  本地磁盘（SSD）      │  μs 级           GB~TB（系统盘 16 GiB、持久盘 20 GiB）
   ├─────────────────────┤
   │  网络存储 / 对象存储    │  ms 级           TB~PB（卷 5 讲分布式存储）
   ▼ 慢 / 便宜 / 容量大 / 断电还在
```

分层的理由就一句话：**没有一种存储同时又快又大又便宜又持久**。所以体系结构用「多级缓存」把最热的少量数据放在最快的一层，把冷数据赶到最慢最便宜的一层。对 SRE 的实际意义：排查性能要知道「这个数据在哪一层」——同样一次读，命中内存是微秒，打到磁盘是毫秒，差了三个数量级。

还需要记住**易失性（volatile）**这条线：内存是易失的，断电即失，所以进程再重要也要能把状态落盘；磁盘是持久的，但持久不等于可靠——单块盘会坏，这就是第 15 章《防火墙与数据备份》「没验证过的恢复等于没备份」的由来。swap 交换分区（10.8 节）就是这座金字塔上一个有意思的中间层：它在磁盘上，内核却把它当内存用——慢，但能把「内存放不下的东西」兜住，让系统不直接 OOM 杀进程。

<br>

## 10.2 看全局：df -h 与 df -i

### 10.2.1 df：每个文件系统还剩多少空间

**先说人话**：`df`（disk free）列出所有已挂载的文件系统及各自的空间用量。SRE 的第一反应命令，没有例外。

```bash
df -h
```

```text
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda1        16G  6.2G  9.1G  41% /
tmpfs           3.9G     0  3.9G   0% /dev/shm
/dev/vdb         20G  2.4G   16G  13% /data
```

逐列解读：

| 列名 | 含义 |
| --- | --- |
| Filesystem | 哪个「文件系统」，物理盘的设备名（`/dev/vda*`、`/dev/vdb*`）或虚拟文件系统（`tmpfs` 是内存盘，`proc`、`sysfs` 之类是内核伪文件系统，吃内存不吃磁盘）。 |
| Size / Used / Avail | 总容量、已用、**可用**。注意 Avail 比「Size 减 Used」略小，一部分空间被 ext4 保留（默认 5%，只给 root 用），是为了「快满了还能让 root 进去救场」。 |
| Use% | 已用百分比，超过 80% 就可以开始盯，超过 90% 该处理，100% 时普通用户已写不进去。 |
| Mounted on | 挂载点，这个文件系统「接」在目录树的哪个位置。磁盘分区本身对用户不可见，用户看到的是挂载点。 |

两条补充习惯：看单个分区用 `df -h /data`（目录落在哪个分区就报哪个）；看量不看单位可以 `df -h` 永远够用，`-i` 是下一小节的真神技。

### 10.2.2 df -i：空间没满，为什么写不进去了

这是本章最值钱的一个案例。某天应用突然报 `No space left on device`，你冲过去 `df -h`：

```text
Filesystem      Size  Used Avail Use% Mounted on
/dev/vdb         20G  4.1G   15G  21% /data
```

空间明明还剩 15G，凭什么写不进去？答案是：**inode 耗尽了**。

```bash
df -i /data
```

```text
Filesystem      Inodes  IUsed  IFree IUse% Mounted on
/dev/vdb      1310720 1310720      0  100% /data
```

IUse% 100%，一个 inode 都不剩了。什么是 inode？10.4 细讲，这里先记住它的角色：**文件系统里每个文件、每个目录都独占一个 inode 当「身份证」**。磁盘是按「块」存数据的，但「有哪些文件」是按 inode 记账的。空间和数据块是两套账：块账（`df -h`）没满，inode 账满了，一样写不了新文件。

真实世界里 inode 是怎么被挥霍掉的：把几十万、上百万个小文件往一个目录倒——应用缓存拆分文件、邮件队列（maildir 格式就是一封邮件一个文件）、解压素材包、甚至某次事故后 `find` 把垃圾文件铺满磁盘。文件都很小（几百字节），数据块占用寥寥，但 inode 一个一个实打实地用光。**排查习惯就此养成：报「空间满」先 `df -h`，`df -h` 不紧张就立刻 `df -i`，两步缺一不可。** 治理手段：删掉海量小文件（用 `find ... -delete`，10.9 的排障流程会演示）、或者重新格式化时指定更多 inode（`mkfs.ext4 -N`，重新格式化的代价本章不展开，生产上先想备份——详见第 15 章）。

<br>

## 10.3 找大目录：du

`df` 告诉你哪个**分区**满了，`du`（disk usage）告诉你空间被哪个**目录**吃掉了。两者配合就是完整的定位链条。

```bash
du -sh /var/log        # s=汇总（summary），只报一个总数；h=人类可读单位
du -h --max-depth=1 /var | sort -rh | head
```

第一条是「/var/log 一共多大」，第二条是排障主力：列出 `/var` 下每个一级子目录的大小，按大小排序，取前几名。逐参数：

- `-s`（summarize）：不展开子目录，每个参数只输出一行总计。
- `-h`（human-readable）：自动选 K/M/G 单位。
- `--max-depth=1`：只往下探一层。`du` 默认会把整棵树递归列出来，几千行根本没法读，`--max-depth` 是让它「只说重点」的开关。
- `sort -rh`：按第一列排序，`-r` 倒序（大的在前），`-h` 认识 1K/2M/3G 这样的单位（普通 `sort -n` 会把 900M 排在 1G 前面，闹笑话）。
- `head`：只留前 10 行。

代表性输出：

```text
du -sh /var/log
```

```text
8.3G	/var/log
```

```bash
du -h --max-depth=1 /var/log | sort -rh | head
```

```text
8.1G	/var/log/nginx
4.2G	/var/log/nginx
112M	/var/log/apt
48M	/var/log/journal
```

顺着这个输出往下钻：`/var/log/nginx` 大 → `du -h --max-depth=1 /var/log/nginx | sort -rh` → 大概率是 `access.log` 和 `error.log`。定位完成，剩下的处理在第 14 章《定时任务与系统日志》用 logrotate 从根上治，本章 10.9 会串一遍完整思路。

几个易错点：`du` 默认统计**磁盘占用**，和 `du --apparent-size`（文件逻辑大小）在稀疏文件上差别巨大——10.7 节用 `truncate` 造出来的那个「逻辑 100M、实际不占块」的稀疏镜像就是例子，两条命令结论完全不同；不带路径参数时 `du` 统计当前目录，要控制统计范围就善用 `--max-depth`；权限不足的目录 `du` 会报 Permission denied，排查「du 加起来比 df 小」时往这个方向想。

<br>

## 10.4 文件系统与 inode：数据的账本

### 10.4.1 为什么需要文件系统

一块裸盘就是一排连续的「块（block）」，每块 4 KiB（ext4 默认），从 0 编号到几十亿。直接往里写字节当然能存数据，但没法回答「file1.txt 在哪几个块、属于谁、什么时候改的、允不允许你读」。**文件系统（File System）就是盘上的一套账本和管理规则**：哪个块被谁占了、哪个块还空着、文件的元信息记在哪、坏了怎么恢复。

Debian 13 的默认文件系统是 **ext4（第四代扩展文件系统）**，它把磁盘切成几个层次：

```text
 ┌───────────────────────────────────────────┐
 │ 超级块（superblock）                         │  → 全盘总账：块大小、总块数、
 │  文件系统整体信息，通常有多份副本             │    inode 总数、挂载次数…
 ├───────────────────────────────────────────┤
 │ 块组 0（block group）  块组 1   块组 2  …     │  → 每组分到一批数据块 + 一批 inode
 │ ┌───────────┬───────────┬───────────┐      │
 │ │组内超级块 │ inode 位图 │ 块位图    │      │  → 位图 = 哪些 inode/块被占用
 │ │副本       │ inode 表  │ 数据块…  │      │
 │ └───────────┴───────────┴───────────┘      │
 └───────────────────────────────────────────┘
```

- **超级块（superblock）**：文件系统的「户口本」。
- **块组（block group）**：把大盘切成若干组，每组管理自己的 inode 和数据块，减少寻道、提高可靠性（超级块在每个组都有备份，坏了能救）。
- **inode 表**：本组的 inode 都住在这。**inode 不存文件名，也不存数据**，只存元数据：文件类型、权限位、属主、时间戳、硬链接数、以及「数据块分布图」（块指针）。
- **位图（bitmap）**：inode 位图记「哪个 inode 空了」，块位图记「哪个块空了」。新建文件的动作就是：inode 位图找一个空位、块位图找若干空位、写数据、把 inode 和文件名关联。

**日志（journal）** 是 ext4 区别于 ext2 的关键：写操作先写日志、再落账，中途断电重启后按日志重做或回滚，大幅降低「崩溃后文件系统烂掉」的概率。代价是少量性能损耗，换来的是 ext4 成为二十年来最稳的默认选择。

### 10.4.2 inode 与硬链接：一个实验讲透

**inode（index node，索引节点）是文件的真正身份证，文件名只是挂在 inode 上的门牌号。** 常规认知里「文件名 → 文件」其实是两条路：目录项里存着「文件名 → inode 号」的映射，inode 里存着「inode 号 → 数据块」的映射。

第 4 章《文件与目录操作》我们做过 `ln` 的实验，那时留了个问题：硬链接和原文件到底什么关系？现在能回答了——**两个文件名指向同一个 inode。** 实验：

```bash
cd /data/home
echo "sre" > a.txt
ln a.txt b.txt          # 硬链接
ls -li a.txt b.txt
```

```text
131073 -rw-r--r-- 2 root root 4 11-13 09:20 a.txt
131073 -rw-r--r-- 2 root root 4 11-13 09:20 b.txt
```

盯住第一列 inode 号：**两个文件同一个号，这就是同一份数据。** 权限、大小、时间戳也完全一致。第三列那个数字「2」是**硬链接数（link count）**——记在 inode 上、被所有指向它的文件名共享的计数。它正好印证了：有两个文件名指向这个 inode。想要更完整的现场，用 stat：

```bash
stat a.txt
```

```text
  File: a.txt
  Size: 4               Blocks: 8          IO Block: 4096   regular file
Device: 259,2h/0d       Inode: 131073      Links: 2
Access (UAC1): 2026-11-13 09:20:41.123456789 +0800
Modify: 2026-11-13 09:20:39.123456789 +0800
Change: 2026-11-13 09:20:39.123456789 +0800
 Birth: 2026-11-13 09:20:39.123456789 +0800
```

`Links: 2` 一览无余。把 b.txt 删掉再 stat：

```bash
rm b.txt && stat a.txt | grep Links
```

```text
Device: 259,2h/0d       Inode: 131073      Links: 1
```

链接数回落。**规律的完整表述是：删除一个文件名只是把 inode 的硬链接数减一，链接数归零，inode 才被释放，数据块才真正可回收。** 这解释了几件事：为什么「删了大文件空间没回来」（有进程还开着这个文件，inode 没释放，处理方式见 10.5.2 的 fuser/lsof）；为什么 `rm` 飞快（撕目录项而已）；为什么 inode 数和文件数直接挂钩（10.2.2 的 inode 耗尽）。

对比一下软链接：`ln -s` 造出来的符号链接（symlink）inode 号不同，它只是一个「指向路径字符串」的小文件，自己另占一个 inode——这也是为什么 `ln -s` 造的链接断掉后一片飘红，而硬链接源文件删除后内容照旧（第 4 章有完整对比，忘了就回去复习）。

> **SRE 贴士**：判断「这两个路径是不是同一份数据」，不要 `diff`（内容相同 ≠ 同一文件），直接比 inode：`ls -li` 或 `stat` 看 Inode 号。排查「日志双写到崩溃」「配置文件被偷偷换成了链接」这类灵异事件，inode 一照便知。

<br>

## 10.5 块设备与挂载：磁盘怎么接进目录树

### 10.5.1 lsblk、blkid、findmnt 三兄弟

Linux 里「盘」的学名是**块设备（block device）**——以块为单位随机读写的硬件（或伪装成硬件的文件）。看设备的家谱：

```bash
lsblk
```

```text
NAME     MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
vda      254:0    0   16G  0 disk
├─vda1   254:1    0   16G  0 part /
└─vda128 254:128  0    4M  0 part /boot/efi
vdb      254:16   0   20G  0 disk
└─vdb1   254:17   0   20G  0 part /data
```

树状结构一眼识骨：`vda` 整盘 16G，下面切了两个分区 `vda1`（挂 `/`）和 `vda128`（EFI 引导分区）；`vdb` 整盘 20G，一个分区 `vdb1` 挂 `/data`。这正好对上第 1 章的事实：**系统盘 16 GiB + 持久盘 20 GiB**（挂载点、分区数以你的机器为准，云电脑各家略有差别）。`RM` 是可移动盘标记（1 = U 盘之类），`RO` 是只读标记，`SIZE` 已经人类可读。

想拿到更稳的身份标识——UUID，用 blkid：

```bash
blkid
```

```text
/dev/vda1: UUID="3f8a1c2e-4b7d-4e9a-9c1f-2a6b8d0e5f11" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d"
/dev/vdb1: UUID="7c9e2d4f-1a3b-4c5d-8e9f-0a1b2c3d4e5f" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="b2c3d4e5-f6a7-4b8c-9d0e-1f2a3b4c5d6e"
```

**UUID（Universally Unique Identifier）是文件系统的身份证号**，和设备名 `/dev/vda1` 的区别在于：设备名会随硬件顺序变（多加一块盘就可能改名），UUID 是格式化时生成、写死在文件系统里的，永久不变。fstab、GRUB、生产运维全都优先认 UUID——认名字是裸设备思维，认 UUID 是可靠运维思维。

看「当前挂了什么、用什么参数挂的」：

```bash
findmnt /data
```

```text
TARGET SOURCE     FSTYPE OPTIONS
/data  /dev/vdb1  ext4   rw,relatime,errors=remount-ro
```

三列分别是挂载点、来源设备、文件类型和挂载选项。`rw` 可读写、`relatime` 是 ext4 默认的「懒更新访问时间」（性能优化，文件被读时不每次都写时间戳）。不带参数的 `findmnt` 输出整棵挂载树，是排「这玩意儿到底挂哪了」的第一现场。

### 10.5.2 mount 与 umount：把设备接到目录上

**挂载（mount）就是把某个文件系统的根，接到现有目录树的某个目录（挂载点，mountpoint）上。** 挂载点必须是一个已存在的空目录，挂载后原目录里的内容被新文件系统暂时「盖住」（卸载后原内容重现）。

Linux 下 mount 的现代用法是「一条龙 + 自动识别文件系统类型」：

```bash
mount /dev/vdb1 /mnt            # 挂块设备到目录（类型可省略，mount 会自动探测）
mount -t ext4 /dev/vdb1 /mnt    # 老写法：显式指定文件系统类型
mount /dev/sr0 /mnt             # 挂光盘这类只读介质的经典姿势
mount -o remount,ro /mnt        # 不卸载，直接把已挂载的文件系统切成只读
```

umount 相反，把文件系统从目录上「摘」下来：

```bash
umount /mnt                     # 按挂载点卸载
umount /dev/vdb1                # 按设备名卸载，两种写法等价
```

**「device is busy」是新手遇到的第一堵墙**：

```text
umount /mnt
umount: /mnt: target is busy.
```

这句话的意思不是「以后也别烦我」，而是「**有进程正在用这个文件系统里的东西，卸了它就要遭殃**」。典型肇事的三种：你的 shell 当前目录就在里面（最常见）；有进程打开了里面的文件；或者它是某个服务的 WorkingDirectory。

两步排查法，先猜后证：

```bash
cd /                          # 第一步：先撤出来，九成问题到这解决
fuser -vm /mnt                # 第二步：还 busy 就列出谁在用
lsof +D /mnt                  # 或者用 lsof 按目录递归列打开的文件
```

```text
                     USER        PID ACCESS COMMAND
/mnt:                root        942 ..c..   sshd
                     root       1142 f..e..   bash
```

（ACCESS 列 `c` = 当前目录、`f` = 打开的文件、`e` = 可执行文件正在跑。PID 和 COMMAND 直接点名。）处理顺序永远是温柔的：先 cd 出来、再礼貌杀进程（`fuser -k -TERM /mnt`——「先 TERM 后 KILL」的信号思路详见第 8 章《进程管理与系统监控》）、实在不行才考虑 umount -l（懒卸载，让内核自己择机收尾）或 -f（强制， nfs 场景用，本地盘强制卸载有风险）。**umount -f / umount -l 是特种工具，不是日常手段，生产上用之前想清楚代价。**

> **警告**：在分区还有进程写入时强行 umount，等于把「半写状态」的文件系统整个拍在盘上——数据损坏、文件系统变只读（`errors=remount-ro` 触发）、下次挂载要 fsck 开盲盒。安全替代做法：一、umount 前先 `findmnt` 确认目标；二、有服务在用就先停下服务再卸载（第 12 章会讲 `systemctl stop`）；三、报 busy 走「cd 出去 → fuser/lsof 点名 → 礼貌杀进程」的固定流程，不要上来就 `-f`/`-l`；四、生产上卸载任何挂载点都属于变更操作，不要以 root 直接执行，执行前确认没有该盘的备份任务在跑（详见第 15 章《防火墙与数据备份》）。

<br>

## 10.6 /etc/fstab：开机自动挂载的账本

`mount` 是当场动作，重启就没了。要让分区每次开机自动挂，登记在 `/etc/fstab`（file system table）：

```bash
cat /etc/fstab
```

```text
# /etc/fstab: static file system information.
#
# <file system>                           <mount point>  <type>  <options>          <dump>  <pass>
UUID=3f8a1c2e-4b7d-4e9a-9c1f-2a6b8d0e5f11  /              ext4    errors=remount-ro  0       1
UUID=7c9e2d4f-1a3b-4c5d-8e9f-0a1b2c3d4e5f  /data          ext4    defaults           0       2
```

**六个字段逐列背下来**，这是运维面试的送分题：

| 字段 | 位置 | 含义 |
| --- | --- | --- |
| 1. 设备 | `<file system>` | 认 UUID（`UUID=...`）是标准姿势。`blkid` 查得到。也可以写 `/dev/vdb1` 或 `LABEL=xxx`，但设备名会变，不推荐。 |
| 2. 挂载点 | `<mount point>` | 接到哪个目录。swap 这个位置固定写 `none`。 |
| 3. 类型 | `<type>` | 文件系统类型：`ext4`、`xfs`、`swap`、`nfs`…… |
| 4. 选项 | `<options>` | 挂载参数，逗号分隔。`defaults` 是一组默认（rw、suid、dev、exec、auto、nouser、async）；`noatime` 关掉访问时间更新，日志盘/数据库盘常用，能减少无谓写入；`ro` 只读；`nofail` 设备不存在也继续启动；`errors=remount-ro` 出错自动转只读自保（Debian 根分区的默认值）。 |
| 5. dump | `<dump>` | 给老牌 dump 备份工具用的位，0/1。今天基本被备份软件取代，**无脑写 0**。 |
| 6. pass | `<pass>` | 开机 fsck 自检的顺序：1 优先（根分区）、2 其次（其他 ext 分区）、0 跳过（swap、proc、sysfs、tmpfs、网络盘）。多个同是 2 的分区会被并行检查，缩短开机时间。 |

mount 有一条对 fstab 专用的读取姿势：**挂载点已存在、fstab 里登记过的，可以只写挂载点**：

```bash
mount /data          # 等价于把 fstab 里 /data 那一行的参数全部展开执行
```

改完 fstab 的正确验证方式，是本能的 `mount -a`：它按 fstab 把「该挂没挂的」全部挂一遍，参数写错当场报错，而**不用重启验证**。

> **警告**：改 `/etc/fstab` 是全服务器最危险的姿势之一——一个手滑的拼写、一个不存在的 UUID，轻则某个分区挂不上，重则开机停在 emergency mode，SSH 进不去，而你多半在远程。三条军规：一、改之前 `cp /etc/fstab /etc/fstab.bak`，真出事还能进救援模式改回来；二、改完立刻 `mount -a` 当场验证，别等重启；三、**永远不要以「重启看看行不行」当测试手段**，出问题你就被关在门外了。生产上优先用 UUID 不用设备名，新盘接入后先 `blkid` 确认 UUID 再写 fstab。生产环境不要直接以 root 改这些文件，走有审计、有回滚预案的变更流程。

只读挂载（read-only mount）是 fstab 里被低估的好朋友。典型场景：给一台机器挂日志盘、审计盘、或只提供静态文件的盘，挂成 `ro` 后连 root 都写不进去，杜绝误改和勒索软件加密。临时切换有现成姿势：

```bash
mount -o remount,ro /data     # 不卸载，直接转只读
mount -o remount,rw /data     # 需要写时再转回来
```

`-o remount` 还常配合 `bind` 玩法（`mount --bind /srv/web /srv/web-ro`）给同一个盘做「一写多读」的两种视图，那是更进阶的话题，知道有这条路就行。

> **SRE 贴士（生产 vs 实验室）**：实验室里 fstab 写错了大不了 `instabox ssh` 进去 `mv fstab.bak fstab`，五秒钟复活；生产机器改 fstab 的规矩是「双人复核 + 保留救援通道 + 变更窗口」，因为你面对的不是一台机器而是几百台，而人一定会在某个深夜写错 UUID。云上场景还多一层：如果这块盘是「 attach 到实例」的（块存储在云上叫卷），fstab 里记得加 `nofail`，否则实例开机时卷还没 attach 完就直接 emergency mode 等你。

<br>

## 10.7 无痛实验：镜像文件与 loop 设备

读到这你肯定心痒：分区、格式化、挂载这一套，能不能拿真盘练？**答案是永远不要手滑真盘。** 云电脑里没有第二块物理盘，而标准替代方案成熟得看不见危险——**loop 设备（loop device）**：内核提供一个「回路」设备，把一头的块访问请求转发到另一头的**普通文件**。于是：**一个文件，伪装成一块盘。**

### 10.7.1 第一步：造一块「假盘」

```bash
truncate -s 100M /data/home/disk.img      # 推荐：稀疏文件，秒建，逻辑 100M
ls -lh /data/home/disk.img
```

```text
-rw-r--r-- 1 root root 100M 11-13 09:25 /data/home/disk.img
```

`truncate` 直接把文件拉伸到 100M，但**块并没有真的分配（稀疏文件）**，所以下一秒完成、不占真空间——10.3 说的 du 和 du --apparent-size 分歧就在这：`du -sh disk.img` 接近 0，`du --apparent-size -sh disk.img` 报 100M。

传统写法是 dd，也是一定要知道的一条（但请配合警告读）：

```bash
dd if=/dev/zero of=/data/home/disk.img bs=1M count=100
```

```text
100+0 records in
100+0 records out
104857600 bytes (105 MB, 100 MiB) copied, 0.31 s, 338 MB/s
```

逐参数解释：

- `if=`（input file）：输入源。`/dev/zero` 是「无穷无尽的零」设备文件，读出多少字节都不心疼。
- `of=`（output file）：输出目标。
- `bs=`（block size）：一次读写的块大小，配 MiB 单位写 `1M`。
- `count=`：总共多少块。100 × 1M = 100M。

> **警告**：**dd 的别名叫 disk destroyer，不是开玩笑的**——它对「写哪个文件」没有任何判断，`of=` 指错一个字符（少打一个字母、多打一个数字）就是不可逆的系统级数据损毁，没有进度条、没有确认、没有回收站。安全姿势：写目标路径时先 `ls` 确认拼写；不确定就先 `echo` 一遍完整命令让眼睛检查 `of=`；生产上严禁在交互 shell 里手敲 dd 涉及系统盘的命令，一律走经过评审的脚本。另外 dd 造盘是「实打实写满 100M」，和 truncate 的稀疏方案各有用处——教学建议：**平时用 truncate，需要模拟真实磁盘写入行为时用 dd。**

### 10.7.2 第二步：格式化、挂载、写入、卸载

「盘」有了，里面还是空白。给它一个文件系统才能存文件：

```bash
mkfs.ext4 /data/home/disk.img
```

> **警告**：**mkfs（make file system，格式化）会清空目标上的所有数据，不可恢复。** 对物理设备执行 mkfs 之前：`lsblk` 确认设备名、`blkid` 确认 UUID、`findmnt` 确认没被挂载（在用的盘格式化会直接毁掉运行中的文件系统）。对 loop 镜像执行 mkfs 是安全的——这正是本节用它做实验的原因。**但养成条件反射：mkfs 敲下去之前，最后一遍核对目标。**

```text
mke2fs 1.47.x (13-Nov-2026)
Creating filesystem with 25600 4k blocks and 25600 inodes
Filesystem UUID: 8e1f4a2b-...
Superblock backups stored on blocks:
        8193, 24577, 40961, 57345, 73729, 12297, 16385, 20481
Allocating group tables: done
Writing inode tables: done
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done
```

（版本号、UUID 以你的实际输出为准。）接着准备挂载点并挂载：

```bash
mkdir -p /mnt/loop                       # 挂载点必须是一个真实存在的空目录
mount -o loop /data/home/disk.img /mnt/loop   # -o loop：mount 自动找空闲 loop 设备、把镜像挂上去
df -h /mnt/loop
```

```text
Filesystem      Size  Used Avail Use% Mounted on
/dev/loop0       93M   24K   91M   1% /mnt/loop
```

一个 93M（扣除文件系统开销后的可用容量）的「新盘」凭空出现。往里写文件、读回来，和真盘没有任何区别：

```bash
echo "hello loop device" > /mnt/loop/a.txt
cat /mnt/loop/a.txt
```

```text
hello loop device
```

看一眼内核 assigned 的 loop 设备：

```bash
losetup -a
```

```text
/dev/loop0: [0009]:12345 (/data/home/disk.img)
```

格式是「loop 设备 → 后端文件」。卸载时先 umount，loop 设备自动释放：

```bash
cd /                                # 别站在挂载点里面，否则 target is busy
umount /mnt/loop
losetup -a
```

```text
```

干干净净。这一套跑完，「分区 → 文件系统 → 挂载 → 读写 → 卸载」的完整闭环你已经亲手做过一次了，而物理数据零风险。10.4 讲的所有概念、10.6 讲的 fstab，都可以拿 loop 镜像反复演练：想练 10.6 就在 fstab 里加一行 `/data/home/disk.img /mnt/loop ext4 loop,defaults 0 0`，跑 `mount -a` 验证，删掉、再 `mount -a` 回来——**犯错成本约等于零，这正是实验环境的正确打开方式。**

> **SRE 贴士**：loop 设备在真实运维里也有一席之地：云主机扩容的临时挂载、挂载系统镜像（loop 挂 iso 装软件）、容器层的挂载（overlayfs 底层也在用它）。知道「一块盘 = 一个文件挂上去」这个抽象，很多云上和容器里的磁盘魔术就瞬间说得通了。但记住前提：实验室练手用 loop；生产动真盘之前，仍回到 10.5 三件套（lsblk/blkid/findmnt）核对身份。

<br>

## 10.8 swap：用磁盘换内存

### 10.8.1 为什么要有 swap

内存（RAM）是易失的、有限的。当所有程序要的内存加起来超过物理内存，内核有两个选择：立刻 OOM killer 枪毙进程（生硬，业务直接挂），或者把「暂时不动的内存页」挪到磁盘上一块专用区域，腾出 RAM 给活跃进程——这块区域就是 **swap（交换分区/交换文件）**。swap 让系统在内存压力下「软着陆」，代价是这些被换出的页再访问时会慢几个数量级（回到 10.1 金字塔，它跨了一层）。

Debian 传统做法是一块独立的 swap 分区；现代发行版（包括 Debian 自己）也都支持 **swapfile**——一个普通文件当 swap。云主机没有多余分区可切，swapfile 就是唯一现实选项。

### 10.8.2 一条命令级别的完整流程

Debian 13 装机默认会带一块 swap 分区，先看你机器上的现状：

```bash
swapon --show
```

```text
NAME       TYPE      SIZE USED PRIO
/dev/vda2  partition 2G   0B   -2
```

```bash
free -h
```

```text
               total        used        free      shared  buff/cache   available
Mem:           7.6Gi       1.2Gi       4.4Gi        88Mi       2.4Gi       6.1Gi
Swap:          2.0Gi          0B       2.0Gi
```

（swapon --show 空输出时表示当前没有激活的 swap，以你的机器为准。）`PRIO` 是优先级，`-2` 表示默认低优先级。想加一块 swapfile，顺序固定——**建文件、锁权限、格式化、启用、登记 fstab**：

```bash
fallocate -s 1G /swapfile     # 分五步里的第一步：分配 1G 连续空间
chmod 600 /swapfile           # 第二步：权限必须是 600，否则拒绝使用
mkswap /swapfile              # 第三步：写入 swap 签名头
swapon /swapfile              # 第四步：立即启用
swapon --show                 # 第五步：确认生效
```

```text
Setting up swapspace version 1, size = 1 GiB (1073737728 bytes)
no label, UUID=3f6d8e1c-...
```

```bash
swapon --show && free -h | grep Swap
```

```text
NAME      TYPE   SIZE USED PRIO
/swapfile file   1G   0B   -3
Swap:          3.0Gi          0B       3.0Gi
```

要用传统 dd 姿势也可以（`dd if=/dev/zero of=/swapfile bs=1M count=1024`，配合 10.7 的警告读——`of=` 拼写错不得），fallocate 更快更现代，但要注意：fallocate 在某些文件系统（xfs、btrfs）上可能产生「打洞」的不可用于 swap 的文件，ext4 上放心用。

**登记 fstab 才能重启后还在**，加一行（快照一个备份先）：

```text
/swapfile none swap sw 0 0
```

对照 10.6 的六字段：第一列文件路径；第二列固定 `none`；第三列类型 `swap`；第四列 `sw`（swap 的惯用选项）；第五六列都 0（不 dump、不 fsck）。**永远不要对 swapfile 写 pass 非 0**，fsck 会试图「检查」它，那没有意义。

> **警告**：swap 不是「内存越大越好」的解药。三件事必须知道：一、**swapon 和 fstab 改动一样，属于会「固化风险」的操作**——确认 fallocate 出来的文件没打洞（`filefrag -v /swapfile` 可查）、`chmod 600` 一步不许省（权限不对 swapon 直接拒绝：「insecure permissions 0644 0600 expected」）；二、生产上加 swap 前先想清楚「物理内存够不够」这个真问题，swap 是缓冲不是扩容，业务持续吃 swap 说明容量规划失败，该加内存了；三、云主机上 swap 还有一层现实：swap 内容写的是内存里的敏感数据，泄密面≈磁盘裸奔，对安全敏感的系统要么加密 swap、要么用完 `swapoff /swapfile && rm /swapfile` 关掉。生产环境这些操作都走变更流程，别直接 root 敲。

### 10.8.3 加，还是不加：实践建议

8 GiB 内存的云电脑，需要 swapfile 吗？分层回答：

- **日常学习/开发**：系统自带的 swap 分区已经够兜底，不用折腾。
- **要跑编译、大数据类任务、多个容器**：`free -h` 里 available 常低于 1G 时，加 1~2G swapfile 是性价比极高的保险——它不让你更快，但能在峰值时保住进程不被 OOM killer 随机枪决（OOM 挑进程的逻辑详见第 8 章）。
- **生产**：不拍脑袋加。按业务峰值的内存画像规划，swap 大小、分区还是文件、要不要加密，是容量设计的一部分。

> **SRE 贴士（生产 vs 实验室）**：实验室里你可以随便 `fallocate -s 8G /swapfile` 造个巨无霸玩；生产的答案是「先量后加」——用 30 天的监控数据说话，看内存峰值和 average，再决定是加内存还是加 swap。云主机和虚拟机还多一个优势：内存不够时升配比调 swap 更接近根治（虽然本书第一卷不展开云上弹性，卷 2 会讲）。

<br>

## 10.9 磁盘满了怎么办：一套标准排查思路

告警来了：`disk usage above 90%`。顺序不要乱，四步走完再动手，全程「只读诊断优先」：

**第一步，定位是哪个分区满了**——用 `df -h`，用百分比排序看最紧的那个。若是系统盘 `/` 满了，99% 的概率不是你的数据，是日志和缓存。

**第二步，inode 维度确认**——`df -i` 看哪个分区 IUse% 高。10.2.2 讲过，两个维度都看才不会漏。

**第三步，du 往下钻**——`du -h --max-depth=1 / | sort -rh | head` 找疑似大目录，逐层下钻，直到看见具体是哪些文件：

```bash
du -h --max-depth=1 /var | sort -rh | head
```

```text
7.8G	/var/log
940M	/var/lib/docker
380M	/var/cache/apt
```

**第四步，对「病灶」分类处置**——SRE 经验里，磁盘大户高度集中在四类：

1. **程序日志**（`/var/log`）：nginx access log、应用日志没人管地疯长。根治靠 logrotate 自动切割压缩（详见第 14 章《定时任务与系统日志》），救急先 `> bigfile.log` 清空（注意是重定向清空不是 rm——rm 掉文件后进程仍持有句柄，空间不释放，这就是玄学「删了日志 df 不降」的原因）。
2. **apt/docker 缓存**（`/var/cache/apt`、`/var/lib/docker`）：`apt autoclean`、`apt clean`（详见第 9 章《软件包管理：apt 与 dpkg》）；docker 用 `docker system prune`。
3. **程序自己的数据**（数据库文件、构建产物、上传目录）：这类别急着删，先搞清楚业务归属，该归档归档、该迁移 /data 迁移 /data。
4. **孤儿文件**：生命周期结束但没人删的临时文件，`find / -name '*.tmp' -mtime +7` 之类的组合清理，注意 `find -delete` 前先 `-print` 预演（find 的执行详见第 4 章）。

还有个灵异现象单独说：**「删了大文件，df 就是不降」**。原因是那个文件还被某个进程开着（日志句柄最典型），inode 没释放。找到它：

```bash
lsof +L1                 # 列出所有「已删除但仍被占用」的文件
```

```text
COMMAND   PID     USER   FD   TYPE DEVICE SIZE/OFF NLINK    NODE NAME
nginx   1147 www-data    6w   REG  259,1        0    0  4589131 /var/log/nginx/access.log (deleted)
```

`(deleted)` 后缀就是元凶。处置：通知进程重开文件（nginx 用 `nginx -s reopen` 或 kill -USR1 优雅重载，信号机制详见第 8 章），或者万不得已 `truncate -s 0 /proc/1147/fd/6` 直接把句柄截零——后一招是手术刀，用前想清楚。

<br>

## 10.10 LVM 概念科普：给磁盘穿上「可调节嫁衣」

Linux 传统分区是「一刀切」：一个分区 16G，一半给 `/` 一半给 `/home`，哪天 /home 满了而 / 空着，只能备份、重划、恢复，劳民伤财。**LVM（Logical Volume Manager，逻辑卷管理）** 在「物理盘」和「文件系统」之间加了一层抽象，让空间可以像水池一样分配、按需伸缩：

```text
 ┌───────────────┐   ┌───────────────┐
 │  物理分区/磁盘 │   │  物理分区/磁盘 │
 │  /dev/sda1    │   │  /dev/sdb1    │
 └───────┬───────┘   └───────┬───────┘
         │  pvcreate         │
         ▼                   ▼
 ┌───────────────┐   ┌───────────────┐
 │  PV 物理卷     │   │  PV 物理卷     │   第一层：物理卷（Physical Volume）
 └───────┬───────┘   └───────┬───────┘
         └─────────┬─────────┘
                   ▼  vgcreate
         ┌─────────────────┐
         │   VG 卷组        │        第二层：卷组（Volume Group）= 存储池
         │   myvg           │        把多块盘合成一池，池容量 = 各 PV 之和
         └────────┬────────┘
        ┌─────────┼─────────┐
        ▼         ▼         ▼  lvcreate（从池里「切」出逻辑卷）
 ┌──────────┐ ┌──────────┐ ┌──────────┐
 │ LV root  │ │ LV home  │ │ LV data  │   第三层：逻辑卷（Logical Volume）
 └────┬─────┘ └────┬─────┘ └────┬─────┘
      ▼            ▼            ▼
  mkfs+挂载    mkfs+挂载    mkfs+挂载
```

三层各管一件事：**PV** 把裸分区注册进 LVM；**VG** 把多个 PV 汇成一个容量池；**LV** 从池里切出「看起来像块设备、可以直接格式化挂载」的逻辑卷。好处三条：容量可以跨物理盘拼接、LV 可以在线扩缩（`lvextend` 加空间 + `resize2fs`/`xfs_growfs` 让文件系统跟着长，全程不卸载）、管理语义清晰。云主机上「系统盘在线扩容、命令敲完空间就变」的底层魔术，多数正发生在这一层。

三条命令看现状（示例输出，以你的机器为准）：

```bash
pvs && vgs && lvs
```

```text
PV         VG        Fmt  Attr PSize   PFree
/dev/vda2  debian-vg lvm2 a--  <15.00g <4.50g
  VG        #PV #LV #SN Attr   VSize   VFree
  debian-vg   1   2   0 wz--n- <15.00g <4.50g
  LV     VG        Attr       LSize   Pool Origin Data%  Meta%
  root   debian-vg -wi-ao----   8.00g
  home   debian-vg -wi-a-----   2.50g
```

看到 `VG` 那一列没有？`root` 和 `home` 两个「分区」其实躺在同一个叫 `debian-vg` 的池子里——这就是 Debian 安装器默认走 LVM 的机器长什么样。想扩容 LV，概念上就两步：`lvextend` 从池里多切空间给 LV，`resize2fs` 让 ext4 文件系统扩展到新边界（在线完成，不卸载、业务无感知）。

**本书的要求到此为止：知道这三层、认得 pvs/vgs/lvs 的输出、知道「生产云盘扩容常用」**。真正拿 LVM 搭生产存储、做快照管理，是中级运维的活，留白给实践——初创期你会遇到的绝大多数场景，一层分区加 fstab 就够了。

> **SRE 贴士（生产 vs 实验室）**：LVM 的「可在线扩容」是生产刚需（加盘不动业务），但 LVM 也让操作链变长——一次扩容涉及三层工具（pvcreate/vgextend/lvextend/resize2fs），每一步都有自己会出错的方式。所以 LVM 不是「更先进就该上」：**数据可丢、环境可重建的机器，简单直分区反而更扛造**；上 LVM 的时机是「你确实需要在线扩、且愿意为它写清楚 runbook」。和前面所有工具一样：复杂度是借来的，要还是得还。

<br>

## 10.11 InstaBox 特有：系统盘与持久盘的分工

最后回到你的云电脑本身，把本章所有概念落到自己的机器上。第 1 章给出的硬件事实，现在用存储视角重新念一遍：

| 盘 | 容量 | 挂载点 | 住的是什么 | 命硬不命硬 |
| --- | --- | --- | --- | --- |
| 系统盘 | 16 GiB | `/` | 操作系统、你装的 apt 软件、`/etc`、`/var` 全部 | **软**：重建/重装后回到初始镜像，`/etc` 改了、包装了、日志清了，一律从零再来 |
| 持久盘 | 20 GiB | `/data`（家目录 `/data/home`） | 你的练习文件、脚本、个人数据 | **硬**：快照保留它，suspend 保留它，重建后还在 |

两条实践结论，都是被无数「系统盘又满了」换来的：

第一，**你的家目录`/data/home`就在持久盘上，这不是巧合，是平台替你做的默认正确选择**——放 /data 的东西比放系统盘抗折腾。养成肌肉记忆：拿不准数据放哪，就 `/data/home`，别往 `/root`、`/tmp` 堆。

第二，**快照不是时间机器，快照保存的是「停机时的机器状态」，现场（内存里的进程、没落盘的数据）不在快照里；而系统盘上的自定义内容在重建时是会丢的**。所以真正的数据纪律只有一条：**重要数据三二一原则**——持久盘一份、快照一份、关键的东西再用 git 推一份到云端仓库（单文件 10 MiB 上限，大文件分卷或走 git，详见第 15 章《防火墙与数据备份》）。本章学完，请至少为你的 `book` 云电脑存一个干净快照，名字就叫 `base`——这是你后面所有「手滑现场」的后悔药。

> **SRE 贴士（生产 vs 实验室）**：实验室里「系统盘随便造、持久盘保数据」的分层已经是你体验过的最好生产预演：生产的持久层叫「对象存储 + 备份」，快照叫「镜像 + 恢复演练」，纪律完全一样——**数据的位置和副本数，是架构设计的一部分，不是运维善后**。另外别忘了免费档 200 GB 总量上限是「机器 + 快照合计」，删机器前先确认你要的数据都在你推得出云端的地方。

<br>

## 本章小结

这一章的主线是「数据在哪、账怎么算、盘怎么接、满了怎么办」：

```bash
df -h / df -i             # 双维度体检：空间账 + inode 账，缺一不可
du -h --max-depth=1 dir | sort -rh   # 下钻定位大目录
lsblk / blkid / findmnt   # 块设备三件套，认 UUID 不认设备名
mount / umount            # 接入与摘除；busy 就 cd 出去 + fuser/lsof
cat /etc/fstab            # 六字段背下来：UUID、挂载点、类型、选项、dump、pass
mount -a                  # 改完 fstab 的验证姿势，不用重启
truncate + mkfs.ext4 + mount -o loop   # loop 设备无痛实验三连
fallocate + mkswap + swapon   # swapfile 标准流程，fstab 登记
pvs/vgs/lvs               # LVM 三层概念，认得输出即可
```

真正要带走的不是命令，是三个思维方式：**「先看账本再动手」**（df/du 看完再说）；**「写操作前想回滚」**（换源、改 fstab 都先备份，`mount -a` 当场验证）；**「复杂度是借来的」**（loop 实验、简单分区、LVM，各用在各自划算的场景）。

## 动手练习

1. 跑 `df -h` 和 `df -i` 各一次，把输出贴进笔记，逐列标出含义；再用 `lsblk`、`blkid`、`findmnt` 看看你的系统盘和 /data 分别是什么设备、什么 UUID、什么挂载参数。
2. 按 10.7 的流程用 loop 设备做一次完整实验：建 100M 镜像、格式化、挂到 `/mnt/loop`、拷几个文件进去、`ls -i` 看看这些文件的 inode、卸掉、再挂回来确认文件还在。
3. 对 `disk.img` 练习 fstab：备份 `/etc/fstab`，添加一行让镜像开机自动挂到 `/mnt/loop`（选项写 `loop,defaults`），`mount -a` 验证，`findmnt /mnt/loop` 确认，然后把 fstab 恢复原状并再次 `mount -a`。
4. 做一次「空间去哪了」演练：`du -h --max-depth=1 /data/home | sort -rh | head`，找出你家里最大的三个目录；再 `du -sh /var/cache/apt/archives`，跑 `apt autoclean` 前后各记一次大小。
5. 查询并记录：`swapon --show` 的输出（或空输出）、`free -h` 的 Swap 行。然后按 10.8.2 的流程加一个 1G 的 swapfile（记得 chmod 600、登记 fstab），用 `free -h` 前后对比。
6. 综合题：模拟一次「磁盘满」排障：`dd` 在 /data/home 造一个 2G 的大文件，然后完整走一遍「df → du → 定位 → 处置（`lsof +L1` 确认无人占用 → rm）→ 再 df」，把每一步的输出存成 /data/home 下的排障笔记。

> **参考答案**：
> 1. 对照 10.2.1、10.5.1 的表格逐列解读即可；注意 /data 大概率是独立设备（如 /dev/vdb1）、`/` 大概率是 /dev/vda1，两台云电脑设备名可能不同，UUID 不会认错。
> 2. 注意 `mount -o loop` 后 `df -h` 多出一个文件系统；`ls -i` 的 inode 号在同一文件系统内唯一；卸载后 inode 号作废，重挂后同一文件 inode 通常不变（ext4 分配稳定）。
> 3. fstab 加行 `/data/home/disk.img /mnt/loop ext4 loop,defaults 0 0`；`mount -a` 挂载成功、`findmnt` 能看到 OPTIONS 含 loop；恢复用备份文件覆盖 + `umount`。
> 4. du 命令会列出 /data/home 下的一级目录；apt autoclean 通常回收几十到几百 MB（缓存大小取决于你装过多少软件）。
> 5. 加 swapfile 前后 Swap 总量应增加约 1G；如果 swapon 报 insecure permissions，回去检查是不是漏了 `chmod 600`；记得把 `/swapfile none swap sw 0 0` 写进 fstab 才重启有效。
> 6. 完整链路：`df -h`（/data 使用率 +10%）→ `du -h --max-depth=1 /data/home | sort -rh` 找到大文件 → `lsof +L1` 或 `fuser` 确认没有进程占用 → `rm bigfile` → 再 `df -h` 确认回收。练完这个流程，你就有了对告警的第一反应。



