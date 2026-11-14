---
title: "第 12 章 systemd 与服务管理"
---

# 第 12 章 systemd 与服务管理

> 第 8 章我们见过进程，也知道 1 号进程是系统的祖宗。在 Debian 13 上，1 号进程就是 systemd。它不仅是所有进程的父进程，还负责启动系统、管理服务、处理日志、定时任务、电源管理。对 SRE 来说，systemd 是日常最高频的工具之一：重启服务、查状态、看日志、设置开机自启，全靠它。本章把 systemd 从"一个奇怪的命令集"讲成一套可预测的服务管理体系。

> **本章目标**
> - 能解释 systemd 是什么、1 号进程的作用，以及"unit（单元）"的概念。
> - 能用 `systemctl` 启动、停止、重启、查询服务，设置与取消开机自启，并用 `systemctl is-active/is-enabled` 判断服务健康状态。
> - 能自定义一个 `.service` unit 文件，把自己的脚本注册为系统服务，做到崩溃自动重启、开机自启。
> - 能用 `journalctl` 查看服务日志和系统启动日志，并按单元、时间过滤。

---

## 12.1 什么是 systemd

**systemd 是 Linux 的系统与服务管理器（system and service manager）**，也是 1 号进程（PID 1）。内核启动后启动的第一个用户态进程，负责把系统从"刚加载完内核"带到"可以登录使用"的状态。

传统的 init 系统（如 SysV init）是顺序启动服务，一个服务启得慢，后面的都得等。systemd 的核心优势是**并行启动**和**按需启动**：互不依赖的服务同时启动，用到某个服务时才把它拉起来，开机更快。

> **SRE 贴士**：`ps -p 1 -o comm=` 确认 1 号进程是谁。如果输出 `systemd`，说明这台机器用 systemd 管理；如果是 `init`，可能是较老或更精简的系统，命令会不一样。你的云电脑（Debian 13）肯定是 systemd。

### unit：systemd 的配置对象

systemd 把一切可管理的对象都称为 **unit（单元）**，以 `.xxx` 结尾命名。常见类型：

| 类型 | 后缀 | 作用 |
| --- | --- | --- |
| service | `.service` | 后台服务，如 nginx、ssh、你自己的脚本 |
| socket | `.socket` | 监听端口，有连接时才启动对应服务（按需启动） |
| timer | `.timer` | 定时触发，类似 cron（见第 14 章对比） |
| target | `.target` | 目标状态，如多用户模式、图形模式，相当于"运行级别" |
| path | `.path` | 监听文件系统变化，变化后触发服务 |

unit 文件放在几个标准位置，优先级从高到低：

- `/etc/systemd/system/`：管理员手写的自定义 unit，**最高优先级**，你写服务都放这里。
- `/run/systemd/system/`：运行时动态生成的 unit。
- `/usr/lib/systemd/system/`：软件包安装的原厂 unit，**不要直接改这里**，更新会被覆盖。

> **警告**：不要直接修改 `/usr/lib/systemd/system/` 下的原厂 unit 文件。软件包升级时会被覆盖，你的改动白费。要改原厂服务的参数，用 `systemctl edit 服务名`，systemd 会在 `/etc/systemd/system/` 里创建覆盖片段，优先级更高且安全。

## 12.2 systemctl：服务的开关与探针

`systemctl`（system control）是 systemd 的主控命令，所有服务操作都经过它。

### 常用操作

| 命令 | 含义 |
| --- | --- |
| `systemctl start 名称` | 立即启动服务 |
| `systemctl stop 名称` | 立即停止服务 |
| `systemctl restart 名称` | 重启服务 |
| `systemctl reload 名称` | 重载配置（不中断服务，如 nginx 重载配置） |
| `systemctl status 名称` | 查看服务运行状态和最近日志 |
| `systemctl is-active 名称` | 查询是否正在运行（active/inactive，适合脚本判断） |
| `systemctl is-enabled 名称` | 查询是否开机自启（enabled/disabled） |
| `systemctl enable 名称` | 设置开机自启 |
| `systemctl disable 名称` | 取消开机自启 |
| `systemctl mask 名称` | 强制禁止启动（即使依赖也启不了，慎用） |
| `systemctl unmask 名称` | 取消禁止 |

服务名称通常带 `.service` 后缀，也可以省略，systemd 会自动补全。

### 实战：查看 ssh 服务状态

```bash
systemctl status ssh
```

