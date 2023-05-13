---
title: 03influx
toc: true
tags: influxdb
categories: 
    - [db]
    - [influxdb]
---

# 第**6**章 如何使用 **FLUX** 语言的文档

## **6.1** 如何查看函数文档

这是 FLUX 语言的文档 https://docs.influxdata.com/flux/v0.x/ ，通常来说我们使用

FLUX 的文档主要是用它来查看一些函数怎么用，如图所示:

<!--more-->

![](./img/2023/04/influx03-1.png)

点击 Standard libaray，就可以看到 FLUX 的所有函数包了。

![](./img/2023/04/influx03-2.png)

点击一个包的左侧的+按钮，就可以看到这个包里的所有函数，任意点击其中一个， 就可以看到这个函数的详细说明，包括会返回什么，调用的时候需要传递什么参数等等。

## **6.2** 避免使用实验中的函数

另外，需要额外注意有一个函数库的名字叫 experimental，这个单词是实验的意思，也就是在未来的 FLUX 版本中，这个函数有可能会变，参数名可能也不是很确定，甚至这个函数可能会在未来的某个版本被放弃。

如果你有升级的打算，那么 experimental 里面的函数应该敬而远之，否则在未来的某 个时间，很有可能会导致重复开发。

## **6.3** 查看函数可以在哪些版本中使用

另外需要注意，每个函数的文档标题正下方都会标记这个函数是从哪个 FLUX 版本开始加入的。比如从下图我们就可以知道 request.do()函数是从 0.173 之后才能用的。

![](./img/2023/04/influx03-3.png)

下面这张图告诉我们 array.concat()函数从 0.173 版本之后就不能再用了。

# 第**7**章 **FLUX** 查询 **InfluxDB**

```ini
car,code=01 rate=20
```



## **7.2 FLUX** 查询 **InfluxDB** 的语法

使用 FLUX 语言查询 InfluxDB，必须以 from -> range 打头。

```ini
from(bucket:"demo-query")
    |> range(start: -1h)
```

range 必须紧跟在 from 后面，不这么写的话会直接报错。

## **7.3** 表、表流以及序列

我们知道 InfluxDB 是使用序列的方式去管理数据的。而 FLUX 语言又企图兼容一些关系型数据库的查询，而关系型数据库里的数据结构就是一个有行有列的 table。因此对于FLUX 语言来说，就需要将序列和表统一成一个东西。

所以 FLUX 引入了表流的概念。

简单来说，FLUX 可以一次性查出多个序列，这个时候一个序列对应一张表，而表流 其实就是多张表的集合。同时表流和表的关系其实是全表和子表的关系，子表是全表按照 _field，tag_set和_measurement进行group by之后的结果。在这种情况下，如果调用聚合函 数，其实只会在子表中进行聚合

![](./img/2023/04/influx03-4.png)

最后，如果一张表对应的是一个序列了，那么一张表里的一行其实就对应着序列中的 一个数据点了。

```ini
from(bucket:"demo-query")
    |> range(start: -2d)
    |> max()
```

![](./img/2023/04/influx03-5.png)

## **7.4 filter** 维度过滤

使用 filert 函数可以对数据按照_measurement、标签集和字段级进行过滤，后面的课程会给大家讲解 filter 的性能问题。

```ini
from(bucket:"demo-query")
    |> range(start: -1h)
    |> filter(fn:(r) => (r["code"]=="02"))
```

![](./img/2023/04/influx03-6.png)

## **7.5** 类型转换函数与下划线字段

Flux 语言中有很多不用指定字段名称的管道函数，比如 toInt()。其实 toInt()这个函数 默认要求你的字段中必须要有_value 字段，没有_value 字段的话也会直接报错。

其实在我们查询出来的数据中，以下划线开头的字段其实代表了一种约定，就是 FLUX 中有很多函数想要正常运行时要依赖于这些下划线打头的字段的。

所以原则上来说，程序员应该遵守这些约定，不要擅自更改下划线开头的字段。

## 7.5表分裂的问题

```ini
test field1="你好啊",fileld2=10,filed3=11i
```

```ini
from(bucket:"demo-query")
    |> range(start: -1h)
    |> filter(fn:(r) => (r["_measurement"]=="test"))
```

![](./img/2023/04/influx03-7.png)

```ini
from(bucket:"demo-query")
    |> range(start: -1h)
    |> filter(fn:(r) => (r["_measurement"]=="test"))
    |> filter(fn: (r) => r["_field"] == "fileld2")

```

![](./img/2023/04/influx03-8.png)

## tiInt函数

