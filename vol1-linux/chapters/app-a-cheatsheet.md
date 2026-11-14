---
title: "附录 A 命令速查表"
---

# 附录 A 命令速查表

> 这份表是给你翻的，不是拿来背的。正文里每个命令都讲透了原理，附录只做一件事：把第 3~15 章讲过的常用命令压缩成一行一条，**日常当词典，忘参数时来查**。每个条目三列——命令是什么、一句话说清它干什么、最常用的参数长什么样。参数只收正文验证过、生产也高频的，宁缺毋滥；拿不准的组合，回到对应章节看完整讲解，或者用 `命令 --help` 先问机器自己。

> **警告**：表里 `rm -rf`、`dd`、`mkfs`、`rsync --delete`、`ufw default deny` 这类命令有一个共同点——**做下去就没有第二次机会**。在生产上执行前，先问自己老三问：设备名/路径瞪清三遍、有没有备份、这是不是唯一副本。实验室里弄坏了大不了重建，生产上一个 Tab 补全的手滑就是一篇事故报告。

> **SRE 贴士**：真正的老手也离不开速查表，只是他们的速查表在脑子里。建议你把本附录打印出来（或抄进自己的笔记），每次真用了一条命令，就在旁边画一道——画满五道的，才是你自己的命令。

## A.1 文件与目录

覆盖第 3 章《命令行入门与文件系统树》、第 4 章《文件与目录操作》。

| 命令 | 一句话说明 | 最常用参数示例 |
| --- | --- | --- |
| `ls` | 列出目录内容，运维出场率最高的命令 | `ls -lah`（长格式 + 全部含隐藏 + 人性化大小） |
| `cd` | 切换工作目录 | `cd /data/home`；`cd -` 跳回上一个目录 |
| `pwd` | 打印当前所在的绝对路径 | `pwd` |
| `cp` | 复制文件或目录，源可多个、目标最后一个 | `cp -a 源目录/ 目标目录/`（-a 递归并保权限时间戳） |
| `mv` | 移动或重命名，不产生副本 | `mv old.log /data/home/logs/` |
| `rm` | 删除文件或目录，删了不可恢复 | `rm -i useless.log`（-i 逐个确认，手滑保险丝） |
| `mkdir` | 创建目录 | `mkdir -p /data/home/logs/app`（-p 一口气建多层） |
| `touch` | 建空文件，或把已有文件的时间戳刷成现在 | `touch /data/home/note.txt` |
| `ln` | 建链接：加 `-s` 是软链接，默认是硬链接 | `ln -s /data/home/real /data/home/link` |
| `find` | 按名字、类型、时间、大小实时查找文件 | `find /data/home -name "*.log" -mtime +7` |
| `file` | 看文件真实类型，不靠扩展名猜 | `file unknown.deb` |
| `cat` | 把整个文件倒进终端，只适合小文件 | `cat -n /etc/hostname`（-n 带行号） |
| `head` / `tail` | 看文件开头 / 结尾若干行 | `tail -n 100 app.log`；`tail -f app.log` 实时跟踪 |
| `less` | 分页浏览，多大的文件都秒开，`man` 的底层 | `less /var/log/dpkg.log`（按键见附录 B） |
| `date` | 看当前时间，或按指定格式输出 | `date "+%F %T"`（年-月-日 时:分:秒） |

## A.2 文本处理

覆盖第 6 章《文本处理：grep / sed / awk》、第 7 章《vim 编辑器》。日志、配置、YAML——SRE 的命门全在这几条命令上。

