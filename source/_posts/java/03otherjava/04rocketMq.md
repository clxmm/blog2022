---
title: 04消息队列RocketMQ的应用
toc: true
tags: RocketMq
categories: 
    - [java]
    - [RocketMq]
---

##  四、事务消息

### **1** 问题引入

这里的一个需求场景是:工行用户A向建行用户B转账1万元。

我们可以使用同步消息来处理该需求场景:

<!--more-->

![](/img/202209/02rocketmq16.png)

1. 工行系统发送一个给B增款1万元的同步消息M给Broker 
2. 消息被Broker成功接收后，向工行系统发送成功ACK
3. 工行系统收到成功ACK后从用户A中扣款1万元
4. 建行系统从Broker中获取到消息M
5. 建行系统消费消息M，即向用户B中增加1万元

> 这其中的问题，若第三步是失败；但消息已经成功发送到baoker。对于Mq来说，只要消息写入成功，那么这个消息就可以被消费了，会出现数据不一致的情况

### **2** 解决思路

解决思路是，让第1、2、3步具有原子性，要么全部成功，要么全部失败。即消息发送成功后，必须要保证扣款成功。如果扣款失败，则回滚发送成功的消息。而该思路即使用 事务消息 。这里要使用 分布 式事务 解决方案。

![](/img/202209/02rocketmq17.png)

使用事务消息来处理该需求场景:

1. 事务管理器TM向事务协调器TC发起指令，开启全局事务

2. 工行系统发一个给B增款1万元的事务消息M给TC

3. TC会向Broker发送半事务消息prepareHalf，将消息M预提交到Broker。此时的建行系统是看不到Broker中的消息M的

4. Broker会将预提交执行结果Report给TC。

5. 如果预提交失败，则TC会向TM上报预提交失败的响应，全局事务结束;如果预提交成功，TC会调用工行系统的 回调操作 ，去完成工行用户A的 预扣款 1万元的操作

6. 工行系统会向TC发送预扣款执行结果，即本地事务的执行状态

7. TC收到预扣款执行结果后，会将结果上报给TM。

> 预扣款结果的三种可能
>
> ```java
> public enum LocalTransactionState {
>     COMMIT_MESSAGE,   // 本地事务执行成功
>     ROLLBACK_MESSAGE, // 本地事务执行失败
>     UNKNOW,   // 不确定
> }
> ```

8. TM会根据上报结果向TC发出不同的确认指令
   - 若预扣款成功(本地事务状态为COMMIT_MESSAGE)，则TM向TC发送Global Commit指令 
   - 若预扣款失败(本地事务状态为ROLLBACK_MESSAGE)，则TM向TC发送Global Rollback指令 
   - 若现未知状态(本地事务状态为UNKNOW)，则会触发工行系统的本地事务状态 回查操作 。回查操作会将回查结果，即COMMIT_MESSAGE或ROLLBACK_MESSAGE Report给TC。TC将结果上 报给TM，TM会再向TC发送最终确认指令Global Commit或Global Rollback
9. TC在接收到指令后会向Broker与工行系统发出确认指令
   - TC接收的若是Global Commit指令，则向Broker与工行系统发送Branch Commit指令。此时 Broker中的消息M才可被建行系统看到;此时的工行用户A中的扣款操作才真正被确认
   -  TC接收到的若是Global Rollback指令，则向Broker与工行系统发送Branch Rollback指令。此时 Broker中的消息M将被撤销;工行用户A中的扣款操作将被回滚

### **3** 基础

#### 分布式事务

对于分布式事务，通俗地说就是，一次操作由若干分支操作组成，这些分支操作分属不同应用，分布在不同服务器上。分布式事务需要保证这些分支操作要么全部成功，要么全部失败。分布式事务与普通事务一样，就是为了保证操作结果的一致性。

#### 事务消息

RocketMQ提供了类似X/Open XA的分布式事务功能，通过事务消息能达到分布式事务的最终一致。XA 是一种分布式事务解决方案，一种分布式事务处理模式。

#### 半事务消息

暂不能投递的消息，发送方已经成功地将消息发送到了Broker，但是Broker未收到最终确认指令，此时该消息被标记成“暂不能投递”状态，即不能被消费者看到。处于该种状态下的消息即半事务消息。

#### 本地事务状态

