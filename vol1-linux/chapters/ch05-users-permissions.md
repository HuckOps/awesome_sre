# 第 5 章 用户、群组与权限管理

> 第 4 章我们把文件系统的"物"玩熟了：建目录、拷文件、删文件。但你有没有意识到一件事——那台云电脑上，你一直是 root，想删谁删谁、想改谁改谁，畅通无阻。而真实世界的服务器绝不是这样：几十个开发共用一台机器，数据库账号碰不到别人的目录，Web 服务的进程连读 `/root` 的资格都没有。从"一个人一台机器"跨到"一群人一台机器"，靠的就是本章的三个字——用户、群组、权限。这套机制是 Linux 安全的地基，也是 SRE 日后做加固、排权限故障、写自动化脚本的基本功。

> **本章目标**
> - 能解释服务器为什么必须分用户（最小权限、责任可追溯、一处沦陷不沦全局），并用自己的话讲清 Linux 的"用户—群组—其他人"三级访问模型。
> - 能在自己的云电脑上创建普通用户 `bookuser`、把它加入 `sudo` 组、用它完成日常操作，并在出问题时用 `id`/`groups`/`who`/`last`/`lastb` 查清"谁干的"。
> - 能逐位念出 `ls -l` 的权限位（`-rwxr-xr-x`）、完成八进制与符号两种 `chmod` 写法，说清 umask 022 与 755/644 的关系，并用实验解释目录 `x` 位的真实意义和 SUID/SGID/Sticky 三个特殊权限。

---

## 5.1 为什么服务器必须分用户

先给你一张真实得有点扎心的图景：一台生产服务器上，三个开发共用一个 root 账号。某天服务器被入侵了，安全团队要回答三个问题——谁干的？从哪进来的？动了什么？运维的回答是："不知道、查不到、没日志。" 因为所有操作都记在同一个 root 头上，像三个室友共用一张信用卡，谁刷的卡永远说不清。

**服务器分用户，至少解决三个问题，每一个都直接对应一类真实事故：**

**第一，最小权限（Least Privilege）。** 每个账号只拥有完成它工作所必需的最小权限，不多给一分。Web 服务的账号不需要读 `/root` 下的东西，备份账号不需要登录 Shell，实习生的账号不该能 `userdel`。权限越小，一个账号沦陷时能造成的破坏就越小——你把炸弹的引线剪短，而不是指望没人去点它。

**第二，责任可追溯（Accountability）。** 每个操作都能落到一个具体的人/服务头上。出事后能回答"谁在什么时间做了什么"，审计（Audit）和溯源才有可能。共用 root 等于把系统的黑匣子摔碎了。

**第三，一处沦陷，不沦全局（Blast Radius Control，爆炸半径控制）。** 这是 SRE 讲安全时最爱用的词：一个组件被攻破，影响范围应该被限制在局部。用户边界就是最原始、最有效的一道隔离墙——入侵者拿到了 `www-data` 的权限，哪怕为所欲为，也动不了 `/etc/shadow`，也登不上别的用户的会话。

> **生产 vs 实验室**：本书的实验云电脑里，你从头到尾都是 root，命令也默认以 root 演示——这是"实验室环境、弄坏就重建"的便利设计。但请从本章开始建立一个反射：**root 是底线工具，不是日常工具。** 生产服务器上直接拿 root 敲命令是审查时的扣分项，理由本章 5.7 节会展开（没有审计、没有缓冲、手滑即事故）。本章 5.5 节我们就在云电脑上创建 `bookuser`，让你亲手体会"平时用普通账号、必要时提权"的正确姿势。

## 5.2 多用户模型总览：用户、群组、其他人

Linux 的多用户模型，用一句话就能说完：**每个对象（文件/目录/进程）都归属于一个用户和一个群组；每个想访问它的主体，按"属主 → 属组 → 其他人"的顺序对号入座，就近获得对应的那一组权限。**

拆开看三层：

| 层级 | 是谁 | 直觉理解 |
| --- | --- | --- |
| 属主（user，u） | 文件的主人，通常是创建者 | "这是我的东西" |
| 属组（group，g） | 文件主人所在的（某一个）群组的成员 | "这是我们小组的东西" |
| 其他人（others，o） | 系统里除了上面两类之外的所有账号 | "外人" |

两个常见直觉误区先拆掉：

- **"创建者就是属主"不总成立。** 文件属主默认是创建它的用户没错，但 root 可以通过 `chown` 把文件送给别人（5.11 节），服务以 `www-data` 身份运行，它创建的文件属主就是 `www-data`，不是某个具体的人。
- **"一个用户只在一个组里"也不对。** 一个用户有一个**主组（primary group）**（`/etc/passwd` 里记录），同时可以参加任意多个**附加组（supplementary groups）**。判断"某人能不能访问某文件"时，三层的匹配顺序是**就近优先**：如果来访者正是属主，就用属主权限判定，看都不看群组权限；不是属主、但是属组成员，用属组权限；两者都不是，用其他人权限。这个顺序是理解后面所有"怪现象"的钥匙。

---

## 5.3 账号的档案库：/etc/passwd 与 /etc/group

### 5.3.1 /etc/passwd：七个字段，逐列念一遍

说人话：**这台机器上"有哪些人、每个人长什么样"，全部登记在一个纯文本的表格文件里，就是 `/etc/passwd`（password 文件）。** 每行一个用户，七个字段用冒号分隔：

```bash
head -12 /etc/passwd
```

```text
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
lp:x:11:7:lp:/var/spool/lpd:/usr/sbin/nologin
```

拿第一行 `root:x:0:0:root:/root:/bin/bash` 逐字段念，这七个字段每个版本 Linux 都一样：

