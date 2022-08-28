---
title: 04jackson的JsonGenerator写JSON的细节。
toc: true
tags: java
categories: 
    - [java]
    - [jackson]
---

## 1.概述

一个框架/库好不好，不是看它的核心功能做得怎么样，而是非核心功能处理得如何。比如后台页面做得咋样？容错机制呢？定制化、可配置化，扩展性等等。

Jackson称得上优秀（甚至最佳）最主要是得益于它优秀的**module**模块化设计，在接触其之前，我们先完成本章节的内容：`JsonGenerator`写JSON的行为控制（配置）。

配置属于程序的一部分，它影响着程序执行的方方面面。`Spring`使用Environment/PropertySource管理配置，对应的在Jackson里会看到有很多Feature类来控制Jackson的读/写行为，均是使用enum枚举类型来管理。

<!--more-->

我们学会了如何使用JsonGenerator去写一个JSON，本文将来学习它的需要掌握的使用细节。同样的，为围绕着JsonGenerator展开。

## 2.JsonGenerator的Feature

它是JsonGenerator的一个**内部**枚举类，共10个枚举值：

```java
public enum Feature {

	// Low-level I/O
	AUTO_CLOSE_TARGET(true),
	AUTO_CLOSE_JSON_CONTENT(true),
	FLUSH_PASSED_TO_STREAM(true),

	// Quoting-related features
	@Deprecated
	QUOTE_FIELD_NAMES(true),
	@Deprecated
	QUOTE_NON_NUMERIC_NUMBERS(true),
	@Deprecated
	ESCAPE_NON_ASCII(false),
	@Deprecated
	WRITE_NUMBERS_AS_STRINGS(false),

	// Schema/Validity support features
	WRITE_BIGDECIMAL_AS_PLAIN(false),
	STRICT_DUPLICATE_DETECTION(false),
	IGNORE_UNKNOWN(false);
	
	...
}
```

> 小贴士：枚举值均为bool类型，括号内为默认值

这个Feature的每个枚举值都控制着`JsonGenerator`写JSON时的不同行为，并且可分为三大类

- Low-level I/O：底层I/O流相关。

> Jackson的流式API指的是I/O流，因此就涉及到关流、flush刷新流等操作

- Quoting-related：双引号””引用相关。

> JSON规范规定key都必须有双引号，但这对于某些场景下并不需要

- Schema/Validity support：约束/规范/校验相关。

> JSON作为K-V结构的数据，那么允许相同key出现吗？这便由这些特征去控制

### 1.AUTO_CLOSE_TARGET(true)

含义即为字面意：自动关闭目标（流）。

- true：调用`JsonGenerator#close()`便会自动关闭底层的I/O流，你无需再关心
- false：底层I/O流请手动关闭

自动关闭：

```java
@Test
public void test1() throws IOException {
    JsonFactory factory = new JsonFactory();
    try (JsonGenerator jg = factory.createGenerator(System.err, JsonEncoding.UTF8)) {
        // doSomething
    }
}
```

如果改为false：那么你就需要自己**手动去close**底层使用的OutputStream或者Writer。形如这样：

```java
@Test
public void test2() throws IOException {
    JsonFactory factory = new JsonFactory();
    try (PrintStream err = System.err; JsonGenerator jg = factory.createGenerator(err, JsonEncoding.UTF8)) {
        // 特征置为false 采用手动关流的方式
        jg.disable(JsonGenerator.Feature.AUTO_CLOSE_TARGET);

        // doSomething
    }
}
```

> 小贴士：例子均采用`try-with-resources`方式关流，所以并没有显示调用close()方法，

### 2.AUTO_CLOSE_JSON_CONTENT(true)

```java
private static void test1() throws IOException {

        JsonFactory factory = new JsonFactory();
        try (JsonGenerator jg = factory.createGenerator(System.err, JsonEncoding.UTF8)) {
            jg.writeStartObject();
            jg.writeFieldName("names");

            // 写数组
            jg.writeStartArray();
            jg.writeString("clx");
            jg.writeString("clxmm");
        }

}
```

输出

```json
{"names":["clx","clxmm"]}
```

wow，竟然输出一切正常。细心的你会发现，我的代码是缺胳膊少腿的：**不管是Object还是Array都只start了，并没有显示调用end进行闭合**。但是呢，结果却正常得很，这便是此Feature的作用了。

