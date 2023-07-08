---
title: 012-负载均衡-Ribbon的调用流程原理
toc: true
tags: springcloud
categories: 
    - [java]
    - [springcloudNetflix]
---

上一章咱通过看 Ribbon 的相关自动配置类，了解了 Ribbon 的几个核心组件，这一章咱通过前面搭建好的测试环境来实际体会下 Ribbon 处理负载均衡的全流程原理。

<!--more-->

上次搭建的环境如下：

![](./img/2023/05/cloud11-1.png)

eureka-client 服务中有一个暴露出的 Rest 服务：

```java
@RestController
public class ProviderController {
    @Value("${eureka.instance.instance-id}")
    private String zone;
    
    @GetMapping("/getInfo")
    public String getInfo() {
        return zone;
    }
}
```

eureka-consumer 服务会调用 eureka-client 的服务：

```java
@RestController
public class ConsumerController {
    @Autowired
    private RestTemplate restTemplate;
    
    @GetMapping("/getInfo")
    public String getInfo() {
        return restTemplate.getForObject("http://eureka-client/getInfo", String.class);
    }
}
```

这里要额外加一个配置，咱把服务的默认负载均衡策略改为轮询（先避开原有的带分区的策略，分区的内容咱放到下一章解释）：

```java
eureka-client.ribbon.NFLoadBalancerRuleClassName=com.netflix.loadbalancer.RoundRobinRule
```

下面咱开始实际的测试流程。

## 1. 请求发起

请求到达 `ConsumerController` 的 `getInfo` 方法，随后执行到 `RestTemplate` 内部：

### 1.1 进入RestTemplate

```java
	@Override
	@Nullable
	public <T> T getForObject(String url, Class<T> responseType, Object... uriVariables) throws 
    RestClientException {
		RequestCallback requestCallback = acceptHeaderRequestCallback(responseType);
		HttpMessageConverterExtractor<T> responseExtractor =
				new HttpMessageConverterExtractor<>(responseType, getMessageConverters(), logger);
		return execute(url, HttpMethod.GET, requestCallback, responseExtractor, uriVariables);
	}
```

随后继续向下执行，到达真正干活的 `doExecute` 方法：

```java
	@Override
	@Nullable
	public <T> T execute(String url, HttpMethod method, @Nullable RequestCallback requestCallback,
			@Nullable ResponseExtractor<T> responseExtractor, Object... uriVariables) throws RestClientException {

		URI expanded = getUriTemplateHandler().expand(url, uriVariables);
		return doExecute(expanded, method, requestCallback, responseExtractor);
	}
```

### 1.2 doExecute实际构建和发送请求

```java
protected <T> T doExecute(URI url, @Nullable HttpMethod method, @Nullable RequestCallback requestCallback,
        @Nullable ResponseExtractor<T> responseExtractor) throws RestClientException {
    // assert
    ClientHttpResponse response = null;
    try {
        ClientHttpRequest request = createRequest(url, method);
        if (requestCallback != null) {
            requestCallback.doWithRequest(request);
        }
        response = request.execute();
        handleResponse(url, method, response);
        return (responseExtractor != null ? responseExtractor.extractData(response) : null);
    }
```

注意此时的 `request` 中内容还是未经过负载均衡的：

![](./img/2023/05/cloud12-1.png)

发送请求那自然是调 `request.execute()` 方法，进入到 `AbstractClientHttpRequest` 的内部，经过几番辗转，最终来到 `InterceptingClientHttpRequest` 的内部类 `InterceptingRequestExecution` 中：

```java
    public ClientHttpResponse execute(HttpRequest request, byte[] body) throws IOException {
        if (this.iterator.hasNext()) {
            ClientHttpRequestInterceptor nextInterceptor = this.iterator.next();
            return nextInterceptor.intercept(request, body, this);
        } // else ......
    }
```

注意它在前面要先执行请求中自带的拦截器，Debug发现确实有一个拦截器，且就是前面咱看到的 `LoadBalancerInterceptor` 

![](./img/2023/05/cloud12-2.png)

## 2. 拦截器触发

进入到 `LoadBalancerInterceptor` 的内部，执行 `intercept` 方法：

```java
public ClientHttpResponse intercept(final HttpRequest request, final byte[] body,
        final ClientHttpRequestExecution execution) throws IOException {
    final URI originalUri = request.getURI();
    String serviceName = originalUri.getHost();
    Assert.state(serviceName != null,
            "Request URI does not contain a valid hostname: " + originalUri);
    return this.loadBalancer.execute(serviceName,
            this.requestFactory.createRequest(request, body, execution));
}
```

