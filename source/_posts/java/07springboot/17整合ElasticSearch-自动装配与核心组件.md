---
title: 17整合ElasticSearch-自动装配与核心组件
toc: true
tags: springboot
categories: 
    - [java]
    - [springboot]
---

好啦，又到了扒源码讲原理的环节，与前面的 SpringData 系列的其他模块不同，对于 SpringDataElasticSearch 来讲，我打算不止讲自动装配的内容，还想给大家看看，新版的 `ElasticsearchRestTemplate` 在底层是如何跟 `RestHighLevelClient` 进行比较好地结合的。

## 1. 自动装配

---

我们先来看自动装配的部分。上一章的最后，我们已经看了部分的自动装配内容，这一章我们来全面梳理整合 ElasticSearch 的组件装配。

从 spring.factories 中注册的与 ElasticSearch 相关的组件，可以得知有如下的几个自动配置类。

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.autoconfigure.data.elasticsearch.ElasticsearchDataAutoConfiguration,\
org.springframework.boot.autoconfigure.data.elasticsearch.ElasticsearchRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.elasticsearch.ReactiveElasticsearchRepositoriesAutoConfiguration,\
org.springframework.boot.autoconfigure.data.elasticsearch.ReactiveElasticsearchRestClientAutoConfiguration,\
org.springframework.boot.autoconfigure.elasticsearch.ElasticsearchRestClientAutoConfiguration,\
```

注意 **Reactive** 开头的自动配置类是与响应式 ElasticSearch 模块相关的，不在我们本次的研究范围之内，感兴趣的小伙伴可以自行深入探究。

### 1.1 ElasticsearchDataAutoConfiguration

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass({ ElasticsearchRestTemplate.class })
@AutoConfigureAfter({ ElasticsearchRestClientAutoConfiguration.class,
        ReactiveElasticsearchRestClientAutoConfiguration.class })
@Import({ ElasticsearchDataConfiguration.BaseConfiguration.class,
        ElasticsearchDataConfiguration.RestClientConfiguration.class,
        ElasticsearchDataConfiguration.ReactiveRestClientConfiguration.class })
public class ElasticsearchDataAutoConfiguration {

}
```

第一个自动配置类 `ElasticsearchDataAutoConfiguration` 中并没有注册任何组件，而是一下子导入了另外三个内部类，虽然这没什么好惊讶的，不过我们也要稍加留意一点，对于 SpringDataElasticSearch 的自动配置，底层好多都是这么写的哦。

下面我们就来看导入的三个配置类。

#### 1.1.1 BaseConfiguration

从类名上就能得知一点，`BaseConfiguration` 主要负责的是基础组件的注册，在源码中它注册了 3 个基础的组件，一一来看。

##### 1）ElasticsearchCustomConversions

```java
    @Bean
    @ConditionalOnMissingBean
    ElasticsearchCustomConversions elasticsearchCustomConversions() {
        return new ElasticsearchCustomConversions(Collections.emptyList());
    }
```

第一个注册的 Bean 是 `ElasticsearchCustomConversions` ，这是个什么东西呢？

如果各位还记得 SpringFramework 里的 `ConversionService` ，或许应该能联想到吧，这个家伙应该是做类型转换的！

何以证明？我们可以点开它的源码，从它的静态代码块中可以看到端倪：

```java
public class ElasticsearchCustomConversions extends CustomConversions {
    // ......
    static {
        List<Converter<?, ?>> converters = new ArrayList<>(GeoConverters.getConvertersToRegister());
        converters.add(StringToUUIDConverter.INSTANCE);
        converters.add(UUIDToStringConverter.INSTANCE);
        converters.add(BigDecimalToDoubleConverter.INSTANCE);
        converters.add(DoubleToBigDecimalConverter.INSTANCE);
        converters.add(ByteArrayToBase64Converter.INSTANCE);
        converters.add(Base64ToByteArrayConverter.INSTANCE);

        STORE_CONVERTERS = Collections.unmodifiableList(converters);
        STORE_CONVERSIONS = StoreConversions.of(ElasticsearchSimpleTypes.HOLDER, STORE_CONVERTERS);
    }
```

