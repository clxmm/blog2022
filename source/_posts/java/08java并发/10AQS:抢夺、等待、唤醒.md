---
title: 10AQS:抢夺、等待、唤醒
toc: true
tags: springboot
categories: 
    - [java]
    - [concurrence]
---

在上一章节中，我们重点介绍了关于 ReentranLock 的底层实现思路，但是有个最核心的部分一直没有介绍，那就是 **AQS**。

AQS 全称是 **AbstractQueuedSynchronizer**，是一个抽象队列同步器，JUC 包的大部分并发工具类都是依托于它去实现的，所以如果希望理解 JUC 中的各种组件原理，就一定要搞懂AQS。

我个人认为，AQS 内部的许多机理，其实和现实生活中的某些场景还是比较相似的，例如银行办理业务场景。

<!--more-->

假设 A、B、C 三个人同时来到银行柜台取钱，但是只有一个柜台可以支持办理业务，那么此时就需要 ABC 三人按照一定的顺序去排队逐个办理，正在办理业务的人会呆在柜台前，而处于等待状态的人需要留在休息区，当 A 办理完成之后，由 A 去通知到 B 进行下一步办理，后续等 B 办理完成之后，由 B 去通知 C，所以整体的流程可以简化为下图 01 所示：

![](./img/202303/thread10-1.png)

其实，这一场景的设计套入到 AQS 的内部也是相同的，大家可以将 AQS 的设计目的理解为：**多线程同时去获取指定的资源的一种处理机制，而AQS的本质就是一个队列，加上一个唤醒机制以及一堆节点的状态，还有一个state。**

既然是多线程去获取指定资源，那么就会面临以下几个问题：

- 多线程环境中如何确保资源夺取的安全性？
- 尝试获取资源失败之后，该如何处理？

带着这些问题，我们一同来看看 AQS 内部的解决手段。

## 1.抢夺

在 AQS 的内部有一个非常重要的字段，叫做 state。

```java
 /**
 * The synchronization state.
 */
private volatile int state;
```

在 AQS 内部有个修改state 字段的关键函数，那就是 acquire 方法，该方法的关键代码内容如下：

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) && //子类去看自己的定义是否允许获取锁
        acquireQueued(addWaiter(Node.EXCLUSIVE) //先去排队, arg)) // 看看需不需要去休息区等待去
        selfInterrupt();
}
```

这段方法的基本逻辑是：尝试去修改 state 字段，如果失败，则会触发两个操作：一个是将请求的线程放入到等待队列中；另一个是释放一个通知信号。所以这个抢夺的方法核心点主要有三项：

1. 修改 state （tryAcquire 方法）；
2. 放入等待队列 （addWaiter 方法）；
3. 释放待通知信号 （acquireQueued 方法）。

下边我们来逐个讲解下这三个核心。

**修改state**

在 AQS 的 tryAcquire 这个函数中，它并没有给出特定的操作，而是将具体实现则交给了各个 JUC 内部组件去进行定义，具体的源代码如下：

```java
protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}
```

例如在 ReentranLock 的非公平锁中，就对 tryAcquire 进行了一层实现，它的内部会调用 nonfairTryAcquire 方法。

```java

/**
 * Sync object for non-fair locks
 */
static final class NonfairSync extends Sync {
    private static final long serialVersionUID = 7316153563782823691L;

    /**
     * Performs lock.  Try immediate barge, backing up to normal
     * acquire on failure.
     */
    final void lock() {
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
            acquire(1);
    }

    protected final boolean tryAcquire(int acquires) {
       //自己封装了一套方法nonfairTryAcquire
        return nonfairTryAcquire(acquires);
    }
}
```

而 nonfairTryAcquire 的内容，我将它们提炼了一下，整理如下所示：

```java
//非公平锁加锁的底层实现方式
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        //这里是一个封装的方法去做cas操作，这里传递的0意味着只有当state字段没有线程修改的时候，cas才可能成功
        //如果传入的是一个大于0的数，那么就可以实现其他功能，例如countDownLatch
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

//通过cas方式去修改stateOffset字段值
protected final boolean compareAndSetState(int expect, int update) {
    // See below for intrinsics setup to support this
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}

//cas修改的那个字段
private static final long stateOffset;


static {
    try {
    //stateOffset字段其实是映射了AQS内部的state字段
        stateOffset = unsafe.objectFieldOffset
            (AbstractQueuedSynchronizer.class.getDeclaredField("state"));
        headOffset = unsafe.objectFieldOffset
            (AbstractQueuedSynchronizer.class.getDeclaredField("head"));
        tailOffset = unsafe.objectFieldOffset
            (AbstractQueuedSynchronizer.class.getDeclaredField("tail"));
        waitStatusOffset = unsafe.objectFieldOffset
            (Node.class.getDeclaredField("waitStatus"));
        nextOffset = unsafe.objectFieldOffset
            (Node.class.getDeclaredField("next"));

    } catch (Exception ex) { throw new Error(ex); }

```

通过上边的这些代码可以看到，在 AQS 的内部维护了一个 state 字段的数值，这个 state 可以理解为是同时允许访问代码临界区的线程个数，线程在访问代码临界区资源之前，都需要先去通过 cas 修改这个 state 字段，对于 state 字段修改失败的线程，其后续的主要逻辑则由 JUC 中的各个组件去定义实现。

其核心流程图如下所示：

![](./img/202303/thread10-2.png)

那么，如果修改 state 字段返回了失败，就会进入到我们接下来要讲解的两个核心，**放入等待队列，并且释放待通知信号。**

## 2.等待队列

在 AQS 类中，设计了一个专门的双向链表用于存储修改 state 失败的线程，这个队列的组成由多个 Node 节点所维护。

Node 可以类比为“图01-银行办理业务”图中的A，B，C的一个抽象概念，只不过这次参与等待的不是人而是请求线程。其主要的核心代码我做了些许精简，大致如下所示：

```java
static final class Node {

    static final Node SHARED = new Node();
    static final Node EXCLUSIVE = null;

    static final int CANCELLED =  1;
    static final int SIGNAL    = -1;
    static final int CONDITION = -2;
    static final int PROPAGATE = -3;
    //等待状态
    volatile int waitStatus;
    
    //双向链表，所以有前驱和后驱指针
    volatile Node prev;
    volatile Node next;
    //关联该节点的线程
    volatile Thread thread;

    Node nextWaiter;
}

```

多个 Node 之间会形成一条链表结构，如果等待过程中的某个节点中途放弃等待了，那么它的 waitStatus 就会变化为 CANCELLED状态。

而 addWaiter 函数则是专门负责将请求修改 state 失败的线程放入到一个队列中，该部分的源代码如下所示：

```java
/**
 * Creates and enqueues node for current thread and given mode.
 *
 * @param mode Node.EXCLUSIVE for exclusive, Node.SHARED for shared
 * @return the new node
 */
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // 这里的一段内容其实和enq的内容有些相似，主要是一种优化的写法，在进入enq
    //方法内部之前先尝试往队尾新增node
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    //将node加入到队尾
    enq(node);
    return node;
}

