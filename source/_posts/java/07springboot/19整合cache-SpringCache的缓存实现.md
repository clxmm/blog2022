---
title: 19整合cache-SpringCache的缓存实现
toc: true
tags: springboot
categories: 
    - [java]
    - [springboot]
---

上一章我们已经回顾了 JSR-107 规范中的缓存抽象模型，本章我们来看看 SpringFramework 中实现的缓存模型 SpringCache ，它本身参考和利用了 JSR-107 规范，且使用自己定义的 `CacheManager` 接口包容了众多缓存实现。

## 1. SpringCache的缓存模型

---

首先我们看看 SpringCache 的缓存模型设计。SpringFramework 认为 JSR-107 规范中的 API 过于复杂，5 个接口混合使用的确给我们开发者造成了一些编码层面的复杂度，于是 SpringCache 简化了 JSR-107 规范中的接口，只保留 `CacheManager` 与 `Cache` 两个接口，就可以完成一个应用级别的缓存模型抽象。

### 1.1 CacheManager

`CacheManager` 的作用跟 JSR-107 规范相同，都是用来创建和管理 `Cache` 对象。`CacheManager` 接口的设计非常简单，它只有两个方法，分别是获取指定 key 的缓存，以及获取当前 `CacheManager` 下所有的缓存对象：

<!--more-->


```java
public interface CacheManager {
    
    Cache getCache(String name);
    
    Collection<String> getCacheNames();
}
```

由此可见，`CacheManager` 的结构应该是 `key - key - value` 中的前半部分：`key - key` 。

另外，`CacheManager` 的实现类非常多，根据不同的缓存技术，有对应的 `RedisCacheManager` 、`EhcacheCacheManager` 、`SimpleCacheManager` 等。

### 1.2 Cache

`Cache` 由 `CacheManager` 创建，根据 `CacheManager` 的实现类不同，创建出来的 `Cache` 对象也不同（如 `RedisCacheManager` 创建出来的就是 `RedisCache` ）。`Cache` 是真正存储缓存数据的容器，它的核心方法设计如下：

```java
public interface Cache {

	String getName();

	@Nullable
	ValueWrapper get(Object key);

	@Nullable
	<T> T get(Object key, @Nullable Class<T> type);

	@Nullable
	<T> T get(Object key, Callable<T> valueLoader);

	void put(Object key, @Nullable Object value);
    // ......
}
```

由此可见，`Cache` 的结构是 `key - key - value` 中的后半部分：`key - value` 。

## 2. SpringCache的基本使用

通过上面简单的了解之后，我们已经可以大概地勾勒出 SpringCache 的使用方式了：在使用缓存时，首先拿到 `CacheManager` ，随后通过 `CacheManager` 取到所需要的缓存对象 `Cache` ，通过操作 `Cache` 对象的方法来存取数据。下面我们就来实际的整合一下 SpringCache 。

### 2.1 搭建工程

其实 SpringCache 不是一个单独的模块，它的核心代码位于 spring-context 包下，所以其实当我们引入 SpringFramework 的最基础依赖时，SpringCache 的依赖也就一并引入了，只不过由于自动装配的原因，使得 SpringCache 相关的组件没有被注册到 IOC 容器中而已（有关原理在第 21 章原理机制中会讲）。而引入 SpringCache 的方式，是在 IOC 容器中导入 `spring-boot-starter-cache` 依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
```

为了更接近于实际的开发场景演示，我们选择使用 MyBatis 连接 MySQL 数据库，以实现与数据库的交互：

```xml
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>2.2.0</version>
</dependency>
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
</dependency>

```

此外为了测试缓存是否起效，我们还需要引入 WebMvc 或 JUnit 的依赖，小册以引入 JUnit 的方式为例，各位可以根据自己熟悉的操作跟进。

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

### 2.2 编写代码引入缓存

随后我们在 `UserMapper` 中再添加几个查询类方法，以丰富查询场景：

```java
    @Select("select * from tbl_user where id = #{value}")
    User get(Integer id);

```

`UserService` 的代码编写也很简单，依旧是引入 `UserMapper` ，直接调用其方法即可：

```java
@Service
public class UserService {

    @Autowired
    private UserMapper userMapper;
    
    public User get(Integer id) {
        return userMapper.get(id);
    }
    
    public List<User> findAll() {
        return userMapper.findAll();
    }
    
