---
title: 09java并发包中的ReentrantLock
toc: true
tags: springboot
categories: 
    - [java]
    - [concurrence]
---

前边一节中，我们介绍了 Java 中的 synchronized 关键字，了解到在面对一些线程安全问题时，需要加入一把锁来进行防范，而 synchronized 就是这方面的代表之一。可能你会有所疑惑，既然有了 synchronized 这把锁，为什么JDK中还会有 Lock 的定义呢？其实这个还和 synchronized 的历史背景有关。

在 JDK 早期版本中，synchronized 关键字性能确实不佳，每一次加锁都需要和操作系统内核打交道，所以后来 JDK 作者索性自己写了一个Lock接口，减少和内核态之间的交道，降低这种性能的损耗。

但是自从 JDK1.6 以后，JDK 作者对 synchronized 进行了优化之后，synchronized 又开始被推荐使用，何况性能问题通过优化，大多都是可以解决的，所以性能并不是`Lock`接口出现的原因。

其实 Lock 接口至今一直被大众所喜爱，**主要不在于说它的性能方面，而是它更加具备有灵活性**。这一点我们从 Lock 的接口定义上可以看到明显的效果，具体代码如下所示：

<!--more-->


```java
public interface Lock {

    void lock();

    void lockInterruptibly() throws InterruptedException;

    boolean tryLock();

    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;

    void unlock();

    Condition newCondition();

}
```

通过这些接口函数的定义，我们可以发现 Lock 要比 synchronized 更具备一些灵活性，尤其是在它的以下几个特性方面：

- 支持获取锁超时机制；
- 支持非阻塞方式获取锁；
- 支持可中断方式获取锁。

在JDK包中，Lock 实现类的代表莫过于是 ReentrantLock，下边我们重点以 ReentrantLock 为具体类进行深入介绍。

## 1.非阻塞方式获取锁

---

ReentrantLock 中提供了一个非阻塞方式获取锁的接口，名字叫 tryLock，其具体使用方式为：

```java
public void tryLockMethod() {
    if (reentrantLock.tryLock()) {
        try {
            i++;
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            reentrantLock.unlock();
        }
    } else {
        //todo
    }
}
```

 这种使用方式在获取锁时如果失败了，则不会继续等待，而是会立马返回一个布尔状态值。这一点相比于 synchronized 关键字而言要灵活些，synchronized 关键字在抢夺锁失败之后，只能够进入一个等待队列中，而 tryLock 可以迅速告知调用方结果，从而进入对应的程序分支中进行处理。

## 2.**锁超时机制**

Lock 支持锁超时机制，这个功能要比单纯的 lock 函数更加强大，当获取锁超过一定时间，便会主动退出等待，例如下边这段案例：

```java
public void tryLockMethod_2() {
    try {
        if (reentrantLock.tryLock(1, TimeUnit.SECONDS)) {
            try {
                i++;
            } catch (Exception e) {
                e.printStackTrace();
            } finally {
                reentrantLock.unlock();
            }
        } else {
            //todo
}
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
}
```

 在这段代码中，通过调用 tryLock 方法，当等待锁超过 3 秒便会自动退出抢夺。要注意，由于是调用了 tryLock 函数，所以程序不一定会获取锁，因此在 finally 模块中，如果不确定当前线程依然是持有锁的状态，那么就需要在释放前先调用 isLocked 函数进行判断。通常使用 tryLock 的时候需要注意可能会导致活锁发生，所以睡眠时间可以采用随机值，更合适一些。

## 3.对可中断方式获取锁

我认为 ReentrantLock.lockInterruptibly 是 Lock 接口定义中非常特别的一个函数，它在使用的时候基本和 Lock#lock 函数是相同效果，使用的案例代码如下所示：

```java
 public void lockInterruptiblyMethod() {
        try {
            reentrantLock.lockInterruptibly();
            i++;
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            if (reentrantLock.isLocked()) {
                reentrantLock.unlock();
            }
        }
}
```

lockInterruptibly 允许在等待时，由其它线程调用等待线程的 Thread.interrupt 方法来中断等待线程的等待，这一点和 Lock.lock 函数以及 synchronized 关键字是有所不同的。下边让我们通过一个实战案例来理解这几种方法在加锁过程中，遇到线程中断之后会有什么不同的表现。

