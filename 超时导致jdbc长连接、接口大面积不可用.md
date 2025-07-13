表现为某个页面长久未响应，之后大部分方法都报jdbc连接异常。

具体异常

```
org.springframework.transaction.CannotCreateTransactionException: Could not open JDBC Connection for transaction; nested exception is java.sql.SQLTransientConnectionException: HikariPool-1 - Connection is not available, request timed out after 30000ms.
	at org.springframework.jdbc.datasource.DataSourceTransactionManager.doBegin(DataSourceTransactionManager.java:306)
	at org.springframework.transaction.support.AbstractPlatformTransactionManager.getTransaction(AbstractPlatformTransactionManager.java:378)
	at org.springframework.transaction.interceptor.TransactionAspectSupport.createTransactionIfNecessary(TransactionAspectSupport.java:475)
	at org.springframework.transaction.interceptor.TransactionAspectSupport.invokeWithinTransaction(TransactionAspectSupport.java:289)
	at org.springframework.transaction.interceptor.TransactionInterceptor.invoke(TransactionInterceptor.java:98)
	at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:186)
	at org.springframework.aop.interceptor.ExposeInvocationInterceptor.invoke(ExposeInvocationInterceptor.java:93)
	at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:186)
	at org.springframework.aop.framework.CglibAopProxy$DynamicAdvisedInterceptor.intercept(CglibAopProxy.java:689)
```

可以看到请求连接池30s了依旧没有可用的jdbc连接。

因为是在测试过程发生的，直接定位到了一个接口上。

> TODO：其它排查手段

某个请求外部http的方法由于之前测试和环境因素，设了很长的超时时间，一直占用着资源。

同时从异常栈第二行可以看出，`jdbc连接`是`getTransaction()`开启事务时就获取的，并不是实际调用数据库时再获取。

因此判断jdbc连接都被这个方法长时间占用导致其它方法获取不到，即通俗的`jdbc长连接`。