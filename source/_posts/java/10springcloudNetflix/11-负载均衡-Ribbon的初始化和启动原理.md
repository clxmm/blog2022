---
title: 11-负载均衡-Ribbon的初始化和启动原理.md
toc: true
tags: springcloud
categories: 
    - [java]
    - [springcloudNetflix]
---

用过 Eureka 的小伙伴都了解，Eureka 一般都会捆绑 Ribbon 一起带到项目中，因为多服务实例下的调用需要均分到每个服务实例上，而不是可着一台机器用到底。接下来的这几篇咱来看 Ribbon 中的几个核心原理。

<!--more-->

## 0. 测试环境搭建

咱以下图的方式搭建一个最简单的负载均衡场景，使用 eureka-consumer 调用两个 eureka-client ：

![](./img/2023/05/cloud11-1.png)

好了，下面进入正题。既然引入 eureka-client 的启动器会一起带入 Ribbon ，那一定能找到 Ribbon 相应的启动类。从引入的 jar 包中可以找到两个相关的自动配置类：`spring-cloud-netflix-ribbon` 中的 `RibbonAutoConfiguration` 、`spring-cloud-netflix-eureka-client` 中的 `RibbonEurekaAutoConfiguration` 。下面咱分别来看。

## 1. RibbonAutoConfiguration

跟往常一样，先翻自动配置类的类声明：

```java
@Configuration
@Conditional(RibbonAutoConfiguration.RibbonClassesConditions.class)
@RibbonClients
@AutoConfigureAfter(name = "org.springframework.cloud.netflix.eureka.EurekaClientAutoConfiguration")
@AutoConfigureBefore({ LoadBalancerAutoConfiguration.class,
		AsyncLoadBalancerAutoConfiguration.class })
@EnableConfigurationProperties({ RibbonEagerLoadProperties.class,
		ServerIntrospectorProperties.class })
public class RibbonAutoConfiguration
```

重点提取几个信息：

### 1.1 RibbonClassesConditions的条件判断

```java
static class RibbonClassesConditions extends AllNestedConditions {

    RibbonClassesConditions() {
        super(ConfigurationPhase.PARSE_CONFIGURATION);
    }

    @ConditionalOnClass(IClient.class)
    static class IClientPresent { }

    @ConditionalOnClass(RestTemplate.class)
    static class RestTemplatePresent { }

    @ConditionalOnClass(AsyncRestTemplate.class)
    static class AsyncRestTemplatePresent { }

    @ConditionalOnClass(Ribbon.class)
    static class RibbonPresent { }

}
```

这种类型的声明可能咱看起来会非常懵，这是什么操作？其实这是 SpringBoot 用于扩展多条件判断的复合写法。当同时需要声明多个条件时，可以声明一个类继承 `AllNestedConditions` ，并在类中声明若干个内部类，并在这些内部类上声明具体的判断条件即可，只有这些内部类上声明的所有条件都通过，整体条件判断才会通过。

### 1.2 @RibbonClients

```java
@Import(RibbonClientConfigurationRegistrar.class)
public @interface RibbonClients

```

这个注解有点似曾相识啊，它内部组合的 `@RibbonClient` 注解咱在自定义负载均衡策略里用过它。注意这个 `@RibbonClients` 注解中还导入了一个 `RibbonClientConfigurationRegistrar` ，它的类型是 `ImportBeanDefinitionRegistrar` 咱都比较熟悉了，直接来看源码吧。

```java
public class RibbonClientConfigurationRegistrar implements ImportBeanDefinitionRegistrar {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
        Map<String, Object> attrs = metadata.getAnnotationAttributes(RibbonClients.class.getName(), true);
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
        Map<String, Object> client = metadata.getAnnotationAttributes(RibbonClient.class.getName(), true);
        String name = getClientName(client);
        if (name != null) {
            registerClientConfiguration(registry, name, client.get("configuration"));
        }
    }

    // ......

    private void registerClientConfiguration(BeanDefinitionRegistry registry, Object name,
            Object configuration) {
        BeanDefinitionBuilder builder = BeanDefinitionBuilder
                .genericBeanDefinition(RibbonClientSpecification.class);
        builder.addConstructorArgValue(name);
        builder.addConstructorArgValue(configuration);
        registry.registerBeanDefinition(name + ".RibbonClientSpecification",
                builder.getBeanDefinition());
    }
}
```