它初始化了一堆 Converter ，并且传入了一系列带有转换规则的 Converter 实例！这不就相当于石锤了**数据类型转换器**的作用了嘛！SpringDataElasticSearch 肯定需要这种组件来实现不同类型数据之间的转换，以完成与 ElasticSearch 交互过程中的数据存取工作。

##### 2）SimpleElasticsearchMappingContext

```java
    @Bean
    @ConditionalOnMissingBean
    SimpleElasticsearchMappingContext mappingContext(ApplicationContext applicationContext,
            ElasticsearchCustomConversions elasticsearchCustomConversions) throws ClassNotFoundException {
        SimpleElasticsearchMappingContext mappingContext = new SimpleElasticsearchMappingContext();
        mappingContext.setInitialEntitySet(new EntityScanner(applicationContext).scan(Document.class));
        mappingContext.setSimpleTypeHolder(elasticsearchCustomConversions.getSimpleTypeHolder());
        return mappingContext;
    }
```

接下来的组件 `SimpleElasticsearchMappingContext` ，我们还是先从类名上猜测：映射上下文？谁跟谁映射呢？肯定不是上面的那些类型转换了吧！想不出来没关系，我们还是点开源码，看一眼它的继承关系：

```
public class SimpleElasticsearchMappingContext
		extends AbstractMappingContext<SimpleElasticsearchPersistentEntity<?>, 
                ElasticsearchPersistentProperty> {
```

划重点：**Persistent** 、**Entity** ，这两个词似乎已经帮我们揭开迷惑了！其实 `SimpleElasticsearchMappingContext` 负责的就是我们在使用 Entity 和 Repository 的过程中，编写的 **“Entity 实体类” 与 “ElasticSearch 中存放的文档” 之间的映射关系**！

##### 3）ElasticsearchConverter

```java
@Bean
@ConditionalOnMissingBean
ElasticsearchConverter elasticsearchConverter(SimpleElasticsearchMappingContext mappingContext,
                            ElasticsearchCustomConversions elasticsearchCustomConversions) {
  MappingElasticsearchConverter converter = new MappingElasticsearchConverter(mappingContext);
  converter.setConversions(elasticsearchCustomConversions);
  return converter;
}
```

等一下，又一个转换器？为什么会出现两个转换器呢？如果小伙伴也有这个疑惑，那请你仔细观察下方法的参数列表 ~ 原来上面两个家伙的注册，最终是为了集成到 `ElasticsearchConverter` 中，由 `ElasticsearchConverter` 负责真正的实体类对象与 ElasticSearch 中的文档之间互相转换。

#### 1.1.2 RestClientConfiguration

下一个配置类就是在上一章中我们看到的 `RestClientConfiguration` ，它只注册一个组件：`ElasticsearchRestTemplate` ，不再重复啰嗦。

```java

@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(RestHighLevelClient.class)
static class RestClientConfiguration {

  @Bean
  @ConditionalOnMissingBean(value = ElasticsearchOperations.class, name = "elasticsearchTemplate")
  @ConditionalOnBean(RestHighLevelClient.class)
  ElasticsearchRestTemplate elasticsearchTemplate(RestHighLevelClient client, 
                                                  ElasticsearchConverter converter) {
    return new ElasticsearchRestTemplate(client, converter);
  }

}
```

#### 1.1.3 ReactiveRestClientConfiguration

与 `RestClientConfiguration` 相似的还有下面的 `ReactiveRestClientConfiguration` ，它可以注册基于响应式的 `ReactiveElasticsearchTemplate` 。但是与注册 `ElasticsearchRestTemplate` 的条件不同，由于响应式需要一些特殊的组件，所以这里想注册 `ReactiveElasticsearchTemplate` ，需要依赖基于 Reactive 的 `ReactiveElasticsearchClient` ，而这部分组件需要导入响应式 ElasticSearch 交互才可以注册，所以我们就不深入探究了，各位知道一下就行。

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass({ WebClient.class, ReactiveElasticsearchOperations.class })
static class ReactiveRestClientConfiguration {

