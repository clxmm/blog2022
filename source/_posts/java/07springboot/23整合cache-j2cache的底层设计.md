---
title: 23整合cache-j2cache的底层设计
toc: true
tags: springboot
categories: 
    - [java]
    - [springboot]
---

经过前一章的学习和使用之后，各位应该都会使用 j2cache 的两层级缓存了吧！下面我们还是来从底层剖析一下，当 SpringBoot 整合 j2cache 后，框架底层都帮我们干了什么，都有哪些核心的组件被装配了，以及这些核心组件的工作机制是怎样的。

## 1. 自动装配

---

首先我们还是先来看自动装配。找到 j2cache 与 SpringBoot 整合的场景启动器 starter 中，我们可以非常容易地找到一个 `spring.factories` 文件，其内部声明了 3 个自动配置类。

<!--more-->


```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  net.oschina.j2cache.autoconfigure.J2CacheAutoConfiguration,\
  net.oschina.j2cache.autoconfigure.J2CacheSpringCacheAutoConfiguration,\
  net.oschina.j2cache.autoconfigure.J2CacheSpringRedisAutoConfiguration
```

下面逐个来看。

### 1.1 J2CacheAutoConfiguration

j2cache 的整合核心自动装配是 `J2CacheAutoConfiguration` ，这个配置类没有什么 `@Import` 注解的使用（这一点还略有让我们感到意外），仅仅是使用 `@Bean` 注册了几个组件。

```java
@ConditionalOnClass(J2Cache.class)
@EnableConfigurationProperties({J2CacheConfig.class})
@Configuration
@PropertySource(value = "${j2cache.config-location}", encoding = "UTF-8", ignoreResourceNotFound = true)
public class J2CacheAutoConfiguration {

    @Autowired
    private StandardEnvironment standardEnvironment;

    // 核心的j2cache配置对象
    @Bean
    public net.oschina.j2cache.J2CacheConfig j2CacheConfig() throws IOException{
        return SpringJ2CacheConfigUtil.initFromConfig(standardEnvironment);
    }

    // 默认注册的CacheChannel
    @Bean
    @DependsOn({"springUtil", "j2CacheConfig"})
    public CacheChannel cacheChannel(net.oschina.j2cache.J2CacheConfig j2CacheConfig) throws IOException {
        return J2CacheBuilder.init(j2CacheConfig).getChannel();
    }

    // 用于全局获取IOC容器
    @Bean
    public SpringUtil springUtil() {
        return new SpringUtil();
    }
}
```

可以发现，在这里面有我们在上一章中使用过的 `CacheChannel` ，它依赖的是 j2cache 封装的配置信息对象 `J2CacheConfig` 。而 `J2CacheConfig` 则是依靠获取 `Environment` 中的数据来构建。下面我们可以简单看一下 `CacheChannel` 构造的细节。

#### 1.1.1 CacheChannel的构造细节

上面的源码中我们看到，`CacheChannel` 的构造分为两步，对应的方法实现就是下面的两个方法。其中第一个方法很好理解，就是 new 了一个建造器 `J2CacheBuilder` 而已，而下面的 `getChannel` 中，除了熟悉的双检锁控制并发之外，其内部有一个匿名内部类实在是有点难顶：

```java
    /**
     * 初始化 J2Cache，这是一个很重的操作，请勿重复执行
     *
     * @param config j2cache config instance
     * @return J2CacheBuilder instance
     */
    public final static J2CacheBuilder init(J2CacheConfig config) {
        return new J2CacheBuilder(config);
    }
```

```java
/**
     * 返回缓存操作接口
     *
     * @return CacheChannel
     */
    public CacheChannel getChannel() {
        if (this.channel == null || !this.opened.get()) {
            synchronized (J2CacheBuilder.class) {
                if (this.channel == null || !this.opened.get()) {
                    this.initFromConfig(config);
                    /* 初始化缓存接口 */
                    this.channel = new CacheChannel(config, holder) {
                        @Override
                        public void sendClearCmd(String region) {
                            policy.sendClearCmd(region);
                        }

                        @Override
                        public void sendEvictCmd(String region, String... keys) {
                            policy.sendEvictCmd(region, keys);
                        }

                        @Override
                        public void close() {
                            super.close();
                            policy.disconnect();
                            holder.shutdown();
                            opened.set(false);
                        }
                    };
                    this.opened.set(true);
                }
            }
        }
        return this.channel;
    }
```

