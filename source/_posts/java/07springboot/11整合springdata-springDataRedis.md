---
title: 11整合springdata-springDataRedis
toc: true
tags: springboot
categories: 
    - [java]
    - [springboot]
---

上一章我们完成了 SpringBoot 与 SpringDataJPA 的整合，也了解和回顾了一些 SpringDataJPA 的使用及特性。本章我们来了解 SpringData 技术栈中的另一个常用的模块：SpringDataRedis 。顾名思义，SpringDataRedis 是 SpringData 用来与 Redis 高速缓存中间件交互的组件，利用它可以比较方便地完成与 Redis 的访问与操作。

<!--more-->

## 1. Redis回顾

### 1.1 Redis

首先我们先回顾下 Redis 这个中间件。Redis 本身是一个开源的、基于内存的 NoSQL 数据库，不过与其说它是数据库，更不如说它是一个中间件，因为其体积小，效率高等特点，决定了它更适合做数据库之前的缓存层组件。

Redis 本身是基于键值对的数据库，它支持多种类型的数据结构（ String 字符串、List 列表、Set 集合、Hash 哈希、ZSet 有序集合）。Redis 本身支持数据的持久化，可以将内存中的数据保存到磁盘中，待 Redis 重启后可以再次加载到内存中使用。另外，高版本的 Redis 也具备消息队列等特性。

Redis 作为键值对型的存储中间件，它相较于其他同类型的存储件而言拥有更多更多样的数据结构，此外它的主从复制、哨兵等高级特性使得它成为最受欢迎的缓存中间件。

### 1.2 SpringDataRedis

SpringDataRedis 本身作为 SpringData 技术体系中的一个模块，它负责的是与 Redis 之间的交互。SpringData 技术体系的宗旨大家可还记得？它致力于降低与目标数据的访问和操控难度，SpringDataRedis 也不例外，SpringDataRedis 通过一个模板化的封装 `RedisTemplate` 可以实现与 Redis 之间的简单快速交互，底层自动管理和维护了与 Redis 之间建立的连接池，完成与 Redis 的数据操纵。

此外，基于 JSR 107 规范的 SpringCache 也有基于 SpringDataRedis 的实现，它使用 Redis 作为缓存抽象的落地实现，从而实现非常简单的数据缓存功能。有关 SpringCache 结合 Redis 的部分，在小册的整合 Cache 部分再进行讲解。

## 2. SpringBoot整合SpringDataRedis

我们继续复用上一章的工程 `spring-boot-integration-04-springdata` 即可，在 `pom.xml` 中我们需要导入 SpringDataRedis 的场景启动器 `spring-boot-starter-data-redis` ：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

> 注意，在默认情况下 SpringDataRedis 会实用 Lettuce 作为底层与 Redis 交互的基础 API 实现，如果小伙伴需要更换 Jedis 作为连接工具，则需要手动排除 Lettuce 的依赖，并引入 Jedis ：
>
> ```xml
> <dependency>
>     <groupId>org.springframework.boot</groupId>
>     <artifactId>spring-boot-starter-data-redis</artifactId>
>     <exclusions>
>         <exclusion>
>             <groupId>io.lettuce</groupId>
>             <artifactId>lettuce-core</artifactId>
>         </exclusion>
>     </exclusions>
> </dependency>
> <dependency>
>     <groupId>redis.clients</groupId>
>     <artifactId>jedis</artifactId>
> </dependency>
> ```

接下来我们在本地准备一个 Redis 实例，可以是 Linux 虚拟机创建的 Redis 实例，也可以是 Windows / MacOS 中创建的，之后在 `application.properties` 中配置连接地址即可（如果是本机启动，且端口号是默认的 6379 没有修改，则可以不用配置）。

默认情况下，SpringBoot 已经帮我们预先初始化了 Redis 的连接工厂，以及两个 RedisTemplate ，所以我们可以在测试类中直接注入：

```java
@DataRedisTest
public class RedisTest {
    
    @Autowired
    private RedisTemplate<Object, Object> redisTemplate;
    
    @Autowired
    private StringRedisTemplate stringRedisTemplate;
    
}
```

注意上方的测试注解，SpringBootTest 提供了针对 SpringDataRedis 的场景测试器，所以可以直接用 `@DataRedisTest` 而不是 `@SpringBootTest` 。

> 小提示：不过由于我们是将多个场景合并到一个工程中，所以 `@DataRedisTest` 注解可能会不生效，所以小册在演示时仍然选择使用 `@SpringBootTest` 注解。

