---
title: 15-负载均衡-Feign的初始化与启动原理
toc: true
tags: springcloud
categories: 
    - [java]
    - [springcloudNetflix]
---

前面几章咱把 Ribbon 的内容基本都解析完成了，不过实际开发中大多数情况还是使用 Feign 配合 Ribbon 进行服务调用。那这一章咱先按照惯例，解析 Feign 的初始化与启动原理。

使用 Feign 时，首要的工作是在主启动类上标注 `@EnableFeignClients` 注解，暂且不考虑自动装配的部分，先看看它都起了什么作用。


<!--more-->

## 1. @EnableFeignClients

```java
@Import(FeignClientsRegistrar.class)
public @interface EnableFeignClients
```

它导入了一个 `FeignClientsRegistrar` ，咱一看类名就知道它肯定是一个 `ImportBeanDefinitionRegistrar` ，直接来到它的核心方法：

```java
public void registerBeanDefinitions(AnnotationMetadata metadata,
        BeanDefinitionRegistry registry) {
    registerDefaultConfiguration(metadata, registry);
    registerFeignClients(metadata, registry);
}
```

这里它直接调了另外两个方法，那很显然它要注册两部分 `BeanDefinition` ，一样一样来看。

### 1.1 registerDefaultConfiguration

```java
private void registerDefaultConfiguration(AnnotationMetadata metadata,
        BeanDefinitionRegistry registry) {
    Map<String, Object> defaultAttrs = metadata
            .getAnnotationAttributes(EnableFeignClients.class.getName(), true);

    if (defaultAttrs != null && defaultAttrs.containsKey("defaultConfiguration")) {
        String name;
        if (metadata.hasEnclosingClass()) {
            name = "default." + metadata.getEnclosingClassName();
        } else {
            name = "default." + metadata.getClassName();
        }
        registerClientConfiguration(registry, name,
                defaultAttrs.get("defaultConfiguration"));
    }
}
```

仔细观察方法实现，是不是感觉就在上一章似曾相识一样？对了，Feign 的设计里面很多内容都跟 Ribbon 相似甚至相同（ `registerClientConfiguration` 连方法名都一样，可想而知里面的实现肯定也基本一致 [ 恩，确实基本一致 ] ）。

这部分的处理也就很容易理解了吧，它就是处理关于 Feign 的默认配置，将来也要注册到子IOC容器中。

### 1.2 registerFeignClients

方法名很容易理解，这个方法是注册 `@FeignClients` 标注的组件。这部分的源码很长，核心注释已标注在源码中：

```java
public void registerFeignClients(AnnotationMetadata metadata,
        BeanDefinitionRegistry registry) {
    // 包扫描器，默认用来扫描@EnableFeignClients标注类所在包及子包下的所有@FeignClients组件
    ClassPathScanningCandidateComponentProvider scanner = getScanner();
    scanner.setResourceLoader(this.resourceLoader);

    Set<String> basePackages;
    
    Map<String, Object> attrs = metadata.getAnnotationAttributes(EnableFeignClients.class.getName());
    AnnotationTypeFilter annotationTypeFilter = new AnnotationTypeFilter(FeignClient.class);
    // 1.2.1 该属性用来决定是否启用包扫描
    final Class<?>[] clients = attrs == null ? null : (Class<?>[]) attrs.get("clients");
    if (clients == null || clients.length == 0) {
        // 设置扫描过滤器，被扫描的类/接口上必须标注@FeignClient注解
        scanner.addIncludeFilter(annotationTypeFilter);
        basePackages = getBasePackages(metadata);
    } else {
        // 省略解析显式声明FeignClient的部分......
    }

    for (String basePackage : basePackages) {
        // 扫描所有被@FeignClient标注的接口
        Set<BeanDefinition> candidateComponents = scanner.findCandidateComponents(basePackage);
        for (BeanDefinition candidateComponent : candidateComponents) {
            if (candidateComponent instanceof AnnotatedBeanDefinition) {
                // verify annotated class is an interface
                AnnotatedBeanDefinition beanDefinition = (AnnotatedBeanDefinition) candidateComponent;
                AnnotationMetadata annotationMetadata = beanDefinition.getMetadata();
                // @FeignClient必须标注接口
                Assert.isTrue(annotationMetadata.isInterface(),
                        "@FeignClient can only be specified on an interface");

                Map<String, Object> attributes = annotationMetadata
                        .getAnnotationAttributes(FeignClient.class.getCanonicalName());

                String name = getClientName(attributes);
                // 1.2.2 注册Feign的配置
                registerClientConfiguration(registry, name, attributes.get("configuration"));

                // 1.2.3 注册FeignClient
                registerFeignClient(registry, annotationMetadata, attributes);
            }
        }
    }
}
```

