---
title: volatile关键字浅析
date: 2022-10-29 11:23:06
tags: Java 多线程
categories: Java多线程
---

volatile 是Java语音提供的一种非常非常轻量级的同步机制。在JDK的源码中，被大量使用。本文将大致介绍volatile底层实现机制。通常用在一个线程写入，而其他线程读取的场景。

## 理论

CPU设计了L1、L2等缓存用来屏蔽CPU运算速度与内存读写上的差异。是同时，CPU为了提升运行速度，对一些指令做了重排序。比如有如下代码：

```text
int a = 0;
int b = 0;
a = 10;
```

CPU 在运算上述代码比如a=0时，需要先load a,然后将0赋值给a后，再将a写回去（写到缓存或者内存上）。如果CPU将a=10放到b=0之前，那么可以省掉再次load等操作。

Java语言中，为了充分利用CPU的高性能，又能屏蔽各种CPU以及JVM 之间的差异，从而定义了JMM的规范。

在JMM规范中，volatile 通过内存屏障的方式实现了轻量级的同步机制。具体来说，就是storeload屏障。

## 示例说明

## JVM级别的指令

首先，来看一段简单的代码：

```text
package com.syn;

/**
 * @author tyb
 * @Description
 * @create 2022-10-26 10:52
 */
public class VolatileTest {
    static int a = 0;

    public static void main(String[] args) {
        a = 10;
    }
}
```

可能有人会说，通过javap -c查看JVM指令是不是就可以看出端倪了呢？

没有volatile关键字的场景下：

```text
$ javap -c VolatileTest.class
Compiled from "VolatileTest.java"
public class com.syn.VolatileTest {
  static int a;

  public com.syn.VolatileTest();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: bipush        10
       2: putstatic     #2                  // Field a:I
       5: return

  static {};
    Code:
       0: iconst_0
       1: putstatic     #2                  // Field a:I
       4: return
}
```

有volatile关键字的场景下：

```text
$ javap -c VolatileTest.class
Compiled from "VolatileTest.java"
public class com.syn.VolatileTest {
  static volatile int a;

  public com.syn.VolatileTest();
    Code:
       0: aload_0
       1: invokespecial #1                  // Method java/lang/Object."<init>":()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: bipush        10
       2: putstatic     #2                  // Field a:I
       5: return

  static {};
    Code:
       0: iconst_0
       1: putstatic     #2                  // Field a:I
       4: return
}
```

从上面2次的JVM级别的指令中，完全是一直的。

实际上，我们需要通过hsdis查看生产的汇编代码。

## 汇编指令

要查看Java的汇编指令，我们需要通过工具hsdis.dll(windows 环境)。 可以下载一个hsdis-amd64.dll放置JAVA_HOME下的\jre\bin\server目录中。 在jvm 参数中指定如下：

```text
-server
-Xcomp
-XX:+UnlockDiagnosticVMOptions
-XX:+PrintAssembly
-XX:CompileCommand=compileonly,*VolatileTest.main
```

> 参数说明：参数 -Xcomp 是让虚拟机以编译模式执行代码，这样代码可以偷懒，不需要执行足够次数来预热都能触发 JIT 编译。两个 -XX:CompileCommand 意思是让编译器仅仅编译，-XX:+PrintAssembly 就是输出反汇编内容

在输出中，我们可找到lock addl相关的输出（截取部分）：

```text
0x000000000384c1a8: sub    $0x30,%rsp         ;*bipush
                                                ; - com.syn.VolatileTest::main@0 (line 12)

  0x000000000384c1ac: movabs $0x7162a06e8,%rsi  ;   {oop(a 'java/lang/Class' = 'com/syn/VolatileTest')}
  0x000000000384c1b6: mov    $0xa,%edi
  0x000000000384c1bb: mov    %edi,0x68(%rsi)
  0x000000000384c1be: lock addl $0x0,(%rsp)     ;*putstatic a
                                                ; - com.syn.VolatileTest::main@2 (line 12)可以用JIT工具查看，可视化效果更好
```

用JIT工具查看如图：

![](/images/volatile.png)

# JVM实现 与storeload

在查看Java的汇编指令之前，有一点需要指出的是：在Java中，对static变量的操作是通过pustatic来完成的。

## putstatic

### Operation

Set static field in class

