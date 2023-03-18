---
title: 32整合RocketMq-自动装配与核心组件
toc: true
tags: springboot
categories: 
    - [java]
    - [springboot]
---

RocketMQ 的最后一个章节，我们还是来深入底层，研究一下 SpringBoot 整合 RocketMQ 的时候，底层都给我们装配了什么东西。

## 1. 自动装配

在 `rocketmq-spring-boot` 的依赖中，我们可以找到一个 `spring.factories` 文件，这里面对于自动装配的配置只有一个：

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.apache.rocketmq.spring.autoconfigure.RocketMQAutoConfiguration
```


<!--more-->

合着 RocketMQ 整合到 SpringBoot 只用一个自动配置类就解决了？那它是简单还是复杂呢？我们赶紧深入看看。

![](./img/202303/32mq1.png)

### 1.1 RocketMQAutoConfiguration

进入到自动配置类的内部，很明显我们想多了，一个自动配置类肯定不可能把所有的活都干完，怎么说也得拆分成多个吧，看 `RocketMQAutoConfiguration` 的类上标注的 `@Import` 注解，它还导入了 5 个配置类，这些配置类我们下面会一个一个看。

```java

@Configuration
@EnableConfigurationProperties(RocketMQProperties.class)
@ConditionalOnClass({MQAdmin.class})
@ConditionalOnProperty(prefix = "rocketmq", value = "name-server", matchIfMissing = true)
@Import({MessageConverterConfiguration.class, ListenerContainerConfiguration.class, 
         ExtProducerResetConfiguration.class, ExtConsumerResetConfiguration.class, 
         RocketMQTransactionConfiguration.class})
@AutoConfigureAfter({MessageConverterConfiguration.class})
@AutoConfigureBefore({RocketMQTransactionConfiguration.class})