Producer 回调操作 执行的结果为本地事务状态，其会发送给TC，而TC会再发送给TM。TM会根据TC发 送来的本地事务状态来决定全局事务确认指令。

```java
public enum LocalTransactionState {
    COMMIT_MESSAGE,   // 本地事务执行成功
    ROLLBACK_MESSAGE, // 本地事务执行失败
    UNKNOW,   // 不确定
}
```

#### 消息回查

![](/img/202209/02rocketmq18.png)

消息回查，即重新查询本地事务的执行状态。本例就是重新到DB中查看预扣款操作是否执行成功。

> 注意，消息回查不是重新执行回调操作，回调操作是进行预扣款操作，而消息回查则是查看预扣款操作执行的结果。
>
> 引发消息回查的原因最常见的有两个“
>
> - 1回调操作返回UNKNOW
> - 2TC没有收得到TM的最终全局事务确认指令

#### **RocketMQ**中的消息回查设置

关于消息回查，有三个常见的属性设置。它们都在broker加载的配置文件中设置，例如:

- transactionTimeout=20，指定TM在20秒内应将最终确认状态发送给TC，否则引发消息回查。默认为60秒
- transactionCheckMax=5，指定最多回查5次，超过后将丢弃消息并记录错误日志。默认15次。
- transactionCheckInterval=10，指定设置的多次消息回查的时间间隔为10秒。默认为60秒。

### **4 XA**模式三剑客

#### **XA**协议

XA(Unix Transaction)是一种分布式事务解决方案，一种分布式事务处理模式，是基于XA协议的。XA协议由Tuxedo(Transaction for Unix has been Extended for Distributed Operation，分布式操作扩展之后的Unix事务系统)首先提出的，并交给X/Open组织，作为资源管理器与事务管理器的接口标准。

XA模式中有三个重要组件:TC、TM、RM。

#### **TC**

Transaction Coordinator，事务协调者。维护全局和分支事务的状态，驱动全局事务提交或回滚。

> *RocketMQ Broker TC*

#### **TM**

Transaction Manager，事务管理器。定义全局事务的范围:开始全局事务、提交或回滚全局事务。它实际是全局事务的发起者。

> *RocketMQ Producer TM*

#### **RM**

Resource Manager，资源管理器。管理分支事务处理的资源，与TC交谈以注册分支事务和报告分支事务的状态，并驱动分支事务提交或回滚。

> *RocketMQ Producer Broker RM*

### **5 XA**模式架构

![](/img/202209/02rocketmq19.png)

XA模式是一个典型的2PC，其执行原理如下:

1. TM向TC发起指令，开启一个全局事务。

2. 根据业务要求，各个RM会逐个向TC注册分支事务，然后TC会逐个向RM发出预执行指令。

3. 各个RM在接收到指令后会在进行本地事务预执行。

4. RM将预执行结果Report给TC。当然，这个结果可能是成功，也可能是失败。

5. TC在接收到各个RM的Report后会将汇总结果上报给TM，根据汇总结果TM会向TC发出确认指 令。

   - 若所有结果都是成功响应，则向TC发送Global Commit指令。

   - 只要有结果是失败响应，则向TC发送Global Rollback指令。

6. TC在接收到指令后再次向RM发送确认指令。



### **6** 注意

- 事务消息不支持延时消息
- 对于事务消息要做好幂等性检查，因为事务消息可能不止一次被消费(因为存在回滚后再提交的情况)

### **7** 代码举例

#### 定义工行事务监听器

```java
public class ICBCTransactionListener implements TransactionListener {


    // 回调操作方法
    // 消息预提交成功就会触发该方法的执行，用于完成本地事务
    @Override
    public LocalTransactionState executeLocalTransaction(Message msg, Object arg) {

        System.out.println("预提交消息成功:" + msg);
        // 假设接收到TAGA的消息就表示扣款操作成功，TAGB的消息表示扣款失败，
        // TAGC表示扣款结果不清楚，需要执行消息回查

        if (StringUtils.equals("TAGA", msg.getTags())) {
            return LocalTransactionState.COMMIT_MESSAGE;
        } else if (StringUtils.equals("TAGB", msg.getTags())) {
            return LocalTransactionState.ROLLBACK_MESSAGE;
        } else if (StringUtils.equals("TAGC", msg.getTags())) {
            return LocalTransactionState.UNKNOW;
        }
        return LocalTransactionState.UNKNOW;

    }


    // 消息回查方法
    // 引发消息回查的原因最常见的有两个:
    // 1)回调操作返回UNKNWON
    // 2)TC没有接收到TM的最终全局事务确认指令
    @Override
    public LocalTransactionState checkLocalTransaction(MessageExt msg) {
        System.out.println("执行消息回查" + msg.getTags());
        return LocalTransactionState.COMMIT_MESSAGE;
    }
}
```

