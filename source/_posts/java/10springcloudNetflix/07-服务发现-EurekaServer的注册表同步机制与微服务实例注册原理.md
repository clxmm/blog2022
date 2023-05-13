---
title: 07-服务发现-EurekaServer的注册表同步机制与微服务实例注册原理
toc: true
tags: springcloud
categories: 
    - [java]
    - [springcloudNetflix]
---

在第 3 章中，咱看到了 EurekaServer 在初始化时，要初始化 EurekaServer 的运行上下文，这里面有很重要的一步是同步集群节点注册表，EurekaServer 也因此可以做成高可用 Server 集群。（这里默认小伙伴已经对 EurekaServer 的集群配置及相关基础都了解了，如果小伙伴还不了解 Eureka 集群的配置以及相关的基础，建议小伙伴先去学习并实际操作一下）

下面的方法是初始化 EurekaServer 的运行上下文方法，这个方法在前面（第3章4.2节）也提到过了：

<!--more-->


```java
protected void initEurekaServerContext() throws Exception {
    // ......
    // Copy registry from neighboring eureka node
    // Eureka复制集群节点注册表
    int registryCount = this.registry.syncUp();
    this.registry.openForTraffic(this.applicationInfoManager, registryCount);
    // ......
}
```

下面展开 `this.registry.syncUp();` 来详细分析 EurekaServer 在初始启动时同步注册表的动作。

## 1. EurekaServer初始启动时同步集群节点注册表

跳转到 `PeerAwareInstanceRegistryImpl` 的 `syncUp` 方法：

```java
public int syncUp() {
    // Copy entire entry from neighboring DS node
    int count = 0;

    for (int i = 0; ((i < serverConfig.getRegistrySyncRetries()) && (count == 0)); i++) {
        // 1.1 重试休眠
        if (i > 0) {
            try {
                Thread.sleep(serverConfig.getRegistrySyncRetryWaitMs());
            } // catch .....
        }
        // 1.2 获取注册实例
        Applications apps = eurekaClient.getApplications();
        for (Application app : apps.getRegisteredApplications()) {
            for (InstanceInfo instance : app.getInstances()) {
                try {
                    if (isRegisterable(instance)) {
                        // 1.3 【微服务实例注册】注册服务到本EurekaServer实例
                        register(instance, instance.getLeaseInfo().getDurationInSecs(), true);
                        count++;
                    }
                } // catch ......
            }
        }
    }
    return count;
}
```

先暂且不看 for 循环的循环体，先研究一下它的判断条件：`(i < serverConfig.getRegistrySyncRetries()) && (count == 0)` ，这里它又是什么设计呢？咱看它提到一个 **registrySyncRetries** 的概念：**同步节点重试次数** ，说明 **Eureka 在节点之间的注册表同步时引入了重试机制**！只要同步失败，且在重试次数之内，它就会一直去尝试同步注册表。默认情况下，重试次数被写死在源码中，它会重试5次。

```java
public int getRegistrySyncRetries() {
    return configInstance.getIntProperty(namespace + "numberRegistrySyncRetries", 5).get();
}
```

好了，看循环体。这个循环体咱拆成几部分来看：

### 1.1 重试的休眠机制

```java
for (int i = 0; ((i < serverConfig.getRegistrySyncRetries()) && (count == 0)); i++) {
    if (i > 0) {
        try {
            Thread.sleep(serverConfig.getRegistrySyncRetryWaitMs());
        } // catch .....
```

一开始，如果 i > 0 ，证明已经走完一次循环，它会让线程休眠30秒，这个30秒也在上面的配置中：

```java
public long getRegistrySyncRetryWaitMs() {
    return configInstance.getIntProperty(namespace + "registrySyncRetryWaitMs", 30 * 1000).get();
}
```

不难猜想，它休眠30秒是为了避免因为突然出现的网络波动导致注册表复制失败，实际在开发场景中可以调低这个数值。

### 1.2 获取注册实例

```java
    Applications apps = eurekaClient.getApplications();
    for (Application app : apps.getRegisteredApplications()) {
        for (InstanceInfo instance : app.getInstances()) {
```

这部分操作很明显是要获取集群中注册中心了，它要借助一个 `eurekaClient` ，不难猜测它的核心就是在服务注册时咱了解的 **`EurekaClient`** 概念，它的存在就是为了集群注册。通过Debug，发现它是一个被 jdk 动态代理的代理对象，源接口就是 `EurekaClient` ：

