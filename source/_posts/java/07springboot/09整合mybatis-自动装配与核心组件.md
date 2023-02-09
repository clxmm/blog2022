---
title: 09整合mybatis-自动装配与核心组件
toc: true
tags: springboot
categories: 
    - [java]
    - [springboot]
---

通过前面两章的内容，我们回顾了 MyBatis 以及 MyBatisPlus 的使用，以及其中的一些重要特性。MyBatis 作为第三方框架，整合进 SpringBoot 时也是使用 starter 场景启动器机制，本章内容我们就来看一下，MyBatis 整合 SpringBoot 后都发生了哪些自动装配，以及其中注册了哪些重要的核心组件。

## 1. 核心自动装配

<!--more-->

翻开 MyBatis 整合 SpringBoot 的启动器，借助 IDE 可以发现我们导入的 `mybatis-spring-boot-starter` 仅仅是一个空的 jar 包，没有具体的部分，而这个依赖本身又依赖了一个 `mybatis-spring-boot-autoconfigure` 的依赖，根据经验可以得知这个依赖中通常包含一个 `spring.factories` 文件。打开 jar 包依赖，可以发现它的确包含一个 `spring.factories` 文件，这个文件的内容中仅仅定义了两个自动配置类。

```properties
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.mybatis.spring.boot.autoconfigure.MybatisLanguageDriverAutoConfiguration,\
org.mybatis.spring.boot.autoconfigure.MybatisAutoConfiguration

```

由此可知，MyBatis 整合 SpringBoot 的核心装配就在这两个自动配置类上，下面我们逐一展开，看看这里面都干了什么。

## 2. MybatisAutoConfiguration

`MybatisAutoConfiguration` 是 MyBatis 整合 SpringBoot 的核心自动配置类，MyBatis 内部支撑运行的所有组件都在这个配置类中创建，下面完整讲解 `MybatisAutoConfiguration` 中的所有配置内容。

### 2.1 SqlSessionFactory

经历过 SSM 整合的小伙伴一定很熟悉，我们需要在 xml 配置文件或者注解配置类中编写注册一个 `SqlSessionFactory` ，它有了 `SqlSessionFactory` 就可以创建 `SqlSession` 对象，进而使用 `SqlSession` 进行 CRUD 操作。

`MybatisAutoConfiguration` 中注册的最关键组件就是这个 `SqlSessionFactory` ，创建 `SqlSessionFactory` 的核心逻辑比较长，我们重点关注其中的重要部分。

```java
@Bean
@ConditionalOnMissingBean
public SqlSessionFactory sqlSessionFactory(DataSource dataSource) throws Exception {
    SqlSessionFactoryBean factory = new SqlSessionFactoryBean();
    // 数据源
    factory.setDataSource(dataSource);
    factory.setVfs(SpringBootVFS.class);
    // 外部MyBatis原生配置文件
    if (StringUtils.hasText(this.properties.getConfigLocation())) {
        factory.setConfigLocation(this.resourceLoader.getResource(this.properties.getConfigLocation()));
    }
    // 应用properties配置
    applyConfiguration(factory);
    if (this.properties.getConfigurationProperties() != null) {
        factory.setConfigurationProperties(this.properties.getConfigurationProperties());
    }
    // 设置插件
    if (!ObjectUtils.isEmpty(this.interceptors)) {
        factory.setPlugins(this.interceptors);
    }
    // 更多set操作......

    return factory.getObject();
}
```

总的来看，创建 `SqlSessionFactory` 的步骤只是把连接数据库的数据源、全局配置文件中，提取出的配置对象、IOC 容器中注册的 MyBatis 拦截器等组件，一一应用在 `SqlSessionFactoryBean` 中，而在一堆操作逻辑之后，实际构建 `SqlSessionFactory` 的动作是整个方法最后一行的 `factory.getObject();` 方法调用。

```java
public SqlSessionFactory getObject() throws Exception {
    if (this.sqlSessionFactory == null) {
        afterPropertiesSet();
    }
    return this.sqlSessionFactory;
}

public void afterPropertiesSet() throws Exception {
    // 判断 ......
    this.sqlSessionFactory = buildSqlSessionFactory();
}
```