暂且不关心这个 `CacheChannel` 匿名实现类是怎么折腾的，请各位先留意一下初始化 `CacheChannel` 的上面一句：`this.initFromConfig(config);` 这个方法内部的实现还是比较重要的。

#### 1.1.2 initFromConfig

往下再读源码的时候会有一种感觉：不愧是中国人写的开源组件，注释都是用中文（不用翻译）！的确，国产开源组件在我们分析底层时，耗费的精力和时间的确会少（当然前提是合理的注释得管够）。简单扫一遍吧，`initFromConfig` 方法的核心全部都是在初始化，包含序列化器、两层级缓存联动的组件，以及二级缓存的集群管理。其中两层级缓存联动的监听器会传入一个 lambda 表达式，这个我们暂且不作研究，后面遇到再说。

```java
/**
     * 加载配置
     *
     * @return
     * @throws IOException
     */
    private void initFromConfig(J2CacheConfig config) {
        SerializationUtils.init(config.getSerialization(), config.getSubProperties(config.getSerialization()));
        //初始化两级的缓存管理
        this.holder = CacheProviderHolder.init(config, (region, key) -> {
            //当一级缓存中的对象失效时，自动清除二级缓存中的数据
            Level2Cache level2 = this.holder.getLevel2Cache(region);
            level2.evict(key);
            if (!level2.supportTTL()) {
                //再一次清除一级缓存是为了避免缓存失效时再次从 L2 获取到值
                this.holder.getLevel1Cache(region).evict(key);
            }
            log.debug("Level 1 cache object expired, evict level 2 cache object [{},{}]", region, key);
            if (policy != null)
                policy.sendEvictCmd(region, key);
        });

        policy = ClusterPolicyFactory.init(holder, config.getBroadcast(), config.getBroadcastProperties());
        log.info("Using cluster policy : {}", policy.getClass().getName());
    }
```

注意初始化完成 `holder` 后，最后生成的 `CacheChannel` 中也组合了它，所以不难理解 `CacheChannel` 具备一二级缓存数据联动同步的能力，本质是内部的 `CacheProviderHolder` 发挥作用。

### 1.2 J2CacheSpringCacheAutoConfiguration

下面我们继续看自动装配，j2cache 能够适配 SpringCache 的使用，本质是为 SpringCache 提供了一个自行实现的 `J2CacheCacheManger` 来接管缓存的具体行为（这也启发我们，如果需要自行实现缓存方案，可以自行提供 `CacheManager` 的实现类）。`J2CacheSpringCacheAutoConfiguration` 的配置内容非常简单，它只是开启了注解式缓存，并且注册了 `J2CacheCacheManger` 。

```java
@Configuration
@ConditionalOnClass(J2Cache.class)
@EnableConfigurationProperties({ J2CacheConfig.class, CacheProperties.class })
@ConditionalOnProperty(name = "j2cache.open-spring-cache", havingValue = "true")
@EnableCaching
public class J2CacheSpringCacheAutoConfiguration {

    private final CacheProperties cacheProperties;
    private final J2CacheConfig j2CacheConfig;
    // ......

    @Bean
    @ConditionalOnBean(CacheChannel.class)
    public J2CacheCacheManger cacheManager(CacheChannel cacheChannel) {
        List<String> cacheNames = cacheProperties.getCacheNames();
        J2CacheCacheManger cacheCacheManger = new J2CacheCacheManger(cacheChannel);
        cacheCacheManger.setAllowNullValues(j2CacheConfig.isAllowNullValues());
        cacheCacheManger.setCacheNames(cacheNames);
        return cacheCacheManger;
    }
}
```

注意 `@EnableCaching` 注解已经由自动配置类帮我们开启了，所以我们无需再在主启动类上标注。

### 1.3 J2CacheSpringRedisAutoConfiguration

j2cache 能够自动支持 Redis ，底层的自动配置类是 `J2CacheSpringRedisAutoConfiguration` 。这个类中配置的东西是这三个自动配置类中最多的，我们分解开来。

