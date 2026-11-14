---
title: "第 16 章 综合实战：从零部署并加固一个 Web 服务"
---

# 第 16 章 综合实战：从零部署并加固一个 Web 服务

> 前面十五章把 Linux 的基础命令、用户权限、文本处理、进程、包管理、磁盘、网络、systemd、Shell 脚本、定时任务、防火墙与备份都讲了一遍。但知识点散在各章里，缺乏一条贯通的实战主线。本章是卷 1 的收官之作：我们要在你的 InstaBox 云电脑上，从零开始部署一个面向外网的 Web 服务，并把它加固到能在生产环境勉强使用的程度。完成本章后，你会拥有一份可运行的、可备份、可监控的 Web 服务，以及一套可复用的部署与运维方法论。

> **本章目标**
> - 能在云电脑上安装 nginx、openssh-server、ufw、fail2ban、rsync，并完成基础系统盘点。
> - 能创建普通用户 bookuser、建立 /srv/www 站点目录，并按最小权限设置属主属组与 755/644 权限。
> - 能编写 index.html 与 health.json，通过 nginx 的 sites-available/sites-enabled 配置站点，用 `nginx -t` 验证语法并 `systemctl reload` 生效。
> - 能用 `instabox port publish book 80` 发布公网 HTTPS 链接，并在 ufw 中放行 80/tcp，实现外网访问。
> - 能做加固：ufw 默认拒绝入站、fail2ban 防 sshd 暴力破解、`server_tokens off` 隐藏版本、`autoindex off` 关闭目录浏览。
> - 能写一个每 5 分钟跑一次的健康检查脚本，把结果落盘到 /data/home/logs，并用 journalctl 查看 nginx 日志。
> - 能用 tar + rsync 做定时备份，做一次真恢复演练，并用 instabox snapshot 做整机基准快照。
> - 能完成验收 checklist，并对常见故障（80 被占、403 权限、配置语法错、防火墙未放行、机器被暂停）进行排查。

## 16.1 需求场景与整体架构

### 16.1.1 要落地的业务

假设你要为一个 3 人小团队落一个简单的 Web 服务，包含两部分内容：

1. **静态官网首页**：访问 `/` 返回一段 HTML，展示团队名称、成员、联系方式。
2. **健康检查 API**：访问 `/health` 返回 JSON `{"status":"ok","service":"book-web"}`，供监控系统或负载均衡器探测服务存活。

这两部分都不需要数据库，不需要后端语言，纯静态文件 + nginx 即可跑通。这是 SRE 最常见的“最小可用服务”模式，也是更复杂应用（带数据库、带缓存、带容器）的基础。

### 16.1.2 架构图

读者 → 端口发布/HTTPS 链接 → 云电脑 nginx → /srv/www 静态目录

用 ASCII 图画出来：

```text
+------------+       HTTPS 链接（instabox port publish）       +-------------------+
|  读者浏览器 | ----------------------------------------------> |  InstaBox 云电脑  |
+------------+                                                  |  (Debian 13)    |
                                                                 |                 |
                                   +--------------+              |  nginx (80/tcp) |
                                   |   ufw/fail2  |------------>|  /srv/www       |
                                   |   ban 防护   |              |  index.html     |
                                   +--------------+              |  health.json    |
                                                                 +-------------------+
```

### 16.1.3 需要准备的软件包

| 包 | 用途 |
| --- | --- |
| nginx | Web 服务器，提供静态文件与健康检查 API |
| openssh-server | SSH 服务（instabox 平台隧道，实例内也可用） |
| ufw | 实例内防火墙，做最小暴露面 |
| fail2ban | 防 sshd 暴力破解 |
| rsync | 增量备份与同步 |
| curl | 健康检查与手工探测 |

## 16.2 步骤 1：装机与盘点

在云电脑里（以 root 身份，`instabox ssh book` 进入后），先更新包索引并安装所需软件：

```bash
apt update
apt install -y nginx openssh-server ufw fail2ban rsync curl
```

安装完毕后做盘点，确认各服务状态：

```bash
systemctl status nginx --no-pager
```

```text
● nginx.service - A high performance web server and a reverse proxy server
     Loaded: loaded (/lib/systemd/system/nginx.service; enabled; preset: enabled)
     Active: active (running) since ...
```

```bash
systemctl status ssh --no-pager
```

```text
● ssh.service - OpenBSD Secure Shell server
     Loaded: loaded (/lib/systemd/system/ssh.service; enabled; preset: enabled)
     Active: active (running) since ...
```

```bash
ufw status
```

```text
Status: inactive
```

