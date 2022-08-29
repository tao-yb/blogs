---
title: 指针压缩CompressedOops
date: 2022-08-19 20:40:33
tags:
---

在介绍指针压缩前，先说几个对应的名称。

# [](#对象头)对象头

众所周知，每个Java对象都有一个对象头。对象头存放着对象在元空间中的地址等等信息。

# [](#名称解释)名称解释：

oop (ordinary object pointer) 即普通对象指针，是JVM 中用于引用对象的句柄。可以理解，它就是在对象头中的。

在堆中，32位的对象引用占4个字节，而64位的对象引用占8个字节。也就是说，64位的对象引用大小是32位的2倍。

# [](#指针压缩)指针压缩

在64位的JVM中，由于对象引用占用了8个字节。这同时意味着存在几个问题：

- 在JVM中的对象会比32位系统占用更多的堆内存，这样会增加GC的压力
- 在相同大小的元空间中，64位的JVM只能存储较少的对象元数据信息
- 降低CPU缓存命中率。64位对象应用的增加，导致CPU只能缓存较少的oop信息，降低了CPU缓存的效率

如果64位机器上，oop与32位机器上一致。那就没问题了。主要是这个问题如何解决呢？ 答案是指针压缩。

> 指针压缩使得不同的JVM上对象占用空间大小都一样。并且在Jdk8中，指针压缩默认是开启的（网上资料介绍Jdk7中也是默认开启的）。

# [](#开启指针压缩到底哪会不一样呢)开启指针压缩到底哪会不一样呢？

在JVM中，通过下面设置，可查看指针压缩的工作模式

```
-XX:+UnlockDiagnosticVMOptions -XX:+PrintCompressedOopsMode
```

# [](#查看结果对比)查看结果对比

```
public class Person {
    double age;

}


@Slf4j(topic = "test")
public class Test {
    static Person p = new Person();

    public static void main(String[] args) {

        log.debug(
        ClassLayout.parseInstance(p).toPrintable());
    }
}
```

## [](#开启指针压缩模式下的结果)开启指针压缩模式下的结果：

```
heap address: 0x00000005c2200000, size: 8158 MB, Compressed Oops mode: Zero based, Oop shift amount: 3

Narrow klass base: 0x0000000000000000, Narrow klass shift: 3
Compressed class space size: 1073741824 Address: 0x00000007c0000000 Req Addr: 0x00000007c0000000
15:57:54.480 [main] DEBUG test[36] - 4.0
15:57:54.482 [main] DEBUG test[36] - hashcode:66133adc
15:57:55.584 [main] DEBUG test[36] - com.syn.classlay.Person object internals:
OFF  SZ     TYPE DESCRIPTION               VALUE
  0   8          (object header: mark)     0x00000066133adc01 (hash: 0x66133adc; age: 0)
  8   4          (object header: class)    0xf8017a61
 12   4          (alignment/padding gap)   
 16   8   double Person.age                0.0
Instance size: 24 bytes
Space losses: 4 bytes internal + 0 bytes external = 4 bytes total
```

## [](#关闭指针压缩的场景)关闭指针压缩的场景：

铜鼓-XX:-UseCompressedOops 关闭指针压缩，输出如下：

```
OFF  SZ     TYPE DESCRIPTION               VALUE
  0   8          (object header: mark)     0x00000066133adc01 (hash: 0x66133adc; age: 0)
  8   8          (object header: class)    0x0000000025d4f1d0
 16   8   double Person.age                0.0
Instance size: 24 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total
```

# [](#结论)结论

从指针压缩关闭开启的场景下分析可以知道： 关闭指针压缩的时候，object header: class size=8,而关闭的时候，size=4

# [](#jvm-实现指针压缩的原理)jvm 实现指针压缩的原理

以下摘录自：[JVM之压缩指针（CompressedOops） - 掘金](https://juejin.cn/post/6844903768077647880)

在实现上，堆中的引用其实还是按照0x0、0x1、0x2...进行存储。只不过当引用被存入64位的寄存器时，JVM将其左移3位（相当于末尾添加3个0），例如0x0、0x1、0x2...分别被转换为0x0、0x8、0x10。而当从寄存器读出时，JVM又可以右移3位，丢弃末尾的0。（oop在堆中是32位，在寄存器中是35位，2的35次方=32G。也就是说，使用32位，来达到35位oop所能引用的堆内存空间）

[JVM之压缩指针（CompressedOops） - 掘金](https://juejin.cn/post/6844903768077647880#heading-12)
