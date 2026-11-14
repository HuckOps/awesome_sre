---
title: "第 15 章 防火墙与数据备份"
---

# 第 15 章 防火墙与数据备份

> 前面几章我们把系统内部——用户、权限、进程、包、磁盘、网络——都捋了一遍。但一台服务器终究要暴露在网络里，暴露就意味着被攻击的风险；与此同时，磁盘会坏、人会手误、服务会被删。这一章讲两件防护性的工作：**防火墙**挡在门外，阻挡不必要的流量；**备份**挡在门内，确保坏了之后还能把数据捡回来。这两项都是 SRE 的“安全网”，没有它们，系统跑得再快，一次事故就可能清零。

> **本章目标**
> - 能解释防火墙的“默认拒绝、最小暴露面”原则，并用 ufw 在云电脑上配置一套可用的规则集。
> - 能说明 nftables / iptables 三代关系，并读懂 `nft list ruleset` 的输出。
> - 能用 fail2ban 为 sshd 配置防暴力破解 jail，并查看封禁状态。
> - 能说清 3-2-1 备份原则，并用 tar + rsync 写一套可验证的备份脚本，做一次真恢复演练。
> - 能区分 instabox 快照与文件备份，知道两者各自适合的场景。

## 15.1 防火墙为什么存在：默认拒绝与最小暴露面

防火墙（Firewall）的核心理念只有一句话：**默认拒绝一切，只放行明确允许的。**

把云电脑想象成一栋办公楼，防火墙就是门卫：没有门卫时，任何人可以走进任何房间；有了门卫，只有登记在册、且目的明确的人才能进对应的房间。SRE 在运维时常说“暴露面（Attack Surface）”——一台机器开放的端口越多、允许的来源IP范围越宽，暴露面越大，被攻击的概率就越高。最小暴露面原则就是：只开业务必须的端口，只允许必须的来源，其余全部丢弃。

> **SRE 贴士**：防火墙不是“装上就万事大吉”。规则写错，自己会被关在门外；规则太宽，攻击者有机可乥。改防火墙规则时，永远保留一条已知可用的出口（比如你当前的SSH会话），不要在还没验证新规则前删掉旧规则。

### 15.1.1 三代防火墙：netfilter / iptables / nftables

Linux 内核里真正做包过滤的是 **netfilter**，它是内核的一套框架，不直接面向用户。用户通过前端工具来写规则，历史上经历了三代：

| 代际 | 工具 | 特点 | 现状 |
| --- | --- | --- | --- |
| 第一代 | ipchains | 早期 Linux 无状态防火墙 | 已淘汰 |
| 第二代 | iptables | 有状态过滤，链/表/规则概念，广泛使用 | 仍在维护，新发行版逐步退出 |
| 第三代 | nftables | 取代 iptables，语法更统一，性能更好 | Debian 13 默认前端 |

Debian 13 默认安装的是 **nftables**。`iptables` 命令在 nftables 兼容层下仍可用，但底层已走 nftables。本书不深入 iptables 古老语法，重点讲 ufw（简单易用）和 nft 基础（看懂规则）。

> **SRE 贴士**：生产服务器上不要同时用 ufw 和 iptables/nft 手写规则，两套工具互相覆盖会导致规则冲突、排障困难。选一种、统一种。

### 15.1.2 ufw：最简单的防火墙前端

ufw（Uncomplicated Firewall）是 Ubuntu / Debian 上最常用的防火墙管理工具，它把复杂的 nftables 规则封装成几条好记的命令。

**安装**：

```bash
apt install -y ufw
```

**查看当前状态**：

```bash
ufw status
```

```text
Status: inactive
```

inactive 表示防火墙尚未启用，所有端口都是开放的（对内网来说是危险的，对云电脑来说平台层已有安全组，但实例内建议仍配置）。

**基础策略：默认拒绝入站，默认允许出站**：

```bash
ufw default deny incoming
ufw default allow outgoing
```

**放行 SSH（22/tcp）**：

```bash
ufw allow OpenSSH
```

**放行 HTTP（80/tcp）**：

```bash
ufw allow 80/tcp
```

**只允许特定网段访问**（例如只允许 10.0.0.0/8 内网访问数据库端口 3306）：

```bash
ufw allow from 10.0.0.0/8 to any port 3306
```

