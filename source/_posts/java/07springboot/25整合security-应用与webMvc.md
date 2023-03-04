---
title: 25整合security-应用与webMvc
toc: true
tags: springboot
categories: 
    - [java]
    - [springboot]
---

通过上一章的内容，想必各位已经对 SpringSecurity 有了一个基本的认识和回顾，本章内容我们就来实际的将 SpringSecurity 整合到 WebMvc 的项目中，实际体会一下 SpringSecurity 在 Web 项目中是如何发挥作用的。

## 1. SpringBoot整合Security的简单使用

通常来讲 SpringSecurity 整合 WebMvc 时有两种开发场景：

- 基于前后端绝对分离的项目，这种项目下后端的 Controller 只需要提供接口即可；
- 基于前后端不分离的项目，这种项目依然需要后端的 Controller 负责页面渲染、跳转等。

<!--more-->

由于目前的开发中更多的是基于前后端分离的项目，所以本小册选择以这种场景为例讲解。

#### 1.1 工程创建

我们先来创建一个工程，工程名为 `spring-boot-integration-07-security-webmvc` ，在这其中需要导入 webmvc 和 security 的依赖，不再导入 test 测试依赖。

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
</dependencies>
```

#### 1.2 测试接口制作

```java
@RestController
public class DemoController {
    
    @GetMapping("/demo")
    public String demo() {
        return "demo";
    }
    
    @GetMapping("/getData")
    public String getData(String name) {
        return "userData-" + name;
    }
}
```



```java
@RestController
@RequestMapping("/api")
public class ApiController {
    
    @GetMapping("/test")
    public String test() {
        return "test";
    }
}
```

稍后我们在进行权限控制时，就针对上述的 3 个接口来限制。

#### 1.3 测试默认自动装配的效果

代码我们先写到这里，观察一下此时此刻直接启动工程的话，效果会是什么样。

主启动类中还是最经典的写法，不需要写任何多余的配置和注解。

```java
@SpringBootApplication
public class Demo7App {


    public static void main(String[] args) {
        SpringApplication.run(Demo7App.class);
    }
}
```

运行 `SpringBootSecurityApplication` ，注意观察控制台中打印了如下两行信息：

```ini
Using generated security password: 5112d051-051a-4fc7-8974-f997f0307381

This generated password is for development use only. Your security configuration must be updated before running your application in production.

