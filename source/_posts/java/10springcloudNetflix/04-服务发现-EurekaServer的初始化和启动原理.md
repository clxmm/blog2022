---
title: 04-服务发现-EurekaServer的初始化和启动原理.md
toc: true
tags: springcloud
categories: 
    - [java]
    - [springcloudNetflix]
---

# 5. EurekaServerAutoConfiguration中配置的核心组件

## 5.1 EurekaController

<!--more-->


```java
	@Bean
	@ConditionalOnProperty(prefix = "eureka.dashboard", name = "enabled", matchIfMissing = true)
	public EurekaController eurekaController() {
		return new EurekaController(this.applicationInfoManager);
	}
```

呦，一看这是个 Controller ，有木有立马想到自己写的那些 Controller ？赶紧点进去瞅一眼：

```java
@Controller
@RequestMapping("${eureka.dashboard.path:/}")
public class EurekaController
```

- `status` - 获取当前 EurekaServer 的状态（即控制台）
- `lastn` - 获取当前 EurekaServer 上服务注册动态历史记录。

这部分咱不展开描述了，有兴趣的小伙伴们可以深入这个类来研究。

## 5.2 PeerAwareInstanceRegistry

```java
@Bean
public PeerAwareInstanceRegistry peerAwareInstanceRegistry(ServerCodecs serverCodecs) {
    this.eurekaClient.getApplications(); // force initialization
    return new InstanceRegistry(this.eurekaServerConfig, this.eurekaClientConfig,
            serverCodecs, this.eurekaClient,
            this.instanceRegistryProperties.getExpectedNumberOfClientsSendingRenews(),
            this.instanceRegistryProperties.getDefaultOpenForTrafficCount());
}
```

这个 `PeerAwareInstanceRegistry` **很重要**，它是 **EurekaServer 集群中节点之间同步微服务实例注册表的核心组件**（这里默认小伙伴已经对 EurekaServer 的集群配置及相关基础都了解了）。集群节点同步注册表的内容咱会另起一章研究，这里咱只是看一下这个类的继承结构，方面后续看到时不至于不认识：

```java
public class InstanceRegistry extends PeerAwareInstanceRegistryImpl implements ApplicationContextAware
public class PeerAwareInstanceRegistryImpl extends AbstractInstanceRegistry implements PeerAwareInstanceRegistry
public abstract class AbstractInstanceRegistry implements InstanceRegistry
```

这里面继承的两个类 **`PeerAwareInstanceRegistryImpl`** 、**`AbstractInstanceRegistry`** ，它们将会在后续研究节点同步时有重要作用，包括里面涉及的功能会在后面的组件（**`EurekaServerContext`** 等）发挥功能时带着一起解释。

## 5.3 PeerEurekaNodes

```java
	@Bean
	@ConditionalOnMissingBean
	public PeerEurekaNodes peerEurekaNodes(PeerAwareInstanceRegistry registry,
			ServerCodecs serverCodecs,
			ReplicationClientAdditionalFilters replicationClientAdditionalFilters) {
		return new RefreshablePeerEurekaNodes(registry, this.eurekaServerConfig,
				this.eurekaClientConfig, serverCodecs, this.applicationInfoManager,
				replicationClientAdditionalFilters);
	}
```

这个 `PeerEurekaNodes` 可以理解成**微服务实例的节点集合**。换言之，一个 `PeerEurekaNode` 就是一个微服务节点实例的包装，`PeerEurekaNodes` 就是这组 `PeerEurekaNode` 的集合，这种节点是可以被 EurekaServer 集群中的各个注册中心节点共享的（`PeerAwareInstanceRegistry`）。翻开 `PeerEurekaNodes` 内部的结构，可以发现有这么几样东西：

```java
public class PeerEurekaNodes {

    protected final PeerAwareInstanceRegistry registry;
    // ......

    private volatile List<PeerEurekaNode> peerEurekaNodes = Collections.emptyList();
    private volatile Set<String> peerEurekaNodeUrls = Collections.emptySet();

    private ScheduledExecutorService taskExecutor;
```

简单解释一下：

- `PeerAwareInstanceRegistry` ：集群间节点同步的核心组件
- `List<PeerEurekaNode>` ：节点集合
- `peerEurekaNodeUrls` ：所有节点所在 url
- `ScheduledExecutorService` ：执行定时任务的线程池

另外 `PeerEurekaNodes` 还提供了一个 `start` 和 `shutdown` 方法：

### 5.3.1 start

