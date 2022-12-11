---
title: 3springboot基础.md
toc: true
tags: springboot
categories: 
    - [java]
    - [springboot]
---

## 1. 自动装配机制的设计

SpringBoot 的自动装配机制，核心是由 SpringFramework 的**模块装配 + 条件装备 + SPI 机制**组合而成，最终落到 `@SpringBootApplication` 注解中予以最终实现。我们先简单回顾自动装配的底层支持。

### 1.1 模块装配

模块装配的核心是把一个场景中所需要的组件，一次性注册到应用的上下文中。模块装配强调的是**整体**，它通过定义一个特殊的注解，配合 `@Import` 注解，即可实现 4 种不同类型组件的导入，从而实现多个组件的一次性导入。

`@Import` 注解可以导入的组件类型有：

<!--more-->

- 普通类
- 注解配置类
- `ImportSelector`
- `ImportBeanDefinitionRegistrar`

### 1.2 条件装配

条件装配的核心是根据不同场景分别导入不同的组件，SpringFramework 中包含基于 Profile 的条件装配，以及基于 Conditional 的条件装配。SpringBoot 中更多使用的是基于 Conditional 的条件装配，它相较于 Profile 来讲适用范围更广，判断条件更加灵活。SpringBoot 由 SpringFramework 的基础之上扩展了一组 `@ConditionalOnxxx` 注解，进一步简化了条件装配的使用。

### 1.3 SPI机制

Spring SPI 机制本身是依赖倒转原则的体现，它可以通过一个指定的接口 / 实现类 / 注解，寻找到预先配置好的实现类（并可以创建实现类的对象）。相较于 jdk 原生的 SPI 机制，SpringFramework 的 SPI 机制更加强大，所以 SpringBoot 直接借用了 SpringFramework 的 SPI 机制，并利用其可以支持注解作为索引的特性，实现了 SpringBoot 中最强大的特性之一：自动装配。

### 1.4 @SpringBootApplication注解

SpringBoot 的主启动类上通常都会标注 `@SpringBootApplication` 注解，用于启用自动装配和组件扫描。

```java
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
        @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication
```

`@ComponentScan` 注解用于扫描主启动类所在的包及子包下的所有组件，`@SpringBootConfiguration` 注解用于标注当前的类是一个配置类，这两个注解都相对简单，最复杂也是最重要的注解是 `@EnableAutoConfiguration` ，它承载着 SpringBoot 自动装配的灵魂。

总的来看，`@EnableAutoConfiguration` 注解的作用是：**启用 SpringBoot 的自动装配，根据导入的依赖、上下文配置合理加载默认的自动配置**。

具体到源码中，它通过 `@AutoConfigurationPackage` 注解将当前主启动类的所在包记录下来，为第三方组件场景整合时提供支持；

```java
@Import(AutoConfigurationPackages.Registrar.class)
public @interface AutoConfigurationPackage
```

导入的 `AutoConfigurationImportSelector` 本身是一个 `ImportSelector` ，它利用 SpringFramework 的 SPI 机制，将当前工程中所有的 `spring.factories` 文件中 `@EnableAutoConfiguration` 对应的自动配置类都取出并进行匹配，最终确定出可以被应用的自动配置类并注册。

```java
protected AutoConfigurationEntry getAutoConfigurationEntry(AnnotationMetadata annotationMetadata) {
    if (!isEnabled(annotationMetadata)) {
        return EMPTY_ENTRY;
    }
    // 加载注解属性配置
    AnnotationAttributes attributes = getAttributes(annotationMetadata);
    // 【加载自动配置类的真实动作】
    List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
    // 自动配置类去重
    configurations = removeDuplicates(configurations);
    // 获取显式配置了要排除掉的自动配置类
    Set<String> exclusions = getExclusions(annotationMetadata, attributes);
    checkExcludedClasses(configurations, exclusions);
    // 排除动作
    configurations.removeAll(exclusions);
    configurations = getConfigurationClassFilter().filter(configurations);
    // 广播AutoConfigurationImportEvent事件
    fireAutoConfigurationImportEvents(configurations, exclusions);
    return new AutoConfigurationEntry(configurations, exclusions);
}

protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
    List<String> configurations = SpringFactoriesLoader.loadFactoryNames(getSpringFactoriesLoaderFactoryClass(), getBeanClassLoader());
    // assert ......
    return configurations;
}
```

