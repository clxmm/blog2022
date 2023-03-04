---
title: 26整合security-SpringSecurity的常用机制.md
toc: true
tags: springboot
categories: 
    - [java]
    - [springboot]
---

## 1. remember-me记住我

---

我们先来了解一个经常接触的机制：记住我。这个机制还是蛮实用的，我们不想打开浏览器访问一些网站的时候每次都要登录，“记住我”这个功能就可以帮我们记住登录状态，在下一次打开浏览器访问网站的时候，能够自动将用户信息加载进来，从而实现不需要重复登录的效果。

如此好用的功能，SpringSecurity 自然也帮我们考虑到了，只是默认情况下它不会开启这个机制，我们可以先验证一下。

<!--more-->

### 1.1 默认不会开启记住我

验证“记住我”的方式很简单，我们启动工程，之后打开浏览器访问 [http://locahost:8080/](http://locahost:8080/) ，随意选择一个用户来登录。登录成功后我们可以访问 /demo 接口来检验是否有效（显然是可以访问的）。之后我们关闭浏览器再重新打开，直接访问 /demo 请求，可以发现浏览器被引导回了登录页。

这就证明了默认情况下 SpringSecurity 会在浏览器关闭时销毁登录状态，也就是所谓的不会记住用户已登录。

### 1.2 开启记住我

那要怎么开启“记住我”的这个特性呢？很简单，我们回到 `WebSecurityConfiguration` 中，在 `configure` 方法中写一行代码即可。

```java
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        // ......
        // 开启记住我
        http.rememberMe();
    }
```

如此就已经开启了“记住我”功能。下面我们再测试一下。

注意，当开启“记住我”后，SpringSecurity 默认带的表单中会多一个复选框：`remember me on this computer` ，勾选它就代表要“记住我”了。勾选后成功登录，这次再关闭浏览器重新打开，访问 `/demo` 请求时，就不需要再登录了，直接就能响应 `"demo"` 字符串。

### 1.3 持久化存储记住我的信息

下面继续测试一个问题：现在浏览器的确是记住我们登录的用户信息了，此时如果我们重启工程，再在浏览器中访问 `/demo` 请求，可以发现浏览器又被引导回登录页了！这又是为什么呢？

很简单，在一开始我们学习 JavaWeb 的时候，接触 Tomcat 的阶段就应该知道，Tomcat 中的 session 是在 Tomcat 重启后就销毁的，在这里 SpringSecurity 的“记住我”存储的信息也是一样，默认情况下与“记住我”相关的信息都是存放到内存中，所以当工程重启后，数据就都丢失了。

怎么解决呢？很简单，只需要将这些“记住我”的数据，存放到一个可以持久化存储的介质即可，SpringSecurity 的内部给我们提供了基于 jdbc 的实现，它通过依赖 `JdbcTemplate` 或者 `DataSource` ，就可以实现将“记住我”的数据，保存到数据库中。那我们就基于数据库的持久化存储来改造一下。

既然是存放到数据库中，那必然需要有一张对应的表，这个表的创建语句如下，各位也可以从 `JdbcTokenRepositoryImpl` 这个类的常亮 `CREATE_TABLE_SQL` 中获取到。

```sql
create table persistent_logins (
    username varchar(64) not null, 
    series varchar(64) primary key, 
    token varchar(64) not null, 
    last_used timestamp not null
);
```

下面就是改变“记住我”的存储介质了，回到 `WebSecurityConfiguration` 中，我们继续修改 `configure` 方法中的代码，此处需要我们初始化一个 `JdbcTokenRepositoryImpl` 对象，并手动设置 `JdbcTemplate` （或 `DataSource` ），随后调用 `http.rememberMe().tokenRepository(remembermeRepository);` 就可以变更存储方式为 jdbc 了。

```java
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        // ......
        // 持久化存储记住我的信息
        JdbcTokenRepositoryImpl remembermeRepository = new JdbcTokenRepositoryImpl();
        remembermeRepository.setJdbcTemplate(jdbcTemplate);
        http.rememberMe().tokenRepository(remembermeRepository);
    }
```

到底能不能行呢？我们来测试一下。重启工程后，使用 `boss` 登录，这必然是能够正常登录成功的。此时我们关闭浏览器，并重启工程，再次访问 `/demo` 请求时，可以看到浏览器还是能正确收到 `"demo"` 字符串，证明这次的”记住我“是真的生效了。

观察数据库中的 `persistent_logins` 表，可以发现此时数据已经正确持久化进来了。

![](./img/202302/25sb-security5.png)

到此为止，我们就完整的实现了“记住我”功能。

## 2. SpringSecurity的授权机制

---

下面是本章的重点，我们要研究一下 SpringSecurity 的授权和权限控制。对于 SpringSecurity 而言，它授权的机制包含两个方面：

- 基于资源的访问，即基于 url 拦截的授权
- 基于方法的访问，即基于代码内方法调用的授权

在展开讲解之前，我们先解释一个问题：经典的 RBAC 模型中需要 5 张数据库表：

- 用户表
- 角色表
- 用户 - 角色关系表
- 权限表
- 角色 - 权限关系表

这 5 张表会构成两个多对多关系，只不过在本小册中为了简化模型，注重 SpringSecurity 的机制本身，所以我们只用 `sys_user` 一张表来体现用户的所有信息，通过 `roles` 和 `resources` 字段存储用户拥有的角色和权限。

下面我们分别就这两种机制展开讲解。

### 2.1 基于url拦截的授权

我们首先来看粒度相对粗的 url 拦截授权。url 拦截的授权还包含两种情况，一种是我们把拦截的规则编写在 `WebSecurityConfiguration` 中，一种是当请求到达服务器时，动态判断是否允许访问，这两种方式小册都会演示。

#### 2.1.1 规则预先定义

还记得我们设置请求访问规则的那个配置吗：

```java
protected void configure(HttpSecurity http) throws Exception {
  // 限制/api开头的接口需要拥有admin角色，其余接口需要认证后访问
  http.authorizeRequests().antMatchers("/api/**").hasRole("admin").anyRequest().authenticated();
}
```

这样编写之后，其实我们就已经预先指定了规则，不过这个规则是限制的角色。如何根据 url 来编写限制规则呢？其实在我们调用 `antMatchers` 方法后，除了可以用 `hasRole` 方法限制角色外，还可以用 `hasAuthority` 方法来限制访问权限的标识（注意这个标识只是普通的字符串，并不一定是 url ）。

##### 2.1.1.1 设置访问规则

我们可以来指定一下规则：限制 `/api/test` 接口需要拥有 `/api/test` 权限（当然也可以是其他任何一个字符串，无所谓），其余的接口仍然是认证后允许访问。

```java
protected void configure(HttpSecurity http) throws Exception {
  // ......
  // 限制/api/test接口需要拥有/api/test权限
  http.authorizeRequests().antMatchers("/api/test").hasAuthority("/api/test").anyRequest().authenticated();
}

```

如此配置后，我们可以重启工程检验一下效果。

##### 2.1.1.2 测试权限拦截

我们仍然使用 `xiaoshuai` 来登录，登录成功后 `/demo` 接口仍然可以正常访问，但是当我们访问 `/api/test` 时，可以发现访问失败，并提示 403 错误，证明我们的配置已经生效。

如何让这个接口能正常访问呢？还记得我们在上一章中，创建 `sys_user` 表中的那个 `resources` 字段吗？它就是用来存放拥有的权限的。我们可以发送一条 update 语句，给 `xiaoshuai` 赋权：

```sql
update sys_user set resources = '/api/test' where username = 'xiaoshuai';

```

赋权之后，我们访问 `/logout` 注销，之后重新使用 `xiaoshuai` 登录，登录成功后访问 `/api/test` ，可以发现这次就可以成功访问了！

##### 2.1.1.3 成功访问的原因

可能会有小伙伴好奇，为什么我们只是执行了一条 update 语句后，这个接口就能访问了呢？

还记得上一章中，我们制作基于数据库的用户管理的功能中，实现的 `UserService` 代码中，有一段构造 `UserDetails` 的代码吗：

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
        org.springframework.security.core.userdetails.User.UserBuilder userBuilder = 
          org.springframework.security.core.userdetails.User
                .withUsername(user.getUsername()).password(user.getPassword());
        if (StringUtils.hasText(user.getRoles())) {
            userBuilder = userBuilder.roles(user.getRoles().split(","));
        }
        // 注意看这里
        if (StringUtils.hasText(user.getResources())) {
            userBuilder = userBuilder.authorities(user.getResources().split(","));
        }
        return userBuilder.build();
    }
}
```

注意看代码的下半部分，我们将 `resources` 属性取出来，并用英文逗号分隔，之后封装到 `UserBuilder` 中，这样最终创建的 `UserDetails` 对象中就包含了我们在数据库中设置的权限标识，由此也就可以成功访问之前受限制的接口资源了。

##### 2.1.1.4 预先定义的不足

下面请各位思考一个问题：如果只是用如上的方式来定义 url 拦截的规则，有一个什么问题？

试想，如果拦截规则发生变化，我们需要怎么办呢？很明显需要修改代码，重启工程才能生效，这很显然不能应用于生产！

理想的情况应该是怎样呢？肯定是连接数据库，通过与数据库配合，动态制作鉴权规则，这样做既能灵活地修改拦截规则，又能即时生效。所以下面我们来讲解如何连接数据库，实现动态判定和拦截。

#### 2.1.2 连接数据库动态判断

基于 RBAC 模型中，用于动态判定一个 url 是否允许访问的规则是：

-  1）该 url 是否为一个受保护的资源（即该 url 在 RBAC 模型的 “权限表” 中有定义）
-  2）用户所拥有的所有角色中，是否有关联了该资源（即两次多对多查询后，是否能查到该 url ）

简单的说，如果一个访问的 url 在数据库中有定义，那么访问这个 url 时，就应该判定当前登录人是否有这个 url 的访问权限。

##### 2.1.2.1 初始化数据库表

既然需要先有一个 “全部资源” 的表，那下面我们就来创建对应的表 `sys_resource` ，并初始化两条数据：（所以整个演示过程其实是 2 张表）

```sql
CREATE TABLE sys_resource (
  id int(11) NOT NULL AUTO_INCREMENT,
  name varchar(255) DEFAULT NULL,
  permission varchar(255) DEFAULT NULL,
  PRIMARY KEY (id)
);