```java
public void start() {
    taskExecutor = Executors.newSingleThreadScheduledExecutor(
            new ThreadFactory() {
                @Override
                public Thread newThread(Runnable r) {
                    Thread thread = new Thread(r, "Eureka-PeerNodesUpdater");
                    thread.setDaemon(true);
                    return thread;
                }
            }
    );
    try {
        updatePeerEurekaNodes(resolvePeerUrls());
        Runnable peersUpdateTask = new Runnable() {
            @Override
            public void run() {
                try {
                    updatePeerEurekaNodes(resolvePeerUrls());
                } // catch ......
            }
        };
        taskExecutor.scheduleWithFixedDelay(
                peersUpdateTask,
                serverConfig.getPeerEurekaNodesUpdateIntervalMs(),
                serverConfig.getPeerEurekaNodesUpdateIntervalMs(),
                TimeUnit.MILLISECONDS
        );
    } // catch ...... log ......
}
```

可以发现 start 方法的核心是**借助线程池完成定时任务**。定时任务的内容是中间那一段实现了 `Runnable` 接口的匿名内部类，它会执行一个 `updatePeerEurekaNodes` 方法来**更新集群节点**。下面定时任务的执行时间，借助 IDEA 跳转到 `EurekaServerConfigBean` 中发现默认的配置是 10 分钟，即**每隔10分钟会同步一次集群节点**。至于 `updatePeerEurekaNodes` 的具体实现，咱同样放到后面跟节点同步放在一起来解析。

### 5.3.2 shutdown

```java
public void shutdown() {
    taskExecutor.shutdown();
    List<PeerEurekaNode> toRemove = this.peerEurekaNodes;

    this.peerEurekaNodes = Collections.emptyList();
    this.peerEurekaNodeUrls = Collections.emptySet();

    for (PeerEurekaNode node : toRemove) {
        node.shutDown();
    }
}
```

这个方法的内容比较简单，它会把线程池的定时任务停掉，并移除掉当前所有的服务节点信息。它被调用的时机是下面要解析的 `EurekaServerContext` 。

## 5.4 EurekaServerContext

```java
@Bean
public EurekaServerContext eurekaServerContext(ServerCodecs serverCodecs,
        PeerAwareInstanceRegistry registry, PeerEurekaNodes peerEurekaNodes) {
    return new DefaultEurekaServerContext(this.eurekaServerConfig, serverCodecs,
            registry, peerEurekaNodes, this.applicationInfoManager);
}
```

它创建了一个 `DefaultEurekaServerContext` ，文档注释原文翻译：

> Represent the local server context and exposes getters to components of the local server such as the registry.
>
> 表示本地服务器上下文，并将 getter 方法暴露给本地服务器的组件（例如注册表）。

可以大概的意识到，它确实跟 SpringFramework 的 **`ApplicationContext`** 差不太多哈，可以这么简单地理解吧，咱还是看看里面比较特殊的内容。

进入到 `DefaultEurekaServerContext` 中，果然发现了两个被 JSR250 规范标注的特殊方法：

```java
@PostConstruct
public void initialize() {
    logger.info("Initializing ...");
    peerEurekaNodes.start();
    try {
        registry.init(peerEurekaNodes);
    } catch (Exception e) {
        throw new RuntimeException(e);
    }
    logger.info("Initialized");
}

@PreDestroy
public void shutdown() {
    logger.info("Shutting down ...");
    registry.shutdown();
    peerEurekaNodes.shutdown();
    logger.info("Shut down");
}
```

果然，是 `EurekaServerContext` 的初始化，带动 `PeerEurekaNodes` 的初始化，`EurekaServerContext` 的销毁带动 `PeerEurekaNodes` 的销毁。

加上 `ApplicationContext` 本身的生命周期，可以大概这样理解这个流程：

![](./img/2023/05/cloud04-1.png)

除了带动 `PeerEurekaNodes` 之前，还有一个 `PeerAwareInstanceRegistry` 也带动初始化了，看一眼它的 `init` 方法吧：

### 5.4.1 PeerAwareInstanceRegistry#init

关键部分注释已标注在源码：

```java
public void init(PeerEurekaNodes peerEurekaNodes) throws Exception {
    // 5.4.1.1 启动续订租约的频率统计器
    this.numberOfReplicationsLastMin.start();
    this.peerEurekaNodes = peerEurekaNodes;
    initializedResponseCache();
    // 5.4.1.2 开启续订租约最低阈值检查的定时任务
    scheduleRenewalThresholdUpdateTask();
    // 5.4.1.3 初始化远程分区注册中心
    initRemoteRegionRegistry();

    try {
        Monitors.registerObject(this);
    } catch (Throwable e) {
        logger.warn("Cannot register the JMX monitor for the InstanceRegistry :", e);
    }
}
```

