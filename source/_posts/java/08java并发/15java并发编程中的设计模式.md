---
title: 15java并发编程中的设计模式
toc: true
tags: springboot
categories: 
    - [java]
    - [concurrence]
---

其实在 Java 并发编程这个领域中，隐藏了许多的“设计模式”，并发编程的设计模式和我们常谈的“单例模式”、“工厂模式”这类“设计模式” ，其实可以理解为**都是对代码精良设计的思想提炼。**

在前边的章节中，我们分别介绍了工作中常用到的并发编程技术，那么在本章节中，我们继续来深入认识下并发编程领域的设计模式。

### Producer Consumer 模式

Producer-Consumer 模式是大众们使用最多的模式之一，它的主要表现形式可以用下边一张图来解释：

<!--more-->

![](./img/202303/thread15-1.png)

生产者往一个缓冲区中投递元素，接着消费者从这个缓冲区中去提取元素，这方面的具体代表如 JDK 中的 **java.util.concurrent.BlockingQueue** 类，这个类虽然实现了 **java.util.Queue** 接口，也都提供了 **offer** 和 **poll** 方法，但是要想利用它的阻塞效果，还是得使用它的 **put** 和 **take** 方法。

另外在它的底层实现方面可以选择的种类有很多种，诸如 **ArrayBlockingQueue、LinkedBlockingQueue、PriorityBlockingQueue、SynchronousQueue 、DelayQueue、ConcurrentLinkedQueue。**

下边我给出一段比较经典的 Producer-Consumer 模式代码，带大家理解下这种模式：

```java
//消息生产者
public class Producer {

    private ArrayBlockingQueue<String> queue;

    public Producer(ArrayBlockingQueue<String> queue) {
        this.queue = queue;
    }

    public boolean put(String content){
        try {
            queue.put(content);
            return true;
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return false;
    }

    public void take(){
        queue.poll();
    }
}

//消息消费者
public class Consumer {

    private ArrayBlockingQueue<String> queue;

    public Consumer(ArrayBlockingQueue<String> queue) {
        this.queue = queue;
    }

    public void start() {
        Thread t = new Thread(new Runnable() {
            public void run() {
                while (true) {
                    try {
                        String content = queue.take();
                        System.out.println("消费数据：" + content);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        });
        t.start();
    }
}

//测试代码
public class TestDemo {
    public static ArrayBlockingQueue<String> queue = new ArrayBlockingQueue<String>(20);

    public static void main(String[] args) throws InterruptedException {
        Producer producer = new Producer(queue);
        Consumer consumer = new Consumer(queue);
        consumer.start();
        while (true){
            System.out.println("投递元素");
            producer.put(UUID.randomUUID().toString());
            Thread.sleep(2000);
        }
    }

}
```

Producer 类内部会往一个阻塞队列中投递消息，而 Consumer 类中会有一个线程不断地从阻塞队列中取出消息，一投一取，从而形成了**生产者/消费者**的模式。

那么，什么场景下会使用这种模式呢？

**在面对一些突发流量的时候，可以尝试利用生产者/消费者的模式来进行削减。** 例如 Nacos 底层对于心跳包的处理机制就是采用了这种方式，在内存中设计了一条阻塞队列，用于接收各个注册方发送过来的心跳包数据，然后在 Nacos 的底层会有一条线程专门处理这些心跳包数据，整体流程如下图所示：

![](./img/202303/thread15-3.png)

这种设计可以有效应对“当有上千节点同时往nacos中心发送心跳包”所带来的高并发请求问题。

### Guarded Suspension 模式

Guarded 和 Suspension 两个单词的含义分别是“守护” + “暂停” 的意思，可以简单理解为：在线程可能遇到问题的时候进入等待状态，当问题解决后再来唤醒。

下边我们来看看这段案例代码：

```java
public class MQProducer {

    private CountDownLatch countDownLatch = new CountDownLatch(1);

    /**
     * 发送消息给到broker节点
     *
     * @param content
     * @param topic
     * @return
     */
    public SendResult sendMsg(String content, String topic) {
        this.sendMsgToBroker(content, topic);
        try {
            boolean countDownStatus = countDownLatch.await(3, TimeUnit.SECONDS);
            if(countDownStatus){
                return SendResult.SUCCESS;
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return SendResult.FAIL;
    }

    /**
     * 当broker返回消息成功写入后回调这里
     */
    private void receiveBrokerMsg(String msg){
        //处理broker返回的心跳包
        countDownLatch.countDown();
        System.out.println("countDown");
    }

    /**
     * 模拟发送消息到broker节点
     *
     * @param content
     * @param topic
     */
    private void sendMsgToBroker(String content, String topic) {
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) {
        final MQProducer mqProducer = new MQProducer();
        Thread t = new Thread(new Runnable() {
            public void run() {
                try {
                    Thread.sleep(500);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                mqProducer.receiveBrokerMsg("");
            }
        });
        t.start();
        SendResult sendResult = mqProducer.sendMsg("test","test-topic");
        System.out.println(sendResult);
    }

}
```

