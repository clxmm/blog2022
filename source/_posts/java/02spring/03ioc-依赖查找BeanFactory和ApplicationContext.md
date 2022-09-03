---
title: 03ioc-依赖查找BeanFactory和ApplicationContext
toc: true
tags: spring
categories: 
    - [java]
    - [spring]
---

上一章，咱了解了 IOC 的两种实现的基本用法，这一章咱来介绍更多关于依赖查找的使用方式。



##  1. 依赖查找的多种姿势【掌握】

<!--more-->

### 1.1 ofType

试想，如果一个接口有多个实现，而咱又想一次性把这些都拿出来，那 `getBean` 方法显然就不够用了，需要使用额外的方式。

回到 `basic_dl` 包下，咱新创建一个 `oftype` 的包，来测试 **ofType** 的查找方式。

#### 1.1.1 声明Bean+配置文件

声明一个 `DemoDao` ，并声明 3 种对应的实现类，分别模拟操作 MySQL 、Oracle  数据库的实现类

代码

```java
public interface DemoDao {
}

public class DemoDaoMysql implements DemoDao {
}
public class DemoDaoOracle implements DemoDao {
}

```

对应的配置类，也把这几个 Bean 都注册上：

```xml
    <bean id="demoDaoMysql" class="org.clxmm.basic_dl.c_oftype.dao.impl.DemoDaoMysql"/>
    <bean id="demoDaoOracle" class="org.clxmm.basic_dl.c_oftype.dao.impl.DemoDaoOracle"/>
```

#### 1.1.2 测试启动类

在启动类中，创建 `BeanFactory` 后，尝试一次性取出多个 Bean ，结果发现 `BeanFactory` 中并没有这样的方法：

没有这种实现吗？？？那我咋获取这些呢？

其实并不是人家没实现，只是咱用错了接口而已 (￣▽￣)／

这样，咱先改，改完了再解释为什么这么改。

#### 1.1.3 改用ApplicationContext

将 `BeanFactory` 接口换为 `ApplicationContext` ，再次尝试调用方法，发现了一个这样的方法：

它可以**传入一个类型，返回一个 Map** ，而 Map 中的 value 不难猜测就是**传入的参数类型对应的那些类 / 实现类**

```java
<T> Map<String, T> getBeansOfType(@Nullable Class<T> var1) throws BeansException;
```

那咱就拿出来，foreach 一下呗：

```java
public static void main(String[] args) throws IOException {
        ApplicationContext ctx = new ClassPathXmlApplicationContext("basic_dl/quickstart-oftype.xml");

        Map<String, DemoDao> beans = ctx.getBeansOfType(DemoDao.class);
        beans.forEach((beanName, bean) -> {
            System.out.println(beanName + ": " + bean.toString());
        });
}
```

console

```ini
demoDaoMysql: org.clxmm.basic_dl.c_oftype.dao.impl.DemoDaoMysql@31ef45e3
demoDaoOracle: org.clxmm.basic_dl.c_oftype.dao.impl.DemoDaoOracle@598067a5
```

这样就实现了传入一个接口 / 抽象类，返回容器中所有的实现类 / 子类。

讲到这里，咱先停下来，解释下为什么换用 `ApplicationContext` 。

## 2. BeanFactory与ApplicationContext【掌握】

借助 IDEA ，发现 `ApplicationContext` 也是一个接口，而且通过接口继承关系发现它是 `BeanFactory` 的子接口。那咱想了解这两个接口，最好的办法还是先翻一翻官方文档，从官方文档中尝试获取最权威的解释。

### 2.1 官方文档的解释