#### 定义事物消息生产者

```java
public class TransactionProducer {


    public static void main(String[] args) throws Exception {

        TransactionMQProducer producer = new TransactionMQProducer("pg");
        producer.setNamesrvAddr(RocketConstant.nameservAddr);

        /**
         * * 定义一个线程池
         * * @param corePoolSize 线程池中核心线程数量
         * * @param maximumPoolSize 线程池中最多线程数
         * * @param keepAliveTime 这是一个时间。当线程池中线程数量大于核心线程数量是，多余空闲线程的存活时长
         * *
         * * @param unit 时间单位
         * * @param workQueue 临时存放任务的队列，其参数就是队列的长度
         * * @param threadFactory 线程工厂
         */
        ThreadPoolExecutor executorService = new ThreadPoolExecutor(2, 5, 100, TimeUnit.SECONDS,
                new ArrayBlockingQueue<Runnable>(2000), new ThreadFactory() {
            @Override
            public Thread newThread(Runnable r) {
                Thread thread = new Thread(r);
                thread.setName("client-transaction-msg-check-thread");
                return thread;
            }
        });


        // 为生产者指定一个线程池
        producer.setExecutorService(executorService);
        // 为生产者添加事务监听器
        producer.setTransactionListener(new ICBCTransactionListener());


        producer.start();

        String[] tags = {"TAGA", "TAGB", "TAGC"};

        for (int i = 0; i < 3; i++) {
            byte[] body = ("Hi," + i).getBytes();
            Message msg = new Message("topicD", tags[i], body);

            // 发送事务消息
            // 第二个参数用于指定在执行本地事务时要使用的业务参数
            TransactionSendResult sendResult = producer.sendMessageInTransaction(msg, null);
            System.out.println("发送消息的结果：" + sendResult.getSendStatus());
        }


    }
}
```

#### 定义消费者

直接使用普通消息的SomeConsumer作为消费者即可。

```java
public class SomeConsumer {


    public static void main(String[] args) throws Exception {
        // 定义一个pull消费者

//        DefaultLitePullConsumer consumer = new DefaultLitePullConsumer("cg");
        // 定义一个push消费者
        DefaultMQPushConsumer consumer = new
                DefaultMQPushConsumer("cg");
        consumer.setNamesrvAddr(RocketConstant.nameservAddr);
        consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);
        // 指定消费topic与tag
        consumer.subscribe("topicD", "*");

        // 指定采用“广播模式”进行消费，默认为“集群模式”
        // consumer.setMessageModel(MessageModel.BROADCASTING);
        consumer.registerMessageListener(new MessageListenerConcurrently() {
            @Override
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
                for (MessageExt msg : msgs) {
                    System.out.println(msg);
                }


                // 消费成功
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });
        // 开启消费者消费
        consumer.start();
        System.out.println("Consumer Started");

    }


}
```

运行成功后，只会生产TAGA，TAGC到broke，TAGB，是`ROLLBACK_MESSAGE`,不会进入消息队列



## 五、批量消息

### **1** 批量发送消息

**发送限制**

生产者进行消息发送时可以一次发送多条消息，这可以大大提升Producer的发送效率。不过需要注意以 下几点:

- 批量发送的消息必须具有相同的Topic 
- 批量发送的消息必须具有相同的刷盘策略 
- 批量发送的消息不能是延时消息与事务消息

**批量发送大小**

默认情况下，一批发送的消息总大小不能超过4MB字节。如果想超出该值，有两种解决方案:

- 方案一:将批量消息进行拆分，拆分为若干不大于4M的消息集合分多次批量发送 
- 方案二:在Producer端与Broker端修改属性

**Producer端需要在发送之前设置Producer的maxMessageSize属性**

