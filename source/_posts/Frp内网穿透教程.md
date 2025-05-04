---
title: Frp内网穿透教程
tags:
  - Linux
categories:
  - 运维手册
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504181203794.png'
toc: true
abbrlink: b2dafbdf
date: 2022-12-02 08:01:00
---

Frp（Fast Reverse Proxy）是一款高性能的内网穿透工具，广泛应用于解决没有公网IP的本地服务如何向公网暴露访问入口的问题。

通过部署Frp服务端于拥有公网IP的云服务器，同时在本地内网环境的主机上运行客户端，便可将内网服务如Web、SSH等安全高效地映射到公网指定端口，实现远程访问。

本教程详细介绍了Frp的部署流程，包括服务端与客户端的安装配置、systemd自启服务的设置，以及如何通过Frp灵活穿透Nginx服务。

同时，文中囊括了常见的安全配置要点和实时流量监控方法，帮助运维和开发人员快速、规范地实现内网穿透。

<!-- more -->

环境准备
---

- 一个拥有公网IP的云服务器
- 一个本地内网环境的虚拟机
- 云服务器与本地虚拟机均需关闭SeLinux

> 笔者使用`lbs`用户安装，并安装到`/home/lbs/software/frp`下，读者可根据实际情况修改。

服务端安装配置
---

1. 安装Frp

    ```shell
    wget https://github.com/fatedier/frp/releases/download/v0.62.0/frp_0.62.0_linux_amd64.tar.gz
    # 注意修改路径
    tar -zxvf frp_0.62.0_linux_amd64.tar.gz -C /home/lbs/software
    mv /home/lbs/software/frp_0.62.0_linux_amd64 /home/lbs/software/frp
    rm -rf frp_0.62.0_linux_amd64.tar.gz
    ```

2. 配置`frps.toml`文件

   ```shell
   tee /home/lbs/software/frp/frps.toml <<'EOF'
   # 监听所有地址，如果你想使用特定的IP地址
   bindAddr = "0.0.0.0"
   
   # 默认使用7000作为服务端口，建议改成其他端口。
   bindPort = 7000
   
   # 用于客户端和服务器通信的身份验证令牌
   auth.method = "token"
   # 秘钥格式建议配置 用户+@+密码 的格式，方便区分用户，如下所示：
   auth.token = "user1@12345678"
   
   # 允许的端口范围，不设置允许所以端口，笔者允许使用7101-7199端口和6001端口。
   allowPorts = [
   { start = 7101, end = 7199 },
   { single = 6001 }
   ]
   
   # 面板相关配置（可选），配置面板后可以查看代理使用的流量统计。
   webServer.addr = "0.0.0.0"
   webServer.port = 7500
   webServer.user = "admin"
   webServer.password = "your_pwd"
   
   # 最大连接池数量
   transport.maxPoolCount = 5
   
   # 指定日志文件存储级别和位置，最大存储日志为7天。
   log.to = "/home/lbs/software/frp/frps.log"
   log.level = "info"
   log.maxDays = 7
   EOF
   ```

3. 配置`systemd`
   > 使用`systemd`管理frps时，不要使用restart命令，而是先stop，再start。

   ```shell
   sudo tee /etc/systemd/system/frps.service  <<'EOF'
   [Unit]
   Description=Frps Service
   Wants=network-online.target
   After=network.target network-online.target
   Requires=network-online.target
   
   [Service]
   Type=simple
   # 注意修改用户及用户组
   User=lbs
   Group=lbs
   # 注意修改路径
   ExecStart=/home/lbs/software/frp/frps -c /home/lbs/software/frp/frps.toml
   ExecStop=/usr/bin/kill -15 $MAINPID
   ExecReload=/usr/bin/kill -HUP $MAINPID
   RemainAfterExit=yes
   PrivateTmp=true
   # file size
   LimitFSIZE=infinity
   # cpu time
   LimitCPU=infinity
   # virtual memory size
   LimitAS=infinity
   # open files
   LimitNOFILE=65535
   # processes/threads
   LimitNPROC=64000
   # locked memory
   LimitMEMLOCK=infinity
   # total threads (user+kernel)
   TasksMax=infinity
   TasksAccounting=false
   
   [Install]
   WantedBy=multi-user.target
   EOF
   ```

4. 启动服务端

   ```shell
   sudo systemctl daemon-reload
   sudo systemctl enable frps
   sudo systemctl start frps
   ```

