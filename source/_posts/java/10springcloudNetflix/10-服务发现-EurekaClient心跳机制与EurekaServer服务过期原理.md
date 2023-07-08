---
title: 10-服务发现-EurekaClient心跳机制与EurekaServer服务过期原理
toc: true
tags: springcloud
categories: 
    - [java]
    - [springcloudNetflix]
---

EurekaServer 判断 EurekaClient 是否还存活的核心机制就是通过心跳，咱在前面也提到过，每次心跳的核心都相当于是一次**续租**，本章咱就来看看 Eureka 的心跳机制（心跳的原理相对比较简单，故篇幅会略短）。

<!--more-->

## 1. EurekaClient发送心跳

### 1.0 搭建Debug环境

咱将Debug环境简化为一个 EurekaServer 与一个 EurekaClient ，之后在 eureka-client 的 `application.properties` 中加入一个配置：

```properties
logging:
  level:
    root: debug
```

咱前面也看了那么多源码了，咱能意识到一个点吧：Eureka 在进行服务间交互时会打debug日志。

启动 server 与 client 后，在 client 的控制台上能找到这样的日志：

```ini
o.a.http.impl.client.DefaultHttpClient   : Connection can be kept alive indefinitely
c.n.d.shared.MonitoredConnectionManager  : Released connection is reusable.
c.n.d.shared.NamedConnectionPool         : Releasing connection [{}->http://eureka-server-9001.com:9001][null]
c.n.d.shared.NamedConnectionPool         : Pooling connection [{}->http://eureka-server-9001.com:9001][null]; keep alive indefinitely
c.n.d.shared.NamedConnectionPool         : Notifying no-one, there are no waiting threads
com.netflix.discovery.DiscoveryClient    : The total number of all instances in the client now is 1
n.d.s.t.j.AbstractJerseyEurekaHttpClient : Jersey HTTP PUT http://eureka-server-9001.com:9001/eureka//apps/EUREKA-CLIENT/eureka-client; statusCode=200
com.netflix.discovery.DiscoveryClient    : DiscoveryClient_EUREKA-CLIENT/eureka-client - Heartbeat status: 200
com.netflix.discovery.DiscoveryClient    : Completed cache refresh task for discovery. All Apps hash code is Local region apps hashcode: UP_1_, is fetching remote regions? false
```

在这里面有两个重要的类：`DiscoveryClient` 、`AbstractJerseyEurekaHttpClient` 。之前咱在读源码时知道 `AbstractJerseyEurekaHttpClient` 中有大量节点间请求的动作，于是咱根据日志找到 `AbstractJerseyEurekaHttpClient` 中对应的方法：`sendHeartBeat` ，咱在这个方法打上断点，一段时间后断点就会停在这个方法上。通过爬断点所在的方法调用栈，找到了来自 DiscoveryClient 的调用位置：**`renew`** 。

### 1.1 renew-DiscoveryClient中的心跳动作来源

```java
boolean renew() {
    EurekaHttpResponse<InstanceInfo> httpResponse;
    try {
        httpResponse = eurekaTransport.registrationClient.sendHeartBeat(instanceInfo.getAppName(), 
                instanceInfo.getId(), instanceInfo, null);
        logger.debug(PREFIX + "{} - Heartbeat status: {}", appPathIdentifier, httpResponse.getStatusCode());
        if (httpResponse.getStatusCode() == Status.NOT_FOUND.getStatusCode()) {
            REREGISTER_COUNTER.increment();
            logger.info(PREFIX + "{} - Re-registering apps/{}", appPathIdentifier, instanceInfo.getAppName());
            long timestamp = instanceInfo.setIsDirtyWithTime();
            boolean success = register();
            if (success) {
                instanceInfo.unsetIsDirty(timestamp);
            }
            return success;
        }
        return httpResponse.getStatusCode() == Status.OK.getStatusCode();
    } // catch ......
}
```

try 结构的第一句就是向 EurekaServer 发送心跳的请求，它利用的恰好就是上面提到的 `AbstractJerseyEurekaHttpClient` ：