| 命令 | 一句话说明 | 最常用参数示例 |
| --- | --- | --- |
| `grep` | 按模式过滤行，文本工具里的老大哥 | `grep -n -r "error" /data/home/logs`（-n 行号、-r 递归目录） |
| `grep -E` | 扩展正则，`|` 不需要反斜杠直接表示「或」 | `grep -E "50[0-9] " access.log` |
| `sed` | 流编辑器，替换、删除、插入一条龙 | `sed -i.bak 's/old/new/g' app.conf`（-i 直接改，`.bak` 留原文件备份） |
| `sed -n` | 配合 `p` 打印指定行，其余不刷屏 | `sed -n '1,10p' access.log` |
| `awk` | 按字段切割统计，写报表的神器 | `awk '{print $1}' access.log \| sort \| uniq -c \| sort -rn \| head -10` |
| `sort` | 排序；`-n` 按数值、`-r` 倒序、`-k` 指定列 | `sort -k2 -rn data.txt` |
| `uniq` | 去重（通常先 `sort`），`-c` 统计重复次数 | `sort access.log \| uniq -c` |
| `cut` | 按定界符切出第几列 | `cut -d: -f1 /etc/passwd` |
| `wc` | 统计行数、词数、字节数 | `wc -l access.log` |
| `tr` | 替换或删除字符，常用来清行尾回车 | `tr -d '\r' < win.txt` |
| `diff` | 比较两个文件差异，`-u` 输出能直接喂给 `patch` | `diff -u old.conf new.conf` |
| `nl` | 输出时加行号（默认空行不编号） | `nl note.txt` |
| `tac` | `cat` 倒着拼，最近的错误排最前 | `tac app.log` |
| `vim` | 终端编辑器，服务器上绕不开 | `vim +42 app.log`（打开直跳第 42 行；键位速查见附录 B） |

## A.3 权限与用户

覆盖第 5 章《用户、群组与权限管理》。实验室里默认是 root，但每条命令在生产上的前提都是「最小权限」。

| 命令 | 一句话说明 | 最常用参数示例 |
| --- | --- | --- |
| `chmod` | 改权限，数字法直观、符号法精准 | `chmod 755 dir/`；`chmod u+x script.sh` |
| `chown` | 改文件的属主与属组 | `chown bookuser:bookuser file` |
| `useradd` | 新建用户，登记进 passwd/shadow/group | `useradd -m -s /bin/bash bookuser` |
| `usermod` | 改用户属性，最常用是追加附属组 | `usermod -aG sudo bookuser` |
| `userdel` | 删除用户，`-r` 连家目录和数据一起 | `userdel -r bookuser`（删前确认数据不要了） |
| `passwd` | 改密码、查密码状态、锁定与解冻账号 | `passwd bookuser`；`passwd -S bookuser` |
| `su` | 切换用户，加 `-` 连登录环境一起换 | `su - bookuser` |
| `sudo` | 以 root 身份执行单条命令，留痕可审计 | `sudo apt update` |
| `visudo` | 安全编辑 `/etc/sudoers`，保存时带语法检查 | `visudo` |
| `id` | 看 uid、gid 和所属全部组 | `id bookuser` |
| `groups` | 只列用户所属的组名，脚本里好用 | `groups bookuser` |
| `umask` | 新建文件/目录的默认权限掩码 | `umask 002`（只对当前 shell 生效） |
| `last` | 翻登录历史，来自 /var/log 的 wtmp | `last -n 20` |
| `lastb` | 翻失败的登录尝试，被爆破时排障用 | `lastb -n 20` |

## A.4 进程与性能

覆盖第 8 章《进程管理与系统监控》。机器卡了、CPU 满了，从这张表抄家伙。

