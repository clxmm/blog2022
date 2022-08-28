---
title: 05json字符串时如何解析的
toc: true
tags: java
categories: 
    - [java]
    - [jackson]
---


## 1.简介
什么叫读JSON？就是把一个JSON 字符串 解析为对象or树模型嘛，因此也称作解析JSON串。Jackson底层流式API使用JsonParser来完成JSON字符串的解析。

<!--more-->

## 2.简单Demo

准备一个POJO：

```java
@Data
public class Person {
    private String name;
    private Integer age;
}
```

测试用例：把一个JSON字符串绑定（封装）进一个POJO对象里

```java
@Test
public void test1() throws IOException {
    String jsonStr = "{\"name\":\"YourBatman\",\"age\":18}";
    Person person = new Person();

    JsonFactory factory = new JsonFactory();
    try (JsonParser jsonParser = factory.createParser(jsonStr)) {
        
        // 只要还没结束"}"，就一直读
        while (jsonParser.nextToken() != JsonToken.END_OBJECT) {
            String fieldname = jsonParser.getCurrentName();
            if ("name".equals(fieldname)) {
                jsonParser.nextToken();
                person.setName(jsonParser.getText());
            } else if ("age".equals(fieldname)) {
                jsonParser.nextToken();
                person.setAge(jsonParser.getIntValue());
            }
        }
        
        System.out.println(person);
    }
}
```

运行程序，输出：

```json
Person(name=YourBatman, age=18)
```

成功把一个JSON字符串的值解析到Person对象。你可能会疑问，怎么这么麻烦？那当然，这是底层流式API，纯**手动档**嘛。你获得了性能，可不要失去一些便捷性嘛。

> 小贴士：底层流式API一般面向“专业人士”，应用级开发使用高阶API `ObjectMapper`即可。

`JsonParser`针对不同的value类型，提供了非常多的方法用于实际值的获取。

```java
// 获取字符串类型
public abstract String getText() throws IOException;

// 数字Number类型值 标量值（支持的Number类型参照NumberType枚举）
public abstract Number getNumberValue() throws IOException;
public enum NumberType {
    INT, LONG, BIG_INTEGER, FLOAT, DOUBLE, BIG_DECIMAL
};

public abstract int getIntValue() throws IOException;
public abstract long getLongValue() throws IOException;
...
public abstract byte[] getBinaryValue(Base64Variant bv) throws IOException;
```

这类方法可能会抛出异常：比如value值本不是数字但你调用了getInValue()方法~

小贴士：如果value值是null，像getIntValue()、getBooleanValue()等这种直接获取方法是会抛出异常的，但getText()不会

**带默认值**的值获取，具有更好安全性：

```java
public String getValueAsString() throws IOException {
    return getValueAsString(null);
}
public abstract String getValueAsString(String def) throws IOException;
...
public long getValueAsLong() throws IOException {
    return getValueAsLong(0);
}
public abstract long getValueAsLong(long def) throws IOException;
...
```

此类方法若碰到数据的转换失败时，**不会抛出异常**，把`def`作为默认值返回。

### 1.组合方法

同`JsonGenerator`一样，JsonParser也提供了**高钙片**组合方法，让你更加便捷的使用

```java
public abstract JsonToken nextToken() throws IOException;
public abstract JsonToken nextValue() throws IOException;

public boolean nextFieldName(SerializableString str) throws IOException
public String nextFieldName() throws IOException

 public String nextTextValue() throws IOException
 public int nextIntValue(int defaultValue)
  
public long nextLongValue(long defaultValue) throws IOException
public Boolean nextBooleanValue() throws IOException
```

### 2.自动绑定

听起来像高级功能，是的，它必须依赖于`ObjectCodec`去实现，因为实际是全部委托给了它去完成的，也就是我们最为熟悉的readXXX系列方法：

```java
public <T> T readValueAs(Class<T> valueType) 
public <T> T readValueAs(TypeReference<?> valueTypeRef) 
public <T> Iterator<T> readValuesAs(Class<T> valueType)
 public <T> Iterator<T> readValuesAs(TypeReference<T> valueTypeRef)
public <T extends TreeNode> T readValueAsTree()
```



我们知道，ObjectMapper就是一个ObjectCodec，它属于高级API，本文显然不会用到ObjectMapper它喽，因此我们自己手敲一个实现来完成此功能。

