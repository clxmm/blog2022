---
title: 01
toc: true
tags: 06-服务发现-EurekaClient的初始化和服务注册原理
categories: 
    - [java]
    - [springcloudNetflix]
---

【接前章】

## 5. EurekaClientAutoConfiguration注册和导入的核心组件

之前在翻看 `EurekaClientAutoConfiguration` 的注解声明时，注意里面有一个 `@Import` ：

```java
@Import(DiscoveryClientOptionalArgsConfiguration.class)

```

<!--more-->

那咱先看看这个导入的配置类都干了什么：

### 5.1 DiscoveryClientOptionalArgsConfiguration

它只注册了两个组件，分别来看。

#### 5.1.1 RestTemplateDiscoveryClientOptionalArgs

```java
@Bean
@ConditionalOnMissingClass("com.sun.jersey.api.client.filter.ClientFilter")
@ConditionalOnMissingBean(value = AbstractDiscoveryClientOptionalArgs.class, search = SearchStrategy.CURRENT)
public RestTemplateDiscoveryClientOptionalArgs restTemplateDiscoveryClientOptionalArgs() {
  return new RestTemplateDiscoveryClientOptionalArgs();
}
```

从类名上也能看得出来，它是支持 `RestTemplate` 的一个服务发现的参数配置组件，这里有一个扩展设计。EurekaClient 在设计与 EurekaServer 通信时，没有限制死必须要用 **Jersey** ，可以换做 `RestTemplate` 来代替 `JerseyApplicationClient`，借助 IDEA 可以搜到一个类叫 `RestTemplateEurekaHttpClient` ，它实现了 `EurekaApplicationClient` 接口（同样 `JerseyApplicationClient` 也实现了）。回到这个组件上，它的设计就是为了万一咱要替换掉原有的 Jersey 来使用 `RestTemplate` 与 EurekaServer 交互，那这个参数配置组件就可以起到支撑效果。如果真的要使用 `RestTemplate` ，可以将 jersey-client 从 Maven 坐标中移除掉，根据上面的 `@ConditionalOnMissingClass` 条件限制，则这个组件会生效。

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
    <exclusions>
        <exclusion>
            <groupId>com.sun.jersey</groupId>
            <artifactId>jersey-client</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

#### 5.1.2 MutableDiscoveryClientOptionalArgs

```java
@Bean
@ConditionalOnClass(name = "com.sun.jersey.api.client.filter.ClientFilter")
@ConditionalOnMissingBean(value = AbstractDiscoveryClientOptionalArgs.class, search = SearchStrategy.CURRENT)
public MutableDiscoveryClientOptionalArgs discoveryClientOptionalArgs() {
    return new MutableDiscoveryClientOptionalArgs();
}

```

这个组件从 `@ConditionalOnClass` 上就已经能看出来，它与上面 `RestTemplateEurekaHttpClient` 的参数配置组件的判断条件正好相反，也说明它是配合 `JerseyApplicationClient` 的。

### 5.2 ManagementMetadataProvider

```java
@Bean
@ConditionalOnMissingBean
public ManagementMetadataProvider serviceManagementMetadataProvider() {
    return new DefaultManagementMetadataProvider();
}
```

这个类从类名上看，叫管理元数据的提供者，再看一眼文档注释，发现就一句话，还就是对这个类名的重复（确定人类的本质不是复读机？）：

> Provider for Eureka-specific management metadata.
>
> Eureka特定管理元数据的提供者。

实际上，这个元数据的提供者，是一个用来处理健康检查和状态页面地址的封装类，它在下面的 `EurekaInstanceConfigBean` 中有加以利用。

### 5.3 EurekaInstanceConfigBean

初始化 `EurekaInstanceConfigBean` 的源码很长，这里咱只抽取出一些关键的部分：

```java
Bean
@ConditionalOnMissingBean(value = EurekaInstanceConfig.class, search = SearchStrategy.CURRENT)
public EurekaInstanceConfigBean eurekaInstanceConfigBean(InetUtils inetUtils,
        ManagementMetadataProvider managementMetadataProvider) {
    // 获取配置
    // ......
    EurekaInstanceConfigBean instance = new EurekaInstanceConfigBean(inetUtils);

    // 设置配置到instance里
    // ......

    ManagementMetadata metadata = managementMetadataProvider.get(instance, serverPort,
            serverContextPath, managementContextPath, managementPort);

    if (metadata != null) {
        instance.setStatusPageUrl(metadata.getStatusPageUrl());
        instance.setHealthCheckUrl(metadata.getHealthCheckUrl());
        if (instance.isSecurePortEnabled()) {
            instance.setSecureHealthCheckUrl(metadata.getSecureHealthCheckUrl());
        }
        Map<String, String> metadataMap = instance.getMetadataMap();
        metadataMap.computeIfAbsent("management.port",
                k -> String.valueOf(metadata.getManagementPort()));
    }
    else {
        // without the metadata the status and health check URLs will not be set
        // and the status page and health check url paths will not include the
        // context path so set them here
        // 如果没有元数据，则不会设置状态和运行状况检查URL，
        // 并且状态页和运行状况检查URL路径将不包含上下文路径，因此请在此处进行设置
        if (StringUtils.hasText(managementContextPath)) {
            instance.setHealthCheckUrlPath(
                    managementContextPath + instance.getHealthCheckUrlPath());
            instance.setStatusPageUrlPath(
                    managementContextPath + instance.getStatusPageUrlPath());
        }
    }

    setupJmxPort(instance, jmxPort);
    return instance;
}
```