**限制 SSH 连接频率**（防暴力破解，每 30 秒最多 6 次连接）：

```bash
ufw limit OpenSSH
```

**启用防火墙**：

```bash
ufw enable
```

启用时会提示命令可能中断现有 SSH 连接，输入 `y` 确认。启用后查看：

```bash
ufw status verbose
```

```text
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    Anywhere
80/tcp                     ALLOW IN    Anywhere
22/tcp (v6)                ALLOW IN    Anywhere (v6)
80/tcp (v6)                ALLOW IN    Anywhere (v6)
```

**删除一条规则**：

```bash
ufw delete allow 80/tcp
```

**禁用防火墙**（恢复全部开放）：

```bash
ufw disable
```

### 15.1.3 nft 基础：读懂底层规则

虽然日常用 ufw，但懂一点 nftables 能帮你读懂系统里到底生效了什么规则。

**列出当前所有规则集**：

```bash
nft list ruleset
```

输出节选：

```text
table inet filter {
    chain input {
        type filter hook input priority 0; policy drop;
        iifname "lo" accept
        ct state established,related accept
        ip protocol icmp accept
        udp dport 53 accept
        tcp dport 22 accept
        tcp dport 80 accept
    }
    chain forward {
        type filter hook forward priority 0; policy drop;
    }
    chain output {
        type filter hook output priority 0; policy accept;
    }
}
```

逐段解读：

- `table inet filter`：一张过滤表，inet 表示同时处理 IPv4 和 IPv6。
- `chain input`：入站链，所有发给本机的包都经过这里。
- `policy drop`：默认策略是丢弃，不匹配任何规则的包直接扔掉。
- `iifname "lo" accept`：回环接口（lo）上的包放行，本机自己跟自己通信不受限。
- `ct state established,related accept`：已经建立的连接和关联的包放行，这是有状态防火墙的关键——你主动发出的请求，回包会被自动放行。
- `tcp dport 22 accept`：目的端口 22（SSH）的包放行。
- `tcp dport 80 accept`：目的端口 80（HTTP）的包放行。

> **警告**：**不要在只有 SSH 连接的机器上执行 `ufw default deny incoming` 而不先放行 22/tcp。** 规则生效后，你的 SSH 会话会立即断开，且无法重新连接，除非你有平台层的控制台（如 instabox 的网页 Computer 标签）能进机器修正。正确顺序是：先 `ufw allow OpenSSH`，再 `ufw default deny incoming`，最后 `ufw enable`。

> **SRE 贴士**：InstaBox 云电脑的 `instabox ssh` 走平台隧道，不是直接暴露的 22 端口。这意味着你在云电脑内部配 ufw 拦 SSH 端口，不会影响 `instabox ssh book` 的连接；但如果你在云电脑上装了 nginx 并监听 80 端口，想通过公网访问，需要同时：(1) ufw 放行 80/tcp；(2) `instabox port publish book 80` 把端口发布成公网 HTTPS 链接。缺一不可。

## 15.2 fail2ban：防暴力破解的第二道防线

防火墙按端口放行，但放行的端口上仍可能有人暴力尝试密码。fail2ban（Fail2Ban）读取日志，发现某 IP 频繁失败登录，就自动把该 IP 加入防火墙黑名单封禁一段时间。

**安装**：

```bash
apt install -y fail2ban
```

fail2ban 安装后自带默认配置 `/etc/fail2ban/jail.conf`，**不要直接改这个文件**（升级会被覆盖），正确做法是复制一份 `jail.local` 并只改你需要的部分：

```bash
cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

编辑 `jail.local`，找到 `[sshd]` 段，确保如下内容：

```ini
[sshd]
enabled = true
port = ssh
logpath = %(sshd_log)s
backend = %(sshd_backend)s
bantime = 1h
findtime = 10m
maxretry = 5
```

逐项解释：

- `enabled = true`：启用该 jail。
- `port = ssh`：监控 SSH 端口。
- `logpath`：读取 sshd 日志路径（Debian 通常是 `/var/log/auth.log`）。
- `bantime = 1h`：封禁该 IP 1 小时。
- `findtime = 10m`：在 10 分钟内。
- `maxretry = 5`：失败达到 5 次，触发封禁。

**重启 fail2ban 使配置生效**：

```bash
systemctl restart fail2ban
```

**查看状态**：

```bash
fail2ban-client status
```

```text
Status
|- Number of jail:	1
`- Jail list:	sshd
```

