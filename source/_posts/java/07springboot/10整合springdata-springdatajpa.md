---
title: 10整合springdata-SpringDataJpa
toc: true
tags: springboot
categories: 
    - [java]
    - [springboot]
---

在我们的日常项目开发中，对于持久层的选型，除了 MyBatis 以及其扩展的 MyBatisPlus 之外，另一个主流的选择是使用 SpringData 。不过请各位注意，SpringData 本身是一整套访问数据的解决方案，对于关系型数据库的访问，SpringData 选择使用 JPA 作为具体的方案。


<!--more-->

## 1. SpringData概述

首先我们先了解和回顾 SpringData 。SpringData 本身是一套访问数据的编程模型，它提供了对不同种类数据源的访问实现，并在具体的实现中尽可能地提供统一编程风格的 API ，供我们开发者使用。

请注意，SpringData 不只有对关系型数据库的支持，更是对各种非关系型数据库（如 Redis 、MongoDB 、ElasticSearch 、LDAP 、Cassandra 等）提供了支持。

SpringData 给我们提供了一些非常强大的特征：

- 强大的 Repository 模型与自定义对象映射；
- Repository 模型中的灵活方法名查询；
- 方便与 Spring 、SpringWebMvc 、SpringBoot 的集成；

## 2. SpringDataJPA概述

正如上一小节的内容所属，SpringData 中包含多个模块的支持，而 SpringDataJPA 只是 SpringData 的其中一个模块而已。SpringDataJPA 的设计旨在简化基于 JPA 规范框架的数据访问方式。使用 SpringDataJPA 之后，开发者在编写单表的数据层操作时只需要编写 Dao 接口，即可拥有单表 CRUD 的能力。

回过头来再看下 JPA ，JPA 本身是 Java Persistence API 的简称，即 Java 持久层的 API ，它是一套基于 ORM 的规范，注意是规范不是具体的实现。正如接口与实现类一样，JPA 仅仅定义了一些 API 接口，具体的实现模型由各个 ORM 框架实现。SpringDataJPA 默认使用 Hibernate 作为具体的 JPA 规范实现。

## 3. SpringBoot整合SpringDataJPA

下面我们来简单整合一下 SpringDataJPA 。

##### 导入依赖

`pom.xml` 中仅需导入 SpringDataJPA 的场景启动器，以及 MySQL 的数据库驱动即可。

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
</dependency>
```

##### 实体模型类

对于实体模型类的编写，主框架部分是一样的，只不过标注的注解不太一样。区别于 MyBatisPlus ，SpringDataJPA 标注的注解几乎全部来自 JPA 规范，也即 `javax.persistence` 包下（注意是 JPA 不是 Hibernate 的！！！）。

```java
import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;
import javax.persistence.Table;

@Entity
@Table(name = "tbl_user")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;
    
    private String name;
    
    private String tel;
    
    // ......
}
```

简单解释下上面注解的作用。

- `@Entity` 注解的作用是告诉 JPA ，当前被标注的类是一个与数据库建立 ORM 映射的实体类，它会关联数据库的一张表；
- `@Table` 注解标注了当前实体类对应的数据库的表名；
- `@Id` 注解标注的属性会被 JPA 认定是对应表的主键；
- `@GeneratedValue` 注解指定了主键的生成策略，其中示例代码中的 `IDENTITY` 代表的是 MySQL 中的主键自增。

##### Dao层

SpringDataJPA 的一个非常重要的特性是提供了一些 Repository 的接口供我们使用，在 SpringDataJPA 中，一个 Dao 接口只需要继承 `JpaRepository` 接口，即可立即拥有单表 CRUD 的能力；如果继承 `JpaSpecificationExecutor` ，则拥有特殊的编程式条件查询的能力（类似于 QBC ）。

```java
@Repository
public interface UserDao extends JpaRepository<User, Integer>, JpaSpecificationExecutor<User> {
    
}
```

##### 编写配置

接下来是 SpringBoot 全局配置文件的编写，除了配置必要的数据源以外，我们还需要简单编写一个 JPA 的配置，如下所示。

```properties
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
spring.datasource.url=jdbc:mysql://localhost:3306/sringboot-dao?characterEncoding=utf8
spring.datasource.username=root
spring.datasource.password=123456