| 字段位置 | 本例的值 | 字段名 | 含义 |
| --- | --- | --- | --- |
| 1 | `root` | 用户名（username） | 登录时敲的名字 |
| 2 | `x` | 密码占位符 | **注意这里不是密码本身**，是个"到别处查"的路标：真正的哈希在 `/etc/shadow`（5.4 节）。历史上密码就明文放在这，后来发现谁都能读文件、等于谁都能拆密码，才改成这个机制 |
| 3 | `0` | UID（User ID，用户号） | 内核只认数字不认名字。**UID 0 的用户就是 root**，可以有两个用户名都映射到 0，那是后门不是配置 |
| 4 | `0` | GID（Group ID，群组号） | 该用户的**主组**编号，这里是 root 组 |
| 5 | `root` | 注释（GECOS/comment） | 随便写，通常是真人姓名或用途；用 `chfn` 可改 |
| 6 | `/root` | 家目录（home directory） | 登录后落在哪、`cd ~` 去哪 |
| 7 | `/bin/bash` | 登录 Shell（login shell） | 登录后跑哪个壳。写成 `/usr/sbin/nologin` 的账号（`daemon`、`lp` 这些）**明确禁止交互登录**——它们是给服务进程用的"影子账号" |

几个必须建立的条件反射：

- **UID 看段位。** 0 是 root；1—999（不同发行版边界略有差异）一般是系统账号/服务账号；1000 及以上才是给你我这样的普通人创建的账号。所以你 `useradd` 出来的用户，UID 从 1000 起跳。
- **UID 才是身份证，用户名只是花名。** 文件系统里记的是"UID 1000 拥有此文件"。哪天你把 `bookuser` 改名成 `bookadmin`，UID 不变，它名下的文件照样认它——内核自始至终只认数字。
- **一台机器 N 个用户 ≠ N 个人。** 每装一个服务（nginx、mysql、backup 工具）往往自动创建一个影子账号，比如 `www-data`。你用 `wc -l /etc/passwd` 数一下，一台"没人用过"的干净机器通常也有二三十个用户。

### 5.3.2 /etc/group：群组花名册

群组信息在 `/etc/group`，每行一个组，四个字段：`组名:组密码占位:GID:成员列表`。

```bash
getent group sudo
```

```text
sudo:x:27:bookuser
```

逐字段：`sudo` 是组名；`x` 同上，组密码机制早已废弃（占位而已）；`27` 是 GID；最后一个字段是**成员列表，用逗号分隔**——这里的成员指的是"把该组当作附加组"的用户。注意一个容易迷惑的细节：**一个用户的主组不会出现在这个成员列表里**。`bookuser` 的主组是 `bookuser`（GID 1000），所以你看 `/etc/group` 里：

```bash
getent group bookuser
```

```text
bookuser:x:1000:
```

成员列表是**空的**。这不是 `bookuser` 不在这个组——它当然在主组里；花名册只记"外人怎么进来的"，不记"原生家庭"。

**为什么需要组？** 因为权限按"用户"颗粒度太粗：一个十人开发团队要共享一个项目目录，按用户授权就要授权十次；建一个 `devteam` 组、把十个人塞进去、授权一次，以后人员进出只改组、不动权限。组就是"批量发许可证"的制度。

> **SRE 贴士**：`getent`（get entries，从系统数据库取记录）比直接 `cat /etc/passwd` 更可靠——它统一查询 `/etc/passwd`、NIS、LDAP（目录服务）、systemd 的 NSS 配置，是"系统最终认谁"的最终答案。查询用户时用 `getent passwd 用户名`、查组时用 `getent group 组名`，这是排查"用户到底存不存在"的金标准。直接 cat 文件只是"配置文件里写了什么"，两者在接入集中式账号系统的机器上可能不一样。

---

## 5.4 /etc/shadow：密码与密码策略的藏身之处

### 5.4.1 只读展示：这个文件，普通人没资格读

既然密码哈希不放 `/etc/passwd` 了，那放哪？放 `/etc/shadow`。看它的权限你就懂了：

```bash
ls -l /etc/passwd /etc/shadow
```

```text
-rw-r--r-- 1 root root  2060 Nov 13 09:11 /etc/passwd
-rw-r----- 1 root shadow 1180 Nov 13 09:11 /etc/shadow
```

`/etc/passwd` 是 `644`：全机器可读（这个必须可读，`ls -l` 要把 UID 翻译成用户名、`whoami` 之类的命令都要查它）。`/etc/shadow` 是 `640`，属组 `shadow`：**只有 root 和 shadow 组成员能读，普通用户连瞄一眼的资格都没有。**

**这就是设计的精髓**：把"需要公开的信息"（用户名、UID、家目录）和"需要保密的信息"（密码哈希、密码策略）分到两个文件，前者开放、后者锁死。早期把所有东西混在 `/etc/passwd` 里，等于把全班同学的密码贴公告栏——那时候暴力破解（暴力枚举候选密码撞哈希）一台机器，成本低得像抄作业。

以 root 身份只读地看一眼：

```bash
head -5 /etc/shadow
```

```text
root:$6$3dE9aBc1$Gh7Qy...wZ2Nf1kP9sR0tLmX4bV7cJ8d:H20688:0:99999:7:::
daemon:*:20688::::::
bin:*:20688::::::
sys:*:20688::::::
sync:*:20688::::::
```

**只看不摸。** 下面按冒号分列解释第一行：

| 字段 | 本例的值 | 含义 |
| --- | --- | --- |
| 1 登录名 | `root` | 与 `/etc/passwd` 对应 |
| 2 密码哈希 | `$6$3dE9aBc1$Gh7Qy...` | 密码的加盐哈希，不是密码本身。**记住 `$6$` 这个前缀——它表示算法为 SHA-512**（`$1$` 是早期的 MD5、`$5$` 是 SHA-256、`$2y$` 是 bcrypt；新一些的系统默认算法还可能不同，认前缀就能判断） |
| 3 上次改密日期 | `20688` | 从 1970-01-01 算起的天数。20688 天换算一下是 2026 年附近 |
| 4 最少修改间隔 | `0` | 改完密码后至少过几天才允许再改，0 表示不限制 |
| 5 密码有效期 | `99999` | 最多多少天必须改一次密码。99999 天约等于 273 年，即"实际上永不过期" |
| 6 到期前警告 | `7` | 到期前 7 天开始每次登录都催你改密 |
| 7 不活跃宽限 | 空 | 密码过期后几天内不登录就锁账号（service account 常用） |
| 8 账号过期日 | 空 | 从 1970-01-01 算起到期，比如某外包账号到期自动失效 |

