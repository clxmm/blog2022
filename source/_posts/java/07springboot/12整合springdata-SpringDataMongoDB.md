---
title: 12整合springdata-SpringDataMongoDB
toc: true
tags: springboot
categories: 
    - [java]
    - [springboot]
---

前面我们了解了 SpringDataJPA 与 SpringDataRedis 的使用，本部分的最后一章我们来整合 MongoDB 数据库。MongoDB 被很多人称为“最像关系型数据库的非关系型数据库”，它的使用场景和使用热度也是比较高的，下面我们开始学习。

## 1. MongoDB回顾

首先我们先回顾 / 了解 MongoDB 数据库。

MongoDB 本身是一个天然分布式的**文档**数据库，它是一个非关系型数据库，但它的一些特性使得看起来很像关系型数据库。MongoDB 对于数据的存储是一种类似于 json 数据形式的 **bson** 格式（而不是我们理解的那种 word 、pdf 之类的文档的概念），它相较于 json 更能存储一些复杂类型和规模的数据。

MongoDB 从 2007 年开始起步，随着版本不断地迭代，其支持的特性越来越多，目前来看 MongoDB 已经是一个天然支持分布式的、可以处理海量数据的、支持分片存储、高可用、分布式事务等高级特性的数据库。

MongoDB 的成熟度和稳定性很高，在所有关系型数据库与非关系型数据库的综合排名中，MongoDB 是唯一一个排在前 5 名的非关系型数据库。

简单看来，MongoDB 具备以下这些主要特点：

<!--more-->

- 基于文档存储数据，数据存储的宽度和深度都很灵活；
- 天然的分布式，支持数据的分片等；
- 支持数据镜像，这使得 MongoDB 的扩展性很强；
- 基于 json 格式的数据存储带来了特别的查询方式；
- ....

总的来看，MongoDB 可以在一定程度上代替传统的关系型数据库，作为存储数据的主要介质

## 2. SpringBoot整合SpringDataMongoDB

下面我们来使用 SpringBoot 整合 SpringDataMongoDB 。

### 2.1 准备MongoDB环境

MongoDB 的安装比较简单，在 Windows 环境下有相应的安装包，Linux 下也可以使用安装包或者 Docker 进行安装，小册选择在 Linux 环境上利用 Docker 快速构建一个 MongoDB 的数据库。

在 Linux 主机上安装好 Docker 后，只需要执行 `docker pull mongo` 命令，就可以拉取 MongoDB 的最新镜像，再使用 docker run 命令即可启动一个 MongoDB 的实例。

```sh
docker run -d --name mongo -p 27017:27017 mongo
```

启动成功后，使用数据库连接工具连接到 MongoDB 可以看到，当前的 MongoDB 实例是一个空实例。

之后我们创建一个空的数据库 `spring-data` ，不需要创建集合（注意这里的“集合”也就是关系型数据库中“表”的概念）：

这样一个简单的 MongoDB 环境就准备就绪了。

### 2.2 SpringData整合

SpringDataMongoDB 同样有在 SpringBoot 场景中的启动器，我们继续修改 `spring-boot-integration-04-springdata` 的 `pom.xml` ，在依赖部分添加 `spring-boot-starter-data-mongodb` 的坐标：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-mongodb</artifactId>
</dependency>
```

之后要做的就是配置 MongoDB 的服务器地址了，小册编写时采用的是虚拟机搭建，所以这里需要配置好虚拟机的地址和对应的数据库。

```properties

spring.data.mongodb.host=192.168.42.103
spring.data.mongodb.port=27018
spring.data.mongodb.database=spring-data
```

仅仅需要编写几行配置就可以了，下面就可以编写实际的代码了。

### 2.3 简单模块的CRUD

与第 10 章类似，我们来写一个对 `User` 的简单维护。

首先我们来编写实体类，注意与 SpringDataJPA 不同，SpringDataMongoDB 对应的注解不再是 `@Table` 而是 `@Document` ，其他的注解是类似的：

```java
import org.springframework.data.mongodb.core.mapping.Document;
import org.springframework.data.mongodb.core.mapping.Field;

@Document(collection = "doc_user") // 用value或者collection都可以
public class User {
    
    @Id
    private String id;
    
    @Field("username")
    private String name;
    
    private Integer age;
    
    // getter  setter  toString ......
}
```

注意一点，MongoDB 中的 id 默认是 32 位不带短横线的 uuid ，所以默认情况下我们编写的 id 类型应为 `String` ，除非我们有自定义的主键生成器予以配合，才可以在 id 属性设置其他的数据类型。

```java
import org.clmm.mongo.entity.User;
import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.stereotype.Repository;

/**
 * @author clxmm
 * @Description
 * @create 2023-01-10 20:49
 */