Debug发现在没有进入 `loadBalancer` 前还是原样的请求：

![](./img/2023/05/cloud12-3.png)

咱直接进入 `execute` 方法吧：

```java
public <T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException {
    return execute(serviceId, request, null);
}

public <T> T execute(String serviceId, LoadBalancerRequest<T> request, Object hint) throws IOException {
    ILoadBalancer loadBalancer = getLoadBalancer(serviceId);
    Server server = getServer(loadBalancer, hint);
    if (server == null) {
        throw new IllegalStateException("No instances available for " + serviceId);
    }
    RibbonServer ribbonServer = new RibbonServer(serviceId, server,
            isSecure(server, serviceId), serverIntrospector(serviceId).getMetadata(server));
    return execute(serviceId, ribbonServer, request);
}
```

最终执行到下面的 `execute` 方法中，这里面有几个动作咱看看。

### 2.1 获取ILoadBalancer

```java
protected ILoadBalancer getLoadBalancer(String serviceId) {
    return this.clientFactory.getLoadBalancer(serviceId);
}
```

这部分逻辑咱在上一章中看过了，它会从IOC容器中取类型为 `ILoadBalancer` 的Bean，默认情况下会取出一个类型为 `ZoneAwareLoadBalancer` 的负载均衡器，之后返回出去。

### 2.2 获取Server

取到 `BaseLoadBalancer` 后要进入 `getServer` 方法：

```java
protected Server getServer(ILoadBalancer loadBalancer, Object hint) {
    if (loadBalancer == null) {
        return null;
    }
    // Use 'default' on a null hint, or just pass it on?
    return loadBalancer.chooseServer(hint != null ? hint : "default");
}
```

很明显它直接拿上一步获取到的负载均衡器来选择要请求的服务实例，此时的 `loadBalancer` 类型为上面取出来的 `ZoneAwareLoadBalancer` ，是带区域概念的负载均衡器，不过目前咱用不到它，可以先进来看一眼：

```java
public Server chooseServer(Object key) {
    if (!ENABLED.get() || getLoadBalancerStats().getAvailableZones().size() <= 1) {
        logger.debug("Zone aware logic disabled or there is only one zone");
        return super.chooseServer(key);
    }
    // ......
```

Debug发现它直接进入到 `super.chooseServer(key)` 了，那后续的逻辑咱也不用看了，直接进入到 `BaseLoadBalancer` 中：

```java
public Server chooseServer(Object key) {
    if (counter == null) {
        counter = createCounter();
    }
    counter.increment();
    if (rule == null) {
        return null;
    } else {
        try {
            return rule.choose(key);
        } // catch ......
    }
}
```

### 2.3 根据负载均衡算法获取目标服务

由于上面咱显式的配置了负载均衡策略为 `RoundRobinRule` ，故直接跳转到它的 `choose` 方法中：（核心注释已标注在源码中）

```java
public Server choose(ILoadBalancer lb, Object key) {
    // null check ......
    Server server = null;
    int count = 0;
    // 最多尝试执行10次轮询（考虑到有线程并发没有获取到的可能）
    while (server == null && count++ < 10) {
        // 获取所有可访问的服务
        List<Server> reachableServers = lb.getReachableServers();
        List<Server> allServers = lb.getAllServers();
        int upCount = reachableServers.size();
        int serverCount = allServers.size();

        // 只要没有可访问的服务，直接返回null
        if ((upCount == 0) || (serverCount == 0)) {
            log.warn("No up servers available from load balancer: " + lb);
            return null;
        }

        // 使用incrementAndGetModulo方法决定出轮询的下标位置并取出
        int nextServerIndex = incrementAndGetModulo(serverCount);
        server = allServers.get(nextServerIndex);

        // 如果没有取出，重试
        if (server == null) {
            /* Transient. */
            Thread.yield();
            continue;
        }

        // 恰巧取出的服务又不可用，重试
        if (server.isAlive() && (server.isReadyToServe())) {
            return (server);
        }

        // Next.
        server = null;
    }

    // 如果重试次数超过10次，打印警告日志，返回null
    if (count >= 10) {
        log.warn("No available alive servers after 10 tries from load balancer: " + lb);
    }
    return server;
}
```

整体逻辑的复杂度不算很高，咱重点关注轮询的核心方法 `incrementAndGetModulo` ：

