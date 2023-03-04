---
title: 01jenkins.md
toc: true
tags: jenkins
categories: 
    - [java]
    - [jenkins]
---



## 1.创建 Jenkins 实例

https://www.jenkins.io/zh/

这里我们将使用 Docker 进行安装一个单机版的 Jenkins（这里假设你了解 Docker 等工具的使用）：

```cmd
docker run -d --name jenkins \
  -p 50000:50000 \
  -p 8080:8080 \
  -v /srv/jenkins:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /usr/bin/docker:/usr/bin/docker \
  -u root \
  --restart always \
  jenkins/jenkins:2.263.4
```

<!--more-->



