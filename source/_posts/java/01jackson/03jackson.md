---
title: 02jackson初始
toc: true
tags: java
categories: 
    - [java]
    - [jackson]
---

## 1.前言

命名为core的模块一般都不简单，`jackson-core`自然也不例外。它是三大核心模块之一，并且是**核心中的核心**，提供了对JSON数据的**完整支持**（包括各种读、写）。它是三者中最强大的模块，具有**最低的**开销和**最快的**读/写操作。

<!--more-->

此模块提供了**最具底层**的Streaming JSON解析器/生成器，这组流式API属于Low-Level API，具有非常显著的特点：

- 开销小，损耗小，性能极高
- 因为是Low-Level API，所以灵活度极高
- 又因为是Low-Level API，所以易错性高，可读性差

**jackson-core**模块提供了两种处理JSON的方式（纵缆整个Jackson共三种）：

1. 流式API：读取并将JSON内容写入作为离散事件 -> `JsonParser`读取数据，而`JsonGenerator`负责写入数据
2. 树模型：JSON文件在内存里以树形式表示。此种方式也很灵活，它类似于XML的DOM解析，层层嵌套的

作为“底层”技术，应用级开发中确实接触不多。为了引起你的重视，提前预告一下：`Spring MVC`对JSON消息的转换器`AbstractJackson2HttpMessageConverter`它就用到了底层流式API -> JsonGenerator写数据。

## 2.版本约定

- Jackson版本：`2.11.0`
- Spring Framework版本：`5.2.6.RELEASE`
- Spring Boot版本：2.3.0.RELEASE
  - 内置的Jackson和Spring版本均和👆保持一致，避免了版本交叉

> 说明：类似2.11.0和2.11.x这种小版本号的差异，你权可认为没有区别

## 3.正文

Jackson提供了一种对性能有极致要求的方式：流式API。它用于对性能有极致要求的场景，这个时候就可以使用此种方式来对JSON进行读写。

###  1.概念解释：流式、增量模式、JsonToken

- 流式（Streaming）：此概念和Java8中的Stream流是不同的。这里指的是**IO流**，因此具有最低的开销和最快的读/写操作（记得关流哦）
- 增量模式（incremental mode）：它表示每个部分一个一个地往上增加，类似于垒砖。使用此流式API读写JSON的方式使用的**均是增量模式**
- JsonToken：每一部分都是一个独立的Token（有不同类型的Token），最终被“拼凑”起来就是一个JSON。这是流式API里很重要的一个抽象概念。

### 2.JsonGenerator使用Demo

`JsonGenerator`定义用于编写JSON内容的公共API的基类（抽象类）。实例使用的工厂方法创建，也就是`JsonFactory`。

> 小贴士：纵观整个Jackson，它更多的是使用抽象类而非接口，这是它的一大“特色”。因此你熟悉的面向接口编程，到这都要转变为面向抽象类编程喽。

demo1

```java
private static void test1() throws IOException {
        JsonFactory factory = new JsonFactory();
   		// 本处只需演示，向控制台写（当然你可以向文件等任意地方写都是可以的）
        JsonGenerator jsonGenerator = factory.createGenerator(System.out, JsonEncoding.UTF8);

        try {
            jsonGenerator.writeStartObject();  // 开始写，也就是符号 "{"
            jsonGenerator.writeStringField("name", "clxmm");
            jsonGenerator.writeNumberField("age", 18);
            jsonGenerator.writeEndObject();   // 结束写，也就是这个符号 "}"

        } finally {
            jsonGenerator.close();
        }
        
        // 输出 {"name":"clxmm","age":18}
    }
```

因为JsonGenerator实现了`AutoCloseable`接口，因此可以使用`try-with-resources`优雅关闭资源（这也是推荐的使用方式），代码改造如下：