#### 1.3.1 生效条件

```java
@Configuration
@AutoConfigureAfter({ RedisAutoConfiguration.class })
@AutoConfigureBefore({ J2CacheAutoConfiguration.class })
@ConditionalOnProperty(value = "j2cache.l2-cache-open", havingValue = "true", matchIfMissing = true)
public class J2CacheSpringRedisAutoConfiguration {
```

`J2CacheSpringRedisAutoConfiguration` 的生效条件只有一条，那就是我们要么不写配置，要么就主动声明开启二级缓存。上一章的讲解中小册为了给各位讲解 j2cache 的配置内容，所以就显式在配置文件中编写了这个配置。

#### 1.3.2 ConnectionFactory

无论是用什么框架去访问 Redis ，底层的连接驱动大概率是 jedis 或者 lettuce ，自动装配中注册的 `ConnectionFactory` 就是根据配置声明的 `j2cache.redis-client` 的值决定初始化基于 jedis 还是 lettuce 的。由于 j2cache-core 核心包中已经内置了 jedis 的客户端，所以即便我们不声明、不导入 SpringDataRedis 的依赖，它也可以帮我们装配一个基于 jedis 的连接工厂，只是我们在上一章中手动指定了使用 lettuce 作为连接工厂，并且希望 j2cache 与 SpringDataRedis 一起使用 lettuce 作为底层连接驱动，所以才会装配基于 lettuce 的 `LettuceConnectionFactory` 。

```java
@Bean("j2CahceRedisConnectionFactory")
@ConditionalOnMissingBean(name = "j2CahceRedisConnectionFactory")
@ConditionalOnProperty(name = "j2cache.redis-client", havingValue = "jedis", matchIfMissing = true)
public JedisConnectionFactory jedisConnectionFactory(net.oschina.j2cache.J2CacheConfig j2CacheConfig) {
  Properties l2CacheProperties = j2CacheConfig.getL2CacheProperties();
  String hosts = l2CacheProperties.getProperty("hosts");
  String mode = l2CacheProperties.getProperty("mode") == null ? "null" : l2CacheProperties.getProperty("mode");
  String clusterName = l2CacheProperties.getProperty("cluster_name");
  String password = l2CacheProperties.getProperty("password");
  int database = l2CacheProperties.getProperty("database") == null ? 0
    : Integer.parseInt(l2CacheProperties.getProperty("database"));
  JedisConnectionFactory connectionFactory = null;
  // ......
  return connectionFactory;
}

@Primary
@Bean("j2CahceRedisConnectionFactory")
@ConditionalOnMissingBean(name = "j2CahceRedisConnectionFactory")
@ConditionalOnProperty(name = "j2cache.redis-client", havingValue = "lettuce")
public LettuceConnectionFactory lettuceConnectionFactory(net.oschina.j2cache.J2CacheConfig j2CacheConfig) {
  // 与上方一致 ......
  LettuceConnectionFactory connectionFactory = null;
  // ......
  return connectionFactory;
}
```

#### 1.3.3 RedisTemplate

与 SpringDataRedis 不一样的是，j2cache 在与 Redis 交互的时候，用的就是我们最最熟悉的 `RedisTemplate` ，不像 SpringDataRedis 那样搞一个什么 `RedisCacheWriter` 作为替代。而构造 `RedisTemplate` 的时候，它指定了 value 的序列化器为 `J2CacheSerializer` ，而 `J2CacheSerializer` 本身内部又支持好几种序列化方式，其实本质上 `J2CacheSerializer` 就是 j2cache 为了支持 `RedisTemplate` 设计的一个适配器而已。

```java
@Bean("j2CacheRedisTemplate")
public RedisTemplate<String, Serializable> j2CacheRedisTemplate(
  @Qualifier("j2CahceRedisConnectionFactory") RedisConnectionFactory j2CahceRedisConnectionFactory,
  @Qualifier("j2CacheValueSerializer") RedisSerializer<Object> j2CacheSerializer) {
  RedisTemplate<String, Serializable> template = new RedisTemplate<String, Serializable>();
  template.setKeySerializer(new StringRedisSerializer());
  template.setHashKeySerializer(new StringRedisSerializer());
  template.setDefaultSerializer(j2CacheSerializer);
  template.setConnectionFactory(j2CahceRedisConnectionFactory);
  return template;
}

@Bean("j2CacheValueSerializer")
@ConditionalOnMissingBean(name = "j2CacheValueSerializer")
public RedisSerializer<Object> j2CacheValueSerializer() {
  return new J2CacheSerializer();
}
```

