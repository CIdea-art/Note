# 告警

## 规则

默认告警规则

- 最近3分钟内服务的平均响应时间超过1秒
- 最近2分钟服务成功率低于80%
- 最近3分钟90%服务响应时间超过1秒
- 最近2分钟内服务实例的平均响应时间超过1秒

其它规则配置在`config/alarm-settings.yml`中

```yaml
rules:
  # Rule unique name, must be ended with `_rule`.
  service_resp_time_rule:
    metrics-name: service_resp_time
    op: ">"
    threshold: 1000
    period: 10
    count: 3
    silence-period: 5
    message: Response time of service {name} is more than 1000ms in 3 minutes of last 10 minutes.
  service_sla_rule:
  	...
  service_resp_time_percentile_rule:
  	...
  service_instance_resp_time_rule:
  	...
  database_access_resp_time_rule:
  	...
  endpoint_relation_resp_time_rule:
  	...
```

## 通知

除了以上默认的几种规则，skywalking还适配了一些钩子（**webhooks**）。其实就是相当于一个回调，一旦触发了上述规则告警，skywalking则会调用配置的webhook，这样开发者就可以定制一些处理方法，比如发送**邮件**、**微信**、**钉钉**通知运维人员处理。

当然这个钩子也是有些规则的，如下：

- POST请求
- **application/json** 接收数据
- 接收的参数必须是AlarmMessage中指定的参数。

> org.apache.skywalking.oap.server.core.alarm.AlarmMessage

# 参考文献

[芋道 Spring Boot 链路追踪 SkyWalking 入门 | 芋道源码 —— 纯源码解析博客 (iocoder.cn)](https://www.iocoder.cn/Spring-Boot/SkyWalking/?github)

[GitHub - SkyAPM/document-cn-translation-of-skywalking: The CN translation version of Apache SkyWalking document](https://github.com/SkyAPM/document-cn-translation-of-skywalking)