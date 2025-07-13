# 前言

`XXL-JOB`版本**2.2.0**，暂时仅包含客户端的代码。

# Client

## 启动，XxlJobExecutor

`XxlJobExecutor`，执行器对象。

```java
// 属性
public class XxlJobExecutor  {
    // 调度中心地址
    private String adminAddresses;
    // 秘钥
    private String accessToken;
    // 注册名称
    private String appname;
    // 本服务暴露地址，涉及转发等情况时需手动设置
    private String address;
    // 本服务暴露IP，address无值时生效，用来和port生成address
    private String ip;
    // 本服务暴露端口
    private int port;
    // 日志路径
    private String logPath;
    // 日志保留天数
    private int logRetentionDays;
    ...
}
```

### #start()

```java
// XxlJobExecutor.class
public void start() throws Exception {

    // init logpath
    XxlJobFileAppender.initLogPath(logPath);

    // <1>init invoker, admin-client
    initAdminBizList(adminAddresses, accessToken);

    // init JobLogFileCleanThread
    JobLogFileCleanThread.getInstance().start(logRetentionDays);

    // <2>
    // init TriggerCallbackThread
    TriggerCallbackThread.getInstance().start();

    // <3>
    // init executor-server
    initEmbedServer(address, ip, port, appname, accessToken);
}
```

日志部分略过不看，非本篇重点。

#### <1>初始化调度中心信息adminBizList

`#initAdminBizList()`方法初始化调度中心信息，即内部静态属性`#adminBizList`。

```java
// XxlJobExecutor.class
private static List<AdminBiz> adminBizList;
private void initAdminBizList(String adminAddresses, String accessToken) throws Exception {
    if (adminAddresses!=null && adminAddresses.trim().length()>0) {
        for (String address: adminAddresses.trim().split(",")) {
            if (address!=null && address.trim().length()>0) {

                AdminBiz adminBiz = new AdminBizClient(address.trim(), accessToken);

                if (adminBizList == null) {
                    adminBizList = new ArrayList<AdminBiz>();
                }
                adminBizList.add(adminBiz);
            }
        }
    }
}
```

用分隔符`,`解析`#adminAddresses`属性，并使用解析到的地址和`#accessToken`创建`AdminBiz`对象，加入到`#adminBizList`中。

可以确定`AdminBiz`对象为`XXL-JOB`调度器服务的信息，并且可以注册到多个调度中心。

#### <2>启动回调处理线程TriggerCallbackThread

```java
// 单例
public class TriggerCallbackThread {
    private static TriggerCallbackThread instance = new TriggerCallbackThread();
    public static TriggerCallbackThread getInstance(){
        return instance;
    }
    ......
}
```

然后调用`#start()`方法。创建两个线程`#triggerCallbackThread`、`#triggerRetryCallbackThread`并`#start()`。

- `#triggerCallbackThread`处理回调；
- `#triggerRetryCallbackThread`处理失败的回调并重试；
- `#callBackQueue`回调任务队列。

```java
// TriggerCallbackThread.class
private Thread triggerCallbackThread;
private Thread triggerRetryCallbackThread;
private LinkedBlockingQueue<HandleCallbackParam> callBackQueue = new LinkedBlockingQueue<HandleCallbackParam>();
private volatile boolean toStop = false;
public void start() {
    ...
    // <2.1>
    triggerCallbackThread = new Thread(new Runnable() {

        @Override
        public void run() {
            ......
        }
    });
    triggerCallbackThread.setDaemon(true);
    triggerCallbackThread.setName("xxl-job, executor TriggerCallbackThread");
    triggerCallbackThread.start();

    // <2.2>
    triggerRetryCallbackThread = new Thread(new Runnable() {
        @Override
        public void run() {
            ......
        }
    });
    triggerRetryCallbackThread.setDaemon(true);
    triggerRetryCallbackThread.start();

}
```

##### <2.1>triggerCallbackThread

线程`#triggerCallbackThread`，由`#toStop`开关控制的自旋。`callBackQueue`是一个`LinkedBlockingQueue<HandleCallbackParam>`队列。大部分代码都是为了获取该队列中的`HandleCallbackParam`对象，然后执行真正的回调方法`#doCallback()`。

```java
private LinkedBlockingQueue<HandleCallbackParam> callBackQueue = new LinkedBlockingQueue<HandleCallbackParam>();
// TriggerCallbackThread#start()
triggerCallbackThread = new Thread(new Runnable() {
    @Override
    public void run() {
        // normal callback
        while(!toStop){
            try {
                HandleCallbackParam callback = getInstance().callBackQueue.take();
                if (callback != null) {

                    // callback list param
                    List<HandleCallbackParam> callbackParamList = new ArrayList<HandleCallbackParam>();
                    int drainToNum = getInstance().callBackQueue.drainTo(callbackParamList);
                    callbackParamList.add(callback);

                    // callback, will retry if error
                    if (callbackParamList!=null && callbackParamList.size()>0) {
                        // 处理callback
                        doCallback(callbackParamList);
                    }
                }
            } catch (Exception e) {
                if (!toStop) {
                    logger.error(e.getMessage(), e);
                }
            }
        }

        // last callback
        // stop以后的收尾回调处理
        try {
            List<HandleCallbackParam> callbackParamList = new ArrayList<HandleCallbackParam>();
            int drainToNum = getInstance().callBackQueue.drainTo(callbackParamList);
            if (callbackParamList!=null && callbackParamList.size()>0) {
                doCallback(callbackParamList);
            }
        } catch (Exception e) {
            if (!toStop) {
                logger.error(e.getMessage(), e);
            }
        }
        logger.info(">>>>>>>>>>> xxl-job, executor callback thread destory.");
    }
});
```