> **SRE 贴士**：软件装完不要以为“跑起来就完事”。生产环境每装一个包都要问：这个包是必须的吗？它自启动了吗？它监听什么端口？它有默认密码或默认配置吗？带着这些问题装机，能避免大量后续安全隐患。

## 16.3 步骤 2：账号与目录

Web 服务不应该用 root 跑。创建专门的普通用户 bookuser，建立站点目录 /srv/www，按最小权限分配。

**创建用户**（参考第 5 章《用户、群组与权限》）：

```bash
useradd -m -s /bin/bash bookuser
usermod -aG sudo bookuser
```

**建立站点目录**：

```bash
mkdir -p /srv/www
chown -R bookuser:bookuser /srv/www
chmod 755 /srv/www
```

验证：

```bash
ls -ld /srv/www
```

```text
drwxr-xr-x 2 bookuser bookuser 4096 ... /srv/www
```

目录权限 755（属主可读写执行，属组和其他人可读可进），文件里的 HTML 会设 644。

> **SRE 贴士**：/srv 是 FHS 规范里专门用于“服务数据”的目录（Service data），把网站内容放在这里比放在 /root 或 /tmp 更符合规范，也更容易被备份脚本找到。详见第 3 章 FHS 目录树介绍。

## 16.4 步骤 3：站点内容

以 bookuser 身份在 /srv/www 下写两个文件。先切到 bookuser：

```bash
su - bookuser
```

**写 index.html**：

```bash
cat > /srv/www/index.html << 'EOF'
<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<title>Book Team 官网</title>
</head>
<body>
<h1>欢迎访问 Book Team 官网</h1>
<p>这是一个基于 nginx + InstaBox 云电脑部署的静态站点。</p>
<p>成员：Alice、Bob、Charlie</p>
<p>联系方式：sre@example.com</p>
</body>
</html>
EOF
```

**写 health.json**：

```bash
cat > /srv/www/health.json << 'EOF'
{"status":"ok","service":"book-web","time":"2026-11-14"}
EOF
```

退回 root（为了后续给 nginx 配置目录权限）：

```bash
exit
```

给文件设置合适权限：

```bash
chmod 644 /srv/www/index.html /srv/www/health.json
ls -l /srv/www
```

```text
total 8
-rw-r--r-- 1 bookuser bookuser 432 ... index.html
-rw-r--r-- 1 bookuser bookuser  63 ... health.json
```

> **警告**：不要把站点目录或文件设成 777 方便“免去权限问题”。777 意味着云电脑上的任何本地用户（包括潜在的恶意进程）都能修改你的网页，植入恶意脚本。生产环境 777 是审查重灾区。

## 16.5 步骤 4：nginx 配置

Debian 13 的 nginx 默认站点配置在 `/etc/nginx/sites-available/`，启用时在 `/etc/nginx/sites-enabled/` 做符号链接。

**创建站点配置文件**：

```bash
cat > /etc/nginx/sites-available/book.conf << 'EOF'
server {
    listen 80;
    server_name _;

    root /srv/www;
    index index.html;

    location = /health {
        default_type application/json;
        return 200 '{"status":"ok","service":"book-web"}';
    }

    location / {
        try_files $uri $uri/ =404;
        autoindex off;
    }

    server_tokens off;
}
EOF
```

逐项解读：

- `listen 80`：监听 80 端口 HTTP。
- `server_name _`：捕-all 主机名，适合只有一个站点的场景。
- `root /srv/www`：静态文件根目录，即第 16.3 节建的目录。
- `index index.html`：访问 `/` 时默认返回 index.html。
- `location = /health`：精确匹配 `/health`，直接返回 JSON，不读文件系统，作为健康检查 API。
- `try_files $uri $uri/ =404`：尝试按 URI 找文件，找不到返回 404。
- `autoindex off`：关闭目录浏览，避免访问没有 index.html 的目录时泄露文件列表。
- `server_tokens off`：响应头里不带 nginx 版本号，减少版本信息泄露。

**启用站点**：在 sites-enabled 做符号链接（参考第 4 章 ln 软链接）：

```bash
ln -s /etc/nginx/sites-available/book.conf /etc/nginx/sites-enabled/book.conf
```

**处理默认站点**：Debian 默认安装会带一个 `default` 站点，它也监听 80，会和 book.conf 冲突。先禁用它：

```bash
rm -f /etc/nginx/sites-enabled/default
```

**验证配置语法**：

```bash
nginx -t
```

```text
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf is tested successfully
```

**重载 nginx 生效**：

```bash
systemctl reload nginx
```

`reload` 比 `restart` 更温和，已有连接不中断，适合生产环境配置变更。

手工验证站点是否可达（本机）：

```bash
curl -I http://127.0.0.1/
```