```java
public EurekaHttpResponse<InstanceInfo> sendHeartBeat(String appName, String id, 
        InstanceInfo info, InstanceStatus overriddenStatus) {
    String urlPath = "apps/" + appName + '/' + id;
    ClientResponse response = null;
    try {
        WebResource webResource = jerseyClient.resource(serviceUrl)
                .path(urlPath)
                .queryParam("status", info.getStatus().toString())
                .queryParam("lastDirtyTimestamp", info.getLastDirtyTimestamp().toString());
        if (overriddenStatus != null) {
            webResource = webResource.queryParam("overriddenstatus", overriddenStatus.name());
        }
        Builder requestBuilder = webResource.getRequestBuilder();
        addExtraHeaders(requestBuilder);
        response = requestBuilder.put(ClientResponse.class);
        // ......
}
```

它这里使用 Jersey 的相关API构建请求，并向 EurekaServer 发送PUT请求。

### 1.2 定时心跳的发送来源

在前面第8章的第2节，咱看过一个注册信息的增量获取，它当时是 `DiscoveryClient` 用了一个定时任务调度器实现的。翻到当时的那个方法，可以发现它还同时开启了心跳的定时任务：

```java
private void initScheduledTasks() {
    // .......
    if (clientConfig.shouldRegisterWithEureka()) {
        int renewalIntervalInSecs = instanceInfo.getLeaseInfo().getRenewalIntervalInSecs();
        int expBackOffBound = clientConfig.getHeartbeatExecutorExponentialBackOffBound();
        logger.info("Starting heartbeat executor: " + "renew interval is: {}", renewalIntervalInSecs);

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
        // ......
}
```

其实这个调度的创建与前面获取增量注册信息的方式非常相似。这里它的实际执行线程是 `HeartbeatThread` ：

```java
private class HeartbeatThread implements Runnable {
    @Override
    public void run() {
        if (renew()) {
            lastSuccessfulHeartbeatTimestamp = System.currentTimeMillis();
        }
    }
}
```

果然在这里调了 `renew` 方法，至此 EurekaClient 的心跳发送也就明了了，逻辑相对简单。

## 2. EurekaServer接收心跳请求

在第7章的1.3.5.2节中咱提到了一个类叫 `InstanceResource` ，它是接收微服务实例请求的一个基于 Jersey 的 Restful WebService 接口，在这里面有一个 `renewLease` 方法，它就是接收心跳请求的：

```java
@PUT
public Response renewLease(
        @HeaderParam(PeerEurekaNode.HEADER_REPLICATION) String isReplication,
        @QueryParam("overriddenstatus") String overriddenStatus,
        @QueryParam("status") String status,
        @QueryParam("lastDirtyTimestamp") String lastDirtyTimestamp) {
    boolean isFromReplicaNode = "true".equals(isReplication);
    boolean isSuccess = registry.renew(app.getName(), id, isFromReplicaNode);

    // ......
}
```

将断点打在方法上，启动 eureka-client ，一段时间后就会收到 client 发送的心跳请求，程序会停在断点处。

方法体的第2行 `renew` 方法就是接到心跳请求后的处理（ EurekaServer 叫 `renew` ，EurekaClient 也叫 `renew` ）。

### 2.1 renew-InstanceRegistry处理服务心跳

```java
public boolean renew(final String appName, final String serverId,
        boolean isReplication) {
    log("renew " + appName + " serverId " + serverId + ", isReplication {}" + isReplication);
    List<Application> applications = getSortedApplications();
    for (Application input : applications) {
        if (input.getName().equals(appName)) {
            InstanceInfo instance = null;
            for (InstanceInfo info : input.getInstances()) {
                if (info.getId().equals(serverId)) {
                    instance = info;
                    break;
                }
            }
            publishEvent(new EurekaInstanceRenewedEvent(this, appName, serverId,
                    instance, isReplication));
            break;
        }
    }
    return super.renew(appName, serverId, isReplication);
}
```

这里可以看得出来，它是循环所有的服务，先把当前心跳的服务找出来，再从服务中找出对应的实例，之后发布 `EurekaInstanceRenewedEvent` 事件，调用父类 `PeerAwareInstanceRegistryImpl` 的 `renew` 方法进行真正地处理。

