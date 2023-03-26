---
title: 08synchronized线程安全问题
toc: true
tags: springboot
categories: 
    - [java]
    - [concurrence]
---

在前几讲中，我们重点介绍了 JDK 中的 Thread 类，并且了解了 Java 中是如何使用线程来执行任务的，以及如何正常地去停止线程。通常在多线程执行的过程中，我们需要考虑一些线程安全的问题，而线程安全问题中最常用的解决策略之一就是 “锁”。

加锁的本质，就是为了解决在多线程场景中对于共享数据访问的安全问题，这类问题通常会被我们称之为线程安全问题。当我们提及到“锁”这个关键字的时候，就不得不了解下 synchronized 了。

在 JDK 的发展史中，synchronized 可谓是解决线程安全问题方面的“资深专家”了，它从 JDK1.0 版本开始就已经存在，一直到今天依旧被很多程序员们使用。那么本节课中，就让我们一同通过各种实战案例去深入认识下 synchronized 的底层原理吧。

### 案例分析

假设有一个模拟扣减库存的程序，这块的相关程序设计如下所示：

<!--more-->


```java
public class StockNumSale {

    //车票剩余数目
    private int stockNum;


    public StockNumSale(int stockNum) {
        this.stockNum = stockNum;
    }


    private boolean isStockEnough() {
        return stockNum > 0;
    }


    /**
     * 锁定库存
     *
     * @return 是否锁定成功
     */
    private boolean lockStock(int num) {
        if (!isStockEnough()) {
            return false;
        }
        for (int i = 0; i < num; i++) {
            stockNum--;
        }
        return true;
    }

    public void printStockNum() {
        if (this.stockNum < 0) {
            System.out.println("库存不足：" + this.stockNum);
        }
    }

    public static void batchTest(int threadNum, int stockNum) {
        CountDownLatch begin = new CountDownLatch(1);
        CountDownLatch end = new CountDownLatch(threadNum);
        StockNumSale stockNumSale = new StockNumSale(stockNum);
        for (int i = 0; i < threadNum; i++) {
            Thread t = new Thread(new Runnable() {
                @Override
                public void run() {
                    try {
                        //等待，模拟并发
                        begin.await();
                        stockNumSale.lockStock(100);
                        end.countDown();
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }
            });
            t.start();
        }
        try {
            begin.countDown();
            end.await();
            stockNumSale.printStockNum();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) {
        for (int i = 0; i < 10; i++) {
            batchTest(200, 1000);
        }
    }

}
```

这段代码的逻辑非常简单，模拟了 200 个线程并发去抢购 1000 个商品，每次批量购买 100 件商品，很明显库存数是不足的，预期当库存被扣减为 0 的时候，就不允许再有线程执行扣减的操作。但是实际的程序运作结果却很容易出现库存不足的情况，并且还会在控制台输出中看到以下内容：

```ini
库存不足：-2
```

而导致这个问题的关键点就在于 stockNum 变量同时被多个线程访问，但是没有去考虑它的线程安全问题。如果用 synchronized 关键字去解决该问题的话，可以对 lockStock 函数进行些许的调整，例如下边所示：

```java
public boolean lockStock(int num) {
    synchronized(this){
        if (!isStockEnough()) {
            return false;
        }
        for (int i = 0; i < num; i++) {
            stockNum--;
        }
        return true;
    }
}
```

在 lockStock 方法内部的代码块加入了一把 synchronized 锁之后，由于该锁所锁住的对象是 this 对象，且多线程下访问的 this 对象均为同一个 StockNumSale 实例，因此当有多个线程尝试执行lockStock 函数时，都需要先去抢夺同一个锁，如果抢夺失败则会进入同步队列中，从而保证了线程安全性。

还记得我们在第二节课中介绍的临界区概念吗，上边的这段代码中，被 synchronized 关键字所包裹的整个代码块内就属于是一个临界区内了。

为什么加入 synchronized 关键字之后，整个方法就具有线程安全性了呢？下边让我们来一起深入了解下 synchronized 关键字的底层原理。

### 1.synchronized 的底层原理

我们先尝试**在字节码层面**去观察它的变化。首先通过 javac 命令将该 Java 程序转换为 class 字节码，接着再使用 javap -c 的指令去将 class 文件转换为字节码文件，然后查看关键的 lockStock 函数部分，会看到大概如下所示的内容：