```text
HTTP/1.1 200 OK
Server: nginx
Date: ...
Content-Type: text/html
Content-Length: 432
...
```

```bash
curl http://127.0.0.1/health
```

```text
{"status":"ok","service":"book-web"}
```

注意：`server_tokens off` 后，响应头里的 `Server: nginx` 不再带版本号。

> **SRE 贴士**：每次改 nginx 配置，先 `nginx -t` 再 `systemctl reload`。`nginx -t` 只检查语法，不重启服务；如果语法错了，`reload` 会失败导致旧配置继续跑，新配置不生效，但至少不会把服务搞挂。直接 `systemctl restart` 在配置有误时会导致服务无法启动，造成停机。

## 16.6 步骤 5：暴露服务

云电脑内部 nginx 已经在 80 端口监听，但外网读者访问不到，还需要两步：平台侧发布端口，实例侧防火墙放行。

### 16.6.1 平台侧：instabox port publish

```bash
instabox port publish book 80
```

输出示例：

```text
Published port 80 on book -> https://book-xxx.instabox.sh
```

这个 HTTPS 链接就是外网访问入口。读者在浏览器打开这个链接，就能访问云电脑上 nginx 提供的站点。

> **SRE 贴士**：InstaBox 免费档每台机器 16 个公网端口，80 端口占用其中一个。不要随意 `port publish` 不需要的端口，每开一个端口就是一条潜在的攻击路径。用完记得 `instabox port unpublish book 80` 回收。

### 16.6.2 实例侧：ufw 放行 80/tcp

```bash
ufw allow 80/tcp
ufw enable
```

启用时输入 `y` 确认。然后验证：

```bash
ufw status verbose
```

应显示 80/tcp 和 22/tcp（OpenSSH）都在 ALLOW IN 列表，默认策略是 deny incoming / allow outgoing。

外网读者打开 `https://book-xxx.instabox.sh`，应能看到官网首页；打开 `https://book-xxx.instabox.sh/health`，应能看到 JSON。

> **警告**：启用 ufw 前务必先放行 OpenSSH（22/tcp）。如果先 `ufw default deny incoming` 再 `ufw enable` 而不放行 22，你的 `instabox ssh` 平台隧道虽不受影响（走平台通道），但若后续需要直接 SSH 到实例内（如卷 2 的 VPS 场景），会被挡在外面。养成“先放 SSH，再开防火墙”的肌肉记忆。

## 16.7 步骤 6：加固

站点能访问后，再做加固，把服务从“能用”提升到“较安全”。

### 16.7.1 ufw 默认拒绝 + 最小放行

```bash
ufw default deny incoming
ufw default allow outgoing
ufw allow OpenSSH
ufw allow 80/tcp
ufw enable
```

此时只暴露 22（SSH）和 80（HTTP），其余端口全部丢弃。

### 16.7.2 fail2ban 防 sshd 暴力破解

```bash
cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

编辑 jail.local，确保 `[sshd]` 段：

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

```bash
systemctl restart fail2ban
fail2ban-client status sshd
```

### 16.7.3 nginx 隐藏版本与关闭目录浏览

book.conf 已写入 `server_tokens off` 和 `autoindex off`，`systemctl reload nginx` 后生效。验证响应头不含版本：

```bash
curl -I http://127.0.0.1/ | grep -i server
```

```text
Server: nginx
```

（没有版本数字，说明 `server_tokens off` 生效。）

## 16.8 步骤 7：监控与日志

### 16.8.1 查看 nginx 日志

nginx 默认日志在 `/var/log/nginx/`：

```bash
ls -l /var/log/nginx/
```

```text
access.log  error.log
```

实时跟踪访问日志：

```bash
tail -f /var/log/nginx/access.log
```

### 16.8.2 journalctl 查看系统服务日志

```bash
journalctl -u nginx -n 50 --no-pager
```

查看 sshd 登录日志（配合 fail2ban）：

```bash
journalctl -u ssh -n 30 --no-pager
```

### 16.8.3 健康检查脚本 + cron

写一个每 5 分钟跑一次的检查脚本，把结果落盘到 /data/home/logs：

```bash
mkdir -p /data/home/logs
cat > /data/home/check.sh << 'EOF'
#!/bin/bash
set -u

LOG_DIR="/data/home/logs"
LOG_FILE="${LOG_DIR}/health-$(date +%F).log"
TIMESTAMP=$(date '+%F %T')

HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1/health 2>/dev/null || echo "000")

if [ "$HTTP_CODE" = "200" ]; then
    echo "[${TIMESTAMP}] health check OK (HTTP 200)" >> "${LOG_FILE}"
