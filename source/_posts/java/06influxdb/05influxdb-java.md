---
title: 05influxdb-java
toc: true
tags: influxdb
categories: 
    - [db]
    - [influxdb]
---

# 第**11**章 **JAVA**操作**InfluxDB**

InfluxDB 客户端可以参考:https://github.com/influxdata/influxdb-client-java

<!--more-->

## **11.1** 创建一个 **maven** 项目

## **11.2** 导入 **maven** 依赖

```xml
<dependency>
  <groupId>com.influxdb</groupId>
  <artifactId>influxdb-client-java</artifactId>
  <version>6.6.0</version>
</dependency>
```

## **11.3** 创建一个 **package**

## **11.4** 示例:查看 **InfluxDB** 健康状态

### **11.4.1** 创建 **ExampleHealthy** 类

### **11.4.2** 创建客户端对象

influxdb-client-java 内部其实封装的是各种 HTTP 请求，所以 token，org 什么 HTTPAPI 上需要的东西，在创建客户端的时候都需要考虑。

![](./img/2023/04/influxdb05-1.png)

从上图可以看到 InfluxDBClientFactory.create 方法其实有多种重载。这是因为不同的接 口它需要的权限和操作的范围不同。比如一个读写权限的 token，它只能对某个存储桶进行 操作，那么建立连接时就应该指定 bucket，也就是使用下图的重载。

```java
 create(@Nonnull String url, @Nonnull char[] token, @Nullable String org, @Nullable String bucket) {

```

但如果你用的是操作员 token，希望完成一些创建组织，删除用户的操作，那么就不应 该在创建连接时指定存储桶。此时，应该使用下图所示的重载。

```java
create(@Nonnull String url, @Nonnull char[] token) {
```

不过、检查 InfluxDB 的健康状态不需要任何权限和 token。此时，我们只需指定一个 URL，那就可以使用下图所示的重载了。

```java
create(@Nonnull String connectionString)
```

InfluxDBClient 对象就是我们的客户端对象。 InfluxDBClient 可以返回各种 Api 对象。 如下图所示。这体现了 java 对 InfluxDB HTTP API 的封装。

```java
InfluxDBClient client = InfluxDBClientFactory.create("http://192.168.42.104:8086/", token.toCharArray(), org, bucket);

```



### **11.4.3** 调用 **API**

一些简单的 api，也可以通过 InfluxDBClient 对象直接调用。比如我们的检查 InfluxDB健康状态，就可以直接调用 InfluxDBClient 对象的 ping 方法。如下所示

```java
System.out.println(influxDBClient.ping());
```

### **11.4.4** 运行

ping 方法会返回一个布尔值。如果 InfluxDB 可以 ping 通，那么就会得到 true，否则返回 flase，并记录一条失败日志。

### **11.4.5** 补充

在之前的版本，有一个测试 InfluxDB 是否健康的 API 叫做 health，不过现在这个接口 已经被标记为废弃。health 方法返回一个 HealthCheck 对象，相对而言，对这个对象的处理 比直接处理布尔值要麻烦很多。在以后的版本，提倡用 ping 方法检查健康状态。

## **11.5** 示例:查询 **InfluxDB** 中的数据

### **11.5.1** 创建一个 **JAVA** 类

### **11.5.2** 加入一个 **main** 方法

稍后 main 方法里面会写我们的查询逻辑。

### **11.5.3** 创建 **InfluxDB** 客户端对象

这次我们要操作 InfluxDB 中具体存储桶的数据，建立连接时，推荐选择图中的重载方法。

这个方法需要 4 个参数。

- url，InfluxDB 服务的 URL，
- token，授权的 token，而且类型还必须得是 char[ ]。
- org，指定要访问的组织。
- bucket，要访问的存储桶。

### **11.5.4** 获取查询 **API** 对象

使用 InfluxDBClient 对象点一下，可以看到 InfluxDBClient 其实提供了两种 API。这也 是为了兼容性来考虑的。InfluxQLQueryAPI 是 InfluxDB2.x 做的向前兼容。这里我们选择 第一个方法，也就是 getQueryApi，这意味着我们使用 v2 api 进行查询。

