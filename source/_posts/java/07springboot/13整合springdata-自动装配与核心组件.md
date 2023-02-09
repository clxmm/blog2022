---
title: 13整合springdata-自动装配与核心组件
toc: true
tags: springboot
categories: 
    - [java]
    - [springboot]
---

整合 SpringData 的最后一个章节，我们来看看当 SpringBoot 的工程中引入了 SpringData 后，底层都帮我们做了哪些工作，并且当引入特定的模块后，又有哪些附加的工作被处理了。

由于 SpringData 本身是一组模块的统称，每个模块对应的装配都不一样，这也使得我们前面几章中了解的模块中甚至都提取不出来自动装配的共有配置。所以本章的内容会分模块剖析其中的核心自动装配。


<!--more-->

## 1. spring-jdbc对应的自动装配

---

SpringDataJPA 的底层使用了 jdbc 相关的内容，而 spring-jdbc 本身就有一些关键的自动装配，具体如下。

### 1.1 DataSourceAutoConfiguration

连接关系型数据库的必备核心是数据库连接池，在 `DataSourceAutoConfiguration` 中有一个专门用来注册数据源的配置类 `PooledDataSourceConfiguration` ，它一次性导入了 SpringBoot 默认支持的所有数据库连接池。

```java
@Configuration(proxyBeanMethods = false)
@Conditional(PooledDataSourceCondition.class)
@ConditionalOnMissingBean({ DataSource.class, XADataSource.class })
@Import({DataSourceConfiguration.Hikari.class, DataSourceConfiguration.Tomcat.class,
         DataSourceConfiguration.Dbcp2.class, DataSourceConfiguration.OracleUcp.class,
         DataSourceConfiguration.Generic.class, DataSourceJmxConfiguration.class })
protected static class PooledDataSourceConfiguration {

}
```

虽说是把所有的配置类都导入进去，但是有条件装配的加持，可以保证最终 IOC 容器中只有一个 `DataSource` 的实现。譬如说 SpringBoot 默认导入的 Hikari 连接池，它会检查当前项目中是否导入了 Hikari 的相关依赖（ `HikariDataSource` ），只有条件都匹配才会生效。

```java
@Configuration(proxyBeanMethods = false)
	@ConditionalOnClass(HikariDataSource.class)
	@ConditionalOnMissingBean(DataSource.class)
	@ConditionalOnProperty(name = "spring.datasource.type", havingValue = "com.zaxxer.hikari.HikariDataSource",
			matchIfMissing = true)
	static class Hikari {

		@Bean
		@ConfigurationProperties(prefix = "spring.datasource.hikari")
		HikariDataSource dataSource(DataSourceProperties properties) {
			HikariDataSource dataSource = createDataSource(properties, HikariDataSource.class);
			if (StringUtils.hasText(properties.getName())) {
				dataSource.setPoolName(properties.getName());
			}
			return dataSource;
		}

	}
```

其他的连接池生效原理同理，不再赘述。

### 1.2 JdbcTemplateAutoConfiguration

spring-jdbc 中支持的最简单的数据库交互工具是 `JdbcTemplate` ，在 SpringBoot 中也有默认的注册，具体表现为 `JdbcTemplateAutoConfiguration` 自动配置类中导入了两个用来注册 `JdbcTemplate` 和 `NamedParameterJdbcTemplate` 的配置类。

```java

@Configuration(
    proxyBeanMethods = false
)
@ConditionalOnClass({DataSource.class, JdbcTemplate.class})
@ConditionalOnSingleCandidate(DataSource.class)
@AutoConfigureAfter({DataSourceAutoConfiguration.class})
@EnableConfigurationProperties({JdbcProperties.class})
@Import({JdbcTemplateConfiguration.class, NamedParameterJdbcTemplateConfiguration.class})
public class JdbcTemplateAutoConfiguration {
    public JdbcTemplateAutoConfiguration() {
    }
}
```

 以 `JdbcTemplateConfiguration` 为例，它构造的 `JdbcTemplate` 依然是传入 `DataSource` ，只不过它针对 SpringBoot 的场景使用中适配了一些外部化配置（譬如 `spring.jdbc.template.max-rows=100` ，指定 `JdbcTemplate` 最多只能查出 100 条数据）当然，如果我们自己注册了 `JdbcTemplate` 后，默认的注册就会自动退出。

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnMissingBean(JdbcOperations.class)
class JdbcTemplateConfiguration {

