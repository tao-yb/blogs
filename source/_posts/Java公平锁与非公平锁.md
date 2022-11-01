---
title: Java公平锁与非公平锁
date: 2022-11-01 21:04:22
tags:
- Java
-  多线程
categories: Java多线程
---

什么是公平锁与非公平锁?他们有什么区别? 我们平常使用的是公平锁还是非公平锁?

带着问题，首先来看ReentrantLock与synchronized加锁2个例子：

# [](#reentrantlock)ReentrantLock

```
package com.demo;

import com.google.common.collect.Lists;
import lombok.SneakyThrows;
import lombok.extern.slf4j.Slf4j;

import java.util.List;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;

/** * @author javaer * @Description * @create 2022-09-27 12:17 */
@Slf4j
public class ReetLock {

    private static final int COUNT = 10;

    private static List<Thread> threads = Lists.newArrayList();

    static ReentrantLock lock = new ReentrantLock();

    @SneakyThrows
    public static void main(String[] args) {

        for (int i = 0; i < COUNT; i++) {
            int finalI = i;
            Thread thread = new Thread(() -> {                lock.lock();                try {                    log.debug("{} print -->{}", Thread.currentThread().getName(), finalI);                } finally {                    lock.unlock();                }            }, "t" + i);            threads.add(thread);        }        lock.lock();        try {            log.debug("main get lock ");            for (int i = 0; i < COUNT; i++) {                threads.get(i).start();                TimeUnit.MICROSECONDS.sleep(50);            }        } finally {            lock.unlock();        }    }}
```

输出如下：

```
18:02:31.468 [main] DEBUG com.demo.ReetLock - main get lock 
18:02:31.485 [t0] DEBUG com.demo.ReetLock - t0 print -->0
18:02:31.488 [t1] DEBUG com.demo.ReetLock - t1 print -->1
18:02:31.488 [t2] DEBUG com.demo.ReetLock - t2 print -->2
18:02:31.488 [t3] DEBUG com.demo.ReetLock - t3 print -->3
18:02:31.488 [t4] DEBUG com.demo.ReetLock - t4 print -->4
18:02:31.488 [t5] DEBUG com.demo.ReetLock - t5 print -->5
18:02:31.489 [t6] DEBUG com.demo.ReetLock - t6 print -->6
18:02:31.489 [t7] DEBUG com.demo.ReetLock - t7 print -->7
18:02:31.489 [t8] DEBUG com.demo.ReetLock - t8 print -->8
18:02:31.489 [t9] DEBUG com.demo.ReetLock - t9 print -->9
```

# [](#synchronized)synchronized

```
package com.demo;

import com.google.common.collect.Lists;
import lombok.SneakyThrows;
import lombok.extern.slf4j.Slf4j;

import java.util.List;
import java.util.concurrent.TimeUnit;

/** * @author tyb * @Description * @create 2022-09-27 10:51 */
@Slf4j
public class Sync {

    private static final int COUNT =10;

    private static List<Thread> threads = Lists.newArrayList();

    @SneakyThrows
    public static void main(String[] args) {
        Object o = new Object();
        for (int i = 0; i < COUNT; i++) {
            int finalI = i;
            Thread thread = new Thread(()->{                synchronized (o){                    log.debug("{} print -->{}",Thread.currentThread().getName(), finalI);                }            },"t"+i);            threads.add(thread);        }        synchronized (o){            log.debug("main get lock ");            for (int i = 0; i < COUNT; i++) {                threads.get(i).start();                TimeUnit.MICROSECONDS.sleep(50);            }        }    }}
```

输出如下：

```
18:09:52.283 [main] DEBUG com.demo.Sync - main get lock 
18:09:52.301 [t9] DEBUG com.demo.Sync - t9 print -->9
18:09:52.304 [t8] DEBUG com.demo.Sync - t8 print -->8
18:09:52.304 [t7] DEBUG com.demo.Sync - t7 print -->7
18:09:52.304 [t6] DEBUG com.demo.Sync - t6 print -->6
18:09:52.304 [t5] DEBUG com.demo.Sync - t5 print -->5
18:09:52.304 [t4] DEBUG com.demo.Sync - t4 print -->4
18:09:52.304 [t3] DEBUG com.demo.Sync - t3 print -->3
18:09:52.304 [t2] DEBUG com.demo.Sync - t2 print -->2
18:09:52.304 [t1] DEBUG com.demo.Sync - t1 print -->1
18:09:52.304 [t0] DEBUG com.demo.Sync - t0 print -->0
```

可以看到，同样的逻辑synchronized与ReentrantLock输出是不同的。

那么ReentrantLock是公平锁还是非公平锁?如果ReentrantLock是公平锁是不是就是说synchronized是非公平锁呢?

