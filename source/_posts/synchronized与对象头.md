---
title: synchronized与对象头
date: 2022-08-29 21:20:58
tags:
categories: Java多线程
---

引叙： Java里通过AQS实现的锁，在加锁时通过设置state 变量值就能实现上锁。但是synchronized内置锁，它是不是也和AQS 类似会修改变量值呢?如果修改的话，它又是修改了对象什么东西?

先给出结论： synchronized上锁时，会修改对象头。

# [](#openjdk关于对象头与锁状态的说明markoophpp)OpenJdk关于对象头与锁状态的说明（MarkOop.hpp）

```
//    [JavaThread* | epoch | age | 1 | 01]       lock is biased toward given thread
//    [0           | epoch | age | 1 | 01]       lock is anonymously biased
//
//  - the two lock bits are used to describe three states: locked/unlocked and monitor.
//
//    [ptr             | 00]  locked             ptr points to real header on stack
//    [header      | 0 | 01]  unlocked           regular object header
//    [ptr             | 10]  monitor            inflated lock (header is wapped out)
//    [ptr             | 11]  marked             used by markSweep to mark an object
//                                               not valid at any other time
```

通过MarkOop中的注释，可以知道在JVM得出如下结论：

- synchronized有偏向锁，轻量锁、重量锁之分
- 101为 偏向锁。偏向锁还有2种情况：一个是匿名偏向，一个是偏向某个线程
- 00 已经上锁后的状态，它指向线程栈中真是的对象头
- 01 无锁
- 10 重量锁。后面的注释意思是膨胀后的锁，这是因为发生了资源竞争，导致synchronized发生膨胀。

# [](#验证synchronized对对象头的修改)验证synchronized对对象头的修改

先通过synchronized的使用看看对对象头的操作，在示例中，先直接打印对象头然后与加锁后对象头比较一下，看区别在哪

```
@Slf4j
public class ObjectHeader {

    @SneakyThrows
    public static void main(String[] args) {

        Person person = new Person();
        log.debug(ClassLayout.parseInstance(person).toPrintable());
        final Thread thread = new Thread(() -> {            synchronized (person){                log.debug("加锁后===========");                log.debug(ClassLayout.parseInstance(person).toPrintable());            }        });        thread.start();    }}
```

输出：

```
17:39:19.720 [main] DEBUG com.thread.ObjectHeader - com.thread.Person object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000000000000001 (non-biasable; age: 0)
  8   4        (object header: class)    0xf800dd95
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

17:39:19.726 [Thread-1] DEBUG com.thread.ObjectHeader - 加锁后===========
17:39:19.728 [Thread-1] DEBUG com.thread.ObjectHeader - com.thread.Person object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000700010d1b908 (thin lock: 0x0000700010d1b908)
  8   4        (object header: class)    0xf800dd95
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

从上面2次打印的对象头可以看到对象头确实是不一样了。

但你有发现有什么问题没有呢？是不是和上面MarkOop里定义的有点不一样？ 可能有人会说哪里不一样？ 其实仔细观察上面2次打印的对象头，确实发现了这样2个问题：

1. 锁状态是不可偏向的（后面打印信息是non-biasable，意思就锁不可偏向）
2. synchronized加锁后，直接成了轻量锁（注意到加锁后，thin lock了吗？）

为什么？这是因为JVM中存在偏向延迟

# [](#什么是偏向延迟)什么是偏向延迟

JVM自身代码中也会用到synchronized。JVM认为自己的锁是不会存在偏向锁的，所以直接改成了轻量级锁。在JVM启动完成后，用户自己使用synchronized可以采用偏向锁模式。 JVM为什么要这样做呢？因为这样可以避免判断是否是偏向锁的逻辑，直接用轻量锁上锁。

# [](#jvm偏向延迟是多久呢)JVM偏向延迟是多久呢

JVM默认的偏向延迟是4秒钟。可以通过如下命令查看：

```
java -XX:+PrintFlagsFinal -version  | grep Delay
     intx BiasedLockingStartupDelay                 = 4000                                {product}
     bool LIRFillDelaySlots                         = false                               {C1 pd product}
     intx SafepointTimeoutDelay                     = 10000                               {product}
     intx SuspendRetryDelay                         = 5                                   {product}
     intx Tier3DelayOff                             = 2                                   {product}
     intx Tier3DelayOn                              = 5                                   {product}
openjdk version "1.8.0_312"
OpenJDK Runtime Environment (build 1.8.0_312-b07)
OpenJDK 64-Bit Server VM (build 25.312-b07, mixed mode)
```

4秒钟之后，我们定义的对象就可使用偏向锁。

修改代码如下：

```
@Slf4j
public class ObjectHeader {

    @SneakyThrows
    public static void main(String[] args) {
        TimeUnit.SECONDS.sleep(5);
        Person person = new Person();
        log.debug(ClassLayout.parseInstance(person).toPrintable());
        final Thread thread = new Thread(() -> {
            synchronized (person){
                log.debug("加锁后===========");
                log.debug(ClassLayout.parseInstance(person).toPrintable());
            }
        });
        thread.start();
        thread.join();
        log.debug("释放锁后===========");
        log.debug(ClassLayout.parseInstance(person).toPrintable());

    }
}
```

输出：

```
18:32:17.985 [main] DEBUG com.thread.ObjectHeader - com.thread.Person object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000000000000005 (biasable; age: 0)
  8   4        (object header: class)    0xf800dd95
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