**Broker端需要修改其加载的配置文件中的maxMessageSize属性**

**生产者发送的消息大小**

![](/img/202209/02rocketmq20.png)

生产者通过send()方法发送的Message，并不是直接将Message序列化后发送到网络上的，而是通过这 个Message生成了一个字符串发送出去的。这个字符串由四部分构成:Topic、消息Body、消息日志 (占20字节)，及用于描述消息的一堆属性key-value。这些属性中包含例如生产者地址、生产时间、 要发送的QueueId等。最终写入到Broker中消息单元中的数据都是来自于这些属性


### **2** 批量消费消息

**修改批量属性**



Consumer的MessageListenerConcurrently监听接口的consumeMessage()方法的第一个参数为消息列 表，但默认情况下每次只能消费一条消息。若要使其一次可以消费多条消息，则可以通过修改 Consumer的consumeMessageBatchMaxSize属性来指定。不过，该值不能超过32。因为默认情况下消 费者每次可以拉取的消息最多是32条。若要修改一次拉取的最大值，则可通过修改Consumer的 pullBatchSize属性来指定。

**存在的问题**

Consumer的pullBatchSize属性与consumeMessageBatchMaxSize属性是否设置的越大越好?当然不 是。

- pullBatchSize值设置的越大，Consumer每拉取一次需要的时间就会越长，且在网络上传输出现 问题的可能性就越高。若在拉取过程中若出现了问题，那么本批次所有消息都需要全部重新拉 取。
- consumeMessageBatchMaxSize值设置的越大，Consumer的消息并发消费能力越低，且这批被消 费的消息具有相同的消费结果。因为consumeMessageBatchMaxSize指定的一批消息只会使用一 个线程进行处理，且在处理过程中只要有一个消息处理异常，则这批消息需要全部重新再次消费 处理。

### **3** 代码举例

该批量发送的需求是，不修改最大发送4M的默认值，但要防止发送的批量消息超出4M的限制。

**定义消息列表分割器**

```java
public class MessageListSplitter implements Iterator<List<Message>> {


    // 消息列表分割器:其只会处理每条消息的大小不超4M的情况。
    // 若存在某条消息，其本身大小大于4M，这个分割器无法处理，
    // 其直接将这条消息构成一个子列表返回。并没有再进行分割


    // 指定极限值为4M
    private final int SIZE_LIMIT = 4 * 1024 * 1024;
    // 存放所有要发送的消息
    private final List<Message> messages;
    // 要进行批量发送消息的小集合起始索引
    private int currIndex;


    public MessageListSplitter(List<Message> messages) {
        this.messages = messages;
    }


    @Override
    public boolean hasNext() {
        // 判断当前开始遍历的消息索引要小于消息总数
        return currIndex < messages.size();
    }

    @Override
    public List<Message> next() {
        int nextIndex = currIndex;
        // 记录当前要发送的这一小批次消息列表的大小
        int totalSize = 0;
        for (; nextIndex < messages.size(); nextIndex++) {
            // 获取当前遍历的消息
            Message message = messages.get(nextIndex);

            // 统计当前遍历的message的大小
            int tmpSize = message.getTopic().length() +
                    message.getBody().length;
            Map<String, String> properties = message.getProperties();
            for (Map.Entry<String, String> entry :
                    properties.entrySet()) {
                tmpSize += entry.getKey().length() +
                        entry.getValue().length();
            }
            tmpSize = tmpSize + 20;

            // 判断当前消息本身是否大于4M
            if (tmpSize > SIZE_LIMIT) {
                if (nextIndex - currIndex == 0) {
                    nextIndex++;
                }
                break;
            }


            if (tmpSize + totalSize > SIZE_LIMIT) {
                break;
            } else {
                totalSize += tmpSize;
            }

        }  // end-for


        // 获取当前messages列表的子集合[currIndex, nextIndex)
        List<Message> subList = messages.subList(currIndex, nextIndex); // 下次遍历的开始索引
        currIndex = nextIndex;
        return subList;
    }


}
```

**定义批量消息生产者**

