---
title: 18-熔断降级-Hystrix的初始化与启动原理.md
toc: true
tags: springcloud
categories: 
    - [java]
    - [springcloudNetflix]
---

## 0. 测试环境搭建

咱基于上一部分 Feign 的工程做一些改造：

为了能达到服务降级的效果，咱需要声明一个 `ProviderFeignClient` 的实现类，来实现服务调用出错时的降级方法：

<!--more-->


```java
@Component
public class ProviderHystrixFallback implements ProviderFeignClient {
    
    @Override
    public String getInfo() {
        return "hystrix fallback getInfo ......";
    }
}
```

在 ProviderFeignClient 接口中配置该降级类：

```java
@FeignClient(value = "eureka-client", fallback = ProviderHystrixFallback.class)
public interface ProviderFeignClient
```

之后 `application.properties` 中加入配置：

```properties
feign.hystrix.enabled=true
```

最后，主启动类上标注 `@EnableHystrix` 或 `@EnableCircuitBreaker` 注解即可。（这俩都一样）

启动 eureka-server 与 eureka-client ，随后启动刚改造好的 hystrix-consumer 工程。

浏览器第一次发送请求，可以正常响应；关闭 eureka-client 服务，再次发起请求，一秒延迟后浏览器响应 `hystrix fallback getInfo ......` 信息，证明服务降级成功。

上面的启动 Hystrix 需要标注 `@EnableHystrix` 注解，按照前面看过的大多数套路，它肯定要么导了配置类，要么导了 `ImportSelector` 。

## 1. @EnableHystrix

```java
@EnableCircuitBreaker
public @interface EnableHystrix
```

看到这里小伙伴就明白为啥我说两个注解都一样了吧（滑稽）。继续进入 `@EnableCircuitBreaker` ：

```java
@Import(EnableCircuitBreakerImportSelector.class)
public @interface EnableCircuitBreaker
```

它果然导了一个 `EnableCircuitBreakerImportSelector` ，那咱就进去看它的作用。

### 1.1 EnableCircuitBreakerImportSelector的继承

```java
public class EnableCircuitBreakerImportSelector
		extends SpringFactoryImportSelector<EnableCircuitBreaker>
```

`EnableCircuitBreakerImportSelector`里面没有实现 `selectImports` 方法，而是由它继承的 `SpringFactoryImportSelector` 实现。注意它还传入了一个泛型，就是 `@EnableCircuitBreaker` 注解本身，那它传入的目的是什么呢？

大胆猜测一波，咱看 **SpringFactory** 这个关键词，应该能联想到 SpringFramework 的 SPI 技术，那传入一个 `@EnableCircuitBreaker` 注解，又让我联想到 `@EnableAutoConfiguration` ，难不成它的底层就是**借助 SpringFramework 的 SPI 去 `spring.factories` 中找对应的配置类**？下面咱就来验证。

### 1.2 SpringFactoryImportSelector#selectImports

```java
public String[] selectImports(AnnotationMetadata metadata) {
    if (!isEnabled()) {
        return new String[0];
    }
    AnnotationAttributes attributes = AnnotationAttributes.fromMap(
            metadata.getAnnotationAttributes(this.annotationClass.getName(), true));
    // assert

    // Find all possible auto configuration classes, filtering duplicates
    List<String> factories = new ArrayList<>(new LinkedHashSet<>(SpringFactoriesLoader
            .loadFactoryNames(this.annotationClass, this.beanClassLoader)));

    if (factories.isEmpty() && !hasDefaultFactory()) {
        throw new IllegalStateException("Annotation @" + getSimpleName()
                + " found, but there are no implementations. Did you forget to include a starter?");
    }
    if (factories.size() > 1) {
        // there should only ever be one DiscoveryClient, but there might be more than one factory
        this.log.warn("More than one implementation " + "of @" + getSimpleName()
                + " (now relying on @Conditionals to pick one): " + factories);
    }
    return factories.toArray(new String[factories.size()]);
}
```

