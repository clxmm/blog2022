---
title: 14-负载均衡-Ribbon自定义策略的实现原理.md
toc: true
tags: springcloud
categories: 
    - [java]
    - [springcloudNetflix]
---

## 0. 问题引入


<!--more-->

前面在第11章中咱解释了自定义 Ribbon 负载均衡策略的方式：

```properties
eureka-client.ribbon.NFLoadBalancerRuleClassName=com.netflix.loadbalancer.RandomRule
```

其实这种配置方式就在 SpringCloudNetflix 的官方文档 7.4 章节有介绍：

![](./img/2023/05/cloud14-1.png)

除了这种方式，官方文档的 7.2 章节还介绍了使用注解方式的自定义配置：

![](./img/2023/04/cloud14-2.png)

这里面还特别说明了一点：声明的自定义配置类不能被 `@SpringBootApplication` 或者 `@ComponentScan` 扫到，否则会被所有自定义策略共享，那也就达不到服务定制化策略的目的了。可官方文档只是这么说了，原因呢？咱这一章就来研究这个问题。

## 1. @RibbonClient

```java
@Configuration
@Import(RibbonClientConfigurationRegistrar.class)
public @interface RibbonClient
```

注意这个注解中导入了一个 `RibbonClientConfigurationRegistrar` ，这个类咱之前在第11章已经看过了，小伙伴们还有印象吗？它可以解析 `@RibbonClient` 注解中的属性，并且向IOC容器中注册一些配置类。

```java
public void registerBeanDefinitions(AnnotationMetadata metadata,
        BeanDefinitionRegistry registry) {
    Map<String, Object> attrs = metadata
            .getAnnotationAttributes(RibbonClients.class.getName(), true);
    if (attrs != null && attrs.containsKey("value")) {
        AnnotationAttributes[] clients = (AnnotationAttributes[]) attrs.get("value");
        for (AnnotationAttributes client : clients) {
            registerClientConfiguration(registry, getClientName(client), client.get("configuration"));
        }
    }
    if (attrs != null && attrs.containsKey("defaultConfiguration")) {
        String name;
        if (metadata.hasEnclosingClass()) {
            name = "default." + metadata.getEnclosingClassName();
        } else {
            name = "default." + metadata.getClassName();
        }
        registerClientConfiguration(registry, name, attrs.get("defaultConfiguration"));
    }
    Map<String, Object> client = metadata
            .getAnnotationAttributes(RibbonClient.class.getName(), true);
    String name = getClientName(client);
    if (name != null) {
        registerClientConfiguration(registry, name, client.get("configuration"));
    }
}

private void registerClientConfiguration(BeanDefinitionRegistry registry, Object name,
        Object configuration) {
    BeanDefinitionBuilder builder = BeanDefinitionBuilder
            .genericBeanDefinition(RibbonClientSpecification.class);
    builder.addConstructorArgValue(name);
    builder.addConstructorArgValue(configuration);
    registry.registerBeanDefinition(name + ".RibbonClientSpecification",
            builder.getBeanDefinition());
}
```

它向IOC容器注册这些 `RibbonClientSpecification` 干嘛用呢？不是说这些配置类不能让IOC容器发现吗？那还注册？带着这个疑问，咱来试着查一下这个 `RibbonClientSpecification` 的使用位置。

## 2. RibbonClientSpecification与SpringClientFactory

借助IDEA，发现 `RibbonClientSpecification` 有4个使用位置：

![](./img/2023/05/cloud4-3.png)

注意最底下，它还是 `SpringClientFactory` 继承的泛型！还有最上面，在 `RibbonAutoConfiguration` 中它定义了一组 `configurations` ，而且在 `SpringClientFactory` 的构造方法中还给了它！

```java
@Bean
public SpringClientFactory springClientFactory() {
    SpringClientFactory factory = new SpringClientFactory();
    factory.setConfigurations(this.configurations); // 【传入】
    return factory;
}
```

那合着能利用 `RibbonClientSpecification` 的只有 `SpringClientFactory` 呗？可这个 `SpringClientFactory` 到底有什么功力能把这些配置类给处理掉呢？咱继续往里看。

### 2.1 SpringClientFactory的父类NamedContextFactory

