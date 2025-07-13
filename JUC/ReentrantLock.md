# ReentrantLock

可重入锁ReentrantLock实现了Lock接口（lock()、lockInterruptibly()、tryLock()、unlock()、newCondition()）。

具体加锁、释放锁实现方法

```java
public void lock() {
    sync.lock();
}
public void unlock() {
    sync.release(1);
}
```

sync是内部抽象类Sync的子类（NonfairSync、FairSync）实例，具体实例在构造方法中赋予

```java
public ReentrantLock() {
    sync = new NonfairSync();
}

public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```



## Sync.lock()

ReentrantLock内部抽象类Sync，继承AbstractQueuedSynchronizer（AQS），AQS使用了模板模式，定义了逻辑框架（final不可重写），并将实现具体逻辑的protected方法tryAcquire(int arg)、tryRelease(int arg)、tryAcquireShared(int arg)、tryReleaseShared(int arg)、isHeldExclusively()交由继承的子类实现，这样外部通过AQS子类，调用父类方法，父类方法根据逻辑框架调用被重写的子类具体逻辑方法，实现队列同步器。

Sync定义了抽象方法lock()，实现了AQS的tryRelease(int arg)、isHeldExclusively()方法，而tryAcquire(int arg)的具体实现交由Sync子类NonfairSync和FairSync实现。

### NonfairSync-1

非公平锁NonfairSync。先看加锁方法lock()。

```java
final void lock() {
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}
```

若state为0，则置为1，并设拥有者为当前线程。否则acquire(int arg)请求锁。

#### compareAndSetState(int expect, int update)

继承自AQS父类模板。经过封装的CAS操作，对state属性进行比较，若为期待值expect，则设为更新值update。CAS操作由操作系统保证原子性。

```java
private static final Unsafe unsafe = Unsafe.getUnsafe();
private static final long stateOffset;
static {
    try {
        stateOffset = unsafe.objectFieldOffset
            (AbstractQueuedSynchronizer.class.getDeclaredField("state"));
    } catch (Exception ex) { throw new Error(ex); }
}
protected final boolean compareAndSetState(int expect, int update) {
    // See below for intrinsics setup to support this
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```

Unsafe类能够使用地址操作Java内存空间，实例在类初始化获取unsafe实例（同时可以看到Unsafe使用了饿汉式单例模式）、记录了state属性的偏移地址stateOffset。在堆中该实例的内存空间查找到stateOffset地址（根据实例首地址和偏移地址），若为期待值expect，设为更新值update（以内存地址为操作元素，类似C指针）。

#### acquire(int arg)

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}

/**
* Creates and enqueues node for current thread and given mode.
*
* @param mode Node.EXCLUSIVE for exclusive, Node.SHARED for shared
* @return the new node
*/
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        //新节点前驱设为尾节点
        if (compareAndSetTail(pred, node)) {
            //??1：CAS，交换二者地址的内存，即node地址指向了tail，pred指向了新的node
            //??2:tail内存地址指向了pre，CAS后指向node
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}
/**
* Inserts node into queue, initializing if necessary. See picture above.
* @param node the node to insert
* @return node's predecessor
*/
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        //链表头节点不存储数据
        if (t == null) { // Must initialize
            //初始化头尾节点，为同一个，continue
            if (compareAndSetHead(new Node()))
                //unsafe.compareAndSwapObject(this, headOffset, null, update);
                //若head为null,则为head指向的内存地址初始化一个空的node
                tail = head;
        } else {
            //将node插入尾节点之前	
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                //（存疑）CAS后t（即tail）地址tailOffset与node地址内存空间数据交换，并且其它变量（t与node）指向对象不变。由于tail为AQS内部类，指向AQS实例的内部地址，因此CAS后引用不变，但地址存储的值更新为node。
                t.next = node;
                return t;
            }
        }
    }
}

final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

AQS模板方法，tryAcquire(int arg)具体逻辑由子类重写实现。若尝试请求锁失败，则将线程加入等待队列。

Node为AQS内部类，链表节点，存储线程thread，前驱prev，后置next。AQS存储了Node链表的头部head和尾部tail，初始均为null。



### NonfairSync-2

```java
protected final boolean tryAcquire(int acquires) {
	return nonfairTryAcquire(acquires);
}
/**
* super
*/
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

实现AQS的具体算法tryAcquire(int arg)。调用父类Sync方法nonfairTryAcquire。如果当前state为0，则同步器未被线程占用，尝试CAS操作，成功则置同步器拥有者为当前线程，否则占用失败。若state不为0且同步器拥有者为当前线程，则重入，state再次加锁。从代码角度看，第二个if代码块是可能存在线程安全问题的，无法保证state取值、修改、赋值操作的原子性，然而从逻辑上看，state修改、赋值仅在同步拥有者为当前线程时才进入（即重入锁），意味着这部分代码对同一个同步器实例最多有且仅有一个线程（同步器的拥有者）在运行，因此并不存在线程安全问题。

### FairSync

公平锁lock()方法。

```java
final void lock() {
    acquire(1);
}
/**
* super
*/
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}

protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
    	if (!hasQueuedPredecessors() && compareAndSetState(0, acquires)) {
            // 当前锁无线程占用state=0，且无等待队列
            // 若CAS成功，则当前线程请求锁成功
            // 遵循FIFO，所以称为公平锁
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    // 重入判断
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
        	throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

对比：

两种锁在请求锁acquire()时先tryAcquire()尝试请求锁（子类重写模板），公平锁会先判断若等待队列需为空，之后两种锁尝试CAS占用锁，CAS失败则加入等待队列。

## Sync.release(int arg)

```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}

private void unparkSuccessor(Node node) {
    /*
         * If status is negative (i.e., possibly needing signal) try
         * to clear in anticipation of signalling.  It is OK if this
         * fails or if status is changed by waiting thread.
         */
    int ws = node.waitStatus;
    if (ws < 0)
        //小于0则置0
        compareAndSetWaitStatus(node, ws, 0);

    /*
         * Thread to unpark is held in successor, which is normally
         * just the next node.  But if cancelled or apparently null,
         * traverse backwards from tail to find the actual
         * non-cancelled successor.
         */
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            //从尾节点向前遍历，直至前驱为null或node，s取最前一个waitStatus<=0的某个节点
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        //？？作用
        LockSupport.unpark(s.thread);
}
```

调用继承自AQS的tryRelease()方法，尝试释放锁，若释放成功则查询离头结点最近的一个waitStatus节点unpark()。

### tryRelease(int releases)

```java
protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    //重入层数
    if (Thread.currentThread() != getExclusiveOwnerThread())
        //非占有线程，异常
        throw new IllegalMonitorStateException();
    boolean free = false;
    if (c == 0) {
        //释放后层数为0，释放锁的占有线程
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

**存疑：**释放锁时没有对链表进行维护，释放线程的node节点依旧留存在链表中。