| 命令 | 一句话说明 | 最常用参数示例 |
| --- | --- | --- |
| `ps` | 进程快照，两大姿势：`aux` 与 `-ef` | `ps aux` |
| `top` | 实时进程与负载面板，进界面再排序 | `top`（界面里 `M` 按内存、`P` 按 CPU） |
| `htop` | `top` 的彩色加强版，需 `apt install htop` | `htop` |
| `kill` | 按 PID 给进程发信号，默认 15（TERM） | `kill -15 <PID>`；`-9` 是强杀，没得商量 |
| `pkill` | 按名字或命令行模式发信号，不用查 PID | `pkill -f "backup.sh"` |
| `killall` | 按进程名杀，同名的一起走 | `killall nginx` |
| `jobs` | 看当前终端的后台任务清单 | `jobs -l` |
| `fg` / `bg` | 把后台任务调回台前 / 继续埋进后台 | `fg %1`；`bg %1` |
| `nohup` | 让任务无视挂断继续跑，输出重定向到日志 | `nohup ./job.sh > job.log 2>&1 &` |
| `tmux` | 终端复用器，会话级保活，断线不丢现场 | `tmux new -s work`；`tmux attach -t work` |
| `free` | 看内存与 swap 用量，认 available 那列 | `free -h` |
| `vmstat` | 系统级性能采样，r 和 b 看排队 | `vmstat 1 5`（每秒一次，共 5 次） |
| `iostat` | 磁盘 I/O 采样，看 %util 和 await | `iostat -x 1`（属 sysstat 包，需另装） |
| `dmesg` | 内核环形缓冲，OOM、硬件大事都在这 | `dmesg -T \| tail -n 20`（-T 人类可读时间） |
| `uptime` | 开机时长与 1/5/15 分钟平均负载 | `uptime` |

## A.5 软件包

覆盖第 9 章《软件包管理：apt 与 dpkg》。记住分层：`apt` 管仓库和依赖，`dpkg` 管单个包。

| 命令 | 一句话说明 | 最常用参数示例 |
| --- | --- | --- |
| `apt update` | 刷新软件源索引，装包、升级前必做 | `apt update` |
| `apt install` | 安装软件包，自动解决依赖 | `apt install htop` |
| `apt remove` / `purge` | 卸载；`purge` 连配置文件一起清 | `apt purge nginx` |
| `apt autoremove` | 清掉没人再依赖的孤儿包 | `apt autoremove` |
| `apt upgrade` | 升级已安装的包，跨大版本用 `full-upgrade` | `apt upgrade` |
| `apt search` / `show` | 搜包 / 看包详情与依赖关系 | `apt search nginx` |
| `apt list` | 列已安装或可升级的包 | `apt list --installed` |
| `apt-mark` | 锁定版本，升级时跳过它 | `apt-mark hold nginx` |
| `dpkg -l` | 查已安装包及版本状态 | `dpkg -l \| grep nginx` |
| `dpkg -L` / `-S` | 包装了哪些文件 / 某个文件属于哪个包 | `dpkg -S /usr/sbin/nginx` |
| `dpkg -i` | 安装本地 .deb 文件 | `dpkg -i ./pkg.deb`（依赖不全再 `apt -f install`） |
| `apt -f install` | 修复残缺的依赖关系 | `apt -f install` |
| `apt clean` | 清空下载缓存，救 20 GiB 小盘 | `apt clean` |

## A.6 磁盘与存储

覆盖第 10 章《磁盘与存储管理》。「磁盘满了」是 SRE 最高频的事故类型，前三行就是排查起手式。

| 命令 | 一句话说明 | 最常用参数示例 |
| --- | --- | --- |
| `df` | 文件系统磁盘用量，满了先看它 | `df -h`；`df -i` 看 inode（文件数） |
| `du` | 目录/文件占用，定位到底谁在吃盘 | `du -sh /data/home`；`du --max-depth=1 /data/home` |
| `lsblk` | 块设备树状一览，盘、分区一目了然 | `lsblk` |
| `blkid` | 查看设备 UUID 与文件系统类型（fstab 要用的） | `blkid` |
| `findmnt` | 看谁挂在哪、用什么参数挂的 | `findmnt` |
| `mount` | 把文件系统挂到挂载点上 | `mount -o loop disk.img /mnt/loop` |
| `umount` | 卸载（注意拼写没有 n），先退出该目录 | `umount /mnt/loop` |
| `mount -a` | 按 `/etc/fstab` 批量挂载，也是验证配置的手段 | `mount -a` |
| `mkfs.ext4` | 格式化（高危，设备名瞪清再回车） | `mkfs.ext4 disk.img`（loop 实验的标准做法） |
| `dd` | 裸读写，建镜像、写盘都靠它（高危） | `dd if=/dev/zero of=disk.img bs=1M count=100` |
| `truncate` | 秒建指定大小的稀疏文件 | `truncate -s 100M disk.img` |
| `fallocate` | 预分配文件，建 swapfile 的标准姿势 | `fallocate -l 2G /swapfile` |
| `mkswap` | 把文件或分区做成 swap 区 | `mkswap /swapfile` |
| `swapon` / `swapoff` | 启用 / 停用 swap | `swapon --show` |
| `lvs` | LVM 逻辑卷一览（概念科普见第 10 章） | `lvs` |

