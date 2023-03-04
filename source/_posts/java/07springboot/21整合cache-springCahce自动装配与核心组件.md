---
title: 21整合cache-springCahce自动装配与核心组件
toc: true
tags: springboot
categories: 
    - [java]
    - [springboot]
---

好了，到此为止，我们已经对 SpringCache 的使用有了一个比较全面和系统的了解。本章我们就按照惯例深入源码，探究 SpringCache 的底层原理，以及其注册的核心组件。

## 1. 最初始自动装配

---

按照自动装配的套路来讲，与缓存直接相关的自动配置类必定是 `CacheAutoConfiguration` 了（从 `spring.factories` 中也没有找到更多的缓存相关的自动配置类了）！借助 IDE 寻找，发现真的有这个类：

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(CacheManager.class)
@ConditionalOnBean(CacheAspectSupport.class)
@ConditionalOnMissingBean(value = CacheManager.class, name = "cacheResolver")
@EnableConfigurationProperties(CacheProperties.class)
@AutoConfigureAfter({ CouchbaseDataAutoConfiguration.class, HazelcastAutoConfiguration.class,
		HibernateJpaAutoConfiguration.class, RedisAutoConfiguration.class })
@Import({ CacheConfigurationImportSelector.class, 
         CacheManagerEntityManagerFactoryDependsOnPostProcessor.class })
public class CacheAutoConfiguration {
  
}
```

<!--more-->

除去上方一大堆的 `@Conditional` 系注解和自动装配排序之外，最关键的是用 `@Import` 注册了一个 `CacheConfigurationImportSelector` 。

### 1.1 CacheConfigurationImportSelector

这个套路我们简直不要太熟悉了！自动装配就是导入 `ImportSelector` 或者 `ImportBeanDefinitionRegistrar` 来实现组件注册 / 配置类导入的！那这个 `CacheConfigurationImportSelector` 都做了什么呢？

```java
	static class CacheConfigurationImportSelector implements ImportSelector {

		@Override
		public String[] selectImports(AnnotationMetadata importingClassMetadata) {
			CacheType[] types = CacheType.values();
			String[] imports = new String[types.length];
			for (int i = 0; i < types.length; i++) {
				imports[i] = CacheConfigurations.getConfigurationClass(types[i]);
			}
			return imports;
		}

	}
```

可以发现，它导入的是一组 ConfigurationClass ，也就是配置类！而支持的配置类是从一个 `CacheType` 枚举中提取的！而具体的配置类名称加载动作，是借助了一个 `CacheConfigurations` 工具类：

```java
    private static final Map<CacheType, String> MAPPINGS;

    static {
        Map<CacheType, String> mappings = new EnumMap<>(CacheType.class);
        mappings.put(CacheType.GENERIC, GenericCacheConfiguration.class.getName());
        mappings.put(CacheType.EHCACHE, EhCacheCacheConfiguration.class.getName());
        mappings.put(CacheType.HAZELCAST, HazelcastCacheConfiguration.class.getName());
        mappings.put(CacheType.INFINISPAN, InfinispanCacheConfiguration.class.getName());
        mappings.put(CacheType.JCACHE, JCacheCacheConfiguration.class.getName());
        mappings.put(CacheType.COUCHBASE, CouchbaseCacheConfiguration.class.getName());
        mappings.put(CacheType.REDIS, RedisCacheConfiguration.class.getName());
        mappings.put(CacheType.CAFFEINE, CaffeineCacheConfiguration.class.getName());
        mappings.put(CacheType.SIMPLE, SimpleCacheConfiguration.class.getName());
        mappings.put(CacheType.NONE, NoOpCacheConfiguration.class.getName());
        MAPPINGS = Collections.unmodifiableMap(mappings);
    }

    static String getConfigurationClass(CacheType cacheType) {
        String configurationClassName = MAPPINGS.get(cacheType);
        Assert.state(configurationClassName != null, () -> "Unknown cache type " + cacheType);
        return configurationClassName;
    }
```

好家伙，这个工具类把 `CacheType` 的所有枚举类型都逐个进行了对应配置类的关系映射。当需要取出某个枚举对应的配置类时，只需要从 Map 中获取即可。

```java
package org.springframework.boot.autoconfigure.cache;

