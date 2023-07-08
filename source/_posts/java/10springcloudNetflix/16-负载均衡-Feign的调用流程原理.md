---
title: 16-负载均衡-Feign的调用流程原理
toc: true
tags: springcloud
categories: 
    - [java]
    - [springcloudNetflix]
---

体会全流程之前，咱还是得把环境先搭建好。

## 0. 测试环境搭建

<!--more-->

还是以 Ribbon 中的测试环境为基准，咱把 eureka-consumer 中加入 Feign ，Maven 中加入依赖：

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

之后编写一个接口，来调用 eureka-client 的服务：

```java
@FeignClient("eureka-client")
public interface ProviderFeignClient {
    
    @RequestMapping(value = "/getInfo", method = RequestMethod.GET)
    public String getInfo();
    
}
```

启动 eureka-server 与两个 eureka-client ，准备开始测试流程。

## 1. 请求发起

请求到达 `ConsumerController` 的 `getInfo` 方法，随后进入 `providerFeignClient` 的 `getInfo` 方法。

### 1.1 providerFeignClient的类型

在进入之前，咱先观察一下这个 `providerFeignClient` 的类型：

![](./img/2023/05/cloud16-1.png)

这个类型叫 `HardCodedTargeter` ，乍一看觉得没见过，实际上咱上一章有见过它：

```java
class DefaultTargeter implements Targeter {
    @Override
    public <T> T target(FeignClientFactoryBean factory, Feign.Builder feign,
            FeignContext context, Target.HardCodedTarget<T> target) {
        return feign.target(target);
    }
}
```

注意传入的参数，就是 `HardCodedTargeter` 类型的。不过这里咱不打算把 `HardCodedTargeter` 展开解释，到下一章咱会解析 `HardCodedTargeter` 的由来，以及 `FeignClientFactoryBean` 等组件的作用。

### 1.2 进入getInfo方法

断点停在 `getInfo` 方法，Debug发现进入的不是 `HardCodedTargeter` ，而是 `ReflectiveFeign` 的内部类 `FeignInvocationHandler` 中：

```java
private final Map<Method, MethodHandler> dispatch;

public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    if ("equals".equals(method.getName())) {
        // ......
    } else if ("hashCode".equals(method.getName())) {
        return hashCode();
    } else if ("toString".equals(method.getName())) {
        return toString();
    }

    return dispatch.get(method).invoke(args);
}

```

![](./img/2023/05/cloud16-2.png)

等等，`InvocationHandler` ？这不是 jdk 动态代理中的核心拦截器吗？上面拦截几个来自 Object 的方法咱就不关心了，主要是底下的 `dispatch.get(method).invoke(args)` ，可以发现这个 `dispatch` 的结构是 `Map` ，那自然可以联想到前面看过的好多套路，这分明是把接口中定义的方法都封装好了，要调哪个方法，直接从 `Map` 中取出对应的 `MethodHandler` 就可以了。

那咱就直接进入到 `MethodHandler` 中。

## 2. 动态代理进入MethodHandler

注意这个 `MethodHandler` 是 Feign 内部自己定义的接口，与 jdk 反射、动态代理无关：

```java
public interface InvocationHandlerFactory {
    // ......
    interface MethodHandler {
        Object invoke(Object[] argv) throws Throwable;
    }
```

跟着Debug来到 `SynchronousMethodHandler` 中：

```java
public Object invoke(Object[] argv) throws Throwable {
    // 2.1 创建RequestTemplate
    RequestTemplate template = buildTemplateFromArgs.create(argv);
    // 2.2 获取参数配置
    Options options = findOptions(argv);
    Retryer retryer = this.retryer.clone();
    while (true) {
        try {
            // 2.3 发起请求
            return executeAndDecode(template, options);
        } // catch throw ......
    }
}
```

这里面有3个步骤是比较重要的：创建 `RequestTemplate` 、获取请求相关参数、发起请求。下面咱展开来分析。

### 2.1 buildTemplateFromArgs.create

