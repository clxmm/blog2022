---
title: 02flink
toc: true
tags: flink
categories: 
    - [java]
    - [flink]
---

# 第 2 章 Flink 快速上手

对 Flink 有了基本的了解后，接下来就要理论联系实际，真正上手写代码了。Flink 底层是 以 Java 编写的，并为开发人员同时提供了完整的 Java 和 Scala API。在本书中，代码示例将全 部用 Java 实现;而在具体项目应用中，可以根据需要选择合适语言的 API 进行开发。

<!--more-->

在这一章，我们将会以大家最熟悉的 IntelliJ IDEA 作为开发工具，用实际项目中最常见的 Maven 作为包管理工具，在开发环境中编写一个简单的 Flink 项目，实现零基础快速上手。

## 2.1 环境准备

- 系统环境为 Windows 10。
- 需提前安装 Java 8。
- 集成开发环境(IDE)使用 IntelliJ IDEA，具体的安装流程参见 IntelliJ 官网。
- 安装IntelliJIDEA之后，还需要安装一些插件——Maven和Git。Maven用来管理项目依赖;通过 Git 可以轻松获取我们的示例代码

## 2.2 创建项目

###  2. 添加项目依赖

在项目的 pom 文件中，增加\<properties>标签设置属性，然后增加\<denpendencies>标签引 入需要的依赖。我们需要添加的依赖最重要的就是 Flink 的相关组件，包括 flink-java、 flink-streaming-java，以及 flink-clients(客户端，也可以省略)。另外，为了方便查看运行日志

 我们引入 slf4j 和 log4j 进行日志管理。


```xml

<properties>
    <flink.version>1.13.0</flink.version>
    <java.version>1.8</java.version>
    <scala.binary.version>2.12</scala.binary.version>
    <slf4j.version>1.7.30</slf4j.version>
</properties>


<dependencies>
        <!-- 引入 Flink 相关依赖-->
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-java</artifactId>
        </dependency>

        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-streaming-java_${scala.binary.version}</artifactId>
        </dependency>

        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-clients_${scala.binary.version}</artifactId>
        </dependency>

        <!-- 引入日志管理相关依赖-->
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
        </dependency>
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-log4j12</artifactId>
        </dependency>
        <dependency>
            <groupId>org.apache.logging.log4j</groupId>
            <artifactId>log4j-to-slf4j</artifactId>
        </dependency>

        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
        </dependency>

    </dependencies>
```

这里做一点解释:

在属性中，我们定义了\<scala.binary.version>，这指代的是所依赖的 Scala 版本。这有一点 奇怪:Flink 底层是 Java，而且我们也只用 Java API，为什么还会依赖 Scala 呢?这是因为 Flink 的架构中使用了 Akka 来实现底层的分布式通信，而 Akka 是用 Scala 开发的。我们本书中用到 的 Scala 版本为 2.12。

### 3. 配置日志管理

在目录 src/main/resources 下添加文件:log4j.properties，内容配置如下:

```properties
log4j.rootLogger=error, stdout
log4j.appender.stdout=org.apache.log4j.ConsoleAppender
log4j.appender.stdout.layout=org.apache.log4j.PatternLayout
log4j.appender.stdout.layout.ConversionPattern=%-4r [%t] %-5p %c %x - %m%n
```

## 2.3 编写代码

搭好项目框架，接下来就是我们的核心工作——往里面填充代码。我们会用一个最简单的 示例来说明 Flink 代码怎样编写:统计一段文字中，每个单词出现的频次。这就是传说中的 WordCount 程序——它是大数据领域非常经典的入门案例，地位等同于初学编程语言时的

Hello World。我们的源码位于 src/main/java 目录下。

我们已经知道，尽管 Flink 自身的定位是流式处理引擎，但它同样拥有批处理的能力。所以接下来，我们会针对不同的处理模式、不同的输入数据形式，分别讲述 WordCount 代码的 实现。

### 2.3.1 批处理

对于批处理而言，输入的应该是收集好的数据集。这里我们可以将要统计的文字，写入一 个文本文档，然后读取这个文件处理数据就可以了。

在工程根目录下新建一个 input 文件夹，并在下面创建文本文件 words.txt

在 words.txt 中输入一些文字，例如:

```tex
hello world
hello flink
hello java
```

3)在 com.atguigu.chapter02 包下新建 Java 类 BatchWordCount，在静态 main 方法中编 写测试代码。

我们进行单词频次统计的基本思路是:先逐行读入文件数据，然后将每一行文字拆分成单 词;接着按照单词分组，统计每组数据的个数，就是对应单词的频次。

具体代码实现如下:

