---
title: 修改KubeSphere外网访问端口
tags:
  - Linux
  - k8s
categories:
  - 容器化
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/20250425103016448.png'
toc: true
abbrlink: 22e6c961
date: 2024-03-28 10:29:27
---

KubeSphere 作为一款强大的容器管理平台，其默认的外网访问端口是 30880。在实际企业网络环境中，出于安全策略或网络规划的考虑，用户可能需要将该端口调整为其它符合公司规范的端口号，以确保访问的灵活性和合规性。本文将详细介绍如何修改 KubeSphere 的外网访问端口，帮助您高效便捷地完成定制化配置。

<!-- more -->

## 了解 NodePort 端口范围的重要性

KubeSphere 的外网访问通常是通过 Kubernetes 的 NodePort 类型服务实现的。NodePort 允许将集群内部服务暴露在指定的端口上，默认范围通常是 30000 到 32767。若要使用非默认端口，则需要先确保 Kubernetes 集群允许该端口范围，否则服务暴露可能会失败或不被访问。

因此，修改访问端口前，需要先确认或调整 Kubernetes 的 NodePort 端口范围，使新端口被系统支持，以保证后续配置有效。

### 调整 NodePort 端口范围示例

请参考对应 Kubernetes 版本的官方文档，或参考我的完整教程：[k8s 修改默认 NodePort 端口范围 30000-32767](https://juejin.cn/spost/7351321232748675109)，通过修改 API Server 的启动参数 `--service-node-port-range` 来实现端口范围的扩展。

示例：

```bash
--service-node-port-range=20000-40000
```

设置后，重启 Kubernetes 控制平面组件，等待配置生效。

## 修改 KubeSphere 控制台的 Service 配置

调整完 NodePort 范围后，接下来修改 KubeSphere 控制台暴露的 NodePort 服务，以使用新的端口。

执行以下命令进入编辑状态：

```bash
kubectl edit svc ks-console -n kubesphere-system
```

找到 `nodePort` 字段，将值从默认的 `30880` 修改为符合新端口范围的端口号。例如：

```yaml
ports:
  - name: https
    port: 30880
    protocol: TCP
    targetPort: 30880
    nodePort: 32000  # 修改为自定义端口
```

保存退出后，Kubernetes 会自动应用变更。

## 端口变更后验证步骤

变更完毕后，请务必确认新端口的网络连通性：

- 通过浏览器访问 `http://<Node-IP>:<自定义端口>`，确认能正确加载 KubeSphere 控制台页面。
- 若遇访问异常，检查防火墙和集群网络策略，确保自定义端口已允许通过。
- 使用 `kubectl get svc ks-console -n kubesphere-system` 查看端口配置是否正确。

## 总结

通过上述步骤，您可以灵活调整 KubeSphere 的外网访问端口，满足企业不同网络环境的需求。重点在于确保 Kubernetes NodePort 端口范围涵盖了您期望使用的端口，从而保障服务的稳定暴露。此外，合理配置网络安全策略，配合端口调整能有效提升访问安全性和可控性。

借助这些技巧，您可以让 KubeSphere 更加适配业务场景，提升容器管理体验。

---

欢迎大家留言交流更多 Kubernetes 和 KubeSphere 相关使用技巧！