# JPA连接数据库的模式为MySQL
spring.jpa.database=mysql
# 对数据库表结构的维护策略为“更新”
spring.jpa.hibernate.ddl-auto=update
# 开启控制台SQL打印
spring.jpa.show-sql=true
```

再就是主启动类了，除了必要的 `@SpringBootApplication` 注解之外，还需要额外标注一个 `@EnableJpaRepositories` ，它的作用是扫描标注有 `@Entity` 注解的那些实体类，并激活与数据库之间的 ORM 映射关系。

##### 单元测试

下面我们就可以编写单元测试来体会 SpringDataJPA 的强大了。SpringBoot 针对特殊的场景有做专门的注解，其好处是快速的场景测试，排除其他模块的组件加载，提高测试速度。针对 SpringDataJPA 的测试，有一个专门的注解 `@DataJpaTest` 。所以单元测试类中可以使用 `@DataJpaTest` 注解代替 `@SpringBootTest` 。

> 请注意，当标注 `@DataJpaTest` 注解后，SpringBoot 会认为我们使用了内存数据库作为测试主体，而实际上我们还是希望使用上一部分中的那个 `springboot-dao` 数据库，所以需要额外标注一个 `@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)` 注解，告诉 SpringBoot 不要给我们替换测试用的数据库。
>
> 另外，如果测试的数据需要真实地插入到数据库，则不要使用 `@DataJpaTest` ，而是选择原始的 `@SpringBootTest` 注解。因为 `@DataJpaTest` 标注的测试类，会在测试完毕后回滚所有事务（SpringBoot 为了保证数据库中原有数据的完整性，以及不污染数据库）。

```java
// @DataJpaTest 不真实保存数据时使用
@SpringBootTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
public class JpaTest {
    
    @Autowired
    private TransactionTemplate transactionTemplate;
    
    @Autowired
    private UserDao userDao;
    
    @Test
    public void test1() throws Exception {
        transactionTemplate.executeWithoutResult(status -> {
            List<User> users = userDao.findAll();
            System.out.println(users);
        });
    }
    
    @Test
    public void test2() throws Exception {
        transactionTemplate.executeWithoutResult(status -> {
            User user = new User();
            user.setName("test springdatajpa");
            user.setTel("123321");
            userDao.save(user);
        });
    }
}

```

请注意！JPA 的操作需要配合事务完成，所以我们编写的测试代码中需要使用到编程式事务 `TransactionTemplate` 配合完成。

编写如上述代码所示，注入 `UserDao` 后，可以直接调用其 `findAll` 方法来实现列表查询，调用 `save` 方法可以完成数据的保存，当然写数据的动作需要在 `TransactionTemplate` 之中完成。

到此为止，我们就完成了 SpringBoot 与 SpringDataJPA 的整合。

## 4. JPA的使用方式

JPA 本身的使用会比 MyBatis 稍微复杂一点，本节内容会讲解 JPA 中常用的一些操作。

### 4.1 主键生成策略

```java
@Entity
@Table(name = "tbl_department")
public class Department {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;
    
    // ......
}
```

回过头来看一眼实体类上的这个 `@GeneratedValue` 注解，JPA 给我们提供的自带的主键生成策略包含以下几种：

- **IDENTITY** ：MySQL 等支持列数据自增的数据库可以使用的主键自增策略；
- **SEQUENCE** ：Oracle 等支持自增序列的数据库可以使用的策略；
- **Table** ：使用一张单独维护的自增序列表维护主键；
- **AUTO** ：自动选择上述三种策略的其中一种（根据底层数据库选择）。

如果以上的主键生成策略不够用，我们也可以借助 Hibernate 内置的一些策略来生成，此时就不能再使用 `@GeneratedValue` 的 `strategy` 属性了，而是使用另外一个属性：`generator` 。下面简单演示一下如何使用 UUID 来生成主键。

Hibernate 默认提供了 uuid 的主键生成实现，我们只需要使用 Hibernate 的一个注解 `@GenericGenerator` ，声明生成器的 `name` 与 `strategy` 即可，其中 `strategy` 可以填的值是固定的，此处填写 `uuid` 对应的生成策略就是 32 位不带短横线的 uuid 值。此外还需要在 `@GeneratedValue` 注解中标注生成器的名称，也就是刚写好的 `userIdGenerator` 。

```java
    @GeneratedValue(generator = "userIdGenerator")
    @GenericGenerator(name = "userIdGenerator", strategy = "uuid")
    private String id;
