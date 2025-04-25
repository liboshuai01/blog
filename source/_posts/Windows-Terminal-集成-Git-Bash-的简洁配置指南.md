---
title: Windows Terminal 集成 Git Bash 的简洁配置指南
tags:
  - Linux
categories:
  - 运维手册
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504260148288.png'
toc: true
abbrlink: f89e76f4
date: 2024-07-30 01:47:14
---

近年来，Windows Terminal 已经成为 Windows 系统中极具人气的终端工具，它支持多种命令行环境的集成，有效提升开发和运维效率。作为使用 Git Bash 进行版本控制的开发者，将 Git Bash 集成到 Windows Terminal 中，能够极大方便终端管理和切换。本文将介绍如何在 Windows Terminal 中配置并优化 Git Bash，解决常见的中文目录显示和命令兼容性问题。

<!-- more -->

### 配置 Git Bash 作为 Windows Terminal 选项

首先，打开 Windows Terminal，按以下步骤操作：

- 点击顶部下拉箭头 → 选择“设置”
- 在设置界面，点击“添加新配置文件”
- 选择“新建空配置文件”，进入“常规”标签页进行自定义

请根据您的 Git 安装路径调整配置，以下为示例配置（假设 Git 安装目录为 `D:\Program Files\Git`）：

- **名称**：Git Bash
- **命令行**：`D:\Program Files\Git\bin\bash.exe`
- **起始目录**：`%USERPROFILE%`
- **图标路径**：`D:\Program Files\Git\mingw64\share\git\git-for-windows.ico`

完成后保存，Windows Terminal 重新启动即可在侧边菜单看到 Git Bash 选项，实现一站式管理多终端环境。

### 优化 Git Bash 中文目录显示及命令兼容

默认 Git Bash 在 Windows Terminal 中，遇到含中文字符的目录时可能出现乱码，且不少 Linux 常用别名命令如 `ll` 默认不可用。为提升体验，请按下述步骤优化：

1. 使用管理员权限打开 Windows Terminal，进入 Git Bash 终端。
2. 编辑全局 bash 配置文件：

```bash
vim /etc/bash.bashrc
```

3. 在文件末尾追加如下配置，开启 UTF-8 编码支持并定义有用别名：

```bash
# 设置 Git Bash 支持中文显示
export LANG="zh_CN.UTF-8"
export LC_ALL="zh_CN.UTF-8"

# 让 ls 和 dir 命令支持显示中文及彩色输出
alias ls='ls --show-control-chars --color=auto'
alias dir='dir -N --color=auto'

# 设置简易中文环境提示
export LANG="zh_CN"

# 设置输出编码
export OUTPUT_CHARSET="utf-8"

# 启用 ll 命令为 ls 的详细列表格式
alias ll='ls -l'
```

4. 保存文件并重新启动 Git Bash。

这样，打开 Git Bash 时不仅能正确显示中文目录，也能直接使用 `ll` 等习惯命令，提升操作效率。

### 版本信息参考

为了保证兼容性，本文测试环境如下：

- **Windows Terminal** 版本：1.11.3471.0
- **Git for Windows** 版本：2.32.0.windows.1

请根据自身软件版本，适当调整相关配置。

### 总结

将 Git Bash 集成至 Windows Terminal，配合编码和别名优化，是提升 Windows 上多终端工作流体验的必备手段。通过简单的配置步骤，您可以实现终端环境的整合与个性化定制，极大便利日常开发工作。欢迎实践并根据业务需求扩展更多配置，实现高效且美观的终端操作体验。

---

如果您在配置中遇到其他问题或有更好建议，欢迎留言交流，共同打造更友好的开发环境！