```

这是个什么东西？为什么会生成一个 uuid 类型的密码呢？为什么控制台打印的信息中告诉我们这个密码只能用于开发呢？不要着急，我们先把问题记下，一会再逐个击破。

下面我们访问 [http://localhost:8080/demo](http://localhost:8080/demo) ，可以发现浏览器被引导跳转到了一个很陌生的表单登录页：

这个页面我们可没有自己写，是 SpringSecurity 给我们默认带的，在登录时用户名输入 **`user`** ，密码就是控制台上的那个 uuid 密码，登录之后就可以正常访问 `/demo` 接口了。

由此可见，SpringBoot 对于安全访问控制的默认规则是：**提供一个内置账号 `user` ，并当 `user` 登录成功后，放开所有接口的访问**。

#### 1.4 Security的配置

SpringBoot 整合 SpringSecurity 时，需要单独编写一个配置类，使其继承 `WebSecurityConfigurerAdapter` ，并标注 `@EnableWebSecurity` 注解。(不标注也会生效)

```java
@EnableWebSecurity
@Configuration
public class WebSecurityConfiguration extends WebSecurityConfigurerAdapter {
    
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        
    }
}
```

注意代码中留白的 `configure` 方法（小心参数类型为 `HttpSecurity` ，不是其他几个重载的方法），这个方法中可以配置的内容非常非常多，我们可以先什么都不做（相当于不重写）。

编写完这个配置类后，我们再访问 [http://localhost:8080/demo]() ，可以发现 `"demo"` 字符串被正常返回，这说明我们向 IOC 容器注册了一个 `WebSecurityConfigurerAdapter` 的子类时，SpringBoot 就不会再给我们装配默认的权限访问控制了（即默认的约定设置退出了）！

#### 1.5 加入认证限制

非常浅显易懂的道理，如果一个系统中的所有接口都可以被任意访问，那么内部的数据就会被随意访问，同时也极易受到攻击。为此我们需要编写配置，限制用户的行为。

我们先加入认证的限制，复原 SpringBoot 在最初给我们提供的认证限制：只有用户登录系统后，才可以正常访问这些接口获取数据。想要限制所有接口都需要认证后访问，就需要在上面重写的 `configure` 方法中编写配置代码了。

```java
@Override
protected void configure(HttpSecurity http) throws Exception {
  // 限制所有接口都需要认证后访问
  http.authorizeRequests().anyRequest().authenticated();
  // 开启表单登录功能
  http.formLogin();
}
```

注意这里我们需要做两件事，使用 `http.authorizeRequests()` 可以将所有的需求设置认证后访问，而开启表单登录是为了能在检测到用户没有登录时，跳转到 SpringSecurity 默认提供的登录页。

配置完成后重新启动，可以发现再访问 `/demo` 请求时就又可以看到刚才的登录页了。

> 注意一个细节，此处先剧透一下，在 SpringSecurity 的默认登录页中有一个看上去比较莫名其妙的隐藏 input ，名称是 `_csrf` ，value 是一串 uuid 。这个东西是 SpringSecurity 默认带入的，后面我们还会碰到它。

#### 1.6 加入授权限制

下面我们再试一下给接口限制权限，比方说我们限制所有 `/api` 开头的请求，需要拥有 `admin` 角色才能访问，否则拒绝请求。对应到 SpringSecurity 的配置中就需要这样写：

```java
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        // 开启表单登录功能
        http.formLogin();
        // 限制/api开头的接口需要拥有admin角色，其余接口需要认证后访问
        http.authorizeRequests().antMatchers("/api/**").hasRole("admin").anyRequest().authenticated();
    }
