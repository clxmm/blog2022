---
title: 09-服务发现-EurekaServer的缓存机制原理
toc: true
tags: springcloud
categories: 
    - [java]
    - [springcloudNetflix]
---

Eureka 之所以是一个遵循 AP 原则的注册中心，内部设计的缓存结构肯定是不可或缺的，有了缓存、自我保护机制等特性，使得 EurekaServer 可以在集群中一些节点宕机时还可以让整体继续提供服务，而不至于因为像 Zookeeper 那样进行 Leader 的重新选举导致集群不可用。本章咱来详细看看 EurekaServer 是如何设计内部缓存的（其实上一章留了个 EurekaServer 的坑，正好这一章填上）。

## 0. 书接上回-EurekaServer处理注册信息获取请求

<!--more-->

来到 `ApplicationsResource` 类的 `getContainers` 方法：

```java
@GET                       // 参数太多已省略
public Response getContainers(.............) {
    // ......
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
        // ......
    } else {
        response = Response.ok(responseCache.get(cacheKey)).build();
    }
    return response;
}
```

上一章咱看到 Key 和 Value 的时候只是说了一下这是缓存机制，就过去了，这一章咱可得好好说道说道。

## 1. EurekaServer设计的缓存

上一章咱也看到了，实现响应缓存的核心是 `ResponseCache` ，它的实现类就是 `ResponseCacheImpl` ，它内部的成员特别多，不过核心基本都在构造方法中，咱来分析构造方法就成：

```java
ResponseCacheImpl(EurekaServerConfig serverConfig, ServerCodecs serverCodecs, AbstractInstanceRegistry registry) {
    this.serverConfig = serverConfig;
    this.serverCodecs = serverCodecs;
    // 配置是否开启读写缓存
    this.shouldUseReadOnlyResponseCache = serverConfig.shouldUseReadOnlyResponseCache();
    this.registry = registry;

    // 缓存更新的时间间隔，默认30秒
    long responseCacheUpdateIntervalMs = serverConfig.getResponseCacheUpdateIntervalMs();
    // 使用CacheBuilder构建读写缓存，构建时需要设置缓存加载器CacheLoader
    this.readWriteCacheMap =
            CacheBuilder.newBuilder().initialCapacity(serverConfig.getInitialCapacityOfResponseCache())
                    .expireAfterWrite(serverConfig.getResponseCacheAutoExpirationInSeconds(), TimeUnit.SECONDS)
                    // 缓存过期策略
                    .removalListener(new RemovalListener<Key, Value>() {
                        @Override
                        public void onRemoval(RemovalNotification<Key, Value> notification) {
                            Key removedKey = notification.getKey();
                            if (removedKey.hasRegions()) {
                                Key cloneWithNoRegions = removedKey.cloneWithoutRegions();
                                regionSpecificKeys.remove(cloneWithNoRegions, removedKey);
                            }
                        }
                    })
                    // 缓存加载策略
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

    // 配置是否开启只读缓存，如果开启则会启用定时任务，将读写缓存的数据定期同步到只读缓存
    if (shouldUseReadOnlyResponseCache) {
        timer.schedule(getCacheUpdateTask(),
                new Date(((System.currentTimeMillis() / responseCacheUpdateIntervalMs) * responseCacheUpdateIntervalMs)
                        + responseCacheUpdateIntervalMs),
                responseCacheUpdateIntervalMs);
    }

    try {
        Monitors.registerObject(this);
    } // catch ......
}
```

可以发现 EurekaServer 中的缓存包含两个部分：**读写缓存**、**只读缓存**，下面咱一一来看。

### 1.1 读写缓存readWriteCacheMap

从设计上将，读写缓存很明显是拿来做实时数据存储的。注意它的类型：

```java
private final LoadingCache<Key, Value> readWriteCacheMap;

```

这个 `LoadingCache` 类型来自于 Google 的 **Guava** 包，研究 `readWriteCacheMap` 其实就是来研究 Guava 中的 `LoadingCache` 。咱分别就写与读两方面来解释。