```java
public class SpringClientFactory extends NamedContextFactory<RibbonClientSpecification>

public abstract class NamedContextFactory<C extends NamedContextFactory.Specification>
		implements DisposableBean, ApplicationContextAware {
    // ......
	private Map<String, AnnotationConfigApplicationContext> contexts = new ConcurrentHashMap<>();
	private Map<String, C> configurations = new ConcurrentHashMap<>();
	private ApplicationContext parent;
	private Class<?> defaultConfigType;
```

注意这里面的设计：`Map<String, AnnotationConfigApplicationContext>` ！是不是突然有了一点想法？现在咱是不是可以大胆猜想，`SpringClientFactory` 内部维护了多个子IOC容器，这些IOC容器各自都存有不同的配置，供外部的应用级IOC容器使用！

带着这个猜想，咱继续往下看。

### 2.2 SpringClientFactory的构造方法

```java
static final String NAMESPACE = "ribbon";

public SpringClientFactory() {
    super(RibbonClientConfiguration.class, NAMESPACE, "ribbon.client.name");
}
```

构造方法中它传入了一个 `RibbonClientConfiguration` ，这个配置类咱前面几章都没见过啊，它又是什么呢？

### 2.3 RibbonClientConfiguration

```java
@Configuration
@EnableConfigurationProperties
@Import({ HttpClientConfiguration.class, OkHttpRibbonConfiguration.class,
        RestClientRibbonConfiguration.class, HttpClientRibbonConfiguration.class })
public class RibbonClientConfiguration
```

这里面它又导入了几个别的配置类，这些暂且不管了，借助IDEA发现它并没有被 `@Import` 标注，也没有包扫描扫到它：

![](./img/2023/05/cloud14-3.png)

那它就只能在父类 `NamedContextFactory` 中使用咯，进入 `NamedContextFactory` 的构造方法中看一眼。

### 2.4 NamedContextFactory的构造方法

```java
public NamedContextFactory(Class<?> defaultConfigType, String propertySourceName,
        String propertyName) {
    this.defaultConfigType = defaultConfigType;
    this.propertySourceName = propertySourceName;
    this.propertyName = propertyName;
}
```

到这里咱发现 `RibbonClientConfiguration` 原来是在 `NamedContextFactory` 中被设置为默认的配置类。可默认配置类又是在哪里起效呢？继续往下看。

### 2.5 RibbonClientConfiguration的使用位置与SpringClientFactory的结构

（这里只列出部分源码）

```java
protected AnnotationConfigApplicationContext createContext(String name) {
    AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
    // ......
    // 这里注册了RibbonClientConfiguration
    context.register(PropertyPlaceholderAutoConfiguration.class,
            this.defaultConfigType);
    // ......
    context.refresh();
    return context;
}
```

`defaultConfigType` 只有这一个地方被使用到，通篇来看它是创建IOC容器的方法，结合前面咱看到的 `NamedContextFactory` 的成员，应该知道它创建的IOC容器都给放到那个 Map 里头了，源码中也是这样实现的：

```java
protected AnnotationConfigApplicationContext getContext(String name) {
    if (!this.contexts.containsKey(name)) {
        synchronized (this.contexts) {
            if (!this.contexts.containsKey(name)) {
                // 双检锁获取不到，创建新的IOC容器
                this.contexts.put(name, createContext(name));
            }
        }
    }
    return this.contexts.get(name);
}
```

看到这里小伙伴们是否会有一点别的疑惑，这些IOC容器对应的 `key` 现在看起来是 `name` ，结合前面的内容来讲，会不会这些 **name 就是服务名称**啊！哎还真别说，就是 `spring.application.name` ，咱通过Debug可以很容易的验证出来：

那咱就明白了，`SpringClientFactory` 中的构造应该是这样的：

![](./img/2023/05/cloud14-4.png)

那咱就明白了，`SpringClientFactory` 中的构造应该是这样的：

![](./img/2023/05/cloud14-5.png)

### 2.6 内部IOC容器的初始化

这次咱来看 `NamedContextFactory` 中 `createContext` 方法的全部的方法体：（核心注释已标注在源码中）

