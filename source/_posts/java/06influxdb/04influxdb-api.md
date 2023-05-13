---
title: 04influxdb-api
toc: true
tags: influxdb
categories: 
    - [db]
    - [influxdb]
---

# 第**8**章 前言:如何与 **InfluxDB** 交互

InfluxDB启动后，会向外提供一套HTTP API。外部程序可以也仅能通过HTTP API与 InfluxDB 进行通信。我们后面要讲到的 influx 命令行、Web UI 和各编程语言的客户端库， 其内部都是封装的对 HTTP API 的调用。

<!--more-->

![](./img/2023/04/influxdb04-1.png)

所以各种客户端同 InfluxDB 交互时，都离不开 API TOKEN。因为 HTTP 是一种支持 官方且简单的协议，这也方便了用户进行二次开发。

# 第**9**章 **InfluxDB HTTP API**

InfluxDB 提供了丰富的 API 和客户端库，可以随时和你的应用程序集成。你也可以随 时使用 curl 和 ApiPost、Postman 这类程序来测试 API 接口。

本课程会先带大家看一些最常用的 API，然后再告诉大家如何使用 API 文档。但本课 程不会对 InfluxDB 的全部 API 进行讲解。

## **9.1** 准备 **token**

在你想尝试使用 HTTP API 与 InfluxDB 进行交互时，首先应该用账号和密码登到 Web UI 上选择或创建一个对应的 API TOKEN。课程中，我们使用 tony's Token，这是一个具有 全部权限的 API Token，实际开发时应谨慎使用，防止 Token 被劫持出现安全问题。

```
L4-eoM8b63_ZedLhMAQgLEp6qHYx_lnVRlHh0oyq7eKG6Eg5DI3w5c9kXPSDmV9T_25yM-AjU1KA8Hxt74L1Mg==
```

![](./img/2023/04/influxdb04-2.png)

在后面的操作中，你每次发出 HTTP 请求时都需要在请求头上携带 token。

## **9.2** 准备接口测试工具

在 shell 中你可以使用 curl 测试接口，不过带图形界面的程序终归是更易用一些。

本课程选用 ApiPost 这一专门的接口测试软件进行演示。ApiPost 是一款国产软件，对 标的是 google 的 postman，截至视频课录制时，ApiPost 的最新版本是 6，易用性比上一个 版本有大幅提升，用起来很顺手。

### **9.2.1** 安装 **ApiPost**

### **9.2.2** 准备调试环境

## **9.3** 接口的授权

### **9.3.1 Token** 授权方式



# 第**10**章 使用 **influx** 命令行工具

从 InfluxDB 2.1 版本之后，influx 命令行和 InfluxDB 数据库服务程序是分开打包的， 所以安装 InfluxDB 并不会附带 influx 命令行工具。用户必须单独安装 influx 命令行工具。

influx 命令行工具包含很多管理 influxDB 的命令。包括存储桶、组织、用户、任务等。

从 2.1 版本之后，安装 InfluxDB 不会附带 influx 命令行工具，现在 influx 工具和 InfluxDB 在源码上也已经分开维护了，下载时需要注意对上版本。

## **10.1** 安装 **influx** 命令行工具

这次，我们另辟蹊径，不看着官方文档安装了，改从 github 上下载安装。

### **10.1.1** 如何去找开源项目的发行版

#### **10.1.1.1** 什么是发行版

这一部分的内容可以详细参考 Gitee 官方文档 https://gitee.com/help/articles/4328#article-header0

所谓发行，就是这个开源项目进行到一定程度，各种特性和功能已经趋于完善和稳定， 到了可以出一个阶段性版本的时候了。

通常来说，github 或者 gitee 上放的是一个项目的源码，但是源码需要经过编译之后才 能运行的，那么当作者觉得自己的项目，目前开发进度差不多，应该没什么坑的时候，他 就可以自己创建一个发行版。这个时候，作者需要自己上传一些附件，比如 v1.0.0 的编译

规范的发行信息里面应该还有比如 changelog(修改记录)这些信息，告诉用户，这个版本相比上个版本，增加了哪些新的功能，又修复了哪些 bug。

#### **10.1.1.2** 如何去找一个项目的发行版

首先，你可以去访问官网，通常来说一个开源项目通常应该有它自己的官网，在它的 官网上，应该可以找到它的历史版本。但是，有些官网就是新版发布了之后就下架旧版的 下载资源，比如 InfluxDB 就是这么干的

另外，通常开源项目都会在 github 或者 gitee 上去维护一个版本的时间线。

打开你关注的开源项目首页，如图所示是 InfluxDB 的项目首页。

点击右下角的 Release。

可以看到这个框架从盘古开天辟地至今的所有发行版。通常，在一个版本记录的最下方，会有这个版本对应的已编译好的可执行程序和源码。你可以把它下载下来使用。

### **10.1.2** 去找 **influx** 命令行工具的开源项目

找到 influx-cli 项目，打开之。https://github.com/influxdata/influx-cli

### **10.1.3** 下载安装发行版

点击 Releases 链接，看到最新的版本。