```bash
fail2ban-client status sshd
```

```text
Status for the jail: sshd
|- Filter
|  |- Currently failed:	0
|  |- Total failed:	0
|  `- File list:	/var/log/auth.log
`- Actions
   |- Currently banned:	0
   |- Total banned:	0
   `- Banned IP list:
```

> **SRE 贴士**：fail2ban 封禁是通过修改防火墙规则实现的（通常往 iptables/nftables 插入一条 DROP 规则）。如果你同时手动管理 ufw/nft 规则，fail2ban 插入的封禁规则可能和你的手动规则冲突。小规模环境下让 fail2ban 自行处理即可；大规模环境建议统一用配置管理工具下发防火墙策略。

## 15.3 备份：3-2-1 原则与 tar / rsync

防火墙挡在门外，备份挡在门内。无论防火墙多严，硬件故障、误操作、勒索软件都可能导致数据丢失。备份是最后一道防线。

### 15.3.1 3-2-1 备份原则

**3-2-1 原则**是备份领域的经典经验法则：

- **3** 份数据副本：1 份生产数据 + 2 份备份。
- **2** 种不同介质：比如一份在本地磁盘，一份在云存储或另一台机器。
- **1** 份异地副本：至少一份备份放在不同物理位置，防止机房火灾、水灾等场域级灾难。

对个人开发者和小团队，做到“本地副本 + 云电脑快照 + 异地归档”已足够应对大部分事故。

### 15.3.2 tar：打包与压缩

tar（Tape ARchive）是 Linux 最常用的打包工具，它本身只负责“把多个文件归档成一个文件”，压缩由 `-z`（gzip）、`-J`（xz）、`--zstd` 等选项完成。

常用选项：

| 选项 | 含义 |
| --- | --- |
| `-c` | create，创建新归档 |
| `-x` | extract，解包 |
| `-t` | list，列出归档内容不解包 |
| `-z` | 用 gzip 压缩/解压（后缀通常 `.tar.gz` 或 `.tgz`） |
| `-J` | 用 xz 压缩/解压（后缀 `.tar.xz`，压缩率更高但慢） |
| `--zstd` | 用 zstd 压缩（速度快，现代系统推荐） |
| `-v` | verbose，显示处理的文件 |
| `-f` | file，指定归档文件名（必须紧跟文件名） |
| `-C` | 切换到指定目录后再操作 |
| `--exclude` | 排除指定文件/目录 |

**打包 /data/home 到 /data/backup**：

```bash
mkdir -p /data/backup
tar -czvf /data/backup/home-$(date +%F).tar.gz -C /data home
```

输出示例：

```text
home/
home/bookuser/
home/bookuser/.bashrc
home/bookuser/.profile
...
tar: Removing leading `/' from member names
```

**列出归档内容（不解压）**：

```bash
tar -tzf /data/backup/home-2026-11-14.tar.gz | head -20
```

**解包到指定目录**：

```bash
tar -xzvf /data/backup/home-2026-11-14.tar.gz -C /tmp/restore-test
```

### 15.3.3 rsync：增量同步与远程备份

tar 每次都是全量打包，适合做定期快照；rsync（remote sync）适合做增量同步——只传输变化的部分，速度快、带宽省。

常用选项：

| 选项 | 含义 |
| --- | --- |
| `-a` | archive，归档模式（保留权限、时间戳、符号链接等，等价于 `-rlptgoD`） |
| `-v` | verbose，显示传输文件 |
| `-z` | 压缩传输 |
| `--progress` | 显示进度 |
| `-n` | dry-run，只演示不执行（改动前先看一遍） |
| `--delete` | 删除目标中源已不存在的文件（保持目标与源一致） |
| `--exclude` | 排除指定模式 |

**本地增量备份（演练用）**：

```bash
rsync -av --delete /data/home/ /data/backup/home-sync/
```

首次运行会拷贝全部文件，第二次只拷贝变化的。

**模拟远程备份**（云电脑内部两个目录间，真实场景可替换为远程主机）：