##### <2.2>triggerRetryCallbackThread

线程`#triggerRetryCallbackThread`，同样由`#toStop`控制自旋，不断调用`#retryFailCallbackFile()`方法。

```java
// TriggerCallbackThread#start()
triggerRetryCallbackThread = new Thread(new Runnable() {
    @Override
    public void run() {
        while(!toStop){
            try {
                retryFailCallbackFile();
            } catch (Exception e) {
                if (!toStop) {
                    logger.error(e.getMessage(), e);
                }
            }
            try {
                TimeUnit.SECONDS.sleep(RegistryConfig.BEAT_TIMEOUT);
            } catch (InterruptedException e) {
                if (!toStop) {
                    logger.error(e.getMessage(), e);
                }
            }
        }
        logger.info(">>>>>>>>>>> xxl-job, executor retry callback thread destory.");
    }
});
```
`#retryFailCallbackFile()`遍历失败的日志文件进行重试，最终同样回到`#doCallback()`方法。

```java
// TriggerCallbackThread.class
private void retryFailCallbackFile(){

    // valid
    File callbackLogPath = new File(failCallbackFilePath);
    if (!callbackLogPath.exists()) {
        return;
    }
    if (callbackLogPath.isFile()) {
        callbackLogPath.delete();
    }
    if (!(callbackLogPath.isDirectory() && callbackLogPath.list()!=null && callbackLogPath.list().length>0)) {
        return;
    }

    // load and clear file, retry
    for (File callbaclLogFile: callbackLogPath.listFiles()) {
        byte[] callbackParamList_bytes = FileUtil.readFileContent(callbaclLogFile);
        List<HandleCallbackParam> callbackParamList = (List<HandleCallbackParam>) JdkSerializeTool.deserialize(callbackParamList_bytes, List.class);

        callbaclLogFile.delete();
        doCallback(callbackParamList);
    }
}
```

##### doCallback

`#doCallback()`中对所有adminBiz终端地址调用api/callback

```java
private void doCallback(List<HandleCallbackParam> callbackParamList){
    boolean callbackRet = false;
    // callback, will retry if error
    for (AdminBiz adminBiz: XxlJobExecutor.getAdminBizList()) {
        try {
            // 对 amdinBiz 的 api/callback 地址进行了rpc调用
            ReturnT<String> callbackResult = adminBiz.callback(callbackParamList);
            if (callbackResult!=null && ReturnT.SUCCESS_CODE == callbackResult.getCode()) {
                callbackLog(callbackParamList, "<br>----------- xxl-job job callback finish.");
                // 成功一条
                callbackRet = true;
                break;
            } else {
                callbackLog(callbackParamList, "<br>----------- xxl-job job callback fail, callbackResult:" + callbackResult);
            }
        } catch (Exception e) {
            callbackLog(callbackParamList, "<br>----------- xxl-job job callback error, errorMsg:" + e.getMessage());
        }
    }
    if (!callbackRet) {
        // 全部失败时写入失败日志，对应上面的triggerRetryCallbackThread线程
        appendFailCallbackFile(callbackParamList);
    }
}
```

> **TODO** `callBackQueue`的来源？？？

#### <3>初始化服务EmbedServer

`#intiEmbedServer()`方法生成合法可用的`address`、`ip`信息，然后实例化`EmbedServer`并`EmbedServer#start()`。

```java
private void initEmbedServer(String address, String ip, int port, String appname, String accessToken) throws Exception {

    // <1>fill ip port
    port = port>0?port: NetUtil.findAvailablePort(9999);
    // <2>
    ip = (ip!=null&&ip.trim().length()>0)?ip: IpUtil.getIp();

    // <3>generate address
    if (address==null || address.trim().length()==0) {
        String ip_port_address = IpUtil.getIpPort(ip, port);   // registry-address：default use address to registry , otherwise use ip:port if address is null
        address = "http://{ip_port}/".replace("{ip_port}", ip_port_address);
    }

    // <4>start
    embedServer = new EmbedServer();
    embedServer.start(address, port, appname, accessToken);
}
```

- <1>获取合法的、可用的端口。当port<=0时，NetUtil#findAvailablePoet(int defaultPort)会根据参数寻找可用的端口。从defaultPort开始不断往上创建ServerSocket(int port)直到成功，创建失败说明端口已被占用，若直到port>=65535仍未成功，则开始从defaultPort往下尝试查找，直到port<=0，则上下均失败，抛出RuntimeException。

- <2>初始化ip。若ip参数为空，则使用SecurityManager初始化为本地的ip地址。

- <3>初始化address。
- <4>实例化`EmbedServer`并`#start()`。

`EmbedServer#start()`启动。

- 调用`#startRegistry()`注册服务到任务调度中心，方法中获取了`ExecutorRegistryThread`实例并`#start()`。
- 创建一个netty服务监听端口，实例化一个`EmbedHttpServerHandler`处理监听的IO信息，任务调度中心会根据注册信息发送调度数据。