看中间，它真的拿 `SpringFactoriesLoader` 调 `loadFactoryNames` 方法，那就与咱的猜测一致了。那下面的内容咱也不用看了，直接去 `spring.factories` 文件中找就OK了。

### 1.3 spring.factories中配置的类

```properties
org.springframework.cloud.client.circuitbreaker.EnableCircuitBreaker=\
    org.springframework.cloud.netflix.hystrix.HystrixCircuitBreakerConfiguration
```

果然这里有一个配置类，那咱就着重研究它。

## 2. HystrixCircuitBreakerConfiguration

首先咱应该注意到，它不是一个自动配置类，那它就没有其他那么多纠葛。这里面注册了两个组件，分别来看：

### 2.1 HystrixShutdownHook

```java
@Bean
public HystrixShutdownHook hystrixShutdownHook() {
    return new HystrixShutdownHook();
}
```

很明显它是一个用于绑定容器关闭时的附带操作，这里面的设计极为简单：

```java
private class HystrixShutdownHook implements DisposableBean {
    @Override
    public void destroy() throws Exception {
        Hystrix.reset();
    }
}
```

它实现了 `DisposableBean` ，并在 `destroy` 中使 Hystrix 重置并释放资源。`reset` 内部的实现咱就不多关心了，知道有这么回事就OK，重点在下面的组件。

### 2.2 HystrixCommandAspect

```java
@Bean
public HystrixCommandAspect hystrixCommandAspect() {
    return new HystrixCommandAspect();
}
```

这个咱太熟悉了，AOP 切面类！赶快点进去看看有什么切面：

```java
@Pointcut("@annotation(com.netflix.hystrix.contrib.javanica.annotation.HystrixCommand)")
public void hystrixCommandAnnotationPointcut() {
}

@Pointcut("@annotation(com.netflix.hystrix.contrib.javanica.annotation.HystrixCollapser)")
public void hystrixCollapserAnnotationPointcut() {
}
```

可以看得出来，这两个切面都是作用于 SpringWebMvc 的 Controller 中标注的 Hystrix 注解方法上！有关这部分内容咱放到后面来研究，由于前面是承接着 Feign 来的，咱们先来探究基于 Feign 的 Hystrix 降级。

那这两个组件都过了一遍，这个 `HystrixCircuitBreakerConfiguration` 也就没别的东西了。回顾一下刚才咱看的 `spring.factories` 文件，这里面还定义了两个自动配置类：

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
    org.springframework.cloud.netflix.hystrix.HystrixAutoConfiguration,\
    org.springframework.cloud.netflix.hystrix.security.HystrixSecurityAutoConfiguration
```

关于 Security 的部分咱就不关心了，咱主要还是看主线的自动配置类。

## 3. HystrixAutoConfiguration

这个类上没有再导入其他的组件，也没有搞别的花板子。咱直接看注册的组件就好。

### 3.1 HystrixHealthIndicator

```java
@Bean
@ConditionalOnEnabledHealthIndicator("hystrix")
public HystrixHealthIndicator hystrixHealthIndicator() {
    return new HystrixHealthIndicator();
}
```

很明显就知道它是关于服务健康检查的。如果小伙伴们对 Hystrix 的健康检查与仪表盘有印象，那这个组件应该很好理解。Hystrix 可以借助 SpringBoot 的 actuator 模块可以直接暴露出 Hystrix 自身的监控数据，并且 Hystrix 自身也有 dashboard ，可以借助前面暴露的监控数据来整理出简单的可视化视图。至于它内部的处理逻辑这个咱就不关心了，感兴趣的小伙伴可以深入了解。

### 3.2 HystrixMetricsBinder

```java
@Bean
public HystrixMetricsBinder hystrixMetricsBinder() {
    return new HystrixMetricsBinder();
}
```

这个组件可能咱搞不懂是什么东西，这里咱需要回到 pom 中去看一样东西：

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
```

打开它的聚合依赖，在依赖列表中看到了这么一条：

```xml
<dependency>
    <groupId>com.netflix.hystrix</groupId>
    <artifactId>hystrix-metrics-event-stream</artifactId>
</dependency>
```