public enum CacheType {
    GENERIC,
    JCACHE,
    EHCACHE,
    HAZELCAST,
    INFINISPAN,
    COUCHBASE,
    REDIS,
    CAFFEINE,
    SIMPLE,
    NONE;

    private CacheType() {
    }
}
```

所以我们可以这样理解，`CacheConfigurationImportSelector` 完成的功能就是**加载了上述源码中标注的所有缓存相关的配置类**。

### 1.2 CacheManagerCustomizers

```java
public class CacheAutoConfiguration {

	@Bean
	@ConditionalOnMissingBean
	public CacheManagerCustomizers cacheManagerCustomizers(ObjectProvider<CacheManagerCustomizer<?>> customizers) {
		return new CacheManagerCustomizers(customizers.orderedStream().collect(Collectors.toList()));
	}
```

除了导入的核心 `ImportSelector` 之外，在 `CacheAutoConfiguration` 中还有一个比较有趣的组件：`CacheManagerCustomizers` ，从类名上就能看得出来，它是一个定制器的组合体，从 `@Bean` 方法传入的参数就能看出，它可以将 IOC 容器中所有的 `CacheManagerCustomizer` 收集起来，合并最终形成一个 `CacheManagerCustomizers` 对象。而 **Customizer** 是 SpringBoot 用来编程式定制组件配置的重要方式之一，通过向 IOC 容器中注册 `CacheManagerCustomizers` ，底层自动装配会收集这些定制器，并且在合适的位置予以回调，从而完成 Bean 组件的编程式定制化。

请注意，定制器不同于上一章的 `RedisCacheManager` 那样根据缓存区定制，**Customizer** 机制会应用于 IOC 容器中注册的所有被定制组件，也就是说，我们注册的所有组件，都会被 **Customizer** 予以处理。这一点一定要区分开。

## 2. 基础缓存模型

----

即便我们不引入任何的第三方组件，SpringCache 依然能生效，其底层一定离不开 jdk 中最原始的能体现缓存的容器 —— `Map` 。那底层 SpringBoot 又是如何帮我们把 Map 型的缓存装配生效的呢？

### 2.1 SimpleCacheConfiguration

仔细观察上方的源码中，每种缓存类型映射的配置类，可以发现一堆配置类中，只有两个看上去最有可能，分别是 **GENERIC** 和 **SIMPLE** 。

那么最终谁能生效呢？我们不妨点进这些类的内部，看一下这些类上有没有什么特殊的标记。

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnBean(Cache.class)
@ConditionalOnMissingBean(CacheManager.class)
@Conditional(CacheCondition.class)
class GenericCacheConfiguration {

	@Bean
	SimpleCacheManager cacheManager(CacheManagerCustomizers customizers, Collection<Cache> caches) {
		SimpleCacheManager cacheManager = new SimpleCacheManager();
		cacheManager.setCaches(caches);
		return customizers.customize(cacheManager);
	}

}
```

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnMissingBean(CacheManager.class)
@Conditional(CacheCondition.class)
class SimpleCacheConfiguration {

	@Bean
	ConcurrentMapCacheManager cacheManager(CacheProperties cacheProperties,
			CacheManagerCustomizers cacheManagerCustomizers) {
		ConcurrentMapCacheManager cacheManager = new ConcurrentMapCacheManager();
		List<String> cacheNames = cacheProperties.getCacheNames();
		if (!cacheNames.isEmpty()) {
			cacheManager.setCacheNames(cacheNames);
		}
		return cacheManagerCustomizers.customize(cacheManager);
	}

}
```

由这些条件装配的注解，可以很明显地分辨出来，当我们手动注册过 Cache 对象时，这两个配置类都会激活；而没有手动注册过 Cache 对象时，则只有 `SimpleCacheConfiguration` 会被注册。因为我们引入 `spring-boot-starter-cache` 依赖时并没有进行任何操作，也没有编写任何多余的代码，很明显能生效的是 `SimpleCacheConfiguration` 。

下面我们看一下 `SimpleCacheConfiguration` 中注册的组件。大多数的 `CacheConfiguration` 中都是只会注册一个组件，那就是全局可用的 `CacheManager` 。

```java
class SimpleCacheConfiguration {

