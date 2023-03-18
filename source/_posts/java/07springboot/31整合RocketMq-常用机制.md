---
title: 31整合RocketMq-常用机制.md
toc: true
tags: springboot
categories: 
    - [java]
    - [springboot]
---

上一章中我们已经安装部署了 RocketMQ ，并且用 SpringBoot 整合了 RocketMQ ，也体会了一些消息的发送和监听。本章我们继续来体验一些 RocketMQ 的特性和机制。

## 1. 自定义消息格式

---

上一章中我们发送的消息一直都是 `String` 类型的，其实我们完全可以发送一些 Entity / VO / DTO 类型的模型对象，作为数据传递的消息正文内容。下面我们就来试验一下。

#### 1.1 编码发送消息

我们可以简单构造一个 DTO ，用来封装一个最简单的用户信息：

<!--more-->


```java
public class UserDTO {
    
    private Integer userId;
    
    private String userName;
    
    private String sexName;
    
    // ......
}
```

随后，在发送消息时，就不再发送字符串了，而是直接把这个 DTO 对象发过去：

```java
    public void sendDto() {
        // 构造发送一个DTO模型
        UserDTO dto = new UserDTO();
        dto.setUserId(1);
        dto.setUserName("haha");
        dto.setSexName("male");
        rocketMQTemplate.convertAndSend("test-dto", dto);
    }
```

对应的 Controller 接口，调用这个 `sendDto` 方法也编写好，代码很简单，小册就不再贴出了。

编写完毕后，重启生产者工程，调用 `sendDTO` 方法后观察 RocketMQ 的 dashboard ，可以发现 DTO 对象被转换为 json 字符串保存到消息正文了：

#### 1.2 消费消息

消息发送出去了，如何将这条消息消费掉呢？方法依然是编写 `RocketMQListener` ，只不过这里的泛型就有两种方式可选了：要么继续用 `String` 接收，之后手动执行 json 转对象的方式拿到原始 DTO 对象；要么直接在泛型类型上标注 DTO 对象。下面的代码中我们使用第二种方式，直接以 `UserDTO` 的形式接收。

```java
@Component
@RocketMQMessageListener(topic = "test-dto", consumerGroup = "dtoconsumer")
public class UserDtoMessageReceiver implements RocketMQListener<UserDTO> {
    
    @Override
    public void onMessage(UserDTO message) {
        System.out.println("接收到DTO消息");
        System.out.println(message.toString());
    }
}
```

> 注意不要写错 `topic` ，以及 `consumerGroup` 要跟上一章的几个区分开。

编写完毕后，我们就可以直接启动消费者工程了，由于此时 RocketMQ 中已经有一条消息了，所以在启动时就会打印这条消息的数据：

```ini
接收到DTO消息
org.clxmm.rocketmq.dto.UserDTO@f4bdb36
```

可以发现，即便是包名不匹配的情况下，json 数据依然可以正常反序列为对象，说明我们测试自定义消息类型已经成功了。

## 2. 延迟消息发送

---

延迟消息？可能各位会觉得很奇怪，消息还有延迟一说吗？是的，在一些特殊的业务场景中，通常会设计一种延迟规则，比方说我们在点外卖的时候，如果各位点了外卖但没有支付，通常平台会告诉你需要在 15 分钟内支付，否则订单将会被取消。这种业务场景就可以使用延迟消息来实现，由于延迟消息是在指定时间之后定时触发，所以也可以称为 “定时消息” 。

下面也来试一下 RocketMQ 的延迟消息的效果。

#### 2.1 发送延迟消息

使用 `RocketMQTemplate` 发送延迟消息的方式很简单，只需要在 `syncSend` 或者 `asyncSend` 方法的原有参数后面追加 2 个参数即可，分别是消息发送的最大等待时间，以及延迟的时间档位。RocketMQ 中内置了 18 个延迟时间档位，分别是 1s / 5s / 10s / 30s / 1m / 2m / 3m / 4m / 5m / 6m / 7m / 8m / 9m / 10m / 20m / 30m / 1h / 2h ，这些档位都是内置的，通常我们只能选择，没办法自行指定。

下面我们就简单演示一下延迟消息的发送吧，这次我们一次性把同步发送和异步发送的方式都列举出来测试。