	@Bean
	@Primary
	JdbcTemplate jdbcTemplate(DataSource dataSource, JdbcProperties properties) {
		JdbcTemplate jdbcTemplate = new JdbcTemplate(dataSource);
		JdbcProperties.Template template = properties.getTemplate();
		jdbcTemplate.setFetchSize(template.getFetchSize());
		jdbcTemplate.setMaxRows(template.getMaxRows());
		if (template.getQueryTimeout() != null) {
			jdbcTemplate.setQueryTimeout((int) template.getQueryTimeout().getSeconds());
		}
		return jdbcTemplate;
	}
}
```

### 1.3 DataSourceTransactionManagerAutoConfiguration

有了数据源，有了操作数据库的工具，要想保证数据操作与业务逻辑的一致性，事务也是必不可少的。spring-jdbc 中针对基础的 `DataSource` 提供了 `DataSourceTransactionManager` 用于基本的事务管理。

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass({ JdbcTemplate.class, TransactionManager.class })
@AutoConfigureOrder(Ordered.LOWEST_PRECEDENCE)
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceTransactionManagerAutoConfiguration {

	@Configuration(proxyBeanMethods = false)
	@ConditionalOnSingleCandidate(DataSource.class)
	static class JdbcTransactionManagerConfiguration {

		@Bean
		@ConditionalOnMissingBean(TransactionManager.class)
		DataSourceTransactionManager transactionManager(Environment environment, DataSource dataSource,
				ObjectProvider<TransactionManagerCustomizers> transactionManagerCustomizers) {
			DataSourceTransactionManager transactionManager = createTransactionManager(environment, dataSource);
			transactionManagerCustomizers.ifAvailable((customizers) -> customizers.customize(transactionManager));
			return transactionManager;
		}

		private DataSourceTransactionManager createTransactionManager(Environment environment, DataSource dataSource) {
			return environment.getProperty("spring.dao.exceptiontranslation.enabled", Boolean.class, Boolean.TRUE)
					? new JdbcTransactionManager(dataSource) : new DataSourceTransactionManager(dataSource);
		}
	}

}

```

阅读上述源码，可能有的小伙伴会感到疑惑：默认情况下注册的事务管理器是 `JdbcTransactionManager` 而不是我们熟悉的 `DataSourceTransactionManager` ，这是为什么呢？于是我们翻开 `JdbcTransactionManager` ，可以发现它本来就是 `DataSourceTransactionManager` 的子类：

```java
public class JdbcTransactionManager extends DataSourceTransactionManager {
    
	@Nullable
	private volatile SQLExceptionTranslator exceptionTranslator;
```

相比起 `DataSourceTransactionManager` 来讲，`JdbcTransactionManager` 的内部多了一个 SQL 的异常翻译机，它可以将 `SQLException` 转换为 `DataAccessException` ，默认的异常翻译机具备识别 `SQLException` 子类、识别异常响应码、错误码等能力。

除此之外，事务管理器的能力依然由 `DataSourceTransactionManager` 提供，负责事务的开启、提交和回滚。

### 1.4 TransactionAutoConfiguration

事务管理的另一部分是编程式事务与声明式事务的支持，在 `TransactionAutoConfiguration` 中有注册 `TransactionTemplate` ，以及使用 `@EnableTransactionManagement` 注解开启注解声明式事务。

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(PlatformTransactionManager.class)
@AutoConfigureAfter({ JtaAutoConfiguration.class, HibernateJpaAutoConfiguration.class,
		DataSourceTransactionManagerAutoConfiguration.class, Neo4jDataAutoConfiguration.class })
@EnableConfigurationProperties(TransactionProperties.class)
public class TransactionAutoConfiguration {
	// ......

	@Configuration(proxyBeanMethods = false)
	@ConditionalOnSingleCandidate(PlatformTransactionManager.class)
	public static class TransactionTemplateConfiguration {

		@Bean
		@ConditionalOnMissingBean(TransactionOperations.class)
		public TransactionTemplate transactionTemplate(PlatformTransactionManager transactionManager) {
			return new TransactionTemplate(transactionManager);
		}
	}

	@Configuration(proxyBeanMethods = false)
	@ConditionalOnBean(TransactionManager.class)
	@ConditionalOnMissingBean(AbstractTransactionManagementConfiguration.class)
	public static class EnableTransactionManagementConfiguration {

		@Configuration(proxyBeanMethods = false)
		@EnableTransactionManagement(proxyTargetClass = false)
		@ConditionalOnProperty(prefix = "spring.aop", name = "proxy-target-class", havingValue = "false")
		public static class JdkDynamicAutoProxyConfiguration { }

		@Configuration(proxyBeanMethods = false)
		@EnableTransactionManagement(proxyTargetClass = true)
		@ConditionalOnProperty(prefix = "spring.aop", name = "proxy-target-class", havingValue = "true", matchIfMissing = true)
		public static class CglibAutoProxyConfiguration { }
	}
}
```

另外，请各位留意一个细节：`TransactionAutoConfiguration` 的类上标注了一个 `@AutoConfigureAfter` ，它指定了一个 `HibernateJpaAutoConfiguration.class` ，为什么事务管理的自动装配需要在这个自动配置类以后呢？如果小伙伴仔细留意观察，可以发现 `HibernateJpaAutoConfiguration` 与 `DataSourceTransactionManagerAutoConfiguration` 是并列的，那是否意味着 `HibernateJpaAutoConfiguration` 也有事务管理器的注册呢？

## 2. SpringDataJPA中的附加组件

SpringDataJPA 除了依赖 spring-jdbc 外，它本身还依赖 Hibernate 以及 JPA 的基础 API ，在 `spring-boot-autoconfigure` 包中用于 SpringDataJPA 的自动配置类主要包含 jpa 包下的 `JpaRepositoriesAutoConfiguration` ，以及 orm 包下的 `HibernateJpaAutoConfiguration` 。下面一一来看。

### 2.1 JpaRepositoriesAutoConfiguration

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnBean(DataSource.class)
@ConditionalOnClass(JpaRepository.class)
@ConditionalOnMissingBean({ JpaRepositoryFactoryBean.class, JpaRepositoryConfigExtension.class })
@ConditionalOnProperty(prefix = "spring.data.jpa.repositories", name = "enabled", havingValue = "true",
		matchIfMissing = true)
@Import(JpaRepositoriesRegistrar.class)
@AutoConfigureAfter({ HibernateJpaAutoConfiguration.class, TaskExecutionAutoConfiguration.class })
public class JpaRepositoriesAutoConfiguration {

```

