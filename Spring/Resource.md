http://svip.iocoder.cn/Spring/IoC-load-Resource/

## Resource



## ResourceLoader

一开始就说了 Spring 将资源的定义和资源的加载区分开了，Resource 定义了统一的资源，**那资源的加载则由 ResourceLoader 来统一定义**。

`org.springframework.core.io.ResourceLoader` 为 Spring 资源加载的统一抽象，具体的资源加载则由相应的实现类来完成，所以我们可以将 ResourceLoader 称作为统一资源定位器。其定义如下：

> FROM 《Spring 源码深度解析》P16 页
>
> ResourceLoader，定义资源加载器，主要应用于根据给定的资源文件地址，返回对应的 Resource 。



### DefaultResourceLoader

与 `AbstractResource` 相似，`org.springframework.core.io.DefaultResourceLoader` 是 ResourceLoader 的默认实现。

#### #construct

```java
@Nullable
private ClassLoader classLoader;

public DefaultResourceLoader() { // 无参构造函数
	this.classLoader = ClassUtils.getDefaultClassLoader();
}

public DefaultResourceLoader(@Nullable ClassLoader classLoader) { // 带 ClassLoader 参数的构造函数
	this.classLoader = classLoader;
}

public void setClassLoader(@Nullable ClassLoader classLoader) {
	this.classLoader = classLoader;
}

@Override
@Nullable
public ClassLoader getClassLoader() {
	return (this.classLoader != null ? this.classLoader : ClassUtils.getDefaultClassLoader());
}
```

#### #getResource