## A.7 网络

覆盖第 11 章《网络配置与远程管理》。示例里的 `host` 泛指目标机器，套你自己场景即可；DNS 示例用 `baidu.com` 只是为了好认。

| 命令 | 一句话说明 | 最常用参数示例 |
| --- | --- | --- |
| `ip addr` | 看网卡与 IP 地址（`ifconfig` 的接班人） | `ip -br addr`（-br 一行一个网卡，清爽） |
| `ip route` | 看路由表与默认网关 | `ip route` |
| `ss` | 看 socket 与端口占用，谁占了 80 秒懂 | `ss -tulnp`（-l 监听、-p 进程） |
| `ping` | 测连通性，`-c` 限次数别没完没了 | `ping -c 4 baidu.com` |
| `traceroute` | 看过去路上每一跳，`mtr` 是它的升级版 | `traceroute baidu.com` |
| `dig` | DNS 查询利器，比 `nslookup` 信息全 | `dig baidu.com`；`dig +short` 只要结果 |
| `getent hosts` | 按系统解析顺序查主机名（先 /etc/hosts 后 DNS） | `getent hosts baidu.com` |
| `hostnamectl` | 查看 / 设置主机名与静态主机信息 | `hostnamectl set-hostname book` |
| `curl` | HTTP 客户端，调试接口、下载都先摸它 | `curl -I https://baidu.com`（-L 跟跳转、-o 存文件、-v 看细节） |
| `wget` | 命令行下载，`-c` 断点续传 | `wget -c https://example.com/pkg.tar.gz` |
| `ssh` | 远程登录，`-p` 指定端口 | `ssh -p 22 bookuser@host` |
| `ssh-keygen` | 生成密钥对，推荐 ed25519 | `ssh-keygen -t ed25519` |
| `ssh-copy-id` | 把公钥装进远端的 authorized_keys | `ssh-copy-id bookuser@host` |
| `scp` | 跨机复制文件，端口参数是大写 `-P` | `scp -P 22 app.log bookuser@host:/data/home/` |
| `sftp` | 交互式文件传输，适合挑挑拣拣传几个文件 | `sftp bookuser@host` |

> **SRE 贴士**：`scp`/`wget`/`rsync` 传文件前看一眼大小——云电脑单文件传输上限 10 MiB，超了分卷压缩，或者走 git。传大文件时踩这个上限，是最没技术含量也最常犯的错。

## A.8 systemd 与日志

覆盖第 12 章《systemd 与服务管理》、第 14 章《定时任务与系统日志》。服务起不来、日志去哪看，全靠这一页。

