---
title: 01jsckson简介
toc: true
tags: java
categories: 
    - [java]
    - [jackson]
---


## 1.jsckson 库的优点
Jackson是一个简单的、功能强大的、基于Java的应用库。它可以很方便完成Java对象和Json对象(xml文档or其它格式）进行互转。Jackson社区相对比较活跃，更新速度也比较快。Jackson库有如下几大特性

<!--more-->

- 高性能且稳定：低内存占用，对大/小JSON串，大/小对象的解析表现均很优秀
- 流行度高：是很多流行框架的默认选择
- 容易使用：提供高层次的API，极大简化了日常使用案例
- 无需自己手动创建映射：内置了绝大部分序列化时和Java类型的映射关系
- 干净的JSON：创建的JSON具有干净、紧凑、体积小等特点
- 无三方依赖：仅依赖于JDK
- **Spring生态加持**：jackson是Spring家族的默认JSON/XML解析器（明白了吧，学完此专栏你对Spring都能更亲近些了，一举两得）

版本约定：本专栏统一使用的版本号固定为`2.10.1`（2019-12-09发布），GAV如下：

```xml
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-core</artifactId>
    <version>2.10.1</version>
</dependency>
```

为了保持版本的统一性，后续的`Spring Boot（2.2.2.RELEASE）/Spring Framework（5.2.2.RELEASE）`使用的均为当前最新版本，因为它内置的jackson也恰好就是本专栏讲解的版本。

细心的朋友从上面的`groupId`里可以看到：`jackson`它隶属于`fasterxml`这个组织。本着追本溯源的精神，可以稍微的了解了解这个组织：[fasterxml官网](http://fasterxml.com/) 

简单翻译：FasterXML是Woodstox流的XML解析器、Jackson流的JSON解析器、Aalto非阻塞XML解析器以及**不断增长**的实用程序库和扩展家族背后的业务。

作为一个高度流行的开源库，这种官网页面应该刷新了你的认知吧。并不是它内容不多，而其实是它的详细介绍都发布在`github`上了，这便是接下来我们来认识它的主要渠道。

> 这种做法貌似已经成为了一种流行的趋势：越来越多的开源软件倾向于把github作为他们的Home Page了

## 2.官网介绍

了解一门新的技术，第一步应该就是看它的官网。上面已然解释了，`fasterxml`组织它把各工程的首页内容都托管在了github上，Jackson当然也不例外。[Jackson官网](https://github.com/FasterXML/jackson) 上对它自己有如下描述：

Jackson旧称为：**Java(或JVM平台)**的标准JSON库，或者是Java的**最佳JSON解析器**，或者简称为“**Java的JSON**”

更重要的是，Jackson是一套JVM平台的 **数据处理（不限于JSON）** 工具集：包括 **一流的** JSON解析器/ JSON生成器、数据绑定库(POJOs to and from JSON)；并且提供了相关模块来支持 Avro, BSON, CBOR, CSV, Smile, Properties, Protobuf, XML or YAML等数据格式，甚至还支持大数据格式模块的设置。

------

### 1.分支：1.x和2.x

Jackson有两个主要的分支：

- .x分支，处于维护模式，只发布bug修复版本（最近一次发布于Jul, 2013）
- 2.x是正在开发的版本（持续更新升级中，2.0.0发布于Mar, 2012）

注意：这两个主要版本使用**不同的Java包名**和Maven GAV，因此它们并不相互兼容，**但可以和平共存**。一个项目可以同时依赖于这两个版本是没有冲突的。这是经过设计而为之，选择这种策略是为了更顺利地从1.x进行迁移2. x

## 3.模块介绍

Jackson是个开源的、且开放的社区。下面列出的大多数项目/模块是由Jackson开发团队**领导的**，但也有一些来自Jackson社区的成员

### 三大核心模块

**core module(核心模块) 是扩展模块构建的基础**。Jackson目前有3个核心模块：

> 说明：核心模块的groupId均为：`<groupId>com.fasterxml.jackson.core</groupId>`，artifactId见下面各模块所示

- Streaming流处理模块(`jackson-core`)：定义底层处理流的API：JsonPaser和JsonGenerator等，并包含**特定于json**的实现。
- Annotations标准注解模块(`jackson-annotations`)：包含标准的Jackson注解
- Databind数据绑定模块(`jackson-databind`)：在streaming包上实现数据绑定(和对象序列化)支持；**它依赖于上面的两个模块**，也是Jackson的高层API(如ObjectMapper)所在的模块

实际应用级开发中，我们只会使用到Databind数据绑定模块，so它是本系列重中之重。下面介绍那些举足轻重的**第三方模块**。

### 2.数据类型模块

这些扩展是Jackson插件模块(通过`ObjectMapper.registerModule()`注册，下同)，并通过添加序列化器和反序列化器来对各种常用Java库数据类型的支持，以便`Jackson databind`包(`ObjectMapper / ObjectReader / ObjectWriter`)能够顺利读写/转换这些类型。

第三方模块有些是Jackson官方人员直接lead和维护的（主流模块），也有些是纯社区行为。现在按照这两个分类分别介绍一下各个模块的作用：

**官方直接维护**

> 说明：官方维护的这些数据类型模块的groupId统一为：`<groupId>com.fasterxml.jackson.datatype</groupId>`，且版本号是和主版本号保持一致的

- 标准集合数据类型模块：
  - Guava：支持Guava的集合数据类型
  - HPPC：略
  - PCollections：略 (Jackson 2.7新增的支持)
- Hibernate：支持Hibernate的一些特性，如懒加载、proxy代理等
- Joda：支持Joda date/time的数据类型
- JDK7：对JDK7的支持（说明：2.7以后就无用了，以为2.7版本后最低的JDK版本要求是7）
- Java8：它分为如下三个子模块来支持Java8
  - `jackson-module-parameter-names`：此模块能够访问构造函数和方法参数的名称，从而允许省略`@JsonProperty`（当然前提是你必须加了编译参数：`-parameters`）
  - `jackson-datatype-jsr310`：支持Java8新增的JSR310时间API
  - `jackson-datatype-jdk8`：除了Java8的时间API外其它的API的支持，如`Optional`
- JSR-353/org.json：略

**非官方直接维护：**

- jackson-datatype-bolts：对 Yandex Bolts collection types 的支持
- jackson-datatype-commons-lang3：支持Apache Commons Lang v3里面的一些类型
- jackson-datatype-money：支持`javax.money`
- jackson-datatype-json-lib：对久远的`json-lib`这个库的支持

### 3.数据格式模块

Data format modules(数据格式模块)提供对**JSON之外**的数据格式的支持。它们中的大多数只是实现streaming API抽象，以便数据绑定组件可以按原样使用。

**官方直接维护：**

> 说明：这些数据格式的模块的groupId均为`<groupId>com.fasterxml.jackson.dataformat</groupId>`，且跟着主版本号走

- Avro/CBOR/Ion/Protobuf/Smile(binary JSON) ：这些均属于二进制的数据格式，它们的artifactId为：`<artifactId>jackson-dataformat-[FORMAT]</artifactId>`
- CSV/Properties/**XML/YAML**：这些格式熟悉吧，同样的支持到了这些常用的文本格式

**非官方直接维护：**
因非官方直接维护的模块过于偏门，因此省略

### 4.JVM平台其它语言

官网有说，Jackson是一个**JVM平台**的解析器，因此语言层面不局限于Java本身，还涵盖了另外两大主流JVM语言：Kotlin和Scala

> 说明：这块的groupId均为：`<groupId>com.fasterxml.jackson.module</groupId>`，版本号跟着主版本号走

- jackson-module-kotlin：处理kotlin源生类型
- jackson-module-scala_[scala版本号]：处理scala源生类型

### 5.模式支持

Jackson注解为POJO定义了预期的属性和预期的处理，除了Jackson本身将其用于读取/写入JSON和其他格式之外，它还允许生成**外部模式**。上面已讲述的数据格式扩展中包含了部分功能，但也仍还有许多独立的模式工具，如：

- Ant Task for JSON Schema Generation：使用Apache Ant时，使用Jackson库和扩展模块从Java类生成JSON
- jackson-json-schema-maven-plugin：maven插件，用于生成JSON

> 说明：本部分因实际应用场景实在太少，为了不要混淆主要内容，此部分后面亦不会再提及

### 6.Jackson jr（用于移动端）

虽然`Jackson databind`（如ObjectMapper）是通用数据绑定的良好选择，但它的**占用空间（Jar包大小）**和**启动开销**在某些领域可能存在问题：比如移动端，特别是对于轻量使用(读或写)。这种case下，完整的Jackson API是让人接受不了的。

由于所有这些原因，Jackson官方决定创建一个**更简单、更小**的库：Jackson jr。它仍旧构建在Streaming API之上，但不依赖于databind和annotation。因此，它的大小(jar和运行时内存使用)要小得多，它的API非常紧凑，所以适合APP等移动端。

```xml
<dependency>
    <groupId>com.fasterxml.jackson.jr</groupId>
    <artifactId>jackson-jr-objects</artifactId>
</dependency>
```

它仅仅只依赖了`jackson-core`模块，所以体积上控制得非常的好。Jackson单单三大核心模块大小合计1700KB左右（320 + 70 + 1370）。而Jackson jr的体积控制在了95KB（就算加上core模块的320也不到500KB）。

而对于开发Java后台的我们对内存并不敏感，简单易用、功能强大才是硬道理。因此`jackson-jr`只是在此处做个简单了解即可，本专栏后面也不会再提及。