源码标注了三个关键的环节，一一来看：

#### 5.4.1.1 numberOfReplicationsLastMin.start()：启动续订租约的频率统计器

```java
private final AtomicLong lastBucket = new AtomicLong(0);
private final AtomicLong currentBucket = new AtomicLong(0);

private final long sampleInterval;

public synchronized void start() {
    if (!isActive) {
        timer.schedule(new TimerTask() {
            @Override
            public void run() {
                try {
                    // Zero out the current bucket.
                    lastBucket.set(currentBucket.getAndSet(0));
                } catch (Throwable e) {
                    logger.error("Cannot reset the Measured Rate", e);
                }
            }
        }, sampleInterval, sampleInterval);
        isActive = true;
    }
}
```

这个方法实现不难理解，它会隔一段时间重置 `lastBucket` 和 `currentBucket` 的值为 0 ，那时间间隔是多少呢？翻看整个类，发现只有构造方法可以设置时间间隔：

```java
public MeasuredRate(long sampleInterval) {
    this.sampleInterval = sampleInterval;
    this.timer = new Timer("Eureka-MeasureRateTimer", true);
    this.isActive = false;
}
```

借助 IDEA ，发现设置 `sampleInterval` 的值有两处，但值都是一样的：**`new MeasuredRate(1000 \* 60 \* 1)`** ，也就是**1分钟重置一次**。可关键的问题是，它这个操作是干嘛呢？为啥非得一分钟统计一次续约次数呢？实际上，这个计算次数会体现在 Eureka 的控制台，以及配合 **Servo** 完成**续约次数监控**（说白了，咱这看着没啥用，微服务监控和治理还是管用的，不然为什么 Eureka 被称为**服务发现与治理**的框架呢）。

#### 5.4.1.2 scheduleRenewalThresholdUpdateTask：开启续订租约最低阈值检查的定时任务

```java
private int renewalThresholdUpdateIntervalMs = 15 * MINUTES;

private void scheduleRenewalThresholdUpdateTask() {
    timer.schedule(new TimerTask() {
                       @Override
                       public void run() {
                           updateRenewalThreshold();
                       }
                   }, serverConfig.getRenewalThresholdUpdateIntervalMs(),
            serverConfig.getRenewalThresholdUpdateIntervalMs());
}
```

可以发现又是一个定时任务，配置项中的默认时间间隔可以发现是 15 分钟。那定时任务中执行的核心方法是 `updateRenewalThreshold` 方法，跳转过去：

```java
private void updateRenewalThreshold() {
    try {
        Applications apps = eurekaClient.getApplications();
        int count = 0;
        for (Application app : apps.getRegisteredApplications()) {
            for (InstanceInfo instance : app.getInstances()) {
                if (this.isRegisterable(instance)) {
                    ++count;
                }
            }
        }
        synchronized (lock) {
            // Update threshold only if the threshold is greater than the
            // current expected threshold or if self preservation is disabled.
            if ((count) > (serverConfig.getRenewalPercentThreshold() * expectedNumberOfClientsSendingRenews)
                    || (!this.isSelfPreservationModeEnabled())) {
                this.expectedNumberOfClientsSendingRenews = count;
                updateRenewsPerMinThreshold();
            }
        }
        logger.info("Current renewal threshold is : {}", numberOfRenewsPerMinThreshold);
    } // catch ......
}
```

上面的 for 循环很明显是检查当前已经注册到本地的服务实例是否还保持连接，由于该方法一定会返回 true （可翻看该部分实现，全部都是 `return true`），故上面第 4 行统计的 count 就是所有的微服务实例数量。

下面的同步代码块中，它会检查统计好的数量是否比预期的多，如果**统计好的服务实例数比预期的数量多**，证明出现了**新的服务注册**，要替换下一次统计的期望数量值，以及重新计算接下来心跳的数量统计。心跳的数量统计方法 `updateRenewsPerMinThreshold()` ：

```java
private int expectedClientRenewalIntervalSeconds = 30;
private double renewalPercentThreshold = 0.85;

protected void updateRenewsPerMinThreshold() {
    this.numberOfRenewsPerMinThreshold = (int) (this.expectedNumberOfClientsSendingRenews
            * (60.0 / serverConfig.getExpectedClientRenewalIntervalSeconds())
            * serverConfig.getRenewalPercentThreshold());
}
```