一眼识别的几个"特殊值"：

- `daemon:*` ——第二字段是 `*`，表示**这个账号没有密码、也永远不能用密码登录**（服务账号常态）。
- 第二字段是 `!` 或 `!` 开头的字符串，表示账户被**锁定（locked）**（5.6 节讲 `passwd -l`）。
- 第二字段为空，表示**免密登录**——极其危险，生产环境视为事故隐患。

### 5.4.2 passwd -S：官方给的"密码状态查询器"

记不住那串字段不要紧，shadow 工具链自带翻译器：

```bash
passwd -S bookuser
```

```text
bookuser P 08/15/2026 0 99999 7 -1 -1
```

逐段念：`bookuser` 用户名；`P` 密码状态（`P`=usable password 可用密码，`L`=locked 已锁定，`NP`=no password 无密码）；`08/15/2026` 上次改密日期（月/日/年）；后面的四个数字就是上次改密到期的策略三连 + 两个宽限字段。**以后判断"这账号是不是被锁了"，`passwd -S` 一条命令搞定，不用去研读 shadow 文件。**

> **警告**：**永远不要用 vim 直接编辑 `/etc/shadow`、`/etc/passwd`、`/etc/sudoers` 这类文件（5.7 节专门讲为什么 sudoers 必须走 visudo）。** 直接用编辑器改 shadow 有四个后果：没有语法校验、没有加锁保护（两个进程同时写能把文件写残缺）、行内一个字符打错就可能让**全场所有人登录不进来**、且改了没有日志。正确姿势是永远走配套命令：改账号用 `usermod`/`chage`，改密码用 `passwd`，它们负责校验、加锁、写日志。万一你真的把 shadow 写坏了，最稳的恢复路径是从快照重建机器（第 1 章讲的快照机制），而不是现场手撕文件。

---

## 5.5 账号管理：useradd / groupadd / usermod / userdel

### 5.5.1 useradd：开一个普通用户 bookuser

说人话：**`useradd`（user add，添加用户）就是"在 /etc/passwd、/etc/shadow、/etc/group 里登记一行，并按需把家目录建好"**。

```bash
useradd -m -s /bin/bash bookuser
```

逐参数：

| 参数 | 含义 |
| --- | --- |
| `-m` | 按骨架目录创建家目录 `/home/bookuser`（**不加 `-m` 就不建**，这是新手第一大坑） |
| `-s /bin/bash` | 指定登录 Shell 为 bash。Debian 13 新用户的默认 Shell 由 `/etc/adduser.conf` 的 `DSHELL` 决定（Debian 上默认就是 bash），但显式写出来是更稳的习惯 |
| `bookuser` | 要创建的用户名。命名规矩：小写字母开头，可含数字、下划线、连字符，别用 `.` 开头、别用中文 |

执行它**一声不吭就是成功**——Linux 的命令大多是"没消息就是好消息"。查一下成果：

```bash
getent passwd bookuser
```

```text
bookuser:x:1000:1000:,,,:/home/bookuser:/bin/bash
```

UID/GID 都是 1000（本机第一个普通用户），家目录 `/home/bookuser`，Shell `/bin/bash`，注释段有三个逗号占位（那是保留给办公室电话、房间号之类的 GECOS 字段，空着即可）。顺手把家目录的归属确认一下：

```bash
ls -ld /home/bookuser
```

```text
drwxr-xr-x 2 bookuser bookuser 4096 Nov 13 09:20 /home/bookuser
```

属主、属组都是 `bookuser` 自己，权限 `755`——下一节你会知道这组数字念作"主人全权、其他人只读可进"。

> **警告**：`userdel -r`（连同家目录一起删除）是不可逆的毁灭操作：用户目录里的文件、邮件、配置**一去不回**。安全替代做法：员工离职/账号回收的标准流程不是"删"，而是先"锁"再归档——`usermod -L bookuser`（或 `passwd -l`）禁止登录 → 确认无人使用数月 → `tar` 归档家目录到 `/data/home/archive/` → 最后才 `userdel -r`。**先锁、再备、后删，命令可以一条条敲，数据不能一键没。**

> **SRE 贴士**：Debian 上还有一个 `adduser`（没有空格、是 Perl 写的"高级封装"）命令，交互式地问密码、全名，一步步带你走完，还会自动建组、拷骨架文件、提示设密码。新手友好，但**不可脚本化**；写自动化一律用低层命令 `useradd` + `chpasswd`/`passwd -e` 组合。两者的命名是新人著名困惑点：别的发行版（RHEL 系）`adduser` 就是 `useradd` 的符号链接，Debian 的 `adduser` 是两码事，看到时先 `which adduser` 或 `ls -l $(which adduser)` 认脸。

### 5.5.2 groupadd：建一个团队群组

```bash
groupadd devteam
```

同样静默成功，验证：

```bash
getent group devteam
```

```text
devteam:x:1001:
```

GID 顺次分配。把 `bookuser` 拉进团队（**注意 `-aG` 的 `-a`，意为 append 追加，绝不能省**）：

```bash
usermod -aG devteam bookuser
```

`usermod`（修改用户）的常用参数：`-aG 组1,组2` 追加附加组；`-s /bin/zsh` 改登录 Shell；`-d /data/home/bookuser -m` 搬家目录并把文件搬过去；`-L` / `-U` 锁定/解锁账户；`-e 2027-01-01` 给账号设到期日。

**这里埋着全章最经典的坑：`-a` 省略事故。** 写成 `usermod -G devteam bookuser`（少了 `-a`），含义变成"把用户的附加组**整体替换**为 devteam 一个组"——他原来在 `sudo` 组、`docker` 组，统统被踢出去。症状是"我明明加了组，反而丢了别的权限"，且不报任何错。口诀：**改组必带 `-aG`，改密码必带用户，删目录必带 `-r`。**

### 5.5.3 userdel：销户，以及那个危险的 -r

```bash
userdel bookuser2
```

只删账号记录，**家目录原样留着**。要连家目录一起端：

```bash
userdel -r bookuser2
```