```java
// EmbedServer.class
public void start(final String address, final int port, final String appname, final String accessToken) {
    executorBiz = new ExecutorBizImpl();
    thread = new Thread(new Runnable() {

        @Override
        public void run() {

            // param
            EventLoopGroup bossGroup = new NioEventLoopGroup();
            EventLoopGroup workerGroup = new NioEventLoopGroup();
            // 线程池，用来处理NIO接收到的请求
            ThreadPoolExecutor bizThreadPool = new ThreadPoolExecutor(
                    0,
                    200,
                    60L,
                    TimeUnit.SECONDS,
                    new LinkedBlockingQueue<Runnable>(2000),
                    new ThreadFactory() {
                        @Override
                        public Thread newThread(Runnable r) {
                            return new Thread(r, "xxl-rpc, EmbedServer bizThreadPool-" + r.hashCode());
                        }
                    },
                    new RejectedExecutionHandler() {
                        @Override
                        public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
                            throw new RuntimeException("xxl-job, EmbedServer bizThreadPool is EXHAUSTED!");
                        }
                    });


            try {
                // start server
                ServerBootstrap bootstrap = new ServerBootstrap();
                bootstrap.group(bossGroup, workerGroup)
                        .channel(NioServerSocketChannel.class)
                        .childHandler(new ChannelInitializer<SocketChannel>() {
                            @Override
                            public void initChannel(SocketChannel channel) throws Exception {
                                // 监听处理器
                                channel.pipeline()
                                        .addLast(new IdleStateHandler(0, 0, 30 * 3, TimeUnit.SECONDS))  // beat 3N, close if idle
                                        .addLast(new HttpServerCodec())
                                        .addLast(new HttpObjectAggregator(5 * 1024 * 1024))  // merge request & reponse to FULL
                                        .addLast(new EmbedHttpServerHandler(executorBiz, accessToken, bizThreadPool));
                            }
                        })
                        .childOption(ChannelOption.SO_KEEPALIVE, true);

                //  监听netty端口，bind
                ChannelFuture future = bootstrap.bind(port).sync();

                logger.info(">>>>>>>>>>> xxl-job remoting server start success, nettype = {}, port = {}", EmbedServer.class, port);

                // 注册到调度中心，start registry
                startRegistry(appname, address);

                // future阻塞直到信道close，wait util stop
                future.channel().closeFuture().sync();

            } catch (InterruptedException e) {
                if (e instanceof InterruptedException) {
                    logger.info(">>>>>>>>>>> xxl-job remoting server stop.");
                } else {
                    logger.error(">>>>>>>>>>> xxl-job remoting server error.", e);
                }
            } finally {
                // stop
                try {
                    workerGroup.shutdownGracefully();
                    bossGroup.shutdownGracefully();
                } catch (Exception e) {
                    logger.error(e.getMessage(), e);
                }
            }

        }

    });
    thread.setDaemon(true);    // daemon, service jvm, user thread leave >>> daemon leave >>> jvm leave
    thread.start();
}

public void startRegistry(final String appname, final String address) {
    // start registry
    ExecutorRegistryThread.getInstance().start(appname, address);
}
```

### 任务处理器中心jobHandlerRepository

`XxlJobExecutor`同时承担着任务处理器注册中心的功能，内部属性`#jobHandlerRepository`是由handlerName和IJobHandler构成的KV结构。包含了两个方法`#registJobHandler()`和`#loadJobHandler`，顾名思义，分别是注册、获取。

```java
// XxlJobExecutor.class
// ---------------------- job handler repository ----------------------
private static ConcurrentMap<String, IJobHandler> jobHandlerRepository = new ConcurrentHashMap<String, IJobHandler>();
public static IJobHandler registJobHandler(String name, IJobHandler jobHandler){
    logger.info(">>>>>>>>>>> xxl-job register jobhandler success, name:{}, jobHandler:{}", name, jobHandler);
    return jobHandlerRepository.put(name, jobHandler);
}
public static IJobHandler loadJobHandler(String name){
    return jobHandlerRepository.get(name);
}
```

#### 任务执行器，IJobHandler

用户实现`IJobHandler`接口的方法`#execute()`，标识这是任务的执行方法，具体调用过程看下文。

```java
public abstract class IJobHandler {
    
    public abstract ReturnT<String> execute(String param) throws Exception;
	...
}
```

### 任务执行线程中心jobThreadRepository

`XxlJobExecutor`同时承担着任务线程注册中心的功能，内部属性`#jobThreadRepository`是由jobId和JobThread构成的KV结构。包含了三个方法`#registJobThread()`、`#removeJobThread()`和`#loadJobThread`，顾名思义，分别是注册、移除、获取。由于创建线程对象有所限制，相较上面的执行器中心`#jobHandlerRepository`多了移除方法和中断处理。添加覆盖、移除时还会通知停止并中断线程。

```java
// XxlJobExecutor.class
// ---------------------- job thread repository ----------------------
private static ConcurrentMap<Integer, JobThread> jobThreadRepository = new ConcurrentHashMap<Integer, JobThread>();
public static JobThread registJobThread(int jobId, IJobHandler handler, String removeOldReason){
    JobThread newJobThread = new JobThread(jobId, handler);
    newJobThread.start();
    logger.info(">>>>>>>>>>> xxl-job regist JobThread success, jobId:{}, handler:{}", new Object[]{jobId, handler});

    JobThread oldJobThread = jobThreadRepository.put(jobId, newJobThread); // putIfAbsent | oh my god, map's put method return the old value!!!
    if (oldJobThread != null) {
        oldJobThread.toStop(removeOldReason);
        oldJobThread.interrupt();
    }

    return newJobThread;
}
public static JobThread removeJobThread(int jobId, String removeOldReason){
    JobThread oldJobThread = jobThreadRepository.remove(jobId);
    if (oldJobThread != null) {
        oldJobThread.toStop(removeOldReason);
        oldJobThread.interrupt();

        return oldJobThread;
    }
    return null;
}
public static JobThread loadJobThread(int jobId){
    JobThread jobThread = jobThreadRepository.get(jobId);
    return jobThread;
}
```