```java
    private static void test2() throws IOException {
        JsonFactory factory = new JsonFactory();
        
        // 本处只需演示，向控制台写（当然你可以向文件等任意地方写都是可以的）
        try (JsonGenerator jsonGenerator = factory.createGenerator(System.out, JsonEncoding.UTF8);) {
            jsonGenerator.writeStartObject();  // 开始写，也就是符号 "{"
            jsonGenerator.writeStringField("name", "clxmm");
            jsonGenerator.writeNumberField("age", 18);
            jsonGenerator.writeEndObject();   // 结束写，也就是这个符号 "}"

        }

        // 输出 {"name":"clxmm","age":18}
    }
```

这是最简使用示例，这也就是所谓的**序列化**底层实现，从示例中对**增量模式**能够有所感受吧。

纯手动档有木有，灵活性和性能极高，但易出错。

### 3.JsonGenerator详细介绍

JsonGenerator是个抽象类，它的继承体系如下：

![](/img/202208/01jackson.png)

- WriterBasedJsonGenerator：基于java.io.Writer处理字符编码（话外音：使用Writer输出JSON）

  - 因为UTF-8编码基本标准化了，因此Jackson内部也提供了`SegmentedStringWriter/UTF8Writer`来简化操作

- `UTF8JsonGenerator`：基于OutputStream + UTF-8处理字符编码（话外音：明确指定了使用UTF-8编码把字节变为字符）

默认情况下（不指定编码），Jackson默认会使用UTF-8进行编码，也就是说会使用`UTF8JsonGenerator`作为实际的JSON生成器实现类，具体逻辑将在讲述`JsonFactory`章节中有所体现，敬请关注。

值得注意的是，抽象基类`JsonGenerator`它只负责JSON的生成，至于把生成好的JSON写到哪里去它并不关心。比如示例中我给写到了控制台，当然你也可以写到文件、写到网络等等。

> Spring MVC中的JSON消息转换器就是向`HttpOutputMessage`（网络输出流）里写JSON数据

## 4.关键API

`JsonGenerator`虽然仅是抽象基类，但Jackson它建议我们使用`JsonFactory`工厂来创建其实例，并不需要使用者去关心其底层实现类，因此我们仅需要**面向此抽象类编程**即可，此为对使用者非常友好的设计。

对于JSON生成器来说，写方法自然是它的灵魂所在。众所周知，JSON属于K-V数据结构，因此针对于一个JSON来说，每一段都k额分为**写key**和**写value**两大阶段。

### 1.写JSON Key

JsonGenerator一共提供了3个方法用于写JSON的key：

`源码`

```java
public abstract void writeFieldName(String name) throws IOException;
public abstract void writeFieldName(SerializableString name) throws IOException;
public void writeFieldId(long id) throws IOException {
        writeFieldName(Long.toString(id));
}
```

`demo`

```java
    private static void test3() throws IOException {
        JsonFactory factory = new JsonFactory();

        try (JsonGenerator jsonGenerator = factory.createGenerator(System.err, JsonEncoding.UTF8)) {
                jsonGenerator.writeStartObject();
                jsonGenerator.writeFieldName("name");
                jsonGenerator.writeEndObject();

        }
    }
```

输出

```
{"name"}
```

可以发现，**key可以独立存在（无需value）**，但value是不能独立存在的哦，下面你会看到效果。而3个方法中的**其它2个方法**：

```java
public abstract void writeFieldName(SerializableString name) throws IOException;

public void writeFieldId(long id) throws IOException {
    writeFieldName(Long.toString(id));
}
```

这两个方法，你可以忘了吧，记住`writeFieldName()`就足够了。

总的来说，写JSON的key非常简单的，这得益于JSON的key有且仅可能是String类型，所以情况单一。下面继续了解较为复杂的写Value的情况。

### 2.写JSON Value

我们知道在Java中数据存在的形式（类型）非常之多，比如String、int、Reader、char[]…，而在JSON中**值的类型**只能是如下形式：

- 字符串（如`{ "name":"YourBatman" }`）
- 数字（如`{ "age":18 }`）
- 对象（JSON 对象）（如`{ "person":{ "name":"YourBatman", "age":18}}`）
- 数组（如`{"names":[ "YourBatman", "A哥" ]}`）
- 布尔（如`{ "success":true }`）
- null（如：`{ "name":null }`）