```java
    public void sendDelayMessage() {
        // 同步发送延迟消息
        Message<String> message = MessageBuilder.withPayload("delay message").build();
        rocketMQTemplate.syncSend("test-delay", message, 3000, 2); // 此处用第2档 5秒
        System.out.println("同步延迟消息发送成功！时间：" + System.currentTimeMillis());
        
        // 异步发送延迟消息
        rocketMQTemplate.asyncSend("test-delay", message, new SendCallback() {
            @Override
            public void onSuccess(SendResult sendResult) {
                System.out.println("异步延迟消息发送成功！时间：" + System.currentTimeMillis());
                System.out.println(sendResult.toString());
            }
    
            @Override
            public void onException(Throwable e) {
                System.out.println("异步延迟消息发送失败！");
                e.printStackTrace();
            }
        }, 3000, 3); // 此处用第3档 10秒
    }
```

为了测试跟消费者的配合，所以现在我们不触发这个方法，先把下面的消费者代码写完。

#### 2.2 消费延迟消息

消费者的消息监听器与前面的编写并无二样，不过我们在接收到消息的时候要打印一下时间戳，以对比和计算一下消息发送和消费的时间差。

```java
@Component
@RocketMQMessageListener(topic = "test-delay", consumerGroup = "delayconsumer")
public class DelayMessageReceiver implements RocketMQListener<String> {
    
    @Override
    public void onMessage(String message) {
        System.out.println("接收到消息：" + message);
        System.out.println(System.currentTimeMillis());
    }
}
```

这样所有的代码就都编写完了，我们可以把生产者和消费者都启动起来。

用浏览器触发 `sendDelayMessage` 方法，首先生产者的消息会打印发送成功，但是消费者那边却没有动静，大概等 5 秒左右第一条消息会被消费者消费掉并有控制台打印，10 秒后第二条消息也成功被消费：

```ini
接收到消息：delay message
1678609004800

接收到消息：delay message
1678609009851

```

这就是延迟消息的使用方式。

## 3. 消息重试机制

消息重试，这个机制相对容易理解。由于代码出错、网络等特殊原因，消息从生产者发送给消费者后，在被消费者消费时会出现无法正常消费的情况，这时 RocketMQ 会尝试重新将消息推送给消费者。

#### 3.1 重试的场景和策略

通常情况下，消息重试会在两种情况下发生：

- 消息在被消费者接收的时候，因为网络中断、异常等问题，导致消息接收不到，这种情况下消息会一直被重试
- 消息已经被消费者拿到，但是在 `onMessage` 方法中消费时抛出了异常，那么这种情况下 Broker 无法收到消费者消费成功的反馈，会不断向消费者发消息重试

注意一点，RocketMQ 如何知道消息应该发给消费者重试呢？依据就是消费者有没有给 Broker 反馈消息消费成功，正常来讲当消费者把 RocketMQ 中的消息消费完成后，会告诉 RocketMQ 此条消息消费成功，RocketMQ 便会标记这条消息的状态。如果消费者中途抛出了异常，那就没办法告诉 RocketMQ ，RocketMQ 发现消费者拉取了消息但不告诉他成功了没，就会反复给消费者推送消息。

当然，RocketMQ 重试消息也不是一直没完的重试，它会在第一次消息消费失败的 10s 后将消息再次发给消费者，如果消费者在消费这条消息的时候又失败了，则会向后顺延一个等级（即 30s ），等待 30s 后再重试，一直持续到 2h 为止（重试的时间等级就是延迟消息的延迟等级后 16 个）。如果最后到了 2h 的等级重试还失败，那么这条消息就会被投放到死信队列中，意味着这条消息已经“死掉了”。

#### 3.2 测试消息重试

下面我们也来测试一下消息的重试机制。这次我们不需要动生产者的代码，只需要改一下消费者的代码，在这其中添加一个除零异常即可。

```java
@Component
@RocketMQMessageListener(topic = "test-sender", consumerGroup = "myconsumer")
public class MessageReceiver implements RocketMQListener<String> {
    
    @Override
    public void onMessage(String message) {
        System.out.println(System.currentTimeMillis());
        System.out.println("收到消息：" + message);
        int i = 1 / 0;
    }
}
```

之后我们重启消费者的工程，之后重新触发生产者的 `/sendMessage` ，让生产者向 `test-sender` 发送一条消息，可以发现由于触发了除零异常，消息没有成功被消费掉，于是在第一次 10 秒后再一次触发了消息的第二次消费，然而第二次肯定也是消费不成功的，于是在 30 秒后继续触发第三次。。。