else
    echo "[${TIMESTAMP}] health check FAILED (HTTP ${HTTP_CODE})" >> "${LOG_FILE}"
fi
EOF

chmod +x /data/home/check.sh
```

手动运行一次验证：

```bash
/data/home/check.sh
cat /data/home/logs/health-$(date +%F).log
```

```text
[2026-11-14 14:30:00] health check OK (HTTP 200)
```

加入 crontab，每 5 分钟运行：

```bash
(crontab -l 2>/dev/null; echo "*/5 * * * * /data/home/check.sh >> /data/home/logs/cron.log 2>&1") | crontab -
```

查看：

```bash
crontab -l
```

> **SRE 贴士**：cron 任务必须把输出重定向到日志文件（`>> ... 2>&1`），否则输出默认尝试发邮件给本地用户，而多数服务器没配邮件，排障时一片空白。详见第 14 章《定时任务与系统日志》。

## 16.9 步骤 8：备份

站点内容、nginx 配置、脚本都需要备份。用 tar + rsync 做一套可验证的备份。

**备份脚本**（复用第 13 章脚本知识，扩展到多目录）：

```bash
cat > /data/home/backup.sh << 'EOF'
#!/bin/bash
set -euo pipefail

BACKUP_DIR="/data/backup"
SOURCE_DIRS="/srv/www /etc/nginx /data/home/check.sh /data/home/backup.sh"
DATE=$(date +%F)
ARCHIVE="${BACKUP_DIR}/web-backup-${DATE}.tar.gz"

mkdir -p "${BACKUP_DIR}"