```text
● ssh.service - OpenBSD Secure Shell server
     Loaded: loaded (/lib/systemd/system/ssh.service; enabled; preset: enabled)
     Active: active (running) since Thu 2026-11-13 09:11:00 UTC; 18min ago
       Docs: man:sshd(8)
     Main PID: 452 (sshd)
        Tasks: 3 (limit 4619)
       Memory: 8.2M
        CPU: 120ms
     CGroup: /system.slice/ssh.service
             └─452 /usr/sbin/sshd -D
```

逐段念：

- **Loaded**：unit 文件路径，`enabled` 表示开机自启，`preset` 是发行版默认策略。
- **Active**：`active (running)` 表示正在运行；其他常见值有 `inactive (dead)`（停止）、`failed`（启动失败）、`activating`（启动中）。
- **Main PID**：主进程号，`Tasks` 线程数，`Memory` 和 `CPU` 资源占用。
- **CGroup**：该服务下的进程都在这个控制组里，systemd 可以整组管理。

开机自启：

```bash
systemctl enable ssh      # 开机自启
systemctl disable ssh     # 取消开机自启
systemctl is-enabled ssh  # 查询：enabled / disabled
```

> **SRE 贴士**：排障时先 `systemctl status` 看服务是不是 `active (running)`。如果是 `failed`，紧接着 `journalctl -u 服务名 -n 50` 看最近 50 行日志，通常第一行就能看到报错原因。不要只重启服务不看日志，否则同样的故障会重复发生。

## 12.3 写自己的 service unit

把你的脚本注册成系统服务，让它开机自启、崩溃重启，是 systemd 最实用的能力。

### 示例：一个简单的监控脚本服务

先写一个脚本，放在 `/usr/local/bin/heartbeat.sh`：

```bash
#!/bin/bash
while true; do
    echo "$(date) heartbeat ok" >> /var/log/heartbeat.log
    sleep 30
done
```

```bash
chmod +x /usr/local/bin/heartbeat.sh
```

再写 unit 文件 `/etc/systemd/system/heartbeat.service`：

```ini
[Unit]
Description=Heartbeat logging service
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/heartbeat.sh
Restart=always
RestartSec=5
User=root

[Install]
WantedBy=multi-user.target
```

各段说明：

- **[Unit]**：元数据。`Description` 是描述；`After=network.target` 表示在网络就绪后再启动此服务（只是启动顺序建议，不是强依赖）。
- **[Service]**：核心。`Type=simple` 表示进程启动后立即视为就绪；`ExecStart` 是启动命令；`Restart=always` 表示进程异常退出总是重启；`RestartSec=5` 重启前等 5 秒；`User=root` 以 root 运行（生产上建议改成专用服务账号，见第 5 章）。
- **[Install]**：`WantedBy=multi-user.target` 表示在多用户模式下启用此服务，即开机自启。

### 注册并启动

```bash
systemctl daemon-reload      # 重读 unit 文件，新写或改过的 unit 必须执行
systemctl enable heartbeat   # 设置开机自启
systemctl start heartbeat    # 立即启动
systemctl status heartbeat   # 查看状态
```

`daemon-reload` 是新手常忘的一步：改完 unit 文件不执行它，systemd 还按旧配置跑，改动看起来"不生效"。

### 查看服务日志

```bash
journalctl -u heartbeat -n 50      # 该服务最近 50 行日志
journalctl -u heartbeat -f         # 实时跟踪日志（Ctrl-C 退出）
journalctl -u heartbeat --since "1 hour ago"  # 过去一小时的日志
```

> **警告**：`systemctl stop` 或 `disable` 一个你不了解的系统服务（如 sshd、systemd-journald、dbus），可能让你失去对云电脑的连接，甚至导致系统无法正常启动。只操作自己创建的服务，或明确知道作用的服务。

## 12.4 常用 unit 管理命令速查

```bash
systemctl list-units --type=service --state=running   # 列出正在运行的服务
systemctl list-unit-files --type=service              # 列出所有 service 及开机自启状态
systemctl list-dependencies ssh                       # 显示 ssh 依赖哪些 unit
systemctl show ssh -p MainPID -p SubState             # 只查某个字段，适合脚本
systemctl reset-failed                                # 重置失败状态，清理 failed 标记
```

## 12.5 target：系统的"运行级别"

systemd 用 `target` 替代传统 SysV init 的运行级别（0~6）。常见 target：

| target | 含义 |
| --- | --- |
| `multi-user.target` | 多用户命令行模式，服务器常态（相当于旧运行级别 3） |
| `graphical.target`  | 图形桌面模式（相当于旧运行级别 5） |
| `rescue.target`     | 救援模式，最小系统，用于修复 |
| `emergency.target`  | 紧急模式，只挂载根文件系统只读 |