```assembly
 public boolean lockStock(int);                                                                                                                                                                   
    Code:                                                                                                                                                                                          
       0: aload_0                                                                                                                                                                                  
       1: dup                                                                                                                                                                                      
       2: astore_2                                                                                                                                                                                 
       3: monitorenter                      //管程进入点                                                                                                                                                       
       4: aload_0                                                                                                                                                                                  
       5: invokespecial #3                  // Method isStockEnough:()Z                                                                                                                            
       8: ifne          15                                                                                                                                                                         
      11: iconst_0                                                                                                                                                                                 
      12: aload_2                                                                                                                                                                                  
      13: monitorexit                       //管程退出点1                                                                                                                                                       
      14: ireturn                                                                                                                                                                                  
      15: iconst_0                                                                                                                                                                                 
      16: istore_3                                                                                                                                                                                 
      17: iload_3                                                                                                                                                                                  
      18: iload_1                                                                                                                                                                                  
      19: if_icmpge     38                                                                                                                                                                         
      22: aload_0                                                                                                                                                                                  
      23: dup                                                                                                                                                                                      
      24: getfield      #2                  // Field stockNum:I                                                                                                                                    
      27: iconst_1                                                                                                                                                                                 
      28: isub                                                                                                                                                                                     
      29: putfield      #2                  // Field stockNum:I                                                                                                                                    
      32: iinc          3, 1                                                                                                                                                                       
      35: goto          17                                                                                                                                                                         
      38: iconst_1                                                                                                                                                                                 
      39: aload_2                                                                                                                                                                                  
      40: monitorexit                       //管程退出点2                                                                                                                                                          
      41: ireturn                                                                                                                                                                                  
      42: astore        4                                                                                                                                                                          
      44: aload_2                                                                                                                                                                                  
      45: monitorexit                       //管程退出点3                                                                                                                                                         
      46: aload         4                                                                                                                                                                          
      48: athrow                                                                                                                                                                                   
    Exception table:                                                                                                                                                                               
       from    to  target type                                                                                                                                                                     
           4    14    42   any                                                                                                                                                                     
          15    41    42   any                                                                                                                                                                     
          42    46    42   any    
```

从字节码层面，我们可以看到，在 lockStock 函数的内部，存在着 monitorenter 和 monitorexit 两条指令，这两条指令中的 monitor 关键字其实就可以理解为是“管程”的意思。

管程将指定的临界区封装起来，然后所有需要访问这个临界区的线程需要通过排队的方式来请求。

![](./img/202303/thread8-1.png)

在第一条 monitorenter 和最后一条 monitorexit 之间的程序代码，意味着都会受到操作系统层面的管程协助，从而实现线程安全的效果。

那么为什么会有多个 monitorexit 的情况发生呢？这里我稍微解释下各个 monitorexit 对应的作用。

- **管程退出点1**：代表着当前程序刚从 isStockEnough 方法中执行结束，需要执行一次退出管程操作。
- **管程退出点2**：代表着当前程序刚从 lockStock 方法中执行结束，需要执行一次退出管程操作。
- **管程退出点3**：防止 lockStock 方法执行了一半，如果出现了异常，则需要有个兜底的退出策略，所以在字节码层面多加了一条 monitorexit 指令。

从字节码层面来看，目前我们只是看到了 monitorenter 和 monitorexit，并没有发现过多的信息，所以下边我们继续深入了解其在 OpenJdk 中的原理。

### 2.synchronized 在 OpenJdk 中的实现

如果想要突破字节码了解 synchronized 的话，可以从 openJdk 的源代码入手分析。下边我们一起来到更加深入的层面去了解 synchronized 关键字。

