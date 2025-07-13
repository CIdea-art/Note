# SpringApplication

Spring Boot项目简单启动main方法。

```java
public static void main(String[] args) {
    SpringApplication.run(Application.class, args);
}
```
静态run方法重载。

```java
public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {
   return run(new Class<?>[] { primarySource }, args);
}

public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
    return new SpringApplication(primarySources).run(args);
}

```
new SpringApplication(Class<?>... primarySources)后调用实例run方法。

```java
public class SpringApplication {

	private ResourceLoader resourceLoader;
	private Set<Class<?>> primarySources;
	private WebApplicationType webApplicationType;
	private List<ApplicationContextInitializer<?>> initializers;
	private List<ApplicationListener<?>> listeners;
	private Class<?> mainApplicationClass;

	public SpringApplication(Class<?>... primarySources) {
       this(null, primarySources);
    }
	public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
		this.resourceLoader = resourceLoader;
		Assert.notNull(primarySources, "PrimarySources must not be null");
		this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
		this.webApplicationType = WebApplicationType.deduceFromClasspath();
        // setInitializers(...)与setListeners(...)作用类似，设置对应属性的值，getSpringFactoriesInstances()为加载类的核心方法，后文解释
		setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
		setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
		this.mainApplicationClass = deduceMainApplicationClass();
	}
    ......
}
```

## run

SpringApplication初始化完后开始run。

```java
public ConfigurableApplicationContext run(String... args) {
   // SpringFactoriesLoader 类，会读取 META-INF 目录下的 spring.factories 文件，获得每个框架定义的需要自动配置的配置类。
   // StopWatch执行监听
   StopWatch stopWatch = new StopWatch();
   stopWatch.start();
   ConfigurableApplicationContext context = null;
   Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
   // 设置System变量java.awt.headless=true
   configureHeadlessProperty();
   // 获取以当前spring boot实例为监听对象的SpringApplicationRunListener集合对象，ClassLoader加载
   SpringApplicationRunListeners listeners = getRunListeners(args);
   listeners.starting();
   try {
      // 提供对用于运行{@link SpringApplication}的参数的访问。
      ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
      // 创建ConfigurableEnvironment并配置到相关对象上
      ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
      // 检查System变量spring.beaninfo.ignore，默认true
      configureIgnoreBeanInfo(environment);
      // 打印 Spring Banner横幅
      Banner printedBanner = printBanner(environment);
      // 创建 Spring 容器。
      context = createApplicationContext();
      // 以当前spring容器为构造参数实例化异常报告器
      exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
            new Class[] { ConfigurableApplicationContext.class }, context);
      // todo
      // 主要是调用所有初始化类的 initialize 方法
      prepareContext(context, environment, listeners, applicationArguments, printedBanner);
      // 初始化 Spring 容器。
      refreshContext(context);
      // 执行 Spring 容器的初始化的后置逻辑。默认实现为空。
      afterRefresh(context, applicationArguments);
      stopWatch.stop();
      // 打印 Spring Boot 启动的时长日志。
      if (this.logStartupInfo) {
         new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
      }
      //  通知 SpringApplicationRunListener 的数组，Spring 容器启动完成。
      listeners.started(context);
      // 调用 ApplicationRunner 或者 CommandLineRunner 的运行方法。
      callRunners(context, applicationArguments);
   }
   catch (Throwable ex) {
      handleRunFailure(context, ex, exceptionReporters, listeners);
      throw new IllegalStateException(ex);
   }

   try {
      // 通知 SpringApplicationRunListener 的数组，Spring 容器运行中。
      listeners.running(context);
   }
   catch (Throwable ex) {
      handleRunFailure(context, ex, exceptionReporters, null);
      throw new IllegalStateException(ex);
   }
   return context;
}
```