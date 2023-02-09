---
title: 01动态生成world
toc: true
tags: springboot
categories: 
    - [java]
    - [springboot]
---

上一章中，我们回顾了单元测试以及 JUnit 框架，比较全面地了解了 JUnit 5 的使用。本章我们着重探讨一个问题：既然 SpringBoot 的测试类只需要标注一个 `@SpringBootTest` 注解，就可以驱动 SpringBoot 应用配合测试，SpringBoot 是如何做到的呢？


<!--more-->

## 1. @SpringBootTest的奥秘

借助 IDE 翻开 `@SpringBootTest` 注解的源码，可以发现这是一个复合注解，它组合了两个其他的注解：

```java
@BootstrapWith(SpringBootTestContextBootstrapper.class)
@ExtendWith(SpringExtension.class)
public @interface SpringBootTest
```

这两个注解我们暂时放下，稍后再来研究，我们先来看一下 `@SpringBootTest` 注解上的 javadoc ，从 javadoc 上可以获取到一些有用的信息。

> Annotation that can be specified on a test class that runs Spring Boot based tests. Provides the following features over and above the regular Spring TestContext Framework:
>
> - Uses SpringBootContextLoader as the default ContextLoader when no specific @ContextConfiguration(loader=...) is defined.
> - Automatically searches for a @SpringBootConfiguration when nested @Configuration is not used, and no explicit classes are specified.
> - Allows custom Environment properties to be defined using the properties attribute.
> - Allows application arguments to be defined using the args attribute.
> - Provides support for different webEnvironment modes, including the ability to start a fully running web server listening on a defined or random port.
> - Registers a TestRestTemplate and/or WebTestClient bean for use in web tests that are using a fully running web server.
>
> 可以在运行基于 Spring Boot 的测试的测试类上指定的注解。在常规 Spring TestContext Framework 之外提供以下功能：
>
> - 当没有定义特定的 `@ContextConfiguration(loader=...)` 时，使用 `SpringBootContextLoader` 作为默认的 `ContextLoader` 。
> - 不使用嵌套 `@Configuration` 时自动搜索 `@SpringBootConfiguration` ，并且未指定显式类。
> - 允许使用 properties 属性定义自定义环境属性。
> - 允许使用 args 属性定义应用程序参数。
> - 提供对不同 webEnvironment 模式的支持，包括启动在定义或随机端口上侦听的完全运行的 Web 服务器的能力。
>
> - 注册一个 `TestRestTemplate` 和/或 `WebTestClient` bean，以便在使用完全运行的 Web 服务器的 Web 测试中使用。

javadoc 中提示了几个关键信息，简单提取如下：

- 不指定 `@ContextConfiguration` 时默认使用 `SpringBootContextLoader` ，后者可能各位还不熟悉，但是前面的 `@ContextConfiguration` 注解想必做过 SpringFramework 单元测试的小伙伴都很眼熟了吧！通过使用 `@ContextConfiguration` 注解可以指定驱动 IOC 容器的 xml 配置文件 / 注解配置类，而 SpringBoot 中驱动单元测试同样需要可以初始化 IOC 容器的测试引导，默认使用的这个类即是文档注释中的 `SpringBootContextLoader` 。

- 自动搜索 `@SpringBootConfiguration` `@SpringBootConfiguration` 注解的作用是告诉 SpringBoot 的单元测试，主启动类在这里，拿它作为主配置源加载即可。

- 提供 webEnvironment 的支持，此处的 webEnvironment 指的是单元测试期间内部 Web 环境的提供方式，有以下 4 种类型可选：

  ```java
  enum WebEnvironment {
      
      // 提供一个可供Mock的Web环境，内置的Servlet容器并没有真实的启动，主要搭配使用@AutoConfigureMockMvc
      MOCK(false),
  
      // 提供一个真实的Web环境，内部的嵌入式Web容器会真实启动，并使用随机端口
      RANDOM_PORT(true),
  
      // 提供一个真实的Web环境，内部的嵌入式Web容器会真实启动，并使用配置文件中定义的端口
      DEFINED_PORT(true),
  
      // 不提供Web环境
      NONE(false);
  }
  ```

上述的分析也给我们引出了一些问题：

- `SpringBootContextLoader` 是什么？它是如何引导初始化 IOC 容器的？
- `@SpringBootConfiguration` 注解是如何被单元测试利用的？如何搜索到它？
- 使用不同的 WebEnvironment 类型，底层在启动单元测试时会有什么差异？