### **11.5.5** 了解查询 **API**

概括性地说，QueryApi 对象下有两个方法 query 和 queryRaw。

两个方法都需要传入一个 FLUX 脚本作为查询语句。但主要的不同点在于返回的结果 上。

- queryRaw 方法返回 API 中的 CSV 格式数据(String 类型)。
- query 方法视图将查询后的结果封装为各种对象(可以自己指定也可以使用 influxdb- client-java 提供的 FluxTable)。

两个方法各有很多不同的实现，其中一大部分是用来制定连接参数的，比如你创建连 接对象的时候没有制定 org 和 bucket，那么可以延迟到调用具体 api 的时候再指定。

### **11.5.6 query**

现在，我们先用 query 方法去查询一下 InfluxDB 中的数据`

```ini
from(bucket: "demo_java1")
  |> range(start: -1h)
  |> filter(fn: (r) => r["_measurement"] == "temperature")
  |> filter(fn: (r) => r["local"] == "ningbo")
  |> aggregateWindow(every: 10s, fn: mean, createEmpty: false)
  |> yield(name: "mean")
```

我们的查询结果 List\<FluxTable>其实对应了 FLUX 查询语言中的表流概念。 我们可以打印一下 query 变量。

我们可以打印一下 query 变量。

现在，我们只取第一个 FluxTable，看看里面有什么。

之前我们讲 FLUX 的时候，有讲过表流和 groupKey 之间的关系。

现在可以打印一下 groupKey，看看里面有什么东西。 代码如下:

```java
List<FluxTable> query = queryApi.query(flaux);
        System.out.println(query.size());
        for (FluxTable fluxTable : query) {
            List<FluxColumn> columns = fluxTable.getColumns();
            System.out.println(columns.size());
            for (FluxColumn column : columns) {
                System.out.println(column);
                System.out.println(column.getLabel());
            }

            System.out.println("---");
            List<FluxRecord> records = fluxTable.getRecords();

            for (FluxRecord record : records) {
                System.out.println(record.toString()+record.getTime().toString());
            }

            System.out.println(fluxTable);
        }
```

### **11.5.7 queryRaw**

唯一不同的地方就是把 queryApi.query 改为了 queryApi.queryRaw。 同时，query 变量的类型也从 List\<FluxTable>变成了 String

可以看到，我们打印出了CSV格式的数据，这是因为InfluxDB HTTP API本来在请求 体中放的就是 CSV 格式的数据。所以 QueryRaw 方法其实就是返回原始的 CSV。

```java
 String s = queryApi.queryRaw(flaux);
       System.out.println(s);
```



## **11.6** 同步写和异步写的区别

- 同步写，就是当我调用写入方法时，立刻向 InfluxDB 发起一个请求，将数据传送过去， 而且当前线程会一直阻塞等待写入操作完成。
- 异步写，其实是我调用写入方法的时候，先不执行写入这个操作，而是将数据放入一 个缓冲区。当缓冲区满了，我再真正地将数据发送给 InfluxDB，这样相当于实现了一个攒 批的效果。

在后面的示例中，我们会向建议先在 InfluxDB 中创建一个名为 example_java 的存储桶， 再学习后面的写入示例。

## **11.7** 示例:同步写入 **InfluxDB**

```java
public class Write1 {

    static String token = "uPOET3pZWTDB3X6PR0FYj4Fi24j_swy8zKFeOXpByijivezUI7gmghunvdVaOLn4UapN4dKoifqLrE-idVdT2A==";
    static String bucket = "demo_java1";
    static String org = "yme";


    public static void main(String[] args) {
//        test1();
//        test2();

        test3();
    }

    private static void test3() {
        WriteApiBlocking writeApiBlocking = getWriteApiBlocking();
        DemoPOJO demoPOJO = new DemoPOJO("temperature", "SZ", 4.0, Instant.now());
        writeApiBlocking.writeMeasurement(WritePrecision.MS,demoPOJO);
    }