这个名就比较熟悉了，这个依赖可以将 Hystrix 的性能监控指标暴露为 endpoint ，只要客户端连接到 Hystrix 的监控仪表盘，这个依赖中的组件就会以 `text/event-stream` 的形式向客户端推送监控指标结果。

那解释到这里，`HystrixMetricsBinder` 的作用场景也就大体清楚了，它也是配合这个性能监控指标来的。

### 3.3 HystrixStreamEndpoint

```java
@Bean
@ConditionalOnEnabledEndpoint
public HystrixStreamEndpoint hystrixStreamEndpoint(HystrixProperties properties) {
    return new HystrixStreamEndpoint(properties.getConfig());
}
```

注意在这个组件的上面，内部类中有一个额外的判断规则：

```java
@ConditionalOnWebApplication(type = SERVLET)

```

这个类咱倒是刚看没什么感觉，只是有个 endpoint 可能感觉跟上面有关系，点进去看一眼设计：

```java
@ServletEndpoint(id = "hystrix.stream")
public class HystrixStreamEndpoint implements Supplier<EndpointServlet> {

	private final Map<String, String> initParameters;

	public HystrixStreamEndpoint(Map<String, String> initParameters) {
		this.initParameters = initParameters;
	}

	@Override
	public EndpointServlet get() {
		return new EndpointServlet(HystrixMetricsStreamServlet.class)
				.withInitParameters(this.initParameters);
	}
}

```

看到底下的这个 `get` 方法，突然明白了它的作用：它是支撑上面所说的监控信息推送的，而且根据 `HystrixMetricsStreamServlet` 的文档注释，可以了解到获取推送数据的 uri 是 `/hystrix.stream` 。

合着到这里来看，`HystrixAutoConfiguration` 里面都是跟监控相关的内容，那也就没什么跟 Hystrix 本身功能相关的了。

---

这个时候疑问就产生了：既然上面的都没有涉及到 Feign ，那 Hystrix 配合 Feign 的内容又在哪里呢？

## 4. 回顾Feign的代理对象初始化流程

还记得17章中，`targeter.target` 的那一步吗？当时咱列举的源码如下：

```java
public <T> T target(FeignClientFactoryBean factory, Feign.Builder feign,
        FeignContext context, Target.HardCodedTarget<T> target) {
    if (!(feign instanceof feign.hystrix.HystrixFeign.Builder)) {
        return feign.target(target);
    }
    // 一些关于Hystrix的处理......

    return feign.target(target);
}
```

正好就省略掉了 Hystrix 的部分，那这里咱就不能省掉了，咱来看这一部分究竟都干什么了：

```java
// HystrixTargeter
public <T> T target(FeignClientFactoryBean factory, Feign.Builder feign,
        FeignContext context, Target.HardCodedTarget<T> target) {
    // ......
    // 强转为Hystrix的Builder，不再是普通的Builder
    feign.hystrix.HystrixFeign.Builder builder = (feign.hystrix.HystrixFeign.Builder) feign;
    String name = StringUtils.isEmpty(factory.getContextId()) ? factory.getName()
            : factory.getContextId();
    SetterFactory setterFactory = getOptional(name, context, SetterFactory.class);
    if (setterFactory != null) {
        builder.setterFactory(setterFactory);
    }
    // 检查接口注解上有没有标注fallback属性
    Class<?> fallback = factory.getFallback();
    if (fallback != void.class) {
        return targetWithFallback(name, context, target, builder, fallback);
    }
    // 检查接口注解上有没有标注fallbackFactory属性
    Class<?> fallbackFactory = factory.getFallbackFactory();
    if (fallbackFactory != void.class) {
        return targetWithFallbackFactory(name, context, target, builder, fallbackFactory);
    }

    return feign.target(target);
}
```

到这里可以发现，如果咱在 `@FeignClient` 注解中标注了 `fallback` 属性，就不会走最底下的 `target` 方法了，而是走另一个 `targetWithFallback` 方法。

### 4.1 targetWithFallback