# [](#reentrantlock-与synchronize)ReentrantLock 与synchronize

ReentrantLock锁通过构造函数就可以指定：

```
    /**
     * Creates an instance of {@code ReentrantLock}.
     * This is equivalent to using {@code ReentrantLock(false)}.
     */
    public ReentrantLock() {
        sync = new NonfairSync();
    }
```

从上面代码以及注释可以看出，默认创建是非公平锁,只有指定参数false时（即：ReentrantLock(false)）才是公平锁。

那么疑问来了，

- 既然是非公平锁，那为什么输出还是顺序的呢？
- 既然上面例子中输出顺序完全相反，是不是说synchronized是公平锁呢?

其实不是这样的。看过之前synchronized介绍的文章，我们指定synchronized是通过系统pthread_mutex 实现了锁的互斥，本质上它是属于非公平锁。

# [](#公平锁与非公平锁的区别)公平锁与非公平锁的区别

- 非公平锁加锁

```
        final void lock() {
            if (compareAndSetState(0, 1))
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }

        public final void acquire(int arg) {
        if (!tryAcquire(arg) &&            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }


      protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }


            /**         * Performs non-fair tryLock.  tryAcquire is implemented in         * subclasses, but both need nonfair try for trylock method.         */
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

- 公平锁加锁

```
    static final class FairSync extends Sync {
        private static final long serialVersionUID = -3000897897090466540L;

        final void lock() {
            acquire(1);
        }

                /**         * Fair version of tryAcquire.  Don't grant access unless         * recursive call or no waiters or is first.         */
        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
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

- 非公平锁上来就通过cas（此时不管队列是否有线程在等待） 修改状态，修改成功，那么就将主机改为拥有锁的线程;而公平锁会先看看是不是没有人持有锁（state=0）并且还会看看有没有排队的线程，没有排队的线程，那么才执行CAS上锁
- 非公平锁第一次CAS失败后，在nonfairTryAcquire中会再次CAS尝试加锁；公平锁则没有这个过程。

# [](#aqs入队)AQS入队

```
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);        }    }
```

线程抢不到锁时，会调用acquireQueued(addWaiter(Node.EXCLUSIVE), arg))。

1. addWaiter(Node.EXCLUSIVE), arg)将线程包装成一个Node节点。
2. 再通过acquireQueued方法将Node节点加入到队列中。

在acquireQueued时，如果前一个节点是head,那么会再一次尝试加锁。这是因为，如果前一个节点是head的话，那么在当前线程来加锁的过程中，head可能会完成加锁业务后释放锁，这样就不需要阻塞当前线程

不管公平锁还是非公平锁，它们都被封装成Node,然后放入AQS的等待队列中。一旦加入队列，那么就排队等待被唤醒

## [](#node的主要结构)Node的主要结构

```
static final class Node {        //等待状态。-1时需要叫醒下一个等待节点
        volatile int waitStatus;

        //等待队列中的前一个节点
        volatile Node prev;

        //等待队列中的下一个节点
        volatile Node next;

        //等待队列中的线程
        volatile Thread thread;

        //指向condition中的等待的节点或者特定Shared node        
        Node nextWaiter;
```

node节点的结构中，有个waitStatus状态值。它表示等待状态。它有几个值：

- 1表示被取消；
- 0 默认（Int的默认值）
- 1 表示需要唤醒下一个节点
- -2 表示在Condition中等待。在ReentrantLock中，可以通过它来做条件等待，类似synchronized中的wait。
- -3 指示下一个 acquireShared 应该无条件传播

上面介绍说通过acquireQueued(addWaiter(Node.EXCLUSIVE), arg))加入队列。在加入队列是，新封装的Node节点的waitStatus值是int类型的默认值0。但是按照JUC中的说明，waitStatus只有是-1的时候才是唤醒状态。那么谁负责唤醒它呢?或者说，如果有新的node节点加入到队列，它是怎样唤醒其他节点的呢？

看看加入队列的 方法：

```
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);        }    }
```

在shouldParkAfterFailedAcquire中，会将waitStatus修改为-1.代码如下：

```
 private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
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
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;        } else {
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
```

注意，这个地方是将要排队的线程封装成Node入队后，将Node的前一个waitStatus改成-1,而当前节点的waitStatus仍然是0,也就是说它不需要唤醒其他节点。 这个是因为，在AQS队列中，是按照线程先后顺序排队的。排在最后的一个线程肯定不要唤醒其他线程

# [](#总结)总结

相同点： 不管是公平锁还是非公平锁，它们的区别主要在加锁的时候。如果一旦进入到等待队列，就会在队列中等待被唤醒。 ReentrantLock 中的队列是FIFO，synchronized则是LIFO模式。
