---
title: 30整合RocketMq-RocketMq的设计与springboot整合.md
toc: true
tags: springboot
categories: 
    - [java]
    - [springboot]
---

上一章中，我们对消息中间件的基础和设计进行了一些了解和回顾。下面我们就进入到 RocketMQ 的具体学习了。本章我们先来安装使用 RocketMQ ，以及使用SpringBoot 整合体验点对点和发布订阅消息。

## 1. 安装RocketMQ

---

安装 RocketMQ 的方式有几种，可以用压缩包解压的方式安装，也可以直接用 Docker 安装。由于 RocketMQ 在 Windows 上还算跑的转，所以我们下面使用 Windows 版的 RocketMQ 来安装。

### 1.1 安装包下载和解压

<!--more-->

访问 RocketMQ 的官方下载页 [rocketmq.apache.org/download/](https://rocketmq.apache.org/download/) ，可以发现当前 RocketMQ 已经出到 5.0.0 了，不过我们为了保险起见，考虑尽可能稳定和市面使用率，所以本小册使用的 RocketMQ 版本为 **4.9.4** 。

下载 `rocketmq-all-4.9.4-bin-release.zip` 后，直接解压，这就相当于完成了安装。

### 1.2 快速扫一遍配置文件

解压后我们可以得到一个 `rocketmq-all-4.9.4-bin-release` 的文件夹，在这里面 `conf` 文件夹就是存放配置文件的位置。配置文件有点多，而且 RocketMQ 还给我们提供了多实例部署的配置文件。

![](./img/202303/30mq1.png)

当前我们先来测试单机版，所以我们后面在改动配置文件时，都是改动外头的这几个。

### 1.3 RocketMQ的组成结构

在开始运行 RocketMQ 之前，我们先思考一个实际的场景。

假设我们项目中有一个消息的生产者和消费者，它们连接到一个 RocketMQ 实例上，如下图所示。

![](./202303/30mq2.png)



随着业务规模的不断扩大，一个 RocketMQ 的实例已经有些不堪重负，于是我们需要将单机版的 RocketMQ 改为 RocketMQ 集群，此时我们不仅需要对 RocketMQ 扩容，还需要改变生产者和消费者的配置，让它们都连接到所有的 RocketMQ 实例上，如下图所示。



![](./202303/30mq3.png)

这样简单的扩容后会产生一个问题：如果 RocketMQ 的实例不断变动，那么消息的生产者和消费者会不断的修改配置，疲于应对 RocketMQ 集群的变动。

如何改善这种麻烦的现状呢？参照 SpringCloud 中服务注册与治理中心的思维，如果可以引入一个 RocketMQ 的注册中心，之后生产者和消费者都直接去连接注册中心，那是不是可以解决呢？

当然可以，RocketMQ 中就有一个对应的组件：**NameServer** 命名服务器，由这个 NameServer 来收集所有注册上线的 RocketMQ 实例（broker）。生产者和消费者只需要连接到 RocketMQ 的 NameServer 即可获得当前存活的 broker ，无需再因为 broker 扩容或者变动而改动配置。

![](./202303/30mq4.png)

如此了解下来，我们就能知道，RocketMQ 中包含一个 NameServer 和若干的 Broker ，由 NameServer 负责收集 Broker 的地址信息，Broker 负责实际的消息收发和存储工作。

### 1.4 启动运行RocketMQ

了解了 RocketMQ 的组成结构后，下面我们就可以来启动 RocketMQ 了。既然是 NameServer 负责收集 Broker 的信息，那么先启动 NameServer 后启动 Broker 会更合理。

##### 1.4.1 修改配置

在启动之前，我们需要先修改一点配置。默认情况下 RocketMQ 会吞掉我们机器的好多好多内存，为了避免 RocketMQ 吞掉过多的内存，所以我们需要修改两个启动文件 `runserver.cmd` 和 `runbroker.cmd` （Linux 系统则是修改 sh 文件），将其中的 java 启动参数中占用内存降低一些。

![](./img/202303/30mq5.png)

改为：`set "JAVA_OPT=%JAVA_OPT% -server -Xms256m -Xmx512m -Xmn256m -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=320m"`

![](./img/202303/30mq6.png)

改为： `JAVA_OPT="${JAVA_OPT} -server -Xms256m -Xmx512m -Xmn256m -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=320m"`

##### 1.4.2 配置环境变量

新版本的 RocketMQ 中需要我们向系统中添加几个环境变量。第一个要配置的是 **ROCKETMQ_HOME** ，这个名是固定的，需要我们指定我们安装解压的 RocketMQ 的目录。

另外最好再设置一个环境变量 **`NAMESRV_ADDR=127.0.0.1:9876`** ，设置这个环境变量后，后续的测试会省事一些（当然也可以不设置自，只是每次都要折腾一条语句：`set NAMESRV_ADDR=127.0.0.1:9876` ）。

##### 1.4.3 启动NameServer

下面就可以开始启动 RocketMQ 了。启动 NameServer ，只需要执行 `mqnamesrv.cmd` 文件即可。

启动成功后，控制台会打印启动成功的日志。

##### 1.4.4 启动Broker

启动 Broker 的时候，如果配置了环境变量，则直接执行 `mqbroker.cmd` 文件即可；如果没有设置环境变量，则需要用 cmd 命令执行如下命令：

```cmd
mqbroker.cmd ‐n 127.0.0.1:9876

```

执行该命令后，Broker 即可成功连接到 NameServer 。

### 1.5 测试是否正常运行

RocketMQ 的 NameServer 和 Broker 都启动完毕了，但是它们是否能够正常工作呢？我们可以借助 RocketMQ 中提供的一个测试工具来检验一下。

在 RocketMQ 解压的 lib 目录下有一个 `rocketmq-example-4.9.4.jar` ，这里面有一些测试工具。如果使用解压缩工具打开这个 jar 包，可以在 `org\apache\rocketmq\example\quickstart` 中找到这样两个类：

顾名思义，这两个类分别代表着消息的生产者和消费者。`Producer` 类会向 RocketMQ 中发送一些消息，而 `Consumer` 类会消费这些消息。通过执行这两个类，就可以判定 RocketMQ 是否在正常运行了。

##### 1.5.1 执行Producer

想要执行 `example` 包中的类，需要利用 `bin` 目录中的那个 `tools.cmd` 文件来辅助执行。来到 `bin` 目录下，执行如下命令：

```
tools.cmd org.apache.rocketmq.example.quickstart.Producer

```

（如果没有配置环境变量的话，还需要先执行 `set NAMESRV_ADDR=127.0.0.1:9876` 设置临时环境变量）

执行命令后，控制台会打印好多好多数据，这每一行打印的日志就是一条消息发送后的结果。

##### 1.5.2 执行Consumer

消息发出去了，下面我们用 `Consumer` 这个测试类触发一下消息的消费。

执行如下命令：

```
tools.cmd org.apache.rocketmq.example.quickstart.Consumer

```

执行完毕后，控制台打印的数据更多了，但是仔细观察能发现这些消息能跟前面 `Producer` 打印的发送的消息数据对起来，这也就说明了 `Consumer` 正确消费了 `Producer` 发送的消息，也就说明 RocketMQ 已经正常启动了。

### 1.6 安装监控端

RocketMQ 本身还有一套 UI 的管理工具，通过 UI 工具我们就可以更容易的管理维护 RocketMQ 。以前这个控制端是放置在 `rocketmq-externals` 仓库中的，叫 `rocketmq-console` ，后来这个模块单独分离出来，并改名为 `rocketmq-dashboard` ，仓库地址是 [github.com/apache/rock…](https://github.com/apache/rocketmq-dashboard)) ，对应的文档地址是 [github.com/apache/rock…](https://github.com/apache/rocketmq-dashboard/blob/master/docs/1_0_0/UserGuide_CN.md) 。

安装的方式有两种，可以使用 docker 的方式直接连接 NameServer 启动，也可以下载源码后，在本地运行。由于小册在编写时使用的是 Windows 环境，再使用 docker 有些许的不便，所以就下载源码咯。

源码下载下来后，我们可以直接解压到一个目录下，解压完成后需要找到 `src/main/resources` 目录下的 `application.properties` ，我们需要改两个配置，告诉 dashboard 我们的 RocketMQ 在哪里，以及启动时暴露的 http 端口：

```properties
server.port=39876
rocketmq.config.namesrvAddr=127.0.0.1:9876
```

修改完配置文件后，回到 `pom.xml` 所在的目录下，执行 `mvn spring-boot:run` 命令，即可启动 RocketMQ 的监控端。

从 dashboard 上可以看出，刚才一系列的操作一共触发了 1000 条消息，且对应的 topic 是 `TopicTest` 。

后面我们需要观察 dashboard 的时候还会再来看它，现阶段各位知道有这么个东西就 OK 。

## 2. SpringBoot整合RocketMQ

好了，RocketMQ 已经安装好了，下面我们就该用 SpringBoot 整合 RocketMQ 了。由于 RocketMQ 不是 SpringBoot 官方支持的，所以整合的 starter 也不是 SpringBoot 官方提供的，我们需要找 RocketMQ 整合 SpringBoot 的场景启动器。

#### 2.1 创建工程+导入依赖

整合 RocketMQ 的话，我们可以用一个工程同时作为消息的发送方和接收方，也可以用两个工程分开实现。本小册选择使用 2 个工程实现，所以我们需要分别创建一个 `spring-boot-integration-08-rocketmq-provider` 和 `spring-boot-integration-08-rocketmq-consumer` 工程，让他们都导入 RocketMQ 的场景启动器。

```xml
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.apache.rocketmq</groupId>
            <artifactId>rocketmq-spring-boot-starter</artifactId>
            <version>2.2.2</version>
        </dependency>
    </dependencies>
```

#### 2.2 配置文件编写

在配置文件中，我们需要配置两方面的内容，一个是每个工程占用的端口号，另一方面是连接 RocketMQ 的 NameServer 的相关配置。

```properties
# producer
server.port=12321

rocketmq.name-server=127.0.0.1:9876
rocketmq.producer.group=myproducer
```

```properties
# consumer
server.port=32123

rocketmq.name-server=127.0.0.1:9876
```

注意一个不同的地方：消息的生产者有单独定义一个 `rocketmq.producer.group` 生产者组，这个配置是必需的，如果没有这个配置的话，工程将无法正常启动。至于生产者组的作用是什么，在下一章讲解事务消息的时候再解释。

#### 2.3 编写生产者发送消息

下面我们先来编写生产者相关的代码。生产者要想发消息，需要注入一个 **`RocketMQTemplate`** ，这是 RocketMQ 整合 Spring 候提供的模板类，我们只需要操作这个类的 API 即可。

作为测试，我们可以先编写一个简单的消息发送代码。RocketMQ 发送消息一共有 3 种方式：同步、异步、单向发送，我们先测试最简单的同步消息发送。

```java
@Service
public class MessageSender {
    
    @Autowired
    private RocketMQTemplate rocketMQTemplate;
    
    public void sendMessage() {
        // 同步发送消息
        rocketMQTemplate.convertAndSend("test-sender", "test message");
    }
}
```

编写完毕后就可以启动工程了，启动完毕后访问 [127.0.0.1:12321/sendMessage]()，让工程向 RocketMQ 发送一条消息。如果浏览器可以收到响应 “success” 字符串，则说明方法调用成功。

来到 RocketMQ 的 dashboard ，切换到 “消息” 页签，并筛选 topic 为 `test-sender` ，可以发现我们刚才发的消息已经出现在 RocketMQ 中了！

![](./img/202303/30mq7.png)

点开消息详情，也可以看到我们发送的消息正文，说明消息已经正确发送！

![](./img/202303/30mq8.png)



#### 2.4 编写消费者消费消息

消息既然已经发出去了，下面我们就来编写消息的消费者，来把刚才的消息消费掉。

与生产者的编写方式不同，RocketMQ 要求我们编写消息的消费者（即消息监听器）时，需要实现一个 `RocketMQListener` 接口，接口中指定的泛型就是原始消息正文的类型。由于上面我们发送消息的时候传入的是字符串，所以这里的监听器泛型类型声明为 `String` 即可。

注意编写的类上还需要标注两个注解，`@Component` 负责将这个消息监听器注册到 IOC 容器中，而 `@RocketMQMessageListener` 用于指定当前监听的 topic 和被消费的消费者组。

```java
@Component
@RocketMQMessageListener(topic = "test-sender", consumerGroup = "myconsumer")
public class MessageReceiver implements RocketMQListener<String> {
    
    @Override
    public void onMessage(String message) {
        System.out.println("收到消息：" + message);
    }
}
```

对于接收到的消息，我们只是简单打印一下即可，不用过多折腾。

完事之后我们就启动消费者工程，由于此时的 RocketMQ 中已经有一条消息了，所以当我们启动工程完成时，控制台就已经打印了一次消息被消费的日志了。

```ini
收到消息：test message
```

对应的，在 RocketMQ dashboard 上再次查看这条消息，可以发现这条消息已经被成功消费掉了。

![](./img/202303/30mq9.png)

那工程正常运行期间，消费者是否能监听到消息呢？正好我们的 Producer 工程还在运行着，我们可以再发一次请求，让 Producer 再生产一条消息，看下图的测试效果，当我们发送请求后，Consumer 的控制台上立即就打印了消息，说明消息能够被消费者即时消费掉。

到此为止，我们就完成了 RocketMQ 与 SpringBoot 的整合。

## 3. RocketMQ发送接收消息

---

快速体会完 RocketMQ 的基本队列作用后，下面我们来了解一些 RocketMQ 的基础知识。

### 3.1 发送消息的方式

对于消息的生产者而言，RocketMQ 本身支持 3 种消息发送的方式，分别是同步发送、异步发送、单向发送。

##### 3.1.1 同步发送

发送同步消息，指的是当消息的发送方将消息发送给 RocketMQ 的整个过程中，发送方的线程会一直阻塞，直到 RocketMQ 响应发送结果为止。

上面我们在快速整合使用时，使用的 `convertAndSend` 方法就是同步发送消息的方式。除了该系列方法之外，还有 `syncSend` 系方法，效果基本一致。

##### 3.1.2 异步发送

相对比于同步发送，异步发送消息时，生产者把消息发给 RocketMQ 后不会阻塞等待，RocketMQ 收到消息后响应发送结果时，会在生产者中生成一个新的线程，并在这个新的线程中回调发送成功或失败。

下面我们可以编写一个异步发送消息的逻辑。`RocketMQTemplate` 中用来发送异步消息的方法统一是 `asyncSend` ，它有非常多的重载方法，我们选择一个相对简单的即可，相较于同步发送只是多了一个 `SendCallback` 参数（也就是发送完毕后 RocketMQ 的回调）：

```java
 public void asyncSend() {
        // 异步发送消息
        rocketMQTemplate.asyncSend("test-sender", "test async message", new SendCallback() {
            @Override
            public void onSuccess(SendResult sendResult) {
                System.out.println("异步消息发送成功");
                System.out.println(sendResult);
            }
    
            @Override
            public void onException(Throwable e) {
                System.out.println("消息发送失败，异常：" + e.getMessage());
                e.printStackTrace();
            }
        });
    }
```

下面可以测试一下，我们可以在 Controller 中编写一个新的接口来触发：

```java
    @GetMapping("/asyncSend")
    public String asyncSend() {
        messageSender.asyncSend();
        return "success";
    }
```

之后重启工程，访问 `/asyncSend` 接口，可以发现消息也能正常发送出去，而且控制台可以打印“异步消息发送成功”的内容。

之后重启工程，访问 `/asyncSend` 接口，可以发现消息也能正常发送出去，而且控制台可以打印“异步消息发送成功”的内容。

##### 3.1.3 单向发送

单向发送消息，指的是消息的生产者向 RocketMQ 发送消息后，不再阻塞等待 RocketMQ 的响应结果，发出去就算完事。这种发送方式由于不需要等待 RocketMQ 的反馈，所以它的效率是最高的；但同时由于不接收 RocketMQ 的反馈，所以消息是否真的发成功了也不知道。（简单来讲一句话：别管好不好使，你就说快不快吧）

单向发送的方法为 sendOneWay ，正好我们来证明一下单向发送的速度是最快的，我们分别给同步发送和单向发送的前后打印时间戳，来观察消息发送的效率。

```java
    public void sendMessage() {
        // 同步发送消息
        System.out.println(System.currentTimeMillis());
        rocketMQTemplate.convertAndSend("test-sender", "test message");
        System.out.println("发送同步消息成功");
        System.out.println(System.currentTimeMillis());
    }
    
    public void sendOneway() {
        System.out.println(System.currentTimeMillis());
        // 发送单向消息
        rocketMQTemplate.sendOneWay("test-sender", "test oneway message");
        System.out.println("发送单向消息成功");
        System.out.println(System.currentTimeMillis());
    }
```

之后在 `MessageController` 的 `sendMessage` 方法中加入 `sendOneway` 方法的调用，重启工程，浏览器重新访问 `/sendMessage` 请求，观察控制台打印的时间戳：

```ini
1678535071319
发送同步消息成功
1678535071326

1678535071326
发送单向消息成功
1678535071327
```

很明显，同步消息发送花了大概 0.2 秒，而单向消息发送只用了 1 毫秒，说明单向消息的确是发送最快的。

### 3.2 消息发送和接收机制

了解了消息的发送方式后，下面我们简单聊一聊消息从生产者发送到 RocketMQ ，以及消费者从 RocketMQ 中消费消息的机制。

##### 3.2.1 发送机制

- 当消息的生产者要把消息发送到 RocketMQ 中，本质是发送到 Broker 中，由于我们的生产者连接的都是 NameServer 而非 Broker ，所以第一步其实是消息的生产者从 NameServer 中获取可以发送的 Broker 。
  - 注意这个环节中，消息的生产者完全可以从 NameServer 中获取到 Broker 的信息，因为 Broker 在启动的时候会把自身包含的所有队列都上报给 NameServer 。生产者从 NameServer 中拿到的 broker 信息主要有三方面内容：broker 的地址、broker 的名称、内部队列的 id 。

- 每个 Broker 中又包含不同的队列容器，消息的生产者拿到所有的 Broker 后，从中选择一个合适的队列，将消息放入这个队列中，从而完成消息的发送。
  - 如果消息发送失败，则下一次再发送消息时，生产者会主动避开这些失败的 broker 。

- 在消息放入队列之前，生产者会先对消息进行校验（如检查消息正文不能为空，消息不能太长等等），检查完毕后会检查选择的 broker 中是否包含消息对应的 topic ，如果没有这个 topic 则会自动创建 topic ，并且默认创建 4 个写队列和 4 个读队列。
  - 一次性创建 4 个队列的目的是考虑到高可用和高性能。

##### 3.2.2 消费机制

4. 消息发送到 RocketMQ 后，下一步就该由消费者来消费这条消息了。消费者在启动时，也会连接到 NameServer ，匹配自己需要监听的 Broker 中的队列。
5. 消费者与 Broker 建立监听关系有如下规则：
   - 一个消费者组中包含多个消费者（即一个消费者组是一个集群）
   - 一个消费者组可以消费多个 topic 中的多个队列（即监听多个 topic ）
   - 一个 Broker 中的一个 topic 中的一个队列只能被一个消费者监听（注意层级关系）

6. 消费者在消费消息时有两种模式：
   - 集群模式（push模式）：topic 下的消息只能被一个消费者消费
   - 广播模式（pull 模式）：topic 下的消息会被监听的所有消费者消费

最后我们用一张图来概述上面的描述，小伙伴们可以参照这张图来辅助理解上面的描述。

![](./img/202303/30mq10.png)

### 3.3 消息的tags

本章的最后我们再来提一个 RocketMQ 中消息的属性：**tags** 。

##### 3.3.1 tags的业务意义

为什么要有这个东西呢？我们可以思考一个现实开发的场景：

我们 RocketMQ 在一个项目中通常不可能只用于一个业务吧，肯定是好多个业务都会用到。如果只是用 topic 区分消息的话，那会产生一种问题：但凡是一个新业务场景，都会开辟一块全新的 topic ，如果这块业务又有很多的二级分类，那要么所有的消息全部一股脑接收，由消费方统一处理，要么为每一个二级分类都划分一个 topic 。

这个法可行吗？可行！优雅吗？貌似不是那么优雅。

那怎么更优雅呢？哎，这就可以用 topic + tags 的方式来解决了。比方说每个大的业务场景用不同的 topic 划分（比方说一个 ERP 系统的销售、采购类业务用不同的 topic ），而每个业务场景的分类则可以用不同的 tags 区分（如采购软件、硬件、耗材等）。

##### 3.3.2 使用tags

原生 RocketMQ 使用 tags 还稍微麻烦点，但是在 SpringBoot 整合 RocketMQ 后会变得特别简单，只需要在发送 / 接收消息的 topic 后添加 `":tags"` 即可。

举个例子吧，我们用一个 topic 后面挂不同的 tag ，连续发两条消息；消费者中声明只过滤 tag 为 `software` 的消息：

```java
    public void sendTagsMessage() {
        // 发送tags为"software"和"hardware"的消息
        rocketMQTemplate.convertAndSend("test-tags:software", "test software message");
        rocketMQTemplate.convertAndSend("test-tags:hardware", "test hardware message");
    }
```

消费者

```java
@Component
@RocketMQMessageListener(topic = "test-tags", consumerGroup = "tagsconsumer", selectorExpression = "software")
public class SoftwareMessageReceiver implements RocketMQListener<String> {
    
    @Override
    public void onMessage(String message) {
        System.out.println("收到software消息：" + message);
    }
}
```

以此法编写后，我们重启生产者和消费者。触发生产者的 `sendTagsMessage` 方法后，可以发现在 RocketMQ dashboard 中消息已经成功附加了 tag ，而且消费者的控制台中只打印了 software 的消息，hardware 的消息没有打印。

以此法编写后，我们重启生产者和消费者。触发生产者的 `sendTagsMessage` 方法后，可以发现在 RocketMQ dashboard 中消息已经成功附加了 tag ，而且消费者的控制台中只打印了 software 的消息，hardware 的消息没有打印。

用这种方式我们就可以实现同一套 topic 的不同 tag 的分发接收。实际在项目开发中，这种设计配合模板方法模式，可以很优雅的实现共有逻辑的抽取和个性逻辑的扩展。

【好了，本章就先接触这么多，通过使用 SpringBoot 整合 RocketMQ 后，各位是否感觉整合起来还是一如既往的容易呢？下一章我们再来接触一些 RocketMQ 的实用特性】