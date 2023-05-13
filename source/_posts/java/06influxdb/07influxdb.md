---
title: 07influxdb.md
toc: true
tags: influxdb
categories: 
    - [db]
    - [influxdb]
---

# 第**17**章 从 **InfluxDB OSS** 迁移数据

## **17.1** 将 **InfluxDB** 中的数据导出

导出 InfluxDB 数据必须使用 influxd 命令(注意，不是 influx 命令)。

<!--more-->

在 InfluxDB2.x 中，数据导出是以存储桶为单位的。

```ini
influxd inspect export-lp \ 
	--bucket-id 12ab34cd56ef \ 
	--engine-path ~/.influxdbv2/engine \ 
	--output-path path/to/export.lp 
	--start 2022-01-01T00:00:00Z \ 
	--end 2022-01-31T23:59:59Z \ 
	--compress
```

- influxd inspect，influxd 是可以操作 InfluxDB 服务进程的命令行工具，inspect 是 influxd 命令的子命令，使用 inspect 可以
- export-lp，是 export xxx to line protocol 的缩写，表示将数据导出为行协议。它是 inspect 的子命令
- bucket-id，inspect 的必须参数。存储桶的 id
- engine-path，inspect 的必须参数，不过有默认值~/.influxdbv2/engine。所以如果你的 数据目录是~/.influxdbv2/engine 那么不指定这个参数也行。
- output-path，inspect 的必须参数，指定输出文件的位置。
- start，非必须，导出数据的开始时间
- end，非必须，导出数据的结束时间。
- compress，建议启用，如果启用了，那么 influxd 会使用 gzip 的方式压缩输出的数据。

## **17.2** 示例:将 **InfluxDB** 中的数据导出

这次，我们尝试导出 demo_java1 的数据导出，截至目前，这个 bucket 里面的数据应该是 当前最多的。

- (1)首先，你可以使用 influx-cli 也可以使用 Web UI 来查看我们想要导出的 bucket 对 应的 ID。这里，课程选择使用 Web UI，可以看到 test_init 存储桶的 ID 为 ebf193d0770ccae7。

- (2)于是，我们运行下面的命令，尝试把数据导出。

  ```ini
  ./influxd inspect export-lp \
  --engine-path /var/lib/influxdb2/engine \
  --bucket-id ebf193d0770ccae7 \
  --output-path ./oh.lp
  ```

  这条命令会把 test_init 存储桶里的数据以 InfluxDB 行协议的格式导出到当前目录下的 oh.lp 文件中。

  正常情况下，程序会输出一系列读写信息。

  ![](./img/2023/04/influxdb07-1.png)

  - (3)使用下面的命令查看当前路径下的文件及其大小。

    ```ini
    ls -lh
    ```

    ls 的 h 参数，可以将文件的字节数打印为更容易阅读的 MB、GB 单位。

  - 4)现在，我们使用 tail 命令来查看一下文件的内容。

    ```ini
    tail -15 ./oh.lp
    ```

## **17.3** 示例:导出数据时压缩

- 1)现在，我们重新运行数据导出的命令，这次在命令的最后加上--compress 参数。

  ````ini
  influxd inspect export-lp \
  --engine-path /var/lib/influxdb2/engine \
  --bucket-id ebf193d0770ccae7 \
  --output-path ./oh.lp \
  --compress
  ````

  不必担心目录下已经存在 oh.lp 文件，程序会直接将其覆盖的。

- 2)使用 ls 命令再次查看文件大小。

# 第**18**章 **FLUX** 查询优化

## **18.1** 使用谓词下推的查询

谓词下推常见于 SQL 查询中，一个 SQL 中的谓词，通常指的是 where 条件。

我们看一个最简单的 SQL 语句。它从一个名为 A 的表中查询数据，并按照 n>10 的条 件对数据进行过滤。

```sql
 select *
from A
 where n > 10
```

![](./img/2023/05/influxdb07-2.png)

