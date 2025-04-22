---
title: Linux通过V2raya科学上网
tags:
  - Linux
  - V2raya
categories:
  - 运维手册
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504130200062.png'
toc: true
abbrlink: acf97f74
date: 2023-10-14 22:00:10
---

本文详细介绍了如何在Linux系统下使用V2raya进行科学上网设置。文章首先提供了一键安装方案，通过简单的命令下载相关脚本和安装包，实现V2ray及V2raya的快速部署。该方法适合对Linux操作较熟练的用户，可以迅速完成基础环境搭建。

另外，博文还介绍了手动安装步骤，通过获取最新的下载地址来更新安装脚本，确保软件正常运行。安装后，用户可通过浏览器创建账号、导入订阅链接并测试节点延迟，最后选择合适的节点启动服务。整个过程配合图示，既有针对RedHat、Arch和Debian等不同系统的详细说明，也涵盖了系统代理规则的设置，适合不同需求的用户参考和实践。

<!-- more -->

一键安装
---

> 注意：要使用root用户，如果链接失效，请进行手动安装

```shell
wget https://lbs-install.oss-cn-shanghai.aliyuncs.com/v2raya/install_v2ray.sh
bash ./install_v2ray.sh
wget https://gh-proxy.com/github.com/v2rayA/v2rayA/releases/download/v2.2.5.8/installer_redhat_x64_2.2.5.8.rpm
sudo rpm -i installer_redhat_x64_2.2.5.8.rpm
sudo systemctl enable v2raya && sudo systemctl start v2raya
```

手动安装
---

### 安装v2ray
1. 浏览器访问install-release.sh，复制网页中的所有内容
   ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504130125572.png)

2. 登录服务器，执行`vim install.sh`命令，粘贴内容

3. 定位到294行，将这里的链接复制到`https://gh-proxy.com/`中，进行下载，然后获取新的下载地址，最后用新的下载地址替换之前的。
   ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504130126293.png)

4. 使用root用户执行bash install.sh命令，进行v2ray的安装
   ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504130126148.png)

5. 执行命令sudo systemctl start v2ray && sudo systemctl status v2ray，测试v2ray是否正常运行
   ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504130126997.png)

6. 执行命令sudo systemctl stop v2ray，关闭v2ray（因后续安装v2raya，不需要单独运行v2ray）

### 安装v2raya

> 保证2017端口没有被占用

1. 执行命令下载安装包
   这里同理，需要到github下载加速网站`https://gh-proxy.com/`中获取新的下载地址
   - redhat系列：`https://github.com/v2rayA/v2rayA/releases/download/v2.2.5.8/installer_redhat_x64_2.2.5.8.rpm`
   - arch系列：`https://github.com/v2rayA/v2rayA/releases/download/v2.2.5.8/installer_archlinux_x64_2.2.5.8.pkg.tar.zst`
   - debain系列：`https://github.com/v2rayA/v2rayA/releases/download/v2.2.5.8/installer_debian_x64_2.2.5.8.deb`

2. 执行命令进行v2raya的安装
   - redhat系列：`sudo rpm -i installer_redhat_x64_2.2.5.8.rpm`
   - arch系列: `sudo pacman -U installer_archlinux_x64_2.2.5.8.pkg.tar.zst`
   - debian系列: `sudo dpkg -i installer_debian_x64_2.2.5.8.deb`

3. 执行下面命令设置v2raya开机自启与并启动v2raya
    ```shell
    sudo systemctl enable v2raya && sudo systemctl start v2raya
    ```

4. 浏览器访问服务器`IP:2017`，创建初始账号密码
   ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504130128876.png)

5. 获取v2ray订阅链接
   ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504130128261.png)

6. 返回v2raya的web页面导入订阅链接
   ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504130128333.png)

7. 测试各个节点的延迟，判断哪个节点可用
   ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504130128948.png)

8. 选择可用的节点（可选择多个，最好6个以内），并启动
   ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504130129164.png)

9. 设置系统代理规则
   ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504130129022.png)