通过第 10 章的简单了解后，小伙伴应该都清楚，JPA 中操作数据库表需要实体类与之映射，而对应实体类的 Dao 模型也就是 repository ，SpringDataJPA 默认会启用 JPA 的 repository 支持，而支持的方式是使用了一个 `JpaRepositoriesRegistrar` 来实现。下面我们先关注它的作用机制。

#### 2.1.1 JpaRepositoriesRegistrar

对类名敏感的小伙伴大概率可以马上猜测出一点，它应该是一个 `ImportBeanDefinitionRegistrar` ，具备向 IOC 容器动态注册 `BeanDefinition` 的能力。结合源码果然发现是如此：

```java
public abstract class AbstractRepositoryConfigurationSourceSupport
		implements ImportBeanDefinitionRegistrar, BeanFactoryAware, ResourceLoaderAware, EnvironmentAware {

	private ResourceLoader resourceLoader;
	private BeanFactory beanFactory;
	private Environment environment;

	@Override
	public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry,
			BeanNameGenerator importBeanNameGenerator) {
		RepositoryConfigurationDelegate delegate = new RepositoryConfigurationDelegate(
				getConfigurationSource(registry, importBeanNameGenerator), this.resourceLoader, this.environment);
		delegate.registerRepositoriesIn(registry, getRepositoryConfigurationExtension());
	}
```

核心重写的 `registerBeanDefinitions` 方法中可以看出，它借助一个 `RepositoryConfigurationDelegate` 来注册 `BeanDefinition` ，那么问题就来了：这是个什么东西？它注册的 `BeanDefinition` 都是什么？这个 `delegate` 的 `registerRepositoriesIn` 方法传入了一组 `getRepositoryConfigurationExtension()` 返回的数据，这组数据又是什么？不要着急，我们逐个分析。

##### 2.1.1.1 getRepositoryConfigurationExtension

我们先看 `getRepositoryConfigurationExtension` 方法，这个方法来自于抽象类 `AbstractRepositoryConfigurationSourceSupport` ，但实现类却有一大把，借助 IDE 可以发现有非常多的实现类，但如果仔细看的话可以知道，每个具体的实现类刚好对应了一个具体的 SpringData 模块：

![](/img/202301/13springdata.png)

对应到 SpringDataJPA 模块中的实现类自然还是上面我们看到导入的 `JpaRepositoriesRegistrar` ，而它的实现是直接创建了一个 `JpaRepositoryConfigExtension` 对象。

```java
protected abstract RepositoryConfigurationExtension getRepositoryConfigurationExtension();
	

@Override
	protected RepositoryConfigurationExtension getRepositoryConfigurationExtension() {
		return new JpaRepositoryConfigExtension();
	}
```

可能小伙伴读到这里还是摸不着头脑，让我知道这个东西干嘛呢？你倒是起作用呀！在哪里起作用呢？别急，马上揭晓答案。

##### 2.1.1.2 registerRepositoriesIn

回到 `registerBeanDefinitions` 方法中，刚才我们看到了，最终注册 `BeanDefinition` 的核心方法在 `RepositoryConfigurationDelegate` 的 `registerRepositoriesIn` 方法中，它要接收的刚好就是上面的 `JpaRepositoryConfigExtension` 对象。那下面我们就看看 registerRepositoriesIn 方法的内部都实现了哪些关键逻辑，`JpaRepositoryConfigExtension` 又起到了什么作用。
由于 `registerRepositoriesIn` 方法的篇幅较长，下面会拆分为多段讲解。

**1）BeanDefinitionBuilder 的预构造**

```java
public List<BeanComponentDefinition> registerRepositoriesIn(BeanDefinitionRegistry registry,
        RepositoryConfigurationExtension extension) {
    // logger ......
    extension.registerBeansForRoot(registry, configurationSource);

    RepositoryBeanDefinitionBuilder builder = new RepositoryBeanDefinitionBuilder(registry, extension,
            configurationSource, resourceLoader, environment);
    List<BeanComponentDefinition> definitions = new ArrayList<>();

    StopWatch watch = new StopWatch();

    // logger ......
    // ......
```