public class RocketMQAutoConfiguration implements ApplicationContextAware {
```

首先注意一点，这个自动配置类的生效条件看上去 `@ConditionalOnProperty(prefix = "rocketmq", value = "name-server", matchIfMissing = true)` 是多余的，但结合下面的一段初始化代码中可以发现，它还是需要我们配置的，否则会打印 RocketMQ 相关 bean 全部跳过的日志，并且真的不会初始化任何组件。

```java
@PostConstruct
public void checkProperties() {
  String nameServer = environment.getProperty("rocketmq.name-server", String.class);
  log.debug("rocketmq.nameServer = {}", nameServer);
  if (nameServer == null) {
    log.warn("The necessary spring property 'rocketmq.name-server' is not defined, 
             all rockertmq beans creation are skipped!");
  }
}
```

那 `RocketMQAutoConfiguration` 中都注册了哪些组件呢？下面一一来看。

#### 1.1.1 DefaultMQProducer

在原生的 RocketMQ 中，我们操纵 `RocketMQTemplate` 生产消费，发送到 RocketMQ 的原始 API 是 `DefaultMQProducer` 这个家伙，虽然我们使用 SpringBoot 整合了 RocketMQ ，但底层还是得依赖这些 API ，所以自动配置类帮我们做的第一件事就是初始化这个基础的 API 。

```java
@Bean(PRODUCER_BEAN_NAME)
@ConditionalOnMissingBean(DefaultMQProducer.class)
@ConditionalOnProperty(prefix = "rocketmq", value = {"name-server", "producer.group"})
public DefaultMQProducer defaultMQProducer(RocketMQProperties rocketMQProperties) {
  RocketMQProperties.Producer producerConfig = rocketMQProperties.getProducer();
  String nameServer = rocketMQProperties.getNameServer();
  String groupName = producerConfig.getGroup();
  Assert.hasText(nameServer, "[rocketmq.name-server] must not be null");
  Assert.hasText(groupName, "[rocketmq.producer.group] must not be null");

  String accessChannel = rocketMQProperties.getAccessChannel();

  String ak = rocketMQProperties.getProducer().getAccessKey();
  String sk = rocketMQProperties.getProducer().getSecretKey();
  boolean isEnableMsgTrace = rocketMQProperties.getProducer().isEnableMsgTrace();
  String customizedTraceTopic = rocketMQProperties.getProducer().getCustomizedTraceTopic();

  DefaultMQProducer producer = RocketMQUtil.createDefaultMQProducer(groupName, ak, sk, isEnableMsgTrace, customizedTraceTopic);
  producer.setNamesrvAddr(nameServer);
  if (!StringUtils.isEmpty(accessChannel)) {
    producer.setAccessChannel(AccessChannel.valueOf(accessChannel));
  }
  // 一堆set ......
  return producer;
}
```

纵观整段构造 `DefaultMQProducer` 的逻辑，大多的代码都是在从 SpringBoot 全局配置文件中获取一些属性配置值，并且应用到 `DefaultMQProducer` 中，最终返回完成构造。这里面本身没有特别需要注意的，各位仅需要知道 `rocketmq.name-server` 和 `rocketmq.producer.group` 要求必须配置，是在这里完成的判断即可。

#### 1.1.2 DefaultLitePullConsumer

对于 RocketMQ 的消费者来讲，消费消息的方式有 2 种：被动等待 Broker 给推送（ push ），以及主动从 Broker 中拉取消息（ pull ）。被动等待的方式，就是之前两章中我们编写的那些 `RocketMQListener` ；而主动从 Broker 中拉取消息的 API ，它使用的是 `DefaultLitePullConsumer` ，这个 API 是从 4.6.0 之后才有的，它的使用方式很简单，整合到 `RocketMQTemplate` 中也有对应的方法，下面我们马上能看到。

而对于源码中的构造，它的方式与上面 Producer 的方式非常类似，都是从 properties 中获取配置属性，并相应的构造 `DefaultLitePullConsumer` 对象并返回。由于我们在之前的工程代码中都没有配置过主动拉取的 `rocketmq.consumer.group` 和 `rocketmq.consumer.topic` ，所以这个组件压根就没初始化。

```java
@Bean(CONSUMER_BEAN_NAME)
@ConditionalOnMissingBean(DefaultLitePullConsumer.class)
@ConditionalOnProperty(prefix = "rocketmq", value = {"name-server", "consumer.group", "consumer.topic"})
public DefaultLitePullConsumer defaultLitePullConsumer(RocketMQProperties rocketMQProperties)
  throws MQClientException {
  RocketMQProperties.Consumer consumerConfig = rocketMQProperties.getConsumer();
  String nameServer = rocketMQProperties.getNameServer();
  String groupName = consumerConfig.getGroup();
  String topicName = consumerConfig.getTopic();
  Assert.hasText(nameServer, "[rocketmq.name-server] must not be null");
  Assert.hasText(groupName, "[rocketmq.consumer.group] must not be null");
  Assert.hasText(topicName, "[rocketmq.consumer.topic] must not be null");

  // 一大堆处理 ......

  DefaultLitePullConsumer litePullConsumer = RocketMQUtil.
    createDefaultLitePullConsumer(nameServer, accessChannel,
        groupName, topicName, messageModel, selectorType, selectorExpression, ak, sk, pullBatchSize, useTLS);
  litePullConsumer.setEnableMsgTrace(consumerConfig.isEnableMsgTrace());
  litePullConsumer.setCustomizedTraceTopic(consumerConfig.getCustomizedTraceTopic());
  litePullConsumer.setNamespace(consumerConfig.getNamespace());
  return litePullConsumer;
}
```

#### 1.1.3 RocketMQTemplate

`RocketMQTemplate` 我们可太熟悉了，它就是我们前面两章一直使用的核心 API 模板类，请注意一点，`RocketMQTemplate` 不仅仅整合了 `DefaultMQProducer` ，也整合了 `DefaultLitePullConsumer` ，这就意味着 `RocketMQTemplate` 可以完成消息主动拉取的工作（拉取的方法是 `receive` ，该方法会传入消息封装的类型，返回的就是消息封装为模型类型对象后的集合）。

```java
@Bean(destroyMethod = "destroy")
@Conditional(ProducerOrConsumerPropertyCondition.class)
@ConditionalOnMissingBean(name = ROCKETMQ_TEMPLATE_DEFAULT_GLOBAL_NAME)
public RocketMQTemplate rocketMQTemplate(RocketMQMessageConverter rocketMQMessageConverter) {
  RocketMQTemplate rocketMQTemplate = new RocketMQTemplate();
  if (applicationContext.containsBean(PRODUCER_BEAN_NAME)) {
    rocketMQTemplate.setProducer((DefaultMQProducer) applicationContext.getBean(PRODUCER_BEAN_NAME));
  }
  if (applicationContext.containsBean(CONSUMER_BEAN_NAME)) {
    rocketMQTemplate.setConsumer((DefaultLitePullConsumer) applicationContext.getBean(CONSUMER_BEAN_NAME));
  }
  rocketMQTemplate.setMessageConverter(rocketMQMessageConverter.getMessageConverter());
  return rocketMQTemplate;
}
```

除了上面 3 个组件之外，剩下的代码就不很重要了，下面我们来逐个阅读 `@Import` 注解导入的 5 个注解配置类。

### 1.2 MessageConverterConfiguration

```java
@Configuration
@ConditionalOnMissingBean(RocketMQMessageConverter.class)
class MessageConverterConfiguration {

    @Bean
    public RocketMQMessageConverter createRocketMQMessageConverter() {
        return new RocketMQMessageConverter();
    }
}

```

`MessageConverterConfiguration` 注解只注册了一个 Bean ，就是 RocketMQ 的消息转换器 `RocketMQMessageConverter` ，用这个转换器可以完成 RocketMQ 消息正文到模型类的转换动作。

有关这个转换器，我们也可以简单看一眼。作为一个消息转换器，它应当具备的能力就是用各种方式去转换消息。由于 RocketMQ 的设计中认为使用 `byte[]` 或者字符串明文（包括 json ）的方式最合适，于是在 `RocketMQMessageConverter` 的构造方法中初始化了字节数组的转换器、字符串的转换器，并且在当前工程中引入 jackson 和 fastjson 的时候，初始化对应的 json 转换器。

```java
    private final CompositeMessageConverter messageConverter;

    public RocketMQMessageConverter() {
        List<MessageConverter> messageConverters = new ArrayList<>();
        ByteArrayMessageConverter byteArrayMessageConverter = new ByteArrayMessageConverter();
        byteArrayMessageConverter.setContentTypeResolver(null);
        messageConverters.add(byteArrayMessageConverter);
        messageConverters.add(new StringMessageConverter());
        if (JACKSON_PRESENT) {
            MappingJackson2MessageConverter converter = new MappingJackson2MessageConverter();
            ObjectMapper mapper = converter.getObjectMapper();
            mapper.disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
            mapper.registerModule(new JavaTimeModule());
            converter.setObjectMapper(mapper);
            messageConverters.add(converter);
        }
        if (FASTJSON_PRESENT) {
            try {
                messageConverters.add(
                    (MessageConverter)ClassUtils.forName(
                        "com.alibaba.fastjson.support.spring.messaging.MappingFastJsonMessageConverter",
                        ClassUtils.getDefaultClassLoader()).newInstance());
            } // catch ignore ......
        }
        messageConverter = new CompositeMessageConverter(messageConverters);
    }
```

请注意一个很重要的设计，最终 `RocketMQMessageConverter` 依赖的消息转换器是一个叫 `CompositeMessageConverter` 的组合对象，这其实是一个来源于 SpringFramework 的非常经典的设计，也就是 **GoF23 中的组合模式** ！在 SpringFramework 的源码中就有很多这种设计，通过一个 **Composite** 开头的对象，内部组合多个接口的实现类对象，就可以完成一个组合对象的统一操作。这种“组合-整体”的设计特征非常明显，只要看到 **Composite** 开头的类，基本上内部都是有维护对应接口 / 类的集合的。

### 1.3 ListenerContainerConfiguration

```java
@Configuration
public class ListenerContainerConfiguration implements ApplicationContextAware, 
SmartInitializingSingleton {
```

先不着急看类的内部定义，从这个配置类实现的接口上可以发现一个很重要的信息：`SmartInitializingSingleton` ，实现了这个接口的对象，在 IOC 容器统一初始化单实例 bean 对象完毕后，`BeanFactory` 会统一回调它们的 `afterSingletonsInstantiated` 方法！为什么这个配置类会实现这个接口呢？`afterSingletonsInstantiated` 方法一定能告诉我们答案，所以我们下面直接跳转到该方法查看。

> 小伙伴们，通过这个类的解读，阿熊想告诉大家的是：阅读源码不一定都是从上往下顺序阅读，在遇到一些关键的 API 或者设计时，往往这些因素才是攻破源码的核心切入点！

#### 1.3.1 afterSingletonsInstantiated

```java
@Override
public void afterSingletonsInstantiated() {
  Map<String, Object> beans = this.applicationContext.getBeansWithAnnotation(RocketMQMessageListener.class)
    .entrySet().stream().filter(entry -> !ScopedProxyUtils.isScopedTarget(entry.getKey()))
    .collect(Collectors.toMap(Map.Entry::getKey, Map.Entry::getValue));

  beans.forEach(this::registerContainer);
}
```

从方法的实现中，我们能非常清晰地看出来：原来我们通过 `@RocketMQMessageListener` 注解标注的消息监听器，最终注册为真正的 RocketMQ 消息监听器的时机就是这里！它会从 IOC 容器中取出所有标注了 `@RocketMQMessageListener` 注解的 bean 对象，并逐个执行下面的 `registerContainer` 方法。我们继续向下深入。

```java
private void registerContainer(String beanName, Object bean) {
    Class<?> clazz = AopProxyUtils.ultimateTargetClass(bean);
    // 检查被标注的类是否实现了RocketMQListener或RocketMQReplyListener ......

    RocketMQMessageListener annotation = clazz.getAnnotation(RocketMQMessageListener.class);

    String consumerGroup = this.environment.resolvePlaceholders(annotation.consumerGroup());
    String topic = this.environment.resolvePlaceholders(annotation.topic());

    // 校验逻辑 ......

    String containerBeanName = String.format("%s_%s", DefaultRocketMQListenerContainer.class.getName(),
        counter.incrementAndGet());
    GenericApplicationContext genericApplicationContext = (GenericApplicationContext) applicationContext;

    genericApplicationContext.registerBean(containerBeanName, DefaultRocketMQListenerContainer.class,
        () -> createRocketMQListenerContainer(containerBeanName, bean, annotation));
    DefaultRocketMQListenerContainer container = genericApplicationContext.getBean(containerBeanName,
        DefaultRocketMQListenerContainer.class);
    if (!container.isRunning()) {
        try {
            container.start();
        } // catch throw ex ......
    }
    
    // logger ......
}
```

既然是注册 Container ，那得有个 Container 吧，我们一开始标注的是 `RocketMQListener` 接口的实现类呀。别急，上面一大堆校验和检查后，在源码的下半部分有一个重要的动作：它将 `ApplicationContext` 强转为 `GenericApplicationContext` （完全可以这么干，因为 SpringBoot 的 IOC 容器全都是注解驱动的，而注解驱动的父类就是 `GenericApplicationContext` ，忘记的小伙伴可以回看 [Spring 小册的第 15 章](https://juejin.cn/book/6857911863016390663/section/6859985463235575808) ），之后调用其 `registerBean` 方法向 IOC 容器注册一个 `DefaultRocketMQListenerContainer` 类型的 Bean ，紧接着又取出，回调它的 `start` 方法，完毕。

源码是看完了，可各位是否感觉有点头脑发懵呢？这么一个操作到底是要干嘛呢？其实周转了一套，核心就是拿着我们注册的那个 `RocketMQListener` ，包装为一个 `DefaultRocketMQListenerContainer` 对象，并启动它。为什么要封装为 `DefaultRocketMQListenerContainer` 才行呢？我们继续往里深入。

#### 1.3.2 创建DefaultRocketMQListenerContainer

```java
    private DefaultRocketMQListenerContainer createRocketMQListenerContainer(String name, Object bean,
        RocketMQMessageListener annotation) {
        DefaultRocketMQListenerContainer container = new DefaultRocketMQListenerContainer();
        container.setRocketMQMessageListener(annotation);
        // 一大堆setter ......
        //注意此处：DefaultRocketMQListenerContainer中组合了一个RocketMQListener
        if (RocketMQListener.class.isAssignableFrom(bean.getClass())) {
            container.setRocketMQListener((RocketMQListener) bean);
        } else if (RocketMQReplyListener.class.isAssignableFrom(bean.getClass())) {
            container.setRocketMQReplyListener((RocketMQReplyListener) bean);
        }
        container.setMessageConverter(rocketMQMessageConverter.getMessageConverter());
        container.setName(name);
        return container;
    }
```

请注意观察上面源码中配的一行注释！我们当前拿到的这个 `RocketMQListener` 类型的对象，最终会封装到 `DefaultRocketMQListenerContainer` 里面，那就意味着 `RocketMQListener` 若想起作用，那就必须服从 `DefaultRocketMQListenerContainer` 的调度才行。

那下面我们的问题还没有结束，为什么集成到 `DefaultRocketMQListenerContainer` 中，就可以让 `RocketMQListener` 收到来自 Broker 的消息呢？这个时候就需要翻看 `DefaultRocketMQListenerContainer` 的结构组成了。

#### 1.3.3 DefaultRocketMQListenerContainer的内部结构

```java
public class DefaultRocketMQListenerContainer implements InitializingBean,
        RocketMQListenerContainer, SmartLifecycle, ApplicationContextAware {
    // ......

    private RocketMQListener rocketMQListener;

    private DefaultMQPushConsumer consumer;

    // ......
```

请注意观察 `DefaultRocketMQListenerContainer` 中的两个成员！上面就是集成进来的我们自己编写的 `RocketMQListener` ，而下面这个 `consumer` 就是 **RocketMQ 原生的用来消费消息的 API** ！它的地位与 `DefaultMQProducer` 是一样的！

所以到此为止，各位是否能意识到一点：`RocketMQListener` 能够收到消息，其实是底层有 `DefaultRocketMQListenerContainer` 负责调度，利用 RocketMQ 原生的 `DefaultMQPushConsumer` 接收 Broker 的消息，并通过一些手段后将消息交给 `RocketMQListener` ，这样完成了消息的监听！至于源码中是如何调度的，下面的第 3 节我们再展开研究。

### 1.4 ExtProducerResetConfiguration

与 `ListenerContainerConfiguration` 类似，`ExtProducerResetConfiguration` 玩的套路也是借助 `SmartInitializingSingleton` 接口实现。

```java
@Configuration
public class ExtProducerResetConfiguration implements ApplicationContextAware, SmartInitializingSingleton {
    // ......
    
    @Override
    public void afterSingletonsInstantiated() {
        Map<String, Object> beans = 
          this.applicationContext.getBeansWithAnnotation(ExtRocketMQTemplateConfiguration.class)
            .entrySet().stream().filter(entry -> !ScopedProxyUtils.isScopedTarget(entry.getKey()))
            .collect(Collectors.toMap(Map.Entry::getKey, Map.Entry::getValue));

        beans.forEach(this::registerTemplate);
    }
```

不过这里它负责的是扩展 `RocketMQTemplate` 了，也就是我们在上一章中讲解的多场景分布式事务消息中，通过扩展多个不同的 `RocketMQTemplate` 来划分业务场景。因为本质上都是 `RocketMQTemplate` ，所以最终都是依赖 `DefaultMQProducer` 来实现，`afterSingletonsInstantiated` 方法的下面会 foreach 循环注册这些 `RocketMQTemplate` ，顺便也创建对应的 `DefaultMQProducer` 。

### 1.5 ExtConsumerResetConfiguration

只看配置类的名称也能猜到，`ExtConsumerResetConfiguration` 跟 `ExtProducerResetConfiguration` 应该是配套出现的，而且它们的套路都是一样的。翻看源码发现果然如此，所以这部分做的事情是给扩展的 `RocketMQTemplate` 中设置 Consumer 部分。

```java
@Configuration
public class ExtConsumerResetConfiguration implements ApplicationContextAware, SmartInitializingSingleton {
    // ......
    
    @Override
    public void afterSingletonsInstantiated() {
        Map<String, Object> beans = this.applicationContext
                .getBeansWithAnnotation(ExtRocketMQConsumerConfiguration.class)
                .entrySet().stream().filter(entry -> !ScopedProxyUtils.isScopedTarget(entry.getKey()))
                .collect(Collectors.toMap(Map.Entry::getKey, Map.Entry::getValue));

        beans.forEach(this::registerTemplate);
    }
```

### 1.6 RocketMQTransactionConfiguration

`RocketMQTransactionConfiguration` 事务消息的配置类，它的套路依然是借助 `SmartInitializingSingleton` 接口来实现，不过这里它扫描的注解是 `@RocketMQTransactionListener` 了，而且从下面的 `registerTransactionListener` 方法上也能看到，它的逻辑是依照 `@RocketMQTransactionListener` 注解上标注的 `rocketMQTemplateBeanName` ，从 IOC 容器中找出对应的bean 对象，并给内部的 Producer 设置线程池、事务监听器，这样就完成了事务消息的相关装配。

有一个值得注意的，上一章中我们提到了一个 `RocketMQTemplate` 只能完成一个分布式事务消息场景的处理，对应的限制就在这里，当取到 `rocketMQTemplateBeanName` 指定的 `RocketMQTemplate` 后，会检查内部的 Producer 是否已经指定了事务监听器，如果已经指定了（即在检查之前已经有一个 `RocketMQLocalTransactionListener` 设置进去了），则会抛出对象状态不合法的异常 `IllegalStateException` 。

```java
@Configuration
public class RocketMQTransactionConfiguration implements ApplicationContextAware, SmartInitializingSingleton {
    // ......
    
    @Override
    public void afterSingletonsInstantiated() {
        Map<String, Object> beans = this.applicationContext.getBeansWithAnnotation(RocketMQTransactionListener.class)
            .entrySet().stream().filter(entry -> !ScopedProxyUtils.isScopedTarget(entry.getKey()))
            .collect(Collectors.toMap(Map.Entry::getKey, Map.Entry::getValue));

        beans.forEach(this::registerTransactionListener);
    }

    private void registerTransactionListener(String beanName, Object bean) {
        Class<?> clazz = AopProxyUtils.ultimateTargetClass(bean);

        if (!RocketMQLocalTransactionListener.class.isAssignableFrom(bean.getClass())) {
            throw new IllegalStateException(clazz + " is not instance of " + RocketMQLocalTransactionListener.class.getName());
        }
        RocketMQTransactionListener annotation = clazz.getAnnotation(RocketMQTransactionListener.class);
        RocketMQTemplate rocketMQTemplate = (RocketMQTemplate) applicationContext.getBean(annotation.rocketMQTemplateBeanName());
        if (((TransactionMQProducer) rocketMQTemplate.getProducer()).getTransactionListener() != null) {
            throw new IllegalStateException(annotation.rocketMQTemplateBeanName() + " already exists RocketMQLocalTransactionListener");
        }
        ((TransactionMQProducer) rocketMQTemplate.getProducer()).setExecutorService(new ThreadPoolExecutor(annotation.corePoolSize(), annotation.maximumPoolSize(),
            annotation.keepAliveTime(), annotation.keepAliveTimeUnit(), new LinkedBlockingDeque<>(annotation.blockingQueueSize())));
        ((TransactionMQProducer) rocketMQTemplate.getProducer()).setTransactionListener(RocketMQUtil.convert((RocketMQLocalTransactionListener) bean));
        log.debug("RocketMQLocalTransactionListener {} register to {} success", clazz.getName(), annotation.rocketMQTemplateBeanName());
    }
```

以上就是与 RocketMQ 相关的自动装配，总的看下来，核心的组件还是 `RocketMQTemplate` ，以及其内部依赖的 Producer 、Consumer 、TransactionListener 等。

## 2. 消息发送的工作原理

---

下面我们来分别研究一下消息发送和消息监听的工作原理。

### 2.1 消息发送的基本流程

RocketMQ 的消息生产者，在向 RocketMQ 发送一条消息的时候，会经历一下几个步骤：

1. 消息校验
2. 路由查找
3. 选择队列
4. 发送消息

而 RocketMQ 在与 SpringBoot 整合后，它的发送过程会在上述步骤之前多加一步：`RocketMQTemplate` 的处理。下面我们逐个步骤来看。

### 2.2 RocketMQTemplate发送消息

以同步发送消息为例，我们使用同步的方式发送消息时，通常操纵的是 `RocketMQTemplate` 的 `convertAndSend` 或者 `syncSend` 方法：

```java
public SendResult syncSend(String destination, Object payload) {
  return syncSend(destination, payload, producer.getSendMsgTimeout());
}

public SendResult syncSend(String destination, Object payload, long timeout) {
  Message<?> message = MessageBuilder.withPayload(payload).build();
  return syncSend(destination, message, timeout);
}
```

无论是哪个方法，只要我们传入的 `payload` 参数是一个 `Object` （即任意类型），就会执行一个动作：`payload` 转 `Message` 。

#### 2.2.1 payload转Message

消息的创建，是通过 `MessageBuilder` 建造器来构建的，这个方法中一个比较关键的细节是最后一行，正常情况下我们创建的 `Message` 对象，其最终实现是一个 `GenericMessage` 。

```java
	public static <T> MessageBuilder<T> withPayload(T payload) {
		return new MessageBuilder<>(payload, new MessageHeaderAccessor());
	}

public Message<T> build() {
  if (this.providedMessage != null && !this.headerAccessor.isModified()) {
    return this.providedMessage;
  }
  MessageHeaders headersToUse = this.headerAccessor.toMessageHeaders();
  if (this.payload instanceof Throwable) {
    // 消息体是异常相关的处理 ......
  } else {
    return new GenericMessage<>(this.payload, headersToUse);
  }
}
```

而这个 `GenericMessage` 的实现又非常简单，它就是一个普通的模型包装类，没有一丁点复杂成分在内。

```java
public class GenericMessage<T> implements Message<T>, Serializable {
	private final T payload;
	private final MessageHeaders headers;
    
    // ......
    
	public GenericMessage(T payload, MessageHeaders headers) {
		Assert.notNull(payload, "Payload must not be null");
		Assert.notNull(headers, "MessageHeaders must not be null");
		this.payload = payload;
		this.headers = headers;
	}
```

#### 2.2.2 syncSend

包装完 `Message` 对象后，下面一步是执行 `syncSend` 方法，这又是一个重载方法，最终调用的方法是下面源码的 4 参数方法：

```java
 public SendResult syncSend(String destination, Message<?> message, long timeout) {
        return syncSend(destination, message, timeout, 0);
    }

    public SendResult syncSend(String destination, Message<?> message, long timeout, int delayLevel) {
        // 前置检查 ......
        try {
            long now = System.currentTimeMillis();
            org.apache.rocketmq.common.message.Message rocketMsg = this.createRocketMqMessage(destination, message);
            if (delayLevel > 0) {
                rocketMsg.setDelayTimeLevel(delayLevel);
            }
            SendResult sendResult = producer.send(rocketMsg, timeout);
            long costTime = System.currentTimeMillis() - now;
            // logger ......
            return sendResult;
        } // catch throw ex ......
    }
```

注意观察整个方法中最重要的两行，一行是创建 `org.apache.rocketmq.common.message.Message` ，另一行是调用 `DefaultMQProducer` 的 `send` 方法。

##### 2.2.2.1 Message转Message

可能小伙伴看到 message 转 rocketMsg 的那一行源码会感觉很莫名其妙，为什么要有这么一个蜜汁操作呢？其实我们如果观察一下 `MessageBuilder` 构造出来的那个 `Message` 对象的包就知道了：

```java
package org.springframework.messaging;

public interface Message<T>
```

合着这个 `Message` 是 SpringFramework 原生的。。。RocketMQ 不认（肯定不认啦，毕竟 RocketMQ 单独使用也能用，人家只认自己的），所以只好来这么一步转换操作了。而转换的动作是利用工具类实现的，感兴趣的小伙伴可以自行深入，小册不在这里追逐过多细节。

##### 2.2.2.2 producer.send

转换完消息后，下一步就到了调用 RocketMQ 原生 API 的位置了！它调用的就是 RocketMQ 原生的 `DefaultMQProducer` 的 `send` 方法来发送消息，这也就到了下面的 4 步了，我们继续点进来看。

```java
    // 已经来到DefaultMQProducer了
    protected final transient DefaultMQProducerImpl defaultMQProducerImpl;
    public SendResult send(Message msg, long timeout) 
            throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
        msg.setTopic(withNamespace(msg.getTopic()));
        return this.defaultMQProducerImpl.send(msg, timeout);
    }

    public SendResult send(Message msg, long timeout) 
            throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
        // 此处传入了发送消息的模式为SYNC，意为同步发送
        return this.sendDefaultImpl(msg, CommunicationMode.SYNC，意为同步发送, null, timeout);
    }
```

注意，`DefaultMQProducer` 也不是真正负责发送消息的组件，真正的组件是 `DefaultMQProducerImpl` （阿熊实在是忍不住想吐槽一句，阿里巴巴的代码是真的迷惑，这两个家伙竟然都是类，而不是一个接口跟对应的实现类。。。）。而下面调用 `sendDefaultImpl` 方法，就是真正的那 4 步了，我们分别拆分来看。

### 2.3 消息校验

```java
private SendResult sendDefaultImpl(
    Message msg,
    final CommunicationMode communicationMode,
    final SendCallback sendCallback,
    final long timeout
) throws MQClientException, RemotingException, MQBrokerException, InterruptedException {
    this.makeSureStateOK();
    Validators.checkMessage(msg, this.defaultMQProducer);
    // ......
```

`DefaultMQProducerImpl` 发送消息的第一步是校验消息是否正确，通过下面的源码中可以看出，它要分别校验消息的 topic 、消息正文是否正确，而且还不能让消息正文太长了。校验的逻辑本身很简单，各位只需要知道有这么一步就行。

```java
public static void checkMessage(Message msg, DefaultMQProducer defaultMQProducer) throws MQClientException {
  if (null == msg) {
    throw new MQClientException(ResponseCode.MESSAGE_ILLEGAL, "the message is null");
  }
  // topic
  Validators.checkTopic(msg.getTopic());
  Validators.isNotAllowedSendTopic(msg.getTopic());

  // body
  if (null == msg.getBody()) {
    throw new MQClientException(ResponseCode.MESSAGE_ILLEGAL, "the message body is null");
  }

  if (0 == msg.getBody().length) {
    throw new MQClientException(ResponseCode.MESSAGE_ILLEGAL, "the message body length is zero");
  }

  if (msg.getBody().length > defaultMQProducer.getMaxMessageSize()) {
    throw new MQClientException(ResponseCode.MESSAGE_ILLEGAL,
                                "the message body size over max value, MAX: " + 
                                defaultMQProducer.getMaxMessageSize());
  }
}
```

### 2.4 路由查找

```java
    // ......
    final long invokeID = random.nextLong();
    long beginTimestampFirst = System.currentTimeMillis();
    long beginTimestampPrev = beginTimestampFirst;
    long endTimestamp = beginTimestampFirst;
    TopicPublishInfo topicPublishInfo = this.tryToFindTopicPublishInfo(msg.getTopic());
    if (topicPublishInfo != null && topicPublishInfo.ok()) {
        // ......
```

接下来的第二步是路由的查找，因为消息最终是发给 RocketMQ 中的 Broker ，而消息发送方要想知道 Broker 在哪里，需要给 NameServer 发请求，收到 NameServer 的响应后，再进行 Broker 的负载均衡，具体实现方法如下面的 `tryToFindTopicPublishInfo` 所示。

我们可以读一下这个方法，获取 Broker 信息时，首先它会先从本地缓存 `topicPublishInfoTable` 中读取，如果没有获取到，则它会尝试从 NameServer 中拉取 Broker 的信息，并且缓存到本地中。整体逻辑很简单，就是逻辑和缓存配合的过程而已。

```java
private final ConcurrentMap<String, TopicPublishInfo> topicPublishInfoTable = new ConcurrentHashMap<>();

private TopicPublishInfo tryToFindTopicPublishInfo(final String topic) {
    TopicPublishInfo topicPublishInfo = this.topicPublishInfoTable.get(topic);
    if (null == topicPublishInfo || !topicPublishInfo.ok()) {
        this.topicPublishInfoTable.putIfAbsent(topic, new TopicPublishInfo());
        this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic);
        topicPublishInfo = this.topicPublishInfoTable.get(topic);
    }

    if (topicPublishInfo.isHaveTopicRouterInfo() || topicPublishInfo.ok()) {
        return topicPublishInfo;
    } else {
        this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic, true, this.defaultMQProducer);
        topicPublishInfo = this.topicPublishInfoTable.get(topic);
        return topicPublishInfo;
    }
}
```

注意留意一个细节，获取到的 Broker 信息被封装到 `TopicPublishInfo` 中，而这其中又组合了路由的信息 `TopicRouteData` ，这个路由信息中就保存了 Broker 的地址信息 `BrokerData` ，以及 Broker 的 topic 中的队列信息 `QueueData` 。下一步在选择队列的时候，这里面的数据马上就会起作用。

```java
public class TopicPublishInfo {
    private boolean orderTopic = false;
    private boolean haveTopicRouterInfo = false;
    private List<MessageQueue> messageQueueList = new ArrayList<MessageQueue>();
    private volatile ThreadLocalIndex sendWhichQueue = new ThreadLocalIndex();
    private TopicRouteData topicRouteData;

public class TopicRouteData extends RemotingSerializable {
    private String orderTopicConf;
    private List<QueueData> queueDatas;
    private List<BrokerData> brokerDatas;
    private HashMap<String, List<String>> filterServerTable;
```

### 2.5 选择队列

```java
    // ......
    TopicPublishInfo topicPublishInfo = this.tryToFindTopicPublishInfo(msg.getTopic());
    if (topicPublishInfo != null && topicPublishInfo.ok()) {
        boolean callTimeout = false;
        MessageQueue mq = null;
        Exception exception = null;
        SendResult sendResult = null;
        int timesTotal = communicationMode == CommunicationMode.SYNC ? 1 + this.defaultMQProducer.getRetryTimesWhenSendFailed() : 1;
        int times = 0;
        String[] brokersSent = new String[timesTotal];
        for (; times < timesTotal; times++) {
            String lastBrokerName = null == mq ? null : mq.getBrokerName();
            MessageQueue mqSelected = this.selectOneMessageQueue(topicPublishInfo, lastBrokerName);
            // ......
```

第三步是选择队列，因为一个 Broker 中的一个 topic 默认会创建四条队列，而一条消息只能放入一个队列中，所以消息的发送方需要选择一个合适的队列来放入。选择消息队列的核心方法时中间 for 循环的 `this.selectOneMessageQueue` 方法，我们进入该方法。

```java
public MessageQueue selectOneMessageQueue(final TopicPublishInfo tpInfo, final String lastBrokerName) {
    return this.mqFaultStrategy.selectOneMessageQueue(tpInfo, lastBrokerName);
}

public MessageQueue selectOneMessageQueue(final TopicPublishInfo tpInfo, final String lastBrokerName) {
    if (this.sendLatencyFaultEnable) {
        // 发送延时故障启动机制，默认false 不启用 ......
    }
    return tpInfo.selectOneMessageQueue(lastBrokerName);
}

```

由上面的 `selectOneMessageQueue` 再往下调到 `mqFaultStrategy` 中的同名方法，这个方法中有一个特殊的判断机制。RocketMQ 内部有设计一个“发送延时故障启动机制”，这个机制可以使得消息的生产者在发送消息时，针对某个发送消息耗时过长的 broker 予以暂时屏蔽，以提高整体的运行效率。不过默认情况下这个 `sendLatencyFaultEnable` 的值是 **false** ，不会开启，所以我们就跳过这个 if 结构了（感兴趣的小伙伴可以自行查看源码），直接往下看 `tpInfo.selectOneMessageQueue` 方法。

```java
public MessageQueue selectOneMessageQueue(final String lastBrokerName) {
    if (lastBrokerName == null) {
        return selectOneMessageQueue();
    } else {
        // 排除掉上一次使用的broker再选择 ......
    }
}
```

注意观察，这个 `selectOneMessageQueue` 方法有传入一个 `lastBrokerName` ，这个设计是图什么呢？其实也不难理解，正常来讲我们不可能把消息全部塞入一个 broker 中（不然哪来的高可用一说呢），而是通过一种负载均衡机制，尽可能合理均匀地将消息放入 broker 集群的每个实例中，而引入 `lastBrokerName` 参数后，就意味着上次使用的那个 broker 这次就不再使用了，下面的 else 块中会去除掉那个 broker ，在剩余的队列中再选择一个。

那第一次发送消息的时候如何选择呢？我们进入到重载的没有参数的 `selectOneMessageQueue` 方法中，可以看到它就是一个简单的取模运算，利用自然数增长算法实现轮询。

```java


    public MessageQueue selectOneMessageQueue() {
        int index = this.sendWhichQueue.incrementAndGet();
        int pos = Math.abs(index) % this.messageQueueList.size();
        if (pos < 0)
            pos = 0;
        return this.messageQueueList.get(pos);
    }
```

轮询完毕后，就相当于我们拿到了当前这条消息要发送的目标 broker 、目标 topic 中的队列。

### 2.6 真正发送消息

```java
// ......
MessageQueue mqSelected = this.selectOneMessageQueue(topicPublishInfo, lastBrokerName);
if (mqSelected != null) {
  mq = mqSelected;
  brokersSent[times] = mq.getBrokerName();
  try {
    beginTimestampPrev = System.currentTimeMillis();
    if (times > 0) {
      //Reset topic with namespace during resend.
      msg.setTopic(this.defaultMQProducer.withNamespace(msg.getTopic()));
    }
    long costTime = beginTimestampPrev - beginTimestampFirst;
    if (timeout < costTime) {
      callTimeout = true;
      break;
    }

    sendResult = this.sendKernelImpl(msg, mq, communicationMode, sendCallback, topicPublishInfo, 
                                     timeout - costTime);
    endTimestamp = System.currentTimeMillis();
    this.updateFaultItem(mq.getBrokerName(), endTimestamp - beginTimestampPrev, false);
    switch (communicationMode) {
      case ASYNC:
        return null;
      case ONEWAY:
        return null;
      case SYNC:
        if (sendResult.getSendStatus() != SendStatus.SEND_OK) {
          if (this.defaultMQProducer.isRetryAnotherBrokerWhenNotStoreOK()) {
            continue;
          }
        }
        return sendResult;
      default:
        break;
    }
    // ......
```

继续往下看，当拿到 `MessageQueue` 后，最后一步就是真正发送消息了，发送的动作在中间的 `this.sendKernelImpl` 方法中，由于这个方法的内部逻辑弯弯道道有点多，我们在这里简单总结一下即可，小伙伴们可以不用深入研究（当然感兴趣的小伙伴可以结合源码翻看）。

1.  sendKernelImpl 方法的内部会先准备好 Broker 的地址（如果 broker 的地址为空，则需要从 NameServer 中拉取），之后给消息设置一个全局唯一 id 标识，再然后会针对大体量消息进行压缩（超过 4k 的消息需要被压缩）。
2. 消息压缩完毕后，就可以真正发送消息了，不过在发送消息之前 RocketMQ 给我们提供了一个切入点，我们可以编写类实现 `CheckForbiddenHook` 接口，并设置到 IOC 容器，以完成对切入点的逻辑扩展。
3. 消息真正发送的时候，它的底层是使用 Netty 给 Broker 发送一个 RPC 调用。
4. 当消息发送完毕后，会返回一个 `SendResult` 对象，其内部封装了消息发送的结果。
5. 以上就是针对 `RocketMQTemplate` 的一次同步消息发送，底层经历的全部过程，异步消息和单向消息的发送过程也是类似的，小伙伴们如果感兴趣，也可以参照源码和上述的过程来阅读和分析，小册就不展开了。

## 3. 消息监听的工作原理

本章的最后，我们再来看看 RocketMQ 在整合 SpringBoot 后，通过注解的方式编写的消息监听器又是如何监听到消息的。

正常情况来讲，消息的消费者在消费消息时，有两种模式：**push 推** 和 **pull 拉** ，pull 很好理解，消息的消费者主动从 RocketMQ 中拉取消息进行消费即可，而 push 的模式则是 RocketMQ 在收到消息时主动推送给消息的消费者。RocketMQ 在实际实现的时候采用的是 pull 模式为主，push 模式是基于 pull 模式进行的一层封装，借助拉取任务来实现的。这个各位可以先了解。

下面我们还是来深入源码，还记得上面我们在看 `@RocketMQMessageListener` 注解的扫描时最终封装的那个家伙吗？我们编写的那些 `RocketMQListener` 最终都被封装为了一个个的 `DefaultRocketMQListenerContainer` ，它内部组合了 `RocketMQListener` 和 RocketMQ 原生的 API `DefaultMQPushConsumer` 。

```java
public class DefaultRocketMQListenerContainer implements InitializingBean,
        RocketMQListenerContainer, SmartLifecycle, ApplicationContextAware {
    // ......

    private RocketMQListener rocketMQListener;

    private DefaultMQPushConsumer consumer;

    // ......

```

请注意，这个类实现了两个关键的生命周期接口：**`InitializingBean`** 和 **`SmartLifecycle`** ，这分别是 SpringFramework 给我们提供的 Bean 的初始化回调扩展，以及 IOC 容器初始化完毕后回调的扩展点接口！这个类在这里实现这两个接口 ，意欲何为呢？我们就从这里开始看起。

### 3.1 DefaultRocketMQListenerContainer#afterPropertiesSet

```java
@Override
public void afterPropertiesSet() throws Exception {
    initRocketMQPushConsumer();

    this.messageType = getMessageType();
    this.methodParameter = getMethodParameter();
    log.debug("RocketMQ messageType: {}", messageType);
}
```

`DefaultRocketMQListenerContainer` 这个类的对象在初始化时，内部是需要初始化 RocketMQ 原生 `DefaultMQPushConsumer` 对象的，所以这个切入点刚好适合初始化。下面的方法我们可以简单扫一遍，它分别有如下几个步骤：

- `DefaultMQPushConsumer` 的创建；
- Consumer 属性配置；
- 设置消息监听模式、过滤模式、监听模式等；
- 扩展的初始化回调（`RocketMQPushConsumerLifecycleListener` 等）；

```java
private void initRocketMQPushConsumer() throws MQClientException {
    // 前置校验 ......
    RPCHook rpcHook = RocketMQUtil.getRPCHookByAkSk(applicationContext.getEnvironment(),
            this.rocketMQMessageListener.accessKey(), this.rocketMQMessageListener.secretKey());
    boolean enableMsgTrace = rocketMQMessageListener.enableMsgTrace();
    if (Objects.nonNull(rpcHook)) {
        consumer = new DefaultMQPushConsumer(consumerGroup, rpcHook, new AllocateMessageQueueAveragely(),
            enableMsgTrace, this.applicationContext.getEnvironment().
            resolveRequiredPlaceholders(this.rocketMQMessageListener.customizedTraceTopic()));
        consumer.setVipChannelEnabled(false);
    } else {
        log.debug("Access-key or secret-key not configure in " + this + ".");
        consumer = new DefaultMQPushConsumer(consumerGroup, enableMsgTrace,
            this.applicationContext.getEnvironment().
                resolveRequiredPlaceholders(this.rocketMQMessageListener.customizedTraceTopic()));
    }
    consumer.setNamespace(namespace);
    consumer.setInstanceName(RocketMQUtil.getInstanceName(nameServer));
    // 一堆set ......
    
    switch (messageModel) {
        case BROADCASTING:
            consumer.setMessageModel(org.apache.rocketmq.common.protocol.heartbeat.MessageModel.BROADCASTING);
            break;
        case CLUSTERING:
            consumer.setMessageModel(org.apache.rocketmq.common.protocol.heartbeat.MessageModel.CLUSTERING);
            break;
        default:
            throw new IllegalArgumentException("Property 'messageModel' was wrong.");
    }

    switch (selectorType) {
        case TAG:
            consumer.subscribe(topic, selectorExpression);
            break;
        case SQL92:
            consumer.subscribe(topic, MessageSelector.bySql(selectorExpression));
            break;
        default:
            throw new IllegalArgumentException("Property 'selectorType' was wrong.");
    }

    switch (consumeMode) {
        case ORDERLY:
            consumer.setMessageListener(new DefaultMessageListenerOrderly());
            break;
        case CONCURRENTLY:
            consumer.setMessageListener(new DefaultMessageListenerConcurrently());
            break;
        default:
            throw new IllegalArgumentException("Property 'consumeMode' was wrong.");
    }
    // if String is not is equal "true" TLS mode will represent the as default value false
    consumer.setUseTLS(new Boolean(tlsEnable));

    if (rocketMQListener instanceof RocketMQPushConsumerLifecycleListener) {
        ((RocketMQPushConsumerLifecycleListener) rocketMQListener).prepareStart(consumer);
    } else if (rocketMQReplyListener instanceof RocketMQPushConsumerLifecycleListener) {
        ((RocketMQPushConsumerLifecycleListener) rocketMQReplyListener).prepareStart(consumer);
    }
}
```

经过 `initRocketMQPushConsumer` 方法的处理后，`DefaultRocketMQListenerContainer` 内部的 `DefaultMQPushConsumer` 就已经初始化好了。

### 3.2 DefaultRocketMQListenerContainer#start

```java
private DefaultMQPushConsumer consumer;

@Override
public void start() {
    if (this.isRunning()) {
        throw new IllegalStateException("container already running. " + this.toString());
    }

    try {
        consumer.start();
    } // catch throw ex ......
    this.setRunning(true);
    log.info("running container: {}", this.toString());
}
```

接下来是实现了 `SmartLifecycle` 的逻辑 `start` 方法，很明显，`DefaultRocketMQListenerContainer` 的 `start` 方法就是回调了内部 `DefaultMQPushConsumer` 的 `start` 方法，而原生 `DefaultMQPushConsumer` 的 `start` 就是真正启动与 RocketMQ 的监听了。

具体来看，`DefaultMQPushConsumer` 的 `start` 方法分为以下几步，且这里面有与 RocketMQ 中 Broker 的联动，所以我们再看源码就不是那么方便了，小册在这里先把步骤列出来：

1. `DefaultMQPushConsumer` 内部组合了一个 `DefaultMQPushConsumerImpl` ，`start` 会触发它的 `start` 方法逻辑；
2. `DefaultMQPushConsumerImpl` 在内部先检查消息接入 RocketMQ 的配置是否合法，之后构建监听 topic 的订阅信息并保存到本地（相当于本地缓存）；
3. 之后构建 `MQClientInstance` （这个 API 在上面消息生产者发送消息的过程中也有使用）以及负载均衡的策略 `RebalanceImpl` 的配置属性设置；
4. 下面有一个消息消费模式的设置，由于广播模式和集群模式对于消费者集群的消费方式不同，所以要对应的分别开来（广播模式只需要将消费进度存到本地，而集群模式需要把消费进度放到 RocketMQ 中）；

> 广播模式由于每个消费者都需要消费消息，所以各自记录各自的就行；而集群模式只能由消费者集群的某一个消费，所以必须要把消息的消费记录记录在 Broker 中。

5. 启动内部的 `ConsumeMessageService` 和 `MQClientInstance` ，开始启动监听服务，注册消费者。

经历过这几个阶段后，Consumer 端的监听就算注册完毕了。后续的源码部分小伙伴们可以自行借助 IDE 查看，内容很多小册就不再贴出了。

### 3.3 消息到达消费者

当消息到达消费者时，如何能让我们编写的 `RocketMQListener` 被触发呢？这就是接下来我们要研究的

还记得上面我们说的，RocketMQ 的消费者监听消息都是 **pull** 模式吗？这个 pull 的动作反映到源码上，可以来到 `DefaultMQPushConsumerImpl` 的 `pullMessage` 方法中查看。这个方法的中间部分，有一个构造 `PullCallback` 的动作，这个动作的 `onSuccess` 方法中，switch 部分的 **FOUND** 分支有一个调用 `consumeMessageService.submitConsumeRequest` 方法的动作：

```java
    DefaultMQPushConsumerImpl.this.consumeMessageService.submitConsumeRequest(
            pullResult.getMsgFoundList(),
            processQueue,
            pullRequest.getMessageQueue(),
            dispatchToConsume);

private final ThreadPoolExecutor consumeExecutor;

public void submitConsumeRequest(
    final List<MessageExt> msgs,
    final ProcessQueue processQueue,
    final MessageQueue messageQueue,
    final boolean dispatchToConsume) {
    final int consumeBatchSize = this.defaultMQPushConsumer.getConsumeMessageBatchMaxSize();
    if (msgs.size() <= consumeBatchSize) {
        ConsumeRequest consumeRequest = new ConsumeRequest(msgs, processQueue, messageQueue);
        try {
            this.consumeExecutor.submit(consumeRequest);
        } // catch ......
    } // else ......
}
```

为什么要看这个方法调用呢，我们往里看，这里面有一个构造 `ConsumeRequest` 的动作，并且将创建的这个对象提交到线程池中（注意构造的时候传入了一个 `List<MessageExt>` 集合，里面存放了拉取到的消息）。各位应该能想到，能提交到线程池的对象肯定都是实现了 `Runnable` 接口或者 `Callable` 接口的吧，那对应的里面的逻辑肯定就是 `run` 或者 `call` 方法，我们继续往里面走。

```java
@Override
public void run() {
    // 前置处理 ......
    MessageListenerConcurrently listener = ConsumeMessageConcurrentlyService.this.messageListener;
    ConsumeConcurrentlyContext context = new ConsumeConcurrentlyContext(messageQueue);
    ConsumeConcurrentlyStatus status = null;
    defaultMQPushConsumerImpl.resetRetryAndNamespace(msgs, defaultMQPushConsumer.getConsumerGroup());

    ConsumeMessageContext consumeMessageContext = null;
    // 构造ConsumeMessageContext ......

    long beginTimestamp = System.currentTimeMillis();
    boolean hasException = false;
    ConsumeReturnType returnType = ConsumeReturnType.SUCCESS;
    try {
        if (msgs != null && !msgs.isEmpty()) {
            for (MessageExt msg : msgs) {
                MessageAccessor.setConsumeStartTimeStamp(msg, String.valueOf(System.currentTimeMillis()));
            }
        }
        status = listener.consumeMessage(Collections.unmodifiableList(msgs), context);
    } // catch logger ......
    // ......
}
```

看这段源码中的 try 块，它会将上面构造 `ConsumeRequest` 时传入的 `List<MessageExt>` 集合继续向 `consumeMessage` 方法中传入，我们还得继续往里深入。不过在此之前我们要先知道一件事：`MessageListenerConcurrently` 也好，或者在另一种消息消费模式的 `MessageListenerOrderly` 中也好，最终都会拿到这组 `MessageExt` 对象，并且逐个读取和消费。

```java
@Override
public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> msgs, ConsumeConcurrentlyContext context) {
    for (MessageExt messageExt : msgs) {
        log.debug("received msg: {}", messageExt);
        try {
            long now = System.currentTimeMillis();
            handleMessage(messageExt);
            long costTime = System.currentTimeMillis() - now;
            log.debug("consume {} cost: {} ms", messageExt.getMsgId(), costTime);
        } // catch ......
    }
    return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
}

private void handleMessage(
    MessageExt messageExt) throws MQClientException, RemotingException, InterruptedException {
    if (rocketMQListener != null) {
        rocketMQListener.onMessage(doConvertMessage(messageExt));
    } // else if ......
}
```

从上面的这段中可以发现，内部要干的事情就是循环读取这组 `MessageExt` 对象，并逐个处理。而处理单条消息的 `handleMessage` 方法中终于看到了我们自己编写的那个 `RocketMQListener` 的调用，也就是这里处理了 `RocketMQListener` 中指定的消息类型的泛型。将 `MessageExt` 转换为具体的消息体类型后，便可以触发 `RocketMQListener` 的 `onMessage` 方法，完成消息的消费。

到此为止，有关消息监听部分的工作原理，我们也就简单了解完毕了。

【通过这 4 章的内容，我们完整的了解和接触了有关消息中间件的设计，已经 RocketMQ 的整合、使用和原理。RocketMQ 在业务场景中使用的还是很多的，各位在实际使用时要结合业务分析，合理使用消息来做到业务逻辑解耦。接下来的 3 章我们来研究另一个经常使用的技术点：定时任务】