这里面怎么初始化的咱不是特别关心，咱只需要联想起一点就OK：在 EurekaServer 中，是不是也有一个类似的叫 `EurekaServerConfigBean` 吧（第 4 章第 6 节）。这个 `EurekaInstanceConfigBean` 也跟它一样，是封装 EurekaClient 相关的一些配置项，后面慢慢会看到它的使用。

### 5.4 DiscoveryClient

```java
@Bean
public DiscoveryClient discoveryClient(EurekaClient client, EurekaClientConfig clientConfig) {
    return new EurekaDiscoveryClient(client, clientConfig);
}
```

这个组件一看就感觉很眼熟，它正好跟 `@EnableDiscoveryClient` 注解相呼应。先看一眼文档注释：

> Represents read operations commonly available to discovery services such as Netflix Eureka or consul.io.
>
> 表示发现服务（例如Netflix Eureka或consul.io）通常可用的读取操作。

可以看出来它是配合服务注册和发现中心的，这里面注入了两个组件：

- EurekaClient：在下面的 `RefreshableEurekaClientConfiguration` 中有初始化
- EurekaClientConfig：上面的配置模型已经封装好

由此也可以猜测：它依赖了 `EurekaClient` ，那是不是啥事都让 `EurekaClient` 干，它自己起一个调度作用呢？答案是肯定的，我用一张图来解释几个组件的关系：

![](./img/20203/05/cloud06-2.png)

由图所示，`EurekaClient` 与真正地 **EurekaServer** 注册中心交互，`EurekaDiscoveryClient` 只是依赖了 `EurekaClient` ，而 `EurekaDiscoveryClient` 又实现了 `DiscoveryClient` ，由此可以实现 **SpringCloud 对 Eureka 的整合**。至于 `EurekaClient` 是如何与 EurekaServer 交互，咱到下面看 `EurekaClient` 的初始化，以及后面专门来解析。

### 5.5 EurekaServiceRegistry

```java
@Bean
public EurekaServiceRegistry eurekaServiceRegistry() {
    return new EurekaServiceRegistry();
}
```

这个组件没有任何文档注释，只能从结构看起了：

```java
public class EurekaServiceRegistry implements ServiceRegistry<EurekaRegistration>

```

它实现了 `ServiceRegistry` 接口（当前开发环境下也只有它自己实现了），而这个接口的描述：

> Contract to register and deregister instances with a Service Registry.
>
> 与服务注册中心签订注册和注销实例的契约。

很直白，它就是**微服务实例与注册中心的连接契约(租约)**，并且这个 `ServiceRegistry` 接口把注册和注销的动作都抽象出来了：

```java
public interface ServiceRegistry<R extends Registration> {
    void register(R registration);
    void deregister(R registration);
    void close();
```

注册和注销时都传入了一个 `Registration` 类型的注册信息对象，而 `Registration` 又继承 `ServiceInstance` 接口，`ServiceInstance` 接口中定义了一些服务实例注册到注册中心的一些核心参数（服务实例id，主机名，端口，服务状态等），那么当服务实例触发 `register` 方法时，注册中心按理应该收到服务实例的属性并处理注册动作。并且咱也能大概猜测，EurekaClient 启动时，`register` 方法会被调用；EurekaClient下线时，`deregister` 方法会被调用。

可翻了这么多，这些信息又起什么作用呢？这个咱得借助官方文档来解释了，我把这一段摘下来：

> Spring Cloud Commons provides a `/service-registry` actuator endpoint. This endpoint relies on a `Registration` bean in the Spring Application Context. Calling `/service-registry` with GET returns the status of the `Registration`. Using POST to the same endpoint with a JSON body changes the status of the current `Registration` to the new value. The JSON body has to include the `status` field with the preferred value. Please see the documentation of the `ServiceRegistry`implementation you use for the allowed values when updating the status and the values returned for the status. For instance, Eureka’s supported statuses are `UP`, `DOWN`, `OUT_OF_SERVICE`, and `UNKNOWN`.

这段文档大体要解释的内容有这么几个：

- 它是要配合 **acturator** 使用的（ `spring-boot-starter-actuator` ）
  - 补充一点，如果引入 `spring-boot-starter-actuator` 依赖，会自动开启安全认证，可以在 `application.properties` 中配置 `management.security.enabled=false` 来禁止安全认证
- 发送 **GET** 请求的 `/service-registry` 可以获取服务实例状态
- 发送 **POST** 请求的 `/service-registry` 可以修改服务实例状态

那这样咱也明白了，这个契约可以获取和动态改变服务实例的状态，它的作用生效位置在下面的 `EurekaAutoServiceRegistration` 中。

### 5.6 EurekaAutoServiceRegistration

