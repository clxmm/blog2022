---
title: 03-服务发现-EurekaServer的初始化和启动原理.md
toc: true
tags: springcloud
categories: 
    - [java]
    - [springcloudNetflix]
---

咱们开始吧，刚学习 SpringCloud 的时候先要学习注册中心，也就是服务发现与治理中心。SpringCloudNetflix 的方案是使用 Eureka，咱也都很清楚了，下面咱先搭建一个只有 EurekaServer 的工程。

# 0. 测试环境搭建

pom 依赖只需要两个：

<!--more-->


```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

启动类上标注 `@EnableEurekaServer`：

```java
@EnableEurekaServer
@SpringBootApplication
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

`application.yml` 中配置一些最基础的信息：

```yaml
server:
  port: 9001

spring:
  application:
    name: eureka-server

eureka:
  instance:
    hostname: eureka-server
  client:
    register-with-eureka: false
    fetch-registry: false
    service-url:
      defaultZone: http://localhost:9001/eureka/
```

之后运行主启动类，EurekaServer 便会运行在 9001 端口上。

如果不标注 `@EnableEurekaServer` 注解，即便导入 eureka-server 的依赖，EurekaServer 也不会启动，说明真正打开 EurekaServer 的是 `@EnableEurekaServer` 注解。

# 1. @EnableEurekaServer

```java
@Import(EurekaServerMarkerConfiguration.class)
public @interface EnableEurekaServer {

}

```

它的文档注释非常简单：

> Annotation to activate Eureka Server related configuration.
>
> 用于激活 EurekaServer 相关配置的注解。

它被标注了一个 `@Import` 注解，导入的是一个 `EurekaServerMarkerConfiguration` 的配置类。



# 2. EurekaServerMarkerConfiguration

```java
@Configuration
public class EurekaServerMarkerConfiguration {

	@Bean
	public Marker eurekaServerMarkerBean() {
		return new Marker();
	}

	class Marker {

	}

}
```

这段源码看上去莫名其妙的，它是一个配置类，然后它定义了一个 `Marker` 的内部类，又把这个 `Marker` 注册到 IOC 容器了。但这光秃秃的，也没点别的逻辑，它到底想干啥？果然还是得靠文档注释：

> Responsible for adding in a marker bean to activate EurekaServerAutoConfiguration.
>
> 负责添加标记 Bean 来激活 `EurekaServerAutoConfiguration` 。

好吧，原来它的作用是**给 IOC 容器中添加一个标记，代表要启用 `EurekaServerAutoConfiguration` 的自动配置类**。

那咱就移步 `EurekaServerAutoConfiguration` 来看它的定义了。

# 3. EurekaServerAutoConfiguration

看到 **AutoConfiguration** 结尾的类，咱马上要想到：这个类肯定在 `spring.factories` 文件标注好了，不然没法生效。

果然，在 `spring-cloud-netflix-eureka-server` 的 jar 包中发现了一个 **`spring.factories`** 文件，而文件内部的声明就是如此的简单：

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  org.springframework.cloud.netflix.eureka.server.EurekaServerAutoConfiguration
```

没得跑，来看它的定义和声明吧：

```java
@Configuration
@Import(EurekaServerInitializerConfiguration.class)
@ConditionalOnBean(EurekaServerMarkerConfiguration.Marker.class)
@EnableConfigurationProperties({ EurekaDashboardProperties.class,
		InstanceRegistryProperties.class })
