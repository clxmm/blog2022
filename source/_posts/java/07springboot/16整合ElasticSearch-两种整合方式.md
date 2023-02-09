---
title: 16整合ElasticSearch-两种整合方式
toc: true
tags: springboot
categories: 
    - [java]
    - [springboot]
---

通过上一章的内容，我们已经简单使用了 ElasticSearch 的原生操作和使用方式。项目开发中必然使用的是整合与 ElasticSearch 相关的库，通过 java 代码操纵 ElasticSearch 完成业务逻辑。下面我们就使用 SpringBoot 来整合 ElasticSearch 。

注意本章的标题，我们提到的是用两种不同的整合方式，这是迎合目前 `(2022年10月)` 市面上大多数的言论：因为 SpringData 的缘故，SpringDataElasticSearch 的版本支持相较于 SpringBoot 、ElasticSearch 的迭代而言有些跟不上，而且 ElasticSearch 官方也提到过，它推荐我们使用官方推出的**高等级客户端**来实现与 ElasticSearch 的交互。为了满足两种不同的整合方式场景需求，让各位小伙伴对这两种方式都有了解，所以本小册最终决定同时演示这两种方式。但是在本章的最后，小册会给各位吃一颗定心丸，让各位打心底里放心。

## 1. 使用SpringData

首先我们使用 SpringData 进行整合。由于本小册使用 ElasticSearch 7.17.6 作为演示版本，而对应的 SpringBoot 版本 2.5.14 才支持到 7.12.1 ，这也就很明显地体会到版本更新迭代的限制了：很多情况下我们不需要那么高的 SpringBoot 版本，但却要因为 ElasticSearch 的版本很高，导致不得不升级 / 退让，这种尴尬的场面着实有点让人恼火（这是市面上多数文章、教辅资料的观点，此处先模仿其语气）。

不过我们还是先来演示如何整合和操作吧！毕竟先谈会不会，再谈好不好。

### 1.1 搭建工程

使用 SpringBoot 整合 ElasticSearch 时，我们可以直接使用 SpringData 的另外一个模块 - SpringDataElasticSearch 来完成与 ElasticSearch 的交互，所以在导入 pom 依赖时，只需要引入 `spring-boot-starter-data-elasticsearch` 即可：（另外导入 `spring-boot-starter-test` 用于单元测试）

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

之后需要在 `application.properties` 中配置 ElasticSearch 的地址即可。

```properties
spring:
  elasticsearch:
    rest:
      uris: http://192.168.42.103:9200
```

### 1.2 编写实体类

SpringData 的一贯操作是用实体类与数据源的模型进行一一映射，而为了从开始创建索引演示，我们来编写一个新的实体类：显卡。

```java
@Document(indexName = "gpu")
public class GraphicsCard {
    
    @Id
    private Integer id;
    
    private String name;
    
    private String brand;
    
    private Integer price;
    
    // getter setter ......
}
```

如何让它用上 SpringData 的东西呢？与之前 SpringDataMongoDB 的方式类似，在类上标注一个 `@Document(indexName = "gpu")` ，这代表我们的 `GraphicsCard` 数据将存放在 `gpu` 索引中（注意，在 SpringDataElasticSearch 4.0 之前，`@Document` 注解是可以手动声明 `type` 属性的，自 4.0 以后该属性被删除，ElasticSearch 6.x 开始不建议再使用 type ）；在 id 属性上标注一个 `@Id` 注解，代表这个属性与 ElasticSearch 中每一条数据的 id 一一对应。

相应的，我们再仿照 `MongoRepository` 的样式，写一个与 ElasticSearch 交互的 Repository 接口：

```java
@Repository
public interface GraphicsCardRepository extends ElasticsearchRepository<GraphicsCard, Integer> {
    
}
```

到此为止，一个数据模型的基本代码就完成了。

### 1.3 测试交互

接下来我们依然使用单元测试的方式，执行我们编写的代码。与保存到数据库、MongoDB 的方式几乎完全一致，我们只需要构造数据，调用 Repository 的 `save` 方法即可：

```java
@SpringBootTest
public class DataElasticSearchTest {
    
    @Autowired
    private GraphicsCardRepository graphicsCardRepository;
    
    @Test
    public void test1() throws Exception {
        GraphicsCard graphicsCard = new GraphicsCard();
        graphicsCard.setId(1);
        graphicsCard.setName("ROG 玩家国度-STRIX-RTX4090-O24G-GAMING");
        graphicsCard.setBrand("ROG");
        graphicsCard.setPrice(15999);
        graphicsCardRepository.save(graphicsCard);
    }
}
```

