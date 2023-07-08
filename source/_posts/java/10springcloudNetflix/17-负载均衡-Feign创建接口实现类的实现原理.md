---
title: 17-负载均衡-Feign创建接口实现类的实现原理
toc: true
tags: springcloud
categories: 
    - [java]
    - [springcloudNetflix]
---

上一章的一开始，在测试 Feign 的调用全流程时，咱有注意到 Feign 创建的接口实现类的类型为 `HardCodedTarget` ，而且咱也说了它的创建与 `FeignClientFactoryBean` 有必然联系。本章咱就来一探究竟。

## 1. FeignClientFactoryBean的作用时机

在第15章的第1节最后咱第一次看到 `FeignClientFactoryBean` 的身影，当时咱只是知道他会创建这样的一些 FactoryBean ，但具体的创建时机咱还没有仔细去看。咱都知道，FactoryBean 的作用时机是 `getObject` ，那咱就把断点打在 `FeignClientFactoryBean` 的 `getObject` 方法中，Debug启动服务，看看它是怎么创建的。

<!--more-->

### 1.1 断点进入

断点停在 `getObject` 方法中，从Debug的方法栈上一步一步往上找，发现它是通过 `getBean` 被触发调用，而调用了 `getBean` 方法的源对象是咱事先声明的 `ConsumerController` ：

![](./img/2023/06/cloud17-1.png)

那进入 `getObject` 方法中，该方法直接调了另外一个 `getTarget` 方法：

```java
public Object getObject() throws Exception {
    return getTarget();
}
```

### 1.2 getTarget

这部分源码不算很长，不过咱只需要看前半部分即可，下面的部分是单独使用 Feign 时的处理逻辑，与 Ribbon 无关。那咱来看这里面的逻辑：（注释已标注在源码中）

```java
<T> T getTarget() {
    // 1.3 这个IOC容器是应用级的，不是Feign内部维护的
    FeignContext context = this.applicationContext.getBean(FeignContext.class);
    // 2. feign(context)的动作
    Feign.Builder builder = feign(context);

    if (!StringUtils.hasText(this.url)) {
        if (!this.name.startsWith("http")) {
            this.url = "http://" + this.name;
        }
        else {
            this.url = this.name;
        }
        this.url += cleanPath();
        // 3. 创建接口实现类的真正步骤。
        return (T) loadBalance(builder, context,
                new HardCodedTarget<>(this.type, this.name, this.url));
    }
    // ......
}
```

可以看到我很清晰的把这部分逻辑拆分成3部分，咱一个一个来看。

### 1.3 FeignContext的注册与获取

在第15章的第2.1节，咱已经介绍过 `FeignContext` 的结构和功能，它继承了 `NamedContextFactory` ：

```java
public class FeignContext extends NamedContextFactory<FeignClientSpecification>
```

能马上回忆起来吧，它内部维护了一组 IOC 容器，key 是微服务名称。对应的每个服务都会维护一个子 IOC 容器，内部存放了关于 Feign 的相关组件。它在初始化时也传入了一组 `configurations` ：

```java
@Bean
public FeignContext feignContext() {
    FeignContext context = new FeignContext();
    context.setConfigurations(this.configurations);
    return context;
}
```

不过，通过Debug可以发现它的内容与 Ribbon 不太一样：

![](./img/2023/06/cloud17-2.png)

可以看得出来它封装的这些配置是基于服务来的，不是基于配置类。不过这些不在咱的研究范围内，大可先略过。

### 1.4 FeignClientsConfiguration

留意一下 `FeignContext` 的构造方法：

```java
public FeignContext() {
    super(FeignClientsConfiguration.class, "feign", "feign.client.name");
}
```

它设置了默认的配置类是 `FeignClientsConfiguration` ，那咱还得看看这个配置类里都干了什么。

#### 1.4.1 编码、解码器

```java
@Bean
@ConditionalOnMissingBean
public Decoder feignDecoder() {
    return new OptionalDecoder(
            new ResponseEntityDecoder(new SpringDecoder(this.messageConverters)));
}

@Bean
@ConditionalOnMissingBean
@ConditionalOnMissingClass("org.springframework.data.domain.Pageable")
public Encoder feignEncoder() {
    return new SpringEncoder(this.messageConverters);
}

```

#### 1.4.2 SpringMvcContract