```java
public RequestTemplate create(Object[] argv) {
    // 2.1.1 根据元信息的请求模板创建新的模板
    RequestTemplate mutable = RequestTemplate.from(metadata.template());
    if (metadata.urlIndex() != null) {
        int urlIndex = metadata.urlIndex();
        checkArgument(argv[urlIndex] != null, "URI parameter %s was null", urlIndex);
        mutable.target(String.valueOf(argv[urlIndex]));
    }
    // 参数封装
    Map<String, Object> varBuilder = new LinkedHashMap<String, Object>();
    for (Entry<Integer, Collection<String>> entry : metadata.indexToName().entrySet()) {
        int i = entry.getKey();
        Object value = argv[entry.getKey()];
        if (value != null) { // Null values are skipped.
            if (indexToExpander.containsKey(i)) {
                value = expandElements(indexToExpander.get(i), value);
            }
            for (String name : entry.getValue()) {
                varBuilder.put(name, value);
            }
        }
    }

    // 2.1.2 请求参数编码/路径拼接
    RequestTemplate template = resolve(argv, mutable, varBuilder);
    if (metadata.queryMapIndex() != null) {
        // add query map parameters after initial resolve so that they take
        // precedence over any predefined values
        Object value = argv[metadata.queryMapIndex()];
        Map<String, Object> queryMap = toQueryMap(value);
        template = addQueryMapQueryParameters(queryMap, template);
    }

    if (metadata.headerMapIndex() != null) {
        template = addHeaderMapHeaders((Map<String, Object>) argv[metadata.headerMapIndex()], template);
    }

    return template;
}
```

整体的方法可以大致分为3个部分：`RequestTemplate` 的初始创建、请求参数封装、请求参数编码 / 路径拼接，整体逻辑结构比较清晰。中间的请求参数封装部分相对简单，咱这里不展开解释，重点来看上面和下面的两个关键操作。

#### 2.1.1 RequestTemplate.from

由方法名应该能大概猜到是根据一个特定的传入，来动态组装新的 `RequestTemplate` ，那咱来看方法实现：

```java
public static RequestTemplate from(RequestTemplate requestTemplate) {
    RequestTemplate template =
            new RequestTemplate(requestTemplate.target, requestTemplate.fragment,
                    requestTemplate.uriTemplate,
                    requestTemplate.method, requestTemplate.charset,
                    requestTemplate.body, requestTemplate.decodeSlash, requestTemplate.collectionFormat);

    if (!requestTemplate.queries().isEmpty()) {
        template.queries.putAll(requestTemplate.queries);
    }

    if (!requestTemplate.headers().isEmpty()) {
        template.headers.putAll(requestTemplate.headers);
    }
    return template;
}
```

可以发现，前面的构造方法调用就是取的原有 `template` 中的属性，只是下面会有一些其他属性的赋值。借助Debug，发现并没有进入下面两个 if 结构中，那也就相当于把原来的模板复制了一份新的而已了。

#### 2.1.2 resolve：请求参数编码/路径拼接

```java
protected RequestTemplate resolve(Object[] argv,
        RequestTemplate mutable, Map<String, Object> variables) {
    return mutable.resolve(variables);
}

public RequestTemplate resolve(Map<String, ?> variables) {
    StringBuilder uri = new StringBuilder();

    /* create a new template form this one, but explicitly */
    RequestTemplate resolved = RequestTemplate.from(this);

    // 巨长......
    return resolved;
}
```

这部分的操作实际上是对请求参数进行请求参数的字符串路径拼接，由于上述测试环境中咱发起的是 **GET** 请求，故进入这个方法，**POST** 请求会先进入 `BuildEncodedTemplateFromArgs` 的 `resolve` 方法，对请求参数进行编码。实现比较简单，这里不过多展开。

```java
private final Encoder encoder;

protected RequestTemplate resolve(Object[] argv,
        RequestTemplate mutable, Map<String, Object> variables) {
    Object body = argv[metadata.bodyIndex()];
    checkArgument(body != null, "Body parameter %s was null", metadata.bodyIndex());
    try {
        encoder.encode(body, metadata.bodyType(), mutable);
    } // catch ......
    return super.resolve(argv, mutable, variables);
}
```

### 2.2 findOptions

`RequestTemplate` 创建好后，下一步要获取实现配置好的请求相关参数：

```java
Options findOptions(Object[] argv) {
    if (argv == null || argv.length == 0) {
        return this.options;
    }
    return (Options) Stream.of(argv)
            .filter(o -> o instanceof Options)
            .findFirst()
            .orElse(this.options);
}
```

这里的处理流程也比较简单，它会找方法参数中第一个类型为 `Options` 的参数，并取出返回；如果找不到，就使用默认的 `Options` 参数。根据上一章 3.3 节咱看到的参数，默认情况下的参数设置为 10 秒的建立连接超时，60 秒的完整响应超时。

### 2.3 executeAndDecode

上面的准备工作都整理完毕后，下面要准备发起请求了。

```java
Object executeAndDecode(RequestTemplate template, Options options) throws Throwable {
    Request request = targetRequest(template);
    // logger ......

    Response response;
    long start = System.nanoTime();
    try {
        // 【核心】配合Ribbon进行负载均衡调用
        response = client.execute(request, options);
    } // catch throw .......
    long elapsedTime = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - start);

    // 请求后的处理 ......
}

```