```ini
1678609294385
收到消息：test oneway message
2023-03-12 16:21:34.385  WARN 43349 --- [ad_myconsumer_2] a.r.s.s.DefaultRocketMQListenerContainer : consume message failed. messageId:7F000001A90F18B4AAC23C289BDF0005, topic:test-sender, reconsumeTimes:0

java.lang.ArithmeticException: / by zero
	
	1678609304435
收到消息：test message
2023-03-12 16:21:44.435  WARN 43349 --- [ad_myconsumer_3] a.r.s.s.DefaultRocketMQListenerContainer : consume message failed. messageId:7F000001A90F18B4AAC23C289B8B0004, topic:test-sender, reconsumeTimes:1

java.lang.ArithmeticException: / by zero
	
	1678609304594
收到消息：test oneway message
2023-03-12 16:21:44.594  WARN 43349 --- [ad_myconsumer_4] a.r.s.s.DefaultRocketMQListenerContainer : consume message failed. messageId:7F000001A90F18B4AAC23C289BDF0005, topic:test-sender, reconsumeTimes:1

java.lang.ArithmeticException: / by zero
	at org.clxmm.rocketmq.consumer.config.MessageReceiver.onMessage(MessageReceiver.java:21) ~[classes/:na]
	at org.clxmm.rocketmq.consumer.config.MessageReceiver.onMessage(MessageReceiver.java:12) ~[classes/:na]

```

另外注意一点，控制台打印的警告日志中可以发现，`reconsumeTimes` 这个值在递增，这个就是重试次数。

这样看来，消息重试机制就已经正确生效了，如果消费者工程一直运行，则我们还会看到 14 次消息的重新消费。

#### 3.3 生产中的处理方式

虽说 RocketMQ 会贴心的给我们重试 16 次消息，但是实际在生产项目中肯定不能让它一直这么干，毕竟有一些业务场景中，消息的消费过程是复杂且耗时长的，倘若一直重试，那么对服务器的影响不能说多严重，但多少还是有些影响的。

怎么优化呢？我们可以这样，我们在每次消息发送的时候检查一下当前的消息是第几次推送的，上面我们已经看到了，第一次推送消息的时候 `reconsumeTimes` 的值是 0 ，第一次重试的时候 `reconsumeTimes` 的值是 1 ，那我们可以限定最多重试 3 次，超过 3 次的话就把这条消息记录到数据库中，且正常返回消费成功，这样至少 RocketMQ 就不会再继续重试，而消息最终如何处理，由我们人工来介入处理即可。

如何获取到 `reconsumeTimes` 呢？这就需要我们在编写消息监听器的时候，泛型做一个小小的改动，之前是直接指定消息的模型类型，而改动后的泛型为固定的类型 `MessageExt` 。

```java
@Override
public void onMessage(MessageExt ext) {
  int reconsumeTimes = ext.getReconsumeTimes();
  if (reconsumeTimes > 3) {
    // 将消息保存到数据库
    // 直接返回，代表消息已经消费成功
    return;
  }
  System.out.println(System.currentTimeMillis());
  String message = new String(ext.getBody(), StandardCharsets.UTF_8);
  System.out.println("收到消息：" + message);
  int i = 1 / 0;
}
```

如此改造后，即可实现多次失败的消息保存到数据库，而消息正文的提取，需要我们先将 `byte[]` 转换为 `String` ，如果再需要转模型对象，则可以利用 json 工具类来转换。

## 4. 事务消息

---

RocketMQ 4.3 版本之后开始完整支持事务消息，有了这个事务消息，我们就可以解决一种分布式事务的场景。

#### 4.1 分布式事务的场景

当传统的单体项目拆分为微服务后，基本上必定会出现跨服务的业务，那么为了保证整个业务的事务完整性，就需要引入分布式事务。

比方说一个经典的场景：转账，A 服务需要扣减指定用户的金额，B 服务负责增加指定用户的金额。如果这个场景中 A 服务执行成功，而 B 服务执行失败，则应当将 A 服务的扣减动作回滚。如果没有分布式事务的控制，则 A 服务的扣减金额无法回滚，造成用户的财产损失。

再比方说：支付成功后创建订单，支付系统支付完成后，订单系统必须要把订单创建出来，而不是回滚支付的逻辑。只要支付完成，则订单必须创建，这也是另一种分布式事务的场景。与上面不同的是，支付成功创建订单的这个场景的最突出特点：A 业务可以成功可以失败，失败了直接回滚即可，但是当 A 业务成功发生后，后面的 B 业务必须要发生。

通常情况上，分布式事务有 3 种分类：

- 多个服务连接多个数据库
- 多个服务同时连一个数据库
- 一个服务同时连多个数据库

针对不同的场景分类，对应的分布式事务解决方案也不完全相同。既然我们是讲解 RocketMQ ，那下面我们就说说 RocketMQ 的事务消息是如何解决分布式事务的。

