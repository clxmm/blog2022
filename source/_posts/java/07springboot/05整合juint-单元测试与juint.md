---
title: 05整合juint-单元测试与juint
toc: true
tags: springboot
categories: 
    - [java]
    - [springboot]
---

经过前面两章的回顾，下面我们正式进入 SpringBoot 整合的环节。第一个讲解的整合技术是单元测试，也即 JUnit 。

单元测试本身是项目开发过程中不可或缺的一部分，好的单元测试可以覆盖尽可能多的业务代码，从而保证编写的程序功能的正确、健壮等。

## 1. 单元测试

首先我们先回顾一下单元测试的相关概念。


<!--more-->

### 1.1 什么是单元

何为单元？对于程序开发来讲，最终能分解的尽可能小的、独立的、可执行的元素，就可以理解为单元。在面向对象的程序开发中，程序可以分解的最小粒度是类中的**方法**。所以在 Java 开发中可以简单理解为：类中定义的一个一个的方法即是一个一个的单元。

### 1.2 什么是单元测试

既然理解了单元，那单元测试也就不难理解，顾名思义，单元测试即是对程序中一个一个的单元作测试。在面向对象的程序开发中，我们编写好一个类，如何确定功能是否正常可用？如何确定有没有 Bug ？这就需要使用单元测试来检验和排查。

### 1.3 为什么要引入单元测试

单元测试的意义是可以隔离程序的组件，通过最小测试范围确定出一个功能单元是否正常可用，通过单元测试的编写和执行，可以在尽可能早期筛查、发现出一些问题；除此之外，单元测试的编写是与程序代码相关的，当一组单元测试编写后，在程序功能设计不发生大改动时，这组测试是可以重复利用的，以此可以提高开发效率。

## 2. JUnit

JUnit 是针对 Java 语言的一个经典单元测试框架，它在测试驱动方面具有重大意义。JUnit 促进了“先测试后编码”的理论，它强调测试数据与程序代码的配合关系，使得开发者在程序开发中形成“编码一点，测试一点”的过程，这种编码习惯可以提高程序的正确性和稳定性，进而提高开发者的产出效率，减少后期排查错误的时间和精力。

JUnit 有着以下的特点：

-  开放的资源框架，用于编写和运行测试；
- 提供注释来识别测试方法；
- 提供断言来测试预期结果；
- 提供测试运行来运行测试；
- 允许编写代码更快，并能提高质量；
- 测试代码编写优雅简洁，花费时间较少；
- 测试代码可以自动运行并且检查自身结果并提供即时反馈，没有必要人工梳理测试结果的报告；
- 测试代码可以被组织为测试套件，包含测试用例，甚至其他的测试套件；

### 2.1 JUnit4

早期使用的 JUnit 版本为 4.x ，这个版本对 jdk 的最低限制是 jdk 1.5 ，整个 JUnit 4 的代码被整合到一个 jar 包中，使用时直接导入即可，主流的 IDE 都有对 JUnit 的原生支持。

使用 JUnit 4 的方式比较简单，只需要编写一个类，并声明一个无入参、无返回值的方法，并标注 `@Test` 注解即可被 IDE 识别且运行。

```java
public class DemoTest {
    
    @Test
    public void test() {
        System.out.println("DemoTest test run ......");
    }
}
```

### 2.2 JUnit5

2017 年 9 月，JUnit 5.0.0 正式发布，它最低支持的 Java 版本是 Java 8 ，而且它的构建不再由一个独立的 jar 包构成，而是以一组 jar 包共同组合而成。抛弃历史包袱，通过支持扩展（Extension），JUnit 5 给用户提供了定制特殊的测试需求与方式。

总的来说，JUnit 5 由 3 个模块构成，分别是 JUnit Platform 、JUnit Jupiter 、JUnit Vintage 。

- JUnit Platform ：基于 JVM 上启动测试框架的基础，不仅支持 JUnit 的测试引擎，也可以兼容其他的测试引擎；
- JUnit Jupiter ：JUnit 5 的核心，提供 JUnit 5 的新的编程模型，内部包含一个测试引擎，该测试引擎会基于 JUnit Platform 运行；
- JUnit Vintage ：兼容 JUnit 4 、JUnit 3 支持的测试引擎。

使用 JUnit 5 的方式跟 JUnit 4 并无太大区别，同样是编写测试类，并声明方法，标注 `@Test` 注解即可，不再编写示例代码解释。

### 2.3 JUnit5 与 JUnit4 的对比

JUnit 5 相较于 JUnit 4 比较大的改动是注解的使用。下表展示了 JUnit 5 跟 JUnit 4 常用注解的对比。

| 注解意义                         | JUnit 5      | JUnit 4      |
| -------------------------------- | ------------ | ------------ |
| 标注一个测试方法（无区别）       | @Test        | @Test        |
| 在每个测试方法前执行             | @BeforeEach  | @Before      |
| 在每个测试方法后执行             | @AfterEach   | @After       |
| 在当前类中的所有测试方法之前执行 | @BeforeAll   | @BeforeClass |
| 在当前类中的所有测试方法之后执行 | @AfterAll    | @AfterClass  |
| 禁用测试方法/类                  | @Disabled    | @Ignore      |
| 标记和过滤                       | @Tag         | @Category    |
| 声明测试工厂进行动态测试（新增） | @TestFactory | /            |
| 嵌套测试（新增）                 | @Nested      | /            |
| 注册自定义扩展（新增）           | @ExtendWith  | /            |

## 3. SpringBoot整合JUnit

下面我们通过一个整合的实例，讲解 SpringBoot 如何整合 JUnit 5 。

> 本模块对应的源码工程：**`spring-boot-integration-02-junit`**

### 3.1 整合方式

SpringBoot 整合 JUnit 测试的方法非常简单，只需要导入 `spring-boot-starter-test` 依赖即可。

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

![](/img/202212/5springboot1.png)