    @Bean
    ConcurrentMapCacheManager cacheManager(CacheProperties cacheProperties,
            CacheManagerCustomizers cacheManagerCustomizers) {
        ConcurrentMapCacheManager cacheManager = new ConcurrentMapCacheManager();
        List<String> cacheNames = cacheProperties.getCacheNames();
        if (!cacheNames.isEmpty()) {
            cacheManager.setCacheNames(cacheNames);
        }
        return cacheManagerCustomizers.customize(cacheManager);
    }
}
```

### 2.2 ConcurrentMapCacheManager

`SimpleCacheConfiguration` 中注册的 `CacheManager` 类型为 `ConcurrentMapCacheManager` 。从类名上我们就能很容易地了解，它的底层实现是通过线程安全的 `Map` 。

```java
public class ConcurrentMapCacheManager implements CacheManager, BeanClassLoaderAware {

    private final ConcurrentMap<String, Cache> cacheMap = new ConcurrentHashMap<>(16);

    protected Cache createConcurrentMapCache(String name) {
        SerializationDelegate actualSerialization = (isStoreByValue() ? this.serialization : null);
        return new ConcurrentMapCache(name, new ConcurrentHashMap<>(256), isAllowNullValues(), actualSerialization);
    }
```

从 `ConcurrentMapCacheManager` 的源码中可以看出，它使用了两层 `ConcurrentMap` 实现了最基础的缓存模型。外层的 `ConcurrentMap<String, Cache> cacheMap` 负责保存所有的 `Cache` 组件，而每个 `Cache` 的组件本质上又是一个 `ConcurrentHashMap` 。

### 2.3 ConcurrentMapCache的存取

具体到 `ConcurrentMapCache` 的缓存数据维护上，我们来关注它的保存数据和提取数据。由于底层是基于 `ConcurrentMap` ，所以 `ConcurrentMapCache` 的存取动作更像是 `Map` 的套壳，本质上操作缓存容器还是在操作 `Map` 。

```java
public class ConcurrentMapCache extends AbstractValueAdaptingCache {

    private final String name;

    private final ConcurrentMap<Object, Object> store;

    @Override
    @Nullable
    protected Object lookup(Object key) {
        return this.store.get(key);
    }
    
    @Override
    public void put(Object key, @Nullable Object value) {
        this.store.put(key, toStoreValue(value));
    }
```

从源码的角度看来，`ConcurrentMapCache` 的实现也是相当的简单，小册就不展开啰嗦了。

## 3. 引入Redis后的装配

------

相比较于原生 Map 的缓存，当项目中引入 Redis 后，从生效的配置类到缓存容器都发生了改变，我们一样一样来看。

```java
Configuration(proxyBeanMethods = false)
@ConditionalOnClass(RedisConnectionFactory.class)
@AutoConfigureAfter(RedisAutoConfiguration.class)
@ConditionalOnBean(RedisConnectionFactory.class)
@ConditionalOnMissingBean(CacheManager.class)
@Conditional(CacheCondition.class)
class RedisCacheConfiguration {

	@Bean
	RedisCacheManager cacheManager(CacheProperties cacheProperties, CacheManagerCustomizers cacheManagerCustomizers,
			ObjectProvider<org.springframework.data.redis.cache.RedisCacheConfiguration> redisCacheConfiguration,
			ObjectProvider<RedisCacheManagerBuilderCustomizer> redisCacheManagerBuilderCustomizers,
			RedisConnectionFactory redisConnectionFactory, ResourceLoader resourceLoader) {
		RedisCacheManagerBuilder builder = RedisCacheManager.builder(redisConnectionFactory).cacheDefaults(
				determineConfiguration(cacheProperties, redisCacheConfiguration, resourceLoader.getClassLoader()));
		List<String> cacheNames = cacheProperties.getCacheNames();
		if (!cacheNames.isEmpty()) {
			builder.initialCacheNames(new LinkedHashSet<>(cacheNames));
		}
		if (cacheProperties.getRedis().isEnableStatistics()) {
			builder.enableStatistics();
		}
		redisCacheManagerBuilderCustomizers.orderedStream().forEach((customizer) -> customizer.customize(builder));
		return cacheManagerCustomizers.customize(builder.build());
	}