由上方 `getAutoConfigurationEntry` 方法往下执行，核心的 `getCandidateConfigurations` 方法即是加载自动配置类的核心方法，对应的源码实现如下：

```java
// 存储SPI机制加载的类及其映射
private static final Map<ClassLoader, MultiValueMap<String, String>> cache = new ConcurrentReferenceHashMap<>();
// SPI的文件名必须为spring.factories
public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";

public static List<String> loadFactoryNames(Class<?> factoryType, @Nullable ClassLoader classLoader) {
    String factoryTypeName = factoryType.getName();
    // 利用缓存机制提高加载速度
    return loadSpringFactories(classLoader).getOrDefault(factoryTypeName, Collections.emptyList());
}

private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
    // 解析之前先检查缓存，有则直接返回
    MultiValueMap<String, String> result = cache.get(classLoader);
    if (result != null) {
        return result;
    }

    try {
        // 真正的加载动作，利用类加载器加载所有的spring.factories(多个)，并逐个配置解析
        Enumeration<URL> urls = (classLoader != null ?
                classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
                ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
        result = new LinkedMultiValueMap<>();
        while (urls.hasMoreElements()) {
            // 取出每个spring.factories文件
            URL url = urls.nextElement();
            UrlResource resource = new UrlResource(url);
            // 以Properties的方式读取spring.factories
            Properties properties = PropertiesLoaderUtils.loadProperties(resource);
            for (Map.Entry<?, ?> entry : properties.entrySet()) {
                // 逐个配置项映射收集
                String factoryTypeName = ((String) entry.getKey()).trim();
                // 如果一个key配置了多个value，则用英文逗号隔开，此处会做切分
                for (String factoryImplementationName : StringUtils.commaDelimitedListToStringArray((String) entry.getValue())) {
                    result.add(factoryTypeName, factoryImplementationName.trim());
                }
            }
        }
        cache.put(classLoader, result);
        return result;
    } // catch ......
}
```

## 2. 自动配置类的设计

自动装配的核心是自动配置类，在 SpringBoot 加载自动配置类后，会一一解析这些自动配置类。下面以几个 SpringBoot 内置的自动配置类为例，提取自动配置类中的核心设计细节。

### 2.1 HttpEncodingAutoConfiguration

`HttpEncodingAutoConfiguration` 的核心作用是解决 WebMvc 的使用场景中请求编码的问题。做过原生 SSM 开发的小伙伴一定不会陌生，在 web.xml 中需要配置一个 `CharacterEncodingFilter` ，使用它就可以解决客户端请求的数据乱码问题。下面通过源码简单观察 `HttpEncodingAutoConfiguration` 的设计。

```java
@Configuration(proxyBeanMethods = false)
@EnableConfigurationProperties(ServerProperties.class)
@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.SERVLET)
@ConditionalOnClass(CharacterEncodingFilter.class)
@ConditionalOnProperty(prefix = "server.servlet.encoding", value = "enabled", matchIfMissing = true)
public class HttpEncodingAutoConfiguration {
```

通过 `HttpEncodingAutoConfiguration` 类上标注的注解，可以得知 `HttpEncodingAutoConfiguration` 的生效条件如下：

- 当前的 Web 类型是 **Servlet** ；
- 当前工程的类路径下存在一个 `CharacterEncodingFilter` ；
- 当前全局配置文件中有声明一个 `server.servlet.encoding.enabled` 属性，且值不为 false （或没有声明该属性）。

当以上条件全部生效时，该自动配置类会做以下几个关键动作：

- 开启 `ServerProperties` 对象与 SpringBoot 全局配置文件的属性绑定；
- 向容器中注册一个 `CharacterEncodingFilter` 。

注意其中注册 `CharacterEncodingFilter` 的源码，创建的方法上方标注了 `@ConditionalOnMissingBean` 注解，意味着如果工程中没有显式注册该组件，则自动配置类会生效并注册，反之亦然。

```java
    private final Encoding properties;

    public HttpEncodingAutoConfiguration(ServerProperties properties) {
        this.properties = properties.getServlet().getEncoding();
    }

    @Bean
    @ConditionalOnMissingBean
    public CharacterEncodingFilter characterEncodingFilter() {
        CharacterEncodingFilter filter = new OrderedCharacterEncodingFilter();
        filter.setEncoding(this.properties.getCharset().name());
        filter.setForceRequestEncoding(this.properties.shouldForce(Encoding.Type.REQUEST));
        filter.setForceResponseEncoding(this.properties.shouldForce(Encoding.Type.RESPONSE));
        return filter;
    }
```

