---
title: 02ioc依赖查找和依赖注入
toc: true
tags: spring
categories: 
    - [java]
    - [spring]
---


##  1. 依赖查找【掌握】

### 1.1 最简单的实验-byName

上一章已经做过最简单的实验了，不再重复。
<!--more-->

### 1.2 根据类型查找-byType

为了与上面的实验区分开，咱复制原有的 `quickstart-byname.xml` ，并拷贝出一份新的 `quickstart-bytype.xml` ，咱在这里面修改。

person

```java
package org.clxmm.basic_dl.b_bytype.bean;

/**
 * @author clxmm
 * @Description
 * @create 2022-08-29 20:33
 */
public class Person {
}

```

声明 bean 时，这次我不再声明 id 属性：

```xml
<bean class="org.clxmm.basic_dl.b_bytype.bean.Person"></bean>
```

启动类 `QuickstartByTypeApplication` 中，这次调用的方法不再是传 name 的 `getBean` 方法，而是直接传 `Class` 类型：

```java
 /**
     * 根据类型查找
     */
    public static void main(String[] args) throws Exception {
        BeanFactory context = new ClassPathXmlApplicationContext("basic_dl/quickstart-bytype.xml");

        Person person = context.getBean(Person.class);
        System.out.println(person);

    }
```

有木有注意到，这次接收的 person 不用强转了！（那不是废话嘛 →_→ ，都把类型传进去了，人家 `BeanFactory` 给你找的时候肯定就是这个类型呀）

运行 `main` 方法，发现可以正常打印出 `Person` 的全限定名：

```
org.clxmm.basic_dl.b_bytype.bean.Person@31ef45e3
```

### 1.3 接口与实现类

咱把之前介绍 IOC 思想的 Dao 拿过来：

```java
package org.clxmm.basic_dl.b_bytype.dao;

public interface DemoDao {

    void sayHello();
}

package org.clxmm.basic_dl.b_bytype.dao.impl;

import org.clxmm.basic_dl.b_bytype.dao.DemoDao;


public class DemoDaoImpl implements DemoDao {
    @Override
    public void sayHello() {
        System.out.println("hello spring");
    }
}


```

之后在 `quickstart-bytype.xml` 中加入 `DemoDaoImpl` 的声明定义：

```xml
<bean class="org.clxmm.basic_dl.b_bytype.dao.impl.DemoDaoImpl"/>
```

之后在启动类 `QuickstartByTypeApplication` 中，借助 `BeanFactory` 取出 `DemoDao` ，并打印 `sayHello` 方法的返回数据：

```java
public static void main(String[] args) throws Exception {
        BeanFactory context = new ClassPathXmlApplicationContext("basic_dl/quickstart-bytype.xml");

        Person person = context.getBean(Person.class);
        System.out.println(person);


        DemoDao demoDao = context.getBean(DemoDao.class);
        demoDao.sayHello();
    }
```

运行 `main` 方法，控制台可以打印出 `hello spring` ，证明 `DemoDaoImpl` 也成功注入，并且 `BeanFactory` 可以根据接口类型，找到对应的实现类。

## 2. 依赖注入【掌握】

由上面的实例可以发现一个问题：创建的 Bean 都是不带属性的！如果我要创建的 Bean 需要一些预设的属性，那该怎么办呢？那就涉及到 IOC 的另外一种实现了，就是**依赖注入**。还是延续 IOC 的思想，**如果你需要属性依赖，不要自己去找，交给 IOC 容器，让它帮你找**，并给你赋上值。

### 2.1 最简单的实验-简单属性值注入

新建一个包 `basic_di` ，咱在这里面写有关依赖注入的实验。

#### 2.1.1 声明类+配置文件

```java
public class Person {
    private String name;
    private Integer age;
    // getter and setter ......
}

public class Cat {
    private String name;
    private Person person;
    // getter and setter ......
}
```

之后，咱在 `resources` 目录下新建 `basic_di` 文件夹，并声明配置文件 `inject-set.xml` ：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="person" class="org.clxmm.basic_di.a_quickstart_set.bean.Person"></bean>

    <bean id="cat" class="org.clxmm.basic_di.a_quickstart_set.bean.Cat"></bean>
</beans>
```

到现在为止，这些操作还都是咱学过的内容吧，没有新的知识。

#### 2.1.2 编写启动类

回到包下，新增一个启动类 `QuickstartInjectBySetXmlApplication` ，并编写 `main` 方法初始化 `BeanFactory` ：

```java
public static void main(String[] args) throws Exception {


        BeanFactory beanFactory = new ClassPathXmlApplicationContext("basic_di/inject-set.xml");
        Person person = beanFactory.getBean(Person.class);
        System.out.println(person);

        Cat cat = beanFactory.getBean(Cat.class);
        System.out.println(cat);
    }
```

运行 `main` 方法，发现打印的 person 与 cat 的所有属性都是 null 

```
Person(name=null, age=null)
Cat(name=null, person=null)
```

#### 2.1.3 给Person赋属性值

下面咱给 Person 的两个属性赋值。在 `<bean>` 标签中

在 person 的 `<bean>` 标签中声明 `property` 标签，这里面有两个属性：**name - 属性名，value - 属性值**。所以咱可以用如下方式来进行属性赋值：

```xml
<bean id="person" class="org.clxmm.basic_di.a_quickstart_set.bean.Person">
  <property name="name" value="clxmm"/>
  <property name="age" value="18"/>
</bean>
```

声明之后，保存，回到启动类，重新运行，发现 person 已经有值了：

```
Person(name=clxmm, age=18)
Cat(name=null, person=null)
```

### 2.2 关联Bean赋值

上面打印的结果明显 cat 还没有值，而且 master 的类型是 `Person` ，下面咱要给 cat 赋值。

对于 `property` 标签，除了可以声明 `value` 之外，还可以声明另外一个属性：**`ref`** ，它代表**要关联赋值的 Bean 的 id** 。 由此，对于 cat 中的 person 属性，可以有如下赋值方法：

```xml
<bean id="cat" class="org.clxmm.basic_di.a_quickstart_set.bean.Cat">
  <property name="name" value="mimi"/>
  <!-- ref引用上面的person对象 -->
  <property name="person" ref="person"/>
</bean>
```

声明好后，重新运行启动类，发现 cat 也有属性了：

```ini
Person(name=clxmm, age=18)
Cat(name=mimi, person=Person(name=clxmm, age=18))
```

最后，咱对比一下这两种 IOC 的实现方式。

## 3. 【面试题】依赖查找与依赖注入的对比

以下答案仅供参考，可根据自己的理解调整回答内容：

- 作用目标不同
  - 依赖注入的作用目标通常是类成员
  - 依赖查找的作用目标可以是方法体内，也可以是方法体外
- 实现方式不同
  - 依赖注入通常借助一个上下文被动的接收
  - 依赖查找通常主动使用上下文搜索