INSERT INTO sys_resource (name, permission) VALUES ('demo', '/demo');
INSERT INTO sys_resource (name, permission) VALUES ('api-test', '/api/test');
```

##### 2.1.2.2 添加对应的实体类和Dao

与数据库表对应的是 SpringDataJPA 中的实体类和 Dao 接口，我们只需要快速来编写即可。

```java
@Data
@Entity
@Table(name = "sys_resource")
public class Resource {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;

    private String name;

    private String permission;
}
```

```java
@Repository
public interface ResourceDao extends JpaRepository<Resource, Integer> {
}
```

##### 2.1.2.3 自定义过滤器

既然需要连接数据库做判定了，那么下面就需要我们自己来制作鉴权的过滤器，对这些 url 进行拦截。

SpringSecurity 中可以应用的过滤器，本质与 Servlet 中的过滤器没有区别，仅仅是实现 `javax.servlet.Filter` 接口即可。那我们就来创建一个 `ResourceAuthorizationFilter` ，实现 `Filter` 接口，并标注 `@Component` 注解，将这个过滤器注册到 IOC 容器中。

```java
@Component
public class ResourceAuthorizationFilter implements Filter {
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        // 在继续向后调用过滤器链之前，连接数据库判断是否当前登录人是否允许访问
        chain.doFilter(request, response);
    }
}
```

下面就是过滤器逻辑的编写。按照上面我们列举的两步判定规则分别实现。

**1）不在限制列表的请求要予以放行**

如果当前请求不在 sys_resource 表的限制范围内，则这个 url 默认是要放行的，所以我们需要先把数据库中所有的限制资源查出，并检查是否包含当前请求：

```java
@Override
public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
  throws IOException, ServletException {
  // 在继续向后调用过滤器链之前，连接数据库判断是否当前登录人是否允许访问
  HttpServletRequest req = (HttpServletRequest) request;
  // 首先判断当前请求是否在限制访问的列表中
  List<String> resources = resourceDao.findAll().stream().map(Resource::getPermission).collect(Collectors.toList());
  String uri = req.getRequestURI();
  if (!resources.contains(uri)) {
    chain.doFilter(request, response);
    return;
  }
  // ......
}
```

**2）在限制列表的请求要予以检查**

如果当前请求在限制资源的列表中，则需要检查当前登录人是否拥有该权限，判断的规则也很简单，首先我们取出当前登录人的信息，从 UserDetails 中可以拿到当前登录人拥有的所有权限，之后检查匹配即可。

```java
@Override
public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
  throws IOException, ServletException {
  // 在继续向后调用过滤器链之前，连接数据库判断是否当前登录人是否允许访问
  HttpServletRequest req = (HttpServletRequest) request;
  // 首先判断当前请求是否在限制访问的列表中
  List<String> resources = resourceDao.findAll().stream().map(Resource::getPermission).
    					collect(Collectors.toList());
  String uri = req.getRequestURI();
  if (!resources.contains(uri)) {
    chain.doFilter(request, response);
    return;
  }
  // 在限制访问的列表中，需要鉴权
  Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
  UserDetails userDetails = (UserDetails) authentication.getPrincipal();
  if (userDetails.getAuthorities().stream().anyMatch(i -> i.getAuthority().equals(uri))) {
    chain.doFilter(request, response);
    return;
  }
  throw new AccessDeniedException("权限不足");
}
```

如此就算编写完成了。

##### 2.1.2.4 配置到SpringSecurity过滤器链

只编写完毕还不够，我们需要将这个过滤器配置到 SpringSecurity 的过滤器链中，以使其生效。这个配置需要我们写到 WebSecurityConfiguration 的 configure 方法中：

```java
    @Autowired
    private ResourceAuthorizationFilter resourceAuthorizationFilter;
    
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        // ......
        // 默认的权限控制只检查是否已登录
        http.authorizeRequests().anyRequest().authenticated();
        // 注入自定义过滤器，连接数据库动态鉴权
        http.addFilterAfter(resourceAuthorizationFilter, FilterSecurityInterceptor.class);
    }