由此可以总结出自动配置类的几个设计细节：

- **`@ConditionalOnXXX` 注解可以控制自动配置类是否生效；**
- **自动配置类生效后，可以激活当前配置类中需要使用的配置属性；**
- **自动配置类注册组件时遵循约定大于配置原则。**

### 2.2 DataSourceAutoConfiguration

`DataSourceAutoConfiguration` 的核心作用是向应用上下文中注册数据源。当引入 `spring-boot-starter-jdbc` 依赖后，该配置类会默认向容器中注册一个 `DataSource` 数据源。

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass({ DataSource.class, EmbeddedDatabaseType.class })
@ConditionalOnMissingBean(type = "io.r2dbc.spi.ConnectionFactory")
@EnableConfigurationProperties(DataSourceProperties.class)
@Import({ DataSourcePoolMetadataProvidersConfiguration.class,
        DataSourceInitializationConfiguration.InitializationSpecificCredentialsDataSourceInitializationConfiguration.class,
        DataSourceInitializationConfiguration.SharedCredentialsDataSourceInitializationConfiguration.class })
public class DataSourceAutoConfiguration {
```

注意观察 `DataSourceAutoConfiguration` 类上的注解可以发现，除了有类似 `HttpEncodingAutoConfiguration` 中的 `@ConditionalOnXXX` 与 `@EnableConfigurationProperties` 注解之外，还有 `@Import` 注解导入其他的配置类。

由此可以知道，自动配置类的另一个设计细节：**自动配置类可以导入其他的配置类（这些配置类也会参与条件装配的规则匹配）**。

除此之外，观察 `DataSourceAutoConfiguration` 内部的几个内部类，可以发现部分内部类本身也是注解配置类，它们也被 `@Import` 注解标注，导入一些其他的配置类。以下源码是导入数据库连接池的注解配置内部类 `PooledDataSourceConfiguration` 的定义。

```java
public class DataSourceAutoConfiguration {

    @Configuration(proxyBeanMethods = false)
    @Conditional(PooledDataSourceCondition.class)
    @ConditionalOnMissingBean({ DataSource.class, XADataSource.class })
    @Import({ DataSourceConfiguration.Hikari.class, DataSourceConfiguration.Tomcat.class,
            DataSourceConfiguration.Dbcp2.class, DataSourceConfiguration.OracleUcp.class,
            DataSourceConfiguration.Generic.class, DataSourceJmxConfiguration.class })
    protected static class PooledDataSourceConfiguration {

    }
```

由此可以再总结一个设计细节：**自动配置类的内部可能会定义其他的注解配置类，这些类会通过内部类的方式定义**。

### 2.3 WebMvcAutoConfiguration

`WebMvcAutoConfiguration` 是基于 WebMvc 场景的核心自动装配，它会向应用上下文注册 WebMvc 相关的核心组件。观察 `WebMvcAutoConfiguration` 类上标注的注解，可以发现一些新的注解。

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnWebApplication(type = Type.SERVLET)
@ConditionalOnClass({ Servlet.class, DispatcherServlet.class, WebMvcConfigurer.class })
@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE + 10)
@AutoConfigureAfter({ DispatcherServletAutoConfiguration.class, TaskExecutionAutoConfiguration.class,
		ValidationAutoConfiguration.class })
public class WebMvcAutoConfiguration {
```

通过上述源码可以发现，`WebMvcAutoConfiguration` 被两个具备排序意义的注解标注，分别是 `@AutoConfigureOrder` 与 `@AutoConfigureAfter` ，它们分别代表着绝对顺序与相对顺序。`@AutoConfigureOrder` 注解会传入一个 order 排序值，数值越小优先级越高；`@AutoConfigureAfter` 与 `@AutoConfigureBefore` 则可以指定当前自动配置类的顺序在指定配置类之后 / 之前。

由此可以再总结出一个设计细节：**自动配置类可以指定被应用的顺序，来达到一些特殊的定制效果**。

【自动装配和自动配置类是 SpringBoot 的关键特性，另一个特性 starter 场景启动器则是针对每个应用场景下的具体制作，下一章会回顾 starter 的设计细节】

