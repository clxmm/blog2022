---
title: 37整合监控-springboot的生成级特性
toc: true
tags: springboot
categories: 
    - [java]
    - [springboot]
---

了解完了前面那么多的整合场景，最后我们来研究一下生产环境下 SpringBoot 的强大之处。之所以 SpringBoot 能如此流行，其内部支持的监控指标绝对是不可或缺的一个点。小册的最后两章，我们就一起来看看 SpringBoot 都给我们提供了哪些生产级别的特性。

## 1. actuator

---

本身在大家学习 SpringBoot 的时候，大概率已经了解过 actuator 这个组件了，只需要在 pom 文件中导入 actuator 的启动器，就可以获得生产级别的指标监控和信息获取等。

<!--more-->


```xml
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
```

> 注意这个 actuator 启动器中还引入了一个 `micrometer-core` 的依赖，它的作用后面我们会看到。

下面我们先快速使用和回顾一下 actuator 中给我们提供的特性。

#### 1.1 开启使用

我们来创建一个新的工程 `spring-boot-integration-11-actuator` ，并引入 web 和actuator 的依赖，这样就整合了 actuator 指标监控了。

默认情况下 actuator 给我们开启的内容很少（考虑到默认导入时的场景），它只给我们暴露了 health 相关的端点，我们可以先启动工程，访问 [http://localhost:8080/actuator](http://localhost:8080/actuator) ，看一看此时给我们提供的信息都有什么。

```json
{
  "_links": {
    "self": {
      "href": "http://localhost:8080/actuator",
      "templated": false
    },
    "health": {
      "href": "http://localhost:8080/actuator/health",
      "templated": false
    },
    "health-path": {
      "href": "http://localhost:8080/actuator/health/{*path}",
      "templated": true
    }
  }
}
```

好家伙，这也太少了吧，只有这么几个端点很明显不够用吧！比方说我们想看看此时应用内部都有哪些 bean 对象、都有哪些缓存组件、日志组件的情况如何，Environment 中存储了什么内容，这些默认通通都有没有给我们提供！

> 其实默认情况下是有的，SpringBootActuator 给我们默认暴露了 JMX 方式的端点，我们可以使用 cmd 命令行运行 jconsole ，找到我们当前启用的 SpringBoot 应用进程，在 `org.springframework.boot` 的 Endpoint 中就可以找到 JMX 提供的端点：
>
> ![](./img/202303/37jiangk.png)

那如果我们使用 HTTP 的方式来访问这些端点该怎么办呢？我们需要在 `application.properties` 中添加两行配置，来开启 actuator 的所有配置：

```properties
management.endpoints.enabled-by-default=true
management.endpoints.web.exposure.include=*
```

之后我们重启工程，再次访问 [http://localhost:8080/actuator](http://localhost:8080/actuator) ，这次得到的端点信息是如下面所示的一个很大的 json 数据

```properties

  "_links": {
    "self": {
      "href": "http://localhost:8080/actuator",
      "templated": false
    },
    "beans": {
      "href": "http://localhost:8080/actuator/beans",
      "templated": false
    },
    "caches-cache": {
      "href": "http://localhost:8080/actuator/caches/{cache}",
      "templated": true
    },
    "caches": {
      "href": "http://localhost:8080/actuator/caches",
      "templated": false
    },
    "health": {
      "href": "http://localhost:8080/actuator/health",
      "templated": false
    },
    "health-path": {
      "href": "http://localhost:8080/actuator/health/{*path}",
      "templated": true
    },
    "info": {
      "href": "http://localhost:8080/actuator/info",
      "templated": false
    },
    "conditions": {
      "href": "http://localhost:8080/actuator/conditions",
      "templated": false
    },
    "shutdown": {
      "href": "http://localhost:8080/actuator/shutdown",
      "templated": false
    },
    "configprops": {
      "href": "http://localhost:8080/actuator/configprops",
      "templated": false
    },
    "configprops-prefix": {
      "href": "http://localhost:8080/actuator/configprops/{prefix}",
      "templated": true
    },
    "env": {
      "href": "http://localhost:8080/actuator/env",
      "templated": false
    },
    "env-toMatch": {
      "href": "http://localhost:8080/actuator/env/{toMatch}",
      "templated": true
    },
    "loggers": {
      "href": "http://localhost:8080/actuator/loggers",
      "templated": false
    },
    "loggers-name": {
      "href": "http://localhost:8080/actuator/loggers/{name}",
      "templated": true
    },
    "heapdump": {
      "href": "http://localhost:8080/actuator/heapdump",
      "templated": false
    },
    "threaddump": {
      "href": "http://localhost:8080/actuator/threaddump",
      "templated": false
    },
    "metrics-requiredMetricName": {
      "href": "http://localhost:8080/actuator/metrics/{requiredMetricName}",
      "templated": true
    },
    "metrics": {
      "href": "http://localhost:8080/actuator/metrics",
      "templated": false
    },
    "scheduledtasks": {
      "href": "http://localhost:8080/actuator/scheduledtasks",
      "templated": false
    },
    "mappings": {
      "href": "http://localhost:8080/actuator/mappings",
      "templated": false
    }
  }
}
```

好家伙，这么多端点吗！那紧接着问题就来了：SpringBoot 给我们提供了这么多的指标监控，那这些指标都有什么含义呢？我们应该关注哪些呢？

其实在 SpringBoot 的官方文档中 [docs.spring.io/spring-boot…](https://docs.spring.io/spring-boot/docs/2.5.14/reference/html/actuator.html#actuator.endpoints) ，它已经给我们都简单概述了每个端点都用来做什么，我们可以以其中几个比较常用和容易理解的端点来加以说明。

#### 1.2 health

health 意为“健康状态监测”，这个端点只会给我们返回一个数据：status ，这个数据的值要么是 UP 要么是 DOWN ，UP 则为正常状态，DOWN 则为宕机。

比方说我们刚搭建的应用，它的 status 值就应该是 UP ，因为所有组件都正常运转。但比方说这样，我们在 `pom.xml` 中引入 SpringDataRedis 的启动器，但是本地不启动 redis ，看看效果如何。

重启工程，访问 `/actuator/health` 后，SpringBoot 的内部会自动去检查 Redis 连接是否正常，然而我们本地并没有启动 Redis 实例，所以此时控制台会抛出连接失败的异常，对应浏览器上可以看到 status 变为 DOWN 。

![](./img/202303/37jk2.png)

#### 1.3 beans

beans 意为“组件信息”，它会将 IOC 容器中所有 bean 的基本信息都罗列出来，我们可以从中得知，当前运行的 SpringBoot 中都注册了哪些 bean 。比方说我们来访问一下 `/actuator/beans` ，会得到一个非常庞大的数据，我们只截取最开始的一小段为各位简单解读下：

```json
{
  "contexts": {
    "application": {
      "beans": {
        "endpointCachingOperationInvokerAdvisor": {
          "aliases": [],
          "scope": "singleton",
          "type": "org.springframework.boot.actuate.endpoint.invoker.cache.CachingOperationInvokerAdvisor",
          "resource": "class path resource [org/springframework/boot/actuate/autoconfigure/endpoint/EndpointAutoConfiguration.class]",
          "dependencies": [
            "org.springframework.boot.actuate.autoconfigure.endpoint.EndpointAutoConfiguration",
            "environment"
          ]
        },
        "defaultServletHandlerMapping": {
          "aliases": [],
          "scope": "singleton",
          "type": "org.springframework.web.servlet.HandlerMapping",
          "resource": "class path resource [org/springframework/boot/autoconfigure/web/servlet/WebMvcAutoConfiguration$EnableWebMvcConfiguration.class]",
          "dependencies": [
            "org.springframework.boot.autoconfigure.web.servlet.WebMvcAutoConfiguration$EnableWebMvcConfiguration"
          ]
        },

```

- `contexts.application.beans` 中的本质是一个 Map ，key 是每个 bean 的名称，value 是这个 bean 的一些基本信息
- 以 `defaultServletHandlerMapping` 为例
- `aliases` 代表 bean 的别名（在 `@Bean` 注解上可以标注，多数为空）
- `scope : singleton` 代表它是一个单实例 bean （同样还会有 prototype 、request 等域）
  - 由此可以看出它取的其实不是 bean 的信息，而是 `BeanDefinition`
- `type` 代表当前这个 bean 的类型
  - 注意这个类型可能不指定某个具体的类型（如果是使用 `@Bean` 注解注册的 bean ，则方法的返回值类型就是 bean 的类型）
- `resource` 代表这个 bean 的加载来源，可能是来自某个类的组件扫描，也可能是来自某个注解配置类，还有可能来自某个 xml 配置文件
- `dependencies` 代表这个类依赖的 bean ，从上面返回的数据中可以看出，它需要依赖 `WebMvcAutoConfiguration` 的内部类 `EnableWebMvcConfiguration`

#### 1.4 conditions

conditions 意为“自动装配的条件报告”，它可以给我们展示 SpringBoot 应用中每个自动装配是否生效，以及对应生效 / 不生效的依据。

我们也可以来访问一下看看，浏览器发送 `/actuator/conditions` 后，可以得到一个更大的 json 数据，我们只从这里面抽取几个相对容易看懂的简单看下即可。

```json
{
  "contexts": {
    "application": {
      "positiveMatches": {
        ......,
        "AopAutoConfiguration": [
          {
            "condition": "OnPropertyCondition",
            "message": "@ConditionalOnProperty (spring.aop.auto=true) matched"
          }
        ],
        "AopAutoConfiguration.ClassProxyingConfiguration": [
          {
            "condition": "OnClassCondition",
            "message": "@ConditionalOnMissingClass did not find unwanted class 'org.aspectj.weaver.Advice'"
          },
          {
            "condition": "OnPropertyCondition",
            "message": "@ConditionalOnProperty (spring.aop.proxy-target-class=true) matched"
          }
        ],
        ......
      }, 
      "negativeMatches": {
        "RabbitAutoConfiguration": {
          "notMatched": [
            {
              "condition": "OnClassCondition",
              "message": "@ConditionalOnClass did not find required class 'com.rabbitmq.client.Channel'"
            }
          ],
          "matched": []
        },
        ......,
      },
      "unconditionalClasses": [
        "org.springframework.boot.autoconfigure.context.ConfigurationPropertiesAutoConfiguration",
        "org.springframework.boot.actuate.autoconfigure.availability.AvailabilityHealthContributorAutoConfiguration",
        "org.springframework.boot.actuate.autoconfigure.info.InfoContributorAutoConfiguration",
        "org.springframework.boot.autoconfigure.context.PropertyPlaceholderAutoConfiguration",
        "org.springframework.boot.autoconfigure.context.LifecycleAutoConfiguration",
        "org.springframework.boot.actuate.autoconfigure.health.HealthContributorAutoConfiguration",
        "org.springframework.boot.actuate.autoconfigure.metrics.integration.IntegrationMetricsAutoConfiguration",
        "org.springframework.boot.actuate.autoconfigure.endpoint.EndpointAutoConfiguration",
        "org.springframework.boot.autoconfigure.availability.ApplicationAvailabilityAutoConfiguration",
        "org.springframework.boot.autoconfigure.info.ProjectInfoAutoConfiguration",
        "org.springframework.boot.actuate.autoconfigure.web.server.ManagementContextAutoConfiguration"
      ]
    }
  }
}
```

- `positiveMatches` ：条件装配生效的自动配置类
  - `AopAutoConfiguration` 生效的依据是一个 `@ConditionalOnProperty` 的属性配置条件，对应的配置 `spring.aop.auto=true`
  - `AopAutoConfiguration.ClassProxyingConfiguration` 生效的依据是 `@ConditionalOnMissingClass` 没有找到 `org.aspectj.weaver.Advice` 这个类，以及 `@ConditionalOnProperty` 的属性配置条件，对应的配置 `spring.aop.proxy-target-class=true`
- `negativeMatches` ：条件装配没有生效的自动配置类
  - `RabbitAutoConfiguration` 没有生效是因为 `@ConditionalOnClass` 没有找到 `com.rabbitmq.client.Channel` 这个类
- `unconditionalClasses` ：没有条件装配的自动配置类

#### 1.5 configprops

configprops 意为“配置属性”，它可以将 SpringBoot 中使用 `@EnableConfigurationProperties` 引用的那些配置属性对象都列举出来，给我们展示其中的配置属性和值。

访问 `/actuator/configprops` 后，我们会得到一个类似于 beans 的对象，只不过内部的那些 bean 大多都是以 Properties 结尾的：

```json
{
  "contexts": {
    "application": {
      "beans": {
        ......,
        "server-org.springframework.boot.autoconfigure.web.ServerProperties": {
          "prefix": "server",
          "properties": {
            "undertow": {
              "maxHttpPostSize": {},
              "eagerFilterInit": true,
              "maxHeaders": 200,
              "maxCookies": 200,
              "allowEncodedSlash": false,
              "decodeUrl": "true",
              "urlCharset": "UTF-8",
              "alwaysSetKeepAlive": true,
              "preservePathOnForward": false,
              "accesslog": ......,
              "threads": {},
              "options": {
                "socket": {},
                "server": {}
              }
            },
            "maxHttpHeaderSize": {},
            "tomcat": ......,
            "servlet": ......,
            "jetty": ......,
            "error": ......,
            "shutdown": "IMMEDIATE",
            "netty": ......
          },
          ......
        },
```

我们以各位比较熟悉的 `ServerProperties` 为例，可以看出来，它可以告诉我们每个配置属性对象对应 `application.properties` 中的配置属性前缀是什么，其中的配置属性都有什么，包括对应的值都能给我们列出来。

#### 1.6 env

env 意为 `Environment` 环境信息，如果小伙伴熟悉 `Environment` 的话，自然这个就很好理解了，它其实就是把 SpringBoot 应用中的那些 properties 属性都列出来（还包括 `systemProperties` 和 `systemEnvironment` ）。

访问 `/actuator/env` 后，可以发现返回的数据包含两部分：`activeProfiles` 和 `propertySources` ，分别代表当前激活的 profile 和所有的配置属性值。仔细看 `propertySources` 的内容可以发现，它除了包含 `systemProperties` 和 `systemEnvironment` 之外，通过 `ServletContext` 封装的初始化参数、`application.properties` 中的配置内容同样被列举进来了。

```json
{
  "activeProfiles": [],
  "propertySources": [
    {
      "name": "server.ports",
      "properties": {
        "local.server.port": {
          "value": 8080
        }
      }
    },
    {
      "name": "servletContextInitParams",
      "properties": {}
    },
    ......,
    {
      "name": "Config resource 'class path resource [application.properties]' via location 'optional:classpath:/'",
      "properties": {
        "management.endpoints.enabled-by-default": {
          "value": "true",
          "origin": "class path resource [application.properties] - 1:41"
        },
        "management.endpoints.web.exposure.include": {
          "value": "*",
          "origin": "class path resource [application.properties] - 2:43"
        }
      }
    }
  ]
}

```

#### 1.7 metrics

metrics 意为“指标”，这个东西相当有用，通过收集 metrics 信息，我们就可以知道当前应用的运行情况。

我们发送一个 `/actuator/metrics` 请求，发现目前 SpringBoot 给我们返回了一组看上去很像配置属性名的列表（而且不乏能看懂其中一些属性的名，比方说发送请求的数量、CPU 内核的数量和使用占比等）：

```json
{
  "names": [
    "http.server.requests",
    "jvm.buffer.count",
    "jvm.buffer.memory.used",
    "jvm.buffer.total.capacity",
    "jvm.classes.loaded",
    "jvm.classes.unloaded",
    "jvm.gc.live.data.size",
    "jvm.gc.max.data.size",
    "jvm.gc.memory.allocated",
    "jvm.gc.memory.promoted",
    "jvm.gc.pause",
    "jvm.memory.committed",
    "jvm.memory.max",
    "jvm.memory.used",
    "jvm.threads.daemon",
    "jvm.threads.live",
    "jvm.threads.peak",
    "jvm.threads.states",
    "logback.events",
    "process.cpu.usage",
    "process.start.time",
    "process.uptime",
    "system.cpu.count",
    "system.cpu.usage",
    "tomcat.sessions.active.current",
    "tomcat.sessions.active.max",
    "tomcat.sessions.alive.max",
    "tomcat.sessions.created",
    "tomcat.sessions.expired",
    "tomcat.sessions.rejected"
  ]
}
```

这些东西怎么使用呢？我们可以直接把其中某个指标名称复制过来，在 `/actuator/metrics` 请求后面直接追加，比方说我们要获取一下当前 CPU 的内核数量，那就可以访问 `/actuator/metrics/system.cpu.count` ，请求发送后，我们可以得到一个类似于下面格式的数据，这里面有指标的描述，指标的单位，当前指标的参数值等等。

```json
{
  "name": "system.cpu.count",
  "description": "The number of processors available to the Java virtual machine",
  "baseUnit": null,
  "measurements": [
    {
      "statistic": "VALUE",
      "value": 20.0
    }
  ],
  "availableTags": []
}
```

## 2. 自定义端点

---

虽说 SpringBoot 给我们提供的端点信息已经足够丰富，但实际项目开发中难免还是会出现需要我们扩展的情况。SpringBoot 2.x 相较于 SpringBoot 1.x ，它的扩展方式更简单、更容易操作，同时使用的 API 更容易被记住，下面我们根据端点扩展的机制，分别来了解。

#### 2.1 扩展health

我们在一开始访问 `/actuator/health` 接口时，得到的信息只有一个 `status` ，如果 `status` 只给我们返回一个 DOWN ，那我们就需要逐个排查项目内部整合的所有框架、数据库、中间件等。如何能让 health 信息表述的更详细呢？

##### 2.1.1 开启详细的health信息

其实 SpringBoot 考虑到了详细健康状态信息的输出，只是默认情况下不会开启罢了，我们只需要在 `application.properties` 中添加一行配置即可：

```properties
management.endpoint.health.show-details=always

```

添加好之后，只要是我们使用 SpringBoot 官方整合的场景启动器，对应的健康状态信息都会自动整合到响应结果中。

那我们就再次导入 SpringDataRedis 的场景启动器，并在本地启动一个 RedisServer 实例。随后，我们重启 SpringBoot 工程，浏览器访问 `/actuator/health` 接口，可以发现这次返回的信息中多了很多内容，除了 Redis 的信息之外，还有本地磁盘的信息：

```json
{
  "status": "UP",
  "components": {
    "diskSpace": {
      "status": "UP",
      "details": {
        "total": 274878951424,
        "free": 214643945472,
        "threshold": 10485760,
        "exists": true
      }
    },
    "ping": {
      "status": "UP"
    },
    "redis": {
      "status": "UP",
      "details": {
        "version": "3.0.504"
      }
    }
  }
}
```

可见 SpringBoot 帮我们考虑的真是非常周到了，简直贴心的不得了！

##### 2.1.2 第三方无法正常提供health信息？

那话又说回来，SpringBoot 官方整合的 OK ，但如果是第三方自己集成到 SpringBoot 的，估计就不会考虑这么周全了吧！比方说我们来整合一下之前封装的 `xxl-job` 的 starter ：

```xml
<dependency>
  <groupId>com.linkedbear.boot</groupId>
  <artifactId>xxl-job-boot-starter</artifactId>
  <version>1.0-SNAPSHOT</version>
</dependency>
```

对应的 `application.properties` 中也要配置几个核心参数：

```properties
xxl.job.admin-addresses=http://127.0.0.1:18899/xxl-job-admin
xxl.job.appname=demo-xxl-job-executor
xxl.job.address=http://127.0.0.1:9999
```

如此配置完毕后接下来我们依次启动 `xxl-job-admin` 和 SpringBoot 工程，到此为止都是之前章节的内容。

下面来考虑一个场景：当 `xxl-job-server` 意外宕机后，`xxl-job` 的定时任务部分将无法工作，如果我们假设这种情况下整个应用是无法正常对外提供服务，则对应的健康监测信息中 `status` 应为 **DOWN** ，但现实情况是怎样呢？

很可惜，当我们访问 `/actuator/health` 时，它返回的 `status` 依然是 **UP** ，并且没有返回 `xxl-job` 的信息，换句话说，即便 `xxl-job` 部分无法正常工作，但健康检查中还是认为服务是正常在线的，这显然不合理，所以我们需要对 `xxl-job` 的 health 信息进行扩展。

##### 2.1.3 自定义扩展health

如何扩展健康监测信息呢？SpringBoot 给我们提供了一个接口：`HealthIndicator` ，我们只需要编写这个接口的实现类（或者继承该接口的抽象实现 `AbstractHealthIndicator` ），并将其注册到 IOC 容器即可。

那我们就来创建一个针对 `xxl-job` 的健康检查监测器 `XxlJobHealthIndicator` ，需要注意一个小细节，`HealthIndicator` 的实现类中，前缀的名叫什么，最终反映到 health 端点中对应的监测属性名就叫什么（如我们起的名中，最终落实的属性名就叫 `xxlJob` ）。

```java
@Component
public class XxlJobHealthIndicator extends AbstractHealthIndicator {
    
    @Override
    protected void doHealthCheck(Health.Builder builder) throws Exception {
        
    }
}
```

继承 `AbstractHealthIndicator` 的话只需要重写 `doHealthCheck` 方法即可，方法的实现，需要我们考虑如何模拟 xxl-job 与 xxl-job-admin 之间的心跳。小册提供了一个实现方式，大家没有必要自己钻研如何编写，直接照抄即可。

```java
    @Autowired
    private XxlJobProperties prop;
    
    @Override
    protected void doHealthCheck(Health.Builder builder) throws Exception {
        RegistryParam registryParam = new RegistryParam(RegistryConfig.RegistType.EXECUTOR.name(), prop.getAppname(),
                prop.getAddress());
        
        List<AdminBiz> adminBizList = XxlJobExecutor.getAdminBizList();
        for (AdminBiz adminBiz : adminBizList) {
            ReturnT<String> ret = adminBiz.registry(registryParam);
            if (ret == null || ReturnT.SUCCESS_CODE != ret.getCode()) {
                builder.down();
                Field field = ReflectionUtils.findField(AdminBizClient.class, "addressUrl");
                ReflectionUtils.makeAccessible(field);
                String addressUrl = (String) ReflectionUtils.getField(field, adminBiz);
                builder.withDetail("errorAdminServer", addressUrl);
                return;
            }
        }
        builder.up();
    }
```

我们简单读一下这段代码的实现：`XxlJobHealthIndicator` 中注入了 `XxlJobProperties` ，目的是取出内部的 `appname` 和 `address` 属性；具体执行健康监测时，我们需要将 `XxlJobExecutor` 中组合的所有 `xxl-job-admin` 地址都取出来，逐个去发送心跳，如果所有心跳均成功发送，则认定当前服务是正常运行的；如果有一个调度服务端没有收到心跳，则认定当前服务宕机。

返回宕机（DOWN）信息的方法是直接调用 `Health.Builder` 的 `down` 方法，除了响应 DOWN 信息后，还可以调用 `builder` 的 `withDetail` 方法附加一条额外的信息，调用 `withDetails` 方法直接传入 `Map` 对象附加一组额外信息。

##### 2.1.4 测试效果

下面我们来试试效果如何。首先我们模拟服务正常的情况，先后启动 xxl-job-admin 和 SpringBoot 应用，随后访问 health 端点，可以收到一个类似于下面的信息，此时 `xxlJob` 模块的 status 是 UP ：

```json
{
  "status": "UP",
  "components": {
    "diskSpace": {
      "status": "UP",
      "details": {
        "total": 274878951424,
        "free": 214634860544,
        "threshold": 10485760,
        "exists": true
      }
    },
    "ping": {
      "status": "UP"
    },
    "redis": {
      "status": "UP",
      "details": {
        "version": "3.0.504"
      }
    },
    "xxlJob": {
      "status": "UP"
    }
  }
}
```

接下来，我们停止 `xxl-job-admin` 服务，随后再次访问 health 端点，在短暂的几秒钟等待后，我们会得到下面的数据，注意此时整个服务的 status 已经变味 DOWN ，`xxlJob` 中的 status 也变为 DOWN ，而且还附带出了我们在代码中附带的 `errorAdminServer` 信息，说明 health 的扩展信息已经成功制作。

```json
{
  "status": "DOWN",
  "components": {
    "diskSpace": {
      "status": "UP",
      "details": {
        "total": 274878951424,
        "free": 214634426368,
        "threshold": 10485760,
        "exists": true
      }
    },
    "ping": {
      "status": "UP"
    },
    "redis": {
      "status": "UP",
      "details": {
        "version": "3.0.504"
      }
    },
    "xxlJob": {
      "status": "DOWN",
      "details": {
        "errorAdminServer": "http://127.0.0.1:18899/xxl-job-admin/"
      }
    }
  }
}
```

#### 2.2 扩展info

除了 health 信息之外，info 信息也是经常被使用的。默认情况下 SpringBoot 不会给我们封装任何 info 信息，访问 `/actuator/info` 时只会得到一个空 json 。

扩展 info 信息的方式有两种，一种是直接在 `application.properties` 中声明：

```properties
info.appName=spring-boot-integration-11-actuator
# 注意使用@@符号可以引用pom中的属性
info.version=@project.version@
```

如此配置后，启动工程，访问 `/actuator/info` 端点即可得到我们配置的这些信息：

```json
{
  "appName": "spring-boot-integration-11-actuator",
  "version": "1.0.0-RELEASE"
}
```

但毕竟这样写还是太“硬”了些，SpringBoot 还给我们提供了一种编程式的扩展方式，那就是实现 `InfoContributor` 接口（注意不再是 Indicator 了），并在其 contribute 方法中设置即可。编写的方法与上面 Indicator 差不多，我们简单写一下就好。

```java
@Component
public class ApplicationInfoContributor implements InfoContributor {
    
    @Override
    public void contribute(Info.Builder builder) {
        builder.withDetail("appName", "spring-boot-integration-11-actuator-contributor");
        Map<String, Object> data = new HashMap<>();
        data.put("version", "1.0.0");
        builder.withDetails(data);
    }
}
```

如此编写完毕后，正好我们来测试一个细节：相同名称的 info 项，谁的优先级更高呢？重启应用，访问 `/actuator/info` 端点，可以发现拿到的信息都是 Contributor 的，说明编程式的优先级要更高。

```json
{
  "appName": "spring-boot-integration-11-actuator-contributor",
  "version": "1.0.0"
}
```

#### 2.3 扩展metrics

最后我们再来看一下扩展 metrics 的方式。从上面 1.7 节的内容可以看出，SpringBoot 默认给我们提供的指标还是不少的，但在实际项目开发中还是有可能要自己扩展一些监控指标，这就需要我们自定义 metrics 了。SpringBoot 给我们提供的扩展方式是利用一个 API ：`MeterRegistry` 。

下面我们通过一个简单的示例来演示一下。假设我们有一个业务接口，我们需要监控的是这个接口的调用次数。那么编码的方式，其实就是利用 `MeterRegistry` 的计数功能来逐次累加。

```java
@RestController
public class DemoController {
    
    @Autowired
    private MeterRegistry meterRegistry;
    
    private Counter counter;
    
    @PostConstruct
    public void init() {
        counter = meterRegistry.counter("demo.request");
    }
    
    @GetMapping("/demo")
    public String demo() {
        counter.increment();
        return "demo";
    }
}

```

注意看这段代码，`MeterRegistry` 这个对象来自于 micrometer ，这也就是本章最开始提到的那个，在 SpringBootActuator 引入时连带引入的包。在 `DemoController` 初始化时，首先会执行依赖注入，即获得了 `MeterRegistry` 对象；随后在 JSR-250 注解 `@PostConstruct` 的作用下，`init` 方法被触发，我们从 `MeterRegistry` 中得到了一个 `Counter` ，这个 `Counter` 就是一个可以逐次递增计数的工具。实际运行时，只要每次调用 /demo 接口，Counter 计数器就会工作，这样也就实现了指标收集。

如此编写完毕后，下面就可以启动测试了。我们启动工程，浏览器访问 `/actuator/metrics` 端口，可以发现返回的信息中多了我们写的那个 `"demo.request"` ；如果继续访问 `/actuator/metrics/demo.request` ，可以得到下面的一串 json 数据：

```json
{
  "name": "demo.request",
  "description": null,
  "baseUnit": null,
  "measurements": [
    {
      "statistic": "COUNT",
      "value": 0.0
    }
  ],
  "availableTags": []
}
```

可以发现，一开始这个指标的值是 0 ，下面我们就来访问几次 `/demo` 后，再来获取指标监控数据。

```json
{
  "name": "demo.request",
  "description": null,
  "baseUnit": null,
  "measurements": [
    {
      "statistic": "COUNT",
      "value": 5.0
    }
  ],
  "availableTags": []
}
```

很明显，这个 value 已经变化了，这就说明指标监控已经起了效果。

除了频数监控外，`MeterRegistry` 还支持聚合统计、耗时监控等，更多的使用方式，小伙伴们完全可以参照 [SpringBoot 的官方文档](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#actuator.metrics.registering-custom)，结合 micrometer 的使用文档来测试。

【可以发现，SpringBoot 给我们提供的生产级特性也是相当强大的，内置的内容丰富之外，还给我们提供了简便的方式扩展，这也使得 SpringBoot 能够稳稳地屹立于 Web 开发王者宝座之位。不过只使用这些接口还是不太方便，有没有更简单的工具来可视化展示呢？还真有，小册的最后一章，我们就来看一个工具：SpringBootAdmin 】