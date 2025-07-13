# AbstractQueuedSynchronizer-1

队列同步器AQS 构建锁或者其他相关同步装置的基础框架 ，用于实现多线程环境下对同一资源的访问控制。使用了int类型的state表示同步器状态。 期望它能够成为实现大部分同步需求的基础。使用的方法是继承，子类通过继承同步器并需要实现它的方法来管理其状态，管理的方式就是通过类似acquire和release的方式来操纵状态。然而多线程环境中对状态的操纵必须确保原子性，因此子类对于状态的把握，需要使用这个同步器提供的getState、setState、 compareAndSetState 三个方法对状态进行操作。注意get then set并不能保证state读写操作的原子性，若state读写存在多线程问题，同步器提供了compareAndSetState，即CAS方法保证读写的原子性。

```java
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {
        /**
     * Head of the wait queue, lazily initialized.  Except for
     * initialization, it is modified only via method setHead.  Note:
     * If head exists, its waitStatus is guaranteed not to be
     * CANCELLED.
     */
    // 头结点
    private transient volatile Node head;

    /**
     * Tail of the wait queue, lazily initialized.  Modified only via
     * method enq to add new wait node.
     */
    // 尾节点
    private transient volatile Node tail;

    /**
     * The synchronization state.
     */
    // 同步器状态
    private volatile int state;
    ......
}
```

AQS内部定义Node类，实现一个双向链表结构的等待队列。

## Node

```java
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {
    
    static final class Node {
        /**
             * Status field, taking on only the values:
             *   SIGNAL:     The successor of this node is (or will soon be)
             *               blocked (via park), so the current node must
             *               unpark its successor when it releases or
             *               cancels. To avoid races, acquire methods must
             *               first indicate they need a signal,
             *               then retry the atomic acquire, and then,
             *               on failure, block.
             *   CANCELLED:  This node is cancelled due to timeout or interrupt.
             *               Nodes never leave this state. In particular,
             *               a thread with cancelled node never again blocks.
             *   CONDITION:  This node is currently on a condition queue.
             *               It will not be used as a sync queue node
             *               until transferred, at which time the status
             *               will be set to 0. (Use of this value here has
             *               nothing to do with the other uses of the
             *               field, but simplifies mechanics.)
             *   PROPAGATE:  A releaseShared should be propagated to other
             *               nodes. This is set (for head node only) in
             *               doReleaseShared to ensure propagation
             *               continues, even if other operations have
             *               since intervened.
             *   0:          None of the above
             *
             * The values are arranged numerically to simplify use.
             * Non-negative values mean that a node doesn't need to
             * signal. So, most code doesn't need to check for particular
             * values, just for sign.
             *
             * The field is initialized to 0 for normal sync nodes, and
             * CONDITION for condition nodes.  It is modified using CAS
             * (or when possible, unconditional volatile writes).
             */
        // 节点状态
        volatile int waitStatus;

        /**
             * Link to predecessor node that current node/thread relies on
             * for checking waitStatus. Assigned during enqueuing, and nulled
             * out (for sake of GC) only upon dequeuing.  Also, upon
             * cancellation of a predecessor, we short-circuit while
             * finding a non-cancelled one, which will always exist
             * because the head node is never cancelled: A node becomes
             * head only as a result of successful acquire. A
             * cancelled thread never succeeds in acquiring, and a thread only
             * cancels itself, not any other node.
             */
        // 前驱节点
        volatile Node prev;

        /**
             * Link to the successor node that the current node/thread
             * unparks upon release. Assigned during enqueuing, adjusted
             * when bypassing cancelled predecessors, and nulled out (for
             * sake of GC) when dequeued.  The enq operation does not
             * assign next field of a predecessor until after attachment,
             * so seeing a null next field does not necessarily mean that
             * node is at end of queue. However, if a next field appears
             * to be null, we can scan prev's from the tail to
             * double-check.  The next field of cancelled nodes is set to
             * point to the node itself instead of null, to make life
             * easier for isOnSyncQueue.
             */
        // 后置节点
        volatile Node next;

        /**
             * The thread that enqueued this node.  Initialized on
             * construction and nulled out after use.
             */
        // 对应线程
        volatile Thread thread;

        /** waitStatus value to indicate thread has cancelled */
        static final int CANCELLED =  1;
        /** waitStatus value to indicate successor's thread needs unparking */
        /** 当前节点的后继节点包含的线程需要运行，也就是unpark */
        static final int SIGNAL    = -1;
        /** waitStatus value to indicate thread is waiting on condition */
        /** 当前节点在等待condition，也就是在condition队列中 */
        static final int CONDITION = -2;
        /**
         * waitStatus value to indicate the next acquireShared should
         * unconditionally propagate
         * 当前节点在sync队列中，等待着获取锁
         */
        static final int PROPAGATE = -3;
        ......
    }
    .......
}
```