```

如此编写之后我们再次重启工程，在登录完成后访问 `/api/test` 接口就会出现 403 权限不足的错误提示。

怎么能让 `/api` 开头的请求能够访问呢，这就需要我们引入用户管理的机制了。SpringSecurity 默认只会注册 1 个用户，且用户名为 `user` ，密码为随机生成的 uuid ，不带任何角色，这显然是不够用的。为此，我们要自己维护一套用户管理的体系。

在 `configure` 方法中，可以使用 `http.userDetailsService(userManager);` 方法指定一个用户管理器，随后我们用 `@Bean` 的方式注册一个 `UserDetailsService` 的实现类对象即可。现阶段为了快速演示如何引入用户管理体系，所以我们选择基于内存的用户管理器；另外对于密码而言，存明文是肯定不可能也是不允许的，所以我们还需要一个密码的加密器，也就是下面代码中的 `PasswordEncoder` ，SpringSecurity 推荐我们使用 `BCryptPasswordEncoder` 作为密码加密器，所以我们直接拿来注册即可。

```java
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        // 开启表单登录功能
        http.formLogin();
        // 限制所有接口都需要认证后访问
        // http.authorizeRequests().anyRequest().authenticated();
        // 限制/api开头的接口需要拥有admin角色，其余接口需要认证后访问
        http.authorizeRequests().antMatchers("/api/**").hasRole("admin").anyRequest().authenticated();
        // 引入用户管理机制
        http.userDetailsService(userDetailsService());
    }
    
    @Bean
    public UserDetailsService userDetailsService() {
        InMemoryUserDetailsManager userManager = new InMemoryUserDetailsManager();
        userManager.createUser(User.withUsername("xiaoshuai").password(passwordEncoder().encode("123456")).roles("admin").build());
        userManager.createUser(User.withUsername("xiaoming").password(passwordEncoder().encode("654321")).roles("user").build());
        userManager.createUser(User.withUsername("boss").password(passwordEncoder().encode("123456")).roles("admin", "manager").build());
        return userManager;
    }
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
```

如此配置完成后，就可以再来测试了。重新运行 `SpringBootSecurityApplication` ，访问 `/api/test` 接口，跳转到登录页后，使用 `xiaoshuai` 或者 `boss` 登录是可以正常访问的，而使用 `xiaoming` 则无法访问。

> 其实默认情况下 SpringSecurity 给我们内置的 `user` 用户也是可以修改用户名、密码和角色的，对应的示例配置如下。
>
> ```properties
> spring.security.user.name=user
> # 注意配置的密码是加密后的，此处仅为示例
> spring.security.user.password=000000
> spring.security.user.roles=admin
> ```
>
> 另外还有一点，在给用户设置角色时，使用大写的形式更好，因为在底层构造角色标识时，会在我们传入的角色拼接 **`ROLE_`** 前缀。

到这里，我们一个最简单的 SpringBoot 整合 SpringSecurity 的工程就做好了，下面就其中一些 demo 级的内容，做更详细的讲解。

 

## 2. SpringSecurity的认证机制

---

本章我们先聊跟认证相关的，在实际的前后端分离项目中，认证阶段包含几个细节，一一来说。

### 2.1 登录方式

SpringSecurity 为我们提供了好多种用户登录的方式，包括：

- **formLogin** ：普通表单登录【单体应用最常用】
- **httpBasic** ：基于 http 协议定制的最基础认证方式，安全性低
- **oauth2Login** ：基于 OAuth 2.0 协议的授权码模式的认证实现，它常用于第三方项目整合的场景中
- **saml2Login** ：SpringSecurity 5.2 之后新扩展的登录方式，基于 SAML 2.0 网络跨域的单点登录协议实现

可以看得出来，基于 SpringSecurity 提供的登录方式中，无论我们开发的项目是前后端分离与否，都推荐使用表单登录的方式。

### 2.2 注销动作

与登录相对的动作必然是注销，即退出登录状态。SpringSecurity 默认给我们提供了注销接口 `/logout` ，通过使用 GET 请求访问 `/logout` ，就可以跳转到确认注销的页面。

如果我们点击 Log Out 按钮，则相当于发送了一个 POST 请求的 `/logout` ，此时就代表真的注销成功了。

> 请注意一个细节，如果我们使用 Postman 相关的工具来发起 `/logout` 请求时，通常会遇到 403 的问题，这是由于 SpringSecurity 有 CSRF 的拦截，这也就是刚才在讲解默认表单登录页中遇到的那个东西。怎么解决呢？下面我们马上来讲。

### 2.3 基于前后端分离的接口登录发起

下面我们根据实际开发的项目来思考：前后端分离的项目中，登录只是一个普通的 post 请求，传入 **username** 和 **password** 即可，这就产生了需求：前端发起请求后，SpringSecurity 还是否能够正常登录？

下面我们使用 Postman 或类似的工具来尝试发起一个登录请求（阿熊使用的是 ApiFox ），却发现拿到的响应信息是一个 html ，再仔细看来，这个 html 就是我们刚才一直在看的表单登录页：

下面我们使用 Postman 或类似的工具来尝试发起一个登录请求（阿熊使用的是 ApiFox ），却发现拿到的响应信息是一个 html ，再仔细看来，这个 html 就是我们刚才一直在看的表单登录页：

![](./img/202302/25sb-security.png)

为什么会出现这种情况呢？注意仔细观察下面红框中的 form 表单，那个 `_csrf` 的隐藏域又出现了，而且这次的 uuid 值还跟之前不一样！这到底是怎么回事呢？

#### 2.3.1 CSRF攻击

下面我们就不得不讲解一下 CSRF 到底是什么了。CSRF 全名为 **Cross-site request forgery** ，意为 **“跨站请求伪造”** ，是一种防范难度很大的网络攻击方式。既然这种攻击方式难度很大，到底大在哪里呢？我们可以通过一个简单的图示来解释。

![](./img/202302/25sb-security1.png)

由上述的流程可以看出，危险网站通过这种特殊的机制，构造一个危险的 url 链接并诱导用户点击，因为此时用户还在访问着金融 / 银行的网站，仍然处于登录状态，若点击链接就有可能发生财产损失或信息泄露。

SpringSecurity 提供的 CSRF 保护机制会保护所有的 POST 、PUT 、DELETE 请求，这三类请求在发起时都会校验是否携带了 CSRF 相关的信息（也就是上面看到的那个隐藏域 `_csrf` ），如果没有携带，则会响应 403 错误。

#### 2.3.2 关闭CSRF

CSRF 是很危险，但是现在的问题是 SpringSecurity 给我们开启了 CSRF 防护，我们通过接口访问的方式没办法登录了，怎么办？

最简单的办法，我们把 CSRF 关闭就可以了。在 `configure` 方法中，我们可以通过 `HttpSecurity` 来关闭 CSRF 防护：

```java
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        // ......
        // 关闭CSRF
        http.csrf().disable();
    }