```java
@Bean
@ConditionalOnMissingBean
public Contract feignContract(ConversionService feignConversionService) {
    return new SpringMvcContract(this.parameterProcessors, feignConversionService);
}
```

表面上可以理解为 “基于SpringWebMvc的协议” ，但它是干什么的呢？回想咱在学习 Feign 的使用，是不是有提到 **Feign 的接口完全可以按照 SpringWebMvc 的格式来编写**，Feign 会自己转换为自己熟悉的格式。负责解析的活就是它干的，至于里头都是怎么干的，咱这里就先不展开了，感兴趣的小伙伴可以自行阅读。

#### 1.4.3 Feign.Builder

```java
@Bean
@Scope("prototype")
@ConditionalOnMissingBean
public Feign.Builder feignBuilder(Retryer retryer) {
    return Feign.builder().retryer(retryer);
}
```

这个 `Builder` 就是咱下面第2节中的 `Builder` ，正好也写到这里了，咱就直接来看 `Builder` 的创建。

## 2. feign(context)

```java
protected Feign.Builder feign(FeignContext context) {
    FeignLoggerFactory loggerFactory = get(context, FeignLoggerFactory.class);
    Logger logger = loggerFactory.create(this.type);

    // @formatter:off
    // 2.1 获取Builder
    Feign.Builder builder = get(context, Feign.Builder.class)
            // required values
            .logger(logger)
            .encoder(get(context, Encoder.class))
            .decoder(get(context, Decoder.class))
            .contract(get(context, Contract.class));
    // @formatter:on

    // 2.2 Builder配置
    configureFeign(context, builder);

    return builder;
}
```

上面的日志工厂咱就不关心了，中间的两个步骤是很关键的，咱分别来看。

### 2.1 获取Builder

```java
protected <T> T get(FeignContext context, Class<T> type) {
    T instance = context.getInstance(this.contextId, type);
    if (instance == null) {
        throw new IllegalStateException(
                "No bean found of type " + type + " for " + this.contextId);
    }
    return instance;
}
```

看到这里又可以会心一笑了，它又是从服务对应的子 IOC 容器中找，那必然是能找到的（上面解析的默认配置类中注册的），之后返回的类型就是 `Feign.Builder` 。

### 2.2 configureFeign

```java
protected void configureFeign(FeignContext context, Feign.Builder builder) {
    // FeignClientProperties即application.yml中的属性映射配置
    FeignClientProperties properties = this.applicationContext
            .getBean(FeignClientProperties.class);
    if (properties != null) {
        if (properties.isDefaultToProperties()) {
            configureUsingConfiguration(context, builder);
            configureUsingProperties(
                    properties.getConfig().get(properties.getDefaultConfig()), builder);
            configureUsingProperties(properties.getConfig().get(this.contextId), builder);
        } else {
            configureUsingProperties(
                    properties.getConfig().get(properties.getDefaultConfig()), builder);
            configureUsingProperties(properties.getConfig().get(this.contextId), builder);
            configureUsingConfiguration(context, builder);
        }
    } else {
        configureUsingConfiguration(context, builder);
    }
}
```

这部分可以看得出来，它是将应用级 IOC 容器中的一些配置放入即将要创建的 Feign 客户端建造器中。注意这里面还有配置生效的顺序：如果从映射文件中能加载到配置，并且都是默认的配置，则先执行配置，后映射属性；如果加载到的配置是自定义过的，则先映射属性，后执行配置。这其实就是 SpringBoot + Feign 对于 JavaConfig 与配置文件的优先级设定，**当设定了 feign.client.default-to-properties=true 时，配置文件中的配置会覆盖掉默认的配置**。

关于这部分的内部实现，咱以 `configureUsingConfiguration` 方法为例，可以看到这部分它就是干的这种赋值操作：

```java
protected void configureUsingConfiguration(FeignContext context,
        Feign.Builder builder) {
    Logger.Level level = getOptional(context, Logger.Level.class);
    if (level != null) {
        builder.logLevel(level);
    }
    Retryer retryer = getOptional(context, Retryer.class);
    if (retryer != null) {
        builder.retryer(retryer);
    }
    // ......
}
```

至此，Builder 已经创建并获取到，下一步要创建真正的接口实现类了。

## 3. loadBalance()：创建接口实现类

这部分的代码逻辑不多，简单扫一眼：