注意观察上面的方法，在 `getObject` 方法中包含一个至关重要的方法 `afterPropertiesSet` ，它会在一些前置判断后执行 `buildSqlSessionFactory` 方法，构建实际的 `SqlSessionFactory` 对象。由此我们可以了解到一个比较特殊的设计：`afterPropertiesSet` 方法本身是来自 `InitializingBean` 接口的方法，可以被 IOC 容器回调，但是它作为一个普通的方法，完全可以由其他方法手动调用。

既然 MyBatis 整合 SpringBoot 时采用了这种特殊的设计，各位可以多想一下，为什么它非要这么做呢？

答案很明确，MyBatis 整合 SpringBoot 时没有向容器注册 `SqlSessionFactoryBean` ，而是直接注册了 `SqlSessionFactory` 对象，这样就没办法通过 IOC 容器触发 `SqlSessionFactoryBean` 的 `afterPropertiesSet` 方法，以初始化 `SqlSessionFactory` 了，所以 MyBatis 在整合 SpringBoot 时，采用的是手动调用的方式。

#### 2.1.1 处理MyBatis全局配置对象

```java
protected SqlSessionFactory buildSqlSessionFactory() throws Exception {
    final Configuration targetConfiguration;

    XMLConfigBuilder xmlConfigBuilder = null;
    // 构造SqlSessionFactoryBean时传入了外部Configuration，则直接处理
    if (this.configuration != null) {
        targetConfiguration = this.configuration;
        if (targetConfiguration.getVariables() == null) {
            targetConfiguration.setVariables(this.configurationProperties);
        } else if (this.configurationProperties != null) {
            targetConfiguration.getVariables().putAll(this.configurationProperties);
        }
    } else if (this.configLocation != null) {
        // 传入全局配置文件路径，则封装XMLConfigBuilder对象以备加载
        xmlConfigBuilder = new XMLConfigBuilder(this.configLocation.getInputStream(), null, this.configurationProperties);
        targetConfiguration = xmlConfigBuilder.getConfiguration();
    } else {
        // 无外部Configuration对象，则执行默认策略
        targetConfiguration = new Configuration();
        Optional.ofNullable(this.configurationProperties).ifPresent(targetConfiguration::setVariables);
    }
    // ......
```

`buildSqlSessionFactory` 方法的第一部分内容核心工作是预准备 MyBatis 的全局配置对象 `Configuration` ，并根据是否事先传入外部的 `configuration` 对象或者传入全局配置文件路径，决定是否准备 `XMLConfigBuilder` 。如果确实需要 `XMLConfigBuilder` 的处理，在第 6 步会有配置文件解析的步骤。

#### 2.1.2 处理内置组件

```java
    // ......
    Optional.ofNullable(this.objectFactory).ifPresent(targetConfiguration::setObjectFactory);
    Optional.ofNullable(this.objectWrapperFactory).ifPresent(targetConfiguration::setObjectWrapperFactory);
    Optional.ofNullable(this.vfs).ifPresent(targetConfiguration::setVfsImpl);
    // ......
```

紧接着的 3 个设置动作分别对应了 MyBatis 中的三个组件，这里简单解释一下。 `ObjectFactory` 对象工厂负责创建 MyBatis 查询封装结果集时创建结果对象（模型类对象、`Map` 等），`ObjectWrapperFactory` 对象包装工厂负责对创建出的对象进行包装，`Vfs` 虚拟文件系统用于加载工程下的资源文件。

在这个环节下 SpringBoot 仅仅对 VFS 的部分进行了扩展，它额外定义了一个 `SpringBootVFS` 用于加载项目中的资源文件，除此之外没有任何多余的扩展，各位读者在此处也不要浪费过多时间。

#### 2.1.3 别名处理

