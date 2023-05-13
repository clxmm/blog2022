---
title: 08-服务发现-EurekaClient的注册信息获取机制与集群注册原理
toc: true
tags: springcloud
categories: 
    - [java]
    - [springcloudNetflix]
---

咱在上一章中留下一个坑，是关于 Eureka 的注册信息的获取，它是围绕着 `recentlyChangedQueue` 来展开：

```java
    // 处理状态、续约时间戳等
    registrant.setActionType(ActionType.ADDED);
    recentlyChangedQueue.add(new RecentlyChangedItem(lease));
    // ......
```

AbstractInstanceRegistry#register

<!--more-->

Eureka 的注册信息获取分为全量获取和增量获取，顾名思义，全量获取是在一开始微服务实例启动时一次性拉取当前所有的服务实例注册信息，而增量获取是在服务启动后运行中的一段时间后定时获取。那咱先来看全量获取的动作：

### 1.0 搭建Debug场景

咱先启动两个 EurekaServer （不玩太多了，启动慢还吃内存），再启动 eureka-client ，让 client 注册到 server 上，之后再Debug启动一个 eureka-client2 ，同样也注册到两个 EurekaServer 上（此时有两个 EurekaServer 、两个 EurekaClient ）。前面咱已经读过 EurekaClient 的自动配置，这里咱直接来到 `EurekaClientAutoConfiguration` 的注册 `EurekaClient` 位置。前面也提到，`CloudEurekaClient` 的实例化可以创建真正地 EurekaClient 并注册到 EurekaServer 上，扒到最底层的构造方法，里面有一句代码是咱之前留下的坑（第6章5.8.1.11节）：

```java
DiscoveryClient(ApplicationInfoManager applicationInfoManager, EurekaClientConfig config, 
                AbstractDiscoveryClientOptionalArgs args, Provider<BackupRegistry> backupRegistryProvider, 
                EndpointRandomizer endpointRandomizer) {
    // ......
    if (clientConfig.shouldFetchRegistry() && !fetchRegistry(false)) {
        fetchRegistryFromBackup();
    }
```

com.netflix.discovery#fetchRegistryFromBackup

这里它调用了 `fetchRegistry` 方法：（节选）

```java
private boolean fetchRegistry(boolean forceFullRegistryFetch) {
    // ......
        if (clientConfig.shouldDisableDelta()
                || (!Strings.isNullOrEmpty(clientConfig.getRegistryRefreshSingleVipAddress()))
                || forceFullRegistryFetch
                || (applications == null)
                || (applications.getRegisteredApplications().size() == 0)
                || (applications.getVersion() == -1)) {
            // ......
            getAndStoreFullRegistry();
        } else {
            getAndUpdateDelta(applications);
        }
     // ......
}
```

注意这里它有两个方法，看方法名也能区分的出来是**全量**和**增量**！咱这里先研究全量获取：

![](./img/2023/05/cloud07-5.png)

往下走，获取到 `apps` 后，看一眼内容：

![](./img/2023/05/cloud07-6.png)

等会，这不对啊，第一个 eureka-client 获取到了是对的，这咋把那两个 server 也获取到了？那岂不是会一起记录下来？别着急，往下看源码：（已去除日志）

```java
    if (httpResponse.getStatusCode() == Status.OK.getStatusCode()) {
        apps = httpResponse.getEntity();
    }

    if (apps == null) { // logger ......
    } else if (fetchRegistryGeneration.compareAndSet(currentUpdateGeneration, currentUpdateGeneration + 1)) {
        localRegionApps.set(this.filterAndShuffle(apps));
```

注意看最底下一行：它在 set 之前干了一件事：`filterAndShuffle`

### 1.2 filterAndShuffle：过滤和乱序

> Gets the applications after filtering the applications for instances with only UP states and shuffling them. The filtering depends on the option specified by the configuration EurekaClientConfig.shouldFilterOnlyUpInstances(). Shuffling helps in randomizing the applications list there by avoiding the same instances receiving traffic during start ups.
>
> 在仅对具有UP状态的实例的应用程序进行过滤并将其乱序后获取。 筛选的动作取决于配置 `EurekaClientConfig.shouldFilterOnlyUpInstances()` 指定的选项，而乱序可避免启动时同一实例接收流量，从而有助于在此处随机分配应用程序列表。

文档注释只是解释了这个方法的实际作用和触发机制，那咱就进入到这个实现机制中稍微看一看。

```java
private Applications filterAndShuffle(Applications apps) {
    if (apps != null) {
        if (isFetchingRemoteRegionRegistries()) { // 默认情况下这个分支不会进入
            Map<String, Applications> remoteRegionVsApps = new ConcurrentHashMap<String, Applications>();
            apps.shuffleAndIndexInstances(remoteRegionVsApps, clientConfig, instanceRegionChecker);
            for (Applications applications : remoteRegionVsApps.values()) {
                applications.shuffleInstances(clientConfig.shouldFilterOnlyUpInstances());
            }
            this.remoteRegionVsApps = remoteRegionVsApps;
        } else {
            apps.shuffleInstances(clientConfig.shouldFilterOnlyUpInstances());
        }
    }
    return apps;
}
```

