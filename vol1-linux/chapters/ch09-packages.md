# 第 9 章 软件包管理：apt 与 dpkg

> 前八章我们学会了看文件、管进程，但有一件事还没碰：**软件是怎么进到系统里去的**。看到教程里写 `apt install nginx` 就照抄，是新手常态；抄完一出错只会重装系统，是新手困境。这一章把 apt 和 dpkg 这套「软件包管理（Package Management）」体系讲透——装、卸、查、锁、清理，全流程都覆盖。能不能管好一台生产服务器上的软件，很大程度上就看这一章的基本功。

> **本章目标**
> - 能解释「上游源码、.deb 包、APT 仓库」三层关系，并说清为什么服务器上不推荐随手 `./configure` 源码编译。
> - 能读懂 Debian 13 的 deb822 软件源配置，独立完成换源、`apt update / upgrade`、安装/卸载/清理的完整流程。
> - 能用 `dpkg -l / -L / -S / -I` 查询软件包与文件的归属，会用 `apt-mark hold` 锁定生产版本，并知道 unattended-upgrades 的作用与取舍。

<br>

## 9.1 为什么不能随便 ./configure

新手装软件的经典路径：搜索引擎点进一个博客，下载一个 `.tar.gz`，然后：

```bash
tar xzf foo-1.0.tar.gz
cd foo-1.0
./configure
make
make install
```

这套动作本身没错——它就是 Unix 世界的传统安装方式，很多经典软件至今还这么发。问题出在**服务器上**把它当默认手段。三个理由，每一个都能变成生产事故。

**第一，依赖地狱（Dependency Hell）。** foo 运行需要 libbar，libbar 又需要 libbaz，libbaz 还得是某个特定版本。你手动把 libbaz 升级了，另一个依赖旧版 libbaz 的软件立刻崩溃。更糟的是若干月后你自己都说不清这台机器上装了什么、为什么装——因为它不是从任何仓库来的，系统里没有任何记录。

**第二，没有升级路径。** 源码装完，`foo` 这个可执行文件就静静躺在 `/usr/local/bin`。上游发了安全补丁，你不知道；知道了，你得重新下载、编译、覆盖安装，还可能撞上新的编译错误。而包管理的核心能力之一就是「统一升级（upgrade）」——一条命令把机器上所有已装软件连同安全补丁一起更新。源码装的软件在这个体系之外，是补丁照不到的盲区。

**第三，卸载困难。** `make install` 往 `/usr/local` 撒了一堆文件，`make uninstall` 很多软件根本不提供，你只能手动猜哪些文件该删。删错一个共享库，系统就开始玄学崩溃。

那什么时候可以源码编译？三种正当场景：软件太新仓库里还没有；你需要开特定编译选项（比如定制 nginx 模块）；你在学编译原理。即便源码编译，也优先用 `checkinstall` 之类的工具生成一个包，让它回到包管理体系里。**服务器上的默认姿势永远是：用发行版的包管理。** 这也是 SRE 反复强调的「可管理性」——机器不是装完就完事，而是要能持续更新、审计、重建。

> **SRE 贴士（生产 vs 实验室）**：在 InstaBox 云电脑上你怎么折腾都行，反正快照一存、机器一重建就回到干净状态，这就是实验室环境给的特权。但生产服务器是「资产」：每一台装了什么、什么版本、有没有安全补丁，合规审计时都要说得出来。源码编译的软件脱离了包数据库，审计时是黑洞。哪怕生产上真编译了，也要用配置管理工具（如 Ansible，卷 2）把过程记录下来，让「重建一台一模一样的机器」成为可能。

<br>

## 9.2 三层结构：上游源码、.deb、APT 仓库

Debian 世界（以及 Ubuntu、Linux Mint 等一大票发行版）的软件分发，可以拆成清晰的三层。理解了这张图，apt 和 dpkg 的分工就不会再混。

```text
 ┌─────────────────────────────────────────────┐
 │ 第 1 层：上游源码（upstream）                 │
 │ nginx 官方、GNU、各项目作者发布的原始代码       │
 └──────────────────┬──────────────────────────┘
                    │  Debian 维护者负责「打包」：
                    │  打补丁、写控制信息、编译成 .deb
                    ▼
 ┌─────────────────────────────────────────────┐
 │ 第 2 层：.deb 软件包（package）               │
 │ 二进制文件 + 元数据（包名/版本/依赖/维护者）      │
 │ 这一层是 dpkg 的地盘                          │
 └──────────────────┬──────────────────────────┘
                    │  按「套件 + 组件」组织进仓库
                    ▼
 ┌─────────────────────────────────────────────┐
 │ 第 3 层：APT 软件源（repository）             │
 │ deb.debian.org 等全球镜像，数千个 .deb 的索引    │
 │ 这一层是 apt 的地盘                           │
 └────────────────┬────────────────────────────┘
                  │  apt update 下载索引到 /var/lib/apt/lists
                  ▼
        你的云电脑：已装软件记录在 /var/lib/dpkg/status
```