过完一遍自动装配后，各位是否有一种感觉：j2cache 在整合 SpringBoot 的时候看上去没有注册多少组件，其实这种优秀的机制还是因为 SpringBoot 的装配机制，加上 SpringCache 缓存模型的高扩展性优势所使。

## 2. CacheChannel的工作机制

下面我们着重来看 `CacheChannel` 的工作机制。既然 `CacheChannel` 能支撑得起两层级缓存，那我们就着重探究以下几个问题：

- `CacheChannel` 如何整合了这两层级的缓存
- 当缓存数据时，`CacheChannel` 如何将数据放入了两层级缓存中
- 当取出数据时，`CacheChannel` 的加载机制是怎样的
- 当缓存销毁后，`CacheChannel` 如何让两层级缓存也同步销毁的

### 2.1 整合两层级缓存的结构

由于 `CacheChannel` 是一个抽象类，而唯一的实现类在 `J2CacheBuilder` 的 `getChannel` 方法中，藏着一个匿名内部类，如下面源码所示。

```java
private ClusterPolicy policy;

// initFromConfig方法中初始化
policy = ClusterPolicyFactory.init(holder, config.getBroadcast(), config.getBroadcastProperties());

```

```java

this.channel = new CacheChannel(config, holder) {
    @Override
    public void sendClearCmd(String region) {
        policy.sendClearCmd(region);
    }

    @Override
    public void sendEvictCmd(String region, String... keys) {
        policy.sendEvictCmd(region, keys);
    }
    // close ......
};
```

从重写的两个方法的方法名上看，这都是与集群缓存清除相关的动作，这不是我们关心的重点，贴出上面的源码只是告诉各位，借助 IDE 找实现类的时候跑到这里不用感到诧异。

那我们就回到抽象类 `CacheChannel` 中。`CacheChannel` 集成的两个核心组件是 `J2CacheConfig` 和 `CacheProviderHolder` ，而 `CacheProviderHolder` 很明显是指的缓存提供者的持有，在 1.1.2 节 `initFromConfig` 方法中被初始化。

```java
public abstract class CacheChannel implements Closeable, AutoCloseable {

    private static final Map<String, Object> _g_keyLocks = new ConcurrentHashMap<>();
    private J2CacheConfig config;
    private CacheProviderHolder holder;
    private boolean defaultCacheNullObject ;
    private boolean closed;
```

下面我们就结合 `CacheProviderHolder` 的初始化逻辑，研究一下 j2cache 的两层级缓存结构。

#### 2.1.1 CacheProviderHolder的初始化

`CacheProviderHolder` 的初始化需要传入全局配置对象 `J2CacheConfig` ，以及一个贯通一二级缓存过期的监听器，而在静态的 `init` 方法中，它分别取出配置中的一二级缓存的类型，并且用 `loadProviderInstance` 方法初始化，初始化完成后调用其 `start` 方法进行内部初始化。源码的逻辑非常简单，各位简单扫一遍即可。

```java
public class CacheProviderHolder {
    private CacheProvider l1_provider;
    private CacheProvider l2_provider;
    
    public static CacheProviderHolder init(J2CacheConfig config, CacheExpiredListener listener) {
        CacheProviderHolder holder = new CacheProviderHolder();

        holder.listener = listener;
        holder.l1_provider = loadProviderInstance(config.getL1CacheName());
        if (!holder.l1_provider.isLevel(CacheObject.LEVEL_1)) {
            throw new CacheException(holder.l1_provider.getClass().getName() + " is not level_1 cache provider");
        }
        holder.l1_provider.start(config.getL1CacheProperties());
        // logger ......

        holder.l2_provider = loadProviderInstance(config.getL2CacheName());
        // 检查配置的缓存是否为二级缓存 ......
        holder.l2_provider.start(config.getL2CacheProperties());
        // logger ......

        return holder;
    }
```