自定义一个ObjectCodec，Person类专用：用于把JSON串自动绑定到实例属性

```java
public class PersonObjectCodec extends ObjectCodec {
	...
    @SneakyThrows
    @Override
    public <T> T readValue(JsonParser jsonParser, Class<T> valueType) throws IOException {
        Person person = (Person) valueType.newInstance();

        // 只要还没结束"}"，就一直读
        while (jsonParser.nextToken() != JsonToken.END_OBJECT) {
            String fieldname = jsonParser.getCurrentName();
            if ("name".equals(fieldname)) {
                jsonParser.nextToken();
                person.setName(jsonParser.getText());
            } else if ("age".equals(fieldname)) {
                jsonParser.nextToken();
                person.setAge(jsonParser.getIntValue());
            }
        }

        return (T) person;
    }
	...
}
```

有了它，就可以实现我们的**自动**绑定了，书写测试用例：

```java
private static void test2() throws IOException {
  String jsonStr = "{\"name\":\"clx\",\"age\":18, \"pickName\":null}";

  JsonFactory factory = new JsonFactory();
  try (JsonParser jsonParser = factory.createParser(jsonStr)) {
    jsonParser.setCodec(new PersonObjectCodec());

    System.out.println(jsonParser.readValueAs(Person.class));
  }

}
```

输出

```json
Person(name=clx, age=18)
```

## 3.JsonToken

在上例解析过程中，有一个非常重要的角色，那便是：JsonToken。它表示解析JSON内容时，用于返回结果的基本**标记类型**的枚举。

```java
public enum JsonToken {
	NOT_AVAILABLE(null, JsonTokenId.ID_NOT_AVAILABLE),
	
	START_OBJECT("{", JsonTokenId.ID_START_OBJECT),
	END_OBJECT("}", JsonTokenId.ID_END_OBJECT),
	START_ARRAY("[", JsonTokenId.ID_START_ARRAY),
	END_ARRAY("]", JsonTokenId.ID_END_ARRAY),

	// 属性名（key）
	FIELD_NAME(null, JsonTokenId.ID_FIELD_NAME),

	// 值（value）
	VALUE_EMBEDDED_OBJECT(null, JsonTokenId.ID_EMBEDDED_OBJECT),
	VALUE_STRING(null, JsonTokenId.ID_STRING),
	VALUE_NUMBER_INT(null, JsonTokenId.ID_NUMBER_INT),
	VALUE_NUMBER_FLOAT(null, JsonTokenId.ID_NUMBER_FLOAT),
	VALUE_TRUE("true", JsonTokenId.ID_TRUE),
	VALUE_FALSE("false", JsonTokenId.ID_FALSE),
	VALUE_NULL("null", JsonTokenId.ID_NULL),
}
```

demo

```java
private static void test3() throws IOException {

        String jsonStr = "{\"name\":\"clxmm\",\"age\":18, \"pickName\":null}";
        System.out.println(jsonStr);
        JsonFactory factory = new JsonFactory();
        try (JsonParser jsonParser = factory.createParser(jsonStr)) {

            while (true) {
                JsonToken token = jsonParser.nextToken();
                System.out.println(token + " -> 值为:" + jsonParser.getValueAsString());

                if (token == JsonToken.END_OBJECT) {
                    break;
                }
            }
        }

    }
```

console

```
{"name":"clxmm","age":18, "pickName":null}

START_OBJECT -> 值为:null

FIELD_NAME -> 值为:name
VALUE_STRING -> 值为:clxmm

FIELD_NAME -> 值为:age
VALUE_NUMBER_INT -> 值为:18

FIELD_NAME -> 值为:pickName
VALUE_NULL -> 值为:null

END_OBJECT -> 值为:null
```

> 小贴士：解析时请确保你的的JSON串是合法的，否则抛出`JsonParseException`异常

## 4.JsonParser的Feature

它是JsonParser的一个内部枚举类，共15个枚举值：