逐层说人话：

- **上游源码**：软件的「原产地」，全人类共享。Debian  mantra 是「上游 Gnome……上游说了算」——能不 patch 就不 patch，保持原汁原味。
- **.deb 包**：Debian 的软件包格式，相当于把 Windows 上的 `.msi`。一个 `.deb` 里装着：要安装的文件、依赖清单（依赖（Dependency）：这个包运行还缺谁）、以及描述、维护者等元数据。装到机器上之后，所有信息登记进 **dpkg 数据库**（`/var/lib/dpkg/status`），这是「本机装过什么」的唯一权威账本。
- **APT 软件源**：仓库（Repository）是 `.deb` 的有组织集合，按**套件（Suite）**和**组件（Component）**分类。Debian 13 的套件叫 `trixie`；组件 `main` 收录完全自由的软件，`contrib` 依赖非自由软件的自由软件，`non-free` 和 `non-free-firmware` 放不符合 DFSG 的固件和驱动。你的源配的就是「从哪个仓库、下哪些套件、要哪些组件」。

dpkg 和 apt 的关系，一句话：**dpkg 只管单机的包，apt 负责从仓库把包连依赖一起请下来再交给 dpkg 装。** 用 dpkg 装一个 `.deb`，它不知道去哪找依赖；用 apt 安装，它先看索引、算依赖、下载齐一套包，再逐层调用 dpkg。所以 apt 是「聪明的包管理器」，dpkg 是「老实的安装工」。

<br>

## 9.3 软件源配置：Debian 13 的 deb822 格式

### 9.3.1 配置文件在哪

Debian 13 的软件源主配置文件在 `/etc/apt/sources.list.d/debian.sources`。注意是 `.sources` 后缀——这是 **deb822 格式**，从 Debian 12 开始启用，用「字段: 值」的写法，比老式的 `sources.list` 更结构化、更不容易写错。老式的 `/etc/apt/sources.list` 也仍然兼容，两种格式的字段含义一模一样，你只要会读一种就行。

cat 出来看一眼（云电脑默认走 deb.debian.org 官方源）：

```bash
cat /etc/apt/sources.list.d/debian.sources
```

```text
Types: deb
URIs: http://deb.debian.org/debian
Suites: trixie trixie-updates
Components: main
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg

Types: deb
URIs: http://security.debian.org/debian-security
Suites: trixie-security
Components: main
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg
```

这是两个「段（stanza）」，空行分隔，字段对照如下：

| 字段 | 含义 |
| --- | --- |
| `Types` | 仓库类型。`deb` 是编译好的二进制包；`deb-src` 是对应的源码包，普通用户用不到。 |
| `URIs` | 仓库地址，官方源是 deb.debian.org（背后是一堆 CDN 镜像自动就近分配）。 |
| `Suites` | 套件名。`trixie` 是 Debian 13 稳定版主套件；`trixie-updates` 是发布后的非紧急更新；`trixie-security` 只放安全补丁。同一行写多个套件表示都要。 |
| `Components` | 组件。`main` 是主力；要不要 `contrib`、`non-free-firmware` 看需求，本书后续基本只用 `main`。 |
| `Signed-By` | 用哪个 GPG 公钥（GPG signature）校验仓库签名。没有它，apt 会对所有下载的包做签名验证，防止中间人篡改——Debian 的设计是「信任链不靠 HTTPS，靠 GPG」。 |

最后一行值得多说一句：apt 通过 GPG 签名（GPG 签名）确认仓库里的包确实是 Debian 签发的，不是某个镜像站偷偷塞进来的私货。这也是「随便找个野鸡源」很危险的原因——野鸡源要你 `apt-key add` 一把来路不明的密钥，等于把系统信任的大门钥匙交出去。

### 9.3.2 为什么换源：演示换成清华源

deb.debian.org 的 CDN 在海外，云电脑在国内网络下时快时慢。教学中最常用的方案是换**清华大学开源软件镜像站（TUNA）**。换源属于配置变更，原则和改 `fstab` 一样：**先备份、再验证、随时能回滚**。

> **警告**：换源不是「复制粘贴」那么简单，三条红线必须守住：一、改之前先 `cp` 备份一份（注意 `.bak` 结尾 apt 不会读，正好当备份）；二、换完必须 `apt update` 跑通才算数，报 GPG 或 404 错误就立刻回滚；三、千万别同时启用多个提供同一套件的镜像源，索引打架会引出「版本倒退」之类的怪问题。生产环境改源要走变更流程、不要直接 root 操作——本书默认 root 只是因为云电脑弄坏就能重建。

 step by step来。先备份：

```bash
cp -a /etc/apt/sources.list.d/debian.sources \
      /etc/apt/sources.list.d/debian.sources.bak
```

`-a` 保持属性原样复制。然后写入清华源：