tar -czvf "${ARCHIVE}" -C / ${SOURCE_DIRS#/ } 2>/dev/null || true

# 保留最近 7 天
find "${BACKUP_DIR}" -name "web-backup-*.tar.gz" -mtime +7 -delete

echo "Backup completed: ${ARCHIVE}"
EOF

chmod +x /data/home/backup.sh
```

> **警告**：`find ... -mtime +7 -delete` 会无条件删除 7 天前的归档。测试阶段先把 `-delete` 换成 `-ls`，确认只会删旧备份再恢复 `-delete`。

手动运行一次：

```bash
/data/home/backup.sh
```

```text
srv/www/
srv/www/index.html
srv/www/health.json
etc/nginx/
etc/nginx/sites-available/book.conf
...
Backup completed: /data/backup/web-backup-2026-11-14.tar.gz
```

**做一次真恢复演练**：

```bash
mkdir -p /tmp/restore-test
tar -xzvf /data/backup/web-backup-2026-11-14.tar.gz -C /tmp/restore-test
ls -l /tmp/restore-test/srv/www/index.html
```

能列出 index.html 且内容正确，说明备份可用。

**加入 crontab，每天凌晨 3 点备份**：

```bash
(crontab -l 2>/dev/null; echo "0 3 * * * /data/home/backup.sh >> /data/home/logs/backup.log 2>&1") | crontab -
```

**instabox 快照做整机基准**：在所有配置做完、服务跑通后，存一次基准快照：

```bash
instabox snapshot save book --name base
```

以后做重大变更（如升级内核、安装复杂软件栈）前，再存一次快照；出问题可用 `instabox create --name job2 --snapshot base` 从基准恢复整机。

> **SRE 贴士**：快照不是备份，备份也不是快照。快照适合整机回滚，备份适合单文件找回。生产环境两者都要有，且必须定期做恢复演练——没验证过的恢复等于没备份。

## 16.10 验收 checklist

部署完成后，按此清单自查：

- [ ] `curl -I http://127.0.0.1/` 返回 HTTP 200
- [ ] `curl http://127.0.0.1/health` 返回 JSON `{"status":"ok",...}`
- [ ] `curl -I http://127.0.0.1/ | grep -i server` 不含版本号（server_tokens off）
- [ ] `ufw status verbose` 显示 `Status: active`，仅放行 22/tcp 和 80/tcp
- [ ] `fail2ban-client status sshd` 显示 Jail: sshd
- [ ] `ls -ld /srv/www` 权限为 755，属主 bookuser
- [ ] `ls -l /srv/www/*.html *.json` 权限为 644
- [ ] `systemctl is-active nginx` 返回 active
- [ ] `systemctl is-active ssh` 返回 active
- [ ] `crontab -l` 包含 check.sh 和 backup.sh 两条任务
- [ ] `/data/backup/` 下有最新的 .tar.gz 归档
- [ ] 从外网打开 `instabox port publish` 生成的 HTTPS 链接，能看到首页
- [ ] 做过一次 tar 归档的真恢复演练，关键文件完整

## 16.11 故障排查表

| 现象 | 可能原因 | 排查命令 | 解决 |
| --- | --- | --- | --- |
| 外网无法访问 80 端口 | ufw 未放行或未启用 | `ufw status verbose` | `ufw allow 80/tcp && ufw enable` |
| 外网无法访问 | 未 `instabox port publish` | `instabox port list book` | `instabox port publish book 80` |
| 浏览器打开 403 Forbidden | /srv/www 权限不对或 nginx 用户无读权 | `ls -ld /srv/www`、`ls -l /srv/www/*` | `chown -R bookuser:bookuser /srv/www && chmod 755 /srv/www` |
| 80 端口被占用 | 其他进程监听 80 | `ss -tlnp \| grep :80` | 停掉占用进程或改 nginx 监听端口 |
| nginx 配置语法错误 | book.conf 写错 | `nginx -t` | 按报错修改后重新 `nginx -t` |
| reload 后新配置不生效 | 没执行 reload | `systemctl status nginx` | `systemctl reload nginx` |
| 健康检查返回非 200 | nginx 未跑或 /health 配置错 | `curl http://127.0.0.1/health`、`journalctl -u nginx` | 重新检查 book.conf |
| 机器无法访问（instabox ssh 也不行） | 云电脑被自动暂停（空闲 30 分钟） | instabox 控制台查看状态 | `instabox resume book` |
| 备份归档为空或缺文件 | tar 路径或排除规则错 | `tar -tzf 归档文件` | 修正备份脚本路径，重新跑 |

## 16.12 复盘：变更记录与回滚预案

SRE 工作不仅是“把东西装上去”，更重要的是“出了事怎么快速恢复”。本章做过的每一步都属于变更，建议记录一个简单的变更日志：

```text
2026-11-14 | 安装 nginx/openssh-server/ufw/fail2ban/rsync | 初始装机
2026-11-14 | 创建 bookuser + /srv/www，权限 755/644 | 站点目录
2026-11-14 | 写 book.conf，启用站点，禁用 default | nginx 配置
2026-11-14 | port publish 80 + ufw 放行 80/tcp | 暴露服务
2026-11-14 | fail2ban [sshd] + server_tokens off + autoindex off | 加固
2026-11-14 | check.sh cron 每 5 分钟健康检查 | 监控
2026-11-14 | backup.sh cron 每天 3 点备份 + snapshot base | 备份
```

**回滚预案**：

- nginx 配置出错：`cp /etc/nginx/sites-available/book.conf.bak /etc/nginx/sites-available/book.conf && systemctl reload nginx`（改前先手动备份）。
- 整机不可用：`instabox create --name book-recover --snapshot base` 从基准快照新建机器。
- 单文件误删：从 `/data/backup/` 最新的 .tar.gz 解包恢复。

## 16.13 挑战任务

完成基础部署后，可尝试以下进阶：

1. **HTTPS**：为站点配置证书（如 Let's Encrypt），让外网链接走 443/tcp（需额外 `instabox port publish book 443` 和 ufw 放行 443/tcp）。
2. **API 反代**：在 nginx 里加一个 `location /api`，把请求反代给本地运行的后端 API（预告卷 2/卷 5 的内容）。
3. **卷 2 方向**：把这套服务从云电脑迁移到一台独立 VPS，学习真实公网 IP、域名解析、SSL 证书。
4. **卷 5 方向**：接入 Prometheus + Grafana，把 nginx 的访问量、响应时间做成可视化监控面板。

## 16.14 nginx 性能基础调优与日志滚动

站点跑通后，做两件让服务更"生产化"的事：给 nginx 做基础性能调优，给日志做滚动，防止日志文件无限增长撑满只有 16 GiB 的系统盘。

### 16.14.1 nginx 性能基础调优

编辑 `/etc/nginx/nginx.conf` 的 `http` 段，常见调优项如下：

```nginx
http {
    worker_processes auto;      # 工作进程数，auto = CPU 核数，你的云电脑 4 核则为 4
    worker_connections 1024;    # 每个工作进程最多并发连接数
    keepalive_timeout 65;       # 长连接保持秒数，减少重复握手开销
    gzip on;                    # 开启 gzip 压缩，减少响应体体积
    gzip_types text/plain text/css application/json application/javascript text/html;
    server_tokens off;          # 已在 book.conf 设置，响应头不带版本号
}
```

修改后 `nginx -t` 验语法，`systemctl reload nginx` 生效。

逐项说明：

- **worker_processes**：nginx 是多进程模型，每个工作进程独立处理请求。`auto` 根据 CPU 核数自动设置，你的云电脑 4 核会按 4 个进程启动。设多了浪费上下文切换，设少了处理能力不足。
- **worker_connections**：单个进程的并发连接上限。理论最大并发 ≈ worker_processes × worker_connections，但受系统文件描述符限制（`ulimit -n`）制约。
- **keepalive_timeout**：客户端与 nginx 的 TCP 长连接保持时间，适当调大可减少频繁建立连接的开销，但占服务器连接资源。
- **gzip**：对文本类内容（HTML、CSS、JSON、JS）压缩传输，显著减少出网流量。你的云电脑出网有 100 GiB/月限额，开启 gzip 能省不少流量。

### 16.14.2 nginx 日志滚动：logrotate

nginx 默认把访问日志写进 `/var/log/nginx/access.log`，不滚动的话文件会越来越大，最终撑满系统盘。Debian 安装 nginx 时通常已带 logrotate 配置，检查：

```bash
cat /etc/logrotate.d/nginx
```

若无此文件，手动创建：

```text
/var/log/nginx/*.log {
    daily
    rotate 7
    compress
    missingok
    notifempty
    create 644 www-data adm
    sharedscripts
    postrotate
        [ -f /var/run/nginx.pid ] && kill -USR1 `cat /var/run/nginx.pid`
    endscript
}
```

各项含义：

- `daily`：每天滚动一次。
- `rotate 7`：保留最近 7 个归档，超过自动删除。
- `compress`：归档用 gzip 压缩，省空间。
- `missingok`：日志文件缺失不报错。
- `notifempty`：日志为空不滚动。
- `create 644 ...`：滚动后新建空日志，权限 644。
- `postrotate/endscript`：滚动后通知 nginx 重新打开日志文件（`kill -USR1` 是 nginx 的优雅重打开日志信号，不中断服务）。

调试不执行真滚动，只看会做什么：

```bash
logrotate -d /etc/logrotate.d/nginx
```

强制立即滚动一次（测试用）：

```bash
logrotate -f /etc/logrotate.d/nginx
```

> **警告**：不要手动 `rm /var/log/nginx/access.log` 删掉正在写的日志文件。nginx 会继续向已删除但未关闭的文件句柄写入，磁盘空间不会释放，且新日志丢失。要滚动日志必须走 logrotate 或 `kill -USR1` 配合重命名。

## 16.15 rsync 增量备份实战

16.9 节的 `backup.sh` 用 tar 做全量归档，适合保留历史版本。日常更频繁的同步，用 **rsync**：它只传输变化的部分，支持断点续传，适合每天多次跑。

```bash
rsync -av --delete /srv/www/ /data/backup/www-sync/
```

参数：`-a` 归档模式（保留权限、时间戳、软链接），`-v` 显示过程，`--delete` 让目标与源保持一致（源里删了的文件，目标也删除）。

**`--delete` 警告**：如果源目录写错（比如把 `/srv/www/` 写成了 `/`），`--delete` 会把目标备份目录里不在源里的文件全部删掉，误删风险极大。先加 `--dry-run` 试跑，只看不执行：

```bash
rsync -av --delete --dry-run /srv/www/ /data/backup/www-sync/
```

确认输出只有预期变更后，再去掉 `--dry-run` 真执行。

**备份策略组合**：tar 归档（版本化，保留每天一份，7 天过期清理）+ rsync 同步（最新副本，快速恢复站点内容）。两者互补，tar 防误删能回溯，rsync 快恢复。

把 rsync 也加进每日备份，改 `/data/home/backup.sh`，在 tar 之后追加：

```bash
rsync -av --delete /srv/www/ /data/backup/www-sync/ >> /data/home/logs/backup.log 2>&1
```

## 16.16 灾难恢复演练全流程

备份没做过恢复演练，等于没备份。下面是一次完整的灾难恢复演练，按顺序操作：

**场景 1：误删站点首页。**

```bash
rm /srv/www/index.html        # 模拟误操作
curl -I http://127.0.0.1/     # 确认 404
```

从 tar 归档恢复单文件：

```bash
tar -xzvf /data/backup/web-backup-2026-11-14.tar.gz -C /tmp/restore-test srv/www/index.html
cp /tmp/restore-test/srv/www/index.html /srv/www/index.html
chown bookuser:bookuser /srv/www/index.html
chmod 644 /srv/www/index.html
curl -I http://127.0.0.1/     # 确认恢复为 200
```

**场景 2：整机不可用（云电脑系统盘损坏或配置彻底搞砸）。**

从 instabox 基准快照恢复整机：

```bash
instabox create --name book-recover --snapshot base
instabox ssh book-recover     # 新机器即有全部基准配置
```

**场景 3：站点目录整体丢失，用 rsync 最新副本恢复。**

```bash
rsync -av /data/backup/www-sync/ /srv/www/
chown -R bookuser:bookuser /srv/www
systemctl reload nginx
```

**两个核心指标**：

- **RTO（恢复时间目标）**：从发现故障到服务恢复的最大可接受时间。单文件恢复应在分钟级，整机恢复取决于快照下载与启动时间。
- **RPO（恢复点目标）**：能接受的最大数据丢失量，即备份间隔。你的设置是每天 3 点 tar + 每 5 分钟健康检查日志，理论上最多丢不到一天的站点内容。

演练完成后，在变更记录里追加一条："2026-11-14 | 完成一次 tar 单文件恢复 + rsync 目录恢复演练，RTO 分钟级，RPO 1 天"，并把演练结果写进运维手册。

## 16.17 HTTPS 与免费证书（挑战任务进阶实操）

16.13 的挑战任务之一就是给站点加 HTTPS。HTTP 明文传输，密码、Cookie、表单数据都可能被窃听或篡改；HTTPS 在 HTTP 外层加 TLS 加密，是生产站点的底线。本节演示用 **Certbot** 申请 Let's Encrypt 免费证书，并给 nginx 配置 HTTPS。

### 16.17.1 前提条件

HTTPS 证书需要两样东西，云电脑免费档都能满足：

1. **一个你拥有的域名**，且域名的 A 记录已解析到云电脑的公网 IP（卷 2 内容，此处假设已有 `book.example.com`）。
2. **80 和 443 端口可达**。InstaBox 免费档每台 16 个公网端口，用 `instabox port publish book 80` 和 `instabox port publish book 443` 发布，再在 ufw 里放行 `80/tcp`、`443/tcp`（第 16.6 节已演示端口发布和 ufw 规则）。

> **警告**：不要对 `book.local` 这类内网别名申请公网证书，Let's Encrypt 只签发公网可解析的域名。内网测试可用自签名证书（`openssl req -x509`），但浏览器会报 "不安全"，仅供本地实验。

### 16.17.2 安装 Certbot 并申请证书

```bash
apt install -y certbot
```

Let's Encrypt 需要验证你控制该域名，最简单的是 standalone 模式：Certbot 自身监听 80 端口，让 Let's Encrypt 服务器来连接验证。申请前先暂停 nginx（避免 80 端口冲突）：

```bash
systemctl stop nginx
certbot certonly --standalone -d book.example.com --agree-tos -m admin@example.com
```

成功后，证书和私钥保存在 `/etc/letsencrypt/live/book.example.com/`：

- `fullchain.pem`：证书链（含中间证书），给 nginx 用；
- `privkey.pem`：私钥，**绝对不能外泄**，权限已由 Certbot 设为 600。

证书有效期 90 天，Certbot 已自动装好系统定时任务（`/etc/cron.d/certbot` 或 systemd timer），到期前自动续期，无需人工干预。

### 16.17.3 nginx 配置 HTTPS

修改 `/etc/nginx/sites-available/book.conf`，新增一个 server 块监听 443：

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name book.example.com;
    # HTTP 全部跳 HTTPS
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name book.example.com;

    ssl_certificate /etc/letsencrypt/live/book.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/book.example.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    root /srv/www;
    index index.html;

    location = /health {
        return 200 '{"status":"ok"}';
        add_header Content-Type application/json;
    }

    location / {
        try_files $uri $uri/ =404;
    }
}
```

要点：

- `listen 443 ssl` 开启 TLS，`ssl_certificate` / `ssl_certificate_key` 指向 Certbot 生成的文件；
- `ssl_protocols` 只允许 TLS 1.2+，禁用已知不安全的 SSLv3、TLS 1.0/1.1；
- `return 301 https://$host$request_uri` 把所有 HTTP 请求永久重定向到 HTTPS，避免明文访问；
- `try_files $uri $uri/ =404`：先尝试静态文件，找不到返回 404。