> 小贴士：像数组、对象等这些“高级”类型可以互相无限嵌套

很明显，Java中的数据类型和JSON中的值类型并不是一一对应的关系，那么这就需要`JsonGenerator`在写入时起到一个桥梁（适配）作用

下面针对不同的Value类型分别作出API讲解，给出示例说明。

- JSON的顺序，和你write的顺序保持一致
- 写任何类型的Value之前请记得先write写key，否则可能无效

#### 1.字符串

api 

```java
public abstract void writeString(String text) throws IOException;
public void writeString(Reader reader, int len) throws IOException {
  // Let's implement this as "unsupported" to make it easier to add new parser impls
  _reportUnsupportedOperation();
}
public abstract void writeString(char[] text, int offset, int len) throws IOException;
public abstract void writeString(SerializableString text) throws IOException;
public abstract void writeRawUTF8String(byte[] text, int offset, int length)
        throws IOException;
public abstract void writeUTF8String(byte[] text, int offset, int length)
        throws IOException;
```

可把Java中的String类型、Reader类型、char[]字符数组类型等等写为JSON的字符串形式。

demo

```java
private static void test4() throws IOException {
        JsonFactory factory = new JsonFactory();

        try (JsonGenerator jsonGenerator = factory.createGenerator(System.err, JsonEncoding.UTF8)) {
            jsonGenerator.writeStartObject();
            jsonGenerator.writeFieldName("name");
            jsonGenerator.writeString("clxmm");

            jsonGenerator.writeFieldName("age");
            jsonGenerator.writeString("18");
            
            jsonGenerator.writeEndObject();
        }

    }
```

输出：

```
{"name":"clxmm","age":"18"}
```

#### 2.数字

api 

```java
public void writeNumber(short v) throws IOException { writeNumber((int) v); }

public abstract void writeNumber(int v) throws IOException;

public abstract void writeNumber(long v) throws IOException;

public abstract void writeNumber(BigInteger v) throws IOException;

public abstract void writeNumber(double v) throws IOException;

public abstract void writeNumber(float v) throws IOException;

public abstract void writeNumber(BigDecimal v) throws IOException;

public abstract void writeNumber(String encodedValue) throws IOException;
```

#### 3.对象（JSON 对象）

api 

```java
public abstract void writeStartObject() throws IOException;

public void writeStartObject(Object forValue) throws IOException
{
  writeStartObject();
  setCurrentValue(forValue);
}

public void writeStartObject(Object forValue, int size) throws IOException
{
  writeStartObject();
  setCurrentValue(forValue);
}

public abstract void writeEndObject() throws IOException;
```

demo

```java
private static void test5() throws IOException {
        JsonFactory factory = new JsonFactory();

        try (JsonGenerator jsonGenerator = factory.createGenerator(System.err, JsonEncoding.UTF8)) {
            jsonGenerator.writeStartObject();
            jsonGenerator.writeFieldName("name");
            jsonGenerator.writeString("clxmm");

            jsonGenerator.writeFieldName("cat");   // 必须要有key
            jsonGenerator.writeStartObject();
            jsonGenerator.writeFieldName("catName");
            jsonGenerator.writeString("mimi");
            jsonGenerator.writeFieldName("age");
            jsonGenerator.writeNumber(3);

            jsonGenerator.writeEndObject();


            jsonGenerator.writeEndObject();
        }
    }
```

输出：

```json
{"name":"clxmm","cat":{"catName":"mimi","age":3}}
```

> 对象属于一个比较特殊的value值类型，可以实现各种嵌套。也就是我们平时所说的JSON套JSON

#### 4.数组

写数组和写对象有点类似，也会有先start再end的闭环思路。

api