@Repository
public interface UserRepository extends MongoRepository<User, Integer> {

}
```

有以上两个简单的类和接口，下面就可以进行测试了。在 test 目录下新建一个 `MongoTest` 的单元测试类，并向其中注入 `UserRepository` ，编写测试方法新增一条数据（注意标准的说法是“文档”）：

```java
//@DataMongoTest 仅用来测试SpringDataMongoDB相关的注解
@SpringBootTest
public class MongoTest {
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    public void test1() throws Exception {
        User user = new User();
        // id让MongoDB生成，此处不需要填充
        user.setName("mongo");
        user.setAge(20);
        userRepository.save(user); // 也可以用insert方法
    }
}
```

执行 test1 方法，控制台可以打印出执行成功的提示，来到数据库中，可以发现 SpringDataMongoDB 帮我们成功地创建了 `doc_user` 集合：

并且也成功地插入了这条数据，顺便还记录了这条数据应该映射的 class 类型：

![](/img/202301/12mongodb.png)

下面我们把这条数据查出来，与 SpringDataJPA 类似，`MongoRepository` 也有相同的 `findAll` 系列方法，我们直接查全表即可：

```java
@Test
public void test2() throws Exception {
    List<User> users = userRepository.findAll();
    System.out.println(users);
}
```

执行结果必然是可以成功查出：

```ini
[User{id=63bea87f0521722339471601, name='mongo', age=20}]
```

## 3. MongoDB维护数据

--------

下面我们就 `MongoRepository` 与 SpringData 系列都有的 `XXXTemplate` ，也即 MongoDB 对应的 `MongoTemplate` 进行讲解，各位了解其中的一些常用操作即可。

### 3.1 保存数据

除了使用 `MongoRepository` 接口的 `save` 方法之外，`MongoTemplate` 也提供了 `save` 方法，以及 `insert` 系列方法，可以保存 1 到多条数据（文档）。

```java
@Test
public void testSave() throws Exception {
    User user = new User();
    user.setName("template");
    user.setAge(12);
    mongoTemplate.save(user);
}
```

执行上述测试方法，也可以成功向 `doc_user` 集合中插入一条文档。

另外 `MongoTemplate` 还支持我们使用链式 API 调用其 `insert` 方法，用来获取一个带主键 id 的持久化对象：

```java
@Test
    public void testSave1() throws Exception {
        User user = new User();
        user.setName("template");
        user.setAge(12);

        User userWithId = mongoTemplate.insert(User.class).one(user);
        System.out.println(user);
        System.out.println(userWithId);

        System.out.println(user == userWithId);
    }
```

如上述测试代码所示，我们分别打印传入的 `User` 以及返回的持久化 `User` 对象，观察两者是否有所差别：

```ini
User{id=63beab85786e915c093da83e, name='template', age=12}
User{id=63beab85786e915c093da83e, name='template', age=12}
true
```

是不是有些出乎意料？期初我们创建的没有 id 值的 `User` 对象，在执行完 `insert` 方法后竟然 id 被填充了！

那顺道再大胆猜测一下：这两个 `User` 对象不会是同一个吧！我们尝试打印 `user == userWithId` ：

竟然真的是同一个对象！那自然也就说明是 SpringDataMongoDB 帮我们做的填充动作。

### 3.2 更新数据

SpringDataMongoDB 的更新数据简单区分还是 `MongoRepository` 接口与 `MongoTemplate` 的操作，下面分开讲解。

#### 3.2.1 Repository的方式

`MongoRepository` 接口中并没有专门的 `update` 方法，而是跟 SpringDataJPA 一样，用 `save` 方法作为保存 / 更新的统一入口。

注意跟 SpringDataJPA 的区别！因为 SpringDataJPA 本身基于关系型数据库，默认会装配事务管理器，更新需要基于事务 + 快照更新；而 SpringDataMongoDB 默认是不会装配事务管理器的，所以需要我们手动调用 `save` 方法以完成更新操作。下面的测试代码中演示了一个从数据库中查询出 `User` 数据后改 `age` 属性的简单示例，在执行 `save` 方法后，我们再把这条数据查出来打印，检查是否已经修改。

```java
@Test
public void testUpdate() throws Exception {
  Optional<User> op = userRepository.findById("63beab85786e915c093da83e");
  op.ifPresent(user -> {
    user.setAge(60);
    userRepository.save(user);
  });

  userRepository.findById("63beab85786e915c093da83e").ifPresent(System.out::println);
}
```

如果不执行 `save` 方法，则第二次查询的结果仍然为旧数据：

只有执行 `save` 方法后，再次查询的数据才是被修改过的：

#### 3.2.2 MongoTemplate的方式

`MongoTemplate` 为我们提供了三种更新相关的方法，分别是链式 API 调用、更新一个、更新多个。

这个链式 API 调用看上去比较新鲜，我们可以先来了解一下它。下面是一个简单的示例，在执行 `update` 方法后拿到一个`ExecutableUpdate` 对象，随后执行其 matching 匹配数据（类似于 SQL 中的 where ），匹配的方式使用 `Criteria` 的静态方法进行判断，示例代码中选择的是类似于 `name = 'mongo'` 的过滤条件；匹配完成后，接下来使用 `replaceWith` 可以将新数据作为替换体覆盖进去。

```java
@Test
public void testUpdate() throws Exception {
    User replace = new User();
    replace.setAge(100);
    mongoTemplate.update(User.class).matching(Criteria.where("name").is("mongo"))
            .replaceWith(replace).as(User.class).findAndReplaceValue();

    userRepository.findById("62f0eb78332a95603b13c35b").ifPresent(System.out::println);
}
```

如此执行完毕后，属性 `name` 为 `mongo` 的文档就会被更新，但是当我们再次取出这条数据并打印时，出现的结果似乎不太符合我们的预期：

```ini
User{id=62f0eb78332a95603b13c35b, name='null', age=100}

