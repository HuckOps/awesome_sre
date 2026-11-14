---
title: "附录 C 术语表"
---

# 附录 C 术语表

本附录收录全卷涉及的约 80 条中英对照术语，按主题分组，每条附一句简短解释。初学者可将其作为阅读全书时的随手查表。

## C.1 SRE 与运维体系

| 术语 | 英文 | 解释 |
| --- | --- | --- |
| SRE | Site Reliability Engineering | 站点可靠性工程，Google 提出的将软件工程方法应用于运维的实践。 |
| SLA | Service Level Agreement | 服务等级协议，服务提供方与用户之间关于可用性等指标的正式承诺。 |
| SLO | Service Level Objective | 服务等级目标，SLA 中具体的、可衡量的目标值（如月可用性 99.9%）。 |
| SLI | Service Level Indicator | 服务等级指标，衡量服务表现的具体指标（如请求成功率、延迟）。 |
| 鸟哥风格 | Bird-style | 鸟哥的私房菜式写作风格，命令讲透语法、逐参数解释、附常见错误解读。 |
| 爆炸半径 | Blast Radius | 事故或变更可能影响的范围，越小越安全。 |
| 最小权限 | Least Privilege | 每个账号/进程只拥有完成工作所必需的最小权限。 |
| 可观测性 | Observability | 从系统外部输出（日志、指标、追踪）推断系统内部状态的能力。 |
| 变更记录 | Change Log | 对系统每一次配置、代码、基础设施变更的时间顺序记录。 |
| 回滚 | Rollback | 将系统恢复到之前已知良好状态的操作。 |

## C.2 Linux 基础与文件系统

| 术语 | 英文 | 解释 |
| --- | --- | --- |
| 发行版 | Distribution | Linux 发行版本，如 Debian、Ubuntu、RHEL。 |
| Debian | Debian | 以稳定著称的 Linux 发行版，本书用 Debian 13（trixie）。 |
| 内核 | Kernel | 操作系统核心，管理硬件与进程，本书内核带 `-instabox` 后缀。 |
| Shell | Shell | 命令解释器，本书默认用 bash。 |
| 提示符 | Prompt | 命令行等待输入时显示的字符，如 `root@book:~#`。 |
| FHS | Filesystem Hierarchy Standard | 文件系统层次标准，定义 / /etc /var 等目录用途。 |
| 绝对路径 | Absolute Path | 从根目录 `/` 开始的完整路径。 |
| 相对路径 | Relative Path | 从当前目录开始的路径。 |
| 家目录 | Home Directory | 用户登录后的默认目录，如 `/home/bookuser`。 |
| 当前目录 | Current Directory | 命令执行所在的目录，用 `.` 表示。 |
| 父目录 | Parent Directory | 当前目录的上一级，用 `..` 表示。 |
| 根目录 | Root Directory | 文件系统的最顶层，用 `/` 表示。 |
| 超级用户 | Superuser / root | UID 为 0 的系统管理员，拥有全部权限。 |
| 普通用户 | Regular User | 非 root 用户，权限受限。 |
| 属主 | Owner | 文件所属的用户（user, u）。 |
| 属组 | Group | 文件所属的群组（group, g）。 |
| 其他人 | Others | 既非属主也非属组的用户（others, o）。 |
| 权限位 | Permission | r（读）/w（写）/x（执行）三种基本权限。 |
| 八进制 | Octal | 用 0-7 数字表示权限，如 755、644。 |
| umask | User Mask | 新建文件的默认权限掩码，如 022 得 644/755。 |
| 隐藏文件 | Hidden File | 以 `.` 开头的文件，`ls` 默认不显示。 |
| 通配符 | Wildcard | `*`、`?`、`[]` 等用于匹配文件名的特殊字符。 |
| 软链接 | Symbolic Link | 指向另一个路径的快捷方式，可跨分区。 |
| 硬链接 | Hard Link | 指向同一 inode 的另一个目录项，不可跨分区。 |
| inode | Index Node | 文件系统内部对文件元数据的索引节点。 |
| 重定向 | Redirection | 把命令输出写入文件或从文件读取输入，如 `>`、`<`。 |
| 管道 | Pipeline | 把一个命令的输出作为另一个命令的输入，用 `\|`。 |
| 退出码 | Exit Code | 命令结束后返回的状态码，0 表示成功。 |

## C.3 用户、群组与权限

| 术语 | 英文 | 解释 |
| --- | --- | --- |
| UID | User ID | 用户 ID，内核只认数字不认用户名。 |
| GID | Group ID | 群组 ID，内核只认数字不认群组名。 |
| 主组 | Primary Group | 用户在 `/etc/passwd` 中登记的默认群组。 |
| 附加组 | Supplementary Group | 用户参加的其他群组，记录在 `/etc/group` 成员列表。 |
| 密码哈希 | Password Hash | 密码经加盐哈希后的结果，存储在 `/etc/shadow`。 |
| 加盐 | Salt | 哈希前加入的随机字符串，防止彩虹表攻击。 |
| 锁定账号 | Locked Account | 禁止密码登录的账号，`passwd -S` 显示 `L`。 |
| 服务账号 | Service Account | 给服务进程用的、禁止交互登录的账号，如 `www-data`。 |
| sudo | Superuser DO | 允许普通用户以 root 身份执行特定命令的机制。 |
| visudo | Visual Sudo | 编辑 `/etc/sudoers` 的安全命令，带语法检查。 |
| SUID | Set User ID | 文件执行时以文件属主权限运行，如 `passwd`。 |
| SGID | Set Group ID | 文件执行时以文件属组权限运行；目录上表示新文件继承目录群组。 |
| Sticky Bit | Sticky Bit | 目录上的特殊权限，只有文件属主和 root 能删除，如 `/tmp`。 |