这里面的核心步骤大概就两步：获取扫描根包，包扫描获取所有 FeignClient 。那咱根据上面标注的关键位置来解析。

#### 1.2.1 basePackages的确定

注意 `@EnableFeignClients` 注解中的一个属性以及它的文档注释：

```java
@Import(FeignClientsRegistrar.class)
public @interface EnableFeignClients {
    // ......

    /**
     * List of classes annotated with @FeignClient. If not empty, disables classpath
     * scanning.
     * @return list of FeignClient classes
     */
    Class<?>[] clients() default {};
```

文档注释写的很清楚，如果 `clients` 不为空，则会禁用类路径的包扫描。如果没有指定 `clients` 属性，则会调用 `getBasePackages` 方法确定扫描根包：

```java
protected Set<String> getBasePackages(AnnotationMetadata importingClassMetadata) {
    Map<String, Object> attributes = importingClassMetadata
            .getAnnotationAttributes(EnableFeignClients.class.getCanonicalName());

    Set<String> basePackages = new HashSet<>();
    for (String pkg : (String[]) attributes.get("value")) {
        if (StringUtils.hasText(pkg)) {
            basePackages.add(pkg);
        }
    }
    for (String pkg : (String[]) attributes.get("basePackages")) {
        if (StringUtils.hasText(pkg)) {
            basePackages.add(pkg);
        }
    }
    for (Class<?> clazz : (Class[]) attributes.get("basePackageClasses")) {
        basePackages.add(ClassUtils.getPackageName(clazz));
    }

    if (basePackages.isEmpty()) {
        basePackages.add(
                ClassUtils.getPackageName(importingClassMetadata.getClassName()));
    }
    return basePackages;
}
```

逻辑很简单，它会去搜 `@EnableFeignClients` 注解中能代表扫描根包的所有属性，如果都没扫到，那就拿 `@EnableFeignClients` 注解标注的类所在的包为根包（跟 SpringBoot 的 `@SpringBootApplication` 思路一致）。

注意这里面没有涉及到 SpringBoot 中提到的 basePackages ，原因也很简单，`@EnableFeignClients` 注解是咱自己声明的，不是像 MyBatis 那样啥也不配置也能运行的，所以它就没有利用 SpringBoot 预先准备好的 basePackages 。

#### 1.2.2 registerClientConfiguration

```java
private void registerClientConfiguration(BeanDefinitionRegistry registry, Object name,
        Object configuration) {
    BeanDefinitionBuilder builder = BeanDefinitionBuilder
            .genericBeanDefinition(FeignClientSpecification.class);
    builder.addConstructorArgValue(name);
    builder.addConstructorArgValue(configuration);
    registry.registerBeanDefinition(
            name + "." + FeignClientSpecification.class.getSimpleName(),
            builder.getBeanDefinition());
}
```

方法实现真的与 Ribbon 中几乎一致，连类型名称都基本一致（`FeignClientSpecification` 与 `RibbonClientSpecification`），不过咱关心的不是具体的实现，咱应该意识到一点：会不会 Feign 的内部设计也跟 Ribbon 一样，内部套一个 Map 放子IOC容器？先不管咱的猜想是否正确，大胆猜就OK，下面自然会有验证。

#### 1.2.3 registerFeignClient

这个方法才是真正的创建 FeignClient 的实现类，方法实现中赋值操作那是真的多：

