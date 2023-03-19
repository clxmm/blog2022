---
title: 36整合swagger-在线文档与swagger
toc: true
tags: springboot
categories: 
    - [java]
    - [springboot]
---

小册进行到这里，我们已经讲解了好多好多项目开发中经常用到的整合技术。随着项目的开发过程中，在线文档和调试就显得尤为重要了，它不仅是供前后端开发人员之间交流的纽带，也是后端开发人员用来测试自己功能的利器。接下来的两章我们就来看看在线文档的有关整合。

## 1. 在线文档

<!--more-->

基于前后端分离的项目开发中，前端和后端是分开开发项目，为此就需要一个接口文档来串起前后端的接口功能说明。传统的做法是手工将每个接口的定义信息全部写在一个 Word 文档或者在线协同文档中，这样写最大的问题是同步问题，如果接口定义发生变动，文档很有可能更新不及时，造成前后端对接效率降低。

基于这种痛点，我们肯定需要一个可以实时展示接口定义的工具，也就是所谓的在线文档。通常情况下，在线文档会有如下的特点：

- 即时的（看到的文档就是接口里真真实实写的）
- 可交互的（不像传统文档那样瀑布式排列）
- 可调试的（可以提供在线调试接口的功能）

对比市面上的几款文档工具可以发现，在线文档的生成方式主要包含两种方式：

- 要么通过解析接口代码上的特定注解信息生成，典型代表是 Swagger
- 要么通过解析接口代码的文档注释生成，典型代表是 SmartDoc

本小册选择介绍 Swagger 在线文档的技术使用，主要原因是它可以生成可以在线调试的文档，且文档是跟随项目的启动随即生成，在开发阶段相对方便。

## 2. Swagger

那下面我们就来了解和使用 Swagger 。Swagger 本身基于一些基本的规范，我们先得了解这些基本规范才可以愉快地展开后续使用。

### 2.1 前行规范

##### 2.1.1 RESTful规范