## C.4 文本处理与 vim

| 术语 | 英文 | 解释 |
| --- | --- | --- |
| 正则表达式 | Regular Expression | 用于模式匹配的字符串，如 `^$`、`*+?`。 |
| 定界符 | Delimiter | sed 中分隔替换模式的字符，默认 `/`。 |
| 贪婪匹配 | Greedy Match | 正则尽可能多匹配字符的行为。 |
| 普通模式 | Normal Mode | vim 默认模式，用于导航和执行命令。 |
| 插入模式 | Insert Mode | vim 用于输入文本的模式，按 `i`/`a`/`o` 进入。 |
| 命令行模式 | Command-line Mode | vim 用于执行 `:w`/`:q` 等命令的模式，按 `:` 进入。 |
| 可视化模式 | Visual Mode | vim 用于选中文本块的模式，按 `v`/`V`/`Ctrl-v` 进入。 |
| 撤销 | Undo | 撤销上一次编辑操作，vim 按 `u`。 |
| 重做 | Redo | 重新执行被撤销的操作，vim 按 `Ctrl-r`。 |
| 寄存器 | Register | vim 用于存放复制/删除内容的缓冲区。 |

## C.5 进程与系统监控

| 术语 | 英文 | 解释 |
| --- | --- | --- |
| 程序 | Program | 磁盘上的静态可执行文件。 |
| 进程 | Process | 程序加载到内存后由内核调度运行的活体实例。 |
| PID | Process ID | 进程标识符，内核分配的唯一编号。 |
| PPID | Parent PID | 父进程标识符，记录进程是谁生出来的。 |
| fork | Fork | 父进程复制出子进程的系统调用。 |
| exec | Exec | 子进程把自己替换成另一个程序的系统调用。 |
| 守护进程 | Daemon | 后台运行、无控制终端的服务进程。 |
| 僵尸进程 | Zombie Process | 已退出但未被父进程回收的进程。 |
| 信号 | Signal | 进程间通知机制，如 `SIGTERM`、`SIGKILL`。 |
| 负载 | Load Average | 想运行但未拿到 CPU 的进程平均数，1/5/15 分钟。 |
| iowait | I/O Wait | 等待磁盘 I/O 的 CPU 时间占比。 |
| RSS | Resident Set Size | 进程实际占用的物理内存。 |
| VSZ | Virtual Memory Size | 进程的虚拟内存大小，含未使用部分。 |
| top | Top | 全屏实时进程监控工具。 |
| htop | Htop | 比 top 更现代的交互式进程查看器。 |

## C.6 软件包与磁盘存储

| 术语 | 英文 | 解释 |
| --- | --- | --- |
| APT | Advanced Package Tool | Debian 基础的高级包管理工具。 |
| dpkg | Debian Package | Debian 底层包管理工具，APT 构建其之上。 |
| 依赖地狱 | Dependency Hell | 软件包间依赖关系复杂导致安装困难的状况。 |
| 仓库 | Repository | 存放软件包的服务器目录，如 Debian 软件源。 |
| 挂载 | Mount | 将存储设备关联到文件系统目录的操作。 |
| fstab | File System Table | 开机自动挂载配置文件，`/etc/fstab`。 |
| UUID | Universally Unique Identifier | 分区或卷的全局唯一标识符，用于稳定挂载。 |
| 交换分区 | Swap | 磁盘上用作虚拟内存的空间，内存不足时使用。 |
| inode 耗尽 | Inode Exhaustion | inode 数量用完，无法新建文件尽管磁盘有空间。 |
| 循环设备 | Loop Device | 把普通文件当作块设备挂载，用于无痛磁盘实验。 |
| LVM | Logical Volume Manager | 逻辑卷管理器，PV/VG/LV 三层抽象。 |

## C.7 网络与 SSH

