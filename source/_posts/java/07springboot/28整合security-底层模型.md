---
title: 28整合security-底层模型
toc: true
tags: springboot
categories: 
    - [java]
    - [springboot]
---

进入到 SpringSecurity 的最后一章，我们来剖析 SpringSecurity 核心的底层模型，希望通过本章的内容，小伙伴能够从根源上理解 SpringSecurity 用于安全访问控制的底层模型，从而能更深层次的理解 SpringSecurity 的工作机制，即便在后续实际项目开发中，也能结合 SpringSecurity 的底层模型，以及自己项目的需求特性，定制适合项目的安全控制规则。

## 1. 过滤器链结构

----

首先我们说，SpringSecurity 核心的底层模型，本质上就是一组**过滤器链**，得益于过滤器这种结构的特性，我们可以在客户端访问服务器时予以拦截，并进行一些附加处理，刚好安全访问控制就是适合应用在过滤器上的附加逻辑处理。

<!--more-->

本章的开始我们先通过 SpringSecurity 的过滤器链结构，以及链上的一些重要过滤器，先有一个大体的印象，后面到了源码分析阶段，看到这些过滤器实现时，也会有相应的心理准备。

#### 1.1 过滤器链内的主要结构

SpringSecurity 的整个过滤器链处理客户端请求的总体逻辑如下图所示（内部的过滤器并没有完全列举出来，只列举了核心的、各位能接触到的一部分）。

![](./img/202303/28sb-security.png)

> 有必要解释一件事：严格来讲，`SecurityContextPersistenceFilter` 并不是整个过滤器链的起点，只是排在他更靠前位置的过滤器不是我们研究的重点，所以在上面的图中我们就把 `SecurityContextPersistenceFilter` 放到了头号位置上。

怎么知道是加载上述的这些过滤器呢？我们可以结合上一章中分析的源码，在 `WebSecurityConfiguration` 配置类的 `springSecurityFilterChain` 方法最后，`this.webSecurity.build()` 这一行上打一个断点，随后将工程还原到一开始什么也没有的状态（即除了 `SpringBootSecurityApplication` 之外什么也没有），以 Debug 的方式运行主启动类，等到程序停在断点处，进入 `build` 方法，可以来到 `AbstractSecurityBuilder` 中：

```java
public final O build() throws Exception {
  if (this.building.compareAndSet(false, true)) {
    this.object = doBuild();
    return this.object;
  }
  throw new AlreadyBuiltException("This object has already been built");
}
```

![](./img/202303/28sb-security1.png)

很明显构建的核心方法是 `doBuild` 方法，不过具体构建的实现逻辑不是我们关注的重点（感兴趣的小伙伴可以自行深入 `AbstractConfiguredSecurityBuilder` 的 `doBuild` 方法，后转 `WebSecurity` 的 `performBuild` 方法查看源码），我们重点是来看构造完成，内部的过滤器链。Debug 执行到 `return this.object;` 行后，可以发现 SpringSecurity 默认一共有 15 个过滤器，如下图所示，这也刚好跟上面我们给出来的图基本吻合。

![](./img/202303/28sb-security2.png)

#### 1.2 内置过滤器的作用

下面简单解释一下这 15 个过滤器都是做什么用的吧：

- `WebAsyncManagerIntegrationFilter` ：整个过滤器链的起点和重点，处理与异步请求相关的机制，确保即便是异步请求，依然能被 SpringSecurity 处理；
- **`SecurityContextPersistenceFilter`** ：它的作用是使后续的过滤器以及业务代码中，能够全局获取到当前登录人的信息；
  - 当前登录人的获取方式：`SecurityContextHolder.getContext()` ，从返回的 `SecurityContext` 中可以获取到 `Authentication` 对象
  - `Authentication` 对象的实现类包含 `UsernamePasswordAuthenticationToken` 、`RememberMeAuthenticationToken` 、`AnonymousAuthenticationToken` 等

- `HeaderWriterFilter` ：能够向 request 的 header 中添加一些特殊的信息，有了这些信息后在传统的前后端不分离项目中，可以在页面上使用 SpringSecurity 的特殊标签取出这些 header 信息，以实现元素的显示与隐藏等；
- `CsrfFilter` ：跨站请求伪造的核心过滤器，会对所有 POST 、PUT 、DELETE 类请求进行验证，检查其中提交的数据是否包含于 CSRF 相关的 Token 信息，如果没有则会响应 403 拒绝访问，以此来起到防止跨站请求伪造的效果；
- `LogoutFilter` ：精准匹配指定的注销登录请求（默认是 `/logout` ），以实现用户注销登录，清除认证信息的功能；

> 由此也可以看出一点，`Filter` 也可以实现类似于 `Servlet` 的功能。

- **`UsernamePasswordAuthenticationFilter`** ：重要的认证过滤器，它会精准匹配指定的登录请求（默认是 `/login` ），通过收集用户名和密码信息，进行认证的核心过滤器；
  - 下面的第 3 节会单独讲解认证的核心工作原理，此处不展开
