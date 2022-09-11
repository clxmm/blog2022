---
title: 07IOC基础-依赖注入-自动注入&复杂类型注入
toc: true
tags: spring
categories: 
    - [java]
    - [spring]
---
上一章咱对最简单基础的属性注入有了比较全面的了解和学习，这一章咱要考虑另外一个问题了：一个 Bean 要依赖另一个 Bean 怎么办？在 xml 中可以声明 ref 属性，用注解怎么办？如果 Bean 里要注入复杂类型（数组、集合、Map 等）又需要怎么办呢？

##  1. 自动注入【掌握】

xml 中的 ref 属性可以在一个 Bean 中注入另一个 Bean ，注解同样也可以这样做，它可以使用的注解有很多种，咱一一来学习。
<!--more-->

### 1.1 @Autowired

在 Bean 中直接在 **属性 / setter 方法** 上标注 `@Autowired` 注解，IOC 容器会**按照属性对应的类型，从容器中找对应类型的 Bean 赋值到对应的属性**上，实现自动注入。

咱来编写一个案例，以下部分均放在 `d_complexfield` 包下。

#### 1.1.1 创建Bean

预先创建好几个 Bean ，用来过会做演示用。

```java
@Data
@Component
public class Person {
    private String name = "clx";
}
```

```java
@Data
@Component
public class Dog {

    @Value("wangwang")
    private String name;

    private Person person;

}
```

#### 1.1.2 给Dog注入Person的三种方式

对于 `@Autowired` 的使用，只需要在属性上标注即可：

```java
@Component
public class Dog {
  // ......
  @Autowired
  private Person person;
```

也可以使用构造器注入方式：

```java
@Autowired
public Dog(Person person) {
  this.person = person;
}
```

亦可以使用 setter 方法注入：

```java
@Autowired
public void setPerson(Person person) {
  this.person = person;
}
```

#### 1.1.3 测试启动类

编写启动类，把上面的 `Person` 和 `Dog` 都扫描进 IOC 容器，之后取出 `Dog` 并打印：

```java
public static void main(String[] args) {
  ApplicationContext ctx = new AnnotationConfigApplicationContext("org.clxmm.basic_di.d_complexfield.bean");
  Dog dog = ctx.getBean(Dog.class);
  System.out.println(dog);
}
```

#### 1.1.4 注入的Bean不存在

将 `Person` 上面的 `@Component` 暂时的注释掉，此时 IOC 容器中应该没有 `Person` 了吧，再次运行启动类，可以发现

```ini
Caused by: org.springframework.beans.factory.NoSuchBeanDefinitionException: No qualifying bean of type 'org.clxmm.basic_di.d_complexfield.bean.Person' available: expected at least 1 bean which qualifies as autowire candidate. Dependency annotations: {}

```

简单概括这个异常，就是说**本来想找一个类型为 `Person` 的 Bean ，但一个也没找到**！那必然没找到啊，`@Component` 注解被注释掉了，自然就不会注册了。如果出现这种情况下又不想让程序抛异常，就需要在 `@Autowired` 注解上加一个属性：**`required = false`** 。

```java
    @Autowired(required = false)
    private Person person;
```

再次运行启动类，可以发现控制台打印 `person=null` ，但没有抛出异常：

```ini
Dog{name='dogdog', person=null}

```

### 1.2 @Autowired在配置类的使用

`@Autowired` 不仅可以用在普通 Bean 的属性上，在配置类中，注册 `@Bean` 时也可以标注：

```java
@Configuration
@ComponentScan("com.linkedbear.spring.basic_di.d_complexfield.bean")
public class InjectComplexFieldConfiguration {

    @Bean
    @Autowired // 高版本可不标注
    public Cat cat(Person person) {
        Cat cat = new Cat();
        cat.setName("mimi");
        cat.setPerson(person);
        return cat;
    }
}

```

由于配置类的上下文中没有 `Person` 的注册了（使用了 `@Component` 模式注解），自然也就没有 `person()` 方法给咱调，那就可以使用 `@Autowired` 注解来进行自动注入了。（其实不用标，SpringFramework 也知道自己得注入了）

将扫描包换为配置类驱动，可以发现 cat 也能打印出来了：

```java
public static void main(String[] args) throws Exception {
    ApplicationContext ctx = new AnnotationConfigApplicationContext(InjectComplexFieldConfiguration.class);

    Cat cat = ctx.getBean(Cat.class);
    System.out.println(cat);
}
```

### 1.3 多个相同类型Bean的自动注入

刚才咱已经使用 `@Component` 模式注解，在 `Person` 类上标注过了，此时 IOC 容器就应该有一个 `Person` 类型的 Bean 了。下面咱在配置类中再注册一个 ：