```bash
cat > /etc/apt/sources.list.d/debian.sources <<'EOF'
Types: deb
URIs: https://mirrors.tuna.tsinghua.edu.cn/debian
Suites: trixie trixie-updates
Components: main
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg

Types: deb
URIs: https://mirrors.tuna.tsinghua.edu.cn/debian-security
Suites: trixie-security
Components: main
Signed-By: /usr/share/keyrings/debian-archive-keyring.gpg
EOF
```

注意安全更新的地址从 `security.debian.org/debian-security` 换成了 `mirrors.tuna.tsinghua.edu.cn/debian-security`——这是 TUNA 镜像 Debian security 仓库的正确路径，两个段都要换，只换一个会出现「trixie-security 找不到」的报错。

接着验证，这是换源成败的判定点：

```bash
apt update
```

```text
Hit:1 https://mirrors.tuna.tsinghua.edu.cn/debian trixie InRelease
Get:2 https://mirrors.tuna.tsinghua.edu.cn/debian trixie-updates InRelease
Hit:3 https://mirrors.tuna.tsinghua.edu.cn/debian-security trixie-security InRelease
Reading package lists... Done
```

三行源地址都是 tuna 且以 OK 收尾，就说明换源成功。不放心可以再问一条：

```bash
apt policy
```

```text
Package files:
 500 https://mirrors.tuna.tsinghua.edu.cn/debian trixie/main amd64 Packages
     release o=Debian,a=trixie,n=trixie,l=Debian,c=main,b=amd64
 500 https://mirrors.tuna.tsinghua.edu.cn/debian trixie-updates/main amd64 Packages
     release o=Debian,a=trixie-updates,n=trixie-updates,l=Debian,c=main,b=amd64
 100 https://mirrors.tuna.tsinghua.edu.cn/debian-security trixie-security/main amd64 Packages
     release o=Debian,a=trixie-security,n=trixie-security,l=Debian,c=main,b=amd64
Pinned packages:
```

如果哪天想换回官方源，一行就够：

```bash
mv /etc/apt/sources.list.d/debian.sources.bak /etc/apt/sources.list.d/debian.sources && apt update
```

### 9.3.3 常见错误：apt update 报错了怎么读

新手看到 `apt update` 一片红就慌。记住三类高频报错，都是纸老虎：

```text
E: Could not open lock file /var/lib/apt/lists/lock - open (13: Permission denied)
```

权限不足——当前用户不是 root，用 sudo（详见第 5 章《用户、群组与权限》），或者回到 `instabox ssh book` 的 root 会话。

```text
E: The repository 'https://mirrors.tuna.tsinghua.edu.cn/debian trixie Release' does not have a Release file.
N: Updating from such a repository can't be done securely, and is therefore disabled by default.
```

「没有 Release 文件」= 地址写错了，通常是 URI 路径拼错、套件名拼错（比如把 trixie 写成 trixie-update）。检查 URIs 和 Suites。

```text
W: GPG error: https://mirrors.tuna.tsinghua.edu.cn/debian trixie InRelease: The following signatures were invalid:
```

签名校验失败——大概率是 Signed-By 指向的密钥环不对，或者终端机和云电脑之间有什么东西篡改了流量。官方和 TUNA 都用同一个 `debian-archive-keyring.gpg`，正常情况不会出这错，真遇到了先检查你机器是不是被塞了代理软件。

<br>

## 9.4 apt 日常操作：更新、升级、安装、卸载

### 9.4.1 apt update 与 apt upgrade：一对最容易混的搭档

**先说人话**：`apt update` 只更新「仓库目录」，`apt upgrade` 才真正升级机器上的软件。两个都要做，顺序是先 update 后 upgrade。很多人抱怨「我 upgrade 了怎么没有新补丁」，因为他只做了后者没做前者。

```bash
apt update      # 从仓库下载最新的包索引（下载的是「目录」，不是软件本身）
apt upgrade     # 按最新索引，把已装软件升级到新版本
```

`apt upgrade` 的行为有一个重要约束：**在需要「删除已装软件」或「安装全新依赖」才能继续升级的场景下，upgrade 宁可不动**（输出里显示 "not upgraded" 或被 held back）。这在服务器上是好事——它不会在你背后悄悄删掉你的运行环境。deb 系里还有更激进的两兄弟：

```bash
apt full-upgrade    # 允许删除、安装新包，求解器给出「最完整」的方案
apt dist-upgrade    # full-upgrade 的老名字，apt-get 时代留下来的别名，行为相同
```

两者的取舍是 SRE 的基本功：**桌面机器、测试机追求最新，可以用 full-upgrade；生产服务器普通升级用 upgrade 足矣，full-upgrade 要认真读它的删除清单再拍板。** Debian 官方口径也是这个倾向。

看看 upgrade 遇到「要保持现状」时的真实表现：

```bash
apt upgrade
```

```text
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
Calculating upgrade... Done
The following packages have been kept back:
  libfoo2
0 upgraded, 0 newly installed, 0 to remove and 1 not upgraded.
```

`kept back` 的意思是：有新版，但升级它要冒出别的变更（比如要新装一个包），保守的 upgrade 方案把它按住了。想放行就用 full-upgrade，它会列出完整方案给你确认。