通过Debug发现，`isFetchingRemoteRegionRegistries` 方法一直返回 false ，进入下面 else 中的 `shuffleInstances` 方法：

```java
private void shuffleInstances(boolean filterUpInstances, 
        boolean indexByRemoteRegions,
        @Nullable Map<String, Applications> remoteRegionsRegistry, 
        @Nullable EurekaClientConfig clientConfig,
        @Nullable InstanceRegionChecker instanceRegionChecker) {
    Map<String, VipIndexSupport> secureVirtualHostNameAppMap = new HashMap<>();
    Map<String, VipIndexSupport> virtualHostNameAppMap = new HashMap<>();
    for (Application application : appNameApplicationMap.values()) {
        if (indexByRemoteRegions) {
            application.shuffleAndStoreInstances(remoteRegionsRegistry, clientConfig, instanceRegionChecker);
        } else {
            application.shuffleAndStoreInstances(filterUpInstances);
        }
        this.addInstancesToVIPMaps(application, virtualHostNameAppMap, secureVirtualHostNameAppMap);
    }
    shuffleAndFilterInstances(virtualHostNameAppMap, filterUpInstances);
    shuffleAndFilterInstances(secureVirtualHostNameAppMap, filterUpInstances);

    this.virtualHostNameAppMap.putAll(virtualHostNameAppMap);
    this.virtualHostNameAppMap.keySet().retainAll(virtualHostNameAppMap.keySet());
    this.secureVirtualHostNameAppMap.putAll(secureVirtualHostNameAppMap);
    this.secureVirtualHostNameAppMap.keySet().retainAll(secureVirtualHostNameAppMap.keySet());
}
```

上面的初始化和下面的重新添加咱就不多关心了，注意看中间的 for 循环，它会遍历每一个微服务应用，分别来进行过滤和乱序。进入到 `application.shuffleAndStoreInstances` 中：

```java
public void shuffleAndStoreInstances(boolean filterUpInstances) {
    _shuffleAndStoreInstances(filterUpInstances, false, null, null, null);
}
```

注意它的方法命名规则，它调用的方法前面有一个**下划线**，这是 Eureka 的代码风格设计。对比之前咱看过的 SpringFramework 的源码，也能大概猜出来吧：SpringFramework 中的 `getBean` 与 `doGetBean` 方法，就是这里面的带下划线与不带下划线。

那咱就直接跳转进去看：（源码略长，省略了一些处理步骤，只留下核心步骤）

```java
private void _shuffleAndStoreInstances(boolean filterUpInstances, boolean indexByRemoteRegions,
                                       @Nullable Map<String, Applications> remoteRegionsRegistry,
                                       @Nullable EurekaClientConfig clientConfig,
                                       @Nullable InstanceRegionChecker instanceRegionChecker) {
    List<InstanceInfo> instanceInfoList;
    synchronized (instances) {
        instanceInfoList = new ArrayList<InstanceInfo>(instances);
    }
    boolean remoteIndexingActive = indexByRemoteRegions && null != instanceRegionChecker
      			&& null != clientConfig
            && null != remoteRegionsRegistry;
    if (remoteIndexingActive || filterUpInstances) {
        Iterator<InstanceInfo> it = instanceInfoList.iterator();
        while (it.hasNext()) {
            InstanceInfo instanceInfo = it.next();
            if (filterUpInstances && InstanceStatus.UP != instanceInfo.getStatus()) {
                it.remove();
            } else if (remoteIndexingActive) {
                String instanceRegion = instanceRegionChecker.getInstanceRegion(instanceInfo);
                if (!instanceRegionChecker.isLocalRegion(instanceRegion)) {
                    // ......
                    it.remove();
                }
            }
        }
    }
    Collections.shuffle(instanceInfoList, shuffleRandom);
    this.shuffledInstances.set(instanceInfoList);
}
```

注意中间这段 if 的迭代器循环步骤，它有两个判断条件会进行元素移除（然而在Debug中没有一个被移除掉）；最后有一个 `Collections.shuffle` 动作进行乱序，结束返回。（乱序的目的是为了防止某一个节点短时间被大量处理导致压力过大）

至此，过滤和乱序动作完毕，可以设置到本地了。

### 1.3 设置本地信息缓存

回到全量获取的源码：

```java
    localRegionApps.set(this.filterAndShuffle(apps));

```

筛选和乱序完成后，它将这些服务实例设置到本地注册信息的缓存中，全量获取结束。

## 2. 注册信息的增量获取

之前在看 `DiscoveryClient` 的初始化（第6章5.8.1.8节）时，咱当时注意过一句话：

```java
// finally, init the schedule tasks (e.g. cluster resolvers, heartbeat, instanceInfo replicator, fetch
initScheduledTasks();
```

它会启动一些定时任务，其中关于注册信息的增量获取，这里也有调度：

```java
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

class CacheRefreshThread implements Runnable {
    public void run() {
        refreshRegistry();
    }
}

void refreshRegistry() {
    try {
        // ......
        boolean success = fetchRegistry(remoteRegionsModified);
        // ....
    } // ......
}
```