| 命令 | 一句话说明 | 最常用参数示例 |
| --- | --- | --- |
| `systemctl start/stop/restart/reload` | 服务生命周期四件套，立即生效 | `systemctl restart nginx`；`reload` 不断服务 |
| `systemctl status` | 状态三行 + 最近日志，排障第一入口 | `systemctl status nginx` |
| `systemctl enable/disable` | 开机自启的开关，只管开机行为 | `systemctl enable nginx` |
| `systemctl daemon-reload` | 改完 unit 文件必须重载，否则不认 | `systemctl daemon-reload` |
| `systemctl mask/unmask` | 彻底禁用 / 解禁一个单元 | `systemctl mask hello.service` |
| `systemctl is-active/is-enabled` | 只问状态，脚本里判断好用 | `systemctl is-active nginx` |
| `systemd-analyze verify` | 静态检查 unit 文件语法，改完先验一遍 | `systemd-analyze verify /etc/systemd/system/hello.service` |
| `journalctl -u` | 看某个服务的全部日志 | `journalctl -u nginx --since today` |
| `journalctl -f / -e` | 实时跟踪 / 直接跳到日志末尾 | `journalctl -f -u nginx` |
| `journalctl -b / -k` | 本次启动的日志 / 内核日志 | `journalctl -k` |
| `journalctl -p` | 按优先级过滤，`err` 只看错误 | `journalctl -p err` |
| `journalctl --vacuum-time` | 按时间清理日志，救磁盘用 | `journalctl --vacuum-time=7d` |
| `crontab` | 定时任务：`-e` 编辑、`-l` 查看、`-r` 清空 | `crontab -e` |
| `logrotate` | 日志轮转，`-d` 干跑调试不真切 | `logrotate -d /etc/logrotate.d/nginx` |
| `logger` | 手动往日志体系写一条，脚本里标进度用 | `logger -t backup "每日备份完成"` |

`systemctl` 的子命令远不止这些：`list-unit-files` 看全部单元、`list-dependencies` 看依赖树、`isolate multi-user.target` 切换运行目标；SysV 时代的 `service` 命令也仍被兼容。详见第 12 章《systemd 与服务管理》。

## A.9 防火墙与备份

覆盖第 15 章《防火墙与数据备份》。这一页的每一个动作，都先想「我会不会把自己关在外面」。

| 命令 | 一句话说明 | 最常用参数示例 |
| --- | --- | --- |
| `ufw` | 防火墙前端，规则写得像人话 | `ufw allow 80/tcp` |
| `ufw status` | 看当前规则，`numbered` 模式方便按编号删除 | `ufw status numbered` |
| `ufw default` | 设默认策略（先放行 22，再默认拒绝） | `ufw default deny incoming` |
| `ufw delete` | 按编号或原规则删除一条 | `ufw delete 3` |
| `ufw limit` | 同一来源限制连接频率，防爆破 | `ufw limit 22/tcp` |
| `nft` | nftables 原生前端，Debian 13 的默认防火墙 | `nft list ruleset` |
| `fail2ban-client` | 查封禁统计与 jail 状态 | `fail2ban-client status sshd` |
| `tar -c` | 打包；`-z`/`-J`/`--zstd` 三选一压缩 | `tar -czf bk-$(date +%F).tar.gz /data/home` |
| `tar -x` | 解包，`-C` 指定解到哪 | `tar -xzf bk.tar.gz -C /tmp` |
| `tar -t` | 只看包里有什么，不解压 | `tar -tzf bk.tar.gz` |
| `tar --exclude` | 打包时排除目录或文件 | `tar -czf bk.tar.gz --exclude=/data/home/logs /data/home` |
| `rsync` | 同步与增量备份，只传差异部分 | `rsync -avz --progress 源目录/ 目标目录/` |
| `rsync` 远程形态 | 推到远端，或从远端拉回来 | `rsync -avz /data/home/ bookuser@host:/backup/` |
| `rsync --delete` | 让目标和源完全一致（高危，删多不补少） | `rsync -av --delete 源目录/ 目标目录/` |
| `restic` / `borg` | 加密 + 去重的现代备份工具（概念科普级） | `restic backup /data/home`（先 `restic init`） |

> **警告**：在只有 SSH 连着的机器上执行 `ufw default deny incoming` 之前，必须已经有一条放行 22 端口的规则，否则下一跳你就把自己关在外面，云电脑 console 都未必救得回来。正确顺序：先 `ufw allow 22/tcp`，再改默认策略。另外注意：在 InstaBox 上 `instabox ssh` 走平台通道，防火墙影响的是你实例内服务的暴露（详见第 15 章《防火墙与数据备份》）。