`-r`（remove）同时删除家目录和邮件池。下面这种输出在真实运维里天天见，要学会读：

```text
userdel: bookuser2 mail spool (/var/mail/bookuser2) not found
```

别被这行唬住：它不是说"删除失败"，只是在说"这台机器没有他的邮件文件，没什么可删的"——账号和家目录通常已经删掉了。判断成败看退出码（`echo $?`，0 为成功）和 `getent passwd bookuser2` 有没有输出。

另一个常见新手事故：**不给 `-r` 就以为"删干净了"，结果 `/home/bookuser2` 变成无人认领的孤儿目录**，占着磁盘，下一个人创建同名用户时还可能撞上一堆历史残留文件。

---

## 5.6 改密码与密码策略：passwd 与 chage

### 5.6.1 passwd：改密，以及"输入时不显示"的真相

```bash
passwd bookuser
```

```text
New password:
Retype new password:
passwd: password updated successfully
```

语法上 root 给别人改密码直接敲 `passwd 用户名`，不需要知道原密码；普通用户只能敲 `passwd` 改自己的，且必须先答对旧密码。**敲新密码时屏幕上光标一动不动是正常设计**——linux 从不回显密码，不是卡死，照着敲完回车就行。第二次 `Retype` 是防手抖。

安全细节顺带建立：

- Debian 的 `passwd` 会做**弱密码检查**（`pam_pwquality`/字典检查），太简单的密码会被劝返。这是帮你，别绞尽脑汁想绕过办法。
- 改密成功那一刻，`/etc/shadow` 里该用户行的"上次改密日期"字段自动更新为今天的天数——这就是你 shadow 表格里第三列数字的来历。
- 写自动化脚本时别在命令行尾跟密码（会进 history、会进 `ps` 输出），用 `chpasswd` 从标准输入读，例如 `echo 'bookuser:初始密码' | chpasswd`，再 `passwd -e bookuser` 强制他下次登录改掉。**这条会在第 16 章开账号时用到。**

### 5.6.2 chage：密码策略的总控台

`chage`（change age，改期限）管理用户的密码/账号年限。先学会"看"：

```bash
chage -l bookuser
```

```text
Last password change          : 11/10/2026
Password expires              : 02/08/2027
Password inactive             : never
Account expires               : never
Minimum number of days between password change    : 0
Maximum number of days between password change    : 99999
Number of days of warning before password expires : 7
```

shadow 第八列的数字被翻译成了人话。常用改法：

```bash
chage -M 90 -W 14 bookuser          # 密码最长 90 天有效，到期前 14 天开始警告
chage -E 2027-06-30 bookuser        # 账号 2027-06-30 起停用（外包、实习账号的自动到期）
```

把口令策略的两个参数记住：`-M`（最长有效期）、`-W`（警告提前量）；账号到期用 `-E`。生产上给**数据库账号、离职交接账号**设 `-E` 到期日，是对"幽灵账号"最省事的防御。

### 5.6.3 锁账户：passwd -l 与 usermod -L

临时禁止某人登录，不用删账号（删了就找不回他的文件权属了），锁上即可：

```bash
passwd -l bookuser
```

```text
passwd: password changed.
```

它做的事只是在 shadow 的哈希前加一个 `!`，密码验证从此必败。解锁是 `passwd -u bookuser`（`usermod -L` / `-U` 等效）。验证状态用 5.4 节学过的 `passwd -S bookuser`，看到 `L` 即锁定。

> **SRE 贴士**：`-l` 锁不住 SSH 密钥、锁不住已登录的会话、也锁不住服务账号的本地进程——它只挡"密码验证"这一条路。生产中要完整踢人下线，正规顺序是：先 `pkill -u 用户名` 杀死其全部进程，再 `usermod -L` 锁账户，最后必要时 `usermod -s /usr/sbin/nologin` 收掉 Shell。权限这件事，**锁一半等于没锁。**

---

## 5.7 su 与 sudo：提权的两副面孔

### 5.7.1 su：整个人切换过去

```bash
su - bookuser
```

先敲密码，然后提示符变成 `bookuser@book:~$`——你现在"是"另一个人了。`su`（switch user / substitute user）不带用户名默认切 root，即 `su` 等价于 `su root`。

**`su` 与 `su -` 的差别值得单独做一次实验，因为它是生产事故的高发点：**

```bash
su bookuser        # 不带连字符
```

```text
$ echo "$USER $HOME $PWD"
root /root /root
$ cd ~
bash: cd: /root: Permission denied
```

看到了吗：身份（UID）确实换成了 `bookuser`，但**环境变量还是 root 的**——`$HOME` 指向 `/root`、当前目录还站在 `/root` 里，结果连自己的"家"都进不去。而 `su - bookuser` 是一次**完整登录**：环境变量重置、家目录切换、登录 Shell 重来，符合"另一个人登进这台机器"的语义。记住这个分工：**日常切换、跑属于别人的任务，用 `su - 用户名`；只借用身份执行一两条命令，考虑用 sudo（下一节）。**

还有个反直觉细节：`sudo su -` 组合里，因为你是 root 或已授权的 sudo 用户，`su` 不会再要密码；而普通用户直接 `su - bookuser` 需要输入 **bookuser 的密码**（不是自己的）——为了绕开"把密码互相通报"，才有下一节的 sudo。

### 5.7.2 sudo：按单授权，全程留痕

sudo（superuser do 的戏称， substitute user do）解决的是 su 的痛点：**不需要把 root 密码发给十个人，每个人用自己的密码，就能被授权执行少数几条管理命令，且每次执行都记日志。**

云电脑是新装的干净 Debian，sudo 不一定在，先装：

```bash
apt install sudo
```

```text
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
The following NEW packages will be installed:
  sudo
0 upgraded, 1 newly installed, 0 to remove and 0 not upgraded.
Need to get 2100 kB of archives.
After this operation, 26.4 MB of additional disk space will be used.
Get:1 http://deb.debian.org/debian trixie/main amd64 sudo amd64 ...
Fetched 2100 kB in 1s (4100 kB/s)
debconf: delaying package configuration, until apt is configured
Selecting previously unselected package sudo.
(Reading database ... 38000 files and directories currently installed.)
Preparing to unpack .../sudo_....deb ...
Unpacking sudo (...) ...
Setting up sudo (...) ...
Processing triggers for libc-bin (...) ...
```