方法的第一个定时任务就是用来处理增量获取的，它 new 了一个 `CacheRefreshThread` ，这个类仅仅是实现了 `Runnable` 接口，对应的方法是 `refreshRegistry` 方法，而这个 `refreshRegistry` 方法中最终调用了 `fetchRegistry` 进行全量和增量的获取。如果在配置中没有显式地配置 `eureka.client.disable-delta=true` ，或者那一大堆 if 的判断条件中有一个成立时，会触发增量获取。而这个定时任务的时间一路找下去可以发现：`private int registryFetchIntervalSeconds = 30;` ，即 **30** 秒触发一次增量获取。

### 2.1 触发前的处理

咱向 EurekaServer1 上发送如下请求：

[eureka-server-9001.com:9001/eureka/apps…]

根据上面咱的了解，很容易看出来它可以让 client **主动停止提供服务**。经过这个处理后，client 的状态会在 EurekaServer 中被覆盖，发送请求后咱Debug盯着IDEA，等待下一次触发增量获取。

### 2.2 触发增量获取

触发后，断点停在 `getAndUpdateDelta` 方法（部分源码已省略，部分注释已标注在源码中）：

```java
private void getAndUpdateDelta(Applications applications) throws Throwable {
    long currentUpdateGeneration = fetchRegistryGeneration.get();

    Applications delta = null;
    // 发起增量获取
    EurekaHttpResponse<Applications> httpResponse = eurekaTransport.queryClient.getDelta(remoteRegionsRef.get());
    if (httpResponse.getStatusCode() == Status.OK.getStatusCode()) {
        delta = httpResponse.getEntity();
    }

    if (delta == null) {
        // 如果获取不到增量数据，直接执行全量获取
        getAndStoreFullRegistry();
    } // else if ......
                // 合并应用
                updateDelta(delta);
                reconcileHashCode = getReconcileHashCode(applications);
    // ......
}
```

这里获取到 EurekaServer 端的增量数据，通过Debug观察 `httpResponse` 里面的差异：

可以发现 EurekaClient 获取到了被 EurekaServer 覆盖的服务实例状态。不过，由于这个状态仅仅是 EurekaServer 中存放的覆盖状态，所以 EurekaClient 本身的状态还是 UP 。这个 `updateDelta` 动作咱看看是怎么合并的（已去除日志，注释已标注在源码中）：

```java
private void updateDelta(Applications delta) {
    int deltaCount = 0;
    // 遍历所有增量的应用集合
    for (Application app : delta.getRegisteredApplications()) {
        for (InstanceInfo instance : app.getInstances()) {
            // 循环每一个微服务实例来处理
            Applications applications = getApplications();
            String instanceRegion = instanceRegionChecker.getInstanceRegion(instance);
            if (!instanceRegionChecker.isLocalRegion(instanceRegion)) {
                Applications remoteApps = remoteRegionVsApps.get(instanceRegion);
                if (null == remoteApps) {
                    remoteApps = new Applications();
                    remoteRegionVsApps.put(instanceRegion, remoteApps);
                }
                applications = remoteApps;
            }

            ++deltaCount;
            // 微服务实例的注册信息对应的动作为添加时，也添加到自己的缓存中
            if (ActionType.ADDED.equals(instance.getActionType())) {
                Application existingApp = applications.getRegisteredApplications(instance.getAppName());
                if (existingApp == null) {
                    applications.addApplication(app);
                }
                applications.getRegisteredApplications(instance.getAppName()).addInstance(instance);
            } else if (ActionType.MODIFIED.equals(instance.getActionType())) {
                // 处理修改动作
                Application existingApp = applications.getRegisteredApplications(instance.getAppName());
                if (existingApp == null) {
                    applications.addApplication(app);
                }
                applications.getRegisteredApplications(instance.getAppName()).addInstance(instance);
            } else if (ActionType.DELETED.equals(instance.getActionType())) {
                // 处理移除下线动作
                Application existingApp = applications.getRegisteredApplications(instance.getAppName());
                if (existingApp != null) {
                    existingApp.removeInstance(instance);
                    if (existingApp.getInstancesAsIsFromEureka().isEmpty()) {
                        applications.removeApplication(existingApp);
                    }
                }
            }
        }
    }

    getApplications().setVersion(delta.getVersion());
    // 过滤、乱序
    getApplications().shuffleInstances(clientConfig.shouldFilterOnlyUpInstances());

    for (Applications applications : remoteRegionVsApps.values()) {
        applications.setVersion(delta.getVersion());
        applications.shuffleInstances(clientConfig.shouldFilterOnlyUpInstances());
    }
}

```

这部分它会循环遍历出所有服务的所有节点实例，逐个处理添加/修改/删除的逻辑，最后所有节点实例乱序，整理流程不算复杂。

合并的动作完成后，增量获取的动作也就基本完成了。下面咱看看 EurekaServer 是如何处理 EurekaClient 发送的注册信息获取请求。

## 3. EurekaServer处理注册信息请求