方法的开始，它构造了一个 `BeanDefinition` 的建造器 `RepositoryBeanDefinitionBuilder` ，各位可以简单地将其理解为 `BeanDefinitionBuilder` ，只不过它创建的 `BeanDefinition` 都是与 repository 相关的。请留意，构造 `RepositoryBeanDefinitionBuilder` 时将刚拿到的 `JpaRepositoryConfigExtension` 传入了进去，说明在其内部会有用武之地。

**2）获取 repository 源的集合**

```java
    // ....
    ApplicationStartup startup = getStartup(registry);
    StartupStep repoScan = startup.start("spring.data.repository.scanning");
    repoScan.tag("dataModule", extension.getModuleName());
    repoScan.tag("basePackages", () -> configurationSource.getBasePackages().stream().collect(Collectors.joining(", ")));
    watch.start();

    Collection<RepositoryConfiguration<RepositoryConfigurationSource>> configurations = extension
            .getRepositoryConfigurations(configurationSource, resourceLoader, inMultiStoreMode);
    // ......
```

在进行指标监测的工作准备完成后，下一个关键的步骤就用到了 `JpaRepositoryConfigExtension` ，它会取出一组 `RepositoryConfiguration` ，意为 repository 的配置。我们可以观察一下获取的方法实现，如下方代码所示。注意观察 for 循环中的元素，可以发现它在循环一组 `BeanDefinition` ，这也就意味着 `configSource.getCandidates(loader)` 方法就已经返回了一组 `BeanDefinition` 对象，随后利用这些 `BeanDefinition` 转换为 `RepositoryConfiguration` 对象。而 `getCandidates` 方法可以扫描的，刚好就是那些继承了 `JpaRepository` 接口的 Dao 接口，for 循环体中的处理也刚好是把这些 Dao 接口一一检查并存入 result 集合中。

```java
public <T extends RepositoryConfigurationSource> Collection<RepositoryConfiguration<T>> getRepositoryConfigurations(
        T configSource, ResourceLoader loader, boolean strictMatchesOnly) {
    // assert ......
    Set<RepositoryConfiguration<T>> result = new HashSet<>();

    for (BeanDefinition candidate : configSource.getCandidates(loader)) {
        RepositoryConfiguration<T> configuration = getRepositoryConfiguration(candidate, configSource);
        // 尝试用类加载器将Dao接口加载出来
        Class<?> repositoryInterface = loadRepositoryInterface(configuration, getConfigurationInspectionClassLoader(loader));
        if (repositoryInterface == null) {
            result.add(configuration);
            continue;
        }
        // 解析Repository接口的泛型类型，并检查是否可以一起返回参与加载
        RepositoryMetadata metadata = AbstractRepositoryMetadata.getMetadata(repositoryInterface);
        boolean qualifiedForImplementation = !strictMatchesOnly || configSource.usesExplicitFilters()
                || isStrictRepositoryCandidate(metadata);
        if (qualifiedForImplementation && useRepositoryConfiguration(metadata)) {
            result.add(configuration);
        }
    }
    return result;
}
```

简单总结下来，在这个环节中，`JpaRepositoryConfigExtension` 的作用是加载这些 `JpaRepository` 接口，并提取出这些元配置信息。

**3）构造 BeanDefinition 并注册**

将上述扫描到的 `JpaRepository` 获取到之后，下面的环节是关键，它会将这些 `JpaRepository` 封装为 `BeanDefinition` ，并注册到 `BeanDefinitionRegistry` 中。注意观察 for 循环中的 `BeanDefinitionBuilder` 对象的生成，它是利用当前方法一开始创建的 `RepositoryBeanDefinitionBuilder` 构造而来。对，各位没有理解错，`RepositoryBeanDefinitionBuilder` 这个家伙构造的对象仍然是一个 Builder （建造[建造器]的建造器）。在 `JpaRepositoryConfigExtension` 对 `BeanDefinitionBuilder` 后置处理完成后，即可获取到真正的 `BeanDefinition` 对象，随后注册到 `BeanDefinitionRegistry` 中，整个逻辑不算很复杂。

```java
 // ......
    Map<String, RepositoryConfiguration<?>> configurationsByRepositoryName = new HashMap<>(configurations.size());

    for (RepositoryConfiguration<? extends RepositoryConfigurationSource> configuration : configurations) {
        configurationsByRepositoryName.put(configuration.getRepositoryInterface(), configuration);
        BeanDefinitionBuilder definitionBuilder = builder.build(configuration);
        extension.postProcess(definitionBuilder, configurationSource);
        if (isXml) {
            extension.postProcess(definitionBuilder, (XmlRepositoryConfigurationSource) configurationSource);
        } else {
            extension.postProcess(definitionBuilder, (AnnotationRepositoryConfigurationSource) configurationSource);
        }
        AbstractBeanDefinition beanDefinition = definitionBuilder.getBeanDefinition();
        beanDefinition.setResourceDescription(configuration.getResourceDescription());

        String beanName = configurationSource.generateBeanName(beanDefinition);
        // logger ......
        beanDefinition.setAttribute(FACTORY_BEAN_OBJECT_TYPE, configuration.getRepositoryInterface());
        
        registry.registerBeanDefinition(beanName, beanDefinition);
        definitions.add(new BeanComponentDefinition(beanDefinition, beanName));
    }
    // ......
```