### 2.2 PeerAwareInstanceRegistryImpl#renew

```java
public boolean renew(final String appName, final String id, final boolean isReplication) {
    if (super.renew(appName, id, isReplication)) {
        replicateToPeers(Action.Heartbeat, appName, id, null, null, isReplication);
        return true;
    }
    return false;
}
```

发现要继续往父类调，继续走到 `AbstractInstanceRegistry` 的 `renew` 方法：

### 2.3 AbstractInstanceRegistry#renew

走到这里就没什么特别好深入探讨的了，源码逻辑比较简单，大概看看都有什么流程处理就OK了（注释已标注在源码）。

```java
public boolean renew(String appName, String id, boolean isReplication) {
    RENEW.increment(isReplication);
    // 先从本地的所有租约中查看是否有当前服务
    Map<String, Lease<InstanceInfo>> gMap = registry.get(appName);
    Lease<InstanceInfo> leaseToRenew = null;
    if (gMap != null) {
        leaseToRenew = gMap.get(id);
    }
    // 如果服务不存在，说明出现异常，给对应的RENEW_NOT_FOUND计数器添加计数
    if (leaseToRenew == null) {
        RENEW_NOT_FOUND.increment(isReplication);
        logger.warn("DS: Registry: lease doesn't exist, registering resource: {} - {}", appName, id);
        return false;
    } else {
        // 租约存在，取出服务实例信息
        InstanceInfo instanceInfo = leaseToRenew.getHolder();
        if (instanceInfo != null) {
            // 刷新服务实例的状态，并在正常状态下设置到服务实例中
            InstanceStatus overriddenInstanceStatus = this.getOverriddenInstanceStatus(
                    instanceInfo, leaseToRenew, isReplication);
            if (overriddenInstanceStatus == InstanceStatus.UNKNOWN) {
                // log ......
                RENEW_NOT_FOUND.increment(isReplication);
                return false;
            }
            if (!instanceInfo.getStatus().equals(overriddenInstanceStatus)) {
                // log ......
                instanceInfo.setStatusWithoutDirty(overriddenInstanceStatus);
            }
        }
        // 计数、记录下一次租约应到的时间
        renewsLastMin.increment();
        leaseToRenew.renew();
        return true;
    }
}
```

其中最后的 `leaseToRenew.renew();` ，动作就是取当前时间，加上预先设置好的心跳间隔时间（默认30秒）。

### 2.4 同步至其他EurekaServer节点

回到 `PeerAwareInstanceRegistryImpl` 中，调用 `AbstractInstanceRegistry` 的 `renew` 方法成功后，会执行 `replicateToPeers` 方法，将本次心跳同步给其他 EurekaServer ：

```java
private void replicateToPeers(Action action, String appName, String id,
                              InstanceInfo info /* optional */,
                              InstanceStatus newStatus /* optional */, boolean isReplication) {
    Stopwatch tracer = action.getTimer().start();
    try {
        if (isReplication) {
            numberOfReplicationsLastMin.increment();
        }
        // If it is a replication already, do not replicate again as this will create a poison replication
        if (peerEurekaNodes == Collections.EMPTY_LIST || isReplication) {
            return;
        }

        for (final PeerEurekaNode node : peerEurekaNodes.getPeerEurekaNodes()) {
            // If the url represents this host, do not replicate to yourself.
            if (peerEurekaNodes.isThisMyUrl(node.getServiceUrl())) {
                continue;
            }
            replicateInstanceActionsToPeers(action, appName, id, info, newStatus, node);
        }
    } finally {
        tracer.stop();
    }
}
```

上面先是进行一些判断，下面的 for 循环中它会同步除自己以外的所有 EurekaServer 节点来同步本次心跳，直接来看 `replicateInstanceActionsToPeers` 方法：

