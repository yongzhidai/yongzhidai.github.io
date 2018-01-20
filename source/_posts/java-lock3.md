---
title: 自己动手写把”锁”之---原子性操作
categories:
  - 技术
  - Java
tags:
  - cas
  - 原子性
date: 2018-01-06 16:05:34
---
所谓的原子性，就是在执行过程中不会被线程调度机制打断的操作，这种操作从开始就一直运行到结束，中间不存在任何上下文切换。
<!--more-->

还是以上篇讲到的x++操作为例。这是一个典型的‘读改写’的操作，在多线程的情况下，必须需要硬件的支持来保证‘读改写’的原子性，底层原理可以简单理解，通过锁总线的方式来实现。不过这里咱们不说硬件，咱们先研究下Java是如何原子性实现++操作的。

在Java中，如果要实现一个在多线程下正常工作的累加计数器，首先想到的就是并发包里的AtomicXXX类，如一下例子代码：


```
public class TestAtomic {
    private static AtomicInteger couter = new AtomicInteger(0);
    public static void main(String[] args)throws Exception {
        Thread t1 = new Thread(new Runnable() {
            public void run() {
                for(int i=0;i<1000;i++)
                    incr();
            }
        });
        Thread t2 = new Thread(new Runnable() {
            public void run() {
                for(int i=0;i<1000;i++)
                    incr();
            }
        });
        t1.start();
        t2.start();
        t1.join();
        t2.join();
        System.out.println(couter.get());
    }
    public static void incr(){
        couter.incrementAndGet();
    }
}
```

这里我们通过AtomicInteger实现累加器，两个线程各执行了一千次++操作，最后正常输出结果2000。

通过分析AtomicInteger的源码，我们可以发现，其内部用来保存具体数值的变量是这么定义的：

```
private volatile int value;
```

它通过volatile来实现了value在多线程之间的可见性，即线程A改变了value的值，线程B读取value时读到的是被修改后的值。

但是之前也说到了，volatile修饰的变量，仅通过++操作是无法实现原子性的，原因上篇说了这里就不多说了。

再来看看如果实现多线程间的原子性++操作，进入AtomicInteger的incrementAndGet方法，他通过调用Unsafe的getAndAddInt方法来实现：

```
public final int incrementAndGet() {
    return unsafe.getAndAddInt(this, valueOffset, 1) + 1;
}
```



Unsafe是Java提供用来访问系统底层的工具类，它大致有这几个能力：
- 直接分队释放堆外内存。Java的直接内存就是通过这个来实现。
- 线程的挂起和恢复。后边咱们要说的LockSupport就是通过这个实现。
- CAS操作。即Compare And Swap，简单地说就是比较并交换。在保证‘读改写’一致性上极其有用。它在写操作时会先比较当前内存里的值是否和改之前读的值是否一致，如果一致则修改成功，不一致则修改失败。

Unsafe在CAS操作一个变量时，用到了这个变量在类中的偏移位置。如AtomicInteger操作value变量时通过如下代码先得到valueOffset：

```
static {
    try {
        valueOffset = unsafe.objectFieldOffset
            (AtomicInteger.class.getDeclaredField("value"));
    } catch (Exception ex) { throw new Error(ex); }
}

```


进入到Unsafe的getAndAddInt方法：

```
int var5;
do {
    var5 = this.getIntVolatile(var1, var2);
} while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

return var5;
```

这里不断读取value变量的值，然后通过compareAndSwapInt操作，即CAS操作，将修改后的值写回去，直到修改成功退出循环。

说到这里应该把AtomicInteger实现原子性++的操作说清楚了。比较简单，总结起来就两点：
1. 通过volatile实现变量value的变更对线程可见
2. 通过Unsafe的CAS操作，避免了一个线程的修改覆盖另一个线程的修改，从而实现结果上的一致性。


这里我们不妨再看看Unsafe的comareAndSwapInt方法的实现，这个方法定义如下：

```
public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);
```

这是一个用native修饰的本地方法，通过openjdk的源码可以找到其本地实现代码：

![image](java-lok2/imag1.png)

这里可以看到，它是先计算出了要修改的变量地址，然后调用Atomic的cmpxchg方法实现cas操作。我们继续跟踪cmpxchg方法：

![image](java-lok2/imag2.png)

这是x86平台下的源码实现，可以看到它用了cmpxchgl汇编指令。也就是说，原子性操作是要硬件层面的支持。