  @Bean
  @ConditionalOnMissingBean(value = ReactiveElasticsearchOperations.class, 
                            name = "reactiveElasticsearchTemplate")
  @ConditionalOnBean(ReactiveElasticsearchClient.class)
  ReactiveElasticsearchTemplate reactiveElasticsearchTemplate(ReactiveElasticsearchClient client,
                                                              ElasticsearchConverter converter) {
    ReactiveElasticsearchTemplate template = new ReactiveElasticsearchTemplate(client, converter);
    template.setIndicesOptions(IndicesOptions.strictExpandOpenAndForbidClosed());
    return template;
  }

}
```

以上就是有关 `ElasticsearchDataAutoConfiguration` 的组件注册了，这本身没什么好说的，概括下来就是一个 Template + 一个类型转换器。

### 1.2 ElasticsearchRepositoriesAutoConfiguration

接下来我们继续往下看。SpringData 提倡的统一玩法是 Entity 与 Repository 接口一把梭，而 Repository 的注册与之前我们看到的其他 SpringData 模块相似，都是采用 Registrar 的方式扫描和注册。

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass({ Client.class, ElasticsearchRepository.class })
@ConditionalOnProperty(prefix = "spring.data.elasticsearch.repositories", 
                       name = "enabled", havingValue = "true", matchIfMissing = true)
@ConditionalOnMissingBean(ElasticsearchRepositoryFactoryBean.class)
@Import(ElasticsearchRepositoriesRegistrar.class)
public class ElasticsearchRepositoriesAutoConfiguration {

}
```

从源码中可以看到，它注册的组件果然还是一个 `ElasticsearchRepositoriesRegistrar` ，这不就类似于 SpringDataJPA 中的 `JpaRepositoriesRegistrar` 嘛！果然这些全家桶都是池里的王八河里的鳖 —— 一色货啊！

我们赶紧点进去源码看一眼，非常惊奇的一幕是整个类的源码并没有多少，甚至都只是重写了几个方法，还有标注了 `@EnableElasticsearchRepositories` 注解开启 Repository 接口的扫描。

```java
class ElasticsearchRepositoriesRegistrar extends AbstractRepositoryConfigurationSourceSupport {

	@Override
	protected Class<? extends Annotation> getAnnotation() {
		return EnableElasticsearchRepositories.class;
	}

	@Override
	protected Class<?> getConfiguration() {
		return EnableElasticsearchRepositoriesConfiguration.class;
	}

	@Override
	protected RepositoryConfigurationExtension getRepositoryConfigurationExtension() {
		return new ElasticsearchRepositoryConfigExtension();
	}

	@EnableElasticsearchRepositories
	private static class EnableElasticsearchRepositoriesConfiguration {

	}

}
```

那逻辑都去哪里了呢？还记得第 13 章中我们讲 `JpaRepositoriesRegistrar` 的源码中，曾经就见过这个父类 `AbstractRepositoryConfigurationSourceSupport` 吗？如果小伙伴忘记了，可一定要回过头去看第 13 章的 2.1.1 节啊！小册就不在这里重复啰嗦了。

### 1.3 ElasticsearchRestClientAutoConfiguration

最后一个配置类 `ElasticsearchRestClientAutoConfiguration` ，其实我们在上一章已经完全看过了，这里就简单快速地回顾下吧。

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(RestHighLevelClient.class)
@ConditionalOnMissingBean(RestClient.class)
@EnableConfigurationProperties(ElasticsearchRestClientProperties.class)
@Import({ RestClientBuilderConfiguration.class, RestHighLevelClientConfiguration.class,
		RestClientSnifferConfiguration.class })
public class ElasticsearchRestClientAutoConfiguration {

}
```

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnMissingBean(RestClientBuilder.class)
static class RestClientBuilderConfiguration {
    // ......

    // 注册RestClientBuilder，以用来注册下方的RestHighLevelClient
    @Bean
    RestClientBuilder elasticsearchRestClientBuilder(ElasticsearchRestClientProperties properties,
            ObjectProvider<RestClientBuilderCustomizer> builderCustomizers) {
        HttpHost[] hosts = properties.getUris().stream().map(this::createHttpHost).toArray(HttpHost[]::new);
        RestClientBuilder builder = RestClient.builder(hosts);
        // ......
        builderCustomizers.orderedStream().forEach((customizer) -> customizer.customize(builder));
        return builder;
    }
```

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnMissingBean(RestHighLevelClient.class)
static class RestHighLevelClientConfiguration {