执行 test1 方法，控制台没有异常输出，测试用例通过，如果此时我们向浏览器发送 GET 请求查询所有的显卡数据，可以发现 ElasticSearch 中已经保存了一条数据：

```json
{
    "took": 0,
    "timed_out": false,
    "_shards": {
        "total": 1,
        "successful": 1,
        "skipped": 0,
        "failed": 0
    },
    "hits": {
        "total": {
            "value": 1,
            "relation": "eq"
        },
        "max_score": 1,
        "hits": [
            {
                "_index": "gpu",
                "_type": "_doc",
                "_id": "1",
                "_score": 1,
                "_source": {
                    "_class": "com.linkedbear.boot.elasticsearch.entity.GraphicsCard",
                    "id": 1,
                    "name": "ROG 玩家国度-STRIX-RTX4090-O24G-GAMING",
                    "brand": "ROG",
                    "price": 15999
                }
            }
        ]
    }
}
```

注意一个细节，保存的数据中有一个 `_class` 属性，指向的是数据映射的模型类类型，有了这个类型，SpringDataElasticSearch 就可以识别并正确转换和封装数据。

### 1.4 SpringDataElasticSearch的问题

下面我们来讨论市面上大多数文章和资料中聊到的一个观点：

在相对低版本的 SpringDataElasticSearch 中有一个比较头疼的问题：由于 SpringDataElasticSearch 的底层是用 `transport-api.jar` ，与 ElasticSearch 的 9300 端口完成通信，但是 ElasticSearch 在 7.x 中就已经明确声明不建议使用，并在 8.x 版本中将其移除。面对这种巨大的变动，我们不得不为此作出改变。ElasticSearch 官方提供了一个高等级的 Rest 风格的客户端，使用这个高等级客户端，可以比较容易的编写出与 ElasticSearch 的交互代码。