| 术语 | 英文 | 解释 |
| --- | --- | --- |
| IP 地址 | IP Address | 网络中设备的唯一标识。 |
| 子网掩码 | Subnet Mask | 用于划分网络位和主机位的掩码。 |
| 网关 | Gateway | 连接不同网络的路由器节点。 |
| DNS | Domain Name System | 域名系统，将域名解析为 IP 地址。 |
| CIDR | Classless Inter-Domain Routing | 无类别域间路由，如 `10.0.0.0/8`。 |
| 私有地址 | Private Address | 内网使用的保留 IP 段，如 `10.0.0.0/8`。 |
| 公有地址 | Public Address | 在互联网上唯一的 IP 地址。 |
| MAC 地址 | MAC Address | 网卡的物理地址，数据链路层标识。 |
| ICMP | Internet Control Message Protocol | 用于诊断的网络协议，ping 基于它。 |
| SSH | Secure Shell | 加密远程登录协议，默认端口 22。 |
| 密钥对 | Key Pair | 公钥 + 私钥，用于 SSH 免密登录。 |
| 指纹 | Fingerprint | SSH 服务器主机密钥的简短标识，防中间人攻击。 |
| known_hosts | Known Hosts | 记录已连接过的 SSH 主机指纹的文件。 |
| 端口转发 | Port Forwarding | 将一个端口的流量转发到另一个端口或主机。 |
| 隧道 | Tunnel | 通过已有连接封装另一种协议的通道。 |

## C.8 systemd 与服务管理

| 术语 | 英文 | 解释 |
| --- | --- | --- |
| systemd | Systemd | Linux 默认的 init 系统，PID 1，管理服务与启动。 |
| Unit | Unit | systemd 的配置单元，如 service、timer、socket。 |
| Service Unit | Service Unit | 定义一个后台服务的 unit 类型。 |
| Target | Target | systemd 的运行级别，如 multi-user.target。 |
| daemon-reload | Daemon Reload | 修改 unit 文件后通知 systemd 重新读取配置。 |
| journal | Journal | systemd 日志系统，由 journald 管理。 |
| journalctl | Journal Control | 查询 systemd 日志的命令行工具。 |
| cgroup | Control Group | 内核功能，用于限制和隔离进程组的资源使用。 |

## C.9 Shell 脚本与自动化

| 术语 | 英文 | 解释 |
| --- | --- | --- |
| Shebang | Shebang | 脚本首行 `#!` 指定解释器，如 `#!/bin/bash`。 |
| 位置参数 | Positional Parameter | 脚本传入的参数，`$1`、`$2`、`$@` 等。 |
| 环境变量 | Environment Variable | 子进程可继承的变量，用 `export` 设置。 |
| 退出码 | Exit Status | 脚本或命令结束返回的状态，0 成功、非 0 失败。 |
| trap | Trap | 捕获信号并执行指定命令，用于脚本清理。 |
| pipefail | Pipe Fail | 管道中任何命令失败则整个管道失败。 |
| cron | Cron | 定时任务守护进程，按 crontab 表达式执行命令。 |
| crontab | Cron Table | 定时任务配置文件，五段式：分 时 日 月 周。 |

## C.10 防火墙与备份

| 术语 | 英文 | 解释 |
| --- | --- | --- |
| 防火墙 | Firewall | 控制网络流量进出的安全系统。 |
| netfilter | Netfilter | Linux 内核的包过滤框架。 |
| iptables | IP Tables | 第二代防火墙前端工具。 |
| nftables | Nftables | 第三代防火墙前端，Debian 13 默认。 |
| ufw | Uncomplicated Firewall | 简单防火墙前端，封装 nftables/iptables。 |
| 默认拒绝 | Default Deny | 未明确允许的流量全部丢弃的策略。 |
| 暴露面 | Attack Surface | 系统可被攻击的接口总和，越小越安全。 |
| fail2ban | Fail2Ban | 读取日志并自动封禁恶意 IP 的工具。 |
| Jail | Jail | fail2ban 中针对特定服务的防护配置。 |
| 3-2-1 原则 | 3-2-1 Rule | 3 份副本、2 种介质、1 份异地的备份原则。 |
| 归档 | Archive | 把多个文件打包成一个文件，如 tar。 |
| 增量备份 | Incremental Backup | 只备份自上次备份以来变化的数据。 |
| 快照 | Snapshot | 整机状态的时点副本，InstaBox 提供此功能。 |
| 去重 | Deduplication | 相同数据只存一份，节省备份空间。 |
| restic | Restic | 现代加密去重备份工具。 |
| 恢复演练 | Recovery Drill | 实际执行备份恢复并验证数据完整性。 |

## C.11 InstaBox 云电脑专有

| 术语 | 英文 | 解释 |
| --- | --- | --- |
| InstaBox | InstaBox | 给 AI Agent 用的云电脑平台，官网 instabox.sh。 |
| 云电脑 | Cloud Computer | 远程运行的虚拟机器，可通过 SSH 或 CLI 访问。 |
| 快照 | Snapshot | InstaBox 整机状态的可恢复副本，免费档 5 个。 |
| 端口发布 | Port Publish | 将云电脑内端口映射为公网 HTTPS 链接。 |
| 端口转发 | Port Forwarding | 本地端口转发到云电脑端口，如 `port-forward`。 |
| 自动暂停 | Auto Suspend | 空闲 30 分钟后机器自动暂停，免费档锁定。 |
| 验货 | Inspection | 拿到机器后用 nproc/free/df 等命令核对配置。 |
| Firecracker | Firecracker | AWS 开源的轻量级微型虚拟机监控程序，InstaBox 基于它。 |
| microVM | MicroVM | 极小型虚拟机，启动快、隔离性强。 |
| agent-browser | Agent Browser | InstaBox 默认模板，含 Docker、Node.js、Python 等工具链。 |