```java
@Bean
@ConditionalOnBean(AutoServiceRegistrationProperties.class)
@ConditionalOnProperty(value = "spring.cloud.service-registry.auto-registration.enabled", matchIfMissing = true)
public EurekaAutoServiceRegistration eurekaAutoServiceRegistration(
        ApplicationContext context, EurekaServiceRegistry registry,
        EurekaRegistration registration) {
    return new EurekaAutoServiceRegistration(context, registry, registration);
}
```

这个组件依赖了上面的 `EurekaServiceRegistry` ，那自然想到可能是它负责调 `EurekaServiceRegistry` 的 `register` 方法实现服务注册咯？看一眼这个类的继承关系：

```java
public class EurekaAutoServiceRegistration implements AutoServiceRegistration,
		SmartLifecycle, Ordered, SmartApplicationListener
```

注意它实现了 `SmartLifecycle` 接口！那咱直接看 `start` 方法：

```java
public void start() {
    // only set the port if the nonSecurePort or securePort is 0 and this.port != 0
    if (this.port.get() != 0) {
        if (this.registration.getNonSecurePort() == 0) {
            this.registration.setNonSecurePort(this.port.get());
        }
        if (this.registration.getSecurePort() == 0 && this.registration.isSecure()) {
            this.registration.setSecurePort(this.port.get());
        }
    }

    // only initialize if nonSecurePort is greater than 0 and it isn't already running
    // because of containerPortInitializer below
    if (!this.running.get() && this.registration.getNonSecurePort() > 0) {
        this.serviceRegistry.register(this.registration);
        this.context.publishEvent(new InstanceRegisteredEvent<>(this,
                this.registration.getInstanceConfig()));
        this.running.set(true);
    }
}
```

前面设置端口的部分咱不关心，注意看下面的 if 结构中，它调用了 `this.serviceRegistry.register(this.registration);` ！在这里它触发了 `EurekaServiceRegistry` 的 `register` 方法！所以咱就知道，`EurekaServiceRegistry` 的触发是借助 `EurekaAutoServiceRegistration` 。

此外，注意下 `EurekaAutoServiceRegistration` 的另一个实现的接口：`SmartApplicationListener` ，它是监听器的实现，由于没有在泛型中标明要监听的事件类型，故可以断定它监听了不止一个事件，在源码中也有体现：

```java
public void onApplicationEvent(ApplicationEvent event) {
    if (event instanceof WebServerInitializedEvent) {
        onApplicationEvent((WebServerInitializedEvent) event);
    }
    else if (event instanceof ContextClosedEvent) {
        onApplicationEvent((ContextClosedEvent) event);
    }
}
```

可以发现事件与动作的对应关系：

- `WebServerInitializedEvent` → `start` → `register`
- `ContextClosedEvent` → `stop` → `deregister`

由此也进一步明白了组件之间这一层一层的调用关系，其实这个调用链很复杂，小册这里先列一下调用顺序，回头看到的时候具体再说：

- **EurekaAutoServiceRegistration#start()** ：`SmartLifecycle` 的生命周期触发
- **EurekaServiceRegistry#register(EurekaRegistration)** ：调用服务注册组件，开始向注册中心发起注册
- **ApplicationInfoManager#setInstanceStatus(reg.getInstanceConfig().getInitialStatus())** ：更新服务实例的状态为**UP**
- **StatusChangeListener#notify(new StatusChangeEvent(prev, next))** ：更新状态导致监听器触发
- **InstanceInfoReplicator#onDemandUpdate()** ：`DiscoveryClient` - 1325行，`InstanceInfoReplicator` 负责给注册中心上报状态变化
- **InstanceInfoReplicator.this#run()** ：`InstanceInfoReplicator` - 101行，由此触发 `DiscoveryClient` 来与注册中心通信
- **DiscoveryClient#register()** ：发起注册实例的动作
- **EurekaTransport.registrationClient#register(instanceInfo)** ：使用 **Jersey** 或 **RestTemplate** 向注册中心请求注册

### 5.7 EurekaClientConfiguration

这部分注册的组件都是在非 Refreshable 环境下的，由于通常情况下咱使用 SpringCloud 微服务都会搭配分布式配置中心，也都会使用动态配置，故这部分不会起效，直接往下看。

### 5.8 RefreshableEurekaClientConfiguration

这个配置类上面的注解注定了它会生效：

```java
@Configuration
@ConditionalOnRefreshScope
protected static class RefreshableEurekaClientConfiguration
```

`@ConditionalOnRefreshScope` ，不看源码实现也知道它是配合 **refresh** 的作用域一起生效。那咱来看它初始化的几个组件。

#### 5.8.1 【启动原理】CloudEurekaClient

这段源码中有一些单行注释，我这里直接翻译成中文了，看起来还好看点：

```java
@Bean(destroyMethod = "shutdown")
@ConditionalOnMissingBean(value = EurekaClient.class, search = SearchStrategy.CURRENT)
@org.springframework.cloud.context.config.annotation.RefreshScope
@Lazy
public EurekaClient eurekaClient(ApplicationInfoManager manager,
        EurekaClientConfig config, EurekaInstanceConfig instance,
        @Autowired(required = false) HealthCheckHandler healthCheckHandler) {
    // 如果我们使用ApplicationInfoManager的代理，当在CloudEurekaClient上调用shutdown时，
    // 我们可能会遇到问题，在这里请求ApplicationInfoManager bean，
    // 但由于我们正在关闭，因此不允许。为了避免这种情况，我们直接使用该对象。
    ApplicationInfoManager appManager;
    if (AopUtils.isAopProxy(manager)) {
        appManager = ProxyUtils.getTargetObject(manager);
    }
    else {
        appManager = manager;
    }
    CloudEurekaClient cloudEurekaClient = new CloudEurekaClient(appManager,
            config, this.optionalArgs, this.context);
    cloudEurekaClient.registerHealthCheck(healthCheckHandler);
    return cloudEurekaClient;
}

```

