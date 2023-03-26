---
title: 06发现Thread背后的特性
toc: true
tags: springboot
categories: 
    - [java]
    - [concurrence]
---

从本章节开始，我们将进入到 Java 并发编程的领域。

在多线程编程过程中，总会或多或少地接触到多线程这个概念。而 Java 的并发编程领域，想要使用线程技术，就不得不得接触到 java.lang.Thread 这个类。

很多程序员都使用过java.lang.Thread 这个类，但是大多数人可能只停留在了下边这种操作情况：

```java
Thread t = new Thread(new Runnable(){....})
t.start();
```

<!--more-->

然后就没有了～

其实呢，Thread 类的内部不单单只是有 run 方法，它还有蛮多特性的，那么这节课，就让我们一起去发现 Thread 背后隐藏的“特性”。

#### 线程的构建

Thread 的含义是指线程，它实现了 java.lang.Runnable 接口，在 JDK8 里面，java.lang.Runnable 是一个函数式接口，其内部定义了 run 方法，而 run 方法就是线程内部具体逻辑的执行入口。

**那么** **，** **在实际使用Thread的时候，我们会如何去操作它呢？** 来看看下边这个案例 **：**

```java
Thread thread = new Thread(() -> {
    while (true) {
        try {
            Thread.sleep(2000);
            System.out.println("i am running ....");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
});
thread.start();
```

这段案例中，通过构建一个 Thread 对象之后，触发它的 start 方法，当 CPU 将时间片分配给到对应线程之后，线程内部的 run 逻辑便会触发执行，这也是大家使用线程的最基本方式。

但是随着系统的复杂性提升，一个进程中通常会运行着各种各样的线程，每个线程都有不同的业务含义，例如 #1001，#1002 线程是负责数据库连接，#1004，#1005 线程是负责第三方 http 请求，#1006，#1007 线程是负责计算任务，等等。

面对各式各样的线程，程序员们开始提出了**线程分组**的设计。

#### 线程组

从名字上来看，线程组就是给不同的线程设计不同的分组，并且在命名上也做区分，在 JDK 中，它的具体表现是 ThreadGroup 这个类，如下边的这段案例：

```ini
线程名: db-conn-thread-1, 所在线程组: 数据库连接线程组
线程名: http-req-thread-1, 所在线程组: 第三方http请求线程组
线程名: http-req-thread-0, 所在线程组: 第三方http请求线程组
线程名: db-conn-thread-0, 所在线程组: 数据库连接线程组
```

同样的，通过利用 VisualVM 相关工具，也可以看到对应的 Java 进程在执行过程中产生的线程信息，具体效果如下图所示：

不过我们一般不会选择在生产环境中使用 VisualVM 这类工具，因为它需要开放相关的端口，存在一定的危险性，所以，通常在生产环境中，我们会使用 Arthas 这款工具，并且通过 Arthas 的 thread 命令去查询相关线程的信息：

通过实战后，我们可以发现，采用了线程组技术之后，对于多线程的管理方面会降低一定的复杂度。

**例如：我们可以通过线程组快速定位到具体是哪个业务模块的线程出现了异常，然后进行快速修复。**

**又或者是针对不同的线程组进行线程监控，了解各个业务模块对于CPU的使用率。**

可能有些细心的同学会发现，使用 ThreadGroup 的时候，需要将它注入到 Thread 类中，这类硬编码的操作比较繁琐，是否有什么合理的方式可以简化相关代码呢？

其实是有的，JDK的开发者在设计的时候还留下了一个叫做 ThreadFacotry 的类。下边让我们一同来了解下这个类的作用。

#### 线程工厂

了解过设计模式中工厂模式的朋友，应该对 ThreadFacotry 不会太陌生，ThreadFactory 是 一个JDK 包中提供的线程工厂类，它的职责就是专门用于生产 Thread 对象。使用了 ThreadFactory 之后，可以帮助我们缩减一些生产线程的代码量，例如下边这个 SimpleThreadFactory 类：

