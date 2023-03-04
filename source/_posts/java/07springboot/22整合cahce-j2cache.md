---
title: 22整合cahce-j2cache
toc: true
tags: springboot
categories: 
    - [java]
    - [springboot]
---

上一章的最后咱们留了一个问题：SpringCache 的设计是同一时间只能让一个缓存模块生效，要么用 simple 在本地使用 Map 的形式缓存数据，要么连接 Redis 使用远程的缓存中间件缓存数据，它们之间是不能共存的。按照以往的设计来看，我们只能在这两种方案之间权衡：

- 本地缓存方案：速度快，延时低，但无法保证多个服务共享一块缓存
- 缓存中间件方案：可以让多个服务共享同一块缓存，但因为有远程交互，速度相对慢

那有没有一种可能，我们可以把这两者结合起来，既保证数据能缓存到本地，同时也能连接到远程的缓存中间件上呢？还真有，这就是本章要讲解的 **j2cache** 。

## 1. j2cache概述

可能有小伙伴看到这个名之后会产生一种感觉：~~j2cache 又是一个新的缓存方案吧~~！其实并不是，j2cache 只是一个缓存的**整合**框架，它允许我们提供不同的缓存实现来组合使用，但是它**自身不提供缓存能力**。

> 注意，j2cache 并不是整合的 SpringCache ！

<!--more-->

如何理解缓存实现的整合呢？举个简单的例子，我们可以使用本地缓存的 simple （即 Map 方案）配合远程的 Redis 来实现两层级缓存，也可以使用本地的 ehcache （一个很强大的本地缓存组件）配合远程的 memcached （老牌缓存中间件之一）来实现，也可以掺杂着相互搭配，都可以。