	private org.springframework.data.redis.cache.RedisCacheConfiguration determineConfiguration(
			CacheProperties cacheProperties,
			ObjectProvider<org.springframework.data.redis.cache.RedisCacheConfiguration> redisCacheConfiguration,
			ClassLoader classLoader) {
		return redisCacheConfiguration.getIfAvailable(() -> createConfiguration(cacheProperties, classLoader));
	}

	private org.springframework.data.redis.cache.RedisCacheConfiguration createConfiguration(
			CacheProperties cacheProperties, ClassLoader classLoader) {
		Redis redisProperties = cacheProperties.getRedis();
		org.springframework.data.redis.cache.RedisCacheConfiguration config = org.springframework.data.redis.cache.RedisCacheConfiguration
				.defaultCacheConfig();
		config = config.serializeValuesWith(
				SerializationPair.fromSerializer(new JdkSerializationRedisSerializer(classLoader)));
		if (redisProperties.getTimeToLive() != null) {
			config = config.entryTtl(redisProperties.getTimeToLive());
		}
		if (redisProperties.getKeyPrefix() != null) {
			config = config.prefixCacheNameWith(redisProperties.getKeyPrefix());
		}
		if (!redisProperties.isCacheNullValues()) {
			config = config.disableCachingNullValues();
		}
		if (!redisProperties.isUseKeyPrefix()) {
			config = config.disableKeyPrefix();
		}
		return config;
	}

}
```

与 `SimpleCacheConfiguration` 类似，`RedisCacheConfiguration` 中也只是注册了一个全局可用的 `RedisCacheManager` ，如果工程中没有再注册其他的 `RedisCacheManager` ，那么所有的缓存都会走这个默认注册的 `RedisCacheManager` 。

除此之外，我想请各位小伙伴观察构造 `RedisCacheManager` 中的一个细节：在使用 `RedisCacheManager` 的建造器构造过程中，有一个 `cacheDefaults` 方法的指定，它调用的是 `determineConfiguration` 方法，传入的参数包含 `CacheProperties` 和 `ObjectProvider<RedisCacheConfiguration>` ，这个设计有何用意呢？我们来到 `determineConfiguration` 方法中一探究竟。（为保证阅读体验，源码中已将全限定类名省略）

```java
	private org.springframework.data.redis.cache.RedisCacheConfiguration determineConfiguration(
			CacheProperties cacheProperties,
			ObjectProvider<org.springframework.data.redis.cache.RedisCacheConfiguration> redisCacheConfiguration,
			ClassLoader classLoader) {
		return redisCacheConfiguration.getIfAvailable(() -> createConfiguration(cacheProperties, classLoader));
	}
```

方法体的逻辑非常简单，它借助 `ObjectProvider` 的特性，在我们没有向 IOC 容器注册 `RedisCacheConfiguration` 对象时，使用 `CacheProperties` 的配置内容初始化一个相应的 `RedisCacheConfiguration` 对象。简单地说，它还是约定大于配置的体现，只要约定了使用 `application.properties` 的内容配置，就不再需要我们手动配置 `RedisCacheConfiguration` 了。

同时这个方法的实现也给我们提供了一个信息：如果我们需要定制 `RedisCacheConfiguration` 的内容时，除了可以修改 `application.properties` 的配置之外，对于一些不能在配置文件中修改的配置，我们还可以直接向 IOC 容器中注册 `RedisCacheConfiguration` 对象，以覆盖配置文件的内容。

### 3.2 RedisCacheManager

`RedisCacheManager` 对标的是 `ConcurrentMapCacheManager` ，它的内部集成了一个 `RedisCacheWriter` 完成缓存数据的写入（注意不是 `RedisTemplate` ）。`RedisCacheManager` 的核心作用是生产和管理 `RedisCache` ，我们可以通过源码来大体了解下 `RedisCache` 是如何被创建和管理的。

##### 创建

我们先从创建缓存开始，创建缓存的时候会指定缓存名称，以及传入 `RedisCacheWriter` 和 `RedisCacheConfiguration` ，这也告诉我们 `CacheManager` 的配置会应用在具体的 `Cache` 对象中。

```java
	protected RedisCache createRedisCache(String name, @Nullable RedisCacheConfiguration cacheConfig) {
		return new RedisCache(name, cacheWriter, cacheConfig != null ? cacheConfig : defaultCacheConfig);
	}
```

##### 获取

那创建缓存容器的方法有了，那这个方法会在什么时机下触发呢？很简单，肯定是缓存容器不存在的时候才会触发吧（不然已经有缓存的话为什么还会创建呢）。我们借助 IDE 翻看一下这个方法的引用，果然有一个非常符合我们猜测的方法：`getMissingCache` 。

```java
protected RedisCache getMissingCache(String name) {
  return this.allowInFlightCacheCreation ? this.createRedisCache(name, this.defaultCacheConfig) : null;
}
```

那继续往上找，哪个方法会尝试获取一个不存在的缓存呢？肯定是获取缓存吧！于是我们就找到了 `CacheManager` 的核心方法：`getCache` 。（注意，`getCache` 方法的实现在父类 `AbstractCacheManager` 中）

```java
    private final ConcurrentMap<String, Cache> cacheMap = new ConcurrentHashMap<>(16);

