



# 简介

应用性能监控平台，可用于分布式系统，支持微服务、云原生、Docker、Kubernetes 等多种架构场景。

## 功能

- 多种监控手段：通过语言探针和 Service Mesh 等手段，获得链路、日志、指标等监控数据
- 多个语言探针：Java、.Net Core、PHP、NodeJS、Golang、LUA、Rust、C++ 等
- 轻量级高性能：无需大数据组件，无需大量的硬件资源，且对应用实例的负载消耗极低
- 模块化架构：数据传输、数据存储，注册发现等模块，可替换不同的基础设施实现
- 端到端的监控：Vue、React 等前端，Java、.Net Core、PHP、NodeJS、Golang、Istio 等后端
- 告警机制：内置 Webhooks 发送事件通知，支持通过 HTTP、gRPC、Slack 等方式
- 可视化界面：好用的监控后台，可支持自定义配置，或是集成你自己的

## 架构

![整体架构](resources\整体架构.png)

- 【左】**Agent**：代理采集点。在应用中，收集 Trace、Log、Metrics 等监控数据，使用 RPC、RESTful API、Kafka 等 Transport 传输方式，发送给 OAP 服务
- 【下】**OAP**：观测分析平台(Observability Analysis Platform)。首先 Receiver 接收 Agent 发送的监控数据，然后 Aggregator 进行聚合计算，之后存储到 Storage 外部存储器，最终提供给 GUI 查询数据
- 【右】**Storage**：存储监控数据，支持 Elasticsearch、MySQL、TiDB、H2 等多种数据库
- 【上】**GUI**：UI 可视化界面，提供监控数据的查询后台



# 参考文献

[芋道 SkyWalking 9.X 极简入门（新版本） | 芋道源码 —— 纯源码解析博客 (iocoder.cn)](https://www.iocoder.cn/SkyWalking/install/?self)

[GitHub - SkyAPM/document-cn-translation-of-skywalking: The CN translation version of Apache SkyWalking document](https://github.com/SkyAPM/document-cn-translation-of-skywalking)