这里前面的部分是获取真正的 `ApplicationInfoManager` 类型的对象，咱直接过了，下面创建的对象类型才是重点：**`CloudEurekaClient`** 。

##### 5.8.1.1 CloudEurekaClient的构造方法

先看一眼它的构造方法：

```java
public CloudEurekaClient(ApplicationInfoManager applicationInfoManager,
        EurekaClientConfig config, AbstractDiscoveryClientOptionalArgs<?> args,
        ApplicationEventPublisher publisher) {
    super(applicationInfoManager, config, args);
    this.applicationInfoManager = applicationInfoManager;
    this.publisher = publisher;
    this.eurekaTransportField = ReflectionUtils.findField(DiscoveryClient.class,
            "eurekaTransport");
    ReflectionUtils.makeAccessible(this.eurekaTransportField);
}
```

除了调父类的构造方法之外，其余的部分也没看出什么花来，那咱就往父类的构造方法里面爬：

```java
public DiscoveryClient(ApplicationInfoManager applicationInfoManager, final EurekaClientConfig config, 
        AbstractDiscoveryClientOptionalArgs args) {
    this(applicationInfoManager, config, args, ResolverUtils::randomize);
}
```

又是构造方法的连环调，那咱继续往里点：

```java
public DiscoveryClient(ApplicationInfoManager applicationInfoManager, final EurekaClientConfig config, 
        AbstractDiscoveryClientOptionalArgs args, EndpointRandomizer randomizer) {
    this(applicationInfoManager, config, args, new Provider<BackupRegistry>() {
        // ......
        }
    }, randomizer);
}
```

还是继续往里调，但这里面由于源码太长了（150行+），这里拆分成几个重点环节来看，整体的初始化流程小册不作概述，小伙伴们可以自行借助IDE翻阅。

##### 5.8.1.2 DiscoveryClient的最长构造方法-填充组件

```java
DiscoveryClient(ApplicationInfoManager applicationInfoManager, EurekaClientConfig config, AbstractDiscoveryClientOptionalArgs args,
                Provider<BackupRegistry> backupRegistryProvider, EndpointRandomizer endpointRandomizer) {
    if (args != null) {
        this.healthCheckHandlerProvider = args.healthCheckHandlerProvider;
        this.healthCheckCallbackProvider = args.healthCheckCallbackProvider;
        this.eventListeners.addAll(args.getEventListeners());
        this.preRegistrationHandler = args.preRegistrationHandler;
    } else {
        this.healthCheckCallbackProvider = null;
        this.healthCheckHandlerProvider = null;
        this.preRegistrationHandler = null;
    }

```

这一步只是将 `AbstractDiscoveryClientOptionalArgs` 中的一些组件填充到 `DiscoveryClient` 中而已，不难理解。看一眼这些组件的作用（仅文档注释）：

- healthCheckHandlerProvider → `Provider<HealthCheckHandler>`

> Eureka server normally just relies on heartbeats to identify the status of an instance. Application could decide to implement their own healthpage check here or use the built-in jersey resource HealthCheckResource.	
>
> EurekaServer 通常仅依靠心跳来确定实例的状态。应用程序可以决定在此处实施自己的健康页检查，或使用内置 Jersey 的资源 `HealthCheckResource` 。		

- healthCheckCallbackProvider → `Provider<HealthCheckCallback>`

  > This provides a more granular healthcheck contract than the existing HealthCheckCallback.
  >
  > 与现有的 `HealthCheckCallback` 相比，这提供了更精细的健康检查契约。

- args.preRegistrationHandler → `PreRegistrationHandler`

  > A handler that can be registered with an EurekaClient at creation time to execute pre registration logic. The pre registration logic need to be synchronous to be guaranteed to execute before registration.
  >
  > 可以在创建时向 EurekaClient 注册以执行预注册逻辑的处理程序。预注册逻辑需要同步，以确保在注册之前执行。

##### 5.8.1.3 填充组件2

```java
    this.applicationInfoManager = applicationInfoManager;
    InstanceInfo myInfo = applicationInfoManager.getInfo();

    clientConfig = config;
    staticClientConfig = clientConfig;
    transportConfig = config.getTransportConfig();
    instanceInfo = myInfo;
    if (myInfo != null) {
        appPathIdentifier = instanceInfo.getAppName() + "/" + instanceInfo.getId();
    } else {
        logger.warn("Setting instanceInfo to a passed in null value");
    }
    this.backupRegistryProvider = backupRegistryProvider;
    this.endpointRandomizer = endpointRandomizer;
    this.urlRandomizer = new EndpointUtils.InstanceInfoBasedUrlRandomizer(instanceInfo);
```