#### 4.2 RocketMQ解决分布式事务的方式

RocketMQ 解决的分布式事务场景，主要是上面咱提到的第 2 种业务场景：A 业务成功发生后 B 业务必须发生。RocketMQ 对于这种场景的处理逻辑如下图所示。

![](./img/202303/31mq1.png)

- 分布式事务的发起方（也是消息的发送方）给 RocketMQ 发送一条 half 半消息，代表即将执行本地事务，此时这条消息的状态不是正常允许消费的，是一个 half （unknown）状态，RocketMQ 不会将这条消息推送给消费方，消费方也无法监听到这条消息；
- RocketMQ 给消息发送方响应 half 消息发送成功的状态，这个时候消息的发送方就确定可以执行本地事务了，而此时发送的那条消息依然是 half 消息；
- 分布式事务的发起方执行本地事务并尝试提交，这个过程可能成功也可能失败；
- 如果分布式事务的发起方的本地事务成功提交，则会给 RocketMQ 发送一个二次确认的消息，告诉 RocketMQ 本地事务已成功提交，RocketMQ 会相应的将这条 half 消息转换为正常消息，并投递给消息的接收方；如果分布式事务的发起方执行本地事务失败，则同样会给 RocketMQ 发送一个二次确认消息，告诉 RocketMQ 本地事务执行失败了，消息没有必要推送，RocketMQ 会相应的将这条 half 消息作废丢弃，消息的消费方也就永远不知道这条消息的存在了；
- 在少数特殊场景中，分布式事务的发起方提交事务后与 RocketMQ 断开连接，此时 RocketMQ 不知道这条事务是否成功提交，则 RocketMQ 会反向查询消息发送方，询问对应事务的提交状态；
- 消息发送方检查本地事务状态后，并告诉 RocketMQ 当前该本地事务的状态是 commit 还是 rollback ；
- RocketMQ 收到消息发送方反馈的本地事务状态后，就可以作出第 4 步的反应了。

#### 4.3 测试事务消息

了解了 RocketMQ 处理事务消息的机制，下面我们就来实际的测试一下。

##### 4.3.1 事务发起方（消息生产方）

为了能正常使用事务，我们在消息的生产方引入 jdbc 的场景启动器，以及 MySQL 的驱动：

```xml
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-jdbc</artifactId>
    </dependency>
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
    </dependency>
```

对应的数据库连接配置不要忘记配置，小册就不再贴出了，拿之前 MyBatis 的部分即可。

```properties
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
spring.datasource.url=jdbc:mysql://127.0.0.1:3306/springboot-dao?characterEncoding=utf8
spring.datasource.username=root
spring.datasource.password=root
```

之后我们来编写一个模拟数据库操作的事务方法，在这里面我们只是在方法上标注 `@Transactional` 注解，不实际操作数据库。

```java
@Service
public class DataService {
    
    @Transactional(rollbackFor = Exception.class)
    public void saveData() {
        System.out.println("执行数据库insert操作。。。");
    }
}
```

再往下就是事务消息的发起了。根据上面的逻辑来看，我们应该先发事务 half 半消息，后等待 RocketMQ 确认后再执行本地事务操作，所以我们先在 `MessageSender` 中编写一个事务消息的发送动作：

```java
    public void sendTransactionalMessage() {
        // 第1步：发送事务消息，为了能识别出某个事务消息对应的本地事务，此处生成一个uuid设置到消息头上
        String txId = UUID.randomUUID().toString();
        rocketMQTemplate.sendMessageInTransaction("test-transactional",
                MessageBuilder.withPayload("事务消息").setHeaderIfAbsent("txId", txId).build(), txId);
        // 第三个参数可以传入任意参数，此处来传入对应的事务标识uuid（根据业务需求来，通常不传）
    }
```

发送事务消息的方法是 `sendMessageInTransaction` ，注意这个方法不能直接放消息正文，而是我们自己使用 `MessageBuilder` 来构建消息实体。为了能唯一识别区分每个本地事务，我们在发送事务消息的时候，在消息头上放了一个 txId 作为事务唯一标识。另外最后一个参数 `arg` 可以传任意内容，这个数据会在第 3 步执行本地事务时回传回来。

那第 3 步如何触发呢？RocketMQ 给我们提供了一个回调接口机制：`RocketMQLocalTransactionListener` ，这个事务监听器具备执行本地事务和查询本地事务状态的能力。执行本地事务代码（也就是执行 Service 中标注了 `@Transactional` 注解的方法）正是在这里面回调。