在官方文档 [docs.spring.io/spring/docs…](https://docs.spring.io/spring/docs/5.2.x/spring-framework-reference/core.html#beans-introduction) 中，有一个段落解释了这两个接口的关系：

> The `org.springframework.beans` and `org.springframework.context` packages are the basis for Spring Framework’s IoC container. The [`BeanFactory`](https://docs.spring.io/spring-framework/docs/5.2.x/javadoc-api/org/springframework/beans/factory/BeanFactory.html interface provides an advanced configuration mechanism capable of managing any type of object. [`ApplicationContext`](https://docs.spring.io/spring-framework/docs/5.2.x/javadoc-api/org/springframework/context/ApplicationContext.html) is a sub-interface of `BeanFactory`. It adds:
>
> - Easier integration with Spring’s AOP features
> - Message resource handling (for use in internationalization)
> - Event publication
> - Application-layer specific contexts such as the `WebApplicationContext` for use in web applications.
>
> `org.springframework.beans` 和 `org.springframework.context` 包是 SpringFramework 的 IOC 容器的基础。`BeanFactory` 接口提供了一种高级配置机制，能够管理任何类型的对象。`ApplicationContext` 是 `BeanFactory` 的子接口。它增加了：
>
> - 与 SpringFramework 的 AOP 功能轻松集成
> - 消息资源处理（用于国际化）
> - 事件发布
> - 应用层特定的上下文，例如 Web 应用程序中使用的 `WebApplicationContext`

这样说下来，给咱的主观感受是：**`ApplicationContext` 包含 `BeanFactory` 的所有功能，并且人家还扩展了好多特性**，其实就是这么回事。

而且，官方文档的下面还有一段，解释了我们为什么应该用 `ApplicationContext` 而不是 `BeanFactory` ：[docs.spring.io/spring/docs…](https://docs.spring.io/spring/docs/5.2.x/spring-framework-reference/core.html#context-introduction-ctx-vs-beanfactory)

> You should use an `ApplicationContext` unless you have a good reason for not doing so, with `GenericApplicationContext` and its subclass `AnnotationConfigApplicationContext` as the common implementations for custom bootstrapping. These are the primary entry points to Spring’s core container for all common purposes: loading of configuration files, triggering a classpath scan, programmatically registering bean definitions and annotated classes, and (as of 5.0) registering functional bean definitions.
>
> 你应该使用 `ApplicationContext` ，除非能有充分的理由解释不需要的原因。一般情况下，我们推荐将 `GenericApplicationContext` 及其子类 `AnnotationConfigApplicationContext` 作为自定义引导的常见实现。这些实现类是用于所有常见目的的 SpringFramework 核心容器的主要入口点：加载配置文件，触发类路径扫描，编程式注册 Bean 定义和带注解的类，以及（从5.0版本开始）注册功能性 Bean 的定义

这段话的下面还给了一张表，对比了 `BeanFactory` 与 `ApplicationContext` 的不同指标：

| **Feature**                                                  | BeanFactory | ApplicationContext |
| ------------------------------------------------------------ | ----------- | ------------------ |
| Bean instantiation/wiring —— Bean的实例化和属性注入          | Yes         | Yes                |
| Integrated lifecycle management —— 生命周期管理              | No          | Yes                |
| Automatic `BeanPostProcessor` registration —— Bean后置处理器的支持 | No          | Yes                |
| Automatic `BeanFactoryPostProcessor` registration —— BeanFactory后置处理器的支持 | No          | Yes                |
| Convenient `MessageSource` access (for internalization) —— 消息转换服务（国际化） | No          | Yes                |
| Built-in `ApplicationEvent` publication mechanism —— 事件发布机制（事件驱动） | No          | Yes                |

由此可以发现，`ApplicationContext` 真的比 `BeanFactory` 强大太多了，所以咱还是选择使用 `ApplicationContext` 吧！

### 2.2 【面试题】BeanFactory与ApplicationContext的对比

`eanFactory` 接口提供了一个**抽象的配置和对象的管理机制**，`ApplicationContext` 是 `BeanFactory` 的子接口，它简化了与 AOP 的整合、消息机制、事件机制，以及对 Web 环境的扩展（ `WebApplicationContext` 等），`BeanFactory` 是没有这些扩展的。

`ApplicationContext` 主要扩展了以下功能：

- AOP 的支持（ `AnnotationAwareAspectJAutoProxyCreator` 作用于 Bean 的初始化之后 ）
- 配置元信息（ `BeanDefinition` 、`Environment` 、注解等 ）
- 资源管理（ `Resource` 抽象 ）
- 事件驱动机制（ `ApplicationEvent` 、`ApplicationListener` ）
- 消息与国际化（ `LocaleResolver` ）
- `Environment` 抽象（ SpringFramework 3.1 以后）

好了，初步了解了 `BeanFactory` 与 `ApplicationContext` 的区别，也知道 `ApplicationContext` 更加强大，下面咱就继续研究依赖查找的花板子吧。

## 3. 继续研究依赖查找

### 3.1 withAnnotation【熟悉】

IOC 容器除了可以根据一个父类 / 接口来找实现类，还可以根据类上标注的注解来查找对应的 Bean 。下面咱来测试包含注解的 Bean 如何被查找。

#### 3.1.1 声明Bean+注解+配置文件

新建一个包 `d_withanno` ，并在里面声明一个注解：`@Color` 。

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Color {
}

```

之后，创建几个类，以及运行 `main` 方法的启动类：

```java
public class Dog {
}

@Color
public class Green {
}

@Color
public class Red {
}
```

对应的，配置文件中声明好这几个类。

```xml

<bean class="org.clxmm.basic_dl.d_withanno.bean.Red"/>
<bean class="org.clxmm.basic_dl.d_withanno.bean.Green"/>
<bean class="org.clxmm.basic_dl.d_withanno.bean.Dog"/>
```

#### 3.1.2 测试启动类

```java
public static void main(String[] args) {
        ApplicationContext ctx = new ClassPathXmlApplicationContext("basic_dl/quickstart-withanno.xml");
        Map<String, Object> beans = ctx.getBeansWithAnnotation(Color.class);
        System.out.println(beans.size());
        beans.forEach((beanName, bean) -> {
            System.out.println(beanName + "--" + bean.toString());
        });

}
```

console

```ini
2
org.clxmm.basic_dl.d_withanno.bean.Red#0--org.clxmm.basic_dl.d_withanno.bean.Red@370736d9
org.clxmm.basic_dl.d_withanno.bean.Green#0--org.clxmm.basic_dl.d_withanno.bean.Green@5f9d02cb
```

### 3.2 获取IOC容器中的所有Bean【熟悉】

如果真的会有这么一个需求，要取出当前 IOC 容器中的所有 bean ，这个时候一个一个取是不现实的，因为你根本不知道都有谁，不知道到底有哪些你还没见过的小盆友们。。。所以这个时候就要用到 `ApplicationContext` 的另一个方法了：`getBeanDefinitionNames` 。

看这个方法名，感觉像是获取 bean 的定义名称，这跟 name 有什么关系吗？这里先告诉你，它获取的就是那些 Bean 的 **id** ，至于这些关于定义的信息，后续也会讲到的，小伙伴们先学习这些比较基础的东西即可。

接下来咱就试一下这个 `getBeanDefinitionNames` 方法的效果，编写一个新的启动类，这次咱就不再造 bean 了，咱直接拿上面刚测试过的吧：

```java
public static void main(String[] args) {
        ApplicationContext ctx = new ClassPathXmlApplicationContext("basic_dl/quickstart-withanno.xml");
        String[] beanNames = ctx.getBeanDefinitionNames();
        // 利用jdk8的Stream快速编写打印方法
        Stream.of(beanNames).forEach(System.out::println);
}
```

```ini
org.clxmm.basic_dl.d_withanno.bean.Red#0
org.clxmm.basic_dl.d_withanno.bean.Green#0
org.clxmm.basic_dl.d_withanno.bean.Dog#0
```

那既然 Bean 的 id 都出来了，那取出来还不是轻而易举？这个咱都很熟悉了，就不再演示了。

## 4. 依赖查找的高级使用——延迟查找【熟悉】

对于一些特殊的场景，需要依赖容器中的某些特定的 Bean ，但当它们不存在时也能使用默认 / 缺省策略来处理逻辑。这个时候，使用上面已经学过的方式倒是可以实现，但编码可能会不很优雅。

### 4.1 使用现有方案实现Bean缺失时的缺省加载

咱把设计做的简单一些，准备两个 bean ：`Cat` 和 `Dog` ，但是在 xml 中咱只注册 `Cat` ，这样 IOC 容器中就只有 `Cat` ，没有 `Dog` 。

```java
public class Cat {
}
public class Dog {

}
```

quickstart-lazylookup.xml

```xml
 <bean id="cat" class="org.clxmm.basic_dl.f_lazylookup.Cat"/>
```

之后，咱来编写启动类。由于 Dog 没有在 IOC 容器中，所以调用 `getBean` 方法时会报 `NoSuchBeanDefinitionException` ，为了保证能在没有找到 Bean 的时候启用缺省策略，咱可以在 catch 块中手动创建，实现代码如下：

```java
public static void main(String[] args) {
        ApplicationContext ctx = new ClassPathXmlApplicationContext("basic_dl/quickstart-lazylookup.xml");
        Cat cat = ctx.getBean(Cat.class);
        System.out.println(cat);

        Dog dog;
        try {
            dog = ctx.getBean(Dog.class);
        } catch (NoSuchBeanDefinitionException e) {
            // 找不到Dog时手动创建
            dog = new Dog();
        }
        System.out.println(dog);
}
```

console

```ini
org.clxmm.basic_dl.f_lazylookup.Cat@598067a5
org.clxmm.basic_dl.f_lazylookup.Dog@553f17c
```

可以发现这种编码方式相当不优雅，而且很别扭（性能也低）。如果真的后续每一个 bean 都这样操作，那编码量岂不是巨大？这肯定不行，一定有改良方案。

### 4.2 改良-获取之前先检查

既然作为一个容器，能获取自然就能有检查，`ApplicationContext` 中有一个方法就可以专门用来检查容器中是否有指定的 Bean ：`containsBean`

```java
    Dog dog = ctx.containsBean("dog") ? (Dog) ctx.getBean("dog") : new Dog();

```

但注意，这个 `containsBean` 方法只能传 bean 的 id ，不能查类型，所以虽然可以改良前面的方案，但还是有问题：如果 Bean 的名不叫 dog ，叫 wangwang ，那这个方法岂不是废了？所以这个方案还是不够好，需要改良。

### 4.3 改良-延迟查找

如果能有一种机制，我想获取一个 Bean 的时候，你可以**先不给我报错，先给我一个包装让我拿着，回头我自己用的时候再拆开决定里面有还是没有**，这样是不是就省去了 IOC 容器报错的麻烦事了呢？在 SpringFramework 4.3 中引入了一个新的 API ：**`ObjectProvider`** ，它可以实现延迟查找。

于是，咱可以改良上面的代码如下：（为了保留上面的案例示范，下面新起一个类）

```java
public static void main(String[] args) throws Exception {
  ApplicationContext ctx = new ClassPathXmlApplicationContext("basic_dl/quickstart-lazylookup.xml");
  Cat cat = ctx.getBean(Cat.class);
  System.out.println(cat);
  // 下面的代码会报Bean没有定义 NoSuchBeanDefinitionException
  // Dog dog = ctx.getBean(Dog.class);

  // 这一行代码不会报错
  ObjectProvider<Dog> dogProvider = ctx.getBeanProvider(Dog.class);
}
```

可以发现，`ApplicationContext` 中有一个方法叫 `getBeanProvider` ，它就是返回上面说的那个**“包装”**。如果直接 `getBean` ，那如果容器中没有对应的 Bean ，就会报 `NoSuchBeanDefinitionException`；如果使用这种方式，运行 `main` 方法后发现并没有报错，只有调用 `dogProvider` 的 `getObject` ，真正要取包装里面的 Bean 时，才会报异常。所以总结下来，`ObjectProvider` 相当于**延后了 Bean 的获取时机，也延后了异常可能出现的时机**。

但是，上面的问题还没有被解决呀，调用 `getObject` 方法还是会报异常，那下面咱就继续研究 `ObjectProvider` 的其他一些方法。

`ObjectProvider` 中还有一个方法：`getIfAvailable` ，它可以在**找不到 Bean 时返回 null 而不抛出异常**。使用这个方法，就可以避免上面的问题了。改良之后的代码如下：

```java
Dog dog = dogProvider.getIfAvailable();
if (dog == null) {
  dog = new Dog();
}
```

### 4.5 ObjectProvider在jdk8的升级

随着 SpringFramework 5.0 基于 jdk8 的发布，函数式编程也被大量用于 SpringFramework 中。`ObjectProvider` 中新加了几个方法，可以使编码更佳优雅。

看着上面 4.4 节的那几行代码，有木有联想到 `Map` 中的 `getOrDefault` ？由此，`ObjectProvider` 在 SpringFramework 5.0 后扩展了一个带 `Supplier` 参数的 `getIfAvailable` ，它可以在找不到 Bean 时直接用 **`Supplier`** 接口的方法返回默认实现，由此上面的代码还可以进一步简化为：

```java
    Dog dog = dogProvider.getIfAvailable(() -> new Dog());

```

或者更简单的，使用方法引用：

```
    Dog dog = dogProvider.getIfAvailable(Dog::new);

```

当然，一般情况下，取出的 Bean 都会马上或者间歇的用到，`ObjectProvider` 还提供了一个 `ifAvailable` 方法，可以在 Bean 存在时执行 `Consumer` 接口的方法：

```
    dogProvider.ifAvailable(dog -> System.out.println(dog)); // 或者使用方法引用

```

以上就是关于延迟查找的内容，这种方案可以使用，但在日常开发中可能使用的不是很多，小伙伴们实际动手操作一遍，有一个印象即可。后续如果升级到更高的位置，这部分或许会在封装组件和底层时用到。