```

重启工程，再次用接口工具访问 `/login` 尝试登录，可以发现这次返回了一段 json 数据：

```java
{
    "timestamp": "2023-02-27T12:46:02.324+00:00",
    "status": 404,
    "error": "Not Found",
    "path": "/"
}
```

看上去这段 json 数据很莫名其妙，这是因为我们没有写 `/` 请求对应的 Controller 方法，我们可以尝试发一个 `/demo` 请求，这个接口是可以正常访问并返回字符串 `"demo"` 的，证明刚才的登录动作已经成功，CSRF 没有再拦截我们。

#### 2.3.3 基于前后端分离的CSRF处理

虽说关了是关了，但毕竟关了 CSRF 还是不安全的，我们还是希望 CSRF 能起作用。在传统的前后端不分离页面中，我们可以借助 jsp 的动态标签，引入 security 的标签，就可以引入 CSRF 的隐藏域了。可我们现在使用的是前后端分离的项目，这种动态标签的想法直接破灭了，那还有什么办法吗？

SpringSecurity 肯定帮我们考虑到了，虽然每个表单中没办法引入 CSRF 的隐藏域，但我们还可以在请求头和 cookie 上做文章呀！SpringSecurity 支持将 CSRF 的 uuid 写到 Cookie 中，当我们发起登录、注销和其他 POST 等类型的请求时，是可以通过读取 Cookie 中的 CSRF 标识，构造到表单请求参数中，这样还是可以做到 CSRF 防护的。

下面我们具体演示一下。这次就不使用 `disable` 禁用 CSRF 了，而是设置一个 CSRF 的 token 仓库，指定 CSRF 的标识放入 Cookie 中，也就是下面代码的写法。

```java