那下面我们就来编写一个 `RocketMQLocalTransactionListener` 的实现类，并在类上标注一个 `@RocketMQTransactionListener` 注解，这都是 RocketMQ 要求我们做的。这个接口有两个需要我们实现的方法，`executeLocalTransaction` 就是对应第 3 步的执行本地事务，`checkLocalTransaction` 方法则是对应第 6 步的本地事务是否执行完毕。

```java
@RocketMQTransactionListener
public class TransactionalStatusChecker implements RocketMQLocalTransactionListener {

    @Autowired
    private DataService dataService;

    @Override
    public RocketMQLocalTransactionState executeLocalTransaction(Message msg, Object arg) {
        String payload = new String((byte[]) msg.getPayload(), StandardCharsets.UTF_8);
        System.out.println(payload);
        System.out.println(arg);

        RocketMQLocalTransactionState state = RocketMQLocalTransactionState.COMMIT;
        try {
            dataService.saveData();
        } catch (Exception e) {
            state = RocketMQLocalTransactionState.UNKNOWN;
        }
        return state;
    }

    @Override
    public RocketMQLocalTransactionState checkLocalTransaction(Message msg) {
        return null;
    }
}
```

我们先实现第一个方法 `executeLocalTransaction` ，为了检验消息和 arg 是否都正常传递过来，我们在方法的一开始打印一下（注意刚拿到的是 `byte[]` ，需要转成 `String` ）；之后我们就可以调用 Service 的方法执行本地事务了。此处我们就发现问题了！如果 Service 的方法中只处理业务数据，那过会检查本地事务是否执行成功的时候就麻爪了，因为我们根本就没有可以判断的依据！所以这个时候就需要改造 `saveData` 方法，接收我们之前在发送事务消息的时候传入的 **txId** 并保存到数据库中！

```java
@Service
public class DataService {
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    @Transactional(rollbackFor = Exception.class)
    public void saveData(String data, String txId) {
        System.out.println("执行数据库insert操作。。。");
        System.out.println(data);
        System.out.println("保存txId......");
        jdbcTemplate.update("insert into tx_id (id) value (?)", txId);
    }
```

> 对应的 tx_id 表可以设计的非常简单：
>
> ```sql
> CREATE TABLE `tx_id` (
>   `id` varchar(64) NOT NULL,
>   PRIMARY KEY (`id`) USING BTREE
> ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
> ```

因为保存数据和保存 **txId** 是在一个事务中，要么全部成功要么全部不执行，所以后面再检查本地事务状态的时候，我们就只需要检查数据库里是否有这个 txId 即可。

```java
    public boolean contains(String txId) {
        return jdbcTemplate.queryForObject("select count(1) from tx_id where id = ?", Integer.class, txId) > 0;
    }

    @Override
    public RocketMQLocalTransactionState checkLocalTransaction(Message msg) {
        return dataService.contains(msg.getHeaders().get("txId", String.class))
                ? RocketMQLocalTransactionState.COMMIT
                : RocketMQLocalTransactionState.ROLLBACK;
    }
```

回过头来，我们再改一下 `TransactionalStatusChecker` 的代码，给 `saveData` 方法传参：

```java
    @Override
    public RocketMQLocalTransactionState executeLocalTransaction(Message msg, Object arg) {
        String payload = new String((byte[]) msg.getPayload(), StandardCharsets.UTF_8);
        System.out.println(payload);
        System.out.println(arg);
        
        RocketMQLocalTransactionState state = RocketMQLocalTransactionState.COMMIT;
        try {
            dataService.saveData(payload, msg.getHeaders().get("txId", String.class));
        } catch (Exception e) {
        	state = RocketMQLocalTransactionState.UNKNOWN;
        }
        return state;
    }
```

这样我们就算完成了事务发起方的所有代码编写。

##### 4.3.2 消息消费方

相对比起来，消息的消费方代码就简单的多了，跟监听普通消息并没有什么区别。

```java
@Component
@RocketMQMessageListener(topic = "test-transactional", consumerGroup = "transactionalconsumer")
public class TransactionalReceiver implements RocketMQListener<String> {
    
    @Override
    public void onMessage(String message) {
        System.out.println("接收到事务消息！");
        System.out.println(message);
    }
}
```

只需要最基本的监听即可，后续就是执行它这一部分的本地事务即可。

##### 4.3.3 测试效果

下面我们来测试一下事务消息是否可以按照上面所述的规则正确发送。

**无异常正常推送事务消息**