```java
public enum Feature {
	AUTO_CLOSE_SOURCE(true),
	
	ALLOW_COMMENTS(false),
	ALLOW_YAML_COMMENTS(false),
	ALLOW_UNQUOTED_FIELD_NAMES(false),
	ALLOW_SINGLE_QUOTES(false),
	@Deprecated
	ALLOW_UNQUOTED_CONTROL_CHARS(false),
	@Deprecated
	ALLOW_BACKSLASH_ESCAPING_ANY_CHARACTER(false),
	@Deprecated
	ALLOW_NUMERIC_LEADING_ZEROS(false),
	@Deprecated
	ALLOW_LEADING_DECIMAL_POINT_FOR_NUMBERS(false),
	@Deprecated
	ALLOW_NON_NUMERIC_NUMBERS(false),
	@Deprecated
	ALLOW_MISSING_VALUES(false),
	@Deprecated
	ALLOW_TRAILING_COMMA(false),
	
	STRICT_DUPLICATE_DETECTION(false),
	IGNORE_UNDEFINED(false),
	INCLUDE_SOURCE_IN_LOCATION(true);
}
```

> 小贴士：枚举值均为bool类型，括号内为默认值

每个枚举值都控制着`JsonParser`不同的行为。下面分类进行解释

### 1.底层I/O流相关

自2.10版本后，使用`StreamReadFeature#AUTO_CLOSE_SOURCE`代替

Jackson的流式API指的是I/O流，所以即使是**读**，底层也是用I/O流（Reader）去读取然后解析的。

#### 1.AUTO_CLOSE_SOURCE(true)

原理和JsonGenerator的`AUTO_CLOSE_TARGET(true)`一样

### 2.支持非标准格式

JSON是有规范的，在它的规范里并没有描述到对注释的规定、对控制字符的处理等等，也就是说这些均属于**非标准**行为。比如这个JSON串：

```json
{
	"name" : "YourBarman", // 名字
	"age" : 18 // 年龄
}
```

你看，若你这么写IDEA都会飘红提示你：

**但是**，在很多使用场景（特别是JavaScript）里，我们会在JSON串里写注释（属性多时尤甚）那么对于这种串，JsonParser如何控制处理呢？它提供了对非标准JSON格式的兼容，通过下面这些特征值来控制。

#### 1.ALLOW_COMMENTS(false)

> 自2.10版本后，使用`JsonReadFeature#ALLOW_JAVA_COMMENTS`代替

是否允许`/* */`或者`//`这种类型的注释出现。

demo

```java
@Test
public void test4() throws IOException {
    String jsonStr = "{\n" +
            "\t\"name\" : \"YourBarman\", // 名字\n" +
            "\t\"age\" : 18 // 年龄\n" +
            "}";

    JsonFactory factory = new JsonFactory();
    try (JsonParser jsonParser = factory.createParser(jsonStr)) {
    	// 开启注释支持
        // jsonParser.enable(JsonParser.Feature.ALLOW_COMMENTS);

        while (jsonParser.nextToken() != JsonToken.END_OBJECT) {
            String fieldname = jsonParser.getCurrentName();
            if ("name".equals(fieldname)) {
                jsonParser.nextToken();
                System.out.println(jsonParser.getText());
            } else if ("age".equals(fieldname)) {
                jsonParser.nextToken();
                System.out.println(jsonParser.getIntValue());
            }
        }
    }
}
```

运行程序，抛出异常

```
com.fasterxml.jackson.core.JsonParseException: Unexpected character ('/' (code 47)): maybe a (non-standard) comment? (not recognized as one since Feature 'ALLOW_COMMENTS' not enabled for parser)
 at [Source: (String)"{
	"name" : "YourBarman", // 名字
	"age" : 18 // 年龄
}"; line: 2, column: 26]
```

放开注释的代码，再次运行程序，**正常work**。

#### 2.ALLOW_YAML_COMMENTS(false)

> 自2.10版本后，使用`JsonReadFeature#ALLOW_YAML_COMMENTS`代替

顾名思义，开启后将支持Yaml格式的的注释，也就是`#`形式的注释语法。

#### 3.ALLOW_UNQUOTED_FIELD_NAMES(false)

> 自2.10版本后，使用`JsonReadFeature#ALLOW_UNQUOTED_FIELD_NAMES`代替

是否允许属性名**不带双引号””**，比较简单

#### 4.ALLOW_SINGLE_QUOTES(false)

> 自2.10版本后，使用`JsonReadFeature#ALLOW_SINGLE_QUOTES`代替

是否允许属性名支持单引号，也就是使用`''`包裹，形如这样：