- 一种是将磁盘里的数据全部读到内存中，再在内存中进行过滤。这种方式我们通常 说它没有内存下推
- 另一种是在查询时，就只从磁盘取自己需要的数据到内存中，再进行下一步的操作。 通常，我们说这种方式实现了谓词下推。
- 虽然说 FLUX 语言表面上是一个脚本语言，但在查询这件事上，它并不是老老实实一 行行执行的，而是有了优化器的参与。FLUX 语言在执行时，会尽可能实现谓词下推的优 化，什么样的查询可以实现谓词下推，可以参考官网文档的优化查询一节 https://docs.influxdata.com/influxdb/v2.4/query-data/optimize-queries/

另外，后面我们会告诉大家如何去查看一个查询的执行计划。

## **18.2** 避免将窗口宽度设得过小

窗口(基于时间间隔对数据进行分组)通常用于聚合和降采样数据。将窗口设长一点 可以提高性能。窗口过窄会导致需要更多的算力来评估每条数据应该分配到哪个窗口，合 理的窗口宽度应该根据查询的总时间宽度来决定。

## **18.3** 避免使用“沉重”的功能

下面的这些函数对于 FLUX 来说会比较很重，这些函数会使用更多的内存和 CPU，使 用这些函数时要想要是否必要。

- map()
- reduce()
- join()
- union()
- pivot()

不过官方又说，InfluxData 一直在优化 FLUX 的性能，所以当前的列表不一定是将来

## **18.4** 尽可能使用 **set**()而不是 **map**()

如果你要给数据查一个静态常量，那么 set 比 map 要有很大的性能优势。map 是我们上一小节说的沉重操作。在后面的示例，我们会比较两种操作的差距。

## **18.5** 平衡数据的时间范围和数据精度

想要保证查询的性能良好，应该平衡好查询的时间范围和数据精度。如果，有一个 measurement 的数据每秒入库一条，你一次请求 6 个月的数据，那么一个序列就能包含 1550 万点数据。如果序列数再多一些，那么数据很可能会变成数十亿点。Flux 必须将这些 数据拉到内存再返回给用户。所以一方面做好谓词下推尽量减少对内存的使用。另外，如 果必须要查询很长时间范围的数据，那应该创建一个定时任务来对数据进行降采样，然后 将查询目标从原始数据改为降采样数据。

## **18.6** 使用 **FLUX** 性能分析工具查看查询性能

执行 FLUX 查询时，你可以导入一个名为 profiler 的包，然后添加一个 option 选项以查

看当前 FLUX 语句的执行计划。比如:

```ini
option profiler.enabledProfilers = ["query", "operator"]
```

这里的 query 和 operator 是查询计划的两个选项，query 表示你要查看整个执行脚本的 的执行情况，operator 表示你要查看一个 FLUX 查询各个算子的执行情况。

## **18.6.1 query**(查询)

query 提供有关整个 Flux 脚本执行的统计信息。启用后，结果将多出一个表，其中包 含以下信息:

- TotalDuration:查询总持续时间(以纳秒为单位)
- CompileDuration:编译查询脚本所花费的时间(以纳秒为单位)
- QueueDuration:排队所花费的时间(以纳秒为单位)
- RequeueDration:重新排队花费的时间(以纳秒为单位)
- PlanDuration:计划查询所花费的时间(以纳秒为单位)
- ExecuteDuration:执行查询所花费的时间(以纳秒为单位)
- Concurrency:并发，分配给处理查询的 goroutines。
- MaxAllocated:查询分配的最大字节数(内存)
- TotalAllocated:查询时分配的总字节数(包括释放然后再次使用的内存)
- RuntimeErrors:查询执行期间返回的错误消息
- flux/query-plan:flux 查询计划
- influxdb/scanned-values:数据库扫描磁盘的数据条数
- influxdb/scanned-buytes:数据库扫描磁盘的字节数

## **18.6.2 operator**(算子)

有关一个查询脚本中每个操作的统计信息。在存储层中执行的操作将作为单个操作返 回。启用此配置后，返回的结果将多出一个表，并包含以下内容

- Type:操作类型
- Label:标签
- Count:执行这个操作的总次数
- MinDuration:操作被执行多次中，最快的一次花费的时间(以纳秒为单位)
- MaxDuration:操作被执行多次中，最慢的一次花费的时间(以纳秒为单位)
- DurationSum:当前操作完成的总持续时间(以纳秒为单位)。
- MeanDuration:操作被执行多次的平均持续时间(以纳秒为单位)。