```java
public class BatchProducer {

    public static void main(String[] args) throws Exception {
        DefaultMQProducer producer = new DefaultMQProducer("pg");
        producer.setNamesrvAddr(RocketConstant.nameservAddr);


        // 指定要发送的消息的最大大小，默认是4M
        // 不过，仅修改该属性是不行的，还需要同时修改broker加载的配置文件中的
        // maxMessageSize属性
        // producer.setMaxMessageSize(8 * 1024 * 1024);
        producer.start();


        // 定义要发送的消息集合
        List<Message> messages = new ArrayList<>();

        for (int i = 0; i < 100; i++) {
            byte[] body = ("Hi," + i).getBytes();
            Message msg = new Message("someTopiE", "someTagE", body);
            messages.add(msg);
        }

        // 定义消息列表分割器，将消息列表分割为多个不超出4M大小的小列表

        MessageListSplitter splitter = new
                MessageListSplitter(messages);


        while (splitter.hasNext()) {
            try {
                List<Message> listItem = splitter.next();
                producer.send(listItem);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }


        producer.shutdown();
    }
}
```

**定义批量消息消费者**

```java
public class BatchConsumer {


    public static void main(String[] args) throws Exception {
        DefaultMQPushConsumer consumer = new
                DefaultMQPushConsumer("cg");

        consumer.setNamesrvAddr(RocketConstant.nameservAddr);

        consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);
        consumer.subscribe("someTopiE", "*");


        // 指定每次可以消费10条消息，默认为1
        consumer.setConsumeMessageBatchMaxSize(10);
        // 指定每次可以从Broker拉取40条消息，默认为32
        consumer.setPullBatchSize(40);

        consumer.registerMessageListener(new MessageListenerConcurrently() {
            @Override
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
                for (MessageExt msg : msgs) {
                    System.out.println(msg);
                }

                // 消费成功的返回结果
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
                // 消费异常时的返回结果
                // return ConsumeConcurrentlyStatus.RECONSUME_LATER;
            }
        });

        consumer.start();
        System.out.println("Consumer Started");

    }

}
```



## 六、消息过滤

消息者在进行消息订阅时，除了可以指定要订阅消息的Topic外，还可以对指定Topic中的消息根据指定条件进行过滤，即可以订阅比Topic更加细粒度的消息类型。

### **1 Tag**过滤

通过consumer的subscribe()方法指定要订阅消息的Tag。如果订阅多个Tag的消息，Tag间使用或运算 符(双竖线||)连接。

```java
DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("CID_EXAMPLE");
consumer.subscribe("TOPIC", "TAGA || TAGB || TAGC");
```

### **2 SQL**过滤

SQL过滤是一种通过特定表达式对事先埋入到消息中的用户属性进行筛选过滤的方式。通过SQL过滤，可以实现对消息的复杂过滤。不过，只有使用PUSH模式的消费者才能使用SQL过滤。

支持的常量类型:

- 数值:比如:123，3.1415 
- 字符:必须用单引号包裹起来，比如:'abc' 
- 布尔:TRUE 或 FALSE NULL:
- 特殊的常量，表示空

支持的运算符有:

- 数值比较:>，>=，<，<=，BETWEEN，= 
- 字符比较:=，<>，IN
- 逻辑运算 :AND，OR，NOT NULL
- 判断:IS NULL 或者 IS NOT NULL

默认情况下Broker没有开启消息的SQL过滤功能，需要在Broker加载的配置文件中添加如下属性，以开 启该功能:

```
enablePropertyFilter = true
```

在启动Broker时需要指定这个修改过的配置文件。例如对于单机Broker的启动，其修改配置文件是 conf/broker.conf，启动时使用如下命令:

```
sh bin/mqbroker -n localhost:9876 -c conf/broker.conf &
```

### **3** 代码举例

定义**Tag**过滤**Producer**

```java
public class FilterByTagProducer {

    public static void main(String[] args) throws Exception {

        DefaultMQProducer producer = new DefaultMQProducer("pg");
        producer.setNamesrvAddr(RocketConstant.nameservAddr);
        producer.start();

        String[] tags = {"myTagA", "myTagB", "myTagC"};
        for (int i = 0; i < 10; i++) {
            byte[] body = ("Hi," + i).getBytes();
            String tag = tags[i % tags.length];
            Message msg = new Message("myTopic", tag, body);
            SendResult sendResult = producer.send(msg);
            System.out.println(sendResult);
        }

        producer.shutdown();
    }
}
```

**定义**Tag**过滤**Consumer