[https://docs.spring.io/spring-data/elasticsearch/docs/4.2.12/reference/html/#preface.requirements](https://docs.spring.io/spring-data/elasticsearch/docs/4.2.12/reference/html/#preface.requirements)

![](/img/202302/16es.png)

从上面的表格中我们可以看到两个信息：1）SpringDataElasticSearch 底层适配的 ElasticSearch 版本是很延后的；2）SpringData 官方也建议我们在适配高版本 ElasticSearch 时采用高等级客户端。

此外还有一个点，高等级 ElasticSearch 的客户端是 ElasticSearch 官方推出的，那就意味着，只要 ElasticSearch 的版本更新，相应的 API 操作客户端就会跟着更新，这样也更有利于整合 ElasticSearch 的不同版本，减少一些不必要的顾虑。

既然理由已经很充分了，下面我们就来使用一下 ElasticSearch 官方提供的这个高等级客户端。

## 2. 使用ElasticSearch Java API Client

---

来到 ElasticSearch 的官方网站中，通过搜寻 ElasticSearch 的文档，可以找到有关 ElasticSearch 的 Java Client API ：

[https://www.elastic.co/guide/en/elasticsearch/client/java-api-client/7.17/introduction.html](https://www.elastic.co/guide/en/elasticsearch/client/java-api-client/7.17/introduction.html)

> 可能有小伙伴看到这个有些面生，其实这个项目的前身就是 Java REST Clients （也就是 High Level Client ），在 7.15.0 之后 Java REST Clients 被废弃了。

使用 Java API Client 时， ~~我们就不能再使用 SpringDataElasticSearch 了。。。~~ （并不是哦）我们依然可以在依赖 SpringDataElasticSearch 时，直接操纵 Java API Client 的相关 API 。为什么可以这样呢，其实是因为自从 2019 年的 SpringDataElasticSearch **3.2.0.RELEASE** 版本之后，SpringDataElasticSearch 会**自动依赖 `elasticsearch-rest-high-level-client`** ，所以当我们导入 `spring-boot-starter-data-elasticsearch` 依赖时，高等级客户端就已经跟着导入进来了！

> 当然，如果小伙伴们用的 SpringDataElasticSearch 的版本低于 3.2.0 的话，那还是移除掉吧，直接引入 `elasticsearch-rest-high-level-client` 的依赖即可。
>
> ```xml
> <dependency>
>     <groupId>org.elasticsearch.client</groupId>
>     <artifactId>elasticsearch-rest-high-level-client</artifactId>
> </dependency>
> ```

高等级客户端如何操作呢？我们可以先在单元测试里试一下。

### 2.1 初始化客户端

初始化高等级客户端的最终 API 是 `RestHighLevelClient` ，它的初始化方式略微复杂，需要涉及到几个陌生的 API ，不过折腾这么多，最终的目的还是传入 ElasticSearch 的目标地址，如以下代码所示：

```java
@Test
public void test1() throws Exception {
  RestClientBuilder builder = RestClient.builder(HttpHost.create("http://192.168.217.142:9200"));
  RestHighLevelClient client = new RestHighLevelClient(builder);
  System.out.println(client);
  client.close();
}
```

执行以上的代码，可以发现 `RestHighLevelClient` 对象可以成功创建并连接到 ElasticSearch 实例上。

```
org.elasticsearch.client.RestHighLevelClient@334ebcaa

```

但是这个主机地址硬编码在程序代码里可不合适吧！还记得上面我们把地址配置在 application.properties 了吗？这里我们照样可以拿来用。

于是优化后的代码就可以变成这样，执行必然也是没有问题的。

```java
    @Value("${spring.elasticsearch.rest.uris}")
    private String esHost;
    
    @Test
    public void test1() throws Exception {
        RestClientBuilder builder = RestClient.builder(HttpHost.create(esHost));
        RestHighLevelClient client = new RestHighLevelClient(builder);
        System.out.println(client);
        client.close();
    }
```

请注意代码中的一个细节：由于 `RestHighLevelClient` 对象是我们手动 new 出来的，所以需要自己管理生命周期，在每次用完时，要记得执行 close 方法。

**一点小整合优化**

为了能让下面的测试代码写的更舒服，我们把以上代码做成一个 `@Bean` ，让 IOC 容器帮我们管理。

```java
@Configuration(proxyBeanMethods = false)
public class ElasticSearchConfiguration {
    
    @Value("${spring.elasticsearch.rest.uris}")
    private String esHost;
    
    @Bean(destroyMethod = "close")
    public RestHighLevelClient restHighLevelClient() {
        RestClientBuilder builder = RestClient.builder(HttpHost.create(esHost));
        return new RestHighLevelClient(builder);
    }
}
```

> 可以发现，让 IOC 容器管理的话，得益于 `@Bean` 注解中的 `destroyMethod` 属性，使得关闭 `RestHighLevelClient` 也转交给 IOC 容器，我们只管使用就行了（这也是 IOC 容器的强大之处：**统一生命周期管理**）。

### 2.2 测试交互

使用高等级客户端的好处，主要体现在与 ElasticSearch 交互时，我们的每个请求都可以找到一个相应的 Request 类来使用，所以使用高等级客户端时，核心的使用方式是拿 client **不断地构造 Request 请求对象并发送**。

#### 2.2.1 索引数据

下面我们使用高等级客户端，再向 ElasticSearch 中索引几条显卡的数据。由于上面注册了 `RestHighLevelClient` 对象，接下来只需要在单元测试类中注入它即可。

索引数据的方式，首先需要构造一个 `IndexRequest` ，它需要指定本次索引数据的目标索引，指定 `id` 后设置原始数据，而设置原始数据的方式，官方给出了不少方法，从下面重载的 `source` 方法就可以看出，这实在是有点令人眼花缭乱：

那我们具体用哪个呢？阿熊比较推荐的是用**传 `Map` 或者传 `String` 的方式**（操作方式最简单易懂）。

首先我们先构造一个显卡的信息（现在我们是在代码中构造，回头用于项目业务开发中，那必然是从数据库查了）：

```java
    @Test
    public void test2() throws Exception {
        GraphicsCard graphicsCard = new GraphicsCard();
        graphicsCard.setId(2);
        graphicsCard.setName("七彩虹（Colorful）战斧GeForce RTX 4090 豪华版");
        graphicsCard.setBrand("七彩虹");
        graphicsCard.setPrice(12999);

    }
```

构造完成后，怎么设置 source 呢？很简单，使用转 json 的工具，或者转 Map 的工具类就可以了。以下是针对几种常见工具集的转 json 字符串或者转 Map 的方式，供小伙伴们参考。

- 对应 Jackson 的转 json 方式：`new ObjectMapper().writeValueAsString(graphicsCard);`
- 对应 fastjson 的转 json 方式：`JSONObject.toJSONString(graphicsCard);`

转换完成后，只需要将数据放入 IndexRequest 即可，全部的代码如下：

```java
 @Test
    public void testIndex() throws Exception {
        GraphicsCard graphicsCard = new GraphicsCard();
        graphicsCard.setId(2);
        graphicsCard.setName("七彩虹（Colorful）战斧GeForce RTX 4090 豪华版");
        graphicsCard.setBrand("七彩虹");
        graphicsCard.setPrice(12999);
        String jsonString = new ObjectMapper().writeValueAsString(graphicsCard);
        // 当转json时，需要指定传入参数的类型是json
        IndexRequest request = new IndexRequest("gpu").id(graphicsCard.getId().toString())
                .source(jsonString, XContentType.JSON);
        restHighLevelClient.index(request, RequestOptions.DEFAULT);
    }
    
    @Test
    public void testIndex3() throws Exception {
        GraphicsCard graphicsCard = new GraphicsCard();
        graphicsCard.setId(3);
        graphicsCard.setName("华硕 ASUS TUF-GeForce RTX 3080 Ti-O12G-GAMING");
        graphicsCard.setBrand("华硕");
        graphicsCard.setPrice(7999);
        // 当转Map时，不需要指定类型。
        Map<String, Object> data = BeanMap.create(graphicsCard);
        IndexRequest request = new IndexRequest("gpu").id(graphicsCard.getId().toString()).source(data);
        restHighLevelClient.index(request, RequestOptions.DEFAULT);
    }
```

> 小提示：除了直接通过 new 的方式创建 `IndexRequest` ，还可以通过建造器 `IndexRequestBuilder` 的方式构造，效果是一样的。

#### 2.2.2 查询数据

数据真的添加上了吗？我们只执行了单元测试方法，但还不确定是不是真的添加成功，所以下面我们来试着查询一下。

查询的方式依然是通过构造 Request 请求来实现，我们来试着把整个 `gpu` 索引下的文档全部查出，跟上面的索引数据方式几乎大同小异，核心的套路都是 **“创建 Request ，利用 Client 发送请求”** 。

##### 2.2.2.1 Search查列表

下面我们先来列表查询。查询使用的 Request 类型叫 `SearchRequest` ：

```java
    @Test
    public void testSearch() throws Exception {
        SearchRequest request = new SearchRequest("gpu");
        SearchResponse response = restHighLevelClient.search(request, RequestOptions.DEFAULT);
        
    }
```

请注意，上述的查询是直接获取整个 `gpu` 索引下的所有文档（因为没有设置查询条件）！

使用 `RestHighLevelClient` 的 `search` 方法后，可以得到一个 `SearchResponse` 对象，其中就包含了我们所需要的查询结果。

但是，查询结果该怎么获取呢？回想一下我们上一章中，使用 postman 等工具发送请求时，得到的是一个什么数据来着？

```json
{
    ......
    "hits": {
        "total": {
            "value": 2,
            "relation": "eq"
        },
        "max_score": 0.6983144,
        "hits": [
            // 数据 ......
        ]
    }
}
```

是所谓的 **hits** **“命中的数据”** 吧！通过拿到 hits 的结果，就可以获取到符合查询条件的所有原始文档了！那 SearchResponse 中能不能也这样呢。。。

果然可以拿到！而且拿到的结果是一个可以迭代的 `Iterable` 类型的集合！那我们赶紧 for 循环迭代一把，把每一条查出来的文档都以字符串的形式打印出来：

```java
  @Test
    public void testSearch() throws Exception {
        SearchRequest request = new SearchRequest("gpu");
        SearchResponse response = restHighLevelClient.search(request, RequestOptions.DEFAULT);
        for (SearchHit hit : response.getHits()) {
            System.out.println(hit.getSourceAsString());
        }
    }
```

执行 `testSearch` 测试方法，可以看到打印出来的数据都是我们存放的原始文档：

```json
{"_class":"com.linkedbear.boot.elasticsearch.entity.GraphicsCard","id":1,"name":"ROG 玩家国度-STRIX-RTX4090-O24G-GAMING","brand":"ROG","price":15999}
{"id":2,"name":"七彩虹（Colorful）战斧GeForce RTX 4090 豪华版","brand":"七彩虹","price":12999}
{"price":7999,"name":"华硕 ASUS TUF-GeForce RTX 3080 Ti-O12G-GAMING","id":3,"brand":"华硕"}
```

> 注意一个细节，由于通过 SpringDataElasticSearch 保存的数据，在保存时会把映射的模型类全限定类名一并保存，而通过高等级客户端操作时不会保存该信息，所以查询结果中要分别对待。

##### 2.2.2.2 Search条件查

接下来是条件查询，跟上一章的节奏一致，我们先来试一下简单的单条件查询。查询条件的设置是一套固定的格式，如下方的中间三行代码所示。`SearchRequest` 在设置查询条件时，需要接收一个 `SearchSourceBuilder` 类型的参数，所有的查询条件都设置在 `SearchSourceBuilder` 中，而具体的条件都是使用 `QueryBuilders` 工具类创建，`QueryBuilders` 中的 API 跟 ElasticSearch 查询语法中的条件过滤方式完全一致。

```java
    @Test
    public void testSearch() throws Exception {
        SearchRequest request = new SearchRequest("gpu");
        
        // 添加查询条件
        SearchSourceBuilder builder = new SearchSourceBuilder();
        builder.query(QueryBuilders.termQuery("name", "3080"));
        request.source(builder);
        
        SearchResponse response = restHighLevelClient.search(request, RequestOptions.DEFAULT);
        for (SearchHit hit : response.getHits()) {
            System.out.println(hit.getSourceAsString());
        }
    }
```

上方我们构建的是一个查询 `name` 中带有 3080 的显卡信息，执行查询后返回的数据依然是一组 `SearchHit` ，遍历打印后得到的只有一条数据，符合我们预期。

```json
{"price":7999,"name":"华硕 ASUS TUF-GeForce RTX 3080 Ti-O12G-GAMING","id":3,"brand":"华硕"}

```

下面我们再来构造相对复杂的条件查询。上一章的最后我们举了一个复合查询的例子，本小节我们也来整一个。我们希望过滤的是名字里带有七彩虹的，并且价格不低于 10000 块钱的显卡。从上一章的内容中我们知道，复合查询需要创建 bool 型查询，分别使用 must 和 filter 来过滤条件，于是对应到高等级客户端的 API 上，我们可以用如下的方式来创建：

```java
@Test
    public void testSearch() throws Exception {
        SearchRequest request = new SearchRequest("gpu");
        
        // 筛选名称中带七彩虹，并且价格不低于10000的
        SearchSourceBuilder builder = new SearchSourceBuilder();
        builder.query(QueryBuilders.boolQuery().must(QueryBuilders.termQuery("name", "七彩虹"))
                .filter(QueryBuilders.rangeQuery("price").gte(10000)));
        request.source(builder);
        
        SearchResponse response = restHighLevelClient.search(request, RequestOptions.DEFAULT);
        for (SearchHit hit : response.getHits()) {
            System.out.println(hit.getSourceAsString());
        }
    }
```

执行查询，可以成功查询到一条数据，复合查询完成。

```json
{"id":2,"name":"七彩虹（Colorful）战斧GeForce RTX 4090 豪华版","brand":"七彩虹","price":12999}

```

##### 2.2.2.3 Get查单条

除了列表和条件查之外，查单条也是必不可少的。查询单条数据的方式是通过构造 `GetRequest` 对象并指定 `id` ，配合 `RestHighLevelClient` 的 get 方法完成。实现方式非常简单，大家快速过一遍即可。

```java
    @Test
    public void testGet() throws Exception {
        GetRequest request = new GetRequest("gpu").id("3");
        GetResponse response = restHighLevelClient.get(request, RequestOptions.DEFAULT);
        System.out.println(response.getSourceAsString());
        // 转化为实体模型类
        GraphicsCard graphicsCard = new ObjectMapper().readValue(response.getSourceAsString(), GraphicsCard.class);
        System.out.println(graphicsCard);
    }
```

上述测试代码中不仅查出了原文档数据，也顺便用 Jackson 将数据转为 `GraphicsCard` 类型的对象打印了一把。实际在项目开发中，这种写法不在少数。

```json
{"price":7999,"name":"华硕 ASUS TUF-GeForce RTX 3080 Ti-O12G-GAMING","id":3,"brand":"华硕"}
GraphicsCard(id=3, name=华硕 ASUS TUF-GeForce RTX 3080 Ti-O12G-GAMING, brand=华硕, price=7999)
```

### 2.3 使用上的不便捷

通过以上的简单示例，小伙伴们是否可以体会到一点：使用高等级客户端是挺好，跟 ElasticSearch 的功能对标也挺严丝合缝的，但是每次索引数据的时候都得转 Map 或者 json 字符串，查询数据后也得将 json 字符串转为实体类对象，这实在是有点费劲。有没有一个更好用的 API 供我们使用呢？

还真有，但。。。那是 SpringData 的东西。。。其实 SpringData 自 3.2.0.RELEASE 版本以后，它针对 ElasticSearch 高等级客户端进行了全新的封装，我们完全可以继续使用 SpringData 来完成与 ElasticSearch 的交互。

## 3. 辗转反侧又回SpringData

SpringData 的经典实用方式，一个是 Repository 接口，另一个是 Template 式操作模板。在第 1 节中，我们就已经演示过 Repository 的使用，本小节中，我们简单看一下 Template 的使用。

### 3.1 Template的使用

使用 Template 时，无需我们多做什么，只需要注入 `ElasticsearchRestTemplate` 即可（SpringBoot 的自动装配都帮我们做了，其实上述的 `RestHighLevelClient` 组件，SpringData 也帮我们注册了）。

具体的增删改查操作，`ElasticsearchRestTemplate` 都有帮我们封装，我们主要来体会一下 Template 帮我们处理的查询类方法（ `save` 、`update` 、`delete` 等增删改方法相对简单，不再演示）。

借助 IDE ，查看 `ElasticsearchRestTemplate` 的 `search` 方法时，我们可以看到一些端倪：1）查询时可以直接指定返回数据的类型；2）传入的是一个 `Query` 对象，不是高等级客户端的 `SearchSourceBuilder` 或 `QueryBuilder` 。

那这个 `Query` 是什么呢？借助 IDE 查看它和它的继承关系，可以发现这是 SpringDataElasticSearch 自己封装的一套查询表达模型，从实现的角度上看，它包含普通的 json 字符串查询、基于 `Criteria` 条件封装的查询，以及原生高等级客户端 API 的条件查询模型，共计三种。

那我们就来演示相对常用的两种吧，`CriteriaQuery` 是 SpringDataElasticSearch 自己封装的一套查询语法，它更接近于 SpringData 的统一风格（包含 where 、is (即 equal ) 、contains (即 like ) 、in 等操作），不过使用它需要先学习一些 Criteria 的设计和使用方式；另一种 `Criteria` 的实现是 `NativeSearchQuery` ，它接收的参数就是我们在高等级客户端下使用的 `QueryBuilder` ，由于前期我们已经接触过 `QueryBuilder` 的使用，所以使用 `NativeSearchQuery` 的上手成本更低，且更容易将查询逻辑从高等级客户端的 API 移植到 `ElasticsearchRestTemplate` 的使用中。

```java
    @Autowired
    private ElasticsearchRestTemplate elasticsearchRestTemplate;
    
    @Test
    public void testTemplate() throws Exception {
        SearchHits<GraphicsCard> hits = elasticsearchRestTemplate
                .search(new CriteriaQuery(Criteria.where("name").contains("ROG")), GraphicsCard.class);
        for (SearchHit<GraphicsCard> hit : hits) {
            System.out.println(hit.getContent());
        }
    
        hits = elasticsearchRestTemplate.search(new NativeSearchQuery(QueryBuilders.termQuery("name", "ROG")), GraphicsCard.class);
        for (SearchHit<GraphicsCard> hit : hits) {
            System.out.println(hit.getContent());
        }
    }
```

有关 `ElasticsearchRestTemplate` 的更多使用方式，各位可以参照 SpringDataElasticSearch 和 ElasticSearch Java API Client 的官方文档进行学习。

### 3.2 为什么说Template可以正常使用

本章的最后，我们来解答开头提出的观点：为什么现版本的 SpringDataElasticSearch 完全可以使用。下面我们通过源码的角度来一探究竟。

#### 3.2.1 RestHighLevelClient的初始化

我们找到配置类 `ElasticSearchConfiguration` ，把上方标注的 `@Configuration` 注释掉，这就意味着我们不再手动注册 `RestHighLevelClient` 的实例了。回到 `ElasticSearchClientTest` 测试类中，可以发现 IDEA 还帮我们提示了有 `RestHighLevelClient` 的 Bean 可以注入。

那这个类是哪来的呢？我们点击左边的图标，发现跳转到了 `ElasticsearchRestClientConfigurations` 中：

好家伙，合着这里本来就帮我们创建过了！果然 **SpringBoot 的自动装配还是王道**啊！

我们继续向上找 `RestClientBuilder` ，果然，在 `ElasticsearchRestClientConfigurations` 的最上面，我们可以找到 `RestClientBuilder` 的初始化代码，它主要注入了一个 `ElasticsearchRestClientProperties` ，这个家伙一看就是映射 `application.properties` 的，而在构造的代码中，第一行我们能看到一个东西：`properties.getUris()` ，这不就是我们在本章工程的一开始，**在配置文件中写的 ElasticSearch 的地址**吗？合着人家完全帮我们做好了，我们只告诉 SpringBoot ，ElasticSearch 的地址在哪里就行了，剩下的我们只管拿来用就行！

```java
class ElasticsearchRestClientConfigurations {

    @Configuration(proxyBeanMethods = false)
    @ConditionalOnMissingBean(RestClientBuilder.class)
    static class RestClientBuilderConfiguration {
        // ......

        @Bean
        RestClientBuilder elasticsearchRestClientBuilder(ElasticsearchRestClientProperties properties,
                ObjectProvider<RestClientBuilderCustomizer> builderCustomizers) {
            // 注意看这里！
            HttpHost[] hosts = properties.getUris().stream().map(this::createHttpHost).toArray(HttpHost[]::new);
            RestClientBuilder builder = RestClient.builder(hosts);
            // 处理 ......
            return builder;
        }
```

#### 3.2.2 ElasticsearchRestTemplate用的谁？

通过上述简短的源码解读，我们目前只能确定一点：SpringBoot 能帮我们初始化 `RestHighLevelClient` ，但到目前为止还没有确定 `ElasticsearchRestTemplate` 的底层呢！`ElasticsearchRestTemplate` 的底层用的是谁呢？

很简单，追踪一下 `ElasticsearchRestTemplate` 的构造器使用位置就知道了，在 `ElasticsearchDataConfiguration` 配置类中，我们找到了一个注册 `ElasticsearchRestTemplate` 的内部类，它刚好就是依赖的 ElasticSearch 官方提供的高等级客户端 `RestHighLevelClient` ！

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(RestHighLevelClient.class)
static class RestClientConfiguration {

    @Bean
    @ConditionalOnMissingBean(value = ElasticsearchOperations.class, name = "elasticsearchTemplate")
    @ConditionalOnBean(RestHighLevelClient.class)
    ElasticsearchRestTemplate elasticsearchTemplate(RestHighLevelClient client, ElasticsearchConverter converter) {
        return new ElasticsearchRestTemplate(client, converter);
    }
}

```

#### 3.2.3 Repository用的谁？

既然 `ElasticSearchRestTemplate` 的确用的 `RestHighLevelClient` ，那我们在第一节中用到的 `ElasticsearchRepository` ，它的底层是否也用的 `RestHighLevelClient` 呢？

想解开这个疑惑也不难，我们找打 `ElasticsearchRepository` 的实现类，发现只有一个 `SimpleElasticsearchRepository` 是现役的最终落地实现，而在这里面它依赖了一个 `ElasticsearchOperations` ：

```java
public class SimpleElasticsearchRepository<T, ID> implements ElasticsearchRepository<T, ID> {

	protected ElasticsearchOperations operations;
    // ......
```

这是个什么东西呢（其实如果有读过 `JdbcTemplate` 源码的小伙伴，应该对这个接口不陌生）？点开它的继承关系，可以发现，它其实就是指代的 `ElasticSearchRestTemplate` ：

另外从这张图中还可以得到一个关键信息：之前基于 9300 端口的 `ElasticsearchTemplate` 已经被标注为**过时**，所以目前主流的那些言论真的可以消停下了。。。人家 SpringData 官方早就知道了，并且在 4.0.0 以后的版本都标记废弃了，所以还是 SpringData 真香就完事了！

【好了，SpringDataElasticSearch 和高等级客户端的使用方式就讲解完毕了，最后一章我们就看看 SpringDataElasticSearch 的一些底层核心自动装配吧】

