---
title: 20整合cache-配合redis实现缓存
toc: true
tags: springboot
categories: 
    - [java]
    - [springboot]
---

上一章的最后我们留了一个问题：在真实的项目开发中，我们更多的是将缓存放在一个外部的缓存中间件中，而不是放在每个服务的进程内存中。这种需求就使得我们需要变更缓存的提供者（有的人称其为“供应商”）。

都能换哪些缓存提供者呢？我们需要找到 `CacheManager` 接口，看一下它的实现类。借助 IDE 可以发现 `CacheManager` 的实现类还真不少，其中还不乏我们熟悉的缓存组件：EhCache 、Redis 等。

<!--more-->

![](/img/202302/20chache.png)

本章我们就以 Redis 为例，讲解 SpringCache 与 Redis 的整合，以实现缓存数据外部化。

## 1. 整合Redis的方式

----

### 1.1 导入依赖

其实 SpringCache 整合 Redis 的方式非常简单，甚至都不需要我们多做什么，只需要在 pom.xml 中再导入 SpringDataRedis 的启动器即可。

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

对，这就是之前我们讲解 SpringData 整合 Redis 时用的依赖。导入这个依赖后，SpringCache 就会自动启用基于 Redis 的缓存配置。

我们可以先来检验一下缓存是否生效吧。

### 1.2 编写测试代码

为了避免跟上一章的代码混淆，本章我们重新创建一个 RedisCacheService ，用它来测试和演示有关 Redis 的声明式缓存。

```java
@Service
public class RedisCacheService {

    @Cacheable(value = "getName")
    public String getName(Integer i) {
        System.out.println("getName invoke ......");
        return "name" + i;
    }
}
```

我们先来测试一下简单类型的存取。`getName` 方法会接收一个 `Integer` 类型的参数，返回 `String` 类型的数据。我们将 `getName` 方法上标注 `@Cacheable` 注解后，就代表 `getName` 方法的数据会缓存到 Redis 中。

下面对应的编写一个测试类来检验效果。测试类的代码非常简单，我们只需要连续调用几次相同的方法，并传入相同的参数即可。

```java
@SpringBootTest
public class RedisCacheTest {
    
    @Autowired
    private RedisCacheService redisCacheService;
    
    @Test
    public void test1() throws Exception {
    	redisCacheService.getName(1);
    	redisCacheService.getName(1);
    	redisCacheService.getName(1);
    }
}
```

执行 `test1` 方法，控制台只打印了一次 `getName invoke` 的输出，这就说明缓存已经生效了。

即便我们再反复执行多次 `test1` 方法，由于数据已经缓存到 Redis 中，所以每次执行也不会有任何打印了，除非我们把 Redis 的数据删除掉。

随后我们来到 Redis 中，通过 Redis Desktop Manager 连接到本地 Redis 实例，可以发现 Redis 中已经创建了一个 hash 结构的数据，且内部包含了一个键值对：

![](/img/202203/20cahce1.png)

### 1.3 自动配置

数据虽然存进去了，但是吧，这个缓存的 value 值未免有些奇怪，它为什么还有一段奇奇怪怪的前缀呢？这其实是因为默认情况下，Redis 使用的数据存储机制是 jdk 序列化。。。

具体的原理我们来找到底层操作数据的 `RedisTemplate` 中，在 `RedisTemplate` 的默认无参构造方法的下方有一个 `afterPropertiesSet` 方法，各位都很熟悉了，这个方法可以写入初始化逻辑！观察一下默认的序列化器创建，很明显就是基于 jdk 序列化机制的。

```java
public RedisTemplate() {}

@Override
public void afterPropertiesSet() {
    super.afterPropertiesSet();
    boolean defaultUsed = false;
    if (defaultSerializer == null) {
        defaultSerializer = new JdkSerializationRedisSerializer(
                classLoader != null ? classLoader : this.getClass().getClassLoader());
    }
    // ......
```

这也就解释了为什么我们看到的 value 数据很奇怪。

### 1.4 测试复杂类型

接下来我们测试一下缓存 `User` 对象。照例我们写一个很简单的 `getUser` 方法，注入 `UserMapper` 从数据库查询 `User` 对象。

```java
@Cacheable(value = "getUser")
public User getUser(Integer id) {
  System.out.println("get user invoke ..");
  User user = new User();
  user.setId(id + "");
  user.setName("name" + id);
  return user;
}
```

下面的动作就简单的不行了吧！来，我们在测试类里写一个测试方法来试一下吧！

```java
    @Test
    public void test2() throws Exception {
    	redisCacheService.getUser(1);
    	redisCacheService.getUser(1);
    	redisCacheService.getUser(1);
    }
```