毫无疑问，这段源码里最值得探讨的有两步：1）`loadProviderInstance` 方法如何初始化缓存提供者；2）`start` 方法内部究竟都做了什么。

本小节我们先解答第一个问题。点开 `loadProviderInstance` 方法后，我们可以非常震惊的看到，源码中对于别名的处理竟然是如此的简单粗暴：

```java
private static CacheProvider loadProviderInstance(String cacheIdent) {
    switch (cacheIdent.toLowerCase()) {
        case "ehcache":
            return new EhCacheProvider();
        case "ehcache3":
            return new EhCacheProvider3();
        case "caffeine":
            return new CaffeineProvider();
        case "redis":
            return new RedisCacheProvider();
        case "readonly-redis":
            return new ReadonlyRedisCacheProvider();
        case "memcached":
            return new XmemcachedCacheProvider();
        case "lettuce":
            return new LettuceCacheProvider();
        case "none":
            return new NullCacheProvider();
    }
    try {
        return (CacheProvider) Class.forName(cacheIdent).newInstance();
    } catch (ClassNotFoundException | InstantiationException | IllegalAccessException e) {
        throw new CacheException("Failed to initialize cache providers", e);
    }
}
```

好家伙，合着我们能写什么类型的缓存，其实是源码中已经帮我们限制死的！如果写的缓存类型都不在枚举中，则只能写具体的 `CacheProvider` 接口的实现类全限定名，源码中会帮我们使用反射创建对象。

根据我们上一章测试的内容来看，一级缓存指定了 `caffeine` ，二级缓存指定的是全限定名 `net.oschina.j2cache.cache.support.redis.SpringRedisProvider` ，这样底层最终 `CacheProviderHolder` 中集成的两层级缓存就是 `CaffeineProvider` 和 `SpringRedisProvider` 了。

#### 2.1.2 一级缓存的初始化

下面我们分别来看这两个层级的缓存是如何初始化的。Caffeine 的初始化逻辑，需要来到实现类 `CaffeineProvider` 中，查看它的 `start` 方法。

在进入源码之前我们要再回顾一个事情，由于 Caffeine 的缓存结构是以逻辑形式划分缓存区，所以底层初始化缓存的时候，本质就是一块大的内存区域中划分不同的区域，起不同的名而已。

![](/img/202302/23j2cache.png)

来到 `start` 方法，可以发现整个方法包含两部分初始化逻辑，最终干的事情都是初始化与缓存配置相关的，而最终保存缓存的动作都是 `saveCacheConfig` 方法，该方法会将我们配置的缓存区大小、过期时间解析出来，并存入 `CaffeineProvider` 中，内部实现很简单，主要是字符串的解析动作，各位可以借助 IDE 查看，小册就不再贴出源码了，我们主要关注核心的两部分配置来源的加载。

```java
public void start(Properties props) {
    for (String region : props.stringPropertyNames()) {
        if (!region.startsWith(PREFIX_REGION)) {
            continue;
        }
        String s_config = props.getProperty(region).trim();
        region = region.substring(PREFIX_REGION.length());
        this.saveCacheConfig(region, s_config);
    }
    // 加载 Caffeine 独立配置文件
    String propertiesFile = props.getProperty("properties");
    if (propertiesFile != null && propertiesFile.trim().length() > 0) {
        InputStream stream = null;
        try {
            stream = getClass().getResourceAsStream(propertiesFile);
            if (stream == null) {
                stream = getClass().getClassLoader().getResourceAsStream(propertiesFile);
            }
            Properties regionsProps = new Properties();
            regionsProps.load(stream);
            for (String region : regionsProps.stringPropertyNames()) {
                String s_config = regionsProps.getProperty(region).trim();
                this.saveCacheConfig(region, s_config);
            }
        } // catch finally ......
    }
}
```

下面分别来看这两部分的配置加载逻辑。