```

如此编写完毕后，即可实现 uuid 的主键生成。

> `@GenericGenerator` 注解的 `strategy` 属性可以填写的值：
>
> - assigned ：程序自行维护主键
> - identity ：同上
> - sequence ：同上
> - select ：与 identity 类似，借助数据库触发器实现
> - increment ：与 identity 类似，自增在本地缓存实现，故不适合集群项目
> - foreign ：同 TABLE
> - native ：同 AUTO
> - uuid ：生成 32 位的 uuid
> - guid ：与数据库联动的 uuid 生成，需要先从数据库中查出一个 uuid 后进行处理生成
> - hilo ：使用一个高/低位算法生成的 long 、short 或 int 类型的标识符，通过特殊算法生成
>
> - seqhilo ：sequence + hilo 的复合实现

### 4.2 数据库表结构同步策略（Hibernate）

JPA 底层默认使用的 ORM 框架是 Hibernate ，Hibernate 的一个强大的特性是实体类的 ORM 映射与数据库表结构对应。由于 Hibernate 本身是一个全自动 ORM 框架，它要求我们的实体类能够尽可能严格映射数据库的表结构，由此它也提供了一个特性：当程序启动时，如何处理 ORM 映射与数据库表结构的对应关系。

这个概念有些晦涩，换一种表述方式就好理解了：假设我们在项目中新添加一个 `Department` 类，并给予它与数据库表结构的映射关系：

```java
@Entity
@Table(name = "tbl_department")
public class Department {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;
    
    private String name;
    
    private String address;
    
    // ......
}
```

注意，此时表 `tbl_department` 还不存在。

在上面的 `application.properties` 配置文件中，我们有编写一个配置项：`spring.jpa.hibernate.ddl-auto=update` ，这个设定代表了当 Hibernate 检测到实体类的映射关系与数据库表结构不一致时，将会比对这些 ORM 映射关系，并对不一致的部分进行更新操作，以达到与数据库表结构完全相同的效果。

接下来我们直接来启动项目，观察控制台的 SQL 打印：

```sql
Hibernate: create table tbl_department (id integer not null auto_increment, address varchar(255), name varchar(255), primary key (id)) engine=InnoDB

```

可以发现，数据库中本来没有 `tbl_department` 表，但是 Hibernate 帮我们成功创建了，这就证明 `ddl-auto` 配置已经生效了。

`ddl-auto` 配置的值有如下几个可选项：

- **create** ：每次程序启动时都会删除数据库中原有的表，并基于当前的 ORM 定义创建新的表；
- **create-drop** ：每次程序启动时删除数据库中原有的表，并基于当前的 ORM 定义创建新的表；程序停止时会删除 ORM 定义映射的表；
- **update** ：每次程序启动时会检查当前 ORM 定义的映射与数据库表结构，并对不一致的映射的表予以创建 / 修改 / 删除，完全一致的表则不会变动；
- **validate** ：每次程序启动时仅检查 ORM 定义的映射关系，如果有不一致的则会抛出异常，程序停止；
- **none** ：功能关闭。

对于一般使用来看，大多数的项目在开发中会选择使用 update 策略，或者 none 不处理的策略。

### 4.3 CRUD操作

JPA 原生的 CRUD 操作相对有些难记（如保存操作叫 `persist` ，更新叫 `merge` 等。。。），好在 SpringDataJPA 的 Repository 提供的方法正常一点。下面简单列举几个 SpringDataJPA 中的单表操作。

**添加操作**，用 `save` 居多，也有 `saveAll` 等其他方法可用。

```java
    @Test
    public void testSave() throws Exception {
        Department department = new Department();
        department.setName("测试部门");
        department.setAddress("test test");
        departmentDao.save(department);
    }