```ini
from(bucket:"demo-query")
    |> range(start: -1h)
    |> filter(fn:(r) => (r["_measurement"]=="test"))
    |> filter(fn: (r) => r["_field"] == "fileld2")
    |> toInt()

```



![](./img/2023/04/influx03-9.png)



## **7.6 map** 函数

map 函数的作用是遍历表流中的每一条数据。

```ini
import "array"
array.from(rows: [{"name":"tony"},{"name":"jack"}]) 
|> map(fn: (r)=> {
	return if r["name"] == "tony" then {"_name": "tony 不是jack"} else {"_name":"jack 不是 tony"} })
```

这里需要注意，map 函数需要我们传递一个参数 fn，它要求传递一个单个参数输入， 且输出必须是 record 的函数，其中输入数据的类型会是 record。

```ini
import "array"
array.from(rows: [{"name":"tony"},{"name":"jack"}])
    |> map(fn: (r) => ({"name":"tony2"}))
```





## **7.7** 自定义管道函数

此处，我们定义一个管道函数，它可以将表流中的_value 字段的值乘上 x 倍。请同学们在接下来的示例中注意声明管道函数时所用的语法。

```ini
big100 = (table=<-,x) => {
   return table
 }
```

接下来我们调用刚才声明的函数，最终整个脚本如下:

```ini
big100 = (table=<-,x) => {
   return table
}
|> map(fn: (r) => ({r with "_value":r["_value"]*x}))
from(bucket: "test_init")
|> range(start: -1h)
|> filter(fn: (r) => r["_measurement"] == "go_goroutines") |> big100(x:100)
```

可以自行运行查看函数效果。

这里需要强调的是，管道函数的第一个参数必须写成 table=<-，它表示通过管道符输 入进来的表流数据，需要注意，table 并不一定写成 table 但是=<-的格式绝对不能变。

## **7.8** 在文档中区分管道函数和普通函数

![](./img/2023/04/influx03-10.png)

当我们看到一个函数文档，它会有一个区域叫做 Function type Signature(函数签名)， 它表示着函数接收哪些参数以及会返回什么。最前面的小括号里的内容就是参数列表，如 果参数列表的第一个参数是<-tables: stream[A]，那就表示它是一个可以接收表流输入的管 道函数。

反之，如果没有<-tables: stream[A]，那么它就是一个普通函数。

## **7.9 window** 和 **aggregateWindow** 函数

window 函数和 aggregateWindow 函数其实代表着 InfluxDB 中的两种开窗方式，两者不 同的地方在于，window 函数会将整个表流重新分组。window 开窗后，是按照序列+窗口的 方式对整个表流进行分组。但是 aggregateWindow 函数会保留原来的分组方式，这样一来， 使用 aggregateWindow 函数进行开窗后的表流，仍然是按照序列的方式来分组的。

## **7.10 yield** 和 **join*

当 flux 脚本中出现未被赋值给某个变量的表流时，InfluxDB 执行 FLUX 脚本时会自动 帮他们在管道的最后面加上|> yield(name: "_result")函数，yield 函数其实是指定了我们当前 这个表流是整个 FLUX 脚本最后返回的结果，而且这个结果的名字叫"_result"。当 FLUX 脚本中出现多个为赋值给变量的表流时，给多个表流自动补上|>yield(name:"_result")就会出 问题了，这是因为当有多个表流后面都有|>yield 时，其实相当于一个 FLUX 脚本会返回多 个结果。但是此处要求名称是不能重复的，所以当有多个未赋值的表流时，就必须显示指 定 yield(name:"xxx")，而且名称千万不可重复。

但是，在一个 FLUX 脚本里同时返回多个结果集并不是推荐的操作，这通常会让程序的逻辑变的很奇怪，我们之所以能在一个 FLUX 脚本里面写多次 from 函数，其实是为了方便我们进行 join 的。

再但是，老师并不建议在 FLUX 脚本中使用 join 操作，这必须要谈到 FLUX 脚本的常

见使用场景，就是每隔一段时间进行一次查询。如果这个时候，我用一个 from 从 InfluxDB 中查询数据，其中有 code=01 等机器编号信息。然后我再用一个 from 去查询 mysql，得到 一张机器的属性表。接下来对两张表进行 join，这在逻辑上很合理，但最大的问题就是 FLUX 脚本无法实现数据的缓存。如果我这个 FLUX 脚本是每 15 秒执行一次，那就会导致 我们需要每 15 秒要去 mysql 上全表扫描一遍机器信息表，效率十分低下。

个人建议仅使用 FLUX 进行简单的查询，然后在应用层的程序里进行 join 操作。因此， 本课程并不讲解 FLUX 语言的 join 操作。