```java
    // ......
    if (hasLength(this.typeAliasesPackage)) {
        scanClasses(this.typeAliasesPackage, this.typeAliasesSuperType).stream()
            .filter(clazz -> !clazz.isAnonymousClass()).filter(clazz -> !clazz.isInterface())
            .filter(clazz -> !clazz.isMemberClass()).forEach(targetConfiguration.getTypeAliasRegistry()::registerAlias);
    }

    if (!isEmpty(this.typeAliases)) {
        Stream.of(this.typeAliases).forEach(typeAlias -> {
            targetConfiguration.getTypeAliasRegistry().registerAlias(typeAlias);
            // logger ......
        });
    }
    // ......
```

这一段源码的动作是别名的包扫描以及某些特定类的别名设置。MyBatis 允许在项目中为实体类定义别名，这种设计可以使得在 mapper.xml 映射文件中简化类型编写，使映射文件更清爽简洁。在默认情况下包扫描注册的别名就是类名本身（首字母大写），如果类上有标注 `@Alias` 注解，则会取注解属性值。

#### 2.1.4 处理插件、类型处理器

```java
    // ......
    if (!isEmpty(this.plugins)) {
        Stream.of(this.plugins).forEach(plugin -> {
            targetConfiguration.addInterceptor(plugin);
            // logger ......
        });
    }

    if (hasLength(this.typeHandlersPackage)) {
        scanClasses(this.typeHandlersPackage, TypeHandler.class).stream().filter(clazz -> !clazz.isAnonymousClass())
            .filter(clazz -> !clazz.isInterface()).filter(clazz -> !Modifier.isAbstract(clazz.getModifiers()))
            .forEach(targetConfiguration.getTypeHandlerRegistry()::register);
    }

    if (!isEmpty(this.typeHandlers)) {
        Stream.of(this.typeHandlers).forEach(typeHandler -> {
            targetConfiguration.getTypeHandlerRegistry().register(typeHandler);
            // logger ......
        });
    }

    targetConfiguration.setDefaultEnumTypeHandler(defaultEnumTypeHandler);
    // ......
```

下面一段源码的逻辑是处理 MyBatis 插件以及 `TypeHandler` 。注意此处的 `this.plugins` 是在 `MybatisAutoConfiguration` 中，利用 `ObjectProvider` 从 IOC 容器中获取到的。由此就可以得出一个简单结论：SpringBoot 整合 MyBatis 后可以直接向 IOC 容器中注册 MyBatis 的关键组件，底层的自动装配可以将这些组件应用到 MyBatis 中。

#### 2.1.5 处理其他内部组件

```java
    // ......
    if (!isEmpty(this.scriptingLanguageDrivers)) {
        Stream.of(this.scriptingLanguageDrivers).forEach(languageDriver -> {
            targetConfiguration.getLanguageRegistry().register(languageDriver);
            // logger ......
        });
    }
    Optional.ofNullable(this.defaultScriptingLanguageDriver)
        .ifPresent(targetConfiguration::setDefaultScriptingLanguage);

    if (this.databaseIdProvider != null) {
        try {
            targetConfiguration.setDatabaseId(this.databaseIdProvider.getDatabaseId(this.dataSource));
        } // catch throw ex ......
    }

    Optional.ofNullable(this.cache).ifPresent(targetConfiguration::addCache);
    // ......
```

除了内部核心的对象工厂、插件、类型处理器等组件外，MyBatis 还有一些地位相对不太重要的内部组件，如脚本语言驱动器（支持多种映射文件格式的编写）、数据库厂商标识器（为数据库可移植性提供了可能）等，在构建 `SqlSessionFactory` 的过程中，该部分也会予以初始化。

#### 2.1.6 解析MyBatis全局配置文件

```java
    // ......
    if (xmlConfigBuilder != null) {
        try {
            xmlConfigBuilder.parse();
            // logger ......
        } // catch finally ......
    }
    // ......
```

在代码清单 11-9 中有一段配置文件的处理，如果项目配置中传入了 `configLocation` ，则此处会使用 `XMLConfigBuilder` 解析 MyBatis 配置文件，并应用于 MyBatis 全局配置对象 `Configuration` 中。SpringBoot 已经对 MyBatis 中的配置尽可能多地移植到 `application.properties` 中，所以代码清单 11-14 中的内容可以忽略。

#### 2.1.7 处理数据源和事务工厂