这里面获取到的 `Applications` 从类名上也可以猜测出它是一组应用，那在一个微服务网络中，`Applications` 就应该是这个网络中的所有微服务咯？通过实际搭建 **EurekaServer** 集群，以及注册 **EurekaClient** ，Debug后可以看得出来，它可以获取到微服务网络中所有的微服务以及对应的实例：

![](./img/2023/05/cloud06-1.png)

这里面获取到的 `Applications` 从类名上也可以猜测出它是一组应用，那在一个微服务网络中，`Applications` 就应该是这个网络中的所有微服务咯？通过实际搭建 **EurekaServer** 集群，以及注册 **EurekaClient** ，Debug后可以看得出来，它可以获取到微服务网络中所有的微服务以及对应的实例：

![](./img/20203/05/cloud07-2.png)

正好根据上面的循环也可以理解，它的结构应该是：**`微服务 -> 实例`** 。循环获取所有的微服务，再获取微服务下面对应的实例，都注册到本地的微服务注册表中。

### 1.3 【微服务实例注册】注册实例动作

```java
    try {
        if (isRegisterable(instance)) {
            register(instance, instance.getLeaseInfo().getDurationInSecs(), true);
            count++;
        }
    } // catch ......
```

这里面核心的方法就这一个 **`register`** 方法，这个方法也是正常情况下 EurekaClient 注册到 EurekaServer 必经的方法。

跳转过去：（源码非常长，关键注释已标注在源码中，先扫一眼得了）

```java
public void register(InstanceInfo registrant, int leaseDuration, boolean isReplication) {
    try {
        read.lock(); // 开启读锁
        Map<String, Lease<InstanceInfo>> gMap = registry.get(registrant.getAppName());
        // REGISTER的类型是EurekaMonitors，它与微服务监控相关，这里是增加注册计数
        REGISTER.increment(isReplication);
        // 给当前微服务实例添加租约信息
        if (gMap == null) {
            final ConcurrentHashMap<String, Lease<InstanceInfo>> gNewMap = new ConcurrentHashMap<>();
            gMap = registry.putIfAbsent(registrant.getAppName(), gNewMap);
            if (gMap == null) {
                gMap = gNewMap;
            }
        }
        Lease<InstanceInfo> existingLease = gMap.get(registrant.getId());
        // Retain the last dirty timestamp without overwriting it, if there is already a lease
        // 当前微服务网络中已存在该实例，保留最后一次续约的时间戳
        if (existingLease != null && (existingLease.getHolder() != null)) {
            Long existingLastDirtyTimestamp = existingLease.getHolder().getLastDirtyTimestamp();
            Long registrationLastDirtyTimestamp = registrant.getLastDirtyTimestamp();
            logger.debug("Existing lease found (existing={}, provided={}", existingLastDirtyTimestamp, registrationLastDirtyTimestamp);

            // this is a > instead of a >= because if the timestamps are equal, we still take the remote transmitted
            // InstanceInfo instead of the server local copy.
            // 如果EurekaServer端已存在当前微服务实例，且时间戳大于新传入微服务实例的时间戳，则用EurekaServer端的数据替换新传入的微服务实例
            // 这个机制是防止已经过期的微服务实例注册到EurekaServer
            if (existingLastDirtyTimestamp > registrationLastDirtyTimestamp) {
                logger.warn("There is an existing lease and the existing lease's dirty timestamp {} is greater" +
                            " than the one that is being registered {}", existingLastDirtyTimestamp, registrationLastDirtyTimestamp);
                logger.warn("Using the existing instanceInfo instead of the new instanceInfo as the registrant");
                registrant = existingLease.getHolder();
            }
        } else {
            // The lease does not exist and hence it is a new registration
            // 租约不存在因此它是一个新的待注册实例
            synchronized (lock) {
                if (this.expectedNumberOfClientsSendingRenews > 0) {
                    // Since the client wants to register it, increase the number of clients sending renews
                    this.expectedNumberOfClientsSendingRenews = this.expectedNumberOfClientsSendingRenews + 1;
                    updateRenewsPerMinThreshold();
                }
            }
            logger.debug("No previous lease information found; it is new registration");
        }
        // 创建微服务实例与EurekaServer的租约
        Lease<InstanceInfo> lease = new Lease<InstanceInfo>(registrant, leaseDuration);
        if (existingLease != null) {
            lease.setServiceUpTimestamp(existingLease.getServiceUpTimestamp());
        }
        // 缓存租约，并记录到“最近注册”
        gMap.put(registrant.getId(), lease);
        synchronized (recentRegisteredQueue) {
            recentRegisteredQueue.add(new Pair<Long, String>(
                System.currentTimeMillis(),
                registrant.getAppName() + "(" + registrant.getId() + ")"));
        }
        // This is where the initial state transfer of overridden status happens
        // 如果当前微服务实例的状态不是“UNKNOWN”，则说明之前已经存在一个状态，此处需要做一次状态覆盖
        if (!InstanceStatus.UNKNOWN.equals(registrant.getOverriddenStatus())) {
            logger.debug("Found overridden status {} for instance {}. Checking to see if needs to be add to the "
                         + "overrides", registrant.getOverriddenStatus(), registrant.getId());
            if (!overriddenInstanceStatusMap.containsKey(registrant.getId())) {
                logger.info("Not found overridden id {} and hence adding it", registrant.getId());
                overriddenInstanceStatusMap.put(registrant.getId(), registrant.getOverriddenStatus());
            }
        }
        InstanceStatus overriddenStatusFromMap = overriddenInstanceStatusMap.get(registrant.getId());
        if (overriddenStatusFromMap != null) {
            logger.info("Storing overridden status {} from map", overriddenStatusFromMap);
            registrant.setOverriddenStatus(overriddenStatusFromMap);
        }

        // Set the status based on the overridden status rules
        // 经过上面的处理后，这里获取到微服务实例的最终状态，并真正地设置进去
        InstanceStatus overriddenInstanceStatus = getOverriddenInstanceStatus(registrant, existingLease, isReplication);
        registrant.setStatusWithoutDirty(overriddenInstanceStatus);

        // If the lease is registered with UP status, set lease service up timestamp
        // 记录微服务实例注册时注册租约的时间
        if (InstanceStatus.UP.equals(registrant.getStatus())) {
            lease.serviceUp();
        }
        // 处理状态、续约时间戳等
        registrant.setActionType(ActionType.ADDED);
        recentlyChangedQueue.add(new RecentlyChangedItem(lease));
        registrant.setLastUpdatedTimestamp();
        invalidateCache(registrant.getAppName(), registrant.getVIPAddress(), registrant.getSecureVipAddress());
        logger.info("Registered instance {}/{} with status {} (replication={})",
                    registrant.getAppName(), registrant.getId(), registrant.getStatus(), isReplication);
    } finally {
        read.unlock(); // 释放读锁
    }
}

```