#### 任务执行线程，JobThread

`JobThread`实现`Thread`接口，作为一个线程调用任务处理器。

- `#jobId`，作为任务的识别编号；

- `#handler`，根据请求可能会有所不同；

- `#triggerQueue`，执行队列。

由于有队列，`#handler`不可能在队列期间重新指定，查找后确实无setter方法，仅构造时传入。所以`#jobId`与`#handler`构成了一个逻辑主键，一类唯一确定的任务仅由一个`JobThread`处理。

```java
public class JobThread extends Thread{
    private static Logger logger = LoggerFactory.getLogger(JobThread.class);

    private int jobId;
    private IJobHandler handler;
    // 执行队列
    private LinkedBlockingQueue<TriggerParam> triggerQueue;
    // 调用记录，每次请求都带有由调度中心生成的、唯一的logId，避免重复执行
    private Set<Long> triggerLogIdSet;    // avoid repeat trigger for the same TRIGGER_LOG_ID

    // 线程停止标识
    private volatile boolean toStop = false;
    private String stopReason;

    // 是否在执行任务
    private boolean running = false;    // if running job
    private int idleTimes = 0;       // idel times

    public JobThread(int jobId, IJobHandler handler) {
        this.jobId = jobId;
        this.handler = handler;
        this.triggerQueue = new LinkedBlockingQueue<TriggerParam>();
        this.triggerLogIdSet = Collections.synchronizedSet(new HashSet<Long>());
    }
    ...
}
```

`JobThread`中一些供外部调用、控制的方法。

```java
// JobThread.class
// 插入任务
public ReturnT<String> pushTriggerQueue(TriggerParam triggerParam) {
   // avoid repeat
   if (triggerLogIdSet.contains(triggerParam.getLogId())) {
      logger.info(">>>>>>>>>>> repeate trigger job, logId:{}", triggerParam.getLogId());
      return new ReturnT<String>(ReturnT.FAIL_CODE, "repeate trigger job, logId:" + triggerParam.getLogId());
   }

   triggerLogIdSet.add(triggerParam.getLogId());
   triggerQueue.add(triggerParam);
       return ReturnT.SUCCESS;
}

// 停止
public void toStop(String stopReason) {
   /**
    * Thread.interrupt只支持终止线程的阻塞状态(wait、join、sleep)，
    * 在阻塞出抛出InterruptedException异常,但是并不会终止运行的线程本身；
    * 所以需要注意，此处彻底销毁本线程，需要通过共享变量方式；
    */
   this.toStop = true;
   this.stopReason = stopReason;
}

// 是否在执行任务
public boolean isRunningOrHasQueue() {
    return running || triggerQueue.size()>0;
}
```

线程`#run()`方法，方法头尾对`#handler`进行初始化和销毁。方法体中使用了内部属性`#toStop`控制自旋开关，不断从队列中取队首，若有则执行。执行时调用`IJobHander`接口方法·`#execute()`的实现。为了可读性，移除其中日志记录的代码。