```java
private void registerFeignClient(BeanDefinitionRegistry registry,
        AnnotationMetadata annotationMetadata, Map<String, Object> attributes) {
    String className = annotationMetadata.getClassName();
    BeanDefinitionBuilder definition = BeanDefinitionBuilder
            .genericBeanDefinition(FeignClientFactoryBean.class);
    validate(attributes);
    definition.addPropertyValue("url", getUrl(attributes));
    definition.addPropertyValue("path", getPath(attributes));
    String name = getName(attributes);
    definition.addPropertyValue("name", name);
    String contextId = getContextId(attributes);
    definition.addPropertyValue("contextId", contextId);
    definition.addPropertyValue("type", className);
    definition.addPropertyValue("decode404", attributes.get("decode404"));
    definition.addPropertyValue("fallback", attributes.get("fallback"));
    definition.addPropertyValue("fallbackFactory", attributes.get("fallbackFactory"));
    definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);

    String alias = contextId + "FeignClient";
    AbstractBeanDefinition beanDefinition = definition.getBeanDefinition();

    boolean primary = (Boolean) attributes.get("primary"); // has a default, won't be
                                                            // null

    beanDefinition.setPrimary(primary);

    String qualifier = getQualifier(attributes);
    if (StringUtils.hasText(qualifier)) {
        alias = qualifier;
    }

    BeanDefinitionHolder holder = new BeanDefinitionHolder(beanDefinition, className,
            new String[] { alias });
    BeanDefinitionReaderUtils.registerBeanDefinition(holder, registry);
}
```

注意它创建的 Bean 的类型是 `FeignClientFactoryBean` ，工厂Bean！不是普通Bean？那这里头的设计估计不会太简单，咱放到后面再详细解释，这里咱先有个印象：**实际创建的 FeignClient 是借助一个工厂Bean - `FeignClientFactoryBean` 搞定的** 。

核心注解看完之后，别忘了还有自动配置类。根据惯例，直接搜索 `FeignAutoConfiguration` 那是必须得出的。

## 2. FeignAutoConfiguration

```java
@Configuration
@ConditionalOnClass(Feign.class)
@EnableConfigurationProperties({ FeignClientProperties.class,
		FeignHttpClientProperties.class })
public class FeignAutoConfiguration
```

发现这个自动配置类上没有标注其他相关的配置类，一向见惯了配置类导入和联动的我们岂能连这点敏锐的嗅觉都没有？马上找到 feign 包的 `spring.factories` 文件，果然不止一个：

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  org.springframework.cloud.openfeign.ribbon.FeignRibbonClientAutoConfiguration,\
  org.springframework.cloud.openfeign.FeignAutoConfiguration,\
  org.springframework.cloud.openfeign.encoding.FeignAcceptGzipEncodingAutoConfiguration,\
  org.springframework.cloud.openfeign.encoding.FeignContentGzipEncodingAutoConfiguration
```

下面两个都是与 gzip 相关的自动配置类，咱就不关心了，咱主要看前两个。

先认真看 `FeignAutoConfiguration` 吧，这个自动配置类上也没标别的关键内容，那咱直接看它注册的核心组件就可以。

### 2.1 FeignContext

```java
@Autowired(required = false)
private List<FeignClientSpecification> configurations = new ArrayList<>();

@Bean
public FeignContext feignContext() {
    FeignContext context = new FeignContext();
    context.setConfigurations(this.configurations);
    return context;
}
```

先不管 `FeignContext` 是什么，看这熟悉的结构和犀利的操作，是不是突然震惊了一下？咱对前面的大胆猜测更加坚定了，这个家伙99.9%内部藏着一个 Map 放子IOC容器！果不其然，这个 `FeignContext` 继承了 `NamedContextFactory` ：

```java
public class FeignContext extends NamedContextFactory<FeignClientSpecification>
```

得了，这一套熟悉的操作我上一集已经看过了，它的骚操作咱早已了然于胸。那它后续的内容小册就不再赘述了哈，前面看 Ribbon 的时候已经差不多都了解了。

### 2.2 Targeter

```java
@Bean
@ConditionalOnClass(name = "feign.hystrix.HystrixFeign")
@ConditionalOnMissingBean
public Targeter feignTargeter() {
    return new HystrixTargeter();
}

