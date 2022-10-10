---
title: 01shiro
toc: true
tags: shiro
categories: 
    - [java]
    - [shiro]
---


##  1.入门概述

<!--more-->

### 1.1 是什么

 Apache Shiro 是一个功能强大且易于使用的 Java 安全(权限)框架。Shiro 可以完 成:认证、授权、加密、会话管理、与 Web 集成、缓存 等。借助 Shiro 您可以快速轻松 地保护任何应用程序——从最小的移动应用程序到最大的 Web 和企业应用程序。

[官网shiro](https://shiro.apache.org/)

### 1.2 为什么要用 Shiro

自 2003 年以来，框架格局发生了相当大的变化，因此今天仍然有很多系统在使用Shiro。这与 Shiro 的特性密不可分。

易于使用:使用 Shiro 构建系统安全框架非常简单。就算第一次接触也可以快速掌握

全面:Shiro 包含系统安全框架需要的功能，满足安全需求的“一站式服务”。

灵活:Shiro 可以在任何应用程序环境中工作。虽然它可以在 Web、EJB 和 IoC 环境灵活:Shiro 可以在任何应用程序环境中工作。虽然它可以在 Web、EJB 和 IoC 环境

强力支持 Web:Shiro 具有出色的 Web 应用程序支持，可以基于应用程序 URL 和Web 协议(例如 REST)创建灵活的安全策略，同时还提供一组 JSP 库来控制页面输出。

兼容性强:Shiro 的设计模式使其易于与其他框架和应用程序集成。Shiro 与Spring、Grails、Wicket、Tapestry、Mule、Apache Camel、Vaadin 等框架无缝集成。

社区支持:Shiro 是 Apache 软件基金会的一个开源项目，有完备的社区支持，文档支持。如果需要，像 Katasoft 这样的商业公司也会提供专业的支持和服务。

### 1.3 Shiro 与 SpringSecurity 的对比

- 1、Spring Security 基于 Spring 开发，项目若使用 Spring 作为基础，配合 SpringSecurity 做权限更加方便，而 Shiro 需要和 Spring 进行整合开发;
- 2、Spring Security 功能比 Shiro 更加丰富些，例如安全维护方面;
- 3、Spring Security 社区资源相对比 Shiro 更加丰富;
- 4、Shiro 的配置和使用比较简单，Spring Security 上手复杂些;
- 5、Shiro 依赖性低，不需要任何框架和容器，可以独立运行.Spring Security 依赖Spring 容器;
- 6、shiro 不仅仅可以使用在 web 中，它可以工作在任何应用环境中。在集群会话时 Shiro最重要的一个好处或许就是它的会话是独立于容器的。

### 1.4 基本功能

#### 1、基本功能点如下图所示

![](/img/202210/shiro/01shiro.png)

#### 2、功能简介

- (1)Authentication:身份认证/登录，验证用户是不是拥有相应的身份;
- (2)Authorization:授权，即权限验证，验证某个已认证的用户是否拥有某个权限;即判断用 户是否能进行什么操作，如:验证某个用户是否拥有某个角色。或者细粒度的验证某个用户 对某个资源是否具有某个权限;
- (3)Session Manager:会话管理，即用户登录后就是一次会话，在没有退出之前，它的所有 信息都在会话中;会话可以是普通 JavaSE 环境，也可以是 Web 环境的;
- (4)Cryptography:加密，保护数据的安全性，如密码加密存储到数据库，而不是明文存储;
- (5)Web Support:Web 支持，可以非常容易的集成到 Web 环境;
- (6)Caching:缓存，比如用户登录后，其用户信息、拥有的角色/权限不必每次去查，这样可 以提高效率;
- (7)Concurrency:Shiro 支持多线程应用的并发验证，即如在一个线程中开启另一个线程，能把权限自动传播过去;
- (8)Testing:提供测试支持;
- (9)Run As:允许一个用户假装为另一个用户(如果他们允许)的身份进行访问;
- (10)Remember Me:记住我，这个是非常常见的功能，即一次登录后，下次再来的话不用登 录了

### 1.5 原理

#### 1、Shiro 架构(Shiro 外部来看)

从外部来看 Shiro ，即从应用程序角度的来观察如何使用 Shiro 完成 工作

![](/img/202210/shiro/02shiro.png)

**Shiro 架构**

- (1)Subject:应用代码直接交互的对象是 Subject，也就是说 Shiro 的对外 API 核心就是 Subject。Subject 代表了当前“用户”， 这个用户不一定 是一个具体的人，与当前应用交互的任何东西都是 Subject，如网络爬虫， 机器人等;与 Subject 的所有交互都会委托给 SecurityManager; Subject 其实是一个门面，SecurityManager 才是实际的执行者;
- (2)SecurityManager:安全管理器;即所有与安全有关的操作都会与 SecurityManager交互;且其管理着所有 Subject;可以看出它是 Shiro 的核心，它负责与 Shiro 的其他组件进行交互，它相当于 SpringMVC 中 DispatcherServlet 的角色
- (3)Realm:Shiro 从 Realm 获取安全数据(如用户、角色、权限)，就是说SecurityManager 要验证用户身份，那么它需要从 Realm 获取相应的用户 进行比较以确定用户身份是否合法;也需要从 Realm 得到用户相应的角色/ 权限进行验证用户是否能进行操作;可以把 Realm 看成 DataSource

#### 2、Shiro 架构(Shiro 内部来看)

![](/img/202210/shiro/03shiro.png)

**Shiro 架构**

- (1)Subject:任何可以与应用交互的“用户”;
- (2)SecurityManager :相当于 SpringMVC 中的 DispatcherServlet;是 Shiro 的心脏; 所有具体的交互都通过 SecurityManager 进行控制;它管理着所有 Subject、且负责进 行认证、授权、会话及缓存的管理。
- (3)Authenticator:负责 Subject 认证，是一个扩展点，可以自定义实现;可以使用认证 策略(Authentication Strategy)，即什么情况下算用户认证通过了;
- (4)Authorizer:授权器、即访问控制器，用来决定主体是否有权限进行相应的操作;即控 制着用户能访问应用中的哪些功能;
- (5)Realm:可以有 1 个或多个 Realm，可以认为是安全实体数据源，即用于获取安全实体 的;可以是 JDBC 实现，也可以是内存实现等等;由用户提供;所以一般在应用中都需要 实现自己的 Realm;
- (6)SessionManager:管理 Session 生命周期的组件;而 Shiro 并不仅仅可以用在 Web环境，也可以用在如普通的 JavaSE 环境
- (7)CacheManager:缓存控制器，来管理如用户、角色、权限等的缓存的;因为这些数据基本上很少改变，放到缓存中后可以提高访问的性能
- (8)Cryptography:密码模块，Shiro 提高了一些常见的加密组件用于如密码加密/解密。