    private static void test2() {
        InfluxDBClient client = InfluxDBClientFactory.create("http://192.168.42.104:8086/", token.toCharArray(), org, bucket);

        WriteApiBlocking writeApiBlocking = client.getWriteApiBlocking();
        Point temperature = Point.measurement("temperature");
        temperature.addField("value", 48.9).addTag("local", "beijing");

        String s = temperature.toLineProtocol();
        System.out.println(s);
        writeApiBlocking.writePoint(temperature);

        client.close();

    }


    private static void test1() {
        InfluxDBClient client = InfluxDBClientFactory.create("http://192.168.42.104:8086/", token.toCharArray(), org, bucket);
        System.out.println(client.ping());
        WriteApiBlocking writeApiBlocking = client.getWriteApiBlocking();
        writeApiBlocking.writeRecord(WritePrecision.MS, "temperature,local=shanghai value=37");
        client.close();
    }


    private static WriteApiBlocking getWriteApiBlocking() {
        InfluxDBClient client = InfluxDBClientFactory.create("http://192.168.42.104:8086/", token.toCharArray(), org, bucket);
        System.out.println(client.ping());
        return client.getWriteApiBlocking();
    }

}
```

### **11.7.2** 获取 **API** 对象

我们可以看到，InfluxDBClient 上有多种方法获取写操作 API 对象。其中WriteApiBlocking 是同步写入，WriteApi 是异步写入。

我们现在先使用 getWriteApiBlocking 方法获取同步写入的 API。

### **11.7.3** 有哪些写入方法

简单归纳，写入 API 给用户提供了 3 类方法写入数据。

- writeMeasurement，用户可以写入自己的 POJO 类
- writePoint，influxdb-client-java 提供了一个 Point 类，用户可以将一条条数据封装为一个个 Point，写入 InfluxDB。
- writeRecord，用户可以用符合 InfluxDB 行协议的字符串向 InfluxDB 写入数据。

另外，这三类方法都有与之对应的带 s 后缀的版本，表示可以一次写入多条。

### **11.7.4** 通过 **Point** 对象写入 **InfluxDB**

#### **11.7.4.1** 构建 **Point** 对象

```java
private static void test2() {
  InfluxDBClient client = InfluxDBClientFactory.create("http://192.168.42.104:8086/", token.toCharArray(), org, bucket);

  WriteApiBlocking writeApiBlocking = client.getWriteApiBlocking();
  Point temperature = Point.measurement("temperature");
  temperature.addField("value", 48.9).addTag("local", "beijing");

  String s = temperature.toLineProtocol();
  System.out.println(s);
  writeApiBlocking.writePoint(temperature);

  client.close();

}
```

使用下面的代码创建一个 point 对象。

这是典型的构造器设计模式，measurement 是一个静态方法，它会帮我们 new 一个 Point。addTag 和 addField 不再解释。最终的 time，我们通过第二个参数指定写入时间戳的 精度。这里是将写入的时间精度确定为了毫秒，如果你传入了一个纳秒时间戳，但精度指 明了毫秒，那超出毫秒的部分会被直接截断。

#### **11.7.4.2** 将 **point** 写出

使用下面的代码，直接将 point 写到 InfluxDB 中。记得在此之前创建 example_java 存储桶。

```java
  writeApiBlocking.writePoint(temperature);
```

#### **11.7.4.3** 验证写入结果

执行程序后，在 InfluxDB DataExplorer 查看 example_java 里面有没有新的数据，并将它展示出来。

### **11.7.5** 通过行协议写入 **InfluxDB**

```java
private static void test1() {
  InfluxDBClient client = InfluxDBClientFactory.create("http://192.168.42.104:8086/", token.toCharArray(), org, bucket);
  System.out.println(client.ping());
  WriteApiBlocking writeApiBlocking = client.getWriteApiBlocking();
  writeApiBlocking.writeRecord(WritePrecision.MS, "temperature,local=shanghai value=37");
  client.close();
}
```

此处我们在行协议中省略时间戳，让 InfluxDB 自动帮我们把时间补上。

#### **11.7.5.3** 验证写入结果

### **11.7.6** 通过 **POJO** 类写入 **InfluxDB**

#### **11.7.6.2** 添加一个静态内部类

```java
public class DemoPOJO {