#### 1.1.1 readWriteCacheMap的更新时机

借助IDEA一步一步向上追踪 `readWriteCacheMap` 的使用，最终可以找到这样一个位置，很明显它是更新缓存的核心

![](./img/2023/05/cloud09-1.png)

一步一步追踪到 `invalidate` 方法，再向上就到 `AbstractInstanceRegistry` 中了，继续往上爬：

![](./img/2023/05/cloud09-2.png)

四个位置分别代表：服务注册、状态更新、服务下线、服务覆盖状态移除。由此可以知道，读写缓存的更新位置是在这四种时机上触发。

#### 1.1.2 readWriteCacheMap的读取和加载

咱上面也知道，`readWriteCacheMap` 的类型是 `LocalCache` ，它的获取数据方式自然是 `get` ：

##### 1.1.2.1 LocalCache#get

```java
final LocalCache<K, V> localCache;

public V get(K key) throws ExecutionException {
    return localCache.getOrLoad(key);
}
```

可以发现它内部是组合了一个一样类型的 `LocalCache` ，根据之前咱看SpringFramework 中 `ApplicationContext` 与 `BeanFactory` 的设计，大概可以意识到它也是用的代理。那咱直接跳到 `getOrLoad` 中：

##### 1.1.2.2 getOrLoad

```java
V getOrLoad(K key) throws ExecutionException {
    return get(key, defaultLoader);
}

V get(K key, CacheLoader<? super K, V> loader) throws ExecutionException {
    int hash = hash(checkNotNull(key));
    return segmentFor(hash).get(key, hash, loader);
}
```

到这里可以发现 `get` 的实际操作是两部分，而且这个操作很容易让咱联想到 `HashMap` ：先根据 key 计算一个 hash 值，再去取值！Hash 的操作咱就不关心了，咱可以大概理解为“获取到的是整个哈希表的一个片段/部分”（其实就可以理解是分段锁嘛），一个片段就是一个 `LocalCache.Segment` ，咱还是重点看 `get` 动作：（这段源码比较复杂，核心注释已标注在源码中）

##### 1.1.2.3 LocalCache.Segment#get

```java
V get(K key, int hash, CacheLoader<? super K, V> loader) throws ExecutionException {
    checkNotNull(key);
    checkNotNull(loader);
    try {
        // 如果count不等于0，说明当前Segment中已经有数据了
        if (count != 0) { // read-volatile
            // don't call getLiveEntry, which would ignore loading values
            ReferenceEntry<K, V> e = getEntry(key, hash);
            // 这一步已经取到缓存中的值了，但还要做过期检查（惰性过期）
            if (e != null) {
                long now = map.ticker.read();
                V value = getLiveValue(e, now);
                if (value != null) {
                    recordRead(e, now);
                    statsCounter.recordHits(1);
                    return scheduleRefresh(e, key, hash, value, now, loader);
                }
                ValueReference<K, V> valueReference = e.getValueReference();
                if (valueReference.isLoading()) {
                    return waitForLoadingValue(e, key, valueReference);
                }
            }
        }

        // at this point e is either null or expired;
        // 没有取到数据，进入真正的数据加载部分
        return lockedGetOrLoad(key, hash, loader);
    } // catch&finally ......
}
```

可以看到基本的加载逻辑：如果能获取到指定 key 的 value ，则进行缓存过期检查和取出；如果没有获取，要进入下面的加锁获取逻辑。继续往下看：

##### 1.1.2.4 LocalCache.Segment#lockedGetOrLoad

源码中前面的大段内容都是与内部数据结构处理有关，咱不多关心，咱主要是看数据是怎么通过 `CacheLoader` 加载生成的：