```

如此这样编写后，我们重启工程，再次访问 `/login` 请求时会发现登录依然失败，但是 cookie 中多了一个 **`XSRF-TOKEN`** ！

![](./img/202302/25sb-security2.png)

各位，到这里是否能想到后续如何处理？既然我们拿到了一个 httponly 为 false 的 cookie ，那么在 js 中就可以读到这个 cookie 的值，从而在 form 表单提交时，动态将这个 CSRF 的标识注入到表单中，这样就实现了防止 CSRF 攻击！

我们赶紧尝试一下，在接口工具中，给提交的表单中添加 `_csrf` 的参数，重新发起请求，可以发现这次登录成功，拿到了我们期望的 404 编码的 json 数据！

### 2.4 登录后的处理

下面我们继续思考：与传统不分离项目不同的是，前后端分离的项目不需要在登录成功后跳转页面，而是需要在登录请求的响应域体现出登录成功的信息，这就产生了需求：如何修改登录后的行为？

SpringSecurity 给我们提供的配置入口，就是在开启表单登录的那个 `http.formLogin()` 方法中，这个方法实际上是给我们提供了一个 `FormLoginConfigurer` 对象，通过调用该对象的方法可以完成有关表单登录的链式配置。借助 IDE 可以发现，它可以配置的内容大概包含以下几个方面：

- 表单登录的 `username` 和 `password` 属性的名（如果我们使用外置表单登录页，有可能表单的 input 标签指定的不是 SpringSecurity 默认的 "username" 和 "password" ，有可能是 user 和 pwd ，这种情况就需要我们调用 `usernameParameter` 和 `passwordParameter` 方法来自行指定）；
- 自定义表单登录页的 uri （大多数情况下，我们一定会使用自定义的外置表单登录页，而不是用 SpringSecurity 给我们提供的如此简陋的登录页，那么就需要我们调用 `loginPage` 方法来指定跳转到自定义表单登录页的 uri ）；
- 发起登录的 uri（SpringSecurity 默认的登录和注销 uri 是 `/login` ，且是 POST 请求，如果我们需要自行指定登录的接口地址，那就需要通过调用 `loginProcessingUrl` 方法来指定）；
- 登录成功和失败的跳转 uri （当我们发起登录动作时，结果必定是要么成功，要么失败，那如果是跳转页面的话，SpringSecurity 默认给我们跳转的是 `/` 和 `/login?error` ，如果我们需要修改跳转地址，则需要通过调用 `defaultSuccessUrl` 和 `failureUrl` 方法来重新指定）；
- 登录成功和失败的处理器（即行为动作，因为默认的动作都是跳转页面，我们可以通过自定义登录成功或失败的处理器，来重写默认的行为）。

大体梳理了可以配置的内容，很明显我们要修改的是登录成功和失败的行为动作，这个配置是需要调用 `successHandler` 和 `failureHandler` 方法来实现的。

> 为了下面演示方便，从现在开始我们禁用 CSRF 防护机制。

具体怎么使用呢？我们可以在外部定义相应的 `successHandler` 和 `failureHandler` ，当然也可以直接在方法内部使用 lambda 表达式快速构造：

```java
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        // 开启表单登录功能
        http.formLogin().successHandler((request, response, authentication) -> {
            // ？？？
        }).failureHandler((request, response, exception) -> {
            // ？？？
        });
        // ......
    }
```

至于 lambda 表达式内部都能写什么，完全取决于我们想怎么办了，因为这相当于接管了 SpringSecurity 默认的登录成功和失败的处理行为。我们可以简单的写一个登录成功和失败的逻辑，比方说登录成功时提示 “登录成功” 的字样，登录失败时提示 “登录失败” 。

```java
   // 开启表单登录功能
    http.formLogin().successHandler((request, response, authentication) -> {
        response.setContentType("application/json;charset=utf-8");
        response.getWriter().write("登录成功");
    }).failureHandler((request, response, exception) -> {
        response.setContentType("application/json;charset=utf-8");
        response.getWriter().write("登录失败");
        exception.printStackTrace();
    });
```

另外注意一点，`successHandler` 和 `failureHandler` 的 lambda 表达式参数中分别包含一个 `authentication` 和 `exception` ，这分别代表认证成功后的用户信息，以及认证失败时内部产生的异常（如用户名不存在、用户名密码错误等）。

配置完成后，下面我们可以来测试一下。我们可以先测试认证成功时的效果，我们传入正确的用户名和密码，可以发现响应的数据的确是 ‘登录成功“ 的字样，而且通过 debug 可以看出，authentication 对象内部包含我们登录的用户名，以及它拥有的角色（此处可以验证我们传入的角色是要拼接 `ROLE_` 前缀的）。

![](./img/202302/25sb-security3.png)

下面再测试认证失败，我们可以随意改动用户名或者密码的其中一个，可以发现此时响应的信息变成了 ”登录失败“ ，而且 debug 看到的 `exception` 信息也体现出了 ”用户名或密码错误“ 的提示。

### 2.5 基于数据库的用户管理

本章的最后我们来做一个贴近实际项目开发的工作，截止到现在我们演示的效果全是基于我们在代码中硬编码构造的几个用户：

```java
    @Bean
    public UserDetailsService userDetailsService() {
        InMemoryUserDetailsManager userManager = new InMemoryUserDetailsManager();
        userManager.createUser(User.withUsername("xiaoshuai")......build());
        userManager.createUser(User.withUsername("xiaoming")......build());
        userManager.createUser(User.withUsername("boss")......build());
        return userManager;
    }