```java
public class SimpleThreadFactory implements ThreadFactory {

    private final int maxThread;
    private final String threadGroupName;
    private final String threadNamePrefix;

    private final AtomicInteger count = new AtomicInteger(0);
    private final AtomicInteger threadSeq = new AtomicInteger(0);

    private final ThreadGroup threadGroup;


    public SimpleThreadFactory(int maxThread, String threadGroupName, String threadNamePrefix) {
        this.maxThread = maxThread;
        this.threadNamePrefix = threadNamePrefix;
        this.threadGroupName = threadGroupName;
        this.threadGroup = new ThreadGroup(threadGroupName);
    }


    @Override
    public Thread newThread(Runnable r) {
        int c = count.incrementAndGet();
        if (c > maxThread) {
            return null;
        }
        Thread t = new Thread(threadGroup, r, threadNamePrefix + threadSeq.getAndIncrement());
        t.setDaemon(false);
        //默认线程优先级
        t.setPriority(Thread.NORM_PRIORITY);
        return t;
    }

    public static void main(String[] args) throws InterruptedException {
        CountDownLatch countDownLatch = new CountDownLatch(1);
        ThreadFactory threadFactory = new SimpleThreadFactory(10, "test-thread-group", "test-thread-");
        Thread t = threadFactory.newThread(new Runnable() {
            @Override
            public void run() {
                System.out.println("this is task");
                try {
                    countDownLatch.await();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
        t.start();
        countDownLatch.await();
    }
}
```

可以看到 ThreadFactory 内部提供了 newThread 方法，这个方法的具体实现中封装了关于线程产生的具体细节，例如线程的分组、命名、优先级，以及是否是守护线程类型。

如果你细心阅读过线程池底层的源代码，那么你应该会发现，线程池在生产线程的时候，其实也是使用了ThreadFactory这个工厂类。 在 Jdk1.8 中的线程池中，定义了两套工厂类，分别是 DefaultThreadFactory 和 PrivilegedThreadFactory，它们其实本质功能都差不多，只不过 PrivilegedThreadFactory 具备了 AccessControlContext 和上下文的类加载器权限。

```java
/**
 * The default thread factory
 */
static class DefaultThreadFactory implements ThreadFactory {
    private static final AtomicInteger poolNumber = new AtomicInteger(1);
    private final ThreadGroup group;
    private final AtomicInteger threadNumber = new AtomicInteger(1);
    private final String namePrefix;

    DefaultThreadFactory() {
        SecurityManager s = System.getSecurityManager();
        group = (s != null) ? s.getThreadGroup() :
                              Thread.currentThread().getThreadGroup();
        namePrefix = "pool-" +
                      poolNumber.getAndIncrement() +
                     "-thread-";
    }

    public Thread newThread(Runnable r) {
        Thread t = new Thread(group, r,
                              namePrefix + threadNumber.getAndIncrement(),
                              0);
        if (t.isDaemon())
            t.setDaemon(false);
        if (t.getPriority() != Thread.NORM_PRIORITY)
            t.setPriority(Thread.NORM_PRIORITY);
        return t;
    }
}

/**
 * Thread factory capturing access control context and class loader
 */
static class PrivilegedThreadFactory extends DefaultThreadFactory {
    private final AccessControlContext acc;
    private final ClassLoader ccl;

    PrivilegedThreadFactory() {
        super();
        SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            // Calls to getContextClassLoader from this class
            // never trigger a security check, but we check
            // whether our callers have this permission anyways.
            sm.checkPermission(SecurityConstants.GET_CLASSLOADER_PERMISSION);

            // Fail fast
            sm.checkPermission(new RuntimePermission("setContextClassLoader"));
        }
        this.acc = AccessController.getContext();
        this.ccl = Thread.currentThread().getContextClassLoader();
    }

    public Thread newThread(final Runnable r) {
        return super.newThread(new Runnable() {
            public void run() {
                AccessController.doPrivileged(new PrivilegedAction<Void>() {
                    public Void run() {
                        Thread.currentThread().setContextClassLoader(ccl);
                        r.run();
                        return null;
                    }
                }, acc);
            }
        });
    }
}
```