这一部分也是填充组件，不过这一次填充的组件是方法形参的其他几个组件了。同样看看这几个组件的作用：

- `ApplicationInfoManager` （5.8.2节）

  - > The class that initializes information required for registration with Eureka Server and to be discovered by other components. The information required for registration is provided by the user by passing the configuration defined by the contract in EurekaInstanceConfig }.AWS clients can either use or extend CloudInstanceConfig. Other non-AWS clients can use or extend either MyDataCenterInstanceConfig or very basic AbstractInstanceConfig.
    >
    >  该类用于初始化向 Eureka Server 注册并被其他组件发现所需的信息。 用户通过传递 `EurekaInstanceConfig` 契约配置中定义的配置来提供注册所需的信息。AWS客户端可以使用或扩展 `CloudInstanceConfig` 。其他非AWS客户端可以使用或扩展 `MyDataCenterInstanceConfig` 或非常基本的 `AbstractInstanceConfig` 。

- backupRegistryProvider → Provider\<BackupRegistry>

  - > A simple contract for eureka clients to fallback for getting registry information in case eureka clients are unable to retrieve this information from any of the eureka servers. This is normally not required, but for applications that cannot exist without the registry information it can provide some additional reslience.
    >
    >  如果 EurekaClient 无法从任何 EurekaServer 检索此信息，则为 EurekaClient 回退以获得注册表信息的简单契约。 这个组件通常这不是必需的，但是对于没有注册表信息就无法存在的应用程序，它可以提供一些额外的可靠性。

- EndpointRandomizer → ResolverUtils::randomize

  - > Randomize server list using local IPv4 address hash as a seed.
    >
    >  使用本地IPv4地址哈希作为种子随机化服务器列表。

##### 5.8.1.4 应用列表的本地缓存

```java
private final AtomicReference<Applications> localRegionApps = new AtomicReference<Applications>();

    localRegionApps.set(new Applications());

    fetchRegistryGeneration = new AtomicLong(0);
```

如果有正在打开着源码的小伙伴们，你们会好奇，这个 `set` 方法的动作跟上面的那一团代码是在一起的，为啥单独把它拆出来呢？其实这个 `Applications` 相对于上面那些组件来讲，能叨叨的话更多，那咱就看看都能叨叨什么吧。

`Applications` 可以理解成一个一个的微服务应用（此处的应用指的是 `spring.application.name` ，一个应用可以有多个实例），微服务实例在初始化后会**定时从注册中心获取已经注册的服务和实例列表**，以备**在注册中心宕机时仍能利用本地缓存向远程服务实例发起通信**。这里的 `Applications` 作用就是缓存这些已经注册的微服务应用和实例。应用和实例列表的同步会在下面看到。

##### 5.8.1.5 区域注册信息

```java
remoteRegionsToFetch = new AtomicReference<String>(clientConfig.fetchRegistryForRemoteRegions());
remoteRegionsRef = 
  new AtomicReference<>(remoteRegionsToFetch.get() == null ? null : remoteRegionsToFetch.get().split(","));
```

这里又看到区域的概念了，它涉及到 Eureka 的**服务分区**特性，咱留到后面聊。

##### 5.8.1.6 初始化远程同步注册表、心跳监控的组件

```java
    if (config.shouldFetchRegistry()) {
        this.registryStalenessMonitor = new ThresholdLevelsMetric(this, 
                METRIC_REGISTRY_PREFIX + "lastUpdateSec_", 
                new long[]{15L, 30L, 60L, 120L, 240L, 480L});
    } else {
        this.registryStalenessMonitor = ThresholdLevelsMetric.NO_OP_METRIC;
    }

    if (config.shouldRegisterWithEureka()) {
        this.heartbeatStalenessMonitor = new ThresholdLevelsMetric(this, 
                METRIC_REGISTRATION_PREFIX + "lastHeartbeatSec_", 
                new long[]{15L, 30L, 60L, 120L, 240L, 480L});
    } else {
        this.heartbeatStalenessMonitor = ThresholdLevelsMetric.NO_OP_METRIC;
    }

```

这两个组件的变量名一看就知道是监控**注册表**和**心跳**的！实际上它们是跟 **Servo** 监控平台关联的，可以实现数据采集，下面咱通过查源码来证明。

注意源码中有一个很特别的数组：`new long[]{15L, 30L, 60L, 120L, 240L, 480L}` ，这个设计看上去很有意思，后一个数都是前一个数的两倍，如果小伙伴们对消息中间件有一些了解，会知道消息的通知速率就像这样越来越慢。那这个时间又是干了些什么呢？咱得深入到 `ThresholdLevelsMetric` 的构造方法中看一下：

```java
public ThresholdLevelsMetric(Object owner, String prefix, long[] levels) {
    this.levels = levels;
    // ......
```

第一句已经告诉我了，它把这组时间赋值到成员上了，而这个成员 `levels` 在下面的 `update` 方法中干了这么一件事：

```java
public void update(long delayMs) {
    long delaySec = delayMs / 1000;
    long matchedIdx;
    if (levels[0] > delaySec) {
        matchedIdx = -1;
    // ......
}
```

这里有使用过这个 `levels` 数组，而这个 `update` 方法只有两个位置有调用过：