节点成为sync队列和condition队列构建的基础，在同步器中就包含了sync队列。同步器拥有三个成员变量：sync队列的头结点head、sync队列的尾节点tail和状态state。对于锁的获取，请求形成节点，将其挂载在尾部，而锁资源的转移（释放再获取）是从头部开始向后进行。对于同步器维护的状态state，多个线程对其的获取将会产生一个链式的结构。 

# AbstractQueuedSynchronizer-2

AQS通过acquire、release、acquireShared、releaseShared方法实现同步器对内部队列的线程请求和线程释放。以上函数会调用各自对应的try方法。 

AQS使用模板模式，定义抽象方法tryAcquire、tryRelease、tryAcquireShared、tryReleaseShared、isHeldExclusively，抽象方法的的具体实现交由子类实现。AQS仅定义了try的抽象方法，需子类重写实现具体的请求释放策略，未重写调用则抛出UnsupportedOperationException。

子类推荐被定义为自定义同步装置的内部类，同步器自身没有实现任何同步接口，它仅仅是定义了若干acquire()之类的方法来供使用。该同步器即可以作为排他模式也可以作为共享模式，当它被定义为一个排他模式时，其他线程对其的获取就被阻止，而共享模式对于多个线程获取都可以成功。

同步器是实现锁的关键，利用同步器将锁的语义实现，然后在锁的实现中聚合同步器。可以这样理解：锁的API是面向使用者的，它定义了与锁交互的公共行为，而每个锁需要完成特定的操作也是透过这些行为来完成的（比如：可以允许两个线程进行加锁，排除两个以上的线程），但是实现是依托给同步器来完成；同步器面向的是线程访问和资源控制，它定义了线程对资源是否能够获取以及线程的排队等操作。锁和同步器很好的隔离了二者所需要关注的领域，严格意义上讲，同步器可以适用于除了锁以外的其他同步设施上（包括锁）。

## acquire

 该方法以排他的方式获取锁，对中断不敏感，完成synchronized语义。 