```

注意一点，这个 `http.addFilterAfter()` 方法是在某个特定的过滤器之后插入过滤器，而对应 `http.authorizeRequests()` 的过滤器类型为 `FilterSecurityInterceptor` ，各位不需要知道为什么是它，也不用记住名，只管拿来设置上即可。

##### 2.1.2.5 测试权限拦截效果

如此配置完毕后，我们就可以检验是否有效了。重启工程，我们依然拿 `xiaoshuai` 来登录测试。

登录成功后，我们首先访问不参与权限拦截的 `/getData` 接口，发送 `/getData?name=abc` 请求，可以发现浏览器成功收到返回的数据：

接着我们访问受控制的 `/api/test` 接口，由于上面我们刚给 xiaoshuai 设置了权限，所以这个接口也可以正常访问。

最后再来测试 `/demo` 请求，由于这个接口没有授权给 `xiaoshuai` ，所以此时会提示 403 权限不足：

三个接口的拦截都符合我们的设计，至此我们就完成了基于数据库的 url 拦截授权。

### 2.2 基于方法注解拦截的授权

基于 url 的拦截授权的粒度相对较粗，只要一个请求判定后准许访问，那么这个接口中无论调用多少方法，都被视为合法的。但是在一些安全性要求高的场景中，我们希望将权限的控制细化到方法上，也就是说，只有拥有指定权限的用户才能调用目标方法。

SpringSecurity 对方法的授权，其实现方式是在需要授权的方法上标注一些特定的注解，并声明需要拥有的权限，当方法被调用时，SpringSecurity 会自动匹配权限是否匹配，并执行相应的放行 / 拦截动作。下面我们也来演示一下。

#### 2.2.1 开启方法注解的拦截授权

默认情况下 SpringSecurity 不会给我们开启方法注解的拦截授权，我们需要自己开启，而开启的方式是在 `WebSecurityConfiguration` 上标注一个新的注解：`@EnableGlobalMethodSecurity` 。

```java
@Configuration
@EnableWebSecurity
@EnableGlobalMethodSecurity(securedEnabled = true, prePostEnabled = true, 
                            jsr250Enabled = true, proxyTargetClass = true)
public class WebSecurityConfiguration extends WebSecurityConfigurerAdapter {
```

各位可以发现我在这个注解上标注了好几个属性是吧，它们都是干嘛的呢？一一解释：

- `securedEnabled` ：是否开启 `@Secured` 注解
- `prePostEnabled` ：是否开启 `@PreAuthorize` 和 `@PostAuthorize` 注解
- `jsr250Enabled` ：是否开启 JSR-250 规范中的 `@DenyAll` 、`@PermitAll` 、`@RolesAllowed` 注解
- `proxyTargetClass` ：是否全部使用 Cglib 动态代理，默认 false ，设置为 true 后会全面使用 Cglib

配置中我们可以将所有支持的注解都打开，不过本小册只介绍 SpringSecurity 中最常用、最符合拦截授权思想的注解 `@PreAuthorize` 。

#### 2.2.2 @PreAuthorize的使用

`@PreAuthorize` 跟 `@PostAuthorize` 注解其实是一对组合的注解，分别代表方法之前拦截授权，以及方法之后拦截，敏锐的小伙伴应该马上能意识到，这就是 AOP 思想的体现吧！那必然是的，前面我们都看到 `proxyTargetClass` 属性了，应该得反应过来啊。

##### 2.2.2.1 代码准备

为了能够演示能进接口，但不能进方法的效果，我们在 UserService 中添加一个根据 id 查用户信息的方法：

```java
    public User get(Integer id) {
        System.out.println("进入UserService的get方法！");
        return userDao.getById(id);
    }
```

之后在 `ApiController` 中注入 `UserService` ，并编写一个 `getUser` 方法调用 `UserService` 的 `get` 方法：

```java
@RestController
@RequestMapping("/api")
public class ApiController {
    
    @Autowired
    private UserService userService;
    
    @GetMapping("/getUser")
    public String getUser(Integer id) {
        System.out.println("进入getUser方法！");
        return userService.get(id).getName();
    }
    