```java
protected AnnotationConfigApplicationContext createContext(String name) {
    AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
    // 2.6.1 注册由@RibbonClient显式声明的配置类
    if (this.configurations.containsKey(name)) {
        for (Class<?> configuration : this.configurations.get(name)
                .getConfiguration()) {
            context.register(configuration);
        }
    }
    // 2.6.2 注册外部传入的两个自动配置类RibbonAutoConfiguration、RibbonEurekaAutoConfiguration
    for (Map.Entry<String, C> entry : this.configurations.entrySet()) {
        if (entry.getKey().startsWith("default.")) {
            for (Class<?> configuration : entry.getValue().getConfiguration()) {
                context.register(configuration);
            }
        }
    }
    // 注册两个默认的配置类
    context.register(PropertyPlaceholderAutoConfiguration.class,
            this.defaultConfigType); // defaultConfigType : RibbonClientConfiguration
    context.getEnvironment().getPropertySources().addFirst(new MapPropertySource(
            this.propertySourceName,
            Collections.<String, Object>singletonMap(this.propertyName, name)));
    if (this.parent != null) {
        // Uses Environment from parent as well as beans
        // 设置为父子容器
        context.setParent(this.parent);
        // jdk11 issue  https://github.com/spring-cloud/spring-cloud-netflix/issues/3101
        context.setClassLoader(this.parent.getClassLoader());
    }
    context.setDisplayName(generateDisplayName(name));
    // 配置类全部设置完毕，启动IOC容器
    context.refresh();
    return context;
}
```

这部分的逻辑走下来很清晰明了，这里面熟悉的部分咱就不多解释了，咱关注一下上面注册配置类的两部分内容。

#### 2.6.1 注册由@RibbonClient显式声明的配置类

注意这部分 C 的类型：

```java
public abstract class NamedContextFactory<C extends NamedContextFactory.Specification>
		implements DisposableBean, ApplicationContextAware
```

它是 `NamedContextFactory.Specification` 类型，而在 Ribbon 中又恰好有一个实现类：`RibbonClientSpecification` 。诶？看到这里是不是又有点似曾相识的感觉？这个家伙不就是这一章节的大标题嘛，合着这个家伙在这里用到的，这些 `@RibbonClient` 注解在被 `RibbonClientConfigurationRegistrar` 解析好之后，把那里头的配置类封装成 `RibbonClientSpecification` 放到 `configurations` 里面了，恰好在创建子容器时又把这些配置类拿出来了，所以才形成了微服务的自定义负载均衡。

#### 2.6.2 注册外部传入的两个自动配置类

在第11章的1.3节咱就看到了，传入了两个自动配置类：`RibbonAutoConfiguration` 、`RibbonEurekaAutoConfiguration` 。



由于在内部也需要进行负载均衡的配置，所以这里注册这些自动配置类都是必要的。

注册好这些配置类，刷新容器，完成子容器的创建。

### 2.7 RibbonClientConfiguration中其他需要注意的

咱知道，默认情况下的负载均衡策略是带分区的轮询 `ZoneAvoidanceRule` ，这个默认的策略注册以及其他负载均衡相关的组件（如负载均衡器 `ZoneAwareLoadBalancer` ），都在 `RibbonClientConfiguration` 中：

```java
@Bean
@ConditionalOnMissingBean
public IRule ribbonRule(IClientConfig config) {
    if (this.propertiesFactory.isSet(IRule.class, name)) {
        return this.propertiesFactory.get(IRule.class, config, name);
    }
    ZoneAvoidanceRule rule = new ZoneAvoidanceRule();
    rule.initWithNiwsConfig(config);
    return rule;
}

@Bean
@ConditionalOnMissingBean
public ILoadBalancer ribbonLoadBalancer(IClientConfig config,
        ServerList<Server> serverList, ServerListFilter<Server> serverListFilter,
        IRule rule, IPing ping, ServerListUpdater serverListUpdater) {
    if (this.propertiesFactory.isSet(ILoadBalancer.class, name)) {
        return this.propertiesFactory.get(ILoadBalancer.class, config, name);
    }
    return new ZoneAwareLoadBalancer<>(config, rule, ping, serverList,
            serverListFilter, serverListUpdater);
}
```

## 小结

1. `SpringClientFactory` 的作用不仅仅是创建负载均衡器，内部还维护了多个子IOC容器，而且是为每个服务名称都创建一个独立的子IOC容器；
2. `@RibbonClient` 的起效位置是在 `NamedContextFactory` 的子IOC容器构建时起作用，故官方文档中描述的不能被应用级IOC容器扫描到是有原理可循的;
3. `RibbonClientConfiguration` 中声明了负载均衡需要的基础组件和默认策略等。

【到此为止，Ribbon 的相关原理也就解析完毕了，不过负载均衡还没有完事。咱在实际开发中，对于负载均衡一般都是借助 Ribbon + Feign 实现，接下来的几篇咱来解析 Feign 的相关原理】



 