    public List<User> findAllByName(String name) {
        return userMapper.findAllByNameLike(name);
    }
}
```

下面我们要引入缓存了，把上面的 `UserService` 复制一份，并命名为 `CachedUserService` ，我们在这个 `CachedUserService` 上做缓存引入。

首先我们介绍 SpringCache 的编程式缓存使用，在使用时需要注入 `CacheManager` 对象，并在所需要的方法中使用 `CacheManager` 获取 `Cache` 对象并操纵，以下我们对 `get` 方法来进行缓存引入：

```java
    @Autowired
    private CacheManager cacheManager;
    
    public User get(Integer id) {
        // 1. 通过CacheManager拿到名为user的缓存对象Cache
        Cache cache = cacheManager.getCache("user");
        // 2. 从Cache中尝试获取一个指定id的User类型的对象
        User user = cache.get(id, User.class);
        // 3. 如果对象数据存在，则直接返回
        if (user != null) {
            return user;
        }
        // 4. 如果数据不存在，则需要查询数据库，并将查询的结果放入Cache中
        User userFromDatabase = userMapper.get(id);
        cache.put(id, userFromDatabase);
        return userFromDatabase;
    }
```

最后！要在主启动类上标注 `@EnableCaching` 注解，表示我们需要开启缓存机制。

```java
@EnableCaching
@SpringBootApplication
public class SpringBootCacheApplication {
    
    public static void main(String[] args) {
        SpringApplication.run(SpringBootCacheApplication.class, args);
    }
}
```

### 2.3 测试效果

单元测试的编写也很简单，核心需要突出的点是连续调用两次 Service 的的方法，观察查询的两个对象是否完全相同，以及控制台中 MyBatis 打印的日志。

```java
@SpringBootTest
public class CacheTest {
    
    @Autowired
    private UserService userService;
    
    @Autowired
    private CachedUserService cachedUserService;
    
    @Test
    public void test1() {
        User user1 = userService.get(1);
        User user2 = userService.get(1);
        System.out.println(user1 == user2);
    }
    
    @Test
    public void test2() {
        User user1 = cachedUserService.get(1);
        User user2 = cachedUserService.get(1);
        System.out.println(user1 == user2);
    }
}
```

`test1` 方法用来测试不带缓存的 `UserService` ，`test2` 方法则是测试带缓存的 `CachedUserService` 。

分别执行两个方法，可以发现，执行 `test1` 方法时 MyBatis 发送了两次 select 语句，最终打印的结果也是 false ：

```ini
DEBUG --- [main] c.l.b.c.m.UserMapper.get  : ==>  Preparing: select * from tbl_user where id = ?
DEBUG --- [main] c.l.b.c.m.UserMapper.get  : ==> Parameters: 1(Integer)
DEBUG --- [main] c.l.b.c.m.UserMapper.get  : <==      Total: 1
DEBUG --- [main] c.l.b.c.m.UserMapper.get  : ==>  Preparing: select * from tbl_user where id = ?
DEBUG --- [main] c.l.b.c.m.UserMapper.get  : ==> Parameters: 1(Integer)
DEBUG --- [main] c.l.b.c.m.UserMapper.get  : <==      Total: 1
false
```

而执行 `test2` 方法时，MyBatis 只打印了一次 select 语句，两个对象也完全相同：

```ini
DEBUG --- [main] c.l.b.c.m.UserMapper.get  : ==>  Preparing: select * from tbl_user where id = ?
DEBUG --- [main] c.l.b.c.m.UserMapper.get  : ==> Parameters: 1(Integer)
DEBUG --- [main] c.l.b.c.m.UserMapper.get  : <==      Total: 1
true
```

由此可以发现缓存已经生效。

### 2.4 编程式缓存的补充

#### 2.4.1 一个类共用一个缓存

如果引入缓存的每个方法都获取一遍 `Cache` 对象，那未免重复编码也太多了点，我们可以利用 SpringFramework 对 Bean 生命周期的管理，在 Service 层的 Bean 初始化时就获取 `Cache` 对象，减少一部分重复代码。

```java
@Service
public class CachedUserService implements InitializingBean {
    
    @Autowired
    private UserMapper userMapper;
    
    @Autowired
    private CacheManager cacheManager;
    
    private Cache cache;
    