验语法并重载：

```bash
nginx -t
systemctl reload nginx
```

用浏览器打开 `https://book.example.com`，地址栏出现锁图标，证书颁发机构显示 "Let's Encrypt"，即配置成功。

### 16.17.4 小结

本章到此，你完成了从零开始的完整闭环：

1. 装机与盘点（InstaBox 云电脑 + Debian 13）；
2. 账号与目录（普通用户 + sudo，Web 目录隔离）；
3. 站点内容（index.html + health.json，权限 644）；
4. nginx 配置（虚拟主机、健康检查、隐藏版本号、关闭目录浏览）；
5. 暴露服务（端口发布 + ufw 放行）；
6. 加固（ufw 默认拒绝、fail2ban 保护 SSH、server_tokens off）；
7. 监控与日志（5 分钟健康检查落盘、journalctl、logrotate 日志滚动）；
8. 备份（tar 归档 + rsync 增量同步 + 7 天过期清理 + instabox 快照）；
9. 验收与故障排查；
10. HTTPS 证书与全站加密。
以上十项构成一台生产 Web 服务器的最小完整闭环，缺一不可，直接决定站点上线后的稳定性与安全性。

## 16.18 生产环境与云电脑实验室的区别

本书所有实操都在 InstaBox 云电脑免费档上完成，它是"实验室环境"：弄坏了可以重建，默认 root 操作，端口随意发布。把本章的站点迁移到真实生产服务器（VPS 或云厂商 ECS）时，请注意以下区别，避免直接照搬实验室配置。下面逐项说明每一项区别在实践中意味着什么，以及迁移时需要调整的配置：