源码很长，咱慢慢地从中抽取出核心注册逻辑，看看它的真正动作：（下面的源码已剔除与主干核心逻辑无关的代码）

#### 1.3.1 处理微服务实例

```java
    // 给当前微服务实例添加租约信息
    if (gMap == null) {
        final ConcurrentHashMap<String, Lease<InstanceInfo>> gNewMap = new ConcurrentHashMap<>();
        gMap = registry.putIfAbsent(registrant.getAppName(), gNewMap);
        if (gMap == null) {
            gMap = gNewMap;
        }
    }
```

这部分它考虑到可能出现的并发问题，使用了双重检查来保证将要注册的微服务实例能唯一的缓存到 `gMap` 变量上。

```java
Lease<InstanceInfo> existingLease = gMap.get(registrant.getId());
    // 当前微服务网络中已存在该实例，保留最后一次续约的时间戳
    if (existingLease != null && (existingLease.getHolder() != null)) {
        Long existingLastDirtyTimestamp = existingLease.getHolder().getLastDirtyTimestamp();
        Long registrationLastDirtyTimestamp = registrant.getLastDirtyTimestamp();
        // 如果EurekaServer端已存在当前微服务实例，且时间戳大于新传入微服务实例的时间戳，则用EurekaServer端的数据替换新传入的微服务实例
        // 这个机制是防止已经过期的微服务实例注册到EurekaServer
        if (existingLastDirtyTimestamp > registrationLastDirtyTimestamp) {
            registrant = existingLease.getHolder();
        }
    } else {
        // 租约不存在因此它是一个新的待注册实例
        synchronized (lock) {
            if (this.expectedNumberOfClientsSendingRenews > 0) {
                // 增加接下来要接收租约续订的客户端数量
                this.expectedNumberOfClientsSendingRenews = this.expectedNumberOfClientsSendingRenews + 1;
                updateRenewsPerMinThreshold();
            }
        }
    }
```