### 9.4.2 安装、卸载、 purging

```bash
apt install hello       # 安装单个包，自动带上依赖
apt install htop nginx  # 一次多个
```

安装过程 apt 会先给出「方案」（哪些新装、哪些升级、占多少空间），按 Y 才继续。真实输出长这样：

```bash
apt install hello
```

```text
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following NEW packages will be installed:
  hello
0 upgraded, 1 newly installed, 0 to remove and 3 not upgraded.
Need to get 55.6 kB of archives.
After this operation, 293 kB of additional disk space will be used.
Get:1 https://mirrors.tuna.tsinghua.edu.cn/debian trixie/main amd64 hello amd64 2.10-3 [55.6 kB]
Fetched 55.6 kB in 0s (196 kB/s)
Selecting previously unselected package hello.
(Reading database ... 187432 files and directories currently installed.)
Preparing to unpack .../hello_2.10-3_amd64.deb ...
Unpacking hello (2.10-3) ...
Setting up hello (2.10-3) ...
Processing triggers for man-db ...
```

（版本号、体积以你机器上的实际输出为准，不同批次镜像略有差异。）顺带说一句 `hello` 是 Debian 仓库里的「教学道具」——GNU 的经典示范程序，打印一句 hello，用来练手查包、看依赖正合适。

卸载有三个由轻到重的档位，语义必须分清：

```bash
apt remove hello        # 删软件，保留配置文件（下次装回来配置还在）
apt purge hello         # 连配置文件一起删干净（针对包生成的配置）
apt autoremove          # 删「当初被自动装进来、如今没人再依赖」的孤儿包
```

- **remove**：卸磨杀驴但留个好念想。`/etc` 下这个包的配置文件保留，状态变成 `rc`（下面 9.7 讲）。
- **purge**：斩草除根。配置文件也清掉，`dpkg -l` 里连痕迹都不剩。
- **autoremove**：大扫除利器。你 apt install A 时它顺手装的依赖 B、C，A 卸载后就没人用了，autoremove 会把它们请出去。**长期不 autoremove 的机器会悄悄囤几百 MB 到几 GB 的孤儿包**，这是「磁盘莫名变满」的高频答案之一（详见第 10 章《磁盘与存储管理》）。

常见错误：卸载一个被其他包依赖的软件，apt 会拒绝并列出「谁依赖你」：

```text
The following packages have unmet dependencies:
 nginx-core : Depends: nginx-common (= 1.28.0-1+deb13u1) but it is not going to be installed
E: Unable to correct problems, you have held broken packages.
```

不用慌，这不是系统坏了，是 apt 在保护你：**它算出你这一步会把依赖树搞崩，于是拒绝执行**。处理顺序：先看清是哪个包出的问题，`apt remove` 换成反向操作，或者 `apt --fix-broken install` 让求解器尝试修复。

<br>

## 9.5 查询软件：search、show、list

装软件之前先查，这是和「浏览器随手搜软件名」对应的正经姿势。

```bash
apt search nginx        # 按包名和描述搜
apt show nginx          # 看一个包的详细信息
apt list --installed    # 列出全部已安装（配合 grep 用）
apt list --upgradable   # 列出有新版可升的
```

`apt search nginx` 的代表性输出：

```text
Sorting... Done
Full Text Search... Done
nginx/trixie-main 1.28.0-1+deb13u1 amd64
  small, powerful, scalable web/proxy server
```

（版本号以你的机器为准。注意它默认搜的是描述里的全文，搜得太泛会刷出一大屏，习惯上加引号或收窄关键词。）

`apt show nginx` 是判断「这个包是什么、依赖谁、占多大」的权威入口：

```text
Package: nginx
Version: 1.28.0-1+deb13u1
Priority: optional
Section: httpd
Maintainer: Debian Nginx Maintainers <pkg-nginx-maintainers@lists.alioth.debian.org>
Installed-Size: 1030 kB
Depends: libc6 (>= 2.38), libpcre2-8-0, libssl3 (>= 3.0.0), nginx-common (<< 1.28.0-1+deb13u1.1~), nginx-common (>= 1.28.0-1+deb13u1)
Homepage: http://nginx.org/
Download-Size: 897 kB
APT-Sources: https://mirrors.tuna.tsinghua.edu.cn/debian trixie/main amd64 Packages
```

几个读法要点：`Depends` 这行就是「依赖（Dependency）」清单——apt install nginx 之前会先把 libssl3 这些全部备齐；`Installed-Size` 是装完占地，不是下载体积（那个是 `Download-Size`）；`APT-Sources` 告诉你是从哪个源看到的它，多源混用时用它裁判「这个包装的到底来自哪」。

`apt list --installed | grep -c ''` 之类组合能回答「这台机器到底装了多少个包」——生产审计的常见问题：

```bash
apt list --installed | wc -l
```

```text
585
```