```java
private void replicateInstanceActionsToPeers(Action action, String appName,
        String id, InstanceInfo info, InstanceStatus newStatus, PeerEurekaNode node) {
    try {
        InstanceInfo infoFromRegistry = null;
        CurrentRequestVersion.set(Version.V2);
        switch (action) {
            // ......
            case Heartbeat:
                InstanceStatus overriddenStatus = overriddenInstanceStatusMap.get(id);
                infoFromRegistry = getInstanceByAppAndId(appName, id, false);
                node.heartbeat(appName, id, infoFromRegistry, overriddenStatus, false);
                break;
            // ......
}
```

这个方法咱之前在第8章4.3节的集群注册中看过，当时咱关注的是 `Register` 的分支，而心跳动作很明显是第一个 `Heartbeat` 分支。这个分支里面的动作就是拿 EurekaServer 中的其他节点，执行心跳的动作。

这些步骤执行完毕后，一个心跳的完整流程也就结束了。

如果服务长时间没有心跳动作，则 EurekaServer 会认定其租约过期，这里面的处理逻辑咱也来了解一下。

## 3. EurekaServer处理服务过期

还记得 EurekaServer 启动时的那个核心方法 `initEurekaServerContext` 吗？

```java
protected void initEurekaServerContext() throws Exception {
    // ......
    // Copy registry from neighboring eureka node
    // 第7章 Eureka复制集群节点注册表
    int registryCount = this.registry.syncUp();
    // 计算最少续租次数等指标、初始化服务剔除定时器
    this.registry.openForTraffic(this.applicationInfoManager, registryCount);
    // ......
}
```

看源码，在 `PeerAwareInstanceRegistry` 同步注册表的动作完成后，下一步还要初始化一些参数指标和定时任务。而这里面的服务剔除定时器，就是处理服务过期的核心，咱进入 `openForTraffic` 方法：

### 3.1 openForTraffic

源码节选中包含部分注释：

```java
public void openForTraffic(ApplicationInfoManager applicationInfoManager, int count) {
    // Renewals happen every 30 seconds and for a minute it should be a factor of 2.
    // 默认情况每个服务实例每分钟需要发生两次心跳动作
    this.expectedNumberOfClientsSendingRenews = count;
    updateRenewsPerMinThreshold();
    // 省略部分不太重要的源码 ......
    applicationInfoManager.setInstanceStatus(InstanceStatus.UP);
    // 3.2 父类AbstractInstanceRegistry初始化定时任务
    super.postInit();
}
```

这里面上面的 `updateRenewsPerMinThreshold` 方法包含之前看到的最小心跳计算逻辑，咱这里不再贴出。下面的 `postInit` 方法，有定时任务的初始化，咱进到这里面看。

### 3.2 AbstractInstanceRegistry#postInit

```java
protected void postInit() {
    renewsLastMin.start();
    if (evictionTaskRef.get() != null) {
        evictionTaskRef.get().cancel();
    }
    // 创建定时任务，并设置定时周期(默认1分钟)
    evictionTaskRef.set(new EvictionTask());
    evictionTimer.schedule(evictionTaskRef.get(),
            serverConfig.getEvictionIntervalTimerInMs(),
            serverConfig.getEvictionIntervalTimerInMs());
}
```

这里它把定时任务 `EvictionTask` 创建出来了，并且设置了调度逻辑。下面咱着重看 `EvictionTask` 的工作逻辑。

## 4. EvictionTask的服务过期原理

作为定时任务，核心方法自然是 run 方法：

### 4.1 run

```java
public void run() {
    try {
        long compensationTimeMs = getCompensationTimeMs();
        logger.info("Running the evict task with compensationTime {}ms", compensationTimeMs);
        evict(compensationTimeMs);
    } // catch ......
}
```

自然的进到 evict 方法。

### 4.2 evict

这段源码比较长，而且有不少单行注释，小册都进行了翻译：