```java
public abstract void writeStartArray() throws IOException;

public void writeStartArray(int size) throws IOException {
  writeStartArray();
}

public void writeStartArray(Object forValue) throws IOException {
  writeStartArray();
  setCurrentValue(forValue);
}

public void writeStartArray(Object forValue, int size) throws IOException {
  writeStartArray(size);
  setCurrentValue(forValue);
}

public abstract void writeEndArray() throws IOException;
```

如何向数组里写入Value值？我们知道JSON数组里可以装任何数据类型，因此往里写值的方法都可使用，形如这样：

demo

```java
private static void test6Array() throws IOException {
        JsonFactory factory = new JsonFactory();

        try (JsonGenerator jsonGenerator = factory.createGenerator(System.err, JsonEncoding.UTF8)) {
            jsonGenerator.writeStartObject();
            jsonGenerator.writeFieldName("name");
            jsonGenerator.writeString("clxmm");

            jsonGenerator.writeFieldName("array");   // key
            jsonGenerator.writeStartArray();

            // 字符串
            jsonGenerator.writeString("red");
            jsonGenerator.writeString("blue");
            // 写对象
            jsonGenerator.writeStartObject();
            jsonGenerator.writeStringField("age", "17");
            jsonGenerator.writeEndObject();
            // 写数字
            jsonGenerator.writeNumber(33);
            jsonGenerator.writeEndArray();

            jsonGenerator.writeEndObject();
        }

    }
```

输出：

```json
{"name":"clxmm","array":["red","blue",{"age":"17"},33]}
```

> 理论上JSON数组里的每个元素可以是不同类型，但**原则上**请确保是同一类型哦

对于JSON数组类型，很多时候里面装载的是数字或者普通字符串类型，因此`JsonGenerator`也很暖心的为此提供了专用方法（可以调用该方法来一次性便捷的写入单个数组）：

api 

```java
public void writeArray(int[] array, int offset, int length)
  
public void writeArray(long[] array, int offset, int length) 
  
 public void writeArray(double[] array, int offset, int length)
  
  
```

demo

```java
private static void test7Array() throws IOException {

  JsonFactory factory = new JsonFactory();

  try (JsonGenerator jsonGenerator = factory.createGenerator(System.err, JsonEncoding.UTF8)) {
    jsonGenerator.writeStartObject();
    jsonGenerator.writeFieldName("name");
    jsonGenerator.writeString("clxmm");


    // 快速写数组，index=2 开始，写3个
    jsonGenerator.writeFieldName("values");
    jsonGenerator.writeArray(new int[]{1, 2, 3, 4, 5, 6, 7}, 2, 3);


    jsonGenerator.writeEndObject();
  }
}
```

输出

```json
{"name":"clxmm","values":[3,4,5]}
```

#### 5.布尔和null

api

```java
public abstract void writeBoolean(boolean state) throws IOException;

public abstract void writeNull() throws IOException;
```

demo

```java
private static void test8Array() throws IOException {
  JsonFactory factory = new JsonFactory();

  try (JsonGenerator jsonGenerator = factory.createGenerator(System.err, JsonEncoding.UTF8)) {
    jsonGenerator.writeStartObject();
    jsonGenerator.writeFieldName("success");
    jsonGenerator.writeBoolean(true);
    jsonGenerator.writeFieldName("myName");
    jsonGenerator.writeNull();

    jsonGenerator.writeEndObject();
  }
}
```

输出

```json
{"success":true,"myName":null}
```

### 3.组合写JSON Key和Value

在写每个value之前，都必须写key。为了**简化书写**，JsonGenerator提供了二合一的组合方法，一个顶两：

api

```java
public final void writeBooleanField(String fieldName, boolean value)
  
public final void writeNullField(String fieldName)
  
public final void writeNumberField(String fieldName, int value)
public final void writeNumberField(String fieldName, long value) 
public final void writeNumberField(String fieldName, double value)
public final void writeNumberField(String fieldName, float value)
public final void writeNumberField(String fieldName, BigDecimal value)
  
public final void writeBinaryField(String fieldName, byte[] data)

public final void writeArrayFieldStart(String fieldName)

public void writeStringField(String fieldName, String value)

```