**第一部分**是直接获取传入的 `Properties` 配置属性中，以 `caffeine.region.` 开头的配置项，并提取出这些配置项，构造逻辑缓存区。注意一点，由于这部分的配置项在方法被调用时只接收了 `application.properties` 中以 `caffeine` 开头的配置项，所以按照上一章的配置内容来看，Debug 时可以发现只会拿到如下图的一条配置项：

如果我们在 `application.properties` 中再配置其他的内容，如声明 `caffeine.region.user=100,2h` ，则代表我们划定了一个名为 `"user"` 的逻辑分区，它的最大存储容量为 100 ，数据过期时间为 2 小时。

**第二部分**解析的是我们传入的 `caffeine.properties` 配置文件，由于我们已经声明了 Caffeine 的独立配置文件，所以源码中会将该配置文件加载，并按照上述相似的逻辑逐个解析这些逻辑分区的配置划定。与第一部分的配置不同，由于是 Caffeine 的独立配置文件，所以在配置时就不用写那一大堆前缀了。

> 由这一点可以给我们一个启示：如果我们要在 SpringBoot 项目中集成一个第三方技术，通常都有两种配置方式，要么在 SpringBoot 全局配置文件 `application.properties` 中统一配置，要么通过一些手段引入第三方技术的独立配置文件（通常是在 `application.properties` 或者使用 `@ImportResource` 等方式指定独立配置文件的位置）。

#### 2.1.3 二级缓存的初始化

下面是二级缓存的初始化，由于我们使用 j2cache 接管了 SpringCache 的缓存体系，所以设置缓存实现类时并没有直接声明 lettuce ，而是与 SpringCache 的整合类 `SpringRedisProvider` ，下面我们进入它的 `start` 方法中一探究竟。

```java
public void start(Properties props) {
    this.namespace = props.getProperty("namespace");
    this.storage = props.getProperty("storage");
    this.config =  SpringUtil.getBean(net.oschina.j2cache.autoconfigure.J2CacheConfig.class);
    if(config.getL2CacheOpen() == false) {
        return;
    }
    this.redisTemplate = SpringUtil.getBean("j2CacheRedisTemplate", RedisTemplate.class);
}

```

从源码中很明显可以发现，`start` 方法并没有使用到我们在 properties 中声明的 lettuce 开头的配置（也就是下图的这 3 样），而是直接从 IOC 容器中取出了一个 `RedisTemplate` 。

那既然没用到，是不是就代表我们可以不写呢？答案是否定的，还记得上面创建 `ConnectionFactory` 的时候吗？那个地方是要用的，不然 `LettuceConnectionFactory` 都没法初始化，进而导致应用都启动不起来，所以这几行配置还是得写的。
等一二级缓存的 `CacheProvider` 都初始化完成后，整个缓存的结构也就初始化完毕了。

### 2.2 编程式缓存的工作流程

下面我们就缓存的几种情形，分别来讨论和分析 `CacheChannel` 是如何利用两层级缓存取出和放入数据的。

各位先总览一遍 `CacheChannel` 的 `get` 方法，从它的实现中可以发现，整个 get 方法的处理逻辑是先借助双检锁，两次从一级缓存中获取数据，如果两次获取都没有拿到数据，下面就会尝试从二级缓存取出数据并返回，并当数据成功从二级缓存中取出时，同步放入一级缓存。

```java
public CacheObject get(String region, String key, boolean...cacheNullObject)  {
    // assert ......
    CacheObject obj = new CacheObject(region, key, CacheObject.LEVEL_1);
    obj.setValue(holder.getLevel1Cache(region).get(key));
    if (obj.rawValue() != null) {
        return obj;
    }
    String lock_key = key + '%' + region;
    synchronized (_g_keyLocks.computeIfAbsent(lock_key, v -> new Object())) {
        // 双检锁第二次尝试 ......
        
        try {
            obj.setLevel(CacheObject.LEVEL_2);
            obj.setValue(holder.getLevel2Cache(region).get(key));
            if (obj.rawValue() != null) {
                holder.getLevel1Cache(region).put(key, obj.rawValue());
            } else {
                boolean cacheNull = (cacheNullObject.length > 0) ? cacheNullObject[0] : defaultCacheNullObject;
                if (cacheNull) {
                    set(region, key, newNullObject(), true);
                }
            }
        } finally {
            _g_keyLocks.remove(lock_key);
        }
    }
    return obj;
}
```