```json
{
    'age' : 18
}
```

#### 5.~~ALLOW_UNQUOTED_CONTROL_CHARS(false)~~

> 自2.10版本后，使用`JsonReadFeature#ALLOW_UNESCAPED_CONTROL_CHARS`代替

是否允许JSON字符串包含非引号**控制字符**（值小于32的ASCII字符，包含制表符和换行符）。 由于JSON规范要求对所有控制字符使用引号，这是一个非标准的特性，因此默认禁用。

那么，哪些字符属于控制字符呢？做个简单科普：我们一般说的ASCII码共128个字符(7bit)，共分为两大类

**制字符**

控制字符，也叫不可打印字符。第**0～32号**及第127号(共34个)是控制字符，例如常见的：**LF（换行）**、**CR（回车）**、FF（换页）、DEL（删除）、BS（退格)等都属于此类。

控制字符**大部分已经废弃**不用了，它们的用途主要是用来操控已经处理过的文字，ASCII值为8、9、10 和13 分别转换为退格、制表、换行和回车字符。它们**并没有特定的图形显示**，但会依不同的应用程序，而对文本显示有不同的影响。

> 话外音：你看不见我，但我对你影响还蛮大

**非控制字符**

也叫可显示字符，或者可打印字符，能从键盘直接输入的字符。比如0-9数字，逗号、分号这些等等。

> 话外音：你肉眼能看到的字符就属于非控制字符

#### 6.~~ALLOW_BACKSLASH_ESCAPING_ANY_CHARACTER(false)~~

> 自2.10版本后，使用`JsonReadFeature#ALLOW_BACKSLASH_ESCAPING_ANY_CHARACTER`代替

否允许**反斜杠\**转义任何字符。这句话不是非常好理解，看下面这个例子：

```java
@Test
public void test4() throws IOException {
    String jsonStr = "{\"name\" : \"YourB\\'atman\" }";

    JsonFactory factory = new JsonFactory();
    try (JsonParser jsonParser = factory.createParser(jsonStr)) {
        // jsonParser.enable(JsonParser.Feature.ALLOW_BACKSLASH_ESCAPING_ANY_CHARACTER);

        while (jsonParser.nextToken() != JsonToken.END_OBJECT) {
            String fieldname = jsonParser.getCurrentName();
            if ("name".equals(fieldname)) {
                jsonParser.nextToken();
                System.out.println(jsonParser.getText());
            }
        }
    }
}
```

运行程序，报错：

```
com.fasterxml.jackson.core.JsonParseException: Unrecognized character escape ''' (code 39)
 at [Source: (String)"{"name" : "YourB\'atman" }"; line: 1, column: 19]
 ...
```

放开注释掉的代码，再次运行程序，**一切正常**，输出：`YourB'atman`

#### 7.~~ALLOW_NUMERIC_LEADING_ZEROS(false)~~

> 自2.10版本后，使用`JsonReadFeature#ALLOW_LEADING_ZEROS_FOR_NUMBERS`代替

是否允许像`00001`这样的“数字”出现（而不报错）。看例子：

```java
@Test
public void test5() throws IOException {
    String jsonStr = "{\"age\" : 00018 }";

    JsonFactory factory = new JsonFactory();
    try (JsonParser jsonParser = factory.createParser(jsonStr)) {
        // jsonParser.enable(JsonParser.Feature.ALLOW_NUMERIC_LEADING_ZEROS);

        while (jsonParser.nextToken() != JsonToken.END_OBJECT) {
            String fieldname = jsonParser.getCurrentName();
            if ("age".equals(fieldname)) {
                jsonParser.nextToken();
                System.out.println(jsonParser.getIntValue());
            }
        }
    }
}
```

运行程序，输出

```
com.fasterxml.jackson.core.JsonParseException: Invalid numeric value: Leading zeroes not allowed
 at [Source: (String)"{"age" : 00018 }"; line: 1, column: 11]
 ...
```

放开注掉的代码，再次运行程序，一切正常。输出`18`。

#### 8.~~ALLOW_LEADING_DECIMAL_POINT_FOR_NUMBERS(false)~~

> 自2.10版本后，使用`JsonReadFeature#ALLOW_LEADING_DECIMAL_POINT_FOR_NUMBERS`代替