```

`name` 属性怎么不见了呢？原因是我们在替换新数据时，的确没有给 `name` 属性赋值！所以上面的这种更新数据的方式相当于是直接替换！日常开发中不是特别常用。

下面介绍的另一种方式就是相对常用的了，它的更新数据方式是以 `Update` 的静态方法 `update` 链式指定需要修改的属性值，如下面的示例代码所示：

```java
@Test
public void testUpdate() throws Exception {
    mongoTemplate.updateFirst(Query.query(Criteria.where("name").is("mongo")), Update.update("age", 200), User.class);
    userRepository.findById("62f0eb78332a95603b13c35b").ifPresent(System.out::println);
}
```

这样更新数据后，`name` 属性不会丢失，仅仅是 `age` 属性变化而已。

```
User{id=62f0eb78332a95603b13c35b, name='mongo', age=200}

```

### 3.3 删除数据

`MongoRepository` 中定义的 `delete` 方法与 `JpaRepository` 基本一致，都是非常直观易懂的方法：

我们可以使用 `deleteById` 完成一个最基本的单条数据的删除。

```java
@Test
public void testDelete() throws Exception {
    userRepository.deleteById("62f0eb78332a95603b13c35b");
    userRepository.findById("62f0eb78332a95603b13c35b").ifPresent(System.out::println);
}
```

对比而言，`MongoTemplate` 中定义的删除方法在命名上不太一样，它的删除方法系列命名是 `remove` ，而且还有与 `find` 系方法相结合的 `remove` 系方法：

与 `update` 类似，我们同样可以传入一个链式构造的 `Query` 对象，来指定满足特定条件的数据进行删除操作。下面的示例代码中我们将 30 岁以上的所有 `User` 删除：

```java
@Test
public void testDelete() throws Exception {
    mongoTemplate.remove(Query.query(Criteria.where("age").gt(30)), User.class);
    mongoTemplate.findAll(User.class).forEach(System.out::println);
}
```

执行 `testDelete` 方法，可以发现数据库中只剩一条 12 岁的数据了。

```
User{id=62f0eb8cc3f47f6068921758, name='template', age=12}

```

### 3.4 查询数据

无论使用 `MongoRepository` 还是 `MongoTemplate` ，在查询数据的时使用的方法都是相似的，下面是几个简单的示例，分别有查全表、排序查、条件查、查单条：

```java
@Test
public void testQuery() throws Exception {
    System.out.println("userRepository.findAll(): " + userRepository.findAll());
    System.out.println("order by: " + userRepository.findAll(Sort.by(Sort.Order.desc("age"))));
    
    System.out.println("mongoTemplate.find: " + mongoTemplate.find(Query.query(Criteria.where("age").lt(18)), User.class));
    System.out.println("mongoTemplate.findById: " + mongoTemplate.findById("62f0f24e618cfd7eb146b14c", User.class));
}
```

执行以上测试代码，控制台可以输出相应的数据，供各位参考。

```
userRepository.findAll(): [User{id=62f0eb8cc3f47f6068921758, name='template', age=12}, User{id=62f0f24e618cfd7eb146b14c, name='mongo', age=20}]
order by: [User{id=62f0f24e618cfd7eb146b14c, name='mongo', age=20}, User{id=62f0eb8cc3f47f6068921758, name='template', age=12}]
mongoTemplate.find: [User{id=62f0eb8cc3f47f6068921758, name='template', age=12}]
mongoTemplate.findById: User{id=62f0f24e618cfd7eb146b14c, name='mongo', age=20}
```

可能有小伙伴注意到了，MongoRepository 中怎么没有定义条件查呢？别着急，这就是下面要讲解的 SpringDataMongoDB 中的扩展内容。

## 4. SpringDataMongoDB的扩展使用

### 4.1 方法命名接口

作为 SpringData 系列的模块，SpringDataMongoDB 同样具备 Dao 接口的方法命名式查询，以此来完成条件查询。以下是几个简单示例：

```java
@Repository
public interface UserRepository extends MongoRepository<User, String> {
    