```java
private <T> T targetWithFallback(String feignClientName, FeignContext context,
        Target.HardCodedTarget<T> target, HystrixFeign.Builder builder,
        Class<?> fallback) {
    T fallbackInstance = getFromContext("fallback", feignClientName, context,
            fallback, target.type());
    return builder.target(target, fallbackInstance);
}
```

看这里，除了下面一句话跟前面一样，前面还创建了一个 `fallbackInstance` ，它就是用于实现服务降级的接口默认实现类。`getFromContext` 的实现如下：

```java
private <T> T getFromContext(String fallbackMechanism, String feignClientName,
        FeignContext context, Class<?> beanType, Class<T> targetType) {
    Object fallbackInstance = context.getInstance(feignClientName, beanType);
    // check and throw ......
    return (T) fallbackInstance;
}
```

又又叕是这个套路，所以降级实现类就是这样创建出来的咯。直接返回就OK了。

### 4.2 builder.target

注意，这个 builder 是 Hystrix 的 builder ，它的逻辑与默认的不同：

```java
public <T> T target(Target<T> target, T fallback) {
    return build(fallback != null ? new FallbackFactory.Default<T>(fallback) : null)
            .newInstance(target);
}

```

注意这个 `build` 方法，这里面默认的把降级实现类传入进去，创建了一个 `FallbackFactory` ：

```java
Feign build(final FallbackFactory<?> nullableFallbackFactory) {
    super.invocationHandlerFactory(new InvocationHandlerFactory() {
        @Override
        public InvocationHandler create(Target target, Map<Method, MethodHandler> dispatch) {
            return new HystrixInvocationHandler(target, dispatch, setterFactory,
                    nullableFallbackFactory);
        }
    });
    super.contract(new HystrixDelegatingContract(contract));
    return super.build();
}

```

注意这里面实际不同的是 `invocationHandlerFactory` 被替换成可以创建基于 Hystrix 的 `InvocationHandler` ，那它必然有特殊功效，瞅瞅它都多干了什么。

### 4.3 HystrixInvocationHandler#invoke

这个方法里面大量的 if else if 结构，下面贴出的源码去掉与服务降级无关的部分，只留下最核心的：

```java
public Object invoke(final Object proxy, final Method method, final Object[] args)
        throws Throwable {
    if ("equals".equals(method.getName())) {
       // ...... else if ......
    }

    HystrixCommand<Object> hystrixCommand =
        new HystrixCommand<Object>(setterMethodMap.get(method)) {
            @Override
            protected Object run() throws Exception {
                try {
                    return HystrixInvocationHandler.this.dispatch.get(method).invoke(args);
                } // catch ......
            }

            @Override
            protected Object getFallback() {
                if (fallbackFactory == null) {
                    return super.getFallback();
                }
                try {
                    // 取出接口对应的降级实现类
                    Object fallback = fallbackFactory.create(getExecutionException());
                    // 反射执行降级方法
                    Object result = fallbackMethodMap.get(method).invoke(fallback, args);
                    if (isReturnsHystrixCommand(method)) {
                        return ((HystrixCommand) result).execute();
                    } // else if ...
                    else {
                        return result;
                    }
                } // catch ......
            }
        };

    if (Util.isDefault(method)) {
        return hystrixCommand.execute();
    } // else if ......
    return hystrixCommand.execute();
}
```

上面的 `run` 就是 `FeignInvocationHandler` 中执行的逻辑，但是下面多了一个 `getFallback` 方法，这里面的逻辑就是**在服务调用出现问题时，降级到这里，取出降级实现类并反射执行对应的方法**。

至此，Hystrix 配合 Feign 完成服务降级的设计就明确了。

## 小结

1. Hystrix 的自动配置中做的工作都是与监控有关，核心的作用机制在 `@EnableHystrix` 导入的 `ImportSelector` 注入的 `HystrixCircuitBreakerConfiguration` 中；
2. Hystrix 配合 Feign 完成服务降级是在执行 `targeter.target` 方法中进行的额外扩展，它改变了 `invocationHandlerFactory` 使其创建具有服务降级的 `HystrixInvocationHandler` 。

![](./img/2023/06/cloud17-6.png)



