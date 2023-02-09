---
title: 07整合mybatis.md
toc: true
tags: springboot
categories: 
    - [java]
    - [springboot]
---

前面两章，我们对 SpringBoot 支持的单元测试有了一个比较完整的了解，也简单挖掘了底层支撑的几个关键组件。从本章开始的连续三章，我们对 MyBatis 以及它的一个扩展 —— MyBatisPlus 进行整合。


<!--more-->

尽管 SpringFramework 和 SpringBoot 在简化与数据库的交互方面已经做了很大的努力，但是在实际的项目开发中更多的场景是整合成熟的持久层框架完成与数据库交互。目前市面上比较流行的持久层框架包括 SpringDataJPA （底层默认依赖 Hibernate ）和 MyBatis ，本章先从 MyBatis 开始讲起，SpringDataJPA 相对简单，我们后续再讲解。

## 1. MyBatis简单回顾

首先我们简单回顾一下 MyBatis 框架。MyBatis 是一款优秀的持久层框架，它支持自定义 SQL、存储过程以及高级映射。MyBatis 免除了几乎所有的 JDBC 代码以及设置参数和获取结果集的工作。MyBatis 可以通过简单的 XML 或注解来配置和映射原始类型、接口和 Java POJO（Plain Old Java Objects，普通老式 Java 对象）为数据库中的记录。

整体上讲，MyBatis 的架构可以分为三层，如下图所示。

![](/img/202212/07sprinboot-mybatis.png)

- 接口层：`SqlSession` 是平时与 MyBatis 完成交互的核心接口（包括整合 SpringFramework 和 SpringBoot 后用到的 `SqlSessionTemplate` ）；
- 核心层：`SqlSession` 执行的方法，底层需要经过配置文件的解析、SQL 解析，以及执行 SQL 时的参数映射、SQL 执行、结果集映射，另外还有穿插其中的扩展插件；
- 支持层：核心层的功能实现，它基于底层的各个模块，共同协调完成。

## 2. SpringBoot整合MyBatis

下面来搭建一个 SpringBoot 整合 MyBatis 的基本工程。

### 2.1 搭建工程

由于 MyBatis 整合 SpringBoot 的场景启动器中，已经对 SpringBoot 原生的 starter 做了整合，所以在导入依赖时只需要引入 `mybatis-spring-boot-starter` 和具体的数据库连接驱动即可。

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

> `mybatis-spring-boot-starter` 2.2.0 版本底层依赖 SpringBoot 2.5.0 ，与小册研究的 SpringBoot 主要版本同属一个中版本，整合的可靠性最高。

### 2.2 初始化数据库

接下来我们来搭建数据库。本小册选择使用 MySQL 作为演示数据库，首先我们来创建一个新的数据库，以及一张用于演示的表 `tbl_user` ，对应的 SQL 脚本如下：

```mysql
create database springboot_dao CHARACTER SET utf8mb4;

CREATE TABLE tbl_user(
  id int(11) NOT NULL AUTO_INCREMENT,
  name varchar(20) NOT NULL,
  tel varchar(20) NULL,
  PRIMARY KEY (id)
);
```

> 注意：本小册的内容以讲解框架的使用为主，对于数据库的部分建议小伙伴从简设计，切勿本末倒置。

### 2.3 编写基础代码和配置

为简单回顾 MyBatis 框架的使用，本部分内容我们采用单元测试的方式来测试和执行示例代码。

##### 实体类

对于原生的 MyBatis 而言，我们只需要编写最基础的实体模型类即可。对标 `tbl_user` 表，我们编写一个相应的 `User` 类：

```java
@Data
public class User {
    private Integer id;

    private String name;

    private String tel;
}
```

##### Mapper接口

MyBatis 推荐我们使用 Mapper 接口动态代理的方式进行 Dao 层开发，所以我们只需要编写一个 `User` 模块的接口 `UserMapper` 。为了快速回顾和演示 MyBatis 的使用，这里我们快速编写两个方法即可。

```java
@Mapper
public interface UserMapper {
    
    void save(User user);
    
    List<User> findAll();
}
```