那话说回来，j2cache 都能让我们用哪些缓存的实现呢？从[官方文档](https://gitee.com/ld/J2Cache)上可以得知有以下的可选空间。

- 一级缓存（进程级缓存）：

  - caffeine
  - ehcache

- 二级缓存（集中式 / 分布式缓存中间件）：

  - redis

  - memcached

当使用了 j2cache 时，数据读取的顺序会变为：**一级缓存 → 二级缓存 → 数据库** ；而当缓存的数据要改变时，方案是从数据库中读取到最新的，依次放入 / 更新一级缓存和二级缓存中。

## 2. SpringBoot整合j2cache

简单了解 j2cache 后，下面我们就来实际的整合一下。为了跟前面的 SpringCache 工程区分开，所以我们重开一个工程吧，工程名叫 `spring-boot-integration-06-j2cache` 。

### 2.1 导入依赖

j2cache 本身有提供整合 SpringBoot 的启动器，但是导入启动器的同时还需要导入 j2cache 的核心包，也就是一共需要导入两个依赖。

```xml
    <dependency>
        <groupId>net.oschina.j2cache</groupId>
        <artifactId>j2cache-spring-boot2-starter</artifactId>
        <version>2.8.0-release</version>
    </dependency>
    <dependency>
        <groupId>net.oschina.j2cache</groupId>
        <artifactId>j2cache-core</artifactId>
        <version>2.8.5-release</version>
    </dependency>
```

此外，为了继续演示与数据库、Redis 的交互，以及与 SpringCache 的整合，我们仍然导入 `spring-boot-starter-cache` ，`mybatis-spring-boot-starter` 、`spring-boot-starter-data-redis` 与 `spring-boot-starter-test` 依赖。

### 2.2 基础代码

基础代码的部分，我们可以完整拷贝一份 `spring-boot-integration-06-cache` 的代码，只留其中最原始的 entity 、mapper 、service 即可。

`application.properties` 中则只留下以下的部分即可（需要改包名的地方记得改呀）。

```properties
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
spring.datasource.url=jdbc:mysql://localhost:3306/sringboot-dao?characterEncoding=utf8
spring.datasource.username=root
spring.datasource.password=123456

mybatis.type-aliases-package=com.linkedbear.boot.j2cache.entity
mybatis.mapper-locations=classpath:mapper/*.xml
mybatis.configuration.map-underscore-to-camel-case=true

logging.level.com.linkedbear.boot.j2cache.mapper.UserMapper=debug
```

### 2.3 配置j2cache

下面是关键的环节了，我们要配置 j2cache 的内容。按常理来讲，整合第三方场景后，所有的配置内容都移步到 `application.properties` 或 `application.yml` 中才是合适的吧，然而当我们在 `application.properties` 中敲 j2 时，却发现只有一行配置：

很明显，这行配置让我们指定 j2cache 的配置文件路径，这个操作实在是不大好顶，我们不会还需要为 j2cache 创建单独的配置文件吧。

放心，其实并不是 `application.properties` 中不能写，而是 j2cache 没有做 `spring-configuration-metadata.json` ，所以 IDE 不会给我们提示更多信息，下面可以先跟着小册的脚步来写。

> 当然，各位也可以使用 j2cache 配置文件的方式去集成，由于 SpringBoot 项目更推荐将所有配置规整到全局配置文件中，所以独立配置文件的方式小册就不展开了，各位可以参照官方文档来尝试。

#### 2.3.1 主要配置

既然敲配置文件内容时不会提示，那小册干脆把需要配置的内容全部列出来，供各位参考吧！配置项中都带有注释，方便各位参考和理解。

```properties
# 一级缓存的提供方，可选择caffeine/ehcache
j2cache.L1.provider_class=caffeine
# 二级缓存的提供方，由于整合了SpringBoot，所以要使用适配Spring的Redis提供方
j2cache.L2.provider_class=net.oschina.j2cache.cache.support.redis.SpringRedisProvider
# 导入SpringDataRedis时默认带的lettuce，而j2cache默认使用jedis，所以此处需要显式配置
j2cache.L2.config_section=lettuce
# 手动开启二级缓存，构成两层级缓存
j2cache.l2-cache-open=true
# 指定Redis的客户端为lettuce（默认是jedis）
j2cache.redis-client=lettuce
# 将一级缓存中的过期时间同步到二级缓存Redis中
j2cache.sync_ttl_to_redis=true
# 向二级缓存保存数据时使用json序列化
j2cache.serialization=json
# 开启SpringCache的适配
j2cache.open-spring-cache=true
# 启用缓存过期的广播，该配置可以实现多节点缓存数据的发布订阅更新
j2cache.broadcast=net.oschina.j2cache.cache.support.redis.SpringRedisPubSubPolicy
# 缓存过期策略，可选值：active:主动清除(二级缓存过期时通知一级缓存),passive:被动清除(一级缓存过期时通知二级缓存),blend:两种模式一起运作
j2cache.cache-clean-mode=passive

# caffeine的配置文件路径
caffeine.properties=/caffeine.properties

# lettuce的配置，指定了缓存key的全局前缀
lettuce.channel=j2cache
lettuce.schema=redis
# 此处需要再指定一次redis的主机地址
lettuce.hosts=${spring.redis.host}:${spring.redis.port}

# SpringCache的配置，设置generic的意图为让j2cache来接管CacheManager
spring.cache.type=generic
```

至于其他的配置项，各位可以参照官方文档，以及 `j2cache-core` 核心包中的 `j2cache.properties` 文件，那里面是最原始的文件，不过里面有些配置在整合 SpringBoot 时会有变化。目前官方文档对整合 SpringBoot 后的配置也没有写明，希望后续能补全完善好吧。。。

#### 2.3.2 caffeine

由于我们指定了一级缓存使用 caffeine 来实现，所以还需要再创建一个 caffeine 的配置文件，来定义 caffeine 的缓存容器规则。默认的配置文件如下所示。

```properties
#########################################
# Caffeine configuration
# [name] = size, xxxx[s|m|h|d]
#########################################

default = 1000, 30m 
```

注释解释的比较清楚，默认的配置文件中配置了一个名为 `default` 的容器，它能存放 1000 个元素，并且该容器的缓存过期时间为 30 分钟。如果我们需要额外指定容器，可以用上述的格式来指定，但是请各位注意一点，Caffeine 中划分的缓存容器只是一个逻辑上的概念，总共内存就那么大，只是我们人为地将这块内存划分为若干个区域而已。

### 2.4 使用缓存

下面我们就来使用一下缓存。j2cache 给我们提供的操纵缓存的 API 是 `CacheChannel` ，并且可以直接使用 `@Autowired` 注入，那我们就来实际的使用一下。

为了不与之前的 `UserService` 冲突，我们新开一个 `CachedUserService` ，拷贝其中的代码，并且注入 `CacheChannel` 。

```java
@Service
public class CachedUserService {
    
    @Autowired
    private CacheChannel cacheChannel;
    
    @Autowired
    private UserMapper userMapper;
    
    public User get(Integer id) {
        return userMapper.get(id);
    }
}
```

下面我们对 get 方法进行改造，仿照之前 SpringBoot 整合 SpringCache 的思路，编程式缓存的设计是先获取数据，有数据则直接返回，没有数据的情况下从数据库查询数据，放到缓存后返回。而当我们尝试查看 `CacheChannel` 的 `get` 系方法时，发现所有的 `get` 方法都需要传入一个 `region` ：

这个 **region** 是个什么东西呢？哎，还记得上面我们刚看到的那个 `caffeine.properties` 吗？那里面定义的 `default` 就是一个 **region** 。这个 region 不是物理上的分区，只是逻辑上的分区。j2cache 的缓存模型本质上是一个 **region-key-value** 的 `key-key-value` 模型。

既然上面 `caffeine.properties` 中只配置了一个 `default` 区域，那我们写 default 就可以。实现的方法最终如下所示。

```java
public User get(Integer id) {
  String cacheKey = "getUser." + id;
  CacheObject cacheObject = cacheChannel.get("default", cacheKey);
  if (cacheObject.getValue() != null) {
    return (User) cacheObject.getValue();
  }
  User user = userMapper.get(id);
  cacheChannel.set("default", cacheKey, user);
  return user;
}
```

注意代码实现中的一个细节：由于目前我们只有一个 region ，所以需要在缓存的 key 上仿照 SpringCache 那样做一个区分，以保证在实际的项目开发中多个位置使用缓存。

### 2.5 测试代码

我们依然选用单元测试的方式来测试整合效果。下面我们编写一个 `J2cacheTest` 测试类，专门测试 j2cache 的整合和工作效果。

```java
@SpringBootTest
public class J2cacheTest {
    
    @Autowired
    private CachedUserService userService;
    
    @Test
    public void test1() throws Exception {
    	userService.get(1);
    	userService.get(1);
    	userService.get(1);
    }
}
```

测试的方法非常简单粗暴，我们直接连续调用 3 次 `CachedUserService` 的 `get` 方法，看看是否可以让缓存生效。

#### 2.4.1 第一次get

当第一次执行 `get` 方法时，我们把断电打在 `CachedUserService#get` 方法的第 3 行 if 判断上，待程序停在断点时，可以发现此时查询的 `CacheObject` 内 value 值是空的，代表一级缓存和二级缓存中都没有数据。

那既然没有数据，if 结构自然不会进入，随后执行的是 `UserMapper` 的 `get` 方法，从数据库中查询到数据，并执行 `CacheChannel` 的 `set` 方法，将数据存放到一级缓存和二级缓存。

当第一次 get 方法执行完毕后，可以观察到 Redis 中已经保存了 `getUser.1` 的数据如下：

> 注意一个小细节：j2cache 已经帮我们在缓存的 key 前缀上追加了 region 的值，并用一个冒号隔开（与 SpringCache 作对比）。

#### 2.4.2 第二次get

缓存进数据之后，第二次再调用 `get` 方法时，由于一级缓存和二级缓存中都有缓存的数据，所以在使用 `CacheChannel` 获取数据时，是可以率先从一级缓存中取到数据的。通过 Debug 也可以发现，缓存数据已经被成功取出，而且 level 是 1 ，代表是一级缓存。

#### 2.4.3 重新运行test1后

如果我们重新 Debug 执行 `test1` 测试方法，那么 Caffeine 中的进程级缓存数据会被清除，但 Redis 中的数据不会清空。所以重新 Debug 后的第一次 `get` 方法触发时，可以发现仍然可以取出缓存的数据，只不过此时取出的数据来自二级缓存。

经过上述的测试之后，这就已经证明了 SpringBoot 整合 j2cache 已经完成。

### 2.6 测试覆盖缓存

下面我们再测试一下缓存被覆盖的情况。对于 `CacheChannel` 来讲，缓存数据和覆盖缓存的操作是一样的，没有任何区别。下面我们来编写一个 update 方法，看一看当数据被更新时，缓存中的数据是否可以被覆盖掉。

```java
    public User update(User user) {
        userMapper.updateById(user);
        String cacheKey = "getUser." + user.getId();
        cacheChannel.set("default", cacheKey, user);
        return user;
    }
```

在测试的时候，我们选择先从数据库中查出数据，再复制一个相同的临时对象用于保存（这样可以避免同一个对象从缓存中取出再放入）。

```java
    @Test
    public void test2() throws Exception {
        User user = userService.get(1);
        // 使用临时对象，防止有误解
        User temp = new User();
        BeanUtils.copyProperties(user, temp);
        temp.setName(temp.getName() + "-j2cache");
        userService.update(temp);
        User user1 = userService.get(1);
        System.out.println(user == user1);
        System.out.println(temp == user1);
    }
```

下面我们执行 `test2` 检验缓存覆盖的效果。Debug 首先执行时，get 动作取到的数据是从二级缓存 Redis 中取出，执行 `UserService` 的 `update` 方法后，此时已经执行了 `CacheChannel` 的 `set` 方法，此时再执行 `get` 方法取数据时，由于一级缓存 Caffeine 中有数据，所以会直接取出。如果此时比对几个对象的引用时，会发现 `temp` 跟 `user1` 是同一个对象，而 `user` 跟 `user1` 不是同一个。

### 2.7 测试移除缓存

最后测试的是缓存的移除，在 `CacheChannel` 中移除缓存的方式是 `evict` （不要因为上面都是 `get` 和 `set` 就下意识想到 `delete` 、`remove` 等操作哦）。

```java
    public void deleteById(Integer id) {
        // userMapper.deleteById(id);
        System.out.println("deleteById invoke ......");
        String cacheKey = "getUser." + id;
        cacheChannel.evict("default", cacheKey);
    }
```

之后是测试方法，我们先把数据从缓存中取出，再删除缓存，之后再获取一次数据，观察控制台上的 SQL 打印即可。

```java
    @Test
    public void test3() throws Exception {
        User user = userService.get(1);
        userService.deleteById(1);
        User user1 = userService.get(1);
        System.out.println(user == user1);
    }
```

执行 `test3` 方法，当第一次 `get` 方法执行时，由于 Redis 中有数据，所以不会触发 SQL 打印，而当 `deleteById` 方法执行后，缓存数据被移除，再执行 `get` 方法时就会触发 `UserMapper` 的数据库查询，MyBatis 的 SQL 日志打印也就跟着触发了。

```ini
deleteById invoke ......
[main] c.l.boot.j2cache.mapper.UserMapper.get   : ==>  Preparing: select * from tbl_user where id = ?
[main] c.l.boot.j2cache.mapper.UserMapper.get   : ==> Parameters: 1(Integer)
[main] c.l.boot.j2cache.mapper.UserMapper.get   : <==      Total: 1
false
```

## 3. 注解式缓存

---

了解完编程式的缓存使用后，想必各位可能会有一些不满吧！前面我们在使用 SpringCache 的时候明明能正常使用注解式缓存，简单又好用，为什么换到 j2cache 中就又回到编程式了呢？不要急，其实 j2cache 也可以支持与 SpringCache 一样的注解式缓存，我们也可以试一下。

### 3.1 兼容SpringCache

由于我们上面在 `application.properties` 中声明了 `j2cache.open-spring-cache=true` 配置项，所以 j2cache 的 starter 已经在底层帮我们适配了 SpringCache ，只需要我们再编写一个配置，将 SpringCache 的模式改为 **`GENERIC`** （即自定义提供）即可。

```properties
# 开启SpringCache的适配
j2cache.open-spring-cache=true
# SpringCache的配置，设置generic的意图为让j2cache来接管CacheManager
spring.cache.type=generic
```

这样两个配置写完之后，其实我们就已经完成了 SpringCache 的整合（倒不如说是开启兼容）。

### 3.2 测试注解式

那下面我们就来试一下在 SpringCache 的讲解中，我们编写的 `AnnotationUserService` 如果直接拷贝过来，能不能直接使用。

> 为了对缓存有一个区分，我们可以把注解中的 key 加一个前缀来区分。
>
> ```java
> @Service
> public class AnnotationUserService {
> 
>        @Autowired
>        private UserMapper userMapper;
>     
>        @Cacheable(value = "j2cache.user.get", key = "#id")
>        public User get(Integer id) {
>            return userMapper.get(id);
>        }
>        // ......
> }
> ```

测试的代码，我们仍然可以直接把测试 SpringCache 注解式缓存的代码拿过来应用。

```java
    @Autowired
    private AnnotationUserService annotationUserService;
    
    @Test
    public void test4() throws Exception {
        User user1 = annotationUserService.get(6);
        User user2 = annotationUserService.get(6);
        System.out.println(user1 == user2);
    }
    
    @Test
    public void test5() throws Exception {
        User user = annotationUserService.get(1);
        user.setName("mybatis test j2cache");
        annotationUserService.update(user);
        User user2 = annotationUserService.get(1);
        System.out.println(user == user2);
    }
    
    @Test
    public void test6() throws Exception {
        User user1 = annotationUserService.get(1);
        User user2 = annotationUserService.get(1);
        annotationUserService.deleteById(1);
        User user3 = annotationUserService.get(1);
        System.out.println(user1 == user3);
    }

```

下面就来实际测试一下。首先执行 `test4` 方法，将数据放入缓存中，控制台中只打印了一次 SQL 语句，而且 Redis 中也已经保存了数据，说明注解式缓存的效果已经生效。

留意一个细节：j2cache 的注解式缓存，最终保存到 Redis 时的 key 前缀就是我们在 @Cacheable 注解上声明的 value ，这个设计与 SpringCache 不同，各位一定要注意分辨。

