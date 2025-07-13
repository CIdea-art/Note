# 搭建

[下载](https://skywalking.apache.org/downloads/#SkyWalkingAPM)

## OAP、GUI

下载SkyWalking APM。

根据操作系统，启动oapService（OAP）和webappService（GUI）。

浏览器打开 [http://127.0.0.1:8080](http://127.0.0.1:8080/) 地址

## AGENTS

### Java

以`Java Spring`作为`Agent`示例。

下载`SkyWalking Java Agent`。并拷贝到 Java 应用所在的服务器上。

启动脚本

```bash
# SkyWalking Agent 配置
export SW_AGENT_NAME=demo-application # 配置 Agent 名字。一般来说，我们直接使用 Spring Boot 项目的 `spring.application.name` 。
export SW_AGENT_COLLECTOR_BACKEND_SERVICES=127.0.0.1:11800 # 配置 Collector 地址。
export SW_AGENT_SPAN_LIMIT=2000 # 配置链路的最大 Span 数量。一般情况下，不需要配置，默认为 300 。主要考虑，有些新上 SkyWalking Agent 的项目，代码可能比较糟糕。
export JAVA_AGENT=-javaagent:/Users/yunai/skywalking/skywalking-agent/skywalking-agent.jar # SkyWalking Agent jar 地址。

# Jar 启动
java -jar $JAVA_AGENT -jar lab-39-demo-2.2.2.RELEASE.jar
```



```bash
-javaagent:/Users/yunai/skywalking/skywalking-agent/skywalking-agent.jar
```

### MySql



## 集群化

不考虑Storage、Agents、注册中心的集群，这两个根据类型有各自的方案。

- `config/application.yml`中`cluster`配置，有zk、nacos等

- `Agent`启动设置`SW_AGENT_COLLECTOR_BACKEND_SERVICES`地址时，配置多个OAP服务地址数组
- 搭建一个 SkyWalking UI 服务的**集群**，同时使用 Nginx 进行负载均衡。另外，在设置 SkyWalking UI 的 `spring.cloud.discovery.client.simple.instances.oap-service` 地址时，也需要设置多个 SkyWalking OAP 服务的地址数组

# 参考文献

[芋道 SkyWalking 9.X 极简入门（新版本） | 芋道源码 —— 纯源码解析博客 (iocoder.cn)](https://www.iocoder.cn/SkyWalking/install/?self)

[GitHub - SkyAPM/document-cn-translation-of-skywalking: The CN translation version of Apache SkyWalking document](https://github.com/SkyAPM/document-cn-translation-of-skywalking)