```java
public class FilterByTagConsumer {


    public static void main(String[] args) throws Exception {
        DefaultMQPushConsumer consumer = new
                DefaultMQPushConsumer("pg");
        consumer.setNamesrvAddr(RocketConstant.nameservAddr);
        consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);
        consumer.subscribe("myTopic", "myTagA || myTagB");
        consumer.registerMessageListener(new MessageListenerConcurrently() {
            @Override
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
                for (MessageExt me : msgs) {
                    System.out.println(me);
                }

                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });
        consumer.start();
        System.out.println("Consumer Started");

    }
}
```

**定义**SQL**过滤**Producer

```java
public class FilterBySQLProducer {


    public static void main(String[] args) throws Exception {
        DefaultMQProducer producer = new DefaultMQProducer("pg");
        producer.setNamesrvAddr(RocketConstant.nameservAddr);
        producer.start();
        for (int i = 0; i < 10; i++) {
            try {
                byte[] body = ("Hi," + i).getBytes();
                Message msg = new Message("myTopic", "myTag", body);
                msg.putUserProperty("age", i + "");
                SendResult sendResult = producer.send(msg);
                System.out.println(sendResult);
            } catch (Exception e) {
                e.printStackTrace();
            } }
        producer.shutdown();





    }
}
```

**定义**SQL**过滤**Consumer

```java
public class FilterBySQLConsumer {

    public static void main(String[] args) throws Exception {
        DefaultMQPushConsumer consumer = new
                DefaultMQPushConsumer("pg");
        consumer.setNamesrvAddr(RocketConstant.nameservAddr);
        consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);
        consumer.subscribe("myTopic", MessageSelector.bySql("age between 0 and 6"));
        consumer.registerMessageListener(new MessageListenerConcurrently() {
            @Override
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
                for (MessageExt me : msgs) {
                    System.out.println(me);
                }

                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });
        consumer.start();
        System.out.println("Consumer Started");

    }
}
```



## 七、消息发送重试机制

### **1** 说明

Producer对发送失败的消息进行重新发送的机制，称为消息发送重试机制，也称为消息重投机制。

对于消息重投，需要注意以下几点:

- 生产者在发送消息时，若采用 同步或异步发送 方式，发送失败 会重试 ，但oneway消息发送方式发送失败是没有重试机制的
- 只有普通消息具有发送重试机制，顺序消息是没有的 
- 消息重投机制可以保证消息尽可能发送成功、不丢失，但可能会造成消息重复。消息重复在 RocketMQ中是无法避免的问题 
- 消息重复在一般情况下不会发生，当出现消息量大、网络抖动，消息重复就会成为大概率事件 producer主动重发、consumer负载变化(发生Rebalance，不会导致消息重复，但可能出现重复 消费)也会导致重复消息
- 消息重复无法避免，但要避免消息的重复消费。 避免消息重复消费的解决方案是，为消息添加唯一标识(例如消息key)，使消费者对消息进行消 费判断来避免重复消费 
- 消息发送重试有三种策略可以选择:同步发送失败策略、异步发送失败策略、消息刷盘失败策略

### **2** 同步发送失败策略

对于普通消息，消息发送默认采用round-robin策略来选择所发送到的队列。如果发送失败，默认重试2 次。但在重试时是不会选择上次发送失败的Broker，而是选择其它Broker。当然，若只有一个Broker其 也只能发送到该Broker，但其会尽量发送到该Broker上的其它Queue。

```java
 // 创建一个producer，参数为Producer Group名称 
DefaultMQProducer producer = new DefaultMQProducer("pg"); \
  // 指定nameServer地址 
  producer.setNamesrvAddr("rocketmqOS:9876");
// 设置同步发送失败时重试发送的次数，默认为2次 
producer.setRetryTimesWhenSendFailed(3);
// 设置发送超时时限为5s，默认3s 
producer.setSendMsgTimeout(5000);
```

同时，Broker还具有 失败隔离 功能，使Producer尽量选择未发生过发送失败的Broker作为目标 Broker。其可以保证其它消息尽量不发送到问题Broker，为了提升消息发送效率，降低消息发送耗时。

如果超过重试次数，则抛出异常，由Producer去保证消息不丢。当然当生产者出现 RemotingException、MQClientException和MQBrokerException时，Producer会自动重投消息。