前面的处理和后面的处理咱们暂且不关心了，中间的 client 要真正的准备发起请求。由于咱是测试 Feign 配合 Ribbon 的负载均衡调用，这里的 client 类型就不是简单的 `Default` 实现，而是配合 Ribbon 的 `LoadBalancerFeignClient` ：

```java
public Response execute(Request request, Request.Options options) throws IOException {
    try {
        // 2.3.1 构建能被Ribbon处理的url
        URI asUri = URI.create(request.url());
        // 获取被调用的服务名称
        String clientName = asUri.getHost();
        // 去掉服务名称后的url
        URI uriWithoutHost = cleanUrl(request.url(), clientName);
        // 2.3.2 借助Feign默认的Client封装请求，用于下面真正的请求发起
        FeignLoadBalancer.RibbonRequest ribbonRequest = new FeignLoadBalancer.RibbonRequest(
                this.delegate, request, uriWithoutHost);

        IClientConfig requestConfig = getClientConfig(options, clientName);
        // 2.3.3 发送请求
        return lbClient(clientName).executeWithLoadBalancer(ribbonRequest, requestConfig).toResponse();
    } // catch ......
}
```

这里面的几个动作部分咱一一来看：

#### 2.3.1 构建、处理url

`URI.create(request.url())` 这部分就是拿基于 Ribbon 的请求地址，下面获取到服务主机名，再将原有的请求地址中主机名部分去掉，整理思路很简单。咱主要是贴一张Debug的图来直观感受这部分的处理，下面可能会用得到。

![](./img/20203/05/cloud16-3.png)

#### 2.3.2 封装FeignLoadBalancer.RibbonRequest

注意这里面的封装内容有一个 `delegate` ，通过Debug可以发现它是 Feign 中默认实现的一个客户端：

```java
private final Client delegate;
```

![](./img/2023/05/cloud16-4.png)

而这个 `Client` 是一个接口，它只有 3 个实现，除了 `Default` 之外，另一个重要实现就是 `LoadBalancerFeignClient` 。

![](./img/2023/05/cloud16-5.png)

由此是不是会意识到一个问题：在12章中，Ribbon 使用 `RequestFactory` 创建特殊的 `RestTemplate` （带拦截器），在执行拦截器时再创建普通的 `RestTemplate` 发送请求。

那 Feign 这里会不会也是相似的操作呢？一开始它先创建 `LoadBalancerFeignClient` ，这里面包装一个普通的 Client ，真正的请求让这个普通的 Client 发，`LoadBalancerFeignClient` 只做前期准备处理。这个猜测肯定是正确的，下面咱进入真正的请求发起部分，咱就可以看到了。

#### 2.3.3 发送请求

```java
    return lbClient(clientName).executeWithLoadBalancer(ribbonRequest, requestConfig).toResponse();
```

注意这里面分两个动作：先根据服务名称取 `FeignLoadBalancer` ，再发起请求。

##### 2.3.3.1 lbClient(clientName)

```java
private CachingSpringLoadBalancerFactory lbClientFactory;

private FeignLoadBalancer lbClient(String clientName) {
    return this.lbClientFactory.create(clientName);
}

```

这手操作咱已经见到过了！而且上一章3.2节咱也写了，这里就不多重复了。

##### 2.3.3.2 executeWithLoadBalancer

这部分涉及到负载均衡器的工作内容了，咱把标题级别往上调一下，另起一行。

## 3. LoadBalancer的处理

```java
public T executeWithLoadBalancer(final S request, final IClientConfig requestConfig) throws ClientException {
    // 3.1 构造、封装参数
    LoadBalancerCommand<T> command = buildLoadBalancerCommand(request, requestConfig);
    try {
        // 3.2 执行请求发起的动作
        return command.submit(
            new ServerOperation<T>() {
                @Override
                public Observable<T> call(Server server) {
                    URI finalUri = reconstructURIWithServer(server, request.getUri());
                    S requestForServer = (S) request.replaceUri(finalUri);
                    try {
                        return Observable.just(AbstractLoadBalancerAwareClient
                                .this.execute(requestForServer, requestConfig));
                    } // catch ......
                }
            })
            .toBlocking()
            .single();
    } // catch ......
}
```

进入到方法体之前，先看一眼此时的 request ：

![](./img/2023/05/cloud16-6.png)

还是封装好的，内部有一个默认的 Feign 客户端。

下面咱分解这部分源码：

### 3.1 buildLoadBalancerCommand