先看上面部分的逻辑：如果这个实例是之前在 EurekaServer 中有记录的，这种情况下的注册会进行一个额外的逻辑：它要**判断当前注册的实例是否是一个已经过期的实例**，方式是**通过拿 EurekaServer 中记录的心跳时间戳，与当前正在注册的微服务实例当初创建时的时间戳进行比对**。

注意微服务实例要注册到 EurekaServer 时，在 EurekaServer 中记录的实例信息模型类 `InstanceInfo` 的构造方法：

```java
private InstanceInfo() {
    this.metadata = new ConcurrentHashMap<String, String>();
    this.lastUpdatedTimestamp = System.currentTimeMillis();
    this.lastDirtyTimestamp = lastUpdatedTimestamp;
}
```

很明显是取**系统当前时间**。（借助IDEA发现下面的 `public` 类型的构造方法没有被引用过，只有这个 `private` 的方法，配合建造器创建的 `InstanceInfo` 对象）

由此不难理解：如果在微服务实例模型创建之后、注册之前，有一个**相同的实例**给 EurekaServer **发送了心跳包**，则 EurekaServer 会认为这次注册是一次**过期注册**，会**使用** EurekaServer **本身已经缓存的** `InstanceInfo` **代替**传入的对象。

下面的 else 部分相对比较简单，全新的实例注册进来只需要增加租约续订的计数即可，这里面涉及到 EurekaServer 的**自我保护机制**。

#### 1.3.2 【特性】EurekaServer的自我保护机制

EurekaServer 中有两个很重要的参数，它们来共同控制和检测 EurekaServer 的状态，以决定在特定的时机下触发自我保护。

```java
protected volatile int numberOfRenewsPerMinThreshold; // 每分钟能接收的最少续租次数
protected volatile int expectedNumberOfClientsSendingRenews; // 期望收到续租心跳的客户端数量
```

注意在 SpringCloudNetflix 版本为 **`Greenwich.SR4`** 时，Eureka 的版本是 `1.9.13` ，在 Eureka 1.8 版本中这部分的两个参数为：

```java
protected volatile int numberOfRenewsPerMinThreshold; // 每分钟能接收的最少续租次数
protected volatile int expectedNumberOfRenewsPerMin; // 期望的每分钟最大续租次数
```

这部分的一个小小改动，可能是由于之前 Eureka **写死了一分钟心跳两次**，换用 1.9 版本的这种方式可以自定义心跳频率。

注意上面的源码中，else 结构里面最后一句话调用了一个 `updateRenewsPerMinThreshold` 方法：

```java
private int expectedClientRenewalIntervalSeconds = 30;
private double renewalPercentThreshold = 0.85;

protected void updateRenewsPerMinThreshold() {
    this.numberOfRenewsPerMinThreshold = (int) (this.expectedNumberOfClientsSendingRenews
            * (60.0 / serverConfig.getExpectedClientRenewalIntervalSeconds())
            * serverConfig.getRenewalPercentThreshold());
}
```

这个计算公式有如下解释：

**每分钟能接收的最少续租次数 = 微服务实例总数 \* ( 60秒 / 实例续约时间间隔 ) \* 有效心跳比率**

举一个栗子：当前 EurekaServer 中有注册了 5 个服务实例，那么在默认情况下，每分钟能接收的最少续租次数就应该是 5 * (60 / 30) * 0.85 = 8.5次，强转为 int 类型 → 8次。那就意味着，如果在这个检测周期中，如果 EurekaServer 收到的有效心跳包少于8个，且没有在配置中显式关闭自我保护，则 EurekaServer 会开启自我保护模式，暂停服务剔除。

#### 1.3.3 创建租约

```java
    // 创建微服务实例与EurekaServer的租约
    Lease<InstanceInfo> lease = new Lease<InstanceInfo>(registrant, leaseDuration);
    if (existingLease != null) {
        lease.setServiceUpTimestamp(existingLease.getServiceUpTimestamp());
    }
    gMap.put(registrant.getId(), lease);
```

这里它会创建一个新的租约，注意租约中的结构：

```java
public Lease(T r, int durationInSecs) {
    holder = r;
    registrationTimestamp = System.currentTimeMillis(); // 租约开始时间
    lastUpdateTimestamp = registrationTimestamp; // 初始情况开始时间即为最后一次的时间
    duration = (durationInSecs * 1000);
}
```