```

**更新操作**，它没有具体的方法，只需要在事务中修改持久态的属性即可，事务提交后会自动发 update 语句。

```java
    @Test
    public void testUpdate() throws Exception {
        transactionTemplate.executeWithoutResult(status -> {
            Department department = departmentDao.getById(1);
            department.setName("测试修改部门");
            departmentDao.save(department);
        });
    }
```

```
Hibernate: select department0_.id as id1_0_0_, department0_.address as address2_0_0_, department0_.name as name3_0_0_ from tbl_department department0_ where department0_.id=?
Hibernate: update tbl_department set address=?, name=? where id=?
```

**删除操作**，也是放在事务中操作即可，用很多 delete 系列方法可选。

```
    @Test
    public void testDelete() throws Exception {
    	transactionTemplate.executeWithoutResult(status -> {
    	    departmentDao.deleteById(1);
        });
    }
```

**列表查询**，Repository 中提供了一系列查询方法，包括查全部、分页查、条件查、检查是否存在等。

```java
    @Test
    public void testQuery() throws Exception {
        List<Department> all = departmentDao.findAll();
        System.out.println(all);
        
        boolean exists = departmentDao.existsById(20);
        System.out.println(exists);
    
        Page<User> userPage = userDao.findAll(PageRequest.of(0, 2));
        System.out.println(userPage.getTotalElements());
        System.out.println(userPage.getTotalPages());
        System.out.println(userPage.getContent());
    }

```

注意观察以上几个查询动作对应的 SQL 打印，对于这些查询，Hibernate 都可以非常优秀地完成工作。

```ini
Hibernate: select department0_.id as id1_0_, department0_.address as address2_0_, department0_.name as name3_0_ from tbl_department department0_
[Department{id=1, name='测试修改部门', address='test test'}]

Hibernate: select count(*) as col_0_0_ from tbl_department department0_ where department0_.id=?
false

Hibernate: select user0_.id as id1_1_, user0_.name as name2_1_, user0_.tel as tel3_1_ from tbl_user user0_ limit ?
Hibernate: select count(user0_.id) as col_0_0_ from tbl_user user0_
3
2
[User{id=1, name='test mybatis', tel='1234567'}, User{id=2, name='test mybatisplus', tel='7654321'}]
```

### 4.4 JPQL查询语言

JPA 的使用中不推荐我们使用原生的 SQL 语句进行查询，而是有一个专门的查询语言：JPQL ，也就是 JPA 查询语言。JPQL 相较于原生的 SQL 语句来讲，一个最大的好处是**语句与数据库无关性**，由于有 ORM 映射关系，所以我们在程序中编写时完全可以用实体类型代替数据库中具体的表，通过 ORM 框架的映射，可以将 JPQL 转化为数据库看得懂的 SQL 。

在原生的 JPA 中就已经可以使用 JPQL 进行查询，下面通过几个简单示例来演示 JPQL 的使用。

**原生的 JPQL 使用**：原生的 JPQL 需要拿到 `EntityManager` ，操作它的 API 构造 JPQL 查询的 `Query` 对象，随后调用其 `getResultList` 或 `getSingleResult` 分别进行列表查询和单条数据的查询。由于原生的查询方式相对复杂，小伙伴们仅作了解或者回顾即可。

```java
    @Test
    public void testJPQL() throws Exception {
        EntityManager entityManager = entityManagerFactory.createEntityManager();
        List<User> userList = entityManager.createQuery("from User").getResultList();
        System.out.println(userList);
    }
```

**Dao 接口的查询**：SpringDataJPA 为我们提供的 JPQL 查询方式是在 Dao 接口上标注 `@Query` 注解，以完成 JPQL 语句的编写与查询。

```java
@Repository
public interface UserDao extends JpaRepository<User, Integer>, JpaSpecificationExecutor<User> {
    