这段代码中我模拟实现了一个 MQ 的发送类，在这个 MQ 的发送类中有一个 sendMsg 方法，它主要负责将消息发送到 broker 方。但是在获取到 broker 返回的确认信号之后，再将是否投递成功结果返回给主线程，代码中加入了一个 CountDownLatch 类用于阻塞，当 broker 返回确认信号之后再放开这个 CountDownLatch。

Guarded Suspension 模式的本质，其实可以理解为是多线程中的一种等待唤醒机制实现的标准，在我们实际工作中所使用的很多中间件都有用到过这种模式，例如 RocketMQ 底层在发送了消息之后，就会处于一个等待状态，当 message 被写入到 broker 之后，broker 回传一个信号给到 provider 节点，最后放开对应的 CountDownLatch 并且告知发送方消息发送成功。

### Balking 模式

Balking 模式其实我们可以将它理解为是 **在多线程环境下使用的一种规范。**

下边这是一个采用了 Balking 模式的代码案例：

```java
/**
*  @Author  idea
*  @Date  created in 6:57 下午 2022/5/17
*/
public class BalkingDemo {

    private boolean isChange = false;

    public void saveFile() {
        synchronized (this) {
            if (isChange) {
                //将缓冲区的代码写入到磁盘中
                System.out.println("保存磁盘");
                isChange = false;
            }
        }
    }

    public void changeFile() {
        synchronized (this) {
            isChange = true;
            //修改缓冲区的数据
            System.out.println("修改缓冲数据");
        }
    }

    public static void main(String[] args) throws InterruptedException {
        BalkingDemo balkingDemo = new BalkingDemo();
        for(int i=0;i<10;i++){
            Thread saveThread = new Thread(new Runnable() {
                @Override
                public void run() {
                    balkingDemo.saveFile();
                }
            });
            Thread changeThread = new Thread(new Runnable() {
                @Override
                public void run() {
                    balkingDemo.changeFile();
                }
            });
            saveThread.start();
            changeThread.start();
            Thread.sleep(1000);
        }

    }
}
```

这段代码中模拟了一个文件保存的案例，当 isChange 变量被外界条件所修改了的话，调用 saveFile 函数就会执行文件保存的动作。在代码案例中，我们模拟了多个线程执行修改和保存的动作，在这种多线程交互的场景中，对于 if 判断的处理尤其关键。

在代码中我们可以看到，对于 isChange 变量的判断，我们都是使用了 synchronized 关键字去访问，这样可以保证每次只会有一个线程去访问到 isChange 变量，从而避免了线程安全问题。

**这里我们如果尝试不使用synchronized关键字，而是使用 volatile 关键字的话，有可能会存在什么问题呢？**

这样的话，代码可能就会变成如下所示：

```java
private volatile boolean isChange = false;

public void saveFile() {
        if (isChange) {
            //将缓冲区的代码写入到磁盘中
            System.out.println("保存磁盘");
            isChange = false;
        }
}

public void changeFile() {
        isChange = true;
        //修改缓冲区的数据
        System.out.println("修改缓冲数据");
}
```

这种方式其实在并发度不高的情况下，可能不会那么容易暴露问题，但是依然是存在一定风险的，因为 volatile 关键字不能保证线程安全性，所以建议在实际工作中，当你需要处理多线程中的 if 判断逻辑时，对于 volatile 关键字要慎重一些。

在 Balking 模式中，我们使用 if 判断的时候，除了可以使用 synchronized 锁之外，还可以使用 JDK 的 Lock 锁，建议大家在工作中遵守这两种使用规范。

### CopyOnWrite 模式

CopyOnWrite 模式的核心我们可以理解为：当某个人想要修改相关内容的时候，就先将真正的内容 Copy 出去形成一个新的内容，然后再去修改。当修改好了之后，再替换掉之前的内容。

这种模式在我们的实际工作中也是经常会使用到的，例如 JDK 中的 COW 系列，下边的这组案例是关于使用 CopyOnWriteArrayList 和 CopyOnWriteArraySet 的介绍：

```java
public class CopyOnWriteDemo {

    @Test
    public void showCopyOnWriteArrayList() {
        CopyOnWriteArrayList<String> copyOnWriteArrayList = new CopyOnWriteArrayList();
        copyOnWriteArrayList.add("idea1");
        copyOnWriteArrayList.add("idea2");
        copyOnWriteArrayList.add("idea3");
        copyOnWriteArrayList.add("idea4");
        System.out.println(copyOnWriteArrayList.get(0));
    }

    @Test
    public void showCopyOnWriteArraySet() {
        CopyOnWriteArraySet copyOnWriteArraySet = new CopyOnWriteArraySet();
        copyOnWriteArraySet.add("t1");
        copyOnWriteArraySet.add("t1");
        copyOnWriteArraySet.add("t1");
        System.out.println(copyOnWriteArraySet.size());
    }

}
```