这里面 `durationInSecs` 的值往前找，发现是在 `sync` 方法中调用 `instance.getLeaseInfo().getDurationInSecs()` 取到的，而 LeaseInfo 中最终寻找到它的默认值：

```java
public class LeaseInfo {
    public static final int DEFAULT_LEASE_RENEWAL_INTERVAL = 30;
    public static final int DEFAULT_LEASE_DURATION = 90;

    private int renewalIntervalInSecs = DEFAULT_LEASE_RENEWAL_INTERVAL;
    private int durationInSecs = DEFAULT_LEASE_DURATION;
```

可以发现是 **90** 秒，这也解释了 **EurekaClient 实例的默认服务过期时间是 90 秒**。

#### 1.3.4 记录最近注册的记录

```java
// 记录到“最近注册”
synchronized (recentRegisteredQueue) {
  recentRegisteredQueue.add(new Pair<Long, String>(
    System.currentTimeMillis(), registrant.getAppName() + "(" + registrant.getId() + ")"));
}
```

这个 `recentRegisteredQueue` 的实际目的是在 EurekaServer 的控制台上，展示最近的微服务实例注册记录。值得注意的，它使用了队列作为存放的容器，这里面有一个小小的讲究：队列的**先进先出**特点决定了刚好可以作为历史记录的存放容器。另外一个注意的点，最近的记录往往都只会展示有限的几个，所以这里的 Queue 并不是 jdk 原生自带的队列，而是扩展的一个 `CircularQueue` ：

```java
private final CircularQueue<Pair<Long, String>> recentRegisteredQueue;

```

它的继承和关键定义如下：

```java
private class CircularQueue<E> extends ConcurrentLinkedQueue<E> {
    private int size = 0;
    
    public CircularQueue(int size) {
        this.size = size;
    }
    
    @Override
    public boolean add(E e) {
        this.makeSpaceIfNotAvailable();
        return super.add(e);
    }

    private void makeSpaceIfNotAvailable() {
        if (this.size() == size) {
            this.remove();
        }
    }
```

从这里面咱可以看到一个很微妙的设计：当队列初始化时，会指定一个队列**最大容量**，当队列中的元素数量达到预先制定的最大容量时，会将最先进入队列的元素剔除掉，以达到**队列中的元素都是最近刚添加的**。

#### 1.3.5 微服务实例的状态覆盖

关于状态的覆盖，这里要先了解一下 Eureka 的服务状态覆盖机制。

##### 1.3.5.1 【扩展】服务状态覆盖机制

在 EurekaServer 中，每个服务都有两种状态：

- **服务自身的状态（status）**，这是每个服务**自身的动态属性**；

- **EurekaServer 自身记录的服务的覆盖状态（overriddenStatus）**，这个状态维护在 EurekaServer 中，用来标注 **EurekaServer 中注册的服务实例的状态**

  - 它的作用是在**被记录的服务**在进行注册、续约等动作时，以这个**覆盖的状态**为准，而不是服务本身的状态。

   

  这个实现机制就是上面源码中马上要贴出来的这部分：

  ```java
      // This is where the initial state transfer of overridden status happens
      // 如果当前微服务实例的状态不是“UNKNOWN”，则说明之前已经存在一个状态，此处需要做一次状态覆盖
      if (!InstanceStatus.UNKNOWN.equals(registrant.getOverriddenStatus())) {
          if (!overriddenInstanceStatusMap.containsKey(registrant.getId())) {
              overriddenInstanceStatusMap.put(registrant.getId(), registrant.getOverriddenStatus());
          }
      }
      InstanceStatus overriddenStatusFromMap = overriddenInstanceStatusMap.get(registrant.getId());
      if (overriddenStatusFromMap != null) {
          registrant.setOverriddenStatus(overriddenStatusFromMap);
      }
  ```

源码的逻辑是不是跟我上面概括的思路一样呢？

下面咱趁热打铁，看看服务状态的覆盖是如何实现的，以及它的实际用途吧。

##### 1.3.5.2 服务状态覆盖的类型和操作接口

EurekaServer 中记录微服务实例的状态有五种，它定义在一个枚举中：

```java
public enum InstanceStatus {
    UP, // Ready to receive traffic
    DOWN, // Do not send traffic- healthcheck callback failed
    STARTING, // Just about starting- initializations to be done - do not send traffic
    OUT_OF_SERVICE, // Intentionally shutdown for traffic
    UNKNOWN;
```