demo

```java
private static void test9() throws IOException {
  JsonFactory factory = new JsonFactory();
  try (JsonGenerator jsonGenerator = factory.createGenerator(System.out, JsonEncoding.UTF8)) {
    jsonGenerator.writeStartObject();

    jsonGenerator.writeStringField("name","clxmm");
    jsonGenerator.writeBooleanField("success",true);
    jsonGenerator.writeNullField("myName");
    // jsonGenerator.writeObjectFieldStart();
    // jsonGenerator.writeArrayFieldStart();

    jsonGenerator.writeEndObject();
  }
}
```

输出

```json
{"name":"clxmm","success":true,"myName":null}
```

### 4.其它写方法

#### 1.**writeRaw()和writeRawValue()**：

api

```java
public abstract void writeRaw(String text)
  
public abstract void writeRaw(char c) throws IOException;

public abstract void writeRaw(String text, int offset, int len)
  
public abstract void writeRaw(char[] text, int offset, int len)
  
public void writeRaw(SerializableString raw) 
  
public abstract void writeRawValue(String text) 

public abstract void writeRawValue(String text, int offset, int len)

public abstract void writeRawValue(char[] text, int offset, int len)
public void writeRawValue(SerializableString raw)
```



该方法将强制生成器**不做任何修改**地逐字复制输入文本（包括不进行转义，也不添加分隔符，即使上下文[array，object]可能需要这样做）。如果需要这样的分隔符，请改用writeRawValue方法。

demo

```java
private static void test10() throws IOException {
  JsonFactory factory = new JsonFactory();
  try (JsonGenerator jsonGenerator = factory.createGenerator(System.out, JsonEncoding.UTF8)) {
    jsonGenerator.writeRaw("{'name':'clxmm'}");
  }
}
```

输出

```json
{'name':'clxmm'}
```

如果换成`writeString()`方法，结果为（请注意比较差异）：

```json
"{'name':'clxmm'}"
```

#### 2.**writeBinary()**：

api

```java
public abstract void writeBinary(Base64Variant bv,
            byte[] data, int offset, int len) throws IOException;

public void writeBinary(byte[] data, int offset, int len) throws IOException {
  writeBinary(Base64Variants.getDefaultVariant(), data, offset, len);
}

public void writeBinary(byte[] data) throws IOException {
        writeBinary(Base64Variants.getDefaultVariant(), data, 0, data.length);
}

public int writeBinary(InputStream data, int dataLength)
        throws IOException {
        return writeBinary(Base64Variants.getDefaultVariant(), data, dataLength);
}

 public abstract int writeBinary(Base64Variant bv,
            InputStream data, int dataLength) throws IOException;。
```

使用Base64编码把数据写进去。

#### .**writeEmbeddedObject()**：

```java
public void writeEmbeddedObject(Object object) throws IOException {
  // 01-Sep-2016, tatu: As per [core#318], handle small number of cases
  if (object == null) {
    writeNull();
    return;
  }
  if (object instanceof byte[]) {
    writeBinary((byte[]) object);
    return;
  }
  throw new JsonGenerationException("No native support for writing embedded objects of type "
                                    +object.getClass().getName(),
                                    this);
}
```

### 5.**writeObject()**（重要）：

写POJO，但前提是你必须给`JsonGenerator`指定一个`ObjectCodec`解码器才能正常work，否则抛出异常：

pojo

```java
@Data
public class User {

    private String name = "clxmm";
    private Integer age = 18;
}
```

demo

```java
private static void test11Object() throws IOException {

  JsonFactory factory = new JsonFactory();
  try (JsonGenerator jsonGenerator = factory.createGenerator(System.out, JsonEncoding.UTF8)) {
    jsonGenerator.writeObject(new User());
  }
}
```

错误信息

```
java.lang.IllegalStateException: No ObjectCodec defined for the generator, can only serialize simple wrapper types (type passed org.clxmm.jackson.bean.User)
	at com.fasterxml.jackson.core.JsonGenerator._writeSimpleObject(JsonGenerator.java:2168)
```