接下来我们简单写几个 demo ，测试与 Redis 的交互是否正常。

```java
@Test
public void test1() throws Exception {
    redisTemplate.opsForValue().set(111, 222);
    stringRedisTemplate.opsForList().rightPush("name", "aaaa");
    stringRedisTemplate.opsForList().rightPush("name", "bbb");
    stringRedisTemplate.opsForList().rightPush("name", "cc");
}

@Test
public void test2() throws Exception {
    Object value = redisTemplate.opsForValue().get(111);
    System.out.println(value);
    List<String> names = stringRedisTemplate.opsForList().range("name", 0, 5);
    System.out.println(names);
}
```

如上我们编写了两个测试方法，一个负责向 Redis 中存数据，一个负责从 Redis 取数据。先后执行 `test1` 与 `test2` 方法，在执行 `test2` 方法后控制台可以成功打印 `222` 与 3 个字符串，证明 SpringBoot 整合 SpringBootRedis 已经成功。

## 3. 操作Redis详解

SpringDataRedis 操作 Redis 的能力非常强大，下面我们就其中重要和常用的部分进行讲解。

### 3.1 key-value的操作

Redis 的最基础数据结构就是 key-value 键值对，通过 `RedisTemplate` 的 `opsForValue` 方法就可以得到一个对值类型的操作器，从而实现对值类型数据的操作。下面是几个常用的方法（包含部分直接使用 `RedisTemplate` 的方法）。

```java
@Test
public void testValue() throws Exception {
    ValueOperations<Object, Object> valueOperations = redisTemplate.opsForValue();
    // 存入指定值
    valueOperations.set("abc", "def");
    valueOperations.setIfAbsent("qaz", 123);
    valueOperations.setIfAbsent("qaz", 456); // 这次不会生效
    // 存入指定值，并设置过期时间为1分钟（三种方式均可）
    valueOperations.set("qqq", 333, 1, TimeUnit.MINUTES);
    valueOperations.set("aaa", 444, TimeUnit.MINUTES.toMillis(1));
    valueOperations.set("zzz", 555, Duration.ofMinutes(1));
    // 取出指定值
    System.out.println("abc的值：" + valueOperations.get("abc"));
    System.out.println("qaz的值：" + valueOperations.get("qaz")); // 取出的是第一次的值
    // 获取指定key的过期时间（单位：秒）
    System.out.println("qqq的剩余有效时间：" + redisTemplate.getExpire("qqq")); // 快速执行，时间还是1分钟
    // 删除指定值，多个值
    redisTemplate.delete("abc");
    redisTemplate.delete(Arrays.asList("qqq", "aaa", "zzz"));
    // 检查某个key是否存在
    System.out.println("qaz是否存在：" + redisTemplate.hasKey("qaz")); // 值还在
    System.out.println("abc是否存在：" + redisTemplate.hasKey("abc")); // 被删了
}
```

执行 `testValue` 方法，控制台可以输出以下内容，均符合我们的预期。

### 3.2 List列表的操作

List 是 Redis 中的有序可重复列表，与 Java 集合中的 List 基本一致，在 Redis 中 List 被设计为左右均可以压入元素的数据结构，但不能像 Java 的 List 那样从中间插入数据。以下代码列举了 List 的常见操作，涵盖了存入、取出、查询等。

```java
@Test
public void testList() throws Exception {
    ListOperations<Object, Object> listOperations = redisTemplate.opsForList();
    // 向一个列表的左侧压入数据
    listOperations.leftPush("leftList", "aaa");
    // 一次性向左侧压入多个数据
    listOperations.leftPushAll("leftList", 123, 456, 789);
    // 获取一个列表的所有数据
    List<Object> leftList = listOperations.range("leftList", 0, 100);
    System.out.println("设置值后的leftList：" + leftList);

    // 向一个列表的右侧压入数据
    listOperations.rightPush("rightList", "qaz");
    // 一次性向右侧压入多个数据（List是允许重复的）
    listOperations.rightPushAll("rightList", 111, 222, 111, 333, 222, 333);
    List<Object> rightList = listOperations.range("rightList", 0, 100);
    System.out.println("设置值后的rightList：" + rightList);

    // 替换列表中的一个数据
    listOperations.set("leftList", 0, 999);
    System.out.println("替换值后的leftList：" + listOperations.range("leftList", 0, 100));

    // 根据索引获取数据
    System.out.println("leftList的第1个元素是：" + listOperations.index("leftList", 1));
    // 检查指定数据在列表的索引（报错说明redis版本低）
    System.out.println("456在leftList的位置：" + listOperations.indexOf("leftList", 456));

    // 向左弹出一个数据
    System.out.println("弹出leftList左侧的元素：" + listOperations.leftPop("leftList"));
    System.out.println("弹出leftList左侧的元素：" + listOperations.leftPop("leftList"));
    System.out.println("弹出leftList左侧的元素：" + listOperations.leftPop("leftList"));
    System.out.println("弹完数据后的leftList：" + listOperations.range("leftList", 0, 100));
    // 删除指定元素，中间的值决定了删除策略 0代表删除所有指定元素，> 0代表从左往右删除n个指定元素，< 0代表从右往左删除n个指定元素
    listOperations.remove("rightList", 0, 111);
    System.out.println("删除111后的rightList：" + listOperations.range("rightList", 0, 100));
    listOperations.remove("rightList", 1, 222);
    System.out.println("删除右侧第一个222后的rightList：" + listOperations.range("rightList", 0, 100));
}
```