查看当前默认 target：

```bash
systemctl get-default
```

```text
multi-user.target
```

设置默认 target（改成图形启动）：

```bash
systemctl set-default graphical.target
```

> **SRE 贴士**：服务器通常保持 `multi-user.target`，不需要图形桌面。你的云电脑虽然有桌面 Chromium，但 SSH 管理时后台运行的是多用户模式，服务都挂在这个 target 下。

## 12.6 timer：systemd 的定时任务

systemd 也能做定时任务，用 `.timer` + `.service` 配对，功能上类似 cron（第 14 章专门讲 cron，两者可选其一，系统服务类定时任务推荐 systemd timer，因为能统一用 `systemctl`/`journalctl` 管理）。

简单示例：每小时跑一次 heartbeat 检查。

`/etc/systemd/system/heartbeat-check.service`：

```ini
[Unit]
Description=Hourly heartbeat check

[Service]
Type=oneshot
ExecStart=/usr/local/bin/heartbeat-check.sh
```

`/etc/systemd/system/heartbeat-check.timer`：

```ini
[Unit]
Description=Run heartbeat check hourly

[Timer]
OnCalendar=hourly
Persistent=true

[Install]
WantedBy=timers.target
```

```bash
systemctl daemon-reload
systemctl enable --now heartbeat-check.timer
```

`--now` 表示启用的同时立即启动；`Persistent=true` 表示如果机器在触发时关机，开机后补执行一次。

查看所有定时器及下次触发时间：

```bash
systemctl list-timers --all
```

systemd timer 和 cron 的选型建议：任务与系统服务强相关、需要随服务一起启停、或日志要统一进 journald，用 systemd timer；简单的脚本定时执行、与系统服务无关，用 cron 更轻量（第 14 章专门讲 cron）。

## 12.7 cgroup：systemd 的资源隔离

systemd 把每个服务放进一个 **cgroup（控制组）**——内核用来限制和隔离进程组资源使用的机制。你在 `systemctl status ssh` 输出里看到的 `CGroup: /system.slice/ssh.service`，就是该服务的控制组路径。

cgroup 能限制：

- CPU：`CPUQuota=50%` 限制服务最多占一半 CPU；
- 内存：`MemoryMax=512M` 限制内存上限，超出触发 OOM 杀死；
- 磁盘 I/O：`Weight=500` 调整块设备 I/O 权重。

在 unit 文件里加限制，示例：

```ini
[Service]
ExecStart=/usr/local/bin/heartbeat.sh
CPUQuota=20%
MemoryMax=256M
```

修改后 `systemctl daemon-reload && systemctl restart heartbeat` 即可生效。对 SRE 来说，cgroup 是防止单个服务把整机 CPU、内存吃光的最后一道防线。

## 12.8 systemd-analyze：开机性能分析

想知道开机慢在哪里，不用猜，用 `systemd-analyze`：

```bash
systemd-analyze blame          # 列出各服务启动耗时，按耗时降序
systemd-analyze critical-chain # 显示启动依赖链，找出拖慢启动的关键路径
```

代表性输出（节选）：

```text
         5.234s networking.service
         2.100s ssh.service
         0.800s fail2ban.service
```

如果某个服务启动耗时异常长，就重点优化它；`critical-chain` 会告诉你是哪个依赖在等它。生产环境上线新服务前，跑一次 `blame` 确认不拖慢启动，是好的工程习惯。

## 12.9 更多常用 unit 指令

在 `[Service]` 段里，除了 `ExecStart` 和 `Restart`，还常用：

| 指令 | 作用 | 示例 |
| --- | --- | --- |
| `ExecStartPre` | 启动主命令前先执行，如建目录、等数据库就绪 | `ExecStartPre=/bin/mkdir -p /run/app` |
| `ExecStartPost` | 主命令启动后再执行 | `ExecStartPost=/bin/ping -c 1 119.29.29.29` |
| `ExecStop` | 服务停止时执行的清理命令 | `ExecStop=/bin/kill -TERM $MAINPID` |
| `WorkingDirectory` | 进程启动时的当前目录 | `WorkingDirectory=/srv/app` |
| `Environment` | 设置环境变量 | `Environment="APP_ENV=prod"` |
| `EnvironmentFile` | 从文件加载环境变量 | `EnvironmentFile=/etc/app/env` |
| `RestartSec` | 重启前等待秒数 | `RestartSec=5` |