可以发现就是上面注册的那两个 **Monitor** 了，以 `heartbeatStalenessMonitor` 为例，跳转过去发现了一个上面咱提到的东西：

```java
@com.netflix.servo.annotations.Monitor(name = METRIC_REGISTRATION_PREFIX + "lastSuccessfulHeartbeatTimePeriod",
        description = "How much time has passed from last successful heartbeat", type = DataSourceType.GAUGE)
private long getLastSuccessfulHeartbeatTimePeriodInternal() {
    // ......
    heartbeatStalenessMonitor.update(computeStalenessMonitorDelay(delay));
    return delay;
}
```

注意 `@Monitor` 的注解所在包：com.netflix.**servo**.annotations ！它果然是跟 **Servo** 监控相关。

##### 5.8.1.7 不注册到注册中心的分支处理

```java
    if (!config.shouldRegisterWithEureka() && !config.shouldFetchRegistry()) {
        logger.info("Client configured to neither register nor query for data.");
        scheduler = null;
        heartbeatExecutor = null;
        cacheRefreshExecutor = null;
        eurekaTransport = null;
        instanceRegionChecker = new InstanceRegionChecker(new PropertyBasedAzToRegionMapper(config), clientConfig.getRegion());

        // This is a bit of hack to allow for existing code using DiscoveryManager.getInstance()
        // to work with DI'd DiscoveryClient
        DiscoveryManager.getInstance().setDiscoveryClient(this);
        DiscoveryManager.getInstance().setEurekaClientConfig(config);

        initTimestampMs = System.currentTimeMillis();
        logger.info("Discovery Client initialized at timestamp {} with initial instances count: {}",
                initTimestampMs, this.getApplications().size());

        return;  // no need to setup up an network tasks and we are done
    }
```

注意上面的分支条件：不注册到 Eureka 、不需要抓取注册信息，这不就是不注册到注册中心嘛。一般情况下这个也不会生效（谁引入了 eureka-client 还不连注册中心呢。。。），咱就稍微看看这里面有哪几个动作就行了。首先上面几个调度器统统没有，下面的 `DiscoveryManager` 把自己设置好，就没了，还是很简单的。

##### 5.8.1.8 初始化定时任务的线程池和定时任务执行器

```java
    try {
        // default size of 2 - 1 each for heartbeat and cacheRefresh
        scheduler = Executors.newScheduledThreadPool(2,
                new ThreadFactoryBuilder()
                        .setNameFormat("DiscoveryClient-%d")
                        .setDaemon(true)
                        .build());

        heartbeatExecutor = new ThreadPoolExecutor(
                1, clientConfig.getHeartbeatExecutorThreadPoolSize(), 0, TimeUnit.SECONDS,
                new SynchronousQueue<Runnable>(),
                new ThreadFactoryBuilder()
                        .setNameFormat("DiscoveryClient-HeartbeatExecutor-%d")
                        .setDaemon(true)
                        .build()
        );  // use direct handoff

        cacheRefreshExecutor = new ThreadPoolExecutor(
                1, clientConfig.getCacheRefreshExecutorThreadPoolSize(), 0, TimeUnit.SECONDS,
                new SynchronousQueue<Runnable>(),
                new ThreadFactoryBuilder()
                        .setNameFormat("DiscoveryClient-CacheRefreshExecutor-%d")
                        .setDaemon(true)
                        .build()
        );  // use direct handoff

```

可以看出来这里面初始化了一个线程池，大小为 **2**，就是为下面两个定时器提供。那咱分别来看这两个定时器吧。

- `heartbeatExecutor` ：心跳定时器，对应的定时任务是 `HeartbeatThread`
- `cacheRefreshExecutor` ：缓存定时器，对应的定时任务是 `CacheRefreshThread`

至于这两个定时任务，在 `DiscoveryClient` 的 `initScheduledTasks` 方法中有初始化：

```java
private void initScheduledTasks() {
    if (clientConfig.shouldFetchRegistry()) {
        // registry cache refresh timer
        int registryFetchIntervalSeconds = clientConfig.getRegistryFetchIntervalSeconds();
        int expBackOffBound = clientConfig.getCacheRefreshExecutorExponentialBackOffBound();
        scheduler.schedule(
                new TimedSupervisorTask(
                        "cacheRefresh",
                        scheduler,
                        cacheRefreshExecutor,
                        registryFetchIntervalSeconds,
                        TimeUnit.SECONDS,
                        expBackOffBound,
                        new CacheRefreshThread()
                ),
                registryFetchIntervalSeconds, TimeUnit.SECONDS);
    }

    if (clientConfig.shouldRegisterWithEureka()) {
        int renewalIntervalInSecs = instanceInfo.getLeaseInfo().getRenewalIntervalInSecs();
        int expBackOffBound = clientConfig.getHeartbeatExecutorExponentialBackOffBound();
        // Heartbeat timer
        scheduler.schedule(
                new TimedSupervisorTask(
                        "heartbeat",
                        scheduler,
                        heartbeatExecutor,
                        renewalIntervalInSecs,
                        TimeUnit.SECONDS,
                        expBackOffBound,
                        new HeartbeatThread()
                ),
                renewalIntervalInSecs, TimeUnit.SECONDS);
```