默认情况下 EurekaServer 中不会记录覆盖状态，通过Debug可以发现当微服务实例发送心跳请求时，微服务节点实例的状态是 **`UP`** ，EurekaServer 自身没有记录覆盖状态：

如果要指定某个微服务实例在 EurekaServer 中记录的覆盖状态，则可以调用一个接口：在 EurekaServer 中，`InstanceResource` 是用来控制微服务实例状态的 Restful WebService 接口，它基于 **Jersey** 构建，在这里面它定义了一个 `statusUpdate` 方法，用于更新微服务实例在 EurekaServer 中存放的状态：

```java
@PUT
@Path("status")
public Response statusUpdate(
        @QueryParam("value") String newStatus,
        @HeaderParam(PeerEurekaNode.HEADER_REPLICATION) String isReplication,
        @QueryParam("lastDirtyTimestamp") String lastDirtyTimestamp)
```

这个接口的实际调用路径是需要按照指定格式发送的：

使用 postman 发送请求后，收到 200 的响应，之后再回到 EurekaServer 的控制台，发现 EurekaClient 的状态已经改变：

由此可以完成对微服务实例的状态覆盖。

#### 1.3.6 决定微服务实例的真正状态

```java
// Set the status based on the overridden status rules
// 经过上面的处理后，这里获取到微服务实例的最终状态，并真正地设置进去
InstanceStatus overriddenInstanceStatus = getOverriddenInstanceStatus(registrant, existingLease, isReplication);
registrant.setStatusWithoutDirty(overriddenInstanceStatus);
```

在获取真正的微服务实例状态时，它又调了 `getOverriddenInstanceStatus` 方法：

```java
protected InstanceInfo.InstanceStatus getOverriddenInstanceStatus(InstanceInfo r,
        Lease<InstanceInfo> existingLease, boolean isReplication) {
    InstanceStatusOverrideRule rule = getInstanceInfoOverrideRule();
    logger.debug("Processing override status using rule: {}", rule);
    return rule.apply(r, existingLease, isReplication).status();
}
```

这个方法第一行获取 `InstanceInfoOverrideRule` 的方法是一个抽象方法，在子类 `PeerAwareInstanceRegistryImpl` 中有定义：

```java
protected InstanceStatusOverrideRule getInstanceInfoOverrideRule() {
    return this.instanceStatusOverrideRule;
}
```

可以看到就是返回自身的 `instanceStatusOverrideRule` 而已。通过搭建一个 EurekaServer 和一个 EurekaClient ，Debug发现 `InstanceStatusOverrideRule` 的类型为 `FirstMatchWinsCompositeRule` ，那进入它的 `apply` 方法：

```java
public StatusOverrideResult apply(InstanceInfo instanceInfo,
                                  Lease<InstanceInfo> existingLease,
                                  boolean isReplication) {
    for (int i = 0; i < this.rules.length; ++i) {
        StatusOverrideResult result = this.rules[i].apply(instanceInfo, existingLease, isReplication);
        if (result.matches()) {
            return result;
        }
    }
    return defaultRule.apply(instanceInfo, existingLease, isReplication);
}
```

这里它会先循环一些预设好的规则，如果都不匹配，则会使用默认规则。通过Debug发现有三种规则和一个默认规则：

![](./img/2023/05/cloud07-3.png)

Debug走下去发现上面的三种匹配规则都没起作用，那就来到默认规则：

```java
public StatusOverrideResult apply(InstanceInfo instanceInfo,
        Lease<InstanceInfo> existingLease, boolean isReplication) {
    logger.debug("Returning the default instance status {} for instance {}", instanceInfo.getStatus(),
                 instanceInfo.getId());
    return StatusOverrideResult.matchingStatus(instanceInfo.getStatus());
}
```

这个 `StatusOverrideResult.matchingStatus` 方法的内容也很简单，它只是将 `InstanceStatus` 封装到 `StatusOverrideResult` 中而已：

```
public static StatusOverrideResult matchingStatus(InstanceInfo.InstanceStatus status) {
    return new StatusOverrideResult(true, status);
}
```

它的这个包装实际上是为了迎合前面的 `rule.apply(r, existingLease, isReplication).status();` ，实际上在 EurekaServer 没有内部覆盖指定时，最终这样匹配下来，就是拿的 EurekaClient 本身的状态，Debug走下来也可以发现确实就是一开始的 UP 状态：

 ![](./img/2023/05/cloud07-4.png)