是否允许小数点`.`打头，也就是说`.1`这种小数格式是否合法。默认是不合法的，需要开启此特征才能支持，例子就略了，基本同上。

#### 9.~~ALLOW_NON_NUMERIC_NUMBERS(false)~~

> 自2.10版本后，使用`JsonReadFeature#ALLOW_NON_NUMERIC_NUMBERS`代替

是否允许一些解析器识别一组**“非数字”(如NaN)**作为合法的浮点数值。这个属性和上篇文章的`JsonGenerator#QUOTE_NON_NUMERIC_NUMBERS`特征值是遥相呼应的。

```java
@Test
public void test5() throws IOException {
    String jsonStr = "{\"percent\" : NaN }";

    JsonFactory factory = new JsonFactory();
    try (JsonParser jsonParser = factory.createParser(jsonStr)) {
        // jsonParser.enable(JsonParser.Feature.ALLOW_NON_NUMERIC_NUMBERS);

        while (jsonParser.nextToken() != JsonToken.END_OBJECT) {
            String fieldname = jsonParser.getCurrentName();
            if ("percent".equals(fieldname)) {
                jsonParser.nextToken();
                System.out.println(jsonParser.getFloatValue());
            }
        }
    }
}
```

运行程序，抛错：

```
om.fasterxml.jackson.core.JsonParseException: Non-standard token 'NaN': enable JsonParser.Feature.ALLOW_NON_NUMERIC_NUMBERS to allow
 at [Source: (String)"{"percent" : NaN }"; line: 1, column: 17]
```

放开注释掉的代码，再次运行，一切正常。输出：

```
NaN
```

> 小贴士：NaN也可以表示一个Float对象，是的你没听错，即使它不是**数字**但它也是Float类型。具体你可以看看Float源码里的那几个常量

#### 10.~~ALLOW_MISSING_VALUES(false)~~

是否允许支持**JSON数组中**“缺失”值。怎么理解：数组中缺失了值表示两个逗号之间，啥都没有，形如这样`[value1, , value3]`。

```java
@Test
public void test6() throws IOException {
    String jsonStr = "{\"names\" : [\"YourBatman\",,\"A哥\",,] }";

    JsonFactory factory = new JsonFactory();
    try (JsonParser jsonParser = factory.createParser(jsonStr)) {
        // jsonParser.enable(JsonParser.Feature.ALLOW_MISSING_VALUES);

        while (jsonParser.nextToken() != JsonToken.END_OBJECT) {
            String fieldname = jsonParser.getCurrentName();
            if ("names".equals(fieldname)) {
                jsonParser.nextToken();

                while (jsonParser.nextToken() != JsonToken.END_ARRAY) {
                    System.out.println(jsonParser.getText());
                }
            }
        }
    }
}
```

运行程序，抛错：

```
YourBatman // 能输出一个，毕竟第一个part（JsonToken）是正常的嘛

com.fasterxml.jackson.core.JsonParseException: Unexpected character (',' (code 44)): expected a valid value (JSON String, Number, Array, Object or token 'null', 'true' or 'false')
 at [Source: (String)"{"names" : ["YourBatman",,"A哥",,] }"; line: 1, column: 27]
```

放开注释掉的代码，再次运行，一切正常，结果为：

```
YourBatman
null
A哥
null
null
```

**请注意：此时数组的长度是5哦。**

> 小贴士：此处用的String类型展示结果，是因为null可以作为String类型（`jsonParser.getText()`得到null是合法的）。但如果你使用的int类型（或者bool类型），那么如果是null的话就报错喽`Current token (VALUE_NULL) not of boolean type`，有兴趣的亲可自行尝试，巩固下理解的效果。报错原因文上已有说明

#### 11.~~ALLOW_TRAILING_COMMA(false)~~

> 自2.10版本后，使用`JsonReadFeature#ALLOW_TRAILING_COMMA`代替

是否允许最后一个多余的逗号（一定是最后一个）。这个特征是**非常重要**的，若开关打开，有如下效果：

- [true,true,]等价于[true, true]
- {“a”: true,}等价于{“a”: true}

当这个特征和上面的`ALLOW_MISSING_VALUES`特征同时使用时，本特征优先级更高。也就是说：会先去除掉最后一个逗号后，再进行数组长度的计算。

