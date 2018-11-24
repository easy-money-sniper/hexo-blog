---
title: AQS
date: 2018-11-24 13:52:38
tags: Java
---
&emsp;&emsp;AQS(AbstractQueuedSynchronizer)是由大神Doug Lea编写的用于构建锁和同步器的框架，位于java.util.concurrent包。该包下很多同步类例如ReentrantLock、ReentrantReadWriteLock、CountDownLatch、Semaphore、CyclicBarrier等，都是基于AQS基类衍生而来。

# AQS原理

## 同步状态
AQS使用一个state变量来表示同步状态，通过FIFO队列来完成线程的排队工作。
```
/**
 * The synchronization state.
 */
private volatile int state; // 使用volatile修饰保证线程可见性
```
对于state操作，AQS提供了三种基本方法
```
protected final int getState() {
    return state;
}
protected final void setState(int newState) {
    state = newState;
}
// CAS原子更新
protected final boolean compareAndSetState(int expect, int update) {
    // See below for intrinsics setup to support this
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```

## 支持的同步方式
- 独占式
>同一时刻只有一个线程可以获取共享资源，其他线程只能等待。
- 共享式
>多个线程可同时获取共享资源。

此外，获取资源时根据是否需要排队又有公平和非公平之分。

对于使用者来讲可以任意实现其中一种，独占式如ReentrantLock，共享式如CountDownLatch、CyclicBarrier、Semaphore，当然也可以同时实现两种，如ReentrantReadWriteLock。

