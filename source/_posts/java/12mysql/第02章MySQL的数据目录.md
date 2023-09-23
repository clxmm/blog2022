---
title: 第02章MySQL的数据目录
toc: true
tags: mysql
categories: 
    - [db]
    - [mysql]
---

#### **1. MySQL8的主要目录结构**

```sh
find / -name mysql
```

<!--more-->

##### **1.1** **数据库文件的存放路径**

```sql
show variables like 'datadir'; # /var/lib/mysql/
```

##### **1.2** **相关命令目录**

**相关命令目录：/usr/bin 和/usr/sbin。**

#####  **1.3** **配置文件目录**

**配置文件目录：/usr/share/mysql-8.0（命令及配置文件），/etc/mysql（如my.cnf）**

#### **2.** **数据库和文件系统的关系**

##### **2.1** **表在文件系统中的表示**

##### 2.2 数据库在文件系统中的表示

看一下我的计算机上的数据目录下的内容:

```shell
cd /var/lib/mysql  
ll
```

这个数据目录下的文件和子目录比较多，除了 information_schema 这个系统数据库外，其他的数据库 在 数据目录 下都有对应的子目录。

以我的 temp 数据库为例，在MySQL5.7 中打开:

```shell
 cd ./temp 
 ll
总用量 1144
-rw-r-----. 1 mysql mysql -rw-r-----. 1 mysql mysql -rw-r-----. 1 mysql mysql -rw-r-----. 1 mysql mysql -rw-r-----. 1 mysql mysql -rw-r-----. 1 mysql mysql -rw-r-----. 1 mysql mysql -rw-r-----. 1 mysql mysql -rw-r-----. 1 mysql mysql -rw-r-----. 1 mysql mysql -rw-r-----. 1 mysql mysql -rw-r-----. 1 mysql mysql -rw-r-----. 1 mysql mysql -rw-r-----. 1 mysql mysql -rw-r-----. 1 mysql mysql -rw-r-----. 1 mysql mysql -rw-r-----. 1 mysql mysql -rw-r-----. 1 mysql mysql
8658 8月 18 11:32 countries.frm 114688 8月 18 11:32 countries.ibd
61 8月 18 11:32 db.opt
8716 8月 18 11:32 departments.frm
147456 8月 18 11:32 departments.ibd
3017 8月 18 11:32 emp_details_view.frm 8982 8月 18 11:32 employees.frm
180224 8月 18 11:32 employees.ibd 8660 8月 18 11:32 job_grades.frm 98304 8月 18 11:32 job_grades.ibd
8736 8月 18 11:32 job_history.frm 147456 8月 18 11:32 job_history.ibd
8688 8月 18 11:32 jobs.frm 114688 8月 18 11:32 jobs.ibd
8790 8月 18 11:32 locations.frm 131072 8月 18 11:32 locations.ibd
8614 8月 18 11:32 regions.frm 114688 8月 18 11:32 regions.ibd
```

在MySQL8.0中打开:

```shell
[root@atguigu01 mysql]# cd ./temp [root@atguigu01 temp]# ll
总用量 1080
-rw-r-----. 1 mysql mysql 
-rw-r-----. 1 mysql mysql 
-rw-r-----. 1 mysql mysql 
-rw-r-----. 1 mysql mysql 
-rw-r-----. 1 mysql mysql 
-rw-r-----. 1 mysql mysql 
-rw-r-----. 1 mysql mysql 
-rw-r-----. 1 mysql mysql
131072 7月 29 23:10 countries.ibd 163840 7月 29 23:10 departments.ibd 196608 7月 29 23:10 employees.ibd 114688 7月 29 23:10 job_grades.ibd 163840 7月 29 23:10 job_history.ibd 131072 7月 29 23:10 jobs.ibd
147456 7月 29 23:10 locations.ibd 131072 7月 29 23:10 regions.ibd
```



## 2.3 表在文件系统中的表示

###  **2.3.1 InnoDB存储引擎模式**

**1.** **表结构**

为了保存表结构，`InnoDB`在`数据目录`下对应的数据库子目录下创建了一个专门用于`描述表结构的文件`

```sh
表名.frm
```

````sql
mysql> USE atguigu;
Database changed
mysql> CREATE TABLE test (
    ->     c1 INT
    -> );
Query OK, 0 rows affected (0.03 sec)
````

**2. 表中数据和索引**

**① 系统表空间（system tablespace）**

默认情况下，InnoDB会在数据目录下创建一个名为`ibdata1`、大小为`12M`的`自拓展`文件，这个文件就是对应的`系统表空间`在文件系统上的表示。

**② 独立表空间(file-per-table tablespace)**

在MySQL5.6.6以及之后的版本中，InnoDB并不会默认的把各个表的数据存储到系统表空间中，而是为`每一个表建立一个独立表空间`，也就是说我们创建了多少个表，就有多少个独立表空间。使用`独立表空间`来存储表数据的话，会在该表所属数据库对应的子目录下创建一个表示该独立表空间的文件，文件名和表名相同。


```java
表名.ibd
```

> MySQL8.0中不再单独提供`表名.frm`，而是合并在`表名.ibd`文件中。

**③ 系统表空间与独立表空间的设置**

我们可以自己指定使用`系统表空间`还是`独立表空间`来存储数据，这个功能由启动参数`innodb_file_per_table`控制

```sql
[server] 
innodb_file_per_table=0 # 0：代表使用系统表空间； 1：代表使用独立表空间
```

**④ 其他类型的表空间**

随着MySQL的发展，除了上述两种老牌表空间之外，现在还新提出了一些不同类型的表空间，比如通用表空间（general tablespace）、临时表空间（temporary tablespace）等。

###  **2.3.2 MyISAM存储引擎模式**

**1.** **表结构**

在存储表结构方面， MyISAM 和 InnoDB 一样，也是在`数据目录`下对应的数据库子目录下创建了一个专门用于描述表结构的文件

```sql
表名.frm
```

**2.** **表中数据和索引**

在MyISAM中的索引全部都是`二级索引`，该存储引擎的`数据和索引是分开存放`的。所以在文件系统中也是使用不同的文件来存储数据文件和索引文件，同时表数据都存放在对应的数据库子目录下。

```sql
test.frm 存储表结构 #MySQL8.0 改为了 b.xxx.sdi
test.MYD 存储数据 (MYData) 
test.MYI 存储索引 (MYIndex
```

## 2.4 小结

举例: 数据库a ， 表b 。

1. 如果表b采用 InnoDB ，data\a中会产生1个或者2个文件:

   1. b.frm :描述表结构文件，字段长度等
   2. 如果采用 系统表空间 模式的，数据信息和索引信息都存储在 ibdata1 中
   3. 如果采用 独立表空间 存储模式，data\a中还会产生 b.ibd 文件(存储数据信息和索引信息) 此外:

   1 MySQL5.7 中会在data/a的目录下生成 db.opt 文件用于保存数据库的相关配置。比如:字符集、比较 规则。而MySQL8.0不再提供db.opt文件。

   2 MySQL8.0中不再单独提供b.frm，而是合并在b.ibd文件中。

2、如果表b采用 MyISAM ，data\a中会产生3个文件:

- MySQL5.7 中: b.frm :描述表结构文件，字段长度等。
- MySQL8.0 中 b.xxx.sdi :描述表结构文件，字段长度等
- b.MYD (MYData):数据信息文件，存储数据信息(如果采用独立表存储模式)
- b.MYI (MYIndex):存放索引信息文件
- 