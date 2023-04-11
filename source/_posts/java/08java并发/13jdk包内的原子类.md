---
title: 13jdk包内的原子类
toc: true
tags: springboot
categories: 
    - [java]
    - [concurrence]
---

在前边的课程中，我们了解到 Java 内部提供了两种方式来解决线程安全问题，一种是加入**synchronized** 关键字，另一种则是使用 Lock 锁。虽然说这两种方式都能解决掉线程安全的问题，但是在某些场景下会稍微有些麻烦，例如下边这个场景，每次请求接口都会对 reqCount 做一次加一操作：


<!--more-->




```java
@RestController
@RequestMapping(value = "/test")
public class TestController {

    private int a = 10;
    private int b = 10;
    private int c = 10;

    @GetMapping(value = "/do-count")
    public void reduce() {
        synchronized (this) {
            if(a==0 || b==0 || c==0){
                System.out.println(a + "," + b + "," + c);
                return;
            }
            a--;
            b--;
            c--;
            System.out.println(a + "," + b + "," + c);
        }
    }
}
```

如果不希望 reqCount 出现统计失误的话，加入一把锁确实是一个合理的操作。但是除了这种方式之外，是否还有更加简单的方法来解决这种线程安全呢？

其实早在 JDK1.5 之后的 java.util.concurrent.atomic 包中就已经有相应的解决方式了。这个包中对各种常用的数据类型都提供了一个原子性操作的封装，能够保证在多线程环境下的数据一致性。下边就让我们一起深入了解下 JDK 包中的原子类吧。

### 1.JDK中的原子类介绍

我们可以将 JDK 中常见的原子类做个简单的分类，下边我们通过一系列的实战案例来认识下这几种原子类的使用方式。

#### **原子更新基本类型**

```java
    @Test
    public void testAtomicInteger() {
        AtomicInteger atomicInteger = new AtomicInteger(1);
        System.out.println(atomicInteger.addAndGet(1));
        System.out.println(atomicInteger.getAndAdd(1));
    }

    @Test
    public void testAtomicBoolean() {
        AtomicBoolean atomicBoolean = new AtomicBoolean(true);
        atomicBoolean.set(false);
        System.out.println(atomicBoolean.get());
    }
```

#### 原子更新数组

```java
@Test
public void testAtomicIntegerArray(){
    int[] value = new int[]{0, 1, 2};
    AtomicIntegerArray ata = new AtomicIntegerArray(value);
    //先指定数组的下标位，在指定需要增加的数值
    ata.addAndGet(0,1);
    System.out.println(ata);
}
```

#### 原子更新引用对象

```java
 @Test
    public void testAtomicReference() {
        AtomicReference<String> stringAtomicReference = new AtomicReference<>("idea");
        System.out.println(stringAtomicReference.compareAndSet("idea", "idea2"));
        System.out.println(stringAtomicReference.compareAndSet("idea2", "idea"));
        System.out.println(stringAtomicReference.compareAndSet("idea", "idea2"));
    }


    /**
     * 更新整个对象的类型
     */
    @Test
    public void updateAccount() {
        AtomicReference<Account> accountAtomicReference = new AtomicReference<>();
        Account account = new Account();
        account.setAge(1);
        account.setId(1);
        account.setName("idea");
        accountAtomicReference.set(account);
        Account account2 = new Account();
        account2.setAge(2);
        account2.setId(2);
        account2.setName("idea2");
        accountAtomicReference.compareAndSet(account, account2);
        System.out.println(accountAtomicReference.get().getAge());
        System.out.println(accountAtomicReference.get().getId());
        System.out.println(accountAtomicReference.get().getName());
    }

    @Data
    class Account {
        int id;
        String name;
        volatile int age;

    }
```

#### 原子更新引用对象字段

```java
 /**
* 更新指定对象的某个字段
*/
@Test
public void updateAccountField(){
    Account account = new Account();
    account.setAge(1);
    account.setId(1);
    account.setName("idea");
    //age字段一定要为volatile类型
    AtomicIntegerFieldUpdater<Account> accountAtomicReference = AtomicIntegerFieldUpdater.newUpdater(Account.class,"age");
    Account account2 = new Account();
    account2.setAge(2);
    account2.setId(2);
    account2.setName("idea2");
    //这里输出的数值中，age会加2
    System.out.println(accountAtomicReference.incrementAndGet(account2));

}
```

通过这些简单的案例，我们可以发现，原子类所提供的方法都是比较简单易懂的。但为什么说，简单调用原子类的接口对数据进行修改，可以预防线程安全问题呢？想要了解这个原理，我们就需要深入到底层，去查看原子类在对不同数据进行修改时的实现逻辑。

下边我们以 AtomicInteger.addAndGet 方法为例，来看看它在底层实现上是如何避免线程安全问题的。

点开该方法的源码实现部分，可以看到以下信息：

```java
public final int addAndGet(int delta) {
    //对应实现在下边
    return unsafe.getAndAddInt(this, valueOffset, delta) + delta;
}

//cas修改变量数值
public final int getAndAddInt(Object o, long offset, int delta) {
   int v;
   do {
       v = getIntVolatile(o, offset);
   } while (!compareAndSwapInt(o, offset, v, v + delta));
   return v;
}

public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);
```

