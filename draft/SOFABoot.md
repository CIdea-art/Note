# SOFABoot

## 简介

### 背景

- Spring Boot只有 Liveness Check （活跃性）的能力，缺少 Readiness Check  （健康），无法检测出服务的健壮性；

- Jar依赖冲突，且难以协调统一；

- 模块日志配置繁杂；

- 模块间共用Spring上下文产生额外的bean管理成本。

### 方案

- 只有在 Readiness Check 通过之后，才将流量引入到应用的实例中。http://localhost:8080/actuator/readiness；
- 中间件集成Starer；
- 基于 SOFAArk 提供了 Spring Boot 上的类隔离的能力；
- SOFABoot 提供了日志空间隔离的能力给 SOFA 中间件，SOFA 中间件中的各个组件采用日志空间隔离的能力之后，自动就会将本身的日志和应用的普通日志隔离开来，并且打印的日志的路径也是相对固定；
- SOFABoot 模块化，各模块隔离Spring上下文。

**？？？**后三项是否可使用微服务解决：冲突的模块细分成独立微服务。各微服务之间依赖、日志、Spring容器都能相互独立。

>
>两种方案有什么区别？
>
>- ？？？微服务过度细分会产生问题，增加维护、管理成本
>- SOFABoot Jar包即插即用



# SOFAArk 

使用不同的ClassLoad加载类，彻底解决包冲突的问题。

基于 SOFAArk 提供的类隔离能力，SOFAArk 支持将多个应用合并打成一个可执行的 Fat Jar 包，也支持运行时通过 API 或者 Zookeeper 动态推送配置达到动态部署应用(模块)的能力。

Ark 包是满足特定目录格式要求的 `Executed Fat Jar`，使用官方提供的 `Maven` 插件 `sofa-ark-maven-plugin`可以将工程应用打包成一个标准格式的 `Ark 包`；使用命令 `java -jar application.jar`即可在 Ark 容器之上启动应用；`Ark 包` 通常包含 `Ark Container`、`Ark Plugin`、 `Ark Biz`；以下我们针对这三个概念简单做下名词解释：

- `Ark Container`: Ark 容器，负责整个运行时的管理；`Ark Plugin` 和 `Ark Biz` 运行在 Ark 容器之上；容器具备管理多插件、多应用的功能；容器启动成功后，会自动解析 classpath 包含的 `Ark Plugin` 和 `Ark Biz` 依赖，完成隔离加载并按优先级依次启动之；
- `Ark Plugin`: Ark 插件，满足特定目录格式要求的 `Fat Jar`，使用官方提供的 `Maven` 插件 `sofa-ark-plugin-maven-plugin` 可以将一个或多个普通的 `Java  Jar` 包打包成一个标准格式的 `Ark Plugin`； `Ark Plugin` 会包含一份配置文件，通常包括插件类导入导出配置、插件启动优先级等；运行时，Ark 容器会使用独立的 `PluginClassLoader` 加载插件，并根据插件配置构建类加载索引表，从而使插件与插件、插件与应用之间相互隔离；
- `Ark Biz`: Ark 业务模块，满足特定目录格式要求的 `Fat Jar` ，使用官方提供的 `Maven` 插件 `sofa-ark-maven-plugin` 可以将工程应用打包成一个标准格式的 `Ark-Biz` 包；是工程应用模块及其依赖包的组织单元，包含应用启动所需的所有依赖和配置；

在运行时，`Ark Container` 优先启动，自动解析 classpath 包含的 `Ark Plugin` 和 `Ark Biz`，并读取他们的配置，构建类加载索引关系；然后使用独立的 ClassLoader 加载他们并按优先级配置依次启动；需要指出的是，`Ark Plugin` 优先 `Ark Biz` 被加载启动；`Ark Plugin` 之间是双向类索引关系，即可以相互委托对方加载所需的类；`Ark Plugin` 和 `Ark Biz` 是单向类索引关系，即只允许 `Ark Biz` 索引 `Ark Plugin` 加载的类，反之则不允许。

打包成一个 `Ark Plugin`，然后让应用工程引入该插件依赖；
