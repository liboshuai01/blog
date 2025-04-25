---
title: 理解 sh 执行脚本与直接执行脚本的区别
abbrlink: aaf0c073
date: 2024-04-30 01:40:26
tags:
  - Linux
  - Shell
categories:
  - 运维手册
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260142290.png'
toc: true
---

在 Unix 和 Linux 系统中，执行 shell 脚本有多种方法，其中常见的两种是使用 `sh` 命令和直接执行脚本文件（前提是脚本具有可执行权限）。本文将详细解释这两种方式的区别及其影响。

<!-- more -->

## 使用 `sh` 命令执行脚本

当你使用 `sh` 命令执行脚本时，实际的执行过程如下：

```bash
sh script.sh
```

### 特点

1. **解释器固定**：使用 `sh` 解释器来执行脚本，而不管脚本的 shebang（`#!`）声明。
2. **无需可执行权限**：脚本文件无需可执行权限，因为脚本内容是作为参数传递给 `sh` 解释器的。

### 示例

假设 `script.sh` 文件内容如下：

```bash
#!/bin/bash
echo "This script is running with Bash"
```

即使脚本文件中声明了 `#!/bin/bash`，使用 `sh script.sh` 时，`sh` 解释器会忽略这个声明，而强制使用 `sh`（通常是 `/bin/sh`）来执行脚本。

### 可能的问题

1. **兼容性问题**：如果脚本依赖于某些特定解释器（如 Bash）的特性，使用 `sh` 解释器可能会导致脚本无法正常运行。`sh` 通常是指向系统上的 POSIX 兼容 shell（如 `dash` 或 `bash` 的 POSIX 模式）的符号链接，这意味着脚本可能会在一个不同于 Bash 的环境中运行。
2. **忽略 shebang**：`sh` 会忽略脚本中的 shebang 行，如果脚本包含 Bash 特有的语法或命令，可能会失败。

## 直接执行脚本文件

当你直接执行脚本文件时，实际的执行过程如下：

```bash
./script.sh
```

### 特点

1. **解释器根据 shebang 决定**：操作系统会读取脚本文件的 shebang 行，并使用指定的解释器来执行脚本。
2. **需要可执行权限**：脚本文件必须具有可执行权限（通过 `chmod +x script.sh` 设置）。

### 示例

同样的 `script.sh` 文件内容：

```bash
#!/bin/bash
echo "This script is running with Bash"
```

在赋予可执行权限后：

```bash
chmod +x script.sh
./script.sh
```

这将确保脚本使用 `#!/bin/bash` 中声明的 Bash 解释器来执行。

### 优点

1. **遵循 shebang**：操作系统会按照脚本中的 shebang 指定的解释器来执行脚本，确保脚本在预期的环境中运行。
2. **适应性强**：可以使用任何脚本解释器（如 `#!/usr/bin/env python`），只要在 shebang 中正确指定。

## 示例演示

假设你在 CentOS 7 系统上，默认的 `/bin/sh` 指向 `bash`。你可以通过以下命令确认这一点：

```bash
ls -l /bin/sh
```

输出结果通常为：

```bash
lrwxrwxrwx 1 root root 4 Jan  1 12:34 /bin/sh -> bash
```

这意味着 `/bin/sh` 是指向 `/bin/bash` 的符号链接。

## 总结

- **使用 `sh script.sh`**：适用于简单的 POSIX 兼容 shell 脚本，不会考虑脚本中的 shebang 声明。适合需要确保使用 POSIX 兼容 shell 执行的场景。
- **使用 `./script.sh`**：更灵活，遵循脚本中的 shebang 声明，确保在指定的解释器环境中运行。适合需要特定解释器（如 Bash）的脚本。

在编写和运行脚本时，选择适当的执行方式可以确保脚本按预期执行。如果你的脚本依赖于 Bash 特性，推荐使用 `./script.sh` 或 `bash script.sh`，以确保在 Bash 解释器下运行。