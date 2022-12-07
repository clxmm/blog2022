---
title: 02克隆虚拟机
toc: true
tags: linux
categories: 
    - [java]
    - [linux]
---


 ## 1.克隆虚拟机

 ## 2.修改配置

 ### 1.开机前，修改mac地址
 设置-》网络适配器-》高级-》重新生成mac地址 

<!--more-->
开机
### 2.登录

### 2.修改ip centos

```sh
cd /etc/sysconfig/network-scripts
vim ifcfg-ens33
```

```
BOOTPROTO="static" #dhcp改为static 
ONBOOT="yes" #开机启用本配置
IPADDR=192.168.126.102 #静态IP，和物理机中网络连接的vmnet8的ip不同
GATEWAY=192.168.126.2 #默认网关
NETMASK=255.255.255.0 #子网掩码
DNS1=114.114.114.119 #DNS 配置
DNS2=114.114.115.119
```




配完保存后重启下网络服务即可
```
service network restart
 nbb
```

./npc -config=/app/app/nps/conf/npc.conf -server=8.136.154.60:9624 -vkey=iywpkpasfuafhzh6 -type=tcp