```java
protected <T> T loadBalance(Feign.Builder builder, FeignContext context,
        HardCodedTarget<T> target) {
    // 此处获得Feign用于远程调用的Client
    Client client = getOptional(context, Client.class);
    if (client != null) {
        builder.client(client);
        // 获得默认的Targeter
        Targeter targeter = get(context, Targeter.class);
        // 4. 建造器创建接口实现类
        return targeter.target(this, builder, context, target);
    }
    throw new IllegalStateException(
            "No Feign Client for loadBalancing defined. Did you forget to include spring-cloud-starter-netflix-ribbon?");
}
```

前面两个动作比较简单，都是从 `FeignContext` 的子 IOC 容器中取组件：

```java
protected <T> T getOptional(FeignContext context, Class<T> type) {
    return context.getInstance(this.contextId, type);
}
```

```java
protected <T> T get(FeignContext context, Class<T> type) {
    T instance = context.getInstance(this.contextId, type);
    if (instance == null) {
        throw new IllegalStateException(
                "No bean found of type " + type + " for " + this.contextId);
    }
    return instance;
}
```

注意这里获取到的 `Targeter` 类型是 `HystrixTargeter` （在15章解释过了）。

最后的 `targeter.target()` 是调用的建造器，注意这里面它传入了一个新创建的 `HardCodedTarget` ，并将接口的字节码传入。

```java
return (T) loadBalance(builder, context,
         new HardCodedTarget<>(this.type, this.name, this.url));
```

借助Debug可以清楚的看出来，咱声明的基于 Feign 的接口已经被解析好了：

![](./img/2023/06/cloud17-3.png)

随后进入 `target()` 方法，它的实现比较复杂，咱单独抽出来解析。

## 4. targeter.target：真正的创建动作

刚说了，`Targeter` 的类型是 `HystrixTargeter` ，那咱进入 `HystrixTargeter` 的 `target` 方法：

![](./img/2023/06/cloud17-4.png)

```java
java
复制代码public <T> T target(FeignClientFactoryBean factory, Feign.Builder feign,
        FeignContext context, Target.HardCodedTarget<T> target) {
    if (!(feign instanceof feign.hystrix.HystrixFeign.Builder)) {
        return feign.target(target);
    }
    // 一些关于Hystrix的处理......

    return feign.target(target);
}
```

中间的部分都是关于 Hystrix 的扩展部分，咱暂时不关心，先确定一下 `feign` 这个对象的类型。其实前面读的过程中早就知道，它就是 `Feign.Builder` 本身的类型，不是派生类。所以下面的部分都不会进，直接就执行与之前看过 `DefaultTargeter` 一样的动作：`feign.target(target);` 。

```java
public <T> T target(Target<T> target) {
    return build().newInstance(target);
}
```

这里面就两句话，但这两句话的内容却很复杂，咱展开来看。

### 4.1 build()

由于 `HystrixFeign` 重写了 `build()` 方法，那咱先看 `HystrixFeign` 重写的实现：

```java
public Feign build() {
    return build(null);
}

Feign build(final FallbackFactory<?> nullableFallbackFactory) {
    super.invocationHandlerFactory(new InvocationHandlerFactory() {
        @Override
        public InvocationHandler create(Target target,
                                                                        Map<Method, MethodHandler> dispatch) {
            return new HystrixInvocationHandler(target, dispatch, setterFactory,
                    nullableFallbackFactory);
        }
    });
    super.contract(new HystrixDelegatingContract(contract));
    return super.build();
}
```

可见它又是向建造器中放了几个组件，而且都是关于 Hystrix 的，这个咱暂时就不关心了，最后它会执行父类的 `build` 方法，跳转过去：

```java
public Feign build() {
    SynchronousMethodHandler.Factory synchronousMethodHandlerFactory =
            new SynchronousMethodHandler.Factory(client, retryer, requestInterceptors, logger,
                    logLevel, decode404, closeAfterDecode, propagationPolicy);
    ParseHandlersByName handlersByName =
            new ParseHandlersByName(contract, options, encoder, decoder, queryMapEncoder,
                    errorDecoder, synchronousMethodHandlerFactory);
    return new ReflectiveFeign(handlersByName, invocationHandlerFactory, queryMapEncoder);
}
```

这里涉及到两个特殊的组件，分别来看：

#### 4.1.1 【MethodHandler】SynchronousMethodHandler

还记得前面看过的第16章第2节，进入的那个 `MethodHandler` 吗？就是在这里预先初始化好的工厂，等到下面进行创建。

