---
title: ReentrantLock条件等待与唤醒的流程
date: 2022-11-02 10:16:46
tags:
- Java
- 多线程
categories: Java多线程
---

ReentrantLock 可以通过newCondition用做条件等待。条件不满足，则等待;被唤醒后则入对等待执行。

# [](#aqs中的node)AQS中的Node

在AQS中，获取不到锁时，线程会被封装成Node结构，然后入队；条件等待中，也会将等待线程封装为node。回顾下Node主要结构：

```
static final class Node {        //等待状态。-2表示处于等待需要被唤醒的节点
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

在ConditonObject中，就是使用的Node：

```
public class ConditionObject implements Condition, java.io.Serializable {
        private static final long serialVersionUID = 1173984872572414699L;
        /** First node of condition queue. */
        private transient Node firstWaiter;
        /** Last node of condition queue. */
        private transient Node lastWaiter;

        /**         * Creates a new {@code ConditionObject} instance.         */
        public ConditionObject() { }

        .....
}
```

使用示例：

```
@Slf4j
public class SignalTest {

    @SneakyThrows
    public static void main(String[] args) {
        ReentrantLock lock = new ReentrantLock();
        final Condition condition = lock.newCondition();

        final Thread t1 = new Thread(() -> {            lock.lock();            try {                log.debug("t1 get lock");                condition.await();            } catch (InterruptedException e) {                e.printStackTrace();            } finally {                lock.unlock();            }        }, "t1");        t1.start();        TimeUnit.SECONDS.sleep(10);        lock.lock();        try {            log.debug("main signal!");            condition.signal();        } catch (Exception e) {            lock.unlock();        }finally {        }        log.debug("main over~~~");    }}
```

输出：

```
17:03:14.540 [t1] DEBUG com.demo.exercise.SignalTest - t1 get lock
17:03:24.551 [main] DEBUG com.demo.exercise.SignalTest - main signal!
17:03:24.551 [main] DEBUG com.demo.exercise.SignalTest - main over~~~
```

示例说明：由于主线程延迟10秒才会获取锁后，因此线程t1肯定会先执行。t1执行时，直接通过condition.await()释放锁并park自己。

# [](#await等待后释放锁)await等待后释放锁

同synchronized中使用一样，在调用同步对象的wait方法后会释放锁。ReentrantLock中的Condition调用await后也会释放自己占用的锁。

## [](#await流程)await流程

先看看它的源码：

```
        public final void await() throws InterruptedException {
            if (Thread.interrupted())
                throw new InterruptedException();
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
```

大致流程如下：

1. 如果线程被打断，那么直接抛出异常
2. 将当前线程封装为一个等待的Node对象
3. 通过fullyRelease释放锁
4. 如果当前await的线程是一个等待signal信号的或者它的上一个节点为null，那就park 自己

### [](#addconditionwaiter)addConditionWaiter

```
     private Node addConditionWaiter() {
            Node t = lastWaiter;
            // If lastWaiter is cancelled, clean out.
            if (t != null && t.waitStatus != Node.CONDITION) {
                unlinkCancelledWaiters();
                t = lastWaiter;
            }
            Node node = new Node(Thread.currentThread(), Node.CONDITION);
            if (t == null)
                firstWaiter = node;            else
                t.nextWaiter = node;            lastWaiter = node;            return node;        }
```

这个方法其实就是将线程封装为一个Node(此时waitStaus=-2)。

### [](#fullyrelease)fullyRelease

```
    final int fullyRelease(Node node) {
        boolean failed = true;
        try {
            int savedState = getState();
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


        public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }

    protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }
```

几个核心的方法就是上面几个：

1. 获取AQS 中state值
2. 然后通过release中调用tryRelease，设置state值以及将独占线程设置为Null。这个操作主要是让下一个线程可以来抢锁

## [](#signal-主要流程)signal 主要流程

先看源码：

```
   public final void signal() {
            if (!isHeldExclusively())
                throw new IllegalMonitorStateException();
            Node first = firstWaiter;
            if (first != null)
                doSignal(first);
        }


      /**
         * Removes and transfers nodes until hit non-cancelled one or
         * null. Split out from signal in part to encourage compilers
         * to inline the case of no waiters.
         * @param first (non-null) the first node on condition queue
         */
        private void doSignal(Node first) {
            do {
                if ( (firstWaiter = first.nextWaiter) == null)
                    lastWaiter = null;
                first.nextWaiter = null;
            } while (!transferForSignal(first) &&
                     (first = firstWaiter) != null);
        }


            /**
     * Transfers a node from a condition queue onto sync queue.
     * Returns true if successful.
     * @param node the node
     * @return true if successfully transferred (else the node was
     * cancelled before signal)
     */
    final boolean transferForSignal(Node node) {
        /*
         * If cannot change waitStatus, the node has been cancelled.
         */
        if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
            return false;

        /*
         * Splice onto queue and try to set waitStatus of predecessor to
         * indicate that thread is (probably) waiting. If cancelled or
         * attempt to set waitStatus fails, wake up to resync (in which
         * case the waitStatus can be transiently and harmlessly wrong).
         */
        Node p = enq(node);        int ws = p.waitStatus;
        if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
            LockSupport.unpark(node.thread);
        return true;
    }
```

调用signal后：

1. 如果不是我占用了锁，此时释放直接抛异常
2. 找condition队列的第一个等待线程，如果存在就将他唤醒

在doSignal 方法中先对condition中的等待队列进行处理：将需要移入到AQS对列中的下一个节点置为Null。注意Node中只定义了nextWaiter，它是单向链表而非AQS中的双向

transferForSignal 方法流程：

1. 将等待节点的waitStatus先改为0。因为从condition队列中加入到AQS队列时，是放在AQS队列中的最后一个。而最后一个不需要唤醒其他人，所以状态是0
2. 再调用enq方法，将node加入到AQS的队列
3. 通过compareAndSetWaitStatus让从condition中入队到AQS中的节点被它的上一个节点叫醒。上面enq(node)返回的就是node 的上一个节点。

最后， main 的lock.unlock 后，会唤醒AQS中的等待节点