EurekaServer 的设计里面，对于接收的请求都是通过一组 XxxResource 来处理，这个注册信息的请求也是如此，`ApplicationsResource` 就是接收该请求的处理类，直接跳转到对应的方法（只截取出关键的部分，注释已标注在源码中）：

### 3.1 getContainers

```java
@GET                       // 参数太多已省略
public Response getContainers(.............) {
    // ......

    // Check if the server allows the access to the registry. The server can
    // restrict access if it is not ready to serve traffic depending on various reasons.
    // 检查服务器是否允许访问注册表。如果服务器由于各种原因尚未准备好服务流量，则可以限制访问。
    if (!registry.shouldAllowAccess(isRemoteRegionRequested)) {
        return Response.status(Status.FORBIDDEN).build();
    }
    CurrentRequestVersion.set(Version.toEnum(version));
    // 响应缓存
    KeyType keyType = Key.KeyType.JSON;
    String returnMediaType = MediaType.APPLICATION_JSON;
    if (acceptHeader == null || !acceptHeader.contains(HEADER_JSON_VALUE)) {
        keyType = Key.KeyType.XML;
        returnMediaType = MediaType.APPLICATION_XML;
    }

    Key cacheKey = new Key(Key.EntityType.Application,
            ResponseCacheImpl.ALL_APPS,
            keyType, CurrentRequestVersion.get(), EurekaAccept.fromString(eurekaAccept), regions
    );

    // 从缓存中取注册信息
    Response response;
    if (acceptEncoding != null && acceptEncoding.contains(HEADER_GZIP_VALUE)) {
        response = Response.ok(responseCache.getGZIP(cacheKey))
                .header(HEADER_CONTENT_ENCODING, HEADER_GZIP_VALUE)
                .header(HEADER_CONTENT_TYPE, returnMediaType)
                .build();
    } else {
        response = Response.ok(responseCache.get(cacheKey)).build();
    }
    return response;
}
```

中间的缓存部分涉及到 EurekaServer 的缓存机制了，这个咱留到下一章再解释。最后边响应的数据是从缓存中取的，咱进到这里面看。

### 3.2 ResponseCacheImpl#get

来到 `ResponseCacheImpl` 中，关于 `ResponseCache` 的概念咱也是放到下一章解释。先看源码：

```java
public String get(final Key key) {
    return get(key, shouldUseReadOnlyResponseCache);
}

String get(final Key key, boolean useReadOnlyCache) {
    Value payload = getValue(key, useReadOnlyCache);
    if (payload == null || payload.getPayload().equals(EMPTY_PAYLOAD)) {
        return null;
    } else {
        return payload.getPayload();
    }
}
```

上面的方法又调了下面的方法，这里面有一个 `Value` 的概念，它是根据指定的 `Key` 取的，不过在本章这个 `Key` 和 `Value` 的意义先暂且放在一边，咱的目的是追踪到一个特定的源码具体实现上。继续往里看 `getValue` 方法：

### 3.3 getValue

```java
Value getValue(final Key key, boolean useReadOnlyCache) {
    Value payload = null;
    try {
        if (useReadOnlyCache) {
            final Value currentPayload = readOnlyCacheMap.get(key);
            if (currentPayload != null) {
                payload = currentPayload;
            } else {
                payload = readWriteCacheMap.get(key);
                readOnlyCacheMap.put(key, payload);
            }
        } else {
            payload = readWriteCacheMap.get(key);
        }
    } catch (Throwable t) {
        logger.error("Cannot get value for key : {}", key, t);
    }
    return payload;
}
```

注意这里面的读取设计很有意思，它会**先去读只读缓存，如果只读缓存没有，再去读可写缓存**，这个机制咱下一章也会解释，这里咱先关心一下 `readWriteCacheMap` 这个成员，它的类型不是 `Map` ，而是一个 `LoadingCache<Key, Value>` ，往上翻代码，在整个 `ResponseCacheImpl` 的构造方法中看到了 `readWriteCacheMap` 的初始化：

### 3.4 readWriteCacheMap的初始化

```java
ResponseCacheImpl(EurekaServerConfig serverConfig, ServerCodecs serverCodecs, 
                  AbstractInstanceRegistry registry) {
    // ......
    this.readWriteCacheMap =
            CacheBuilder.newBuilder().initialCapacity(serverConfig.getInitialCapacityOfResponseCache())
                    // ......
                    .build(new CacheLoader<Key, Value>() {
                        @Override
                        public Value load(Key key) throws Exception {
                            if (key.hasRegions()) {
                                Key cloneWithNoRegions = key.cloneWithoutRegions();
                                regionSpecificKeys.put(cloneWithNoRegions, key);
                            }
                            Value value = generatePayload(key);
                            return value;
                        }
                    });
    // ......
}
```

中间与本篇无关的部分我直接注释掉了，我想让你看的部分是在创建 `readWriteCacheMap` 时获取 `Value` 的部分，它会借助一个 `CacheLoader` 来创建，而获取 `Value` 的动作在 `ResponseCacheImpl` 的下面就定义了：

### 3.5 generatePayload：获取指定Key对应的Value

