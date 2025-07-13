## 特征

事件驱动，反向代理，热部署

默认多线程，支持多进程

## 网络事件

Nginx 在启动后，会有一个 master 进程和多个 worker 进程。

master 进程主要用来管理 worker 进程，

- 接收来自外界的信号
- 向各 worker 进程发送信号
- 监控 worker 进程的运行状态
- 当 worker 进程退出后(异常情况下)，会自动重新启动新的 worker 进程

work 进程用来处理基本的网络事件。

worker 进程的个数是可以设置的，一般我们会设置与机器cpu核数一致，这里面的原因与 Nginx 的进程模型以及事件处理模型是分不开的。`nginx为了更好的利用多核特性，提供了 cpu 亲缘性的绑定选项，我们可以将某一个进程绑定在某一个核上，这样就不会因为进程的切换带来 cache 的失效。`

> 更多的 worker 数，只会导致进程来竞争 cpu 资源了，从而带来不必要的上下文切换

- 一个请求，只可能在一个 worker 进程中处理，一个 worker 进程，不可能处理其它进程的请求。

- 多个 worker 进程之间是对等的，他们同等竞争来自客户端的请求，各进程互相之间是独立的。

每个 worker 进程都是从 master 进程 fork 过来，在 master 进程里面，先建立好需要 listen 的 socket（listenfd）之后，然后再 fork 出多个 worker 进程。所有 worker 进程的 listenfd 会在新连接到来时变得可读，为保证只有一个进程处理该连接，所有 worker 进程在注册 listenfd 读事件前抢 accept_mutex，抢到互斥锁的那个进程注册 listenfd 读事件，在读事件里调用 accept 接受该连接。当一个 worker 进程在 accept 这个连接之后，就开始读取请求，解析请求，处理请求，产生数据后，再返回给客户端，最后才断开连接

异步非阻塞

阻塞，具体到系统底层，就是读写事件，只能等待读写事件准备完毕。阻塞调用会进入内核等待，CPU空闲，资源浪费。

- 加线程会导致切换上下文成本增加，典型就是apache的线程模型。

非阻塞，事件没有准备好，马上返回 EAGAIN，并间隔重试直到准备好。

异步，把询问事件做成异步监控，有事件准备好就返回。
### 例子

kill -HUP pid

master 进程在接到信号后，会先重新加载配置文件，然后再启动新的 worker 进程，并向所有老的 worker 进程发送信号。老的 worker 在收到来自 master 的信号后，就不再接收新的请求，并且在当前进程中的所有未处理完的请求处理完成后，再退出。

./nginx -s ${command}

执行命令时，我们是启动一个新的 Nginx 进程，而新的 Nginx 进程在解析到 command 参数后，它会向 master 进程发送信号，然后接下来的动作，就和我们直接向 master 进程发送信号一样了。


## 信号

有一些特定的信号，代表着特定的意义。信号会中断掉程序当前的运行，在改变状态后，继续执行。如果是系统调用，则可能会导致系统调用的失败，需要重入。

例子：对于 Nginx 来说，如果nginx正在等待事件（epoll_wait 时），如果程序收到信号，在信号处理函数处理完后，epoll_wait 会返回错误，然后程序可再次进入 epoll_wait 调用。

## 定时器

epoll_wait 等函数在调用的时候是可以设置一个超时时间的，所以 Nginx 借助这个超时时间来实现定时器。

伪代码

```c
while (true) {
    // 执行队列
    for t in run_tasks:
        t.handler();
    update_time(&now);
    timeout = ETERNITY;
    // 检测超时
    for t in wait_tasks: /* sorted already */
        if (t.time <= now) {
            t.timeout_handler();
        } else {
            timeout = t.time - now;
            break;
        }
    // poll出未处理的，加入run执行队列
    nevents = poll_function(events, timeout);
    for i in nevents:
        task t;
        if (events[i].type == READ) {
            t.handler = read_handler;
        } else { /* events[i].type == WRITE */
            t.handler = write_handler;
        }
        run_tasks_add(t);
}
```