`ExecStartPre/Post` 失败会导致服务启动失败，适合做启动前检查；`WorkingDirectory` 建议写绝对路径，避免服务在意外目录下运行找不到文件。

---

## 本章小结

| 知识点 | 核心要点 |
| --- | --- |
| systemd | Linux 系统与服务管理器，PID 1，并行启动、按需启动、全面接管服务与日志 |
| unit | service/socket/timer/target/path，`.service` 最常见，放在 `/etc/systemd/system/` |
| systemctl | start/stop/restart/status/enable/disable/is-active/is-enabled，服务全生命周期 |
| unit 文件 | [Unit][Service][Install] 三段，`ExecStart` 指命令，`Restart=always` 崩溃重启 |
| daemon-reload | 改完 unit 文件必须执行，systemd 才会重读配置 |
| journalctl | `-u` 指服务，`-f` 实时跟踪，`--since` 按时间过滤，服务日志的统一入口 |
| target | multi-user.target 多用户命令行，graphical.target 图形桌面，`get-default/set-default` 查改 |
| timer | 系统级定时任务，OnCalendar 定时，Persistent 补执行，替代部分 cron 场景 |
| mask | 强制禁止服务启动，慎用，可能导致依赖服务也无法启动 |

本章讲了"怎么让程序稳定地在后台运行、开机自启、崩溃重启"。下一章（第 13 章《Shell 脚本入门》）我们学习用脚本把零散命令串成可自动化的任务，为第 16 章的综合实战打基础。

---

## 动手练习

在自己的云电脑上完成以下练习。

**练习 1**：确认 1 号进程是 systemd；用 `systemctl status ssh` 查看 ssh 服务状态，写出它的 Active 状态和 Main PID；用 `systemctl is-active ssh` 和 `systemctl is-enabled ssh` 分别查询运行与开机自启状态。

**练习 2**：写一个脚本 `/usr/local/bin/hello-service.sh`，内容是每 10 秒往 `/var/log/hello.log` 追加一行时间戳；编写 `/etc/systemd/system/hello.service` unit 文件，设置 `Restart=always`，执行 `daemon-reload`、`enable --now`，验证 `systemctl status hello` 显示 `active (running)`。

**练习 3**：故意杀掉 hello 服务的主进程（`kill <PID>`），等待 6 秒左右，再 `systemctl status hello`，确认 systemd 自动重启了服务，且 PID 发生了变化。

**练习 4**：用 `journalctl -u hello -n 30` 查看 hello 服务最近日志；再用 `journalctl -u hello -f` 实时跟踪一段日志（Ctrl-C 退出）。

**练习 5**：用 `systemctl disable hello` 取消开机自启，`systemctl is-enabled hello` 确认变为 `disabled`；最后用 `systemctl stop hello` 停止服务，并 `systemctl reset-failed` 清理可能的失败状态。

---

## 参考答案

**答案 1**：

```bash
ps -p 1 -o comm=
systemctl status ssh
systemctl is-active ssh
systemctl is-enabled ssh
```

预期：第一条输出 `systemd`；`status` 中 `Active: active (running)`，Main PID 是 sshd 进程号；`is-active` 输出 `active`；`is-enabled` 输出 `enabled`。

**答案 2**：

```bash
cat > /usr/local/bin/hello-service.sh <<'EOF'
#!/bin/bash
while true; do
    echo "$(date) hello from systemd" >> /var/log/hello.log
    sleep 10
done
EOF
chmod +x /usr/local/bin/hello-service.sh
```

unit 文件 `/etc/systemd/system/hello.service`：

```ini
[Unit]
Description=Hello systemd service
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/hello-service.sh
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable --now hello
systemctl status hello
```

**答案 3**：

```bash
PID=$(systemctl show hello -p MainPID --value)
kill $PID
sleep 6
systemctl status hello
```

预期：status 里显示 `Active: active (running)`，且 Main PID 已变成新的进程号，证明 systemd 自动重启了服务。

**答案 4**：

```bash
journalctl -u hello -n 30
journalctl -u hello -f
```

第一条输出最近 30 行，第二条进入实时跟踪，新日志行会持续打印，Ctrl-C 退出。

**答案 5**：

```bash
systemctl disable hello
systemctl is-enabled hello    # 预期 disabled
systemctl stop hello
systemctl reset-failed
systemctl status hello        # 预期 inactive (dead)
```

关键点：`daemon-reload` 在改完 unit 后必须执行；`Restart=always` 是服务高可靠的基础配置；`journalctl -u` 是排查服务故障的第一入口。