> **SRE 贴士**：新机器上手第一件事就是 `apt list --installed` 扫一眼，把从不认识的包记下来查。很多「机器怎么变慢了」的悬案，答案就在这份清单里的某个多余服务上。另外 `apt list` 输出带 `[installed]`/`[upgradable]` 标记，可以直接用 `grep` 过滤，比 `dpkg -l` 更适合日常快查；dpkg 的长项在 9.7 讲。

<br>

## 9.6 锁定版本：apt-mark hold

生产环境有个反直觉但极其重要的原则：**不是所有包都该追新**。数据库、运行时（比如 Node.js、Docker Engine 的特定版本）往往和业务代码有耦合，一次「顺手 upgrade」把运行时小版本升了，应用第二天就可能莫名起不来。apt 的答案是 **apt-mark**：给包打上「hold（保持）」标记，apt 升级时会跳过它。

```bash
apt-mark hold nginx      # 锁定，不升了
apt-mark unhold nginx    # 解锁，恢复跟随
apt-mark showhold        # 看当前锁了哪些
```

输出简短到不值得一个输出块：

```text
nginx set on hold.
```

解锁时则是 `Canceled hold on nginx.`。`showhold` 一行一个包名列出来。锁定之后 `apt upgrade` 的输出里，这个包会出现在 "The following packages have been kept back" 里，明明白白告诉你「它没动，是你锁的」，而不是悄悄漏掉。

锁版本在生产里的典型用法：先用 apt 装好某个软件的当前版本，跑通业务后立刻 hold 住，后续升级排进变更窗口、经过测试再逐个放行。**注意 hold 只挡 apt 自己的 upgrade，挡不住你手贱 apt install 指定版本覆盖，也挡不住源码装的同名软件互相 shadow——真正负责的做法是把「该锁哪些、为什么锁」写进变更文档。**

> **SRE 贴士（生产 vs 实验室）**：实验室里你的座右铭是「全都要最新」，因为环境弄坏就重建；生产上的座右铭是「没有理由不动的东西就别动」，因为每次升级都是有风险的变更。version pinning（版本锁定）还有更精细的玩法——给某个源或某个包版本号打优先级（apt preferences），那是配置管理（卷 2）的领域，本章记住 `apt-mark hold` 这个开关就够入门了。

<br>

## 9.7 dpkg：单机包管理的老黄牛

apt 再聪明，底层装包的还是 dpkg。更要紧的是：**当 apt 都解决不了问题、机器处于半坏状态时，能干活的往往只剩 dpkg**。所以 dpkg 四个查询参数必须刻进肌肉记忆。

```bash
dpkg -l nginx       # 看包的安装状态
dpkg -L nginx       # 看包装了哪些文件（L = List）
dpkg -S /usr/bin/hello   # 反查：某个文件是哪个包装的（S = Search）
dpkg -I hello_2.10-3_amd64.deb   # 看一个 .deb 文件的元信息（I = Info）
```

先看 `dpkg -l`，它的表头先别跳过——那是整个 Debian 包状态机的压缩图：

```text
Desired=Unknown/Install/Remove/Purge/Hold
| Status=Not/Inst/Conf-files/Unpacked/halF-conf/Half-inst/trig-aWait/Trig-pend
|/ Err?=(none)/Reinst-required (Status,Err: uppercase=bad)
||/ Name           Version        Architecture Description
+++-==============-==============-============-=================================
ii  hello          2.10-3         amd64        example package based on GNU hello
ii  nginx-common   1.28.0-1+deb13u1 amd64      common files and configuration for nginx
rc  libfoo1        1.0-1          amd64        foo runtime library (residual config)
un  zsh            <none>         amd64        shell with lots of features
```

（版本号以你的机器为准。）最实用的是每行开头那两个字母，这叫**状态两列**：

| 状态 | 含义 | 要不要管 |
| --- | --- | --- |
| `ii` | 第一列 `i`=要求安装；第二列 `i`=已装好且配置完成 | 正常状态 |
| `rc` | 第一列 `r`=你 remove 过；第二列 `c`=配置文件（Conf-files）还在 | 想彻底干净就 `apt purge` |
| `un` | `u`=从未安装（unknown），只是被索引提及 | 不用管 |
| `iU` / `iF` | 装了一半或配置失败（半坏状态） | 要处理：`apt --fix-broken install` |
| `hi` | 第一列 `h`=hold | 你锁过版本 |

日常体检只需盯两个状态：前者是没卸干净的配置残留，后者是没装完的半成品。

`dpkg -L` 回答「这包装的东西都在哪」，卸载前拿它核对，做清单比对也用它：

```text
dpkg -L hello
```

```text
/.
/usr
/usr/bin
/usr/bin/hello
/usr/share
/usr/share/doc
/usr/share/doc/hello
/usr/share/doc/hello/changelog.Debian.gz
/usr/share/doc/hello/copyright
/usr/share/man
/usr/share/man/man1
/usr/share/man/man1/hello.1.gz
/usr/share/locale
...
```