（版本号、大小以你的机器为准，别抄我的数字。装包细节详见第 9 章《软件包管理：apt 与 dpkg》。）

然后把 `bookuser` 加入 sudo 组——Debian 的默认规矩：**sudo 组的人，就拥有完整的 sudo 提权能力**：

```bash
usermod -aG sudo bookuser
```

> **注意**：附加组的变更**对已登录的会话不生效**。`bookuser` 此刻如果正登录着，他输 `sudo` 还是"不在 sudoers 中"。必须先 logout 再重新登录（或者新开一个 SSH 会话），新会话才会带上 `sudo` 组身份。这个"改组要重登"的规矩，生产上坑过无数值班工程师。

验证授权边界：

```bash
su - bookuser
```

```text
$ sudo -l
Matching Defaults entries for bookuser on book:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

User bookuser may run the following commands on book:
    (ALL : ALL) ALL
```

末行 `(ALL : ALL) ALL` 是 Debian 的默认配置赋予 sudo 组的权限：可以在**所有主机（ALL）**上、以**任意用户和任意组（ALL : ALL）**的身份、执行**任意命令（ALL）**。这就是"能用 sudo 的人等于事实上的 root"——所以**加入 sudo 组等于交出事实上的 root 权限**，加人需谨慎，跟发 root 密码是同一级别的事。

真刀真枪试一次（输的是 bookuser 自己的密码，不是 root 的）：

```bash
sudo apt update
```

```text
[sudo] password for bookuser:
Hit:1 http://deb.debian.org/debian trixie InRelease
...
Reading package lists... Done
All packages are up to date.
```

### 5.7.3 visudo 与最小授权配置

sudo 的规矩写在 `/etc/sudoers`。**改它有专用入口 `visudo`（vi + sudoers）**：

```bash
visudo
```

为什么非得用它，不能 `vim /etc/sudoers`？三个理由，每个都对应一次生产事故：其一，**语法校验**——visudo 保存时自动检查语法，写错了它拦住你，而不是等你把人锁在门外；其二，**文件锁**——防止两个人同时改把文件写乱；其三，**权限修复**——sudoers 的权限必须是 `0440`、属主 root，visudo 保证这些属性不被改坏。

**用 vim 直接改坏 sudoers 的后果是：所有人的 sudo 全部失效，严重时连 root 修复自己都费劲。** 这是"改配置把自己关在门外"的经典案例，和 5.4 节讲的 shadow 是同级别的禁区。**安全替代做法：改动前 `cp /etc/sudoers /root/sudoers.bak.$(date +%F)`，改完在另一个终端里保持一个已登录的 root 会话做救援，验证新配置可用了再退出。**

现行文件里默认就有这一行：

```text
%sudo ALL=(ALL:ALL) ALL
```

四段逐块拆开，这是本章唯一需要背下来的"配置语法"：

| 段 | 本例的值 | 含义 |
| --- | --- | --- |
| 谁 | `%sudo` | `%` 开头表示**群组**（组内每个人都生效）；写 `bookuser` 则是单人 |
| 在哪 | 第一个 `ALL` | 允许在哪些主机上执行（单机配置没差别，集中管理时才有意义） |
| 以谁的身份 | 第二个 `(ALL:ALL)` | 括号里是"可以变成哪个用户 : 哪个组的成员"，`ALL:ALL` 是 root 及其组 |
| 干什么 | 最后一个 `ALL` | 允许哪些命令（ALL=任意命令；写成 `/usr/bin/systemctl restart nginx` 就是单命令授权） |

生产环境的**最小授权写法**示范——只允许运维组重启 Web 服务，别的一律不给：

```text
%ops ALL=(root) /usr/bin/systemctl restart nginx, /usr/bin/systemctl status nginx
```

再配一条"每次 sudo 都记下输入的命令"（默认 `log_input`/`log_output` 由发行版打包选项决定，想强化审计可以显式打开）：

```text
Defaults log_input, log_output
Defaults iolog_dir="/var/log/sudo-io"
```

日志去哪了？默认写进 syslog/journald，第 14 章《定时任务与系统日志》讲日志时用 `grep sudo /var/log/auth.log` 就能把"谁在什么时候提权干了什么"翻出来——**这就是 sudo 相对 su 的最大价值：审计线索。**

> **SRE 贴士**：区分 `su -` 和 `sudo` 的使用场景，是运维成熟度的分水岭——`su - root` 是"我整个变成管理员"，适合连续一串管理操作；`sudo 单条命令` 是"我只借一次权力"，适合零散的提权动作，且每笔都有记录。生产规范里我们统一要求：**能用 `sudo cmd` 就别 `su -`**，因为前者精确、后者粗放；能留痕的别用不留痕的。另外记住 `sudo -i`（切到 root 的登录环境）和 `sudo -s`（root 的 shell 但保留当前环境）两个变体，`man sudo` 一节值得通读。

> **生产 vs 实验室**：云电脑上你一直用 root，因为"坏了就重建"，没人审计你。生产上直接 root 登录是明确违规：其一，没有个人审计，出了事只能定位到"root"，无法问责；其二，root 无缓冲，一个 `rm` 手滑就是事故，而普通用户会被权限拦下来；其三，多数合规要求（等保、SOC2）明确要求管理操作留痕到具体自然人。**你在这台云电脑上练习用 `bookuser` + `sudo` 的每一次敲击，都是在提前适应生产规范。**

---

## 5.8 弄清"我是谁"：id / groups / who / last / lastb

排权限问题时，第一件要弄清的事永远是"我现在到底是谁、在哪个组里"。

### 5.8.1 id：一行输出，三个关键数字

```bash
id
```

```text
uid=0(root) gid=0(root) groups=0(root)
```

换成 `bookuser` 的身份看：

```bash
id bookuser
```

