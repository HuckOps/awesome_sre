# 第 11 章 网络配置与远程管理

> 前十章我们一直在一台机器内部挖掘：文件、用户、进程、文本。但服务器的价值不在于它自己跑得多快，而在于它能否被其他机器、其他人可靠地访问。没有网络，Web 服务只是本地一个监听端口，数据库只是本地一个文件，备份脚本只能备份自己。本章把镜头从机器内部移到机器之间：IP 地址、路由、DNS、SSH、端口，以及在 InstaBox 云电脑里怎么把内部服务暴露到公网。

> **本章目标**
> - 能说清 IP、MAC、子网掩码、网关、DNS 五个网络基础概念各自作用，并用 `ip addr`/`ip route` 读出云电脑的网络配置。
> - 能配置静态 IP 与 DHCP、修改 DNS 解析器、排查 `/etc/resolv.conf` 生效问题。
> - 能用 `ping`/`traceroute`/`dig` 判断"是网络不通还是服务挂了"，并用 `ss`/`lsof` 看出哪个进程占了哪个端口。
> - 能用 SSH 密钥登录云电脑、配置 `~/.ssh/config` 简化连接、用 `scp` 传文件，并理解端口转发在云电脑里的作用。

---

## 11.1 网络基础：五个概念

任何一台接网的设备，要和其他设备通信，至少需要搞清楚下面五样：

| 概念 | 人话理解 | 例子 |
| --- | --- | --- |
| IP 地址 | 设备在网络里的"门牌号" | `10.1.2.3` |
| MAC 地址 | 网卡出厂烧写的物理标识，全球唯一 | `00:1a:2b:3c:4d:5e` |
| 子网掩码 | 划定"哪些 IP 在同一个局域网内" | `255.255.255.0`（即 /24） |
| 网关 | 本地局域网出去的"大门"，去外网的第一站 | `10.1.2.1` |
| DNS | 把域名翻译成 IP 地址的电话本 | `example.com → 93.184.216.34` |

**IP 地址版本**：IPv4（如 `10.1.2.3`，最常见）和 IPv6（如 `240e:xxx::1`）。本书以 IPv4 为主，IPv6 了解即可。

**私有地址与公有地址**：局域网内常用私有地址段（`10.x.x.x`、`172.16.x.x`、`192.168.x.x`），外网访问需要经过网关做 NAT 地址转换。你的云电脑内网 IP 可能是私有地址，公网端口映射到它，外部才能访问。

**CIDR 表示法**：`10.1.2.0/24` 表示前 24 位是网络号，后 8 位是主机号，这个子网里可用地址是 `10.1.2.1` ~ `10.1.2.254`。

> **SRE 贴士**：排障时第一件事是"先确定网络分层"——能 ping 通网关就说明本地网卡和局域网正常；能 ping 通 8.8.8.8 说明出网正常；能解析域名说明 DNS 正常；三者都通但网站打不开，问题通常在应用层（Web 服务没启或防火墙挡住了），而不是网络层。

## 11.2 查看网络状态：ip 命令

现代 Linux 用 `ip` 命令（iproute2 工具集）查看和配置网络，老命令 `ifconfig`（net-tools）已逐渐淘汰，生产上可能装不着，建议直接学 `ip`。

### 11.2.1 查看网卡和地址

```bash
ip addr
```

代表性输出（节选）：

```text
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:1a:2b:3c:4d:5e brd ff:ff:ff:ff:ff:ff
    inet 10.1.2.3/24 brd 10.1.2.255 scope global dynamic eth0
       valid_lft 86123sec preferred_lft 86123sec
    inet6 fe80::21a:2bff:fe3c:4d5e/64 scope link
       valid_lft forever preferred_lft forever
```

逐项解读：

- `lo`：回环接口（loopback），`127.0.0.1`，机器自我对话，不经过网卡。
- `eth0`：第一块以太网卡，云电脑的主网卡。
- `link/ether`：MAC 地址。
- `inet 10.1.2.3/24`：IPv4 地址及子网掩码（/24 = 255.255.255.0），`dynamic` 表示由 DHCP 自动分配。
- `state UP`：网卡已启用且连接正常。