    // 注册RestHighLevelClient，用于创建ElasticsearchRestTemplate
    @Bean
    RestHighLevelClient elasticsearchRestHighLevelClient(RestClientBuilder restClientBuilder) {
        return new RestHighLevelClient(restClientBuilder);
    }
}
```

可以发现，对于 SpringDataElasticSearch 模块而言，它并没有什么很新的东西，基本都是之前那一套的东西。其实这样也好，既给我们开发者统一了使用方式，同时尽可能统一的底层在后期官方开发人员维护时也更容易，此可谓“高质量代码”了。

## 2. ElasticsearchRestTemplate的工作机制

----

最后我们来看一下 `ElasticsearchRestTemplate` 的工作机制，再次从底层源码的角度切入，探究 SpringData 在底层都帮我们干了什么，同时也再次印证一下新版本的 SpringDataElasticSearch 的特性完全可以使用。

### 2.1 构造器

进入 `ElasticsearchRestTemplate` 的源码，从构造器上就可以看得出来，`ElasticsearchRestTemplate` 的内部是组合了一个 `RestHighLevelClient` ，后面的交互也全都是基于这个 client 完成发起。

```java
public class ElasticsearchRestTemplate extends AbstractElasticsearchTemplate {

    private static final Logger LOGGER = LoggerFactory.getLogger(ElasticsearchRestTemplate.class);

    private final RestHighLevelClient client;
    private final ElasticsearchExceptionTranslator exceptionTranslator = 
      								new ElasticsearchExceptionTranslator();

    // region Initialization
    public ElasticsearchRestTemplate(RestHighLevelClient client) {
        Assert.notNull(client, "Client must not be null!");
        this.client = client;
        initialize(createElasticsearchConverter());
    }

    public ElasticsearchRestTemplate(RestHighLevelClient client, 
                                     ElasticsearchConverter elasticsearchConverter) {
        Assert.notNull(client, "Client must not be null!");
        this.client = client;
        initialize(elasticsearchConverter);
    }
```

### 2.2 索引数据的动作

索引数据使用的方法是 `save` ，我们点进源码中看一眼。

#### 2.2.1 定位索引

```java
public <T> T save(T entity) {
    Assert.notNull(entity, "entity must not be null");
    return save(entity, getIndexCoordinatesFor(entity.getClass()));
}
```

可以看到，在非空判定之后，它会先根据传入的实体类对象的类型，寻找到对应 ElasticSearch 中的索引名称。怎么去找呢？从源码中可以看出，它根据实体类类型信息，找到映射过去的信息，获取一个 `IndexCoordinates` ：

```java
public IndexCoordinates getIndexCoordinatesFor(Class<?> clazz) {
    return getRequiredPersistentEntity(clazz).getIndexCoordinates();
}
```

这个 `IndexCoordinates` 是个啥呢？其实就是 `@Document(indexName = "xxx")` ，从源码中的成员属性就看出来了：

```java
public class IndexCoordinates {
    // ......
    