`dpkg -S` 反着用：看到一个陌生文件，想知道是哪来的。这是运维排障的高频动作——比如某天发现 `/usr/bin/htop` 不知道谁装的：

```bash
dpkg -S /usr/bin/htop
```

```text
htop: /usr/bin/htop
```

注意它会精确匹配文件路径。记不全路径时用通配符：`dpkg -S '*htop*'`。

`dpkg -I` 是装包前的尽调：从网上下载的 `.deb`，先看它声明了什么、依赖什么，再决定装不装：

```bash
dpkg -I ./hello_2.10-3_amd64.deb
```

```text
 new Debian package, version 2.0.
 size 55800 bytes: control archive=1048 bytes.
     728 bytes,    12 lines      control
     246 bytes,     3 lines      md5sums
     320 bytes,    14 lines   *  postinst             #!/bin/sh
 Package: hello
 Version: 2.10-3
 Architecture: amd64
 Maintainer: Debian Games Team <pkg-games-devel@lists.alioth.debian.org>
 Installed-Size: 293
 Depends: libc6 (>= 2.38)
 Section: games
 Priority: optional
 Homepage: http://www.gnu.org/software/hello/
 Description: example package based on GNU hello
  The GNU hello program produces a familiar, friendly greeting.
  It also provides an examples of ...
```

读到 `Depends` 就是要动手的信号：dpkg -i 不会给你准备依赖，先 `apt show libc6` 之类的确认齐了没。

<br>

## 9.8 安装本地 .deb：dpkg -i 与 apt -f install 的标准舞步

什么时候会装本地 `.deb`？三种情况：官方只提供 deb 的专有软件（比如某些数据库、远程桌面客户端）；内网环境不通外网，用离线 deb 分发；软件还没进 Debian 仓库。标准流程是固定的两步：

```bash
dpkg -i ./mytool_1.0_amd64.deb    # 第一步：dpkg 解包安装
apt --fix-broken install          # 第二步：apt 补依赖
```

> **警告**：两步都不能省。dpkg 是「老实的安装工」，它只管把包解开装到文件系统，**不会下载依赖包**。如果第一步报依赖错误后就此停手不去修复，系统里就留下一个「解压了但配置失败（iU 状态）」的半成品包——轻则那个软件不能用，重则影响依赖它的其他软件。安全姿势：只从软件官网或可信渠道下载 `.deb`（网上搜来的 deb 站点是恶意软件高发区）；装之前先 `dpkg -I` 看元信息；install 报错先跑 `apt --fix-broken install` 修复，别反复重装。生产环境别直接 root 操作这些，交给带审计的配置管理流程。

先看失败的样子——这是新手装本地 deb 最常见的翻车：

```bash
dpkg -i ./mytool_1.0_amd64.deb
```

```text
Selecting previously unselected package mytool.
(Reading database ... 187432 files and directories currently installed.)
Preparing to unpack mytool_1.0_amd64.deb ...
Unpacking mytool (1.0) ...
dpkg: dependency problems prevent configuration of mytool:
 mytool depends on libbar1 (>= 2.0); however:
  Package libbar1 is not installed.
 mytool depends on libbaz2; however:
  Package libbaz2 is not installed.

dpkg: error processing package mytool (--install):
 dependency problems - leaving unconfigured
Errors were encountered while processing:
 mytool
```

注意最后一句 `leaving unconfigured`——包解压了（文件已经落地），但没配置完成。这时 `dpkg -l mytool` 会看到 `iU` 状态。修复就是第二步：

```bash
apt --fix-broken install
```

```text
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
Correcting dependencies... Done.
The following additional packages will be installed:
  libbar1 libbaz2
The following NEW packages will be installed:
  libbar1 libbaz2
0 upgraded, 2 newly installed, 0 to remove and 3 not upgraded.
Need to get 1,204 kB of archives.
Get:1 https://mirrors.tuna.tsinghua.edu.cn/debian trixie/main amd64 libbar1 amd64 2.0-1 [614 kB]
Get:2 https://mirrors.tuna.tsinghua.edu.cn/debian trixie/main amd64 libbaz2 amd64 1.3-2 [590 kB]
Fetched 1,204 kB in 1s (1,006 kB/s)
Selecting previously unselected package libbar1.
(Reading database ... 187434 files and directories currently installed.)
Preparing to unpack .../libbar1_2.0-1_amd64.deb ...
Unpacking libbar1 (2.0-1) ...
Selecting previously unselected package libbaz2.
Preparing to unpack .../libbaz2_1.3-2_amd64.deb ...
Unpacking libbaz2 (1.3-2) ...
Setting up libbar1 (2.0-1) ...
Setting up libbaz2 (1.3-2) ...
Setting up mytool (1.0) ...
Processing triggers for man-db ...
```

apt 自动算出了缺的两个依赖、从仓库装好、最后把 mytool 配置完成。再 `dpkg -l mytool` 一看，状态回到 `ii`。**记住这个舞步，比记任何一条命令都值钱。**