| 项目 | 云电脑实验室（本书） | 生产环境 |
| --- | --- | --- |
| 身份 | 默认 root，方便实验 | 禁止日常用 root；建普通用户 + sudo，最小权限 |
| 端口 | 免费档 16 个公网端口，随意发布 | 仅开业务必须端口（80/443/22），其余防火墙默认拒绝 |
| 出网流量 | 100 GiB/月，个人实验绰绰有余 | 按量付费，开 gzip/CDN、避免大文件直出 |
| 快照 | 5 个，免费，随时恢复 | 云厂商快照通常收费，且需定期测试恢复 |
| 机器状态 | 空闲 30 分钟自动暂停，免费档不可调 | 7×24 不可中断，需高可用与自动恢复 |
| 域名与证书 | 可用 book.local 等内网别名测试 | 需公网域名、ICP 备案（国内）、Let's Encrypt 证书 |
| 用户隔离 | 单用户实验，权限不敏感 | 多团队共享，必须做用户、群组、文件权限隔离（第 5 章） |
| 备份 | tar + rsync + instabox 快照 | 多副本、异地备份、离线副本，定期恢复演练（3-2-1 原则） |
| 日志 | journalctl + nginx access.log | 集中日志（ELK/Loki）、日志保留策略、告警通知 |
| 监控 | check.sh 每 5 分钟健康检查落盘 | Prometheus + Grafana + 报警（卷 2 / 卷 5 内容） |