通篇看来，它完成的事情是将 `@RibbonClient` 注解中定义的信息取出，并封装为一个个的 `RibbonClientSpecification` 类型的Bean。这个 `RibbonClientConfigurationRegistrar` 的真正作用咱后面第14章还会提到，小伙伴们可以先记住它。

除了这两个注解以外，剩余的就是这个类中提到的其他自动配置类了，这些内容咱放到下面解释。再看看这个自动配置类中都声明了哪些核心组件。

### 1.3 SpringClientFactory

```java
@Bean
public SpringClientFactory springClientFactory() {
    SpringClientFactory factory = new SpringClientFactory();
    factory.setConfigurations(this.configurations);
    return factory;
}
```

这个 `SpringClientFactory` 看上去像是一个与基础配置有关的组件，它的文档注释原文翻译：

> A factory that creates client, load balancer and client configuration instances. It creates a Spring ApplicationContext per client name, and extracts the beans that it needs from there.
>
> 创建客户端，负载均衡器和客户端配置实例的工厂。它为每个客户端名称创建一个 Spring `ApplicationContext` ，并从那里提取所需的 Bean。

文档注释写得还算明白，它是创建 负载均衡器 和 客户端配置 的工厂，注意在创建 `SpringClientFactory` 的时候它把当前的一组配置传入进去了，可以大概猜测这个 `SpringClientFactory` 只是个代工厂而已，还需要借助它去创建别的组件，下面咱就能看到一部分内容。

留意下这个 `configurations` ，Debug发现它包含两个自动配置类，后续它还会起作用：]

（这个组件中使用的设计比较特殊，第14章咱在解析原理时会着重提它）

![](./img/2023/05/cloud11-2.png)

### 1.4 LoadBalancerClient

```java
@Bean
@ConditionalOnMissingBean(LoadBalancerClient.class)
public LoadBalancerClient loadBalancerClient() {
    return new RibbonLoadBalancerClient(springClientFactory());
}
```

果然它直接拿 `SpringClientFactory` 传入进去，创建了一个类型为 `RibbonLoadBalancerClient` 的负载均衡器。注意这个 `RibbonLoadBalancerClient` 的类定义：

```java
public class RibbonLoadBalancerClient implements LoadBalancerClient

public interface LoadBalancerClient extends ServiceInstanceChooser

```

相当于 `RibbonLoadBalancerClient` 实现了两个接口，这两个接口中有几个核心方法：（文档注释已标注在源码中）

```java
public interface ServiceInstanceChooser {
    // 从LoadBalancer中为指定服务选择一个ServiceInstance
    ServiceInstance choose(String serviceId);
}

public interface LoadBalancerClient extends ServiceInstanceChooser {
    // 使用LoadBalancer中的ServiceInstance对指定服务执行请求。
    <T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException;
    
    // 使用指定的ServiceInstance对指定服务执行请求。
    <T> T execute(String serviceId, ServiceInstance serviceInstance,
            LoadBalancerRequest<T> request) throws IOException;
    
    // 创建具有真实主机和端口的适当URI，以供系统使用（某些系统使用带有逻辑服务名称的URI作为主机）
    URI reconstructURI(ServiceInstance instance, URI original);
}
```

从这几个方法中大概可以发现这个组件的重要性：`choose` 方法负责找服务，`execute` 负责执行请求。这里咱先看一眼 `choose` 方法的逻辑，`execute` 部分咱后面遇到了再解释。

#### 1.4.1 ServiceInstanceChooser#choose

```java
public ServiceInstance choose(String serviceId) {
    return choose(serviceId, null);
}

public ServiceInstance choose(String serviceId, Object hint) {
    Server server = getServer(getLoadBalancer(serviceId), hint);
    if (server == null) {
        return null;
    }
    return new RibbonServer(serviceId, server, isSecure(server, serviceId),
            serverIntrospector(serviceId).getMetadata(server));
}

protected ILoadBalancer getLoadBalancer(String serviceId) {
    return this.clientFactory.getLoadBalancer(serviceId);
}

protected Server getServer(ILoadBalancer loadBalancer, Object hint) {
    if (loadBalancer == null) {
        return null;
    }
    // Use 'default' on a null hint, or just pass it on?
    return loadBalancer.chooseServer(hint != null ? hint : "default");
}
```

