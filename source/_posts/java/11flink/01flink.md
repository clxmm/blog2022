---
title: 01初识Flink
toc: true
tags: flink
categories: 
    - [java]
    - [flink]

---

# 第1章 初识Flink

[https://flink.apache.org/](https://flink.apache.org/)

<!--more-->

在 Flink 官网主页的顶部可以看到，项目的核心目标，是“数据流上的有状态计算”(StatefulComputations over Data Streams)。

![](./img/2023/05/flink1-01.png)

## 1.3 流式数据处理的发展和演变

我们已经了解，Flink 的主要应用场景，就是处理大规模的数据流。那为什么一定要用 Flink 呢?数据处理还有没有其他的方式?要解答这个疑惑，我们就需要先从流处理和批处理的概念 讲起。

### 1.3.1 流处理和批处理

数据处理有不同的方式。

对于具体应用来说，有些场景数据是一个一个来的，是一组有序的数据序列，我们把它叫 作“数据流”;而有些场景的数据，本身就是一批同时到来，是一个有限的数据集，这就是批 量数据(有时也直接叫数据集)。

容易想到，处理数据流，当然应该“来一个就处理一个”，这种数据处理模式就叫作流处 理;因为这种处理是即时的，所以也叫实时处理。与之对应，处理批量数据自然就应该一批读 入、一起计算，这种方式就叫作批处理，也叫作离线处理。

那真实的应用场景中，到底是数据流更常见、还是批量数据更常见呢?

生活中，这两种形式的数据都有，如图 1-4 所示。比如我们日常发信息，可以一句一句地 说，也可以写一大段一起发过去。一句一句的信息，就是一个一个的数据，它们构成的序列就 是一个数据流;而一大段信息，是一组数据的集合，对应就是批量数据(数据集)。

当然，有经验的人都会知道，一句一句地发，你一言我一语，有来有往这才叫聊天;一大 段信息直接砸过去，别人看着都眼晕，很容易就没下文了——如果是很重要的整篇内容(比如 表白信)，写成文档或者邮件发过去可能效果会更好。

当然，有经验的人都会知道，一句一句地发，你一言我一语，有来有往这才叫聊天;一大 段信息直接砸过去，别人看着都眼晕，很容易就没下文了——如果是很重要的整篇内容(比如 表白信)，写成文档或者邮件发过去可能效果会更好。

起来，统一传输、统一处理(当然我们还可以进一步较真:处理也是流式的，字得一个一个读)。 不论传输处理的方式是怎样的，数据的生成，一般都是流式的。

### 1.3.2 传统事务处理

IT 互联网公司往往会用不同的应用程序来处理各种业务。比如内部使用的企业资源规划 (ERP)系统、客户关系管理(CRM)系统，还有面向客户的 Web 应用程序。这些系统一般都 会进行分层设计:“计算层”就是应用程序本身，用于数据计算和处理;而“存储层”往往是传统

的关系型数据库，用于数据存储，如图 1-5 所示。

![](./img/2023/05/flink1-02.png)

我们发现，这里的应用程序在处理数据的模式上有共同之处:接收的数据是持续生成的事 件，比如用户的点击行为，客户下的订单，或者操作人员发出的请求。处理事件时，应用程序 需要先读取远程数据库的状态，然后按照处理逻辑得到结果，将响应返回给用户，并更新数据 库状态。一般来说，一个数据库系统可以服务于多个应用程序，它们有时会访问相同的数据库 或表。

这就是传统的“事务处理”架构。系统所处理的连续不断的事件，其实就是一个数据流。 而对于每一个事件，系统都在收到之后进行相应的处理，这也是符合流处理的原则的。所以可 以说，传统的事务处理，就是最基本的流处理架构。

对于各种事件请求，事务处理的方式能够保证实时响应，好处是一目了然的。但是我们知 道，这样的架构对表和数据库的设计要求很高;当数据规模越来越庞大、系统越来越复杂时， 可能需要对表进行重构，而且一次联表查询也会花费大量的时间，甚至不能及时得到返回结果。 于是，作为程序员就只好将更多的精力放在表的设计和重构，以及 SQL 的调优上，而无法专 注于业务逻辑的实现了——我们都知道，这种工作费力费时，却没法直接体现在产品上给老板 看，简直就是噩梦。

 那有没有更合理、更高效的处理架构呢?

### 1.3.3 有状态的流处理

不难想到，如果我们对于事件流的处理非常简单，例如收到一条请求就返回一个“收到”， 那就可以省去数据库的查询和更新了。但是这样的处理是没什么实际意义的。在现实的应用中， 往往需要还其他一些额外数据。我们可以把需要的额外数据保存成一个“状态”，然后针对这 条数据进行处理，并且更新状态。在传统架构中，这个状态就是保存在数据库里的。这就是所 谓的“有状态的流处理”。

为了加快访问速度，我们可以直接将状态保存在本地内存，如图 1-6 所示。当应用收到一 个新事件时，它可以从状态中读取数据，也可以更新状态。而当状态是从内存中读写的时候， 这就和访问本地变量没什么区别了，实时性可以得到极大的提升。

另外，数据规模增大时，我们也不需要做重构，只需要构建分布式集群，各自在本地计算就可以了，可扩展性也变得更好。

因为采用的是一个分布式系统，所以还需要保护本地状态，防止在故障时数据丢失。我们

可以定期地将应用状态的一致性检查点(checkpoint)存盘，写入远程的持久化存储，遇到故 障时再去读取进行恢复，这样就保证了更好的容错性。

![](./img/2023/05/flink1-03.png)

有状态的流处理是一种通用而且灵活的设计架构，可用于许多不同的场景。具体来说，有

以下几种典型应用。

#### 1. 事件驱动型(Event-Driven)应用

![](./img/2023/05/flink1-04.png)

事件驱动型应用是一类具有状态的应用，它从一个或多个事件流提取数据，并根据到来的 事件触发计算、状态更新或其他外部动作。比较典型的就是以 Kafka 为代表的消息队列几乎都 是事件驱动型应用。

这其实跟传统事务处理本质上是一样的，区别在于基于有状态流处理的事件驱动应用，不 再需要查询远程数据库，而是在本地访问它们的数据，如图 1-7 所示，这样在吞吐量和延迟方 面就可以有更好的性能。

另外远程持久性存储的检查点保证了应用可以从故障中恢复。检查点可以异步和增量地完 成，因此对正常计算的影响非常小。

#### 2. 数据分析(DataAnalysis)型应用

![](./img/2023/05/flink1-05.png)

所谓的数据分析，就是从原始数据中提取信息和发掘规律。传统上，数据分析一般是先将 数据复制到数据仓库(Data Warehouse)，然后进行批量查询。如果数据有了更新，必须将最 新数据添加到要分析的数据集中，然后重新运行查询或应用程序。

如今，Apache Hadoop 生态系统的组件，已经是许多企业大数据架构中不可或缺的组成部 分。现在的做法一般是将大量数据(如日志文件)写入 Hadoop 的分布式文件系统(HDFS)、 S3 或 HBase 等批量存储数据库，以较低的成本进行大容量存储。然后可以通过 SQL-on-Hadoop 类的引擎查询和处理数据，比如大家熟悉的 Hive。这种处理方式，是典型的批处理，特点是 可以处理海量数据，但实时性较差，所以也叫离线分析。

如果我们有了一个复杂的流处理引擎，数据分析其实也可以实时执行。流式查询或应用程 序不是读取有限的数据集，而是接收实时事件流，不断生成和更新结果。结果要么写入外部数 据库，要么作为内部状态进行维护。

Apache Flink 同事支持流式与批处理的数据分析应用，如图 1-8 所示。

与批处理分析相比，流处理分析最大的优势就是低延迟，真正实现了实时。另外，流处理 不需要去单独考虑新数据的导入和处理，实时更新本来就是流处理的基本模式。当前企业对流 式数据处理的一个热点应用就是实时数仓，很多公司正是基于 Flink 来实现的。

#### 3. 数据管道(Data Pipeline)型应用

![](./img/2023/05/flink1-06.png)

ETL 也就是数据的提取、转换、加载，是在存储系统之间转换和移动数据的常用方法。 在数据分析的应用中，通常会定期触发 ETL 任务，将数据从事务数据库系统复制到分析数据 库或数据仓库。

所谓数据管道的作用与 ETL 类似。它们可以转换和扩展数据，也可以在存储系统之间移 动数据。不过如果我们用流处理架构来搭建数据管道，这些工作就可以连续运行，而不需要再 去周期性触发了。比如，数据管道可以用来监控文件系统目录中的新文件，将数据写入事件日志。连续数据管道的明显优势是减少了将数据移动到目的地的延迟，而且更加通用，可以用于 更多的场景。

如图 1-9 所示，展示了 ETL 与数据管道之间的区别。

有状态的流处理架构上其实并不复杂，很多用户基于这种思想开发出了自己的流处理系 统，这就是第一代流处理器。Apache Storm 就是其中的代表。Storm 可以说是开源流处理的先 锋，最早是由 Nathan Marz 和创业公司 BackType 的一个团队开发的，后来才成为 Apache 软 件基金会下属的项目。Storm 提供了低延迟的流处理，但是它也为实时性付出了代价:很难实 现高吞吐，而且无法保证结果的正确性。用更专业的话说，它并不能保证“精确一次”

(exactly-once);即便是它能够保证的一致性级别，开销也相当大。关于状态一致性和 exactly-once，我们会在后续的章节中展开讨论。

### 1.3.4 Lambda 架构

对于有状态的流处理，当数据越来越多时，我们必须用分布式的集群架构来获取更大的吞 吐量。但是分布式架构会带来另一个问题:怎样保证数据处理的顺序是正确的呢?

对于批处理来说，这并不是一个问题。因为所有数据都已收集完毕，我们可以根据需要选 择、排列数据，得到想要的结果。可如果我们采用“来一个处理一个”的流处理，就可能出现“乱序”的现象:本来先发生的事件，因为分布处理的原因滞后了。怎么解决这个问题呢?

以 Storm 为代表的第一代分布式开源流处理器，主要专注于具有毫秒延迟的事件处理，特 点就是一个字“快”;而对于准确性和结果的一致性，是不提供内置支持的，因为结果有可能 取决于到达事件的时间和顺序。另外，第一代流处理器通过检查点来保证容错性，但是故障恢复的时候，即使事件不会丢失，也有可能被重复处理——所以无法保证 exactly-once。

与批处理器相比，可以说第一代流处理器牺牲了结果的准确性，用来换取更低的延迟。而批处理器恰好反过来，牺牲了实时性，换取了结果的准确。

我们自然想到，如果可以让二者做个结合，不就可以同时提供快速和准确的结果了吗?正是基于这样的思想，Lambda 架构被设计出来，如图 1-10 所示。我们可以认为这是第二代流处 理架构，但事实上，它只是第一代流处理器和批处理器的简单合并。

![](./img/2023/05/flink1-07.png)

Lambda 架构主体是传统批处理架构的增强。它的“批处理层”(Batch Layer)就是由传统 的批处理器和存储组成，而“实时层”(Speed Layer)则由低延迟的流处理器实现。数据到达 之后，两层处理双管齐下，一方面由流处理器进行实时处理，另一方面写入批处理存储空间， 等待批处理器批量计算。流处理器快速计算出一个近似结果，并将它们写入“流处理表”中。 而批处理器会定期处理存储中的数据，将准确的结果写入批处理表，并从快速表中删除不准确 的结果。最终，应用程序会合并快速表和批处理表中的结果，并展示出来。

Lambda 架构现在已经不再是最先进的，但仍在许多地方使用。它的优点非常明显，就是 兼具了批处理器和第一代流处理器的特点，同时保证了低延迟和结果的准确性。而它的缺点同 样非常明显。首先，Lambda 架构本身就很难建立和维护;而且，它需要我们对一个应用程序， 做出两套语义上等效的逻辑实现，因为批处理和流处理是两套完全独立的系统，它们的 API 也完全不同。为了实现一个应用，付出了双倍的工作量，这对程序员显然不够友好。

### 1.3.5 新一代流处理器

之前的分布式流处理架构，都有明显的缺陷，人们也一直没有放弃对流处理器的改进和完 善。终于，在原有流处理器的基础上，新一代分布式开源流处理器诞生了。为了与之前的系统 区分，我们一般称之为第三代流处理器，代表当然就是 Flink。

第三代流处理器通过巧妙的设计，完美解决了乱序数据对结果正确性的影响。这一代系统 还做到了精确一次(exactly-once)的一致性保障，是第一个具有一致性和准确结果的开源流 处理器。另外，先前的流处理器仅能在高吞吐和低延迟中二选一，而新一代系统能够同时提供 这两个特性。所以可以说，这一代流处理器仅凭一套系统就完成了 Lambda 架构两套系统的工 作，它的出现使得 Lambda 架构黯然失色。

除了低延迟、容错和结果准确性之外，新一代流处理器还在不断添加新的功能，例如高可 用的设置，以及与资源管理器(如 YARN 或 Kubernetes)的紧密集成等等。

在下一节，我们会将 Flink 的特性做一个总结，从中可以体会到新一代流处理器的强大。

## 1.4 Flink 的特性总结

Flink 是第三代分布式流处理器，它的功能丰富而强大。

### 1.4.1 Flink 的核心特性

Flink 区别与传统数据处理框架的特性如下。

- 高吞吐和低延迟。每秒处理数百万个事件，毫秒级延迟。

- 结果的准确性。Flink 提供了事件时间(event-time)和处理时间(processing-time) 语义。对于乱序事件流，事件时间语义仍然能提供一致且准确的结果。

- 精确一次(exactly-once)的状态一致性保证。

- 可以连接到最常用的存储系统，如 Apache Kafka、Apache Cassandra、Elasticsearch、JDBC、Kinesis 和(分布式)文件系统，如 HDFS 和 S3。

- 高可用。本身高可用的设置，加上与K8s，YARN和Mesos的紧密集成，再加上从故

  障中快速恢复和动态扩展任务的能力，Flink 能做到以极少的停机时间 7×24 全天候

  运行。

- 能够更新应用程序代码并将作业(jobs)迁移到不同的 Flink 集群，而不会丢失应用

  程序的状态。

### 1.4.2 分层 API

除了上述这些特性之外，Flink 还是一个非常易于开发的框架，因为它拥有易于使用的分 层 API，整体 API 分层如图 1-11 所示。

![](./img/2023/05/flink1-11.png)

最底层级的抽象仅仅提供了有状态流，它将处理函数(Process Function)嵌入到了 DataStream API 中。底层处理函数(Process Function)与 DataStream API 相集成，可以对某 些操作进行抽象，它允许用户可以使用自定义状态处理来自一个或多个数据流的事件，且状态 具有一致性和容错保证。除此之外，用户可以注册事件时间并处理时间回调，从而使程序可以 处理复杂的计算。

实际上，大多数应用并不需要上述的底层抽象，而是直接针对核心 API(Core APIs) 进 行编程，比如 DataStream API(用于处理有界或无界流数据)以及 DataSet API(用于处理有界 数据集)。这些 API 为数据处理提供了通用的构建模块，比如由用户定义的多种形式的转换

(transformations)、连接(joins)、聚合(aggregations)、窗口(windows)操作等。DataSet API为有界数据集提供了额外的支持，例如循环与迭代。这些 API 处理的数据类型以类(classes) 的形式由各自的编程语言所表示。

Table API 是以表为中心的声明式编程，其中表在表达流数据时会动态变化。Table API 遵 循关系模型:表有二维数据结构(schema)(类似于关系数据库中的表)，同时 API 提供可比 较的操作，例如 select、join、group-by、aggregate 等。

尽管 Table API 可以通过多种类型的用户自定义函数(UDF)进行扩展，仍不如核心 API 更具表达能力，但是使用起来代码量更少，更加简洁。除此之外，Table API程序在执行之前 会使用内置优化器进行优化。

我们可以在表与 DataStream/DataSet 之间无缝切换，以允许程序将 Table API 与 DataStream 以及 DataSet 混合使用。

Flink 提供的最高层级的抽象是 SQL。这一层抽象在语法与表达能力上与 Table API 类似， 但是是以 SQL 查询表达式的形式表现程序。SQL 抽象与 Table API 交互密切，同时 SQL 查询 可以直接在 Table API 定义的表上执行。

目前 Flink SQL 和 Table API 还在开发完善的过程中，很多大厂都会二次开发符合自己需 要的工具包。而 DataSet 作为批处理 API 实际应用较少，2020 年 12 月 8 日发布的新版本 1.12.0, 已经完全实现了真正的流批一体，DataSet API 已处于软性弃用(soft deprecated)的状态。用 Data Stream API 写好的一套代码, 即可以处理流数据, 也可以处理批数据，只需要设置不同的 执行模式。这与之前版本处理有界流的方式是不一样的，Flink 已专门对批处理数据做了优化 处理。本书中以介绍 DataStream API 为主，采用的是目前最新版本 Flink 1.13.0。

## 1.5 Flink vs Spark

谈到大数据处理引擎，不能不提 Spark。Apache Spark 是一个通用大规模数据分析引擎。 它提出的内存计算概念让大家耳目一新，得以从 Hadoop 繁重的 MapReduce 程序中解脱出来， 可以说是划时代的大数据处理框架。除了计算速度快、可扩展性强，Spark 还为批处理(Spark SQL)、流处理(Spark Streaming)、机器学习(Spark MLlib)、图计算(Spark GraphX)提供 了统一的分布式数据处理平台，整个生态经过多年的蓬勃发展已经非常完善。

然而正在大家认为 Spark 已经如日中天、即将一统天下之际，Flink 如一颗新星异军突起， 使得大数据处理的江湖再起风云。很多读者在最初接触都会有这样的疑问:想学习一个大数据 处理框架，到底选择 Spark，还是 Flink 呢?

这就需要我们了解两者的主要区别，理解它们在不同领域的优势。

### 1.5.1 数据处理架构

我们已经知道，数据处理的基本方式，可以分为批处理和流处理两种。

批处理针对的是有界数据集，非常适合需要访问海量的全部数据才能完成的计算工作，一 般用于离线统计。

流处理主要针对的是数据流，特点是无界、实时, 对系统传输的每个数据依次执行操作， 一般用于实时统计。

从根本上说，Spark 和 Flink 采用了完全不同的数据处理方式。可以说，两者的世界观是 截然相反的。

Spark 以批处理为根本，并尝试在批处理之上支持流计算;在 Spark 的世界观中，万物皆 批次，离线数据是一个大批次，而实时数据则是由一个一个无限的小批次组成的。所以对于流 处理框架 Spark Streaming 而言，其实并不是真正意义上的“流”处理，而是“微批次”

(micro-batching)处理，如图 1-12 所示。。

![](./img/2023/05/flink1-12.png)

而 Flink 则认为，流处理才是最基本的操作，批处理也可以统一为流处理。在 Flink 的世 界观中，万物皆流，实时数据是标准的、没有界限的流，而离线数据则是有界限的流。如图 1-13 所示，就是所谓的无界流和有界流。

#### 1. 无界数据流(Unbounded Data Stream)

所谓无界数据流，就是有头没尾，数据的生成和传递会开始但永远不会结束，如图 1-13 所示。我们无法等待所有数据都到达，因为输入是无界的，永无止境，数据没有“都到达”的 时候。所以对于无界数据流，必须连续处理，也就是说必须在获取数据后立即处理。在处理无 界流时，为了保证结果的正确性，我们必须能够做到按照顺序处理数据。

#### 2. 有界数据流(Bounded Data Stream)

对应的，有界数据流有明确定义的开始和结束，如图 1-13 所示，所以我们可以通过获取 所有数据来处理有界流。处理有界流就不需要严格保证数据的顺序了，因为总可以对有界数据 集进行排序。有界流的处理也就是批处理。

![](./img/2023/05/flink1-13.png)

正因为这种架构上的不同，Spark 和 Flink 在不同的应用领域上表现会有差别。一般来说， Spark 基于微批处理的方式做同步总有一个“攒批”的过程，所以会有额外开销，因此无法在 流处理的低延迟上做到极致。在低延迟流处理场景，Flink 已经有明显的优势。而在海量数据 的批处理领域，Spark 能够处理的吞吐量更大，加上其完善的生态和成熟易用的 API，目前同 样优势比较明显。

### 1.5.2 数据模型和运行架构

除了三观不合，Spark 和 Flink 在底层实现最主要的差别就在于数据模型不同。

Spark 底层数据模型是弹性分布式数据集(RDD)，Spark Streaming 进行微批处理的底层 接口 DStream，实际上处理的也是一组组小批数据 RDD 的集合。可以看出，Spark 在设计上本 身就是以批量的数据集作为基准的，更加适合批处理的场景。

而 Flink 的基本数据模型是数据流(DataFlow)，以及事件(Event)序列。Flink 基本上是 完全按照 Google 的 DataFlow 模型实现的，所以从底层数据模型上看，Flink 是以处理流式数 据作为设计目标的，更加适合流处理的场景。

数据模型不同，对应在运行处理的流程上，自然也会有不同的架构。Spark 做批计算，需 要将任务对应的 DAG 划分阶段(Stage)，一个完成后经过 shuffle 再进行下一阶段的计算。而 Flink 是标准的流式执行模式，一个事件在一个节点处理完后可以直接发往下一个节点进行处 理。

### 1.5.3 Spark 还是 Flink?

通过前文的分析，我们已经可以看出，Spark 和 Flink 可以说目前是各擅胜场，批处理领 域 Spark 称王，而在流处理方面 Flink 当仁不让。具体到项目应用中，不仅要看是流处理还是 批处理，还需要在延迟、吞吐量、可靠性，以及开发容易度等多个方面进行权衡。

如果在工作中需要从 Spark 和 Flink 这两个主流框架中选择一个来进行实时流处理，我们 更加推荐使用 Flink，主要的原因有:

- Flink的延迟是毫秒级别，而SparkStreaming的延迟是秒级延迟。
- Flink提供了严格的精确一次性语义保证。
- Flink的窗口API更加灵活、语义更丰富。
- Flink提供事件时间语义，可以正确处理延迟数据。
- Flink提供了更加灵活的对状态编程的API。

基于以上特点，使用 Flink 可以解放程序员, 加快编程效率, 把本来需要程序员花大力气 手动完成的工作交给框架完成。

当然，在海量数据的批处理方面，Spark 还是具有明显的优势。而且 Spark 的生态更加成熟，也会使其在应用中更为方便。相信随着 Flink 的快速发展和完善，这方面的差距会越来越 小。

另外，Spark 2.0 之后新增的 Structured Streaming 流处理引擎借鉴 DataFlow 进行了大量优 化，同样做到了低延迟、时间正确性以及精确一次性语义保证;Spark 2.3 以后引入的连续处 理(Continuous Processing)模式，更是可以在至少一次语义保证下做到 1 毫秒的延迟。而 Flink 自 1.9 版本合并 Blink 以来，在 SQL 的表达和批处理的能力上同样有了长足的进步。

那如果现在要学习一门框架的话，优先选 Spark 还是 Flink 呢?其实我们可以看到，不同 的框架各有利弊，同时它们也在互相借鉴、取长补短、不断发展，至于未来是 Spark 还是 Flink、 甚至是其他新崛起的处理引擎一统江湖，都是有可能的。作为技术人员，我们应该对不同的架 构和思想都有所了解，跳出某个框架的限制，才能看到更广阔的世界。

## 1.6 本章总结

本章作为学习 Flink 的入门和综述，主要介绍了 Flink 的源起和应用，引出了流处理相关 的一些重要概念，并通过介绍数据处理架构发展演变的过程，为读者展示了 Flink 作为新一代 分布式流处理器的架构思想。最后我们还将 Flink 与时下同样火热的处理引擎 Spark 进行了对 比，详细阐述了 Flink 在流处理方面的优势。

通过本章的学习，大家不仅可以初步了解 Flink，而且能够建立起数据处理的宏观思维， 这对以后学习框架中的一些重要特性非常有帮助。