```java
@Bean
public Person master() {
  Person master = new Person();
  master.setName("master");
  return master;
}
```

这样 IOC 容器就应该有两个 `Person` 对象了吧！接下来咱改一个地方，给 `Person` 的 `@Component` 加一个名称：

```java
@Component("administrator")
```

此时，两个 person 一个叫 master ，一个叫 administrator 。

下面咱直接运行测试启动类，可以发现控制台会报这样一个错误：

```ini
Caused by: org.springframework.beans.factory.NoUniqueBeanDefinitionException: No qualifying bean of type 'com.linkedbear.spring.basic_di.d_complexfield.bean.Person' available: expected single matching bean but found 2: administrator,master

```

IOC 容器发现有两个类型相同的 `Person` ，它也不知道注入哪一个了，索性直接 “我选择死亡” ，就挂了。

出现这个问题不能就这样不管啊，得先办法啊。SpringFramework 针对这种情况专门提供了两个注解，可以使用两种方式解决该问题。

#### 1.3.1 @Qualifier：指定注入Bean的名称

`@Qualifier` 注解的使用目标是要注入的 Bean ，它配合 `@Autowired` 使用，可以显式的指定要注入哪一个 Bean ：

```java
    @Autowired
    @Qualifier("administrator")
    private Person person;
```

重新运行测试类，可以发现 `Dog` 中注入的 `Person` 是 administrator 。

#### 1.3.2 @Primary：默认Bean

`@Primary` 注解的使用目标是被注入的 Bean ，在一个应用中，一个类型的 Bean 注册只能有一个，它配合 `@Bean` 使用，可以指定默认注入的 Bean ：

```java
@Bean
@Primary
public Person master() {
  Person master = new Person();
  master.setName("master");
  return master;
}
```

重新运行测试类，发现 `Cat` 中注入的 `Person` 是 master ，`Dog` 中注入的还是 administrator ，可见 `@Qualifier` 不受 `@Primary` 的干扰。

同样的，在 xml 中可以指定 `<bean>` 标签中的 `primary` 属性为 true ，跟上面标注 `@Primary` 注解是一样的。

#### 1.3.3 另外的办法

```java
    @Autowired
    private Person administrator;
```

重新运行测试类，发现可以注入了，而且没有报 `expected single matching bean but found 2` 的异常。

#### 1.3.4 【面试题】@Autowired注入的原理逻辑

由此可以总结出 `@Autowired` 的注入逻辑：（以下答案仅供参考，可根据自己的理解调整回答内容）

**先拿属性对应的类型，去 IOC 容器中找 Bean ，如果找到了一个，直接返回；如果找到多个类型一样的 Bean ， 把属性名拿过去，跟这些 Bean 的 id 逐个对比，如果有一个相同的，直接返回；如果没有任何相同的 id 与要注入的属性名相同，则会抛出 `NoUniqueBeanDefinitionException` 异常。**

### 1.4 多个相同类型Bean的全部注入

上面都是注入一个 Bean 的方式，通过两种不同的办法来保证注入的唯一性。但如果需要一下子把所有指定类型的 Bean 都注入进去应该怎么办呢？其实答案也挺简单的，**注入一个用单个对象接收，注入一组对象就用集合来接收**：

```java
@Component
public class Dog {
    // ......
    
    @Autowired
    private List<Person> persons;
```

如上就可以实现一次性把所有的 `Person` 都注入进来，重新运行启动类，可以发现 persons 中有两个对象：

```ini
Dog{name='dogdog', person=Person{name='administrator'}, persons=[Person{name='administrator'}, Person{name='master'}]}
```

以上就是关于 `@Autowired` 的使用，下面咱再介绍两个用于自动注入的规范。

### 1.5 JSR250-@Resource

介绍 JSR250 规范之前，先简单了解下 JSR 。