```java
// JobThread.class
@Override
public void run() {

    // init
    try {
        handler.init();
    } catch (Throwable e) {
        logger.error(e.getMessage(), e);
    }

    // 自旋，execute
    while(!toStop){
        running = false;
        idleTimes++;

        TriggerParam triggerParam = null;
        ReturnT<String> executeResult = null;
        try {
            // to check toStop signal, we need cycle, so wo cannot use queue.take(), instand of poll(timeout)
            triggerParam = triggerQueue.poll(3L, TimeUnit.SECONDS);
            if (triggerParam!=null) {
                // 运行中标识
                running = true;
                // 闲置次数，闲置太久销毁线程
                idleTimes = 0;
                // 开始执行，移除logId
                triggerLogIdSet.remove(triggerParam.getLogId());

                // 开始execute
                // 是否传入了超时时间
                if (triggerParam.getExecutorTimeout() > 0) {
                    // limit timeout
                    Thread futureThread = null;
                    try {
                        final TriggerParam triggerParamTmp = triggerParam;
                        FutureTask<ReturnT<String>> futureTask = new FutureTask<ReturnT<String>>(new Callable<ReturnT<String>>() {
                            @Override
                            public ReturnT<String> call() throws Exception {
                                return handler.execute(triggerParamTmp.getExecutorParams());
                            }
                        });
                        futureThread = new Thread(futureTask);
                        futureThread.start();

                        executeResult = futureTask.get(triggerParam.getExecutorTimeout(), TimeUnit.SECONDS);
                    } catch (TimeoutException e) {
                        // 超时
                        executeResult = new ReturnT<String>(IJobHandler.FAIL_TIMEOUT.getCode(), "job execute timeout ");
                    } finally {
                        futureThread.interrupt();
                    }
                } else {
                    // just execute
                    executeResult = handler.execute(triggerParam.getExecutorParams());
                }
                // 执行结束

                if (executeResult == null) {
                    executeResult = IJobHandler.FAIL;
                } else {
                    executeResult.setMsg(
                        (executeResult!=null&&executeResult.getMsg()!=null&&executeResult.getMsg().length()>50000)
                        ?executeResult.getMsg().substring(0, 50000).concat("...")
                        :executeResult.getMsg());
                    executeResult.setContent(null);    // limit obj size
                }
            } else {
                if (idleTimes > 30) {
                    // 闲置太久销毁线程
                    if(triggerQueue.size() == 0) { // avoid concurrent trigger causes jobId-lost
                        XxlJobExecutor.removeJobThread(jobId, "excutor idel times over limit.");
                    }
                }
            }
        } catch (Throwable e) {
            StringWriter stringWriter = new StringWriter();
            e.printStackTrace(new PrintWriter(stringWriter));
            String errorMsg = stringWriter.toString();
            executeResult = new ReturnT<String>(ReturnT.FAIL_CODE, errorMsg);
        } finally {
            if(triggerParam != null) {
                // 回调，callback handler info
                if (!toStop) {
                    // commonm
                    TriggerCallbackThread.pushCallBack(new HandleCallbackParam(triggerParam.getLogId(), triggerParam.getLogDateTime(), executeResult));
                } else {
                    // is killed
                    ReturnT<String> stopResult = new ReturnT<String>(ReturnT.FAIL_CODE, stopReason + " [job running, killed]");
                    TriggerCallbackThread.pushCallBack(new HandleCallbackParam(triggerParam.getLogId(), triggerParam.getLogDateTime(), stopResult));
                }
            }
        }
    }

    // callback trigger request in queue
    while(triggerQueue !=null && triggerQueue.size()>0){
        TriggerParam triggerParam = triggerQueue.poll();
        if (triggerParam!=null) {
            // 移除剩余的任务并回调
            // is killed
            ReturnT<String> stopResult = new ReturnT<String>(ReturnT.FAIL_CODE, stopReason + " [job not executed, in the job queue, killed.]");
            TriggerCallbackThread.pushCallBack(new HandleCallbackParam(triggerParam.getLogId(), triggerParam.getLogDateTime(), stopResult));
        }
    }

    // destroy
    try {
        handler.destroy();
    } catch (Throwable e) {
        logger.error(e.getMessage(), e);
    }
}
```

## 调度的处理，EmbedHttpServerHandler

在上文中`EmbedServer#start()`时，使用构造方法初始化了一个`EmbedHttpServerHandler`用来处理调度中心发送的数据。

```java
public static class EmbedHttpServerHandler extends SimpleChannelInboundHandler<FullHttpRequest> {
    private static final Logger logger = LoggerFactory.getLogger(EmbedHttpServerHandler.class);
    // 分派执行器，区分各类调用，默认实现ExecutorBizImpl
    private ExecutorBiz executorBiz;
    // 秘钥
    private String accessToken;
    // 线程池，用来处理接收到的请求
    private ThreadPoolExecutor bizThreadPool;
    public EmbedHttpServerHandler(ExecutorBiz executorBiz, String accessToken, ThreadPoolExecutor bizThreadPool) {
        this.executorBiz = executorBiz;
        this.accessToken = accessToken;
        this.bizThreadPool = bizThreadPool;
    }
    ...
}
```

`EmbedHttpServerHandler`实现了netty的`SimpleChannelInboundHandler`接口。在重写的`#channelRead0()`方法中，解析`FullHttpRequest`，获取调度信息。由于NIO是自旋的伪异步，所以用初始化时赋予的线程池`#bizThreadPool`处理调度信息。

```java
// EmbedHttpServerHandler.class
@Override
protected void channelRead0(final ChannelHandlerContext ctx, FullHttpRequest msg) throws Exception {

    // request parse
    //final byte[] requestBytes = ByteBufUtil.getBytes(msg.content());    // byteBuf.toString(io.netty.util.CharsetUtil.UTF_8);
    String requestData = msg.content().toString(CharsetUtil.UTF_8);
    String uri = msg.uri();
    HttpMethod httpMethod = msg.method();
    boolean keepAlive = HttpUtil.isKeepAlive(msg);
    String accessTokenReq = msg.headers().get(XxlJobRemotingUtil.XXL_JOB_ACCESS_TOKEN);

    // 异步处理，避免NIO阻塞，invoke
    bizThreadPool.execute(new Runnable() {
        @Override
        public void run() {
            // <1>do invoke，执行调用
            Object responseObj = process(httpMethod, uri, requestData, accessTokenReq);

            // to json
            String responseJson = GsonTool.toJson(responseObj);

            // <2>write response，响应
            writeResponse(ctx, keepAlive, responseJson);
        }
    });
}
```
### 分派调用

`#process()`方法根据参数分派到内部执行器`#executorBiz`的各个方法执行。