```text
uid=1000(bookuser) gid=1000(bookuser) groups=1000(bookuser),27(sudo),1001(devteam)
```

`uid` 用户号、`gid` 主组号、`groups` 是全部所属组（主组 + 附加组，**主组也会出现在这里**，这点和 `/etc/group` 成员列表的规矩不同，别混淆）。判断"能不能访问某文件"时，比的正是这个列表里的 GID。

### 5.8.2 groups：精简版

```bash
groups bookuser
```

```text
bookuser : bookuser sudo devteam
```

只列组名，适合在脚本里快速取用：`id -nG bookuser`（只输出组名、不带 uid= 前缀）更好用，第 13 章写脚本时会大量见到它。

### 5.8.3 who / w：谁正登在这台机器上

```bash
who
```

```text
root     pts/0        2026-11-13 09:12 (10.1.2.3)
```

`pts/0` 是伪终端编号（SSH 登录的终端形态），后面是登录时间和来源 IP。`w` 命令则更进一步，顺带汇报每个会话在跑什么进程。这两个命令在生产上的典型用法：**"这机器上还有谁在线？我要发重启通告了。"** 注意它只报"交互登录"，服务进程、cron 作业不在其列——看全量进程是第 8 章《进程管理与系统监控》的事。

### 5.8.4 last / lastb：登录历史的账本

`last`（读 `/var/log/wtmp`）列出**成功登录**的历史，`lastb`（读 `/var/log/btmp`）列出**登录失败**的记录：

```bash
last -n 5
```

```text
root     pts/0        10.1.2.3         Thu Nov 13 09:12   still logged in
reboot   system boot  6.18.0-instabox  Thu Nov 13 09:11   still running

wtmp begins Thu Nov 13 09:11
```

`-n 5` 限制只显示最近 5 条（默认全部）。`reboot   system boot` 是关机/重启记录。这个"账本"从开机起持续追加，**排"账号是不是被盗了"必查**：一堆陌生 IP 的失败记录（配合 `lastb`）就是告警。

```bash
lastb
```

```text
btmp begins Thu Nov 13 09:11
```

如果这台机器从没被暴力破解过，`lastb` 就是空的（或只有一行开头时间戳）——**空即是好消息**。真出事时长这样：几十条来自同一个 IP 段、间隔几秒的 `ssh:notty` 记录，那就是爆破（暴力破解）扫描。

> **警告**：`wtmp`/`btmp` 只在**本机**有效，且 root 可写——高权限入侵者可以清账。所以"翻 last 没发现异常"不等于"绝对安全"，只是第一道筛查。真正的审计要靠集中日志与不可抵赖机制，那是第 14 章之后、卷 5 的方向。

### 5.9 权限位模型：rwx 到底控制什么

`ls -l` 的第一列输出 `-rwxr-xr-x` 共 10 个字符，第一个字符是文件类型（`-` 普通文件、`d` 目录、`l` 符号链接），后面 9 个字符是权限位，按每 3 个一组，分别对应属主（user）、属组（group）、其他人（others）：

| 位置 | 含义 | r | w | x |
| --- | --- | --- | --- | --- |
| 第 2-4 字符 | 属主权限 | 可读 | 可写 | 可执行/可进入 |
| 第 5-7 字符 | 属组权限 | 可读 | 可写 | 可执行/可进入 |
| 第 8-10 字符 | 其他人权限 | 可读 | 可写 | 可执行/可进入 |

字符含义：

- `r` (read)：文件可读；目录可列出内容（`ls`）。
- `w` (write)：文件可修改；目录可创建、删除、重命名内部条目（**目录的 w 位必须和 x 位配合才有效，见 5.13 节**）。
- `x` (execute)：文件可被执行（二进制或脚本）；目录可 `cd` 进入，或访问其内部已知文件名的元数据和内容。
- `-`：该位无权限。

例子：`-rw-r--r--` 表示属主可读写，属组只读，其他人只读。

> **SRE 贴士**：权限的本质是内核在 `open()` 系统调用时做的访问控制检查。`ls -l` 看到的只是文件 inode 里的 9 位权限掩码，内核按"属主 → 属组 → 其他人"就近匹配，匹配到哪一层就按那层判定，不会叠加。

可以把 rwx 看成二进制权值：r=4，w=2，x=1，三位一组相加得到一位八进制数字。例如 `rwx` = 7，`r-x` = 5，`r--` = 4。这就是后面 chmod 数字法的基础。

### 5.10 chmod：改变文件权限

`chmod`（change mode，改变模式）有两种写法：**数字法**（八进制）和**符号法**（u/g/o + +/-/= 与 r/w/x）。

#### 5.10.1 数字法

```bash
chmod 755 script.sh
chmod 644 note.txt
chmod 700 private.txt
```

数字含义：4=读，2=写，1=执行，三位分别对应属主/属组/其他人。`755` = 属主 rwx(7) + 属组 r-x(5) + 其他人 r-x(5)；`644` = 属主 rw-(6) + 属组 r--(4) + 其他人 r--(4)。

常用组合速查：

| 数字 | 属主 | 属组 | 其他人 | 典型场景 |
| --- | --- | --- | --- | --- |
| 700 | rwx | --- | --- | 私有文件，只有自己能用 |
| 600 | rw- | --- | --- | 私有文本/密钥文件 |
| 755 | rwx | r-x | r-x | 可执行脚本、普通目录 |
| 644 | rw- | r-- | r-- | 普通文本文件 |
| 660 | rw- | rw- | --- | 团队共享文件（属组可写） |
| 770 | rwx | rwx | --- | 团队共享目录 |

> **警告**：`chmod 777`（所有人全权限）是懒人的逃避，也是安全审计的红线。生产环境上任何 777 文件都要被视为待修复的漏洞，尤其在 Web 服务目录里——攻击者上传一个可执行的 777 脚本，等于拿到了服务器的执行权。

#### 5.10.2 符号法

符号法语法：`chmod 谁 操作 权限 文件`，其中"谁"是 u（属主）、g（属组）、o（其他人）、a（all 全体）；"操作"是 +（加）、-（减）、=（设为）；"权限"是 r/w/x。

