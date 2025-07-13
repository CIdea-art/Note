## 描述

项目中用`ThreadLocal`存储公共核心数据作为Context上下文。

但是在生产环境中，出现了Context错位，A获取到了B的Context的情况。

```java
// 设置代码，切面和手动结合触发，逻辑类似putIfAbsent()。
private ThreadLocal<Obj> holder = new ThreadLocal<>();

public void setContext(...){
    // 获取context
    Obj context = new Context();
    trySet(context);
}

public void trySet(Obj context){
    if(holder.get() == null){
        holder.set(appId);
    }
}
```

根据日志排查下来，确定入参对应的Context无误。

那就定位到`#get()`了。回顾一下逻辑，`ThreadLocal`里面维护了一个键值对，Key是线程号。

判断为线程号已经存在Context。每次创建的新线程是基本不可能相等的，只能是线程对象复用，那就是线程池。

某个方法中因为调用三方接口较多，单个响应慢，串行要二十秒，用了线程池并发处理，每个新线程都用上面方法设置Context，且没有clear。

**总结**：因为线程池复用线程对象的关系，导致复用线程时`ThreadLocal`对象内Key的存储位置一致，并且在结束时未clear，导致设置新Context失效，还保留有上次复用时留下的Context。

## 解决方案

1. 线程执行完时clear一下；
2. 用ttl包代替`ThreadLocal`；