    User findByName(String name);
    
    List<User> findAllByNameLike(String nameLike);
    
    List<User> findAllByAgeGreaterThan(Integer age);
    
    List<User> findAllByNameInOrderByAgeDesc(List<String> names);
}
```

具体的测试代码，各位可以自行构造测试来体会，小册不再展开。

### 4.2 MongoDB的事务

MongoDB 从 4.0 之后支持了与关系型数据库类似的 ACID 事务，这也使得 MongoDB 在进行数据存储时有了更可靠的机制支撑。SpringDataMongoDB 的事务控制与使用传统关系型数据库的模型是类似的，都是使用“事务管理器 + 事务模板/增强器”的方式。下面简单介绍 SpringDataMongoDB 中如何使用事务。

#### 4.2.1 已有集合的事务操作

我们先拿当前已有的 `doc_user` 集合进行带事物的操作。在编写事务操作代码之前，需要先向 IOC 容器中注册一个基于 MongoDB 的事务管理器，因为默认情况下 SpringDataMongoDB 不会进行事务管理。

```java
@Configuration(proxyBeanMethods = false)
public class MongoConfiguration {
    
    @Bean
    public MongoTransactionManager mongoTransactionManager(MongoDatabaseFactory databaseFactory) {
        return new MongoTransactionManager(databaseFactory);
    }
}
```

之后的编码套路跟传统的 SpringDataJPA 、MyBatis 等是一样的，都是在需要开启事务的方法上标注 `@Transactional` 注解即可。下面的示例代码中开启了事务，并人为地制造了一个最经典的除零异常：

```java
@Service("mongoUserService")
public class UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    @Transactional(rollbackFor = Exception.class)
    public void saveAndPrint() {
        User user = new User();
        user.setName("save");
        user.setAge(20);
        userRepository.save(user);
        
        int i = 1 / 0;
    
        System.out.println(userRepository.findAll());
    }
}
```

另在测试类中注入，并调用其方法。

```java
@Test
public void testUserService() throws Exception {
    userService.saveAndPrint();
}
```

执行 `testUserService` 方法，控制台可以正常打印除零异常，此时我们观察数据库中 `doc_user` 集合的数据，可以发现是没有新的数据添加的，证明 MongoDB 的事务控制起到了作用。

#### 4.2.2 新集合的事务操作？

同样的套路，我们再创建一个全新的集合试一下，相应的测试代码如下：

```java
@Document(collection = "doc_dept")
public class Department {
    
    @Id
    private String id;
    
    private String name;
    
    private String tel;
    
    // getter setter toString ......
}
```

```java
@Service
public class DepartmentService {
    
    @Autowired
    private MongoTemplate mongoTemplate;
    
    @Transactional(rollbackFor = Exception.class)
    public void save() {
        Department department = new Department();
        department.setName("dept");
        department.setTel("1234321");
        mongoTemplate.save(department);
    }
}
```

根据前面的经验来看，当数据库中没有这个 doc_dept 集合时，MongoDB 会帮我们自动创建，然而当我们执行测试代码，调用 DepartmentService 的 save 方法时，尽管 save 方法的内部没有抛出异常，但测试还是失败了：

```
Command failed with error 20 (IllegalOperation): 'Transaction numbers are only allowed on a replica set member or mongos' on server 192.168.217.128:27017. The full response is {"ok": 0.0, "errmsg": "Transaction numbers are only allowed on a replica set member or mongos", "code": 20, "codeName": "IllegalOperation"}

```

为什么会出现这样的报错信息呢？这就需要各位了解一个 MongoDB 中事务机制的限制条件：MongoDB 的事务只对已经存在的集合起作用，如果想要进行操作的集合在 MongoDB 中还没有创建，则必须事先创建该集合，否则当该集合进行插入操作时，会报类似于上面的错误。

【有关 SpringDataMongoDB 的使用本章就先讲解到这儿，MongoDB 本身作为一个最接近关系型数据库的非关系型数据库，它的诸多特性也帮助它成为比较流行的数据存储介质之一，感兴趣的小伙伴可以结合官方文档和有关资料进行更深层次的学习 】