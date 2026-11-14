---
title: "第 1 章 搭建实验环境：VMware + Ubuntu 本地虚拟机"
---

# 第 1 章 搭建实验环境：VMware + Ubuntu 本地虚拟机

> 本章带你在自己的电脑上装一台 Ubuntu 虚拟机，作为全书的实验环境。所有后续章节的命令，都可以在这台本地虚拟机里验证。**禁止把 VMware 或虚拟机文件安装到 C 盘**，请全部选 D 盘或其他非系统盘。

## 本章目标

- 下载 VMware Workstation Player 并安装到非 C 盘
- 下载 Ubuntu 24.04 LTS ISO 镜像
- 创建 Ubuntu 虚拟机，虚拟磁盘和配置文件均放在非 C 盘
- 从 ISO 启动并完成 Ubuntu 安装
- 安装 VMware Tools，验证虚拟机环境可用
- 截取每一步的安装截图，补充到本书（见下文“截图补充说明”）

---

## 1.1 下载 VMware Workstation Player

VMware Workstation Player 个人非商业用途免费。

1. 访问 VMware 官网下载页面，下载 **VMware Workstation Player for Windows**。
2. 运行下载的安装引导程序，等待它联网下载完整安装包。
3. **关键：安装向导中选择“自定义安装”，将安装路径改为非 C 盘**，例如：
   ```
   D:\VMware\Workstation Player
   ```
   千万不要使用默认的 `C:\Program Files (x86)\VMware\...`。
4. 安装完成后，如提示重启电脑，请重启。
5. 启动 VMware Workstation Player，首次运行选择 **“免费用于个人非商业用途”**。

> **截图占位 1.1**：`vol1-linux/assets/ch01/01-vmware-install-path.png`  
> 内容：安装向导中“自定义安装路径”已改为 D 盘（或非 C 盘）的画面。

---

## 1.2 下载 Ubuntu 镜像

1. 访问 Ubuntu 官网：`ubuntu.com/download/desktop`
2. 下载 **Ubuntu 24.04 LTS**（64-bit PC（AMD64）桌面版）的 `.iso` 文件，约 5 GB。
3. 将 ISO 文件保存到非 C 盘目录，例如：
   ```
   D:\ISO\ubuntu-24.04-desktop-amd64.iso
   ```

> **截图占位 1.2**：`vol1-linux/assets/ch01/02-ubuntu-iso-download.png`  
> 内容：下载完成后文件资源管理器中显示 ISO 文件及其大小约 5 GB。

---

## 1.3 创建 Ubuntu 虚拟机

1. 打开 VMware Workstation Player，点击 **“创建新虚拟机”**。
2. 选择 **“典型”**（推荐初学者）。
3. 在“客户机操作系统安装介质”页面，选择 **“稍后安装操作系统”**（稍后手动指定 ISO，更稳妥）。
4. 客户机操作系统：选择 **Linux**，版本选择 **Ubuntu 64-bit**。
5. 虚拟机名称：填 `book`。
6. **位置：选择非 C 盘目录**，例如：
   ```
   D:\VMs\book
   ```
   **不要放在 C 盘。**
7. 固件类型：保持默认 BIOS。
8. 磁盘容量：建议 **40 GB 或以上**，选择 **“将虚拟磁拆分为多个文件”**。
9. 点击 **“完成”**。虚拟机会出现在左侧库中，此时尚未启动。

> **截图占位 1.3**：`vol1-linux/assets/ch01/03-vm-create-location.png`  
> 内容：创建虚拟机向导中，“位置”字段已设置为 D:\VMs\book（非 C 盘）。

---

## 1.4 挂载 Ubuntu ISO 并启动安装

1. 在虚拟机库中选中 `book`，点击 **“编辑虚拟机设置”**。
2. 在 **CD/DVD (SATA)** 设置中，选择 **“使用 ISO 映像文件”**，浏览并指向：
   ```
   D:\ISO\ubuntu-24.04-desktop-amd64.iso
   ```
3. 勾选 **“连接时启动”**，点击 **“确定”**。
4. 点击 **“开启此虚拟机”**，Ubuntu 安装程序会从 ISO 启动。

> **截图占位 1.4**：`vol1-linux/assets/ch01/04-iso-mounted.png`  
> 内容：虚拟机设置中 CD/DVD 指向了下载的 Ubuntu ISO 文件。

---

## 1.5 安装 Ubuntu 系统

1. Ubuntu 安装程序启动后，选择语言 **中文（简体）**。
2. 键盘布局保持默认。
3. 安装类型选择 **“擦除磁盘并安装 Ubuntu”**。  
   ⚠️ 这是虚拟机内部的虚拟磁盘，**不会删除你本机 C 盘或 D 盘的数据**。
4. 设置你的姓名、计算机名、用户名和密码。**请牢记，后面 sudo 和 SSH 都要用。**
5. 登录选项：实验机建议 **不要求密码自动登录**，方便调试；生产环境请勿这样做。
6. 等待安装完成（约 15–30 分钟），完成后点击 **“立即重启”**。
7. 重启后，在 VMware 菜单栏选择 **“虚拟机” → “可移动设备” → “CD/DVD” → “断开连接”**，避免再次从 ISO 启动。