```java
protected LoadBalancerCommand<T> buildLoadBalancerCommand(final S request, final IClientConfig config) {
    RequestSpecificRetryHandler handler = getRequestSpecificRetryHandler(request, config);
    LoadBalancerCommand.Builder<T> builder = LoadBalancerCommand.<T>builder()
            .withLoadBalancerContext(this)
            .withRetryHandler(handler)
            .withLoadBalancerURI(request.getUri());
    customizeLoadBalancerCommandBuilder(request, config, builder);
    return builder.build();
}
```

这里它将请求的内容和一些参数配置都放入了一个 `LoadBalancerCommand` 中，Debug返回后发现这里面已经具备了必要的属性和一些配置值，注意这里面 `loadBalancerContext` 的类型为 `FeignLoadBalancer` ，也就说明接下来的负载均衡是由 Feign 控制完成（Feign借助Ribbon）。

![](./img/2023/05/cloud16-7.png)

### 3.2 submit

必要的参数都构建好，接下来要进入一段特别长且复杂的源码，小册这里不把所有的内容都贴出来了，咱只抽取出最核心的部分：（已调整代码缩进，源码真的太复杂而且缩进超大）

```java
public Observable<T> submit(final ServerOperation<T> operation) {
    // ......
    Observable<T> o = 
        // 3.3 选择目标服务实例
        (server == null ? selectServer() : Observable.just(server))
        .concatMap(new Func1<Server, Observable<T>>() {
            @Override
            // Called for each server being selected
            public Observable<T> call(Server server) {
                context.setServer(server);
                final ServerStats stats = loadBalancerContext.getServerStats(server);

                // Called for each attempt and retry
                Observable<T> o = Observable
                    .just(server)
                    .concatMap(new Func1<Server, Observable<T>>() {
                        @Override
                        public Observable<T> call(final Server server) {
                            // ......
                            // 3.4 根据目标服务实例信息构造真实请求的url
                            return operation.call(server).doOnEach(new Observer<T>() {
                                // ......
    // ......
}

```

这里面我只留下了两撮关键的源码内容，上面的 `selectServer` 动作是负载均衡获取服务实例，下面的 `operation.call(server)` 用来完成 url 重构。

### 3.3 【Ribbon】selectServer

```java
private Observable<Server> selectServer() {
    return Observable.create(new OnSubscribe<Server>() {
        @Override
        public void call(Subscriber<? super Server> next) {
            try {
                Server server = loadBalancerContext.getServerFromLoadBalancer(loadBalancerURI, loadBalancerKey);   
                next.onNext(server);
                next.onCompleted();
            } // catch ......
        }
    });
}
```

这里它会使用 `loadBalancerContext` 去取具体的服务实例，进入 `getServerFromLoadBalancer` 方法：（只截取核心源码）

```java
public Server getServerFromLoadBalancer(@Nullable URI original, @Nullable Object loadBalancerKey) throws ClientException {
    // ......

    ILoadBalancer lb = getLoadBalancer();
    if (host == null) {
        if (lb != null){
            Server svc = lb.chooseServer(loadBalancerKey);
            // ......
}
```

可以发现核心动作还是 **Ribbon 负载均衡器的 `chooseServer` 方法**！至此可以发现 Ribbon 起作用的位置。

### 3.4 Feign配合Ribbon完成url重构

注意上面我留下来的源码最后一句，在 return 中它执行了 `operation` 的 `call` 方法，而这个 `operation` 就是上面 `executeWithLoadBalancer` 方法中传入的匿名内部类：

```java
new ServerOperation<T>() {
        @Override
        public Observable<T> call(Server server) {
            URI finalUri = reconstructURIWithServer(server, request.getUri());
            S requestForServer = (S) request.replaceUri(finalUri);
            try {
                return Observable.just(AbstractLoadBalancerAwareClient
                        .this.execute(requestForServer, requestConfig));
            } // catch ......
        }
```

这部分实现中，先是重新构造好 url ，再将构造好的 url 放入请求中，最后发起请求。将断点打在 call 方法中，咱来看 Feign 是如何拿 Ribbon 选择好的服务实例完成真实请求 url 的替换。Debug来到 `FeignLoadBalancer` ：

```java
public URI reconstructURIWithServer(Server server, URI original) {
    URI uri = updateToSecureConnectionIfNeeded(original, this.clientConfig,
            this.serverIntrospector, server);
    return super.reconstructURIWithServer(server, uri);
}

// RibbonUtils
public static URI updateToSecureConnectionIfNeeded(URI uri, IClientConfig config,
        ServerIntrospector serverIntrospector, Server server) {
    String scheme = uri.getScheme();

    if (StringUtils.isEmpty(scheme)) {
        scheme = "http";
    }

    if (!StringUtils.isEmpty(uri.toString())
            && unsecureSchemeMapping.containsKey(scheme)
            && isSecure(config, serverIntrospector, server)) {
        return upgradeConnection(uri, unsecureSchemeMapping.get(scheme));
    }
    return uri;
}
```

