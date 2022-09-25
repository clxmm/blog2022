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



