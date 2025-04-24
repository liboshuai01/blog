---
title: Linux离线安装Harbor-2.9.1全攻略
abbrlink: dc7d32cd
date: 2023-12-02 18:22:01
tags:
  - Linux
  - Centos
  - Docker
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250422182310000.png'
toc: true
---

在企业内部构建高效、可靠的私有Docker镜像仓库，是保障容器化应用稳定交付的关键。Harbor作为业界领先的云原生镜像仓库项目，具备强大的安全策略、权限管理与镜像扫描能力。本文将围绕
**Harbor 2.9.1版本的离线安装**展开，从环境准备、安装包获取、配置策略到日后管理，进行全面且细致的梳理，助你快速搭建稳定的私有镜像仓库。

<!-- more -->

---

## Harbor离线安装前的准备工作

### 1. 环境硬件需求

| 硬件配置 | 最低要求  | 推荐配置   |
|------|-------|--------|
| CPU  | 2 核   | 4 核    |
| 内存   | 4 GB  | 8 GB   |
| 硬盘   | 40 GB | 160 GB |

*说明*：合理的硬件资源配置能保证Harbor的流畅运行，尤其是在镜像存储和访问压力较大的企业级场景。

### 2. 软件依赖版本要求

| 软件组件           | 版本要求          |
|----------------|---------------|
| Docker Engine  | 17.06.0-ce及以上 |
| Docker Compose | 1.18.0及以上     |

> **注意**：Docker及Compose版本过低会导致Harbor安装失败或运行异常，建议先升级至官方推荐版本。

---

## 部署规划细节

明确Harbor服务的网络地址和存储路径，有助于后续运维和管理。

| 配置项      | 具体规划                                 |
|----------|--------------------------------------|
| 服务器IP    | 10.0.0.38                            |
| Harbor端口 | 8004（默认HTTP端口80改为8004以避免冲突）          |
| 安装目录     | /opt/module/harbor/harbor-2.9.1      |
| 数据目录     | /opt/module/harbor/harbor-2.9.1/data |
| 日志目录     | /opt/module/harbor/harbor-2.9.1/logs |
| 管理员密码    | 自定义密码，比如12345678                     |

> **说明**：
> - 本次部署处于公司内网环境，无需开启HTTPS，HTTP协议即可使用，但端口应自定义，防止80/443端口冲突。
> - 推荐将数据和日志目录统一放置于安装目录下，方便统一备份和管理。

---

## 离线安装包准备

在无公网或受限网络环境下，离线安装是保证服务顺利部署的必要手段。

### 1. Harbor安装包下载

从Harbor官方GitHub Release页面下载对应版本的离线安装包：

```bash
链接：https://github.com/goharbor/harbor/releases/tag/v2.9.1
文件名：harbor-offline-installer-v2.9.1.tgz
```

### 2. 生成goharbor_prepare镜像包

prepare镜像用于Harbor初始化及数据迁移等预处理，需预先保存为tar包上传服务器：

```bash
docker pull goharbor/prepare:v2.9.1
docker save -o goharbor_prepare_v2.9.1.tar goharbor/prepare:v2.9.1
```

> 将`harbor-offline-installer-v2.9.1.tgz`及`goharbor_prepare_v2.9.1.tar`上传至目标服务器。

---

## 离线安装详细步骤

### 1. 解压及目录整理

```bash
tar -zxvf harbor-offline-installer-v2.9.1.tgz -C /opt/module/
mv /opt/module/harbor /opt/module/harbor-2.9.1
mkdir -p /opt/module/harbor
mv -f /opt/module/harbor-2.9.1 /opt/module/harbor
cd /opt/module/harbor/harbor-2.9.1
```

> 说明：此步骤旨在规范安装目录结构，保证数据和日志路径可控。

### 2. 配置Harbor主配置文件

复制模板，生成实际使用配置文件并修改关键字段：

```bash
cp harbor.yml.tmpl harbor.yml
```

#### 推荐修改项：

