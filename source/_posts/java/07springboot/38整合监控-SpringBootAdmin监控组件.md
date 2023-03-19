---
title: 38整合监控-SpringBootAdmin监控组件.md
toc: true
tags: springboot
categories: 
    - [java]
    - [springboot]
---

上一章中，我们感受了 SpringBoot 的强大生产级特性，不过那种使用方式未免也太原始了些，如果要是有那种可视化工具，能够把这些数据展示出来，哪怕只是收集起来放在可视化页面上也好呀！

哎，还真有。SpringBootAdmin 就是一个不错的监控组件，虽然它不是 SpringBoot 官方提供的，但这个东西还挺实用，而且用的人也不少，所以本章我们就来把玩一下这个东西。

## 1. 使用SpringBootAdmin

SpringBootAdmin 的代码都托管在 GitHub 上：[github.com/codecentric…](https://github.com/codecentric/spring-boot-admin) ，并且在代码仓库的 README 中还给了一个 quick guide ，我们顺着走就可以找到 SpringBootAdmin 给我们提供的使用文档 [codecentric.github.io/spring-boot…](https://codecentric.github.io/spring-boot-admin/2.5.1/#getting-started) 。

<!--more-->

#### 1.1 AdminServer搭建

一上来的第一句话就告诉我们，要想用 SpringBootAdmin，需要我们先创建一个服务端 AdminServer ，由它做统一的监控数据采集和展示。所以首先我们要先创建一个 Maven 模块 `spring-boot-integration-11-admin` 。

引入的 pom 依赖中需要引入 WebMvc 的依赖，再就是 SpringBootAdminServer 的依赖（注意版本选择 2.5.6 ，与本小册使用的 SpringBoot 大版本保持一致）。

```xml
<dependencies>
    <dependency>
        <groupId>de.codecentric</groupId>
        <artifactId>spring-boot-admin-starter-server</artifactId>
        <version>2.5.6</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```



随后我们来编写一个 SpringBoot 主启动类 `SpringBootAdminApplication` ，并在启动类上标注一个 `@EnableAdminServer` 注解，代表当前项目充当 AdminServer 的角色。

```java
@EnableAdminServer
@SpringBootApplication
public class SpringBootAdminApplication {
    
    public static void main(String[] args) {
        SpringApplication.run(SpringBootAdminApplication.class, args);
    }
}
```

另外注意一点，本章我们要启动多个服务，所以我们需要配置一下端口号了，我们设定当前 AdminServer 的端口号是 8989 。

```properties
server.port=8989

```

这样我们的服务端就算搭建完毕了，我们可以访问一下 [http://localhost:8989](http://localhost:8989) ，会在浏览器上打开一个如下图所示的界面。

至于这个界面都有什么，我们下面用到哪些说哪些。

#### 1.2 普通服务连接AdminServer

有了服务端，自然也得有客户端，我们上一章创建的 `spring-boot-integration-11-actuator` 就完全可以当做一个客户端，入驻到 AdminServer 中。

> 注意一点，我们全程没有用到 SpringCloud 的东西，不会涉及到微服务相关的知识。

![](./img/202303/38admin.png)

服务端要想连接 AdminServer ，需要我们导入一个 `spring-boot-admin-starter-client` 的依赖：

```xml-dtd
    <dependency>
        <groupId>de.codecentric</groupId>
        <artifactId>spring-boot-admin-starter-client</artifactId>
        <version>2.5.6</version>
    </dependency>
```

对应的，我们需要配置一下当前应用的名称（就跟整合 SpringCloud 的时候必须要声明一样），以及连接 AdminServer 的地址、暴露真实 IP 地址。

```properties
spring.application.name=spring-boot-integration-11-actuator
spring.boot.admin.client.url=http://localhost:8989
spring.boot.admin.client.instance.prefer-ip=true
```

配置完毕后，下面就可以启动工程了。当工程启动后，回到 SpringBootAdmin 的服务端，可以发现这里面多了一个服务实例，而且刚好就是对应我们刚配置的这个！

![](./img/202303/38admin2.png)

#### 1.3 服务实例的信息

接下来的内容就有意思了，我们可以从 AdminServer 面板中获取到非常多的信息。下面一一来看。

##### 1.3.1 应用的基础信息

点开这个应用后，第一眼我们看到的就是下面图示的内容，这里面都是一个 SpringBoot 中最基础的那些信息，包含 info 的信息、health 的信息，以及下面线程的信息、CPU 内存磁盘等等。可以简单地说，这个“细节”面板中可以让我们对一个 SpringBoot 的服务实例有一个最基本的了解。

![](./img/202303/38admin3.png)

##### 1.3.2 性能

“性能”板块其实就是 metrics ，我们可以从中选择一些自己感兴趣的指标监控，添加到常驻面板上，这样就能得到实时的数据监控。比方说下面的图中我添加了 3 个指标，分别监控 `/demo` 接口调用的次数，以及 CPU 核心数、占用率。除此之外，各位是否观察到右侧还有一个数值选择框，我们可以根据监控的指标数据类型，让它展示的更合适。

![](./img/202303/38admin4.png)

##### 1.3.3 环境

“环境”其实对标的就是 `Environment` ，也就是我们在上一章中看到的那个 `/actuator/env` 端点返回的数据，只不过这里用可视化的界面帮我们展示出来了而已。

![](./img/202303/38admin5.png)

##### 1.3.4 类

“类”这个词翻译的不好，原文应该是 **beans** ，也就是 IOC 容器中注册的那些 bean 对象，它对应的是 `/actuator/beans` 端点返回的数据。

![](./img/202303/38admin6.png)

##### 1.3.5 配置属性

“配置属性”就是上一章中我们看到的那个 **configprop** ，它对应的是 `/actuator/configprop` 端点返回的数据。与上面的展示类似，它都是按照一个一个的 properties 类分组展示的，并没有什么多余的功能。

![](./img/202303/38admin7.png)

##### 1.3.6 日志配置

下面我们看看日志配置的内容，这个就很有意思了，因为默认我们没有配置的情况下，所有日志输出级别都是 INFO ，但是 SpringBootAdmin 可以让我们动态修改某个包下（甚至全局）的日志打印级别！这个功能非常有用，正常情况下我们在开发环境时可能使用 INFO 级别，在生产环境用 WARN 或者 ERROR 级别，而当出现问题需要调试时，会切换到 DEBUG 级别，每次切换都要重新启动工程，费时费力，SpringBootAdmin 直接给我们提供可视化切换面板了，即便上了生产环境，发现哪里有问题的时候，直接切换日志输出级别，配合日志收集分析，排查问题也容易的多。

![](./img/202303/38admin8.png)

其余的各个面板，感兴趣的小伙伴完全可以实操一下，都点点看看，这个东西还是有不少好玩的东西的。

【不光是单体应用可以对接，基于 SpringCloud 的微服务场景中 SpringBootAdmin 也完全 hold 的住，感兴趣的小伙伴可以继续参照 SpringBootAdmin 的官方文档来调试，因为本小册不涉及 SpringCloud 的内容，所以就不继续展开了】