```java
public class LockInterruptDemo {

    static class LockInterruptThread_1 implements Runnable {
        private ReentrantLock lock = new ReentrantLock();

        @Override
        public void run() {
            try {
                //外界可以中断这里面的处理
                lock.lockInterruptibly();
                try {
                    System.out.println("enter");
                    long startTime = System.currentTimeMillis();
                    for (; ; ) {
                        if (System.currentTimeMillis() - startTime >= 5000) {
                            break;
                        }
                    }
                    System.out.println("end");
                } catch (Exception e) {
                    e.printStackTrace();
                } finally {
                    if (lock.isLocked()) {
                        lock.unlock();
                    }
                }
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }


    static class LockInterruptThread_2 implements Runnable {
        @Override
        public void run() {
            synchronized (this) {
                System.out.println("enter");
                long startTime = System.currentTimeMillis();
                for (; ; ) {
                    if (System.currentTimeMillis() - startTime >= 5000) {
                        break;
                    }
                }
                System.out.println("end");
            }
        }
    }

    static class LockInterruptThread_3 implements Runnable {
        private ReentrantLock lock = new ReentrantLock();

        @Override
        public void run() {
            lock.lock();
            try {
                System.out.println("enter");
                long startTime = System.currentTimeMillis();
                for (; ; ) {
                    if (System.currentTimeMillis() - startTime >= 5000) {
                        break;
                    }
                }
                System.out.println("end");
            } catch (Exception e) {
                e.printStackTrace();
            } finally {
                if (lock.isLocked()) {
                    lock.unlock();
                }
            }
        }
    }


    @Test
    public void testLockInterruptThread_1() throws InterruptedException {
        LockInterruptThread_1 lockInterruptThread_1 = new LockInterruptThread_1();
        //使用ReentrantLock#lockInterruptibly 在没有获取到锁处于等待过程中是可以被中断的。
        Thread t1 = new Thread(lockInterruptThread_1);
        Thread t2 = new Thread(lockInterruptThread_1);

        t1.start();
        t2.start();

        Thread.sleep(200);
        System.out.println("开始触发中断操作");
        t2.interrupt();
        System.out.println("发起了中断操作");
    }


    @Test
    public void testLockInterruptThread_2() throws InterruptedException {
        LockInterruptThread_2 lockInterruptThread_2 = new LockInterruptThread_2();
        //使用synchronized关键字 在没有获取到锁处于等待过程中是无法随意中断的。
        Thread t1 = new Thread(lockInterruptThread_2);
        Thread t2 = new Thread(lockInterruptThread_2);

        t1.start();
        t2.start();

        Thread.sleep(200);
        System.out.println("开始触发中断操作");
        t2.interrupt();
        System.out.println("发起了中断操作");
    }

    @Test
    public void testLockInterruptThread_3() throws InterruptedException {
        LockInterruptThread_3 lockInterruptThread_3 = new LockInterruptThread_3();
        //使用synchronized关键字 在没有获取到锁处于等待过程中是无法随意中断的。
        Thread t1 = new Thread(lockInterruptThread_3);
        Thread t2 = new Thread(lockInterruptThread_3);

        t1.start();
        t2.start();

        Thread.sleep(200);
        System.out.println("开始触发中断操作");
        t2.interrupt();
        System.out.println("发起了中断操作");
    }
}
```

首先我们可以将这段代码执行一下，看看对应的效果。

下边是三种不同的测试用例在执行之后的效果：

```
enter
开始触发中断操作
发起了中断操作
java.lang.InterruptedException
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.doAcquireInterruptibly(AbstractQueuedSynchronizer.java:898)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.acquireInterruptibly(AbstractQueuedSynchronizer.java:1222)
	at java.util.concurrent.locks.ReentrantLock.lockInterruptibly(ReentrantLock.java:335)
```

```ini
enter
开始触发中断操作
发起了中断操作

```

```ini
enter
开始触发中断操作
发起了中断操作
```