可以看出来它的默认计算数是：**每隔 30 秒发一次心跳**（一分钟心跳两次），而且必须所有的服务实例的心跳总数要达到前面计算数量的 **85%** 才算整体微服务正常，其实这也就是 **EurekaServer 的自我保护机制**。

#### 5.4.1.3 initRemoteRegionRegistry：初始化远程分区注册中心

```java
protected void initRemoteRegionRegistry() throws MalformedURLException {
    Map<String, String> remoteRegionUrlsWithName = serverConfig.getRemoteRegionUrlsWithName();
    if (!remoteRegionUrlsWithName.isEmpty()) {
        allKnownRemoteRegions = new String[remoteRegionUrlsWithName.size()];
        int remoteRegionArrayIndex = 0;
        for (Map.Entry<String, String> remoteRegionUrlWithName : remoteRegionUrlsWithName.entrySet()) {
            RemoteRegionRegistry remoteRegionRegistry = new RemoteRegionRegistry(
                    serverConfig,
                    clientConfig,
                    serverCodecs,
                    remoteRegionUrlWithName.getKey(),
                    new URL(remoteRegionUrlWithName.getValue()));
            regionNameVSRemoteRegistry.put(remoteRegionUrlWithName.getKey(), remoteRegionRegistry);
            allKnownRemoteRegions[remoteRegionArrayIndex++] = remoteRegionUrlWithName.getKey();
        }
    }
    logger.info("Finished initializing remote region registries. All known remote regions: {}",
            (Object) allKnownRemoteRegions);
}
```

这里面提到了一个概念：**`RemoteRegionRegistry`** ，它的文档注释原文翻译：

> Handles all registry operations that needs to be done on a eureka service running in an other region. The primary operations include fetching registry information from remote region and fetching delta information on a periodic basis.
>
> 处理在其他区域中运行的 eureka 服务上需要完成的所有注册表操作。主要操作包括从远程区域中获取注册表信息以及定期获取增量信息。

文档注释的解释看着似懂非懂，它没有把这个类的作用完全解释清楚。实际上这里涉及到 Eureka 的**服务分区**，这个咱留到后面解释 Eureka 的高级特性时再聊。

#### 5.4.2 PeerAwareInstanceRegistry#shutdown

当 `EurekaServerContext` 被销毁时，会回调 `@PreDestory` 标注的 `shutdown` 方法，而这个方法又调到 `PeerAwareInstanceRegistry` 的 `shutdown` 方法。

```java
public void shutdown() {
    try {
        DefaultMonitorRegistry.getInstance().unregister(Monitors.newObjectMonitor(this));
    } // catch .......
    try {
        peerEurekaNodes.shutdown();
    } // catch .......
    numberOfReplicationsLastMin.stop();
    super.shutdown();
}
```

这里它干的事情不算麻烦，它首先利用 `DefaultMonitorRegistry` 做了一个注销操作，`DefaultMonitorRegistry` 这个组件本身来源于 **servo** 包，它是做监控使用，那自然能猜出来这部分是**关闭监控**。

接下来它会把那些微服务节点实例全部注销，停止计数器监控，最后回调父类的 `shutdown` 方法：

```java
public void shutdown() {
    deltaRetentionTimer.cancel();
    evictionTimer.cancel();
    renewsLastMin.stop();
}
```

可以发现也是跟监控相关的组件停止，不再赘述。

### 5.5 EurekaServerBootstrap

```java
@Bean
public EurekaServerBootstrap eurekaServerBootstrap(PeerAwareInstanceRegistry registry,
        EurekaServerContext serverContext) {
    return new EurekaServerBootstrap(this.applicationInfoManager,
            this.eurekaClientConfig, this.eurekaServerConfig, registry,
            serverContext);
}
```

这个咱上面已经提过了，有了 `EurekaServerBootstrap` 才能引导启动 `EurekaServer` ，原理在上一章第 4 节提过了，忘记的小伙伴可以翻回去看看哦。

### 5.6 ServletContainer

```java
public static final String DEFAULT_PREFIX = "/eureka";

@Bean
public FilterRegistrationBean jerseyFilterRegistration(javax.ws.rs.core.Application eurekaJerseyApp) {
    FilterRegistrationBean bean = new FilterRegistrationBean();
    bean.setFilter(new ServletContainer(eurekaJerseyApp));
    bean.setOrder(Ordered.LOWEST_PRECEDENCE);
    bean.setUrlPatterns(Collections.singletonList(EurekaConstants.DEFAULT_PREFIX + "/*"));
    return bean;
}
```