- true：自动补齐（闭合）`JsonToken#START_ARRAY`和`JsonToken#START_OBJECT`类型的内容
- false：啥都不做（不会主动抛错哦）

不过还是要啰嗦一句：虽然Jackson通过此Feature做了容错，但是自己在使用时，**请务必**显示书写闭合

### 3.FLUSH_PASSED_TO_STREAM(true)

在使用带有缓冲区的I/O写数据时，缺少“临门一脚”是初学者很容易犯的错误，比如下面这个例子：

```java
private static void test2() throws IOException {

        JsonFactory factory = new JsonFactory();
        JsonGenerator jg = factory.createGenerator(System.err, JsonEncoding.UTF8);

        jg.writeStartObject();
        jg.writeStringField("name","clxmm");
        jg.writeEndObject();

        // jg.flush();
        // jg.close();

    }
```

运行程序，**控制台没有任何输出**。把注释代码放开任何一行，再次运行程序，控制台正常输出：

```json
{"name":"clxmm"}
```

- true：当JsonGenerator调用close()/flush()方法时，自动强刷I/O流里面的数据
- false：请手动处理

#### 为何需要flush()？

对于此问题这里小科普一下。因为向磁盘、网络写入数据的时候，出于效率的考虑，**操作系统**（话外音：这是操作系统为之）并不是输出一个字节就立刻写入到文件或者发送到网络，而是把输出的字节先放到内存的一个缓冲区里（本质上就是一个byte[]数组），等到缓冲区写满了，再一次性写入文件或者网络。对于**很多IO设备**来说，一次写一个字节和一次写1000个字节，花费的时间几乎是完全一样的，所以OutputStream有个flush()方法，能**强制**把缓冲区内容输出。

> 小贴士：**InputStream**是没有flush()方法的哦

**通常情况下**，我们不需要调用这个flush()方法，因为缓冲区写满了，OutputStream会自动调用它，并且，在调用close()方法关闭OutputStream之前，也会自动调用flush()方法强制刷一次缓冲区。但是，在某些情况下，我们必须手动调用flush()方法，比如上例子，比如发IM消息…

### 4.~~QUOTE_FIELD_NAMES(true)~~

此属性自`2.10`版本后已过期，使用`JsonWriteFeature#QUOTE_FIELD_NAMES`代替，应用在JsonFactory上，后文详解

JSON对象字段名是否为使用””双引号括起来，这是JSON规范（RFC4627）规定的。

- true：字段名使用””括起来 -> 遵循JSON规范
- false：字段名**不**使用””括起来 -> **不**遵循JSON规范

```java
private static void test3() throws IOException {
        JsonFactory factory = new JsonFactory();
        try (JsonGenerator jg = factory.createGenerator(System.err, JsonEncoding.UTF8)) {
            // jg.disable(QUOTE_FIELD_NAMES);

            jg.writeStartObject();
            jg.writeStringField("name","clxmm");
            jg.writeEndObject();
        }
}
```

console

```json
{"name":"clxmm"}
```

99.99%的情况下我们不需要改变默认值。Jackson添加了**禁用引号**的功能以支持那非常不常见的情况，最常见的情况直接从Javascript中使用时可能会发生。

打开注释掉的语句，再次运行程序，输出：

```json
{name:"clxmm"}
```

### 5.~~QUOTE_NON_NUMERIC_NUMBERS(true)~~

> 此属性自`2.10`版本后已过期，使用`JsonWriteFeature#WRITE_NAN_AS_STRINGS`代替，应用在JsonFactory上，

这个特征挺有意思，看例子（以写Float为例）：

```java
private static void test4() throws IOException {
        JsonFactory factory = new JsonFactory();
        try (JsonGenerator jg = factory.createGenerator(System.err, JsonEncoding.UTF8)) {

            // jg.disable(JsonGenerator.Feature.QUOTE_NON_NUMERIC_NUMBERS);
          
            jg.writeNumber(0.9);
            jg.writeNumber(1.9);
            jg.writeNumber(Float.NaN);
            jg.writeNumber(Float.NEGATIVE_INFINITY);
            jg.writeNumber(Float.POSITIVE_INFINITY);

        }
    }
```

console

```
0.9 1.9 "NaN" "-Infinity" "Infinity"
```