> 小提示：可能部分小伙伴的 IDEA 在注入 `UserMapper` 接口时，会提示 IOC 容器没有该 bean 的错误，这个时候只需要在 `UserMapper` 接口上再标注一个 `@Repository` 注解即可。

##### mapper.xml

对应的，我们还需要编写一个 UserMapper.xml ，以完成与 UserMapper 接口的对应。

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">



<mapper namespace="org.clxmm.mapper.UserMapper">
    <insert id="save" parameterType="User">
        insert into tbl_user (name, tel) values (#{name}, #{tel})
    </insert>


    <select id="findAll" resultType="User">
        select * from tbl_user
    </select>

</mapper>
```

##### Service

Service 层需要注入 Mapper 接口，完成业务逻辑的执行。对应的 `UserService` 编写如下，同样是为了快速回顾和演示，我们在此处只需要分别调用一下 Mapper 接口的两个方法，验证执行无误即可。

```java
@Service
public class UserService {
    
    @Autowired
    private UserMapper userMapper;
    
    @Transactional(rollbackFor = Exception.class)
    public void test() {
        User user = new User();
        user.setName("test mybatis");
        user.setTel("1234567");
        userMapper.save(user);
        
        List<User> userList = userMapper.findAll();
        userList.forEach(System.out::println);
    }
}
```

##### 配置文件&主启动类

最后不要忘记编写 SpringBoot 的配置文件，配置文件中包含数据源的配置，以及一些简单的 MyBatis 的配置：

```properties
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
spring.datasource.url=jdbc:mysql://127.0.0.1:3306/springboot-dao?characterEncoding=utf8
spring.datasource.username=root
spring.datasource.password=root


mybatis.type-aliases-package=org.clxmm.entity
mybatis.mapper-locations=classpath:mapper/*.xml
mybatis.configuration.map-underscore-to-camel-case=true

```

以及 SpringBoot 的主启动类：

```java
package org.clxmm;

import org.mybatis.spring.annotation.MapperScan;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.transaction.annotation.EnableTransactionManagement;

/**
 * @author clxmm
 * @Description
 * @create 2022-12-18 21:06
 */
//@MapperScan
@SpringBootApplication
@EnableTransactionManagement
public class SpringbootMybatisApp {

    public static void main(String[] args) {
        SpringApplication.run(SpringApplication.class, args);
    }
}

```

### 2.4 单元测试

下面我们快速编写一下单元测试的代码，向容器中注入 `UserMapper` 与 `UserService` ，并调用其方法，以检验代码编写是否正确。

```java
@SpringBootTest
public class MyBatisTest {
    
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
        List<User> users = userMapper.findAll();
        System.out.println(users);
    }
}
```

分别执行 `test1` 与 `test2` 测试方法，单元测试均通过。到此为止，我们就已经完成了 SpringBoot 与 MyBatis 框架的整合。

## 3. MyBatis的使用回顾

### 3.1 可供选择的配置

借助 IDE ，在编写 `application.properties` 时可以发现，有关 MyBatis 的配置可以分为两个部分：一部分是 `mybatis.*` ，另一部分是 `mybatis.configuration.*` ，这些配置的核心基本全部与 MyBatis 全局配置文件 `mybatis-config.xml` 相对应。

![](/img/202212/07springboot-mybatis1.png)

### 3.2 注解Mapper接口

Mapper 接口除了可以与 mapper.xml 对应之外，还可以直接在 Mapper 接口中，通过使用 CRUD 注解编写 statement ，以下是几个简单的示例。

```java
@Mapper
@Repository
public interface UserMapper {
    
    void save(User user);
    
    List<User> findAll();
    
    @Select("select * from tbl_user where name like concat('%', #{name}, '%')")
    List<User> findAllByNameLike(@Param("name") String name);
    