顺便多提一嘴吧，上面看到的那几个匹配规则：

- `DownOrStartingRule` ：匹配实例状态为 `DOWN` 或者 `STARTING`
- `OverrideExistsRule` ：匹配被 EurekaServer 本身覆盖过的实例状态
- `LeaseExistsRule` ：匹配 Eureka Server 本地存放的微服务实例状态是否等于 `UP` 或 `OUT_OF_SERVICE`
- `AlwaysMatchInstanceStatusRule` ：直接通过，不进行任何匹配

#### 1.3.7 记录租约、时间戳等

```java
 // If the lease is registered with UP status, set lease service up timestamp
    // 记录微服务实例注册时注册租约的时间
    if (InstanceStatus.UP.equals(registrant.getStatus())) {
        lease.serviceUp();
    }
    // 处理状态、续约时间戳等
    registrant.setActionType(ActionType.ADDED);
    recentlyChangedQueue.add(new RecentlyChangedItem(lease));
    registrant.setLastUpdatedTimestamp();
    // 让当前注册的微服务实例缓存失效，后续的处理中会重新构建缓存
    invalidateCache(registrant.getAppName(), registrant.getVIPAddress(), registrant.getSecureVipAddress());
```

上面的 if 结构中调了一个 `serviceUp` 方法，点进去可以发现就是记录时间戳：

```java
public void serviceUp() {
    if (serviceUpTimestamp == 0) {
        serviceUpTimestamp = System.currentTimeMillis();
    }
}
```

下面的 `recentlyChangedQueue` 是一个比较关键的点，它涉及到 **EurekaClient 的注册信息的获取机制**。除了这个点之外，微服务实例的注册动作就读完了，注册表的同步机制也就读完了。至于 EurekaClient 的注册信息获取机制，咱放到下一章解释（这部分有点多且容易混乱）。

上面都是关于第一次的同步，但 EurekaServer 的集群节点之间还有一个定时更新的设计，咱下面来看这部分。

## 2. PeerEurekaNodes#updatePeerEurekaNodes：10分钟更新一次集群节点注册信息

在 `PeerEurekaNodes` 的初始化时，会被调用一个 `start` 方法，在这里会开启集群节点注册信息的更新定时任务。注意这里面在开启定时任务之前先同步了一次（注意 try 中的第一句）。

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
        // 先执行一次
        updatePeerEurekaNodes(resolvePeerUrls());
        Runnable peersUpdateTask = new Runnable() {
            @Override
            public void run() {
                try {
                    updatePeerEurekaNodes(resolvePeerUrls());
                } // catch ......
            }
        };
        // 再开始调度定时任务
        taskExecutor.scheduleWithFixedDelay(
                peersUpdateTask,
                serverConfig.getPeerEurekaNodesUpdateIntervalMs(),
                serverConfig.getPeerEurekaNodesUpdateIntervalMs(),
                TimeUnit.MILLISECONDS
        );
    } // catch ......
}