它这里面组合了几个核心的组件，一一列举：

- `client` ：真正发起请求的客户端，在16章咱已经看过了，底层使用 jdk 最基本的网络相关API
- `retryer` ：重试器，根据不同的重试策略，在请求失败时执行对应的逻辑
- `requestInterceptors` ：发起请求前的拦截器组

通过Debug，发现这些类型基本都有，但是没有拦截器：

![](./img/2023/06/cloud17-5.png)

#### 4.1.2 ParseHandlersByName

从类名上，可以猜测它可能是将一些东西根据名称转换成一些特定的 Handler ？先保留这个猜测，下面马上就可以看到它的真实作用。

### 4.2 newInstance

上面的两个组件都构造好，下面就可以创建 `ReflectiveFeign` ，随后执行 `newInstance` 方法了，这段逻辑可以分为4个部分，咱下面一一来看：

```java
private final ParseHandlersByName targetToHandlersByName;

public <T> T newInstance(Target<T> target) {
    // 4.2.1 解析目标接口的方法
    Map<String, MethodHandler> nameToHandler = targetToHandlersByName.apply(target);
    Map<Method, MethodHandler> methodToHandler = new LinkedHashMap<Method, MethodHandler>();
    List<DefaultMethodHandler> defaultMethodHandlers = new LinkedList<DefaultMethodHandler>();

    // 4.2.2 接口方法包装执行器
    for (Method method : target.type().getMethods()) {
        if (method.getDeclaringClass() == Object.class) {
            continue;
        } else if (Util.isDefault(method)) {
            DefaultMethodHandler handler = new DefaultMethodHandler(method);
            defaultMethodHandlers.add(handler);
            methodToHandler.put(method, handler);
        } else {
            methodToHandler.put(method, nameToHandler.get(Feign.configKey(target.type(), method)));
        }
    }
    // 4.2.3 创建接口的代理对象
    InvocationHandler handler = factory.create(target, methodToHandler);
    T proxy = (T) Proxy.newProxyInstance(target.type().getClassLoader(),
            new Class<?>[] {target.type()}, handler);

    // 4.2.4 将default类型的方法绑定到代理对象上
    for (DefaultMethodHandler defaultMethodHandler : defaultMethodHandlers) {
        defaultMethodHandler.bindTo(proxy);
    }
    return proxy;
}

```

#### 4.2.1 ParseHandlersByName#apply

以下源码只截取 apply 方法的第一句，这一句才是整个方法的核心：

##### 4.2.1.1 Contract解析Feign接口

```java
private final Contract contract;

public Map<String, MethodHandler> apply(Target key) {
    List<MethodMetadata> metadata = contract.parseAndValidatateMetadata(key.type());
    // ......
}
```

这个 `contract` 咱上面刚在 1.4.2 看到过，它是基于 SpringWebMvc 的编写规范。在 `SpringMvcContract` 中并没有直接实现返回值为 `List<MethodMetadata>` 的 `parseAndValidatateMetadata` 方法，是由它的父类 `BaseContract` 实现：

```java
public List<MethodMetadata> parseAndValidatateMetadata(Class<?> targetType) {
    // check and validate ......
    Map<String, MethodMetadata> result = new LinkedHashMap<String, MethodMetadata>();
    // 取接口中的所有方法
    for (Method method : targetType.getMethods()) {
        if (method.getDeclaringClass() == Object.class ||
                (method.getModifiers() & Modifier.STATIC) != 0 ||
                Util.isDefault(method)) {
            continue;
        }
        // 模板方法解析单个方法
        MethodMetadata metadata = parseAndValidateMetadata(targetType, method);
        checkState(!result.containsKey(metadata.configKey()), "Overrides unsupported: %s",
                metadata.configKey());
        result.put(metadata.configKey(), metadata);
    }
    return new ArrayList<>(result.values());
}

```

这里面的单个方法的解析，才是 `SpringMvcContract` 负责的：

##### 4.2.1.2 SpringMvcContract解析单个接口方法

核心注释已标注在源码中：

