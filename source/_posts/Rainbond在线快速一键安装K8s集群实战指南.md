---
title: Rainbond在线快速一键安装K8s集群实战指南
tags:
  - 环境搭建
  - Linux
  - k8s
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425110426306.png'
toc: true
abbrlink: '45248e98'
date: 2024-04-14 11:02:41
---

新人学习k8s最难的第一步就是安装k8s集群，笔者为了找到一个省心省力的安装方式也是煞费苦心。前前后后使用过`kubeadm`、`kubekey`、
`rancher`等方式，但是无不一存在的大大小小的坑, 属实让我吃了不少的罪。但是今天介绍的[Rainbond](https://www.rainbond.com/)
，真的惊艳到了我，全程一二十分钟、零报错，一个k8s集群从0到1就起来了【当然也有坑，但是我已经解决了】。真的是小白的福利哦，下面就跟着我开始吧！

<!-- more -->

## 系统环境

![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425110028568.png)

> 最好之前把防火墙关闭

## 集群规划

> 完全可以不按照笔者的规则来，甚至只需要一个节点也可以

| 角色    | IP地址      | hostname | 配置（最少）     |
|-------|-----------|----------|------------|
| 主节点   | 10.0.0.87 | master   | 2核4G 硬盘40G |
| 普通节点1 | 10.0.0.88 | node1    | 2核4G 硬盘40G |
| 普通节点2 | 10.0.0.89 | node2    | 2核4G 硬盘40G |
| 普通节点3 | 10.0.0.90 | node2    | 2核4G 硬盘40G |

> 建议您的操作系统处于干净状态（不安装任何其他软件），否则可能会发生冲突。

集群安装成功后的版本情况：

- docker: 24.0.7
- docker-compose: v2.22.0
- k8s: v.1.23.10
- helm: v.3.10.1

## 正式安装

1. 所有节点执行安装docker的命令
    ```shell
    curl -sfL https://get.rainbond.com/install_docker | bash
    ```

2. 在master节点执行安装Rainbond控制台的命令
    ```shell
    docker run -d -p 7070:7070 \
    --name=rainbond-allinone --restart=always \
    -v ~/.ssh:/root/.ssh \
    -v ~/rainbonddata:/app/data \
    registry.cn-hangzhou.aliyuncs.com/goodrain/rainbond:v5.17.1-release-allinone
    ```

3. 浏览器访问`http://IP:7070`，登录 Rainbond 后，进入 **平台管理 > 集群 -> 添加集群 -> 从主机开始安装** 进入图形化安装页面。

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425110028795.png)

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425110028875.png)

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425110028752.png)

4. 复制[如图所示的命令 + 下面的命令]到所有节点执行一遍后，再点击“开始安装，然后等待10分钟左右。
    ```shell
    # 上图中rainbond生成的初始化节点命令（省略）
    
    # 同步时间
    yum install -y ntpdate
    ntpdate -u ntp.aliyun.com
    
    # 硬件时间同步
    hwclock -w
    ```

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425110029130.png)

5. 配置`kubectl`和`helm`命令行工具
   > 可以只在主节点上执行，也可以各个节点都执行
    ```shell
    wget https://pkg.goodrain.com/pkg/kubectl/v1.23.10/kubectl -O /usr/local/bin/kubectl && chmod +x /usr/local/bin/kubectl
    wget https://pkg.goodrain.com/pkg/helm/v3.10.1/helm -O /usr/local/bin/helm && chmod +x /usr/local/bin/helm
    mkdir ~/.kube/
    vi ~/.kube/config
    ```
   其中`vi ~/.kube/config`是为了将下图中的文件内容复制粘贴过来。

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425110028710.png)
   最后执行下面的命令，解决权限问题。
    ```shell
    chmod g-rw ~/.kube/config && chmod o-r ~/.kube/config
    ```

   验证：分别执行`kubectl get no`和`helm ls`，有结果输出即命令行工具配置成功。

6. 下一步初始化平台集群

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425110031699.png)

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425110036942.png)

7. 等待系统自动完成初始化集群的操作即可，大概10分钟左右

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425110037330.png)

8. 最后设置一下集群的ID和名称，k8s集群正式安装完毕

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425110037169.png)

9. 验证k8s集群节点和pod情况

   ![image.png](https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425110037221.png)

## 结语

整体来说，通过rainbond在线安装k8s全程都是十分丝滑的，没有遇到大坑让人到处找资料半天解决不了。

如果安装过程中还是遇到的报错的情况，可以参考官网给出的[异常案例解决方案](https://www.rainbond.com/docs/troubleshooting/installation/ui/)。