    public Cache getCache(String name) {
        // 先从cacheMap中获取
        Cache cache = this.cacheMap.get(name);
        if (cache != null) {
            return cache;
        }

        // cacheMap中没有，再尝试获取和创建，并缓存到cacheMap中
        Cache missingCache = getMissingCache(name);
        if (missingCache != null) {
            // Fully synchronize now for missing cache registration
            synchronized (this.cacheMap) {
                cache = this.cacheMap.get(name);
                if (cache == null) {
                    cache = decorateCache(missingCache);
                    this.cacheMap.put(name, cache);
                    updateCacheNames(name);
                }
            }
        }
        return cache;
    }
```

`getCache` 的方法实现其实是很简单的，这很类似于 `Map` 的 `computeIfAbsent` 方法。

这样下来一趟，有关 `RedisCacheManager` 的核心功能我们就已经看完了，总体难度还是不大的。

### 3.3 RedisCache

本章的最后我们来看看 `RedisCache` 的核心内容。具体的缓存容器中我们还是来关心数据是如何保存到 Redis ，以及如何从 Redis 中取出数据。

#### 缓存数据

缓存数据的本质是把数据存放到 Redis 中。注意在这里面有两个细节。

```java
public void put(Object key, @Nullable Object value) {
    Object cacheValue = preProcessCacheValue(value);
    if (!isAllowNullValues() && cacheValue == null) {
         // throw ex ......
    }
    // 缓存数据时会将key和value进行处理
    cacheWriter.put(name, createAndConvertCacheKey(key), serializeCacheValue(cacheValue), cacheConfig.getTtl());
}
```

##### key的转化

还记得上一章中我们看到 的 key 前缀设计吗？它默认会把当前缓存的 `name` 挂在缓存 key 的前面，而这里面的逻辑实现就是上面 `put` 方法的 `createAndConvertCacheKey` 方法调用。

我们结合下面的源码来一步一步看。从 `createAndConvertCacheKey` 方法一路向下调用，在 `createCacheKey` 方法中会有一个 `usePrefix` 的判断（这也就是前缀是否添加的拦截判断），之后到了拼接 key 前缀的时候，它最终会使用一个 `CacheKeyPrefix` 来拼接实际的 key 前缀。

```java
    private byte[] createAndConvertCacheKey(Object key) {
        return this.serializeCacheKey(this.createCacheKey(key));
    }

	protected String createCacheKey(Object key) {

		String convertedKey = convertKey(key);

		if (!cacheConfig.usePrefix()) {
			return convertedKey;
		}

		return prefixCacheKey(convertedKey);
	}


	private String prefixCacheKey(String key) {

		// allow contextual cache names by computing the key prefix on every call.
		return cacheConfig.getKeyPrefixFor(name) + key;
	}