```java
public MethodMetadata parseAndValidateMetadata(Class<?> targetType, Method method) {
    // configKey可以返回[类名#方法名]格式的字符串，用该办法来存储Feign的每个接口方法
    this.processedMethods.put(Feign.configKey(targetType, method), method);
    // 4.2.1.3 BaseContract定义解析流程规范
    MethodMetadata md = super.parseAndValidateMetadata(targetType, method);

    // 检查接口上是否也标注了RequestMapping，如果有，合并接口信息
    RequestMapping classAnnotation = findMergedAnnotation(targetType,
            RequestMapping.class);
    if (classAnnotation != null) {
        // produces - use from class annotation only if method has not specified this
        if (!md.template().headers().containsKey(ACCEPT)) {
            parseProduces(md, method, classAnnotation);
        }
        // consumes -- use from class annotation only if method has not specified this
        if (!md.template().headers().containsKey(CONTENT_TYPE)) {
            parseConsumes(md, method, classAnnotation);
        }
        // headers -- class annotation is inherited to methods, always write these if present
        parseHeaders(md, method, classAnnotation);
    }
    return md;
}
```

上面的 `configKey` 方法返回的格式可以通过Debug查看，内部逻辑比较简单，这里就不再展开了。下面的处理是合并 Feign 接口上标注的 `@RequestMapping` 注解，由于之前的测试接口中没有，故这里也不会处理。那核心的方法就是调父类 `BaseContract` 的 `parseAndValidateMetadata` 方法。

##### 4.2.1.3 BaseContract#parseAndValidateMetadata

核心注释已标注在源码中：

```java
protected MethodMetadata parseAndValidateMetadata(Class<?> targetType, Method method) {
    MethodMetadata data = new MethodMetadata();
    // 获取接口方法的类型
    data.returnType(Types.resolve(targetType, targetType, method.getGenericReturnType()));
    // 解析出configKey
    data.configKey(Feign.configKey(targetType, method));

    // 解析接口上的@RequestMapping注解
    if (targetType.getInterfaces().length == 1) {
        processAnnotationOnClass(data, targetType.getInterfaces()[0]);
    }
    processAnnotationOnClass(data, targetType);

    // 4.2.1.4 解析接口方法上的@RequestMapping注解
    for (Annotation methodAnnotation : method.getAnnotations()) {
        processAnnotationOnMethod(data, methodAnnotation, method);
    }
    // check ......
    // 获取方法参数和泛型类型
    Class<?>[] parameterTypes = method.getParameterTypes();
    Type[] genericParameterTypes = method.getGenericParameterTypes();

    // 获取方法参数上标注的注解
    Annotation[][] parameterAnnotations = method.getParameterAnnotations();
    int count = parameterAnnotations.length;
    // 解析方法参数及参数注解
    for (int i = 0; i < count; i++) {
        boolean isHttpAnnotation = false;
        if (parameterAnnotations[i] != null) {
            isHttpAnnotation = processAnnotationsOnParameter(data, parameterAnnotations[i], i);
        }
        if (parameterTypes[i] == URI.class) {
            data.urlIndex(i);
        } else if (!isHttpAnnotation && parameterTypes[i] != Request.Options.class) {
            // check ......
            data.bodyIndex(i);
            data.bodyType(Types.resolve(targetType, targetType, genericParameterTypes[i]));
        }
    }

    // check ......
    return data;
}
```

可以发现它的解析逻辑是很有序的：**接口 → 方法 → 方法参数**。解析接口的部分也只是把接口中的 `@RequestMapping` 注解标注的内容先提取出来，用于和下面解析方法的部分合并。那咱关心的部分当然是解析接口方法了，而这个方法又是个模板方法，由 `SpringMvcContract` 实现：

##### 4.2.1.4 processAnnotationOnMethod：解析接口方法

核心注释已标注在源码中：

```java
protected void processAnnotationOnMethod(MethodMetadata data,
        Annotation methodAnnotation, Method method) {
    if (!RequestMapping.class.isInstance(methodAnnotation) && !methodAnnotation
            .annotationType().isAnnotationPresent(RequestMapping.class)) {
        return;
    }

    // 获取请求uri和请求方式
    RequestMapping methodMapping = findMergedAnnotation(method, RequestMapping.class);
    // HTTP Method
    RequestMethod[] methods = methodMapping.method();
    if (methods.length == 0) {
        methods = new RequestMethod[] { RequestMethod.GET };
    }
    checkOne(method, methods, "method");
    data.template().method(Request.HttpMethod.valueOf(methods[0].name()));

    // path
    checkAtMostOne(method, methodMapping.value(), "value");
    if (methodMapping.value().length > 0) {
        // 获取uri，并进行一些容错处理
        String pathValue = emptyToNull(methodMapping.value()[0]);
        if (pathValue != null) {
            pathValue = resolve(pathValue);
            // Append path from @RequestMapping if value is present on method
            if (!pathValue.startsWith("/") && !data.template().path().endsWith("/")) {
                pathValue = "/" + pathValue;
            }
            // 方法的uri与接口的uri拼接
            data.template().uri(pathValue, true);
        }
    }

    // produces 处理特殊的请求、处理类型，不是重点暂不关心
    parseProduces(data, method, methodMapping);
    // consumes
    parseConsumes(data, method, methodMapping);
    // headers
    parseHeaders(data, method, methodMapping);

    data.indexToExpander(new LinkedHashMap<Integer, Param.Expander>());
}
```