    // ......
}
```

##### 2.2.2.2 加入@PreAuthorize注解

下面我们加入方法注解 `@PreAuthorize` 。在 `UserService` 的 `get` 方法上，我们标注 `@PreAuthorize` ，这里面有个 value 需要传入。可关键是，它能写啥呢？我们可以打一个 "has" ，此时 IDE 的反应让我们很惊喜，竟然有提示！而且提示的非常及时，这种写法跟 `WebSecurityConfiguration` 的 `configure` 方法中声明的简直一模一样！

那我们就标注一个 `@PreAuthorize("hasAuthority('getUser')")` 吧，代表当前登录人需要有 `"getUser"` 权限才能继续调用。

##### 2.2.2.3 测试效果

下面我们重启工程，继续使用 `xiaoshuai` 来登录。登录成功后，我们访问 [http://localhost:8080/api/getUser?id=1](http://localhost:8080/api/getUser?id=1) 请求，可以发现此时浏览器给我们响应了 403 错误，而从控制台上可以发现，`ApiController` 中的控制台输出的确执行了，说明进入到了接口中，只是 `UserService` 的方法没有调用成功，被 SpringSecurity 拦截了！

```
进入getUser方法！
Hibernate: select resource0_.id as id1_0_, resource0_.name as name2_0_, resource0_.permission as permissi3_0_ from sys_resource resource0_

```

下面我们再执行一条 SQL ，给小明授予这个权限：

```sql
update sys_user set resources = 'getUser' where username = 'xiaoming';

```

之后注销，使用 `xiaoming` 登录，再次访问 `/api/getUser` 方法时，就可以成功访问并拿到 1 号用户的姓名了！

到此为止，我们就完成了基于方法注解的拦截授权。

## 3. 整合jwt实现无状态session

---

下面我们来介绍本章的最后一个重难点：**整合 jwt** 。在开始整合之前，我们要先思考一下使用 jwt 令牌的缘由，也分析一下利弊。

### 3.1 为什么要整合jwt

基于前后端分离的项目中，我们更多地是使用 jwt 作为用户信息的存储载体，将 jwt 令牌放到浏览器上，前端每次向后端发送请求时，只需要带上 jwt 信息，后端也配合解析 jwt 令牌，就可以实现无状态的 session 。这种情况下，因为用户信息存储在 jwt 中，所以即便后端服务重启，用户的登录状态也不会丢失。

另一种常见的情景下，一个分布式系统中，统一身份认证和资源服务可能是分离的，这种情况下又会出现一个新的问题：不可能每个服务都设置一套 session 吧，即便不同服务之间做 session 复制也是不太现实的。所以整合无状态 session 还有一个意图，将用户信息转储到 jwt 令牌后，浏览器只需要向统一身份认证发送登录请求，获取到 jwt 令牌，之后再携带 jwt 令牌访问资源服务，由资源服务来鉴权判定是否允许访问即可。

相应的简单流程图如下：

![](./img/202302/26sb-security.png)

### 3.2 jwt令牌

那既然 jwt 可以帮我们解决一些问题，下面我们就来了解一下跟 jwt 令牌相关的一些概念和设计。

#### 3.2.1 jwt令牌概述

jwt ，全程是 JSON Web Token ，严格意义上讲 jwt 应该是一个行业标准（ RFC 7519 ），这点可以从官方网站 [jwt.io/](https://jwt.io/) 中得到考证：

> JSON Web Tokens are an open, industry standard [RFC 7519](https://tools.ietf.org/html/rfc7519) method for representing claims securely between two parties.
>
> JSON Web Tokens 是一种开放的行业标准 RFC 7519 ，用于在两方之间安全地表示声明。

相应的行业标准网页也贴到这里 [www.rfc-editor.org/rfc/rfc7519](https://www.rfc-editor.org/rfc/rfc7519) ，感兴趣的小伙伴也可以去了解一下。

话说回来，jwt 是一个行业标准，它定义的是一种简洁的、自包含的协议格式，使用这种格式可以安全地、轻便地在需要通信的双方传递 json 格式的用户数据。而且 jwt 考虑到用户数据可能会被劫持和篡改的的风险，它可以使用 RSA 非对称加密算法，对生成的 jwt 令牌进行加密，以达到防篡改的效果。

简单看下来，jwt 令牌的优点还是很多的：解析方便、存储内容的易扩展、安全性高，等等。当然这世界上没有十全十美的东西，jwt 的一个很要命的问题是生成的 jwt 令牌真的很大很长，占用的存储空间和请求传递时会消耗一定的流量，至于有多大，下面我们整合的时候各位就可以看到了。

#### 3.2.2 jwt令牌的结构

一个 jwt 令牌包含以下的 3 部分组成：

- **header** 头部：包含令牌的类型（通常就是 `JWT` ）以及使用的哈希算法（ RSA 、HMAC 、SHA256 等）
- **payload** 载荷：包含用户的信息，这部分是我们自定义的，可以放的多也可以少
  - 通常这部分我们就用来存储系统中的用户信息，包括用户名、所属部门组织、角色等，但是不要放密码（载荷部分可以被解码得到明文）！
- **signature** 签名：签名的部分会将前两部分使用 Base64 编码后连接起来，并进行一次签名（加密）操作，生成的签名结果就是第三部分

最终的这 3 部分内容会使用 Base64 编码，分别生成一串内容，最后用英文句号（.）连接，这样就形成了一个 jwt 令牌。

#### 3.2.3 RSA非对称加密

既然生成 jwt 令牌中包含不可或缺的一步：签名，而签名的方式大多是加密，下面我们介绍一个非常经典的非对称加密算法：RSA 。

先解释下什么叫非对称加密，顾名思义，非对称加密的加密跟解密不是使用同一个办法的，也就是说，使用加密的逆操作并不能解密。非对称加密会使用两把密钥，分别是公钥和私钥，其中私钥需要服务器自行保存，且设置安全防护防止泄露；公钥则是分发给各个受信任的客户端，由他们持有即可。

一个简单的加密和解密的过程是：

- 持有公钥方的客户端将数据加密后，只有持有私钥的服务器才可以解密获得数据
- 持有私钥方的服务器将数据加密后，持有公钥的所有客户端都可以解密获得数据（私钥也可以解密私钥加密的数据）

相对比起对称加密的算法来看，非对称加密的安全性高，但是效率相对低。

RSA 算法就是非对称加密的经典体现，它是历史上三位数学家 Rivest 、Shamir 和 Adleman 设计了一种算法，RSA 就是取他们三个人的名字首字母命名的。

### 3.3 SpringSecurity整合jwt

简单了解相关的背景知识后，下面我们就来实际的整合一下 jwt 。在使用 jwt 之前，需要在当前 `spring-boot-integration-07-security-webmvc` 工程中再导入几个跟 jwt 相关的依赖：

```xml
    <!-- jwt -->
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-api</artifactId>
        <version>0.11.5</version>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-impl</artifactId>
        <version>0.11.5</version>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-jackson</artifactId>
        <version>0.11.5</version>
    </dependency>
```

> 不要看到版本号是 0 开头的就发憷，这已经是 jjwt 现存最新的版本了，阿熊现在用于生产环境的版本还是 0.11.2 ，是 2020 年的版本。。。

#### 3.3.1 封装载荷对象

jwt 中可以任意封装信息的部分就是载荷，这里面我们可以将用户的信息、jwt 过期时间等信息封装进来：

```java
@Data
public class TokenPayload {

    private String id;

    private User user;

    private Date expiration;

}

```

注意一点，如果各位在封装的时候不确定 `TokenPayload` 中会放什么，那完全可以制作成泛型。

#### 3.3.2 导入jwt的工具类

下面是几个 jwt 相关的工具类，各位没有必要自己写，也没必要深究具体实现，工具类我们就只管拿来用。

`JsonUtils` ，用于快速简单的处理 json 相关的工作，当然各位也可以导入 fastjson 、Hutool 等工具包处理。

```java
public abstract class JsonUtils {

