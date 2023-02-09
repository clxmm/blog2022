---
title: 08整合mybatis-plus
toc: true
tags: springboot
categories: 
    - [java]
    - [springboot]
---

上一章中，我们简单回顾了 MyBatis 框架，作为一个持久层框架，我们希望使用的框架最好具备一些基本的通用功能，而不是每次编写新的模块时都需要频繁生成代码（甚至手动编写），于是 MyBatisPlus 应运而生。MyBatisPlus 本质是对 MyBatis 的补充，而不是作为 MyBatis 的代替品。

## 1. MyBatisPlus概述

从 MyBatisPlus 的官网可以看到，MyBatisPlus 的作者给 MyBatisPlus 的定位是：只做增强不做改变、效率至上功能丰富。从这些关键词上可以提取的信息：[https://baomidou.com/](https://baomidou.com/)

- 无侵入
- 性能影响小
- 开发测试中的实用特性

此外，MyBatisPlus 本身还内置了不少现成的插件，以及对多种数据库的支持（包含多种国产数据库），这些都是在实际的项目开发中提供的较大效率提升特性。

<!--more-->

MyBatisPlus 的官方文档中提供了一张架构图，由该图可以非常明显地看出，MyBatisPlus 实际是利用了自身扩展的一些特性（新注解、新扩展、代码生成器等），并注入增强 MyBatis 的原有功能，使得我们开发者在使用 Dao 层时可以体会到更简单易用的 API 体验。

![](/img/202212/08springboot-mybatis.png)

## 2. SpringBoot整合MyBatisPlus

### 2.1 搭建工程

MyBatisPlus 在整合到 SpringBoot 时同样提供了 starter 场景启动器，我们只需要导入场景启动器，还有 MySQL 的连接驱动即可。同时为了方便后续的单元测试，一并导入 `spring-boot-starter-test` 依赖。

```xml
<dependencies>
    <dependency>
        <groupId>com.baomidou</groupId>
        <artifactId>mybatis-plus-boot-starter</artifactId>
        <version>3.5.0</version>
    </dependency>
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>

```

> MyBatisPlus 3.5.0 版本底层依赖 SpringBoot 2.5.3 ，与小册研究的 SpringBoot 主要版本同属一个中版本，整合可靠性也是很高的。

### 2.2 编写基础代码和配置

MyBatisPlus 拥有一些内置的基础能力，所以在编写基础代码时会有所不同。

##### 实体类

MyBatisPlus 拥有非常优秀的单表 CRUD 基础能力，而这个能力需要在实体类上做一些改动。通过标注 `@TableName` 注解，相当于告诉 MyBatisPlus 当前这个 `User` 类要映射到 `tbl_user` 表（默认表名策略是驼峰转下划线）；通过给 `id` 属性标注 `@TableId` 注解，并声明 ID 类型为 auto ，相当于适配 MySQL 中的自增主键。其他属性与数据库中映射均一致，就不再需要添加新注解了。

```java
@Data
@TableName("tbl_user")
public class User {

    @TableId(type = IdType.AUTO)
    private Integer id;

    private String name;

    private String tel;
}
```

##### Mapper接口&mapper.xml

MyBatisPlus 的单表 CRUD 能力来自一个内置的基础接口 `BaseMapper` ，通过继承 `BaseMapper` 并注明实体类的泛型类型，即可拥有单表的 CRUD 能力。如果没有多余的 SQL 编写需要，则 mapper.xml 甚至都可以不用编写。

```java
@Mapper
@Repository
public interface UserMapper extends BaseMapper<User> {

}
```

##### Service

MyBatisPlus 为了给我们开发者在三层架构的开发模型中予以更多基础支持，它提供了一个 `IService` 接口与 `ServiceImpl` 基础实现类，在这个基础的 `ServiceImpl` 中已经定义了很多 CRUD 方法，足够日常开发使用。我们只需要编写自己的业务方法即可。

```java
@Service
public class UserService extends ServiceImpl<UserMapper, User> {
    
    @Autowired
    private UserMapper userMapper;
    
    @Transactional(rollbackFor = Exception.class)
    public void test() {
        User user = new User();
        user.setName("test mybatisplus");
        user.setTel("7654321");
        userMapper.insert(user);
    
//        int i = 1 / 0;
        
        List<User> userList = userMapper.selectList(null);
        userList.forEach(System.out::println);
    }
}
```

##### 配置文件&主启动类

使用 MyBatisPlus 与使用 MyBatis 的配置内容几乎没有区别，仅仅是将 mybatis 开头的配置替换为 mybatis-plus 即可：

```properties
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
spring.datasource.url=jdbc:mysql://127.0.0.1:3306/springboot-dao?characterEncoding=utf8
spring.datasource.username=root
spring.datasource.password=root


mybatis-plus.type-aliases-package=org.clxmm.entity
mybatis-plus.mapper-locations=classpath:mapper/*.xml
mybatis-plus.configuration.map-underscore-to-camel-case=true

```

SpringBoot 的主启动类没有任何区别。

```java
@SpringBootApplication
@EnableTransactionManagement
public class MybatisPlusApp {

    public static void main(String[] args) {
        SpringApplication.run(MybatisPlusApp.class);
    }

}
```

### 2.3 单元测试

MyBatisPlus 的单元测试代码编写，与 MyBatis 也没有什么区别，我们分别注入 `UserMapper` 与 `UserService` ，看一下它们是否都正常可用。

```java
@SpringBootTest
public class MyBatisPlusTest {
    
    @Autowired
    private UserMapper userMapper;
    
    @Autowired
    private UserService userService;
    
    @Test
    public void test1() {
        userService.test();
    }
    
    @Test
    public void test2() {
        userMapper.selectList(null).forEach(System.out::println);
        System.out.println("==================");
        userService.list().forEach(System.out::println);
        System.out.println(userService.count());
    }
}
```

分别运行两个方法，可以发现方法都可以正常使用，其中 test2 中调用 mapper 与 service 的查询结果都是正确的，且 service 中的方法名称更容易理解。

```ini
User{id=1, name='test mybatis', tel='7654321'}
User{id=2, name='test mybatisplus', tel='7654321'}
==================
User{id=1, name='test mybatis', tel='7654321'}
User{id=2, name='test mybatisplus', tel='7654321'}
2
```

到此为止，我们就已经完成了 SpringBoot 与 MyBatisPlus 框架的整合。

## 3. MyBatisPlus的核心特性使用

MyBatisPlus 作为 MyBatis 的增强框架，官方给出一个比较有趣的说法是：MyBatis 与 MyBatisPlus 就好像魂斗罗里的比尔和兰斯，两者搭配效率翻倍。强大的支撑构筑了一些实用的特性，本章会列举一些日常使用频率比较高的特性讲解，详细的功能特性使用可以参照 [MyBatisPlus 的官方文档](https://baomidou.com/)。

### 3.1 CRUD基础接口

MyBatisPlus 提供的重要基础能力，就是替我们开发者实现了基本的单表 CRUD 操作，我们在编写具体的业务模块时，单表的 CRUD 可以完全不需要编写了，仅需要继承 `BaseMapper` 接口，该 Mapper 接口就可以自动拥有单表 CRUD 的能力。在上面的测试代码中，`UserMapper` 继承了 `BaseMapper` 后，它已经拥有了包含 insert 、update 、delete 、selectOne 、selectList 、count 等操作单表数据的能力。

##### insert型接口

```java
// 插入一条记录
int insert(T entity);
```

##### delete型接口

```java
// 根据 entity 条件，删除记录
int delete(@Param(Constants.WRAPPER) Wrapper<T> wrapper);
// 删除（根据ID 批量删除）
int deleteBatchIds(@Param(Constants.COLLECTION) Collection<? extends Serializable> idList);
// 根据 ID 删除
int deleteById(Serializable id);
// 根据 columnMap 条件，删除记录
int deleteByMap(@Param(Constants.COLUMN_MAP) Map<String, Object> columnMap);
```

##### update型接口

```java
// 根据 whereWrapper 条件，更新记录
int update(@Param(Constants.ENTITY) T updateEntity, @Param(Constants.WRAPPER) Wrapper<T> whereWrapper);
// 根据 ID 修改
int updateById(@Param(Constants.ENTITY) T entity);
```

##### select型接口

```java
// 根据 ID 查询
T selectById(Serializable id);
// 根据 entity 条件，查询一条记录
T selectOne(@Param(Constants.WRAPPER) Wrapper<T> queryWrapper);

// 查询（根据ID 批量查询）
List<T> selectBatchIds(@Param(Constants.COLLECTION) Collection<? extends Serializable> idList);
// 根据 entity 条件，查询全部记录
List<T> selectList(@Param(Constants.WRAPPER) Wrapper<T> queryWrapper);
// 查询（根据 columnMap 条件）
List<T> selectByMap(@Param(Constants.COLUMN_MAP) Map<String, Object> columnMap);
// 根据 Wrapper 条件，查询全部记录
List<Map<String, Object>> selectMaps(@Param(Constants.WRAPPER) Wrapper<T> queryWrapper);
// 根据 Wrapper 条件，查询全部记录。注意： 只返回第一个字段的值
List<Object> selectObjs(@Param(Constants.WRAPPER) Wrapper<T> queryWrapper);

// 根据 entity 条件，查询全部记录（并翻页）
IPage<T> selectPage(IPage<T> page, @Param(Constants.WRAPPER) Wrapper<T> queryWrapper);
// 根据 Wrapper 条件，查询全部记录（并翻页）
IPage<Map<String, Object>> selectMapsPage(IPage<T> page, @Param(Constants.WRAPPER) Wrapper<T> queryWrapper);
// 根据 Wrapper 条件，查询总记录数
Integer selectCount(@Param(Constants.WRAPPER) Wrapper<T> queryWrapper);
```

### 3.2 Wrapper机制

Wrapper 是 MyBatisPlus 编程式查询、修改数据的重要特性，这种特性类似于 Hibernate 中的 Criteria 机制（也就是 QBC 查询）。MyBatis 提供的 Wrapper 机制拥有对单表查询的灵活条件构造、投影查询、聚合查询等能力。下面通过几个简单示例来了解 Wrapper 的使用。

##### 简单使用

我们先通过一个简单的模糊查询来演示 Wrapper 的基本使用。

```java
@Test
public void test3() throws Exception {
    List<User> users = userMapper.selectList(Wrappers.<User>query().like("tel", "123"));
    users.forEach(System.out::println);
}
```

由上述测试代码可见，我们通过构造一个 `User` 类的条件查询器，并模糊查询 tel 属性包含 “123” 的数据。执行单元测试，控制台可以成功打印一条数据，证明条件查询已经起了效果。

```ini
User{id=1, name='test mybatis', tel='1234567'}

```

##### lambda式使用

上面的使用有一个问题：当 `User` 类的属性发生变化时，相应的 Wrapper 中条件构造也需要修改，但是 IDE 并不会感知到这些属性被使用。为了更容易维护可能变化的实体模型类属性，MyBatisPlus 提供了 LambdaWrapper ，使用这种类型的 Wrapper 将属性的字符串变量改为 Lambda 表达式，以此实现代码的高可维护性。

```java
@Test
public void test4() throws Exception {
    List<User> users = userMapper.selectList(Wrappers.lambdaQuery(User.class).eq(User::getTel, "7654321"));
    users.forEach(System.out::println);
}
```

上述测试代码中，将 `tel` 属性的引用改为 Lambda 表达式，即便以后 `tel` 属性改名叫 `phone` ，使用 IDE 修改时这些 Lambda 表达式也会跟着修改。

```ini
User{id=2, name='test mybatisplus', tel='7654321'}

```

##### 投影查询

如果一个表的列特别多，而我们只需要查其中几列数据时，投影查询就显得非常重要了，通过指定需要查询的列，来达到节省数据库流量带宽的目的。

```java
@Test
public void test5() throws Exception {
    List<User> users = userMapper.selectList(Wrappers.lambdaQuery(User.class).select(User::getName));
    users.forEach(System.out::println);
}
```

如上述测试代码所示，在构造 `Wrapper` 时指定了只查询 `name` 列，这样查询出来的 `User` 对象中就只有 `name` 属性。

```ini
User{id=null, name='test mybatis', tel='null'}
User{id=null, name='test mybatisplus', tel='null'}
```

##### 聚合查询

对于单表查询来讲，聚合查询也是一个常见的查询场景。虽然 MyBatisPlus 没有对几种聚合函数提供 API 的定义，不过我们可以传入 SQL 片段来曲线实现聚合查询。小册只是举一个简单的例子，我们来查询最大 id 的 `User` 信息：

```java
@Test
public void test6() throws Exception {
    User user = userMapper.selectOne(Wrappers.<User>query().select("max(id) as id"));
    System.out.println(user);
}
```

需要注意的是，在使用聚合函数查询时无法使用 `LambdaQueryWrapper` 来实现，只能使用普通的 `QueryWrapper` 。

##### 更灵活的条件构造

在实际的项目开发中，可能存在一些条件构造的场景：只有满足 xxx 条件后，才构造 Wrapper 中的查询条件。下面看一个简单示例：只有查询的 QO 中 name 属性值不为空时，才执行条件构造。

```java
@Test
public void test7() throws Exception {
    User qo = new User();

    List<User> users = userMapper.selectList(
            Wrappers.lambdaQuery(User.class).like(StringUtils.hasText(qo.getName()), User::getName, qo.getName()));
    System.out.println(users.size());
    
    qo.setName("mybatisplus");
    users = userMapper.selectList(
            Wrappers.lambdaQuery(User.class).like(StringUtils.hasText(qo.getName()), User::getName, qo.getName()));
    System.out.println(users.size());
}
```

观察上述测试代码，第一次查询时 qo 的 `name` 属性为空，所以条件不成立，`Wrapper` 不会构造 like 条件；第二次设置 `name` 属性值后，`StringUtils.hasText` 方法判断成立，like 条件构造。所以两次打印的 size 值分别为 2 和 1 。

### 3.3 主键策略与ID生成器

MyBatisPlus 考虑到我们在项目开发中可能会用到的几种主键类型，它给予了一些基础实现和配置：

- AUTO ：数据库主键自增
- ASSIGN_ID ：雪花算法 ID
- ASSIGN_UUID ：不带短横线的 uuid
- INPUT ：程序手动设置的 id （或配合序列填充，Oracle 、SQLServer 等使用）
- NONE ：逻辑主键，数据库表中没有定义主键

默认情况下，MyBatisPlus 使用的主键策略是使用了雪花算法的 ASSIGN_ID 策略。

### 3.4 逻辑删除

下面简单了解 MyBatisPlus 中的两个简单实用的特性。

逻辑删除是代替 delete 物理删除的一种更适合项目开发的数据删除机制，它通过设置一个特殊的标志位，将需要删除的数据设置为“不可见”，并在每次查询数据时只查询标志位数据值为“可见”的数据，这样的设计即是逻辑删除。MyBatisPlus 使用逻辑删除非常简单，只需要两步即可。

**1）application.properties 中配置**

```properties
mybatis-plus.global-config.db-config.logic-delete-field=deleted
mybatis-plus.global-config.db-config.logic-delete-value=1
mybatis-plus.global-config.db-config.logic-not-delete-value=0
```

**2）在实体类上标记 @TableLogic 注解**

```java
@TableName("tbl_user")
public class User {
    
    @TableId(type = IdType.AUTO)
    private Integer id;
    
    private String name;
    
    private String tel;
    
    @TableLogic
    private Integer deleted;
```

如此配置后，在创建数据库表时也随之带上 `deleted` 列，两者就可以搭配完成逻辑删除了。

### 3.5 乐观锁插件

乐观锁是高并发下控制的手段，它假设多用户并发的事务在处理时不会彼此互相影响，各事务能够在不产生锁的情况下处理各自影响的那部分数据。换句话说，乐观锁希望一条即将被更新的数据，没有被其他人操作过。

乐观锁的实现方式如下：

- 给数据添加 version 属性
- 当查询数据时，把 version 数据一并带出
- 更新数据时，将查询的 version 数据值一并传入
- 执行 update / delete 语句时，额外在 where 条件中添加 version = ? 语句
- 如果 version 数据与数据库中的不一致，则更新 / 删除失败

MyBatisPlus 中实现的乐观锁机制是通过插件实现。使用乐观锁需要以下两个步骤：

**1）注册乐观锁插件**

```java
@Configuration(proxyBeanMethods = false)
public class MyBatisPlusConfiguration {
    
    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        // 添加乐观锁插件
        interceptor.addInnerInterceptor(new OptimisticLockerInnerInterceptor());
        return interceptor;
    }
}
```

**2）实体类添加 `@Version` 注解**

```java
@TableName("tbl_user")
public class User {
    
    @TableId(type = IdType.AUTO)
    private Integer id;
    
    @Version
    private Integer version;
    // ......
}
```

以此法编写完毕后，在 `tbl_user` 表的单表数据操作时，乐观锁就会介入处理。

> 需要注意的是，MyBatisPlus 支持的乐观锁，可以对以下的数据类型予以支持：
>
> - int long
> - Integer Long
> - Date Timestamp LocalDateTime

### 3.6 分页插件

分页查询是项目开发中非常常见的业务场景，对于 MyBatis 的分页插件而言可能之前比较常见的是 PageHelper ，MyBatisPlus 已经考虑到了分页查询的场景，它提供了一个专门用于分页的插件，通过简单的配置就可以使用分页的特性。

**1）添加分页插件**

```java
@Bean
public MybatisPlusInterceptor mybatisPlusInterceptor() {
    MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
    // 添加乐观锁插件
    interceptor.addInnerInterceptor(new OptimisticLockerInnerInterceptor());
    // 添加分页插件
    interceptor.addInnerInterceptor(new PaginationInnerInterceptor(new MySqlDialect()));
    return interceptor;
}
```

**2）使用分页模型**

在进行分页查询时，需要传入 `IPage` 对象，并返回 `IPage` 模型或 `List` 集合。以下是几个示例。

```java
    /**
     * 使用IPage作为入参和返回值
     * @param query
     * @return
     */
    IPage<User> page(IPage<User> query);
    
    /**
     * 使用集合作为返回值
     * @param query
     * @return
     */
    List<User> pageList(IPage<User> query);
    
    /**
     * 使用IPage和其他参数共同作为入参
     * @param page
     * @param params
     * @return
     */
    IPage<User> pageParams(@Param("page") IPage<User> page, @Param("params") Map<String, Object> params);
```

### 3.7 代码生成器

日常开发中不乏会编写好多好多几乎一样的代码，包括但不限于单表的 Mapper、Service 、Controller 等代码，以及其中的增删改查代码等等。MyBatisPlus 为了方便我们的开发，它提供了一个代码生成器，利用这个代码生成器可以生成包括实体模型类、Mapper 接口、mapper.xml 、Service 、Controller 等代码。

使用代码生成器时，需要在引入 MyBatisPlus 核心包的基础上，再依赖一个 `mybatis-plus-generator` 以及模板引擎 FreeMarker 即可：

```xml
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-generator</artifactId>
    <version>3.5.2</version>
</dependency>

<dependency>
    <groupId>org.freemarker</groupId>
    <artifactId>freemarker</artifactId>
    <version>2.3.31</version>
</dependency>
```

之后只需要编写一个简单的测试代码，执行如下的代码即可：

```java
public class CodeGenerator {
    
    public static void main(String[] args) {
        FastAutoGenerator.create("jdbc:mysql://localhost:3306/sringboot-dao?characterEncoding=utf8", "root", "123456")
            .globalConfig(builder -> {
                builder.author("LinkedBear") // 设置代码的author
                    .outputDir("D://codegenerator"); // 指定输出目录
            })
            .packageConfig(builder -> {
                builder.parent("com.linkedbear.boot.mybatisplus") // 设置父包名
                    .moduleName("user") // 设置父包模块名
                    .pathInfo(Collections.singletonMap(OutputFile.xml, "D://codegenerator")); // 设置mapperXml生成路径
            })
            .strategyConfig(builder -> {
                builder.addInclude("tbl_user") // 设置需要生成的表名
                    .addTablePrefix("tbl_"); // 设置过滤表前缀
            })
            .templateEngine(new FreemarkerTemplateEngine()) // 使用Freemarker引擎模板，默认的是Velocity引擎模板
            .execute();
    }
}
```

之后就可以执行 main 方法来生成代码了。执行 main 方法之后可以生成如下的一些代码。

**Controller ：**

```java
package com.linkedbear.boot.mybatisplus.user.controller;

import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.stereotype.Controller;

/**
 * <p>
 *  前端控制器
 * </p>
 *
 * @author LinkedBear
 * @since 2000-01-01
 */
@Controller
@RequestMapping("/user/user")
public class UserController {

}
```

**ServiceImpl ：**

```java
package com.linkedbear.boot.mybatisplus.user.service.impl;

import com.linkedbear.boot.mybatisplus.user.entity.User;
import com.linkedbear.boot.mybatisplus.user.mapper.UserMapper;
import com.linkedbear.boot.mybatisplus.user.service.IUserService;
import com.baomidou.mybatisplus.extension.service.impl.ServiceImpl;
import org.springframework.stereotype.Service;

/**
 * <p>
 *  服务实现类
 * </p>
 *
 * @author LinkedBear
 * @since 2000-01-01
 */
@Service
public class UserServiceImpl extends ServiceImpl<UserMapper, User> implements IUserService {

}
```

怎么说呢，能用吗？能用！够用吗？好像不是那么够用！其实我们可以自行编写代码模板（配合 FreeMarker 的语法），之后在生成代码时指定模板即可。详细的使用方式可以参考官方文档，小册不再啰嗦。

【有关 MyBatisPlus 的介绍本章就介绍到这里。MyBatis 整合 SpringBoot 后都发生了什么？都装配了哪些核心的组件？下一章的重点内容我们就来探讨 MyBatis 整合 SpringBoot 后的自动装配与核心组件】