### 11.2.2 查看路由

```bash
ip route
```

```text
default via 10.1.2.1 dev eth0 proto dhcp metric 100
10.1.2.0/24 dev eth0 proto kernel scope link src 10.1.2.3
```

- `default via 10.1.2.1`：去不认识的目标（包括公网）全部交给网关 `10.1.2.1`。
- `10.1.2.0/24`：本地局域网直接通过 eth0 送达，不经过网关。

### 11.2.3 查看邻居（ARP 表）

```bash
ip neighbor
```

```text
10.1.2.1 dev eth0 lladdr 00:aa:bb:cc:dd:ee REACHABLE
```

本地网段内其他设备的 IP 到 MAC 地址映射。

> **SRE 贴士**：`ip` 命令有分层子命令：`ip link`（链路层，网卡）、`ip addr`（网络层地址，IP）、`ip route`（路由）。记不全就记 `ip -s -s addr`（详细地址信息）和 `ip route get 8.8.8.8`（查询去 8.8.8.8 的路径，排障神器）。

## 11.3 配置网络：DHCP 与静态 IP

云电脑默认用 DHCP 自动获取 IP，重启后 IP 可能变化。如果你要长期把某个端口映射出去，建议给网卡配静态 IP。

Debian 13 传统上用 ifupdown，配置文件是 `/etc/network/interfaces`；桌面镜像也可能用 NetworkManager。下面以 ifupdown 为例（最通用，适合云电脑无头模式）。

### 11.3.1 查看当前配置来源

```bash
cat /etc/network/interfaces
```

典型内容：

```text
# This file describes the network interfaces available on your system
source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
allow-hotplug eth0
iface eth0 inet dhcp
```

`iface eth0 inet dhcp` 表示 eth0 用 DHCP。

### 11.3.2 改成静态 IP

```text
iface eth0 inet static
    address 10.1.2.10/24
    gateway 10.1.2.1
    dns-nameservers 119.29.29.29 8.8.8.8
```

修改后重启网络使生效：

```bash
systemctl restart networking
```

> **警告**：通过 SSH 远程修改网络配置时，**一错就可能断开连接**，甚至让云电脑再也 SSH 不进去（比如把网关写错、把 IP 写成和网关不在同一子网）。最安全的做法是：先用 InstaBox 网页控制台的 Computer 标签（内嵌浏览器终端，不依赖 SSH）操作，或者保留一条能回滚的会话不关闭，直到新配置验证通过。改完后立刻 `ip addr` 确认地址正确，再尝试新 SSH 会话，旧会话别急着关。

### 11.3.3 临时配置（不重启生效，重启失效）

```bash
ip addr add 10.1.2.10/24 dev eth0
ip route add default via 10.1.2.1
```

适合现场试错，重启后恢复。

## 11.4 DNS 与域名解析

域名到 IP 的转换靠 DNS，配置在 `/etc/resolv.conf`：

```bash
cat /etc/resolv.conf
```

```text
nameserver 119.29.29.29
nameserver 8.8.8.8
```

`nameserver` 就是 DNS 服务器地址，按顺序查询。云电脑通常由 DHCP 自动下发，手动改 `/etc/resolv.conf` 可能被 DHCP 覆盖；永久生效建议写进 `/etc/network/interfaces` 的 `dns-nameservers`，或用 `resolvconf`/`systemd-resolved`（视镜像而定）。

测试 DNS 解析：

```bash
dig example.com
```

若未安装 dig（dnsutils 包，见第 9 章软件包管理），可用 `nslookup example.com` 或 `curl -I https://example.com` 间接验证。

> **SRE 贴士**："网络不通"的排查顺序——先 `ip addr` 看有没有 IP，再 `ip route` 看有没有默认路由，再 `ping 119.29.29.29` 看 IP 层通不通，最后 `dig example.com` 看 DNS 能不能解析。大部分"网络故障"其实是 DNS 或防火墙问题，不是真正的链路断了。