**迁移检查清单**：在真实服务器上重复本章所有步骤前，先核对：
1. 已创建非 root 用户并授 sudo，日常以该用户登录；
2. ufw 默认拒绝，仅放行 22/80/443，SSH 密钥登录并禁用密码；
3. nginx `server_tokens off`、`autoindex off`，响应头不泄露版本；
4. 证书来自可信 CA（Let's Encrypt），且自动续期已配置；
5. 备份已跑通至少一次恢复演练，且备份介质与生产不在同一机房；
6. 关键服务已配置 systemd 开机自启，`systemctl is-enabled` 为 enabled；
7. 日志已配置 logrotate，防止日志文件无限增长撑满系统盘。

做到这七条，你的站点就已经超过绝大多数"直接 root 搭建、裸奔 80 端口、无备份"的初创服务器了。

卷 1 的 16 章全部撰写完毕，你已经掌握了从零到部署一个生产化 Web 服务的完整流程，包括加固、监控、备份与灾难恢复。下一卷（网络与系统服务）将把 nginx 从单机静态站点扩展到反向代理、负载均衡、数据库与缓存，继续深入。

> **参考答案**：
> 1. 安装包：`apt install -y nginx openssh-server ufw fail2ban rsync curl`；验证：`systemctl is-active nginx ssh` 应返回 active。
> 2. 用户目录：`useradd -m -s /bin/bash bookuser && usermod -aG sudo bookuser && mkdir -p /srv/www && chown -R bookuser:bookuser /srv/www && chmod 755 /srv/www`。
> 3. 站点文件：index.html 含 `<html>` 标签，health.json 含 `{"status":"ok"...}`，权限 644。
> 4. nginx 配置：`listen 80; root /srv/www; location = /health { return 200 '...'; }`，`nginx -t` 通过，`systemctl reload nginx`。
> 5. 暴露服务：`instabox port publish book 80` 得到 HTTPS 链接；`ufw allow 80/tcp && ufw enable`。
> 6. 加固：ufw default deny incoming/allow outgoing，fail2ban [sshd] enabled，server_tokens off，autoindex off。
> 7. 监控：check.sh 用 curl 检查 /health 返回 200，crontab `*/5 * * * *` 加入，日志落 /data/home/logs。
> 8. 备份：backup.sh 打包 /srv/www /etc/nginx 等，crontab 每天 3 点，做过 tar 真恢复演练，`instabox snapshot save book --name base`。