请注意一个非常关键的动作：`BeanDefinitionBuilder definitionBuilder = builder.build(configuration);` ，这个动作可以生成一个 `BeanDefinitionBuilder` ，我们进入到这个方法的实现中，从方法的开头可以得到一个至关重要的信息：

```java
public BeanDefinitionBuilder build(RepositoryConfiguration<?> configuration) {
    // assert ......
    BeanDefinitionBuilder builder = BeanDefinitionBuilder
            .rootBeanDefinition(configuration.getRepositoryFactoryBeanClassName());
    // ......
    return builder;
}
```

请注意观察，`BeanDefinition` 中的核心信息之一：`beanClassName` ，取得是 `configuration` 中存储的 `RepositoryFactoryBean` 的类型名称，而进入到 `JpaRepositoryConfigExtension` 中，可以发现它指定的类型是：

```java
public String getRepositoryFactoryBeanClassName() {
    return JpaRepositoryFactoryBean.class.getName();
}
```

折腾这么多，想给各位小伙伴讲解的一个关键点：每个 `JpaRepository` 接口的实现类，其实是由 **`JpaRepositoryFactoryBean`** 创建而来。

进一步翻看 `JpaRepositoryFactoryBean` 中的 `getObject` 方法，可以发现它创建的对象其实是由 SpringFramework 内部的 API 创建的接口动态代理对象，动态代理的原始目标对象类型是 **`JpaRepositoryImplementation`** ，感兴趣的小伙伴可以深入 `RepositoryFactorySupport#getRepository` 方法中一探究竟，源码篇幅较多，小册不再贴出。

**4）收尾工作**

注册完 `BeanDefinition` 后，剩下的只有一些收尾工作，主线逻辑已经执行完毕，细枝末节我们不再关心。

```java
    // ......
    potentiallyLazifyRepositories(configurationsByRepositoryName, registry, configurationSource.getBootstrapMode());

    watch.stop();

    repoScan.tag("repository.count", Integer.toString(configurations.size()));
    repoScan.end();

    // logger ......
    return definitions;
}
```

#### 2.1.2 JpaRepositoriesAutoConfiguration中的其他组件

`JpaRepositoriesAutoConfiguration` 中除了导入最重要的 `JpaRepositoriesRegistrar` 之外，在配置类中还有一个组件的注册：`EntityManagerFactoryBuilderCustomizer` 。从类名可以很容易了解到，它是一个定制器，而观察源码可以发现，它主要实现的功能是配置异步任务执行。

```java
@Bean
@Conditional(BootstrapExecutorCondition.class)
public EntityManagerFactoryBuilderCustomizer entityManagerFactoryBootstrapExecutorCustomizer(
        Map<String, AsyncTaskExecutor> taskExecutors) {
    return (builder) -> {
        AsyncTaskExecutor bootstrapExecutor = determineBootstrapExecutor(taskExecutors);
        if (bootstrapExecutor != null) {
            builder.setBootstrapExecutor(bootstrapExecutor);
        }
    };
}

private AsyncTaskExecutor determineBootstrapExecutor(Map<String, AsyncTaskExecutor> taskExecutors) {
    if (taskExecutors.size() == 1) {
        return taskExecutors.values().iterator().next();
    }
    return taskExecutors.get(TaskExecutionAutoConfiguration.APPLICATION_TASK_EXECUTOR_BEAN_NAME);
}
```

到此为止，与 JPA 的 Repository 接口相关的配置就全部了解完了。

### 2.2 HibernateJpaAutoConfiguration