	private final CacheKeyPrefix keyPrefix;
	public String getKeyPrefixFor(String cacheName) {

		Assert.notNull(cacheName, "Cache name must not be null!");

		return keyPrefix.compute(cacheName);
	}
```

为什么源码到这里要断开一下呢？是因为接下来的代码读起来会稍稍有些怪。`CacheKeyPrefix` 本身是一个函数式接口，它只有一个 `compute` 方法，如果我们借助 IDE 去找实现，会发现 IDE 并不会给我们提示，而其实 `compute` 方法的实现就是下面两个 static 方法，一个是拿 `缓存的名称 + 两个冒号` 当前缀，另一个是拿 `全局统一前缀 + 缓存名称 + 两个冒号` 当前缀。

```java
@FunctionalInterface
public interface CacheKeyPrefix {

    String SEPARATOR = "::";

    String compute(String cacheName);

    static CacheKeyPrefix simple() {
        return name -> name + SEPARATOR;
    }

    static CacheKeyPrefix prefixed(String prefix) {
        Assert.notNull(prefix, "Prefix must not be null!");
        return name -> prefix + name + SEPARATOR;
    }
}
```

所以看到这里，各位就已经能知道默认情况下缓存 key 的前缀到底是怎么来的了吧！底层源码一环套一环，这种单一职责的原则真是体现的淋漓尽致！

##### value的序列化

在上一章中我们知道，对于实体模型类这种复杂对象来讲，在保存到 Redis 中需要序列化，所以在 `RedisCache` 中在 `put` 方法的设计时，对要缓存的数据进行了缓存（也就是 `put` 方法中最后一行的 `serializeCacheValue` 方法）。

```java
	protected byte[] serializeCacheValue(Object value) {

		if (isAllowNullValues() && value instanceof NullValue) {
			return BINARY_NULL_VALUE;
		}

		return ByteUtils.getBytes(cacheConfig.getValueSerializationPair().write(value));
	}
```

注意看最后一行，它使用了一个 `ValueSerializationPair` 把需要缓存的数据转换成了 `ByteBuffer` ，进而转换为一个字节数组。那这个 `ValueSerializationPair` 是个啥呢？

回到我们上一章中写的那个自定义配置类：

```java
    @Bean
    public RedisCacheConfiguration redisCacheConfiguration() {
        return RedisCacheConfiguration.defaultCacheConfig().serializeValuesWith(
                RedisSerializationContext.SerializationPair.fromSerializer(RedisSerializer.json()));
    }
```

各位还记得吧，我们为了让序列化后的数据变得更容易阅读，我们把 jdk 原生的序列化机制改为了 json 序列化，而这里面用到的配置是 `serializeValuesWith()` 方法，这个方法传入的就是一个 `ValueSerializationPair` ！所以其实 value 的序列化就是把我们设置好的 json 序列化器拿到，然后去序列化缓存数据！

#### 取出数据

```java
public <T> T get(Object key, Callable<T> valueLoader) {
    ValueWrapper result = get(key);
    if (result != null) {
        return (T) result.get();
    }
    // ......
}

public ValueWrapper get(Object key) {
    return toValueWrapper(lookup(key));
}

protected Object lookup(Object key) {
    byte[] value = cacheWriter.get(name, createAndConvertCacheKey(key));
    if (value == null) {
        return null;
    }
    return deserializeCacheValue(value);
}
```

有了上面缓存数据的源码阅读后，再看取出数据的源码就非常容易了吧！这里面唯一的一点不同，是缓存数据的时候是序列化数据，取出的时候是**反序列化**。其他涉及到的逻辑几乎一模一样，这里就不再阐述了。

【好了，经过这几章的学习之后，想必各位已经掌握了 SpringCache 的使用。SpringCache 本身是 SpringFramework 自己封装的一套模型，并且充分利用了 IOC 容器和 Spring 的一些特性，使我们在操纵缓存时变得更加容易。不过目前的话似乎还有一个问题：SpringCache 只能设置一个缓存目的地，如果我们需要将数据缓存到多个位置，形成层级关系，进一步提高响应速度，有什么办法呢？下一章我们就来揭晓答案】