@PropertySource("classpath:/eureka/server.properties")
public class EurekaServerAutoConfiguration extends WebMvcConfigurerAdapter
```

注意看 `@ConditionalOnBean` 的条件：IOC 容器中必须有一个 `EurekaServerMarkerConfiguration.Marker` 类型的 Bean，该配置类才会生效（原来它是这样做自动配置开关的）。

另外注意到它继承了 `WebMvcConfigurerAdapter` ，但全篇没有找到跟 `WebMvcConfigurer` 相关的部分，也没重写对应的方法。那它这是几个意思？这个时候咱要了解一个小背景：

> 在 SpringFramework5.0+ 后，因为接口可以直接声明 **default** 方法，所以 `WebMvcConfigurerAdapter` 被废弃（被标注 `@Deprecated`），替代方案是直接实现 `WebMvcConfigurer` 接口。

那既然是这样， 它还继承着这个适配器类，那咱可以大概猜测：它应该是**旧版本的遗留**。

回到正题，咱看 `EurekaServerAutoConfiguration` 的类定义声明上还有什么值得注意的。除了上面说的，那就只剩下一个了：它导入了一个 `EurekaServerInitializerConfiguration` 。

# 4. 【启动】EurekaServerInitializerConfiguration

```java
@Configuration
public class EurekaServerInitializerConfiguration implements ServletContextAware, SmartLifecycle, Ordered
```

注意它实现了 `SmartLifecycle` 接口，它的核心方法是 `start` ：

```java
public void start() {
    new Thread(new Runnable() {
        @Override
        public void run() {
            try {
                // TODO: is this class even needed now?
                // 初始化、启动 EurekaServer
                eurekaServerBootstrap.contextInitialized(
                        EurekaServerInitializerConfiguration.this.servletContext);
                log.info("Started Eureka Server");

                // 发布Eureka已注册的事件
                publish(new EurekaRegistryAvailableEvent(getEurekaServerConfig()));
                // 修改 EurekaServer 的运行状态
                EurekaServerInitializerConfiguration.this.running = true;
                // 发布Eureka已启动的事件
                publish(new EurekaServerStartedEvent(getEurekaServerConfig()));
            } // catch ......
        }
    }).start();
}
```

> （至此应该进一步意识到为什么上面 `EurekaServerAutoConfiguration` 继承了一个过时的类，`Runnable` 都没换成 Lambda 表达式。。。当然也跟 Eureka 1.x 不继续更新有关吧）

注意看，这个 `start` 方法只干了一件事，它起了一个新的线程来启动 EurekaServer 。这里面核心的 **run** 方法执行了这么几件事，都已经标注在源码中了。

这里面最重要的步骤就是第一步：**初始化、启动 EurekaServer** 。

在继续展开这部分源码之前，要带小伙伴了解一点前置知识。

EurekaServer 本身应该是一个完整的 Servlet 应用，在原生的 EurekaServer 中，EurekaServer 的启动类 `EurekaBootStrap` 类会实现 **`ServletContextListener`** 接口（ Servlet3.0 规范）来引导启动 EurekaServer 。而基于 SpringBoot 构建的应用一般使用嵌入式 Web 容器，没有所谓 Servlet3.0 规范作用的机会了，所以需要另外的启动方式，于是 SpringCloud 在整合这部分时，借助 `ApplicationContext` 中支持的 `LifeCycle` 机制，以此来触发 EurekaServer 的启动。

这段话也可以这样理解：原本 EurekaServer 是个 war 包（像下图的大黑框那样），扔到 Tomcat 里就能启动。结果 SpringCloud 的项目都是通过 SpringBoot 构建的，而 SpringBoot 内置的嵌入式 Web 容器又不像平时咱在外头解压就能用的那种，可以直接把 war 包放到 webapps 下就好使的，所以 SpringCloud 的开发者就得想啊，我咋把 EurekaServer 整合到 SpringBoot 的工程中，还能让它起来呢？emmmm 诶有了！`ApplicationContext` 不是有个生命周期的机制嘛，**当所有单实例 Bean 都初始化完成后，就会回调所有实现了 `Lifecycle` 接口的 Bean 的 `start` 方法！** 那我就这样呗，之前的那个 Bootstrap 类，这次你改成实现 `Lifecycle` 接口，一起注册到 IOC 容器中，这样就可以通过生命周期的回调来初始化 EurekaServer 了。

![](./img/2023/05/cloud03-1.png)

好了下面咱就开始研究 `eurekaServerBootstrap` 的初始化过程。

## 4.0 eurekaServerBootstrap.contextInitialized：初始化、启动EurekaServer

```java
public void contextInitialized(ServletContext context) {
    try {
        initEurekaEnvironment();
        initEurekaServerContext();
        context.setAttribute(EurekaServerContext.class.getName(), this.serverContext);
    } // catch......
}
```

这里面又分为两个部分，依此来看：

## 4.1 initEurekaEnvironment：初始化Eureka的运行环境

```java
private static final String TEST = "test";
private static final String DEFAULT = "default";
	private static final String EUREKA_DATACENTER = "eureka.datacenter";


