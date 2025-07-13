**加载**

读取类路径对应的类字节码。加载到内存中，生成方法区和Class入口。

**连接**

验证->准备->解析

**初始化**

## 概念

类加载器是一个负责加载类的对象，用于实现类加载过程中的加载这一步。

- 每个 Java 类都有一个引用指向加载它的 `ClassLoader`。

	> Class的getClassLoader()方法		

- 数组类不是通过 `ClassLoader` 创建的（数组类没有对应的二进制字节流），是由 JVM 直接生成的。

特性

- 根据需要动态加载

- 递归委托父加载器先去尝试加载

  > 双亲委派模型中”委派“的来源
  >
  > 但明明是单亲为什么叫双亲？据说是翻译问题，双亲更多地表达的是“父母这一辈”

- 只加载一次

  > 存放在ClassLoader内部属性Vector<Class<?>>集合中
  >
  
 - 判断相等的依据是类路径和类加载器是否都一致，同样的类可能被不同加载器解析成不同含义

## 内置加载器

层级依次降低，且下一个是由上一个加载的。

**`BootstrapClassLoader`**

最顶层的加载类，由 C++实现，在 Java 中是没有与之对应的类。通常表示为 null，null时即是BootstrapClassLoader。

并且没有父级，主要用来加载 JDK 内部的核心类库（ `%JAVA_HOME%/lib`目录下的 `rt.jar` 、`resources.jar` 、`charsets.jar`等 jar 包和类）以及被 `-Xbootclasspath`参数指定的路径下的所有类。

**`ExtensionClassLoader`**

主要负责加载 `%JRE_HOME%/lib/ext` 目录下的 jar 包和类以及被 `java.ext.dirs` 系统变量所指定的路径下的所有类。

> Java9更名为`PlatformClassLoader`

> 线上环境踩了一次坑，详见《一次lib包引发的双亲委派思考》

**`AppClassLoader`**

面向我们用户的加载器，负责加载当前应用 classpath 下的所有 jar 包和类。

## 自定义加载器

除了 `BootstrapClassLoader` 其他类加载器均由 Java 实现且全部继承自`java.lang.ClassLoader`。

加载器的父子关系通过单向链表表示。

```java
public abstract class ClassLoader {
  ...
  // 组合
  private final ClassLoader parent;
  protected ClassLoader(ClassLoader parent) {
       this(checkCreateClassLoader(), parent);
  }
  ...
}
```



`委派`逻辑的核心方法`#loadClass(String name, boolean resolve)`

若想要打破委派模型，重写此方法即可。

> 比如Tomcat的WebappClassLoader，实现了webapp之间的隔离，详见《Tomcat》或资料《深入拆解 Tomcat & Jetty》。

```java
// ClassLoader.class
protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
{
    synchronized (getClassLoadingLock(name)) {
        // First, check if the class has already been loaded
        Class<?> c = findLoadedClass(name);
        if (c == null) {
            long t0 = System.nanoTime();
            try {
                if (parent != null) {
                    // 从下至上，递归委派父类判断是否加载过
                    c = parent.loadClass(name, false);
                } else {
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // ClassNotFoundException thrown if class not found
                // from the non-null parent class loader
            }

            if (c == null) {
                // If still not found, then invoke findClass in order
                // to find the class.
                long t1 = System.nanoTime();
                // 从上至下，查找类
                c = findClass(name);

                // this is the defining class loader; record the stats
                sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                sun.misc.PerfCounter.getFindClasses().increment();
            }
        }
        if (resolve) {
            resolveClass(c);
        }
        return c;
    }
}

```





加载的具体实现`#findClass(String name)`。比如修改Class来源，对加密Class解密。

```java
// ClassLoader.class
protected Class<?> findClass(String name) throws ClassNotFoundException {
    throw new ClassNotFoundException(name);
}
```



双亲委派模型保证了 Java 程序的稳定运行，可以避免类的重复加载，也保证了 Java 的核心 API 不被篡改。

> 自己写的java.lang.Object其实是不生效的，因为BootstrapClassLoader先加载自带的java.lang.Object





# 参考资料

https://mp.weixin.qq.com/s/CCQW0vtr_XZJkjDRKU4jkQ