    @Override
    public void afterPropertiesSet() throws Exception {
        this.cache = cacheManager.getCache("user");
    }
```

#### 2.4.2 Cache的其他方法

`Cache` 存取数据的方式，除了上面用到的 get + put 操作之外，还有一种更简单的编写方式：

```java
    public List<User> findAllByName(String name) {
        return cache.get(name, () -> userMapper.findAllByNameLike(name));
    }
```

## 3. 注解式缓存使用

----

即便是有上述的方法，可以让我们在使用编程式缓存时尽可能简单地编码，但毕竟还是需要注入组件，并在每个组件中写初始化逻辑。SpringCache 自然帮我们想到了这一点，在 3.1 版本后 SpringFramework 全面支持注解，SpringCache 也不例外，它提供了几个好用的注解帮我们简化缓存的使用，本章的重点我们就来了解这些注解的使用。

### 3.1 @Cacheable

`@Cacheable` 注解是注解式缓存的核心注解，它可以标注在一个方法上，表示该方法的执行之前会先查缓存，并且会当缓存中没有数据时执行目标方法，将方法返回值放入缓存中。`@Cacheable` 缓存的工作机制其实跟上面我们所说的方式，在本质上还是一模一样的。下面我们就来体验一下。

#### 3.1.1 最简单使用

将 `UserService` 再复制出一份，并命名为 `AnnotationUserService` ，我们在这个类上测试注解式缓存。使用 `@Cacheable` 注解的方式非常简单，只需要在需要控制缓存的方法上标注 `@Cacheable` 注解，并传入一个 `value` 值即可：

```java
@Service
public class AnnotationUserService {

    @Autowired
    private UserMapper userMapper;
    
    @Cacheable("user.get")
    public User get(Integer id) {
        return userMapper.get(id);
    }
```

> 注意：如果不传入 `value` 值的话，IDEA 会给出一个警告，提示我们这个注解需要提供一个 `cacheName` ，并且如果无视该注解直接启动程序运行的话，也是会抛出 `IllegalStateException` 的异常，提示 `At least one cache should be provided per cache operation.` 的信息。
>
> 为什么会有这种限定呢？回想一下我们上面反复提到的 SpringCache 模型，`CacheManager` 是全局统一的，但是它不负责存储数据，存储数据的是通过 `CacheManager` 创建的一个一个的 `Cache` 对象，而创建 `Cache` 对象时需要提供一个 `cacheName` 才可以。同样的道理，当使用注解式缓存时，同样需要告诉 SpringCache ，我们在标注的 `@Cacheable` 注解时对应的 `cacheName` 是什么，如果不提供，那么 SpringCache 也没有办法为我们生成 `Cache` 对象，自然也就抛出异常了。

配置完成后，下面就可以编写测试代码来检验缓存是否生效了。与上面的编程式缓存测试类似，编写的单元测试代码几乎完全一致：

```java
    @Autowired
    private AnnotationUserService annotationUserService;
    