    public static final ObjectMapper OBJECT_MAPPER = new ObjectMapper();

    /**
     * json转Map
     *
     * @param json
     * @return
     */
    public static Map<String, Object> parseObject(String json) {
        if (StringUtils.hasText(json)) {
            return parseObject(json, Map.class);
        }
        return new HashMap<>();
    }

    /**
     * json转指定类型
     *
     * @param json
     * @param type
     * @param <T>
     * @return
     */
    public static <T> T parseObject(String json, Class<T> type) {
        try {
            return OBJECT_MAPPER.readValue(json, type);
        } catch (IOException e) {
            throw new JsonParseException(e);
        }
    }

    public static String toJson(Object obj) {
        if (obj == null) {
            return "";
        }
        try {
            return OBJECT_MAPPER.writeValueAsString(obj);
        } catch (JsonProcessingException e) {
            throw new JsonParseException(e);
        }
    }
}
```

`JwtUtils` 。用于生成 jwt 的访问令牌和刷新令牌（这两个概念下面会展开讲）。特别说明一点，这个工具类就是阿熊**本人正在生产项目上使用的**，各位也可以拿来稍作修改，之后放到自己项目中。

> 如果各位要直接拿来用于项目中，有几个细节需要调整：
>
> 1）小册中为了先演示访问令牌，后演示刷新令牌，所以下面的代码中将刷新令牌的生成逻辑暂时注释掉了，如果小伙伴想拿来用，记得调整注释；
>
> 2）小册为了引导各位动手 + 思考 + 扩展新的知识点，所以下面的代码中埋了一个 “小坑” ，不过不用担心，如果各位把本节内容读完后，这个 “小坑” 自然也就解除了；
>
> 3）阿熊自己的项目中具体的代码跟下面还不完全一样，因为真实的项目开发中，我们很少直接用 SpringSecurity 默认给我们提供的 `User` 模型，为了扩展自己项目中的一些用户信息，通常都会自行封装模型，实现 `UserDetails` 接口，这样下面在解析 jwt 令牌的 `parseToken` 方法时，解析 json 封装的模型就是我们自己写的模型了，小册只是为了方便演示，所以没有这么做而已。

```java
public abstract class JwtUtils {
    
    public static final String JWT_PAYLOAD = "user";
    public static final String JWT_HEADER = "authorization";
    /** 分割jwt与刷新令牌的标识 */
    public static final String JWT_SLIPTER = "abcdefg.uvwxyz";
    
    public static final char ENCODE_CHAR = 'a';
    
    /**
     * 向客户端响应jwt访问(+刷新)令牌
     * @param response
     * @param user
     * @param privateKey
     * @param accessExpire
     * @param refreshExpire
     */
    public static void writeJwtToken(HttpServletResponse response, User user, PrivateKey privateKey,
            int accessExpire, int refreshExpire) {
        String jwtToken = JwtUtils.generateJwtToken(user, privateKey, accessExpire);
        // String refreshToken = JwtUtils.generateRefreshToken(user, privateKey, refreshExpire);
        response.addHeader(JWT_HEADER, jwtToken);
        // response.addHeader(JWT_HEADER, jwtToken + JWT_SLIPTER + refreshToken);
    }
    
    /**
     * 生成jwt令牌
     * @param user
     * @param privateKey
     * @param expire 秒
     * @return
     */
    public static String generateJwtToken(User user, PrivateKey privateKey, int expire) {
        return Jwts.builder().claim(JWT_PAYLOAD, JsonUtils.toJson(user)).setId(createJtl())
                .setExpiration(new Date(System.currentTimeMillis() + expire * 1000))
                .signWith(privateKey, SignatureAlgorithm.RS256).compact();
    }
    
    /**
     * 生成刷新令牌
     * @param user
     * @param expire 秒
     * @return
     */
    public static String generateRefreshToken(User user, PrivateKey privateKey, int expire) {
        String tokenJson = JsonUtils.toJson(user);
        String encodedTokenJson = encodeRefreshToken(tokenJson);
        return Jwts.builder().claim(JWT_PAYLOAD, encodedTokenJson).setId(createJtl())
                .setExpiration(new Date(System.currentTimeMillis() + expire * 1000))
                .signWith(privateKey, SignatureAlgorithm.RS256).compact();
    }
    
    /**
     * 解析jwt令牌
     * @param token
     * @param publicKey
     * @return
     */
    public static TokenPayload parseToken(String token, PublicKey publicKey) {
        Jws<Claims> claimsJws = Jwts.parserBuilder().setSigningKey(publicKey).build().parseClaimsJws(token);
        Claims claims = claimsJws.getBody();
        TokenPayload payload = new TokenPayload();
        payload.setId(claims.getId());
        payload.setExpiration(claims.getExpiration());
        payload.setUser(JsonUtils.parseObject(claims.get(JWT_PAYLOAD).toString(), User.class));
        return payload;
    }
    
    /**
     * 解析jwt令牌中的载荷部分
     * @param token
     * @param publicKey
     * @return
     */
    public static String getTokenPayload(String token, PublicKey publicKey) {
        Jws<Claims> claimsJws = Jwts.parserBuilder().setSigningKey(publicKey).build().parseClaimsJws(token);
        Claims claims = claimsJws.getBody();
        return claims.get(JWT_PAYLOAD).toString();
    }
    
    public static String createJtl() {
        return new String(Base64.getEncoder().encode(UUID.randomUUID().toString().getBytes()));
    }
    
    /**
     * 编码token中的信息
     * @param tokenJson
     * @return
     */
    public static String encodeRefreshToken(String tokenJson) {
        StringBuilder sb = new StringBuilder(tokenJson.length() + 10);
        sb.append("refresh ");
        char[] chars = tokenJson.toCharArray();
        for (char c : chars) {
            c ^= ENCODE_CHAR;
            sb.append(c);
        }
        String encodedTokenJson = sb.toString();
        return new String(Base64Utils.encode(encodedTokenJson.getBytes()));
    }
    
