---
title: 09-服务发现-EurekaServer的缓存机制原理
toc: true
tags: springcloud
categories: 
    - [java]
    - [springcloudNetflix]
---

Eureka 之所以是一个遵循 AP 原则的注册中心，内部设计的缓存结构肯定是不可或缺的，有了缓存、自我保护机制等特性，使得 EurekaServer 可以在集群中一些节点宕机时还可以让整体继续提供服务，而不至于因为像 Zookeeper 那样进行 Leader 的重新选举导致集群不可用。本章咱来详细看看 EurekaServer 是如何设计内部缓存的（其实上一章留了个 EurekaServer 的坑，正好这一章填上）。

## 0. 书接上回-EurekaServer处理注册信息获取请求

<!--more-->

来到 `ApplicationsResource` 类的 `getContainers` 方法：

```java
@GET                       // 参数太多已省略
public Response getContainers(.............) {
    // ......
    // 响应缓存
    KeyType keyType = Key.KeyType.JSON;
    String returnMediaType = MediaType.APPLICATION_JSON;
    if (acceptHeader == null || !acceptHeader.contains(HEADER_JSON_VALUE)) {
        keyType = Key.KeyType.XML;
        returnMediaType = MediaType.APPLICATION_XML;
    }

    Key cacheKey = new Key(Key.EntityType.Application,
            ResponseCacheImpl.ALL_APPS,
            keyType, CurrentRequestVersion.get(), EurekaAccept.fromString(eurekaAccept), regions
    );

    // 从缓存中取注册信息
    Response response;
    if (acceptEncoding != null && acceptEncoding.contains(HEADER_GZIP_VALUE)) {
        // ......
    } else {
        response = Response.ok(responseCache.get(cacheKey)).build();
    }
    return response;
}
```