    @Delete("delete from tbl_user where id = #{id}")
    int deleteById(String id);
}
```

可以发现，MyBatis 本身提供了 CRUD 的基础注解，可以编写类似于 mapper.xml 中的 SQL 语句一样，从而完成更加快速的 Mapper 编写。除此之外，还可以通过使用 Provider 机制，通过编程式构造 SQL 并执行。

### 3.3 动态SQL

MyBatis 相较于 `spring-jdbc` 中的 `JdbcTemplate` ，DbUtils 中的 `QueryRunner` 等简单的 jdbc 封装，其一大优势就是灵活的动态 SQL 机制，合理利用动态 SQL 的机制，可以编写出非常多样的符合业务场景的查询、写入数据库的 SQL 语句。

简单来看，动态 SQL 可以分为 select 类、update 类，以及通用抽取的 SQL 片段三种类别。其中 select 类的动态 SQL 可以使用的标签最多，可以实现判断、选择、循环、截取等动态 SQL 逻辑，update 类动态 SQL 也可以针对 SQL 语法中的一些场景做比较实用的处理。

### 3.4 缓存机制

MyBatis 为了考虑运行期间的查询效率，它引入了两层级缓存机制，其中一级缓存是 `SqlSession` 级别的，二级缓存是 `SqlSessionFactory` 级别的。

通常情况下，MyBatis 的一级缓存是默认开启并使用的，一级缓存基于 `SqlSession` ，所以我们可以直接创建 `SqlSessionFactory` ，并从中开启一个新的 `SqlSession` ，默认情况下它会自动开启事务，所以一级缓存会自动使用。以下是一个简单的示例：

```java
public class Level1Application {
    
    public static void main(String[] args) throws Exception {
        InputStream xml = Resources.getResourceAsStream("mybatis-config.xml");
        SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(xml);
        SqlSession sqlSession = sqlSessionFactory.openSession();
    
        DepartmentMapper departmentMapper = sqlSession.getMapper(DepartmentMapper.class);
        System.out.println("第一次执行findAll......");
        departmentMapper.findAll();
        System.out.println("第二次执行findAll......");
        departmentMapper.findAll();
    
        sqlSession.close();
    }
}
```

MyBatis 的二级缓存需要手动开启，且二级缓存是 namespace 隔离的，一个 namespace 共享一个二级缓存区域。二级缓存是 `SqlSessionFactory` 级别的，在一个 `SqlSessionFactory` 的范围中创建的 `SqlSession` 均可以共享这些缓存。以下是一个证明二级缓存的示例：

```java
public class Level2Application {
    
    public static void main(String[] args) throws Exception {
        InputStream xml = Resources.getResourceAsStream("mybatis-config.xml");
        SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(xml);
        SqlSession sqlSession = sqlSessionFactory.openSession();
    
        DepartmentMapper departmentMapper = sqlSession.getMapper(DepartmentMapper.class);
        System.out.println("第一次执行findAll......");
        departmentMapper.findAll();
        System.out.println("第二次执行findAll......");
        departmentMapper.findAll();
        
        sqlSession.close();
    
        // 开启一个新的SqlSession，测试二级缓存
        SqlSession sqlSession2 = sqlSessionFactory.openSession();
        DepartmentMapper departmentMapper2 = sqlSession2.getMapper(DepartmentMapper.class);
        System.out.println("sqlSession2执行findAll......");
        departmentMapper2.findAll();
        
        sqlSession2.close();
    }
}
```

### 3.5 插件机制

MyBatis 中的最后一个重要机制是插件机制，也就是所谓的拦截器。MyBatis 的插件本身是一些能拦截某些 MyBatis 核心组件方法，增强功能的拦截器，MyBatis 允许我们在 SQL 语句执行过程中的某些点进行拦截增强，共有四种可供增强的切入点：

- `Executor` ( update, query, flushStatements, commit, rollback, getTransaction, close, isClosed )
- `ParameterHandler` ( getParameterObject, setParameters )
- `ResultSetHandler` ( handleResultSets, handleOutputParameters )
- `StatementHandler` ( prepare, parameterize, batch, update, query )

MyBatis 的插件适用范围还是比较广的，可以应用于分页、数据权限过滤、性能分析等场景。

【本章内容回顾了 Dao 层框架 MyBatis 的使用，在实际开发中我们可能更倾向于使用一个功能更强大，能帮助我们做更多事情的增强框架，下一章我们来研究 MyBatis 的增强“伙伴” MyBatisPlus 】

