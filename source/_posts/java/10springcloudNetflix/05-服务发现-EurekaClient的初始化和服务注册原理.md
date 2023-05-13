---
title: 05-服务发现-EurekaClient的初始化和服务注册原理
toc: true
tags: springcloud
categories: 
    - [java]
    - [springcloudNetflix]
---

前面两章咱了解了 EurekaServer 的初始化和启动原理，只有 server 没有 client 那不算一个合理的微服务，接下来的两章咱来研究 EurekaClient 的初始化原理，以及 EurekaClient 是如何注册到 EurekaServer 上的。

接下来的两章基于一个 EurekaServer 和一个 EurekaClient 的环境演示：

<!--more-->

## 0. 测试环境搭建

前面已经有 EurekaServer 了，下面咱搭一个 EurekaClient 的基本环境。pom 的依赖也是引入两个即可：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

启动类上标注 `@EnableDiscoveryClient` ：

```java
@EnableDiscoveryClient
@SpringBootApplication
public class EurekaClientApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaClientApplication.class, args);
    }
}
```

同样的，`application.yml` 中配置一些最基础的信息：

```yaml
server:
  port: 8080

spring:
  application:
    name: eureka-client

eureka:
  instance:
    instance-id: eureka-client
    prefer-ip-address: true
  client:
    service-url:
      defaultZone: http://localhost:9001/eureka/
```

运行 SpringBoot 的主启动类，EurekaClient 便会运行在 8080 端口上。

即便不标注 `@EnableDiscoveryClient` 注解，同样也会启动 EurekaClient 并按照配置规则注册到 EurekaServer 上，证明 `@EnableDiscoveryClient` 的作用没有那么大，而是决定在 `@EnableAutoConfiguration` 上，且不难猜出自动配置类的名称应该是 **`EurekaClientAutoConfiguration`** ，不过咱还是找找对应的 jar 包。

翻看 `spring-cloud-starter-netflix-eureka-client` 包，很惊奇的发现并没有 `spring.factories` 文件，那对应的自动配置类在哪里呢？从它的 pom 中，发现了它还依赖 `spring-cloud-netflix-eureka-client` （注意没有 starter ），那咱就翻到这个 jar 包中，果然发现了有 `spring.factories` 文件。打开来看，发现了咱猜测的 `EurekaClientAutoConfiguration` ：

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.cloud.netflix.eureka.config.EurekaClientConfigServerAutoConfiguration,\
org.springframework.cloud.netflix.eureka.config.EurekaDiscoveryClientConfigServiceAutoConfiguration,\
org.springframework.cloud.netflix.eureka.EurekaClientAutoConfiguration,\
org.springframework.cloud.netflix.ribbon.eureka.RibbonEurekaAutoConfiguration,\
org.springframework.cloud.netflix.eureka.EurekaDiscoveryClientConfiguration
```

不过这里面配置的好多，咱一个一个来看，先挑上面咱一下子猜出来的 `EurekaClientAutoConfiguration` 。

## 1. EurekaClientAutoConfiguration

这个类的头部注解声明的是真的多。。。

```java
@Configuration
@EnableConfigurationProperties
@ConditionalOnClass(EurekaClientConfig.class)
@Import(DiscoveryClientOptionalArgsConfiguration.class)
@ConditionalOnBean(EurekaDiscoveryClientConfiguration.Marker.class)
@ConditionalOnProperty(value = "eureka.client.enabled", matchIfMissing = true)
@ConditionalOnDiscoveryEnabled
@AutoConfigureBefore({ NoopDiscoveryClientAutoConfiguration.class,
		CommonsClientAutoConfiguration.class, ServiceRegistryAutoConfiguration.class })
@AutoConfigureAfter(name = {
		"org.springframework.cloud.autoconfigure.RefreshAutoConfiguration",
		"org.springframework.cloud.netflix.eureka.EurekaDiscoveryClientConfiguration",
		"org.springframework.cloud.client.serviceregistry.AutoServiceRegistrationAutoConfiguration" })