> **截图占位 1.5**：`vol1-linux/assets/ch01/05-ubuntu-install-type.png`  
> 内容：Ubuntu 安装程序的“安装类型”页面，选中“擦除磁盘并安装 Ubuntu”。  
> **截图占位 1.6**：`vol1-linux/assets/ch01/06-ubuntu-install-complete.png`  
> 内容：安装完成、提示“立即重启”的画面。

---

## 1.6 安装 VMware Tools（推荐）

VMware Tools 能改善显示分辨率、剪贴板共享、文件拖拽等体验。

1. Ubuntu 桌面登录后，在 VMware 菜单栏选择 **“虚拟机” → “安装 VMware Tools”**。
2. 系统会挂载一个虚拟光盘，打开终端运行：
   ```bash
   cd /media/$USER/VMware\ Tools
   tar xzvf VMwareTools-*.tar.gz
   cd vmware-tools-distrib
   sudo ./vmware-install.pl -d
   ```
3. 安装完成后重启虚拟机。

> **截图占位 1.7**：`vol1-linux/assets/ch01/07-vmware-tools-install.png`  
> 内容：终端中运行 `sudo ./vmware-install.pl -d` 并显示安装完成的画面。

---

## 1.7 验证环境

在 Ubuntu 终端中运行以下验货命令：

```bash
uname -a          # 查看内核版本
nproc             # 查看 CPU 核心数
free -h           # 查看内存
df -h /           # 查看根分区空间
ip addr           # 查看虚拟机 IP
```

预期输出示例：

```text
Linux book 6.8.0-...-generic #...Ubuntu SMP ... x86_64 ... GNU/Linux
2
              total        used        free      shared  buff/cache   available
Mem:           7.6Gi       1.2Gi       5.1Gi        12Mi       1.4Gi       6.1Gi
Swap:            2Gi          0B        2Gi
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda2        38G   5.2G     31G   15% /
```

> **截图占位 1.8**：`vol1-linux/assets/ch01/08-verification-commands.png`  
> 内容：Ubuntu 终端中运行上述验货命令并输出结果的画面。

如果能正常输出，VMware + Ubuntu 实验环境已就绪，后续章节的所有命令都可在此机器上执行。

---

## 1.8 常见问题

| 问题 | 解决办法 |
|------|----------|
| 虚拟机无法启动，提示 VT-x/AMD-V 未启用 | 重启电脑进入 BIOS，开启虚拟化技术（Intel VT-x 或 AMD-V） |
| 安装卡在“准备安装” | 检查 ISO 文件是否完整，可重新下载 |
| 分辨率过小、无法全屏 | 安装 VMware Tools 后重启虚拟机 |
| 想把虚拟机移到另一块盘 | 直接复制虚拟机文件夹（如 `D:\VMs\book`）到新位置，在 VMware 中移除后重新添加 |

---

## 截图补充说明（请你完成）

上面标注的 **截图占位 1.1–1.8** 需要你在实际操作时拍摄，并保存为：

```
vol1-linux/assets/ch01/
  01-vmware-install-path.png
  02-ubuntu-iso-download.png
  03-vm-create-location.png
  04-iso-mounted.png
  05-ubuntu-install-type.png
  06-ubuntu-install-complete.png
  07-vmware-tools-install.png
  08-verification-commands.png
```

拍摄后把对应图片文件放入 `vol1-linux/assets/ch01/`，再告诉我，我把 Markdown 中的占位符替换为真实图片引用，并提交到 GitHub。

---

## 本章小结

本章把全书的本地实验环境搭好了，四件事：

1. **VMware 安装到非 C 盘**（如 `D:\VMware`）
2. **Ubuntu 24.04 LTS ISO 保存在非 C 盘**（如 `D:\ISO\`）
3. **虚拟机 `book` 创建在非 C 盘**（如 `D:\VMs\book`），并从 ISO 安装完成
4. **安装 VMware Tools，通过验货命令确认环境可用**

下一章，我们正式登录 Ubuntu，认识你的第一个提示符，跑人生第一组 Linux 命令。

## 动手练习

1. 按 1.1–1.3 节下载 VMware 和 Ubuntu ISO，全部保存到非 C 盘，把下载完成后的文件资源管理器截图贴进笔记本。
2. 按 1.4–1.5 节创建虚拟机并安装 Ubuntu，把安装类型和重启提示的截图贴进笔记本。
3. 安装 VMware Tools，重启后在终端运行 `uname -a`、`nproc`、`free -h`、`df -h /`、`ip addr`，把输出贴进笔记本并逐条判读（如：nproc 输出 2，所以虚拟机分配了 2 个 vCPU）。
4. 故意把虚拟机配置文件尝试放到 C 盘，观察 VMware 是否报错或受限，记录观察结果（体会“禁止安装到 C 盘”的原因）。