```java
public void evict(long additionalLeaseMs) {
    logger.debug("Running the evict task");

    // 4.2.1 如果服务过期策略被禁用，则不进行服务过期
    if (!isLeaseExpirationEnabled()) {
        logger.debug("DS: lease expiration is currently disabled.");
        return;
    }
    
    // We collect first all expired items, to evict them in random order. For large eviction sets,
    // if we do not that, we might wipe out whole apps before self preservation kicks in. By randomizing it,
    // the impact should be evenly distributed across all applications.
    // 我们首先收集所有过期的租约，以随机顺序将其逐出。对于比较大规模的服务过期，
    // 如果不这样做的话，我们可能会在自我保护开始之前先清除整个应用程序。
    // 通过将其随机化，影响应在所有应用程序中平均分配。
    // 4.2.2 这一段的核心逻辑就是将所有判定为“过期”的租约都取出来
    List<Lease<InstanceInfo>> expiredLeases = new ArrayList<>();
    for (Entry<String, Map<String, Lease<InstanceInfo>>> groupEntry : registry.entrySet()) {
        Map<String, Lease<InstanceInfo>> leaseMap = groupEntry.getValue();
        if (leaseMap != null) {
            for (Entry<String, Lease<InstanceInfo>> leaseEntry : leaseMap.entrySet()) {
                Lease<InstanceInfo> lease = leaseEntry.getValue();
                if (lease.isExpired(additionalLeaseMs) && lease.getHolder() != null) {
                    expiredLeases.add(lease);
                }
            }
        }
    }

    // To compensate for GC pauses or drifting local time, we need to use current registry size as a base for
    // triggering self-preservation. Without that we would wipe out full registry.
    // 为了补偿GC暂停或本地时间偏移，我们需要使用当前注册表大小作为触发自我保存的基础。否则，我们将清除完整的注册表。
    // 4.2.3 这个设计是考虑到AP了，它一次性不会把所有的服务实例都过期掉
    int registrySize = (int) getLocalRegistrySize();
    int registrySizeThreshold = (int) (registrySize * serverConfig.getRenewalPercentThreshold());
    int evictionLimit = registrySize - registrySizeThreshold;

    int toEvict = Math.min(expiredLeases.size(), evictionLimit);
    if (toEvict > 0) {
        logger.info("Evicting {} items (expired={}, evictionLimit={})", toEvict, expiredLeases.size(), evictionLimit);

        // 4.2.4 将服务实例过期
        Random random = new Random(System.currentTimeMillis());
        for (int i = 0; i < toEvict; i++) {
            // Pick a random item (Knuth shuffle algorithm)
            int next = i + random.nextInt(expiredLeases.size() - i);
            Collections.swap(expiredLeases, i, next);
            Lease<InstanceInfo> lease = expiredLeases.get(i);

            String appName = lease.getHolder().getAppName();
            String id = lease.getHolder().getId();
            EXPIRED.increment();
            logger.warn("DS: Registry: expired lease for {}/{}", appName, id);
            // 4.2.5 【服务过期】
            internalCancel(appName, id, false);
        }
    }
}
```

下面，咱逐步来看。

#### 4.2.1 服务策略被禁用时不过期

```java
 if (!isLeaseExpirationEnabled()) {
        logger.debug("DS: lease expiration is currently disabled.");
        return;
    }

public boolean isLeaseExpirationEnabled() {
    // 如果自我保护模式被禁用，则直接认为服务策略没有被禁用，允许过期服务实例
    if (!isSelfPreservationModeEnabled()) {
        // The self preservation mode is disabled, hence allowing the instances to expire.
        return true;
    }
    return numberOfRenewsPerMinThreshold > 0 && getNumOfRenewsInLastMin() > numberOfRenewsPerMinThreshold;
}
```

这里咱看到那个熟悉的概念了：如果自我保护模式被禁用，则不会出现开启自我保护的情况，该过期的服务实例就是会被下线掉。之后下面有一个上面已经看到的概念：`numberOfRenewsPerMinThreshold` 。上面的 `updateRenewsPerMinThreshold` 动作会设置好每分钟应该接收的最少心跳次数（默认情况是所有已注册的服务实例数 * 2 * 0.85 ）咱都比较熟悉了，那这里面就判断这一个时间周期内，总共收到的心跳是否比最少接收心跳次数多。如果多，那我就认为我 Eureka 目前是正常的，否则就认为自己所处的网络环境可能出现了问题或者其他原因，这样就相当于进了自我保护模式，就不再过期服务实例。

#### 4.2.2 取出过期实例