这段逻辑中就可以把请求 uri 和请求方式都取出来了，uri 的拼接部分在解析 path 部分的最后一句：`data.template().uri(pathValue, true);` ，它会将实现解析好接口 uri 的部分取出，并与方法上的 uri 拼接。

经过这些解析，其实 `apply` 方法执行的核心也就过去了，剩下的补充和完善的动作咱这里就不具体解释了，感兴趣的小伙伴可以跟着Debug一步一步走走看。

#### 4.2.2 接口方法包装执行器

```java
    // 4.2.2 接口方法包装执行器
    for (Method method : target.type().getMethods()) {
        if (method.getDeclaringClass() == Object.class) {
            continue;
        } else if (Util.isDefault(method)) {
            DefaultMethodHandler handler = new DefaultMethodHandler(method);
            defaultMethodHandlers.add(handler);
            methodToHandler.put(method, handler);
        } else {
            methodToHandler.put(method, nameToHandler.get(Feign.configKey(target.type(), method)));
        }
    }
```

这部分的动作，它会将 Feign 接口中的方法再次取出，并放入 `methodToHandler` 集合中。由于这部分的 `nameToHandler` 就是上面解析好的那一组 `MethodHandler` ，故这里取出的方法就会依次放入 `methodToHandler` 中。注意这里面的 else if 块，它会单独处理接口中声明了 **default** 关键字的方法，至于为什么要单独处理，咱往下看到 4.2.4 就知道啦。

#### 4.2.3 【InvocationHandler】创建接口的代理对象

```java
    // 4.2.3 创建接口的代理对象
    InvocationHandler handler = factory.create(target, methodToHandler);
    T proxy = (T) Proxy.newProxyInstance(target.type().getClassLoader(),
            new Class<?>[] {target.type()}, handler);
```

这部分来到最熟悉的 jdk 动态代理环节。注意它创建的 `InvocationHandler` ：

```java
public InvocationHandler create(Target target, Map<Method, MethodHandler> dispatch) {
    return new ReflectiveFeign.FeignInvocationHandler(target, dispatch);
}
```

这个 `FeignInvocationHandler` 不就是在16章1.2节看到的动态代理拦截器吗？咱上一节还看到过它。它的结构如下：

```java
static class FeignInvocationHandler implements InvocationHandler {
    private final Target target;
    private final Map<Method, MethodHandler> dispatch;
```

它的核心方法 `invoke` ：

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    // ......
    return dispatch.get(method).invoke(args);
}
```

等一下。。。感觉不对劲，怎么 `target` 直接没在远程调用中起作用？

（建议小伙伴快速翻一下16章，回味一下整个调用过程中 `HardCodedTargeter` 有没有在哪里用到过）

如果小伙伴已经翻完回来了，会发现貌似真的没起作用。。。那就对了嘛，`HardCodedTargeter` 本身就没什么逻辑实现，还指望它干啥事？（滑稽）

至此，代理对象已经创建完毕，`FeignClientFactoryBean` 的 `getObject` 方法执行完毕，Feign 代理对象的初始化完毕。

## 小结

1. `FeignClientFactoryBean` 的核心方法 `getObject` 可以返回基于 `HardCodedTargeter` 的代理对象，内部的拦截器 `MethodHandler` 是发送请求的核心；
2. Feign 接口的解析由 `ParseHandler` 交予 `Contract` 完成，在引入 webmvc 后底层会使用 SpringWebMvc 的规范来解析 Feign 接口。

【至此，Feign 的相关原理解析也就基本完毕了。这几章咱多次频繁看到了 Hystrix ，说明负载均衡通常与熔断处理一起协作完成远程调用，那接下来的几章咱就来看关于 Hystrix 的原理】