举个例子：当然这两个特征开关都打开时，[true,true,]等价于[true, true]好理解；**并且呢，`[true,true,,]`是等价于`[true, true, null]`的哦，可千万别忽略最后的这个null**。

```java
@Test
public void test7() throws IOException {
    String jsonStr = "{\"results\" : [true,true,,] }";

    JsonFactory factory = new JsonFactory();
    try (JsonParser jsonParser = factory.createParser(jsonStr)) {
        jsonParser.enable(JsonParser.Feature.ALLOW_MISSING_VALUES);
        // jsonParser.enable(JsonParser.Feature.ALLOW_TRAILING_COMMA);

        while (jsonParser.nextToken() != JsonToken.END_OBJECT) {
            String fieldname = jsonParser.getCurrentName();
            if ("results".equals(fieldname)) {
                jsonParser.nextToken();

                while (jsonParser.nextToken() != JsonToken.END_ARRAY) {
                    System.out.println(jsonParser.getBooleanValue());
                }
            }
        }
    }
}
```

运行程序，输出：

```
YourBatman
null
A哥
null
null
```

这完全就是上例的效果嘛。现在我放开注释掉的代码，再次运行，结果为：

```
YourBatman
null
A哥
null
```

**请注意对比前后的结果差异，并自己能能自己合理解释**。

### 3.校验相关

Jackson在JSON标准**之外**，给出了两个校验相关的特征。

#### 1.STRICT_DUPLICATE_DETECTION(false)

> 自2.10版本后，使用`StreamReadFeature#STRICT_DUPLICATE_DETECTION`代替

是否允许JSON串有两个相同的属性key，默认是**允许的**。

```java
@Test
public void test8() throws IOException {
    String jsonStr = "{\"age\":18, \"age\": 28 }";

    JsonFactory factory = new JsonFactory();
    try (JsonParser jsonParser = factory.createParser(jsonStr)) {
        // jsonParser.enable(JsonParser.Feature.STRICT_DUPLICATE_DETECTION);

        while (jsonParser.nextToken() != JsonToken.END_OBJECT) {
            String fieldname = jsonParser.getCurrentName();
            if ("age".equals(fieldname)) {
                jsonParser.nextToken();
                System.out.println(jsonParser.getIntValue());
            }
        }
    }
}
```

运行程序，正常输出：

```
18
28

```

若放开注释代码，再次运行，则抛错：

```
18 // 第一个数字还是能正常输出的哟

com.fasterxml.jackson.core.JsonParseException: Duplicate field 'age'
 at [Source: (String)"{"age":18, "age": 28 }"; line: 1, column: 17]
```

#### 2.IGNORE_UNDEFINED(false)

> 自2.10版本后，使用`StreamReadFeature#IGNORE_UNDEFINED`代替

是否忽略**没有定义**的属性key。和`JsonGenerator.Feature#IGNORE_UNKNOWN`的这个特征一样，它作用于预先定义了格式的数据类型，如`Avro、protobuf`等等，JSON是不需要预先定义的哦~

同样的，你可以通过这个API预先设置格式：

```java
JsonParser:

    public void setSchema(FormatSchema schema) {
    	...
    }
```

### 4.其它

#### 1.INCLUDE_SOURCE_IN_LOCATION(true)

> 自2.10版本后，使用`StreamReadFeature#INCLUDE_SOURCE_IN_LOCATION`代替

是否构建`JsonLocation`对象来表示每个part的来源，你可以通过`JsonParser#getCurrentLocation()`来访问。作用不大，就此略过。

## 总结

本文介绍了底层流式API JsonParser读JSON的方式，它不仅仅能够处理**标准JSON**，也能通过Feature特征值来控制，开启对一些非标准但又比较常用的JSON串的支持，这不正式一个优秀框架/库应有的态度麽：**兼容性**。

结合上篇文章对写JSON时`JsonGenerator`的描述，能够总结出两点原则：

- 写：100%遵循规范
- 读：最大程度**兼容**并包

写代表你的输出，遵循规范的输出能确保第三方在用你输出的数据时不至于对你破口大骂，所以这是你应该做好的**本分**。读代表你的输入，能够处理规范的格式是你的职责，但我若还能额外的处理一些非标准格式（一般为常用的），那绝对是闪耀点，也就是你给的**情分**。