```java
    // ......
    targetConfiguration.setEnvironment(new Environment(this.environment,
            this.transactionFactory == null ? new SpringManagedTransactionFactory() : this.transactionFactory,
            this.dataSource));
    // ......
```

这段源码只有一行代码，但它完成了两个组件的初始化：**数据源与事务工厂**，默认情况下 MyBatis 与 SpringBoot 整合之后，底层使用的事务工厂是 `SpringManagedTransactionFactory` ，了解 MyBatis 底层原理的读者可能会了解 `JdbcTransactionFactory` ，它是 MyBatis 原生控制事务的事务工厂，`SpringManagedTransactionFactory` 与 `JdbcTransactionFactory` 的底层事务控制并无太大差别。

#### 2.1.8 处理Mapper

```java
    // ......
    if (this.mapperLocations != null) {
        if (this.mapperLocations.length == 0) {
            // logger ......
        } else {
            for (Resource mapperLocation : this.mapperLocations) {
                if (mapperLocation == null) {
                    continue;
                }
                try {
                    XMLMapperBuilder xmlMapperBuilder = new XMLMapperBuilder(mapperLocation.getInputStream(),
                            targetConfiguration, mapperLocation.toString(), targetConfiguration.getSqlFragments());
                    xmlMapperBuilder.parse();
                } // catch finally ......
                // logger ......
            }
        }
    } // else logger ......
    return this.sqlSessionFactoryBuilder.build(targetConfiguration);
}

```

由于 `SqlSessionFactoryBean` 的构造中只能传入 mapper.xml 映射文件的路径，所以 `SqlSessionFactoryBean` 本身的逻辑中并无 Mapper 接口的扫描。源码中可以发现，解析 mapper.xml 映射文件的逻辑是借助 `XMLMapperBuilder` 组件实现，这个组件会逐个解析 mapper.xml ，封装 `MappedStatement` 并注册到 MyBatis 的全局配置对象 `Configuration` 中。

经过一个庞大的 `afterPropertiesSet` 方法后，`SqlSessionFactory` 被成功创建，MyBatis 的核心也就初始化完毕了。

### 2.2 SqlSessionTemplate

相比较于 `SqlSessionFactory` 的构造过程，`SqlSessionTemplate` 的创建逻辑非常简单。

```java
@Bean
@ConditionalOnMissingBean
public SqlSessionTemplate sqlSessionTemplate(SqlSessionFactory sqlSessionFactory) {
    ExecutorType executorType = this.properties.getExecutorType();
    if (executorType != null) {
        return new SqlSessionTemplate(sqlSessionFactory, executorType);
    } else {
        return new SqlSessionTemplate(sqlSessionFactory);
    }
}
```

`SqlSessionTemplate` 本身是一个实现了 `SqlSession` 接口的模板类，它可以非常简单地调用 MyBatis 的核心 CRUD 方法，而不必关心 `SqlSession` 的生命周期。在构建 `SqlSessionTemplate` 时，需要的依赖组件是 `SqlSessionFactory` ，毕竟只有传入 `SqlSessionFactory` 之后 `SqlSessionTemplate` 才能获取到实际的 `SqlSession` 并调用其方法。除此之外，构造方法的另外一个参数我们也可以关注一下，它的类型是一个枚举：`ExecutorType`。

`ExecutorType` 从类名上理解，它指代的是 SQL 语句的执行类型。MyBatis 内置的 `ExecutorType` 有三种，默认的类型是 `SIMPLE` ，这种执行类型会在 `SqlSession` 执行具体的 CRUD 操作时，为每条 SQL 语句创建一个预处理对象 `PreparedStatement` 并逐条执行；`REUSE` 模式会复用同一条 SQL 对应创建的 `PreparedStatement` ，该模式会一定程度的提高 MyBatis 的执行效率；`BATCH` 模式下不仅会复用 `PreparedStatement` 对象，还会执行批量操作，这也使得 `BATCH` 模式的效率是最高的。但是使用 `BATCH` 模式有一个缺陷，是在执行 insert 语句时，如果插入的数据库表主键是自增序列，则在事务提交之前，无法从数据库获得实际的自增 id 值，这种设计在某些业务场景下是不符合要求的。

