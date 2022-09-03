---
title: 01SpringFramework概述与IOC的依赖查找
toc: true
tags: spring
categories: 
    - [java]
    - [spring]
---


##  1.IOC的思想引入【重点】

```java
private DemoDao dao = new DemoDaoImpl();

private DemoDao dao = (DemoDao) BeanFactory.getBean("demoDao");
```
<!--more-->

上面的是强依赖 / 紧耦合，在编译期就必须保证 `DemoDaoImpl` 存在；下面的是弱依赖 / 松散耦合，只有到运行期反射创建时才知道 `DemoDaoImpl` 是否存在。

再对比看，上面的写法是主动声明了 `DemoDao` 的实现类，只要编译通过，运行一定没错；而下面的写法没有指定实现类，而是由 `BeanFactory` 去帮咱查找一个 name 为 `demoDao` 的对象，倘若 `factory.properties` 中声明的全限定类名出现错误，则会出现强转失败的异常 `ClassCastException` 。

仔细体会下面这种对象获取的方式，本来咱开发者可以使用上面的方式，主动声明实现类，但如果选择下面的方式，那就不再是咱自己去声明，而是**将获取对象的方式交给了 `BeanFactory`** 。这种**将控制权交给别人**的思想，就可以称作：**控制反转（ Inverse of Control , IOC ）**。而 `BeanFactory` 根据指定的 `beanName` 去获取和创建对象的过程，就可以称作：**依赖查找（ Dependency Lookup , DL ）**。

## 2. SpringFramework概述【了解】

### 2.1 官方网站主页

引用官方网站主页的说明，Spring 官方对 SpringFramework 的描述是这样的：