```java
package org.clxmm.flink.demo1;

import org.apache.flink.api.common.typeinfo.Types;
import org.apache.flink.api.java.ExecutionEnvironment;
import org.apache.flink.api.java.operators.AggregateOperator;
import org.apache.flink.api.java.operators.DataSource;
import org.apache.flink.api.java.operators.FlatMapOperator;
import org.apache.flink.api.java.operators.UnsortedGrouping;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.datastream.KeyedStream;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.util.Collector;

import java.io.File;
import java.util.Arrays;

/**
 * @author clxmm
 * @Description
 * @create 2023-05-18 20:32
 */
public class BatchWorldCount {


    public static void main(String[] args) throws Exception {

        // 1. 创建流式执行环境
        ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
        // 2. 读取文件
        DataSource<String> lineDSS = env.readTextFile("input/words.txt");

        // 3.将每行数据进行拆分，转换成二元组
        FlatMapOperator<String, Tuple2<String, Long>> wordAndOne = lineDSS.flatMap((String line, Collector<Tuple2<String, Long>> out) -> {
            String[] words = line.split(" ");
            for (String word : words) {
                out.collect(Tuple2.of(word, 1L));
            }
        }).returns(Types.TUPLE(Types.STRING, Types.LONG));//当Lambda表达式 使用 Java 泛型的时候, 由于泛型擦除的存在, 需要显示的声明类型信息


        // 4. 按照 word 进行分组
        UnsortedGrouping<Tuple2<String, Long>> wordAndOneUG = wordAndOne.groupBy(0);

        // 5. 分组内聚合统计
        AggregateOperator<Tuple2<String, Long>> sum = wordAndOneUG.sum(1);

        // 6. 打印结果
        sum.print();

    }


}

```



```ini
(flink,1)
(world,1)
(hello,3)
(java,1)

```

代码说明和注意事项:

- Flink 在执行应用程序前应该获取执行环境对象，也就是运行时上下文环境。

  - ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

- 2 Flink 同时提供了 Java 和 Scala 两种语言的 API，有些类在两套 API 中名称是一样的。所以在引入包时，如果有 Java 和 Scala 两种选择，要注意选用 Java 的包。

- 3 直接调用执行环境的 readTextFile 方法，可以从文件中读取数据。

- 4我们的目标是将每个单词对应的个数统计出来，所以调用 flatmap 方法可以对一行文字 进行分词转换。将文件中每一行文字拆分成单词后，要转换成(word,count)形式的二元组，初 始count都为1。returns方法指定的返回数据类型Tuple2，就是Flink自带的二元组数据类型

- 5 在分组时调用了 groupBy 方法，它不能使用分组选择器，只能采用位置索引或属性名 称进行分组。

  - ```java
    // 使用索引定位 
    dataStream.groupBy(0)
    // 使用类属性名称 
      dataStream.groupBy("id")
    ```

- 5 在分组之后调用 sum 方法进行聚合，同样只能指定聚合字段的位置索引或属性名称。

需要注意的是，这种代码的实现方式，是基于 DataSet API 的，也就是我们对数据的处理 转换，是看作数据集来进行操作的。事实上 Flink 本身是流批统一的处理架构，批量的数据集

本质上也是流，没有必要用两套不同的 API 来实现。所以从 Flink 1.12 开始，官方推荐的做法 是直接使用 DataStream API，在提交任务时通过将执行模式设为 BATCH 来进行批处理:

```ini
$ bin/flink run -Dexecution.runtime-mode=BATCH BatchWordCount.jar
```

这样，DataSet API 就已经处于“软弃用”(soft deprecated)的状态，在实际应用中我们只 要维护一套 DataStream API 就可以了。这里只是为了方便大家理解，我们依然用 DataSet API 做了批处理的实现。

### 2.3.2 流处理

我们已经知道，用 DataSet API 可以很容易地实现批处理;与之对应，流处理当然可以用 DataStream API 来实现。对于 Flink 而言，流才是整个处理逻辑的底层核心，所以流批统一之 后的 DataStream API 更加强大，可以直接处理批处理和流处理的所有场景。

DataStream API 作为“数据流”的处理接口，又怎样处理批数据呢?

回忆一下上一章中我们讲到的 Flink 世界观。在 Flink 的视角里，一切数据都可以认为是 流，流数据是无界流，而批数据则是有界流。所以批处理，其实就可以看作有界流的处理。

对于流而言，我们会在获取输入数据后立即处理，这个过程是连续不断的。当然，有时我 们的输入数据可能会有尽头，这看起来似乎就成了一个有界流;但是它跟批处理是截然不同的 ——在输入结束之前，我们依然会认为数据是无穷无尽的，处理的模式也仍旧是连续逐个处理。

