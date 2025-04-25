---
title: Linux命令行终端实现简单回收站功能
tags:
  - Linux
categories:
  - 运维手册
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260132291.png '
toc: true
abbrlink: bab3c95a
date: 2024-07-10 01:30:56
---

在 Linux 系统中，`rm` 命令会直接删除文件，无法撤销。这容易导致误删文件的情况发生。为了避免这种风险，我们可以为 `rm` 命令添加一个回收站功能。本文将介绍如何编写并部署一个 Bash 脚本，使系统中的所有用户在使用 `rm` 命令时，文件会被移动到回收站目录，而不是被永久删除。

<!-- more -->

## 脚本实现

以下是实现回收站功能的 Bash 脚本：

```bash
#!/bin/bash

alias rm='trash'
alias rl='trashlist'

LocalTrash="$HOME/.local/share/Trash/files"

trash() {
    local files=""
    [ ! -d $LocalTrash ] && mkdir -p $LocalTrash
    while [ -n "$1" ]; do
        # 移除 -r, -f 等参数
        if [[ ! $1 =~ ^- ]]; then
            if [ -z "$files" ]; then
                files=$1
            else
                files=$files" $1"
            fi
        fi
        shift
    done
    mv --backup="numbered" $files $LocalTrash
}

trashlist() {
    echo -e "\n================= Trash Contents ================="
    echo -e "Location: $LocalTrash"
    echo -e "\nUse 'cleartrash' to permanently delete all files in Trash!\n"
    ls --color=auto -lh -S $LocalTrash | more
}

cleartrash() {
    echo -e "\nWARNING: This will permanently delete all files in Trash."
    echo -ne "Are you sure you want to clear the Trash? (y/N): "
    read confirm
    if [[ "$confirm" =~ ^[Yy]$ ]]; then
        /bin/rm -rf $LocalTrash/* &> /dev/null
        /bin/rm -rf $LocalTrash/.* &> /dev/null
        echo -e "Trash has been cleared.\n"
    else
        echo -e "Clear operation cancelled.\n"
    fi
}
```

### 代码解释

- **定义别名**：
    - `alias rm='trash'`：将 `rm` 命令替换为 `trash` 函数，防止误删文件。
    - `alias rl='trashlist'`：定义 `rl` 命令，用于列出回收站中的文件。

- **定义回收站目录**：
    - `LocalTrash="$HOME/.local/share/Trash/files"`：定义回收站目录路径。

- **`trash` 函数**：
    - 检查回收站目录是否存在，不存在则创建。
    - 移除 `-r`, `-f` 等参数，避免影响文件移动。
    - 将文件移动到回收站目录，并使用 `--backup="numbered"` 参数防止文件名冲突。

- **`trashlist` 函数**：
    - 显示回收站中的文件列表。
    - 提示如何清空回收站。

- **`cleartrash` 函数**：
    - 提示用户确认是否清空回收站。
    - 使用 `rm -rf` 命令删除回收站中的所有文件。

## 部署脚本

为了让系统中的每个用户都能使用此脚本，需要将脚本添加到全局的 Bash 配置文件中。以下是具体步骤：

### 步骤 1: 创建脚本文件

将脚本内容保存到一个全局可访问的位置，例如 `/usr/local/bin/trash.sh`：

```bash
sudo vim /usr/local/bin/trash.sh
```

将上述脚本内容粘贴到文件中，保存并退出。

### 步骤 2: 设置脚本的执行权限

```bash
sudo chmod +x /usr/local/bin/trash.sh
```

### 步骤 3: 修改全局配置文件

将脚本加载到所有用户的环境中。可以通过编辑 `/etc/profile` 文件做到这一点：

```bash
sudo vim /etc/profile
```

在文件末尾添加以下内容：

```bash
# Load custom trash script for all users
if [ -f /usr/local/bin/trash.sh ]; then
    source /usr/local/bin/trash.sh
fi
```

保存并退出。

### 步骤 4: 重新加载配置

为了使更改立即生效，可以提示用户重新加载 Bash 配置文件，或者重启终端会话：

```bash
source /etc/profile
```

## 使用教程

### 1. 删除文件

现在，所有用户在使用 `rm` 命令删除文件时，文件将被移动到回收站：

```bash
rm file.txt
```

### 2. 查看回收站

使用 `rl` 命令查看回收站中的文件：

```bash
rl
```

### 3. 清空回收站

使用 `cleartrash` 命令清空回收站：

```bash
cleartrash
```

用户需确认后才会执行删除操作。

### 4. 直接删除文件，不放入回收站

```bash
/usr/bin/rm file.txt
```

## 结语

通过上述步骤，我们实现了在系统中为所有用户添加自定义的回收站功能。每当用户执行 `rm` 命令时，文件将被安全地移动到回收站目录，而不是被直接删除。这为用户提供了一个安全网，防止误删文件。希望这些步骤对你有所帮助。