```yaml
hostname: 10.0.0.38        # 服务器IP地址
http:
  port: 8004              # 将默认80改为8004端口
harbor_admin_password: 12345678   # 管理员密码
data_volume: /opt/module/harbor/harbor-2.9.1/data   # 数据存储路径
log:
  local:
    location: /opt/module/harbor/harbor-2.9.1/logs  # 日志存储路径
```

> 注意：确认`hostname`和端口配置保持一致，防止访问失败。

### 3. 加载prepare镜像

```bash
docker load -i goharbor_prepare_v2.9.1.tar
```

### 4. 执行预处理脚本

预处理完成环境的基础配置等：

```bash
./prepare
```

### 5. 运行安装脚本

启动Harbor各组件并完成系统初始化：

```bash
./install.sh
```

安装过程将拉取并启动Harbor核心容器服务。

### 6. 创建 Harbor 系统d服务

方便服务托管和自动启动，创建`harbor.service`：

```bash
vim /lib/systemd/system/harbor.service
```

寫入内容：

```ini
[Unit]
Description=Harbor
Documentation=http://github.com/vmware/harbor
After=docker.service systemd-networkd.service systemd-resolved.service
Requires=docker.service

[Service]
Type=simple
Restart=on-failure
RestartSec=5
ExecStart=/usr/local/bin/docker-compose -f /opt/module/harbor/harbor-2.9.1/docker-compose.yml up
ExecStop=/usr/local/bin/docker-compose -f /opt/module/harbor/harbor-2.9.1/docker-compose.yml down

[Install]
WantedBy=multi-user.target
```

### 7. 配置Docker守护进程镜像仓库源

修改Docker的`daemon.json`以添加私有仓库地址，允许非安全仓库访问：

```bash
vim /etc/docker/daemon.json
```

示范配置：

```json
{
  "registry-mirrors": [
    "https://ppc7nwnq.mirror.aliyuncs.com"
  ],
  "insecure-registries": [
    "10.0.0.38:8004"
  ]
}
```

> 作用：加速Docker镜像下载，同时允许访问私有Harbor仓库。

### 8. 重启Docker及Harbor

```bash
systemctl daemon-reload
systemctl restart docker
systemctl status docker

systemctl start harbor
```

### 9. 验证Harbor服务状态

查看Harbor相关容器是否正常启动：

```bash
docker ps
```

正常应该看到包含 `nginx-photon`, `harbor-core`, `harbor-db`, `registry`, `redis` 等服务运行中，且状态为healthy。

### 10. 访问Harbor Web界面

在浏览器访问：

```
http://10.0.0.38:8004
```

出现登录界面后，输入管理员账号密码（默认`admin/12345678`或你自定义的密码）即可登录。

---

## Harbor使用示例：推送镜像到私有仓库

1. 登录Harbor私有仓库：

    ```bash
    docker login 10.0.0.38:8004
    ```

2. 拉取公有镜像示例：

    ```bash
    docker pull mysql:latest
    ```

3. 打标签，标记为私有仓库镜像：

    ```bash
    docker tag mysql:latest 10.0.0.38:8004/library/mysql:latest
    ```

4. 推送镜像到Harbor：

    ```bash
    docker push 10.0.0.38:8004/library/mysql:latest
    ```

成功后，私有仓库中即可看到该镜像版本。

---

## 总结及最佳实践建议

- **离线安装优势**：适合网络受限环境，避免因网络波动导致安装失败。
- **端口自定义**：企业内部若有端口冲突，灵活定制Harbor监听端口，降低风险。
- **数据和日志目录统一管理**：方便监控、备份和故障排查。
- **系统服务化管理**：借助systemd管理Harbor，确保服务自启动和稳定。
- **Docker配置同步**：在客户端及服务器统一配置私有镜像源。

通过上述步骤，您能在公司内网环境中快速、安全搭建完整的Harbor 2.9.1私有镜像仓库，为企业DevOps和容器化部署提供坚实基础。

---

> **更多资源与文档**
> - [Harbor 官方文档](https://goharbor.io/docs/)
> - [Harbor GitHub仓库](https://github.com/goharbor/harbor)