- `DefaultLoginPageGeneratingFilter` ：默认登录页生成的过滤器，如果我们没有使用 `http.loginPage(xxx)` 方法指定登录页，则由它生成内置的表单登录页；
- `DefaultLogoutPageGeneratingFilter` ：默认注销确认页生成的过滤器，无论我们如何配置，它都会生成；
  - 可能会有小伙伴好奇，如果我们配置了自定义的 `logoutUrl` ，那它还能生效吗？答案是肯定的，它必定会拦截 GET 方式请求的 `/logout` 路径
- `BasicAuthenticationFilter` ：提供一个 HTTP basic 的认证方式，这个过滤器会自动截取请求头中 `Authentication` ，且内容是以 `Basic` 开头的数据，之后按照特定的规则解析出用户名和密码；
- `RequestCacheAwareFilter` ：借助一些外部容器（默认是 session ）缓存 `HttpServletRequest` 对象，可以解决一些重定向后重复验证身份等问题；
- `SecurityContextHolderAwareRequestFilter` ：将 `HttpServletRequest` 进行一层包装，使其功能更强大；
- **`AnonymousAuthenticationFilter`** ：重要的过滤器，当 `UsernamePasswordAuthenticationFilter` 和 `BasicAuthenticationFilter` 都没有有效提取并封装用户信息时，会由该过滤器生成一个匿名用户的信息 `AnonymousAuthenticationToken` ，并保存到 `SecurityContextHolder` 中；
- `SessionManagementFilter` ：该过滤器可以控制单个用户的不同 session 情况，可以限制单个用户可以开启多个 session 的数量，超额踢下线等；
- **`ExceptionTranslationFilter`** ：重要的过滤器，它可以收集并处理整个过滤器链上的异常并转换；
- **`FilterSecurityInterceptor`** ：重要的过滤器，它会根据当前登录信息，以及正在访问的资源路径，判定是否准许访问；
  - 下面的第 4 节会单独讲解授权的核心工作原理，此处不展开

既然过滤器这么多，它们都是如何加载上来的呢？这就需要我们深入源码了。

## 2. 过滤器链的加载机制

---

通过上一章的源码分析，我们已经知道 SpringSecurity 的核心过滤器是在 `WebSecurityConfiguration` 配置类中的 `springSecurityFilterChain` 方法创建的。

#### 2.1 DefaultSecurityFilterChain

各位，还记得刚才上面我们看到的 15 个过滤器构成的集合是在哪里吗？是不是封装在一个 `DefaultSecurityFilterChain` 中啊！也就是我们上面画的图中那整个过滤器链吧！那这个 `DefaultSecurityFilterChain` 是在哪里创建的呢？

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnDefaultWebSecurity
@ConditionalOnWebApplication(type = Type.SERVLET)
class SpringBootWebSecurityConfiguration {

	@Bean
	@Order(SecurityProperties.BASIC_AUTH_ORDER)
	SecurityFilterChain defaultSecurityFilterChain(HttpSecurity http) throws Exception {
		http.authorizeRequests().anyRequest().authenticated().and().formLogin().and().httpBasic();
		return http.build();
	}

}

```

看到这段源码后，各位是否感觉有些似曾相识？对了，这就是上一章中看到的那个自动配置类 `SecurityAutoConfiguration` 导入的配置类 `SpringBootWebSecurityConfiguration` ，这里面生成的过滤器链中包含的就是默认的那 15 个过滤器。

从方法中很明显可以看出，要生成 `SecurityFilterChain` ，需要前置依赖一个 `HttpSecurity` ，也就是我们在自己写的配置类 `configure` 方法中频繁使用的那个 `http` 对象，拿到之后调用我们熟悉的那些方法，完成请求资源的限制、表单登录等配置。

那下一个问题来了：这个 `HttpSecurity` 又是哪来的呢？

> 可能会有小伙伴好奇：我们在自己写的 `WebSecurityConfigurerAdapter` 子类中重写 `configure` 方法中可没写 `http.build();` ，只有前面的那些，为什么呢？不要着急，随着源码的一层一层剖析，各位都会知道的。

#### 2.2 HttpSecurity

借助 IDEA ，我们可以找到在 `HttpSecurityConfiguration` 配置类中有一个能够生产原型 `HttpSecurity` 的方法：

```java
    @Bean(HTTPSECURITY_BEAN_NAME)
    @Scope("prototype")
    HttpSecurity httpSecurity() throws Exception {
        WebSecurityConfigurerAdapter.LazyPasswordEncoder passwordEncoder = new WebSecurityConfigurerAdapter.LazyPasswordEncoder(
                this.context);
        AuthenticationManagerBuilder authenticationBuilder = new WebSecurityConfigurerAdapter
                .DefaultPasswordEncoderAuthenticationManagerBuilder(this.objectPostProcessor, passwordEncoder);
        authenticationBuilder.parentAuthenticationManager(authenticationManager());
        HttpSecurity http = new HttpSecurity(this.objectPostProcessor, authenticationBuilder, createSharedObjects());
        // @formatter:off
        http
            .csrf(withDefaults())
            .addFilter(new WebAsyncManagerIntegrationFilter())
            .exceptionHandling(withDefaults())
            .headers(withDefaults())
            .sessionManagement(withDefaults())
            .securityContext(withDefaults())
            .requestCache(withDefaults())
            .anonymous(withDefaults())
            .servletApi(withDefaults())
            .apply(new DefaultLoginPageConfigurer<>());
        http.logout(withDefaults());
        // @formatter:on
        return http;
    }
