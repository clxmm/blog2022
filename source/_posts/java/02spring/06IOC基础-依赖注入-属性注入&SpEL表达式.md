---
title: 06IOC基础-依赖注入-属性注入&SpEL表达式
toc: true
tags: spring
categories: 
    - [java]
    - [spring]
---


##  1. setter属性注入【掌握】

论依赖注入哪个最简单，那当属 “ setter 注入” 。下面咱来介绍最简单的 Bean 的 setter 注入。

### 1.1 xml方式的setter注入

在最开始的入门阶段，其实咱就已经学过基于 xml 的 setter 注入了，简单回顾一下吧：

```xml
    <bean id="person" class="org.clxmm.basic_di.a_quickstart_set.bean.Person">
        <property name="name" value="clxmm"/>
        <property name="age" value="18"/>
    </bean>
```

<!--more-->

### 1.2 注解方式的setter注入

注解形式的 setter 注入，咱之前学过的是在 bean 的创建时，编程式设置属性：

```java
@Bean
public Person person() {
    Person person = new Person();
    person.setName("test-person-anno-byset");
    person.setAge(18);
    return person;
}
```

## 2. 构造器注入【掌握】

### 2.1 修改Bean

有一些 bean 的属性依赖，需要在调用构造器（构造方法）时就设置好；或者另一种情况，有一些 bean 本身没有无参构造器，这个时候就必须使用**构造器注入**了。

```java
public Person(String name, Integer age) {
    this.name = name;
    this.age = age;
}
```

加上这个构造方法后，默认的无参构造方法就没了，这样原来的 `<bean>` 标签创建时就会失效，提示没有默认的构造方法：

```
Caused by: java.lang.NoSuchMethodException: com.linkedbear.spring.basic_di.b_constructor.bean.Person.<init>()

```

为此，咱需要学习新的 bean 构造和属性注入方法。

### 2.2 xml方式的构造器注入

在 `<bean>` 标签的内部，可以声明一个子标签：`constructor-arg` ，顾名思义，它是指构造器参数，由它可以指定构造器中的属性，来进行属性注入。`constructor-arg` 标签的编写规则如下：

```xml
<bean id="person" class="com.linkedbear.spring.basic_di.b_constructor.bean.Person">
    <constructor-arg index="0" value="test-person-byconstructor"/>
    <constructor-arg index="1" value="18"/>
</bean>
```

一个标签中有两部分，分别指定构造器的参数索引和参数值。这个地方真的能体现出 IDEA 的强大，如果没有在 `<bean>` 标签中声明 `constructor-arg` ，它会直接报红并提示帮你生成

由此，可以对这些属性进行注入。

### 2.3 注解式构造器属性注入

注解驱动的 bean 注册中，也是直接使用编程式赋值即可：

```java
@Bean
public Person person() {
    return new Person("test-person-anno-byconstructor", 18);
}
```

## 3. 注解式属性注入【掌握】

看到这里，是不是突然有点迷？哎，上面不是都介绍了注解式的 setter 和构造器的注入了吗？为什么又突然开了一节介绍呢？

回想一下，注册 bean 的方式不仅有 `@Bean` 的方式，还有组件扫描呢！那些声明式注册好的组件，它们的属性怎么处理呢？所以这一节咱就专门拿出来介绍这部分，如果这部分出现了一些新的内容，咱也同样在 xml 的方式下演示。

### 3.1 @Component下的属性注入

先介绍最简单的属性注入方式：**`@Value`** 。

新建一个 `Black` 类，并声明 `name` 和 `order` ，不过这次咱不设置 setter 方法了：

```java
@Component
public class Black {
    private String name;
    private Integer order;
    
    @Override
    public String toString() {
        return "Black{" + "name='" + name + '\'' + ", order=" + order + '}';
    }
}
```

实现注解式属性注入，可以直接在要注入的字段上标注 **`@Value`** 注解：

```java
@Value("black-value-anno")
private String name;

@Value("0")
private Integer order;
```