带着这些问题，我们开始深入 `@SpringBootTest` 注解中组合的两个注解进行分析。

## 2. @ExtendWith(SpringExtension.class)

`@ExtendWith` 这个注解，实质上可以简单理解为“等价于 JUnit 4 的 `@RunWith` ”，对应的 `SpringExtension` 来自 SpringFramework 5.0 ，它其实就是代替 `SpringRunner` 的（通过包名也能看得出来，它来自 SpringFramework 5 原生的测试包 `org.springframework.test.context.junit.jupiter` ）。

对于 `@ExtendWith` 与传入的 `SpringExtension` ，本身并没有什么可以研究的价值，我们只需要知道标注了它之后，JUnit 5 即可帮我们执行单元测试，仅此而已。

## 3. @BootstrapWith(SpringBootTestContextBootstrapper.class)

`@SpringBootTest` 注解的关键是 `@BootstrapWith(SpringBootTestContextBootstrapper.class)` ，注解与传入的类分别来自于 SpringFramework 原生测试包，与 SpringBoot 的测试整合包。下面分别来看。

### 3.1 @BootstrapWith

`@BootstrapWith` 注解是 SpringFramework 用于标注单元测试引导的启动类，它要求指定一个 `TestContextBootstrapper` 的实现类，而 `TestContextBootstrapper` 具备引导单元测试启动、构造测试上下文 `TestContext` 等能力，从 `TestContextBootstrapper` 接口定义的方法也可以看出这一点。

```java
public interface TestContextBootstrapper {

    void setBootstrapContext(BootstrapContext bootstrapContext);
    BootstrapContext getBootstrapContext();

    TestContext buildTestContext();

    // ......
}
```

借助 IDE ，可以发现 `TestContextBootstrapper` 的实现类有很多，而 `SpringBootTestContextBootstrapper` 是其他所有具体场景的启动器的基础实现。（具体场景的场景启动器都是继承自 `SpringBootTestContextBootstrapper` ）。

`@BootstrapWith` 注解在具体的 `TestContextManager` 测试上下文管理器中会被读取，将其中指定的 `TestContextBootstrapper` 实现类型取出，这部分内容在下面的源码中会看到。

### 3.2 SpringBootTestContextBootstrapper

上面刚说到，`SpringBootTestContextBootstrapper` 是 `TestContextBootstrapper` 的最基础实现，它的内部实现了读取解析 `@SpringBootConfiguration` 注解、构造 `TestContext` 对象等逻辑。下面就其中的几个关键方法进行了解。

#### 3.2.1 构造测试上下文

```java
@Override
public TestContext buildTestContext() {
    TestContext context = super.buildTestContext();
    verifyConfiguration(context.getTestClass());
    WebEnvironment webEnvironment = getWebEnvironment(context.getTestClass());
    if (webEnvironment == WebEnvironment.MOCK && deduceWebApplicationType() == WebApplicationType.SERVLET) {
        context.setAttribute(ACTIVATE_SERVLET_LISTENER, true);
    }
    else if (webEnvironment != null && webEnvironment.isEmbedded()) {
        context.setAttribute(ACTIVATE_SERVLET_LISTENER, false);
    }
    return context;
}

// AbstractTestContextBootstrapper
public TestContext buildTestContext() {
    return new DefaultTestContext(getBootstrapContext().getTestClass(), buildMergedContextConfiguration(),
            getCacheAwareContextLoaderDelegate());
}	
```

`TestContextBootstrapper` 本身具备创建 `TestContext` 的能力，而最终创建的测试上下文类型为 `DefaultTestContext` ，它会获取到 `BootstrapContext` 并从中获取一些必要信息进行组装。在创建完 `DefaultTestContext` 后，`buildTestContext` 方法还会根据测试使用的 Web 类型，给测试上下文中添加其他的属性。

顺便我们再关注一下 `BootstrapContext` 的构造吧，毕竟有了 `BootstrapContext` 才有 `TestContext` 。`SpringBootTestContextBootstrapper` 内部的 `BootstrapContext` 对象本身存在一对 getter / setter ，那就说明 `SpringBootTestContextBootstrapper` 本身不产生 `BootstrapContext` ，而是有其他的 API 负责创建。借助 IDE 可以发现，在 `resolveTestContextBootstrapper` 方法中有 `setBootstrapContext` 的动作，而调用 `resolveTestContextBootstrapper` 方法的位置是 `TestContextManager` 的构造方法中：