```java
// EmbedHttpServerHandler.class
private ExecutorBiz executorBiz;
private Object process(HttpMethod httpMethod, String uri, String requestData, String accessTokenReq) {

    // valid
    if (HttpMethod.POST != httpMethod) {
        return new ReturnT<String>(ReturnT.FAIL_CODE, "invalid request, HttpMethod not support.");
    }
    if (uri==null || uri.trim().length()==0) {
        return new ReturnT<String>(ReturnT.FAIL_CODE, "invalid request, uri-mapping empty.");
    }
    if (accessToken!=null
            && accessToken.trim().length()>0
            && !accessToken.equals(accessTokenReq)) {
        return new ReturnT<String>(ReturnT.FAIL_CODE, "The access token is wrong.");
    }

    // services mapping
    try {
        if ("/beat".equals(uri)) {
            return executorBiz.beat();
        } else if ("/idleBeat".equals(uri)) {
            IdleBeatParam idleBeatParam = GsonTool.fromJson(requestData, IdleBeatParam.class);
            return executorBiz.idleBeat(idleBeatParam);
        } else if ("/run".equals(uri)) {
            TriggerParam triggerParam = GsonTool.fromJson(requestData, TriggerParam.class);
            return executorBiz.run(triggerParam);
        } else if ("/kill".equals(uri)) {
            KillParam killParam = GsonTool.fromJson(requestData, KillParam.class);
            return executorBiz.kill(killParam);
        } else if ("/log".equals(uri)) {
            LogParam logParam = GsonTool.fromJson(requestData, LogParam.class);
            return executorBiz.log(logParam);
        } else {
            return new ReturnT<String>(ReturnT.FAIL_CODE, "invalid request, uri-mapping("+ uri +") not found.");
        }
    } catch (Exception e) {
        logger.error(e.getMessage(), e);
        return new ReturnT<String>(ReturnT.FAIL_CODE, "request error:" + ThrowableUtil.toString(e));
    }
}
```

#### 默认执行器，ExecutorBizImpl

`ExecutorBizImpl`是`ExecutorBiz`接口的实现之一，在`EmbedServer`初始化`EmbedHttpServerHandler`时赋予的默认实现。

```java
public interface ExecutorBiz {

    public ReturnT<String> beat();

    public ReturnT<String> idleBeat(IdleBeatParam idleBeatParam);

    public ReturnT<String> run(TriggerParam triggerParam);

    public ReturnT<String> kill(KillParam killParam);

    public ReturnT<LogResult> log(LogParam logParam);

}
```

##### run

方法可以概括为，从上文中的`XxlJobExecutor`的获取、注册任务线程`jobThread`。

确定`jobThread`后调用`#pushTriggerQueue()`加入队列，`jobThread`线程就会从队列里取出调度信息执行。

```java
@Override
public ReturnT<String> run(TriggerParam triggerParam) {
    // 获取缓存过的任务线程，load old：jobHandler + jobThread
    JobThread jobThread = XxlJobExecutor.loadJobThread(triggerParam.getJobId());
    IJobHandler jobHandler = jobThread!=null?jobThread.getHandler():null;
    String removeOldReason = null;

    // valid：jobHandler + jobThread
    GlueTypeEnum glueTypeEnum = GlueTypeEnum.match(triggerParam.getGlueType());
    if (GlueTypeEnum.BEAN == glueTypeEnum) {

        // 根据请求的JobHandler加载new jobhandler
        IJobHandler newJobHandler = XxlJobExecutor.loadJobHandler(triggerParam.getExecutorHandler());

        // newJobHandler与缓存中jobThread的jobHandler不一致时，valid old jobThread
        if (jobThread!=null && jobHandler != newJobHandler) {
            // change handler, need kill old thread
            removeOldReason = "change jobhandler or glue type, and terminate the old job thread.";

            jobThread = null;
            jobHandler = null;
        }

        // 确定jobHandler，valid handler
        if (jobHandler == null) {
            jobHandler = newJobHandler;
            if (jobHandler == null) {
                return new ReturnT<String>(ReturnT.FAIL_CODE, "job handler [" + triggerParam.getExecutorHandler() + "] not found.");
            }
        }

    } else if (GlueTypeEnum.GLUE_GROOVY == glueTypeEnum) {
        ...
    } else if (glueTypeEnum!=null && glueTypeEnum.isScript()) {
        ...
    } else {
        return new ReturnT<String>(ReturnT.FAIL_CODE, "glueType[" + triggerParam.getGlueType() + "] is not valid.");
    }

    // 已存在旧任务时，执行策略，默认为阻塞队列对应的策略单机串行，executor block strategy
    if (jobThread != null) {
        ExecutorBlockStrategyEnum blockStrategy = ExecutorBlockStrategyEnum.match(triggerParam.getExecutorBlockStrategy(), null);
        if (ExecutorBlockStrategyEnum.DISCARD_LATER == blockStrategy) {
            // 丢弃后续调度，返回阻塞失败信息，discard when running
            if (jobThread.isRunningOrHasQueue()) {
                return new ReturnT<String>(ReturnT.FAIL_CODE, "block strategy effect："+ExecutorBlockStrategyEnum.DISCARD_LATER.getTitle());
            }
        } else if (ExecutorBlockStrategyEnum.COVER_EARLY == blockStrategy) {
            // 覆盖之前调度，kill running jobThread
            if (jobThread.isRunningOrHasQueue()) {
                removeOldReason = "block strategy effect：" + ExecutorBlockStrategyEnum.COVER_EARLY.getTitle();
             	// 置空，后续重新注册创建时会作废旧的jobThread
                jobThread = null;
            }
        } else {
            // just queue trigger
        }
    }

    // 确定jobThread，replace thread (new or exists invalid)
    if (jobThread == null) {
    	// 注册并创建新的jobThread
        jobThread = XxlJobExecutor.registJobThread(triggerParam.getJobId(), jobHandler, removeOldReason);
    }

    // push data to queue，加入jobThread的执行队列
    ReturnT<String> pushResult = jobThread.pushTriggerQueue(triggerParam);
    return pushResult;
}
```