> **SRE 贴士**：内网离线装机的标准做法，就是在能上网的机器上 `apt-get download 包名`（只下载不安装）攒一批 `.deb`，再用 `dpkg -i *.deb && apt --fix-broken install` 灌到内网机器上。apt 装过的 deb 下载缓存默认在 `/var/cache/apt/archives`，`apt clean` 清掉之前，那里就是一个现成的离线弹药库（下一节讲空间管理时你会用到这点）。

<br>

## 9.9 缓存与磁盘空间：apt clean 的时机

apt 下载的每个 `.deb` 装完后并不会自动删除，而是留在 `/var/cache/apt/archives` 里。设计初衷是好的：卸载重装时不用再下一遍。但对磁盘紧张的环境，这就是一笔沉默的占用——机器用久了这里躺着几百 MB 到几个 GB 很常见。

```bash
apt clean        # 清空整个 /var/cache/apt/archives（下载缓存全删）
apt autoclean    # 只删「仓库里已经没有对应新版」的过期 deb，温和版
```

`clean` 是「全删重来」，`autoclean` 是「删过期、留可能还有用的」。日常维护用 autoclean 就够；系统盘（你的云电脑上只有 16 GiB）告急时直接 `apt clean`。

结合我们第一章就记住的事实：InstaBox 云电脑的**持久盘 20 GiB 挂在 `/data`，系统盘只有 16 GiB ext4**。apt 缓存、Docker 镜像、构建产物最爱堆系统盘，这就是为什么第 1 章的铁律说「数据放持久盘，别往系统盘堆东西」。养成几个和本章直接相关的空间习惯：

```bash
df -h /                 # 先看系统盘用了多少（详见第 10 章）
du -sh /var/cache/apt/archives   # 缓存到底占多少
apt autoclean           # 日常清理
apt autoremove          # 顺手清孤儿包（9.4.2 讲过）
```

> **SRE 贴士（生产 vs 实验室）**：实验室里磁盘满了 `apt clean` 一把梭；生产服务器上清缓存只是缓兵之计，真正的答案是「监控 + 根源治理」——软件包缓存大头往往来自频繁的构建或 CI，要么挪到独立分区、要么镜像缓存，要么上镜像仓库而不是每台机器各下一遍。另外重申一次：**本章所有安装卸载动作，实验室以 root 演示没问题，生产上请用普通账号 + sudo，让每一次 apt 操作都留痕在 auth.log 里**（审计日志详见第 14 章《定时任务与系统日志》）。

<br>

## 9.10 Snap 与最小安装原则

在 Ubuntu 的教程里你会频繁见到 `snap install`。Snap 是 Canonical 推出的另一套打包分发体系，自带沙箱、自更新、跨发行版。但本书的立足点是 Debian：**Debian 13（trixie）默认不带 snapd 也没有 snap 生态，日常 95% 的软件 apt 都能装（装不了的先找 Flatpak、上游 .deb、源码编译的替代品，而不是急着 snap install）**。知道「Ubuntu 世界有 snap、Debian 世界基本是 apt 的地盘」这个差异，是为了让你在看跨发行版教程时心里有数：命令不能照抄，先看目标机器是什么发行版、用什么包管理器。

比「知道有几种格式」更重要的，是 SRE 的一条核心原则：**最小安装原则（Minimal Installation）——生产服务器上装的包越少越好。** 每个装上的包都意味着：更多补丁要跟、更大攻击面、更长的 CVE 响应清单。

最小安装的具体做法：装完系统先 `apt list --installed` 盘点；装新软件前问自己「这个功能是不是系统自带的、是不是非它不可」；能 `apt install --no-install-recommends`（不自动拉 recommends 建议包）就加上，推荐包经常是一串用不上的大家伙；定期 `apt autoremove` + `apt autoclean`。装得少，不是因为抠磁盘，是因为**每一个包都是一份要持续维护的责任**。

## 9.11 unattended-upgrades：安全更新的自动挡

Debian 的安全补丁来得勤，人工跟总有惰性，unattended-upgrades 就是官方的「自动安装安全更新」方案。它默认随系统安装（`dpkg -l unattended-upgrades` 能看到 `ii`），核心配置在 `/etc/apt/apt.conf.d/50unattended-upgrades`，由 `apt-daily-upgrade.timer` 定时唤醒（systemd 定时器详见第 12 章《systemd 与服务管理》）。

它装了什么、跑没跑，一条命令看状态：

```bash
systemctl list-timers apt-daily-upgrade.timer
```

```text
NEXT                        LEFT     LAST                        PASSED     UNIT                ACTIVATES
Wed 2026-11-18 06:11:24 CST 5 days   Tue 2026-11-17 06:11:19 CST 7h ago    apt-daily-upgrade.timer apt-daily-upgrade.service
```

它的升级记录落在 `/var/log/unattended-upgrades/`，想看自动装过什么：

```bash
tail -n 8 /var/log/unattended-upgrades/unattended-upgrades.log
```