```java
static TestContextBootstrapper resolveTestContextBootstrapper(BootstrapContext bootstrapContext) {
    Class<?> testClass = bootstrapContext.getTestClass();

    Class<?> clazz = null;
    try {
        clazz = resolveExplicitTestContextBootstrapper(testClass);
        if (clazz == null) {
            clazz = resolveDefaultTestContextBootstrapper(testClass);
        }
        // logger ......
        TestContextBootstrapper testContextBootstrapper =
                BeanUtils.instantiateClass(clazz, TestContextBootstrapper.class);
        // 此处会在TestContextBootstrapper创建后设置BootstrapContext
        testContextBootstrapper.setBootstrapContext(bootstrapContext);
        return testContextBootstrapper;
    } // catch throw ex ......
}

public TestContextManager(Class<?> testClass) {
    // BootstrapUtils.createBootstrapContext静态方法创建
    this(BootstrapUtils.resolveTestContextBootstrapper(BootstrapUtils.createBootstrapContext(testClass)));
}
```

如果追踪 `TestContextManager` 的构造方法继续向上，会来到 `SpringExtension` 中，这也就是类似于 JUnit 4 中使用的 `SpringRunner` 一样了，在 JUnit 5 中它**使用 `SpringExtension` 先创建 `TestContextManager` ，随后创建出 `BootstrapContext` ，再然后是创建 `TestContextBootstrapper` 并设置 `BootstrapContext`** ，整个流程一气呵成。

```java
// SpringExtension
static TestContextManager getTestContextManager(ExtensionContext context) {
    Assert.notNull(context, "ExtensionContext must not be null");
    Class<?> testClass = context.getRequiredTestClass();
    Store store = getStore(context);
    return store.getOrComputeIfAbsent(testClass, TestContextManager::new, TestContextManager.class);
}
```

#### 3.2.2 检索@SpringBootConfiguration注解

下一个关注的点是 `@SpringBootConfiguration` 注解的检索，在 `SpringBootTestContextBootstrapper` 中有一个专门的 `getOrFindConfigurationClasses` 方法，它会利用 `AnnotatedClassFinder` 去找 **SpringBoot 单元测试类所属的包及父包**中是否有标注 `@SpringBootConfiguration` 注解的类（也即 SpringBoot 主启动类）。

```java
protected Class<?>[] getOrFindConfigurationClasses(MergedContextConfiguration mergedConfig) {
    Class<?>[] classes = mergedConfig.getClasses();
    if (containsNonTestComponent(classes) || mergedConfig.hasLocations()) {
        return classes;
    }
    Class<?> found = new AnnotatedClassFinder(SpringBootConfiguration.class)
            .findFromClass(mergedConfig.getTestClass());
    // assert logger ......
    return merge(found, classes);
}
```

注意上一段话中的重点：单元测试类所属的包及父包，因为单元测试类上肯定不会标注 `@SpringBootConfiguration` 注解，而是 SpringBoot 主启动类标注，所以当初我们学习单元测试类编写的时候说到，必须把测试 SpringBoot 的单元测试类写到 SpringBoot 主启动类所在包或子包下，而对应的底层逻辑支撑就在这个 `AnnotatedClassFinder` 中。看下面的源码，从 `findFromClass` 一直调用到最下面的 `scanPackage` 包扫描方法，可以发现最终的扫描规则是一层一层的向上扫描，直到扫描到具体的类，或者扫描不到返回 null 。由此也就解释了编写 SpringBoot 单元测试类的所属包规则。

```java
public Class<?> findFromClass(Class<?> source) {
    Assert.notNull(source, "Source must not be null");
    return findFromPackage(ClassUtils.getPackageName(source));
}

public Class<?> findFromPackage(String source) {
    Assert.notNull(source, "Source must not be null");
    Class<?> configuration = cache.get(source);
    if (configuration == null) {
        configuration = scanPackage(source);
        cache.put(source, configuration);
    }
    return configuration;
}

private Class<?> scanPackage(String source) {
    while (!source.isEmpty()) {
        Set<BeanDefinition> components = this.scanner.findCandidateComponents(source);
        if (!components.isEmpty()) {
            // assert ......
            return ClassUtils.resolveClassName(components.iterator().next().getBeanClassName(), null);
        }
        source = getParentPackage(source);
    }
    return null;
}
```

#### 3.2.3 解析构造SpringBootContextLoader