```java
private Value generatePayload(Key key) {
    Stopwatch tracer = null;
    try {
        String payload;
        switch (key.getEntityType()) {
            case Application:
                boolean isRemoteRegionRequested = key.hasRegions();
                if (ALL_APPS.equals(key.getName())) {
                    if (isRemoteRegionRequested) {
                        tracer = serializeAllAppsWithRemoteRegionTimer.start();
                        payload = getPayLoad(key, registry.getApplicationsFromMultipleRegions(key.getRegions()));
                    } else {
                        tracer = serializeAllAppsTimer.start();
                        payload = getPayLoad(key, registry.getApplications());
                    }
                } else if (ALL_APPS_DELTA.equals(key.getName())) {
                    if (isRemoteRegionRequested) {
                        tracer = serializeDeltaAppsWithRemoteRegionTimer.start();
                        versionDeltaWithRegions.incrementAndGet();
                        versionDeltaWithRegionsLegacy.incrementAndGet();
                        payload = getPayLoad(key, registry.getApplicationDeltasFromMultipleRegions(key.getRegions()));
                    } else {
                        tracer = serializeDeltaAppsTimer.start();
                        versionDelta.incrementAndGet();
                        versionDeltaLegacy.incrementAndGet();
                        payload = getPayLoad(key, registry.getApplicationDeltas());
                    }
                } else {
                    tracer = serializeOneApptimer.start();
                    payload = getPayLoad(key, registry.getApplication(key.getName()));
                }
                break;
            // ......
}
```

可以发现这里面的处理逻辑是根据请求的类型来执行对应的逻辑。上面的全量获取、增量获取在这里面分别代表 `ALL_APPS` 与 `ALL_APPS_DELTA` ，对应的处理逻辑分别是 `getApplicationsFromMultipleRegions` 与 `getApplicationDeltasFromMultipleRegions` （后者方法名中多一个 Deltas）。

读到这里，可以稍微停一下了，接下来咱提上一章留下的坑。

### 3.6 EurekaServer中recentlyChangedQueue对增量获取的作用

上一章的3.7节，微服务实例注册的 `register` 方法源码中提到了 `recentlyChangedQueue` 的处理：

```java
    recentlyChangedQueue.add(new RecentlyChangedItem(lease));

```

它的变量名的后缀是 `Queue` ，它的类型跟上一章提到的 `recentRegisteredQueue` 不同，它只是一个普通的 `ConcurrentLinkedQueue` 而已，它记录的是所有 EurekaClient 的注册动作或者状态修改等事件（泛型类型为 `RecentlyChangedItem` ）。

借助IDEA，发现 `recentlyChangedQueue` 被调用的位置中有添加、清空、遍历，没有出队列的动作，可以由此先大胆猜测，它是记录一段时间后，被同步完成就清空掉。（其实这个猜测是不对的，移除队列的元素不止有 `remove` 、`pop` 操作，还有另外一种，在下面会解释）

几个 `add` 的操作分别在 `register` （微服务注册）、`internalCancel` （微服务下线）、`statusUpdate` （微服务状态更新）、`deleteStatusOverride` （微服务的覆盖状态移除）方法中，这四个方法都会涉及到微服务实例的状态变化，这个队列都要记录下来。而这个队列记录的微服务状态变化记录，为的就是 EurekaServer 集群中其他节点同步这些服务实例的状态。

状态存储到 `recentlyChangedQueue` 后，在 `AbstractInstanceRegistry#getApplicationDeltasFromMultipleRegions()` 中整理出最近一段时间发生状态变化的微服务实例，对这就是上面我突然停下来的那个部分，就是调的这个方法，部分源码（已去除日志打印）：

```java
write.lock();
Iterator<RecentlyChangedItem> iter = this.recentlyChangedQueue.iterator();
while (iter.hasNext()) {
  Lease<InstanceInfo> lease = iter.next().getLeaseInfo();
  InstanceInfo instanceInfo = lease.getHolder();
  Application app = applicationInstancesMap.get(instanceInfo.getAppName());
  if (app == null) {
    app = new Application(instanceInfo.getAppName());
    applicationInstancesMap.put(instanceInfo.getAppName(), app);
    apps.addApplication(app);
  }
  app.addInstance(new InstanceInfo(decorateInstanceInfo(lease)));
}
```

可以看出来，这部分它就是拿到 `recentlyChangedQueue` 的所有微服务实例的状态变更记录，一一整理和和拼装、保存到 `apps` 中，最后返回出去，这些返回出去的这些服务和节点实例就算是增量获取的结果。

### 3.7 recentlyChangedQueue的获取与清除

上面咱已经大概的了解了 `recentlyChangedQueue` 的设置与使用，但 EurekaServer 中节点如何获取 `recentlyChangedQueue` 的数据，以及获取之后如何清除已经同步过的数据，咱下面也了解一下。

根据上面借助IDEA查看 `recentlyChangedQueue` 的调用位置截图，咱能看到的是只有四个位置有 `add` 操作，通常情况下框架也不会瞎用反射去修改内部的成员，那只能从这四个 `add` 的动作入手了。回想上面看到的，在 `syncUp` 方法中有记录服务实例变动的痕迹，以及下面的几个方法 `internalCancel` 、`statusUpdate` 、`deleteStatusOverride` ，分别对应了上面截图中看到的四个 add 动作。但这些动作都只是在本地实例的 EurekaServer 上处理，没有看到远程获取。