```java
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    /**
     * Acquires in exclusive mode, ignoring interrupts.  Implemented
     * by invoking at least once {@link #tryAcquire},
     * returning on success.  Otherwise the thread is queued, possibly
     * repeatedly blocking and unblocking, invoking {@link
     * #tryAcquire} until success.  This method can be used
     * to implement method {@link Lock#lock}.
     *
     * @param arg the acquire argument.  This value is conveyed to
     *        {@link #tryAcquire} but is otherwise uninterpreted and
     *        can represent anything you like.
     */
    public final void acquire(int arg) {
    	// tryAcquire请求同步器的具体实现交予子类实现，若请求失败，则执行模板化操作，新建节点并加入等待队列，然后中断自身线程
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
        if (pred != null) {// 队列已初始化
            node.prev = pred;
            // 新节点前驱设为尾节点
            if (compareAndSetTail(pred, node)) {
                // CAS，若pred与tail地址相同，则交换双方地址内容，令tail指向的地址（即tailOffset）变为node对象，node指向的地址变为tail，参数pred依旧指向tail命名。
                // 则令tail指向node地址，若CAS失败，则tail与pred地址不同，出现了脏读，存在另一个线程抢先进入队列改变了tail，重新循环
                pred.next = node;
                return node;
            }
        }
        enq(node);// 队列未初始化，执行初始化
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
            // 链表头节点不存储数据
            if (t == null) { 
                // Must initialize
                // 若Node初始化成功，头尾指针均指向其
                if (compareAndSetHead(new Node()))
                    // unsafe.compareAndSwapObject(this, headOffset, null, update);
                    // 若head为null,则为head指向的内存地址初始化一个空的node
                    tail = head;
            } else {
                //将node插入尾节点之前	
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    // ??3: CAS后t（即tail）地址tailOffset与node地址内存空间数据交换，并且其它变量（t与node）指向对象不变。由于tail为AQS内部类，指向AQS实例的内部地址，因此CAS后引用不变，但地址存储的值更新为node。
                    t.next = node;
                    return t;
                }
            }
        }
    }
    /**
     * Acquires in exclusive uninterruptible mode for thread already in
     * queue. Used by condition wait methods as well as acquire.
     *
     * @param node the node
     * @param arg the acquire argument
     * @return {@code true} if interrupted while waiting
     */
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
    /**
     * Convenience method to interrupt current thread.
     */
    static void selfInterrupt() {
        Thread.currentThread().interrupt();
    }
	......
}
```

## release(int arg)

```java

public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    /**
     * Releases in exclusive mode.  Implemented by unblocking one or
     * more threads if {@link #tryRelease} returns true.
     * This method can be used to implement method {@link Lock#unlock}.
     *
     * @param arg the release argument.  This value is conveyed to
     *        {@link #tryRelease} but is otherwise uninterpreted and
     *        can represent anything you like.
     * @return the value returned from {@link #tryRelease}
     */
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
    ......
}
```

## ConditionObject

AQS的条件控制器，对AQS等待队列中的线程进行条件休眠、唤醒。

```java
public class ConditionObject implements Condition, java.io.Serializable {
    /** First node of condition queue. */
    // 首节点指针
    private transient Node firstWaiter;
    /** Last node of condition queue. */
    // 尾节点指针，链表单节点时为null
	private transient Node lastWaiter;
    ......
}
```

休眠await()

```java
public abstract class AbstractQueuedSynchronizer
    extends AbstractOwnableSynchronizer
    implements java.io.Serializable {
	public class ConditionObject implements Condition, java.io.Serializable {
        public final void await() throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
            // 添加新节点入链表
            Node node = addConditionWaiter();
            int savedState = fullyRelease(node);
            int interruptMode = 0;
            while (!isOnSyncQueue(node)) {
                LockSupport.park(this);
                if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                    break;
            }
            if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
                interruptMode = REINTERRUPT;
            if (node.nextWaiter != null) // clean up if cancelled
                unlinkCancelledWaiters();
            if (interruptMode != 0)
                reportInterruptAfterWait(interruptMode);
        }
        /**
         * Adds a new waiter to wait queue.
         * @return its new wait node
         */
        private Node addConditionWaiter() {
            Node t = lastWaiter;
            // If lastWaiter is cancelled, clean out.
            if (t != null && t.waitStatus != Node.CONDITION) {
                unlinkCancelledWaiters();
                t = lastWaiter;
            }
            // 入链尾
            Node node = new Node(Thread.currentThread(), Node.CONDITION);
            if (t == null)
                firstWaiter = node;
            else
                t.nextWaiter = node;
            lastWaiter = node;
            return node;
        }
        ......
    }


    /**
         * Invokes release with current state value; returns saved state.
         * Cancels node and throws exception on failure.
         * @param node the condition node for this wait
         * @return previous sync state
         */
    final int fullyRelease(Node node) {
        boolean failed = true;
        try {
            int savedState = getState();
            // 释放同步器的占用状态，置0
            if (release(savedState)) {
                failed = false;
                return savedState;
            } else {
                throw new IllegalMonitorStateException();
            }
        } finally {
            if (failed)
                node.waitStatus = Node.CANCELLED;
        }
    }
    ......
}
```