从上面一路调用下来，最终到了最底下，它要拿负载均衡器去根据一个特殊的 `hint` 值去找服务，又由于上面一开始指定的是 null ，所以相当于直接拿 "default" 值去找，往下找发现走到 `ILoadBalancer` 接口了，根据上面的 `getLoadBalancer` 方法可以获取到对应的 `ILoadBalancer` ，那咱就继续往里深入。

#### 1.4.2 SpringClientFactory#getLoadBalancer

```java
public ILoadBalancer getLoadBalancer(String name) {
    return getInstance(name, ILoadBalancer.class);
}

public <C> C getInstance(String name, Class<C> type) {
    C instance = super.getInstance(name, type);
    if (instance != null) {
        return instance;
    }
    IClientConfig config = getInstance(name, IClientConfig.class);
    return instantiateWithConfig(getContext(name), type, config);
}

public <T> T getInstance(String name, Class<T> type) {
    AnnotationConfigApplicationContext context = getContext(name);
    if (BeanFactoryUtils.beanNamesForTypeIncludingAncestors(context,
            type).length > 0) {
        return context.getBean(type);
    }
    return null;
}
```

从上往下一直调用到 `NamedContextFactory` 的 `getInstance` 方法，终于发现了底层的获取方式，它是直接拿 IOC 容器去取类型为 `ILoadBalancer` 的负载均衡器，直接返回。通过Debug，可以发现可以获取到默认的负载均衡器是 `ZoneAwareLoadBalancer` ：

至于这个 `ZoneAwareLoadBalancer` 负载均衡器是怎么被注册进 IOC 容器的，咱后面再看。

回到上面：

```java
protected Server getServer(ILoadBalancer loadBalancer, Object hint) {
    // ......
    return loadBalancer.chooseServer(hint != null ? hint : "default");
}
```

拿到负载均衡器后就可以调用 `chooseServer` 方法进行实际的负载均衡选择服务了，这部分涉及到负载均衡策略，咱后面单开一章专门介绍负载均衡的策略，这里不再展开。

### 1.5 PropertiesFactory

```java
@Bean
@ConditionalOnMissingBean
public PropertiesFactory propertiesFactory() {
    return new PropertiesFactory();
}
```

从名字上也能看得出来它是一个配置工厂，那它都存了些什么配置呢？咱直接跳到它的源码中去看。

#### 1.5.1 PropertiesFactory的初始化

```java
public class PropertiesFactory {

	@Autowired
	private Environment environment;

	private Map<Class, String> classToProperty = new HashMap<>();

	public PropertiesFactory() {
		classToProperty.put(ILoadBalancer.class, "NFLoadBalancerClassName");
		classToProperty.put(IPing.class, "NFLoadBalancerPingClassName");
		classToProperty.put(IRule.class, "NFLoadBalancerRuleClassName");
		classToProperty.put(ServerList.class, "NIWSServerListClassName");
		classToProperty.put(ServerListFilter.class, "NIWSServerListFilterClassName");
	}
```

可以发现它预先存放了几个特定的名称，但是这些名称看上去都没有什么规律，那咱就看看这个配置工厂还有什么别的方法可以找出一点蛛丝马迹。

```java
static final String NAMESPACE = "ribbon";

public <C> C get(Class<C> clazz, IClientConfig config, String name) {
    String className = getClassName(clazz, name);
    if (StringUtils.hasText(className)) {
        try {
            Class<?> toInstantiate = Class.forName(className);
            return (C) SpringClientFactory.instantiateWithConfig(toInstantiate, config);
        } // catch ......
    }
    return null;
}

public String getClassName(Class clazz, String name) {
    if (this.classToProperty.containsKey(clazz)) {
        String classNameProperty = this.classToProperty.get(clazz);
        String className = environment.getProperty(name + "." + NAMESPACE + "." + classNameProperty);
        return className;
    }
    return null;
}
```

注意在 `getClassName` 方法中有对 `classToProperty` 的利用，它是想在 Spring 的配置中获取一个格式为 `[微服务名称].ribbon.[类名对应的特定值]` 的配置，且返回一个全限定类名。借助IDEA查找 `get` 方法的使用位置，刚好发现了7处，且恰好与上面的 `PropertyFactory` 构造方法中的初始化内容一致：

![](./img/2023/05/cloud11-3.png)