#### 3.7.1 搭建测试获取动作的Debug场景

又要改Debug场景了，这次我把 eureka-client1 以正常方式注册到两个注册中心，eureka-client2 只注册到 eureka-server1 上：

```properties
eureka.client.service-url.defaultZone=http://eureka-server-9001.com:9001/eureka/

```

之后给 eureka-server2 以Debug启动，其余应用全部正常启动，给 `register` 方法打上断点。

#### 3.7.2 启动eureka-client2，观察eureka-server2的同步接收

等 eureka-client2 注册到 eureka-server1 时，eureka-server2 的 `register` 方法被触发，通过爬方法调用栈，找到了这个方法所在的 url ：`/eureka/peerreplication/batch` ，刨去必要的 `/eureka` 前缀，得到真实调用的 uri ：**`/peerreplication/batch`** ，意为**批量对等复制**。借助IDEA，发现这个 API 定义在 `PeerReplicationResource` 中：

```java
@Path("/{version}/peerreplication")
@Produces({"application/xml", "application/json"})
public class PeerReplicationResource {
    // ......
    @Path("batch")
    @POST
    public Response batchReplication(ReplicationList replicationList) {
        try {
            ReplicationListResponse batchResponse = new ReplicationListResponse();
            for (ReplicationInstance instanceInfo : replicationList.getReplicationList()) {
                try {
                    batchResponse.addResponse(dispatch(instanceInfo));
                } catch (Exception e) {
                    batchResponse.addResponse(new ReplicationInstanceResponse(Status.INTERNAL_SERVER_ERROR.getStatusCode(), null));
                    logger.error("{} request processing failed for batch item {}/{}",
                            instanceInfo.getAction(), instanceInfo.getAppName(), instanceInfo.getId(), e);
                }
            }
            return Response.ok(batchResponse).build();
        } catch (Throwable e) {
            logger.error("Cannot execute batch Request", e);
            return Response.status(Status.INTERNAL_SERVER_ERROR).build();
        }
    }

```

从这个方法中也能看得出来，它就是拿到了 EurekaServer 集群中其他所有节点的注册信息，去逐个遍历复制。通过Debug，发现 eureka-server2 确实接收到了来自 eureka-server1 的注册信息。

循环体中的核心方法 `batchResponse.addResponse(dispatch(instanceInfo));` ，这里面需要先 `dispatch` 后 `add` ，很明显 `dispatch` 是核心。

```java
private ReplicationInstanceResponse dispatch(ReplicationInstance instanceInfo) {
    ApplicationResource applicationResource = createApplicationResource(instanceInfo);
    InstanceResource resource = createInstanceResource(instanceInfo, applicationResource);

    String lastDirtyTimestamp = toString(instanceInfo.getLastDirtyTimestamp());
    String overriddenStatus = toString(instanceInfo.getOverriddenStatus());
    String instanceStatus = toString(instanceInfo.getStatus());

    Builder singleResponseBuilder = new Builder();
    switch (instanceInfo.getAction()) {
        case Register:
            singleResponseBuilder = handleRegister(instanceInfo, applicationResource);
            break;
        case Heartbeat:
            singleResponseBuilder = handleHeartbeat(serverConfig, resource, lastDirtyTimestamp, overriddenStatus, instanceStatus);
            break;
        case Cancel:
            singleResponseBuilder = handleCancel(resource);
            break;
        case StatusUpdate:
            singleResponseBuilder = handleStatusUpdate(instanceInfo, resource);
            break;
        case DeleteStatusOverride:
            singleResponseBuilder = handleDeleteStatusOverride(instanceInfo, resource);
            break;
    }
    return singleResponseBuilder.build();
}
```

这个 `dispatch` 方法干的事情很容易理解了，switch 结构中有5种动作（注册、心跳、注销、状态变更等），这里会根据其他 EurekaServer 节点上同步过来的信息来分别处理。到这里差不多也就可以理解 EurekaServer 的同步接收了。但有接收必定有发起，下面咱看看 eureka-server1 是如何把注册信息发给 eureka-server2 的。

#### 3.7.3 【集群注册】eureka-server1的同步动作发起

eureka-server2 的接收动作被触发，来源于 eureka-server1 的同步动作，借助IDEA的字符串搜索，发现同步动作来自于 `JerseyReplicationClient` ：

```java
public static final String BATCH_URL_PATH = "peerreplication/batch/";

public EurekaHttpResponse<ReplicationListResponse> submitBatchUpdates(ReplicationList replicationList) {
    ClientResponse response = null;
    try {
        response = jerseyApacheClient.resource(serviceUrl)
                .path(PeerEurekaNode.BATCH_URL_PATH)
                .accept(MediaType.APPLICATION_JSON_TYPE)
                .type(MediaType.APPLICATION_JSON_TYPE)
                .post(ClientResponse.class, replicationList);
        if (!isSuccess(response.getStatus())) {
            return anEurekaHttpResponse(response.getStatus(), ReplicationListResponse.class).build();
        }
        ReplicationListResponse batchResponse = response.getEntity(ReplicationListResponse.class);
        return anEurekaHttpResponse(response.getStatus(), batchResponse).type(MediaType.APPLICATION_JSON_TYPE).build();
    } finally {
        if (response != null) {
            response.close();
        }
    }
}
```