**同为Float数字类型**，有的输出有””双引号包着，有的没有。放开注释的语句（禁用此特征），再次运行程序，输出：

```
0.9 1.9 NaN -Infinity Infinity
```

明显，如果你是这么输出为一个JSON的话，那它就会是**非法的JSON**，是不符合JSON标准的（因为像NaN、Infinity这种明显是字符串嘛，必须用””包起来才是合法的value值）。

由于JSON规范中对数字的严格定义，加上Java可能具有的**开放式数字集**（如上例中Float类型并不100%是数字），很难做到既安全又方便，因此有了此特征让你根据需要来控制。

### 6.~~ESCAPE_NON_ASCII(false)~~

> 此属性自`2.10`版本后已过期，使用`JsonWriteFeature#ESCAPE_NON_ASCII`代替，应用在JsonFactory上，

```java
    private static void test5() throws IOException {
        JsonFactory factory = new JsonFactory();
        try (JsonGenerator jg = factory.createGenerator(System.err, JsonEncoding.UTF8)) {
           //  jg.enable(ESCAPE_NON_ASCII);
            jg.writeString("A个");
        }
    }
```

console

```
"A个"
```

放开注掉的代码（开启此属性），再次运行，输出：

```
"A\u4E2A"
```

### 7.~~WRITE_NUMBERS_AS_STRINGS(false)~~

> 此属性自`2.10`版本后已过期，使用`JsonWriteFeature#WRITE_NUMBERS_AS_STRINGS`代替，应用在JsonFactory上

该特性**强制**将**所有**Java数字写成字符串，即使底层数据格式真的是数字。

- true：所有数字**强制**写为字符串
- false：不做处理

```java
private static void test6() throws IOException {
        JsonFactory factory = new JsonFactory();
        try (JsonGenerator jg = factory.createGenerator(System.err, JsonEncoding.UTF8)) {
            // jg.enable(WRITE_NUMBERS_AS_STRINGS);

            jg.writeNumber(Long.MAX_VALUE);
        }
}
```

consloe

```
9223372036854775807
```

放开注释代码（开启此特征），再次运行程序，输出：

```
"9223372036854775807"
```

有什么使用场景？一个用例是避免Javascript限制的问题：因为Javascript标准规定所有的数字处理都应该使用**64位ieee754浮点值**来完成，结果是一些64位整数值不能被精确表示（因为尾数只有51位宽）。

> 采坑提醒：时间戳后端用Long类型反给前端是没有问题的。但如果你是**很大的一个Long值**（如雪花算法算出的很大的Long值），直接返回前端的话，Javascript就会出现精度丢失的bug

### 8.WRITE_BIGDECIMAL_AS_PLAIN(false)

控制写`java.math.BigDecimal`的行为：

- true：使用`BigDecimal#toPlainString()`方法输出
- false： 使用默认输出方式（取决于BigDecimal是如何构造的）

```java
@Test
public void test7() throws IOException {
    JsonFactory factory = new JsonFactory();
    try (JsonGenerator jg = factory.createGenerator(System.err, JsonEncoding.UTF8)) {
        // jg.enable(JsonGenerator.Feature.WRITE_BIGDECIMAL_AS_PLAIN);

        BigDecimal bigDecimal1 = new BigDecimal(1.0);
        BigDecimal bigDecimal2 = new BigDecimal("1.0");
        BigDecimal bigDecimal3 = new BigDecimal("1E11");
        jg.writeNumber(bigDecimal1);
        jg.writeNumber(bigDecimal2);
        jg.writeNumber(bigDecimal3);
    }
}
```

console

```
1 1.0 1E+11
```

放开注释代码，再次运行程序，输出：

```
1 1.0 100000000000
```

### 9.STRICT_DUPLICATE_DETECTION(false)

是否去严格的检测重复属性名。

- true：检测是否有重复字段名，若有，则抛出`JsonParseException`异常
- false：不检测JSON对象重复的字段名，即：相同字段名都要解析

```java
private static void test7() throws IOException {
        JsonFactory factory = new JsonFactory();
        try (JsonGenerator jg = factory.createGenerator(System.err, JsonEncoding.UTF8)) {
            // jg.enable(JsonGenerator.Feature.STRICT_DUPLICATE_DETECTION);
            jg.writeStartObject();
            jg.writeStringField("name","clx");
            jg.writeStringField("name","clxmm");
            jg.writeEndObject();
        }

}
```