由此咱应该能意识到一件事情：如果咱能自己配置一些特殊的规则，是不是可以对一些服务做个性化配置？答案是肯定的，下面咱实际测试一下效果。

#### 1.5.2 测试定制化负载均衡策略

在 eureka-consumer 的全局配置文件 `application.properties` 中加入如下配置：

```properties
eureka-client.ribbon.NFLoadBalancerRuleClassName=com.netflix.loadbalancer.RandomRule

```

启动 eureka-server 、两个 eureka-client 、eureka-consumer ，浏览器中多次发送请求测试效果，发现返回的结果没有规律，说明当前的负载均衡策略已经不再是轮询策略，负载均衡策略定制成功。

由此同理，其他的配置也可以如法炮制，小册不多演示，感兴趣的小伙伴可以自己动手试一试。

### 1.6 RibbonApplicationContextInitializer

```java
@Bean
@ConditionalOnProperty("ribbon.eager-load.enabled")
public RibbonApplicationContextInitializer ribbonApplicationContextInitializer() {
    return new RibbonApplicationContextInitializer(springClientFactory(),
            ribbonEagerLoadProperties.getClients());
}
```

这个组件看上去好陌生，但注意上面标注的属性：`ribbon.eager-load.enabled` ，表面上理解为“开启迫切加载”，这个概念似乎容易让咱联想到单例模式中的饿汉单例，这个确实得介绍下了。

#### 1.6.1 【扩展】Ribbon调用客户端的迫切初始化

Ribbon 在进行客户端负载均衡调用时，采用的都是延迟加载机制：第一次调用服务时才会创建负载均衡客户端。这种情况可能会出现一种现象：负载均衡客户端的创建过程耗时比较长，但对应的服务超时时间设置的又很短，会导致**第一次调用出错，但后续正常**。

解决该问题的方案：将这些创建过程耗时较长的服务设置为迫切初始化加载。在 `application.properties` 中开启如下配置：

```properties
ribbon.eager-load.enabled=true
ribbon.eager-load.clients=eureka-client
```

设置后，应用启动时就会将声明的服务对应的负载均衡客户端创建好，就不会出现这种问题了。

#### 1.6.2 迫切初始化的原理

上面创建的 `RibbonApplicationContextInitializer` ，本质是一个监听器：

```java
public class RibbonApplicationContextInitializer implements ApplicationListener<ApplicationReadyEvent>

```

它监听 `ApplicationReadyEvent` 事件，刚好是应用都启动好，准备完成后触发的。咱看监听器的核心方法：

```java
public void onApplicationEvent(ApplicationReadyEvent event) {
    initialize();
}

protected void initialize() {
    if (clientNames != null) {
        for (String clientName : clientNames) {
            this.springClientFactory.getContext(clientName);
        }
    }
}
```

可以发现它将设置好的那些需要被迫切初始化的服务都取一次，由于第一次取会进行初始化，就达到了迫切初始化的目的。

注意开启、配置迫切初始化会降低应用启动速度，实际开发时可以根据服务的特性按需开启。

### 1.7 RestTemplateCustomizer

```java
@Bean
public RestTemplateCustomizer restTemplateCustomizer(
        final RibbonClientHttpRequestFactory ribbonClientHttpRequestFactory) {
    return restTemplate -> restTemplate
            .setRequestFactory(ribbonClientHttpRequestFactory);
}
```

Customizer 这个咱看到可太熟悉了啊，它可以编程式的对一些组件做定制化/配置，咱主要看它内部干了什么。源码中只有一句话，它将一个传入的 `RibbonClientHttpRequestFactory` 放入了 `RestTemplate` 中。而这个 `RibbonClientHttpRequestFactory` 就在下面的配置中。

### 1.8 RibbonClientHttpRequestFactory

```java
@Bean
public RibbonClientHttpRequestFactory ribbonClientHttpRequestFactory() {
    return new RibbonClientHttpRequestFactory(this.springClientFactory);
}
```

初始化的逻辑非常简单，咱主要关心它配合 `RestTemplate` 的作用。它内部只定义了一个方法：

```java
public ClientHttpRequest createRequest(URI originalUri, HttpMethod httpMethod) throws IOException {
    String serviceId = originalUri.getHost();
    if (serviceId == null) {
        throw new IOException("Invalid hostname in the URI [" + originalUri.toASCIIString() + "]");
    }
    IClientConfig clientConfig = this.clientFactory.getClientConfig(serviceId);
    RestClient client = this.clientFactory.getClient(serviceId, RestClient.class);
    HttpRequest.Verb verb = HttpRequest.Verb.valueOf(httpMethod.name());

    return new RibbonHttpRequest(originalUri, verb, client, clientConfig);
}
```