```

但是实际开发中绝对不会有这么个干法，我们都是将用户、角色、权限的相关数据保存到数据库中，在认证和授权阶段读库来实现。这一小节我们就来实现基于数据库的用户管理。

#### 2.5.1 初始化数据库

数据库的话我们就依然使用之前在 SpringData 的章节中用过的 `springboot-dao` 数据库，只不过数据库表就不能用那些了，我们来创建一张新的表加以区分。

```mysql
CREATE TABLE sys_user (
  id int(11) NOT NULL AUTO_INCREMENT,
  username varchar(20) DEFAULT NULL COMMENT '登录名',
  `password` varchar(64) DEFAULT NULL COMMENT '密码',
  name varchar(20) DEFAULT NULL COMMENT '用户姓名',
  tel varchar(16) DEFAULT NULL COMMENT '电话',
  roles varchar(128) DEFAULT NULL COMMENT '拥有的角色',
  resources varchar(256) DEFAULT NULL COMMENT '拥有的权限',
  PRIMARY KEY (id)
);
```

注意该表中 password 是关键字，需要用特殊符号标识（当然各位也可以使用 pwd 来避开这种问题）。

#### 2.5.2 初始化用户数据

下面我们来初始化几条用户数据。为了能够简单的将数据保存到数据库，以及后续与数据库进行交互，我们可以引入 SpringDataJPA 和 `spring-boot-starter-test` 快速操作。

```xml
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>mysql</groupId>
        <artifactId>mysql-connector-java</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
```

**基础代码**

接下来快速编写一下对应的 entity 和 Dao 接口吧。

```java
import javax.persistence.*;

@Entity
@Table(name = "sys_user")
@Data
public class User {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;
    
    private String username;
    
    private String password;
    
    private String name;
    
    private String tel;
    
    private String roles;
    
    private String resources;
 
}
```



```java
@Repository
public interface UserDao extends JpaRepository<User, Integer>, JpaSpecificationExecutor<User> {
    
    List<User> findAllByUsername(String username);
}
```

**配置数据库连接和JPA**

不要忘记引入 SpringDataJPA 后需要配置数据源哦。

```properties
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
spring.datasource.url=jdbc:mysql://localhost:3306/springboot_dao?characterEncoding=utf8
spring.datasource.username=root
spring.datasource.password=root

spring.jpa.database=mysql
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
```

**添加数据**

最后我们向数据库中添加几条数据吧。

```java
@SpringBootTest
class Demo7AppTest {
    @Autowired
    private PasswordEncoder passwordEncoder;

    @Autowired
    private UserDao userDao;

    @Test
    public void testAddUser() throws Exception {
        User user = new User();
        user.setUsername("xiaoshuai");
        user.setPassword(passwordEncoder.encode("123456"));
        user.setName("小帅");
        user.setTel("123456789");
        user.setRoles("admin");
        userDao.save(user);

        user = new User();
        user.setUsername("xiaoming");
        user.setPassword(passwordEncoder.encode("123456"));
        user.setName("小明");
        user.setTel("987654321");
        user.setRoles("user");
        userDao.save(user);

        user = new User();
        user.setUsername("boss");
        user.setPassword(passwordEncoder.encode("123456"));
        user.setName("老板");
        user.setTel("147852369");
        user.setRoles("admin,manager");
        userDao.save(user);
    }

}
```

执行 `testAddUser` 方法后保存的用户数据;

#### 2.5.3 自定义UserDetailsService

下面我们要解释一个重要的组件，就是这个 `UserDetailsService` ，它就是做用户管理的核心 API 。在上面的示例中，我们使用的是 SpringSecurity 自带的基于内存的用户管理器 `InMemoryUserDetailsManager` ，但是基于数据库的用户管理就需要我们来自行实现了。下面我们就来自定义一个 `UserDetailsService` 的实现。

```java
@Service
public class UserService implements UserDetailsService {