//放入队列的部分
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
        //注意，这里是先修改prev引用，然后才是修改next引用
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

在放入队列的过程中有个细节点我们需要注意下，就是程序会优先修改新加入 node 节点的 prev 指针，然后才是去修改 next 指针，这里是为了保证 node 队列在并发情况下的准确性。

## 3.唤醒机制

当我们请求的线程没有获取到锁，那么它就会被放入到一个等待队列中，这个等待队列的核心部分是acquireQueued函数，其中的设计源代码如下所示：

```java
/**
 * Acquires in exclusive uninterruptible mode for thread already in
 * queue. Used by condition wait methods as well as acquire.
 *
 * @param node the node
 * @param arg the acquire argument
 * @return {@code true} if interrupted while waiting
 */
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
            //这里会有一个对等待队列头节点的waitStatus做修改的操作
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }

```

在 acquireQueued 函数的内部，我们重点需要关注的是 shouldParkAfterFailedAcquire 方法，这个方法的底层代码如下：

```java
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
        pred.next = node;
    } else {
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

稍微解释下，如果单独看 shouldParkAfterFailedAcquire 函数，可能不是太好理解，但是如果将它结合着release方法一起来理解的话，就比较清楚了。shouldParkAfterFailedAcquire函数的主要目的其实是在修改头节点的 waitStatus 状态为 SIGNAL 状态 *，* 而这一点正好会在 AQS 唤醒中有所映射。

当修改 state 成功的线程处理完自己的任务之后，就需要去唤醒等待队列中的下一个线程，这部分逻辑主要体现在了 AQS的 **release** 方法中。下边我们一起来看看这部分源代码的设计：

```java
/**
 * Releases in exclusive mode.  Implemented by unblocking one or
 * more threads if {@link #tryRelease} returns true.
 * This method can be used to implement method {@link Lock#unlock}.
 *
 * @param arg the release argument.  This value is conveyed to
 *        {@link #tryRelease} but is otherwise uninterpreted and
 *        can represent anything you like.
 * @return the value returned from {@link #tryRelease}
 */
public final boolean release(int arg) {
    //tryRelease是一个由子类去实现的函数
    if (tryRelease(arg)) {
        Node h = head;
        //代码片段--001 
        if (h != null && h.waitStatus != 0)
        //核心的唤醒下一个线程的代码，传入的参数是等待队列的头节点
            unparkSuccessor(h);
        return true;
    }
    return false;
}

/**
 * Wakes up node's successor, if one exists.
 *
 * @param node the node
 */
private void unparkSuccessor(Node node) {
    /*
     * If status is negative (i.e., possibly needing signal) try
     * to clear in anticipation of signalling.  It is OK if this
     * fails or if status is changed by waiting thread.
     */
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    /*
     * Thread to unpark is held in successor, which is normally
     * just the next node.  But if cancelled or apparently null,
     * traverse backwards from tail to find the actual
     * non-cancelled successor.
     */
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        //这里一点需要注意，队列中可能会存在等待中途退出的节点，所以这类会需要进行修复
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
        //唤醒头节点的线程
        LockSupport.unpark(s.thread);
}
```

## 课后小节

本节的内容大部分都涉及到了源代码方面的分析，所以需要有一定的耐心去阅读。最后我们对 AQS 做一番整理和总结，巩固下我们学习到的知识点。

AQS 是一套 JUC 内部各个基础组件都会使用的公共抽象部分，其主要组成部分为：一个队列，一个唤醒机制，一堆节点的状态以及一个 state。而它内部的运作流程可以概括为：

- 通过 cas 的方式尝试修改 state，修改成功，则有权利进入代码临界区。
- 修改 state 失败的线程会被放入到等待队列中进行等待。
- 修改 state 成功的线程离开代码临界区的时候，需要去通知等待队列中的下一个线程。

## 课后思考

在Java体系中，其实对于锁的称呼和分类有许多种类别，这一块为了方便大家去理解和认识它们，我设计了一张图表来带大家理解这些知识概念，希望能对你有所帮助。

![](./img/202303/thread10-3.png)

在实际工作中，其实 AQS 的产物并不只有 ReentranLock 这么一种，例如 CountDownLatch，CyclicBarrier，Semaphore 都是对应的代表产物，你在工作中有接触过这三个类吗？欢迎大家在评论区中展开讨论。