```bash
rsync -avz --progress -n /data/home/ bookuser@127.0.0.1:/data/home/backup/
```

`-n` 参数让 rsync 只展示会传输什么，不实际传输。确认无误后去掉 `-n` 执行真传输。

> **警告**：`rsync --delete` 会删除目标目录中源已不存在的文件。如果源路径写错（比如把 `/data/home/` 写成 `/data/home` 少了斜杠，或者把源和目标搞反），可能误删除重要数据。执行前务必先用 `-n` 干跑一遍，核对输出文件列表再去掉 `-n`。

## 15.4 instabox 快照 vs 文件备份

InstaBox 云电脑提供“快照（snapshot）”功能，很多新手会把快照当备份。两者有本质区别：

| 比较项 | instabox 快照 | 文件备份（tar/rsync） |
| --- | --- | --- |
| 粒度 | 整机（系统盘 + /data 持久盘） | 文件/目录级别 |
| 恢复速度 | 快，秒级回到快照时的完整现场 | 较慢，需重新解包/同步文件 |
| 存储开销 | 大，每个快照占完整磁盘镜像空间 | 小，可增量、可压缩 |
| 跨机恢复 | 只能在 instabox 平台恢复 | 可拷贝到任意机器恢复 |
| 操作影响 | 保存前需停机，只存环境不存运行中进程内存 | 可在线进行，不影响运行 |
| 适合场景 | 系统级重大变更前（如升级内核、安装复杂软件栈） | 日常数据、代码、配置文件 |

**创建快照**：

```bash
instabox snapshot save book --name base
```

**从快照创建新机**：

```bash
instabox create --name job2 --snapshot base
```

快照的 5 个名额和 200 GB 总量上限意味着：不要每天都快照，适合在重大操作前做一次基准快照（base），日常数据靠 tar/rsync 备份。

> **SRE 贴士**：快照不是备份，备份也不是快照。快照适合“系统出问题时整机回滚”，备份适合“单个文件误删后找回”。生产环境两者都要有，且**必须定期做恢复演练**——没验证过的恢复等于没备份。

## 15.5 restic / borg 概念科普（加密去重备份）

如果你需要更专业的备份方案，restic 和 borg 是两款现代备份工具，核心特性是：

- **加密**：备份内容加密后再上传，存储提供商无法读取。
- **去重（Deduplication）**：相同的数据块只存一份，多次备份只增量变化部分，节省空间。
- **多目标**：可备份到本地目录、SFTP、S3、REST 服务器等。

安装 restic（Debian 13 软件源可用）：

```bash
apt install -y restic
```

初始化一个本地备份仓库：

```bash
restic init --repo /data/backup/restic-repo
```

备份 /data/home：

```bash
restic backup /data/home --repo /data/backup/restic-repo
```

列出快照：

```bash
restic snapshots --repo /data/backup/restic-repo
```

这两款工具配合 cron 即可形成一套自动化、加密、去重的备份系统。本书不展开细节，知道有这类工具、会找文档即可。

## 15.6 备份脚本：自动化执行与验证

复用第 13 章《Shell 脚本入门》的脚本知识，写一个每日运行的备份脚本，把 /data/home 打包打包到 /data/backup，并保留最近 7 天。

```bash
cat > /data/home/backup.sh << 'EOF'
#!/bin/bash
set -euo pipefail

BACKUP_DIR="/data/backup"
SOURCE_DIR="/data/home"
DATE=$(date +%F)
ARCHIVE="${BACKUP_DIR}/home-${DATE}.tar.gz"

mkdir -p "${BACKUP_DIR}"

# 打包（排除已有的备份文件，避免递归包含）
tar -czvf "${ARCHIVE}" -C /data home --exclude='home/backup'

# 保留最近 7 天，旧文件删除
find "${BACKUP_DIR}" -name "home-*.tar.gz" -mtime +7 -delete

echo "Backup completed: ${ARCHIVE}"
EOF

chmod +x /data/home/backup.sh
```

手动运行一次验证：

```bash
/data/home/backup.sh
```

```text
home/
home/bookuser/
home/bookuser/.bashrc
...
Backup completed: /data/backup/home-2026-11-14.tar.gz
```

**做一次真恢复演练**：把归档解包到临时目录，核对关键文件是否完整：