在 JDK 的原子类内部，很多对于数据的修改，其实都利用到了 Unsafe 包的代码。在 sum.misc 包中的 Unsafe 类里面，提供了许多硬件级别的原子操作，可以支持类似于 C 的指针一样的功能，直接指定内存地址进行操作。但是这个包对于很多程序员来说，并不是太提倡使用，因为稍微一不注意，就有可能将其他内存地址的数据给修改了。

其实上边这段源代码展示的核心并不在于使用了 Unsafe，而是原子类对于无锁操作实现的设计思想。这里的 compareAndSwapInt 方法就是 CAS 的具体体现了。

**那么什么是CAS呢？为什么使用CAS就能避免线程安全问题呢？**

别急，下边听我一一道来。

### 2.深入理解 CAS

首先，让我们来看看 CAS 的全称是如何定义的。Compare And Swap，这个公式是 CAS 的全称，意思就是比较并且交换，这种方式也是无锁算法的一种思路，其思路如下所示：

在 compareAndSwap(V,E,N) 函数中，V 表示要被更新的变量值，E 表示预期值，N 表示期望更新的变量值。在调用 compareAndSwap 函数的时候会判断，是否当前变量的数值和 E 是相同的，如果相同则进行修改，否则就说明当前有其他线程修改该值，从而放弃本次修改操作。

通常 CAS 操作会结合一个循环一起使用，当尝试执行 cas 操作却修改失败之后，就会重新将 V 值读出来赋予给 E，然后接着再执行修改操作，直达最终修改成功为止。这个不断尝试修改的过程，我们一般称之为**自旋操作**。所以常见的 cas 结合循环重试的整体流程如下图所示：

![](./img/202303/thread12-6.png)

现在我们再回过头来看看 JDK 底层对于 CAS 操作的实现部分：

```java
//cas修改变量数值
public final int getAndAddInt(Object o, long offset, int delta) {
   int v;
   do {
       v = getIntVolatile(o, offset);
   } while (!compareAndSwapInt(o, offset, v, v + delta));
   return v;
}
```

通过源代码可以看到，JDK 的底层是通过循环尝试修改对象 o 的偏移处的值，最后判断当前的内存值是否是 v，是则修改，否则重新尝试。

而 compareAndSwapInt 函数其实本质是一个 native 函数，它在 openJdk 中的实现是调用了 CPU 的 cmpxchg 指令，具体的源代码体现在这个[地址](https://github.com/JetBrains/jdk8u_hotspot/blob/master/src/share/vm/runtime/atomic.cpp))可以看到。这条执行会比较寄存器中的 V 和 E 是否相同，并且根据实际情况做修改。

现在看来，似乎 CAS 操作还是挺合理的，通过底层调用了 CPU 的 cmpxchg 指令去直接对内存地址的数据做原子性修改，如果修改失败了还会进行重试，这相比之前的加锁机制要简单了许多。但是这种实现方式是否也存在一定缺陷呢？

答案是肯定的，CAS 也有一定的缺点，下边我列举了一些 CAS 操作存在的不足点。

**自旋导致CPU消耗升高**

当一个原子类在进行CAS操作过多的时候，会导致CPU一直被占用，一直消耗资源。通常这种情况发生在高并发场景下会比较多。

- **ABA问题**

CAS 需要在操作值的时候检查内存值是否发生变化，没有发生变化才会更新内存值。但是如果内存值原来是 A，后来变成了 B，然后又变成了 A，那么CAS进行检查时会发现值没有发生变化，但是实际上是有变化的。

JDK 从 1.5 开始提供了 AtomicStampedReference 类来解决 ABA 问题，具体操作封装在 compareAndSet() 中。

compareAndSet() 首先检查当前引用和当前标志与预期引用和预期标志是否相等，如果都相等，则以原子方式将引用值和标志的值设置为给定的更新值。

不过目前来说这个类比较”鸡肋”，大部分情况下 ABA 问题并不会影响程序并发的正确性，如果需要解决 ABA 问题，使用传统的互斥同步可能比原子类更加高效。

只能保证一个共享变量的原子操作。对一个共享变量执行操作时，CAS 能够保证原子操作，但是对多个共享变量操作时，CAS 是无法保证操作的原子性的。

### 课后小节

好了，通过本章节的学习，我们认识了JDK原子类中的常用代表类，以及对应的函数接口，并且通过对原子类的底层原理学习发现，其底层是通过调用 CPU 的 cmpxchg 指令来进行原子性的更新内存操作。

### 课后思考

**上节课答疑**

在这里，我将自己之前使用线程池所遇到过的一些问题做了一份列表，和大家分享下：

- 线程池没有做优雅关闭，导致部分运行到一半的任务被杀掉；
- 线程池执行submit的时候，内部出了异常，但是没有加入try-catch代码块，导致异常被吞；
- 线程池的队列长度配置过大，导致应用服务出现 oom；
- 线程池的内部希望获取到外部的线程变量，可以使用 TransmittableThreadLocal 来获取；
- 高并发场景下不适合使用线程池接收任务，适合使用 MQ 来替代。

**本节课思考**

最后我们留下一道思考题，在 JDK8 中，Doug Lea 大佬提供了几个新的代表类，其中就有包含DoubleAdder、LongAdder 这两个类，你知道这两个类的底层实现思路是怎样的吗？欢迎大家在评论区中展开讨论。



 

 

 

 