## 11.5 连通性测试

### 11.5.1 ping

```bash
ping -c 4 119.29.29.29
```

```text
PING 119.29.29.29 (119.29.29.29) 56(84) bytes of data.
64 bytes from 119.29.29.29: icmp_seq=1 ttl=51 time=12.4 ms
64 bytes from 119.29.29.29: icmp_seq=2 ttl=51 time=11.8 ms
64 bytes from 119.29.29.29: icmp_seq=3 ttl=51 time=12.1 ms
64 bytes from 119.29.29.29: icmp_seq=4 ttl=51 time=12.6 ms

--- 119.29.29.29 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3003ms
rtt min/avg/max/mdev = 11.8/12.2/12.6/0.3 ms
```

- `-c 4` 只发 4 个包然后停，否则一直 ping 直到 Ctrl-C。
- `0% packet loss` 表示链路通畅；`ttl` 是生存时间，每经过一路由器减 1。

> **警告**：有些服务器或云安全组默认禁止 ICMP（ping），ping 不通不一定代表服务挂了——对端 Web 端口 80/443 可能正常，只是不回 ping。要区分"主机不通"和"端口不通"。

### 11.5.2 traceroute / mtr

```bash
traceroute 119.29.29.29
```

显示数据包经过的每一跳路由器，找出哪一跳丢包或延迟高。`mtr` 结合了 ping 和 traceroute，实时显示，更适合持续观察，需额外安装。

## 11.6 远程登录：SSH

SSH（Secure Shell，安全外壳协议）是运维连接远程 Linux 的标准方式，InstaBox 的 `instabox ssh book` 底层也是 SSH。

### 11.6.1 SSH 密钥

密码登录容易被暴力破解，生产上推荐密钥登录。在本地（或云电脑内）生成密钥对：

```bash
ssh-keygen -t ed25519 -C "book@instabox"
```

一路回车，默认生成 `~/.ssh/id_ed25519`（私钥，绝对不能外传）和 `~/.ssh/id_ed25519.pub`（公钥，可以放到服务器上）。

把公钥拷贝到目标机器，实现免密登录：

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub root@book.cloud
```

之后 `ssh root@book.cloud` 就不用输入密码了。

### 11.6.2 ~/.ssh/config：给常用连接起别名

每次敲长命令很烦，写配置文件：

```text
Host book
    HostName book.instabox.cloud
    User root
    Port 22
    IdentityFile ~/.ssh/id_ed25519
```

之后 `ssh book` 即可连接，`scp` 也能用 `book:` 这个别名。

> **SRE 贴士**：私钥文件权限必须严格，否则 SSH 会拒绝使用：`chmod 600 ~/.ssh/id_ed25519`。公钥可以随意传，私钥泄露等于账号沦陷。

### 11.6.3 文件传输：scp

```bash
scp /local/file.txt root@book.cloud:/tmp/
scp root@book.cloud:/var/log/syslog ./syslog.backup
```

带 `-r` 递归传目录。大量文件或大文件更推荐 `rsync`（可断点续传，见后续章节预告）。

## 11.7 端口与连接查看：ss / lsof

想知道本机哪些端口在监听、哪个进程占了 80 端口：

```bash
ss -tlnp
```

```text
State   Recv-Q   Send-Q   Local Address:Port   Peer Address:Port   Process
LISTEN  0        128      0.0.0.0:22           0.0.0.0:*           users:(("sshd",pid=1,fd=3))
LISTEN  0        128      0.0.0.0:80           0.0.0.0:*           users:(("nginx",pid=284,fd=6))
```

参数：`-t` TCP、`-l` 仅监听、`-n` 不解析服务名（显示端口数字，更快）、`-p` 显示进程。

`lsof -i :80` 也能查端口占用（需安装 lsof）。

`ss -tunp` 加上 `-u` 显示 UDP，`-a` 显示所有连接（不只监听）。

> **SRE 贴士**：`netstat` 是老命令（net-tools），新系统可能不装，`ss` 是替代品，且更快、信息更全。记得 `ss -tlnp` 这个排障口诀，看端口占用时默认先敲它。

## 11.8 云电脑里的网络：InstaBox 端口转发

你的云电脑跑在 InstaBox 上，默认有 16 个公网端口可用。如果在云电脑里启了一个 Web 服务监听本地 3000 端口，外网怎么访问？

```bash
instabox port publish book 3000
```

这会把容器内 3000 端口发布成一个 HTTPS 链接，外网浏览器直接访问。

或者本地端口转发：

```bash
instabox port-forward book 8080:3000
```

本地浏览器访问 `http://localhost:8080`，流量转发到云电脑的 3000 端口。