```

注意看这个方法体！`HttpSecurity` 在此处被显式 new 出来，之后进行了一长串方法的链式调用！这段调用中一共调用了 9 个内置的方法，手动注册了一个过滤器，以及手动应用了一个 Configurer ，总共是 11 个特性配置。虽然整段代码看上去还是很蒙圈，但我们可以**先大胆猜测**呀！这一个一个的方法，背后就是对应注册上面所说的那些过滤器！调用一个方法就注册一个！

是不是真如我们猜想那样，下面来通过 Debug 的方式来检验。

#### 2.3 Debug检验

将断点分别打在 `httpSecurity()` 方法的最后一行，以及 `SpringBootWebSecurityConfiguration` 的 `defaultSecurityFilterChain` 方法中，然后以 Debug 的方式启动工程，待断点停在 `httpSecurity()` 方法后，观察此时此刻的 `HttpSecurity` 。仔细观察 Debug 截图中框起来的几个部分，可以发现经过上面一系列调用后，最终存储到 `HttpSecurity` 中的是 1 个 `Filter` ，10 个 `Configurer` ，而且每个 `Configurer` 看上去的确都对应一个过滤器。

![](./img/202303/28sb-security3.png)

接下来我们放行这个断点，等断点来到 `SpringBootWebSecurityConfiguration` 的 `defaultSecurityFilterChain` 方法时，可以发现 **return** 语句执行前，`HttpSecurity` 中的 Configurer 已经累积到 13 个了，多出来的 3 个刚好对应了 `authorizeRequests()` 、`formLogin()` 、`httpBasic()` 三个方法。

![](./img/202303/28sb-security4.png)

截止到现在，总共数下来是 1 个过滤器和 13 个 Configurer 。

下面该执行 `HttpSecurity` 的 `build` 方法，构造过滤器链 `SecurityFilterChain` 并返回了。在老生常谈的 `build` → `doBuild` 方法调用后，我们来到 `AbstractConfiguredSecurityBuilder` 的 `doBuild` 方法中，乍一看这个方法好像很简单，但仔细再看会发现，这里面貌似定义的全都是逻辑骨架，具体的实现都藏在这一个一个的方法中（是不是突然想到了 WebMvc 中经典的模板方法实现）。

```java
    protected final O doBuild() throws Exception {
        synchronized (this.configurers) {
            this.buildState = BuildState.INITIALIZING;
            beforeInit();
            init();
            this.buildState = BuildState.CONFIGURING;
            beforeConfigure();
            configure();
            this.buildState = BuildState.BUILDING;
            O result = performBuild();
            this.buildState = BuildState.BUILT;
            return result;
        }
    }
```

在这一堆模板方法中，有一个方法无疑是最扎眼的：`configure()` ，这个方法会触发刚才在 `HttpSecurity` 中收集的那些 Configurer ，所以**研究过滤器链的加载机制，其实就是看这些 Configurer 都是怎么注册过滤器的**。

#### 2.4 Configurer的生效机制

进入 `configure` 方法中，可以看到它会将当前对象中已经收集好的 Configurer 都取出来（类型是 `SecurityConfigurer` ），并逐个调用其 `configure` 方法应用。

```java
    private void configure() throws Exception {
        Collection<SecurityConfigurer<O, B>> configurers = getConfigurers();
        for (SecurityConfigurer<O, B> configurer : configurers) {
            configurer.configure((B) this);
        }
    }
```

在此处如果打断点观察可以发现，取到的 13 个 Configurer 刚好就是跟前面配置类中调用的顺序完全一致，如下图所示。

![](./img/202303/28sb-security5.png)

我们以其中 `CsrfConfigurer` 和 `ExceptionHandingConfigurer` 两个为例，可以看得出来，无论是哪个 Configurer ，在其 `configure` 方法中一定有一个与之相匹配的 `Filter` 被初始化，并在方法最后调用 `http.addFilter(filter);` 将这个过滤器注册到 `HttpSecurity` 中。

```java
    // CsrfConfigurer
    public void configure(H http) {
        CsrfFilter filter = new CsrfFilter(this.csrfTokenRepository);
        // ......
        http.addFilter(filter);
    }

    // ExceptionHandingConfigurer
    public void configure(H http) {
        AuthenticationEntryPoint entryPoint = getAuthenticationEntryPoint(http);
        ExceptionTranslationFilter exceptionTranslationFilter = new ExceptionTranslationFilter(entryPoint,
                getRequestCache(http));
        // ......
        http.addFilter(exceptionTranslationFilter);
    }