首先我们测试在没有异常发生的情况下，消息是否可以正常发送。编写对应的 Controller 方法，让它触发 `MessageSender` 的 `sendTransactionalMessage` 方法，观察控制台的打印：

```ini
事务消息
a142fe24-46eb-4ad8-b692-c7ff32c3098c

执行数据库insert操作。。。
事务消息
保存txId......
```

可以发现，前两行是在 `TransactionalStatusChecker` 的 `executeLocalTransaction` 方法中打印的内容，而后三行是进入到 `DataService` 中打印的。

同时来到消费者这边，可以发现消息已经成功推送过来了，`TransactionalReceiver` 也接收到消息了。

```ini
接收到事务消息！
事务消息
```

这就说明，在一切正常的情况下，事务消息可以被正确推送。

**出现业务异常不推送消息**

下面我们在事务代码中增加一个异常，模拟业务代码中出现了未预见的错误情况。

```java
@Service
public class DataService {
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    @Transactional(rollbackFor = Exception.class)
    public void saveData(String data, String txId) {
        System.out.println("执行数据库insert操作。。。");
        System.out.println(data);
        System.out.println("保存txId......");
        jdbcTemplate.update("insert into tx_id (id) value (?)", txId);
        int i = 1 / 0;
    }
```

之后重启 Producer 工程，重新触发事务消息，这次发送虽然 Producer 控制台的内容依然正常打印，但 Consumer 中的控制台却一点反应也没有了。

这也就证明了事务发起方的本地事务执行失败时，整个分布式事务就不会触发了。

#### 4.4 测试多个主题的事务消息场景

虽说讲到这里，有关 RocketMQ 的事务消息就可以算作讲完了，但是阿熊还是想多讲一个小节。毕竟我们在项目开发中，**不可能一整个项目只有一个分布式事务的场景吧**！那面对多个不同的业务场景下，我们又该如何用 RocketMQ 来区分呢？

##### 4.4.1 无法区分业务场景的痛点

很明显，发送消息的时候指定不同的 topic 可以区分，消费者在监听消息队列主题时也可以区分，最头疼的问题是 `RocketMQLocalTransactionListener` 这里，因为一个项目中只能注册一个 `RocketMQLocalTransactionListener` （不信邪的小伙伴可以自行测试一下，如果注册多个，会在启动过程中抛出异常），只有一个 Listener 就没办法从 bean 的范围区分了，那我们只好在方法体内部来区分。

但问题又来了，方法体内部区分又怎么区分呢？我们怎么知道调用哪个 Service 的哪个方法呢？这些逻辑如果放到 `RocketMQLocalTransactionListener` 中，那代码复杂度可想而知得多复杂。。。

那问题就来了，有什么好办法能区分呢？

##### 4.4.2 解决方案：利用不同的Template区分

RocketMQ 自然帮我们考虑到了这一点吗，它给出的方案是用不同的 `RocketMQTemplate` 来区分不同的业务场景：一个分布式事务场景用一个单独的 `RocketMQTemplate` 来发送事务消息。这挺上去貌似挺离谱的，但的确只能这么办。

下面我们来演示一下。

##### 4.4.3 扩展RocketMQTemplate实现业务场景区分

扩展 `RocketMQTemplate` 的方式不像我们之前用 `@Bean` 方法的方式注册，而是通过 RocketMQ 给我们提供的一个特殊注解来标注。在扩展时，我们需要编写一个 `RocketMQTemplate` 的子类，并在这个类上标注 `@ExtRocketMQTemplateConfiguration` 注解。

```java
@ExtRocketMQTemplateConfiguration
public class DataUpdateRocketMQTemplate extends RocketMQTemplate {
    
}
```

之后在区分业务场景时，我们需要注入不同的 Template 并使用，比如下面我们来演示一个数据更新的场景。在注入 `RocketMQTemplate` 时，我们需要使用 `@Autowired` 和 `@Qualifer` 注解（或只使用 `@Resource` 注解）来强制指定扩展的 `dataUpdateRocketMQTemplate` 。随后我们编写一个简单的方法，发送一个 update 的事务消息。

```java
    @Autowired
    @Qualifier("dataUpdateRocketMQTemplate")
    private RocketMQTemplate dataUpdateRocketMQTemplate;

    public void sendUpdateTransactionalMessage() {
        // 换一个Template发送
        String txId = UUID.randomUUID().toString();
        dataUpdateRocketMQTemplate.sendMessageInTransaction("test-transactionalupdate", 
                MessageBuilder.withPayload("update事务消息").setHeaderIfAbsent("txId", txId).build(), null);
    }
```