值得注意的是，Jackson里我们最为熟悉的API `ObjectMapper`它就是一个ObjectCodec解码器，具体我们在**数据绑定**章节会再详细讨论，下面我给出个简单的使用示例模拟一把：

准备一个User对象，以及解码器UserObjectCodec：

```java
// 自定义ObjectCodec解码器 用于把User写为JSON
public class UserObjectCodec extends ObjectCodec {
   // 因为本例只关注write写，因此只需要实现此这一个方法即可
    @Override
    public void writeValue(JsonGenerator gen, Object value) throws IOException {
        User user = (User) value;

        gen.writeStartObject();
        gen.writeStringField("name",user.getName());
        gen.writeNumberField("age",user.getAge());
        gen.writeEndObject();
    }
  
 
  .......
}
```

```java
private static void test11Object() throws IOException {

  JsonFactory factory = new JsonFactory();
  try (JsonGenerator jsonGenerator = factory.createGenerator(System.out, JsonEncoding.UTF8)) {
    jsonGenerator.setCodec(new UserObjectCodec());
    jsonGenerator.writeObject(new User());
  }
}
```

输出

```json
{"name":"clxmm","age":18}
```

😄这就是`ObjectMapper`的原理雏形，

### 6.**writeTree()**：

顾名思义，它便是Jackson大名鼎鼎的**树模型**。可惜的是core模块并没有提供树模型TreeNode的实现，以及它也是得依赖于ObjectCodec才能正常完成解码。

方法用来编写给定的JSON树（表示为树，其中给定的JsonNode是根）。这通常只调用给定节点的writeObject，但添加它是为了方便起见，并使代码在专门处理树的情况下更显式。

可能你会想，已经有了`writeObject()`方法还要它干啥呢？这其实是蛮有必要的，因为有时候你并不想定义POJO时，就可以用它快速写/读数据，同时它也可以达到**模糊掉类型的概念**，做到更抽象和更公用。

>  说到模糊掉类型的的操作，你也可以辅以Spring的`AnnotationAttributes`的设计和使用来理解

准备一个TreeNode的实现UserTreeNode：

```java
public class UserTreeNode implements TreeNode {

    private User user;


    public User getUser() {
        return user;
    }

    public UserTreeNode(User user) {
        this.user = user;
    }
  
  .......
}
```

UserObjectCodec改写如下：

```java
 // 因为本例只关注write写，因此只需要实现此这一个方法即可
    @Override
    public void writeValue(JsonGenerator gen, Object value) throws IOException {
        User user =null;
        if (value instanceof User) {
            user = (User) value;
        } else if (value instanceof TreeNode) {
            user = ((UserTreeNode) value).getUser();
        }


        gen.writeStartObject();
        gen.writeStringField("name",user.getName());
        gen.writeNumberField("age",user.getAge());
        gen.writeEndObject();
    }
```

demo

```java
private static void test12() throws IOException {
  JsonFactory factory = new JsonFactory();
  try (JsonGenerator jsonGenerator = factory.createGenerator(System.err, JsonEncoding.UTF8)) {
    jsonGenerator.setCodec(new UserObjectCodec());
    jsonGenerator.writeObject(new UserTreeNode(new User()));
  }
}
```

consle

```java
{"name":"clxmm","age":18}
```

本案例绕过了`TreeNode`的真实处理逻辑，是因为**树模型**这块会放在databind数据绑定模块进行更加详细的描述

## 5.总结

本文介绍了jackson-core模块的流式API，以及JsonGenerator写JSON的使用，相信对你理解Jackson生成JSON方面是有帮助的。它作为JSON处理的基石，虽然并不推荐直接使用，但仅仅是**应用开发级别**不推荐哦，如果你是个框架、中间件开发者，这些原理你很可能绕不过。

本文介绍它的目的并不是建议大家去项目上使用，而是为了后面理解`ObjectMapper`夯实基础