通过实验结果可以发现，在多线程场景下，使用 **lock函数和synchronized** 关键字的代码块在尝试获取锁的过程中，如果获取失败了，就会进入等待队列中等待，而此时使用 **Thread.interupt** 是无法直接中断在等待状态中的线程的。

但是对于使用 **lockInterruptibly** 方法的线程来说，是可以在 **Thread.interupt** 调用之后立马进入到中断状态的。

## 课后小结

本章节中，我们重点介绍了 Lock 中的各种常用接口，以及它们在使用场景中存在的不同点，我做一些简单的归纳，主要如下所示：

- Lock 类中提供了 tryLock() 函数，这个方法支持线程以非阻塞性的方式去获取锁 ，如果获取失败则立马返回结果；
- Lock 类中提供了 tryLock(long timeout, TimeUnit unit) 函数，这个方法支持锁的超时机制，当获取锁的时候会先进入等待状态，当等待时间达到预期之后才会返回结果。
- lockInterruptibly 是一个支持线程中断的函数，当线程在执行该函数之后会进入等待状态，在等待的过程中是可以通过 Thread.interupt 函数去进行中断的。

## 课后思考

**上节课答疑**

在使用 synchronized 关键字的时候，如果线程没有抢夺到锁，那么就会被放入到一个 _cxq 的队列中，然后在一定时间点才会被转移到 entryList 中，这里的具体实现，其实要看 synchronized 内部的出队策略 QMode 值的设置。

QMode 一共有 5 种值，0、1、2、3、4，不同的 QMode ，会影响 _cxq 和 EntryList 的优先级，默认情况下，QMode 为 0。

 

- QMode=2，如果此时cxq队列不为空，则优先将 cxq 队列的头节点进行唤醒，然后才是去唤醒EntryList 的头节点。

- QMode=3，把 cxq 队列放置到 EntryList 的尾部。

- QMode=4，把 cxq 队列放置到 EntryList 的头部

- QMode=1，将 cxq 队列翻转，然后赋值给 EntryList，唤醒 EntryList 的头部。

- QMode=0，将 cxq 赋值给到 EntryList，并且转换为双向链表，唤醒 EntryList 的头部。

下边是在 OpenJDK 源代码中找到的主要实现逻辑，和大家分享下：

