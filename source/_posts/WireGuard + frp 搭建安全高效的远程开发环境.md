---
title: WireGuard + frp 搭建安全高效的远程开发环境
tags:
  - Linux
  - WireGuard
  - Frp
  - Vpn
categories:
  - 运维手册
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202505042344955.png'
toc: true
abbrlink: ef27da93
date: 2025-05-05 01:24:11
---

作为一名开发者，你是否经常遇到这样的痛点：家里的开发测试环境（如虚拟机集群、NAS）资源丰富，但一旦离开家（比如去公司或出差），就无法访问这些宝贵的内网资源？本文将详细介绍如何利用现代 VPN 解决方案 WireGuard 和内网穿透工具 frp，搭建一个安全、高效、稳定的远程访问通道，让你随时随地都能连接到家庭或办公室的内部网络，如同身处局域网之中。

<!-- more -->

## 背景痛点

在家中搭建了强大的开发环境（例如，运行多个 Linux 虚拟机的台式主机）是非常常见的做法，这为本地开发和测试提供了极大的便利。然而，物理位置的限制往往带来不便——离开家就意味着与这些内部资源断开连接。传统的解决方案可能涉及复杂的端口转发、动态 DNS 或安全性较差的 PPTP/L2TP VPN。

现代化的 WireGuard 以其简洁、高效和安全著称，成为了构建虚拟专用网络的优秀选择。但如果你的家庭网络没有公网 IP，或者公网 IP 是动态变化的，直接从外部访问 WireGuard 服务端就变得困难。这时，内网穿透工具 frp 就派上了用场。frp 可以将内网服务安全地暴露到公网服务器上，从而打通访问的最后一道屏障。

本教程将引导你结合 WireGuard（通过易于使用的 `wg-easy` 界面）和 frp，实现以下目标：

1. 在内网的 Linux 服务器上部署 WireGuard 服务。
2. 利用具有公网 IP 的云服务器和 frp，将 WireGuard 服务端口安全地暴露到公网。
3. 在外部设备（如 Windows 笔记本）上配置 WireGuard 客户端，实现对内网服务器及其所在局域网资源的访问。

## 环境准备

你需要准备以下环境：

1. **一台拥有公网 IP 的云服务器 (VPS):** 用于运行 frp 服务端 (frps)。必须确保防火墙允许 frp 所需的端口通信。
2. **一台内网 Linux 服务器:** 作为 WireGuard 服务端，也是 frp 客户端 (frpc) 的运行载体。这台服务器需要能够访问你想远程连接的其他内网设备。
3. **一台需要远程访问的客户端设备 (如 Windows/macOS/Linux):** 用于安装 WireGuard 客户端软件。

**示例环境说明:**

| 主机名     | 角色             | 公网 IP           | 内网 IP           | 操作系统    | 说明                                 |
|:--------|:---------------|:----------------|:----------------|:--------|:-----------------------------------|
| vmiss   | frp 服务端 (frps) | `38.47.108.108` | (通常有内网 IP)      | Linux   | 公网 VPS，提供 frp 中转服务。 **需要配置防火墙**。   |
| test    | Wg 服务端 + frpc  | 无               | `192.168.6.201` | Linux   | 家庭/办公室 内网服务器，运行 wg-easy 和 frpc。    |
| windows | Wg 客户端         | (动态/无)          | (动态/无)          | Windows | 需要访问内网资源的远程设备。                     |
| 其他内网设备  | (被访问)          | 无               | `192.168.6.x`   | (任意)    | 与 `test` 服务器同网段的其他设备，如其他虚拟机、NAS 等。 |

*(请将示例中的 IP 地址和主机名替换为你自己的实际配置)*

## 搭建思路与架构

核心思路是利用 frp 打破 NAT 限制，将内网的 WireGuard 服务端口映射到公网 VPS 上，外部客户端连接公网 VPS 的映射端口，frp 会将流量安全地转发给内网的 WireGuard 服务器。

**数据流:**

`WireGuard 客户端 (Windows)` -> `公网 (Internet)` -> `公网 VPS (frps + 防火墙)` -> `frp 隧道` -> `内网 Linux (frpc)` ->
`内网 Linux (wg-easy)` -> `目标内网设备 (192.168.6.x)`