    private final String[] indexNames;
```

`indexNames` ，这个名不要太直接，所以 `save` 方法的第一步动作就是定位这条数据，最终要索引到哪些索引中。

#### 2.2.2 索引动作

接下来的动作就是如何将数据索引到 ElasticSearch 中了。进入重载的两参数 `save` 方法，别的咱不说，就中间的那行 **`doIndex`** 方法，各位是不是觉得特别扎眼！想都不要想，直接点击去就完事了！

```java
public <T> T save(T entity, IndexCoordinates index) {
    // assert ......
    T entityAfterBeforeConvert = maybeCallbackBeforeConvert(entity, index);

    IndexQuery query = getIndexQuery(entityAfterBeforeConvert);
    doIndex(query, index);

    T entityAfterAfterSave = maybeCallbackAfterSave(entityAfterBeforeConvert, index);
    return entityAfterAfterSave;
}
```

点进来之后，我们立马就看到了熟悉的操作：**`client.index(request, RequestOptions.DEFAULT)`** ！这不就是操作 `RestHighLevelClient` 的 `index` 方法索引数据嘛！

```java
public String doIndex(IndexQuery query, IndexCoordinates index) {
    IndexRequest request = prepareWriteRequest(requestFactory.indexRequest(query, index));
    IndexResponse indexResponse = execute(client -> client.index(request, RequestOptions.DEFAULT));

    // ......
    return indexResponse.getId();
}
```

所以看下来之后各位是不是会感觉一点：其实 Template 也没有多神奇嘛，说破天还是帮我们完成一些琐碎的事情，让我们操作尽可能简化之后的 API ，把更多的精力都放在业务开发上。

### 2.3 查询数据的动作

我们再来看一个查询数据的动作。这里我们以 `search` 为例，传入 `Query` 对象作为查询条件封装：

```java
public <T> SearchHits<T> search(Query query, Class<T> clazz) {
    return search(query, clazz, getIndexCoordinatesFor(clazz));
}
```

还记得吗？`Query` 的 3 个实现中，除了 String 类型的条件封装之外，还有基于 SpringDataElasticSearch 自己封装的一套 `CriteriaQuery` ，以及基于 `RestHighLevelClient` 原生封装的 `NativeSearchQuery` 。`ElasticsearchRestTemplate` 是如何将这三种不同的 `Query` 最终合为一用呢？我们继续往下看。

```java
@Nullable protected RequestFactory requestFactory;

@Override
public <T> SearchHits<T> search(Query query, Class<T> clazz, IndexCoordinates index) {
    SearchRequest searchRequest = requestFactory.searchRequest(query, clazz, index);
    SearchResponse response = execute(client -> client.search(searchRequest, RequestOptions.DEFAULT));

    SearchDocumentResponseCallback<SearchHits<T>> callback = new ReadSearchDocumentResponseCallback<>(clazz, index);
    return callback.doWith(SearchDocumentResponse.from(response));
}
```

点进重载的三参数 `search` 方法后，第二行代码显然是拿到前一步构建好的 `SearchRequest` ，发起搜索的请求了，那关键就在于 `SearchRequest` 的创建，它借助的是一个 `RequestFactory` 对象。

进入到 `RequestFactory` 的 `searchRequest` 方法中，我们可以看到 `SearchRequest` 、`QueryBuilder` 都是在这个方法内部构造和应用。

```java
public SearchRequest searchRequest(Query query, @Nullable Class<?> clazz, IndexCoordinates index) {
    elasticsearchConverter.updateQuery(query, clazz);
    SearchRequest searchRequest = prepareSearchRequest(query, clazz, index);
    // 注意这一步是关键！
    QueryBuilder elasticsearchQuery = getQuery(query);
    QueryBuilder elasticsearchFilter = getFilter(query);

    searchRequest.source().query(elasticsearchQuery);
    // ......
    return searchRequest;
}
```

构造 `SearchRequest` 的过程我们可以暂且不关注，主要是应该研究查询条件的封装 `QueryBuilder` 是怎么来的，我们继续进入 `getQuery` 方法中。

```java
private QueryBuilder getQuery(Query query) {
    QueryBuilder elasticsearchQuery;
    if (query instanceof NativeSearchQuery) {
        NativeSearchQuery searchQuery = (NativeSearchQuery) query;
        elasticsearchQuery = searchQuery.getQuery();
    } else if (query instanceof CriteriaQuery) {
        CriteriaQuery criteriaQuery = (CriteriaQuery) query;
        elasticsearchQuery = new CriteriaQueryProcessor().createQuery(criteriaQuery.getCriteria());
    } else if (query instanceof StringQuery) {
        StringQuery stringQuery = (StringQuery) query;
        elasticsearchQuery = wrapperQuery(stringQuery.getSource());
    } // else throw ex ......
    return elasticsearchQuery;
}
```

好家伙，到最后终于看到了，底层是逐个判断传入的 `Query` 对象类型，来分别调用其方法，最终构造出 / 获取到 `QueryBuilder` 对象，这样一来查询条件也就有了。

拿到 `SearchRequest` 和 `QueryBuilder` 后，下面的 `search` 方法还是调用 `RestHighLevelClient` 的 `search` 方法，就可以完成搜索的动作了，底层源码还是非常简单的。