好了，现在我们大概已经了解了该怎么去优雅地构建一个线程对象，以及如何去较好地管理多个线程，但是在实际工作中，线程还会有许多不同的应用场景，例如后台监控就是一类非常适合使用线程技术去完成的场景。

而 JDK 的开发者似乎也很早就预料到了这一点，所以他在设计 Thread 类的时候，还专门留下了一个叫做 daemon 的属性，这个属性主要是用于定义当前线程是否属于守护线程。

#### 守护线程

守护线程其实是 JVM 中特殊定义的一类线程，这类线程通常都是以在后台单独运作的方式存在，常见的代表，例如 JVM 中的 Gc 回收线程，可以通过 Arthas 的 Thread 指令区查询这类线程：

**那么，** **为什么需要守护线程呢** **？** **常规的线程也可以实现在后台执行的效果啊**，下边我们来看一组实战代码案例：

```java
public class DaemonThreadDemo {

    public static void main(String[] args) throws InterruptedException {
        Runtime.getRuntime().addShutdownHook(new Thread(() -> System.out.println("jvm exit success!! ")));

        Thread testThread = new Thread(() -> {
            while (true) {
                try {
                    Thread.sleep(2000);
                    System.out.println("thread still running ....");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
        testThread.start();
    }

}
```

（ps：在上边的守护线程代码案例中，我使用了一个 **ShutdownHook**的钩子函数，用于监听当前JVM是否退出。）

![](./img/202303/thread5-8.png)

可以看到，main 线程中构建了一个非守护线程 testThread，testThread 的内部一直在执行 while 循环，导致 main 线程迟迟都无法结束执行。而如果我们尝试将 testThread 设置为守护线程类型的话，结果就会发生变化：

```java
public class DaemonThreadDemo {

    public static void main(String[] args) throws InterruptedException {
        Runtime.getRuntime().addShutdownHook(new Thread(() -> System.out.println("jvm exit success!! ")));

        Thread testThread = new Thread(() -> {
            while (true) {
                try {
                    Thread.sleep(2000);
                    System.out.println("thread still running ....");
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
        testThread.setDaemon(true);
        testThread.start();
    }
}
```

执行结果：

```
jvm exit success!! 

```

通过上边的这个实验可以发现，**守护线程具有在JVM退出的时候也自我销毁的特点，而非守护线程不具备这个特点，这也是为什么GC回收线程被设置为守护线程类型的主要原因。**

#### 线程的异常捕获器

在线程的内部，有一个叫做异常捕获器的概念，当线程在执行过程中产生了异常，就会回调到该接口，来看看下边的这个案例代码：

```java
public static void main(String[] args) {
        Thread thread = new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("this is test");
                int i = 10/0;
            }
        });
        thread.setUncaughtExceptionHandler(new Thread.UncaughtExceptionHandler() {
            //这里是对Throwable对象进行监控，所以无论是error或者exception都能识别到
            @Override
            public void uncaughtException(Thread t, Throwable e) {
                System.err.println("thread is "+t.getName());
                e.printStackTrace();
            }
        });
        thread.start();
    }
```

![](./img/202303/thread5-9.png)

可以看到，当线程出现异常的时候，会回调到 UncaughtExceptionHandler 中，而异常回调器其实本身也是一个函数接口，当线程出现异常的时候，JVM 会默认携带线程信息和异常内容回调到这个接口中:

```java
@FunctionalInterface
public interface UncaughtExceptionHandler {
    /**
     * Method invoked when the given thread terminates due to the
     * given uncaught exception.
     * <p>Any exception thrown by this method will be ignored by the
     * Java Virtual Machine.
     * @param t the thread
     * @param e the exception
     */
    void uncaughtException(Thread t, Throwable e);
}
```

在 ThreadGroup 类中，其实就是对 UncaughtExceptionHandler 进行了单独的实现，所以每次当线程报错的时候才会有异常信息展示，这部分可以通过阅读 ThreadGroup 内部的源代码进行深入了解，下边我将这部分源代码粘出来给大家了解下：

```java
//Jdk1.8中对于线程异常堆栈打印逻辑的源代码
public void uncaughtException(Thread t, Throwable e) {
    if (parent != null) {
        parent.uncaughtException(t, e);
    } else {
        Thread.UncaughtExceptionHandler ueh =
            Thread.getDefaultUncaughtExceptionHandler();
        if (ueh != null) {
            ueh.uncaughtException(t, e);
        } else if (!(e instanceof ThreadDeath)) {
            System.err.print("Exception in thread \""
                             + t.getName() + "\" ");
            e.printStackTrace(System.err);
        }
    }
}
```

如果我们希望当线程运行过程中出现异常后做些上报功能，可以通过采用异常捕获器的思路来实现。

上边我们所学习的各种属性，都是 Thread 类内部比较有用的属性，但是除开这些属性之外，Thread 中还有一个很容易误导开发者的属性，它就是 priority。

#### 线程优先级

在 Thread 的内部还有一个叫做优先级的参数，具体设置可以通过 setPriority 方法去修改。例如下边这段代码：

```java
public class ThreadPriorityDemo {

    static class InnerTask implements Runnable {

        private int i;

        public InnerTask(int i) {
            this.i = i;
        }

        public void run() {
            for(int j=0;j<10;j++){
                System.out.println("ThreadName is " + Thread.currentThread().getName()+" "+j);
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Thread t1 = new Thread(new InnerTask(10),"task-1");
        t1.setPriority(1);
        Thread t2 = new Thread(new InnerTask(2),"task-2");
        //优先级只能作为一个参考数值，而且具体的线程优先级还和操作系统有关
        t2.setPriority(2);
        Thread t3 = new Thread(new InnerTask(3),"task-3");
        t3.setPriority(3);

        t1.start();
        t2.start();
        t3.start();
        Thread.sleep(2000);
    }
}
```

不过“优先级”这个参数通常并不是那么地“靠谱”，理论上说线程的优先级越高，分配到时间片的几率也就越高，**但是在实际运行过程中却并非如此，优先级只能作为一个参考数值，而且具体的线程优先级还和操作系统有关，** 所以大家在编码中如果使用到了“优先级”的设置，请不要强依赖于它。

#### 课后思考

在上一章节中，我留下了一道思考题：并行和并发的理解，下边说说我自己的理解：

- **并行** **，** 通常是指同一时间内有多个进程同时运作，这种情况通常发生在多核心 CPU 或者多个 CPU 共同计算的场景中。
- **并发** **，** 这类场景通常是单个核心的单 CPU 在非常快速地切换多个线程任务，从而让用户感觉同一时刻似乎有多个程序在同时执行，但是实际上每一时刻只能有一个程序在 CPU 上运作。

在互联网场景中，并发场景主要会在一些业务系统中出现地比较多，例如下单接口、秒杀接口等等，而并行的场景更多会发生在一些大数据部门中，例如多台机器一同进行数据清洗之类的。

**最后，我们常说的并发是针对单核 CPU 而言，它指的是 CPU 交替执行不同任务的能力。而并行这个概念更多是针对多核 CPU 或者多个CPU而言，它指的是多个核心同时执行多个任务的能力。**

**本章节思考**

在Thread类中存在几个被声明了抛弃的函数方法：stop，resume，suspend，你对它们有所了解吗，欢迎在下边的评论区中说说你的看法。