```java
    List<Lease<InstanceInfo>> expiredLeases = new ArrayList<>();
    for (Entry<String, Map<String, Lease<InstanceInfo>>> groupEntry : registry.entrySet()) {
        Map<String, Lease<InstanceInfo>> leaseMap = groupEntry.getValue();
        if (leaseMap != null) {
            for (Entry<String, Lease<InstanceInfo>> leaseEntry : leaseMap.entrySet()) {
                Lease<InstanceInfo> lease = leaseEntry.getValue();
                if (lease.isExpired(additionalLeaseMs) && lease.getHolder() != null) {
                    expiredLeases.add(lease);
                }
            }
        }
    }
```

这一段逻辑倒也不算复杂，判断服务实例是否过期的核心方法也能看出来是最里面的 `lease.isExpired` 方法，咱看看这个方法的实现：

```java
public boolean isExpired(long additionalLeaseMs) {
    return (evictionTimestamp > 0 || System.currentTimeMillis() > (lastUpdateTimestamp + duration + additionalLeaseMs));
}

```

这个过期时间计算，它默认是判断最近一次心跳时间是否已经在 90 秒之前（duration 的默认值）。不过这个地方有个问题，因为在服务实例续租的时候，已经加了一次 duration ：

```java
public void renew() {
    lastUpdateTimestamp = System.currentTimeMillis() + duration;
}
```

所以在极端情况下，过期一个服务实例的最大时间阈值是 90 + 90 = 180 秒。

最终判断并筛选出要过期的服务实例之后，进入下一个步骤。

#### 4.2.3 计算部分过期数量

```java
private double renewalPercentThreshold = 0.85;

    int registrySize = (int) getLocalRegistrySize();
    int registrySizeThreshold = (int) (registrySize * serverConfig.getRenewalPercentThreshold());
    int evictionLimit = registrySize - registrySizeThreshold;

    int toEvict = Math.min(expiredLeases.size(), evictionLimit);
```

这个部分，它会计算这一次服务过期的服务数量。由于 Eureka 设计的原则是符合 AP ，它一次性不会把所有过期的服务都下线掉。默认情况下，`renewalPercentThreshold` 的初始值是 0.85 ，这就意味着每次过期服务触发时，都是过期整体所有服务实例数量的 (1-0.85)=15% 。

举个实际的栗子：

如果当前有15个服务，但挂了7个，整体过期数量大于 15% ，需要执行分批过期。下面推演整个分批过期的逻辑。

1. 第一轮过期，15 * 85% = 12.75 ，向下取整为 12 ，代表本轮计划要过期 3 个服务实例。下面执行 Math.min 的动作，得出最终数量为 3 ，代表第一轮过期数量为 3 个，剩余 12 个服务实例，4 个服务过期；
2. 第二轮过期，12 * 85% = 10.2 ，向下取整为 10 ，代表本轮计划要过期 2 个服务实例。执行 Math.min 的动作后得出最终数量为 2 ，代表第二轮过期数量为 2 个，剩余 10 个服务实例，2 个服务过期；
3. 第三轮过期，10 * 85% = 8.5 ，向下取整为 8 ，代表本轮计划要过期 2 个服务实例。执行 Math.min 的动作后得出最终数量为 2 ，代表第三轮过期数量为 2 个，剩余 8 个服务实例，均为正常实例。

#### 4.2.4 服务实例过期动作准备

```java
if (toEvict > 0) {
  // logger ......
  Random random = new Random(System.currentTimeMillis());
  for (int i = 0; i < toEvict; i++) {
    // Pick a random item (Knuth shuffle algorithm)
    int next = i + random.nextInt(expiredLeases.size() - i);
    Collections.swap(expiredLeases, i, next);
    Lease<InstanceInfo> lease = expiredLeases.get(i);

    String appName = lease.getHolder().getAppName();
    String id = lease.getHolder().getId();
    EXPIRED.increment();
    // ......
  }
}
```

这部分可以看到，它的过期动作不是直接按顺序过期，也不是按照服务名称过期（它这么设计也是为了怕一下子把一个服务的所有实例都干掉，那就相当于没有这个服务了），而是采用随机的方式，随机下线。思路也都可以理解，不再多展开了，小伙伴们跟着源码走一遍体会下即可。

