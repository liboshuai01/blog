---
title: Linux多台服务器之间免密登录
tags:
  - Linux
  - ssh
categories:
  - 运维手册
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504192355814.png'
toc: true
abbrlink: f7c16155
date: 2023-09-11 23:53:28
---

在日常的后端开发和运维工作中，我们经常需要频繁登录不同的服务器进行部署、排查问题或上传文件。传统的基于用户名和密码的登录方式既不安全又不高效。为此，我们可以利用 SSH 公钥认证的方式，实现无密码（免密）登录远程服务器，提升安全性与操作效率。

本文将一步步带你完成 SSH 公钥创建及远程服务器配置，实现 SSH 免密登录机群。

<!-- more -->

SSH 公钥认证简介
---

SSH（Secure Shell）支持多种认证方式，其中公钥认证使用非对称加密（如 RSA、ECDSA、ED25519）机制，客户端通过私钥发起认证请求，远程服务器用已保存的公钥进行验证。它的优势为：

- 更高的安全性：避免密码泄露或暴力破解
- 更好的自动化支持：适用于脚本、CI/CD 流程中免交互登录
- 更高效的运维体验：无需每次连接服务器都输入密码

生成 SSH 密钥对
---

打开终端，输入以下命令即可：

```bash
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```

参数说明：

- -t rsa：指定密钥类型为 RSA；
- -b 4096：密钥长度，建议使用 4096 位，增强安全性；
- -C：用于加一个标识注释，通常填写你的邮箱或用途说明。

执行后会提示：

```
Enter file in which to save the key (/home/youruser/.ssh/id_rsa):
```

你可以直接回车，默认保存在 ~/.ssh/id_rsa（私钥）和 ~/.ssh/id_rsa.pub（公钥）文件中。

接下来是：

```
Enter passphrase (empty for no passphrase): 
Enter same passphrase again:
```

为了增强安全性，建议设置私钥密码。如果是脚本或自动化环境中使用，可空密码，但需慎重评估风险。

将公钥添加到远程主机（server）
---

使用 ssh-copy-id 命令轻松将本地公钥写入远程服务器的 ~/.ssh/authorized_keys 文件中。

格式如下：

```bash
ssh-copy-id -i ~/.ssh/id_rsa.pub 用户名@服务器IP
```

示例：

```bash
ssh-copy-id -i ~/.ssh/id_rsa.pub hadoop@192.168.0.1
```

系统会提示输入该服务器的登录密码，认证成功后即可完成绑定。

如果该命令提示找不到 ssh-copy-id，可使用以下命令手动复制：

```bash
cat ~/.ssh/id_rsa.pub | ssh 用户名@远程IP "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

测试是否免密登录成功
---

公钥拷贝完成后，使用 ssh 命令连接远程服务器：

```bash
ssh hadoop@192.168.0.1
```

如无意外，将直接登录进入远程主机，不再需要输入密码，即可完成免密登录验证。

适用于多个目标主机（批量推送）
---

如果你需要将公钥推送至多台服务器，比如用于机器学习分布式训练、Hadoop/Spark 集群部署，可使用 shell 脚本进行批量操作。例如：

```bash
#!/bin/bash

USER="hadoop"
HOSTS=("192.168.0.1" "192.168.0.2" "192.168.0.3")

for HOST in "${HOSTS[@]}"; do
    echo "Copying SSH key to $USER@$HOST"
    ssh-copy-id -i ~/.ssh/id_rsa.pub "$USER@$HOST"
done
```

根据实际配置调整用户名或主机列表。

安全建议
---

- 私钥文件建议设置为 600 权限，仅允许用户本人读写：

  ```bash
  chmod 600 ~/.ssh/id_rsa
  ```

- 禁止将私钥随意上传到远程服务器，或在代码仓库存储私钥；
- 为私钥加密短语（passphrase），在安全与自动化之间寻求平衡；
- 若同一机器需连接多个远端主机，可以为每个主机使用配置文件（~/.ssh/config）统一管理多个密钥与主机别名，提高可维护性。

示例 ~/.ssh/config：

```bash
Host server1
    HostName 192.168.0.1
    User hadoop
    IdentityFile ~/.ssh/id_rsa

Host server2
    HostName 192.168.0.2
    User hadoop
    IdentityFile ~/.ssh/id_rsa
```

使用时无需填写完整命令：

```bash
ssh server1
```

小结
---

通过以上步骤，你就可以轻松地为自己的开发、部署流程实现高效的免密登录。无论是单机还是集群环境，该技巧都能极大地提升工作效率并加强安全性。建议将这项操作作为开发者或运维人员的必备基础技能。

如果你想在 CI/CD、Git 提交或远程 API 自动化中实现安全连接，也可以基于 SSH 做更深入的免密自动化处理。

希望本文对你有所帮助，如果遇到问题，欢迎留言讨论。🔥