```java
V lockedGetOrLoad(K key, int hash, CacheLoader<? super K, V> loader)
        throws ExecutionException {
    // ......

    if (createNewEntry) {
        try {
            // Synchronizes on the entry to allow failing fast when a recursive load is
            // detected. This may be circumvented when an entry is copied, but will fail fast most
            // of the time.
            synchronized (e) {
                // 异步加载数据
                return loadSync(key, hash, loadingValueReference, loader);
            }
        } finally {
            statsCounter.recordMisses(1);
        }
    } else {
        // The entry already exists. Wait for loading.
        return waitForLoadingValue(e, key, valueReference);
    }
}

```

##### 1.1.2.5 loadSync：异步加载数据

```java
V loadSync(K key, int hash, LoadingValueReference<K, V> loadingValueReference,
           CacheLoader<? super K, V> loader) throws ExecutionException {
    ListenableFuture<V> loadingFuture = loadingValueReference.loadFuture(key, loader);
    return getAndRecordStats(key, hash, loadingValueReference, loadingFuture);
}

public ListenableFuture<V> loadFuture(K key, CacheLoader<? super K, V> loader) {
    stopwatch.start();
    V previousValue = oldValue.get();
    try {
        if (previousValue == null) {
            V newValue = loader.load(key);
            return set(newValue) ? futureValue : Futures.immediateFuture(newValue);
        }
        // ......
    } // catch ......
}

```

这一部分咱找到真正的加载数据的执行位置，它将 `CacheLoader` 传入，并在 `loadFuture` 方法中真正调用了 `CacheLoader` 的 `load` 方法，完成数据的加载。

至于 `readWriteCacheMap` 是如何根据指定的 `key` 加载 `value` ，咱在上一章的 3.5 节已经解释过了（借助 `InstanceRegistry` 发起全量/增量获取），这里就不再展开赘述。

##### 1.1.2.6 数据回存

数据加载出来后，在上面的 `loadSync` 中看到了还有第二步 `getAndRecordStats` ：

```java
 getAndRecordStats(K key, int hash, LoadingValueReference<K, V> loadingValueReference,
        ListenableFuture<V> newValue) throws ExecutionException {
    V value = null;
    try {
        value = getUninterruptibly(newValue);
        if (value == null) {
            throw new InvalidCacheLoadException("CacheLoader returned null for key " + key + ".");
        }
        // 性能信息记录
        statsCounter.recordLoadSuccess(loadingValueReference.elapsedNanos());
        // 这一步是数据回存
        storeLoadedValue(key, hash, loadingValueReference, value);
        return value;
    } finally {
        if (value == null) {
            statsCounter.recordLoadException(loadingValueReference.elapsedNanos());
            removeLoadingValue(key, hash, loadingValueReference);
        }
    }
}
```

这个方法的执行意图算是比较清晰的，它会把真正的值取出，并做好性能监测记录，随后保存到缓存中，将值返回，结束。由于数据回存的步骤更像是 `ConcurrentHashMap` 的 `put` 操作（并发集合的存储），内部很复杂，有兴趣的小伙伴可以点进去看看，小册这里就不再展开。

以上咱了解了关于读写缓存 `readWriteCacheMap` 的设计和使用，下面咱看只读缓存 `readOnlyCacheMap` 的设计和使用。

### 1.2 只读缓存readOnlyCacheMap

与 `readWriteCacheMap` 的类型不同，`readOnlyCacheMap` 的类型仅为 `ConcurrentHashMap` ：

```java
private final ConcurrentMap<Key, Value> readOnlyCacheMap = new ConcurrentHashMap<>();

```

与上面对比，咱能意识到一点：只读缓存里的数据是不存在数据有效期的！上面也看了，每次服务实例的状态发生变动时，都是只影响 `readWriteCacheMap` ，不影响 `readOnlyCacheMap` ，所以会出现**可用性强但数据可能不强一致**的情况（说到底 Eureka 设计符合AP而不是CP）。

#### 1.2.1 缓存同步

在 `ResponseCacheImpl` 的构造方法中咱提到，如果开启只读缓存，则会使用一个定时任务来同步读写缓存中的数据：