```

那咱就来看核心的 `updatePeerEurekaNodes(resolvePeerUrls());` 方法。

### 2.1 resolvePeerUrls

```java
protected List<String> resolvePeerUrls() {
    // 获取EurekaServer集群的所有地址
    InstanceInfo myInfo = applicationInfoManager.getInfo();
    String zone = InstanceInfo.getZone(clientConfig.getAvailabilityZones(clientConfig.getRegion()), myInfo);
    List<String> replicaUrls = EndpointUtils
            .getDiscoveryServiceUrls(clientConfig, zone, new EndpointUtils.InstanceInfoBasedUrlRandomizer(myInfo));

    // 去掉自己
    int idx = 0;
    while (idx < replicaUrls.size()) {
        if (isThisMyUrl(replicaUrls.get(idx))) {
            replicaUrls.remove(idx);
        } else {
            idx++;
        }
    }
    return replicaUrls;
}
```

可以发现这段逻辑是很简单的，它会取出所有的注册中心（即 EurekaServer 集群）的 url ，之后去掉自己，返回出去。通过搭建双注册中心的集群，且互相配置对方为注册中心，Debug发现确实只拿到了在 yml 中配置的 eureka-server1 的地址：

### 2.2 updatePeerEurekaNodes

取出所有的 EurekaServer 集群的 url 后，就进入 `updatePeerEurekaNodes` 方法了：

```java
protected void updatePeerEurekaNodes(List<String> newPeerUrls) {
    if (newPeerUrls.isEmpty()) {
        logger.warn("The replica size seems to be empty. Check the route 53 DNS Registry");
        return;
    }

    // 统计现有节点中除去注册中心的节点
    Set<String> toShutdown = new HashSet<>(peerEurekaNodeUrls);
    toShutdown.removeAll(newPeerUrls);
    // 统计新增的节点中除去现有的节点
    Set<String> toAdd = new HashSet<>(newPeerUrls);
    toAdd.removeAll(peerEurekaNodeUrls);
    
    if (toShutdown.isEmpty() && toAdd.isEmpty()) { // No change
        return;
    }

    // Remove peers no long available 删除不再用的节点
    List<PeerEurekaNode> newNodeList = new ArrayList<>(peerEurekaNodes);
    if (!toShutdown.isEmpty()) {
        logger.info("Removing no longer available peer nodes {}", toShutdown);
        int i = 0;
        while (i < newNodeList.size()) {
            PeerEurekaNode eurekaNode = newNodeList.get(i);
            if (toShutdown.contains(eurekaNode.getServiceUrl())) {
                newNodeList.remove(i);
                eurekaNode.shutDown();
            } else {
                i++;
            }
        }
    }

    // Add new peers 添加新的节点
    if (!toAdd.isEmpty()) {
        logger.info("Adding new peer nodes {}", toAdd);
        for (String peerUrl : toAdd) {
            newNodeList.add(createPeerEurekaNode(peerUrl));
        }
    }
    this.peerEurekaNodes = newNodeList;
    this.peerEurekaNodeUrls = new HashSet<>(newPeerUrls);
}

```

上面的两个 Set 的计算比较容易理解，中间的移除和下面的添加咱来瞅瞅。

中间的移除动作，它是拿出所有在 `toShutdown` 集合中的节点信息，执行 `shutDown` 方法。而 shutdown 方法中只有两个动作，都是停止定时任务调度器。

```java
private final TaskDispatcher<String, ReplicationTask> batchingDispatcher;
private final TaskDispatcher<String, ReplicationTask> nonBatchingDispatcher;

public void shutDown() {
    batchingDispatcher.shutdown();
    nonBatchingDispatcher.shutdown();
}
```

而下面向 `newNodeList` 中添加的新注册的节点，要通过 `createPeerEurekaNode` 方法构造 `PeerEurekaNode` ：

```java
protected PeerEurekaNode createPeerEurekaNode(String peerEurekaNodeUrl) {
    JerseyReplicationClient replicationClient = JerseyReplicationClient
            .createReplicationClient(serverConfig, serverCodecs, peerEurekaNodeUrl);

    this.replicationClientAdditionalFilters.getFilters()
            .forEach(replicationClient::addReplicationClientFilter);

    String targetHost = hostFromUrl(peerEurekaNodeUrl);
    if (targetHost == null) {
        targetHost = "host";
    }
    return new PeerEurekaNode(registry, targetHost, peerEurekaNodeUrl,
            replicationClient, serverConfig);
}
```

一开始的这个 `JerseyReplicationClient` 比较重要，它是 EurekaServer 集群中，自身请求其他 EurekaServer 节点的客户端，这个 `createReplicationClient` 方法小册就不展开了，方法比较长而且都是网络通信相关的部分，感兴趣的小伙伴可以借助IDE自行简单阅读一下，咱只关心一点：它**继承了 `AbstractJerseyEurekaHttpClient`** ，而这个 `AbstractJerseyEurekaHttpClient` 就是之前和后面看所有关于 **Eureka 节点之间通信的核心客户端**！

最后，把这个客户端包含在 `PeerEurekaNode` 中，构造出对象，返回，创建也就算完成了。

到这里咱应该意识到一点：EurekaServer 在存储节点时，采取的策略是**一个微服务实例节点一个客户端**，每个客户端都有自己的标识，对应发送的请求路径也就不同。

## 小结

1. EurekaServer 在初始启动时会同步集群中其他 EurekaServer 节点的注册表，并保存到本地，同步的方法是 syncUp ；
2. EurekaClient 注册到 EurekaServer 的动作是 `register` 方法，它是注册动作的核心；
3. EurekaServer 每隔一段时间（默认10分钟）会向集群中更新一次节点的注册信息。

【了解了 EurekaServer 的注册表同步，下一章咱来了解 EurekaClient 的注册信息获取机制（此举是为了保证注册中心宕机时微服务仍能尝试调用远程的其他服务实例）】