    /**
     * 解码token中的信息
     * @param encodedToken
     * @return
     */
    public static String decodeRefreshToken(String encodedToken) {
        if (!StringUtils.hasText(encodedToken)) {
            return "";
        }
        
        String encodedTokenJson = new String(Base64Utils.decode(encodedToken.getBytes()));
        if (encodedTokenJson.length() <= 8) {
            return "";
        }
        encodedTokenJson = encodedTokenJson.substring(8);
        StringBuilder sb = new StringBuilder(encodedTokenJson.length());
        char[] chars = encodedTokenJson.toCharArray();
        for (char c : chars) {
            c ^= ENCODE_CHAR;
            sb.append(c);
        }
        return sb.toString();
    }
}
```

#### 3.3.3 认证成功后的处理

准备工作完成后，下面我们就来考虑将 jwt 融合到认证流程中。登录方式的话我们还是选择使用表单方式即可（用 Postman 等接口工具也能调得通），关键的动作是通过用户名和密码校验后，我们如何处理认证成功的动作。传统基于 session 的认证中，当认证成功后会把用户信息放入 session ，但是基于 jwt 的认证中，认证成功后应该是响应 jwt 令牌吧！下面我们就来试一下。

修改认证成功的响应，我们需要回到 `WebSecurityConfiguration` 的 `configure` 方法中，在 `formLogin()` 系列配置中重新修改 `successHandler` ：

```java
    private PrivateKey privateKey;
    
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        // ......
    
        // 开启表单登录功能，设置认证成功后响应jwt信息
        this.privateKey = RsaUtils.getPrivateKey("jwt_rsa");
        http.formLogin().successHandler((request, response, authentication) -> {
            User user = (User) authentication.getPrincipal();
            // 从authentication中取到User，生成jwt并写入response
            JwtUtils.writeJwtToken(response, user, privateKey, 600, 3600);
            response.setContentType("application/json;charset=utf-8");
            response.getWriter().write("登录成功");
        }); // 省略failureHandler ......
    }
```

简单解释一下修改后的代码，首先我们在 `WebSecurityConfiguration` 中添加了一个内部成员 `PrivateKey` ，并在 `configure` 方法中借助 `RsaUtils.getPrivateKey` 方法加载到私钥；下面设置 `successHandler` 的时候，认证成功后将 User 的信息取出，并使用 `JwtUtils.writeJwtToken` 方法生成 jwt 令牌，并写入响应头中。

注意 `JwtUtils.writeJwtToken` 方法中最后两个参数的作用，第一个 600 是指下发的 jwt 令牌有效时间为 600 秒（即 10 分钟），第二个 3600 是指配套的 jwt 刷新令牌有效时间为 1 小时（刷新令牌在 3.4 章节讲解）。

#### 3.3.4 解析jwt的过滤器

只下发了 jwt 令牌还不够，我们还需要编写配套的过滤器，来解析 jwt 令牌，从而解析出对应的用户信息才行。

那我们就来写过滤器吧，套路还是跟上面一样，实现 `javax.servlet.Filter` 即可，不过注意一个小细节，由于解析 jwt 令牌需要使用公钥，所以在过滤器的构造方法中，我们需要把公钥加载进来，缓存到过滤器中：

```java
@Component
public class JwtTokenFilter extends SecurityContextPersistenceFilter {
    
    private PublicKey publicKey;
    
    public JwtTokenFilter() {
        this.publicKey = RsaUtils.getPublicKey("jwt_rsa.pub");
    }
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        // ？？？
    }
}
```

下面是解析的逻辑，如何取出并解析 jwt 令牌呢？其实思路很简单，首先尝试从请求头上探测下是否包含 jwt 令牌（也就是 request.getHeader(xxx) 看看有没有值，xxx 具体是什么我们自己说了算），如果可以获取到令牌，则尝试使用我们提供的 JwtUtils 解析一下，看看是否能成功解析，解析成功的话就可以封装身份信息了，否则认定当前登录信息过期，提示客户端该令牌失效。

对应的代码实现如下，各位可以先对着读一遍逻辑，实在搞不明白也没事，直接照抄就行。

```java
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        HttpServletRequest req = (HttpServletRequest) request;
        try {
            // 解析jwt令牌
            String jwt = req.getHeader(JwtUtils.JWT_HEADER);
            if (StringUtils.hasText(jwt)) {
                TokenPayload payload = JwtUtils.parseToken(jwt, publicKey);
                User user = payload.getUser();
                // 重要：设置用户信息
                SecurityContextHolder.setContext(new SecurityContextImpl(
                        new UsernamePasswordAuthenticationToken(user, null, user.getAuthorities())));
            } else {
                // 没有获取到jwt信息，则存入一个空用户信息
                SecurityContextHolder.setContext(SecurityContextHolder.createEmptyContext());
            }
            chain.doFilter(request, response);
        } catch (ExpiredJwtException e) {
            response.setContentType("application/json;charset=utf-8");
            response.getWriter().write("jwt令牌已过期！");
            response.getWriter().flush();
            response.getWriter().close();
        } catch (JwtException e) {
            response.setContentType("application/json;charset=utf-8");
            response.getWriter().write("解析jwt令牌失败！");
            response.getWriter().flush();
            response.getWriter().close();
        } finally {
            SecurityContextHolder.clearContext();
        }
    }
```

注意实现代码中的设置用户信息那一行，这个地方要把已经从 jwt 令牌中解析出来的 `User` 对象，存入到一个 `UsernamePasswordAuthenticationToken` 中，这个 token 对象是个什么东西？我们之前一直没有遇到过！不要着急，各位现在可以先简单理解为 “当前登录人信息的持有者” ，下面两章讲解 SpringSecurity 的执行机制和原理时，我们还会碰到它的。

过滤器编写完毕后，我们还需要回到 `WebSecurityConfiguration` 的 `configure` 方法中，再加两行配置，分别是添加过滤器，以及设置当前的 Session 管理模式为无状态 session 。

```java
    @Autowired
    private JwtTokenFilter jwtTokenFilter;
    
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        // ......
        // 设置jwt令牌的解析过滤器执行时机比AnonymousAuthenticationFilter靠前
        http.addFilterBefore(jwtTokenFilter, AnonymousAuthenticationFilter.class);
        // 设置无状态session
        http.sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS);
    }
```

如此编写完毕后，我们就可以测试了。

#### 3.3.5 测试效果

##### 3.3.5.1 测试登录

首先我们先测试登录，使用接口工具访问 /login 请求，使用正确的用户名和密码，可以发现在收到 “登录成功” 的信息之外，我们还从响应头信息中拿到了 jwt 令牌！如下图所示。

![](./img/202302/26sb-security1.png)

这一长串被 Base64 编码后的字符串就是 jwt 令牌了，我们要把它保存起来，下面访问资源时还要用。

##### 3.3.5.2 测试接口访问（带踩坑过程，不想看可直接跳转到3.3.5.4）

下面我们来访问一下 `xiaoshuai` 准许访问的接口 `/api/test` ，注意在访问的时候需要在请求头上附带 jwt 令牌：

发送请求，可以发现打印了一堆 html 代码，而且后端的控制台也报错了！

```ini
Caused by: com.fasterxml.jackson.databind.exc.InvalidDefinitionException: Cannot construct instance of `org.springframework.security.core.userdetails.User` (no Creators, like default constructor, exist): cannot deserialize from Object value (no delegate- or property-based Creator)
 at [Source: (String)"{"password":null,"username":"xiaoshuai","authorities":[{"authority":"/api/test"}],"accountNonExpired":true,"accountNonLocked":true,"credentialsNonExpired":true,"enabled":true}"; line: 1, column: 2]