```java
// 配置是否开启只读缓存，如果开启则会启用定时任务，将读写缓存的数据定期同步到只读缓存
if (shouldUseReadOnlyResponseCache) {
  timer.schedule(getCacheUpdateTask(),
                 new Date(((System.currentTimeMillis() / responseCacheUpdateIntervalMs) * responseCacheUpdateIntervalMs)
                          + responseCacheUpdateIntervalMs),
                 responseCacheUpdateIntervalMs);
}
```

执行的定时任务就是 `getCacheUpdateTask()` ：

#### 1.2.2 getCacheUpdateTask

```java
private TimerTask getCacheUpdateTask() {
    return () -> {
        logger.debug("Updating the client cache from response cache");
        for (Key key : readOnlyCacheMap.keySet()) {
            // logger
            try {
                CurrentRequestVersion.set(key.getVersion());
                Value cacheValue = readWriteCacheMap.get(key);
                Value currentCacheValue = readOnlyCacheMap.get(key);
                if (cacheValue != currentCacheValue) {
                    readOnlyCacheMap.put(key, cacheValue);
                }
            } // catch ......
        }
    };
}

```

小伙伴们看到这里可能会疑惑，为什么这里只有 readOnlyCacheMap 中已经存在的 key 作更新，那些新加入的 key 怎么办呢？其实这个问题在上面的 readWriteCacheMap 中能找到答案：

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
    // ......
}
```

如果在 `readOnlyCacheMap` 中取不到数据，会从 `readWriteCacheMap` 中取数据，并设置到 `readOnlyCacheMap` 中，这样就保证两个缓存中的 key 保持一致了。

以上咱了解了 EurekaServer 中对于缓存的设计，那这个缓存又是如何运用在全量获取中的，咱可以来尝试整理总结一下。

## 2. 全量获取中的响应缓存处理总结

梳理一下思路哈，咱想一想整个流程是什么样的。假设有一个 EurekaServer 与两个 EurekaClient，咱分别起名为 eureka-server 、client1 、client2 ，其中呢eureka-server 为先行启动的服务器，client1 已经注册到 eureka-server上，现在 client2 要启动并注册到 eureka-server 上。整体流程如下：

1. client2 被启动，内部创建 `DiscoveryClient` ；
2. `DiscoveryClient` 的 `fetchRegistryFromBackup` 方法被触发，执行注册信息的全量获取；
   - `DiscoveryClient` 中的方法调用链：`fetchRegistryFromBackup` → `fetchRegistry` → `getAndStoreFullRegistry`
3. eureka-server 接收到 client2 的全量获取请求，在 EurekaServer 内部接收请求的是 `ApplicationsResource` 的 `getContainer` 方法；
4. EurekaServer 根据同步的类型、范围等信息收集好，从 `readOnlyCacheMap` 中尝试获取全量注册信息；
5. `readOnlyCacheMap` 中获取不到对应的全量信息，转而交由 `readWriteCacheMap` 获取；
6. `readWriteCacheMap` 也获取不到全量信息，触发 `ResponseCacheImpl` 中的 `generatePayload` 方法，向 `InstanceRegistry` 发起全量获取请求；
7. `ResponseCacheImpl` 拿到全量信息后，放入自身的 `readWriteCacheMap` 与 `readOnlyCacheMap` 中，返回全量信息；
8. `DiscoveryClient` 接收到全量信息，放入自身的 `localRegionApps` 中，全量获取结束。

## 小结

1. EurekaServer 的内部缓存包括两部分：读写缓存、只读缓存；
2. 读写缓存中的数据在添加时会同时放入只读缓存；
3. 默认情况下，只读缓存每隔一段时间会同步读写缓存中最新的数据。

【EurekaServer 的缓存机制到这儿基本就解释完毕了，还剩下一些边边角角的内容，接下来咱看一个小伙伴们都比较熟知的现象：Eureka 心跳】



 