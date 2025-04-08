---
title: Hexo搭建个人博客至服务器
date: 2025-04-08 14:15:24
tags: [环境搭建,Linux,Hexo]
categories: [环境搭建]
cover: https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250408141637337.png
---

## 环境推荐

- 本地环境：Windows11
- 服务器环境：RockyLinux 8.10（Centos系列）

## 前言

笔者之前一直在[稀土掘金](https://juejin.cn/user/1878395516357111)中记录个人技术博文，但是掘金毕竟是第三方独立运营的平台，而且还不提供历史文章导出，等于就将笔者绑死在掘金平台了。
一旦掘金出现了什么删库跑路、或者历史数据丢失等问题，我的技术博文就和说拜拜了，让我很没有安全感，于是想起来自己之前折腾过的个人博客。随又捡起了Hexo博客，并记录一下搭建过程。

## 服务器环境搭建

> 为了方便我直接使用了root用户进行操作，如果想要使用其他用户进行搭建要主要权限问题哦！

### 安装git环境

```shell
sudo yum install -y git
```

### 安装nodejs与pnpm环境

打开[node.js-download](https://nodejs.org/en/download)这个链接，选择符合自己的系统环境复制命令到终端进行下载即可。

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250408144451542.png)

### 安装nginx环境

```shell
# 通过yum安装nginx
sudo yum install -y nginx
# 启动nginx开机自启
sudo systemctl enable nginx
# 启动nginx服务
sudo systemctl start nginx
# 查看nginx状态
sudo systemctl status nginx
```

### 创建博客目录与git钩子

> 创建git钩子是为了当本地环境执行`hexo d`时可以实时更新服务器的博文内容

```shell
# 创建博客目录
mkdir -p /root/document/blog
# 创建git钩子
cd /root/document && git init --bare hexo.git
sudo tee /root/document/hexo.git/hooks/post-receive <<'EOF'
git --work-tree=/root/document/blog --git-dir=/root/document/hexo.git checkout -f
EOF
# 基于钩子文件执行权限
sudo chmod +x /root/document/hexo.git/hooks/post-receive
```

### 配置nginx代理

1. 修改`/etc/nginx/nginx.conf`文件

   ```txt
   user root;
   worker_processes auto;
   error_log /var/log/nginx/error.log;
   pid /run/nginx.pid;
   
   # Load dynamic modules. See /usr/share/doc/nginx/README.dynamic.
   include /usr/share/nginx/modules/*.conf;
   
   events {
       worker_connections 1024;
   }
   
   http {
       log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                         '$status $body_bytes_sent "$http_referer" '
                         '"$http_user_agent" "$http_x_forwarded_for"';
   
       access_log  /var/log/nginx/access.log  main;
   
       sendfile            on;
       tcp_nopush          on;
       tcp_nodelay         on;
       keepalive_timeout   65;
       types_hash_max_size 2048;
   
       include             /etc/nginx/mime.types;
       default_type        application/octet-stream;
   
       include /etc/nginx/conf.d/*.conf;
   }
   ```

2. 新增`/etc/nginx/conf.d/blog.conf`配置文件
   
   > 如果仅有公网ip、没有域名的情况，配置文件内容如下

   ```txt
   server {
      listen        80;
      listen   [::]:80;
      # 配置为自己的公网IP
      server_name  106.14.19.58;
      
      location / {
        # 配置为服务器中个人博客的目录
        root   /root/document/blog;
        index  index.html index.htm;
      }
      
      error_page   500 502 503 504  /50x.html;
      location = /50x.html {
        root   /usr/share/nginx/html;
      }
   }
   ```

   > 如果有域名、且有SSL证书的情况，配置文件内容如下

   ```txt
   server {
       listen        80;
       listen   [::]:80;
       # 配置为自己的域名
       server_name  liboshuai.icu;
   
       rewrite ^(.*)$ https://${server_name}$1 permanent;
   }
   
   server {
       listen  443 ssl;
       # 配置为自己的域名
       server_name  liboshuai.icu;
   
       # 配置为自己ssl证书pem文件的路径
       ssl_certificate   /etc/nginx/ssl/liboshuai.icu.pem;
       # 配置为自己ssl证书key文件的路径
       ssl_certificate_key   /etc/nginx/ssl/liboshuai.icu.key;
   
       ssl_session_cache   shared:SSL:1m;
       ssl_session_timeout   5m;
   
       ssl_ciphers HIGH:!aNULL:!MD5;
       ssl_prefer_server_ciphers   on;
      
       location / {
           proxy_set_header   X-Real-IP        $remote_addr;
           proxy_set_header   Host             $http_host;
           proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
           # 配置为服务器中个人博客的目录
           root   /root/document/blog;
           index  index.html index.htm;
       }
   
       error_page   500 502 503 504  /50x.html;
       location = /50x.html {
           root   /usr/share/nginx/html;
       }
   }
   ```
   
3. 重新加载nginx配置

   ```shell
   # 检查nginx配置文件是否正确
   /usr/sbin/nginx -t
   # 热加载nginx配置文件
   /usr/sbin/nginx -s reload
   ```

## 本地环境搭建

### 安装git环境

打开[git-download](https://git-scm.com/downloads)链接，选择符合自己的系统点击进行下载即可，随后打开安装包一直进行下一步即可。

随后配置git全局变量

```shell
# 修改为自己的邮箱
git config --global user.email "liboshuai01@gmail.com"
# 修改为自己的英文名
git config --global user.name "BoShuai Li"
```

### 安装nodejs与yarn环境

打开[node.js-download](https://nodejs.org/en/download)这个链接，选择符合自己的系统环境复制命令到终端进行下载安装即可。

![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250408144428675.png)

### 安装hexo环境

1. 新建一个目录用于存储个人博客数据，例如：`C:\Me\Project\other`。

2. 在新建的目录中打开`git bash`终端，执行下面的命令。

    > 一定要在`C:\Me\Project\other`目录下执行

    ```shell
    # 全局安装hexo-cli
    yarn install -g hexo-cli
    # 创建并进入博客目录
    hexo init blog && cd blog
    # 安装Hexo项目依赖
    yarn install
    # 启动Hexo本地服务
    hexo s
    ```
        
    > 下面是安装结束后的目录结构

    ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250408145801328.png)

    > 在浏览器打开`http://localhost:4000`查看页面

    ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250408144547941.png)

3. 安装icarus主题

    > 一定要在`C:\Me\Project\other\\blog`目录下执行
    
    ```shell
    # 通过yarn安装icarus主题
    yarn add hexo-theme-icarus hexo-renderer-inferno
    # 配置hexo当前主题为icarus
    hexo config theme icarus
    # 重新启动hexo验证主题是否生效
    hexo s
    ```
   
   > icarus主题生效后，目录中多出了`_config.icarus.yml`这个文件
   
   ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250408145905186.png)
   
   > 在浏览器打开`http://localhost:4000`查看页面
   
   ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250408150011050.png)

4. 配置远程部署地址，修改`_config.yml`文件中的`deploy`项

   > 注意：我的ssh端口为22222，而非22

   ```yaml
   deploy:
     type: 'git'
     repo: ssh://root@106.14.19.58:22222/root/document/hexo.git
     branch: master
   ```
   
5. 配置本地SSH免密登录服务器

   ```shell
   # 生成密钥，随后一路回车
   ssh-keygen -t rsa
   # 这一步的目的是将本地生产的`id_rsa.pub`文件的内容拷贝到服务器上的`~/.ssh/authorized_keys`文件中
   # 当然也可以自己手动将生成的`id_rsa.pub`文件内容拷贝到服务器上的`~/.ssh/authorized_keys`文件中
   ssh -p 22222 root@106.14.19.58 "cat >> ~/.ssh/authorized_keys" < C:\Users\libos\.ssh\id_rsa.pub
   ```
   
   > 现在需要测试一下本地是否可以免密登录到服务器了

   ```shell
   ssh -p 22222 root@106.14.19.58
   ```
   
6. 发布博客内容到服务器
   
   ```shell
   hexo clean && hexo d -g
   ```
   
7. 浏览器打开`https://liboshuai.icu`链接访问博客页面，进行查看

## 主题美化

## 结语