随后，咱使用组件扫描的形式，将这个 `Black` 类扫描到 IOC 容器，并取出打印：

```java
@Component
public class Black {
    @Value("black-value-anno")
    private String name;

    @Value("0")
    private Integer order;

    @Override
    public String toString() {
        return "Black{" + "name='" + name + '\'' + ", order=" + order + '}';
    }
}
```

运行 `main` 方法，发现 `Black` 的属性已经注入进去了：

```ini
simple value : Black{name='black-value-anno', order=0}

```

### 3.2 外部配置文件引入-@PropertySource

不是讲属性注入吗？怎么又扯到外部配置文件了？回想一下咱一开始学 IOC 思想的时候，咱不是搞了一个 properties 文件嘛，如果咱需要在 SpringFramework 中使用的话，应该怎么办呢？还是像之前那样用 Properties 类去 IO 读取？SpringFramework 自然能想到这个需求，于是就又扩展出了一个注解，用于导入外部的配置文件：`@PropertySource` 。

#### 3.2.1 创建Bean+配置文件

新建一个 `Red` 类，结构与 `Black` 完全一致。

之后在工程的 resources 目录下新建一个 `red.properties` ，用于存放 `Red` 的属性的配置：

```properties
red.name=red-value-byproperties
red.order=1
```

#### 3.2.2 引入配置文件

使用时，只需要将 **`@PropertySource`** 注解标注在配置类上，并声明 properties 文件的位置，即可导入外部的配置文件：

```java
@Configuration
// 顺便加上包扫描
@ComponentScan("org.clxmm.basic_di.c_value_spel.bean")
@PropertySource("classpath:basic_di/value/red.properties")
public class InjectValueConfiguration {
}
```

#### 3.2.3 Red类的属性注入

对于 properties 类型的属性，`@Value` 需要配合**占位符**来表示注入的属性，我先写，写完你一下子就明白了：

```java
    @Value("${red.name}")
    private String name;
    
    @Value("${red.order}")
    private Integer order;
```

是不是突然熟悉！这不跟 jsp 里的 el 表达式一个样吗？哎没错，还真就这样！

#### 3.2.4 测试启动类

修改启动类，将包扫描启动改为配置类启动，随后将 `Red` 取出：

```java
    public static void main(String[] args) throws Exception {
        ApplicationContext ctx = new AnnotationConfigApplicationContext(InjectValueConfiguration.class);
        Red red = ctx.getBean(Red.class);
        System.out.println("properties value : " + red);
    }
```

运行 `main` 方法，控制台打印 `Red` 的信息，证明配置文件的属性已经成功注入：

```ini
properties value : Red{name='red-value-byproperties', order=1}

```

#### 3.2.5 xml中使用占位符

对于 xml 中，占位符的使用方式与 `@Value` 是一模一样的：

```xml
<bean class="com.linkedbear.spring.basic_di.c_value_spel.bean.Red">
    <property name="name" value="${red.name}"/>
    <property name="order" value="${red.order}"/>
</bean>
```

#### 3.2.6 占位符的取值范围【理解】

作为一个 **properties 文件**，它加载到 SpringFramework 的 IOC 容器后，**会转换成 Map 的形式来保存**这些配置，而 SpringFramework 中本身在初始化时就有一些配置项，这些配置项也都放在这个 Map 中。**占位符的取值就是从这些配置项中取**。

> 多提一嘴，实际上这些配置属性和值存放的真实位置是一个叫 **`Environment`** 的抽象中，在后面的 IOC 进阶和高级部分，小册会拿出专门开篇幅讲解 `Environment` 的设计，以及 properties 文件的加载

### 3.3 SpEL表达式【了解会用】

不是在讲属性注入吗？怎么突然提到 SpEL 了？试想一下，如果咱要在属性注入时，使用一些特殊的数值（如一个 Bean 需要依赖另一个 Bean 的某个属性，或者需要动态处理一个特定的属性值），这种情况 **${}** 的占位符方式就办不了了（占位符只能取配置项），需要一种更强大的表达方式来满足这种需求，这种表达方式就是 SpEL 表达式。

