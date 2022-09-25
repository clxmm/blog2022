---
title: 03消息队列RocketMQ的应用
toc: true
tags: RocketMq
categories: 
    - [java]
    - [RocketMq]
---

## 一、普通消息

### **1** 消息发送分类

Producer对于消息的发送方式也有多种选择，不同的方式会产生不同的系统效果。

<!--more-->

#### 同步发送消息

同步发送消息是指，Producer发出一条消息后，会在收到MQ返回的ACK之后才发下一条消息。该方式的消息可靠性最高，但消息发送效率太低。

![](/img/202209/02rocketmq7.png)

#### 异步发送消息

异步发送消息是指，Producer发出消息后无需等待MQ返回ACK，直接发送下一条消息。该方式的消息可靠性可以得到保障，消息发送效率也可以。

![](/img/202209/02rocketmq8.png)

####  单向发送消息

单向发送消息是指，Producer仅负责发送消息，不等待、不处理MQ的ACK。该发送方式时MQ也不返回ACK。该方式的消息发送效率最高，但消息可靠性较差。

![](/img/202209/02rocketmq9.png)

### **2** 代码举例

消息状态

```java
public enum SendStatus {
    SEND_OK,  // 发送成功
    FLUSH_DISK_TIMEOUT, // 刷盘超时。当Broker设置的刷盘策略为同步刷盘时才可能出 现这种异常状态。异步刷盘不会出现
    FLUSH_SLAVE_TIMEOUT, // Slave同步超时。当Broker集群设置的Master-Slave的复 
  // 制方式为同步复制时才可能出现这种异常状态。异步复制不会出现
    SLAVE_NOT_AVAILABLE, // 没有可用的Slave。当Broker集群设置为Master-Slave的 复
  // 制方式为同步复制时才可能出现这种异常状态。异步复制不会出现
}

```

####  定义同步消息发送生产者
```java
public static void main(String[] args) throws Exception {
        // 创建一个producer，参数为Producer Group名称
        DefaultMQProducer producer = new DefaultMQProducer("pg");
        // 指定nameServer地址
        producer.setNamesrvAddr("192.168.42.105:9876");
        // 设置当发送失败时重试发送的次数，默认为2次
        producer.setRetryTimesWhenSendFailed(3);
        // 设置发送超时时限为5s，默认3s
        producer.setSendMsgTimeout(5_000);

        // 开启生产者
        producer.start();

        // 生产并发送100条消息
        for (int i = 0; i < 100; i++) {
            byte[] body = ("Hi," + i).getBytes();
            Message msg = new Message("someTopic", "someTag", body);

            // 发送消息
            SendResult sendResult = producer.send(msg);
            System.out.println(sendResult);

        }

        // 关闭
        producer.shutdown();
    }
```

console

```ini
SendResult [sendStatus=SEND_OK, msgId=7F00000164DA18B4AAC26672DBC70061, offsetMsgId=C0A82A6900002A9F0000000000004119, messageQueue=MessageQueue [topic=someTopic, brokerName=broker-a, queueId=2], queueOffset=24]
SendResult [sendStatus=SEND_OK, msgId=7F00000164DA18B4AAC26672DBCD0062, offsetMsgId=C0A82A6900002A9F00000000000041C3, messageQueue=MessageQueue [topic=someTopic, brokerName=broker-a, queueId=3], queueOffset=24]
SendResult [sendStatus=SEND_OK, msgId=7F00000164DA18B4AAC26672DBD20063, offsetMsgId=C0A82A6900002A9F000000000000426D, messageQueue=MessageQueue [topic=someTopic, brokerName=broker-a, queueId=0], queueOffset=24]

```



#### 定义异步消息发送生产者