执行测试方法，各位可以根据控制台打印的结果，与我们操作的内容进行对比，看结果是否一致。

```ini
设置值后的leftList：[789, 456, 123, aaa]
设置值后的rightList：[qaz, 111, 222, 111, 333, 222, 333]
替换值后的leftList：[999, 456, 123, aaa]
leftList的第1个元素是：456
456在leftList的位置：1
弹出leftList左侧的元素：999
弹出leftList左侧的元素：456
弹出leftList左侧的元素：123
弹完数据后的leftList：[aaa]
删除111后的rightList：[qaz, 222, 333, 222, 333]
删除左起第1个222后的rightList：[qaz, 333, 222, 333]
```

### 3.3 Set无序集合的操作

Set 存放的是不重复的元素，它天然有去重的特性。Redis 中的集合与 Java 中的 Set 也是比较相似，下面的测试代码中列举了操作 Set 的常用动作。

```java
@Test
public void testSet() throws Exception {
    SetOperations<Object, Object> setOperations = redisTemplate.opsForSet();
    // 添加元素（可多个）
    setOperations.add("names", "aaa", "bbb", "ccc");
    // 获取一个集合的所有元素
    Set<Object> names = setOperations.members("names");
    System.out.println("初次添加元素后的集合：" + names);
    // 从集合中随机取出一个/多个元素
    System.out.println("随机取出一个元素：" + setOperations.randomMember("names"));
    System.out.println("随机取出两个元素：" + setOperations.randomMembers("names", 2));
    // 移除一个指定的元素
    setOperations.remove("names", "aaa");
    System.out.println("删除aaa后的集合：" + setOperations.members("names"));
    // 随机移除一个元素
    setOperations.pop("names");
    System.out.println("随机删除一个元素后的集合：" + setOperations.members("names"));
    //删除整个集合
    redisTemplate.delete("names");
    System.out.println(setOperations.members("names"));
}
```

执行测试方法，观察控制台的输出，与我们的预期也是一致的。

```ini
初次添加元素后的集合：[bbb, ccc, aaa]
随机取出一个元素：bbb
随机取出两个元素：[bbb, ccc]
删除aaa后的集合：[bbb, ccc]
随机删除一个元素后的集合：[bbb]
[]
```

### 3.4 Hash哈希的操作

Hash 作为一种键值对结构，使用它时会产生一种 **key-key-value** 的二重 key 结构，各位也可以理解为 Hash 是一个值为 Map 的特殊结构。下面我们还是通过一些重要常见的操作来解释 Hash 的使用。

```java
@Test
public void testHash() throws Exception {
    HashOperations<Object, Object, Object> hashOperations = redisTemplate.opsForHash();
    // 向Hash中存数据
    hashOperations.put("scores", "bob", 80);
    hashOperations.put("scores", "lily", 90);
    hashOperations.put("scores", "tom", 60);
    // 取出指定值的所有数据
    Map<Object, Object> scores = hashOperations.entries("scores");
    System.out.println("当前scores的所有元素：" + scores);
    // 一次性把一个Map存入Hash
    Map<String, Object> data = new HashMap<>();
    data.put("john", 100);
    data.put("jack", 0);
    hashOperations.putAll("scores", data);
    System.out.println("当前scores的所有元素：" + hashOperations.entries("scores"));
    // 取出指定key指定hash的value
    System.out.println("lily的分数是：" + hashOperations.get("scores", "lily"));
    // 一次性取出多个hash的value
    System.out.println("任取三人的分数是：" + hashOperations.multiGet("scores", Arrays.asList("bob", "tom", "jack")));
    // 取出指定key的所有hash/values
    System.out.println("当前scores的所有人员：" + hashOperations.keys("scores"));
    System.out.println("当前scores的所有分数：" + hashOperations.values("scores"));
    // 移除指定key的指定hash（可以是多个）
    hashOperations.delete("scores", "lily");
    System.out.println("当前scores的所有元素：" + hashOperations.entries("scores"));
}
```