免费档每台云电脑 16 个公网端口，空闲 30 分钟自动暂停，暂停后端口转发也断开，`resume` 后恢复。

> **SRE 贴士**：云电脑是"实验室环境"，端口随意发布方便调试，但生产服务器上只开业务必须的端口，其余全部防火墙拒绝（详见第 15 章《防火墙与数据备份》）。

## 11.9 本地主机名解析：/etc/hosts

在 DNS 生效之前，Linux 先查本地文件 `/etc/hosts`。它把主机名直接映射到 IP，优先级高于 DNS，适合给云电脑起一个容易记住的本地别名。

```bash
cat /etc/hosts
```

典型内容：

```text
127.0.0.1   localhost
::1         localhost
```

给云电脑内网 IP 起别名（以 root 编辑，需谨慎，错误条目会导致该主机名无法访问）：

```text
10.1.2.3    book.local
```

之后 `ping book.local` 或 `ssh book.local` 都能直接走到 `10.1.2.3`，不需要 DNS 查询。

**优先级**：`/etc/hosts` → DNS。这意味着如果你在 `/etc/hosts` 里写了 `example.com 1.2.3.4`，本机访问 `example.com` 就会去 `1.2.3.4`，哪怕真实 DNS 解析结果完全不同——这是排障时一个经典的"坑"，若发现某个域名本机解析奇怪，先看 `/etc/hosts`。

**局限**：`/etc/hosts` 只对本机生效，不是给全网用的。想让全网用你的域名，还要走 DNS 服务商（卷 2 内容）。

## 11.10 SSH 端口转发隧道

除了登录远程终端，SSH 还能做"端口转发隧道"：把一台机器的端口流量，通过加密的 SSH 连接，转发到另一台机器的端口。三种常见模式：

**本地转发 `-L`**：本地端口 → 远程机器上的某个端口。场景：云电脑内 nginx 监听 80，你想在本机浏览器访问，但不想用 `instabox port publish`：

```bash
ssh -L 8080:localhost:80 root@book
```

之后本机浏览器打开 `http://localhost:8080`，流量经 SSH 加密隧道送到云电脑的 80 端口。`-L 本地端口:目标主机:目标端口`。

**远程转发 `-R`**：远程端口 → 本地机器上的某个端口。场景：你本机跑着一个开发服务（如 3000 端口），想让云电脑能访问：

```bash
ssh -R 3000:localhost:3000 root@book
```

云电脑内访问 `localhost:3000`，就能到你本机的服务。

**动态转发 `-D`**：把 SSH 连接当 SOCKS5 代理，浏览器流量全走隧道：

```bash
ssh -D 1080 root@book
```

浏览器代理设为 `socks5://localhost:1080`，所有网页请求经云电脑出网。

InstaBox 的 `instabox port-forward book 8080:3000` 本质上就是平台封装的本地转发，免去你手写 SSH 命令。两者选其一即可，不要对同一端口重复转发，否则会报 "address already in use"。

---

## 本章小结