```java
public class AsyncProducer {


    /**
     * 异步消息生产
     */
    public static void main(String[] args) throws Exception {
        // 创建一个producer，参数为Producer Group名称
        DefaultMQProducer producer = new DefaultMQProducer("pg");
        // 指定nameServer地址
        producer.setNamesrvAddr("192.168.42.105:9876");
        // 设置当发送失败时重试发送的次数，默认为2次
        producer.setRetryTimesWhenSendAsyncFailed(3);
        // 设置发送超时时限为5s，默认3s
        producer.setSendMsgTimeout(5_000);
        // 指定新创建的Topic的Queue数量为2，默认为4
        producer.setDefaultTopicQueueNums(2);

        // 开启生产者
        producer.start();

        // 生产并发送100条消息
        for (int i = 0; i < 100; i++) {
            byte[] body = ("Hi," + i).getBytes();
            Message msg = new Message("asyncTopic", "asyncTag", body);

            msg.setKeys("key-" + i);

            // 发送消息
            producer.send(msg, new SendCallback() {
                @Override
                public void onSuccess(SendResult sendResult) {
                    System.out.println(sendResult);
                }

                @Override
                public void onException(Throwable e) {
                    e.printStackTrace();
                }
            });

        }

        /**
         * sleep一会儿
         *  由于采用的是异步发送，所以若这里不sleep，
         *  则消息还未发送就会将producer给关闭，报错
         */
        TimeUnit.SECONDS.sleep(3);
        // 关闭
        producer.shutdown();


    }
}
```

#### 定义单向消息发送生产者

```java
 /**
     * 单向消息生产者
     */
    public static void main(String[] args) throws Exception {
        DefaultMQProducer producer = new DefaultMQProducer("pg");
        producer.setNamesrvAddr("192.168.42.105:9876");

        producer.start();

        for (int i = 0; i < 10; i++) {
            byte[] bytes = ("hi" + i).getBytes();
            Message message = new Message("onewayTopic", "someTag", bytes);
            producer.sendOneway(message);
        }

        producer.shutdown();

        System.out.println("end");

    }
```

#### 定义消息消费者

```java
public class SomeConsumer {


    /**
     * 消费者消费
     */
    public static void main(String[] args) throws Exception {

        // // 定义一个pull消费者
        DefaultLitePullConsumer pullConsumer = new DefaultLitePullConsumer("cg");

        // 定义一个push消费者
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("cg");
        consumer.setNamesrvAddr(RocketConstant.nameservAddr);
        // 指定从第一条消息开始消费
        consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);
        // 指定消费topic与tag
        consumer.subscribe("someTopicC", "*");

        // 指定采用“广播模式”进行消费，默认为“集群模式”
//        consumer.setMessageModel(MessageModel.BROADCASTING);

        consumer.registerMessageListener(new MessageListenerConcurrently() {
            // 一旦broker中有了其订阅的消息就会触发该方法的执行，
            // 其返回值为当前consumer消费的状态
            @Override
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
                // 逐条消费消息
                for (MessageExt msg : msgs) {
                    System.out.println(msg);
                }
                // 返回消费状态:消费成功
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });
        // 开启消费者消费
        consumer.start();
        System.out.println("Consumer Started");

    }
}

```



## 二、顺序消息

### **1** 什么是顺序消息

顺序消息指的是，严格按照消息的 发送顺序 进行 消费 的消息(FIFO)。

默认情况下生产者会把消息以Round Robin轮询方式发送到不同的Queue分区队列;而消费消息时会从多个Queue上拉取消息，这种情况下的发送和消费是不能保证顺序的。如果将消息仅发送到同一个Queue中，消费时也只从这个Queue上拉取消息，就严格保证了消息的顺序性。

### **2** 为什么需要顺序消息

例如，现在有TOPIC ORDER_STATUS(订单状态)，其下有4个Queue队列，该Topic中的不同消息用于描述当前订单的不同状态。假设订单有状态: 未支付 、 已支付 、 发货中 、 发货成功 、 发货失败 。

根据以上订单状态，生产者从 时序 上可以生成如下几个消息:

订单T0000001:未支付 --> 订单T0000001:已支付 --> 订单T0000001:发货中 --> 订单T0000001:发货失败

消息发送到MQ中之后，Queue的选择如果采用轮询策略，消息在MQ的存储可能如下:

![](/img/202209/02rocketmq10.png)

 这种情况下，我们希望Consumer消费消息的顺序和我们发送是一致的，然而上述MQ的投递和消费方 式，我们无法保证顺序是正确的。对于顺序异常的消息，Consumer即使设置有一定的状态容错，也不 能完全处理好这么多种随机出现组合情况。

![](/img/202209/02rocketmq11.png)

基于上述的情况，可以设计如下方案:对于相同订单号的消息，通过一定的策略，将其放置在一个 Queue中，然后消费者再采用一定的策略(例如，一个线程独立处理一个queue，保证处理消息的顺序 性)，能够保证消费的顺序性。

### **3** 有序性分类

根据有序范围的不同，RocketMQ可以严格地保证两种消息的有序性:分区有序与全局有序。

#### 全局有序

![](/img/202209/02rocketmq12.png)

当发送和消费参与的Queue只有一个时所保证的有序是整个Topic中消息的顺序， 称为全局有序。

> 创建Topic指定Queue的数量
>
> - 在代码中，创建Producer时，可以指定其自动创建的Topic的Queue的数量
> - 在可视化工具中设置
> - 使用mqadmin命令创建Topic时指定Queue的数量

#### 分区有序

![](/img/202209/02rocketmq13.png)

如果有多个Queue参与，其仅可保证在该Queue分区队列上的消息顺序，则称为分区有序。

> 如何实现Queue地选择？在定义Producer时我们可以指定消息队列选择器，而这个选择器是我们自己实现了MessageQueueSelector接口定义的
>
> 在定义选择器的算法时，一般需要使用选择key，这个key可以是消息的key，也可以时其它数据，但无论谁做选择key，都不能重复，都是唯一的。
>
> 一般性的选择算法时，让选择key（或其它hash值）与该Topic所包含的Queue的数量取模，其结果即位选择出的Queue的QueueId
>
> 取模算法存在的问题，不同选择key与queue数量取模结果可能会是相同的，即不同选择key的消息可能会出现在相同的Queue，即同一个Consumer可能会消费到不同选择key的消息，这个问题如何解决？一般性的做法是，从消息中获取到选择key，对其进行判断，若是当前的Consumer需要消费的消息，则直接消费，否则，什么也不做，这种做法，要求选择key要能够随着消息一起被Consumer获取到，此时使用消息key作为选择key是比较好的做法。
>
> 以上做法会出现的问题？不属于那个Consumer的消息被拉取走了，那么应该消费该消息的Consumer是否还能再消费该消息？同一个Queue的消息不可能被同一个Group中的不同Consumer同时消费。所以，消费现一个Group的不同选择key的消息的Consumer一定属于不同的Group。而不同的Group中的Consumer间的消费是相互隔离的，互不影响。

### **4** 代码举例

```java
public static void main(String[] args) throws Exception {
        DefaultMQProducer producer = new DefaultMQProducer("pg");
        producer.setNamesrvAddr(RocketConstant.nameservAddr);
        producer.start();

        for (int i = 0; i < 100; i++) {
            // 演示，使用整型数作为orderId
            Integer orderId = i;
            byte[] body = ("Hi," + i).getBytes();
            Message message = new Message("TopicA", "TagA", body);
            SendResult sendResult = producer.send(message, new MessageQueueSelector() {
                // 具体的选择算法
                @Override
                public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
                    Integer id = (Integer) arg;
                    System.out.println("--" + id);
                    int index = id % mqs.size();
                    return mqs.get(index);
                }
            }, orderId);
            System.out.println(sendResult);
        }

        producer.shutdown();


    }
```

## 三、延时消息

### **1** 什么是延时消息

当消息写入到Broker后，在指定的时长后才可被消费处理的消息，称为延时消息。

采用RocketMQ的延时消息可以实现 定时任务 的功能，而无需使用定时器。典型的应用场景是，电商交易中超时未支付关闭订单的场景，12306平台订票超时未支付取消订票的场景。

### **2** 延时等级

