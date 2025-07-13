## jar目录

- `META-INF`目录 ：通过 `MANIFEST.MF` 文件提供 `jar` 包的**元数据**，声明了 `jar` 的启动类。
- `org` 目录：为 Spring Boot 提供的 [`spring-boot-loader`](https://github.com/spring-projects/spring-boot/blob/master/spring-boot-project/spring-boot-tools/spring-boot-loader/) 项目。
-  `BOOT-INF/lib` 目录：我们 Spring Boot 项目中引入的**依赖**的 `jar` 包们。`spring-boot-loader` 项目很大的一个作用，就是**解决 `jar` 包里嵌套 `jar` 的情况**，如何加载到其中的类。
-  `BOOT-INF/classes` 目录：我们在 Spring Boot 项目中 Java 类所编译的 `.class`、配置文件等等。

## MANIFEST.MF

```java
//Main-Class: org.springframework.boot.loader.PropertiesLauncher
Main-Class: org.springframework.boot.loader.JarLauncher
Start-Class: com.charlotte.spring.Application
Spring-Boot-Classes: BOOT-INF/classes/
Spring-Boot-Lib: BOOT-INF/lib/
Created-By: Apache Maven 3.6.3
Build-Jdk: 1.8.0_251
Specification-Version: 1.0
```

`Main-class`：Java 规定的 `jar` 包的启动类，这里即Spring Boot的`spring-boot-loader`启动类，均为`Launcher`的子类。

`Start-Class` ：`spring-boot-loader`加载项，Spring Boot 规定的**主**启动类，这里设置为定义的 Application 类。

> 通过 Spring Boot 提供的 Maven 插件 [`spring-boot-maven-plugin`](https://github.com/spring-projects/spring-boot/blob/master/spring-boot-project/spring-boot-tools/spring-boot-maven-plugin/) 进行打包，该插件将该配置项写入到 `MANIFEST.MF` 中，从而能让 `spring-boot-loader` 能够引导启动 Spring Boot 应用。

> Java 规定可执行器的 `jar` 包禁止嵌套其它 `jar` 包。但是我们可以看到 `BOOT-INF/lib` 目录下，实际有 Spring Boot 应用依赖的所有 `jar` 包。因此，`spring-boot-loader` 项目自定义实现了 ClassLoader 实现类 [LaunchedURLClassLoader](https://github.com/spring-projects/spring-boot/blob/master/spring-boot-project/spring-boot-tools/spring-boot-loader/src/main/java/org/springframework/boot/loader/LaunchedURLClassLoader.java)，支持加载 `BOOT-INF/classes` 目录下的 `.class` 文件，以及 `BOOT-INF/lib` 目录下的 `jar` 包。

## Launcher

### ExecutableArchiveLauncher

抽象类，继承自`Launcher`，提供了`Start-Class`的操作，并定义一些抽象提供子类`JarLauncher`、`WarLauncher`实现。

```java
public abstract class ExecutableArchiveLauncher extends Launcher {
	// <1>
    private final Archive archive;

	public ExecutableArchiveLauncher() {
		try {
            // <2>
			this.archive = super.createArchive();
		}
		catch (Exception ex) {
			throw new IllegalStateException(ex);
		}
	}
    ...
}
```

<1>.`Archive`为`spring-boot-loader`定义的对于文件/目录的处理类。

<2>. `Launcher#createArchive()`：获取当前执行`jar`包的`URI`，根据是否为目录返回`ExplodedArchive`(？？？？猜测提供给war包使用)或`JarFileArchive`对象。

#### JarLauncher

```java
public class JarLauncher extends ExecutableArchiveLauncher {

   static final String BOOT_INF_CLASSES = "BOOT-INF/classes/";

   static final String BOOT_INF_LIB = "BOOT-INF/lib/";

   public JarLauncher() {
      // super();
   }

   protected JarLauncher(Archive archive) {
      super(archive);
   }

   @Override
   protected boolean isNestedArchive(Archive.Entry entry) {
      // BOOT-INF/classes/ 目录被归类为一个 Archive 对象，而 BOOT-INF/lib/ 目录下的每个内嵌 jar 包都对应一个 Archive 对象。
      // 如果是目录的情况，只要 BOOT-INF/classes/ 目录
      if (entry.isDirectory()) {
         return entry.getName().equals(BOOT_INF_CLASSES);
      }
      // 如果是文件的情况，只要 BOOT-INF/lib/ 目录下的 `jar` 包
      return entry.getName().startsWith(BOOT_INF_LIB);
   }

   public static void main(String[] args) throws Exception {
      // 隐式调用父类构造方法初始化archive属性
      new JarLauncher().launch(args);
   }

}
```

`JarLauncher`为`Main-Class`定义的启动类之一，`JarLauncher#main(String[] args)`中创建了JarLauncher对象，并隐式调用父类`ExecutableArchiveLauncher`的构造方法，初始化`Archive`，之后`Launcher#launch(String[] args)`。

##### Launcher#launch(String[] args)

```java
// Launcher.class

protected void launch(String[] args) throws Exception {
   // <1> 注册 URL 协议的处理器
   // 注册HANDLERS_PACKAGE属性为org.springframework.boot.loader
   JarFile.registerUrlProtocolHandler();
   // <2> 创建类加载器
   // 基于获得的 Archive 数组，创建自定义 ClassLoader 实现类 LaunchedURLClassLoader，通过它来加载 BOOT-INF/classes 目录下的类，以及 BOOT-INF/lib 目录下的 jar 包中的类。
   ClassLoader classLoader = createClassLoader(getClassPathArchives());
   // <3> 执行启动类的 main 方法
   launch(args, getMainClass(), classLoader);
}
```

`JarFile.registerUrlProtocolHandler()`中对系统参数进行了设置/追加。

`Launcher#getClassPathArchives()`是抽象方法，子类提供了实现。

```java
// ExecutableArchiveLauncher.class   
@Override
protected List<Archive> getClassPathArchives() throws Exception {
    // <1> 获得所有 Archive
    List<Archive> archives = new ArrayList<>(this.archive.getNestedArchives(this::isNestedArchive));
    // 等同于如下代码，使用匿名内部类的lambda
    //    List<Archive> archives = new ArrayList<>(this.archive.getNestedArchives(new Archive.EntryFilter() {
    //       @Override
    //       public boolean matches(Archive.Entry entry) {
    //          return isNestedArchive(entry);
    //       }
    //    }));
    // <2> 后续处理，？？？？空实现，子类中也不存在重写，意义存疑
    postProcessClassPathArchives(archives);
    return archives;
}
```

`ExecutableArchiveLauncher#isNestedArchive(Archive.Entry entry)`也是抽象方法。

```java
// JarLauncher.class

static final String BOOT_INF_CLASSES = "BOOT-INF/classes/";

static final String BOOT_INF_LIB = "BOOT-INF/lib/";

@Override
protected boolean isNestedArchive(Archive.Entry entry) {
   // BOOT-INF/classes/ 目录被归类为一个 Archive 对象，而 BOOT-INF/lib/ 目录下的每个内嵌 jar 包都对应一个 Archive 对象。
   // 如果是目录的情况，只要 BOOT-INF/classes/ 目录
   if (entry.isDirectory()) {
      return entry.getName().equals(BOOT_INF_CLASSES);
   }
   // 如果是文件的情况，只要 BOOT-INF/lib/ 目录下的jar包
   return entry.getName().startsWith(BOOT_INF_LIB);
}
```

回到`Launcher#launch(String[] args)`中

```java
   ClassLoader classLoader = createClassLoader(getClassPathArchives());
   // <3> 执行启动类的 main 方法
   launch(args, getMainClass(), classLoader);
```

继续执行`Launcher#createClassLoader(List<Archive> archives)`，使用加载的`Archive`集合作为`URL`参数创建`LaunchedURLClassLoader`对象。`LaunchedURLClassLoader`是`spring-boot-loader`定义的类加载器。

`Launcher#getMainClass()`抽象方法，返回 Start-Class的类全名。

```java
// ExecutableArchiveLauncher.class
@Override
protected String getMainClass() throws Exception {
   // 获得启动的类的全名
   Manifest manifest = this.archive.getManifest();
   String mainClass = null;
   // 从 jar 包的 MANIFEST.MF 文件的 Start-Class 配置项，，获得我们设置的 Spring Boot 的主启动类。
   if (manifest != null) {
      mainClass = manifest.getMainAttributes().getValue("Start-Class");
   }
   if (mainClass == null) {
      throw new IllegalStateException("No 'Start-Class' manifest entry specified in " + this);
   }
   return mainClass;
}
```

##### Launcher#launch(String[] args, String mainClass, ClassLoader classLoader)

```java
protected void launch(String[] args, String mainClass, ClassLoader classLoader) throws Exception {
   // 设置 LaunchedURLClassLoader 作为类加载器
   Thread.currentThread().setContextClassLoader(classLoader);
   // 创建 MainMethodRunner 对象，并执行 run 方法，启动 Spring Boot 应用
   createMainMethodRunner(mainClass, args, classLoader).run();
}

protected MainMethodRunner createMainMethodRunner(String mainClass, String[] args, ClassLoader classLoader) {
    return new MainMethodRunner(mainClass, args);
}
```
##### MainMethodRunner

```java
public MainMethodRunner(String mainClass, String[] args) {
   this.mainClassName = mainClass;
   this.args = (args != null) ? args.clone() : null;
}

public void run() throws Exception {
   // 通过 LaunchedURLClassLoader 类加载器，加载到我们设置的 Spring Boot 的主启动类。
   Class<?> mainClass = Thread.currentThread().getContextClassLoader().loadClass(this.mainClassName);
   // 通过反射调用主启动类的 #main(String[] args) 方法，启动 Spring Boot 应用。这里也告诉了我们答案，为什么我们通过编写一个带有 #main(String[] args) 方法的类，就能够启动 Spring Boot 应用。
   Method mainMethod = mainClass.getDeclaredMethod("main", String[].class);
   mainMethod.invoke(null, new Object[] { this.args });
}
```

#### WarLauncher

顾名思义，用于加载war。

```java
public class WarLauncher extends ExecutableArchiveLauncher {

   private static final String WEB_INF = "WEB-INF/";

   private static final String WEB_INF_CLASSES = WEB_INF + "classes/";

   private static final String WEB_INF_LIB = WEB_INF + "lib/";

   private static final String WEB_INF_LIB_PROVIDED = WEB_INF + "lib-provided/";

   public WarLauncher() {
		// super();
   }

   protected WarLauncher(Archive archive) {
      super(archive);
   }

   @Override
   public boolean isNestedArchive(Archive.Entry entry) {
      if (entry.isDirectory()) {
         // 若为目录，仅加载WEB-INF/classes/
         return entry.getName().equals(WEB_INF_CLASSES);
      }
      else {
         // 否则，仅加载WEB-INF/lib/或WEB-INF/lib/lib-provided/下的文件
         return entry.getName().startsWith(WEB_INF_LIB) || entry.getName().startsWith(WEB_INF_LIB_PROVIDED);
      }
   }

   public static void main(String[] args) throws Exception {
      new WarLauncher().launch(args);
   }

}
```

除`WarLauncher#isNestedArchive(Archive.Entry entry)`的实现不同，其余均与`JarLauncher`一致。

### PropertiesLauncher

// todo

PropertiesLauncher继承自Launcher，无子类实现，直接查看main方法。实例化PropertiesLauncher。

```java
// PropertiesLauncher.class
public static void main(String[] args) throws Exception {
    PropertiesLauncher launcher = new PropertiesLauncher();
    args = launcher.getArgs(args);
    launcher.launch(args);
}
```
PropertiesLauncher构造方法。
```java
public class PropertiesLauncher extends Launcher {
	private final File home;
	private Archive parent;
    
    
    public PropertiesLauncher() {
        try {
            this.home = getHomeDirectory();
            initializeProperties();
            initializePaths();
            this.parent = createArchive();
        }
        catch (Exception ex) {
            throw new IllegalStateException(ex);
        }
    }
    // ......
}
```

`initializeProperties()`


```java
public class PropertiesLauncher extends Launcher {
    
	/**
	 * Properties key for config file location (including optional classpath:, file: or
	 * URL prefix).
	 */
	public static final String CONFIG_LOCATION = "loader.config.location";
    
	/**
	 * Properties key for name of external configuration file (excluding suffix). Defaults
	 * to "application". Ignored if {@link #CONFIG_LOCATION loader config location} is
	 * provided instead.
	 */
	public static final String CONFIG_NAME = "loader.config.name";
    
	private void initializeProperties() throws Exception, IOException {
		List<String> configs = new ArrayList<>();
		if (getProperty(CONFIG_LOCATION) != null) {
			configs.add(getProperty(CONFIG_LOCATION));
		}
		else {
			String[] names = getPropertyWithDefault(CONFIG_NAME, "loader").split(",");
			for (String name : names) {
				configs.add("file:" + getHomeDirectory() + "/" + name + ".properties");
				configs.add("classpath:" + name + ".properties");
				configs.add("classpath:BOOT-INF/classes/" + name + ".properties");
			}
		}
		for (String config : configs) {
			try (InputStream resource = getResource(config)) {
				if (resource != null) {
					debug("Found: " + config);
					loadResource(resource);
					// Load the first one we find
					return;
				}
				else {
					debug("Not found: " + config);
				}
			}
		}
	}
}
```

