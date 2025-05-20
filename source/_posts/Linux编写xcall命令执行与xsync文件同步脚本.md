---
title: Linux编写xcall命令执行与xsync文件同步脚本
tags:
  - Linux
  - 脚本
categories:
  - 运维手册
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504200424630.png'
toc: true
abbrlink: c8cd0634
date: 2024-05-01 04:22:58
---

在多机集群环境中，运维人员经常需要对多台服务器执行相同操作或同步分发文件，这不仅繁琐而且容易出错。为了提升工作效率和操作一致性，本文介绍了两个实用的 Linux 脚本工具——xcall 和 xsync。xcall 脚本支持多主机并行执行指定命令，极大地方便了集群管理和批量任务执行；xsync 脚本则通过 rsync 实现了多台机器间的文件目录同步，确保环境配置和数据一致。本文详细说明了脚本的创建方法与使用示例，帮助大家轻松完成中间件集群安装与配置工作。

<!-- more -->

## 环境准备

- 一台以上的centos服务器
- 各个机器之间完成的ssh免密

> 关于如何`ssh免密`，推荐我的博文 [# centos之间ssh免密](https://lbs.wiki/pages/Linux%E5%A4%9A%E5%8F%B0%E6%9C%8D%E5%8A%A1%E5%99%A8%E4%B9%8B%E9%97%B4%E5%85%8D%E5%AF%86%E7%99%BB%E5%BD%95/)

## `xsync`

### 创建配置

1. 集群中的机器都需要执行下面的命令，进行安装`rsync`
    ```shell
    sudo yum install rsync -y
    ```

2. 在需要的节点上执行下面的命令，以便创建`xsync`
   > 执行命令之前，注意修改`for i in master node1 node2`为自己的主机ip

   ```
   sudo tee /usr/local/bin/xsync <<'EOF'
   #!/bin/bash
   
   # 获取输出参数，如果没有参数则直接返回
   pcount=$#
   if [ $pcount -eq 0 ]; then
       echo "No parameter found!";
       exit 1;
   fi
   
   # 获取传输文件名
   p1=$1
   filename=$(basename "$p1")
   echo "Load file $p1 success!"
   
   # 获取文件的绝对路径
   pdir=$(cd -P $(dirname "$p1") && pwd)
   echo "File path is $pdir"
   
   # 获取当前用户（如果想使用root用户权限拷贝文件，在命令后加入-root参数即可）
   user=$2
   case "$user" in
       "-root")
           user="root"
           ;;
       "")
           user=$(whoami)
           ;;
       *)
           echo "Illegal parameter $user"
           exit 1
           ;;
   esac
   
   echo "Using user: $user"
   
   # 确保从机的目标目录存在
   for i in master node1 node2; do
       echo "================current host is $i================="
       ssh "$user@$i" "mkdir -p $pdir"
       rsync -av "$pdir/$filename" "$user@$i:$pdir"
       if [ $? -ne 0 ]; then
           echo "Rsync to $i failed!"
           exit 1
       fi
   done
   
   echo "Complete!"
   EOF
   sudo chmod +x /usr/local/bin/xsync
   ```

### 使用示例
将`one.txt`分发到集群的其他机器的相同路径下
```shell
[admin@master tmp]$ xsync one.txt 
load file one.txt success !
file path is /home/admin/tmp
admin
================current host is master=================
sending incremental file list

sent 43 bytes  received 12 bytes  110.00 bytes/sec
total size is 2  speedup is 0.04
================current host is node1=================
sending incremental file list
one.txt

sent 92 bytes  received 35 bytes  254.00 bytes/sec
total size is 2  speedup is 0.02
================current host is node2=================
sending incremental file list
one.txt

sent 92 bytes  received 35 bytes  84.67 bytes/sec
total size is 2  speedup is 0.02
complate !
```

## `xcall`

### 创建配置

在需要的节点上执行下面命令即可完成创建配置`xcall`
> 执行命令之前，注意修改`for i in master node1 node2`为自己的主机ip

```
sudo tee /usr/local/bin/xcall <<'EOF'
#!/bin/bash

# 获取控制台指令
cmd="$*"

# 判断指令是否为空
if [ -z "$cmd" ]; then
        echo "command cannot be null!"
        exit 1
fi

# 获取当前登录用户
user=$(whoami)

# 在从机执行指令,这里需要根据你具体的集群情况配置，host与具体主机名一致
for i in master node1 node2; do
        echo "================current host is $i================="
        echo "--> execute command "$cmd""

        # 通过 SSH 执行命令
        if ssh ${user}@${i} "source ~/.bash_profile; $cmd"; then
            echo "Command executed successfully on $i"
        else
            echo "Failed to execute command on $i"
            exit 1
        fi
done

echo "All commands executed successfully!"
EOF
sudo chmod +x /usr/local/bin/xcall
```

### 使用示例
使用 `xcall` 脚本查看集群中所有机器的java进程。
```shell
[admin@master bin]$ xcall jps
================current host is master=================
--> execute command "jps"
25985 JobHistoryServer
24419 DataNode
26261 Jps
24219 NameNode
25292 ResourceManager
25437 NodeManager
24750 SecondaryNameNode
Command executed successfully on master
================current host is node1=================
--> execute command "jps"
3545 DataNode
3690 NodeManager
3948 Jps
Command executed successfully on node1
================current host is node2=================
--> execute command "jps"
3609 DataNode
3754 NodeManager
4013 Jps
Command executed successfully on node2
All commands executed successfully!
```

## 结语

通过 xcall 和 xsync 脚本的应用，运维人员能够显著简化多机环境下的重复操作与文件同步任务，降低人为操作风险，提升工作效率。这两个工具兼具易用性与实用性，适合各种中间件集群及多节点部署场景。希望本文分享的脚本和方法能为您的运维管理带来便利，推动集群运维自动化的进程。后续可根据实际需求进一步扩展脚本功能，实现更智能化的集群管理