## 搭建步骤

### 第一步：在内网 Linux (test) 安装并配置 WireGuard 服务端 (wg-easy)

`wg-easy` 是一个提供 Web UI 来管理 WireGuard 配置的 Docker 应用，极大地简化了 WireGuard 的部署和客户端管理。

#### 1. 检查 Linux 内核版本

WireGuard 需要较新的 Linux 内核支持。通常要求 **内核版本 >= 5.6**。

```bash
uname -r
```

如果版本低于 5.6，你需要升级内核。可以参考你的 Linux 发行版官方文档或相关教程进行升级，例如：[Rocky Linux 8.10 内核升级实战：拥抱 Linux 6.1 LTS](https://lbs.wiki/pages/fced2319/)。

#### 2. 确保 Docker 网络依赖的内核模块已加载

Docker（尤其是其网络功能）通常依赖 `ip_tables` 和 `iptable_nat` 模块。

```bash
lsmod | grep ip_tables
lsmod | grep iptable_nat
```

如果这两条命令都有输出，则表示模块已加载。如果没有，需要手动加载。具体方法请参考：[解决 Docker 容器网络问题：加载 ip_tables 和 iptable_nat 内核模块](https://lbs.wiki/pages/a92b80/)。通常可以通过 `modprobe ip_tables` 和 `modprobe iptable_nat` 来加载，并配置开机自动加载。

#### 3. 安装 Docker 和 Docker Compose

如果你的 `test` 服务器上尚未安装 Docker 和 Docker Compose，请先安装。

* **Docker 安装:** 参考 Docker
  官方文档或 [CentOS系统指定版本Docker与Docker Compose在线安装教程](https://lbs.wiki/pages/987e17e9/)。
* **Docker Compose 安装:** 参考 Docker 官方文档或上述教程。

#### 4. 配置并启动 wg-easy

创建一个名为 `docker-compose.yml` 的文件，内容如下：

```yaml
version: '3.8' # 建议指定版本

volumes:
  etc_wireguard: # 持久化存储 WireGuard 配置

services:
  wg-easy:
    image: ghcr.io/wg-easy/wg-easy:latest # 使用 latest 或指定稳定版本，如 :14
    container_name: wg-easy
    environment:
      # --- 核心配置 ---
      # ！！！重要：这里填写你的 frp 公网服务器的 IP 地址 ！！！
      # 这是客户端配置文件中 Endpoint 的地址
      - WG_HOST=38.47.108.108

      # ！！！重要：wg-easy Web UI 的管理员密码哈希 ！！！
      # 强烈建议修改默认密码 'foobar123'
      # 可以使用 `docker run --rm ghcr.io/wg-easy/wg-easy:latest --pw your_strong_password` 生成哈希
      # 或者使用在线 bcrypt 生成器 (注意 rounds=5, 示例为 'foobar123' 的哈希)
      - PASSWORD_HASH=$2y$05$6n2x0AdkZ5lJ8kHufbXrr.dU.UHMBh04HKYXoPb0w484wN75877Eu # (需要双 $$ 符号)

      # --- 网络配置 ---
      # WireGuard 服务监听的 UDP 端口 (容器内)
      - WG_PORT=51820
      # wg-easy Web UI 监听的 TCP 端口 (容器内)
      - PORT=51821

      # VPN 客户端分配的 IP 地址范围 (选择一个未使用的私有网段)
      - WG_DEFAULT_ADDRESS=10.8.0.x # 客户端将获取 10.8.0.2, 10.8.0.3 ...
      # 客户端使用的 DNS 服务器 (可使用公共 DNS 或内网 DNS)
      - WG_DEFAULT_DNS=114.114.114.114, 8.8.8.8

      # ！！！重要：客户端允许访问的 IP 范围 ！！！
      # 决定了哪些流量会通过 VPN 隧道
      # 10.8.0.0/24: 允许客户端之间互相访问 (如果需要)
      # 192.168.6.0/24: 允许客户端访问 test 服务器所在的局域网
      - WG_ALLOWED_IPS=10.8.0.0/24, 192.168.6.0/24

      # --- 可选配置 ---
      # 保持连接，防止 NAT 超时断开 (单位: 秒)
      - WG_PERSISTENT_KEEPALIVE=25
      # Web UI 语言 (支持 en, chs 등)
      - LANG=chs
      # 启用/禁用 Web UI 中的流量统计图表 (true/false)
      # - UI_TRAFFIC_STATS=true
      # 在 Web UI 中允许通过 HTTP 访问，如果前面没有反向代理做HTTPS，则需要设置为 true
      # 生产环境强烈建议配合反向代理使用 HTTPS
      - INSECURE=true


    volumes:
      # 将 WireGuard 配置持久化到宿主机
      - etc_wireguard:/etc/wireguard
    ports:
      # 将宿主机的端口映射到容器端口
      # 格式: "宿主机端口:容器端口/协议"
      - "51820:51820/udp" # WireGuard 数据端口 (与 WG_PORT 一致)
      - "51821:51821/tcp" # wg-easy Web UI 端口 (与 PORT 一致)
    restart: unless-stopped
    cap_add:
      # 赋予容器必要的网络管理权限
      - NET_ADMIN
      - SYS_MODULE
    sysctls:
      # 启用 IP 转发，允许 VPN 客户端通过此服务器访问其他网络
      - net.ipv4.ip_forward=1
      # 确保 WireGuard 的网络标记正常工作
      - net.ipv4.conf.all.src_valid_mark=1
      # - net.ipv6.conf.all.forwarding=1 # 如果需要 IPv6 支持，取消注释
```

**关键配置说明:**

* `WG_HOST`: **必须**填写你的**公网 VPS** 的 IP 地址。这是最终客户端连接的目标地址。
* `PASSWORD_HASH`: **必须**修改默认密码，并生成新的 bcrypt 哈希值。这是保护 `wg-easy` Web 界面的关键。
* `WG_ALLOWED_IPS`: 这个设置非常重要。它定义了当客户端连接 VPN 后，哪些目标 IP 地址的流量会**通过 VPN 隧道**传输。
    * `10.8.0.0/24` (与 `WG_DEFAULT_ADDRESS` 对应的网段) 允许 VPN 客户端之间相互通信（如果需要）。
    * `192.168.6.0/24` (你的内网 `test` 服务器所在的局域网段) 允许 VPN 客户端访问该局域网内的所有设备。请务必修改为你*
      *实际的内网网段**。
* `ports`: 确保这里映射的宿主机端口 (`51820/udp`, `51821/tcp`) 没有被其他程序占用。
* `sysctls`: `net.ipv4.ip_forward=1` 必须启用，否则客户端无法通过 VPN 服务器访问其他网络。

**启动服务:**

在 `docker-compose.yml` 文件所在的目录下执行：

```bash
docker-compose up -d
```

**查看日志:**

```bash
docker-compose logs -f wg-easy
# 或者
docker logs wg-easy -f
```

现在，你应该可以通过 `http://<test服务器内网IP>:51821` (例如 `http://192.168.6.201:51821`) 在浏览器中访问 `wg-easy` 的 Web UI 了。使用你设置的密码登录（如果你用了示例哈希，密码是 `foobar123`）。

### 第二步：使用 frp 进行内网穿透，暴露 WireGuard 端口

> 关于frp内网穿透的详细教程请参考：[frp内网穿透](https://lbs.wiki/pages/b2dafbdf/)，这里不再赘述，只是简单给出配置文件。

云服务器上的 `frps` 服务器配置文件示例：

```toml
# frps.toml (示例)
bindAddr = "0.0.0.0" # 监听所有网络接口
bindPort = 47000     # frp 客户端连接服务端的端口，可自定义，防火墙需放行

# 认证配置，增强安全性
auth.method = "token"
auth.token = "YourSecretToken_ChangeMe!" # 设置一个强密码，客户端需要使用相同的 Token

# Web 控制台（可选，方便查看状态）
webServer.addr = "0.0.0.0"
webServer.port = 47500      # Web 控制台端口，可自定义，防火墙需放行
webServer.user = "admin"
webServer.password = "AdminPassword_ChangeMe!" # 控制台登录密码

# 日志配置（可选）
log.to = "./frps.log" # 日志文件路径
log.level = "info"    # 日志级别
log.maxDays = 7       # 日志保留天数
```

内网linux服务器上的 `frpc` 客户端配置文件示例：

```toml
# frpc.toml (示例)
serverAddr = "38.47.108.108" # frps 服务器的公网 IP 地址
serverPort = 47000           # 与 frps.toml 中的 bindPort 一致

# 认证配置，必须与 frps.toml 中的设置完全一致
auth.method = "token"
auth.token = "YourSecretToken_ChangeMe!" # 使用与 frps 相同的 Token

# 日志配置（可选）
log.to = "./frpc.log"
log.level = "info"
log.maxDays = 7

# --- 定义需要穿透的代理 ---

# 代理1: 穿透 WireGuard 的 UDP 数据端口
[[proxies]]
name = "wg-data"          # 代理名称，自定义
type = "udp"              # 协议类型必须是 UDP
localIP = "127.0.0.1"     # wg-easy 运行在本机
localPort = 51820         # wg-easy 监听的 UDP 端口 (与 docker-compose ports 一致)
remotePort = 51820        # 暴露在公网 VPS 上的 UDP 端口 (防火墙需放行)

# 代理2: 穿透 wg-easy 的 TCP Web UI 端口
[[proxies]]
name = "wg-ui"            # 代理名称，自定义
type = "tcp"              # 协议类型必须是 TCP
localIP = "127.0.0.1"     # wg-easy 运行在本机
localPort = 51821         # wg-easy 监听的 TCP 端口 (与 docker-compose ports 一致)
remotePort = 51821        # 暴露在公网 VPS 上的 TCP 端口 (防火墙需放行)
```

> 注意要在云服务器和内网linux服务器上开放内网穿透的对应端口.

### 第三步：在 Windows 客户端安装和配置 WireGuard

#### 1. 下载并安装 WireGuard 客户端

访问 [WireGuard 官方安装页面](https://www.wireguard.com/install/)，下载并安装适用于 Windows 的客户端。

#### 2. 获取 WireGuard 客户端配置文件

* 通过 **frp 穿透后的地址** 访问 `wg-easy` Web UI：`http://<公网VPS_IP>:51821` (例如 `http://38.47.108.108:51821`)。
* 登录后，点击 "New Client" 创建一个新的客户端连接，给它起个名字（例如 `MyLaptop`)。
* 创建成功后，找到该客户端条目，点击 "Download" 按钮，下载对应的 `.conf` 配置文件。

  [![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202505050104611.png)](https://www.wireguard.com/install/)
  ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202505050105112.png)

#### 3. 导入配置文件到 WireGuard 客户端

* 打开 Windows 上的 WireGuard 应用程序。
* 点击左下角的 "Import tunnel(s) from file" 按钮。
* 选择刚刚下载的 `.conf` 文件并导入。

  ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202505050107896.png)

#### 4. 连接 VPN

* 在 WireGuard 客户端界面中，找到刚才导入的隧道配置。
* 点击 "Activate" 按钮。

  ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202505050107712.png)

* 如果连接成功，状态会变为 "Active"，并显示接收和发送的数据量。日志中应该能看到握手成功的信息。

## 第四步：验证连接

当 WireGuard 客户端显示 "Active" 后，表示你已经成功连接到家里的内网环境了。现在，你应该能够从你的 Windows 客户端访问 `test`
服务器所在局域网 (`192.168.6.0/24`) 中的设备了。

**验证方法:**

1. **Ping 测试:** 打开 Windows 的命令提示符 (cmd) 或 PowerShell，尝试 ping `test` 服务器的内网 IP 或该网段的其他设备 IP。
   ```bash
   ping 192.168.6.201 # Ping wg-easy 服务器
   ping 192.168.6.100 # Ping 局域网内的另一台虚拟机或设备
   ```
   如果能收到回复，说明网络层是通的。

2. **访问内网服务:** 尝试访问部署在内网设备上的 Web 服务、SSH、文件共享等。
    * 例如，访问 `test` 服务器或其他内网机器上运行的 Portainer、NAS 管理界面、网站等，**直接使用它们的内网 IP 地址**。
      ![](https://lbs-images.oss-cn-shanghai.aliyuncs.com/202505050110714.png) *(这张图展示了通过内网 IP 访问
      Portainer，验证成功)*
    * 尝试 SSH 连接到内网的 Linux 服务器：
      ```bash
      ssh user@192.168.6.201
      ```

如果以上验证都成功，恭喜你！你已经成功搭建了通过 WireGuard 和 frp 的远程访问通道。

## 安全性考量

* **防火墙:** 务必在公网 VPS (frps) 上配置严格的防火墙规则，仅允许必要的端口（frp 服务端口、映射的 WireGuard 端口、frp Web
  控制台端口等）访问。
* **frp Token:** 使用复杂度高的 `auth.token`，并确保 frps 和 frpc 配置一致。
* **wg-easy 密码:** 使用强密码保护 `wg-easy` Web UI，并妥善保管 `PASSWORD_HASH`。考虑为 Web UI 配置 HTTPS（例如通过 Nginx
  反向代理）。
* **WireGuard 密钥:** WireGuard 本身使用现代加密技术，密钥对的安全性至关重要。`wg-easy` 会自动管理这些密钥。
* **AllowedIPs:** 在 `wg-easy` (服务器端) 和客户端配置文件中，精确配置 `AllowedIPs`，遵循最小权限原则，仅允许访问必要的网络范围。避免使用过于宽泛的
  `0.0.0.0/0`，除非你确实希望所有流量都通过 VPN。
* **软件更新:** 及时更新 WireGuard、frp、Docker 以及服务器操作系统，以修复潜在的安全漏洞。

## 故障排查

如果遇到连接问题，可以按以下步骤排查：

1. **检查 WireGuard 客户端日志:** 客户端界面通常会显示最后的握手时间和错误信息。
2. **检查 `wg-easy` 容器日志:** `docker logs wg-easy -f` 查看是否有错误或连接信息。
3. **检查 `frpc` 日志:** 在 `test` 服务器上查看 `frpc.log`，确认是否成功连接到 frps，代理是否正常。
4. **检查 `frps` 日志:** 在公网 VPS 上查看 `frps.log`，确认 frpc 是否连接，是否有错误信息。
5. **检查公网 VPS 防火墙:** 再次确认所有需要的端口（frps 的 `bindPort`，frpc 的 `remotePort` UDP/TCP）都已经正确放行。
6. **检查内网 `test` 服务器防火墙:** 如果 `test` 服务器本身开启了防火墙 (如 `firewalld` 或 `iptables`)，确保允许来自
   Docker 容器 (`wg-easy`) 的流量以及 `frpc` 的出站连接。特别注意 `net.ipv4.ip_forward=1` 是否生效 (
   `sysctl net.ipv4.ip_forward`)。
7. **网络连通性测试:**
    * 在 `test` 服务器上 `ping <公网VPS_IP>` 确认能访问 frps 服务器。
    * 在 `test` 服务器上使用 `telnet <公网VPS_IP> <frps_bindPort>` (如 `telnet 38.47.108.108 47000`) 确认 TCP 端口可达。
    * 从**外部网络**（非家庭网络）尝试 `telnet <公网VPS_IP> <wg_ui_remotePort>` (如 `telnet 38.47.108.108 51821`) 确认 frp
      TCP 代理端口可访问。UDP 端口测试相对麻烦，可以用 `nc -vzu <公网VPS_IP> <wg_data_remotePort>` (如
      `nc -vzu 38.47.108.108 51820`) 尝试。

## 结语

通过结合 WireGuard 的高效安全与 frp 的强大穿透能力，我们成功构建了一个稳定可靠的远程访问方案。这不仅解决了开发者远程访问内网资源的痛点，也为需要安全连接不同网络环境的场景提供了一个优秀的实践范例。`wg-easy` 的使用大大降低了 WireGuard 的配置复杂度，使得整个过程更加友好。

希望本教程能帮助你打通地域限制，随时随地高效地利用你的内网开发环境。如有任何疑问或建议，欢迎在评论区交流！