@Bean
@ConditionalOnMissingClass("feign.hystrix.HystrixFeign")
@ConditionalOnMissingBean
public Targeter feignTargeter() {
    return new DefaultTargeter();
}
```

注意这两个 `Targeter` 是不共存的，上面的 `HystrixTargeter` 带熔断降级，下面的 `DefaultTargeter` 不带。由于目前咱还没谈到熔断降级，故咱先看 `DefaultTargeter` 的核心实现（实际上创建的是 `HystrixTargeter` ，是因为 **Feign 会把 Hystrix 一并依赖进来**，不过这里先不看复杂的 `HystrixTargeter` ，先看一眼默认的简单实现，后续看 Feign 接口的动态实现类时再展开 `HystrixTargeter` ）：

```java
class DefaultTargeter implements Targeter {
    @Override
    public <T> T target(FeignClientFactoryBean factory, Feign.Builder feign,
            FeignContext context, Target.HardCodedTarget<T> target) {
        return feign.target(target);
    }
}
```

可以发现 `DefaultTargeter` 的源码简单的很！这里它直接拿 `Feign.Builder` 构造实际的 FeignClient 而已，至于内部的构造，咱现在先不点进去，后续咱遇到时再详细介绍（其实是太麻烦了怕吓到小伙伴们）。

除了上述两个组件之外，剩余的都是不符合 @Conditional 条件的组件了，咱就不多关心。直接来看下面的自动配置类。

## 3. FeignRibbonClientAutoConfiguration

```java
@Configuration
@ConditionalOnClass({ ILoadBalancer.class, Feign.class })
@AutoConfigureBefore(FeignAutoConfiguration.class)
@EnableConfigurationProperties({ FeignHttpClientProperties.class })
@Import({ HttpClientFeignLoadBalancedConfiguration.class,
		OkHttpFeignLoadBalancedConfiguration.class,
		DefaultFeignLoadBalancedConfiguration.class })
public class FeignRibbonClientAutoConfiguration
```

这次看到这么多注解，咱心里才安稳啊！类上面标注的注解最值得咱关注的就是 `DefaultFeignLoadBalancedConfiguration` ，咱先来看它。

### 3.1 DefaultFeignLoadBalancedConfiguration

这个配置类只注册了一个 `LoadBalancerFeignClient` ：

```java
@Configuration
class DefaultFeignLoadBalancedConfiguration {
    @Bean
    @ConditionalOnMissingBean
    public Client feignClient(CachingSpringLoadBalancerFactory cachingFactory,
            SpringClientFactory clientFactory) {
        return new LoadBalancerFeignClient(new Client.Default(null, null), cachingFactory,
                clientFactory);
    }
}
```

这个 `LoadBalancerFeignClient` 组件一看就能猜到它是真正发起基于 Feign 的负载均衡请求的客户端。它里面的功能咱也是先放一放，到下一章咱介绍 Feign 的整体调用流程时，咱会慢慢深入这里面看的。

除了这个配置类之外，其余的两个配置类都因为 classpath 下缺少指定的类而导致无法被装配，那咱就回过头看 `FeignRibbonClientAutoConfiguration` 注册的组件吧。

### 3.2 CachingSpringLoadBalancerFactory

```java
@Bean
@Primary
@ConditionalOnMissingBean
@ConditionalOnMissingClass("org.springframework.retry.support.RetryTemplate")
public CachingSpringLoadBalancerFactory cachingLBClientFactory(
        SpringClientFactory factory) {
    return new CachingSpringLoadBalancerFactory(factory);
}