## CLH锁队列
CLH(Craig, Landin, Hagersten)锁是一种基于链表的可扩展、高性能、公平的自旋锁，申请线程仅仅在本地变量上自旋，它不断轮询前驱的状态，假设发现前驱释放了锁就结束自旋。
![CLH](https://raw.githubusercontent.com/easy-money-sniper/hexo-blog/master/source/images/AQS-CLH.png "CLH")

## 设计模式
AQS是基于模版方法模式的，它定义好了一套线程并发争抢资源的逻辑骨架。不同类型的同步器争抢及释放资源的方式不同，但是资源获取失败、线程的排队、阻塞、唤醒等等复杂操作AQS已经实现好了，使用者只需要重写对共享资源的争抢和释放即可。

# 源码解析
接下来分析源代码，首先介绍Node节点，此后按照独占式的资源获取-释放&共享式的资源获取-释放的顺序介绍。

## Node
Node节点是AQS的一个静态内部类，前面提到的内置的FIFIO队列其实就是一个双向链表，由一个个Node节点组成。

```
static final class Node {
    /** Marker to indicate a node is waiting in shared mode */
    // 声明该节点是在共享模式下获取资源
    static final Node SHARED = new Node();
    /** Marker to indicate a node is waiting in exclusive mode */
    // 声明该节点是在独占模式下获取资源
    static final Node EXCLUSIVE = null;

    /** waitStatus value to indicate thread has cancelled */
    // 该节点处于取消排队的状态（无效状态）
    static final int CANCELLED =  1;
    /** waitStatus value to indicate successor's thread needs unparking */
    // 表明该节点的后继节点需要被唤醒
    static final int SIGNAL    = -1;
    /** waitStatus value to indicate thread is waiting on condition */
    // 表明该节点在等待某种条件唤醒
    static final int CONDITION = -2;
    /**
     * waitStatus value to indicate the next acquireShared should
     * unconditionally propagate
     */
    // 仅与共享模式有关
    static final int PROPAGATE = -3;

    volatile int waitStatus;

    volatile Node prev;

    volatile Node next;

    volatile Thread thread;

    // Link to next node waiting on condition, or the special value SHARED
    Node nextWaiter;
    ....
}
```

## acquire(int)
该方法是独占模式下线程获取资源的入口，若获取到资源则返回，否则将该线程封装成Node节点入队，直到获取资源为止，且该过程不响应中断。

```
// 定义了获取资源的逻辑框架
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

从该方法中可以先简单理下逻辑：
1. tryAcquire尝试去获取资源，若成功则返回；
2. 若tryAcquire失败，则通过addWaiter构造一个独占模式的节点入队；
3. acquireQueued使线程在等待队列中获取资源，如果整个过程被中断则返回true（之后补上中断），否则返回false。

```
// 其中tryAcquire由子类根据自定义同步器的功能去实现
protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}
```

如果tryAcquire获取共享资源失败，则需要将该线程入队，等待时机再次获取。

```
private Node addWaiter(Node mode) {
    // 将该线程封装成Node节点，并声明独占模式
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        // 这里先快速入队，通过CAS更换队尾
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    // 若上一步失败则通过enq入队
    enq(node);
    return node;
}
```

```
private Node enq(final Node node) {
    // 自旋入队尾
    for (;;) {
        Node t = tail;
        // 第一次入队，先初始化头尾节点
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            // CAS更改队尾
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

acquireQueued方法是该线程在队列中获取资源的逻辑。

```
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true; // 标记获取资源是否失败
    try {
        boolean interrupted = false; // 标记等待过程中是否被中断过
        for (;;) {
            // 找到前驱节点，若前驱是头节点，则表明自己可获取资源
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                // 获取成功则将该节点设置为头节点
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            // 如果不是头节点或者此时占有资源的线程还未释放，则判断是否需要阻塞
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt()) // 阻塞该线程并检测中断
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}  
```

```
// 判断该线程节点是否该被阻塞
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
    int ws = pred.waitStatus; // 前驱节点等待状态
    if (ws == Node.SIGNAL)
        // 如果前驱状态为SIGNAL，则表明自己可以被park了，因为前驱节点已经知道自己释放后需要唤醒后继节点了
        /*
         * This node has already set status asking a release
         * to signal it, so it can safely park.
         */
        return true;
    if (ws > 0) {
        /*
         * Predecessor was cancelled. Skip over predecessors and
         * indicate retry.
         */
        // 若前驱取消（放弃排队），则前驱是无效节点了，需要往前找到一个状态有效的节点，并排在后边
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
        /*
         * waitStatus must be 0 or PROPAGATE.  Indicate that we
         * need a signal, but don't park yet.  Caller will need to
         * retry to make sure it cannot acquire before parking.
         */
        // 若前驱状态正常，则将其设置为SIGNAL，告诉前驱苟富贵勿相忘～记得unpark后面排队的小弟
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

```
// 阻塞线程直至被unpark并返回是否被中断
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}
```

## release(int)

```
// 定义释放资源的逻辑框架
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

```
// 同样交由子类实现
protected boolean tryRelease(int arg) {
    throw new UnsupportedOperationException();
}
```

```
// unpark后继节点
private void unparkSuccessor(Node node) {
    /*
     * If status is negative (i.e., possibly needing signal) try
     * to clear in anticipation of signalling.  It is OK if this
     * fails or if status is changed by waiting thread.
     */
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    /*
     * Thread to unpark is held in successor, which is normally
     * just the next node.  But if cancelled or apparently null,
     * traverse backwards from tail to find the actual
     * non-cancelled successor.
     */
    Node s = node.next;
    // 此处还未清楚什么时候后继节点为NULL
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

## acquireShared(int)
该方法是共享模式下线程获取资源的入口。

```
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
```

```
// 小于0表示获取失败，等于0表示后续节点不可获取成功，大于0表示后续节点可获取成功
protected int tryAcquireShared(int arg) {
    throw new UnsupportedOperationException();
}
```

```
private void doAcquireShared(int arg) {
    // 构造节点入队，并声明共享模式
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    // 获取成功则将当前节点设置为HEAD，若还有可用资源，传播下去，即唤醒后继结点
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
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

```
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head; // Record old head for check below
    setHead(node);
    /*
     * Try to signal next queued node if:
     *   Propagation was indicated by caller,
     *     or was recorded (as h.waitStatus either before
     *     or after setHead) by a previous operation
     *     (note: this uses sign-check of waitStatus because
     *      PROPAGATE status may transition to SIGNAL.)
     * and
     *   The next node is waiting in shared mode,
     *     or we don't know, because it appears null
     *
     * The conservatism in both of these checks may cause
     * unnecessary wake-ups, but only when there are multiple
     * racing acquires/releases, so most need signals now or soon
     * anyway.
     */
    // 若propagate（state值）大于0，则表示可继续acquire
    // 此处进行了两次头节点的判断，若非空且状态被设置过
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        // 若后续节点不存在且为共享模式则释放并唤醒
        if (s == null || s.isShared())
            doReleaseShared();
    }
}
```

为什么后续节点不存在也需要唤醒，注释中有句话是这么说的：

>The conservatism in both of these checks may cause unnecessary wake-ups, but only when there are multiple racing acquires/releases, so most need signals now or soon anyway.

这种保守的检查会导致不必要的唤醒，但是这种情况只发生在并发获取或释放，所以大部分时候线程都是立即需要一个信号的。这个信号就是unpark，LockSupport在执行unpark的时候相当于给出了一个信号，即使此时该线程并没有被阻塞，但后续执行park方法并不会使该线程被阻塞。

共享模式下的资源获取逻辑与独占模式基本一致，只是当获取成功之后并不是像独占模式一样直接返回，而是需要将“获取资源成功”这个消息传递给下个节点，这就是共享模式，有福同享～。

## releaseShared(int)

```
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```

```
protected boolean tryReleaseShared(int arg) {
    throw new UnsupportedOperationException();
}
```

```
private void doReleaseShared() {
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            // 若头节点状态为SIGNAL，证明队列中的第一个非HEAD节点需要被unpark
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            // 若状态是0，即没被更改过状态，则设置为传播状态
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
```

## ConditionObject(//TODO)

# 总结
本章基于源码分析了JAVA同步器框架AQS的实现原理。AQS基于模版方法模式提供了一套多线程并发争抢共享资源的逻辑框架，处理了获取失败、线程入队、阻塞、唤醒等逻辑，自定义同步器只需要选择性实现tryAcquire-tryRelease、tryAcquireShared-tryReleaseShared即可。其实代码中还提供了acquireInterruptibly（响应中断的资源获取）、tryAcquireNanos（响应超时）的方法，逻辑和介绍的几个方法差不多，只是加上了中断和超时抛异常的操作。