```

仔细观察这个报错，它提示的是 `org.springframework.security.core.userdetails.User` 这个类无法被实例化，因为没有默认的无参构造器！为什么会出现这种情况呢？

##### 3.3.5.3 【知识点扩展】jackson的序列化机制与问题解决

这一小节就是前面在列举 `JwtUtils` 的时候，给各位提的醒了，这个地方是阿熊留的一个小坑，帮助各位补充一个小知识点（或者说是知识点回顾）。Jackson 这个 json 序列化工具，在进行字符串转模型类对象时有一个要求，那就是要求转换后的模型类需要有一个默认的无参构造方法，或者我们使用 `@JsonCreator` 和 `@JsonProperty` 注解来指定有参构造方法！而 SpringSecurity 给我们默认提供的 `org.springframework.security.core.userdetails.User` 这个类中并没有无参构造方法，所以就造成了上述的问题。

怎么解决呢？改源码那肯定是不可能的了，不过注意这个 `org.springframework.security.core.userdetails.User` 类没有被声明为 **final** ，那就说明我们可以继承它，从而做一些手脚。说干就干，我们来编写一个 `SecurityUser` 类，来继承原生内置的 `User` ：

```java
/**
 * 用于解决User无法使用jackson反序列的问题
 */
public class SecurityUser extends User {
  public SecurityUser(String username, String password, Collection<? extends GrantedAuthority> authorities) {
    // 密码不要泄露了，此处直接改成 ""
    super(username, "", authorities);
  }
}
```

注意，在重写构造器时，我们就需要标注 jackson 给我们提供的注解了：

```java
/**
 * 用于解决User无法使用jackson反序列的问题
 */
public class SecurityUser extends User {

  @JsonCreator
  public SecurityUser(@JsonProperty("username") String username, @JsonProperty("password") String password, 
                      @JsonProperty("authorities") Collection<? extends GrantedAuthority> authorities) {
    super(username, "", authorities);
  }
}
```

之后在 `JwtUtils` 中，将原本转换为 `User` 的 `parseToken` 方法，转换为我们扩展的子类 `SecurityUser` ：

```java
    public static TokenPayload parseToken(String token, PublicKey publicKey) {
        // ......
        // 注意改这一行！
        payload.setUser(JsonUtils.parseObject(claims.get(JWT_PAYLOAD).toString(), SecurityUser.class));
        return payload;
    }
```

这样是否就改好了呢？不妨直接重启试试！重启之后再次访问 /api/test ，发现还是打印一大堆 html 代码，再次观察控制台会发现这次内容变了：

```ini
Caused by: com.fasterxml.jackson.databind.exc.InvalidDefinitionException: Cannot construct instance of `org.springframework.security.core.GrantedAuthority` (no Creators, like default constructor, exist): abstract types either need to be mapped to concrete types, have custom deserializer, or contain additional type information

```

`GrantedAuthority` ？合着这个东西也要自己写吗？哎，其实不必（也写不了），我们观察它的实现类就知道了，`GrantedAuthority` 有一个最简单的实现类 `SimpleGrantedAuthority` ，我们只需要将本应该封装到这个对象的信息，先用字符串接收过来，再手动构造就可以了！

但是这里面还有一个需要注意的哦！注意观察错误日志中给我们打印的 json 字符串原文：

```json
{
  "password": null,
  "username": "xiaoshuai",
  "authorities": [
    {
      "authority": "/api/test"
    }
  ],
  "accountNonExpired": true,
  "accountNonLocked": true,
  "credentialsNonExpired": true,
  "enabled": true
}
```

可以发现 `authorities` 内部装的是一个一个的对象！所以还不能用字符串接哦，这样吧，我们用 `Map` 接，之后从 `Map` 中取出 `authority` 属性，再构造 `SimpleGrantedAuthority` ，就可以达到我们刚才说的效果了。

```java
@JsonCreator
public SecurityUser(@JsonProperty("username") String username, @JsonProperty("password") String password,
                    @JsonProperty("authorities") List<Map<String, String>> authorities) {
  super(username, "", authorities.stream().
        map(i -> new SimpleGrantedAuthority(i.get("authority"))).collect(Collectors.toList()));
}
```

如此改完后，再重启。

##### 3.3.5.4 修复后再测试接口访问

这次我们再带 jwt 令牌访问 `/api/test` 接口，可以发现浏览器成功响应出了 `"test"` 字符串！说明我们通过 jwt 令牌传递的用户信息，在后端已经成功解析了！

什么？你不信？那我们访问一个 `xiaoshuai` 没有的权限：`/demo` ，可以发现由于我们自己编写的 `ResourceAuthorizationFilter` 给过滤，所以依然可以成功限制访问，并响应 403 错误码。

```json
{
    "timestamp": "2023-03-04T11:18:19.613+00:00",
    "status": 403,
    "error": "Forbidden",
    "path": "/demo"
}
```

### 3.4 引入刷新令牌

到这里还不算完，各位再考虑一个问题：jwt 令牌是有过期时间的，那如果 jwt 令牌过期了，那我们应该怎么办？

很简单，再重新登录呗！过期了重新登录这不天经地义嘛！

可是再仔细一想，即便是重新登录，这真的合理吗？我们前面设置 jwt 的令牌生效时间只有 10 分钟，总不能让用户 10 分钟登录一次吧。

1. 拉长 jwt 令牌的过期时间，从 10 分钟拉长到 2 小时
   - 这种设置有个问题：jwt 令牌万一被泄露盗取，那么除非这个令牌过期，否则盗取方拿到 jwt 令牌后便可以在能力范围内攻击我们的系统，有可能造成损失的风险。
2. 当 jwt 只剩 2 - 3 分钟就过期时，给用户推送一个新 jwt 令牌
   - 这种设计也有个问题：如果用户恰好最后 2 - 3 分钟没有访问，那相当于没设计；如果设计的监控时间过长，则会频繁生成新 jwt 令牌。
3. 在响应 jwt 令牌时，再提供一个特殊的令牌，使用这个令牌就可以换取一个新的 jwt 令牌
   - 这种设计相较于前两者就好很多：客户端在收到 jwt 令牌时，同时也能收到一个特殊的标识，只要客户端发送请求到服务器发现 jwt 令牌过期时，就可以持有这个特殊标识，去服务器换取一个新的 jwt 令牌，这样只要这个特殊的标识不过期，我们就可以一直获取新的 jwt 信息，这样既保证了登录时长的拉长，同时又尽可能缩小了避免 jwt 令牌意外被盗取后产生风险的可能。

这样简单对比后，这个 “特殊令牌” 的机制就成了最优解，这也就是本节要讲的 **“刷新令牌”** 。通常我们在设计时，刷新令牌本质上是一个跟 jwt 访问令牌并无太大区别的令牌，它同样保存了用户的信息，只不过它的有效期更长。使用刷新令牌后，客户端需要同时保存 jwt 访问令牌和刷新令牌，使用访问令牌去请求资源，当访问令牌过期时，拿刷新令牌去换新的访问令牌。

下面我们也来实现一下刷新令牌的机制。

#### 3.4.1 制作刷新令牌的接口

刷新令牌的接口也很简单，直接照抄下面的代码即可，不必自己编写。简单解释代码的设计：

- 首先刷新令牌既需要公钥来解密信息，又需要私钥来加密新的令牌信息，所以构造方法中需要读取公钥和私钥；
- 刷新的接口中需要先从请求头上获取刷新令牌（注意刷新令牌的请求头名称跟访问令牌不同！），获取到后用 JwtUtils 工具类反解析出 User 的信息，之后重新构造 jwt 访问令牌和刷新令牌即可。

```java
RestController
public class RefreshTokenController {