这里就可以看出来，默认情况下这两个部分都会初始化（除非显式声明禁用 `shouldFetchRegistry` 和 `registration.enabled` ），使用扩展的定时任务 `TimedSupervisorTask` 来实现，至于这个类的结构和构造方法小册不展开描述，读到这里知道定时任务是如此创建即可。

##### 5.8.1.9 初始化通信网关

```java
  eurekaTransport = new EurekaTransport();
    scheduleServerEndpointTask(eurekaTransport, args);
```

这里面初始化了一个 `EurekaTransport` 的组件，这个组件是 `DiscoveryClient` 的内部类：

```java
private static final class EurekaTransport {
    private ClosableResolver bootstrapResolver;
    private TransportClientFactory transportClientFactory;

    private EurekaHttpClient registrationClient;
    private EurekaHttpClientFactory registrationClientFactory;

    private EurekaHttpClient queryClient;
    private EurekaHttpClientFactory queryClientFactory;
```

可以发现它又聚合了好多组件，这些组件都在下面的 `scheduleServerEndpointTask` 方法中被初始化，初始化的方法很长，感兴趣的小伙伴们可以自行翻看。

##### 5.8.1.10 初始化微服务实例区域检查器

```java
AzToRegionMapper azToRegionMapper;
if (clientConfig.shouldUseDnsForFetchingServiceUrls()) {
  azToRegionMapper = new DNSBasedAzToRegionMapper(clientConfig);
} else {
  azToRegionMapper = new PropertyBasedAzToRegionMapper(clientConfig);
}
if (null != remoteRegionsToFetch.get()) {
  azToRegionMapper.setRegionsToFetch(remoteRegionsToFetch.get().split(","));
}
instanceRegionChecker = new InstanceRegionChecker(azToRegionMapper, clientConfig.getRegion());
```

这个 `InstanceRegionChecker` 组件包含一部分关于亚马逊 AWS 的，咱不关心。然而去掉跟 AWS 相关的部分，就没有什么实质性的东西了，换句话说，这个组件不太重要，小册不多深究。

##### 5.8.1.11 拉取注册信息

```java
    if (clientConfig.shouldFetchRegistry() && !fetchRegistry(false)) {
        fetchRegistryFromBackup();
    }
```

这个方法只从方法名上就可以读出意思：从备份中拉取注册表。它的文档注释解释的更清楚：

> Fetch the registry information from back up registry if all eureka server urls are unreachable.
>
> 如果所有 EurekaServer 的URL均无法访问，则从备份注册表中获取注册表信息。

正常情况下，由于 `fetchRegistry` 方法返回 true ，故下面的 `fetchRegistryFromBackup` 方法不会被触发（方法中只有出现异常才返回 false ）。至于 `fetchRegistry` 方法的实现，咱留到第 7 章的 2.1 节来解释（涉及到注册信息的全量获取）。

##### 5.8.1.12 回调注册前置处理器

```java
    if (this.preRegistrationHandler != null) {
        this.preRegistrationHandler.beforeRegistration();
    }
```

这个处理逻辑可能会让咱想起 `BeanPostProcessor` 是吧，大概能回忆起来就好，不过这里跟 `BeanPostProcessor` 并没有关系。。。而且，借助 IDEA ，发现这个 `preRegistrationHandler` 并没有实现类，通过 Debug 也发现它是 null ，这部分小册也不多深究。

##### 5.8.1.13 初始化定时任务

```java
    // finally, init the schedule tasks (e.g. cluster resolvers, heartbeat, instanceInfo replicator, fetch
    initScheduledTasks();
```

这个方法就是前面 5.8.1.8 节提到的初始化动作，不再赘述。

##### 5.8.1.14 向Servo监控注册自己

```java
    try {
        Monitors.registerObject(this);
    } catch (Throwable e) {
        logger.warn("Cannot register timers", e);
    }
```

看到 `Monitors` 自然就想到监控了，前面提到的监控又一直是 Servo，那这里也就理所当然的推测就是向 Servo 注册咯（`Monitors` 所在的包：`com.netflix.servo.monitor`）。

##### 5.8.1.15 注册完成

```java
    DiscoveryManager.getInstance().setDiscoveryClient(this);
    DiscoveryManager.getInstance().setEurekaClientConfig(config);

    initTimestampMs = System.currentTimeMillis();
```

这部分就是很简单的记录了，不再多说。

至此，整个 `CloudEurekaClient` 的初始化及注册动作解析完毕。

#### 5.8.2 ApplicationInfoManager

```java
@Bean
@ConditionalOnMissingBean(value = ApplicationInfoManager.class, search = SearchStrategy.CURRENT)
@org.springframework.cloud.context.config.annotation.RefreshScope
@Lazy
public ApplicationInfoManager eurekaApplicationInfoManager(
        EurekaInstanceConfig config) {
    InstanceInfo instanceInfo = new InstanceInfoFactory().create(config);
    return new ApplicationInfoManager(config, instanceInfo);
}
```

这个 `ApplicationInfoManager` 组件咱刚才在 5.8.1.3 节见过了，这里咱就不再贴文档注释等等东西了，咱留意一下在上面的过程中，`ApplicationInfoManager` 干了什么。