```
Caused by: java.lang.IllegalArgumentException: DefaultSerializer requires a Serializable payload but received an object of type [com.linkedbear.boot.cache.entity.User]
	at org.springframework.core.serializer.DefaultSerializer.serialize(DefaultSerializer.java:43)
	at org.springframework.core.serializer.Serializer.serializeToByteArray(Serializer.java:56)
	at org.springframework.core.serializer.support.SerializingConverter.convert(SerializingConverter.java:60)
	... 81 more
```

emmmm 好像情况跟我们想的不一样。。。这次序列化失败了，而失败的原因很简单，因为 **jdk 的序列化机制需要被序列化的类实现 `Serializable` 接口**才行！而我们写的 `User` 类并没有实现 `Serializable` 接口，所以肯定就不能成功了！

## 2. 复杂类型的缓存

好，有了上面的承载后，我们来处理复杂数据类型的缓存。对于我们自己编写的实体类也好，模型类也好，只要是使用默认的 jdk 序列化机制作为存储数据的实现，那就必须实现 `Serializable` 接口。

### 2.1 使用Serializable接口

实现 Serializable 接口的同时，不要忘记添加上一个固定的 `serialVersionUID` ，以免出现跨机器反序列化失败的可能。

```java
public class User implements Serializable {
    private static final long serialVersionUID = 1L;
    
    private Integer id;
    
    private String name;
    
    private String tel;
```

之后再执行 `test2` 方法，可以发现这次就可以成功缓存 `User` 对象了。

借助 Redis Desktop Manager 可以发现 `User` 对象也以 jdk 序列化的方式，保存到 Redis 中了：

保存是保存上了，但是这个可读性未免太差了吧！有没有更好的办法呢？

### 2.2 使用json序列化

我们在上一小节中看到，`RedisTemplate` 保存数据之前要进行序列化，那既然 jdk 序列化出来的结果没啥可读性可言，那是不是可以换成 json 序列化的方式呢？哎，还真可以，`RedisTemplate` 配套的序列化器中还真就有一个 json 序列化器，我们结合自动装配的源码，来看如何配置。

#### 2.2.1 配置json序列化器

如果各位还记得上面我们看到默认注册的 `RedisTemplate` ，会怀疑如果我们覆盖这个 `RedisTemplate` 的注册，并指定基于 json 的序列化器，是不是就可以实现了呢？不好意思，我只能说你的思路是没问题的，但配置方式不对。在 SpringBoot 1.x 中这样配置是没有问题的，但是 SpringBoot 2.x 中它对缓存的配置做了一层封装，以隔离我们对 `RedisTemplate` 的操控。

具体代码来看，SpringBoot 2.x 中给我们提供的 API 是一个叫 `RedisCacheConfiguration` 的家伙，通过修改它的默认配置，可以使全局的 Redis 缓存器全部应用该配置。下面的代码就是指定了所有的 value 序列化统一使用 json 的方式。

```java
@Configuration(proxyBeanMethods = false)
public class RedisConfiguration {
    
    @Bean
    public RedisCacheConfiguration redisCacheConfiguration() {
        return RedisCacheConfiguration.defaultCacheConfig().serializeValuesWith(
                RedisSerializationContext.SerializationPair.fromSerializer(RedisSerializer.json()));
    }
}
```

> 注意！如果当前项目中依赖的基础依赖是 `spring-boot-starter` ，则还需要单独引入整合 json 的启动器：
>
> ```java
>     <dependency>
>         <groupId>org.springframework.boot</groupId>
>         <artifactId>spring-boot-starter-json</artifactId>
>     </dependency>
> ```

#### 2.2.2 测试序列化效果

这样配置完成是否可行呢？我们可以来试一下。将 Redis 中的 getName 数据整体移除（即移除 Redis 中的这个 key ），随后再次执行 test2 方法，观察 Redis 中的数据：

由此就指明了修改 Redis 数据序列化的方式。

![](/img/202302/20chache2.png)

## 3. 使用Redis缓存的一些其他细节

在使用 Redis 缓存时，除了正常使用之外，还有一些其他的小细节需要我们简单了解一下。

### 3.1 key前缀设置

#### 3.1.1 use-key-prefix

不知道小伙伴在查看缓存的时候，有没有留意到一个细节：无论是 `getName` 的缓存数据也好，`getUser` 的也好，内部的 key 都是以当前缓存的名称开头，中间是两个冒号，最后才是我们指定的缓存 key 。

为什么 SpringCache 要帮我们在缓存数据时附带一个额外的前缀呢？这其实是底层的一个约定大于配置：

```properties
spring.cache.redis.use-key-prefix=true

```

默认情况下这个配置的值是 true ，代表缓存数据时会带入默认的 key 前缀。我们可以将这个配置设置为 false ，这样缓存数据时，key 的值就是我们声明好的了。

