---
title: Linux安装GCC指南
tags:
  - Linux
  - GCC
categories:
  - 运维手册
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504270010957.png'
toc: true
abbrlink: c6d2456d
date: 2023-12-08 00:05:55
---

GCC（GNU Compiler Collection）是 Linux 系统中最主流、最核心的编程语言编译器套件，支持多种语言，如 C、C++、Fortran、Objective-C 等。在 Linux 系统上进行 C/C++ 开发、编译软件或构建内核模块时，GCC 都是必不可少的工具。

在本文中，我们将基于 CentOS（适用于 7 或其他兼容 RHEL 的发行版），通过手动下载并安装 GCC 编译器的方式，完成从零构建环境的全过程。

<!-- more -->

---

## GCC 与 Linux 内核的关系

首先，需要澄清一个常见误区：Linux 内核本身并不“依赖”已经安装好的 GCC 环境。GCC 是用户空间中的一个构建工具，它用于将 Linux 内核的源代码编译为可执行的二进制镜像（如 vmlinuz），或者用于开发运行于内核之上的应用及模块。

但在我们日常的开发工作中，特别是在构建源码、调试系统底层功能或安装某些软件包时，GCC 是不可缺少的。

---

## 检查系统是否已安装 GCC

在大多数 Linux 系统中可以通过以下命令检测是否已安装 gcc，以及查看版本号：

```bash
gcc -v
```

示例输出（表示已安装）：

```
gcc version 4.8.5 20150623 (Red Hat 4.8.5-44) (GCC)
```

若未安装，会提示类似如下内容：

```
bash: gcc: command not found
```

此时你就需要手动安装 GCC，下面是详细的安装步骤。

---

## 手动安装 GCC（以 GCC 4.8.5 为例）

> 适用于无法使用 yum 安装或者在服务器上无公网环境的用户。

### 步骤一：下载 GCC 安装包

你可以从 GNU 官方或清华大学开源镜像站下载 GCC 的 RPM 安装包。

- 官方 GNU 镜像：https://ftp.gnu.org/gnu/gcc/
- 推荐使用国内清华源（更快）：https://mirrors.tuna.tsinghua.edu.cn/gnu/gcc/

为方便演示，本文使用已打包好的 GCC 4.8.5 安装包，包含依赖项：

- 下载链接（网盘）：[https://niceyoo.lanzoux.com/i5nIdmx2n9e](https://niceyoo.lanzoux.com/i5nIdmx2n9e)

⚠️ 建议使用 wget、curl、scp 或 xftp 等工具将下载好的 gcc 安装包传输到 Linux 服务器。

### 步骤二：上传/移动安装包至目标目录

以 `/usr/local/gcc` 为我们的 GCC 安装目录：

```bash
mkdir -p /usr/local/gcc
mv gcc.tar.gz /usr/local/gcc
cd /usr/local/gcc
```

### 步骤三：解压安装包

```bash
tar -zxvf gcc.tar.gz
```

此目录下解压后应包含多个 `.rpm` 文件，如下所示：

```bash
ls
gcc
gcc-c++
libstdc++...
```

### 步骤四：使用 rpm 安装所有包

```bash
cd gcc
rpm -Uvh *.rpm --nodeps --force
```

参数说明：

- `-Uvh`：表示升级或安装软件包，且以详细模式输出
- `--nodeps`：忽略依赖关系（不建议生产环境使用）
- `--force`：强制覆盖已存在的同名包

⚠️ 注意：系统建议先备份相关组件，尤其是在存在已有较高版本 GCC 时，以免依赖链紊乱。

### 步骤五：验证安装成功

```bash
gcc -v
```

预期输出：

```
gcc version 4.8.5 20150623 (Red Hat 4.8.5-44) (GCC)
```

如出现找不到 gcc 或版本依旧旧，尝试检查是否软链接版本不同：

```bash
which gcc
ls -l /usr/bin/gcc
```

可通过更新软链接指定你刚安装的 gcc：

```bash
ln -sf /usr/local/gcc/bin/gcc /usr/bin/gcc
```

（路径根据实际安装位置调整）

---

## 常见问题与解决思路

| 问题 | 可能原因 | 解决方法 |
|------|-----------|-----------|
| rpm 安装报依赖错误 | 缺少依赖库 | 使用 `--nodeps` 暂时忽略，或提前安装依赖包 |
| 安装成功但运行 `gcc` 显示未找到 | 未添加到 PATH | 将 path 添加到 `/etc/profile` 或创建软链接 |
| 无法编译 C 文件 | 缺少标准库（如 glibc） | 确保 `glibc`, `libgcc`, `libstdc++` 等一并安装 |

---

## 使用 yum 安装 GCC（推荐方法）

如果服务器有互联网并希望自动解决依赖项，还是建议优先使用 yum 安装：

```bash
yum install -y gcc gcc-c++ make
```

安装完成后查看版本：

```bash
gcc -v
```

---

## 总结

本文介绍了在 CentOS 环境下安装 GCC 编译器的完整流程，从检查、下载、上传、解压、安装到测试验证，涵盖了手动方式和自动方式的差异。

作为后端开发人员，无论你在构建 JNI 模块、接入 Python 的 C 扩展、编译 Elastic 自定义插件，还是在部署 AI 模型时衔接 C/C++ 库，对 GCC 的掌握都是基本技能的一部分。

通过手动安装方式，也能帮助你深入理解 Linux 底层依赖结构、RPM 包管理方式，并提升解决实际线上问题的能力。

---

👍 如果您觉得这篇文章帮助到了你，欢迎点赞、收藏或关注我持续分享更多 Linux、后端开发相关内容！

> 原文整理参考自： [https://www.cnblogs.com/niceyoo/p/14532228.html](https://www.cnblogs.com/niceyoo/p/14532228.html)，并结合实践内容进行了扩展与优化整理。