从逻辑中大概可以看出来，它会根据请求的路径，提取出服务名称（使用 Ribbon 负载均衡时，发送的url将不再是具体的域名/ip地址，而是微服务名称），在下面构建出 `ClientHttpRequest` ，这个东西咱可能没有概念，它是如何对 `RestTemplate` 产生影响呢？下面咱解释下 `ClientHttpRequest` 的实际用途。

#### 1.8.1 【扩展】RestTemplate的内部调用机制

咱在操作 `RestTemplate` 时常用的几种调用方式：`getForObject` 、`getForEntity` 、`postForObject` 、`postForEntity` ，这些方法的底层都是调的 RestTemplate 内部的 `execute` 方法：

```java
public <T> T execute(String url, HttpMethod method, @Nullable RequestCallback requestCallback,
        @Nullable ResponseExtractor<T> responseExtractor, Map<String, ?> uriVariables)
        throws RestClientException {
    URI expanded = getUriTemplateHandler().expand(url, uriVariables);
    return doExecute(expanded, method, requestCallback, responseExtractor);
}
```

继续往下调 `doExecute` 方法：

```java
protected <T> T doExecute(URI url, @Nullable HttpMethod method, @Nullable RequestCallback requestCallback,
        @Nullable ResponseExtractor<T> responseExtractor) throws RestClientException {
    // assert ......
    ClientHttpResponse response = null;
    try {
        ClientHttpRequest request = createRequest(url, method);
        if (requestCallback != null) {
            requestCallback.doWithRequest(request);
        }
        response = request.execute();
        handleResponse(url, method, response);
        return (responseExtractor != null ? responseExtractor.extractData(response) : null);
    } // catch finally ......
}
```

注意 try 块的第一行，它会调用 `createRequest` 方法来创建一个类型为 `ClientHttpRequest` 的请求对象，而这个 `createRequest` 方法的实现：

```java
private ClientHttpRequestFactory requestFactory = new SimpleClientHttpRequestFactory();

protected ClientHttpRequest createRequest(URI url, HttpMethod method) throws IOException {
    ClientHttpRequest request = getRequestFactory().createRequest(url, method);
    // logger
    return request;
}
```

它会拿 `RequestFactory` 来进行创建，默认情况下它使用的 `RequestFactory` 类型为 `SimpleClientHttpRequestFactory` ，它使用最基本的 `HttpURLConnection` 来进行请求创建。

#### 1.8.2 RibbonClientHttpRequestFactory的应用位置

在上面的 `RestTemplateCustomizer` 中还记得那句操作吗？它给 `RestTemplate` 中设置了 `RequestFactory` 为 `RibbonClientHttpRequestFactory` 类型，其实就是做了一次请求工厂的替换，这样创建出来的请求就是被负载均衡处理过的请求，也就相当于实现了客户端调用的负载均衡。

```java
restTemplate -> restTemplate.setRequestFactory(ribbonClientHttpRequestFactory)
```

到这里，`RibbonAutoConfiguration` 的所有核心组件就全部介绍完毕了，接下来咱看看 `RibbonAutoConfiguration` 提到的其他自动配置类的作用。

## 2. RibbonEurekaAutoConfiguration

在 `RibbonAutoConfiguration` 中声明了执行顺序，`RibbonEurekaAutoConfiguration` 要等到 `RibbonAutoConfiguration` 执行完成后才可以执行。而这个 `RibbonEurekaAutoConfiguration` 那是相当的干净：

```java
@Configuration
@EnableConfigurationProperties
@ConditionalOnRibbonAndEurekaEnabled
@AutoConfigureAfter(RibbonAutoConfiguration.class)
@RibbonClients(defaultConfiguration = EurekaRibbonClientConfiguration.class)
public class RibbonEurekaAutoConfiguration {

}
```

刨去判断条件和开启配置等内容，那就只剩下一个可以关注的部分：`@RibbonClients(defaultConfiguration = EurekaRibbonClientConfiguration.class)` ，它声明了默认的配置就是 Eureka 与 Ribbon 整合的默认配置规则 `EurekaRibbonClientConfiguration` 。这个配置类中又注册了几个Bean，咱一一来看。