```bash
chmod u+x script.sh       # 给属主加执行权
chmod g-w note.txt        # 去掉属组写权
chmod o-rwx secret.txt    # 其他人全部拿掉
chmod a+rw log.txt        # 全体可读写
chmod =r-x file.txt       # 设所有人都只有可读可执行（重置所有位）
```

多组操作可用逗号分隔：`chmod u+x,go-w file.sh`。

> **SRE 贴士**：符号法的 `+`/`-` 只改动指定位，其余位保持不变；数字法会一次性重设所有 9 位，不小心会把不该动的权限也改掉。脚本里若只想加/减某位，用符号法更安全；要把文件恢复到"标准权限"，用数字法更干脆。

### 5.11 chown / chgrp：改变属主与属组

`chown`（change owner，改变属主）和 `chgrp`（change group，改变属组）用于转移文件的归属。

```bash
chown bookuser file.txt           # 把文件送给 bookuser
chown bookuser:devteam file.txt   # 同时改属主和属组
chgrp devteam file.txt            # 只改属组
chown -R bookuser:devteam /data/project   # -R 递归改整个目录树
```

只有 root 能随意改属主；普通用户只能把文件送给自己所在的群组（且必须是文件的当前属主）。

> **警告**：`chown -R` 搭配错误的路径（比如漏写路径变成了根目录 `/`）会把全机器文件的归属全部篡改，系统基本报废。执行前先用 `ls -ld` 确认目标路径，或者先不带 `-R` 跑一遍 `ls` 确认范围。

> **SRE 贴士**：Web 服务以 `www-data` 身份运行，它上传的文件属主就是 `www-data`；你用 root 建的文件若属主是 root，Web 服务就写不进去。部署时常见的"权限不足"，根源常在这里——`ls -l` 看属主，比盲目 `chmod 777` 管用得多。

### 5.12 umask：新文件默认权限从哪来

你新建文件时，为什么默认不是 777 或 000，而是 644 或 755？答案是 **umask**（user file-creation mode mask，用户文件创建屏蔽码）。

umask 是一组"屏蔽位"，新文件的初始权限 = 最大允许权限 & ~umask：
- 文件最大允许 666（文件默认不给执行位），目录最大允许 777。

常见 umask `022`：
- 属主：666 & ~022 → 文件 644，目录 755
- 属组、其他人：各去掉写位

验证当前 umask：

```bash
umask
```

```text
0022
```

临时改 umask（只影响当前 Shell 会话）：

```bash
umask 077
touch secret.txt
ls -l secret.txt
```

```text
-rw------- 1 root root 0 Nov 13 09:45 secret.txt
```

`077` 意味着属组和其他人全部没有权限，适合建私密文件。结束后记得 `umask 022` 收回，否则后续新建的文件对别人都不可见，容易引发"明明文件在，别人却读不了"的排障。

永久生效：把 `umask 022` 写入 `~/.bashrc` 或 `/etc/profile`（详见第 3 章和第 13 章）。

> **SRE 贴士**：生产服务器上统一 umask（多为 022 或 077）是防止"新建文件权限过宽"的最低防线。部署脚本里若用 `install -m 600` 显式指定权限，反而比依赖 umask 更稳。

### 5.13 目录的 x 位：进得去不等于能看清

目录的权限和文件不完全一样，这是新手最大的困惑点：

| 权限 | 对目录的真实含义 |
| --- | --- |
| r | 能 `ls` 列出目录内的文件名 |
| w | 能在目录内创建、删除、重命名文件（前提是目录也有 x） |
| x | 能 `cd` 进入，能 `stat`/`cat` 目录内已知文件名的元数据和内容 |

关键实验：在 `/tmp` 建一个测试目录，剥夺 x 位：

```bash
mkdir -p /tmp/permtest/inner
echo "hello" > /tmp/permtest/inner/file.txt
chmod 700 /tmp/permtest
ls -l /tmp/permtest
```

```text
drwx------ 2 root root 4096 Nov 13 09:50 permtest
```

现在去掉 x（目录仍可列，但进不去）：

```bash
chmod 500 /tmp/permtest   # r-x，有 r 有 x 没 w
ls /tmp/permtest          # 能列出 inner
```

```text
inner
```

```bash
cd /tmp/permtest          # 有 x，能进
```

再把 x 拿掉，只留 r：

```bash
chmod 400 /tmp/permtest   # r--，有 r 无 x
ls /tmp/permtest          # 能列出
cd /tmp/permtest          # 进不去：Permission denied
```

输出：

```text
bash: cd: /tmp/permtest: Permission denied
```

结论：**目录没 x 位，就无法进入，也无法访问其内部文件（哪怕文件本身是 777）**；目录没 r 位，列不出内部文件名，但如果你知道确切文件名且目录有 x，仍可直接访问。这两个位缺一不可，才是"能正常用一个目录"。

> **警告**：不要随便给 `/tmp` 或共享目录加上全员 w+x，结合 Sticky 位（5.14 节）才是安全的共享目录做法，否则任何人都能删别人的文件。

### 5.14 特殊权限：SUID / SGID / Sticky

除了 rwx，执行位还可出现三个特殊标志：

| 标志 | 字符位置 | 作用 |
| --- | --- | --- |
| SUID（Set User ID） | 属主 x 位变成 `s` 或 `S` | 可执行文件以**文件属主的身份**运行，而非启动者的身份 |
| SGID（Set Group ID） | 属组 x 位变成 `s` 或 `S` | 文件以属组身份运行；**目录内新建文件自动继承目录的属组** |
| Sticky（粘滞位） | 其他人 x 位变成 `t` 或 `T` | 目录内只有文件属主、目录属主和 root 能删除/重命名文件 |

SUID 经典例子：`passwd` 命令。普通用户改密码要写 `/etc/shadow`（root 才能读写），但 `/usr/bin/passwd` 设了 SUID 且属主是 root，所以运行时暂时拥有 root 权限，能合法地改密码文件：

```bash
ls -l /usr/bin/passwd
```

```text
-rwsr-xr-x 1 root root 67728 Nov 13 09:00 /usr/bin/passwd
```