| 知识点 | 核心要点 |
| --- | --- |
| 五要素 | IP 是门牌号，MAC 是物理地址，子网掩码划局域网，网关是出口，DNS 解析域名 |
| ip 命令 | `ip addr` 看地址，`ip route` 看路由，`ip neighbor` 看邻居，`ip route get` 查路径 |
| 静态 IP | 改 `/etc/network/interfaces`，`systemctl restart networking`，远程改网配先用控制台终端 |
| DNS | `/etc/resolv.conf` 的 `nameserver`，`dig` 测试解析，DHCP 下发的配置可能被覆盖 |
| 连通性 | `ping` 测 IP 层，`traceroute` 看路径，`mtr` 持续观察，ICMP 不通不等于服务挂 |
| SSH | 密钥登录优于密码，`ssh-keygen`/`ssh-copy-id`，`~/.ssh/config` 起别名，私钥 `chmod 600` |
| 文件传输 | `scp` 单次传文件，大量文件用 `rsync` |
| 端口查看 | `ss -tlnp` 看 TCP 监听端口和进程，替代老旧 `netstat` |
| InstaBox | `port publish` 发布公网 HTTPS 链接，`port-forward` 本地转发，免费档 16 个端口 |

本章把"机器之间怎么通"讲透了。下一章（第 12 章《systemd 与服务管理》）我们回到机器内部，学习怎么管理开机启动的服务、怎么让程序稳定在后台运行。

---

## 动手练习

在自己的云电脑上完成以下练习。

**练习 1**：用 `ip addr` 查看云电脑网卡，写出主网卡名称、IP 地址、子网掩码、MAC 地址；再用 `ip route` 写出默认网关。

**练习 2**：用 `ping -c 4` 测试默认网关和 119.29.29.29，记录丢包率和平均延迟；再用 `dig example.com` 测试 DNS 解析（未安装 dig 则用 `nslookup` 或 `curl -I https://example.com`）。

**练习 3**：用 `ss -tlnp` 列出当前监听的 TCP 端口和对应进程，找出 SSH 占用的端口。

**练习 4**：在本地机器（或云电脑内另一终端）用 `ssh-keygen -t ed25519` 生成密钥，用 `ssh-copy-id` 把公钥发给云电脑的 root 账号，验证免密登录成功；在 `~/.ssh/config` 为云电脑写一个别名，用 `ssh <别名>` 连接。

**练习 5**：在云电脑里用 Python 或任意工具监听一个本地端口（如 `python3 -m http.server 8000`），用 `instabox port publish book 8000` 发布公网链接，在浏览器里访问确认可达；之后用 `instabox port-forward book 8080:8000` 在本地访问 `localhost:8080`。

---

## 参考答案

**答案 1**：

```bash
ip addr
ip route
```

预期输出示例（实际值以机器为准，不写死）：

```text
2: eth0: ... inet 10.1.2.3/24 ... link/ether 00:1a:2b:3c:4d:5e
default via 10.1.2.1 dev eth0
```

主网卡通常是 eth0，IP 如 10.1.2.3，掩码 /24，网关 10.1.2.1。

**答案 2**：

```bash
ping -c 4 10.1.2.1
ping -c 4 119.29.29.29
dig example.com
```

关键点：`0% packet loss` 表示通，`dig` 的 `ANSWER SECTION` 出现解析结果表示 DNS 正常。

**答案 3**：

```bash
ss -tlnp
```

关注 `Local Address:Port` 列和 `Process` 列，SSHD 通常监听 22 端口。

**答案 4**：

```bash
ssh-keygen -t ed25519 -C "book@instabox"
ssh-copy-id -i ~/.ssh/id_ed25519.pub root@book.cloud
ssh root@book.cloud      # 免密登录
cat ~/.ssh/config
# 写入：
# Host book
#     HostName book.instabox.cloud
#     User root
#     IdentityFile ~/.ssh/id_ed25519
ssh book                  # 用别名连接
```

注意：私钥权限 `chmod 600 ~/.ssh/id_ed25519`。

**答案 5**：

```bash
python3 -m http.server 8000 &
instabox port publish book 8000
# 浏览器打开输出的 HTTPS 链接，看到目录列表即成功
instabox port-forward book 8080:8000
# 本地浏览器打开 http://localhost:8080
```

`port publish` 返回公网链接，`port-forward` 在本地机器上监听 8080 端口，两种方式都能把云电脑内服务暴露出来。