```java
private AtomicInteger nextServerCyclicCounter;

private int incrementAndGetModulo(int modulo) {
    for (;;) {
        int current = nextServerCyclicCounter.get();
        int next = (current + 1) % modulo;
        if (nextServerCyclicCounter.compareAndSet(current, next)) {
            return next;
        }
    }
}
```

可见它的设计还是很简单的，轮询实质上就是取模运算，它借助 `AtomicInteger` 的目的也很明确，保证线程安全，每次轮询到的服务不会撞车。

### 2.4 回到RibbonLoadBalancerClient

返回具体的 Server 后，在 `RibbonLoadBalancerClient` 中将这些组件构建为一个 `RibbonServer` ，传入下面的 `execute` 方法。

```java
public <T> T execute(String serviceId, LoadBalancerRequest<T> request, Object hint) throws IOException {
    // .....
    RibbonServer ribbonServer = new RibbonServer(serviceId, server,
            isSecure(server, serviceId), serverIntrospector(serviceId).getMetadata(server));
    return execute(serviceId, ribbonServer, request);
}
```

下面的 `execute` 方法就是真正发送请求的动作了。

## 3. execute发送请求

核心注释已标注在源码中：

```java
public <T> T execute(String serviceId, ServiceInstance serviceInstance,
        LoadBalancerRequest<T> request) throws IOException {
    // 如果传入的是RibbonServer，会取出内部包装的真实的Server
    Server server = null;
    if (serviceInstance instanceof RibbonServer) {
        server = ((RibbonServer) serviceInstance).getServer();
    }
    if (server == null) {
        throw new IllegalStateException("No instances available for " + serviceId);
    }

    // 调用记录跟踪
    RibbonLoadBalancerContext context = this.clientFactory
            .getLoadBalancerContext(serviceId);
    RibbonStatsRecorder statsRecorder = new RibbonStatsRecorder(context, server);

    try {
        // 3.1 发送请求
        T returnVal = request.apply(serviceInstance);
        statsRecorder.recordStats(returnVal);
        return returnVal;
    } // catch ......
    return null;
}

```

前面的准备工作大概内容也都比较简单，try 中的 request.apply 是真正发送请求的位置，咱进入来看：

### 3.1 request.apply

```java
public LoadBalancerRequest<ClientHttpResponse> createRequest(
        final HttpRequest request, final byte[] body,
        final ClientHttpRequestExecution execution) {
    return instance -> {
        // 3.1.1 包装原始请求
        HttpRequest serviceRequest = new ServiceRequestWrapper(request, instance, this.loadBalancer);
        if (this.transformers != null) {
            // 3.1.2 转换请求
            for (LoadBalancerRequestTransformer transformer : this.transformers) {
                serviceRequest = transformer.transformRequest(serviceRequest, instance);
            }
        }
        // 3.2 发送请求
        return execution.execute(serviceRequest, body);
    };
}
```

注意进入的位置是上一章3.2节看到的 `LoadBalancerRequestFactory` ！回头注意看拦截器的最后一句：

`loadBalancer.execute(serviceName, this.requestFactory.createRequest(request, body, execution));`

它调用 `requestFactory` 的 `createRequest` 方法创建了实际用来发起的请求，而这个方法返回的就是上面的这段源码！咱看看它都干了什么。

#### 3.1.1 ServiceRequestWrapper包装原始请求

这个 `ServiceRequestWrapper` 类从类名上就可以很容易理解，它就是将原有的类包装了一下（ 联想 SpringFramework 中的 `BeanWrapper` ），这里面它在 `getURI` 方法上动了下手脚，咱下面就可以看到了。

#### 3.1.2 转换请求

这部分会拿所有的 `LoadBalancerRequestTransformer` 对包装过的请求再进一步的转换，Debug发现这部分根本就没有转换器，故直接忽略。

### 3.2 发送请求

到了真正的发送请求了，咱来到 `ClientHttpRequestExecution` 中看 `execute` 的逻辑：