#### 4.2.5 服务实例过期的实际动作

上面注释的部分就剩这一句代码了：

```java
    internalCancel(appName, id, false);

```

一看这个动作，感觉好像之前见过是吧，那肯定的，在第 8 章的 3.6 节提到了，它就是服务下线的核心方法。看一眼里面的实现逻辑：

```java
protected boolean internalCancel(String appName, String id, boolean isReplication) {
    handleCancelation(appName, id, isReplication);
    return super.internalCancel(appName, id, isReplication);
}

private void handleCancelation(String appName, String id, boolean isReplication) {
    log("cancel " + appName + ", serverId " + id + ", isReplication "
            + isReplication);
    publishEvent(new EurekaInstanceCanceledEvent(this, appName, id, isReplication));
}

```

子类的扩展仅仅是发布了一个 `EurekaInstanceCanceledEvent` 事件而已，还是关注父类的实现吧。

源码比较长，该加的注释都加到位了：

```java
protected boolean internalCancel(String appName, String id, boolean isReplication) {
    try {
        read.lock();
        // 下线记录次数(用于监控)
        CANCEL.increment(isReplication);
        Map<String, Lease<InstanceInfo>> gMap = registry.get(appName);
        Lease<InstanceInfo> leaseToCancel = null;
        if (gMap != null) {
            // 仅移除本地的租约信息
            leaseToCancel = gMap.remove(id);
        }
        // 记录最近下线信息(用于EurekaServer的Dashboard)
        synchronized (recentCanceledQueue) {
            recentCanceledQueue.add(new Pair<Long, String>(System.currentTimeMillis(), appName + "(" + id + ")"));
        }
        // 移除服务状态覆盖的记录(已不再需要)
        InstanceStatus instanceStatus = overriddenInstanceStatusMap.remove(id);
        if (instanceStatus != null) {
            logger.debug("Removed instance id {} from the overridden map which has value {}", id, instanceStatus.name());
        }
        if (leaseToCancel == null) {
            // ......
        } else {
            leaseToCancel.cancel();
            InstanceInfo instanceInfo = leaseToCancel.getHolder();
            String vip = null;
            String svip = null;
            if (instanceInfo != null) {
                // 服务实例状态设置为已删除
                instanceInfo.setActionType(ActionType.DELETED);
                // EurekaServer集群同步服务变动设计
                recentlyChangedQueue.add(new RecentlyChangedItem(leaseToCancel));
                instanceInfo.setLastUpdatedTimestamp();
                vip = instanceInfo.getVIPAddress();
                svip = instanceInfo.getSecureVipAddress();
            }
            // 从缓存中移除
            invalidateCache(appName, vip, svip);
            logger.info("Cancelled instance {}/{} (replication={})", appName, id, isReplication);
            return true;
        }
    } finally {
        read.unlock();
    }
}
```

总的来看，思路也不算复杂，跟前面微服务的注册动作 `register` 步骤差不多但刚好相反。

至此，服务实例就算在 EurekaServer 中过期了。

## 小结

1. EurekaClient 发送心跳包是通过 `DiscoveryClient` 的 `heartbeatExecutor` 执行 `HeartbeatThread` 的线程方法，默认时间是 30 秒一次。
2. EurekaServer 接收心跳请求是通过 `InstanceResource` 的 `renewLease` 方法，先自己接收并处理来自 EurekaClient 的心跳，后将该心跳同步至 EurekaServer 集群中的其它节点。
3. EurekaServer 在初始化时会额外启动一个 `EvictionTask` 定时任务，默认每分钟检查一次心跳收集情况，并对长时间没有心跳的服务实例执行过期处理。

【对于 Eureka 常见的内容只剩下一个前面留下的坑了，就是 EurekaServer 的服务分区，这部分目前介绍可能有点超前，里面涉及到负载均衡的内容了，所以咱放到后面来介绍。那到此为止关于“服务治理与服务发现-Eureka”的部分就解析完成了，接下来咱进入下一个部分：**服务调用与负载均衡-Ribbon**】