> JSR 全程 **Java Specification Requests** ，它定义了很多 Java 语言开发的规范，有专门的一个组织叫 JCP ( Java Community Process ) 来参与定制。
>
> 有关 JSR250 规范的说明文档可参考官方文档：[jcp.org/en/jsr/deta…](https://jcp.org/en/jsr/detail?id=250)

回到正题，`@Resource` 也是用来属性注入的注解，它与 `@Autowired` 的不同之处在于：**`@Autowired` 是按照类型注入，`@Resource` 是直接按照属性名 / Bean的名称注入**。

是不是突然有点狂喜，这个 **`@Resource` 注解相当于标注 `@Autowired` 和 `@Qualifier`** 了！实际开发中，`@Resource` 注解也是用的很多的，可以根据情况来进行选择。

为了不与上面的代码起冲突，咱另创建一个 `Bird` ，也注入 `Person` ，不过这次咱直接用 `@Resource` 注解指定要注入的 Person ：

```java
@Component
public class Bird {
    
    @Resource(name = "master")
    private Person person;
```

之后在启动类中取出 `Bird` 并打印，可以发现确实正常注入了 name 为 "master" 的 `Person` 。

```ini
Bird{person=Person{name='master'}}

```

### 1.6 JSR330-@Inject

JSR330 也提出了跟 `@Autowired` 一样的策略，它也是**按照类型注入**。不过想要用 JSR330 的规范，需要额外导入一个依赖：

```xml
<!-- jsr330 -->
<dependency>
    <groupId>javax.inject</groupId>
    <artifactId>javax.inject</artifactId>
    <version>1</version>
</dependency>
```

剩下的使用方式就跟 SpringFramework 原生的 `@Autowired` + `@Qualifier` 一样了：

```java
@Component
public class Cat {
    
    @Inject // 等同于@Autowired
    @Named("admin") // 等同于@Qualifier
    private Person master;

```

#### 1.6.1 @Autowired与@Inject对比

可能会有小伙伴问了，那这个 `@Inject` 都跟 SpringFramework 原生的 `@Autowired` 一个作用，那我还用它干嘛？来看一眼包名：

```java
import org.springframework.beans.factory.annotation.Autowired;

import javax.inject.Inject;
```

是不是突然明白了点什么？如果万一项目中没有 SpringFramework 了，那么 `@Autowired` 注解将失效，但 `@Inject` 属于 **JSR 规范，不会因为一个框架失效而失去它的意义**，只要导入其它支持 JSR330 的 IOC 框架，它依然能起作用。

### 1.7 【面试题】依赖注入的注入方式

| 注入方式   | 被注入成员是否可变 | 是否依赖IOC框架的API                                         | 使用场景                           |
| ---------- | ------------------ | ------------------------------------------------------------ | ---------------------------------- |
| 构造器注入 | 不可变             | 否（xml、编程式注入不依赖）                                  | 不可变的固定注入                   |
| 参数注入   | 不可变             | 否（高版本中注解配置类中的 `@Bean` 方法参数注入可不标注注解） | 注解配置类中 `@Bean` 方法注册 bean |
| 属性注入   | 不可变             | 是（只能通过标注注解来侵入式注入）                           | 通常用于不可变的固定注入           |
| setter注入 | 可变               | 否（xml、编程式注入不依赖）                                  | 可选属性的注入                     |

### 1.8 【面试题】自动注入的注解对比

| **注解**   | **注入方式** | **是否支持@Primary** | **来源**                   | Bean不存在时处理                   |
| ---------- | ------------ | -------------------- | -------------------------- | ---------------------------------- |
| @Autowired | 根据类型注入 | 是                   | SpringFramework原生注解    | 可指定required=false来避免注入失败 |
| @Resource  |              | 是                   | JSR250规范                 | 容器中不存在指定Bean会抛出异常     |
| @Inject    | 根据类型注入 | 是                   | JSR330规范 ( 需要导jar包 ) | 容器中不存在指定Bean会抛出异常     |

`@Qualifier` ：如果被标注的成员/方法在根据类型注入时发现有多个相同类型的 Bean ，则会根据该注解声明的 name 寻找特定的 bean

`@Primary` ：如果有多个相同类型的 Bean 同时注册到 IOC 容器中，使用 “根据类型注入” 的注解时会注入标注 `@Primary` 注解的 bean

## 2. 复杂类型注入

这部分咱介绍的复杂类型注入包括如下几种：

- 数组
- List / Set
- Map
- Properties

### 2.1 创建复杂对象

```java
public class Person {

    private String[] names;
    private List<String> tels;
    private Set<Cat> cats;
    private Map<String, Object> events;
    private Properties props;
    // setter
```



### 2.2 xml复杂注入【掌握】

xml 注入复杂类型相对比较简单，咱先在 xml 中注册一个 `Person` （不要扫描 `Person` 所在的包）：

```java
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean class="com.linkedbear.spring.basic_di.g_complexfield.bean.Person"></bean>
</beans>
```

接下来，咱把其中的属性都一一赋值。

#### 2.2.1 数组注入

`<bean>` 标签中，要想给属性赋值，统统都是用 `<property>` 标签，对于简单注入和 Bean 的注入，可以通过 **value** 和 **ref** 完成，但复杂类型就必须在标签体内写子标签了。

`<property>` 标签中有好多子标签：

必然的，注入数组是用 `<array>` 标签咯，于是数组的注入可以这么写：

```xml
<property name="names">
    <array>
        <value>张三</value>
        <value>三三来迟</value>
    </array>
</property>
```

#### 2.2.2 List注入

array 与 list 的形式基本是一模一样的（本身 List 的底层就可以是数组），于是 List 的注入也很简单：

```xml
<property name="tels">
    <list>
        <value>13888</value>
        <value>15999</value>
    </list>
</property>
```

#### 2.2.3 Set注入

有木有小伙伴注意到，在写了 `<array>` 或者 `<list>` 标签后，里面的标签还是上面那一堆，说明集合是可以继续嵌套集合的吧！

在上面的 `Person` 模型中，Set 集合的泛型是 `Cat` ，也就说明咱要在这个集合中注入一组 Bean 。对于这些 Bean ，可以直接声明创建，也可以直接引用现有的 Bean 进行注入，如下所示：

```xml
<!-- 已经提前声明好的Cat -->
<bean id="mimi" class="com.linkedbear.spring.basic_di.g_complexfield.bean.Cat"/>
---

<property name="cats">
    <set>
        <bean class="com.linkedbear.spring.basic_di.g_complexfield.bean.Cat"/>
        <ref bean="mimi"/>
    </set>
</property>
```

#### 2.2.4 Map注入

**Map** 的底层是键值对，迭代的时候都是用 **Entry** 来取 key 和 value ，那在这里面也是这样设计的：（ key 和 value 都可以是 Bean 的引用）

```xml
<property name="events">
  <map>
    <entry key="8:00" value="起床"/>
    <!-- 撸猫 -->
    <entry key="9:00" value-ref="mimi"/>
    <!-- 买猫 -->
    <entry key="14:00">
      <bean class="com.linkedbear.spring.basic_di.g_complexfield.bean.Cat"/>
    </entry>
    <entry key="18:00" value="睡觉"/>
  </map>
```

#### 2.2.5 Properties注入

Properties 类型与 Map 其实是一模一样的，注入的方式也基本一样，只不过有一点：Properties 的 key 和 value 只能是 String 类型。

```xml
<property name="props">
    <props>
        <prop key="sex">男</prop>
        <prop key="age">18</prop>
    </props>
</property>
```

#### 2.2.6 测试启动类

编写启动类，加载 xml 文件，并取出 `Person` 打印，可以发现属性都被注入成功了：

```ini
Person{
  names=[张三, 三三来迟],
  tels=[13888, 15999],
  cats=[Cat{name='cat'}, Cat{name='cat'}],
  events={8:00=起床, 9:00=Cat{name='cat'}, 14:00=Cat{name='cat'}, 18:00=睡觉},
  props={age=18, sex=男}
}
```

### 2.3 注解复杂注入【会用】

为了能演示 Bean 的引用，咱给 `Cat` 加上 `@Component` 注解，并带上名称：

```java
@Component("miaomiao")
public class Cat {
    private String name = "cat";
```

下面咱进行注解注入。其实对于注解的注入，说白了还是借助 SpEL 表达式，上一章咱也说了，它的功能很强大，这一节咱继续体会一下。

这部分咱直接全部列出吧，使用 SpEL 表达式实现注解注入的方式：（江郎才尽了已经开始不知道注入什么东西好了 ~ ~ ~ 以下内容开始胡言乱语。。。）

```java
@Component
public class Person2 {
    
    @Value("#{new String[] {'张三', '张仨'}}")
    private String[] names;
    
    @Value("#{{'333333', '3333', '33'}}")
    private List<String> tels;
    
    // 引用现有的Bean，以及创建新的Bean
    @Value("#{{@miaomiao, new com.linkedbear.spring.basic_di.g_complexfield.bean.Cat()}}")
    private Set<Cat> cats;
    
    @Value("#{{'喵喵': @miaomiao.name, '猫猫': new com.linkedbear.spring.basic_di.g_complexfield.bean.Cat().name}}")
    private Map<String, Object> events;
    
    @Value("#{{'123': '牵着手', '456': '抬起头', '789': '我们私奔到月球'}}")
    private Properties props;
```

相应的，也都可以正常打印：

```
Person{
  names=[张三, 张仨],
  tels=[333333, 3333, 33],
  cats=[Cat{name='cat'}, Cat{name='cat'}],
  events={喵喵=miaomiao, 猫猫=cat},
  props={123=牵着手, 456=抬起头, 789=我们私奔到月球}
}
```

学是这么学，不过估计没什么人真的会在实际开发中这么肆无忌惮的用 SpEL 注入这种属性吧。。。

当然，别忘了，还有一种方式能做到复杂类型的注入：编程式。。。直接在配置类中声明 `@Bean` 注册也是可以的，这种方式最灵活。