```java
public ClientHttpResponse execute(HttpRequest request, byte[] body) throws IOException {
    if (this.iterator.hasNext()) {
        ClientHttpRequestInterceptor nextInterceptor = this.iterator.next();
        return nextInterceptor.intercept(request, body, this);
    } else {
        HttpMethod method = request.getMethod();
        Assert.state(method != null, "No standard HTTP method");
        // 3.2.1 & 3.2.2 重写uri
        ClientHttpRequest delegate = requestFactory.createRequest(request.getURI(), method);
        request.getHeaders().forEach((key, value) -> delegate.getHeaders().addAll(key, value));
        if (body.length > 0) {
            if (delegate instanceof StreamingHttpOutputMessage) {
                StreamingHttpOutputMessage streamingOutputMessage = (StreamingHttpOutputMessage) delegate;
                streamingOutputMessage.setBody(outputStream -> StreamUtils.copy(body, outputStream));
            }
            else {
                StreamUtils.copy(body, delegate.getBody());
            }
        }
        return delegate.execute();
    }
}
```

等等，感觉似曾相识？刚才在第2节不是一上来就进入这里了吗？Debug发现刚才 `iterator` 中的拦截器就是 Ribbon 的拦截器！所以这个地方是上一个拦截器执行完毕了，可以继续往下执行了。Debug发现只有一个拦截器，所以进入 else 部分。else 的第三行又要创建 `ClientHttpRequest` ，注意这个地方它调用了 `getURI` 方法，触发了上面 `ServiceRequestWrapper` 的包装对象重写的方法，那进入到 `ServiceRequestWrapper` 的内部实现去：

```java
public URI getURI() {
    URI uri = this.loadBalancer.reconstructURI(this.instance, getRequest().getURI());
    return uri;
}
```

它借助负载均衡器重新构建了 uri ，那咱就深入进去看一眼是怎么构建的。

#### 3.2.1 RibbonLoadBalancerClient的重写uri

核心注释已标注在源码中：

```java
public URI reconstructURI(ServiceInstance instance, URI original) {
    Assert.notNull(instance, "instance can not be null");
    // 取出服务名称
    String serviceId = instance.getServiceId();
    RibbonLoadBalancerContext context = this.clientFactory.getLoadBalancerContext(serviceId);

    URI uri;
    Server server;
    // 如果服务实例是被Ribbon封装过的，要取出内部真实的服务实例
    if (instance instanceof RibbonServer) {
        RibbonServer ribbonServer = (RibbonServer) instance;
        server = ribbonServer.getServer();
        uri = updateToSecureConnectionIfNeeded(original, ribbonServer);
    } else {
        server = new Server(instance.getScheme(), instance.getHost(), instance.getPort());
        IClientConfig clientConfig = clientFactory.getClientConfig(serviceId);
        ServerIntrospector serverIntrospector = serverIntrospector(serviceId);
        uri = updateToSecureConnectionIfNeeded(original, clientConfig,
                serverIntrospector, server);
    }
    // 借助RibbonLoadBalancerContext重新构建uri
    return context.reconstructURIWithServer(server, uri);
}
```

一路走过来，最底下的 return 部分才是核心的构建逻辑，那咱继续往里跳：

#### 3.2.2 RibbonLoadBalancerContext的重写uri

这部分逻辑相对就简单多了，而且很容易理解，源码的关键部分已标注注释：

```java
public URI reconstructURIWithServer(Server server, URI original) {
    // 获取服务实例的IP、端口号、请求协议类型
    String host = server.getHost();
    int port = server.getPort();
    String scheme = server.getScheme();
    
    if (host.equals(original.getHost()) && port == original.getPort()
            && scheme == original.getScheme()) {
        return original;
    }
    // 如果服务实例中没有标注请求协议，则使用uri中的
    if (scheme == null) {
        scheme = original.getScheme();
    }
    if (scheme == null) {
        scheme = deriveSchemeAndPortFromPartialUri(original).first();
    }

    try {
        // 字符串拼接真正的请求uri
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

上述源码可以很清晰明确地把实际调用的 uri 拼接，并返回给 `requestFactory` 。

注意此时的 `RequestFactory` 就不再是 Ribbon 里头那个特殊的工厂了，使用 `RestTemplate` 原本使用的普通工厂就可以创建并发送请求了。

至此，Ribbon 的一次完整的负载均衡调用流程解析完毕。

## 小结

1. Ribbon 实现负载均衡的核心是借助 `LoadBalancerInterceptor` ；
2. Ribbon 实现负载均衡的核心逻辑是通过负载均衡算法，将应该调用的服务实例取出并替换原有的请求 uri 。

【大致对 Ribbon 的整个负载均衡调用机制有一个认识后，其实 Ribbon 也就没多少东西了。下一章咱介绍 Ribbon 中内置的几种常见的负载均衡策略，以及前面 Eureka 中留下的最后一部分：服务分区】