@Bean
@Primary
@ConditionalOnMissingBean
@ConditionalOnClass(name = "org.springframework.retry.support.RetryTemplate")
public CachingSpringLoadBalancerFactory retryabeCachingLBClientFactory(
        SpringClientFactory factory, LoadBalancedRetryFactory retryFactory) {
    return new CachingSpringLoadBalancerFactory(factory, retryFactory);
}
```

跟上面的 `Targeter` 类似，这两个Bean也是不共存，不过好在都是 `CachingSpringLoadBalancerFactory` 类型的，只是一个只放入了 `SpringClientFactory` ，另一个还额外放入了 `LoadBalancedRetryFactory` 。那咱主要看的还是主线调用流程，关注普通的 `CachingSpringLoadBalancerFactory` 就够了。

先看一眼它的内部结构设计：

```java
public class CachingSpringLoadBalancerFactory {
	protected final SpringClientFactory factory;
	protected LoadBalancedRetryFactory loadBalancedRetryFactory = null;
	private volatile Map<String, FeignLoadBalancer> cache = new ConcurrentReferenceHashMap<>();
```

它内部会注入一个 `SpringClientFactory` ，而且下面还有一个 Map 放的是 `FeignLoadBalancer` ，想都不用想肯定 key 是服务名称。先看一眼它内部的方法吧，它只有一个 `create` 方法：

```java
public FeignLoadBalancer create(String clientName) {
    FeignLoadBalancer client = this.cache.get(clientName);
    if (client != null) {
        return client;
    }
    IClientConfig config = this.factory.getClientConfig(clientName);
    ILoadBalancer lb = this.factory.getLoadBalancer(clientName);
    ServerIntrospector serverIntrospector = this.factory.getInstance(clientName,
            ServerIntrospector.class);
    client = this.loadBalancedRetryFactory != null
            ? new RetryableFeignLoadBalancer(lb, config, serverIntrospector,
                    this.loadBalancedRetryFactory)
            : new FeignLoadBalancer(lb, config, serverIntrospector);
    this.cache.put(clientName, client);
    return client;
}
```

这个套路好像又在 Ribbon 里见过！小伙伴们如果还记得 `LoadBalancerInterceptor` 的话（第12章的第2节），那里面的几个动作跟这里有几分相似。咱关注下这里面的核心步骤。首先它拿到原始的 Ribbon 的负载均衡器、以及Ribbon 中的负载均衡配置，封装为一个 `FeignLoadBalancer` ，相当于把原本 Ribbon 的组件们进行了一次包装，最后放入 Map 缓存中完成整体创建流程。至于创建出来的 `FeignLoadBalancer` 具体都要干什么事情，咱放到后面 Feign 的运行全过程时再好好研究。

### 3.3 Request.Options

```java
@Bean
@ConditionalOnMissingBean
public Request.Options feignRequestOptions() {
    return LoadBalancerFeignClient.DEFAULT_OPTIONS;
}
```

`Options` 是配置的意思，它里面维护了3个变量：

```java
public static class Options {
    private final int connectTimeoutMillis; // 建立连接的超时时间
    private final int readTimeoutMillis; // 完整响应的超时时间
    private final boolean followRedirects; // 是否开启重定向
```

默认情况下，`Options` 的设置就是调空参构造方法：

```java
public Options() {
    this(10 * 1000, 60 * 1000);
}

public Options(int connectTimeoutMillis, int readTimeoutMillis) {
    this(connectTimeoutMillis, readTimeoutMillis, true);
}
```

可以发现是 10 秒的建立连接超时，60 秒的完整响应超时。这里解释下完整响应的概念：服务调用时，从客户端发起请求到建立连接，再到服务响应回数据并由客户端完整接收，这整个过程的总耗时就是完整响应时间。

#### 3.3.1 自定义超时配置

如果默认的配置不能满足开发需求或者实际线上需求，可以手动创建 Options 类来覆盖原有配置。`Request.Options` 的Bean创建处有 `@ConditionalOnMissingBean` 注解，意味着如果自定义了 Options 配置，则自动配置中的Bean不会被创建。如：

```java
@Bean
public Request.Options options() {
    return new Request.Options(2000, 10000);
}
```

## 小结

1. `@EnableFeignClients` 中导入了一个 `FeignClientsRegistrar` ，它负责扫描所有 `@FeignClient` 标注的接口；
2. `FeignAutoConfiguration` 自动配置类中创建的 `FeignContext` 与 Ribbon 的 `SpringClientFactory` 套路几乎一致；
3. `CachingSpringLoadBalancerFactory` 中缓存了每个服务对应的 `FeignLoadBalancer` ，它是真正进行服务调用的执行器。

![](./img/2023/05/cloud14-6.png)



 