### 2.3 AutoConfiguredMapperScannerRegistrar

由于 `SqlSessionFactory` 的构建中没有处理 Mapper 接口的扫描，所以 MyBatis 在整合 SpringBoot 时专门提供了一个适配 SpringBoot 工程模式的 Mapper 接口扫描注册器，这个扫描器会在 SpringBoot 工程中没有标注 `@MapperScan` 注解时生效。

```java
@Configuration
@Import(AutoConfiguredMapperScannerRegistrar.class)
@ConditionalOnMissingBean({MapperFactoryBean.class, MapperScannerConfigurer.class})
public static class MapperScannerRegistrarNotFoundConfiguration 
        implements InitializingBean { 
    // ...... 
}

@Import(MapperScannerRegistrar.class)
public @interface MapperScan

```

由于 `@MapperScan` 注解会向 IOC 容器中导入一个 `MapperScannerRegistrar` 组件，那么当工程中没有标注该注解时，缺省的 `AutoConfiguredMapperScannerRegistrar` 就会被导入并生效（约定大于配置）。而从 `registerBeanDefinitions` 方法的实现逻辑中可以看出，默认注册的 `MapperScannerConfigurer` 扫描器中扫描规则是扫描 SpringBoot 主启动类所在包及子包下的所有标注了 `@Mapper` 注解的接口。

```java
public static class AutoConfiguredMapperScannerRegistrar implements BeanFactoryAware, ImportBeanDefinitionRegistrar {
  
  private BeanFactory beanFactory;

  @Override
  public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
    // logger ......
    // 取出SpringBoot主启动类所在包
    List<String> packages = AutoConfigurationPackages.get(this.beanFactory);
    
    BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition(MapperScannerConfigurer.class);
    builder.addPropertyValue("processPropertyPlaceHolders", true);
    // 扫描标识@Mapper接口的接口
    builder.addPropertyValue("annotationClass", Mapper.class);
    builder.addPropertyValue("basePackage", StringUtils.collectionToCommaDelimitedString(packages));
    BeanWrapper beanWrapper = new BeanWrapperImpl(MapperScannerConfigurer.class);
    Stream.of(beanWrapper.getPropertyDescriptors())
        .filter(x -> x.getName().equals("lazyInitialization")).findAny()
        .ifPresent(x -> builder.addPropertyValue("lazyInitialization", "${mybatis.lazy-initialization:false}"));
    registry.registerBeanDefinition(MapperScannerConfigurer.class.getName(), builder.getBeanDefinition());
  }
```

以此法注册 `MapperScannerConfigurer` 后，在项目开发中只需要编写 Mapper 接口并标注 `@Mapper` 注解，即可被 `MapperScannerConfigurer` 自动扫描并注册到 IOC 容器中。

## 3. MybatisLanguageDriverAutoConfiguration

从 `MybatisLanguageDriverAutoConfiguration` 的类名中可以了解到一个点，它的配置与“语言驱动”有关。如何理解“语言驱动”这个概念，需要小伙伴先了解一个 MyBatis 中的设计。

MyBatis 默认使用的映射文件编写是使用 xml 文件作为载体，且 xml 映射文件中的内容是固定的几个标签，但是从 MyBatis 3.2 开始支持使用第三方模板引擎框架作为编写映射文件的实现，而使用的 xml 配置文件编写仅仅是 MyBatis 默认提供的映射文件缺省实现而已。

![](/img/202212/09springboot-mybatis.png)

从上图的 `MybatisLanguageDriverAutoConfiguration` 源码截图中可以发现，MyBatis 支持的第三方模板引擎框架包含 FreeMarker 、Velocity 、Thymeleaf 等，但是默认场景下 MyBatis 不会引入过多的依赖，这也使得我们开发者仅仅熟悉原生的 xml 映射文件编写即可。本身这个配置类不重要，小伙伴们不要在此耗费过多的精力。

【到此为止，有关 MyBatis 的研究探讨就先告一段落。除了 MyBatis 之外，SpringData 也是 Dao 层框架的一个主流之选，接下来的几章就来研究 SpringData 的使用】

