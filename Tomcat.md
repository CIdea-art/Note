## 类加载机制

![image-20210508171241490](C:\Users\Charlotte\AppData\Roaming\Typora\typora-user-images\image-20210508171241490.png)

- Common 通⽤类加载器加载Tomcat使⽤以及应⽤通⽤的⼀些类，位于CATALINA_HOME/lib下，⽐如servlet-api.jar；

- CatalinaClassLoader ⽤于加载服务器内部可⻅类，这些类应⽤程序不能访问；

- SharedClassLoader ⽤于加载应⽤程序共享类，这些类服务器不会依赖；

- WebappClassLoader，每个应⽤程序都会有⼀个独⼀⽆⼆的WebappClassLoader，他⽤来加载本应⽤程序 /WEB-INF/classes 和 /WEB-INF/lib 下的类。

### WebappClassLoader

每个WebappClassLoader应用类加载器对应一个应用。

WebappClassLoader重写了loadClass()方法，打破双亲委派机制，使其可以自己优先加载类，使每个应用都使用了自己的类加载器加载类。以此实现了不同应用的类的隔离性。同时，每个应用的类加载器和加载的类相互独立，可单独部署、移除。

> 若遵循双亲委派机制，应用的大部分类加载操作就会被委托给共同的父加载器AppClassLoader。
>
> 例：两个应用引用了不同版本的spring，但是spring版本不一致。对不同版本的同一个类，全限定名相同，代码却不一致，歧义就此产生。

```java
// WebappClassLoaderBase.class，WebappClassLoader的父类，loadClass()方法继承自此
@Override
public Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {

    synchronized (JreCompat.isGraalAvailable() ? this : getClassLoadingLock(name)) {
        if (log.isDebugEnabled())
            log.debug("loadClass(" + name + ", " + resolve + ")");
        Class<?> clazz = null;

        // Log access to stopped class loader
        checkStateForClassLoading(name);

        // (0) Check our previously loaded local class cache
        // 从当前ClassLoader的本地缓存中加载类，如果找到则返回
        clazz = findLoadedClass0(name);
        if (clazz != null) {
            if (log.isDebugEnabled())
                log.debug("  Returning class from cache");
            if (resolve)
                resolveClass(clazz);
            return clazz;
        }

        // (0.1) Check our previously loaded class cache
        // 本地缓存没有的情况下，调用ClassLoader的findLoadedClass方法查看jvm是否已经加载过此类，如果已经加载则直接返回。
        clazz = JreCompat.isGraalAvailable() ? null : findLoadedClass(name);
        if (clazz != null) {
            if (log.isDebugEnabled())
                log.debug("  Returning class from cache");
            if (resolve)
                resolveClass(clazz);
            return clazz;
        }

        // (0.2) Try loading the class with the system class loader, to prevent
        //       the webapp from overriding Java SE classes. This implements
        //       SRV.10.7.2
        String resourceName = binaryNameToPath(name, false);

        // 此时的javaseClassLoader是扩展类加载器  是把扩展类加载器赋值给了javaseClassLoader
        ClassLoader javaseLoader = getJavaseClassLoader();
        boolean tryLoadingFromJavaseLoader;
        try {
            //如果可以用getResource得到
            //如果能用扩展类加载器的getResource得到就证明可以被扩展类加载器加载到接下来安排扩展类加载器加载
            URL url;
            if (securityManager != null) {
                PrivilegedAction<URL> dp = new PrivilegedJavaseGetResource(resourceName);
                url = AccessController.doPrivileged(dp);
            } else {
                url = javaseLoader.getResource(resourceName);
            }
            tryLoadingFromJavaseLoader = (url != null);
        } catch (Throwable t) {
            // Swallow all exceptions apart from those that must be re-thrown
            ExceptionUtils.handleThrowable(t);
            // The getResource() trick won't work for this class. We have to
            // try loading it directly and accept that we might get a
            // ClassNotFoundException.
            tryLoadingFromJavaseLoader = true;
        }
        if (tryLoadingFromJavaseLoader) {
            try {
                clazz = javaseLoader.loadClass(name);
                if (clazz != null) {
                    if (resolve)
                        resolveClass(clazz);
                    return clazz;
                }
            } catch (ClassNotFoundException e) {
                // Ignore
            }
        }

        // (0.5) Permission to access this class when using a SecurityManager
        if (securityManager != null) {
            int i = name.lastIndexOf('.');
            if (i >= 0) {
                try {
                    securityManager.checkPackageAccess(name.substring(0,i));
                } catch (SecurityException se) {
                    String error = sm.getString("webappClassLoader.restrictedPackage", name);
                    log.info(error, se);
                    throw new ClassNotFoundException(error, se);
                }
            }
        }

        boolean delegateLoad = delegate || filter(name, true);

        // (1) Delegate to our parent if requested
        // 如果是true就是尝试用父类加载器进行加载
        if (delegateLoad) {
            if (log.isDebugEnabled())
                log.debug("  Delegating to parent classloader1 " + parent);
            try {
                clazz = Class.forName(name, false, parent);
                if (clazz != null) {
                    if (log.isDebugEnabled())
                        log.debug("  Loading class from parent");
                    if (resolve)
                        resolveClass(clazz);
                    return clazz;
                }
            } catch (ClassNotFoundException e) {
                // Ignore
            }
        }

        // (2) Search local repositories
        // 尝试自己加载
        if (log.isDebugEnabled())
            log.debug("  Searching local repositories");
        try {
            clazz = findClass(name);
            if (clazz != null) {
                if (log.isDebugEnabled())
                    log.debug("  Loading class from local repository");
                if (resolve)
                    resolveClass(clazz);
                return clazz;
            }
        } catch (ClassNotFoundException e) {
            // Ignore
        }

        // (3) Delegate to parent unconditionally
        // 补偿，自己加载失败且未委托给父类(1)时，委托给父类加载
        if (!delegateLoad) {
            if (log.isDebugEnabled())
                log.debug("  Delegating to parent classloader at end: " + parent);
            try {
                clazz = Class.forName(name, false, parent);
                if (clazz != null) {
                    if (log.isDebugEnabled())
                        log.debug("  Loading class from parent");
                    if (resolve)
                        resolveClass(clazz);
                    return clazz;
                }
            } catch (ClassNotFoundException e) {
                // Ignore
            }
        }
    }

    throw new ClassNotFoundException(name);
}
```