感兴趣的同学可以参见官方文档：[https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-6.html#jvms-6.5.putstatic](https://link.zhihu.com/?target=https%3A//docs.oracle.com/javase/specs/jvms/se7/html/jvms-6.html%23jvms-6.5.putstatic)

在JVM 的字节码解析器bytecodeInterpreter.cpp文件中，我们看到对putstatic的处理：

```text
CASE(_putstatic):
{

.....
if (cache->is_volatile()) {
            if (tos_type == itos) {
              obj->release_int_field_put(field_offset, STACK_INT(-1));
            } else if (tos_type == atos) {
              VERIFY_OOP(STACK_OBJECT(-1));
              obj->release_obj_field_put(field_offset, STACK_OBJECT(-1));
            } else if (tos_type == btos) {
              obj->release_byte_field_put(field_offset, STACK_INT(-1));
            } else if (tos_type == ztos) {
              int bool_field = STACK_INT(-1);  // only store LSB
              obj->release_byte_field_put(field_offset, (bool_field & 1));
            } else if (tos_type == ltos) {
              obj->release_long_field_put(field_offset, STACK_LONG(-1));
            } else if (tos_type == ctos) {
              obj->release_char_field_put(field_offset, STACK_INT(-1));
            } else if (tos_type == stos) {
              obj->release_short_field_put(field_offset, STACK_INT(-1));
            } else if (tos_type == ftos) {
              obj->release_float_field_put(field_offset, STACK_FLOAT(-1));
            } else {
              obj->release_double_field_put(field_offset, STACK_DOUBLE(-1));
            }
            OrderAccess::storeload();
}
```

通过cache->is_volatile()可以看到如果变量被volatile修饰，那么最后会通过OrderAccess::storeload()来处理。

OrderAccess有很多种不同平台上的实现。看下Linux上的实现：

在Linux的orderAccess_linux_x86.hpp文件中：

```text
inline void OrderAccess::storeload()  { fence();            }

inline void OrderAccess::fence() {
   // always use locked addl since mfence is sometimes expensive
#ifdef AMD64
  __asm__ volatile ("lock; addl $0,0(%%rsp)" : : : "cc", "memory");
#else
  __asm__ volatile ("lock; addl $0,0(%%esp)" : : : "cc", "memory");
#endif
  compiler_barrier();
}
```

在Windows下（orderAccess_windows_x86.hpp文件中）：

```text
inline void OrderAccess::storeload()  { fence(); }

inline void OrderAccess::acquire()    { compiler_barrier(); }
inline void OrderAccess::release()    { compiler_barrier(); }

inline void OrderAccess::fence() {
#ifdef AMD64
  StubRoutines_fence();
#else
  __asm {
    lock add dword ptr [esp], 0;
  }
#endif // AMD64
  compiler_barrier();
}
```

从上可以看到，通过汇编级别的指令，通过lock addl操作 锁住总线的方式实现内存屏障。

> 在IA32 手册5.20 SYSTEM INSTRUCTIONS 部分，lock LOCK (prefix) Lock Bus. 下载地址：[https://www.intel.cn/content/www/cn/zh/architecture-and-technology/64-ia-32-architectures-software-developer-vol-1-manual.html](https://link.zhihu.com/?target=https%3A//www.intel.cn/content/www/cn/zh/architecture-and-technology/64-ia-32-architectures-software-developer-vol-1-manual.html)

## 结论：

- volatile实现通过锁总线的方式实现同步。但volatile只是一种轻量的同步机制。
- 虽然volatile可以实现轻量级的同步机制，但是由于对volatitle修饰的变量的操作不是原子的，所以它不能保证原子性

参考：

 [JVM执行篇：使用HSDIS插件分析JVM代码执行细节_Java_周志明_InfoQ精选文章](https://www.infoq.cn/article/zzm-java-hsdis-jvm)

JIT 工具 [如何在windows平台下使用hsdis与jitwatch查看JIT后的汇编码 - stevenczp - 博客园](https://www.cnblogs.com/stevenczp/p/7975776.html) [GitHub - AdoptOpenJDK/jitwatch: Log analyser / visualiser for Java HotSpot JIT compiler. Inspect inlining decisions, hot methods, bytecode, and assembly. View results in the JavaFX user interface.](https://github.com/AdoptOpenJDK/jitwatch)