### 2.1 NIWSDiscoveryPing

```java
@Bean
@ConditionalOnMissingBean
public IPing ribbonPing(IClientConfig config) {
    if (this.propertiesFactory.isSet(IPing.class, serviceId)) {
        return this.propertiesFactory.get(IPing.class, config, serviceId);
    }
    NIWSDiscoveryPing ping = new NIWSDiscoveryPing();
    ping.initWithNiwsConfig(config);
    return ping;
}
```

看这个类型，是不是很容易想起来 ping 命令？对了就是心跳的那些东西。Ribbon 在实现负载均衡时，有一个比较重要的环节就是判断服务是否可用，不可用的服务不会进行负载均衡调用。在自动装配中咱看到它注册的 ping 测试类型为 `NIWSDiscoveryPing` ，它的实现逻辑：

```java
public boolean isAlive(Server server) {
    boolean isAlive = true;
    if (server != null && server instanceof DiscoveryEnabledServer) {
        DiscoveryEnabledServer dServer = (DiscoveryEnabledServer) server;                
        InstanceInfo instanceInfo = dServer.getInstanceInfo();
        if (instanceInfo != null) {                    
            InstanceStatus status = instanceInfo.getStatus();
            if (status != null) {
                isAlive = status.equals(InstanceStatus.UP);
            }
        }
    }
    return isAlive;
}
```

可以发现它拿 EurekaServer 中的 `InstanceInfo` 信息来判断是否是正常上线运行的状态，换言之，`NIWSDiscoveryPing` 的 ping 依据来源是 EurekaServer 的注册信息。除了 `NIWSDiscoveryPing` 之外，还有几种预设的 ping 策略：

- `DummyPing` ：直接返回 true ，内部无其他逻辑
- `NoOpPing` ：直接返回 true ，内部无其他逻辑
- `PingConstant` ：返回预先设定的值，它内部维护了一个 `constant` 值，每次固定返回该值

具体的 ping 动作触发，这个咱到后面看到时再解释。

### 2.2 DomainExtractingServerList

```java
@Bean
@ConditionalOnMissingBean
public ServerList<?> ribbonServerList(IClientConfig config,
        Provider<EurekaClient> eurekaClientProvider) {
    if (this.propertiesFactory.isSet(ServerList.class, serviceId)) {
        return this.propertiesFactory.get(ServerList.class, config, serviceId);
    }
    DiscoveryEnabledNIWSServerList discoveryServerList = new DiscoveryEnabledNIWSServerList(
            config, eurekaClientProvider);
    DomainExtractingServerList serverList = new DomainExtractingServerList(
            discoveryServerList, config, this.approximateZoneFromHostname);
    return serverList;
}
```

暂且不看 `DomainExtractingServerList` 的设计，它返回的类型是 `ServerList` ，这是一个接口，它里面定义了两个方法：

```java
public interface ServerList<T extends Server> {
    public List<T> getInitialListOfServers(); // 获取初始化的服务实例列表
    public List<T> getUpdatedListOfServers(); // 获取更新的服务实例列表
}

```

看到这两个方法，大概已经可以猜测出来，Ribbon 是通过 `ServerList` 获取服务列表，从而进行后续的负载均衡逻辑。既然前面看了 `NIWSDiscoveryPing` 的套路，大胆猜测 `DomainExtractingServerList` 也是借助 Eureka 的服务实例列表来进行获取。进入到 `DomainExtractingServerList` 的内部实现中：

```java
private ServerList<DiscoveryEnabledServer> list;

public List<DiscoveryEnabledServer> getInitialListOfServers() {
    List<DiscoveryEnabledServer> servers = setZones(this.list.getInitialListOfServers());
    return servers;
}

public List<DiscoveryEnabledServer> getUpdatedListOfServers() {
    List<DiscoveryEnabledServer> servers = setZones(this.list.getUpdatedListOfServers());
    return servers;
}
```

发现它并不是自己进行获取操作，而是组合了另一个上面源码中看到的 `DiscoveryEnabledNIWSServerList` 来实际获取（有点装饰者的味道了）。

进入到被装饰的 `DiscoveryEnabledNIWSServerList` 的内部实现中，发现这里面两个接口的方法都调到同一个方法去了：（核心注释已标注在源码中）

