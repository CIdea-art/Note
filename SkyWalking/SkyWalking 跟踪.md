# 跟踪

在使用 SkyWalking 排查问题的时候，我们可能希望能够跟链路的日志进行关联，那么我们可以将链路编号( SkyWalking TraceId )记录到日志中，从而进行关联。

引入依赖

```xml
<!-- SkyWalking 对 Logback 的集成 -->
<dependency>
    <groupId>org.apache.skywalking</groupId>
    <artifactId>apm-toolkit-logback-1.x</artifactId>
    <version>6.6.0</version>
</dependency>
```

`logback-spring.xml`打印格式中增加链路ID`%tid`。

并使用替换`TraceIdPatternLogbackLayout`将`%tid`替换成SkyWalking TraceId

```xml
<property name="FILE_LOG_PATTERN" value="%d{${LOG_DATEFORMAT_PATTERN:-yyyy-MM-dd HH:mm:ss.SSS}} ${LOG_LEVEL_PATTERN:-%5p} ${PID:- } %tid --- [%t] %-40.40logger{39} : %m%n${LOG_EXCEPTION_CONVERSION_WORD:-%wEx}"/>
<appender name="file" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>${LOG_FILE}</file>
    <!--滚动策略，基于时间 + 大小的分包策略 -->
    <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
        <fileNamePattern>${LOG_FILE}.%d{yyyy-MM-dd}.%i.gz</fileNamePattern>
        <maxHistory>7</maxHistory>
        <maxFileSize>10MB</maxFileSize>
    </rollingPolicy>
    <!-- 日志的格式化 -->
    <encoder class="ch.qos.logback.core.encoder.LayoutWrappingEncoder">
        <layout class="org.apache.skywalking.apm.toolkit.log.logback.v1.x.TraceIdPatternLogbackLayout">
            <Pattern>${FILE_LOG_PATTERN}</Pattern>
        </layout>
    </encoder>
</appender>
```

## 定制化

### `@Trace`

实现 SkyWalking 指定方法的追踪，会创建一个 `SkyWalking LocalSpan`。同时，可以通过 `operationName` 属性，设置操作名。

引入

```xml
<!-- SkyWalking 工具类 -->
<dependency>
    <groupId>org.apache.skywalking</groupId>
    <artifactId>apm-toolkit-trace</artifactId>
    <version>6.6.0</version>
</dependency>
```



通过 `ActiveSpan#tag(String key, String value)` 方法，设置该 LocalSpan 的标签。

### OpenTracing

手动创建Span。

引入

```xml
<!-- SkyWalking Opentracing 集成 -->
<dependency>
    <groupId>org.apache.skywalking</groupId>
    <artifactId>apm-toolkit-opentracing</artifactId>
    <version>6.6.0</version>
</dependency>
```

代码

```java
Tracer tracer = new SkywalkingTracer();
tracer.buildSpan("custom_operation").withTag("mp", "芋道源码").startManual().finish();
```



# 参考文献

[芋道 Spring Boot 链路追踪 SkyWalking 入门 | 芋道源码 —— 纯源码解析博客 (iocoder.cn)](https://www.iocoder.cn/Spring-Boot/SkyWalking/?github)
