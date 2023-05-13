---
title: 06influxdb
toc: true
tags: influxdb
categories: 
    - [db]
    - [influxdb]
---

# 第**12**章 使用 **InfluxDB** 模板

InfluxDB 模板是一份 yaml 风格的配置文件。它包含了一套完整的仪表盘、Telegraf 配 置和报警配置。InfluxDB 模板力图保证开箱即用，把 yaml 文件下载下来，往 InfluxDB 里 一导，从数据采集一直到数据监控报警就全部为你创建好。

InfluxDB 官方在 github 上收录了一批模板。开发前可以在这里逛一逛，看有没有可以 直接拿来用的

[https://github.com/influxdata/community-templates#templates](https://github.com/influxdata/community-templates#templates)

<!--more-->

# 第**13**章 定时任务

## **13.1** 什么是定时任务

InfluxDB 任务是一个定时执行的 FLUX 脚本，它先查询数据，然后以某种方式进行修 改或聚合，然后将数据写回 InfluxDB 或执行其他操作。

## **13.2** 示例:将数据转成 **json** 发给别的应用

### **13.2.1** 创建任务的途径

有很多方式可以帮你创建任务，比如 DataExplorer、Notebook、HTTP API 和 influx-cli。 不过，此处我们为了还愿定时任务的本来面貌，我们会使用 influx-cli 去创建定时任务。

### **13.2.2** 本任务的需求

-  (1)每 30s 调度一次
- (2)查询最近 30s 的第一条数据
- (3)将数据转为 json
- (4)将 json 通过 HTTP 发送 SimpleHttpPostServer。

### **13.2.5** 在 **DataExplorer** 中编写 **FLUX** 脚本

我们首先在 DataExplorer 中把查询到转为 json 的这段逻辑写好。

我们要查询的是 test_init 存储桶下的 go_goroutines 测量。这个测量反应的是我们当前 InfluxDB 程序中的 goroutines(轻量级线程)数量。

打开 DataExplrer，编写如下代码



```ini
import "json"
import "http"

from(bucket: "demo_java1")
    |> range(start:-30s)
    |> filter(fn: (r) => r["_measurement"] == "temperature") 
    |> first(column: "_value")
    |> map(fn: (r) => { 
            code = http.post(url:"http://192.168.42.101:8080/test", data:json.encode(v:r))
                return {r with status_code:code}
        
    })

```



```ini
import "json"
import "http"



option task = { 
  name: "demo_30s",
  every: 5s,
}

from(bucket: "demo_java1")
    |> range(start:-30s)
    |> filter(fn: (r) => r["_measurement"] == "temperature") 
    |> first(column: "_value")
    |> map(fn: (r) => { 
            code = http.post(url:"http://192.168.42.101:8080/test", data:json.encode(v:r))
                return {r with status_code:code}
        
    })
```

**代码解释:**

- from -> range -> filter，指定了数据源并取出了我们想要的序列，其中 range 的 start 参数我们写死-30s。
- first 函数，Flux 查询 InfluxDB 返回的数据默认是按照时间从先到后排序的，first 函 数配合前面的查询相当于只取了最近 30s 的第一条数据。
- map 函数，我们在 map 函数里完成数据的发送。这里需要注意，因为 map 函数要求 必须返回 record，而且输出的 record 不能和输入的 record 一模一样。所以，结尾 return 的 时候，我们使用了 with 语法，给 map 输出的 record 增加了一个字段。也就是我们发出 http 请求响应的状态码。在 map 中的匿名函数，我们使用了 http.post 向 http://localhost:8080/发 送了一条 json 格式的数据。

### **13.2.6** 运行代码并观察效果

现在，我们点击 SUBMIT 按钮，执行这个 FLUX 查询脚本，观察 DataExplorer 返回的数据和 simpleHttpPostServer 的输出。

#### **13.2.6.1** 观察 **DataExplorer**

#### **13.2.6.2** 观察 **SimpleHttpPostServer**

截至目前，说明我们的 FLUX 脚本可以实现需求，现在的问题就是如何将这份脚本设 为定时任务。

### **13.2.7** 配置定时调度

现在，在我们的查询逻辑前面插入一行。

```ini
option task = { name: "example_task", every: 30s, offset:0m }
```

表示，我们做了一个设定，指定了一个名为 example_task 的任务。这个任务每隔 30 秒 执行一次。offset 这里暂时设为 0m，后面我们会专门讲这里的 offset 有什么意义。

### **13.2.8** 使用 **influx-cli** 创建任务

虽然我们在 DataExplorer 里写了一个 option，但是具体到 option 会不会生效，必须要 看FLUX脚本再跟InfluxDB的那个HTTP API进行交互。所以这里点击SUBMIT按钮只会 再执行一次查询，option task 并不会生效。这里，我们会先用 influx-cli 创建一遍任务。

先把 Flux 脚本复制出来，在/opt/modules/examples 里创建一个文件，就叫 example_task.flux 吧。将之前写的脚本粘贴进去。

```ini
import "json"
import "http"
option task = { name: "example_task", every: 30s, offset:0m }
from(bucket: "test_init")
|> range(start:-30s)
|> filter(fn: (r) => r["_measurement"] == "go_goroutines") |> first(column: "_value")
|> map(
fn: (r) => {
status_code = http.post(url:"http://localhost:8080/", data:
json.encode(v:r))
return {r with status_code:status_code}
} )
```

使用下面的命令，创建 FLUX 任务。

```ini
./influx task create --org atguigu -f
/opt/module/examples/example_task.flux
```

### **13.2.9** 在 **Web UI** 上查看定时任务

### **13.2.11** 使用 **DataExplorer** 创建任务

这一次，我们用 DataExplorer 来创建任务。现在，我们先将已经存在的任务删除。

打开 DataExplorer，编辑 FLUX 脚本，将我们之前写的查询脚本粘进去。注意，要删 除 option 一行。如下图所示:

完成上面的操作后，点击DataExplorer页面右上方的SAVE AS按钮，在弹出的对话框 中选择 TASK 选项卡。

- Name，填写为 example_task
- Every，填写 30s

- Offset 可以空着，这样默认就是 0。此处显示的 20m 是前端渲染效果，和具体的任 务执行无关
- 此处的 OutputBucket 的填写需要注意，本来我们自己编写的脚本并没有指定要把数 据回写到 InfluxDB，但是如果从 Web UI 创建定时任务的话，Output Bucket 不能不设，这 样会造成回写操作。这也是为什么我们之前不用 Web UI 创建任务的原因。

### **13.2.12** 再次查看任务详情(注意 **Web UI** 的小动作)

点击 EDIT TASK 按钮，查看我们的 FLUX 脚本。你会惊奇的发现，我们的 FLUX 代 码居然被修改了，原本不回写数据库的操作被强制加上了一个 to 函数。另外，可以看到我 们代码的前面也被加上了一个 option task 代码，这说明 Web UI 的页面点按操作只不过是帮 我们完成了手敲代码的几个步骤。

的来说，看开发者能否接收代码被隐式修改。如果这种行为无法接受，那么强烈建 议用 influx-cli 的方式去创建任务。

## **13.3** 数据迟到问题

option 中的 offset 是专门用来帮我们处理迟到问题的。

首先，我们来关注一个迟到的场景，如下图所示。我们的定时任务每次查询最近 30 秒 的数据。同时，调度的间隔设为每 30 秒执行一次。

![](./img/2023/04/influxdb06-1.png)

这个时候，由于网络的延迟，本该 1 分 20 秒入库的数据，1 分 32 秒的时候才来。但在 1 分 30 秒的时候，我们的查询已经执行完了。这个时候我们错过了 1 分 20 秒的数据。

这个时候，如果我们将 offset 设为 5 秒。如下图所示:

![](./img/2023/04/influxdb06-2.png)

定时任务的执行时间向后延迟了 5 秒，但是查询的还是原来范围的数据。这个时候定 时任务执行的时间是 1 分 35 秒。原先的迟到数据就能被我们查询到了。

## **13.4 cron** 表达式

## **13.5** 补充:**InfluxDB** 抓取任务的本质

之前我们在 Web UI 里设置的抓取任务，其背后其实就是定时执行的 FLUX 脚本。只不过 InfluxDB 在 API 上将他们分开了。

在当前的 FLUX 版本中，experimental/prometheus 库为我们提供了采集 prometheus 格式 数据的能力。详细可以参考 https://docs.influxdata.com/flux/v0.x/prometheus/scrape- prometheus/

# 第**14**章 **InfluxDB** 仪表盘

## **14.1** 什么是 **InfluxDB** 仪表盘

# 第**15**章 **InfluxDB** 服务进程参数(**influxd** 命令的用法)

## **15.1 influxd** 命令罗列

我们的 InfluxDB 下载好后，解压目录下的 influxd 就是我们 InfluxDB 服务进程的启动 命令。本课程不会介绍 influxd 的全部命令，通过下面的命令列表，大家可以窥探 InfluxDB 的一些可配置的能力。

详情可以参考:https://docs.influxdata.com/influxdb/v2.4/reference/cli/influx/

| 命令         |          |                                                              |
| ------------ | -------- | ------------------------------------------------------------ |
| downgrade    | 降级     | 将元数据格式降级以匹配旧的发行版                             |
| help         | 帮助     | 打印 influxd 命令的帮助信息                                  |
| inspect      | 检查     | 检查磁盘上数据库的数据                                       |
| print-config | 打印配置 | (此命令 2.4 已被废弃)打印完整的 influxd 在当前环境的配置信 息 |
| recovery     | 恢复     | 恢复对 InfluxDB 的操作权限，管理 token、组织和用户           |
| run          | 运行     | 运行 influxd 服务(默认)                                      |
| upgrade      | 升级     | 将 InfluxDB 从 1.x 升级到 InfluxDB2.4                        |
| version      | 版本     | 打印 InfluxDB 的当前版本                                     |

不一定必须通过 influxd 命令来查看 InfluxDB 的当前配置。你还可以使用 influx-cli 的 命令:

```ini
influx server-config
```

## **15.2 influxd** 的两个重要命令

在生产条件下最有可能用到的两个命令就是 inspect 和 recovery。下面，我们对这两个 命令做一下详细的介绍。

### **15.2.1 inspect** 命令

你可以使用下面的命令来查看 inspect 这个子命令的帮助信息

```ini
./influxd inspect -h
```

```ini
Commands for inspecting on-disk database data

Usage:
  influxd inspect [flags]
  influxd inspect [command]

Available Commands:
  build-tsi         Rebuilds the TSI index and (where necessary) the Series File.
  check-schema      Check for conflicts in the types between shards
  delete-tsm        Deletes a measurement from a raw tsm file.
  dump-tsi          Dumps low-level details about tsi1 files.
  dump-tsm          Dumps low-level details about tsm1 files
  dump-wal          Dumps TSM data from WAL files
  export-index      Exports TSI index data
  export-lp         Export TSM data as line protocol
  merge-schema      Merge a set of schema files from the check-schema command
  report-db         Estimates cloud 2 cardinality for a database
  report-tsi        Reports the cardinality of TSI files
  report-tsm        Run TSM report
  verify-seriesfile Verifies the integrity of series files.
  verify-tombstone  Verify the integrity of tombstone files
  verify-tsm        Verifies the integrity of TSM files
  verify-wal        Check for WAL corruption

Flags:
  -h, --help   help for inspect

Use "influxd inspect [command] --help" for more information about a command.

```

你会发现 inspect 这个子命令下还有很多子命令。

这里出现的 tsi、tsm、wal 都跟 InfluxDB 底层的存储引擎相关，本课程并不涉及这一 部分的内容。这里可以稍微点一下，你可以使用下面的命令查看 InfluxDB 中数据存储的大概情况

```ini
./influd inspect report-tsm
```

展示出来的信息中包含了 InfluxDB 的数据存储情况，比如当前整个 InfluxDB 有多少 序列，每个存储桶中又有多少序列等等。

另外，还有一个比较重要的 export-tsm 命令，它可以将某个存储桶中的数据全部导出 为 InfluxDB 行协议。后面我们会在一个示例中详细演示它的使用。

### **15.2.2 recovery** 命令

recovery 是恢复的意思。

可以先用下面的命令查看 recovery 这一子命令的帮助信息。

```ini
./influd recovery -h
```

```ini
root@c8b7d25d31f2:/# influxd recovery -h
Commands used to recover / regenerate operator access to the DB

Usage:
  influxd recovery [flags]
  influxd recovery [command]

Available Commands:
  auth        On-disk authorization management commands, for recovery
  org         On-disk organization management commands, for recovery
  user        On-disk user management commands, for recovery

Flags:
  -h, --help   help for recovery

Use "influxd recovery [command] --help" for more information about a command.

```

recovery 下面还有 3 个子命令，分别是 auth、org 和 user。它们分别与 token、组织和用 户有关。

下面主要是讲解 auth 子命令的用法，使用下面的命令可以进一步查看 auth 子命令的帮

助信息

```ini
./influxd recovery auth -h
```

```ini
root@c8b7d25d31f2:/# influxd recovery auth -h
On-disk authorization management commands, for recovery

Usage:
  influxd recovery auth [flags]
  influxd recovery auth [command]

Available Commands:
  create-operator Create new operator token for a user
  list            List authorizations

Flags:
  -h, --help   help for auth

Use "influxd recovery auth [command] --help" for more information about a command.

```

- create-operator:为一个用户创建一个新的操作者 token。
- list:列出当前数据库中的全部 token。

使用下面的命令就可以为 tony 用户再次创建一个 operator-token 了。

```ini
./influxd recovery auth create-operator --username tony --org atguigu
 
```

## **15.3 influxd** 常用配置项

influxd 的可用配置项超多，本课程不会全部讲解。

详细可以参考:https://docs.influxdata.com/influxdb/v2.4/reference/config-options/#assets- path

以下是一些常用的参数

- bolt-path:BoltDB 文件的路径。
- engine-path:InfluxDB 文件的路径
- qlit-path:sqlite 的路径，InfluxDB 里面还用到了 sqllite，它里面会存放一些关于任务执行的元数据，
- flux-log-enabled:是否开启日志，默认是 false。
- log-level:日志级别，支持 debug、info、error 等。默认是 info。

## **15.4** 如何对 **influxd** 进行配置

有 3 种方式可以对 influxd 的配置。这里以 http-bind-address 进行操作，为大家演示。

### **15.4.1** 命令行参数

进行如下操作前，记得关闭当前正在运行的 influxd。你可以使用下面的命令来杀死当 然的 influxd 进程。否则，原先的 influxd 进程会锁住 BoltDB 数据库，别的进程不能访问。 当然你也可以修改 BlotDB 路径，但是那样太过麻烦。

```ini
ps -ef | grep influxd | grep -v grep | awk '{print $2}' | xargs kill
 
```

用户 influxd 命令启动 InfluxDB 时，通过命令行参数来传递一个配置项。

比如

```ini
./influxd --http-bind-address=:8088
```

可以尝试访问 8088 端口，看服务有没有挂到端口上

### **15.4.2** 环境变量

同样，还是先杀死之前的 influxd 进程。

运行下面的命令。

```ini
ps -ef | grep influxd | grep -v grep | awk '{print $2}' | xargs kill
```

用户可以声明一个环境变量，对 influxd 进行配置

比如:

```ini
export INFLUXD_HTTP_BIND_ADDRESS=:8089
```

现在，我们启动一下 influxd 看下效果。

最后，因为我们用的是 export 命令，临时搞了一个环境变量，如果你觉得当前 shell 会 话不重要，可以关闭当前 shell 会话。否则，你可以使用 unset 命令来销毁这个环境变量。

```ini
unset INFLUXD_HTTP_BIND_ADDRESS
```

### **15.4.3** 配置文件

你还 nfluxd 所在的目录下放一个 config 文件，它可以是 config.json，config.toml， config.yaml。这 3 种格式 influxd 都能识别，不过文件中的内容一定要合法。influxd 启动时 会自动检测这个文件。

在 InfluxDB 的安装目录下创建一个 config.json 文件。

```ini
vim /opt/module/influxdb2_linux_amd64/config.json
```

编辑如下内容。

```json
{
"http-bind-address": ":9090" 
}
```

启动之前记得停掉之前的 InfluxDB 进程。

```ini
ps -ef | grep influxd | grep -v grep | awk '{print $2}' | xargs kill
 
```

现在再启动一下，看看效果。

可以看到端口已经变成 9090。配置同样是生效的。

### **15.4.4** 小结

最后，如果要做配置的修改，建议一定要参考 InfluxDB 的官方文档，这一部分写的非 常清楚，而且官网已经给出了进行配置的各种模板。用好官方文档，可以大大提高开发效 率.

# 第**16**章 时序数据库是怎么存储用户名和密码的

InfluxDB 内部自带了一个用 Go 语言写的 BlotDB，BlotDB 是一个键值数据库，它的功 能比较有限，基本上就是专注于存值、读值。同时，因为功能有限，它也可以做的很小很 轻量。

InfluxDB 就是把用户名、密码、token 什么的信息存在这样的键值数据库里的。默认情 况下，BlotDB 的数据会存储在一个单独的文件中，这个文件会在~/.influxdbv2/ 路径下，名 称为 influxd.bolt。

这个文件的路径可以在 influxd 通过 bolt-path 配置项来进行修改。