根据两层级缓存中的数据是否存在，简单分情况讨论：

| 一级缓存 | 二级缓存 | 处理策略                                                     |
| -------- | -------- | ------------------------------------------------------------ |
| 无       | 无       | 两次尝试从一级缓存中获取数据，之后尝试从二级缓存获取，最终返回 null 数据 |
| 无       | 有       | 两次尝试从一级缓存中获取数据，之后从二级缓存获取成功，并放入一级缓存，返回二级缓存的数据 |
| 有       | 有       | 数据从一级缓存中获取成功，返回一级缓存的数据                 |

实际的 Debug 动作，各位可以借助上一章的测试代码，自行体验和感受 `CacheChannel` 的缓存加载机制，小册不再分情况展开。

## 3. J2CacheCacheManger适配SpringCache的工作机制

---

最后一个小节，我们来简单探讨下 j2cache 用于适配 SpringCache 的 `J2CacheCacheManger` ，看一下它是如何接管 SpringCache 默认的那些实现的。

### 3.1 J2CacheCacheManger生成J2CacheCache

结合第 21 章的内容，想必各位已经知道，`CacheManager` 的核心工作就是通过 `getCache` 方法获取指定的 `Cache` 对象，利用 `Cache` 对象进行缓存数据的存取动作。如果各位有些不记得的话，看看下面的源码是否能够回忆起来。

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
            // ......
        }
        return cache;
    }
```

有印象了吧！`getCache` 方法中有能力创建 `Cache` 对象的核心方法是 `getMissingCache` ，而这个方法在 `J2CacheCacheManger` 中的确有重写：

```java
	protected Cache getMissingCache(String name) {
		return this.dynamic ? new J2CacheCache(name, cacheChannel, allowNullValues) : null;
	}
```

可以很明显地看出，`J2CacheCacheManger` 的实现类中有构造 `J2CacheCache` 的动作，这也符合 SpringCache 中制定的规范：`CacheManager` 具备获取和创建 `Cache` 的能力。

> 有小伙伴可能会产生一个疑问：`dynamic` 是什么？如果我们结合上下文的源码会发现，`J2CacheCacheManger` 其实是有一个特殊设计的。如果我们在 SpringBoot 全局配置文件中有显式定义 `spring.cache.cache-names` 属性，则 `J2CacheCacheManger` 中会在工程启动时立即初始化这些 `Cache` ，并且在工程运行期间不会再创建新的 `Cache` 对象。
>
> ```java
> public void setCacheNames(Collection<String> cacheNames) {
>     Set<String> newCacheNames = CollectionUtils.isEmpty(cacheNames) ? Collections.<String> emptySet()
>              : new HashSet<String>(cacheNames);
>     this.cacheNames = newCacheNames;
>     this.dynamic = newCacheNames.isEmpty();
> }
> ```

### 3.2 J2CacheCache的工作机制

既然 `J2CacheCacheManger` 创建出了 `J2CacheCache` 对象，那么实现类 `J2CacheCache` 中又是如何存取缓存数据的呢？我们还是关注内部核心的 `put` 和 `lookup` 方法。

```java
public void put(Object key, Object value) {
    cacheChannel.set(j2CacheName, String.valueOf(key), value, super.isAllowNullValues());
}
```

`put` 方法的实现，就是拿 `CacheChannel` 调用它的 `set` 方法。需要注意的是，在适配 SpringCache 时传入的 `region` 就是 `cacheName` ，也即 `@Cacheable` 等注解中的 `value` 属性。

```java
protected Object lookup(Object key) {
    CacheObject cacheObject = cacheChannel.get(j2CacheName, String.valueOf(key));
    if (cacheObject.rawValue() != null && cacheObject.rawValue().getClass().equals(NullObject.class) && super.isAllowNullValues()) {
        return NullValue.INSTANCE;
    }
    return cacheObject.getValue();
}
```

`lookup` 方法的实现，就是利用 `CacheChannel` 的 `get` 方法，从两层级缓存中获取指定的数据，并在判断数据存在时返回数据本身，数据不存在时返回 `NullValue` 对象。