## A.10 InstaBox CLI 速查表

InstaBox 的命令都在**本地读者终端**执行，不是在云电脑里跑。安装 CLI 两种方式，按你的机器选：

```bash
# macOS / Linux / Git Bash
curl -fsSL https://api.instabox.sh/install.sh | sh

# 本机有 Node.js 时（Windows 最省事）
npx instabox@latest sh
```

| 命令 | 一句话说明 | 最常用参数示例 |
| --- | --- | --- |
| `instabox login` | 设备码登录，在任意设备的浏览器完成授权 | `instabox login` |
| `instabox create` | 创建云电脑，本书机器名统一叫 book | `instabox create --name book` |
| `instabox ssh` | SSH 进入云电脑，直接得到 root shell | `instabox ssh book` |
| `instabox suspend` | 暂停：文件、内存、进程现场全保留，只算存储费 | `instabox suspend book` |
| `instabox resume` | 恢复，通常秒级回到离开时的现场 | `instabox resume book` |
| `instabox exec` | 不进交互 shell，远程执行单条命令 | `instabox exec book -- uname -a` |
| `instabox exec --json` | 以 JSON 返回 stdout/stderr/退出码等，脚本友好 | `instabox exec book --json -- df -h` |
| `instabox snapshot save` | 存快照（保存前会停机，只存环境不存现场） | `instabox snapshot save book --name base` |
| `instabox create --snapshot` | 从快照开一台新机器 | `instabox create --name job2 --snapshot base` |
| `instabox port publish` | 把云电脑端口发布成 HTTPS 链接 | `instabox port publish book 3000` |
| `instabox port-forward` | 本机端口转发到云电脑的端口 | `instabox port-forward book 8080:3000` |

> **SRE 贴士**：`instabox create --name x --template universal` 会被拒（免费档锁死默认模板，报 `403 template_not_allowed`），从快照开机不受此限；并发打满会报 `429 quota_exceeded`。六个坑的完整解读见第 1 章《搭建实验环境：InstaBox 云电脑》。

## A.11 环境速记：免费档的关键数字

每台云电脑的硬边界，考前默背级：

| 项目 | 数值 | 容易忘的坑 |
| --- | --- | --- |
| CPU / 内存 | 4 vCPU / 8 GiB | 不能选配，`free -h` 显示略少是内核占用，正常 |
| 系统盘 | 16 GiB，ext4 | 只装系统和软件，别往上堆数据 |
| 持久盘 | 20 GiB，挂载点 `/data` | 家目录就是 `/data/home`，练习文件都放这 |
| 快照 | 5 个；总量上限 200 GB（机器 + 快照合计） | 保存前会停机，且不存进程现场 |
| 出网流量 | 100 GiB / 月 | 别把它当下载盘 |
| 公网端口 | 每台 16 个 | 对外暴露服务用 `instabox port publish` |
| 自动暂停 | 空闲 30 分钟自动暂停（免费档不可调） | 长任务丢 tmux/nohup，进度随手存快照 |
| exec 限制 | 单条命令上限 10 分钟，超时退出码 124 | 输出跑完才整段返回，不流式 |
| 传输限制 | 单文件 10 MiB | 大文件分卷压缩，或走 git |
| 并发 | 32 个并发，全账号共享 | 打满报 `429 quota_exceeded` |
| 系统事实 | Debian 13（trixie）、amd64、systemd | 内核版本尾巴带 `-instabox`，跑在 Firecracker microVM 上 |
| 模板 | 免费档只能用默认模板（`agent-browser`） | 加 `--template` 被拒（`403 template_not_allowed`）；从快照开机不受限 |

拿到机器先验货，第 1 章的五条命令，到现在也该背下来了：

```bash
nproc          # 4
free -h        # 总量约 7.x GiB（被内核占一部分属正常）
df -h /data    # 20G
docker info    # 能连上 daemon
uname -a       # 内核版本尾巴带 -instabox
```