#### 1.读取文件

我们同样试图读取文档 words.txt 中的数据，并统计每个单词出现的频次。这是一个“有 界流”的处理，整体思路与之前的批处理非常类似，代码模式也基本一致。

```java
package org.clxmm.flink.demo1;

import org.apache.flink.api.common.typeinfo.Types;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.datastream.KeyedStream;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.util.Collector;

import java.util.Arrays;

/**
 * @author clxmm
 * @Description
 * @create 2023-05-22 20:16
 */
public class BoundedStreamWordCount {


    public static void main(String[] args) throws Exception {
        // 1. 创建流式执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        // 2.读取文本流
        DataStreamSource<String> lineDss = env.readTextFile("input/words.txt");


        // 3. 转换数据格式
        SingleOutputStreamOperator<Tuple2<String, Long>> wordAndOne = lineDss
                .flatMap((String line, Collector<String> words) -> {
                    Arrays.stream(line.split(" ")).forEach(words::collect);
                })
                .returns(Types.STRING)
                .map(word -> Tuple2.of(word, 1L))
                .returns(Types.TUPLE(Types.STRING, Types.LONG));
        // 4. 分组
        KeyedStream<Tuple2<String, Long>, String> wordAndOneKS = wordAndOne
                .keyBy(t -> t.f0);

        // 5.
        SingleOutputStreamOperator<Tuple2<String, Long>> result = wordAndOneKS
                .sum(1);
        // 6. 打印
        result.print();
        // 7. 执行
        env.execute();

    }
}

```

主要观察与批处理程序 BatchWordCount 的不同:

- 创建执行环境的不同，流处理程序使用的是StreamExecutionEnvironment。
- 每一步处理转换之后，得到的数据对象类型不同。
- 分组操作调用的是 keyBy 方法，可以传入一个匿名函数作为键选择器(KeySelector)，指定当前分组的 key 是什么。
- 代码末尾需要调用env的execute方法，开始执行任务。

运行程序，控制台输出结果如下:

```ini
3> (world,1)
2> (hello,1)
4> (flink,1)
2> (hello,2)
2> (hello,3)
1> (java,1)
```

我们可以看到，这与批处理的结果是完全不同的。批处理针对每个单词，只会输出一个最 终的统计个数;而在流处理的打印结果中，“hello”这个单词每出现一次，都会有一个频次统计 数据输出。这就是流处理的特点，数据逐个处理，每来一条数据就会处理输出一次。我们通过 打印结果，可以清晰地看到单词“hello”数量增长的过程。

看到这里大家可能又会有新的疑惑:我们读取文件，第一行应该是“hello flink”，怎么这 里输出的第一个单词是“world”呢?每个输出的结果二元组，前面都有一个数字，这又是什 么呢?

我们可以先做个简单的解释。Flink 是一个分布式处理引擎，所以我们的程序应该也是分 布式运行的。在开发环境里，会通过多线程来模拟 Flink 集群运行。所以这里结果前的数字， 其实就指示了本地执行的不同线程，对应着 Flink 运行时不同的并行资源。这样第一个乱序的 问题也就解决了:既然是并行执行，不同线程的输出结果，自然也就无法保持输入的顺序了。

另外需要说明，这里显示的编号为 1~4，是由于运行电脑的 CPU 是 4 核，所以默认模拟 的并行线程有 4 个。这段代码不同的运行环境，得到的结果会是不同的。关于 Flink 程序并行 执行的数量，可以通过设定“并行度”(Parallelism)来进行配置，我们会在后续章节详细讲解 这些内容。

#### 2. 读取文本流

在实际的生产环境中，真正的数据流其实是无界的，有开始却没有结束，这就要求我们需 要保持一个监听事件的状态，持续地处理捕获的数据。

为了模拟这种场景，我们就不再通过读取文件来获取数据了，而是监听数据发送端主机的 指定端口，统计发送来的文本数据中出现过的单词的个数。具体实现上，我们只要对 BoundedStreamWordCount 代码中读取数据的步骤稍做修改，就可以实现对真正无界流的处理。

(1) 新建一个 Java 类 StreamWordCount，将 BoundedStreamWordCount 代码中读取文件 数据的 readTextFile 方法，替换成读取 socket 文本流的方法 socketTextStream。具体代码实现如 下:

```java
package org.clxmm.flink.demo1;

import org.apache.flink.api.common.typeinfo.Types;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.streaming.api.datastream.DataStreamSource;
import org.apache.flink.streaming.api.datastream.KeyedStream;
import org.apache.flink.streaming.api.datastream.SingleOutputStreamOperator;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.util.Collector;

import java.util.Arrays;

/**
 * @author clxmm
 * @Description 读取文本流
 * @create 2023-05-22 20:42
 */
public class StreamWordCount {


    public static void main(String[] args) throws Exception {

        // 1. 创建流式执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        // 2. 读取文本流
        DataStreamSource<String> lineDSS = env.socketTextStream ("kubernetes.docker.internal", 7777);

        // 3. 转换数据格式
        SingleOutputStreamOperator<Tuple2<String, Long>> wordAndOne = lineDSS
                .flatMap((String line, Collector<String> words) -> {
                    Arrays.stream(line.split(" ")).forEach(words::collect);
                })
                .returns(Types.STRING).
                map(word -> Tuple2.of(word, 1L))
                .returns(Types.TUPLE(Types.STRING, Types.LONG));

        // 4. 分组
        KeyedStream<Tuple2<String, Long>, String> wordAndOneKS = wordAndOne
                .keyBy(t -> t.f0);

        // 5. 求和
        SingleOutputStreamOperator<Tuple2<String, Long>> result = wordAndOneKS
                .sum(1);

        // 6. 打印
        result.print();

        // 7. 执行
        env.execute();
    }




}

```

代码说明和注意事项:

	-  socket文本流的读取需要配置两个参数:发送端主机名和端口号。这里代码中指定了主机“hadoop102”的 7777 端口作为发送数据的 socket 端口，读者可以根据 测试环境自行配置。
	-  在实际项目应用中，主机名和端口号这类信息往往可以通过配置文件，或者 传入程序运行参数的方式来指定。
	-  socket文本流数据的发送，可以通过Linux系统自带的netcat工具进行模拟。 (2)在 Linux 环境的主机 hadoop102 上，执行下列命令，发送数据进行测试:

在 Linux 环境的主机 hadoop102 上，执行下列命令，发送数据进行测试:

```shell
nc -lk 7777
```

启动 StreamWordCount 程序

我们会发现程序启动之后没有任何输出、也不会退出。这是正常的——因为 Flink 的流处 理是事件驱动的，当前程序会一直处于监听状态，只有接收到数据才会执行任务、输出统计结 果。

从 hadoop102 发送数据:

```ini
hello flink
hello world
hello java
```

可以看到控制台输出结果如下:

```ini
4> (flink,1)
2> (hello,1)
3> (world,1)
2> (hello,2)
2> (hello,3)
1> (java,1)
```

我们会发现，输出的结果与之前读取文件的流处理非常相似。而且可以非常明显地看到， 每输入一条数据，就有一次对应的输出。具体对应关系是:输入“hello flink”，就会输出两条 统计结果(flink，1)和(hello，1);之后再输入“hello world”，同样会将 hello 和 world 的个数统计输出，hello 的个数会对应增长为 2。

## 2.4 本章总结

本章主要实现一个 Flink 开发的入门程序——词频统计 WordCount。通过批处理和流处理 两种不同模式的实现，可以对 Flink 的 API 风格和编程方式有所熟悉，并且更加深刻地理解批 处理和流处理的不同。另外，通过读取有界数据(文件)和无界数据(socket 文本流)进行流 处理的比较，我们也可以更加直观地体会到 Flink 流处理的方式和特点。

这是我们 Flink 长征路上的第一步，是后续学习的基础。有了这番初体验，想必大家会发 现 Flink 提供了非常易用的 API，基于它进行开发并不是难事。之后我们会逐步深入展开，为 大家打开 Flink 神奇世界的大门。

```java
echo "192.168.42.109  tes1" >> /etc/hosts
```



```ini
kubeadm init \
--apiserver-advertise-address=192.168.42.109 \
--control-plane-endpoint=test1 \
--image-repository registry.cn-hangzhou.aliyuncs.com/lfy_k8s_images \
--kubernetes-version v1.20.9 \
--service-cidr=10.96.0.0/16 \
--pod-network-cidr=192.168.0.0/16
```





```ini
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of control-plane nodes by copying certificate authorities
and service account keys on each node and then running the following as root:

  kubeadm join test1:6443 --token 0czgvo.46ne8s2f649sueyr \
    --discovery-token-ca-cert-hash sha256:a3eec992a2dfae6c229c05f709687921547100535c4ee1ffe8c4860cd4c5b92e \
    --control-plane 

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join test1:6443 --token 0czgvo.46ne8s2f649sueyr \
    --discovery-token-ca-cert-hash sha256:a3eec992a2dfae6c229c05f709687921547100535c4ee1ffe8c4860cd4c5b92e 
```



```
curl https://docs.projectcalico.org/archive/v3.21/manifests/calico.yaml -O
```





