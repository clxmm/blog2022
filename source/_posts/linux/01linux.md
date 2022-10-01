---
title: 01linux 的日志查询
toc: true
tags: linux
categories: 
    - [java]
    - [linux]
---


##  1.linux中如何搜索日志信息

<!--more-->

### 1.查询实时日志信息

register.log日志文件


```sh
tail -f register.log
```

### 2.通过关键字搜索日志信息

如：搜索日志文件中**data**关键字

```sh
grep 'data' register.log
```

### 3.搜索某个具体时间的日志信息

```sh
 grep 'data' register.log | grep '2022-09-25 13:33:49'
```

### 4查询上下文几行的信息

-C   2 ：查询上下两行的日志信息

```sh
grep 'data' register.log | grep '2022-09-25 13:33:49'
```

### 5查询前几行的日志信息

-B 2 ：查询前两行的日志信息

```sh
grep -B 2 'data' register.log | grep '2022-09-25 13:33:49'
```

### 6查询后几行的日志信息

-A 2: 查询后两行的日志信息

```sh
grep -A 2 'data' register.log | grep '2022-09-25 13:33:49'
```

如

```ini
2022-09-25 13:33:49.238+0000 [id=43]    INFO    h.m.DownloadService$Downloadable#load: Obtained the updated data file for hudson.tools.JDKInstaller
2022-09-25 13:33:49.239+0000 [id=43]    INFO    hudson.util.Retrier#start: Performed the action check updates server successfully at the attempt #1
2022-09-25 13:33:49.243+0000 [id=43]    INFO    hudson.model.AsyncPeriodicWork#lambda$doRun$1: Finished Download metadata. 25,461 ms
```

front-systemdoc-web  云文库前端
evaluation-survey-web 能源评价前端网页
grape-system-doc   云文库后端