注意在 5.8.1.3 节中，`applicationInfoManager` 的赋值之后还干了一件事，它把其中的 `InstanceInfo` 取出来了。

```java
    this.applicationInfoManager = applicationInfoManager;
    InstanceInfo myInfo = applicationInfoManager.getInfo();
    // ......
    instanceInfo = myInfo;
    if (myInfo != null) {
        appPathIdentifier = instanceInfo.getAppName() + "/" + instanceInfo.getId();
```

它把 `instanceInfo` 的信息取出来之后，拼装成一个 `appPathIdentifier` ，而这个 `appPathIdentifier` 的作用却异常简单：只用来打日志。。。（在 `getApplications` 和 `register` 方法中有利用）除此之外 `InstanceInfo` 的作用就散布在 `DiscoveryClient` 的一些方法中了，咱在这里不列举了，后续遇到的时候再关注。

#### 5.8.3 EurekaRegistration

```java
@Bean
@org.springframework.cloud.context.config.annotation.RefreshScope
@ConditionalOnBean(AutoServiceRegistrationProperties.class)
@ConditionalOnProperty(value = "spring.cloud.service-registry.auto-registration.enabled", matchIfMissing = true)
public EurekaRegistration eurekaRegistration(EurekaClient eurekaClient,
        CloudEurekaInstanceConfig instanceConfig,
        ApplicationInfoManager applicationInfoManager,
        @Autowired(required = false) ObjectProvider<HealthCheckHandler> healthCheckHandler) {
    return EurekaRegistration.builder(instanceConfig).with(applicationInfoManager)
            .with(eurekaClient).with(healthCheckHandler).build();
}
```

上面咱也遇到过这个组件了，它就是 Eureka 实例的服务注册信息，从上面的 `@Conditional` 注解中标注也能看的出来，只有声明 `spring.cloud.service-registry.auto-registration.enabled` 为 false 时才不会创建。它的内部结构如下：

```java
public class EurekaRegistration implements Registration {
	private final EurekaClient eurekaClient;
	private final AtomicReference<CloudEurekaClient> cloudEurekaClient = new AtomicReference<>();
	private final CloudEurekaInstanceConfig instanceConfig;
	private final ApplicationInfoManager applicationInfoManager;
	private ObjectProvider<HealthCheckHandler> healthCheckHandler;
```

可以看得出来它也是一堆组件的整合而已，关于它的作用咱上面也看到了（ 5.5，5.6 节），这里不再赘述。

### 5.9 EurekaHealthIndicatorConfiguration

在 `EurekaClientAutoConfiguration` 的最底下还有一个内部类的配置类，它只注册了一个组件：`EurekaHealthIndicator`

```java
@Bean
@ConditionalOnMissingBean
@ConditionalOnEnabledHealthIndicator("eureka")
public EurekaHealthIndicator eurekaHealthIndicator(EurekaClient eurekaClient,
        EurekaInstanceConfig instanceConfig, EurekaClientConfig clientConfig) {
    return new EurekaHealthIndicator(eurekaClient, instanceConfig, clientConfig);
}
```

这个 `EurekaHealthIndicator` 上面没有文档注释（没有文档注释看着是真的难），它实现的接口 `DiscoveryHealthIndicator` 倒是有一行：

> A health indicator interface specific to a DiscoveryClient implementation.
>
> 特定于 `DiscoveryClient` 实现的运行状况指示器接口。

可以看的出来，它是负责提供应用运行状态的一个指示器，借助 IDEA 翻看这个接口的使用位置，发现在 `DiscoveryCompositeHealthIndicator` 中有个 for 循环的使用：

```java
@Autowired
public DiscoveryCompositeHealthIndicator(HealthAggregator healthAggregator,
        List<DiscoveryHealthIndicator> indicators) {
    super(healthAggregator);
    for (DiscoveryHealthIndicator indicator : indicators) {
        Holder holder = new Holder(indicator);
        addHealthIndicator(indicator.getName(), holder);
        this.healthIndicators.add(holder);
    }
}

```

注意这里面添加进去的集合：`healthIndicators` ，果然它与服务实例运行的健康状态有关。实际上，这个 `EurekaHealthIndicator` 最终起作用是在API请求为 `/health` 的接口上，提供与当前服务实例相关的一些状态信息，这部分咱不作为重点关注，感兴趣的小伙伴可以结合实际的 API 搜查一下即可。

## 小结

1. EurekaClient 的启动核心在 `CloudEurekaClient` 的构造方法中。
2. `EurekaClientAutoConfiguration` 注册的核心组件包括核心服务发现组件 `EurekaClient` 、`DiscoveryClient` 、修改服务状态的 `EurekaServiceRegistry` 、服务注册的契约 `EurekaAutoServiceRegistration` 等，其中服务实例的注册是在 `CloudEurekaClient` 的构造方法中处理完毕。

最后看一眼 EurekaClient 涉及到的所有组件，以及 EurekaClient 的启动流程吧：

![](./img/2023/05/cloud06-3.png)

【两个自动配置部分都了解的差不多了，大概有个印象即可。接下来，咱要深入 Eureka 的一些高级特性来看了，下一章介绍 EurekaServer 的注册表同步机制】