不要忘记，JPA 仅仅是一个规范，而最终能用的还得是实现框架，Hibernate 作为 SpringDataJPA 的默认实现，它也有一个与 JPA 相关的自动装配（在 `JpaRepositoriesAutoConfiguration` 的 `@AutoConfigureAfter` 上我们也看到了），我们也要来了解一下。

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass({ LocalContainerEntityManagerFactoryBean.class, EntityManager.class, SessionImplementor.class })
@EnableConfigurationProperties(JpaProperties.class)
@AutoConfigureAfter({ DataSourceAutoConfiguration.class })
@Import(HibernateJpaConfiguration.class)
public class HibernateJpaAutoConfiguration {

}
```

非常干净的自动配置类，它导入了一个 `HibernateJpaConfiguration` ，我们继续往下看。

#### 2.2.1 HibernateJpaConfiguration

```java
@Configuration(proxyBeanMethods = false)
@EnableConfigurationProperties(HibernateProperties.class)
@ConditionalOnSingleCandidate(DataSource.class)
class HibernateJpaConfiguration extends JpaBaseConfiguration {
```

注意观察，`HibernateJpaConfiguration` 本身还继承了 `JpaBaseConfiguration` ，稍后我们再来看它。另外这个类上有一个特殊的条件装配注解：`@ConditionalOnSingleCandidate` ，它只希望一个应用中存在一个数据源，如果我们做了多个数据源在 IOC 容器中，那么这个配置类将不再会生效。

纵观整个 `HibernateJpaConfiguration` 的内容，我们并没有找到有 `@Bean` 注解标注的方法，而是重写了一大堆父类的方法，那我们就继续向上扒父类吧

#### 2.2.2 JpaBaseConfiguration

`JpaBaseConfiguration` 没有继续向上继承父类了，所以它就是整个 JPA 配置的基础类。在这个类中定义了不少 `@Bean` 注解标注的方法，一一来看。

##### 2.2.2.1 PlatformTransactionManager

```java
@Bean
@ConditionalOnMissingBean(TransactionManager.class)
public PlatformTransactionManager transactionManager(
        ObjectProvider<TransactionManagerCustomizers> transactionManagerCustomizers) {
    JpaTransactionManager transactionManager = new JpaTransactionManager();
    transactionManagerCustomizers.ifAvailable((customizers) -> customizers.customize(transactionManager));
    return transactionManager;
}
```

事务管理器，作为 jdbc 模块的必备组件，Hibernate 为此准备了特制的事务管理器 `JpaTransactionManager` 。与 `DataSourceTransactionManager` 不同，Hibernate 的架构中以 `SessionFactory` （即 JPA 中的 `EntityManagerFactory` ）为基准进行事务管理，所以 `JpaTransactionManager` 对应的其实是 `EntityManagerFactory` 。

不过从具体的工作效果来看，`JpaTransactionManager` 与 `DataSourceTransactionManager` 的最终效果并没有什么差别，所以各位大可直接类比。

##### 2.2.2.2 JpaVendorAdapter

```java
@Bean
@ConditionalOnMissingBean
public JpaVendorAdapter jpaVendorAdapter() {
    AbstractJpaVendorAdapter adapter = createJpaVendorAdapter();
    adapter.setShowSql(this.properties.isShowSql());
    if (this.properties.getDatabase() != null) {
        adapter.setDatabase(this.properties.getDatabase());
    }
    if (this.properties.getDatabasePlatform() != null) {
        adapter.setDatabasePlatform(this.properties.getDatabasePlatform());
    }
    adapter.setGenerateDdl(this.properties.isGenerateDdl());
    return adapter;
}
```

`JpaVendorAdapter` 本身是一个接口，它的作用是给 JPA 规范的实现框架一个扩展的机制，让每个实现 JPA 规范的框架都能植入自己独有的特性。

从源码角度来看，`createJpaVendorAdapter` 方法本身是一个抽象方法，而具体的实现则是由每个 JPA 规范实现的框架负责重写，Hibernate 中提供的实现类是 `HibernateJpaVendorAdapter` ，它提供了一些数据库方言、配置项的扩展等特性。

##### 2.2.2.3 EntityManagerFactoryBuilder

```java
@Bean
@ConditionalOnMissingBean
public EntityManagerFactoryBuilder entityManagerFactoryBuilder(JpaVendorAdapter jpaVendorAdapter,
        ObjectProvider<PersistenceUnitManager> persistenceUnitManager,
        ObjectProvider<EntityManagerFactoryBuilderCustomizer> customizers) {
    EntityManagerFactoryBuilder builder = new EntityManagerFactoryBuilder(jpaVendorAdapter,
            this.properties.getProperties(), persistenceUnitManager.getIfAvailable());
    customizers.orderedStream().forEach((customizer) -> customizer.customize(builder));
    return builder;
}
```

`EntityManagerFactoryBuilder` ，从类名上就能非常直观地了解到，它就是创建 `EntityManagerFactory` 的建造器，从构造的过程来看，它应用了一组 `EntityManagerFactoryBuilderCustomizer` 来实现 `EntityManagerFactoryBuilder` 的定制化配置，从而间接实现了 `EntityManagerFactory` 的定制化。

`EntityManagerFactoryBuilder` 的作用是辅助创建 `EntityManagerFactory` ，而真正用来创建的是下面的 `LocalContainerEntityManagerFactoryBean` 。

##### 2.2.2.4 LocalContainerEntityManagerFactoryBean

```java
@Bean
@Primary
@ConditionalOnMissingBean({ LocalContainerEntityManagerFactoryBean.class, EntityManagerFactory.class })
public LocalContainerEntityManagerFactoryBean entityManagerFactory(EntityManagerFactoryBuilder factoryBuilder) {
    Map<String, Object> vendorProperties = getVendorProperties();
    customizeVendorProperties(vendorProperties);
    return factoryBuilder.dataSource(this.dataSource).packages(getPackagesToScan()).properties(vendorProperties)
            .mappingResources(getMappingResources()).jta(isJta()).build();
}
```

从类名上看，`LocalContainerEntityManagerFactoryBean` 创建的实际对象是 `LocalContainerEntityManagerFactory` ，这也就指明了在默认情况下 `EntityManagerFactory` 的实现类类型。

通过查看 `getObject` 可以发现，它获取到的居然是一个已经创建好的 `EntityManagerFactory` ，而非每次调用都全新创建的：

```java
@Override
@Nullable
public EntityManagerFactory getObject() {
    return this.entityManagerFactory;
}
```

那这个 `EntityManagerFactory` 是什么时候创建的呢？这就得找 `this.entityManagerFactory` 的赋值动作了，从 `afterPropertiesSet` 方法中，我们可以看到创建的动作：

```java
@Override
public void afterPropertiesSet() throws PersistenceException {
    JpaVendorAdapter jpaVendorAdapter = getJpaVendorAdapter();
    if (jpaVendorAdapter != null) {
        // 好多附加逻辑 ......
    }

    AsyncTaskExecutor bootstrapExecutor = getBootstrapExecutor();
    if (bootstrapExecutor != null) {
        this.nativeEntityManagerFactoryFuture = bootstrapExecutor.submit(this::buildNativeEntityManagerFactory);
    }
    else {
        this.nativeEntityManagerFactory = buildNativeEntityManagerFactory();
    }

    // 在此处，是真正创建EntityManagerFactory的位置...
    this.entityManagerFactory = createEntityManagerFactoryProxy(this.nativeEntityManagerFactory);
}
```

原生的 MyBatis 整合 SpringFramework 时，创建 `SqlSessionFactory` 的逻辑也是靠 `afterPropertiesSet` 方法完成的！

## 3. SpringDataRedis中的附加组件

看完了 SpringDataJPA 整合的内容，下面我们再来看看 SpringDataRedis 场景中底层都做了哪些自动装配的工作。从 `spring.factories` 中可以找到，核心的自动配置类有两个，一一来看。

### 3.1 RedisAutoConfiguration

从类名上就可以很明确的看出，它就是装配 Redis 相关组件的核心自动装配，而它内部注册的组件也不多，就一个全类型通用的 `RedisTemplate` ，以及一个只支持 String 类型的 `StringRedisTemplate` ，注册的逻辑都相当简单：

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(RedisOperations.class)
@EnableConfigurationProperties(RedisProperties.class)
@Import({ LettuceConnectionConfiguration.class, JedisConnectionConfiguration.class })
public class RedisAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean(name = "redisTemplate")
    @ConditionalOnSingleCandidate(RedisConnectionFactory.class)
    public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
        RedisTemplate<Object, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory);
        return template;
    }

    @Bean
    @ConditionalOnMissingBean
    @ConditionalOnSingleCandidate(RedisConnectionFactory.class)
    public StringRedisTemplate stringRedisTemplate(RedisConnectionFactory redisConnectionFactory) {
        StringRedisTemplate template = new StringRedisTemplate();
        template.setConnectionFactory(redisConnectionFactory);
        return template;
    }
}
```

除此之外，`RedisAutoConfiguration` 还使用 `@Import` 导入了两个 Redis 连接工具的配置类，分别是我们熟悉的 Jedis 和 Lettuce 。SpringBoot 1.x 默认使用的是 Jedis ，而到了 SpringBoot 2.x 后默认的连接工具变为 Lettuce 。

以 Lettuce 为例，由于默认情况下 SpringBoot 2.x 本就选择 Lettuce 作为连接工具，所以条件装配中 `@ConditionalOnProperty` 注解中就设置了 `matchIfMissing = true` 。而在配置类内部，它注册的核心组件就是 `RedisConnectionFactory` 的具体实现 `LettuceConnectionFactory` 。

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(RedisClient.class)
@ConditionalOnProperty(name = "spring.redis.client-type", havingValue = "lettuce", matchIfMissing = true)
class LettuceConnectionConfiguration extends RedisConnectionConfiguration {
        // ......

