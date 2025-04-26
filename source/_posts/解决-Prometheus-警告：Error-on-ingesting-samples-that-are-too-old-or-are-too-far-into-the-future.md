---
title: >-
  解决 Prometheus 警告：Error on ingesting samples that are too old or are too far
  into the future
tags:
  - Linux
  - Prometheus
categories:
  - 运维手册
cover: 'https://lbs-images.oss-cn-shanghai.aliyuncs.com/202504262328256.png'
toc: true
abbrlink: 2b961351
date: 2025-02-12 23:27:00
---

在使用 Prometheus 进行监控时，可能会遇到如下警告日志：

```
level=warn ts=2021-08-16T03:20:04.960Z caller=scrape.go:1507 component="scrape manager" scrape_pool=mtail target=http://18.167.146.20:3903/metrics msg="Error on ingesting samples that are too old or are too far into the future" num_dropped=78
level=warn ts=2021-08-16T03:20:04.961Z caller=scrape.go:1203 component="scrape manager" scrape_pool=mtail target=http://18.167.146.20:3903/metrics msg="Appending scrape report failed" err="out of bounds"
```

类似的日志经常反复出现，报错内容包括“samples that are too old”或“too far into the future”，且伴随 "err=out of bounds"。本文将分析该问题的原因，并给出实用的解决方案。

<!-- more -->

## 问题原因分析

该错误的产生是因为 Prometheus 在抓取（scrape）目标指标时，收到的指标数据时间戳不符合期望的时间范围，具体表现在以下两个方面：

- **时间戳过旧**：Prometheus 默认不接受比当前时间点早超过一定阈值（一般是 1 个采集周期）过多的样本数据。
- **时间戳过新（未来时间）**：样本的时间戳可能比 Prometheus 当前系统时间还未来，通常是因为被抓取的目标时间不同步或时钟漂移。

这种时间上的异常导致 Prometheus 拒绝写入该样本，触发 “Error on ingesting samples that are too old or are too far into the future” 的警告。

典型的原因包括：

- 被监控目标机器时间设置不正确，与 Prometheus 服务端时间不匹配。
- 目标指标导出程序（exporter）采集数据时存在时间戳异常。
- 网络延迟或重试造成指标的时间戳异常。
- Prometheus 服务端系统时间问题。

## 影响

这种问题会导致部分指标数据无法入库，影响监控数据的完整性，长期积累还可能影响告警的准确性。

## 解决方案

针对该问题，推荐如下处理步骤：

### 校准系统时间

确保 Prometheus 服务端与所有被抓取的监控目标机器的时间同步一致，建议统一使用 NTP（Network Time Protocol）进行时间同步，避免时钟漂移。

### 检查监控导出程序

确认各个 Exporter 版本是否兼容 Prometheus，是否存在时间戳异常，必要时升级到稳定版本，或者排查其配置中有关时间同步的逻辑。

### 清理 Prometheus 数据存储

该问题有时和 Prometheus 本地存储中的异常旧数据相关，可以尝试：

- **备份现有 Prometheus 数据目录**，保障数据安全。
- 删除 Prometheus 当前的 storage 数据目录（一般是 `data` 文件夹）。
- 重启 Prometheus 服务，重新开始数据采集和存储。

> 如果 Prometheus 依赖外部数据库（如 InfluxDB）存储数据，数据不会丢失，清理本地存储影响有限。

### 调整 scrape 配置

- 减少 scrape 周期过大导致时间错乱。
- 避免配置重叠目标。

### 监控系统时间漂移

结合告警监控服务端和各监控目标的系统时间，确保长期稳定。

## 总结

Prometheus 日志中提示“Error on ingesting samples that are too old or are too far into the future”通常是由于时间戳异常导致样本被拒绝写入造成的。通过统一时间同步、升级和检查 exporter、清理本地数据目录及优化 scrape 配置，可以有效解决此问题，保障监控数据的准确和完整。

---

希望本文对遇到类似 Prometheus 报错的同学有所帮助。对于生产环境建议在非业务高峰期谨慎操作数据清理和服务重启，确保数据安全和监控稳定。