由于 Hash 的操作更像是操作一个 key-key-value 的 Map ，所以操作的方式各位也应该比较熟悉。执行测试代码，各位可以对比控制台的输出来验证自己写的代码是否正确执行。

```ini
当前scores的所有元素：{lily=90, bob=80, tom=60}
当前scores的所有元素：{lily=90, bob=80, tom=60, john=100, jack=0}
lily的分数是：90
任取三人的分数是：[80, 60, 0]
当前scores的所有人员：[jack, bob, lily, tom, john]
当前scores的所有分数：[0, 80, 90, 60, 100]
移除lily后scores的所有元素：{jack=0, bob=80, tom=60, john=100}
```

### 3.5 ZSet有序集合的操作

ZSet 是 Redis 中的有序集合，它集成了 List 和 Set 的双重特征，所以它既可以做到元素有序，也可以保证数据不重复。ZSet 做到元素有序的方式是引入一个排序值，根据排序值对元素进行排序，与 Java 中对应的集合类型更像是 `TreeSet` 。

ZSet 的操作与 List 比较相似，下面还是通过一些 API 的使用来演示。

```java
@Test
public void testZSet() throws Exception {
    ZSetOperations<Object, Object> zsetOperations = redisTemplate.opsForZSet();
    // 设置指定排序值的数据
    zsetOperations.add("pockets", "Pikachu", 5);
    zsetOperations.add("pockets", "Psyduck", 20);
    // 多个数据可以占用一个排序值
    zsetOperations.addIfAbsent("pockets", "Starmie", 30);
    zsetOperations.addIfAbsent("pockets", "Bulbasaur", 30);
    // 取出指定集合的所有元素
    Set<Object> pockets = zsetOperations.range("pockets", 0, 100);
    System.out.println("添加完毕后的宝可梦：" + pockets);
    // 调高/调低指定值得排序值
    zsetOperations.incrementScore("pockets", "Starmie", 5);
    zsetOperations.incrementScore("pockets", "Psyduck", -10);
    System.out.println("调整顺序后的宝可梦：" + zsetOperations.range("pockets", 0, 100));
    // 获取指定值的排序值和索引值(从0开始)
    System.out.println("可达鸭的当前排序值：" + zsetOperations.score("pockets", "Psyduck"));
    System.out.println("可达鸭的当前索引位置：" + zsetOperations.rank("pockets", "Psyduck"));
    // 获取指定排序值之间的元素
    System.out.println("0-20分之间的宝可梦：" + zsetOperations.rangeWithScores("pockets", 0, 20).stream()
                       .map(ZSetOperations.TypedTuple::getValue).collect(Collectors.toList()));
    // 移除指定元素（remove，不再演示）
    // 移除指定索引值范围的元素（闭区间）
    zsetOperations.removeRange("pockets", 2, 3);
    System.out.println("删除指定位置的数据后的宝可梦：" + zsetOperations.range("pockets", 0, 100));
    // 移除指定分数区间的元素（闭区间）
    zsetOperations.removeRangeByScore("pockets", 5, 10);
    System.out.println("删除指定排序值后的宝可梦：" + zsetOperations.range("pockets", 0, 100));
}
```

可以发现，在操作的过程中虽说大面上差不离，但细节上还是有区别的。执行测试代码，控制台输出如下信息，各位可以跟着体会一下。

```ini
添加完毕后的宝可梦：[Pikachu, Psyduck, Starmie, Bulbasaur]
调整顺序后的宝可梦：[Pikachu, Psyduck, Bulbasaur, Starmie]
可达鸭的当前排序值：10.0
可达鸭的当前索引位置：1
0-20分之间的宝可梦：[Pikachu, Psyduck, Bulbasaur, Starmie]
删除指定位置的数据后的宝可梦：[Pikachu, Psyduck]
删除指定排序值后的宝可梦：[]
```

【有关 SpringDataRedis 的使用本章就讲这么多，Redis 本身还有一些高级特性，譬如主从、哨兵、Lua 脚本等，这部分内容在 SpringDataRedis 中也有使用，感兴趣的小伙伴可以参照官方文档和有关资料进行学习。】

