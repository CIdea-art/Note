定义一个接口

```java
public interface FunctionThrowable<T, R> {

    R apply(T t) throws Exception;

//    void run();
}
```

逐步以匿名实现接口方式


```java
    FunctionThrowable<Integer, Integer> functionThrowable = null;
    // 普通匿名实现接口写法
    functionThrowable = new FunctionThrowable<Integer, Integer>() {
        @Override
        public Integer apply(Integer i) throws Exception {
            return i * 10;
        }
    };
    // 单接口的匿名实现接口写法简化，即Lambda表达式的原形，只保留入参和方法体
    functionThrowable = (Integer i) -> {
        return i * 100;
    };
    // 单参数再次简化
    functionThrowable = i -> {
        return i * 1000;
    };
    // 单行语句再次简化
    functionThrowable = i -> 1 * 20;
	// 如果再定义一个方法接收上面的Integer i作为参数，假定方法签名为Integer add(Integer i);
    functionThrowable = this::add;
    try {
        functionThrowable.apply(1);
    } catch (Exception e) {
        e.printStackTrace();
    }
```

`FunctionThrowable#apply(Integer i)`接口声明异常的原因：**（只限显式的异常声明）**异常声明可将Lambda的异常交由外部调用Lambda的方法处理，否则需要Lambda内部处理，因为匿名实现接口不可声明超出父类接口的异常、不可向外抛出。