> 注意：
>
> 1）SpringBoot 2.2.0 以后会引入 JUnit 5 ，在此之前的版本则是引入 JUnit 4 ；
>
> 2）SpringBoot 2.4.0 及以上版本已移除 Vintage 模块，意味着 JUnit 4 的功能已不被支持。如果需要支持 JUnit 4 ，则需要额外导入一段依赖，详情参照官方文档 feature 的第 26 章：[docs.spring.io/spring-boot…](https://docs.spring.io/spring-boot/docs/2.5.14/reference/html/features.html#features.testing) 。

除此之外，还可以导入最基础的 WebMvc 依赖 `spring-boot-starter-web` ，或者导入无 Web 场景的依赖 `spring-boot-starter` 。

### 3.2 编写基础代码

对于一个 SpringBoot 应用而言，最简单的工程结构仅仅是一个主启动类，本章讲解的是 JUnit ，与具体的业务代码开发无太大关系，所以我们只需要编写一些最简单的代码即可。

主启动类：

```java
@SpringBootApplication
public class AppJuint {

    public static void main(String[] args) {
        SpringApplication.run(AppJuint.class,args);
    }

}

```

注解配置类（用于注册一些组件用于测试）：

```java
@Configuration
public class WebConfig {
    
    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

### 3.3 编写测试类



对于 SpringBoot 的测试类而言，需要在 `src/test/java` 目录下创建一个与主启动类同包的测试类，并标注测试相关的注解。在使用 JUnit 4 的时代，需要在 SpringBoot 测试类上标注两个注解，分别是 `@SpringBootTest` 与 `@RunWith(SpringRunner.class)` ，而到了 JUnit 5 时代，只需要标注一个 `@SpringBootTest` 注解即可。

```java
@SpringBootTest
class AppJuintTest {

    @Test
    public void test1() {
        System.out.println("springboot tset1");
    }

}
```

以此标注好的 SpringBoot 测试类就具备了驱动 SpringBoot 应用的能力。运行 `test1` 方法，可以发现控制台成功打印 SpringBoot 应用初始化的日志。

被 `@SpringBootTest` 注解标注的类，本身具备被依赖注入的能力，我们可以把上面 `WebConfig` 中注册的 `RestTemplate` 注入到 `SpringBootApplicationTest` 中：

```java
@SpringBootTest
class AppJuintTest {
    
    @Autowired
    private RestTemplate restTemplate;
    

    @Test
    public void test2() {
        System.out.println(restTemplate);
    }

}
```

运行 `test2` 方法，控制台可以正常打印出 `RestTemplate` 的引用地址，证明 SpringBoot 测试类可以正常使用依赖注入的相关注解。

#### 3.3.1 手动指定主启动类

下面解释一个特殊情况：我们编写 SpringBoot 测试类时，不可能把全部的测试类都放到与 SpringBoot 主启动类同包下，当测试类一多，整个 test 目录会非常混乱。为了方便寻找与管理，我们还是需要将单元测试类也分包管理。但是请各位注意，当 SpringBoot 测试类被放到其他包的时候，运行 SpringBoot 测试类是有区别的。

以下是将测试类放在 SpringBoot 主启动类所在包的子包下，当运行单元测试时，控制台仍然可以正常打印，说明放到 SpringBoot 主启动类所在包的子包下是完全可行的。

```java
package org.clxmm.juint.demo;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;

/**
 * @author clxmm
 * @Description
 * @create 2022-12-13 21:33
 */
@SpringBootTest
public class InnerTest {

    @Test
    public void test3() throws Exception {
        System.out.println("Inner test run ......");
    }
}

```

以下是将测试类放在 SpringBoot 主启动类所在包之外，当运行单元测试时，会提示一个异常：

```java
package demo;

import org.junit.jupiter.api.Test;
import org.springframework.boot.test.context.SpringBootTest;

/**
 * @author clxmm
 * @Description
 * @create 2022-12-13 21:33
 */
@SpringBootTest
public class OuterTest {

    @Test
    public void test4() throws Exception {
        System.out.println("Outer test run ......");
    }

}

```

```
测试已忽略.

java.lang.IllegalStateException: Unable to find a @SpringBootConfiguration, you need to use @ContextConfiguration or @SpringBootTest(classes=...) with your test

	at org.springframework.util.Assert.state(Assert.java:76)
	at org.springframework.boot.test.context.SpringBootTestContextBootstrapper.getOrFindConfigurationClasses(SpringBootTestContextBootstrapper.java:236)
	at org.springframework.boot.test.context.SpringBootTestContextBootstrapper.processMergedContextConfiguration(SpringBootTestContextBootstrapper.java:152)
	at org.springframework.test.context.support.AbstractTestContextBootstrapper.buildMergedContextConfiguration(AbstractTestContextBootstrapper.java:392)
	at org.springframework.test.context.support.AbstractTestContextBootstrapper.buildDefaultMergedContextConfiguration(AbstractTestContextBootstrapper.java:309)
	at org.springframework.test.context.support.AbstractTestContextBootstrapper.buildMergedContextConfiguration(AbstractTestContextBootstrapper.java:262)
	at org.springframework.test.context.support.AbstractTestContextBootstrapper.buildTestContext(AbstractTestContextBootstrapper.java:107)
	at org.springframework.boot.test.context.SpringBootTestContextBootstrapper.buildTestContext(SpringBootTestContextBootstrapper.java:102)
	at org.springframework.test.context.TestContextManager.<init>(TestContextManager.java:137)
	at org.springframework.test.context.TestContextManager.<init>(TestContextManager.java:122)
	at org.junit.jupiter.engine.execution.ExtensionValuesStore.lambda$getOrComputeIfAbsent$4(ExtensionValuesStore.java:86)
	at org.junit.jupiter.engine.execution.ExtensionValuesStore$MemoizingSupplier.computeValue(ExtensionValuesStore.java:223)
	at org.junit.jupiter.engine.execution.ExtensionValuesStore$MemoizingSupplier.get(ExtensionValuesStore.java:211)
	at org.junit.jupiter.engine.execution.ExtensionValuesStore$StoredValue.evaluate(ExtensionValuesStore.java:191)
	at org.junit.jupiter.engine.execution.ExtensionValuesStore$StoredValue.access$100(ExtensionValuesStore.java:171)
	at org.junit.jupiter.engine.execution.ExtensionValuesStore.getOrComputeIfAbsent(ExtensionValuesStore.java:89)
	at org.junit.jupiter.engine.execution.ExtensionValuesStore.getOrComputeIfAbsent(ExtensionValuesStore.java:93)
	at org.junit.jupiter.engine.execution.NamespaceAwareStore.getOrComputeIfAbsent(NamespaceAwareStore.java:61)
	at org.springframework.test.context.junit.jupiter.SpringExtension.getTestContextManager(SpringExtension.java:294)
	at org.springframework.test.context.junit.jupiter.SpringExtension.beforeAll(SpringExtension.java:113)
	at org.junit.jupiter.engine.descriptor.ClassBasedTestDescriptor.lambda$invokeBeforeAllCallbacks$8(ClassBasedTestDescriptor.java:368)
	at org.junit.platform.engine.support.hierarchical.ThrowableCollector.execute(ThrowableCollector.java:73)
	at org.junit.jupiter.engine.descriptor.ClassBasedTestDescriptor.invokeBeforeAllCallbacks(ClassBasedTestDescriptor.java:368)
	at org.junit.jupiter.engine.descriptor.ClassBasedTestDescriptor.before(ClassBasedTestDescriptor.java:192)
	at org.junit.jupiter.engine.descriptor.ClassBasedTestDescriptor.before(ClassBasedTestDescriptor.java:78)
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.lambda$executeRecursively$5(NodeTestTask.java:136)
	at org.junit.platform.engine.support.hierarchical.ThrowableCollector.execute(ThrowableCollector.java:73)
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.lambda$executeRecursively$7(NodeTestTask.java:129)
	at org.junit.platform.engine.support.hierarchical.Node.around(Node.java:137)
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.lambda$executeRecursively$8(NodeTestTask.java:127)
	at org.junit.platform.engine.support.hierarchical.ThrowableCollector.execute(ThrowableCollector.java:73)
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.executeRecursively(NodeTestTask.java:126)
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.execute(NodeTestTask.java:84)
	at java.util.ArrayList.forEach(ArrayList.java:1259)
	at org.junit.platform.engine.support.hierarchical.SameThreadHierarchicalTestExecutorService.invokeAll(SameThreadHierarchicalTestExecutorService.java:38)
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.lambda$executeRecursively$5(NodeTestTask.java:143)
	at org.junit.platform.engine.support.hierarchical.ThrowableCollector.execute(ThrowableCollector.java:73)
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.lambda$executeRecursively$7(NodeTestTask.java:129)
	at org.junit.platform.engine.support.hierarchical.Node.around(Node.java:137)
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.lambda$executeRecursively$8(NodeTestTask.java:127)
	at org.junit.platform.engine.support.hierarchical.ThrowableCollector.execute(ThrowableCollector.java:73)
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.executeRecursively(NodeTestTask.java:126)
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.execute(NodeTestTask.java:84)
	at org.junit.platform.engine.support.hierarchical.SameThreadHierarchicalTestExecutorService.submit(SameThreadHierarchicalTestExecutorService.java:32)
	at org.junit.platform.engine.support.hierarchical.HierarchicalTestExecutor.execute(HierarchicalTestExecutor.java:57)
	at org.junit.platform.engine.support.hierarchical.HierarchicalTestEngine.execute(HierarchicalTestEngine.java:51)
	at org.junit.platform.launcher.core.EngineExecutionOrchestrator.execute(EngineExecutionOrchestrator.java:108)
	at org.junit.platform.launcher.core.EngineExecutionOrchestrator.execute(EngineExecutionOrchestrator.java:88)
	at org.junit.platform.launcher.core.EngineExecutionOrchestrator.lambda$execute$0(EngineExecutionOrchestrator.java:54)
	at org.junit.platform.launcher.core.EngineExecutionOrchestrator.withInterceptedStreams(EngineExecutionOrchestrator.java:67)
	at org.junit.platform.launcher.core.EngineExecutionOrchestrator.execute(EngineExecutionOrchestrator.java:52)
	at org.junit.platform.launcher.core.DefaultLauncher.execute(DefaultLauncher.java:96)
	at org.junit.platform.launcher.core.DefaultLauncher.execute(DefaultLauncher.java:75)
	at com.intellij.junit5.JUnit5IdeaTestRunner.startRunnerWithArgs(JUnit5IdeaTestRunner.java:71)
	at com.intellij.rt.junit.IdeaTestRunner$Repeater.startRunnerWithArgs(IdeaTestRunner.java:33)
	at com.intellij.rt.junit.JUnitStarter.prepareStreamsAndStart(JUnitStarter.java:220)
	at com.intellij.rt.junit.JUnitStarter.main(JUnitStarter.java:53)


```



出现这种现象的原因是 SpringBoot 没有找到主启动类的位置，解决方案是在 `@SpringBootTest` 注解上显式声明主启动类的位置，声明完成后，再执行即可正常运行单元测试。

```java
@SpringBootTest(classes = AppJuint.class)
public class OuterTest {

    @Test
    public void test4() throws Exception {
        System.out.println("Outer test run ......");
    }

}
```

### 3.4 JUnit的断言机制

下面讲解 JUnit 5 中的经典使用方式：断言。在 JUnit 4 中我们使用 `Assert` 类进行断言，而到了 JUnit 5 中使用的类是 **`Assertions`** ，类名变了，使用方式却大差不差，下面通过几个简单示例讲解 JUnit 5 的断言使用。

#### 3.5.1 基础的简单断言

Assertions 提供的最简单的断言方法，包含比对两个值是否相等、两个对象是否是同一个、对象是否为 null ，以及全场景通用的判断表达式的值为 true / false 。下面是一个简单的使用示例。

```java
@Test
public void testSimple() {
    // 最简单的断言，断言计算值与预期值是否相等
    int num = 3 + 5;
    Assertions.assertEquals(num, 8);

    double result = 10.0 / 3;
    // 断言计算值是否在浮点数的指定范围内上下浮动
    Assertions.assertEquals(result, 3, 0.5);
    // 如果浮动空间不够，则会断言失败
    // Assertions.assertEquals(result, 3, 0.2);
    // 传入message可以自定义错误提示信息
    Assertions.assertEquals(result, 3, 0.2, "计算数值偏差较大！");

    // 断言两个对象是否是同一个
    Object o1 = new Object();
    Object o2 = o1;
    Object o3 = new Object();
    Assertions.assertSame(o1, o2);
    Assertions.assertSame(o1, o3);

    // 断言两个数组的元素是否完全相同
    String[] arr1 = {"aa", "bb"};
    String[] arr2 = {"aa", "bb"};
    String[] arr3 = {"bb", "aa"};
    Assertions.assertArrayEquals(arr1, arr2);
    Assertions.assertArrayEquals(arr1, arr3);
}

```

#### 3.5.2 组合条件断言

组合条件断言，实际上是要在一条断言中组合多个断言，要求这些断言同时、全部通过，则外部的组合断言才能通过。这种设计有点类似于父子断言，下面通过一个简单示例演示。

```java
@Test
public void testCombination() {
    Assertions.assertAll(
        () -> {
            int num = 3 + 5;
            Assertions.assertEquals(num, 8);
        },
        () -> {
            String[] arr1 = {"aa", "bb"};
            String[] arr2 = {"bb", "aa"};
            Assertions.assertArrayEquals(arr1, arr2);
        }
    );
}
```

运行该单元测试，结果必然是失败的，不过请各位观察控制台的打印，这里面提示了组合断言的失败个数为 1 个，这是组合断言的一个小特征。

```

org.opentest4j.AssertionFailedError: array contents differ at index [0], expected: <aa> but was: <bb>

	at org.junit.jupiter.api.AssertionUtils.fail(AssertionUtils.java:39)
	at org.junit.jupiter.api.AssertArrayEquals.failArraysNotEqual(AssertArrayEquals.java:432)
	at org.junit.jupiter.api.AssertArrayEquals.assertArrayElementsEqual(AssertArrayEquals.java:392)
	at org.junit.jupiter.api.AssertArrayEquals.assertArrayEquals(AssertArrayEquals.java:349)
	at org.junit.jupiter.api.AssertArrayEquals.assertArrayEquals(AssertArrayEquals.java:162)
	at org.junit.jupiter.api.AssertArrayEquals.assertArrayEquals(AssertArrayEquals.java:158)
	at org.junit.jupiter.api.Assertions.assertArrayEquals(Assertions.java:1435)
	at demo.AssertTest.lambda$testCombination$1(AssertTest.java:55)
	at org.junit.jupiter.api.AssertAll.lambda$assertAll$1(AssertAll.java:68)
	at java.util.stream.ReferencePipeline$3$1.accept(ReferencePipeline.java:193)
	at java.util.stream.ReferencePipeline$11$1.accept(ReferencePipeline.java:373)
	at java.util.Spliterators$ArraySpliterator.forEachRemaining(Spliterators.java:948)
	at java.util.stream.AbstractPipeline.copyInto(AbstractPipeline.java:482)
	at java.util.stream.AbstractPipeline.wrapAndCopyInto(AbstractPipeline.java:472)
	at java.util.stream.ReduceOps$ReduceOp.evaluateSequential(ReduceOps.java:708)
	at java.util.stream.AbstractPipeline.evaluate(AbstractPipeline.java:234)
	at java.util.stream.ReferencePipeline.collect(ReferencePipeline.java:499)
	at org.junit.jupiter.api.AssertAll.assertAll(AssertAll.java:77)
	at org.junit.jupiter.api.AssertAll.assertAll(AssertAll.java:44)
	at org.junit.jupiter.api.AssertAll.assertAll(AssertAll.java:38)
	at org.junit.jupiter.api.Assertions.assertAll(Assertions.java:2894)
	at demo.AssertTest.testCombination(AssertTest.java:47)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at org.junit.platform.commons.util.ReflectionUtils.invokeMethod(ReflectionUtils.java:688)
	at org.junit.jupiter.engine.execution.MethodInvocation.proceed(MethodInvocation.java:60)
	at org.junit.jupiter.engine.execution.InvocationInterceptorChain$ValidatingInvocation.proceed(InvocationInterceptorChain.java:131)
	at org.junit.jupiter.engine.extension.TimeoutExtension.intercept(TimeoutExtension.java:149)
	at org.junit.jupiter.engine.extension.TimeoutExtension.interceptTestableMethod(TimeoutExtension.java:140)
	at org.junit.jupiter.engine.extension.TimeoutExtension.interceptTestMethod(TimeoutExtension.java:84)
	at org.junit.jupiter.engine.execution.ExecutableInvoker$ReflectiveInterceptorCall.lambda$ofVoidMethod$0(ExecutableInvoker.java:115)
	at org.junit.jupiter.engine.execution.ExecutableInvoker.lambda$invoke$0(ExecutableInvoker.java:105)
	at org.junit.jupiter.engine.execution.InvocationInterceptorChain$InterceptedInvocation.proceed(InvocationInterceptorChain.java:106)
	at org.junit.jupiter.engine.execution.InvocationInterceptorChain.proceed(InvocationInterceptorChain.java:64)
	at org.junit.jupiter.engine.execution.InvocationInterceptorChain.chainAndInvoke(InvocationInterceptorChain.java:45)
	at org.junit.jupiter.engine.execution.InvocationInterceptorChain.invoke(InvocationInterceptorChain.java:37)
	at org.junit.jupiter.engine.execution.ExecutableInvoker.invoke(ExecutableInvoker.java:104)
	at org.junit.jupiter.engine.execution.ExecutableInvoker.invoke(ExecutableInvoker.java:98)
	at org.junit.jupiter.engine.descriptor.TestMethodTestDescriptor.lambda$invokeTestMethod$6(TestMethodTestDescriptor.java:210)
	at org.junit.platform.engine.support.hierarchical.ThrowableCollector.execute(ThrowableCollector.java:73)
	at org.junit.jupiter.engine.descriptor.TestMethodTestDescriptor.invokeTestMethod(TestMethodTestDescriptor.java:206)
	at org.junit.jupiter.engine.descriptor.TestMethodTestDescriptor.execute(TestMethodTestDescriptor.java:131)
	at org.junit.jupiter.engine.descriptor.TestMethodTestDescriptor.execute(TestMethodTestDescriptor.java:65)
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.lambda$executeRecursively$5(NodeTestTask.java:139)
	at org.junit.platform.engine.support.hierarchical.ThrowableCollector.execute(ThrowableCollector.java:73)
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.lambda$executeRecursively$7(NodeTestTask.java:129)
	at org.junit.platform.engine.support.hierarchical.Node.around(Node.java:137)
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.lambda$executeRecursively$8(NodeTestTask.java:127)
	at org.junit.platform.engine.support.hierarchical.ThrowableCollector.execute(ThrowableCollector.java:73)
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.executeRecursively(NodeTestTask.java:126)
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.execute(NodeTestTask.java:84)
	at java.util.ArrayList.forEach(ArrayList.java:1259)
	at org.junit.platform.engine.support.hierarchical.SameThreadHierarchicalTestExecutorService.invokeAll(SameThreadHierarchicalTestExecutorService.java:38)
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.lambda$executeRecursively$5(NodeTestTask.java:143)
	at org.junit.platform.engine.support.hierarchical.ThrowableCollector.execute(ThrowableCollector.java:73)
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.lambda$executeRecursively$7(NodeTestTask.java:129)
	at org.junit.platform.engine.support.hierarchical.Node.around(Node.java:137)
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.lambda$executeRecursively$8(NodeTestTask.java:127)
	at org.junit.platform.engine.support.hierarchical.ThrowableCollector.execute(ThrowableCollector.java:73)
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.executeRecursively(NodeTestTask.java:126)
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.execute(NodeTestTask.java:84)
	at java.util.ArrayList.forEach(ArrayList.java:1259)
	at org.junit.platform.engine.support.hierarchical.SameThreadHierarchicalTestExecutorService.invokeAll(SameThreadHierarchicalTestExecutorService.java:38)
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.lambda$executeRecursively$5(NodeTestTask.java:143)
	at org.junit.platform.engine.support.hierarchical.ThrowableCollector.execute(ThrowableCollector.java:73)
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.lambda$executeRecursively$7(NodeTestTask.java:129)
	at org.junit.platform.engine.support.hierarchical.Node.around(Node.java:137)
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.lambda$executeRecursively$8(NodeTestTask.java:127)
	at org.junit.platform.engine.support.hierarchical.ThrowableCollector.execute(ThrowableCollector.java:73)
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.executeRecursively(NodeTestTask.java:126)
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.execute(NodeTestTask.java:84)
	at org.junit.platform.engine.support.hierarchical.SameThreadHierarchicalTestExecutorService.submit(SameThreadHierarchicalTestExecutorService.java:32)
	at org.junit.platform.engine.support.hierarchical.HierarchicalTestExecutor.execute(HierarchicalTestExecutor.java:57)
	at org.junit.platform.engine.support.hierarchical.HierarchicalTestEngine.execute(HierarchicalTestEngine.java:51)
	at org.junit.platform.launcher.core.EngineExecutionOrchestrator.execute(EngineExecutionOrchestrator.java:108)
	at org.junit.platform.launcher.core.EngineExecutionOrchestrator.execute(EngineExecutionOrchestrator.java:88)
	at org.junit.platform.launcher.core.EngineExecutionOrchestrator.lambda$execute$0(EngineExecutionOrchestrator.java:54)
	at org.junit.platform.launcher.core.EngineExecutionOrchestrator.withInterceptedStreams(EngineExecutionOrchestrator.java:67)
	at org.junit.platform.launcher.core.EngineExecutionOrchestrator.execute(EngineExecutionOrchestrator.java:52)
	at org.junit.platform.launcher.core.DefaultLauncher.execute(DefaultLauncher.java:96)
	at org.junit.platform.launcher.core.DefaultLauncher.execute(DefaultLauncher.java:75)
	at com.intellij.junit5.JUnit5IdeaTestRunner.startRunnerWithArgs(JUnit5IdeaTestRunner.java:71)
	at com.intellij.rt.junit.IdeaTestRunner$Repeater.startRunnerWithArgs(IdeaTestRunner.java:33)
	at com.intellij.rt.junit.JUnitStarter.prepareStreamsAndStart(JUnitStarter.java:220)
	at com.intellij.rt.junit.JUnitStarter.main(JUnitStarter.java:53)


org.opentest4j.MultipleFailuresError: Multiple Failures (1 failure)
	org.opentest4j.AssertionFailedError: array contents differ at index [0], expected: <aa> but was: <bb>

	at org.junit.jupiter.api.AssertAll.assertAll(AssertAll.java:80)
	at org.junit.jupiter.api.AssertAll.assertAll(AssertAll.java:44)
	at org.junit.jupiter.api.AssertAll.assertAll(AssertAll.java:38)
	at org.junit.jupiter.api.Assertions.assertAll(Assertions.java:2894)
	at demo.AssertTest.testCombination(AssertTest.java:47)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at org.junit.platform.commons.util.ReflectionUtils.invokeMethod(ReflectionUtils.java:688)
	at org.junit.jupiter.engine.execution.MethodInvocation.proceed(MethodInvocation.java:60)
	at org.junit.jupiter.engine.execution.InvocationInterceptorChain$ValidatingInvocation.proceed(InvocationInterceptorChain.java:131)
	at org.junit.jupiter.engine.extension.TimeoutExtension.intercept(TimeoutExtension.java:149)
	at org.junit.jupiter.engine.extension.TimeoutExtension.interceptTestableMethod(TimeoutExtension.java:140)
	at org.junit.jupiter.engine.extension.TimeoutExtension.interceptTestMethod(TimeoutExtension.java:84)
	at org.junit.jupiter.engine.execution.ExecutableInvoker$ReflectiveInterceptorCall.lambda$ofVoidMethod$0(ExecutableInvoker.java:115)
	at org.junit.jupiter.engine.execution.ExecutableInvoker.lambda$invoke$0(ExecutableInvoker.java:105)
	at org.junit.jupiter.engine.execution.InvocationInterceptorChain$InterceptedInvocation.proceed(InvocationInterceptorChain.java:106)
	at org.junit.jupiter.engine.execution.InvocationInterceptorChain.proceed(InvocationInterceptorChain.java:64)
	at org.junit.jupiter.engine.execution.InvocationInterceptorChain.chainAndInvoke(InvocationInterceptorChain.java:45)
	at org.junit.jupiter.engine.execution.InvocationInterceptorChain.invoke(InvocationInterceptorChain.java:37)
	at org.junit.jupiter.engine.execution.ExecutableInvoker.invoke(ExecutableInvoker.java:104)
	at org.junit.jupiter.engine.execution.ExecutableInvoker.invoke(ExecutableInvoker.java:98)
	at org.junit.jupiter.engine.descriptor.TestMethodTestDescriptor.lambda$invokeTestMethod$6(TestMethodTestDescriptor.java:210)
	at org.junit.platform.engine.support.hierarchical.ThrowableCollector.execute(ThrowableCollector.java:73)
	at org.junit.jupiter.engine.descriptor.TestMethodTestDescriptor.invokeTestMethod(TestMethodTestDescriptor.java:206)
	at org.junit.jupiter.engine.descriptor.TestMethodTestDescriptor.execute(TestMethodTestDescriptor.java:131)
	at org.junit.jupiter.engine.descriptor.TestMethodTestDescriptor.execute(TestMethodTestDescriptor.java:65)
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.lambda$executeRecursively$5(NodeTestTask.java:139)
	at org.junit.platform.engine.support.hierarchical.ThrowableCollector.execute(ThrowableCollector.java:73)
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.lambda$executeRecursively$7(NodeTestTask.java:129)
	at org.junit.platform.engine.support.hierarchical.Node.around(Node.java:137)
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.lambda$executeRecursively$8(NodeTestTask.java:127)
	at org.junit.platform.engine.support.hierarchical.ThrowableCollector.execute(ThrowableCollector.java:73)
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.executeRecursively(NodeTestTask.java:126)
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.execute(NodeTestTask.java:84)
	at java.util.ArrayList.forEach(ArrayList.java:1259)
	at org.junit.platform.engine.support.hierarchical.SameThreadHierarchicalTestExecutorService.invokeAll(SameThreadHierarchicalTestExecutorService.java:38)
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.lambda$executeRecursively$5(NodeTestTask.java:143)
	at org.junit.platform.engine.support.hierarchical.ThrowableCollector.execute(ThrowableCollector.java:73)
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.lambda$executeRecursively$7(NodeTestTask.java:129)
	at org.junit.platform.engine.support.hierarchical.Node.around(Node.java:137)
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.lambda$executeRecursively$8(NodeTestTask.java:127)
	at org.junit.platform.engine.support.hierarchical.ThrowableCollector.execute(ThrowableCollector.java:73)
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.executeRecursively(NodeTestTask.java:126)
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.execute(NodeTestTask.java:84)
	at java.util.ArrayList.forEach(ArrayList.java:1259)
	at org.junit.platform.engine.support.hierarchical.SameThreadHierarchicalTestExecutorService.invokeAll(SameThreadHierarchicalTestExecutorService.java:38)
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.lambda$executeRecursively$5(NodeTestTask.java:143)
	at org.junit.platform.engine.support.hierarchical.ThrowableCollector.execute(ThrowableCollector.java:73)
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.lambda$executeRecursively$7(NodeTestTask.java:129)
	at org.junit.platform.engine.support.hierarchical.Node.around(Node.java:137)
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.lambda$executeRecursively$8(NodeTestTask.java:127)
	at org.junit.platform.engine.support.hierarchical.ThrowableCollector.execute(ThrowableCollector.java:73)
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.executeRecursively(NodeTestTask.java:126)
	at org.junit.platform.engine.support.hierarchical.NodeTestTask.execute(NodeTestTask.java:84)
	at org.junit.platform.engine.support.hierarchical.SameThreadHierarchicalTestExecutorService.submit(SameThreadHierarchicalTestExecutorService.java:32)
	at org.junit.platform.engine.support.hierarchical.HierarchicalTestExecutor.execute(HierarchicalTestExecutor.java:57)
	at org.junit.platform.engine.support.hierarchical.HierarchicalTestEngine.execute(HierarchicalTestEngine.java:51)
	at org.junit.platform.launcher.core.EngineExecutionOrchestrator.execute(EngineExecutionOrchestrator.java:108)
	at org.junit.platform.launcher.core.EngineExecutionOrchestrator.execute(EngineExecutionOrchestrator.java:88)
	at org.junit.platform.launcher.core.EngineExecutionOrchestrator.lambda$execute$0(EngineExecutionOrchestrator.java:54)
	at org.junit.platform.launcher.core.EngineExecutionOrchestrator.withInterceptedStreams(EngineExecutionOrchestrator.java:67)
	at org.junit.platform.launcher.core.EngineExecutionOrchestrator.execute(EngineExecutionOrchestrator.java:52)
	at org.junit.platform.launcher.core.DefaultLauncher.execute(DefaultLauncher.java:96)
	at org.junit.platform.launcher.core.DefaultLauncher.execute(DefaultLauncher.java:75)
	at com.intellij.junit5.JUnit5IdeaTestRunner.startRunnerWithArgs(JUnit5IdeaTestRunner.java:71)
	at com.intellij.rt.junit.IdeaTestRunner$Repeater.startRunnerWithArgs(IdeaTestRunner.java:33)
	at com.intellij.rt.junit.JUnitStarter.prepareStreamsAndStart(JUnitStarter.java:220)
	at com.intellij.rt.junit.JUnitStarter.main(JUnitStarter.java:53)
	Suppressed: org.opentest4j.AssertionFailedError: array contents differ at index [0], expected: <aa> but was: <bb>
		at org.junit.jupiter.api.AssertionUtils.fail(AssertionUtils.java:39)
		at org.junit.jupiter.api.AssertArrayEquals.failArraysNotEqual(AssertArrayEquals.java:432)
		at org.junit.jupiter.api.AssertArrayEquals.assertArrayElementsEqual(AssertArrayEquals.java:392)
		at org.junit.jupiter.api.AssertArrayEquals.assertArrayEquals(AssertArrayEquals.java:349)
		at org.junit.jupiter.api.AssertArrayEquals.assertArrayEquals(AssertArrayEquals.java:162)
		at org.junit.jupiter.api.AssertArrayEquals.assertArrayEquals(AssertArrayEquals.java:158)
		at org.junit.jupiter.api.Assertions.assertArrayEquals(Assertions.java:1435)
		at demo.AssertTest.lambda$testCombination$1(AssertTest.java:55)
		at org.junit.jupiter.api.AssertAll.lambda$assertAll$1(AssertAll.java:68)
		at java.util.stream.ReferencePipeline$3$1.accept(ReferencePipeline.java:193)
		at java.util.stream.ReferencePipeline$11$1.accept(ReferencePipeline.java:373)
		at java.util.Spliterators$ArraySpliterator.forEachRemaining(Spliterators.java:948)
		at java.util.stream.AbstractPipeline.copyInto(AbstractPipeline.java:482)
		at java.util.stream.AbstractPipeline.wrapAndCopyInto(AbstractPipeline.java:472)
		at java.util.stream.ReduceOps$ReduceOp.evaluateSequential(ReduceOps.java:708)
		at java.util.stream.AbstractPipeline.evaluate(AbstractPipeline.java:234)
		at java.util.stream.ReferencePipeline.collect(ReferencePipeline.java:499)
		at org.junit.jupiter.api.AssertAll.assertAll(AssertAll.java:77)
		... 69 more
```

#### 3.5.3 异常抛出断言

异常抛出的断言，指的是被测试的内容最终运行时必定会抛出一个异常，如果没有抛出异常则断言失败。换句话说，编写异常抛出断言时，我期望你这段代码一定会出现异常，有异常是对的，没有异常反而有问题。以下是一个示例。

```java
@Test
public void testException() {
    Assertions.assertThrows(ArithmeticException.class, () -> {
        int i = 1 / 0;
    });
}
```

在上述测试代码中，lambda 表达式中编写的代码一定会抛出算术异常（即 `ArithmeticException` ），但我们断言时就指定了你会抛出这个异常，所以这个断言反而是运行通过的。如果 lambda 表达式中运行正常了，没有抛出异常，那才是奇怪的，这种情况下就要断言失败了（明明你要出问题，结果你反而正常了 ？？？）。

#### 3.5.4 执行超时断言

执行超时断言是，针对的是被测试代码的执行速度。有关超时的概念想必不用多解释各位也都明白，下面提供一个简单示例演示超时断言的使用。

```java
@Test
public void testTimeout() {
    Assertions.assertTimeout(Duration.ofMillis(500), () -> {
        System.out.println("testTimeout run ......");
        TimeUnit.SECONDS.sleep(1);
        System.out.println("testTimeout finished ......");
    });
}
```

```
testTimeout run ......
testTimeout finished ......

org.opentest4j.AssertionFailedError: execution exceeded timeout of 500 ms by 503 ms
```

在上述测试代码中，我们指定了被测试逻辑需要在 500 ms 之内运行完毕，但是测试代码中有一个 1 秒的线程睡眠，则最终一定会因为执行超时而抛出异常。

#### 3.5.5 强制失败

强制失败，它的使用方式比较类似于最原始的抛出异常的方式，我们先看一个简单示例。

```java
@Test
public void testFail() {
    if (ZonedDateTime.now().getHour() > 12) {
        Assertions.fail();
    }
}
```

这种写法是不是类似于 我们在业务逻辑编写时，如果需要一些校验逻辑，则可能会有以下的代码：

```java
public void service(String name) {
    if (StringUtils.isBlank(name)) {
        throw new ServiceException("name is blank !");
    }
    ......
}
```

上下代码对比之后各位是否可以发现，写法基本是一样的，这就是强制失败的断言使用，也是非常简单的。

### 3.5 前置条件检查机制

前置条件的检查机制，同样应用在断言的场景中，它指的是：如果一个单元测试的前置条件不满足，则当前的测试会**被跳过**，后续的测试不会执行。使用前置条件检查机制，可以避免一些无谓的测试逻辑执行，从而提高单元测试的执行效率。

前置条件的检查使用的 API 是 `Assumptions` ，下面简单看几个使用方式。

```java
@Test
public void testAssumptions() throws Exception {
    int num = 3 + 5;
    Assumptions.assumeTrue(num < 10);

    Assertions.assertTrue(10 - num > 0);
}
```

在这个单元测试中，num 是通过一个业务逻辑计算出的值（这里为了演示没有编写复杂的业务逻辑），在前置条件检查时，它会先判断 num 的值是否小于 10 ，如果检查通过，则会执行后续的断言逻辑；而如果检查不通过，则这个单元测试会被跳过（ ignore ），后续的断言逻辑不会执行。

### 3.6 高级特性：嵌套测试

嵌套测试是 JUnit 5 的一个高级特性，它支持我们在编写单元测试类时，以内部类的方式组织一些有关联的测试逻辑。有关嵌套测试的演示代码，在 JUnit 5 的官方文档中提供了一个[非常全面的示例](https://junit.org/junit5/docs/current/user-guide/#writing-tests-nested)，小册将这个测试代码同步拉取了下来，并同步到了代码仓库中。以下是该示例代码的核心部分：

```java
package demo;

import java.util.EmptyStackException;
import java.util.Stack;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.assertThrows;
import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertFalse;
import static org.junit.jupiter.api.Assertions.assertTrue;


@DisplayName("A stack")
class TestingAStackDemo {
    
    Stack<Object> stack;
    
    @Test
    @DisplayName("is instantiated with new Stack()")
    void isInstantiatedWithNew() {
        new Stack<>();
    }
    
    @Nested
    @DisplayName("when new")
    class WhenNew {
        
        @BeforeEach
        void createNewStack() {
            stack = new Stack<>();
        }
        
        @Test
        @DisplayName("is empty")
        void isEmpty() {
            assertTrue(stack.isEmpty());
        }
        
        @Test
        @DisplayName("throws EmptyStackException when popped")
        void throwsExceptionWhenPopped() {
            assertThrows(EmptyStackException.class, () -> stack.pop());
        }
        
        @Test
        @DisplayName("throws EmptyStackException when peeked")
        void throwsExceptionWhenPeeked() {
            assertThrows(EmptyStackException.class, () -> stack.peek());
        }
        
        @Nested
        @DisplayName("after pushing an element")
        class AfterPushing {
            
            String anElement = "an element";
            
            @BeforeEach
            void pushAnElement() {
                stack.push(anElement);
            }
            
            @Test
            @DisplayName("it is no longer empty")
            void isNotEmpty() {
                assertFalse(stack.isEmpty());
            }
            
            @Test
            @DisplayName("returns the element when popped and is empty")
            void returnElementWhenPopped() {
                assertEquals(anElement, stack.pop());
                assertTrue(stack.isEmpty());
            }
            
            @Test
            @DisplayName("returns the element when peeked but remains not empty")
            void returnElementWhenPeeked() {
                assertEquals(anElement, stack.peek());
                assertFalse(stack.isEmpty());
            }
        }
    }
}

```

官方提供的测试代码都是可以执行通过的，从这段测试代码中需要各位了解的几个关键特性：

- 单元测试类可以通过编写内部类，并标注 `@Nested` 注解，表明内部类也是一个单元测试类；
- 内部的单元测试类可以直接使用外部的成员属性，且可以利用外部定义的 `@BeforeEach` 、`@BeforeAll` 、`@AfterEach` 、`@AfterAll` 等前后置逻辑注解标注的方法；
- 外部的单元测试无法利用内部类定义的前后置逻辑注解。

具体的测试代码执行，各位可以借助 IDE 自行测试，小册不再展开啰嗦。

### 3.7 带入参的单元测试

JUnit 5 中的一个重要特性是支持单元测试方法的参数依赖注入，这也打破了我们已有的认知。通常我们编写的测试方法是不能有方法入参的，但是 JUnit 5 允许我们在编写单元测试方法中予以声明方法入参。默认情况下 JUnit 5 支持以下几个参数类型的依赖注入：

- `TestInfo` ：内部组装了当前单元测试所属的 Class 、Method ，以及对应的展示名（DisplayName）等；
- `RepetitionInfo` ：如果一个方法被标注了 `@RepeatedTest` ，或者该方法是一个 `@BeforeEach` / `@AfterEach` 方法，则可以拿到 `RepetitionInfo` 的信息，可以通过 `RepetitionInfo` 获取到当前重复信息以及相应的`@RepeatedTest`的重复总数；
- `TestReporter` ：注入 `TestReporter` 后可以获得数据发布能力，可以向测试结果中注册一些特殊的数据，这些数据可以被 `TestExecutionListener` 获取到。

具体的使用方式，各位可以参照 JUnit 5 的官方文档，小册不对此展开（本知识点是对下一个知识点的引导）。

### 3.8 高级特性：参数化测试

了解了 JUnit 5 的单元测试方法可以使用方法入参来注入数据，本章的最后一节我们来了解另一个重要的高级特性：参数化测试。

参数化测试是 JUnit 5 中提高单元测试效率的重要手段，它通过给单元测试方法传入特定的参数，可以使得 JUnit 在执行单元测试时逐个参数来检验和测试，这样做的好处是更加规整和高效地执行单元测试。

参数化测试支持我们使用如下的方式赋予参数：

- 基本类型：8 种基本数据类型 + `String` + `Class` ；
- 枚举类型：自定义的枚举；
- CSV 文件：可传入一个 CSV 格式的表格文件，使用表格文件中的数据作为入参；
- 方法的数据返回：可以通过一个方法返回需要测试入参的数据（流的形式返回）。

下面通过几个示例来讲解参数化测试的使用。

#### 3.8.1 基本类型的参数化测试

我们先看一下如何利用简单的基本类型完成参数化测试。在使用参数化测试时，标注的注解不再是 `@Test` ，取而代之的是 `@ParameterizedTest` ；另外还需要声明指定需要传入的数据，对于简单的基本类型而言，使用 `@ValueSource` 注解即可指定。以下是一个简单的示例代码：

```java
@ParameterizedTest
@ValueSource(strings = {"aa", "bb", "cc"})
public void testSimpleParameterized(String value) throws Exception {
    System.out.println(value);
    Assertions.assertTrue(value.length() < 3);
}
```

上述的测试代码中手动声明了 3 个需要测试的值，执行以上单元测试，控制台会输出三个数值，并且测试窗口中会分别展示三个测试值的结果，如下图所示。

![](/img/202212/05springboot2.png)

#### 3.8.2 方法数据流返回的参数化测试

参数化测试最吸引人的点是可以引用一个方法来实现测试数据的参数化，既然可以在一个方法中构造单元测试的入参数据，那么完全可以从数据库 / 缓存等任意位置加载数据，并构造为流的形式返回。以此法编写的参数化测试具有极大的灵活度和自由度。下面通过一个示例简单了解。

```java
    @ParameterizedTest
    @MethodSource("dataProvider")
    public void testDataStreamParameterized(Integer value) throws Exception {
        System.out.println(value);
        Assertions.assertTrue(value < 10);
    }

    private static Stream<Integer> dataProvider() {
        return Stream.of(1, 2, 3, 4, 5);
    }
```

观察被引用的方法特征，它必须是一个静态方法，且方法的返回值是一个 `Stream` 对象，但是访问修饰符没有限制。示例代码中的数据构造是写死的，在实际的单元测试中可以使用各种各样的数据加载方式。

运行单元测试，控制台可以打印出 Stream 中封装的所有数据的测试结果与打印。

【本章内容回顾了单元测试 以及 Junit 相关的基本知识，随后使用 SpringBoot 简单整合了 JUnit 5 并简单测试了单元测试的效果，最后讲解了 JUnit 5 中的核心功能使用。下一章会针对 SpringBoot 测试类的核心注解 `@SpringBootTest` 予以详细剖析】