随后是对应的 `RocketMQLocalTransactionListener` ，注意这次标注 `@RocketMQTransactionListener` 注解时，还需要指定 `rocketMQTemplateBeanName` 发送消息的 `RocketMQTemplate` 的 Bean 名称（这样就完成了事务消息与本地事务调用的呼应关系）。至于方法内部的实现，大框架上跟上面的没有什么区别，只是最终调用的本地事务方法不同而已。

```java
@RocketMQTransactionListener(rocketMQTemplateBeanName = "dataUpdateRocketMQTemplate")
public class DataUpdateTransactionalStatusChecker implements RocketMQLocalTransactionListener {
    
    @Autowired
    private DataService dataService;
    
    @Override
    public RocketMQLocalTransactionState executeLocalTransaction(Message msg, Object arg) {
        String payload = new String((byte[]) msg.getPayload(), StandardCharsets.UTF_8);
        System.out.println(payload);
        System.out.println(arg);
    
        RocketMQLocalTransactionState state = RocketMQLocalTransactionState.COMMIT;
        try {
            // 此处调用的方法不同
            dataService.updateData(msg.getHeaders().get("txId", String.class));
        } catch (Exception e) {
            state = RocketMQLocalTransactionState.UNKNOWN;
        }
        return state;
    }
    
    @Override
    public RocketMQLocalTransactionState checkLocalTransaction(Message msg) {
        System.out.println(new String((byte[]) msg.getPayload(), StandardCharsets.UTF_8));
        return dataService.contains(msg.getHeaders().get("txId", String.class))
                ? RocketMQLocalTransactionState.COMMIT
                : RocketMQLocalTransactionState.ROLLBACK;
    }
}

```

对应的 `DataService` 中的 `updateData` 方法也很简单，当然不要忘记非常重要的那个 **txId** 保存动作。

```java
    @Transactional(rollbackFor = Exception.class)
    public void updateData(String txId) {
        System.out.println("执行数据库update操作。。。");
        jdbcTemplate.update("insert into tx_id (id) value (?)", txId);
    }
```

##### 4.4.4 测试效果

下面我们测试一下区分开分布式事务业务场景后，效果是否可以如我们预期那样。我们编写 Controller 方法，触发 `sendUpdateTransactionalMessage` 方法，重启 Producer 工程后调用，观察 Consumer 工程的控制台可以发现消息可以成功发送，这也就证明了区分事务场景也成功了。

## 5. RocketMQ的最佳实践

本章的最后，我们来讲一些实际项目开发和生产中可能遇到的情景，以及对应的最佳实践方案，毕竟我们接触与框架、中间件相关的技术后，最终还是要落地开发和生产中。

#### 5.1 消息的顺序消费

在一些具备业务顺序意义的场景中，我们发送的消息可能是有业务逻辑上的先后顺序的，这种情况就需要我们要求消息的消费者来按照顺序消费这些消息。

##### 5.1.1 顺序消费的场景

比方说这样一个场景：一个用户在不同的两个时间点下分别购买了一件商品（假设第一件商品 2 块，第二件商品 200 块），则他应该进行两次支付操作，对应生成两条订单信息。按照我们之前常提的“支付成功创建订单”场景，则两次支付的顺序应该对应到订单的创建顺序是一致的。但如果订单服务在消费这两条消息的时候顺序出现了问题，则会让用户十分疑惑：明明先支付的 2 块，为什么第一条订单信息是 200 块的那个呢？

由此我们就可以发现，这种需要顺序消费消息的场景，如果出现顺序差错，那么会引起用户的疑惑，甚至出现业务逻辑的问题。所以我们要想办法解决。

##### 5.1.2 解决思路

消费者在消费消息的时候，有两种顺序消费的场景：

- 全局消息有序：消息队列的所有消息都是带顺序的，消费者只要监听这个 RocketMQ ，就必须按照顺序进行消费
- 局部消息有序：在消息队列的某一个区间内，这部分的消息是有顺序的，需要消费者按顺序消费

我们先说全局有序，正常来讲 RocketMQ 的集群中可能有多个 Broker ，每个 Broker 中针对 topic 会创建 4 条队列，那多个 Broker 和多条队列并行处理，并发自然没什么问题，但并发就会牺牲顺序，所以如果要实现全局有序，那么一个 RocketMQ 集群中就只能在一个 Broker 中创建一条队列，这样才能做到严格的全局消息顺序消费（鱼和熊掌不可兼得）。