    @Test
    public void test3() throws Exception {
        User user1 = annotationUserService.get(1);
        User user2 = annotationUserService.get(1);
        System.out.println(user1 == user2);
    }
```

执行 `test3` 方法，可以发现 MyBatis 还是只发了一次 SQL ，并且两个对象的引用完全一致，这也证明注解式缓存已经生效。

```ini
true
```

#### 3.1.2 缓存的Key

在上述的示例代码中，我们只看到了一个注解和相应的方法入参以及返回值，那缓存本身是基于 key-value 的映射关系，那默认情况下注解式缓存的映射规则是什么呢？

很明显，既然在上述的 `get` 方法中，它的本质是输入一个 `Integer` 类型的 id ，返回一个 `User` 类型的对象，那么对于默认情况下，它的缓存映射关系就应该是 `id → User` 。

> 如果被标注 `@Cachable` 注解的方法参数包含多个，则 key 的规则是所有参数的组合体。

在部分场景中，我们可能需要替换掉默认的 key 设置规则，SpringCache 利用 SpringFramework 的 SpEL 表达式机制，给我们提供了一些可选的设置规则，下面我们也来了解一下在编写 key 的 SpEL 表达式中，都可以写哪些内容。

##### 1）methodName & method

既然是利用方法注解的声明式缓存，那必然可以取到的元素之一就是方法信息。key 的 SpEL 表达式编写基本都是以 **`#root`** 开头的，引用方法名或者方法信息的写法就是：`#root.methodName` 和 `#root.method.name` 。

```java
    @Cacheable(value = "user.get", key = "#root.methodName")
    public User get(Integer id) {
        return userMapper.get(id);
    }
```

但是这样写之后，无论方法如何执行，`user.get` 中只会有一条数据，且会在第一次执行后保存，之后永远返回缓存的数据。

##### 2）target & targetClass

除了声明与方法相关的属性之外，我们还可以从 root 中拿到当前对象和当前类的信息，比方说我们可以拿到当前类的类名：

```java
    @Cacheable(value = "user.get", key = "#root.targetClass.simpleName")
    public User get(Integer id) {
        return userMapper.get(id);
    }
```

这样写之后，key 就固定在 `"AnnotationUserService"` 了，运行的效果跟上面一样，都是只有一条数据，且会在第一次执行后保存，之后永远返回缓存的数据。

##### 3）args & arg name

前两者通常在开发中都不会使用，常用的方式是引用参数列表的某一个或某几个参数作为 key ，在上述示例中只有一个参数，则 SpEL 表达式可写为 `#id` ，或直接引用 #root 的参数列表。

```java
    @Cacheable(value = "user.get", key = "#root.args[0]")
    public User get(Integer id) {
        return userMapper.get(id);
    }
```

这两种写法，阿熊更倾向于直接引用参数名称，相较于参数索引的方式，引用参数名称的可读性和可维护性会更高一些。

##### 4）caches

由于一个方法可以绑定在多个缓存器中（如 `@Cacheable(value = {"user.get", "user.find"})` ），这就使得我们在声明 key 时，还可以直接引用某一个 cache 的名称，引用的方式是使用索引下标值：

```java
@Cacheable(value = {"user.get", "user.find"}, key = "#root.caches[1]")
public User get(Integer id) {
  return userMapper.get(id);
}
```

上述是一个编写示例，我们使用 “方法名+第一个参数的值” 作为缓存的 key 。

#### 3.1.3 @Cacheable的其他属性

除了指定 key 以及 `keyGenerator` 之外，`@Cacheable` 注解中还有一些其他的属性。

##### condition

`condition` 属性可以指定符合什么条件的数据参与缓存，比方说我们查询 `User` 用户数据的时候，如果查询的 id 是奇数才缓存，偶数不缓存，这种场景下就需要声明 `condition` 属性，写法如下：（ SpEL 表达式可以直接写判断式）

```java
    @Cacheable(value = "user.get", key = "#id", condition = "#id % 2 == 1")
    public User get(Integer id) {
        return userMapper.get(id);
    }
```

##### unless

unless 属性可以指定哪些数据不参与缓存，比方说还是查询 `User` 用户数据，如果查询的结果中 `name` 包含 "jpa" ，那就不予缓存了，这种场景就需要声明 `unless` 属性。可是方法返回值的数据怎么获取呢？同样跟上面一样，我们可以使用 SpEL 表达式来指定。

```java
    @Cacheable(value = "user.get", key = "#id", unless = "#result.name.contains('jpa')")
    public User get(Integer id) {
        return userMapper.get(id);
    }

```

注意看，我们可以直接使用 `#result` 取到方法返回值，然后获取它的 `name` 属性，调用 `contains` 方法就可以完成判断（不要忘记 SpEL 表达式还可以调用对象的方法）。

能不能生效呢？我们不妨用测试代码来试一下，我们两次获取到一个 name 中包含 jpa 的数据，并判断它俩是否为同一个对象。

```java
    @Test
    public void test3() throws Exception {
        User user1 = annotationUserService.get(6);
        User user2 = annotationUserService.get(6);
        System.out.println(user1 == user2);
    }
```

执行测试代码，发现控制台打印了两次 SQL 语句查询，并且判断的结果也是 false ，这就说明 unless 已经生效。

##### sync

在某些并发量很高的场景中，可能一个需要被缓存数据的方法会在同一时间有多个线程执行，假设有两个相同入参的线程同时进入了一个方法，那么即便标注了 `@Cacheable` 注解，也会因为前后没有拉开时间差而导致方法被执行两遍。如何解决这个问题呢？这就需要声明 `@Cacheable` 注解的 `sync = true` ，使缓存变为同步式缓存，这样就可以避免这个问题了。

### 3.2 @CachePut

缓存放进去了，但有些情况下还需要更新缓存，这种情况就需要使用第二个缓存注解：`@CachePut` 。@CachePut 的工作机制是：被标注的方法每次执行完毕后，都会将方法的返回值放入缓存（即覆盖缓存）。下面我们也来使用一下。

既然是更新数据，那我们就来写一套 `update` 的方法吧。

```java
// UserService
public User update(User user) {
    userMapper.updateById(user);
    return user;
}

// UserMapper
@Update("update tbl_user set name = #{name} where id = #{id}")
void updateById(User user);
```

简单这样编写一下就好，不用整的太费劲了，我们的重心是如何使用缓存。

使用 `@CachePut` 的方法也非常简单，直接在 `update` 方法上标注就行，不过有一点要注意！因为 `update` 方法的入参不再是 id ，而是 `User` 对象，所以此处需要显式声明 `key` 为 `#user.id` ：

```java
    @CachePut(value = "user.get", key = "#user.id")
    public User update(User user) {
        userMapper.updateById(user);
        return user;
    }
```

注意，如果是 IDEA 选手的话，当我们写完 value 之后，左侧会出现一个特殊的图标：

当我们点击这个图标时，会发现 IDEA 将光标跳转到了上方的 `get` 方法上，说明 IDEA 也帮我们感知到了 **`user.get`** 这个缓存的所有操作方法！

简单解释一下测试方法要完成的事情。首先我们先查出 id 为 1 的用户，修改其名称后执行 `update` 方法更新数据，之后再尝试查询 id 为 1 的用户并比较。

如果注解式缓存生效的话，则当 `update` 方法执行后，缓存是会被更新的，并且在第二次执行 `get` 方法时不走数据库查询，而是直接从数据库中取出 `name` 被修改过的 `User` 对象。

很明显，整个测试过程中只发起了一次 **select** 查询和一次 **update** 更新，没有第二次从数据库查询，这就说明新的数据已经更新到缓存中，并且在查询时生效。

### 3.3 @CacheEvict

有缓存、有更新，那必然也得有移除。`@CacheEvict` 扮演的就是移除缓存的角色，标注这个注解的方法在执行时，就会触发缓存的移除动作。下面我们也来快速演示一下。

```java
    @CacheEvict(value = "user.get", key = "#id")
    public void deleteById(Integer id) {
        // userMapper.deleteById(id);
        System.out.println("deleteById invoke ......");
    }
```

使用方式跟前面也是完全一样的，相当的简单。不过为了能顺利演示效果，我们这个方法内部就不真正执行 delete 动作了，只是打印一个控制台输出就行。

接下来是测试代码，为了检验执行 `@CacheEvict` 注解标注的方法执行后，缓存是否被正确移除，我们在两次查询后执行 `delete` 方法，随后再次执行 `get` 方法查询同样的数据，看一看是否还会有查询的动作。

```java
    @Test
    public void test5() throws Exception {
        User user1 = annotationUserService.get(1);
        User user2 = annotationUserService.get(1);
        annotationUserService.deleteById(1);
        User user3 = annotationUserService.get(1);
        System.out.println(user1 == user3);
    }
```

执行 `test5` 方法后，可以发现，在 `deleteById invoke` 打印之前只发送了一次 select 的语句，而在 `deleteById` 方法执行后，`get` 方法正确地重新执行了 select 语句，并且这次查询的 `User` 对象，跟之前缓存的对象不是同一个对象，说明缓存的移除也正常执行了。

```ini
[main] c.l.boot.cache.mapper.UserMapper.get : ==>  Preparing: select * from tbl_user where id = ?
[main] c.l.boot.cache.mapper.UserMapper.get : ==> Parameters: 1(Integer)
[main] c.l.boot.cache.mapper.UserMapper.get : <==      Total: 1
deleteById invoke ......
[main] c.l.boot.cache.mapper.UserMapper.get : ==>  Preparing: select * from tbl_user where id = ?
[main] c.l.boot.cache.mapper.UserMapper.get : ==> Parameters: 1(Integer)
[main] c.l.boot.cache.mapper.UserMapper.get : <==      Total: 1
false
```

##### allEntries

顾名思义，`allEntries` 意味着会移除所有数据节点，简单思考也能够理解，如果指定要移除所有缓存数据的话，对应的方法执行一定是类似于 `deleteAll` 或者 `removeAll` 的方法。

##### beforeInvocation

默认情况下，`@CacheEvict` 标记的方法会在方法执行完成后完成缓存移除的动作，而在一些特殊场景下，我们需要先把缓存移除掉，再执行被执行的方法，此时就需要设置 `beforeInvocation` 的值为 true 。

这些属性的使用，各位可以自行测试，小册不再展开测试。

【跟着小册下来之后，我们可以很容易地实现缓存的使用。请各位思考一个问题：现在的缓存数据都放在哪儿了呢？如果需要切换缓存介质该怎么办呢？下一章的内容我们就来解答这个问题】