        @Bean
        @ConditionalOnMissingBean(RedisConnectionFactory.class)
        LettuceConnectionFactory redisConnectionFactory(
                        ObjectProvider<LettuceClientConfigurationBuilderCustomizer> builderCustomizers,
                        ClientResources clientResources) {
                LettuceClientConfiguration clientConfig = getLettuceClientConfiguration(builderCustomizers, clientResources,
                                getProperties().getLettuce().getPool());
                return createLettuceConnectionFactory(clientConfig);
        }
```

### 3.2 RedisRepositoriesAutoConfiguration

另一个核心的自动装配是 `RedisRepositoriesAutoConfiguration` ，它的作用是导入一个 `RedisRepositoriesRegistrar` 。

```java
@Configuration(proxyBeanMethods = false)
// ......
@Import(RedisRepositoriesRegistrar.class)
@AutoConfigureAfter(RedisAutoConfiguration.class)
public class RedisRepositoriesAutoConfiguration {

}
```

参照 `JpaRepositoriesRegistrar` ，我们很容易就能想到，其实 SpringDataRedis 也是可以使用 Repository 机制的，只不过我们通常都不会使用，而是直接使用 `RedisTemplate` 来完成与 Redis 的交互。

SpringDataRedis 的自动装配就这么多，没有特别多需要额外介绍的，下面是 SpringDataMongoDB 。

## 4. SpringDataMongoDB中的附加组件

---

SpringDataMongoDB 的装配内容，大体上也跟 SpringDataJPA 、SpringDataRedis 大体相似，我们简单过一遍即可。

### 4.1 MongoDataAutoConfiguration

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass({ MongoClient.class, MongoTemplate.class })
@EnableConfigurationProperties(MongoProperties.class)
@Import({ MongoDataConfiguration.class, MongoDatabaseFactoryConfiguration.class,
        MongoDatabaseFactoryDependentConfiguration.class })
@AutoConfigureAfter(MongoAutoConfiguration.class)
public class MongoDataAutoConfiguration {

}

```

`MongoDataAutoConfiguration` 作为核心自动装配类，它本身没有做什么事，而是直接用 `@Import` 注解导入了 3 个配置类。

### 4.2 MongoDataConfiguration

```java
@Configuration(proxyBeanMethods = false)
class MongoDataConfiguration {

    @Bean
    @ConditionalOnMissingBean
    MongoMappingContext mongoMappingContext(ApplicationContext applicationContext, MongoProperties properties,
            MongoCustomConversions conversions) throws ClassNotFoundException {
        PropertyMapper mapper = PropertyMapper.get().alwaysApplyingWhenNonNull();
        MongoMappingContext context = new MongoMappingContext();
        mapper.from(properties.isAutoIndexCreation()).to(context::setAutoIndexCreation);
        context.setInitialEntitySet(new EntityScanner(applicationContext).scan(Document.class));
        Class<?> strategyClass = properties.getFieldNamingStrategy();
        if (strategyClass != null) {
            context.setFieldNamingStrategy((FieldNamingStrategy) BeanUtils.instantiateClass(strategyClass));
        }
        context.setSimpleTypeHolder(conversions.getSimpleTypeHolder());
        return context;
    }

    @Bean
    @ConditionalOnMissingBean
    MongoCustomConversions mongoCustomConversions() {
        return new MongoCustomConversions(Collections.emptyList());
    }
}
```

`MongoDataConfiguration` 中注册的核心组件是用于完成实体模型类与 MongoDB 中文档的映射关系的上下文 `MongoMappingContext` ，有了它之后，SpringDataMongoDB 就拥有了根据实体模型类定义的映射关系，自动创建 MongoDB 的集合等能力。

### 4.3 MongoDatabaseFactoryConfiguration

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnMissingBean(MongoDatabaseFactory.class)
@ConditionalOnSingleCandidate(MongoClient.class)
class MongoDatabaseFactoryConfiguration {