但是话又说回来，为什么 SpringCache 会默认设置这个 key 的机制呢？请各位思考一下。

很简单的道理，如果 key 的前面不带 `getUser::` 这样的前缀，那就只剩下一个 `1` 了，不用说别的，就刚才咱写的这俩方法，就会因为 key 值相同而撞车！

```java
    @Test
    public void test1() throws Exception {
    	redisCacheService.getName(1);
    }
    
    @Test
    public void test2() throws Exception {
    	redisCacheService.getUser(1);
    }
```

所以有 `@Cacheable(value = "xxx")` 的设置，就可以完好地区分开不同的缓存区。

#### 3.1.2 global key-prefix

除了默认的每个缓存区都有自己专属的前缀之外，我们还可以设置一个全局的 key 前缀：

```properties
spring.cache.redis.key-prefix=boot_

```

这种设置的目的也相对容易理解：如果有多个服务使用同一个 Redis 实例，而这些服务中有使用相同的 key ，那还是会出现撞车的情况。如果给每个服务中设置不同的全局 key-prefix ，就可以避免多服务之间的 Redis key 撞车的可能。

### 3.2 过期时间

下面我们再考虑一个问题：Redis 中存放的数据是可以有过期时间的，那么对应到 SpringCache 整合 Redis 的场景中，它依然可以设置全局的过期时间。具体的设置方式是在 `application.properties` 中编写如下的配置：

```properties
spring.cache.redis.time-to-live=10s

```

注意这个配置的值指定的是一个 `Duration` ，可以指定不同的时间单位。

默认情况下，通过 SpringCache 放入 Redis 的数据，其过期时间是永久，意为永不过期。

### 3.3 定制缓存器

接着上面说，我们说在一些实际的业务场景中，不同的缓存可能有对应不同的缓存过期时间，如果只是靠配置 `application.properties` 的话，那也只能做到全局的统一过期时间。如何为每一套缓存定制不同的过期时间呢？

可能各位小伙伴第一时间想到的是从 `@Cacheable` 注解上入手，可是很遗憾，在 `@Cacheable` 注解上并没有与过期时间相匹配的属性供我们配置（因为过期时间也只是一些缓存中间件的特性而已，最原始的 `Map` 可没有数据过期时间这一说）。

那应该怎么办呢？其实入手的位置是对的，就是 `@Cacheable` 注解，但是我们要配置的是另外一个属性：`cacheManager` 。

```java
public @interface Cacheable {
    // ......
    
    String cacheManager() default "";
}
```

这个 `cacheManager` 缓存管理器的属性允许我们传入一个 IOC 容器中注册的 `CacheManager` 实现类的 bean 的名称，指定该属性后，对应的 `@Cacheable` 注解标注的方法在执行时，就会直接使用配置的 `CacheManager` 去管理缓存，而不是用自动装配中默认的。可能说起来有些抽象难理解，这样吧，我们来实际地定制一个缓存管理器来体会一下。

#### 3.3.1 定制缓存管理器

定制缓存管理器的方法，可以直接 new 一个 `RedisCacheManager` ，也可以借助建造器来链式创建。从 API 编写丝滑的角度来讲，还是用建造器要舒服一些。所以就可以有如下的 `RedisCacheManager` 注册代码：

```java
    @Bean
    public RedisCacheManager userCacheManager(RedisConnectionFactory connectionFactory,
            RedisCacheConfiguration redisCacheConfiguration) {
        return RedisCacheManager.builder(connectionFactory)
                .cacheDefaults(redisCacheConfiguration.entryTtl(Duration.ofMinutes(1))).build();
    }
```

注意一个关键点：在传入 `RedisConnectionFactory` 后，还要传入之前我们构造的全局使用的 `RedisCacheConfiguration` （因为它的内部有指定 json 序列化），在这个全局默认的 `RedisCacheConfiguration` 的基础上设置过期时间 `ttl` ，在上述代码中我们设置了缓存过期时间为 1 分钟。

#### 3.3.2 测试过期时间

注册好缓存管理器后，我们还需要在 `@Cacheable` 注解中声明这个缓存管理器：

```java
@Cacheable(value = "getUser", cacheManager = "userCacheManager")
public User getUser(Integer id) {
  return userMapper.get(id);
}

```

如此编写完成后，我们就可以测试了。

去掉 Redis 中缓存的 `getUser` 数据后，重新执行 `RedisCacheTest` 的 `test2` 方法，让 SpringCache 重新把数据放入 Redis 中。通过 Redis Desktop Manager 可以发现，此时放入的数据就已经有过期时间了。

【以上就是 SpringCache 整合 Redis 场景中常见的一些配置和定制化。下一章我们会深入源码，详细讲解 SpringCache 的底层配置，以及其中注册的核心组件】