    @Autowired
    private UserDao userDao;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {

        List<User> users = userDao.findAllByUsername(username);
        if (users.isEmpty()) {
            throw new UsernameNotFoundException("用户名" + username + "不存在！");
        }
        User user = users.get(0);
        org.springframework.security.core.userdetails.User.UserBuilder userBuilder = org.springframework.security.core.userdetails.User
                .withUsername(user.getUsername()).password(user.getPassword());
        if (StringUtils.hasText(user.getRoles())) {
            userBuilder = userBuilder.roles(user.getRoles().split(","));
        }
        if (StringUtils.hasText(user.getResources())) {
            userBuilder = userBuilder.authorities(user.getResources().split(","));
        }
        return userBuilder.build();


    }


}
```

简单看一下这段代码的实现吧，由于 SpringSecurity 只认 `UserDetailsService` ，所以我们就需要定义一个 `UserService` ，让它来实现 `UserDetailsService` ，重写其 `loadUserByUsername` 方法，在这里面我们需要从数据库中查出指定 username 对应的用户信息，如果用户名不存在，则会抛出 `UsernameNotFoundException` 的异常；当用户存在时，我们使用 `User` 的静态方法 `withUsername` 来初始化一个 `UserBuilder` 建造器，并传入用户的 password 以及拥有的角色、资源，最终调用 `UserBuilder` 的 `build` 方法，生成一个 `User` 对象，这个 `User` 对象就是 SpringSecurity 可以正确识别的用户信息。

> 源码看起来似乎有点不和谐，那是因为由于两个 `User` 类名撞车了，所以我们只好对 SpringSecurity 提供的这个 `User` 类写成全限定名。

`UserDetailsService` 的实现类注册到 IOC 容器后，下面就可以在 `WebSecurityConfiguration` 中来注入 `UserDetailsService` 并引用了。

```java
    @Autowired
    private UserDetailsService userDetailsService;
    
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        // ......
        // 引用IOC容器中注册的UserDetailsService实现（即基于数据库管理的实现）
        http.userDetailsService(userDetailsService);
    }
```

到此为止，自定义的工作就完成了。

下面我们来验证一下制作的效果如何。以 Debug 的方式重启工程，并在 `UserService` 的 `loadUserByUsername` 方法中打上断点。。

我们先测试一个用户名不存在的情况，传入的 `username` 为 `xiaoshuai123` ，借助接口工具重新发起登录请求，可以发现程序成功地停在了断点处，可以发现数据库中没有查到 `xiaoshuai123` 的用户，于是就可以成功抛出异常。

下面测试一个正确的用户，我们使用 xiaoshuai 和 123456 来发起登录，这次进入 `loadUserByUsername` 就可以正确查到数据，并封装 User 对象。走到 `UserBuilder` 的 `build` 方法中，我们可以看到此时用户的核心数据都已经被封装进来了。

随后我们可以再在 `WebSecurityConfiguration` 的 `successHandler` 方法传入的 lambda 表达式中打入断点，观察此时 `authentication` 的数据。可以发现其中封装的 `principal` 就是前面 `UserBuilder` 构造的 `User` 对象，两者唯一的区别是登录成功后的 password 被清空了（防止泄露）。

![](./img/202302/25sb-security4.png)

由此可见，我们已经完成了基于数据库的用户管理。

【通过本章的内容，想必各位已经能够实现 SpringBoot 与 SpringSecurity 的整合。下一章我们还会继续介绍更多 SpringSecurity 中的常用特性】