**下载到 /opt/software/**

**解压到 /opt/module/**

```ini
tar -zxvf influxdb2-client-2.4.0-linux-amd64.tar.gz -C /opt/module/
```

## **10.2** 配置 **influx-cli**

### **10.2.1** 创建配置

influx 命令行工具是你每执行一次操作时，调用一次命令。并不是开启一个持续性的 会话。而 influx 其实底层还是封装的对 InfluxDB 的服务进程的 http 请求。也就是它还是需 要配置 Token 什么的来获取授权。

所以，为了避免以后每次请求的时候都在命令行里面写一遍 token。我们应该先去搞个 配置文件。

使用下面的命令可以创 influx 命令行的配置。

[[Install and use the influx CLI | InfluxDB OSS 2.7 Documentation (influxdata.com)](https://docs.influxdata.com/influxdb/v2.7/tools/influx-cli/#Copyright)](https://docs.influxdata.com/influxdb/v2.7/tools/influx-cli/#Copyright)

```ini
./influx config create --config-name influx.conf \
  --host-url http://localhost:8086 \
  --org yme \
  --token uPOET3pZWTDB3X6PR0FYj4Fi24j_swy8zKFeOXpByijivezUI7gmghunvdVaOLn4UapN4dKoifqLrE-idVdT2A== \
  --active

```

```ini
 ./influx bucket list
 
```

这个命令其实会在~/.influxdbv2/目录下创建一个 configs 文件，这个文件中，就是我们命令行中写的各项配置。如图所示

![](./img/2023/04/influxdb04-3.png)

### **10.2.2** 更改配置

如果你中途配置错误了，再使用上文的命令，它会说这个配置已经存在。

也就是说，在 /home/dengziqi/.influxdbv2/configs 文件中，["name"]配置快不能重复必须 全局唯一。

这个时候如果你想调整配置，应该把 create 换成 update。 也就是

```ini
./influx config update --config-name influx.conf xxxxxxxx
```

### **10.2.3** 在多份配置之间切换

我们现在用下面的命令再创建一个配置，直接复制 influx.conf 中的内容，把名字修改成 influx2.conf

```ini
./influx config create --config-name influx2.conf \
--host-url http://localhost:8086 \
--org atguigu \
--token ZA8uWTSRFflhKhFvNW4TcZwwvd2NHFW1YIVlcj9Am5iJ4ueHawWh49_jszoKybEym HqgR5mAWg4XMv4tb9TP3w== \
--active
```

命令成功执行后，再次打开 ~/.influxdbv2/configs 文件。

可以看到 configs 中的文件内容变了，多了一个名为["influx2.conf"]的配置块，而且， 旧的["influx.conf"]从 active="true"变成了 previous="true"，同时["influx2.conf"]中有一个 active="true"的键值对。说明，如果现在使用 influx-cli 执行操作，那会直接使用 influx2.conf 配置块中的内容。

 你还可以使用下面的命令切换当前正在使用的配置。

![](./img/2023/04/influxdb04-4.png)

再次查看 ~/.influxdbv2/configs 文件

![](./img/2023/04/influxdb04-5.png)

### **10.2.4** 删除一个配置

influx2.conf 现在对我们来说是多余的了，现在，我们将它删除掉。

使用下面的命令删除 influx2.conf。

```ini
./influx config remove influx2.conf
```

执行后，再次查看~/.influxdbv2/config 文件

可以看到，["influx2.conf"]消失了。而且，我们的 influx.conf 自动变成了 active=true。

## **10.3 influx-cli** 命令罗列

我们已经知道influx-cli背后封装的是对InfluxDB HTTP API的请求。那么influx-cli有 多少功能基本上就取决于它封装了多少命令，本课程不会介绍 influx-cli 的全部功能。通过 下表，同学们可以一探 influx-cli 的功能。

[https://docs.influxdata.com/influxdb/v2.4/reference/cli/influx/](https://docs.influxdata.com/influxdb/v2.4/reference/cli/influx/)

| 命令          | 直译      | 解释                                                         |
| ------------- | --------- | ------------------------------------------------------------ |
| apply         | 应用      | 应用一个 InfluxDB 模板                                       |
| auth          | 认证      | 管理 API Token 的相关                                        |
| Backup        | 备份      | 备份数据(只支持 InfluxDB OSS)                                |
| bucket        | 桶        | 管理存储桶的命令                                             |
| bucket-schema | 桶模式    | 管理存储桶模式的命令                                         |
| completion    | 完成      | 生成完成的脚本                                               |
| config        | 配置      | 管理配置文件                                                 |
| dashboards    | 仪表盘    | 列出所有的仪表盘                                             |
| delete        | 删除      | 在 InfluxDB 中删除数据点。                                   |
| export        | 导出      |                                                              |
| help          |           |                                                              |
| org           | 组织      |                                                              |
| ping          |           |                                                              |
| query         | 查询      | 执行一个 FLUX 脚本的查询                                     |
| restore       | 恢复      | 从备份出来的数据恢复(只有 InfluxOSS 支持)                    |
| scripts       | 脚本      | 管理 InfluxDB 上的交互本(只有 InfluxDB Cloud 支持)           |
| secret        | 机密      | 管理机密                                                     |
| setup         | 设置      | 命令行版本的初始化操作，首次安装 InfluxDB 时的设置用户 名、密码、组织、存储桶等等。 |
| stacks        | 堆栈      | 你可以将堆栈理解为一个 InfluxDB 模板的实例。你可以使用 stacks 管理这些实例。 |
| task          | 任务      |                                                              |
| telegrafs     | telegrafs |                                                              |
| template      | 模板      | 总结和验证 InfluxDB 模板                                     |
| user          | 用户      | 管理用户相关的命令                                           |
| v1            | 版本1     | 使用 InfluxDB V1 兼容的 API                                  |
| version       |           |                                                              |
| write         | 写        | 向 InfluxDB 写数据                                           |