```

可能有小伙伴立马反应过来不对劲！明明在初始化的时候我们看到的是 1 个手动注册的过滤器，还有 13 个 Configurer ，那为什么会出来 15 个过滤器呢？那是因为在 `DefaultLoginPageConfigurer` 中有添加两个过滤器（下图 Debug 实测也证明了两个过滤器都会被注册进去）。

![](./img/202303/28sb-security6.png)

由此我们就彻底找出了过滤器链中每个过滤器的加载机制。

## 3. 认证的核心工作原理

---

下面我们来着重关注 SpringSecurity 过滤器链中负责认证和授权的过滤器工作原理。有关认证的过滤器，其实就是在 1.2 小节中我们看到的那个跟用户名、密码相关的 `UsernamePasswordAuthenticationFilter` ，本小节我们就重点研究它，看看它是如何在底层完成认证工作的。

#### 3.1 内部结构

借助 IDE 翻开 `UsernamePasswordAuthenticationFilter` ，发现它本身内部没有什么组件集成，只是定义了几个常量，而向上查看它的父类 `AbstractAuthenticationProcessingFilter` 时，发现其中集成的组件非常多，当然正因为太多了，所以下面的源码中小册只贴出了与本小节核心源码分析相关的组件，其他的组件各位可以结合 IDE 自行查阅。

```java
public class UsernamePasswordAuthenticationFilter extends AbstractAuthenticationProcessingFilter