关于 cow 的核心思想，这一点我们可以在 CopyOnWriteArrayList 的底层中查看到：

```java
//添加元素
public boolean add(E e) {
    return al.addIfAbsent(e);
}

public boolean addIfAbsent(E e) {
    //先拷贝一个快照对象
    Object[] snapshot = getArray();
    return indexOf(e, snapshot, 0, snapshot.length) >= 0 ? false :
        addIfAbsent(e, snapshot);
}

private boolean addIfAbsent(E e, Object[] snapshot) {
    final ReentrantLock lock = this.lock;
    //加入了锁，从而避免并发写的问题
    lock.lock();
    try {
        //目前已有的数据
        Object[] current = getArray();
        int len = current.length;
        if (snapshot != current) {
            //如果此时数组已经被其他线程有所修改，则需要判断目前元素是否完全相同
            int common = Math.min(snapshot.length, len);
            for (int i = 0; i < common; i++)
                if (current[i] != snapshot[i] && eq(e, current[i]))
                    return false;
            if (indexOf(e, current, common, len) >= 0)
                    return false;
        }
        //将原先的内容拷贝出来
        Object[] newElements = Arrays.copyOf(current, len + 1);
        //修改
        newElements[len] = e;
        //重新复制操作
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
```

当使用 CopyOnWriteArrayList 读取元素的时候，依然是直接从对应的集合中提取即可。

```java
public E get(int index) {
    return get(getArray(), index);
}

@SuppressWarnings("unchecked")
private E get(Object[] a, int index) {
    return (E) a[index];
}

//直接返回数组对象
final Object[] getArray() {
    return array;
}
```

通常 COW 这种思想在高并发场景中会有所使用。例如当我们在应用中设计了一份黑名单用户表，在内存中可以采用 CopyOnWriteSet 集合将用户的 id 存储起来，通常我们会在服务的网关处去核对该用户是否属于黑名单用户。另外这种场景是属于经典的读多写少的场景，而 COW 的设计效果正好符合，当需要修改黑名单数据的时候，copy 一份数据出来修改，当修改完后重新赋值即可。

整体的流程图如下所示：

![](./img/202303/thread14-5.png)

### 课后小结

通过本章节的学习，我们认识了几种并发编程场景中会使用到的设计模式，我将它们整理了下如下表所示，希望能对各位朋友有所帮助。

| 模式名称               | 模式含义                                                     | 应用场景                                                     |
| ---------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Producer Consumer模式  | 在多线程场景中，一方作为生产者产生数据，另一方作为消费者消费数据。当生产者没有生产数据的时候，消费者就无法继续处理请求。 | 线程池就是经典的一个生产者消费者模型，一端产生数据，一端消费数据。 |
| Guarded Suspension模式 | 在多线程场景中，当某个线程遇到特定条件的情况下就会一直处于等待的状态，如果条件发生了变化，则继续执行 | 例如RocketMQ底层对于消息的发送，当发送方没有收到broker返回的写入成功信号就会处于一个等待的模式。 |
| Balking模式            | 多线程下对于使用if判断的一种规范，通常会采用互斥锁来实现。   | 例如多线程中对一些竞争条件的判断，可以适当加入JDK中的Lock和synchronized 进行保护。 |
| CopyOnWrite模式        | 拷贝修改的设计思路，先将数据拷贝出来一份副本，然后针对副本进行修改，当修改完毕后重新赋值到原先内存地址中。 | 在一些并发读多，并发写少的情况下可以采用该设计思想，例如JDK中的COW组件。 |

### 课后思考

**上节课答疑**

关于 InheritableThreadLocal 和 ThreadContextTransmitter 两者的介绍，我整理了一张表格供大家了解：

| 名字                         | **特点**                                                     |
| ---------------------------- | ------------------------------------------------------------ |
| **InheritableThreadLocal**   | 主要用于子线程创建时，需要自动继承父线程的 ThreadLocal 变量，方便必要信息的进一步传递。具体原理是在创建子线程的“一刹那”，将父线程的 ThreadLocalMap 复制给子线程。但是在一些应用场景中，其功能并没有 **TransmittableThreadLocal** 强大。 |
| **TransmittableThreadLocal** | 它是基于 ThreadLocal 和 InheritableThreadLocal 的基础上进行设计的一款组件，可以将 A 线程的本地变量透传到线程池内部的线程中，在实现全链路追踪的时候经常会被用到。 |