```java
public List<DiscoveryEnabledServer> getInitialListOfServers(){
    return obtainServersViaDiscovery();
}
public List<DiscoveryEnabledServer> getUpdatedListOfServers(){
    return obtainServersViaDiscovery();
}

private List<DiscoveryEnabledServer> obtainServersViaDiscovery() {
    List<DiscoveryEnabledServer> serverList = new ArrayList<>();

    if (eurekaClientProvider == null || eurekaClientProvider.get() == null) {
        logger.warn("EurekaClient has not been initialized yet, returning an empty list");
        return new ArrayList<DiscoveryEnabledServer>();
    }

    // 借助EurekaClient获取服务列表
    EurekaClient eurekaClient = eurekaClientProvider.get();
    if (vipAddresses != null) {
        for (String vipAddress : vipAddresses.split(",")) {
            // if targetRegion is null, it will be interpreted as the same region of client
            List<InstanceInfo> listOfInstanceInfo = eurekaClient
                    .getInstancesByVipAddress(vipAddress, isSecure, targetRegion);
            // 获取到服务列表后进行判断、过滤、封装
            for (InstanceInfo ii : listOfInstanceInfo) {
                // 只取状态为UP的服务实例
                if (ii.getStatus().equals(InstanceStatus.UP)) {
                    if (shouldUseOverridePort) {
                        // logger
                        InstanceInfo copy = new InstanceInfo(ii);
                        if (isSecure) {
                            ii = new InstanceInfo.Builder(copy).setSecurePort(overridePort).build();
                        } else {
                            ii = new InstanceInfo.Builder(copy).setPort(overridePort).build();
                        }
                    }

                    // 构建可以调用的服务实例
                    DiscoveryEnabledServer des = createServer(ii, isSecure, shouldUseIpAddr);
                    serverList.add(des);
                }
            }
            if (serverList.size() > 0 && prioritizeVipAddressBasedServers) {
                break; // if the current vipAddress has servers, we dont use subsequent vipAddress based servers
            }
        }
    }
    return serverList;
}
```

上面的逻辑大概可以看得出来分为几个步骤：获取、筛选、封装，最后返回。至于这个 `ServerList` 是如何被使用，咱也是后面看到再解释。

### 2.3 ServerIntrospector

```java
@Bean
public ServerIntrospector serverIntrospector() {
    return new EurekaServerIntrospector();
}
```

**Introspector** 这个概念可能有些小伙伴们会比较陌生，在 JavaSE 中有一个概念叫**内省**，它可以简单理解为针对 Bean 的属性、方法进行的获取和操作。听起来有点反射的意思，其实内省就是借助反射进行一些比反射更灵活的操作。那么根据这个思想，`ServerIntrospector` 的用途也就大概可以猜想：它可以**根据一个服务实例的ID或者name，获取服务实例的一些元信息或者状态信息**。果不其然，这个接口定义的方法还就是 `getMetadata` ：

```java
// EurekaServerIntrospector的实现
public Map<String, String> getMetadata(Server server) {
    if (server instanceof DiscoveryEnabledServer) {
        DiscoveryEnabledServer discoveryServer = (DiscoveryEnabledServer) server;
        return discoveryServer.getInstanceInfo().getMetadata();
    }
    return super.getMetadata(server);
}
```

可见这里面拿到服务实例的 `InstanceInfo` ，并以元信息的形式返回出去。

如果小伙伴们还有一点印象的话，在1.4.1节的源码中咱其实见过它：

```java
    return new RibbonServer(serviceId, server, isSecure(server, serviceId),
            serverIntrospector(serviceId).getMetadata(server));
```

这里在构建 `RibbonServer` 时把服务实例的元信息也注入进去了，获取服务实例云信息的途径就是走 `ServerIntrospector` 。

后续可能还会遇到 `ServerIntrospector` ，咱到时候看到时再多加留意。

## 3. LoadBalancerAutoConfiguration

`LoadBalancerAutoConfiguration` 的执行时机比 `RibbonAutoConfiguration` 早，它主要完成的内容是负载均衡的一些基础组件，咱依此来看（默认情况下不存在 retry 相关的组件，小册不作介绍）。

### 3.1 SmartInitializingSingleton