// 内部组合的组件较多，只展示重要的几个
public abstract class AbstractAuthenticationProcessingFilter extends GenericFilterBean
        implements ApplicationEventPublisherAware, MessageSourceAware {
    // 【核心】认证管理器
    private AuthenticationManager authenticationManager;
    // 处理记住我相关逻辑的组件
    private RememberMeServices rememberMeServices = new NullRememberMeServices();
    // 匹配登录地址的匹配器
    private RequestMatcher requiresAuthenticationRequestMatcher;
    // 认证成功和失败后的处理器
    private AuthenticationSuccessHandler successHandler = new SavedRequestAwareAuthenticationSuccessHandler();
    private AuthenticationFailureHandler failureHandler = new SimpleUrlAuthenticationFailureHandler();
    // ......
```

各位，还记得本章一开始小册画的那个图吗？在图上 `UsernamePasswordAuthenticationFilter` 的右侧还画了一个 `AuthenticationManager` ，其实从代码的角度上看，这两者应该是组合关系。`UsernamePasswordAuthenticationFilter` 这个家伙不愿意自己处理认证的逻辑，于是就喊了一个干活的，把这部分认证逻辑甩给这个干活的，也就是所谓的 `AuthenticationManager` （看似是不愿意干，实则为了解耦）。

#### 3.2 AuthenticationManager

既然 `UsernamePasswordAuthenticationFilter` 组合了 `AuthenticationManager` 用来实际地完成认证动作，那这个东西我们也应该简单知道下。

借助 IDE 找到 `AuthenticationManager` 的源码，可以发现它就是一个只有一个方法的接口而已：

```java
public interface AuthenticationManager {
    Authentication authenticate(Authentication authentication) throws AuthenticationException;
}
```

而 `AuthenticationManager` 接口的实现类好多都是代理型的（而且都是内部类），只有一个普通的实现类 `ProviderManager` ，那大概率这就是最终的实现了。我们也是先进入类的内部看一下结构：

```java
public class ProviderManager implements AuthenticationManager, MessageSourceAware, InitializingBean {
    // 认证功能提供方
    private List<AuthenticationProvider> providers = Collections.emptyList();
    // 具备层级关系
    private AuthenticationManager parent;
    // 其他成员 ......
```

上面源码中依然是展示了最重要的片段，两个成员属性分别意味着：

- `providers` 是一个集合，代表 `ProviderManager` 还不是那个真正负责认证判断的组件，而是委派给内部的 `AuthenticationProvider` 们；
- `parent` 属性的出现意味着 `AuthenticationManager` 是具备层级关系的，如果子认证器无法完成认证，则会交予父认证器再次认证，直到认证逻辑执行或者全部无法认证时抛出异常；

如果我们继续向下追查 `AuthenticationProvider` 的实现类，会发现这次可麻烦了，它有一大堆实现类：

![](./img/202303/28sb-security7.png)

我们应该看哪个呢？乍一看有两个 “嫌疑者” ，分别是 `AbstractUserDetailsAuthenticationProvider` 和 `DaoAuthenticationProvider` ，然而当我们点开源码时会发现，`DaoAuthenticationProvider` 就是前者的具体实现子类。。。所以基本没跑了，我们关心这个 `DaoAuthenticationProvider` 就可以。

#### 3.3 工作流程源码分析

好了，通过前面两个小环节简单了解了认证过滤器和认证器的内部构造后，下面我们通过 Debug 的方式来探究一下，整个认证过程中内部都发生了什么。

##### 3.3.1 AbstractAuthenticationProcessingFilter#doFilter

Debug 启动我们在前面 26 ~ 28 章编写好的完全体工程，通过浏览器或者 API 工具发送一个登录请求，将断点打在 `AbstractAuthenticationProcessingFilter` 的 `doFilter` 方法的第一行（注意是方法参数为 `HttpServletRequest` 和 `HttpServletResponse` 的那个），待程序停在断点处，开始逐行执行。

我们先统览下这个 `doFilter` 方法，整个方法分为 5 个关键步骤，相关注释已标注在源码中，各位先自行阅读一遍。

```java
  private void doFilter(HttpServletRequest request, HttpServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        // 1. 当前请求路径不是 /login，放行
        if (!requiresAuthentication(request, response)) {
            chain.doFilter(request, response);
            return;
        }
        try {
            // 2. 【核心】身份认证
            Authentication authenticationResult = attemptAuthentication(request, response);
            if (authenticationResult == null) {
                // return immediately as subclass has indicated that it hasn't completed
                return;
            }
            // 3. 处理session
            this.sessionStrategy.onAuthentication(authenticationResult, request, response);
            // Authentication success
            if (this.continueChainBeforeSuccessfulAuthentication) {
                chain.doFilter(request, response);
            }
            // 4. 认证成功后的处理
            successfulAuthentication(request, response, chain, authenticationResult);
        } catch (InternalAuthenticationServiceException failed) {
            this.logger.error("An internal error occurred while trying to authenticate the user.", failed);
            unsuccessfulAuthentication(request, response, failed);
        } catch (AuthenticationException ex) {
            // Authentication failed
            // 5. 认证失败后的处理
            unsuccessfulAuthentication(request, response, ex);
        }
    }
```

下面开始 Debug ，很明显整个认证的核心是第 2 步 `attemptAuthentication` 方法中，而这个方法来自最终子类 `UsernamePasswordAuthenticationFilter` 中，我们跟进去。

##### 3.3.2 UsernamePasswordAuthenticationFilter#attemptAuthentication

```java
public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response)
  throws AuthenticationException {
  if (this.postOnly && !request.getMethod().equals("POST")) {
    throw new AuthenticationServiceException("Authentication method not supported: " + request.getMethod());
  }
  String username = obtainUsername(request);
  username = (username != null) ? username : "";
  username = username.trim();
  String password = obtainPassword(request);
  password = (password != null) ? password : "";
  UsernamePasswordAuthenticationToken authRequest = 
    new UsernamePasswordAuthenticationToken(username, password);
  // Allow subclasses to set the "details" property
  setDetails(request, authRequest);
  return this.getAuthenticationManager().authenticate(authRequest);
}
```

进入 `attemptAuthentication` 方法中，它首先要收集表单中的 `username` 和 `password` 属性，封装一个 `UsernamePasswordAuthenticationToken` ，之后交给 `AuthenticationManager` 去执行认证动作。请记住这个 `UsernamePasswordAuthenticationToken` ，这个类型是后续判断的因素之一。

##### 3.3.3 ProviderManager#authenticate

由于 `authenticate` 方法的篇幅比较长，而我们重点关注的是 `AuthenticationManager` 委托 `AuthenticationProvider` 的逻辑，所以小册只会展示前 1/3 左右的源码（关键注释已标注在源码中）。

```java
public Authentication authenticate(Authentication authentication) throws AuthenticationException {
    Class<? extends Authentication> toTest = authentication.getClass();
    AuthenticationException lastException = null;
    AuthenticationException parentException = null;
    Authentication result = null;
    Authentication parentResult = null;
    int currentPosition = 0;
    int size = this.providers.size();
    // 循环检查每个AuthenticationProvider，检查是否能支持当前传入的token
    for (AuthenticationProvider provider : getProviders()) {
        if (!provider.supports(toTest)) {
            continue;
        }
        // logger ......
        try {
            // 使用AuthenticationProvider进行身份验证
            result = provider.authenticate(authentication);
            if (result != null) {
                copyDetails(authentication, result);
                break;
            }
        } catch (AccountStatusException | InternalAuthenticationServiceException ex) {
            prepareException(ex, authentication);
            throw ex;
        } catch (AuthenticationException ex) {
            // 认证失败，将异常记录到临时变量中 ......
            lastException = ex;
        }
    }
    // ......
}
```

注意中间的 for 循环，它会将当前 `ProviderManager` （也即 `AuthenticationManager` ）中的那个 `AuthenticationProvider` 集合拿过来，循环去检查是否能够支持当前传入的 `UsernamePasswordAuthenticationToken` ，并且当支持时，执行其 `authenticate` 方法真正执行身份验证的逻辑（很明显这又是委托的体现了）。

每个 `AuthenticationProvider` 如何知道自己能否支持当前的 token 呢？它是通过类型判断的，而 Debug 可知，`DaoAuthenticationProvider` 支持的类型刚好就是我们正在使用的 `UsernamePasswordAuthenticationToken` 。

```java
    public boolean supports(Class<?> authentication) {
        return (UsernamePasswordAuthenticationToken.class.isAssignableFrom(authentication));
    }
```

##### 3.3.4 AuthenticationProvider#authenticate（1）

由于 `authenticate` 方法的源码篇幅略长，所以下面我们拆分为两段来看。

AbstractUserDetailsAuthenticationProvider

```java
public Authentication authenticate(Authentication authentication) throws AuthenticationException {
    // assert ......
    String username = determineUsername(authentication);
    boolean cacheWasUsed = true;
    UserDetails user = this.userCache.getUserFromCache(username);
    if (user == null) {
        cacheWasUsed = false;
        try {
            // 关键步骤
            user = retrieveUser(username, (UsernamePasswordAuthenticationToken) authentication);
        } catch (UsernameNotFoundException ex) {
            // 处理异常 ......
        }
        Assert.notNull(user, "retrieveUser returned null - a violation of the interface contract");
    }
    // ......
```

第一段的核心逻辑是加载 `UserDetails` ，也就是加载一个能被 SpringSecurity 正确识别的用户对象，如果加载过程中抛出了 `UsernameNotFoundException` 异常，则意味着用户名不存在；如果返回的 `UserDetails` 对象是 null ，那也意味着没有找到一个正确的用户信息。

那 `UserDetails` 是怎么被加载的呢？还记得我们在 25 章学习接入数据库认证时，有自己重写过一个 `UserDetailsService` 吗？对了，这个 `retrieveUser` 方法就是调用 `UserDetailsService` 的 `loadUserByUsername` 方法，由我们自定义的逻辑来加载。

```java
	protected final UserDetails retrieveUser(String username, UsernamePasswordAuthenticationToken authentication)
			throws AuthenticationException {
		prepareTimingAttackProtection();
		try {
            // 调用我们自己写的UserDetailsService
			UserDetails loadedUser = this.getUserDetailsService().loadUserByUsername(username);
			if (loadedUser == null) {
				// throw ex ......
			}
			return loadedUser;
		} // catch ex ......
	}
```

`retrieveUser` 方法执行完毕后，此时我们拿到的 `UserDetails` 对象就应该是具备用户名、被加密后的密码、拥有的角色、权限等附加信息于一身的了。

##### 3.3.5 AuthenticationProvider#authenticate（2）

```java
    // ......
    try {
        this.preAuthenticationChecks.check(user);
        // 【关键】附加的检查条件
        additionalAuthenticationChecks(user, (UsernamePasswordAuthenticationToken) authentication);
    } // catch ex ......
    this.postAuthenticationChecks.check(user);
    if (!cacheWasUsed) {
        this.userCache.putUserInCache(user);
    }
    Object principalToReturn = user;
    if (this.forcePrincipalAsString) {
        principalToReturn = user.getUsername();
    }
    return createSuccessAuthentication(principalToReturn, authentication, user);
}
```

用户被成功查出后，还需要比对密码一致才可以认定登录成功。在 authenticate 方法的后半段中，上面的 try 块第二行代码 `additionalAuthenticationChecks` 方法就是密码比对的实现。

我们再次来到子类 `DaoAuthenticationProvider` 中，找到 `additionalAuthenticationChecks` 方法的实现逻辑，发现这里做了两次检查：1）密码不能为空；2）传入的密码在被 `PasswordEncoder` 加密后能够跟数据库中查出的用户密码比对匹配成功。比对成功后，则方法可以正常执行完毕，后续就可以执行认证成功的逻辑；而比对失败时，我们看到抛出的异常是 `BadCredentialsException` 。

```java
protected void additionalAuthenticationChecks(UserDetails userDetails,
        UsernamePasswordAuthenticationToken authentication) throws AuthenticationException {
    if (authentication.getCredentials() == null) {
        this.logger.debug("Failed to authenticate since no credentials provided");
        throw new BadCredentialsException(this.messages
                .getMessage("AbstractUserDetailsAuthenticationProvider.badCredentials", "Bad credentials"));
    }
    String presentedPassword = authentication.getCredentials().toString();
    if (!this.passwordEncoder.matches(presentedPassword, userDetails.getPassword())) {
        this.logger.debug("Failed to authenticate since password does not match stored value");
        throw new BadCredentialsException(this.messages
                .getMessage("AbstractUserDetailsAuthenticationProvider.badCredentials", "Bad credentials"));
    }
}
```

##### 3.3.6 AbstractAuthenticationProcessingFilter#successfulAuthentication

```java
	protected Authentication createSuccessAuthentication(Object principal, Authentication authentication,
			UserDetails user) {
		// Ensure we return the original credentials the user supplied,
		// so subsequent attempts are successful even with encoded passwords.
		// Also ensure we return the original getDetails(), so that future
		// authentication events after cache expiry contain the details
		UsernamePasswordAuthenticationToken result = new UsernamePasswordAuthenticationToken(principal,
				authentication.getCredentials(), this.authoritiesMapper.mapAuthorities(user.getAuthorities()));
		result.setDetails(authentication.getDetails());
		this.logger.debug("Authenticated user");
		return result;
	}
```

登录成功后，我们拿到的 `UsernamePasswordAuthenticationToken` 中就包含了 `principal` ，也即 `UserDetails` 了。最后来到 `AbstractAuthenticationProcessingFilter` 的第 4 个环节 `successfulAuthentication` 方法中，最后一行会回调 AuthenticationSuccessHandler ，完成我们自定义的页面跳转 / json 数据响应等。

```java
protected void successfulAuthentication(HttpServletRequest request, HttpServletResponse response, FilterChain chain,
        Authentication authResult) throws IOException, ServletException {
    SecurityContextHolder.getContext().setAuthentication(authResult);
    // logger ......
    // 当记住我功能打开时，回调其逻辑
    this.rememberMeServices.loginSuccess(request, response, authResult);
    // 广播事件 ......
    // 回调AuthenticationSuccessHandler
    this.successHandler.onAuthenticationSuccess(request, response, authResult);
}
```

至此，一个完整的认证动作执行完毕，可以看到核心的逻辑是内部两层委托，最终是从 `UserDetailsService` 中取出用户，比对密码是否一致。

## 4. 授权的核心工作原理

下面我们再来看授权的工作原理。负责授权的过滤器是在整个过滤器链最底端的 `FilterSecurityInterceptor` ，在本章一开始给出的图中有展示，`FilterSecurityInterceptor` 也不是自己干活的，而是交给 `AccessDecisionManager` 来完成实际的授权工作。下面我们也来先统览 `FilterSecurityInterceptor` 的结构。

#### 4.1 内部结构

`FilterSecurityInterceptor` 本身也是有继承关系的，它继承的是 `AbstractSecurityInterceptor` ，从它以及父类集成的组件可以看出，`FilterSecurityInterceptor` 主要依赖的是一个 `FilterInvocationSecurityMetadataSource` 和 `AccessDecisionManager` 。

```java
public class FilterSecurityInterceptor extends AbstractSecurityInterceptor implements Filter {

    private FilterInvocationSecurityMetadataSource securityMetadataSource;
```

```java
public abstract class AbstractSecurityInterceptor
        implements InitializingBean, ApplicationEventPublisherAware, MessageSourceAware {
    // ......

    private AccessDecisionManager accessDecisionManager;
```

这两个组件分别都是干什么的呢？不妨先简单了解下。

`FilterInvocationSecurityMetadataSource` 负责将当前请求的路径 uri 转换为一组权限标识，用于 `AccessDecisionManager` 校验比对；

`AccessDecisionManager` 负责拿到 `SecurityMetadataSource` 转换的权限标识，跟当前登录人的状态、拥有的权限等综合比对，决定是否准许放行。

对于这个 `AccessDecisionManager` ，小册还需要再展开说说，它本身也是一个接口，一个最常见的实现类是 `AffirmativeBased` ，而它的内部设计与 `AuthenticationManager` 有些类似，都是内部又组合了其他的对象，在 `AffirmativeBased` 的构造方法中可以看出，它组合的是 `AccessDecisionVoter` ，类名直译为 “访问判定投票器” 。

```java
public class AffirmativeBased extends AbstractAccessDecisionManager {

	public AffirmativeBased(List<AccessDecisionVoter<?>> decisionVoters) {
		super(decisionVoters);
	}
```

而借助 IDEA 可以找到 `AccessDecisionVoter` 的实现类如下图所示，虽然不能完全看懂，但也能看得出来有 JSR-250 相关注解的投票器、基于角色的投票器等。至于这些投票器如何在 `AccessDecisionManager` 中发挥作用，下面我们马上来结合工作流程的源码来分析。

#### 4.2 工作流程源码分析

现在我们的测试代码按道理是无法测试出 `FilterSecurityInterceptor` 的作用，为了让它起到相应的作用，我们需要修改 `WebSecurityConfiguration` 中的 `configure` 方法，暂时屏蔽掉我们自己写的 `resourceAuthorizationFilter` ，而转用 `antMatcher` 的方式匹配限制资源访问规则。

```java
// http.addFilterAfter(resourceAuthorizationFilter, FilterSecurityInterceptor.class);
   http.authorizeRequests().antMatchers("/demo").hasRole("user")
           .antMatchers("/api/test").hasAuthority("/api/test").anyRequest().authenticated();
```

如此限制后，访问 `/demo` 接口需要 `user` 角色，访问 `/api/test` 接口需要 `/api/test` 权限标识，这样就可以来测试了。

##### 4.2.1 FilterSecurityInterceptor#doFilter

以 Debug 的方式启动工程，使用接口工具登录成功后，我们拿到 jwt 令牌，携带令牌访问 `/api/test` 接口，进入 `FilterSecurityInterceptor` 中。我们在 `invoke` 方法中打入断点，待程序停在断点时开始 Debug 。整个 `invoke` 方法中，最核心的动作是 `super.beforeInvocation()` ，如果这个方法能够正常执行通过，那么下一步就会继续向后执行了（也即放行了请求）。

```java
public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
        throws IOException, ServletException {
    invoke(new FilterInvocation(request, response, chain));
}

public void invoke(FilterInvocation filterInvocation) throws IOException, ServletException {
    // 前置准备 ......
    // 
    InterceptorStatusToken token = super.beforeInvocation(filterInvocation);
    try {
        filterInvocation.getChain().doFilter(filterInvocation.getRequest(), filterInvocation.getResponse());
    } finally {
        super.finallyInvocation(token);
    }
    super.afterInvocation(token, null);
}
```

##### 4.2.2 beforeInvocation

```java
protected InterceptorStatusToken beforeInvocation(Object object) {
    // 前置检查 .......
    Collection<ConfigAttribute> attributes = this.obtainSecurityMetadataSource().getAttributes(object);
    // 检查 .......
    Authentication authenticated = authenticateIfRequired();
    // logger ......
    // Attempt authorization
    attemptAuthorization(object, attributes, authenticated);
    // 认证后的动作 .......
}

private void attemptAuthorization(Object object, Collection<ConfigAttribute> attributes, Authentication authenticated) {
    try {
        this.accessDecisionManager.decide(authenticated, object, attributes);
    } // catch throw ex ......
}
```

进入到 `beforeInvocation` 方法中，虽然这个方法篇幅很长，但是抽取出核心的源码就三行：获取当前请求 uri 所需要的权限规则、获取当前登录人信息，以及执行授权操作。而授权动作执行的 `attemptAuthorization` ，则是直接委托给 `AccessDecisionManager` 了。

##### 4.2.3 DefaultFilterInvocationSecurityMetadataSource#getAttributes

```java
    private final Map<RequestMatcher, Collection<ConfigAttribute>> requestMap;

    public Collection<ConfigAttribute> getAttributes(Object object) {
        final HttpServletRequest request = ((FilterInvocation) object).getRequest();
        int count = 0;
        for (Map.Entry<RequestMatcher, Collection<ConfigAttribute>> entry : this.requestMap.entrySet()) {
            if (entry.getKey().matches(request)) {
                return entry.getValue();
            } // else logger ......
        }
        return null;
    }
```

我们首先看解析权限的环节，默认的实现是 `DefaultFilterInvocationSecurityMetadataSource` ，它会将前面我们在 `configure` 方法中使用 `http.authorizeRequests()` 设置的规则封装为一个 `requestMap` ，在请求到达时匹配获得对应的授权规则。

Debug 至此时，可以看到 `/api/test` 请求对应的授权规则已经被成功匹配到，共有 1 个规则，如下图所示。

![](./img/202303/28sb-security8.png)

##### 4.2.4 AffirmativeBased#decide

```java
public void decide(Authentication authentication, Object object, Collection<ConfigAttribute> configAttributes)
        throws AccessDeniedException {
    int deny = 0;
    for (AccessDecisionVoter voter : getDecisionVoters()) {
        int result = voter.vote(authentication, object, configAttributes);
        switch (result) {
        case AccessDecisionVoter.ACCESS_GRANTED:
            return;
        case AccessDecisionVoter.ACCESS_DENIED:
            deny++;
            break;
        default:
            break;
        }
    }
    if (deny > 0) {
        throw new AccessDeniedException(this.messages.getMessage("AbstractAccessDecisionManager.accessDenied", "Access is denied"));
    }
    // To get this far, every AccessDecisionVoter abstained
    checkAllowIfAllAbstainDecisions();
}
```

拿到当前请求必备的权限后，下面我们进入到 `AccessDecisionManager` 的实现类 `AffirmativeBased` 中。一进来我们马上看到了委托 `AccessDecisionVoter` 的逻辑，不过与 `AuthenticationManager` 不同的是，`AccessDecisionManager` 实行的是 “投票机制” ，让当前对象内的所有 `AccessDecisionVoter` 都判定一遍，只要有一个投反对票，则认定当前请求不允许访问，拦截响应 403 错误。

![](./img/202302/28sb-security9.png)

Debug 至此时发现投票器只有一个：

这个 `WebExpressionVoter` 可以处理的就是 `hasRole` 、`hasAuthority` 等 SpEL 表达式，我们进入它的 `vote` 方法中一探究竟。

```java
    public int vote(Authentication authentication, FilterInvocation filterInvocation,
            Collection<ConfigAttribute> attributes) {
        // assert .......
        WebExpressionConfigAttribute webExpressionConfigAttribute = findConfigAttribute(attributes);
        if (webExpressionConfigAttribute == null) {
            return ACCESS_ABSTAIN;
        }
        // 解析、执行SpEL表达式
        EvaluationContext ctx = webExpressionConfigAttribute.postProcess(
                this.expressionHandler.createEvaluationContext(authentication, filterInvocation), filterInvocation);
        boolean granted = ExpressionUtils.evaluateAsBoolean(webExpressionConfigAttribute.getAuthorizeExpression(), ctx);
        if (granted) {
            return ACCESS_GRANTED;
        }
        this.logger.trace("Voted to deny authorization");
        return ACCESS_DENIED;
    }
```

很明显，vote 方法中的核心就是解析和执行 SpEL 表达式，并且最终收到一个 boolean 值，决定是放行还是拦截。至于 SpEL 表达式的内部是如何匹配的，我们就不再深入探讨，各位只需要知道如何触发的 SpEL 表达式即可。

> 对 SpEL 表达式感兴趣的小伙伴可以跳转到 `SecurityExpressionRoot` 中查看。

至此，一个完整的授权过程也就执行完毕了。

【通过这两章的内部结构、自动装配、工作机制分析后，各位对 SpringSecurity 是否有更深的认识了呢？SpringSecurity 本身设计的非常复杂，这也使得它很强大，在实际的项目开发中一定要灵活运用这些特性。】

![](./img/202303/28sb-security10.png)