    @Column(measurement = true)
    private String measurement;

    @Column(tag = true)
    private String local;

    @Column
    private Double value;

    @Column(timestamp = true)
    private Instant instant;


    public DemoPOJO(String measurement, String local, Double value, Instant instant) {
        this.measurement = measurement;
        this.local = local;
        this.value = value;
        this.instant = instant;
    }
  
}
```

```java
    private static void test3() {
        WriteApiBlocking writeApiBlocking = getWriteApiBlocking();
        DemoPOJO demoPOJO = new DemoPOJO("temperature", "SZ", 4.0, Instant.now());
        writeApiBlocking.writeMeasurement(WritePrecision.MS,demoPOJO);
    }
```



## **11.8** 示例:异步写入 **InfluxDB**

### **11.8.2** 获取 **API** 对象

现在鼓励使用的是 makeWriteApi 方法。实际上，在目前版本，getWriteApi 的内部实现已经是直接调用 makeWriteApi 了。

```java
public class AsyncWrite {

    static String token = "uPOET3pZWTDB3X6PR0FYj4Fi24j_swy8zKFeOXpByijivezUI7gmghunvdVaOLn4UapN4dKoifqLrE-idVdT2A==";
    static String bucket = "demo_java1";
    static String org = "yme";

    private static InfluxDBClient getWriteApiBlocking() {
        InfluxDBClient client = InfluxDBClientFactory.create("http://192.168.42.104:8086/", token.toCharArray(), org, bucket);
        System.out.println(client.ping());
        return client;
    }


    public static void main(String[] args) {
        test1();
    }

    private static void test1() {
        InfluxDBClient client = getWriteApiBlocking();
        WriteOptions build = WriteOptions.builder()
                .batchSize(999)
                .flushInterval(10_000) // 10s
                .build();

        WriteApi writeApi = client.makeWriteApi(build);
        writeApi.writeRecord(WritePrecision.MS, "temperature,local=ningbo value=41");


        for (int i = 0; i < 999; i++) {
            writeApi.writeRecord(WritePrecision.MS, "temperature,local=ningbo value=" + i + ".0");
        }





//        writeApi.flush();

//        client.close();

    }


}
```

### **11.8.3** 编写写入代码

和之前的同步写入一样，writeApi 对象也有

### **11.8.4** 验证写入结果(写入失败)

可以看到，这里没有出现我们刚才的数据，这表示我们刚才的写入失败了。但是，我 们的 java 程序没有报错。

这是因为 WriteApi 会使用一个守护线程，帮我们管理缓冲区，它会在缓冲区满或者距 离上次写出数据过 1 秒时将数据写出去。我们刚才就放了一条数据，缓冲区没满、write 方 法调用完程序就立刻退出了，所以后台线程压根就没有做写的操作。

### **11.8.5** 修改代码

现在，有两种方式让守护线程执行写的操作。

- 手动触发缓冲区刷写	
  - writeApi.flush();
- 关闭 InfluxDBClient
  - influxDBClient.close();

### **11.8.6** 验证写入结果(写入成功)

### **11.8.7** 小结:异步写入工作逻辑

- writeApi 里有一个缓冲区，这个缓冲区的大小默认是 10000 条数据。
- 虽然有缓冲区但是 writeApi 写出数据并不是一次把整个缓冲区都写出去，而是按照 批次(默认是 1000 条)的单位来写。
- 当产生被压或者写入失败时，守护线程会自动重试写入数据。

### **11.8.8** 异步写入的配置

- 异步攒批的操作的守护线程隐式进行的，好在它的行为我们可以进行具体的配置。
- influxdb-client-java 为我们提供了一个 WriteOption 对象，调用 makeWriteApi 时可以传 入这个对象，通过上图的提示我们可以看到，缓冲区的大小，批的大小，刷写的间隔我们 都是可以进行明确指定的。

### **11.8.9** 默认配置



### **11.9** 兼容 **V1 Api**

使用 InfluxDBClientFactory 创建 Client 对象时，调用 createV1 方法。