在这个动作中它将一组动作信息发送给远程的其他 EurekaServer 集群节点，而这个方法的调用在打断点Debug时发现调用栈只有3层，再往上是 `Runnable` 的 `run` 方法了，根据对前面源码的阅读，可以大胆猜测这是**线程调度触发**的。通过一层一层往上检索（这部分过程有点复杂，小册就不记录了，小伙伴们可以尝试检索一下），最终发现线程触发的调度器初始化在 `PeerEurekaNode` 的构造方法中：（节选）

```java
PeerEurekaNode(...............) { // 构造方法参数过多已省略
    // ......
    String batcherName = getBatcherName();
    ReplicationTaskProcessor taskProcessor = new ReplicationTaskProcessor(targetHost, replicationClient);
    this.batchingDispatcher = TaskDispatchers.createBatchingTaskDispatcher(
            batcherName,
            config.getMaxElementsInPeerReplicationPool(),
            batchSize,
            config.getMaxThreadsForPeerReplication(),
            maxBatchingDelayMs,
            serverUnavailableSleepTimeMs,
            retrySleepTimeMs,
            taskProcessor
    );
    // ......
}
```

在这里它使用 `TaskDispatchers` 创建了批量任务的定时任务调度器，而创建调度器的方法中有一句关键代码：`TaskExecutors.batchExecutors(id, workerCount, taskProcessor, acceptorExecutor);` ，它这里面创建了一个 `BatchWorkerRunnable` ，而这个 `BatchWorkerRunnable` 从类名上也知道它肯定实现了 `Runnable` 接口：

```java
static class BatchWorkerRunnable<ID, T> extends WorkerRunnable<ID, T> {

  @Override
  public void run() {
    try {
      while (!isShutdown.get()) {
        List<TaskHolder<ID, T>> holders = getWork();
        metrics.registerExpiryTimes(holders);

        List<T> tasks = getTasksOf(holders);
        ProcessingResult result = processor.process(tasks);
        switch (result) {
          case Success:
            break;
          case Congestion:
          case TransientError:
            taskDispatcher.reprocess(holders, result);
            break;
          case PermanentError:
            logger.warn("Discarding {} tasks of {} due to permanent error", holders.size(), workerName);
        }
        metrics.registerTaskResult(result, tasks.size());
      }
    } // catch ......
  }
```

这个 `run` 方法就是上面查方法调用栈的起点，至此也就知道了触发来源。继续往下走，核心的方法是这里面的 `processor.process(tasks);` ：（去掉日志）

```java
public ProcessingResult process(List<ReplicationTask> tasks) {
    ReplicationList list = createReplicationListOf(tasks);
    try {
        EurekaHttpResponse<ReplicationListResponse> response = replicationClient.submitBatchUpdates(list);
        int statusCode = response.getStatusCode();
        if (!isSuccess(statusCode)) {
            if (statusCode == 503) {
                return ProcessingResult.Congestion;
            } else {
                // Unexpected error returned from the server. This should ideally never happen.
                return ProcessingResult.PermanentError;
            }
        } else {
            handleBatchResponse(tasks, response.getEntity().getResponseList());
        }
    } // catch ......
    return ProcessingResult.Success;
}

```

try 结构的第一句就是发送请求，就是本节一上来看到的 `JerseyReplicationClient` 中的 `submitBatchUpdates` 方法。

捋清楚动作发起的来源，那这个线程的启动触发又是从哪里来的呢？咱继续往下看。

前面咱提到调度器的创建在 `PeerEurekaNode` 中，有创建就一定有使用，翻看 `PeerEurekaNode` 的方法列表，发现有一个方法叫 `register` ，这方法未免也太熟悉了吧，直接跳转过去。

```java
public void register(final InstanceInfo info) throws Exception {
    long expiryTime = System.currentTimeMillis() + getLeaseRenewalOf(info);
    batchingDispatcher.process(
            taskId("register", info),
            new InstanceReplicationTask(targetHost, Action.Register, info, null, true) {
                public EurekaHttpResponse<Void> execute() {
                    return replicationClient.register(info);
                }
            }, expiryTime
    );
}
```

等等，这不就是 EurekaClient 注册到 EurekaServer 的方法嘛，借助IDEA查看这个方法被使用的位置，发现来到了 `PeerAwareInstanceRegistryImpl` ：