局部有序的话，我们就不需要考虑多个 Broker 的问题了，只需要针对每个 Broker 中的那个 topic 来做文章。由于每个 Broker 中针对 topic 会创建 4 条队列，那么消息的生产者在发送消息时，RocketMQ 会将这些消息按照合理的规则放入这 4 条队列中，而消息的消费者在消费这些消息的时候肯定不知道这些顺序的，所以就会造成消息的消费顺序混乱。对应的解决办法就相对简单了，我们不需要阉割 Broker 中 topic 的队列数量，只需要通过一些特殊机制，将同一个类别的消息全部放到一个队列中，这样也能保证消息有序，而相应的机制其实就是 hash 。

##### 5.1.3 普通场景的消息消费

首先我们构造一组消息，使用 for 循环一次性发送 10 条消息到 MQ 中：

```java
    public void send10Message() {
        for (int i = 0; i < 10; i++) {
            rocketMQTemplate.syncSend("test-ordered", "message" + i);
        }
    }
```

对应的消费者监听器没什么好说的，还是普通的消息监听即可。

```java
@Component
@RocketMQMessageListener(topic = "test-ordered", consumerGroup = "orderedconsumer")
public class OrderedMessageReceiver implements RocketMQListener<String> {
    
    @Override
    public void onMessage(String message) {
        System.out.println(message);
    }
}
```

如果按照此法来启动 Producer 和 Consumer ，在消息发送后 Consumer 监听这些消息的顺序是很乱的

```ini
message2
message6
message3
message1
message7
message5
message9
message0
message4
message8

```

由此就可以看出，默认情况下消息是没有严格顺序的，我们需要对此加以限制。

##### 5.1.4 改良：单个队列的消息消费

既然消息是均匀发给一个 topic 的所有队列，那我们可以使用 RocketMQ 给我们提供的 hash 机制，将消息全部归到一个队列中即可。使用的方式是使用 `RocketMQTemplate` 的 `xxxOrderly` 系列方法，这个方法需要我们在方法参数列表的最后传入一个 hash 的值（相当于 Map 中的 key ），`RocketMQTemplate` 在底层实际发送消息时会先计算传入的 hash key 应该对应 topic 中的哪个队列，计算完成后将消息放入算好的那个队列中。

```java
    public void sendOrderedMessage() {
        for (int i = 0; i < 10; i++) {
            rocketMQTemplate.syncSendOrderly("test-ordered", "ordered-message" + i, "ordered");
        }
    }

```

之后我们重启 Producer 方法，这次触发 `sendOrderedMessage` 方法，观察 Consumer 中的控制台打印，发现虽然顺序大体上没问题，但 6 和 7 的顺序还是反了！说明我们的消息顺序控制还是有问题！

##### 5.1.5 改良：并发消费改为顺序消费

那既然这样该如何改良呢？生产者端已经没招了，我们发消息本来就是顺序发送了，那就只好从消费者上考虑了。

默认情况下，RocketMQ 的消息监听器，在消费消息时是并发消费的，并不是顺序消费，所以才会出现上面的顺序错乱问题。如何调整呢？在 `@RocketMQMessageListener` 注解上有一个属性 `consumeMode` ，它可以指定是并发消费还是顺序消费。我们把这个属性显式指定为 `ORDERLY` 顺序消费。

```java
@Component
@RocketMQMessageListener(topic = "test-ordered", consumerGroup = "orderedconsumer", consumeMode = ConsumeMode.ORDERLY)
public class OrderedMessageReceiver implements RocketMQListener<String> {
```

之后重启 Consumer 工程，让 Producer 再次发送一组顺序消息，观察控制台可以发现这次消息的确是顺序消费了，大功告成。

#### 5.2 消息积压

项目生产过程中还有一种有可能遇到的现象：消息积压。顾名思义，太多消息堆积在 RocketMQ 的内部消费不掉，那就形成了消息积压现象。这种也不是什么好现象，我们要予以处理。

##### 5.2.1 积压出现的原因

针对上面所说的 3 个问题，下面简单聊聊对应的解决思路

1. 消费者的消费效率低 —— 加机器可能好使，也可能不好使
   - 如果是默认情况下，一个 topic 上有 4 条队列，那当消费者的应用实例数 < 4 时是好使的，如果超过了，那多出来的消费者没有监听的队列，那就是“徒增功耗”
2. 性能阻塞瓶颈 —— 可以考虑将这些很慢的操作改为异步操作
3. 本地缓存保存的消息太多 —— 其实这个问题的本质还是消费的慢，只是单纯为了解决这个问题的话，可以通过调整参数来适应，治标不治本