为了能更好地了解 synchronized 关键字，我们需要有一份 openJdk 的源代码，下边是我在 github 中搜索出来的源代码地址，有需要的同学可以收藏一波：[源代码地址](https://github.com/JetBrains/jdk8u_hotspot)。

在 OpenJDK 的源码里面有个叫做 ObjectMonitor.hpp 的文件，这里面定义了管程的一些细节要点。代码的地址：[地址](https://github.com/JetBrains/jdk8u_hotspot/blob/master/src/share/vm/runtime/objectMonitor.hpp)。

其内部对于 ObjectMonitor 的定义如下所示：

```assembly
    ObjectMonitor() {
    _header       = NULL;
    _count        = 0;
    _waiters      = 0,
    _recursions   = 0;
    _object       = NULL;  
    _owner        = NULL; //这是指持有当前 objectMonitor 的线程(通常每个线程对应一个 objectMonitor)
    _WaitSet      = NULL;  //进入到 wait 状态的线程队列
    _WaitSetLock  = 0 ;
    _Responsible  = NULL ;
    _succ         = NULL ;
    _cxq          = NULL ;
    FreeNext      = NULL ;
    _EntryList    = NULL ;  //进入等待 monitor 的线程队列
    _SpinFreq     = 0 ;
    _SpinClock    = 0 ;
    OwnerIsThread = 0 ;
  }
```

在 ObjectMonitor 源代码中，还存在着 WaitSet 和 EntryList 的定义，具体如下所示：

```assembly
private:
protected:
  ObjectWaiter * volatile _WaitSet; // LL of threads wait()ing on the monitor
 
protected:
  ObjectWaiter * volatile _EntryList ;     // Threads blocked on entry or reentry.
```

从 waitSet 和 entryList 变量的定义中可以发现，它们的本身其实是一条双向链表结构，对应了一个叫做 ObjectWaiter 的对象。

ObjectWaiter 对象在 hpp 文件中也是可以发现其具体的定义，其具体的源代码如下，通过对 ObjectWaiter 的源码阅读分析，可以看出它是组织成一条双向链表的核心组成元素。

```assembly
  
 //双向链表结构 
class ObjectWaiter : public StackObj {
 public:
  enum TStates { TS_UNDEF, TS_READY, TS_RUN, TS_WAIT, TS_ENTER, TS_CXQ } ;
  enum Sorted  { PREPEND, APPEND, SORTED } ;
  ObjectWaiter * volatile _next; //前指针
  ObjectWaiter * volatile _prev; //后指针
  Thread*       _thread;  //当前线程
  jlong         _notifier_tid;
  ParkEvent *   _event;
  volatile int  _notified ;
  volatile TStates TState ;
  Sorted        _Sorted ;           // List placement disposition
  bool          _active ;           // Contention monitoring is enabled
 public:
  ObjectWaiter(Thread* thread);

  void wait_reenter_begin(ObjectMonitor *mon);
  void wait_reenter_end(ObjectMonitor *mon);
};
```

在 ObjectMonitor 中，还存在一个 _owner 变量，这个变量可以简单理解为是一个指针类型，这一点可以通过阅读 hpp 文件中的注释了解到：

```assembly
 protected:                         // protected for jvmtiRawMonitor
     void *  volatile _owner;          // pointer to owning thread OR BasicLock
```

现在我们大概知道了 entryList、waitSet 的数据结构类型，以及 owner 变量的含义，那么它们在 ObjectMonitor 中的具体分工合作又是怎么一个流程呢？

为了方便大家的理解，我将相关的设计通过绘图的方式和大家展示了出来，主要划分为了以下几个模块： entryList、owner、waitSet ，请见下图：

![](./img/202303/thread8-2.png)

被锁定的资源会被 owner 进行监管，尝试获取资源的线程会通过 cas 的方式直接去将 owner 指针指向自身，如果失败，则会将当前线程挂起并且放入到 entryList 中（其实本质是先放入到一个_cxq队列中，然后在一定时机才会被放入到entryList中，这里我们为了简单理解，将它统一称为entryList），当 owner 监管的资源被之前所占领的线程释放了之后，entryList 中原先处于挂起状态的线程就会进行抢夺。

另外还有一个叫做 waitSet 的模块，该模块主要是用于存储那些在临界区内调用了 wait 函数的线程，这部分线程在调用了 wait 函数之后，会“释放掉”在 owner 所监管的资源权限（从宏观来看就是释放当前锁），并且将线程状态调整为等待，然后存放在 waitSet 区域，不再占用 owner 区，直到有其他线程调用了 notify 或者 notifyAll 函数之后，它们才会被唤醒参与抢夺。

图中的 thread4，thread5 表示了多个线程对 owner 区域的资源进行竞争时，实际上会通过 CAS 的方式判断 owner 内部是否有其他线程占有，如果为空，则把 owner 指向为当前自己（当然这里面会有面临 ABA 的问题需要考虑）。获取到 owner 成功了之后，_recursions 值会自增加 1。

上边我给出了一份 C++ 的头文件，这里我给出一份 cpp 文件，方便大家对底层源码更加深入的分析：[cpp 文件](https://github.com/JetBrains/jdk8u_hotspot/blob/master/src/share/vm/runtime/objectMonitor.cpp)。

设置 owner 的代码内容大致如下：

```assembly
bool ObjectMonitor::try_enter(Thread* THREAD) {
 //判断 owner 是否为线程自己
  if (THREAD != _owner) {
   //是否将 owner 成功锁定为自己
    if (THREAD->is_lock_owned ((address)_owner)) {
       assert(_recursions == 0, "internal state error");
       _owner = THREAD ;
       _recursions = 1 ;
       OwnerIsThread = 1 ;
       return true;
    }
    //锁定失败，再次通过 CAS 进行锁定操作
    if (Atomic::cmpxchg_ptr (THREAD, &_owner, NULL) != NULL) {
      return false;
    }
    return true;
  } else {
  //当前 owner 是自己，锁的重入次数加一
    _recursions++;
    return true;
  }
}
```

从这单源代码中可以看见，其实在 synchronized 的底层设置了 _recursions 变量，该变量可以实现锁的一个重入特性，而且底层对于锁定的动作会采取 cas 的方式进程尝试。

Atomic::cmpxchg_ptr 的调用，其实是使用了 Atomic 类中的 cmpxchg 方法，这个函数的底层是使用了汇编指令的 cmpxchg 进行运作的，其具体含义可以理解为是一次硬件层面的 CAS 操作。

通过梳理 synchronized 的底层原理，我们发现使用了 synchronized 关键字之后，在字节码层面上，在加锁的前后会有 monitorenter 和 monitoerexit 指令保护。

在汇编层面来看的话，操作系统会默认给锁定的对象关联一个 monitor 对象，这个对象就是 ObjectMonitor.hpp 文件中定义的那个类，它的内部存在一个 owner 指针，用于指向当前获取到锁资源的线程，同时还有个 recursions字段用于记录锁的重入次数。对于抢夺锁没有成功的线程，会被放入到 entryList 队列中等待，而获取到了锁之后再调用 wait 函数的线程，会主动释放锁并且进入到 waitSet 集合中休息。

当然，加入了 synchronized 关键字之后，也并不是会立马就“惊动”到操作系统层面上，毕竟从用户态发起对内核态的调用是一件开销比较大的事情，所以 JDK 的开发者在 JDK1.6 之后引入了锁升级的概念。

### 3.synchronized 的锁升级

要想了解锁升级的过程，我们需要提前先了解下什么是对象头。下边我将基于 HotSpot 虚拟机的模型来进行原理讲解。

当我们使用 Java 程序 new 了一个对象之后，该对象通常都会存在于堆内存中（内存逃逸情况除外），我用下边的一张图来对此时对象的一个内存布局进行演示：

![](./img/202303/thread8-3.png)

关于一个对象的内存布局中是如何分布的，我么可以通过 jol-core 小工具去进行查看，例如下边这段案例代码：

```xml
<dependency>
  <groupId>org.openjdk.jol</groupId>
  <artifactId>jol-core</artifactId>
  <version>0.16</version>
</dependency>
```



```java
public class MarkWordDemo_1 {

    public static void main(String[] args) {
        Object o = new Object();
        //查看该对象的内存布局
        System.out.println(ClassLayout.parseInstance(o).toPrintable());
    }
}
```

通过调用对应的 API 接口，可以在控制台打印出对象的内存布局情况：

![](./img/202303/thread8-4.png)

#### **对象头部存储了什么**

在常规对象的分布图里，Header 存储着重要的数据信息，下边是一张关于 Header 存储数据信息的布局图：

![](./img/202303/thread8-5.png)

可以看到 Header 中存储着各种和对象本身有关联的运行时数据，例如：hashcode、对象的分带年龄，而且似乎随着锁状态的变化，内部存储的信息也在发生变动，通常业界也会把这部分数据称之为 Mark Word。

在对对象头部有了基本的认识之后，下边我们根据不同的锁状态对锁升级进行深入探讨。

#### **无锁状态**

首先来说说无锁。当一个对象没有被多个线程锁访问的时候，它是不存在数据竞争问题的，此时处于“无锁”状态。

那么这个时候，该对象内部的 Mark Word 布局会如下所示：

![](./img/202303/thread8-6.png)

#### **偏向锁状态**

当出现了多个线程同时访问同一被加锁的对象的时候，会先进入到偏向锁阶段。

![](./img/202303/thread8-7.png)

其实偏向锁的设计本意是为了减少锁在多线程竞争下对机器性能的消耗。具体的方式是：当一个线程访问到 monitor 对象的时候，会在 Mark Word 里记录请求的线程 id，并且将偏向锁 id 进行标记。

这样的好处在于下一次有请求线程访问的时候，只需要读取该 monitor 的线程 id 是否和请求线程的 id 一致即可。这里需要注意一点，偏向锁在进行加锁的时候是通过 CAS 操作来修改 Mark Word 的，但是一旦出现了多个线程同时访问同个 monitor 的时候，偏向锁就会进入撤销状态。

进入撤销状态之前，会做一个全局性的检测，判断当前时间点里是否有其他的字节码在执行，如果没有则会进入撤销状态。撤销过程的相关细节点如下所示：

![](./img/202303/thread8-8.png)

如果一旦出现了多个线程竞争偏向锁，那么此时偏向锁就会进行撤销然后进入一个锁升级的步骤，进入到了轻量级锁的环节中。

#### **轻量级锁状态**

进入了轻量级锁状态之后，原先对象头内部的那些线程 id、epoch、分代年龄会在一个叫做 Lock Record 的位置上存储着。

![](./img/202303/thread8-9.png)

当多个竞争的线程抢夺该 monitor 的时候，会采用 CAS 的方式，当抢夺次数超过 10 次，或者当前 CPU 资源占用大于 50% 的时候，该锁就会从轻量级锁的状态上升为了重量级锁。

#### **重量级锁状态**

在之前所说的轻量级锁中，都是基于 JVM 层面的，相比于介入内核态的操作来说是属于轻量化的操作。但是这里我们需要先弄清楚一点：**并非说一直采用 CAS 的轻量级锁就一定会比重量级锁的性能要好** **。**

假设有十万个线程都在执行 CAS 操作，那么此时对于 CPU 的开销会是非常巨大的，这种场景下可以通过借助 OS 内核态的排队机制来做优化，因此轻量级锁在某种程度上晋升为重量级锁也是一种优化的手段。而重量级锁的状态下，对象头部的基本结构如下所示：

![](./img/202303/thread8-10.png)

进入到重量级锁的层面的话，具体的抢夺就需要靠操作系统的内核层面去处理了。

### 课后思考

**上节课答疑**

上节课的末尾，我们留下了一道思考题，如何避免多线程中死锁？下边给出一些我的思考：

首先，多线程中的死锁是指两个或两个以上的线程在执行过程中，因争夺资源而造成的一种互相等待的现象，若无外力作用，它们都将无法推进下去。这是一个严重的问题，因为死锁会让你的程序挂起无法完成任务，死锁的发生必须满足以下四个条件：

- 互斥条件：一个资源每次只能被一个进程使用。
- 请求与保持条件：一个进程因请求资源而阻塞时，对已获得的资源保持不放。
- 不剥夺条件：进程已获得的资源，在末使用完之前，不能强行剥夺。
- 循环等待条件：若干进程之间形成一种头尾相接的循环等待资源关系。

所以，避免死锁最简单的方法就是阻止循环等待条件，将系统中所有的资源设置标志位、排序，规定所有的进程申请资源必须以一定的顺序（升序或降序）做操作来避免死锁。

**本节课思考**

好了，通过本章节的学习，我们对于 synchronized 关键字的底层有了一定的了解，未抢夺到锁的线程会先被放入到_cxq的队列中，然后接着才会被丢入到entryList里面，那么_cxq中的线程是如何被放入到entryList中的呢，这里面有哪些策略进行实现呢？欢迎小伙伴们在评论区进行讨论。



 

 

 