```java
private void replicateInstanceActionsToPeers(Action action, String appName,
         String id, InstanceInfo info, InstanceStatus newStatus, PeerEurekaNode node) {
    try {
        InstanceInfo infoFromRegistry = null;
        CurrentRequestVersion.set(Version.V2);
        switch (action) {
            case Cancel:
                node.cancel(appName, id);
                break;
            case Heartbeat:
                InstanceStatus overriddenStatus = overriddenInstanceStatusMap.get(id);
                infoFromRegistry = getInstanceByAppAndId(appName, id, false);
                node.heartbeat(appName, id, infoFromRegistry, overriddenStatus, false);
                break;
            case Register:
                node.register(info);
                break;
            case StatusUpdate:
                infoFromRegistry = getInstanceByAppAndId(appName, id, false);
                node.statusUpdate(appName, id, newStatus, infoFromRegistry);
                break;
            case DeleteStatusOverride:
                infoFromRegistry = getInstanceByAppAndId(appName, id, false);
                node.deleteStatusOverride(appName, id, infoFromRegistry);
                break;
        }
    } // catch ......
}

```

注意中间的 switch-case 的第三个 case ：`node.register(info);` ，它就是**触发一个 EurekaServer 节点收到注册后广播给其他 EurekaServer 节点的方法**。而这个 `replicateInstanceActionsToPeers` 方法被调用的位置是这个类的 `replicateToPeers` ：

```java
private void replicateToPeers(Action action, String appName, String id,
         InstanceInfo info, InstanceStatus newStatus, boolean isReplication) {
    // ......
        for (final PeerEurekaNode node : peerEurekaNodes.getPeerEurekaNodes()) {
            // If the url represents this host, do not replicate to yourself.
            if (peerEurekaNodes.isThisMyUrl(node.getServiceUrl())) {
                continue;
            }
            replicateInstanceActionsToPeers(action, appName, id, info, newStatus, node);
        }
    } // ......
}
```

继续往上爬方法，调用 replicateToPeers 方法的上一层方法有5个位置：

这是注册的动作，自然咱去找 `Action.Register` 的那个方法咯：

```java
public void register(final InstanceInfo info, final boolean isReplication) {
    int leaseDuration = Lease.DEFAULT_DURATION_IN_SECS;
    if (info.getLeaseInfo() != null && info.getLeaseInfo().getDurationInSecs() > 0) {
        leaseDuration = info.getLeaseInfo().getDurationInSecs();
    }
    super.register(info, leaseDuration, isReplication);
    replicateToPeers(Action.Register, info.getAppName(), info.getId(), info, null, isReplication);
}
```

这个 `super.register` 方法就是咱在之前在第7章1.3节中看到的那个微服务实例注册的源码，那它这里就是给方法做了增强，这个增强的目的就是**保证在注册到本地 EurekaServer 的同时，还能一块注册到其他 EurekaServer 节点上**。（顺便还看到了 EurekaClient 向 EurekaServer 的部分节点上注册的联动同步效果）

而且根据前面的源码中也知道，`register` 方法中有对 `recentlyChangedQueue` 的添加，那也就知道 `recentlyChangedQueue` 真正被远程添加的原理了。

#### 3.7.4 recentlyChangedQueue中元素的清除

队列中的变动信息添加上之后，上面也提到了，看到了 `clean` 方法，但没看到别的方法移除元素，难不成真的只有一次性清除吗？那出现先后注册的问题，它又怎么确定的呢？这个时候咱不得不去翻一下 `recentlyChangedQueue` 所在类 `AbstractInstanceRegistry` 的成员中看看有没有什么踪迹可寻。当翻到构造方法的时候注意到一点：

```java
protected AbstractInstanceRegistry(EurekaServerConfig serverConfig, 
       EurekaClientConfig clientConfig, ServerCodecs serverCodecs) {
    // ......
    this.deltaRetentionTimer.schedule(getDeltaRetentionTask(),
            serverConfig.getDeltaRetentionTimerIntervalInMs(),
            serverConfig.getDeltaRetentionTimerIntervalInMs());
}
```

这里有一个定时器，而且这个定时器的名称有点意思：增量信息保留定时器。跳转到 `getDeltaRetentionTask` 方法，发现了 `recentlyChangedQueue` 的出队方式：

```java
private TimerTask getDeltaRetentionTask() {
    return new TimerTask() {
        @Override
        public void run() {
            Iterator<RecentlyChangedItem> it = recentlyChangedQueue.iterator();
            while (it.hasNext()) {
                if (it.next().getLastUpdateTime() <
                        System.currentTimeMillis() - serverConfig.getRetentionTimeInMSInDeltaQueue()) {
                    it.remove();
                } else {
                    break;
                }
            }
        }
    };
}
```

它使用**迭代器**来从前往后遍历队列并移除！因为队列中前面的实例变动信息发生的时间一定早于后面的变动信息，所以当循环遍历到时间差小于配置的最大保留时间时，就停止循环了，这个时候最近一段时间的实例变动信息就被保留下来了，之前的信息被删除。

## 小结

1. EurekaClient 的注册信息包括全量获取与增量获取，注册信息获取时会把注册中心一起获取到；
2. EurekaServer 在处理注册请求时会借助 `recentlyChangedQueue` 整合微服务实例变动情况，并由此支撑增量注册信息的获取；
3. `recentlyChangedQueue` 队列中保存了最近一段时间内微服务实例变动的所有动作记录，它会定时清除时间靠前的信息，只保留指定一段时间之内的变动记录。