##### kill

调用`XxlJobExecutor#removeJobThread()`终止。

```java
// ExecutorBizImpl.class
@Override
public ReturnT<String> kill(KillParam killParam) {
    // kill handlerThread, and create new one
    JobThread jobThread = XxlJobExecutor.loadJobThread(killParam.getJobId());
    if (jobThread != null) {
        XxlJobExecutor.removeJobThread(killParam.getJobId(), "scheduling center kill job.");
        return ReturnT.SUCCESS;
    }

    return new ReturnT<String>(ReturnT.SUCCESS_CODE, "job thread already killed.");
}
```

## 注册服务，ExecutorRegistryThread

```java
public void start(final String appname, final String address){

    // valid
    if (appname==null || appname.trim().length()==0) {
        logger.warn(">>>>>>>>>>> xxl-job, executor registry config fail, appname is null.");
        return;
    }
    if (XxlJobExecutor.getAdminBizList() == null) {
        logger.warn(">>>>>>>>>>> xxl-job, executor registry config fail, adminAddresses is null.");
        return;
    }

    registryThread = new Thread(new Runnable() {
        @Override
        public void run() {

            // registry
            while (!toStop) {
                try {
                    RegistryParam registryParam = new RegistryParam(RegistryConfig.RegistType.EXECUTOR.name(), appname, address);
                    for (AdminBiz adminBiz: XxlJobExecutor.getAdminBizList()) {
                        try {
                            ReturnT<String> registryResult = adminBiz.registry(registryParam);
                            if (registryResult!=null && ReturnT.SUCCESS_CODE == registryResult.getCode()) {
                                registryResult = ReturnT.SUCCESS;
                                logger.debug(">>>>>>>>>>> xxl-job registry success, registryParam:{}, registryResult:{}", new Object[]{registryParam, registryResult});
                                break;
                            } else {
                                logger.info(">>>>>>>>>>> xxl-job registry fail, registryParam:{}, registryResult:{}", new Object[]{registryParam, registryResult});
                            }
                        } catch (Exception e) {
                            logger.info(">>>>>>>>>>> xxl-job registry error, registryParam:{}", registryParam, e);
                        }

                    }
                } catch (Exception e) {
                    if (!toStop) {
                        logger.error(e.getMessage(), e);
                    }

                }

                try {
                    if (!toStop) {
                        TimeUnit.SECONDS.sleep(RegistryConfig.BEAT_TIMEOUT);
                    }
                } catch (InterruptedException e) {
                    if (!toStop) {
                        logger.warn(">>>>>>>>>>> xxl-job, executor registry thread interrupted, error msg:{}", e.getMessage());
                    }
                }
            }

            // registry remove
            try {
                RegistryParam registryParam = new RegistryParam(RegistryConfig.RegistType.EXECUTOR.name(), appname, address);
                for (AdminBiz adminBiz: XxlJobExecutor.getAdminBizList()) {
                    try {
                        ReturnT<String> registryResult = adminBiz.registryRemove(registryParam);
                        if (registryResult!=null && ReturnT.SUCCESS_CODE == registryResult.getCode()) {
                            registryResult = ReturnT.SUCCESS;
                            logger.info(">>>>>>>>>>> xxl-job registry-remove success, registryParam:{}, registryResult:{}", new Object[]{registryParam, registryResult});
                            break;
                        } else {
                            logger.info(">>>>>>>>>>> xxl-job registry-remove fail, registryParam:{}, registryResult:{}", new Object[]{registryParam, registryResult});
                        }
                    } catch (Exception e) {
                        if (!toStop) {
                            logger.info(">>>>>>>>>>> xxl-job registry-remove error, registryParam:{}", registryParam, e);
                        }

                    }

                }
            } catch (Exception e) {
                if (!toStop) {
                    logger.error(e.getMessage(), e);
                }
            }
            logger.info(">>>>>>>>>>> xxl-job, executor registry thread destory.");

        }
    });
    registryThread.setDaemon(true);
    registryThread.setName("xxl-job, executor ExecutorRegistryThread");
    registryThread.start();
}
```

## Sping环境的XxlJobExecutor，XxlJobSpringExecutor

`XxlJobSpringExecutor`是spring环境下的任务执行器，继承`XxlJobExecutor`，大多数功能都在父类`XxlJobExecutor`中。

`XxlJobSpringExecutor`继承了spring环境。

> `XxlJobSpringExecutor`实现`SmartInitializingSingleton`接口，spring beanFactory在单例实例化后会调用`#afterSingletonsInstantiated()`方法。

```java
public class XxlJobSpringExecutor extends XxlJobExecutor implements ApplicationContextAware, SmartInitializingSingleton, DisposableBean {
    private static final Logger logger = LoggerFactory.getLogger(XxlJobSpringExecutor.class);

    // start
    @Override
    public void afterSingletonsInstantiated() {

        // init JobHandler Repository (for method)
        initJobHandlerMethodRepository(applicationContext);

        // refresh GlueFactory
        GlueFactory.refreshInstance(1);

        // super start
        try {
            super.start();
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
    ...
}
```

`#initJobHandlerMethodRepository()`方法通过spring容器获取任务的处理器`IJobHandler`，并初始化任务注册中心。