    @Bean
    MongoDatabaseFactorySupport<?> mongoDatabaseFactory(MongoClient mongoClient, MongoProperties properties) {
        return new SimpleMongoClientDatabaseFactory(mongoClient, properties.getMongoClientDatabase());
    }
}
```

`MongoDatabaseFactoryConfiguration` 只注册了一个组件：`MongoDatabaseFactorySupport` ，它比较类似于 `DataSource` 的管理器，有了它就可以找到程序连接的 MongoDB 数据库，默认的 `SimpleMongoClientDatabaseFactory` 就是连接一个 MongoDB 数据库，如果我们对其进行扩展，甚至可以做到 MongoDB 数据库的动态切换。

### 4.4 MongoDatabaseFactoryDependentConfiguration

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnBean(MongoDatabaseFactory.class)
class MongoDatabaseFactoryDependentConfiguration {
    // ......

    @Bean
    @ConditionalOnMissingBean(MongoOperations.class)
    MongoTemplate mongoTemplate(MongoDatabaseFactory factory, MongoConverter converter) {
        return new MongoTemplate(factory, converter);
    }
    
    @Bean
    @ConditionalOnMissingBean(MongoConverter.class)
    MappingMongoConverter mappingMongoConverter(MongoDatabaseFactory factory, MongoMappingContext context,
            MongoCustomConversions conversions) {
        // ......
    }
    
    // ......
```

`MongoDatabaseFactoryDependentConfiguration` 中核心注册的组件，必然是跟前面几个模块一样的模板操作类 `MongoTemplate` ，它的创建需要依赖 `MongoDatabaseFactory` 以及下面的 `MongoConverter` 。

### 4.5 MongoRepositoriesAutoConfiguration

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass({ MongoClient.class, MongoRepository.class })
@ConditionalOnMissingBean({ MongoRepositoryFactoryBean.class, MongoRepositoryConfigurationExtension.class })
@ConditionalOnRepositoryType(store = "mongodb", type = RepositoryType.IMPERATIVE)
@Import(MongoRepositoriesRegistrar.class)
@AutoConfigureAfter(MongoDataAutoConfiguration.class)
public class MongoRepositoriesAutoConfiguration {

}
```

与前两者几乎完全相同，它也是导入了一个 Registrar ，用它来支持 Repository 机制。

【到此为止，有关 SpringData 部分的讲解就告一段落。SpringData 的模块非常多，小册介绍的也仅仅是使用频率比较多的模块，当然从使用方式和底层原理上看，我们也都能看得出，SpringData 的各个模块，在使用方式上都大同小异，所以各位在实际使用时无需担心，触类旁通说的可能就是 SpringData 这种吧！】