console

```json
{"name":"clx","name":"clxmm"}
```

打开注释掉的哪行代码：开启此特征值为true。再次运行程序，输出：

```json
{"name":"clx"}
```

**注意：谨慎打开此开关，如果检查的话性能会下降20%-30%。**

### IGNORE_UNKNOWN(false)

如果**底层数据格式**需要输出所有属性，以及如果**找不到**调用者试图写入的属性的定义，则该特性确定是否要执行的操作。

可能你听完还一脸懵逼，什么底层数据格式，什么找不到，我明明是写JSON啊，何解？其实这不是针对于写JSON来说的，**对于JSON，这个特性没有效果，因为属性不需要预先定义**。通常，大多数文本数据格式不需要模式信息，而某些二进制数据格式需要定义（如Avro、protobuf），因此这个属性是为它们而生（Smile、BSON等这些二进制也是不需要预定模式信息的哦）。

> 强调：`JsonGenerator`不是只能写JSON格式，毕竟底层是I/O流嘛，理论上啥都能写

- true：启动该功能

可以预先调用（在写数据之前）这个API设定好模式信息即可：

```java
JsonGenerator：

	public void setSchema(FormatSchema schema) {
		...
	}
```

- false：禁用该功能。如果底层数据格式需要所有属性的知识才能输出，那就抛出JsonProcessingException异常

## 3.定制Feature

通过上一part知晓了控制`JsonGenerator`的特征值们，以及其作用是。Feature的每个枚举值都有个默认值（括号里面），那么如果我们希望对**不同的JsonGenerator实例**应用不同的配置该怎么办呢？

自然而然的JsonGenerator提供了相关API供以我们操作：

```java
// 开启
public abstract JsonGenerator enable(Feature f);
// 关闭
public abstract JsonGenerator disable(Feature f);
// 开启/关闭
public final JsonGenerator configure(Feature f, boolean state) { ... };

public abstract boolean isEnabled(Feature f);
public boolean isEnabled(StreamWriteFeature f) { ... };
```

## 4.替换者：StreamWriteFeature

本类是2.10版本新增的，用于完全替换上面的Feature。目的：完全独立的属性配置，不依赖于任何后端格式，因为`JsonGenerator`并不局限于写JSON，因此把Feature放在JsonGenerator作为内部类是不太合适的，所以单独摘出来。

StreamWriteFeature用在`JsonFactory`里，后面再讲解到它的构建器`JsonFactoryBuilder`时再详细探讨。

## 5.序列化POJO对象

上篇文章用代码演示过了如何使用`writeObject(Object pojo)`来把一个POJO一次性序列化成为一个JSON串，它主要依赖于ObjectCodec去完成

```java
public abstract JsonGenerator setCodec(ObjectCodec oc);
```

ObjectCodec可谓是Jackson里极其重要的一个基础组件，我们最熟悉的`ObjectMapper`它就是一个解码器，实现了序列化和反序列化、树模型等操作。

## 6.输出漂亮的JSON格式

我们知道JSON之所以快速流行的原因之一是得益于它的**可读性好**，可读性好又表现在它漂亮的（规则）的展示格式上。

默认情况下，使用`JsonGenerator`写JSON时，所有的部分都是输出在同一行里，显然这种格式对人阅读来说是不够友好的。作为最流行的JSON库自然考虑到了这一点，提供了格式化器来**美化输出**：

```java
// 自己指定漂亮格式打印器
public JsonGenerator setPrettyPrinter(PrettyPrinter pp) { ... }

// 应用默认的漂亮格式打印器
public abstract JsonGenerator useDefaultPrettyPrinter();
```

使用不同的实现类，对输出结果的影响如下：

```java
什么都不设置：
MinimalPrettyPrinter：
{"zhName":"A哥","enName":"YourBatman","age":18}

DefaultPrettyPrinter：
useDefaultPrettyPrinter():
{
  "zhName" : "A哥",
  "enName" : "YourBatman",
  "age" : 18
}
```

由此可见，在什么都不设置的情况下，结果会全部在一行显示（紧凑型输出）。`DefaultPrettyPrinter`表示带层级格式的输出（可读性好），若有此需要，建议直接调用更为快捷的`useDefaultPrettyPrinter()`方法，而不用自己去new一个实例。