`SpringBootTestContextBootstrapper` 中还有一个关键的方法是获取上下文的加载器，这个加载器最终会构造一个用于单元测试的 `ApplicationContext` 对象。默认情况下，在不使用 `@ContextConfiguration` 注解指定主配置源时，默认使用的是 `SpringBootContextLoader` ，这一点在源码的 `resolveContextLoader` 方法中一路向下找，可以找到 `getDefaultContextLoaderClass` 的默认实现。

```java
protected ContextLoader resolveContextLoader(Class<?> testClass, List<ContextConfigurationAttributes> configAttributesList) {
    Class<?>[] classes = getClasses(testClass);
    if (!ObjectUtils.isEmpty(classes)) {
        for (ContextConfigurationAttributes configAttributes : configAttributesList) {
            addConfigAttributesClasses(configAttributes, classes);
        }
    }
    return super.resolveContextLoader(testClass, configAttributesList);
}

protected ContextLoader resolveContextLoader(Class<?> testClass,
        List<ContextConfigurationAttributes> configAttributesList) {
    // assert ......
    Class<? extends ContextLoader> contextLoaderClass = resolveExplicitContextLoaderClass(configAttributesList);
    if (contextLoaderClass == null) {
        contextLoaderClass = getDefaultContextLoaderClass(testClass);
    }
    // logger ......
    return BeanUtils.instantiateClass(contextLoaderClass, ContextLoader.class);
}

protected Class<? extends ContextLoader> getDefaultContextLoaderClass(Class<?> testClass) {
    return SpringBootContextLoader.class;
}
```

### 3.3 SpringBootContextLoader

既然上面提到了，`SpringBootContextLoader` 作为构造加载 `ApplicationContext` 的默认测试实现，它内部有什么特殊能力，或者有哪些关键的设计值得我们关注呢？下面进入到它的内部构造中一探究竟。

#### 3.3.1 继承结构

`SpringBootContextLoader` 本身继承了 `AbstractContextLoader` ，这个父类在 SpringFramework 2.5 就已经出现了，在原生 SpringFramework 的测试中，它作为 `@ContextConfiguration` 注解的配合者，在指定 xml 配置文件路径或者注解配置类时有对应的 `GenericXmlContextLoader` 与 `AnnotationConfigContextLoader` 作为实现类，而在 SpringBoot 中这几个实现类都不太合适直接复用，所以也就有了新的 `SpringBootContextLoader` 。

![](/img/202212/06springboottest1.png)



#### 3.3.2 构造ApplicationContext

下面着重来看 `SpringBootContextLoader` 是如何构造 `ApplicationContext` 的。进入到 `loadContext` 方法中，我们可以非常清晰地看出，`SpringBootContextLoader` 构造 `ApplicationContext` 的过程，其实就是把 `SpringApplication` 的那一套构造、运行的过程，在 `loadContext` 方法中执行了一遍！（下方源码已配有注释）

```java
@Override
public ApplicationContext loadContext(MergedContextConfiguration config) throws Exception {
    Class<?>[] configClasses = config.getClasses();
    String[] configLocations = config.getLocations();
    // assert ......
    // 构造SpringApplication对象
    SpringApplication application = getSpringApplication();
    application.setMainApplicationClass(config.getTestClass());
    // 指定主启动类
    application.addPrimarySources(Arrays.asList(configClasses));
    application.getSources().addAll(Arrays.asList(configLocations));
    List<ApplicationContextInitializer<?>> initializers = getInitializers(config, application);
    // 设置Web类型
    if (config instanceof WebMergedContextConfiguration) {
        application.setWebApplicationType(WebApplicationType.SERVLET);
        if (!isEmbeddedWebEnvironment(config)) {
            new WebConfigurer().configure(config, application, initializers);
        }
    } else if (config instanceof ReactiveWebMergedContextConfiguration) {
        application.setWebApplicationType(WebApplicationType.REACTIVE);
    } else {
        application.setWebApplicationType(WebApplicationType.NONE);
    }
    // 一些其他的处理 ......
    String[] args = SpringBootTestArgs.get(config.getContextCustomizers());
    // 执行run方法启动SpringApplication
    return application.run(args);
}
```

由此我们也就可以明白，为什么在 SpringBoot 的单元测试类执行时，控制台也会打印类似于 SpringBoot 应用启动时的 Banner 、日志等，本质上底层都是执行了 `SpringApplication` 的 `run` 方法，自然打印的东西也就大同小异了。