public class EurekaClientAutoConfiguration
```

一个一个来看，先把那些不太重要的都看一眼：

- `@EnableConfigurationProperties` ：激活配置文件到实体类的映射，这里面需要的是 `EurekaClientConfig`
- 各种 `@ConditionalOnxxx` ，都是走条件判断的，不再赘述
- `@AutoConfigureAfter` ：设定自动配置类的先后顺序，注意这里面的几个自动配置类都很关键

由于这几个自动配置类，以及自动配置类中嵌套的内部配置类是否起作用，可以设置 `logging.level.root=debug` 观察 SpringBoot 打印的自动配置报告，下面咱一个一个解释上面几个自动配置类在自动配置报告中匹配生效的部分。

## 2. RefreshAutoConfiguration

```java
@Configuration
@ConditionalOnClass(RefreshScope.class)
@ConditionalOnProperty(name = RefreshAutoConfiguration.REFRESH_SCOPE_ENABLED, matchIfMissing = true)
@AutoConfigureBefore(HibernateJpaAutoConfiguration.class)
public class RefreshAutoConfiguration
```

这个自动配置类并没有继续往里导入配置类或组件，或者设置其他的关键顺序。它的文档注释原文翻译：

> Autoconfiguration for the refresh scope and associated features to do with changes in the Environment (e.g. rebinding logger levels).
>
> 刷新范围和相关功能的自动配置与环境中的更改有关（例如重新绑定记录器级别）。

由文档注释咱先明确这个自动配置类的作用核心：**刷新**，换句话说，这个自动配置类的组件都应该是带有一些**刷新**的味道。这里涉及到**配置中心与配置刷新**的部分了，详细的内容咱会放在**分布式配置中心 Config** 部分详细解析，这里咱只是看一看这些组件，大概有一个印象。下面咱看 `RefreshAutoConfiguration` 配置的内容。

### 2.1 RefreshScope

```java
@Bean
@ConditionalOnMissingBean(RefreshScope.class)
public static RefreshScope refreshScope() {
    return new RefreshScope();
}
```

这个组件从名字上能很容易理解，它是一个**Bean的作用域刷新器**，它的文档注释很长，截取出关键部分如下：

> A Scope implementation that allows for beans to be refreshed dynamically at runtime (see refresh(String) and refreshAll()). If a bean is refreshed then the next time the bean is accessed (i.e. a method is executed) a new instance is created. All lifecycle methods are applied to the bean instances, so any destruction callbacks that were registered in the bean factory are called when it is refreshed, and then the initialization callbacks are invoked as normal when the new instance is created. A new bean instance is created from the original bean definition, so any externalized content (property placeholders or expressions in string literals) is re-evaluated when it is created.
>
> Bean的范围实现，它允许在运行时动态刷新bean（请参见 `refresh(String)` 和 `refreshAll()` ）。如果刷新了Bean，则下次访问该Bean（即执行一个特定的方法）时，将创建一个新实例。所有生命周期方法都适用于Bean实例，因此刷新时将调用在Bean工厂中注册的所有销毁回调，然后在创建新实例时照常调用初始化回调。根据原始bean定义创建一个新bean实例，因此在创建任何外部化内容（属性占位符或字符串文字中的表达式）时都会对其进行重新刷新。

这部分已经足以看明白这个组件的用途了。`RefreshScope` 可以在应用配置刷新时重新加载一些特定的 Bean ，它的底层是扩展了 `@Scope` 的类型，并配合这个组件实现 Bean 的刷新。这个组件解释起来很复杂，咱放到**分布式配置中心 Config** 部分再详细解析，这里大概有个印象即可。

### 2.2 LoggingRebinder

```java
@Bean
@ConditionalOnMissingBean
public static LoggingRebinder loggingRebinder() {
    return new LoggingRebinder();
}
```

这个组件从名字上也能很容易理解，它是一个日志的**重绑定器**，它的文档注释原文翻译：

> Listener that looks for EnvironmentChangeEvent and rebinds logger levels if any changed.
>
> 查找 `EnvironmentChangeEvent` 并重新绑定日志级别（如果有任何更改）的监听器。

文档注释很明确了，它负责**刷新日志打印级别**。可以猜测，它与上面类似，也是在配置刷新时起作用。

### 2.3 ContextRefresher

```java
@Bean
@ConditionalOnMissingBean
public ContextRefresher contextRefresher(ConfigurableApplicationContext context, RefreshScope scope) {
    return new ContextRefresher(context, scope);
}
```

这个 `ContextRefresher` 就是前面提到的，当配置刷新时首先触发的组件了。它与下面的 `RefreshEventListener` 共同配合实现配置刷新后的容器和组件刷新。

### 2.4 RefreshEventListener

```java
@Bean
public RefreshEventListener refreshEventListener(ContextRefresher contextRefresher) {
    return new RefreshEventListener(contextRefresher);
}
```

一看就知道是监听器，它实现了 `SmartApplicationListener` ，对应的核心方法如下：

```java
@Override
public void onApplicationEvent(ApplicationEvent event) {
    if (event instanceof ApplicationReadyEvent) {
        handle((ApplicationReadyEvent) event);
    }
    else if (event instanceof RefreshEvent) {
        handle((RefreshEvent) event);
    }
}
```

可以发现它监听了两个事件：`ApplicationReadyEvent` 、`RefreshEvent` ，也不难理解，这个监听器会在应用准备就绪、和应用发出刷新事件时触发，完成配置刷新。

### 2.5 RefreshScopeBeanDefinitionEnhancer

这个类是一个标有 `@Component` 的组件，它没有文档注释，只能靠经验来看了。

它实现了 `BeanDefinitionRegistryPostProcessor` 接口，它的作用我在 boot 源码的第 12 章 5.1 节中提到过，如果小伙伴们对这个组件不了解的话一定要回头去看，这里咱直接说，它的核心方法是 `postProcessBeanDefinitionRegistry` ：

#### 2.5.1 postProcessBeanDefinitionRegistry

```java
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
    bindEnvironmentIfNeeded(registry);
    for (String name : registry.getBeanDefinitionNames()) {
        BeanDefinition definition = registry.getBeanDefinition(name);
        if (isApplicable(registry, name, definition)) {
            // ......
```

看它的处理逻辑，它会把 IOC 容器中所有的 `BeanDefinition` 都取出来，进行一个判断：`isApplicable` 。

#### 2.5.2 isApplicable

逻辑思路已标注在源码中：

```java
public static final String REFRESH_SCOPE_NAME = "refresh";
private Set<String> refreshables = new HashSet<>(
        Arrays.asList("com.zaxxer.hikari.HikariDataSource"));

private boolean isApplicable(BeanDefinitionRegistry registry, String name,
        BeanDefinition definition) {
    // 先检查Bean的scope是否就是refresh
    String scope = definition.getScope();
    if (REFRESH_SCOPE_NAME.equals(scope)) {
        // Already refresh scoped
        return false;
    }
    // 如果不是，则获取Bean的类型
    String type = definition.getBeanClassName();
    if (!StringUtils.hasText(type) && registry instanceof BeanFactory) {
        Class<?> cls = ((BeanFactory) registry).getType(name);
        if (cls != null) {
            type = cls.getName();
        }
    }
    // 判断类型是否在refreshables集合中
    if (type != null) {
        return this.refreshables.contains(type);
    }
    return false;
}
```

这部分的判断逻辑比较明晰，可能唯一存在疑问的是这个 `refreshables` 集合的内容。难得这个变量上有文档注释，咱看一眼：

> Class names for beans to post process into refresh scope. Useful when you don't control the bean definition (e.g. it came from auto-configuration).
>
> Bean 的类名称，用于将进程发布到 scope 为 refresh 的 Bean 中。当您不控制 BeanDefinition 时很有用（例如，它来自 自动配置）

看上去有些绕，但核心想表达的意思咱是能 get 到的，它会**把一些作用域为 refresh 的 Bean 存放到这个 set 中，用于触发重刷新效果**。

那到这里，咱就大概理解 `isApplicable` 方法的含义了：**判断一个 Bean 是否是一个可刷新的 Bean**（ scope 为 **refresh**）。

#### 2.5.3 回到if结构

```java
    if (isApplicable(registry, name, definition)) {
        BeanDefinitionHolder holder = new BeanDefinitionHolder(definition,
                name);
        BeanDefinitionHolder proxy = ScopedProxyUtils
                .createScopedProxy(holder, registry, true);
        definition.setScope("refresh");
        if (registry.containsBeanDefinition(proxy.getBeanName())) {
            registry.removeBeanDefinition(proxy.getBeanName());
        }
        registry.registerBeanDefinition(proxy.getBeanName(),
                proxy.getBeanDefinition());
    }
```

它这里面判断如果确实 scope 是 refresh ，则会触发一个**代理对象**的创建（ `ScopedProxyUtils` 类来自于 aop 包）。它将原有的 `BeanDefinition` 取出来，封装到一个 `BeanDefinitionHolder` 中，再利用 `ScopedProxyUtils` 创建一个代理对象的 `BeanDefinitionHolder` ，之后设置 `BeanDefinition` 的 scope 为 refresh ，最后将 IOC 容器中原有的 `BeanDefinition` 移除掉，放入代理对象对应的 `BeanDefinition` 。这段逻辑看下来，核心的作用也就明晰了：**它将声明了 scope 为 refresh 的但没有标注 refresh 的 Bean 修改为 refresh 作用域**。

这部分可能不是很好理解，辅一个流程图来解释下这段逻辑：

![](./img/2023/05/cloud05-1.png)

至此，`RefreshAutoConfiguration` 的所有配置解析完毕。

## 3. AutoServiceRegistrationAutoConfiguration

```java
@Configuration
@Import(AutoServiceRegistrationConfiguration.class)
@ConditionalOnProperty(value = "spring.cloud.service-registry.auto-registration.enabled", 
                       matchIfMissing = true)
public class AutoServiceRegistrationAutoConfiguration
```

这个配置类相比起来讲更简单，它只导入了一个配置类：`AutoServiceRegistrationConfiguration` 。而它：

```java
@Configuration
@EnableConfigurationProperties(AutoServiceRegistrationProperties.class)
@ConditionalOnProperty(value = "spring.cloud.service-registry.auto-registration.enabled", 
                       matchIfMissing = true)
public class AutoServiceRegistrationConfiguration
```

仅仅是激活了 `AutoServiceRegistrationProperties` 的配置映射而已，且配置类中也没有多余的声明。

## 4. EurekaDiscoveryClientConfiguration

```java
@Configuration
@EnableConfigurationProperties
@ConditionalOnClass(EurekaClientConfig.class)
@ConditionalOnProperty(value = "eureka.client.enabled", matchIfMissing = true)
@ConditionalOnDiscoveryEnabled
public class EurekaDiscoveryClientConfiguration
```

这上面的声明也不多，且没有多余的导入组件，那咱可以来看它配置的组件。

### 4.1 eurekaDiscoverClientMarker

```java
@Bean
public Marker eurekaDiscoverClientMarker() {
    return new Marker();
}
```

这一看很眼熟嘛，跟 `EurekaServerAutoConfiguration` 的套路肯定是一样的，不再赘述。

### 4.2 EurekaHealthCheckHandlerConfiguration

```java
@Configuration
@ConditionalOnProperty(value = "eureka.client.healthcheck.enabled", matchIfMissing = false)
protected static class EurekaHealthCheckHandlerConfiguration {
    @Autowired(required = false)
    private HealthAggregator healthAggregator = new OrderedHealthAggregator();

    @Bean
    @ConditionalOnMissingBean(HealthCheckHandler.class)
    public EurekaHealthCheckHandler eurekaHealthCheckHandler() {
        return new EurekaHealthCheckHandler(this.healthAggregator);
    }
}

```

又是一个配置类，它里面配置了一个 `EurekaHealthCheckHandler` ，从类名上可以简单看出它是一个健康检查处理器。它的文档注释原文翻译：

> A Eureka health checker, maps the application status into InstanceInfo.InstanceStatus that will be propagated to Eureka registry. On each heartbeat Eureka performs the health check invoking registered HealthCheckHandler. By default this implementation will perform aggregation of all registered HealthIndicator through registered HealthAggregator.
>
> 一个Eureka运行状况检查器，将应用程序状态映射到 `InstanceInfo.InstanceStatus`，该实例将传播到Eureka注册表。在每个心跳上，Eureka都会执行运行状况检查，调用已注册的 `HealthCheckHandler` 。默认情况下，此实现将通过注册的 `HealthAggregator` 执行所有注册的 `HealthIndicator` 的汇总。

文档注释提到了应用程序的状态和服务的状态，这个地方在 `EurekaHealthCheckHandler` 的私有成员中得以体现：

```java
private static final Map<Status, InstanceInfo.InstanceStatus> STATUS_MAPPING = new HashMap<Status, InstanceInfo.InstanceStatus>() {
    {
        put(Status.UNKNOWN, InstanceStatus.UNKNOWN);
        put(Status.OUT_OF_SERVICE, InstanceStatus.OUT_OF_SERVICE);
        put(Status.DOWN, InstanceStatus.DOWN);
        put(Status.UP, InstanceStatus.UP);
    }
};
```

它使用初始化代码块分别映射了四种 Eureka 的服务状态，注意这个 `Status` 来自于 `org.springframework.boot.actuate.health` ，即 `spring-boot-actuator` 包。这样也就明白文档注释的意思了，它将 SpringBoot 原生有的应用健康状态检查同步到 SpringCloud 的 EurekaClient 上，并同步给 EurekaServer ，让 EurekaServer 也能感知到每个服务自身的应用健康状态。

### 4.3 EurekaClientConfigurationRefresher

```java
@Configuration
@ConditionalOnClass(RefreshScopeRefreshedEvent.class)
protected static class EurekaClientConfigurationRefresher
        implements ApplicationListener<RefreshScopeRefreshedEvent>
```

看类名，我觉得可以大胆猜测它跟上面的 `RefreshScopeBeanDefinitionEnhancer` 有关系或者协同工作。注意它是一个监听器，监听的事件是 `RefreshScopeRefreshedEvent` 。这个事件的发布来源就是上面 2.1 节提到的 `RefreshScope` ：

```java
public boolean refresh(String name) {
    // ......
    if (super.destroy(name)) {
        this.context.publishEvent(new RefreshScopeRefreshedEvent(name));
        return true;
    }
    return false;
}
```

那 `RefreshScope` 发布了事件，被 `EurekaClientConfigurationRefresher` 监听，那这个刷新器肯定要执行额外的操作，直接来到它的核心方法 `onApplicationEvent` ：

```java
private EurekaAutoServiceRegistration autoRegistration;

public void onApplicationEvent(RefreshScopeRefreshedEvent event) {
    // This will force the creation of the EurkaClient bean if not already created
    // to make sure the client will be reregistered after a refresh event
    if (eurekaClient != null) {
        eurekaClient.getApplications();
    }
    if (autoRegistration != null) {
        // register in case meta data changed
        this.autoRegistration.stop();
        this.autoRegistration.start();
    }
}
```

首先它回调了 `eurekaClient` 的 `getApplications` 方法，从上面的单行注释也能理解，它是为了避免 Bean 还有没初始化好的：

> 如果尚未创建，则将强制创建 EurkaClient bean，以确保刷新事件后将重新注册客户端。

之后它对 `autoRegistration` 做了相当于一次重启动作，这个 `autoRegistration` 的类型是 `EurekaAutoServiceRegistration` ，这个会在下一章解读 `EurekaClientAutoConfiguration` 时提到。

至此，`EurekaDiscoveryClientConfiguration` 也都解析完成了。

## 小结

1. SpringBoot 的主启动类上不需要标注 `@EnableDiscoveryClient` 也可以启动 EurekaClient ，证明 EurekaClient 相关的自动配置类不依赖于某个特定的对象组件。
2. `RefreshAutoConfiguration` 负责与分布式配置中心配合，完成应用配置的刷新。
3. `EurekaDiscoveryClientConfiguration` 注册了服务发现的组件、微服务健康检查的组件等。

【EurekaClient 除自身的 `EurekaClientAutoConfiguration` 之外的组件注册都解析完成了，下一章咱开始看关于 `EurekaClientAutoConfiguration` 的组件注册，以及 EurekaClient 的启动原理】