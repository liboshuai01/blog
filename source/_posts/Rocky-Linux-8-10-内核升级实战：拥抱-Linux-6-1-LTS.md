---
title: Rocky Linux 8.10 内核升级实战：拥抱 Linux 6.1 LTS
tags:
  - Linux
  - Rocky Linux
categories:
  - 运维手册
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202505050026782.png'
toc: true
abbrlink: fced2319
date: 2025-05-05 00:22:35
---

Rocky Linux 8.x 系列作为一个稳定、可靠的企业级 Linux 发行版，默认搭载的内核版本通常是经过长期测试和验证的稳定版本（在 8.10 中通常是 4.18.x 系列）。然而，在某些场景下，我们可能需要更新的内核版本来获得：

*   **更好的硬件支持:** 特别是对于较新的 CPU、GPU、网络适配器或存储设备。
*   **新功能和特性:** 利用内核新版本带来的性能优化、安全增强或文件系统改进。
*   **特定的软件依赖:** 某些应用程序或驱动程序可能需要特定或更新的内核版本。

Linux Kernel 6.1 是一个重要的 **长期支持 (LTS)** 版本，提供了许多改进和新特性，并且将获得较长时间的维护支持。本文将详细介绍如何在 Rocky Linux 8.10 系统上，使用广泛认可的第三方仓库 **ELRepo**，安全地将内核升级到 6.1 LTS 版本。

**❗ 重要警告:** 内核是操作系统的核心。升级内核涉及系统级别的更改，存在一定的风险，可能导致系统无法启动或出现不稳定。**在执行任何操作之前，请务必：**

1.  **备份您的重要数据！**
2.  **如果可能，在非生产环境或虚拟机中进行测试。**
3.  **确保您了解如何通过 GRUB 引导加载程序选择旧内核启动，以备不时之需。**

<!-- more -->

## 环境准备

*   **操作系统:** Rocky Linux 8.10
*   **目标内核:** Linux Kernel 6.1.x (LTS)
*   **升级方法:** 使用 ELRepo 仓库
*   **用户权限:** root 或具有 sudo 权限的用户
*   **网络连接:** 需要从互联网下载仓库信息和内核软件包

## 升级步骤

### 第一步：更新现有系统

在进行内核升级之前，建议先将系统上的所有软件包更新到最新版本，以确保系统处于一致和最新的状态。

```bash
sudo dnf update -y
```

更新完成后，建议重启一次系统以应用所有更新（特别是如果有 `glibc`、`systemd` 或现有内核的小版本更新）。

```bash
sudo reboot
```

### 第二步：导入 ELRepo GPG 密钥

为了验证从 ELRepo 下载的软件包的真实性和完整性，需要先导入其 GPG 密钥。

```bash
sudo rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
```

### 第三步：安装 ELRepo 仓库

接下来，安装 ELRepo 仓库的配置文件。这将使 `dnf` 能够找到 ELRepo 提供的软件包。

```bash
sudo dnf install -y https://www.elrepo.org/elrepo-release-8.el8.elrepo.noarch.rpm
```

### 第四步：查看可用的内核版本（可选但推荐）

你可以查看 ELRepo 仓库中提供的内核版本，特别是 `kernel-lt` (Long Term Support) 和 `kernel-ml` (Mainline Stable)。我们的目标是 6.1 LTS，它通常由 `kernel-lt` 包提供。

```bash
dnf --disablerepo="*" --enablerepo="elrepo-kernel" list available | grep kernel-lt
```

你应该能在输出中看到类似 `kernel-lt.x86_64` 和 `kernel-lt-devel.x86_64` 等包，确认 6.1.x 版本是否可用。

*   `kernel-lt`: 长期支持版内核 (Long Term Support) - **这是我们本次的目标**
*   `kernel-ml`: 主线稳定版内核 (Mainline Stable) - 通常是最新的非 LTS 稳定版

### 第五步：安装 Linux Kernel 6.1 (kernel-lt)

确认 ELRepo 仓库已启用并且 6.1 LTS (`kernel-lt`) 可用后，执行以下命令安装内核：

```bash
sudo dnf --enablerepo=elrepo-kernel install kernel-lt -y
```

如果你需要对应的开发包（例如，编译内核模块时需要），可以同时安装：

```bash
sudo dnf --enablerepo=elrepo-kernel install kernel-lt kernel-lt-devel -y
```

安装过程通常会自动更新 GRUB 引导加载程序的配置，并将新安装的内核设置为默认启动项。**但有时这个自动过程可能不会生效，或者你希望确保新内核是默认选项。**

---

### 第六步：设置默认启动内核 (可选)

安装过程通常会自动更新 GRUB 引导加载程序的配置，并将新安装的内核设置为默认启动项。然而，这个过程有时可能不会按预期工作，或者你可能希望手动确认并设置默认内核。在 Rocky Linux 8 系统上，**推荐使用 `grubby` 工具**来完成此操作，因为它更可靠且能处理不同的系统配置（如 BIOS 和 UEFI）。