```java
@Bean
public SmartInitializingSingleton loadBalancedRestTemplateInitializerDeprecated(
        final ObjectProvider<List<RestTemplateCustomizer>> restTemplateCustomizers) {
    return () -> restTemplateCustomizers.ifAvailable(customizers -> {
        for (RestTemplate restTemplate : LoadBalancerAutoConfiguration.this.restTemplates) {
            for (RestTemplateCustomizer customizer : customizers) {
                customizer.customize(restTemplate);
            }
        }
    });
}
```

这部分逻辑就很简单了，它会给每个 `RestTemplate` 都执行一次定制器的处理，定制器的内容咱上面1.7、1.8节有看到，不再赘述。

### 3.2 LoadBalancerRequestFactory

```java
@Bean
@ConditionalOnMissingBean
public LoadBalancerRequestFactory loadBalancerRequestFactory(LoadBalancerClient loadBalancerClient) {
    return new LoadBalancerRequestFactory(loadBalancerClient, this.transformers);
}
```

看类名也能看懂它的用途，它是用来创建“带有负载均衡的请求”的工厂，注意虽然它的命名与前面1.8节看到的 `RibbonClientHttpRequestFactory` 有点类似，但它并没有实现任何接口。它的真正作用是配合下面的 `LoadBalancerInterceptor` 进行实际的负载均衡。

### 3.3 LoadBalancerInterceptor

```java
@Bean
public LoadBalancerInterceptor ribbonInterceptor(LoadBalancerClient loadBalancerClient,
        LoadBalancerRequestFactory requestFactory) {
    return new LoadBalancerInterceptor(loadBalancerClient, requestFactory);
}
```

看到拦截器，想必小伙伴已经能大概猜到 Ribbon 的负载均衡思路了吧，就是在发请求之前用拦截器先截住，替换 url 后再发请求。它组合了 `LoadBalancerClient` 与 `LoadBalancerRequestFactory` ：

```java
public class LoadBalancerInterceptor implements ClientHttpRequestInterceptor {
	private LoadBalancerClient loadBalancer;
	private LoadBalancerRequestFactory requestFactory;
```

作为拦截器，核心方法一定是 `intercept` ：

```java
public ClientHttpResponse intercept(final HttpRequest request, final byte[] body,
        final ClientHttpRequestExecution execution) throws IOException {
    final URI originalUri = request.getURI();
    String serviceName = originalUri.getHost();
    Assert.state(serviceName != null, "Request URI does not contain a valid hostname: " + originalUri);
    return this.loadBalancer.execute(serviceName, this.requestFactory.createRequest(request, body, execution));
}
```

逻辑很简单，它会拿到当前请求的 url 和服务名称，之后交给 `loadBalancer` 进行实际处理，这个 `loadBalancer` 就是上面1.4节咱看到的 `RibbonLoadBalancerClient` 。具体拦截器与负载均衡器是如何配合工作运行的，咱放到下一章一起研究。

### 3.4 RestTemplateCustomizer

```java
@Bean
@ConditionalOnMissingBean
public RestTemplateCustomizer restTemplateCustomizer(
        final LoadBalancerInterceptor loadBalancerInterceptor) {
    return restTemplate -> {
        List<ClientHttpRequestInterceptor> list = new ArrayList<>(
                restTemplate.getInterceptors());
        list.add(loadBalancerInterceptor);
        restTemplate.setInterceptors(list);
    };
}
```

又是一个定制器，这里面的定制规则是将上面的 `LoadBalancerInterceptor` 放入 `RestTemplate` 中的拦截器列表，以保证 `LoadBalancerInterceptor` 可以起作用。

## 小结

----

1. Ribbon 的核心组件包含 `LoadBalancerClient`（负载均衡器）、`RibbonClientHttpRequestFactory`（代替原有的请求工厂以创建带有负载均衡的请求）、`LoadBalancerInterceptor`（负载均衡的核心拦截器）等；
2. Ribbon 的负载均衡核心是拦截器，它配合负载均衡器实现服务调用的负载均衡；
3. 服务调用的负载均衡可以在配置文件中显式的为特殊的服务进行定制化，底层会借助 `PropertiesFactory` 完成负载均衡策略的读取；
4. `RestTemplate` 发起请求的底层是先通过 `HttpRequestFactory` 创建请求，再进行请求的实际发起，通过切换不同的 `HttpRequestFactory` 可以实现不同的请求创建。

![](./img/2023/05/cloud11-4.png)