protected void initEurekaEnvironment() throws Exception {
    log.info("Setting the eureka configuration..");

    // Eureka的数据中心
    String dataCenter = ConfigurationManager.getConfigInstance()
            .getString(EUREKA_DATACENTER);
    if (dataCenter == null) {
        log.info(
                "Eureka data center value eureka.datacenter is not set, defaulting to default");
        ConfigurationManager.getConfigInstance()
                .setProperty(ARCHAIUS_DEPLOYMENT_DATACENTER, DEFAULT);
    }
    else {
        ConfigurationManager.getConfigInstance()
                .setProperty(ARCHAIUS_DEPLOYMENT_DATACENTER, dataCenter);
    }
    // Eureka运行环境
    String environment = ConfigurationManager.getConfigInstance()
            .getString(EUREKA_ENVIRONMENT);
    if (environment == null) {
        ConfigurationManager.getConfigInstance()
                .setProperty(ARCHAIUS_DEPLOYMENT_ENVIRONMENT, TEST);
        log.info(
                "Eureka environment value eureka.environment is not set, defaulting to test");
    }
    else {
        ConfigurationManager.getConfigInstance()
                .setProperty(ARCHAIUS_DEPLOYMENT_ENVIRONMENT, environment);
    }
}
```

这里面的逻辑咱乍一看，貌似都长得差不多啊，都是 **获取 → 判断 → 设置** ，而且它们都有对应的默认值（源码中已标注）。至于这部分是干嘛的呢，咱不得不关注一下 `setProperty` 方法中的两个常量：

```java
private static final String ARCHAIUS_DEPLOYMENT_ENVIRONMENT = "archaius.deployment.environment";
private static final String ARCHAIUS_DEPLOYMENT_DATACENTER = "archaius.deployment.datacenter";

```

配置项的前缀是 `archaius` ，它是 Netflix 旗下的一个**配置管理组件**（提到这里，是不是产生了一种感觉：它会不会跟 SpringCloudConfig 有关系？然而并不是，当引入 SpringCloudConfig 时，archaius 并不会带进来），这个组件可以实现更强大的动态配置，它的底层是 **Apache** 的 `commons-configuration` ：

![](./img/2023/05/cloud03-2.png)

对于这个组件，小册不展开研究了，小伙伴们**只需要知道有这么回事就可以**了，下面的才是重点。

## 4.2 initEurekaServerContext：初始化EurekaServer的运行上下文

```java
protected void initEurekaServerContext() throws Exception {
    // For backward compatibility  兼容低版本Eureka
    JsonXStream.getInstance().registerConverter(new V1AwareInstanceInfoConverter(),
            XStream.PRIORITY_VERY_HIGH);
    XmlXStream.getInstance().registerConverter(new V1AwareInstanceInfoConverter(),
            XStream.PRIORITY_VERY_HIGH);

    if (isAws(this.applicationInfoManager.getInfo())) {
        this.awsBinder = new AwsBinderDelegate(this.eurekaServerConfig,
                this.eurekaClientConfig, this.registry, this.applicationInfoManager);
        this.awsBinder.start();
    }

    // 注册EurekaServerContextHolder，通过它可以很方便的获取EurekaServerContext
    EurekaServerContextHolder.initialize(this.serverContext);

    log.info("Initialized server context");

    // Copy registry from neighboring eureka node
    // 【复杂】Eureka复制集群节点注册表
    int registryCount = this.registry.syncUp();
    this.registry.openForTraffic(this.applicationInfoManager, registryCount);

    // Register all monitoring statistics.
    EurekaMonitors.registerAllStats();
}
```

前面的一大段都是为了低版本兼容而做的一些额外工作，咱不关心这些。中间又是注册了一个 “注册 `EurekaServerContextHolder` 的组件”，通过它可以直接获取 `EurekaServerContext` （它的内部使用简单的单例实现，实现非常简单，小伙伴可自行查看）。

注意最后几行，倒数第二个单行注释的内容：

> Copy registry from neighboring eureka node。
>
> 从相邻的 eureka 节点复制注册表。

节点复制注册表？这很明显是为了 **EurekaServer 集群**而设计的！由此可知 EurekaServer 集群能保证后起来的节点也不会出问题，是这里同步了注册表啊！这一步的操作非常复杂，咱后续另开一章解释。

除了这部分之外，`EurekaServerInitializerConfiguration` 已经没有要配置的组件。

# 小结

1. `@EnableEurekaServer` 注解会激活 EurekaServer 的自动配置，核心是向 IOC 容器注册一个 `Marker` 的内部类。
2. `EurekaServerInitializerConfiguration` 负责 EurekaServer 的初始化，初始化的过程包括本身初始化、运行环境初始化、运行上下文的初始化。
3. EurekaServer 的启动原本是由 `EurekaBootStrap` 借助 Servlet3.0 规范启动，在 SpringCloudNetflix 中由 `EurekaServerBootstrap` 负责启动。

【由 `EurekaServerAutoConfiguration` 引发的其他自动配置类本章都差不多解析完成了，下一章咱开始看 `EurekaServerAutoConfiguration` 本身注册的组件】