5. 查看控制面板

   > 浏览器访问`云主机IP:7500`，并输入用户名密码

   ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504182211874.png)

客户端安装配置
---

1. 安装Frp

    ```shell
    wget https://github.com/fatedier/frp/releases/download/v0.62.0/frp_0.62.0_linux_amd64.tar.gz
    # 注意修改路径
    tar -zxvf frp_0.62.0_linux_amd64.tar.gz -C /home/lbs/software
    mv /home/lbs/software/frp_0.62.0_linux_amd64 /home/lbs/software/frp
    rm -rf frp_0.62.0_linux_amd64.tar.gz
    ```

2. 配置`frpc.toml`文件

   ```shell
   tee /home/lbs/software/frp/frpc.toml <<'EOF'
   # 指定服务器地址与端口号。
   serverAddr = "云主机IP"
   serverPort = 7000
   # 代理名称将会改为 {user}.{proxy}，建议配置，取服务器上的token中@前部分。
   user = "user1"
   auth.method = "token"
   auth.token = "user1@12345678"
   
   # 面板相关配置（可选），配置面板后可以查看代理使用的流量统计。
   webServer.addr = "0.0.0.0"
   webServer.port = 7500
   webServer.user = "admin"
   webServer.password = "your_pwd"
   
   # 指定日志文件存储级别和位置，最大存储日志为7天。
   log.to = "/home/lbs/software/frp/frpc.log"
   log.level = "info"
   log.maxDays = 7
   
   [[proxies]]
   # name为应用名称，type为协议类型，localIP为内网服务的地址，localPort为内网服务的端口，remotePort为服务器上监听的端口，transport.bandwidthLimit为限速，服务端不限制使用端口，不建议使用1-1024端口。
   name = "nginx"
   type = "tcp"
   localIP = "127.0.0.1"
   localPort = 80
   remotePort = 7101
   transport.bandwidthLimit = "1MB"
   EOF
   ```

3. 配置`systemd`
   > 使用`systemd`管理frpc时，不要使用restart命令，而是先stop，再start。

   ```shell
   sudo tee /etc/systemd/system/frpc.service  <<'EOF'
   [Unit]
   Description=Frpc Service
   Wants=network-online.target
   After=network.target network-online.target
   Requires=network-online.target
   
   [Service]
   Type=simple
   # 注意修改用户及用户组
   User=lbs
   Group=lbs
   # 注意修改路径
   ExecStart=/home/lbs/software/frp/frpc -c /home/lbs/software/frp/frpc.toml
   ExecStop=/usr/bin/kill -15 $MAINPID
   ExecReload=/usr/bin/kill -HUP $MAINPID
   RemainAfterExit=yes
   PrivateTmp=true
   # file size
   LimitFSIZE=infinity
   # cpu time
   LimitCPU=infinity
   # virtual memory size
   LimitAS=infinity
   # open files
   LimitNOFILE=65535
   # processes/threads
   LimitNPROC=64000
   # locked memory
   LimitMEMLOCK=infinity
   # total threads (user+kernel)
   TasksMax=infinity
   TasksAccounting=false
   
   [Install]
   WantedBy=multi-user.target
   EOF
   ```

4. 启动服务端

   ```shell
   sudo systemctl daemon-reload
   sudo systemctl enable frpc
   sudo systemctl start frpc
   ```

5. 查看控制面板

   > 浏览器访问`虚拟机IP:7500`，并输入用户名密码

   ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504182232322.png)

验证测试
---

> 需要提前在虚拟机上安装nginx，并配置80端口页面

1. 浏览器访问`虚拟机IP:80`，查看nginx页面

   ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504182234827.png)

2. 浏览器访问`云主机IP:7101`，同样可以查看到nginx页面，表示内网穿透成功。

   ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504182232012.png)

结语
---

通过Frp的部署与配置，可以显著提升远程管理和运维内网主机的效率，解决了传统端口映射和VPN复杂部署的问题，实现了数据安全和运维便捷的平衡。

随着云计算和分布式架构的普及，Frp等内网穿透工具在异地办公、IoT设备远程管理等场景下愈加重要。

建议在运用过程中注意身份认证、端口规控等安全措施，确保数据安全和服务稳定。

希望本教程能够帮助你顺利搭建内网穿透环境，为业务创新和系统运维提供有力支撑。