它注册的 `FilterRegistrationBean` 我在之前的 boot 小册中有提过（第 6 章 4.1.2 节），这里咱直接说核心的 `Filter` 是 `ServletContainer` ：

```java
package com.sun.jersey.spi.container.servlet;

public class ServletContainer extends HttpServlet implements Filter
```

注意它所在的包，里面有一个很关键的词：**jersey** ，它是一个类似于 SpringWebMvc 的框架，由于 Eureka 本身也是一个 Servlet 应用，只是它使用的 Web 层框架不是 SpringWebMvc 而是 Jersey 而已，Jersey 在 Eureka 的远程请求、心跳包发送等环节起到至关重要的作用，而这个 `ServletContainer` 的作用，可以理解为 SpringWebMvc 中的 **`DispatcherServlet`** ，以及 Struts2 中的 **`StrutsPrepareAndExecuteFilter`** 。

另外注意一个点，这里面有一个 **`DEFAULT_PREFIX`** ，翻过去发现前缀是 **`/eureka`** ，这也解释了为什么**微服务注册到 EurekaServer 的时候，`defaultZone` 要在 `ip:port` 后面加上 `/eureka`** ，以及后面在**调用 EurekaServer 的一些接口时前面也要加上 `/eureka`** 。

### 5.7 Application

```java
@Bean
public javax.ws.rs.core.Application jerseyApplication(Environment environment,
        ResourceLoader resourceLoader) {
    // ......
}
```

这个类的创建咱不是很关心，瞅一眼这个类的子类，发现全部都是来自 **Jersey** 的：

而且上面的 `ServletContainer` 中正好也用到了这个 `Application` ，那大概也明白它是配合上面的过滤器使用，后续咱会跟上面的 **Jersey** 一起解释。

### 5.8 HttpTraceFilter

```java
	@Bean
	public FilterRegistrationBean traceFilterRegistration(
			@Qualifier("httpTraceFilter") Filter filter) {
		FilterRegistrationBean bean = new FilterRegistrationBean();
		bean.setFilter(filter);
		bean.setOrder(Ordered.LOWEST_PRECEDENCE - 10);
		return bean;
	}
```

它注册了一个名为 `httpTraceFilter` 的过滤器，借助IDEA发现这个过滤器来自 `HttpTraceAutoConfiguration` 的内部类 `ServletTraceFilterConfiguration` ：

```java
@Configuration
@ConditionalOnWebApplication(type = Type.SERVLET)
static class ServletTraceFilterConfiguration {
    @Bean
    @ConditionalOnMissingBean
    public HttpTraceFilter httpTraceFilter(HttpTraceRepository repository, HttpExchangeTracer tracer) {
        return new HttpTraceFilter(repository, tracer);
    }
}
```

这个过滤器的作用也很容易猜想，**trace** 的概念咱从日志系统里也接触过，它打印的内容非常非常多，且涵盖了上面的几乎所有级别。这个类的文档注释也恰好印证了我们的猜想：

> Servlet Filter that logs all requests to an HttpTraceRepository.
>
> 记录所有请求日志的Servlet过滤器。

## 6. EurekaServerConfigBeanConfiguration

EurekaServerAutoConfiguration` 还有一个内部的配置类：`EurekaServerConfigBeanConfiguration

```java
@Configuration
protected static class EurekaServerConfigBeanConfiguration {
    @Bean
    @ConditionalOnMissingBean
    public EurekaServerConfig eurekaServerConfig(EurekaClientConfig clientConfig) {
        EurekaServerConfigBean server = new EurekaServerConfigBean();
        if (clientConfig.shouldRegisterWithEureka()) {
            // Set a sensible default if we are supposed to replicate
            server.setRegistrySyncRetries(5);
        }
        return server;
    }
}
```

它就是注册了默认的 EurekaServer 的配置模型 `EurekaServerConfigBean` ，这个模型类里的配置咱上面也看到一些了，后面的部分咱还会接触它，先有一个印象即可。

## 小结

回顾一下 EurekaServer 中提到的组件吧，挺多的，但关键的就那么几个，先混个脸熟就好：

![](./img/2023/05/cloud04-2.png)

【至此，EurekaServer 的部分也就看的差不多了，下面的两章咱开始了解 EurekaClient 的初始化以及它的启动原理】