[spring.io/projects/sp…](https://spring.io/projects/spring-framework)

> The Spring Framework provides a comprehensive programming and configuration model for modern Java-based enterprise applications - on any kind of deployment platform.
>
> A key element of Spring is infrastructural support at the application level: Spring focuses on the "plumbing" of enterprise applications so that teams can focus on application-level business logic, without unnecessary ties to specific deployment environments.
>
> Spring 框架为**任何类型的部署平台**上的**基于 Java** 的现代**企业应用程序**提供了全面的**编程和配置模型**。
>
> Spring 的一个关键元素是在**应用程序级别的基础架构支持**：Spring 专注于企业应用程序的 “**脚手架**” ，以便团队可以**专注于应用程序级别的业务逻辑**，而不必与特定的部署环境建立不必要的联系。

这段描述的内容只能用一个词概括：要素过多！对这里面的一些概念作一些解释，方便咱更好地理解这段话。

- 任何类型的部署平台：无论是操作系统，还是 Web 容器（ Tomcat 等）都是可以部署基于 SpringFramework 的应用
- 企业应用程序：包含 JavaSE 和 JavaEE 在内，它被称为一站式解决方案
- 编程和配置模型：基于框架编程，以及基于框架进行功能和组件的配置
- 基础架构支持：SpringFramework 不含任何业务功能，它只是一个底层的应用抽象支撑
- 脚手架：使用它可以更快速的构建应用

### 2.2 官方文档

上面看到的只是官方网站的 SpringFramework 工程首页的介绍概述，进入到 5.2 版本的官方文档中，这里面也有一段解释：

[docs.spring.io/spring/docs…](https://docs.spring.io/spring/docs/5.2.x/spring-framework-reference/overview.html#overview)

> Spring makes it easy to create Java enterprise applications. It provides everything you need to embrace the Java language in an enterprise environment, with support for Groovy and Kotlin as alternative languages on the JVM, and with the flexibility to create many kinds of architectures depending on an application’s needs.
>
> Spring 使创建企业级 Java 应用程序变得容易。它提供了在企业环境中使用Java语言所需的一切，并支持 Groovy 和 Kotlin 作为 JVM 上的替代语言，并且可以根据应用程序的需求灵活地创建多种体系结构。

这个描述算是更为概括和笼统吧，它同样解释了 SpringFramework 的强大和使用范围之广，另外它还提了一嘴 “运行在 JVM 上的第二语言”，这些东西咱可能很陌生，那就先不看它了。

### 2.3 网络流传概述

- SpringFramework 是一个分层的、JavaSE / JavaEE 的一站式轻量级开源框架，以 IOC 和 AOP 为内核，并提供表现层、持久层、业务层等领域的解决方案，同时还提供了整合第三方开源技术的能力。
- SpringFramework 是一个 JavaEE 编程领域的轻量级开源框架，它是为了解决企业级编程开发中的复杂性，实现敏捷开发的应用型框架。SpringFramework 是一个容器框架，它集成了各个类型的工具，通过核心的 IOC 容器实现了底层的组件实例化和生命周期管理。
- SpringFramework 是一个开源的容器框架，核心是 IOC 和 AOP ，它为了简化企业级开发而生。SpringFramework 有诸多优良特性（非侵入、容器管理、组件化、轻量级、一站式等）。

观察这些概述，对比官方文档中的描述，可以额外的提取出几个关键词：

-  IOC & AOP：SpringFramework 的两大核心特性：Inverse of Control 控制反转、Aspect Oriented Programming 面向切面编程
- 轻量级：对比于重量级框架，它的规模更小（可能只有几个 jar 包）、消耗的资源更少
- 一站式：覆盖企业级开发中的所有领域
- 第三方整合：SpringFramework 可以很方便的整合进其他的第三方技术（如持久层框架 MyBatis / Hibernate ，表现层框架 Struts2 ，权限校验框架 Shiro 等）
- 容器：SpringFramework 的底层有一个管理对象和组件的容器，由它来支撑基于 SpringFramework 构建的应用的运行

### 1.4 【面试题】面试中如何概述SpringFramework

**SpringFramework 是一个开源的、松耦合的、分层的、可配置的一站式企业级 Java 开发框架，它的核心是 IOC 与 AOP ，它可以更容易的构建出企业级 Java 应用，并且它可以根据应用开发的组件需要，整合对应的技术。**

解释下这样概括的要点：

- 加入 “**松耦合**” 的概念是为了描述 IOC 和 AOP ，如果面试继续问 IOC 或耦合相关的内容，那这部分就可以拿去做回应
- 加入 “**可配置**” 是为了给 SpringBoot 垫底（可能还没到这一步，不过现在记住就好啦，后续会讲的）
- IOC 和 AOP 可提可不提，毕竟你只要学了它就肯定知道（人家 Spring 官方都懒得提它。。。）
- 没有提 “轻量级” ，是考虑到现在的大环境趋势早已经没有 EJB 的身影了（EJB是什么东西下面就会提到）
- 没有提 “容器” 的概念，是因为 SpringFramework 不仅仅是一个容器，如果只是限定死容器那相当于说窄了
- 注意对比 “企业级Java开发” 与 “JavaEE开发” 的区别：SpringFramework 不仅能构建在 Web 项目，对于普通的 JavaSE 项目、GUI 项目，也是可以用 SpringFramework 的

## 2. 为什么使用SpringFramework【重点，面试题】

通过上面对 SpringFramework 的概述，想必也能总结出一些优点和强大之处：

【以下内容可用于面试题】

- **IOC**：组件之间的解耦（咱上一章已经体会到了）
- **AOP**：切面编程可以将应用业务做统一或特定的功能增强，能实现应用业务与增强逻辑的解耦
- **容器**与事件：管理应用中使用的组件Bean、托管Bean的生命周期、事件与监听器的驱动机制
- Web、事务控制、测试、与**其他技术的整合**

## 3. SpringFramework的发展历史【了解】

### 1.SpringFramework的版本迭代

下面咱列出一个 SpringFramework 的重要版本更新时间及重大特性，现阶段小伙伴们可以只是看一眼，后面咱讲到具体的内容时都会提及到

| SpringFramework版本 | **对应jdk版本** | **重要特性**                                                 |
| ------------------- | --------------- | ------------------------------------------------------------ |
| SpringFramework 1.x | jdk 1.3         | 基于 xml 的配置                                              |
| SpringFramework 2.x | jdk 1.4         | 改良 xml 文件、初步支持注解式配置                            |
| SpringFramework 3.x | Java 5          | 注解式配置、JavaConfig 编程式配置、Environment 抽象          |
| SpringFramework 4.x | Java 6          | SpringBoot 1.x、核心容器增强、条件装配、WebMvc 基于 Servlet3.0 |
| SpringFramework 5.x | Java 8          | SpringBoot 2.x、响应式编程、SpringWebFlux、支持 Kotlin       |

## 4. SpringFramework包含的模块【熟悉，面试题】

大致了解一下 SpringFramework 的核心模块，以及包含的技术，混个脸熟。

【以下内容可用于面试题】

- beans、core、context、expression 【核心包】
- aop 【切面编程】
- jdbc 【整合 jdbc 】
- orm 【整合 ORM 框架】
- tx 【事务控制】
- web 【 Web 层技术】
- test 【整合测试】
- ......

## 5. 快速入门-IOC-DL【掌握】

下面咱用一个最最简单的实例，来体会 SpringFramework 中对于依赖查找的使用。

### 5.1 引入依赖

对于快速入门阶段来讲，咱只需要引入一个依赖即可：`spring-context` （此处引入的版本是 5.2.8 ）

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>5.2.8.RELEASE</version>
</dependency>
```

### 5.2 创建配置文件

像前面咱推演出来的规矩差不多，SpringFramework 实现 IOC 可以借助配置文件的方式来描述类和对象的定义信息。在工程的 `resources` 目录下，咱创建一个 `quickstart-byname.xml` 文件（为了使配置文件的存放更具条理，且容易维护，咱提前创建好一个文件夹）：

`basic_dl`

这里面的初始内容是预先规定好的，从 SpringFramework 的官方文档中可以找到 xml 配置文件的空架子：[docs.spring.io/spring/docs…](https://docs.spring.io/spring/docs/5.2.x/spring-framework-reference/core.html#beans-factory-instantiation)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">

</beans>
```

将这段内容粘贴到 `quickstart-byname.xml` 中即可。

### 5.3 [可选] 配置IDEA项目工程中的应用上下文

小伙伴们可能注意到了，粘贴了上面这段 xml 后，IDEA 会弹出一段提示：

由此可见 IDEA 是多么的智能，它意识到你要在工程中添加 SpringFramework 的配置文件，它就想让你把这个配置文件配置到 IDEA 的项目工作环境下，那咱只需要按照提示，点击 `Configure application context` ，随后点击 `Create new application context...` ，会弹出一个对话框，让你创建应用上下文：

啥也不用管，直接点 OK ，就完事了。此番动作是让 IDEA 也知道，咱在使用 SpringFramework 开发应用，IDEA 会自动识别咱写的配置，可以帮我们省很多心。

### 5.4 声明一个普通的类

由于我的 `src/main/java` 中创建的包结构是按照咱小册的章节划分的，小伙伴们可以根据自己的习惯和喜好，划分包结构。

```java
package org.clxmm.basic_dl.a_quickstart_byname.bean;

public class Person {

}
```

创建好就可以放那儿了，也不用在里面写这写那的。

### 5.5 在配置文件中加入Person声明

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="person" class="org.clxmm.basic_dl.a_quickstart_byname.bean.Person"></bean>
</beans>



```

可以看到声明的规则很简单，跟之前咱写的 properties 形式几乎一个意思，也是 key 和 value ，只不过这里分别对应的是 id 和 class 罢了。

这么写完之后 IDEA 会报 xml 标签体为空，根据咱学过的 HTML 基础，应该知道，没有标签体的情况下是可以省略闭合标签的，咱这里就不省略了，后续还要往里面加东西

### 5.6 创建启动类

有了配置文件，下一步可以来读这个配置文件了。在 `a_quickstart_byname` 包下创建一个 `QuickstartByNameApplication` 类，并声明 `main` 方法：

`main` 方法中要读这个配置文件，方法有很多种，咱快速入门中先来使用一种比较简单的方法：

```java
public static void main(String[] args) throws Exception {
  BeanFactory factory = new ClassPathXmlApplicationContext("basic_dl/quickstart-byname.xml");
  Person person = (Person) factory.getBean("person");
  System.out.println(person);
}
```

解释一下这段代码的意思。读取配置文件，需要一个载体来加载它，这里咱选用 `ClassPathXmlApplicationContext` 来加载。加载完成后咱直接使用 `BeanFactory` 接口来接收（多态思想）。下一步就可以从 `BeanFactory` 中获取 `person` 了，由于咱在配置文件中声明了 id ，故这里就可以直接把 id 传入，`BeanFactory` 就可以给我们返回 `Person` 对象。

运行 `main` 方法，可以成功打印出 `Person` 的全限定类名 + 内存地址，证明编写成功。

到这里，就可以轻松上手 SpringFramework 中 IOC 依赖查找的实现了。