属主的 `x` 变成了 `s`，就是 SUID。

SGID 目录例子：团队共享目录，设了 SGID 后，任何人在里面建文件，属组都自动是目录的属组，避免"新建文件属主是自己、别人碰不了"的问题。

Sticky 位经典例子：`/tmp`：

```bash
ls -ld /tmp
```

```text
drwxrwxrwt 1 root root 4096 Nov 13 09:00 /tmp
```

其他人位的 `t` 就是 Sticky：`/tmp` 人人可写，但每人只能删自己的文件，不会互相删库。

大写 `S` / `T` 表示该位原本就没有 x 位（罕见，通常出现在配置错误的文件上），小写 `s` / `t` 表示有 x 且带特殊标志。

设置方法（数字法前加一位）：

```bash
chmod 4755 /usr/local/bin/sudo-helper   # 4 = SUID
chmod 2775 /data/team                   # 2 = 目录 SGID
chmod 1777 /tmp                         # 1 = Sticky（/tmp 标准就是 1777）
```

> **警告**：SUID 是本章最危险的权限——可执行文件以属主（常为 root）身份运行，若该程序有漏洞，就是本地提权（Local Privilege Escalation）的入口。生产服务器上 SUID 文件应尽可能少，排查方法：`find / -perm -4000 -type f 2>/dev/null` 列出全部 SUID 文件，不认识的一律重视。

> **SRE 贴士**：排查"明明给了权限还是进不去"时，除了看文件权限，还要顺着路径向上检查每一级目录的 x 位。比如 `/data/home/bookuser/notes.txt` 权限再开放，如果 `/data` 或 `/data/home` 中间某级对你没 x，你一样连文件本体都摸不着。

---

## 本章小结

| 知识点 | 核心要点 |
| --- | --- |
| 多用户模型 | 属主 → 属组 → 其他人就近匹配，不叠加 |
| /etc/passwd | 7 个字段，UID 才是身份证，0 是 root |
| /etc/shadow | 只能 root 读，`passwd -S` 查密码状态 |
| 账号管理 | `useradd -m -s /bin/bash`，改组务必带 `-aG`，删除先锁后归档 |
| 密码策略 | `passwd` 改密，`chage` 改策略，弱密码会被 pam 拦截 |
| su / sudo | `su -` 换身份，`sudo` 单命令提权，`visudo` 编辑 sudoers |
| 查谁在线 | `id` / `groups` / `who` / `w` / `last` / `lastb` |
| 权限位 | r=4 w=2 x=1，目录 x 位是"能否进入"的开关 |
| chmod | 数字法一次性重设，符号法增量改动，慎用 777 |
| chown/chgrp | 只有 root 能随意改属主，`-R` 递归要小心 |
| umask | 新文件权限 = 最大值 & ~umask，022 得 644/755 |
| 特殊权限 | SUID 以属主身份运行，SGID 目录继承属组，Sticky 只让属主删文件 |

本章把"谁能对什么文件做什么"讲透了。下一章（第 6 章《文本处理：grep / sed / awk / vim》）我们要处理文件**里面的内容**，不仅是文件本身。

---

## 动手练习

在自己的云电脑上完成以下练习，每步都用 `ls -l` 验证结果。

**练习 1**：创建普通用户 `opsuser`，把它加入 `sudo` 组（记得 `-aG`），给它设一个密码，然后用 `id opsuser` 确认 UID/GID 和附加组。

**练习 2**：在 `/tmp` 创建文件 `perm-demo.txt`，内容任意。先 `chmod 600`，用 `cat` 确认自己能读；再切换到 `opsuser` 用 `cat` 读，应被拒绝；最后 `chmod 644`，`opsuser` 就能读了。

**练习 3**：建目录 `/tmp/dir-demo`，`chmod 700`，确认其他人无法 `cd` 进入；再 `chmod 755`，进入并建一个子文件，验证目录 x 位是"入场券"。

**练习 4**：查看 `/usr/bin/passwd` 和 `/tmp` 的权限，分别指出 SUID 和 Sticky 位，并解释各自作用。

**练习 5**：临时设 `umask 077`，新建一个文件，查看其权限是否为 `600`；然后恢复 `umask 022`。

---

## 参考答案

**答案 1**：

```bash
useradd -m -s /bin/bash opsuser
usermod -aG sudo opsuser
passwd opsuser
id opsuser
```

预期输出（UID 视机器而定，不写死）：

```text
uid=1001(opsuser) gid=1001(opsuser) 组=1001(opsuser),27(sudo)
```

关键点：`usermod` 必须带 `-aG`，漏了 `-a` 会把用户从其他附加组踢出去。

**答案 2**：

```bash
echo "demo content" > /tmp/perm-demo.txt
chmod 600 /tmp/perm-demo.txt
ls -l /tmp/perm-demo.txt
cat /tmp/perm-demo.txt       # root 能读
su - opsuser                 # 切换到普通用户
cat /tmp/perm-demo.txt       # Permission denied
exit                         # 回 root
chmod 644 /tmp/perm-demo.txt
su - opsuser -c "cat /tmp/perm-demo.txt"   # 能读，输出 demo content
```

**答案 3**：

```bash
mkdir -p /tmp/dir-demo
chmod 700 /tmp/dir-demo
su - opsuser -c "cd /tmp/dir-demo"   # Permission denied
chmod 755 /tmp/dir-demo
cd /tmp/dir-demo
echo "inside" > inner.txt
```

**答案 4**：

- `/usr/bin/passwd` 显示 `-rwsr-xr-x`，属主执行位是 `s`，即 SUID：普通用户运行 passwd 时以 root 身份执行，能写 `/etc/shadow` 修改密码。
- `/tmp` 显示 `drwxrwxrwt`，其他人执行位是 `t`，即 Sticky：人人可写，但每人只能删除/重命名自己的文件。

**答案 5**：

```bash
umask 077
touch /tmp/umask-test.txt
ls -l /tmp/umask-test.txt   # 预期 -rw-------（600）
umask 022
```

注意：umask 只影响当前 Shell 会话，关终端后失效；要永久生效写进 `~/.bashrc`。