    public static final String REFRESH_HEADER = "refreshtoken";

    private PrivateKey privateKey;
    private PublicKey publicKey;

    public RefreshTokenController() {
        this.privateKey = RsaUtils.getPrivateKey("jwt_rsa");
        this.publicKey = RsaUtils.getPublicKey("jwt_rsa.pub");
    }

    @PostMapping("/refreshToken")
    public void refreshToken(HttpServletRequest request, HttpServletResponse response) throws IOException {
        String refreshJwt = request.getHeader(REFRESH_HEADER);
        if (!StringUtils.hasText(refreshJwt)) {
            response.setContentType("application/json;charset=utf-8");
            response.getWriter().write("没有携带刷新令牌！");
            response.getWriter().flush();
            response.getWriter().close();
        }
        // 解析刷新jwt令牌
        try {
            String payloadEncodedJson = JwtUtils.getTokenPayload(refreshJwt, publicKey);
            String userTokenJson = JwtUtils.decodeRefreshToken(payloadEncodedJson);
            SecurityUser user = JsonUtils.parseObject(userTokenJson, SecurityUser.class);
            JwtUtils.writeJwtToken(response, user, privateKey, 600, 3600);
            response.setContentType("application/json;charset=utf-8");
            response.getWriter().write("刷新成功！");
            response.getWriter().flush();
            response.getWriter().close();
        } catch (ExpiredJwtException e) {
            response.setContentType("application/json;charset=utf-8");
            response.getWriter().write("刷新令牌已过期！");
            response.getWriter().flush();
            response.getWriter().close();
        } catch (JwtException e) {
            response.setContentType("application/json;charset=utf-8");
            response.getWriter().write("刷新令牌解析失败！");
            response.getWriter().flush();
            response.getWriter().close();
        }
    }


}
```

#### 3.4.2 测试效果

OK 代码改好后，下面就可以来测试了。为了测试方便，我们把生成访问令牌的有效时间改为 30 秒，方便它过期。

```java
// 开启表单登录功能，设置认证成功后响应jwt信息
this.privateKey = RsaUtils.getPrivateKey("jwt_rsa");
http.formLogin().successHandler((request, response, authentication) -> {
  User user = (User) authentication.getPrincipal();
  // 此处改为30秒
  JwtUtils.writeJwtToken(response, user, privateKey, 30, 3600);
  response.setContentType("application/json;charset=utf-8");
  response.getWriter().write("登录成功");
})......;
```

重启工程，再次使用 `xiaoshuai` 访问 `/login` 接口登录，可以发现这次响应头中的 token 一下子变得好长：

![](./img/202303/26sb-security.png)

注意观察红框中的内容，我们可以在中间部分找到之前制作的分隔符 `"abcdefg.uvwxyz"` ，在这之前的是访问令牌，这之后的是刷新令牌。

30 秒时间很短，等令牌过期后我们可以拿后半截 刷新令牌 ，以 POST 请求来访问 [http://localhost:8080/refreshToken](http://localhost:8080/refreshToken) ，并在请求头上附带刷新令牌。我们的预期效果是能够看到响应头中包含新的访问令牌和刷新令牌，殊不知当我们发起请求后，看到的却是我们熟悉的表单登录页 html ：

![](./img/202303/26sb-security1.png)

为什么会出现如此诡异的现象呢？还记得我们一开始设计的安全访问规则吗？如果一个请求不在数据库的限制访问资源列表中，则认定这个请求需要认证后访问。很明显我们现在的认证信息已经过期了，那就相当于没有登录，自然就把我们引导到登录页了！

怎么改呢？可能有小伙伴想到要改 `ResourceAuthorizationFilter` ，但其实要改的不是它，这个 `ResourceAuthorizationFilter` 只负责限制数据库中规定的限制访问资源，很明显 `/refreshToken` 请求没有被定义在数据库中，所以这个过滤器不会起效。真正起效果的其实是 `WebSecurityConfiguration` 中的这一行：

```java
    http.authorizeRequests().anyRequest().authenticated();

```

很明显，那我们要对一些不该限制的请求直接放行了，所以要做如下修改：

```java
    private static String[] ignoreUrls = {"/login", "/logout", "/refreshToken"};
    
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        // ......
        // 设置访问白名单
        http.authorizeRequests().antMatchers(ignoreUrls).permitAll().anyRequest().authenticated();
    }
```

如此修改完毕后重启工程，再次访问 `/refreshToken` 请求，这次就可以刷新成功了。

到此为止，有关 jwt 的整合也就完毕了。

【经过这两章的内容，各位是否已经深切体会到 SpringSecurity 的强大了呢？到现在为止我们还只是在应用层面体会，接下来的两章，我们要深入到源码了，看看底层中 SpringBoot 和 SpringSecurity 都为安全访问控制做了哪些充足的工作】