在好几年前，Java 开发圈子里的一位大佬阮一峰发表过一篇文章 [《理解RESTful架构》](http://www.ruanyifeng.com/blog/2011/09/restful.html) ，在这篇文章中他解释了 RESTful 的一些相关的概念，小册在这里也引用这篇文章的一些内容来讲解。

RESTful ，全名叫 Representational State Transfer ，表现层状态转化 。这个名听起来很抽象，没关系，咱用比较容易理解的话来解释。

HTTP 协议中不是有 8 种请求方式嘛，里面包含 **GET POST PUT DELETE** ，RESTful 的编码风格将这四种方式赋予真正的意义：

- **GET ：获取资源 / 数据**
- **POST ：新建资源 / 数据**
- **PUT ：更新资源 / 数据**
- **DELETE ：删除资源 / 数据**

注意，RESTful 只是一种编码的风格，**不是标准、不是协议、不是规范**，仅仅是风格而已。只是说，如果用这种编码风格的话，设计的 API 请求路径看上去更简洁、有层次性。

对于目前来讲，我们说的 RESTful 风格的编码，一般都是在接口 API 层面的 RESTful ，也就是基于 uri 的 RESTful 。这种风格的编码，小册给各位列一个对比：

| 请求路径定义 | 普通接口请求路径                    | RESTful请求路径         |
| ------------ | ----------------------------------- | ----------------------- |
| 根据id查部门 | 【GET】/department/findById?id=1    | 【GET】/department/1    |
| 新增部门     | 【POST】/department/save            | 【POST】/department     |
| 修改部门     | 【POST】/department/update          | 【PUT】/department/1    |
| 删除部门     | 【POST】/department/deleteById?id=1 | 【DELETE】/department/1 |

能看出来这样路径的对比吧，我们平时写的这些普通的接口请求路径，都是通过路径名称来区分动作的；而 RESTful 的请求路径是通过 HTTP 请求方式来区分的。另外，传参的方式也不一样，我们写的普通接口，传参 id 是放在 url 后面拼接参数，而 RESTful 请求的传参是直接写在 uri 上的，所以这样看上去才比较简洁。

相对比而言，Swagger 更希望我们使用遵循 RESTful 规范的方式来编写接口，目的是回头在渲染接口文档的时候能够更好地区分开这几种动作。

##### 2.1.2 OpenAPI规范

RESTful 是基于接口编码的规范，那 OpenAPI 定义的就是接口文档内容的规范了。一个接口文档应该如何编写才能规整的生成，并且还能做到容易阅读和使用，这就是 OpenAPI 考虑的事了。

简单的说，OpenAPI 规范下的接口文档，本身是一套 json 数据，它可以使用 json 或者 yaml 的形式表示出来。OpenAPI 文档的编写方式有些麻烦，而且它内部的元素很多，所以这里小册就不展开说了，感兴趣的小伙伴可以参照 OpenAPI 的规范格式简单了解（对应的规范文档在这里 [swagger.io/specificati…](https://swagger.io/specification/) ），我们还是重点看 Swagger 的功能。

### 2.2 SpringBoot整合Swagger2.9

首先我们先来整合一下 Swagger 吧。原生的 Swagger 文档有 2 个比较常用的版本，分别是 2.9 和 3.0 ，我们挨个来整合。

##### 2.2.1 创建工程

与前面的方式相似，我们先来创建一个新的模块 `spring-boot-integration-10-swagger` 。

我们首先整合 Swagger 2.9 ，这个版本通常都是整合相对老的 SpringBoot 版本（阿熊之前拿 SpringBoot 2.1.x 的时候整合的就是 Swagger 2.9 ）。这个版本在导入依赖时，只能是导入 Swagger 原生的两个坐标：swagger 的核心包和 UI 包（此外不要忘记导入 web 依赖）。

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>io.springfox</groupId>
        <artifactId>springfox-swagger2</artifactId>
        <version>2.9.2</version>
    </dependency>
    <dependency>
        <groupId>io.springfox</groupId>
        <artifactId>springfox-swagger-ui</artifactId>
        <version>2.9.2</version>
    </dependency>
</dependencies>
```

##### 2.2.2 编写配置

因为不是 starter 场景启动器那样，所以我们需要手动编写一些配置，我们创建一个配置类 `Swagger2Configuration` ，并在这其中注册一个 `Docket` 对象，如下面的代码所示。由于创建 `Docket` 时需要传入 Swagger 文档的一些基础信息（诸如文档标题、简介、作者、联系方式、版本等），以及要扫描的 RestController 的位置，而这一切都可以使用链式方法调用的方式来完成，非常舒适。

```java

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import springfox.documentation.builders.ApiInfoBuilder;
import springfox.documentation.builders.PathSelectors;
import springfox.documentation.builders.RequestHandlerSelectors;
import springfox.documentation.service.ApiInfo;
import springfox.documentation.service.Contact;
import springfox.documentation.spi.DocumentationType;
import springfox.documentation.spring.web.plugins.Docket;
import springfox.documentation.swagger2.annotations.EnableSwagger2;

@Configuration(proxyBeanMethods = false)
@EnableSwagger2
public class Swagger2Configuration {

    @Bean
    public Docket createRestApi() {
        return new Docket(DocumentationType.SWAGGER_2)
                .apiInfo(apiInfo())
                .select()
                .apis(RequestHandlerSelectors.basePackage("org.clxmm.swagger2.controller"))
                .paths(PathSelectors.any())
                .build();
    }

    private ApiInfo apiInfo() {
        return new ApiInfoBuilder()
                .title("SpringBoot手动集成Swagger2")
                .description("test swagger document")
                .contact(new Contact("https://github.com/LinkedBear", "", ""))
                .version("1.0")
                .build();
    }
}
```

如此编写完毕后，其实我们要做的配置就完成了。

##### 2.2.3 编码测试

下面我们来编写一个简单的接口来试一下效果。创建一个 Controller ，并标注 `@RestController` 注解，随后声明一个 GET 方式请求的接口方法即可。

```java
@RestController
public class DemoController {
    
    @GetMapping("/test")
    public String test() {
        return "test";
    }
}
```

这样编码就完成了，下面编写 SpringBoot 的主启动类并启动即可。

启动工程后，我们来访问 [http://localhost:8080/swagger-ui.html](http://localhost:8080/swagger-ui.html) ，即可打开如下所示的 Swagger 接口文档，它是一个绿色基调的页面。

![](../img/202303/36swagger1.png)

在页面上我们可以看到它识别出来了一个 controller ，就是我们刚写的那个 `DemoController` ，并且解析出来了那个 GET 方式请求的 `/test` 接口。

如果我们点开这个 `/test` 接口，可以发现它描述的内容包含请求参数、返回结果和状态码描述。在接口的右上角还有一个 `“Try it out”` 的按钮，意思是可以在线调试！

而当我们点击 `Try it now` 按钮后，会出现一个 `Execute` 的按钮，点击之后就相当于 Swagger 帮我们模拟发送了一次 GET 方式的 `/test` 请求。点击之后，可以发现下面把模拟发送的命令也帮我们生成出来了，并且在下面的 `Server response` 栏目中可以看到，这次请求返回了 200 状态码，并且也成功拿到了响应的内容 `“test”` 字符串，包括响应头信息。

由此可见，Swagger 文档还是很好用的，能解析我们编写的接口，同时还能在线调试，属实是非常香了。

### 2.3 SpringBoot整合Swagger3.0

不过话说回来，现在用低版本 SpringBoot 的小伙伴应该已经很少了吧，我们还是重点来看相对高一点版本的 SpringBoot 整合同样高一点的版本 Swagger 3.0 ，看看它整合起来会不会简单点。

##### 2.3.1 更换坐标

Swagger 3.0 终于懂得了整合 SpringBoot 的重要性，这次它官方就提供了场景启动器 `springfox-boot-starter` 。

```xml
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-boot-starter</artifactId>
    <version>3.0.0</version>
</dependency>
```

##### 2.3.2 测试效果

既然有场景启动器，那是不是我们啥也不用编写就行呢？答案是可以的，有自动装配的加持后，我们不再需要编写配置类、声明注解。

我们直接重新编译、启动 `SpringBootSwaggerApplication` ，这次访问的路径就不一样了，我们需要访问的是 [http://localhost:8080/swagger-ui/index.html](http://localhost:8080/swagger-ui/index.html) 。

这次访问之后可以发现，界面相比起 Swagger 2 来讲大同小异，主要是色调变了，文档上的一些文字（文档名称、简介等）变成默认的了。另外还有一个细节，这里它还扫描到了一个 `BasicErrorController` ，这是 SpringWebMvc 内置的用于响应异常和错误数据的 Controller （了解基础的小伙伴肯定一下子就反应过来了）。

![](./img/202303/36swagger2.png)

默认的终归是不大好，我们还是需要定制一下吧。

##### 2.3.3 修改文档主要信息

本着约定大于配置的原则，那我们就手动配置一下，让约定的自动装配退出。对比 Swagger 2 来看，基于 Swagger 3 的配置并没有什么区别，我们只需要拿来稍作改动即可，改动的点主要在于不需要加注解，以及创建 `Docket` 时需要传入的 `DocumentationType` 类型不一样。

```java
@Configuration(proxyBeanMethods = false)
public class Swagger3Configuration {
    
    @Bean
    public Docket createRestApi() {
        return new Docket(DocumentationType.OAS_30).apiInfo(apiInfo())
                .select()
                .apis(RequestHandlerSelectors.basePackage("org.clxmm.swagger2.controller"))
                .paths(PathSelectors.any()).build();
    }
    
    private ApiInfo apiInfo() {
        return new ApiInfoBuilder()
                .title("SpringBoot自动集成Swagger3")
                .description("test swagger document")
                .contact(new Contact("LinkedBear", "https://github.com/LinkedBear", ""))
                .version("1.0")
                .build();
    }
}
```

这样再启动后，文档的内容就会改成我们配置的那样了。

## 3. 使用Swagger

---

下面我们来展开说说 Swagger 中的核心注解使用。Swagger 的文档展示什么内容，取决于我们编写在 Controller 、Entity 实体类等上面的 Swagger 注解都写了什么。为此，我们需要了解这些 Swagger 注解的意义。

### 3.1 @Api

Swagger 汇总接口是按照一个一个的 RestController 类分组的，默认情况下生成的接口分组名就是 Controller 名，如果我们需要修改这个接口分组的名，可以使用 `@Api` 注解来标注。

`@Api` 注解用在请求的 RestController 类上，表示对类的说明，下面我们来简单试一下。找到我们刚写好的 `DemoController` 类，在上面标注 `@Api` 注解，并指定 `tags` 属性（注意不要用 `value` ，写了在页面上也看不见）。

```java
@RestController
@Api(tags = "这是一个测试接口类")
public class DemoController {
    
    // ......
}
```

完事我们重启一下工程，刷新 Swagger 文档，可以发现接口分组名已经变了。

### 3.2 @ApiOperation

Controller 的名称被规范化了，那每个接口的名称如何规范呢？这就需要用到下一个注解：`@ApiOperation` 。这个注解有 3 个主要使用的属性，其中 `value` 用于标注这个接口的名称，`notes` 相当于 comment / description 备注，比方说我们上面写的 test 方法就可以这样编写方法文档。

```java
    @GetMapping("/test")
    @ApiOperation(value = "这是一个测试的接口", notes = "仅供测试，切勿当真")
    public String test() {
        return "test";
    }
```

如此编写后重启工程，刷新 Swagger 文档就可以发现，我们编写的这些内容都上来了。

另外还有一个属性可能会用到：`hidden` ，不用我说你也懂得这是什么意思，如果一个接口我们不希望它生成接口文档，那就需要标注 `hidden = true` 来隐藏它。

### 3.3 @ApiParam

上面我们编写的 `test` 方法没有入参，那如果我们写的接口有入参，Swagger 会如何给我们渲染呢？

我们可以在 `test` 方法上添加两个参数：

```java
    public String test(String name, Integer age) {
        System.out.println(name);
        System.out.println(age);
        return "test";
    }
```

相当于我们在调用 `/test` 接口时，可以传入 name 和 age 两个参数了。重启工程，刷新 Swagger 文档，可以发现在 Parameters 栏中多出来两个参数，而且这两个参数的位置、数据类型都解析出来了：

如果我们需要描述这两个参数的意义，那就要用到接下来的这个注解了：`@ApiParam` ，这个注解专门用来修饰一个简单方法参数，这里面的常用属性有不少，我们快速来看一遍。

| 属性名称        | 属性类型 | 默认值 | 作用                     |
| --------------- | -------- | ------ | ------------------------ |
| name            | String   | 空     | 定义参数的名称           |
| value           | String   | 空     | 定义参数的简单描述       |
| defaultValue    | String   | 空     | 定义参数的默认值         |
| allowableValues | String   | 空     | 定义参数的取值范围       |
| required        | boolean  | false  | 定义参数是否必填         |
| allowMultiple   | boolean  | false  | 定义参数能否接收多个数值 |
| hidden          | boolean  | false  | 定义参数是否展示在文档上 |
| example         | String   | 空     | 给出一个合法的参数值示例 |

对应的，下面提供一个 `@ApiParam` 注解的使用示例：

```java
@GetMapping("/test")
@ApiOperation(value = "这是一个测试的接口", notes = "仅供测试，切勿当真")
public String test(@ApiParam(value = "测试姓名", defaultValue = "zhangsan",
                             allowableValues = "zhangsan,lisi,wangwu", required = true) String name,
                   @ApiParam(value = "测试年龄", allowableValues = "10, 20, 30, 40, 50", example = "10") Integer age) {
  System.out.println(name);
  System.out.println(age);
  return "test";
}
```

如此编写后，重启工程并刷新 Swagger 文档，可以发现我们编写的内容都体现在文档上了：

### 3.4 @ApiModel与@ApiModelProperty

如果我们的接口在接收请求参数，或者响应数据时使用到了模型类 / VO 类，那如何将模型类中的那些属性都描述清楚，就成了必须要考虑的事了。Swagger 自然也帮我们考虑到了，它提供了一对注解来实现：`@ApiModel` 与 `@ApiModelProperty` ，从字面上也很好理解，它就是分别解释整个类，和类中的各个属性的。

下面我们也来试验一下。首先我们来编写一个实体类，并给这上面标注 `@ApiModel` 与 `@ApiModelProperty` 注解，声明这些属性的含义、可接收的参数类型，以及备注等。

```java
@ApiModel(value = "用户类", description = "封装用户的基本信息")
public class User {
    
    @ApiModelProperty(value = "用户id", required = true)
    private String id;
    
    @ApiModelProperty(value = "用户姓名", required = true)
    private String name;
    
    @ApiModelProperty(value = "用户年龄", notes = "只能传入大于0的数")
    private Integer age;
    
    // getter setter ......
```

紧接着，我们需要再写一个新的接口，用于接收这个实体模型类的参数，接口的文档我们就不写了，上面已经讲解过了。

> （为了方便各位查看实体类作为入参和返回值的不同效果，接口声明时小册选择 “接什么吐什么” 了）。

```java
    @PostMapping("/save")
    public User save(User user) {
        System.out.println(user);
        return user;
    }
```

之后直接重启工程即可，刷新 Swagger 文档，可以发现多了一个 POST 请求的接口，点开可以发现，我们声明的 `User` 对象在请求入参时被解析掉了，取而代之的是 `User` 对象中的各个属性；而在响应信息定义中，我们可以发现完整的 `User` 又被作为 Schema 体现在文档上了。

先不要着急归因，各位注意，我们继续往下看。如果在 `save` 方法的 `User` 入参上添加一个 `@RequestBody` 注解，再次重启工程并刷新 Swagger 文档，可以发现这次请求体的格式也变成 `User` 类的 Schema 了，与上面 Response 的部分一模一样！

这看上去很奇怪的现象是为何呢？为什么 Swagger 会解析出两种完全不同的文档内容呢？

要想理清楚这个原因，还需要各位回忆一下 SpringWebMvc 部分的内容。不标注 `@RequestBody` 时，WebMvc 解析参数的方式是从 form 表单域中提交，此时接受的 `content-type` 为 `application/x-www-form-urlencoded` ，此时我们看到的效果就相当于是把模型类中的每个属性都单独列出来；而标注 `@RequestBody` 后，接受的 `content-type` 变为 `application/json` ，此时接收的数据是来自请求体的 json 字符串，Swagger 自然就把 json 数据的结构（也就是 `User` 模型）给我们呈现出来了。

