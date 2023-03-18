---
title: 34整合定时任务-分布式定时任务xxl-job
toc: true
tags: springboot
categories: 
    - [java]
    - [springboot]
---

Quartz 虽说作为单体应用的定时任务调度是很好，但现在很多项目都是采用微服务架构来制作，这种情况下让各个微服务来自己管理定时任务，情况就跟之前不一样了：如果某个微服务部署了 3 个实例，而其中某个定时任务只能在某个时间内执行一次，那就得控制让这 3 个实例中只有一个触发，这个就麻烦了；倘若某个实例在触发定时任务的过程中失败了，需要再让另一个实例重试，这个实现起来也是很麻烦的。

<!--more-->

为此，我们需要一个能适应分布式 / 微服务架构的定时任务调度方案，那就是本章我们要接触的 xxl-job 。

## 1. xxl-job的来历

首先我们先得知道 xxl-job 是个什么东西。

既然 xxl-job 的中文名称是“分布式任务调度平台”，它首先强调的是分布式，那它擅长做的就是接入分布式架构的系统中，去完成定时任务调度的工作。它的作者就是中间件名称的前三个字母 xxl ，许雪里，这位大佬有好几个针对分布式应用开发场景的技术解决方案，如果各位感兴趣的话可以跳转到他的主页 [www.xuxueli.com/page/projec…](https://www.xuxueli.com/page/projects.html) 查看。

  说回 xxl-job ，其实在分布式任务调度的解决方案中还有一个类似的东西叫 elastic-job ，为什么本小册选择讲解 xxl-job 而不是 elastic-job 呢？最大的原因是 xxl-job 相对来讲学习起来更简单，入门门槛低，而且 xxl-job 本身很轻量级，也容易扩展和接入，并且还提供了很多的特性供我们使用。

除此之外，xxl-job 还弥补了 Quartz 的一些不足，诸如定时任务的调用方式（xxl-job 可以使用 Web 端可视化调用），任务调度与业务系统分离（xxl-job 可以使用单独的库维护任务定义信息，而 Quartz 需要耦合到业务系统的数据库中）等等。由此可见，xxl-job 将任务执行和任务调度相分离，这种设计可以充分发挥每个部分的性能，整体的设计也更优。

xxl-job 的源码托管在 [GitHub](https://github.com/xuxueli/xxl-job) 和 [Gitee](http://gitee.com/xuxueli0323/xxl-job) ，我们可以从这两个地方找到，下面我们马上就要用到。

## 2. xxl-job的使用

简单知道 xxl-job 之后，下面我们就可以来实际使用一下。

### 2.1 获取源码

xxl-job 的调度器是需要我们自己从 GitHub 或 Gitee 上下载的，本小册我们使用的是 xxl-job-2.3.1 ，对应的源码可以直接点击这个链接下载：[codeload.github.com/xuxueli/xxl…](https://codeload.github.com/xuxueli/xxl-job/zip/refs/tags/2.3.1) 。

下载的源码是一个 zip 包，我们解压后直接用 IDE 打开即可。

在这个工程中，`xxl-job-admin` 就是调度中心，负责我们在 Web 端管理调度的任务；`xxl-job-core` 是我们在工程 / 微服务中底层的公共依赖，与调度中心的通信就得依赖这个家伙；`xxl-job-executor-samples` 中包含了一些示例工程供我们参考。

### 2.2 xxl-job数据库初始化

由于 xxl-job 在设计的时候就用到了数据库，所以我们在部署使用 xxl-job 之前，需要先把对应的数据库初始化好。

从 GitHub 或 Gitee 中找到数据库初始化脚本 [github.com/xuxueli/xxl…](https://github.com/xuxueli/xxl-job/blob/master/doc/db/tables_xxl_job.sql) ，并在我们本地的数据库中执行。由于这个 SQL 脚本中已经有 `create database` 的语句，所以不需要我们再手动创建新的数据库，我们直接把这段 SQL 脚本拿来执行一下即可。

### 2.3 配置调度中心

数据库有了，下面我们需要把调度中心连接到 MySQL 上，找到源码的 `xxl-job-admin` 工程，来到其中的 SpringBoot 全局配置文件 `application.properties` 中，我们需要改几个地方：`server.port` 调整端口，`spring.datasource.url` 调整 MySQL 的地址和库的名称，`spring.datasource.password` 调整 root 的密码。

```properties
### web
server.port=18899
server.servlet.context-path=/xxl-job-admin

### xxl-job, datasource
spring.datasource.url=jdbc:mysql://127.0.0.1:3306/xxl_job?useUnicode=true&characterEncoding=UTF-8&autoReconnect=true&serverTimezone=Asia/Shanghai
spring.datasource.username=root
spring.datasource.password=123456
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

### 注意在靠下位置还有一个access_token的设置
xxl.job.accessToken=default_token

```

### 2.4 启动任务调度中心

上面的内容配置完毕后，下面我们就可以启动 `xxl-job-admin` 的应用了，我们找到 `src/main/java` 下的 `com.xxl.job.admin.XxlJobAdminApplication` ，直接运行它的 `main` 方法即可。工程启动后，我们可以访问 [http://localhost:18899/xxl-job-admin/](http://localhost:18899/xxl-job-admin/) ，浏览器会跳转到 `/toLogin` 表单登录页，也就是下面截图的这个样子：

输入默认的用户名密码 `admin / 123456` 之后，就可以来到管理主界面了。

至此，一个任务调度的管理端就完成了，下面该我们编码了。

### 2.5 编码业务应用

回到我们自己的 SpringBoot 项目中，创建一个全新的模块 `spring-boot-integration-09-xxljob` ，并导入 `spring-boot-starter-web` 和 xxl-job 的核心依赖：

```xml
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>com.xuxueli</groupId>
        <artifactId>xxl-job-core</artifactId>
        <version>2.3.1</version>
    </dependency>
```

之后我们要编写几个配置项，主要是告诉我们的业务模块，xxl-job 的任务调度器在哪里，以及我们服务的名称是什么。此外，因为 `xxl-job-admin` 调度中心的配置文件中有声明 `xxl.job.accessToken` ，所以这里我们也需要配置好：

```properties
xxl.job.admin.addresses=http://127.0.0.1:18899/xxl-job-admin
xxl.job.executor.appname=demo-xxl-job-executor
xxl.job.accessToken=default_token
```

与我们之前接触的 SpringBoot 场景整合不同，因为上面我们导入的依赖不是一个 starter ，所以所有的配置还是要跟整合普通 SpringFramework 应用那样，一个一个的自己写（没有导入 starter 是 xxl-job 官方就没有提供。。。），所幸只需要注册一个 `XxlJobSpringExecutor` 的 Bean 即可，如下面代码所示，小伙伴没有必要记住如何写，直接复制粘贴过去就好。

```java
@Configuration(proxyBeanMethods = false)
public class XxlJobConfiguration {
    
    @Value("${xxl.job.admin.addresses}")
    private String adminAddresses;
    @Value("${xxl.job.accessToken:}")
    private String accessToken;
    @Value("${xxl.job.executor.appname}")
    private String appname;
    @Value("${xxl.job.executor.address:}")
    private String address;
    @Value("${xxl.job.executor.ip:}")
    private String ip;
    @Value("${xxl.job.executor.port:9999}")
    private int port;
    @Value("${xxl.job.executor.logpath:}")
    private String logPath;
    @Value("${xxl.job.executor.logretentiondays:30}")
    private int logRetentionDays;
    
    @Bean
    public XxlJobSpringExecutor xxlJobExecutor() {
        XxlJobSpringExecutor xxlJobSpringExecutor = new XxlJobSpringExecutor();
        xxlJobSpringExecutor.setAdminAddresses(adminAddresses);
        xxlJobSpringExecutor.setAppname(appname);
        xxlJobSpringExecutor.setAddress(address);
        xxlJobSpringExecutor.setIp(ip);
        xxlJobSpringExecutor.setPort(port);
        xxlJobSpringExecutor.setAccessToken(accessToken);
        xxlJobSpringExecutor.setLogPath(logPath);
        xxlJobSpringExecutor.setLogRetentionDays(logRetentionDays);
        return xxlJobSpringExecutor;
    }
}
```

接下来我们就可以来编写被定时调度的方法了，我们可以来编写一个非常简单的 Service 类，并在其中声明一个 test 方法。使用 xxl-job 的调度方法，只需要在方法上标注一个 `@XxlJob` 注解，并给这个方法起一个名字即可（注意这个名字全局要做到唯一，就跟我们创建的那些 Bean 一样也是名称唯一）。

```java
@Service
@Slf4j
public class DemoService {

    @XxlJob("demoTest")
    public void test() {
        log.info("触发定时任务 。。。");
    }
}
```

这样写完之后，业务代码就做好了。

### 2.6 配置调度执行器

接下来我们要回到 `xxl-job-admin` 任务调度中心上，刚才我们配置好了 `xxl.job.executor.appname` ，但是 `xxl-job-admin` 不知道，所以我们要告诉 `xxl-job-admin` 我们刚声明了一个新的调度服务，来到“执行器管理”页面，手动添加一个新的执行器即可，注意这里的注册方式选择“自动注册”即可，无需手动录入（xxl-job 内部有一个简单的服务注册与发现机制）。

配置完成后，我们启动调度服务，稍等片刻后刷新页面，就可以发现 `demo-xxl-job-executor` 有一个实例上线了。

![](./img/202303/34xxx-job1.png)

### 2.7 配置调度任务

有了任务调度的执行器，下面就可以创建定时调度任务了。来到“任务管理”页面，我们来创建一个新的任务，执行器选择我们刚创建的 `demo-xxl-job` ，任务描述和负责人简单写下就行，重点是中间的那两部分：

- 调度配置：选择 cron 模式，并声明 cron 表达式，此处我们选择 5 秒触发一次；
- 任务配置：选择 BEAN 模式，并指定我们上面写好的那个测试方法上标注的 `@XxlJob` 的 value ，也就是 `demoTest`

![](./img/202303/34xxx-job2.png)

填写好之后就可以保存了，这条数据会真实的保存到 `xxl-job-admin` 的数据库中。

### 2.8 启动定时调度

下面我们可以来启动一下这个任务，点击“操作”右侧的倒三角符号（此处阿熊吐槽一下，竟然不是点“操作”按钮是我很难受的。。。），在展开的菜单中点击“启动”，即可启动这个定时任务。

启动后，在控制台上能看到 “触发定时任务 。。。” 的定时打印，在 `xxl-job-admin` 的“调度日志”页面中也能看到每次定时调度的成功结果：

到这里为止，xxl-job 的基本使用就完成了。

## 3. 封装starter

有一说一，如果只是这样导入 xxl-job 后再写这些配置，那整合 SpringBoot 实在是太 low 了，怎么说我们也是得让约定大于配置的理念发挥出来吧。下面我们就自己简单封装下 xxl-job 的启动器吧。

### 3.1 创建maven工程

注意啊，由于是封装 starter ，所以我们不能再拿着 `spring-boot-integration-parent` 当 parent 了，而是拿 SpringBoot 原生的 `spring-boot-starter-parent` 作为父工程依赖。

导入的依赖中，我们只需要引入基本的 `spring-boot-starter` 以及 `xxl-job-core` 核心依赖即可。

```xml-dtd
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.5.14</version>
    </parent>

    <groupId>com.linkedbear.boot</groupId>
    <artifactId>xxl-job-boot-starter</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <java.version>1.8</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>

        <dependency>
            <groupId>com.xuxueli</groupId>
            <artifactId>xxl-job-core</artifactId>
            <version>2.3.1</version>
        </dependency>
    </dependencies>
```

### 3.2 编写properties

能与 SpringBoot 全局配置文件相呼应的是 properties 类，我们来简单编写一个能跟 application.properties 一一映射的模型类：

```java
@ConfigurationProperties(prefix = "xxl.job")
public class XxlJobProperties {
    
    private String adminAddresses;
    
    private String accessToken = "default_token";
    
    private String appname;
    
    private String address;
    
    private String ip;
    
    private int port = 9999;
    
    private String logPath;
    
    private int logRetentionDays = 30;
    
    // getter setter ......
}
```

注意我们还需要为其中的几个 `int` 值设置默认值，否则它们会被初始化为 0 （设置为 `Integer` 虽然可以避免问题，但最终在初始化 `XxlJobSpringExecutor` 时会出现一些不必要的麻烦）。

### 3.3 编写自动配置类

虽说要创建的 Bean 就一个，但也得由自动配置类来负责承载。我们来编写对应的自动配置类。由于这个类中没有 Bean 与 Bean 之间创建的方法调用，所以我们在标注 `@Configuration` 注解时，最好标注 `proxyBeanMethods = false` 。

另外为了激活 `XxlJobProperties` 的使用，需要使用 `@EnableConfigurationProperties` 启用和映射匹配。

```java
@Configuration(proxyBeanMethods = false)
@EnableConfigurationProperties(XxlJobProperties.class)
public class XxlJobAutoConfiguration {
    
    @Autowired
    private XxlJobProperties properties;
    
    @Bean
    @ConditionalOnMissingBean
    public XxlJobSpringExecutor xxlJobExecutor() {
        XxlJobSpringExecutor xxlJobSpringExecutor = new XxlJobSpringExecutor();
        xxlJobSpringExecutor.setAdminAddresses(properties.getAdminAddresses());
        xxlJobSpringExecutor.setAppname(properties.getAppname());
        xxlJobSpringExecutor.setAddress(properties.getAddress());
        xxlJobSpringExecutor.setIp(properties.getIp());
        xxlJobSpringExecutor.setPort(properties.getPort());
        xxlJobSpringExecutor.setAccessToken(properties.getAccessToken());
        xxlJobSpringExecutor.setLogPath(properties.getLogPath());
        xxlJobSpringExecutor.setLogRetentionDays(properties.getLogRetentionDays());
        return xxlJobSpringExecutor;
    }
}
```

注意这里小册在编写 `@Bean` 注册 Bean 时还添加了一个 `@ConditionalOnMissingBean` ，这是考虑到有可能会有覆盖默认 `XxlJobSpringExecutor` 创建的特殊情况（其实各位看之前与 SpringBoot 整合的模块也能知道，对于核心组件而言，大多数都是有标注 `@ConditionalOnMissingBean` 的）。

### 3.4 测试效果

简单封装好了，下面我们可以测试一下效果。将 `spring-boot-integration-09-xxljob` 工程中的依赖改为我们封装好的 `xxl-job-boot-starter` ，之后去掉 `XxlJobConfiguration` 类，再就是调整一下 `application.properties` 中的配置信息：

```properties
xxl.job.admin-addresses=http://127.0.0.1:18899/xxl-job-admin
xxl.job.appname=demo-xxl-job-executor
xxl.job.accessToken=default_token
```

之后就可以启动工程查看效果了，只要应用能成功启动，并且还能正常被调度任务，就说明我们的封装没有问题。

【更多有关 xxl-job 的使用，大家完全可以参照 xxl-job 的官方文档来测试和使用。下一章我们将分析 xxl-job 与 SpringFramework 整合后使用的 `XxlJobSpringExecutor` 具体的工作机制，以及 xxl-job 底层的相关工作原理】