在 `FeignLoadBalancer` 中的额外处理仅仅是处理下请求协议，它会将 http 更改为 https ，当然如果要请求的服务不需要建立安全连接，那也不必。咱搭建的肯定都是 http 就可以的，这部分直接忽略就好。进入到父类的 `reconstructURIWithServer` 方法：

```java
public URI reconstructURIWithServer(Server server, URI original) {
    String host = server.getHost();
    int port = server.getPort();
    String scheme = server.getScheme();
    
    if (host.equals(original.getHost()) 
            && port == original.getPort()
            && scheme == original.getScheme()) {
        return original;
    }
    if (scheme == null) {
        scheme = original.getScheme();
    }
    if (scheme == null) {
        scheme = deriveSchemeAndPortFromPartialUri(original).first();
    }

    try {
        StringBuilder sb = new StringBuilder();
        sb.append(scheme).append("://");
        if (!Strings.isNullOrEmpty(original.getRawUserInfo())) {
            sb.append(original.getRawUserInfo()).append("@");
        }
        sb.append(host);
        if (port >= 0) {
            sb.append(":").append(port);
        }
        sb.append(original.getRawPath());
        if (!Strings.isNullOrEmpty(original.getRawQuery())) {
            sb.append("?").append(original.getRawQuery());
        }
        if (!Strings.isNullOrEmpty(original.getRawFragment())) {
            sb.append("#").append(original.getRawFragment());
        }
        URI newURI = new URI(sb.toString());
        return newURI;            
    } // catch ......
}
```

好的又是熟悉的配方熟悉的味道，字符串拼接的操作咱就不多解释了吧，Debug来看一眼最后拼出来的结果：

![](./img/2023/05/cloud16-8.png)

已经正常拼接好 url ，可以发起请求了。

## 4. 发起请求

与 Ribbon 不同，Feign 的请求发起不再借助 `RestTemplate` ，而是在 `FeignLoadBalancer` 中执行。回到 `executeWithLoadBalancer` 方法中，中间的 try 块中只有一行代码：

```java
return Observable.just(AbstractLoadBalancerAwareClient.this.execute(requestForServer, requestConfig));
```

就是在这个位置真正发送了请求。咱来到 `execute` 方法：

```java
public RibbonResponse execute(RibbonRequest request, IClientConfig configOverride)
        throws IOException {
    Request.Options options;
    if (configOverride != null) {
        RibbonProperties override = RibbonProperties.from(configOverride);
        options = new Request.Options(override.connectTimeout(this.connectTimeout),
                override.readTimeout(this.readTimeout));
    }
    else {
        options = new Request.Options(this.connectTimeout, this.readTimeout);
    }
    Response response = request.client().execute(request.toRequest(), options);
    return new RibbonResponse(request.getUri(), response);
}
```

上面的参数获取咱就不关心了，倒数第二行它取出 `RibbonRequest` 中真正的请求，去执行请求的发起：

```java
public Response execute(Request request, Options options) throws IOException {
    HttpURLConnection connection = convertAndSend(request, options);
    return convertResponse(connection, request);
}
```

到这里，应该能冥冥之中意识到一个事情：它该不会用的 jdk 最底层的方式去请求吧。跳转到 `convertAndSend` 方法，发现果然如此：

```java
HttpURLConnection convertAndSend(Request request, Options options) throws IOException {
        final URL url = new URL(request.url());
        final HttpURLConnection connection = this.getConnection(url);
        // ......

        if (request.requestBody().asBytes() != null) {
            // ......
            OutputStream out = connection.getOutputStream();
            // ......
        }
        return connection;
    }
}
```

至此咱也就完整走了一遍 Feign 的调用全流程。

## 小结

1. Feign 进行远程调用的核心是 jdk 动态代理中的一个 `InvocationHandler`；
2. Feign 进行负载均衡调用时，底层还是使用 Ribbon 的API；
3. Feign 的远程调用使用 jdk 自带的网络请求，Ribbon 的使用方式是增强 `RestTemplate` 。

【Feign 的调用全流程咱已经过了一遍，但有没有注意一个问题：这是一个 jdk 动态代理中的 `InvocationHandler` ，具体的 Feign 创建出来的接口实现类呢？下一章咱来深入探究 Feign 创建的接口实现类及相关机制】