18:32:17.988 [Thread-1] DEBUG com.thread.ObjectHeader - 加锁后===========
18:32:17.989 [Thread-1] DEBUG com.thread.ObjectHeader - com.thread.Person object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x00007fcd4e393805 (biased: 0x0000001ff3538e4e; epoch: 0; age: 0)
  8   4        (object header: class)    0xf800dd95
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

18:32:17.989 [main] DEBUG com.thread.ObjectHeader - 释放锁后===========
18:32:17.990 [main] DEBUG com.thread.ObjectHeader - com.thread.Person object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x00007fcd4e393805 (biased: 0x0000001ff3538e4e; epoch: 0; age: 0)
  8   4        (object header: class)    0xf800dd95
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

上面输出中2个都是偏向锁，是因为过了JVM偏向延迟时间后，对象可以使用偏向锁，但是没有线程加锁所以锁10。也就是MarkOop中说到的匿名偏向。在synchronized中，因为已经上锁了，所以还是101。此时输出的数据0x0000001ff625c58c中，包含偏向的线程ID等信息。

> 也通过-XX:BiasedLockingStartupDelay=0关闭偏向延迟

还有一点需要注意的是，即使释放锁后对象头打印的也没有改变。是因为偏向锁如果下次再来加锁，直接比较对象头就行了，CAS也不需要了。这样的效率是最高的。如果此时对象头还原成了匿名锁对象头一样，那下次线程再来加锁，还需要进行CAS操作。

# [](#重量锁的对象头)重量锁的对象头

首先修改代码如下：

```
    @SneakyThrows
    public static void main(String[] args) {
        Person person = new Person();
        log.debug(ClassLayout.parseInstance(person).toPrintable());
        final Thread thread = new Thread(() -> {            synchronized (person){                log.debug("加锁后===========");                log.debug(ClassLayout.parseInstance(person).toPrintable());            }        });        thread.start();        //thread.join();        synchronized (person){            log.debug("main 线程加锁成功===========");            log.debug(ClassLayout.parseInstance(person).toPrintable());        }    }
```

输出：

```
18:39:54.432 [main] DEBUG com.thread.ObjectHeader - com.thread.Person object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000000000000005 (biasable; age: 0)
  8   4        (object header: class)    0xf800dd95
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

18:39:54.435 [main] DEBUG com.thread.ObjectHeader - main 线程加锁成功===========
18:39:54.436 [main] DEBUG com.thread.ObjectHeader - com.thread.Person object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x00007f8826814dda (fat lock: 0x00007f8826814dda)
  8   4        (object header: class)    0xf800dd95
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

18:39:54.436 [Thread-1] DEBUG com.thread.ObjectHeader - 加锁后===========
18:39:54.437 [Thread-1] DEBUG com.thread.ObjectHeader - com.thread.Person object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x00007f8826814dda (fat lock: 0x00007f8826814dda)
  8   4        (object header: class)    0xf800dd95
 12   4        (object alignment gap)    
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

这样只看到成了fat lock。将jol修改成0.10版本后，再看下：

```
18:42:41.120 [main] DEBUG com.thread.ObjectHeader - com.thread.Person object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           05 00 00 00 (00000101 00000000 00000000 00000000) (5)
      4     4        (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4        (object header)                           95 dd 00 f8 (10010101 11011101 00000000 11111000) (-134161003)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

18:42:41.123 [main] DEBUG com.thread.ObjectHeader - main 线程加锁成功===========
18:42:41.124 [main] DEBUG com.thread.ObjectHeader - com.thread.Person object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           ea f5 01 6f (11101010 11110101 00000001 01101111) (1862399466)
      4     4        (object header)                           c5 7f 00 00 (11000101 01111111 00000000 00000000) (32709)
      8     4        (object header)                           95 dd 00 f8 (10010101 11011101 00000000 11111000) (-134161003)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total

18:42:41.124 [Thread-1] DEBUG com.thread.ObjectHeader - 加锁后===========
18:42:41.125 [Thread-1] DEBUG com.thread.ObjectHeader - com.thread.Person object internals:
 OFFSET  SIZE   TYPE DESCRIPTION                               VALUE
      0     4        (object header)                           ea f5 01 6f (11101010 11110101 00000001 01101111) (1862399466)
      4     4        (object header)                           c5 7f 00 00 (11000101 01111111 00000000 00000000) (32709)
      8     4        (object header)                           95 dd 00 f8 (10010101 11011101 00000000 11111000) (-134161003)
     12     4        (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

可以看见，输出中的ea对应的二进制11101010的最后2位确实是10

以ea对应的二进制11101010中进行说明：

- 第一个0未使用，
- 后面4个0代表对象分代年龄
- 后一个0 代表是否可以偏向
- 最后2位代表是否有锁

此致，所有锁对应的对象头验证完毕。

# [](#结论)结论

synchronized 加锁时修改了对象头中的信息。synchronized锁分为偏向锁、轻量锁、重量锁3种类别。