```ini
void ATTR ObjectMonitor::exit(bool not_suspended, TRAPS) {
   Thread * Self = THREAD ;if (THREAD != _owner) {if (THREAD->is_lock_owned((address) _owner)) { // _owner指向锁记录，如果锁记录是由参数线程在其栈桢上分配// Transmute _owner from a BasicLock pointer to a Thread address.// We don't need to hold _mutex for this transition.// Non-null to Non-null is safe as long as all readers can// tolerate either flavor.
       assert (_recursions == 0, "invariant") ;
       _owner = THREAD ; // 将_owner指向线程
       _recursions = 0 ; // 此时肯定不是重入
       OwnerIsThread = 1 ; // 当前_owner是线程指针} else { // 锁记录不是由参数线程在其栈桢上分配，其实是出错了// NOTE: we need to handle unbalanced monitor enter/exit// in native code by throwing an exception.// TODO: Throw an IllegalMonitorStateException ?TEVENT (Exit - Throw IMSX) ;assert(false, "Non-balanced monitor enter/exit!");if (false) {THROW(vmSymbols::java_lang_IllegalMonitorStateException());}return;}}if (_recursions != 0) { // 递归重入的情况，此时参数线程还持有监视器
     _recursions--;        // this is simple recursive enterTEVENT (Inflated exit - recursive) ;return ;}// Invariant: after setting Responsible=null an thread must execute// a MEMBAR or other serializing instruction before fetching EntryList|cxq.if ((SyncFlags & 4) == 0) {
      _Responsible = NULL ;}#if INCLUDE_TRACE// get the owner's thread id for the MonitorEnter event// if it is enabled and the thread isn't suspendedif (not_suspended && Tracing::is_event_enabled(TraceJavaMonitorEnterEvent)) {
     _previous_owner_tid = SharedRuntime::get_java_tid(Self);}
#endiffor (;;) {
      assert (THREAD == _owner, "invariant") ; // 此时参数线程还持有监视器// 省略一些代码，没太看懂这部分

      guarantee (_owner == THREAD, "invariant") ;

      ObjectWaiter * w = NULL ;
      int QMode = Knob_QMode ;// cxq链表和EntryList链表均不为空的情况，接下来的三个if很显然要求cxq链表不为空，EntryList链表不为空的要求在1188行if (QMode == 2 && _cxq != NULL) { // Knob_QMode是2意味着cxq链表的优先级比EntryList链表高，优先从cxq中唤醒第一个线程// QMode == 2 : cxq has precedence over EntryList.// Try to directly wake a successor from the cxq.// If successful, the successor will need to unlink itself from cxq.
          w = _cxq ;
          assert (w != NULL, "invariant") ;
          assert (w->TState == ObjectWaiter::TS_CXQ, "Invariant") ;
          ExitEpilog (Self, w) ;return ;}if (QMode == 3 && _cxq != NULL) { // Knob_QMode是3意味着需要将cxq链表中的元素链接到EntryList链表后面// Aggressively drain cxq into EntryList at the first opportunity.// This policy ensure that recently-run threads live at the head of EntryList.// Drain _cxq into EntryList - bulk transfer.// First, detach _cxq.// The following loop is tantamount to: w = swap (&cxq, NULL)
          w = _cxq ;for (;;) { // 使_cxq为NULL，w指向原cxq链表
             assert (w != NULL, "Invariant") ;
             ObjectWaiter * u = (ObjectWaiter *) Atomic::cmpxchg_ptr (NULL, &_cxq, w) ;if (u == w) break ;
             w = u ;}
          assert (w != NULL              , "invariant") ;

          ObjectWaiter * q = NULL ;
          ObjectWaiter * p ;for (p = w ; p != NULL ; p = p->_next) { // cxq链表是单向的，为了接到EntryList需要变为双向的
              guarantee (p->TState == ObjectWaiter::TS_CXQ, "Invariant") ;
              p->TState = ObjectWaiter::TS_ENTER ; // 因为会进入EntryList链表，所以将等待状态从TS_CXQ变为TS_ENTER
              p->_prev = q ;
              q = p ;}// Append the RATs to the EntryList// TODO: organize EntryList as a CDLL so we can locate the tail in constant-time.
          ObjectWaiter * Tail ;for (Tail = _EntryList ; Tail != NULL && Tail->_next != NULL ; Tail = Tail->_next) ;if (Tail == NULL) {
              _EntryList = w ;} else {
              Tail->_next = w ;
              w->_prev = Tail ;}// 此时原cxq链表已经链接到EntryList后面了// Fall thru into code that tries to wake a successor from EntryList}if (QMode == 4 && _cxq != NULL) { // Knob_QMode是4意味着需要将cxq链表中的元素链接到EntryList链表前面// Aggressively drain cxq into EntryList at the first opportunity.// This policy ensure that recently-run threads live at the head of EntryList.// Drain _cxq into EntryList - bulk transfer.// First, detach _cxq.// The following loop is tantamount to: w = swap (&cxq, NULL)
          w = _cxq ;for (;;) {
             assert (w != NULL, "Invariant") ;
             ObjectWaiter * u = (ObjectWaiter *) Atomic::cmpxchg_ptr (NULL, &_cxq, w) ;if (u == w) break ;
             w = u ;}
          assert (w != NULL              , "invariant") ;

          ObjectWaiter * q = NULL ;
          ObjectWaiter * p ;for (p = w ; p != NULL ; p = p->_next) {
              guarantee (p->TState == ObjectWaiter::TS_CXQ, "Invariant") ;
              p->TState = ObjectWaiter::TS_ENTER ;
              p->_prev = q ;
              q = p ;}// Prepend the RATs to the EntryListif (_EntryList != NULL) {
              q->_next = _EntryList ;
              _EntryList->_prev = q ;}
          _EntryList = w ;// 此时原cxq链表已经链接到EntryList链表前面了，_EntryList指向新的EntryList链表// Fall thru into code that tries to wake a successor from EntryList}// 对上面两个if判断来说，w指向新的EntryList链表，若不符合上述判断条件则w依然指向原EntryList链表
      w = _EntryList  ;if (w != NULL) { // 若EntryList链表不空，则唤醒EntryList链表中的第一个线程// I'd like to write: guarantee (w->_thread != Self).// But in practice an exiting thread may find itself on the EntryList.// Lets say thread T1 calls O.wait().  Wait() enqueues T1 on O's waitset and// then calls exit().  Exit release the lock by setting O._owner to NULL.// Lets say T1 then stalls.  T2 acquires O and calls O.notify().  The// notify() operation moves T1 from O's waitset to O's EntryList. T2 then// release the lock "O".  T2 resumes immediately after the ST of null into// _owner, above.  T2 notices that the EntryList is populated, so it// reacquires the lock and then finds itself on the EntryList.// Given all that, we have to tolerate the circumstance where "w" is// associated with Self.
          assert (w->TState == ObjectWaiter::TS_ENTER, "invariant") ;
          ExitEpilog (Self, w) ;return ;}// cxq链表和EntryList链表均为空时重试// If we find that both _cxq and EntryList are null then just// re-run the exit protocol from the top.
      w = _cxq ;if (w == NULL) continue ;// 以下是cxq链表不为空但EntryList链表为空的情况// Drain _cxq into EntryList - bulk transfer.// First, detach _cxq.// The following loop is tantamount to: w = swap (&cxq, NULL)for (;;) {
          assert (w != NULL, "Invariant") ;
          ObjectWaiter * u = (ObjectWaiter *) Atomic::cmpxchg_ptr (NULL, &_cxq, w) ;if (u == w) break ;
          w = u ;}TEVENT (Inflated exit - drain cxq into EntryList) ;

      assert (w != NULL              , "invariant") ;
      assert (_EntryList  == NULL    , "invariant") ;// Convert the LIFO SLL anchored by _cxq into a DLL.// The list reorganization step operates in O(LENGTH(w)) time.// It's critical that this step operate quickly as// "Self" still holds the outer-lock, restricting parallelism// and effectively lengthening the critical section.// Invariant: s chases t chases u.// TODO-FIXME: consider changing EntryList from a DLL to a CDLL so// we have faster access to the tail.if (QMode == 1) { // Knob_QMode是1意味着EntryList链表为空，需要使cxq链表成为EntryList链表并反转顺序// QMode == 1 : drain cxq to EntryList, reversing order// We also reverse the order of the list.
         ObjectWaiter * s = NULL ;
         ObjectWaiter * t = w ;
         ObjectWaiter * u = NULL ;while (t != NULL) {
             guarantee (t->TState == ObjectWaiter::TS_CXQ, "invariant") ;
             t->TState = ObjectWaiter::TS_ENTER ;
             u = t->_next ;
             t->_prev = u ;
             t->_next = s ;
             s = t;
             t = u ;}
         _EntryList  = s ;
         assert (s != NULL, "invariant") ;} else { // 其他情况只需要使cxq链表成为EntryList链表，不需要反转// QMode == 0 or QMode == 2
         _EntryList = w ;
         ObjectWaiter * q = NULL ;
         ObjectWaiter * p ;for (p = w ; p != NULL ; p = p->_next) {
             guarantee (p->TState == ObjectWaiter::TS_CXQ, "Invariant") ;
             p->TState = ObjectWaiter::TS_ENTER ;
             p->_prev = q ;
             q = p ;}}// In 1-0 mode we need: ST EntryList; MEMBAR #storestore; ST _owner = NULL// The MEMBAR is satisfied by the release_store() operation in ExitEpilog().// See if we can abdicate to a spinner instead of waking a thread.// A primary goal of the implementation is to reduce the// context-switch rate.if (_succ != NULL) continue;

      w = _EntryList  ;if (w != NULL) { // 唤醒新EntryList链表中的第一个线程
          guarantee (w->TState == ObjectWaiter::TS_ENTER, "invariant") ;
          ExitEpilog (Self, w) ;return ;}}
}
```

 **本节课后思考**

在 Java 体系中存在着各种各样对锁的称呼，例如公平锁、非公平锁、读写锁、乐观锁、悲观锁等等，你对它们有所了解吗？欢迎同学们在评论区中讨论。



 