延时消息的延迟时长 不支持随意时长 的延迟，是通过特定的延迟等级来指定的。延时等级定义在RocketMQ服务端的 MessageStoreConfig 类中的如下变量中:

即，若指定的延时等级为3，则表示延迟时长为10s，即延迟等级是从1开始计数的。

当然，如果需要自定义的延时等级，可以通过在broker加载的配置中新增如下配置(例如下面增加了1天这个等级1d)。配置文件在RocketMQ安装目录下的conf目录中。

```java
messageDelayLevel = 1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h 1d
```



### **3** 延时消息实现原理

![](/img/202209/02rocketmq14.png)

 具体实现方案是:

**修改消息**

![](/img/202209/02rocketmq15.png)

Producer将消息发送到Broker后，Broker会首先将消息写入到commitlog文件，然后需要将其分发到相 应的consumequeue。不过，在分发之前，系统会先判断消息中是否带有延时等级。若没有，则直接正 常分发;若有则需要经历一个复杂的过程:

- 修改消息的Topic为SCHEDULE_TOPIC_XXXX

- 根据延时等级，在consumequeue目录中SCHEDULE_TOPIC_XXXX主题下创建出相应的queueId目录与consumequeue文件(如果没有这些目录与文件的话)。

  > 延迟等级*delayLevel queueId queueId = delayLevel -1*
  >
  > 录目个哪建创级等迟延个哪到用是而 ，毕完建创部全录目的应对级等迟延有所将地性次一是不并，时录目 建创在，意注要需

- 修改消息索引单元内容。索引单元中的Message Tag HashCode部分原本存放的是消息的Tag的 Hash值。现修改为消息的 投递时间 。投递时间是指该消息被重新修改为原Topic后再次被写入到

  commitlog中的时间。投递时间 = 消息存储时间 + 延时等级时间。消息存储时间指的是消息

  被发送到Broker时的时间戳。

- 将消息索引写入到SCHEDULE_TOPIC_XXXX主题下相应的consumequeue中

**投递延时消息**

Broker内部有一个延迟消息服务类ScheuleMessageService，其会消费SCHEDULE_TOPIC_XXXX中的消 息，即按照每条消息的投递时间，将延时消息投递到目标Topic中。不过，在投递之前会从commitlog 中将原来写入的消息再次读出，并将其原来的延时等级设置为0，即原消息变为了一条不延迟的普通消 息。然后再次将消息投递到目标Topic中。



**将消息重新写入**commitlog

延迟消息服务类ScheuleMessageService将延迟消息再次发送给了commitlog，并再次形成新的消息索引条目，分发到相应Queue。

> 这其实就是一次普通消息发送，只不过这次消息Producer是延迟消息服务类*ScheuleMessageService*

### **4** 代码举例

**定义**DelayProducer**类**

```java
public class DelayProducer {


    /**
     * 延迟消息
     */
    public static void main(String[] args) throws Exception {
        DefaultMQProducer producer = new DefaultMQProducer("pg");
        producer.setNamesrvAddr(RocketConstant.nameservAddr);
        producer.start();


        for (int i = 0; i < 10; i++) {

            byte[] body = ("Hi-" + i).getBytes();

            Message message = new Message("TipiC", "tagC", body);
            // 指定消息延迟等级为3级，即10s
            message.setDelayTimeLevel(3);

            SendResult send = producer.send(message);

            // 输出消息发送时间
            System.out.println(LocalDateTime.now().toString());
            System.out.println(send);

        }
        producer.shutdown();
    }


}
```

定义**OtherConsumer**类

```java
public class OtherConsumer {


    public static void main(String[] args) throws Exception {

        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("cg");
        consumer.setNamesrvAddr(RocketConstant.nameservAddr);


        consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);

        consumer.subscribe("TipiC", "*");

        consumer.registerMessageListener(new MessageListenerConcurrently() {
            @Override
            public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {

                for (MessageExt msg : msgs) {
                    System.out.println(LocalDateTime.now().toString());
                    System.out.println(msg);
                }

                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
            }
        });


        consumer.start();

        System.out.println("consumer started");


    }


}
```