    @Query("from User")
    List<User> findAllByQuery();
}
```

这种编写方式与上面的原生方式，最终的运行效果是一样的。

**带参数的查询**：JPQL 可以使用 SQL 中的几乎所有过滤查询，它可以使用两种方式进行传参，分别如下所示。

```java
    @Query("from User where name like %?1%")
    List<User> findAllByNameLike(String name);
    
    @Query("from User where tel = :tel")
    List<User> findAllByTel(@Param("tel") String tel);
```

上面的方式需要保证 JPQL 中定义的参数顺序与接口方法的参数顺序一一对应，而下面的方式使用具体的参数名定义，好处是即便接口方法参数顺序再乱，通过参数名的映射也能正确匹配上。

**传入集合参数**：JPQL 同样可以处理集合类型的参数：

```java
    @Query("from User where id in :ids")
    List<User> findAllByIds(@Param("ids") List<Integer> ids);
```

这种编写方式，生成的 SQL 中会自动把集合中传的值全部拼接到 SQL 上。

**分页查询**：JPQL 本身不负责分页，不过原生的 JPA 操作，或者 SpringDataJPA 中的接口可以通过传入 Pageable 接口实现分页。

```java
    @Query("from User where name like %?1%")
    List<User> findAllByNameLikePage(Pageable pageable, String name);
```

## 5. SpringDataJPA提供的特性

SpringDataJPA 既然是基于 JPA 进一步进行了封装，那它必然提供了一些更好用的特性，下面介绍其中两个最常用的特性。

### 5.1 Dao接口方法查询

其实在上面的 JPQL 查询演示中，我们编写的一些方法根本就不需要写 JPQL 语句，只从方法名上，SpringDataJPA 就可以帮我们完成 SQL 语句的生成了，根本不需要费劲巴倒的写 JPQL 语句。

譬如：

```java
List<User> findAllByNameLike(String name);

```

对应的 SQL 就是：

```sql
select * from tbl_user where name like ?

```

这种带具体语义的 Dao 接口方法，就是 SpringDataJPA 提供的巨方便的特性。同样，想要使用这种高级特性，方法的命名规则需要我们了解，下面的表格中列举了部分 SpringDataJPA 支持的方法命名规则。

[https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#repositories.query-methods](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/#repositories.query-methods)



### 5.2 动态条件构造查询

使用方法命名的查询方式固然可以很方便地完成查询，但在一些条件查询中，查询的规则是基于客户端发起的过滤属性来动态构造，这种情况下上述的查询方式就显得有些捉襟见肘。SpringDataJPA 给我们提供了一个动态构造查询条件的 QBC 查询，也即 `Specification` ，使用 `Specification` 进行动态条件构造查询的方法，在实际的项目开发中经常使用。

下面通过一个简单的示例介绍 QBC 的使用。如下方代码所示，上面的 `example` 是一个模拟客户端请求传入的查询条件模型对象，我们拿到这个对象后，在调用 `UserDao` 的 `findAll` 时传入一个 `Specification` 的匿名内部类对象，并操作其 API 构造查询条件。由于查询条件中只有 `name` 需要查询，所以我们只需要调用 `criteriaBuilder` 的 `like` 方法，指定 `name` 属性需要模糊查询，并拼接模糊匹配的查询值即可。

```java
User example = new User();
example.setName("my");

List<User> users = userDao.findAll((root, query, criteriaBuilder) -> {
    List<Predicate> criterias = new ArrayList<>();
    criterias.add(criteriaBuilder.like(root.get("name").as(String.class), "%" + example.getName() + "%"));
    return criteriaBuilder.and(criterias.toArray(new Predicate[0]));
});
users.forEach(System.out::println);
```

在上面的查询条件构造时，我们手动构造了一个 `List<Predicate> criterias` 集合，这种设计的意图是什么呢？

显然，在面对客户端的请求时，我们并不清楚最终构造的查询条件有几个，为此就需要通过一个查询条件的集合，来存放所有被构造的查询条件。在实际开发中，更多的写法就是逐个检查需要过滤的属性，并在值不为空时拼接特定的查询条件，这种写法与 MyBatis 的动态 SQL 非常相似。