#### 3.3.1 快速了解SpEL

SpEL 全称 Spring Expression Language ，它从 SpringFramework 3.0 开始被支持，它本身可以算 SpringFramework 的组成部分，但又可以被独立使用。它可以支持调用属性值、属性参数以及方法调用、数组存储、逻辑计算等功能。

> 如果有接触过 Struts2 / FreeMarker 的小伙伴应该知道 OGNL ，它也是一种表达式语言，只不过 OGNL 是一个单独的开源项目，而 SpEL 是由 Spring 推出的表达式语言，而且 SpEL 默认本身内嵌在 SpringFramework 中。

#### 3.3.2 SpEL属性注入

下面咱使用 SpEL进行最简单的属性注入。SpEL 的语法统一用 **`#{}`** 表示，花括号内部编写表达式语言。

创建一个 `Blue` ，也是像上面一样声明 name 和 order ，并提供 getter 、setter 方法（为了方便后续操作）和 `toString()` 方法，最后用 `@Component` 标注。

使用 `@Value` 配合 SpEL 完成字面量的属性注入，需要额外在花括号内部加单引号：

```java
@Component
@Data
public class Blue {
    @Value("#{'blue-value-byspel'}")
    private String name;

    @Value("#{2}")
    private Integer order;
}

```

修改启动类，从 IOC 容器中取 `Blue` 并打印，可以发现字面量被成功注入：

```ini
Blue{name='blue-value-byspel', order=2}

```

#### 3.3.3 Bean属性引用

如果 SpEL 的功能仅仅是这样，那真的太弱鸡了，SpEL 可以取 IOC 容器中其它 Bean 的属性，下面咱来演示。

上面的注入中咱已经注册了 `Blue` ，下面咱再创建一个 `Green` ，以同样的方式对字段和方法进行声明，同时标注 `@Component` 注解。

在 name 属性上，咱希望直接拿 `Blue` 的 name 贴过来；order 属性希望它比 blue 的 order 大 1，则可以这样编写：

```java
@Component
@data
public class Green {
    
    @Value("#{'copy of ' + blue.name}")
    private String name;
    
    @Value("#{blue.order + 1}")
    private Integer order;
}
```

修改启动类，测试运行，发现 `Blue` 的属性已经成功被取出了：

```
use spel bean property : Green{name='copy of blue-value-byspel', order=3}

```

xml 的使用方式也很简单：

```xml
<bean class="com.linkedbear.spring.basic_di.c_value_spel.bean.Green">
    <property name="name" value="#{'copy of ' + blue.name}"/>
    <property name="order" value="#{blue.order + 1}"/>
</bean>
```

#### 3.3.4 方法调用

SpEL 表达式不仅可以引用对象的属性，还可以直接引用类常量，以及调用对象的方法等，下面咱演示方法调用和常量引入。

新建一个 `White` ，以同样的方式初始化属性、`toString()` 、注解。

咱设想一个简单的需求，让 name 取 blue 属性的前 3 个字符，order 取 `Integer` 的最大值，则使用 SpEL 可以这样写：

```java
@Component
public class White {
    
    @Value("#{blue.name.substring(0, 3)}")
    private String name;
    
    @Value("#{T(java.lang.Integer).MAX_VALUE}")
    private Integer order;
}
```

注意，直接引用类的属性，需要在类的全限定名外面使用 **T()** 包围。

修改启动类，测试运行，发现 `White` 的属性已经是处理之后的值了：

```ini
use spel methods : White{name='blu', order=2147483647}

```

xml 的方式，同样都是使用 value 属性：

```xml
<bean class="com.linkedbear.spring.basic_di.c_value_spel.bean.White">
    <property name="name" value="#{blue.name.substring(0, 3)}"/>
    <property name="order" value="#{T(java.lang.Integer).MAX_VALUE}"/>
</bean>
```