```bash
mkdir -p /tmp/restore-test
tar -xzvf /data/backup/home-2026-11-14.tar.gz -C /tmp/restore-test
ls -l /tmp/restore-test/home/bookuser/.bashrc
```

如果 `.bashrc` 能正常列出且内容正确，说明备份可用。

> **警告**：备份脚本里的 `set -euo pipefail` 和 `find ... -delete` 组合比较强力。`find -mtime +7 -delete` 会无条件删除 7 天前的归档，如果脚本路径或变量写错，可能误删非备份目录。测试阶段先把 `-delete` 改成 `-ls` 看一眼会删哪些文件，确认无误再恢复 `-delete`。

## 15.7 定时备份与 cron（预告）

第 14 章《定时任务与系统日志》会详细讲 cron。这里只给一个 crontab 示例，每天凌晨 3 点运行备份脚本：

```bash
crontab -e
```

写入：

```cron
0 3 * * * /data/home/backup.sh >> /data/home/logs/backup.log 2>&1
```

> **SRE 贴士**：cron 任务的输出默认会尝试发邮件给本地用户，而大多数服务器没有配置邮件系统，导致输出丢失。生产环境的 cron 任务必须把 stdout/stderr 重定向到日志文件（如上例的 `>> ... 2>&1`），否则排障时一片空白。

## 本章小结

| 知识点 | 关键命令/概念 |
| --- | --- |
| 默认拒绝策略 | `ufw default deny incoming` |
| 放行 SSH / HTTP | `ufw allow OpenSSH` / `ufw allow 80/tcp` |
| 查看底层规则 | `nft list ruleset` |
| 防暴力破解 | fail2ban + `jail.local` `[sshd]` |
| 3-2-1 备份原则 | 3 份副本、2 种介质、1 份异地 |
| 打包归档 | `tar -czvf 文件.tar.gz -C 目录 源` |
| 增量同步 | `rsync -av --delete 源/ 目标/` |
| 快照 vs 备份 | instabox snapshot 是整机；tar/rsync 是文件级 |
| 加密去重备份 | restic / borg |
| 恢复验证 | `tar -xzvf 归档 -C 测试目录`，核对关键文件 |

## 动手练习

1. 在云电脑上安装 ufw，配置“默认拒绝入站、默认允许出站”，放行 SSH（22/tcp）和 HTTP（80/tcp），启用防火墙并用 `ufw status verbose` 确认生效。
2. 用 `nft list ruleset` 查看 ufw 生成的底层规则，找出 `policy drop` 和 `tcp dport 22 accept` 两行。
3. 安装 fail2ban，复制 `jail.conf` 为 `jail.local`，开启 `[sshd]`，设置 `bantime = 1h`、`maxretry = 5`，重启后用 `fail2ban-client status sshd` 查看。
4. 用 tar 把 /data/home 打包为带日期的 `.tar.gz` 归档，保存到 /data/backup，再用 `tar -tzf` 列出前 20 个文件。
5. 写一个备份脚本 `/data/home/backup.sh`（含 `set -euo pipefail`、7 天过期清理），手动运行一次，然后做一次真恢复演练：解包到 `/tmp/restore-test` 并核对 `.bashrc` 存在。

> **参考答案**：
> 1. `apt install -y ufw && ufw default deny incoming && ufw default allow outgoing && ufw allow OpenSSH && ufw allow 80/tcp && ufw enable`，然后 `ufw status verbose` 应显示 `Status: active` 且 `Default: deny (incoming), allow (outgoing)`。
> 2. `nft list ruleset` 输出中应有 `table inet filter`、`chain input`、`policy drop;` 以及 `tcp dport 22 accept`。
> 3. `cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local`，改 `[sshd]` 段，`systemctl restart fail2ban`，`fail2ban-client status sshd` 显示 Jail: sshd。
> 4. `tar -czvf /data/backup/home-$(date +%F).tar.gz -C /data home`，`tar -tzf /data/backup/home-*.tar.gz \| head -20` 列出 home/ 开头的文件。
> 5. 脚本如 15.6 节所示；运行后 `mkdir -p /tmp/restore-test && tar -xzvf /data/backup/home-*.tar.gz -C /tmp/restore-test && ls /tmp/restore-test/home/bookuser/.bashrc` 应能看到文件。
