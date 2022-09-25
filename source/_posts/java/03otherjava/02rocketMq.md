---
title: 02消息队列RocketMQ
toc: true
tags: RocketMq
categories: 
    - [java]
    - [RocketMq]
---

##   1**RocketMQ**工作原理

<!--more-->

### 一、消息的生产

#### **1** 消息的生产过程

Producer可以将消息写入到某Broker中的某Queue中，其经历了如下过程:

- Producer发送消息之前，会先向NameServer发出获取 消息Topic的路由信息 的请求
- NameServer返回该Topic的 路由表 及 Broker列表
- Producer根据代码中指定的Queue选择策略，从Queue列表中选出一个队列，用于后续存储消息
- Produer对消息做一些特殊处理，例如，消息本身超过4M，则会对其进行压缩
- Producer向选择出的Queue所在的Broker发出RPC请求，将消息发送到选择出的Queue

#### **2 Queue**选择算法

对于无序消息，其Queue选择算法，也称为消息投递算法，常见的有两种:

**轮询算法**

默认选择算法。该算法保证了每个Queue中可以均匀的获取到消息。

**最小投递延迟算法**

该算法会统计每次消息投递的时间延迟，然后根据统计出的结果将消息投递到时间延迟最小的Queue。如果延迟相同，则采用轮询算法投递。该算法可以有效提升消息的投递性能。

### 二、消息的存储

RocketMQ中的消息存储在本地文件系统中，这些相关文件默认在当前用户主目录下的store目录中。

- abort:该文件在Broker启动后会自动创建，正常关闭Broker，该文件会自动消失。若在没有启动Broker的情况下，发现这个文件是存在的，则说明之前Broker的关闭是非正常关闭。
- checkpoint:其中存储着commitlog、consumequeue、index文件的最后刷盘时间戳
- commitlog:其中存放着commitlog文件，而消息是写在commitlog文件中的
- config:存放着Broker运行期间的一些配置数据
- consumequeue:其中存放着consumequeue文件，队列就存放在这个目录中
- index:其中存放着消息索引文件indexFile
- lock:运行期间使用到的全局资源锁

**1 commitlog**文件

目录与文件

commitlog目录中存放着很多的mappedFile文件，当前Broker中的所有消息都是落盘到这些mappedFile文件中的。mappedFile文件大小为1G(小于等于1G)，文件名由20位十进制数构成，表示当前文件的第一条消息的起始位移偏移量。

需要注意的是，一个Broker中仅包含一个commitlog目录，所有的mappedFile文件都是存放在该目录中的。也就是说，这些消息在Broker中存放时并没有被按照Topic进行分类存放。