```text
2026-11-11 06:14:02,123 INFO Initial blacklisted packages:
2026-11-11 06:14:02,124 INFO Starting unattended-upgrades script
2026-11-11 06:14:02,125 INFO Allowed origins: ['origin=Debian,codename=trixie-security,label=Debian-Security']
2026-11-11 06:15:31,447 INFO Packages that will be upgraded: openssl libssl3
2026-11-11 06:15:41,552 INFO All upgrades installed
```

要不要开、开到多大，是一个**分层决策**：

- **只开安全更新（默认档）**：`trixie-security` 源的补丁自动装，其余一律等人工。生产服务器最推荐的姿势——安全补丁不等人，功能更新要走变更窗口。
- **全自动**：所有源的更新都自动装。省事，但一次有问题的上游升级也可能让业务挂掉，除非你有完善的监控兜底，否则不推荐生产这么干。
- **实验室/云电脑**：不开也能接受，因为你本来就会手动 `apt upgrade`，机器还随时可重建。

> **SRE 贴士（生产 vs 实验室）**：自动安全更新不是「开了就万事大吉」。真正的生产姿势是：开 `trixie-security` 自动更新 + 关键服务（数据库、容器运行时）该 hold 的 hold 住（9.6 节）+ 用监控盯服务存活 + 保留「变更记录」。自动补丁也可能和锁定策略打架——比如你 hold 了某个包、它的安全补丁恰好在这个包里，这种情况要么接受风险并记录，要么安排人工变更窗口，别两头都想占。

<br>

## 本章小结

这一章把软件从「仓库」到「本机」的全流程走了一遍，核心是三层结构：**上游源码 → .deb 包（dpkg 的地盘）→ APT 源仓库（apt 的地盘）**，以及这对搭档的分工——apt 聪明（算依赖、下 Index、管升级），dpkg 老实（只会解包装包）。

日常最高频的操作其实只有七条，按顺序记：

```bash
apt update && apt upgrade   # 日常升级：先更目录，再升软件
apt install 包名             # 安装；装前先 search / show
apt remove / purge 包名      # 卸载两档；之后 apt autoremove 清孤儿
apt search / show / list    # 查询三件套
apt-mark hold / showhold    # 生产锁版本
dpkg -l / -L / -S / -I      # 状态、文件清单、反查归属、本地 deb 尽调
dpkg -i xxx.deb + apt --fix-broken install   # 本地 deb 两步舞
```

再记三条心法：**源码编译是例外不是常态**（可管理性优先）；**能 apt 装的不手编，装得越少越好**（最小安装）；**换源和改配置一个待遇——先备份、再验证、可回滚**。

## 动手练习

1. 找出你机器上软件源配置文件的位置，用 cat 读懂每一个字段；然后按 9.3.2 的流程备份并换成清华源，`apt update` 验证成功后，再换回官方源。
2. `apt search` 找一个小工具（比如 `tree`），`apt show` 看它的依赖和体积，然后 `apt install` 装上并实际运行一次，最后 `apt remove` 卸掉、观察配置文件残留，再 `apt purge` 彻底清干净。
3. 用 `dpkg -l | grep '^rc'` 找出所有「配置残留」的包，用 `apt purge` 逐个清理；再用 `dpkg -l | grep -v '^ii' | grep -v '^rc'` 检查有没有 `iU` 之类的半坏状态（没有就是对的）。
4. 用 `dpkg -S` 反查 `/usr/bin/env` 是哪个包装的，再用 `dpkg -L` 列出同一个包的所有文件，说说这个包在系统里的角色。
5. 对 `hello` 包执行 `apt-mark hold`，跑一次 `apt upgrade` 观察 kept back 输出，再 `apt-mark unhold` 恢复。
6. 综合题：`df -h` 看系统盘占用，`du -sh /var/cache/apt/archives` 看 apt 缓存多大，然后 `apt autoclean` 后再看一次两个命令的输出，把变化记下来。

> **参考答案**：
> 1. `/etc/apt/sources.list.d/debian.sources`；字段含义见表；备份用 `cp -a ...bak`；换回用 `mv ...bak debian.sources && apt update`。
> 2. 顺序：`apt search tree` → `apt show tree` → `apt install tree` → `tree /data/home` → `apt remove tree`（`dpkg -l tree` 显示 `rc`）→ `apt purge tree`（状态消失）。
> 3. `dpkg -l | grep '^rc' | awk '{print $2}'` 拿到包名列表；`apt purge <包名>` 清理；半坏状态出现时用 `apt --fix-broken install` 修复。
> 4. `dpkg -S /usr/bin/env` 指向 `coreutils` 包（以你的机器为准）；`dpkg -L coreutils` 会列出几十个基础命令——它是「GNU 核心工具集」，系统最基础的一批命令都来自它。
> 5. hold 后 `apt upgrade` 输出里出现 `The following packages have been kept back: hello`；unhold 后恢复正常。
> 6. 缓存变小、`apt autoclean` 只删过期 deb；空间变化以你的机器为准，重点是把「apt 缓存也是磁盘占用大户」建立成条件反射。