1.  **列出所有已安装的内核及其信息:**
    使用 `grubby --info=ALL` 命令可以获取系统中 GRUB 能识别的所有内核条目的详细信息，包括它们的索引、内核文件路径和对应的 GRUB 标题。

    ```bash
    sudo grubby --info=ALL
    ```
    仔细查看输出，找到你刚刚安装的 6.1 内核条目。记下它的 `index` 和 **`kernel`** (内核文件路径，例如 `/boot/vmlinuz-6.1.55-1.el8.elrepo.x86_64`) 以及 `title`。

    输出示例可能类似（已简化）：
    ```
    index=0
    kernel="/boot/vmlinuz-6.1.55-1.el8.elrepo.x86_64"
    args="..."
    root="..."
    initrd="/boot/initramfs-6.1.55-1.el8.elrepo.x86_64.img"
    title="Rocky Linux (6.1.55-1.el8.elrepo.x86_64) 8.10 (Green Obsidian)"
    id="..."
    index=1
    kernel="/boot/vmlinuz-4.18.0-513.18.1.el8_9.x86_64"
    args="..."
    root="..."
    initrd="/boot/initramfs-4.18.0-513.18.1.el8_9.x86_64.img"
    title="Rocky Linux (4.18.0-513.18.1.el8_9.x86_64) 8.10 (Green Obsidian)"
    id="..."
    ...
    ```

2.  **查看当前的默认内核:**
    你可以使用以下命令检查当前配置的默认内核：

    ```bash
    sudo grubby --default-kernel
    ```
    或者查看默认内核的索引：
    ```bash
    sudo grubby --default-index
    ```

3.  **设置新的 6.1 内核为默认启动项:**
    **推荐使用内核文件路径**来设置默认项，因为这比使用可能变动的索引号或标题更稳定。使用你在第一步中找到的 6.1 内核的 `kernel` 路径：

    ```bash
    # 将下面的路径替换为你实际看到的 6.1 内核的 vmlinuz 文件路径
    sudo grubby --set-default /boot/vmlinuz-6.1.55-1.el8.elrepo.x86_64
    ```
    执行此命令后，`grubby` 会自动更新 GRUB 的配置。

4.  **验证设置:**
    再次运行 `grubby --default-kernel` 来确认默认内核路径是否已更新为你刚刚设置的 6.1 内核路径。

    ```bash
    sudo grubby --default-kernel
    ```
    输出应该显示你设置的 6.1 内核文件路径。

### 第七步：重启系统

现在，重启系统以加载新的 6.1 LTS 内核。

```bash
sudo reboot
```

## 验证内核升级

系统重启后，登录并执行以下命令来检查当前正在运行的内核版本：

```bash
uname -r
```

如果一切顺利，输出应该显示类似 `6.1.x.el8_x.elrepo.x86_64` 的版本号，确认内核已成功升级到 6.1 版本并且是当前运行的内核。

你也可以再次检查 GRUB 的默认启动项确认无误：

```bash
sudo grub2-editenv list
```

## 故障排除：如果系统无法启动

如果在重启后系统无法正常启动或者遇到问题（即使新内核被设置为默认）：

1.  **不要慌张！** 在启动时看到 GRUB 引导菜单（如果没有看到，尝试在启动初期按 `Shift` 或 `Esc` 键）。
2.  在 GRUB 菜单中，选择 "Advanced options for Rocky Linux" 或类似选项。
3.  你应该能看到一个列表，其中包含新安装的 6.1 内核和之前可以正常工作的旧内核（例如 4.18.x）。
4.  **手动选择旧的、可以正常工作的内核版本** 启动系统。
5.  系统成功启动后，你可以分析 `/var/log/messages` 或使用 `journalctl` 命令检查启动失败时的日志，排查具体原因（可能是驱动不兼容、硬件问题等）。

如果确定新内核有问题，你可以：

*   **切换默认启动项：** 启动到旧内核后，按照 **第六步：(可选) 手动设置默认启动内核** 中的说明，使用 `grub2-set-default` 命令将默认启动内核改回旧版本（选择旧内核对应的标题或索引）。
*   **卸载新内核：** 如果确认不再需要或无法使用 6.1 内核，可以启动到旧内核后，使用以下命令卸载它：
    ```bash
    sudo dnf remove kernel-lt kernel-lt-devel # 如果安装了 devel 包也一并移除
    ```
    卸载后，GRUB 配置通常会自动更新，移除对应的启动项。

## 总结

通过使用 ELRepo 仓库，我们可以相对轻松且安全地为 Rocky Linux 8.10 系统升级到 Linux Kernel 6.1 LTS 版本。这使得我们能够利用新内核带来的硬件兼容性改进、性能优化和新功能，同时享受 LTS 版本的长期支持。通过了解如何手动配置 GRUB 默认启动项，我们可以更好地控制内核选择过程，应对自动配置可能失效的情况。

请始终牢记，内核升级是一项核心系统操作，务必在操作前做好充分的备份和风险评估。希望这篇更新后的博文能帮助你顺利完成内核升级！