```java
// XxlJobSpringExecutor.class
private void initJobHandlerMethodRepository(ApplicationContext applicationContext) {
    if (applicationContext == null) {
        return;
    }
    // init job handler from method
    String[] beanDefinitionNames = applicationContext.getBeanNamesForType(Object.class, false, true);
    for (String beanDefinitionName : beanDefinitionNames) {
        Object bean = applicationContext.getBean(beanDefinitionName);

        Map<Method, XxlJob> annotatedMethods = null;   // referred to ：org.springframework.context.event.EventListenerMethodProcessor.processBean
        try {
            // match method annotated by {@XxlJob}
            annotatedMethods = MethodIntrospector.selectMethods(bean.getClass(),
                    new MethodIntrospector.MetadataLookup<XxlJob>() {
                        @Override
                        public XxlJob inspect(Method method) {
                            // Class中带有XxlJob注解的方法
                            return AnnotatedElementUtils.findMergedAnnotation(method, XxlJob.class);
                        }
                    });
        } catch (Throwable ex) {
            logger.error("xxl-job method-jobhandler resolve error for bean[" + beanDefinitionName + "].", ex);
        }
        if (annotatedMethods==null || annotatedMethods.isEmpty()) {
            continue;
        }

        for (Map.Entry<Method, XxlJob> methodXxlJobEntry : annotatedMethods.entrySet()) {
            Method method = methodXxlJobEntry.getKey();
            XxlJob xxlJob = methodXxlJobEntry.getValue();
            if (xxlJob == null) {
                continue;
            }

            String name = xxlJob.value();
            if (name.trim().length() == 0) {
                throw new RuntimeException("xxl-job method-jobhandler name invalid, for[" + bean.getClass() + "#" + method.getName() + "] .");
            }
            if (loadJobHandler(name) != null) {
                throw new RuntimeException("xxl-job jobhandler[" + name + "] naming conflicts.");
            }

            // execute method, validate method
            // 参数列为1且String类型
            if (!(method.getParameterTypes().length == 1 && method.getParameterTypes()[0].isAssignableFrom(String.class))) {
                throw new RuntimeException("xxl-job method-jobhandler param-classtype invalid, for[" + bean.getClass() + "#" + method.getName() + "] , " +
                        "The correct method format like \" public ReturnT<String> execute(String param) \" .");
            }
            // 返回ReturnT类型
            if (!method.getReturnType().isAssignableFrom(ReturnT.class)) {
                throw new RuntimeException("xxl-job method-jobhandler return-classtype invalid, for[" + bean.getClass() + "#" + method.getName() + "] , " +
                        "The correct method format like \" public ReturnT<String> execute(String param) \" .");
            }
            method.setAccessible(true);

            // init and destory
            Method initMethod = null;
            Method destroyMethod = null;

            if (xxlJob.init().trim().length() > 0) {
                // 匹配注解声明的init、destory方法
                try {
                    initMethod = bean.getClass().getDeclaredMethod(xxlJob.init());
                    initMethod.setAccessible(true);
                } catch (NoSuchMethodException e) {
                    throw new RuntimeException("xxl-job method-jobhandler initMethod invalid, for[" + bean.getClass() + "#" + method.getName() + "] .");
                }
            }
            if (xxlJob.destroy().trim().length() > 0) {
                try {
                    destroyMethod = bean.getClass().getDeclaredMethod(xxlJob.destroy());
                    destroyMethod.setAccessible(true);
                } catch (NoSuchMethodException e) {
                    throw new RuntimeException("xxl-job method-jobhandler destroyMethod invalid, for[" + bean.getClass() + "#" + method.getName() + "] .");
                }
            }

            // registry jobhandler
            registJobHandler(name, new MethodJobHandler(bean, method, initMethod, destroyMethod));
        }
    }
}
```

由上方可知，在Spring环境下，注册`XXL-JOB`的任务处理器需要三个条件。

- 执行类需在Spring IOC容器初始化时就注入（后续手动注入无效，正常添加Spring组件注解就好），

  `XxlJobSpringExecutor`实例化后会从IOC容器中获取所有实例化的`bean`；

- 执行方法需要添加注解`@XxlJob`；

- 方法入参有且仅有一个，且类型归属于String；

- 方法出参归属于ReturnT。

### 执行目标的封装，MethodJobHandler

继承抽象类`IJobHandler`，使用构造传入的参数实现`#execute()`、`#init()`、`#destroy()`方法。

```java
public class MethodJobHandler extends IJobHandler {

    // 目标bean
    private final Object target;
    // 目标方法
    private final Method method;
    private Method initMethod;
    private Method destroyMethod;

    public MethodJobHandler(Object target, Method method, Method initMethod, Method destroyMethod) {
        this.target = target;
        this.method = method;

        this.initMethod =initMethod;
        this.destroyMethod =destroyMethod;
    }

    @Override
    public ReturnT<String> execute(String param) throws Exception {
        return (ReturnT<String>) method.invoke(target, new Object[]{param});
    }

    @Override
    public void init() throws InvocationTargetException, IllegalAccessException {
        if(initMethod != null) {
            initMethod.invoke(target);
        }
    }

    @Override
    public void destroy() throws InvocationTargetException, IllegalAccessException {
        if(destroyMethod != null) {
            destroyMethod.invoke(target);
        }
    }

    @Override
    public String toString() {
        return super.toString()+"["+ target.getClass() + "#" + method.getName() +"]";
    }
}
```
