---
title: AQS
date: 2018-11-21 13:52:38
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
>同一时刻只有一个线程可获取到资源，其他线程只能等待，独占式根据是否需要排队又有公平和非公平之分。
- 共享式
>多个线程可同时获取共享资源。

对于使用者来讲可以任意实现其中一种，独占式如ReentrantLock，共享式如CountDownLatch、Semaphore，当然也可以同时实现两种，如ReentrantReadWriteLock。

## CLH队列锁
CLH锁是一种基于链表的可扩展、高性能、公平的自旋锁，申请线程仅仅在本地变量上自旋，它不断轮询前驱的状态，假设发现前驱释放了锁就结束自旋。
![CLH](https://github.com/easy-money-sniper/hexo-blog/source/images/AQS-CLH.png, "CLH")

## 设计思想
AQS设计思想是基于模版方法模式的。不同类型的同步器争抢及释放资源的方式不同，但是资源获取失败、线程的排队、挂起、唤醒等等复杂操作AQS已经实现好了，使用者只需要重写对共享资源的争抢和释放即可。

# 源码解析

## 获取资源
```
// 定义了获取资源的逻辑框架
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```
```
// 其中tryAcquire由子类根据自定义同步器的功能去实现
protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}
// 
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

## 释放资源
