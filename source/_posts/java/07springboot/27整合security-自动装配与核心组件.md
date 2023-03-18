---
title: 27整合security-自动装配与核心组件
toc: true
tags: springboot
categories: 
    - [java]
    - [springboot]
---

通过前面两章，我们已经了解了很多 SpringSecurity 的强大机制，想必各位也能够强烈的感受到 SpringSecurity 给我们考虑之全面。接下来的两章，我们来研究一下 SpringSecurity 在整合 SpringBoot 后，底层都装配了什么，注册了哪些核心组件，以及这些核心组件是如何构成 SpringSecurity 的安全控制模型。

## 1. 自动装配

从 `spring.factories` 中寻找 `EnableAutoConfiguration` 的配置部分，我们可以很容易地找到与 SpringSecurity 相关的自动配置类：

<!--more-->


```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
    org.springframework.boot.autoconfigure.security.servlet.SecurityAutoConfiguration,\
    org.springframework.boot.autoconfigure.security.servlet.UserDetailsServiceAutoConfiguration,\
    org.springframework.boot.autoconfigure.security.servlet.SecurityFilterAutoConfiguration,\
    org.springframework.boot.autoconfigure.security.reactive.ReactiveSecurityAutoConfiguration,\
    org.springframework.boot.autoconfigure.security.reactive.ReactiveUserDetailsServiceAutoConfiguration,\
    org.springframework.boot.autoconfigure.security.rsocket.RSocketSecurityAutoConfiguration,\
    org.springframework.boot.autoconfigure.security.saml2.Saml2RelyingPartyAutoConfiguration,\
    org.springframework.boot.autoconfigure.security.oauth2.client.servlet.OAuth2ClientAutoConfiguration,\
    org.springframework.boot.autoconfigure.security.oauth2.client.reactive.ReactiveOAuth2ClientAutoConfiguration,\
    org.springframework.boot.autoconfigure.security.oauth2.resource.servlet.OAuth2ResourceServerAutoConfiguration,\
    org.springframework.boot.autoconfigure.security.oauth2.resource.reactive.ReactiveOAuth2ResourceServerAutoConfiguration,\
    ......
```

![](./img/202303/27sb-security.png)

可以发现 SpringSecurity 的自动装配涉及到的方面非常多，不过仔细来看，我们在整合 WebMvc 时，只会用到上面的 3 个自动配置类，所以我们重点来关注它们即可。

## 2. SecurityAutoConfiguration

`SecurityAutoConfiguration` 负责的是 SpringSecurity 的总体核心组件注册，通过前面几个模块的研究，我们应该能总结出一个规律：这种与模块名直接相关的自动配置类，大概率都是起到一个核心注册的效果，还有导入其他配置类的功效。借助 IDE 观察源码，可以发现 `SecurityAutoConfiguration` 也不例外。

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(DefaultAuthenticationEventPublisher.class)
@EnableConfigurationProperties(SecurityProperties.class)
@Import({ SpringBootWebSecurityConfiguration.class, WebSecurityEnablerConfiguration.class,
		SecurityDataConfiguration.class })
public class SecurityAutoConfiguration {

	@Bean
	@ConditionalOnMissingBean(AuthenticationEventPublisher.class)
	public DefaultAuthenticationEventPublisher authenticationEventPublisher(
    ApplicationEventPublisher publisher) {
		return new DefaultAuthenticationEventPublisher(publisher);
	}

}
```

除了配置类中注册了一个 `DefaultAuthenticationEventPublisher` 之外，它还导入了 3 个配置类，下面一一来看。

#### 2.1 SpringBootWebSecurityConfiguration

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

`SpringBootWebSecurityConfiguration` 负责的工作是给我们 SpringSecurity 中装配表单登录页、 http-basic 认证机制，以及拦截所有资源的访问。这也印证了我们在第 25 章的案例第一环节看到的默认装配效果！

说到这里我们顺便解答一个疑惑：为什么当我们向 IOC 容器中注册一个 `WebSecurityConfigurerAdapter` 的子类后，默认的安全控制行为就退出了呢？在 `defaultSecurityFilterChain` 的方法上并没有标注 `@Conditional` 系列注解呀。那是因为退出装配的是整个配置类，我们需要关注一下 `SpringBootWebSecurityConfiguration` 类上标注的这个特殊注解 `@ConditionalOnDefaultWebSecurity` 。

```java
@Conditional(DefaultWebSecurityCondition.class)
public @interface ConditionalOnDefaultWebSecurity {
}
```

这个注解使用 `@Conditional` 注解导入了一个特殊的条件判断类 `DefaultWebSecurityCondition` ，这其实就是原生条件注解的使用。而点开 `DefaultWebSecurityCondition` 的实现，可以发现它其实是一个由多个条件构成的复合条件。而在这些条件中有一个就是解答这个疑惑的点： **`@ConditionalOnMissingBean(WebSecurityConfigurerAdapter.class)`** ，这就很好理解了，我们没有向 IOC 容器中注册 `WebSecurityConfigurerAdapter` 的子类时这个条件才能判断生效，否则相反。

```java
class DefaultWebSecurityCondition extends AllNestedConditions {

    DefaultWebSecurityCondition() {
        super(ConfigurationPhase.REGISTER_BEAN);
    }

    @ConditionalOnClass({ SecurityFilterChain.class, HttpSecurity.class })
    static class Classes { }

    @ConditionalOnMissingBean({ WebSecurityConfigurerAdapter.class, SecurityFilterChain.class })
    static class Beans { }
}
```

#### 2.2 WebSecurityEnablerConfiguration

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnMissingBean(name = BeanIds.SPRING_SECURITY_FILTER_CHAIN)
@ConditionalOnClass(EnableWebSecurity.class)
@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.SERVLET)
@EnableWebSecurity
class WebSecurityEnablerConfiguration {

}
```

下一个配置类 `WebSecurityEnablerConfiguration` 只从类名上就足以读懂它的作用：开启 WebSecurity ，毫无疑问，它的作用就是帮助我们标注 `@EnableWebSecurity` 注解而已。当然，在实际的项目开发中，我们一般都跟随 `WebSecurityConfigurerAdapter` 的子类上一起标注了，所以这个配置类我们可以不用那么重视，仅仅是知道有个帮我们兜底的即可。

#### 2.3 SecurityDataConfiguration

最后一个配置类默认是不会生效的，因为默认导入 `spring-boot-starter-security` 时，整个工程中是缺少 `SecurityEvaluationContextExtension` 类的，所以根本不会生效。

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(SecurityEvaluationContextExtension.class)
public class SecurityDataConfiguration {

	@Bean
	@ConditionalOnMissingBean
	public SecurityEvaluationContextExtension securityEvaluationContextExtension() {
		return new SecurityEvaluationContextExtension();
	}

}
```

> 简单解释一下，`SecurityEvaluationContextExtension` 的作用是与 SpringData 的整合，需要我们手动导入 `spring-security-data` 依赖才会生效。

#### 2.4 注册的其他组件

过完 `SecurityAutoConfiguration` 导入的其他配置类后，最后再来看看它内部还注册的一个组件：`DefaultAuthenticationEventPublisher` 。

```java
@Bean
@ConditionalOnMissingBean(AuthenticationEventPublisher.class)
public DefaultAuthenticationEventPublisher 
  authenticationEventPublisher(ApplicationEventPublisher publisher) {
  return new DefaultAuthenticationEventPublisher(publisher);
}
```

`DefaultAuthenticationEventPublisher` 这个家伙其实是 SpringSecurity 3.0 就有的组件，负责的核心工作是在用户认证成功和失败时，向 IOC 容器中广播事件。我们可以先看一下它实现的接口：

```java
public class DefaultAuthenticationEventPublisher
        implements AuthenticationEventPublisher, ApplicationEventPublisherAware 
```

毫无疑问，这个家伙归属的根接口就是 `AuthenticationEventPublisher` ，而这个接口只定义了两个方法，恰好对应了认证成功和失败：

```java
public interface AuthenticationEventPublisher {
    void publishAuthenticationSuccess(Authentication authentication);
    void publishAuthenticationFailure(AuthenticationException exception, Authentication authentication);
}
```

那认证成功和失败的时候，它都会广播什么事件呢？我们也可以来看一下。

**认证成功**

```java
public void publishAuthenticationSuccess(Authentication authentication) {
    if (this.applicationEventPublisher != null) {
        this.applicationEventPublisher.publishEvent(new AuthenticationSuccessEvent(authentication));
    }
}
```

认证成功时，它广播的事件类型为 `AuthenticationSuccessEvent` ，所以我们只需要编写一个 `ApplicationListener` ，监听这个 `AuthenticationSuccessEvent` 事件，就可以在用户登录成功后做一些额外的处理。

> 当然小伙伴可能会想到 `successHandler` ，当然也是可以的，只不过使用监听器的方式有一个最大的好处：与 SpringSecurity 的核心代码解耦，这也是事件驱动机制想要解决的核心问题。

**认证失败**

比起认证成功的千篇一律，认证失败的事件可就“多姿多彩”了。各位如果乍一看源码，还会以为是不是写错了，为什么没有找到事件的类型：

```java
public void publishAuthenticationFailure(AuthenticationException exception, Authentication authentication) {
  // 此处竟然会根据异常类型去寻找构造器？？？
  Constructor<? extends AbstractAuthenticationEvent> constructor = getEventConstructor(exception);
  AbstractAuthenticationEvent event = null;
  if (constructor != null) {
    try {
      event = constructor.newInstance(authentication, exception);
    } // catch ......
  }
  if (event != null) {
    if (this.applicationEventPublisher != null) {
      this.applicationEventPublisher.publishEvent(event);
    }
  } // else logger ......
}
```

根据捕获到的异常去寻找构造器？寻找什么构造器？很明显，下面要广播的事件的构造器嘛！不过它整这么一出是图个啥呢？我们得进到 getEventConstructor 方法中一探究竟才行。

```java
private Constructor<? extends AbstractAuthenticationEvent> getEventConstructor(
  AuthenticationException exception) {
  Constructor<? extends AbstractAuthenticationEvent> eventConstructor = this.exceptionMappings
    .get(exception.getClass().getName());
  return (eventConstructor != null) ? eventConstructor : this.defaultAuthenticationFailureEventConstructor;
}
```

怎么读起来还兜兜转转的？说是找构造器，怎么又去一个 `exceptionMappings` 集合中去 get 呢？这又是个啥？来，往下看，下面就是答案：

```java
private final HashMap<String, Constructor<? extends AbstractAuthenticationEvent>> exceptionMappings = new HashMap<>();
// ......
public DefaultAuthenticationEventPublisher(ApplicationEventPublisher applicationEventPublisher) {
  this.applicationEventPublisher = applicationEventPublisher;
  addMapping(BadCredentialsException.class.getName(), AuthenticationFailureBadCredentialsEvent.class);
  addMapping(UsernameNotFoundException.class.getName(), AuthenticationFailureBadCredentialsEvent.class);
  addMapping(AccountExpiredException.class.getName(), AuthenticationFailureExpiredEvent.class);
  addMapping(ProviderNotFoundException.class.getName(), AuthenticationFailureProviderNotFoundEvent.class);
  addMapping(DisabledException.class.getName(), AuthenticationFailureDisabledEvent.class);
  addMapping(LockedException.class.getName(), AuthenticationFailureLockedEvent.class);
  addMapping(AuthenticationServiceException.class.getName(), AuthenticationFailureServiceExceptionEvent.class);
  addMapping(CredentialsExpiredException.class.getName(), AuthenticationFailureCredentialsExpiredEvent.class);
  addMapping("org.springframework.security.authentication.cas.ProxyUntrustedException",
             AuthenticationFailureProxyUntrustedEvent.class);
  addMapping("org.springframework.security.oauth2.server.resource.InvalidBearerTokenException",
             AuthenticationFailureBadCredentialsEvent.class);
 }

private void addMapping(String exceptionClass, Class<? extends AbstractAuthenticationFailureEvent> eventClass) {
  try {
    Constructor<? extends AbstractAuthenticationEvent> constructor = eventClass
      .getConstructor(Authentication.class, AuthenticationException.class);
    this.exceptionMappings.put(exceptionClass, constructor);
  }
  catch (NoSuchMethodException ex) {
    throw new RuntimeException(
      "Authentication event class " + eventClass.getName() + " has no suitable constructor");
  }
}
```

好家伙，合着 SpringSecurity 把每种用户认证失败时对应的异常都做了相应的事件啊！找到对应的事件，再获取默认的构造器，最终就可以得到每个异常对应的事件构造器了。真可谓：“朴实的登录成功千篇一律，异常的登录失败五花八门”。

好了，到此为止有关 `SecurityAutoConfiguration` 的配置就全部读完了，我们继续往下阅读。

## 3. UserDetailsServiceAutoConfiguration

第二个自动配置类是与 `UserDetailsService` 相关的。可能有的小伙伴会好奇，为什么我们默认导入 SpringSecurity 的时候什么也没做，内部就已经有了一个 `user` 用户了呢？既然说 SpringSecurity 对于用户管理体系只认 `UserDetailsService` ，那默认的用户又放到哪里了呢？

好，带着这两个问题，我们来看 `UserDetailsServiceAutoConfiguration` 的源码。整个 `UserDetailsServiceAutoConfiguration` 的内部只注册了一个 Bean ，不过他刚好就是这个我们在 25 章中演示的基于内存的用户管理实现 `InMemoryUserDetailsManager` 。

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(AuthenticationManager.class)
@ConditionalOnBean(ObjectPostProcessor.class)
@ConditionalOnMissingBean(
        value = { AuthenticationManager.class, AuthenticationProvider.class, UserDetailsService.class },
        type = {......})
public class UserDetailsServiceAutoConfiguration {
    // ......

    @Bean
    @Lazy
    public InMemoryUserDetailsManager inMemoryUserDetailsManager(SecurityProperties properties,
            ObjectProvider<PasswordEncoder> passwordEncoder) {
        SecurityProperties.User user = properties.getUser();
        List<String> roles = user.getRoles();
        return new InMemoryUserDetailsManager(
                User.withUsername(user.getName()).password(getOrDeducePassword(user, passwordEncoder.getIfAvailable()))
                        .roles(StringUtils.toStringArray(roles)).build());
    }
    // ......
}
```

从装配注册的 `inMemoryUserDetailsManager` 方法中可以看出，它引用的是 SpringBoot 全局配置文件中的 `user` 信息，也就是 `spring.security.user` 开头的那一部分，之后创建一个 `InMemoryUserDetailsManager` ，内部填充我们配置（或 SpringBoot 默认提供）的内置用户信息，这也就是在没有任何配置的前提下，SpringSecurity 还能装配一个 `user` 用户供我们调试使用。而当我们向 IOC 容器中显式注册了 `UserDetailsService` 后，这个自动装配就退出了。

## 4. SecurityFilterAutoConfiguration

---

Security 相关的最后一个自动配置类是 `SecurityFilterAutoConfiguration` ，从类名上也能看得出来，它的核心工作是注册与 SpringSecurity 相关的过滤器。从源码上看，这个自动配置类也是只注册了一个 Bean ：

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnWebApplication(type = Type.SERVLET)
@EnableConfigurationProperties(SecurityProperties.class)
@ConditionalOnClass({ AbstractSecurityWebApplicationInitializer.class, SessionCreationPolicy.class })
@AutoConfigureAfter(SecurityAutoConfiguration.class)
public class SecurityFilterAutoConfiguration {

    private static final String DEFAULT_FILTER_NAME = 
      AbstractSecurityWebApplicationInitializer.DEFAULT_FILTER_NAME;

    @Bean
    @ConditionalOnBean(name = DEFAULT_FILTER_NAME)
    public DelegatingFilterProxyRegistrationBean securityFilterChainRegistration(
            SecurityProperties securityProperties) {
        DelegatingFilterProxyRegistrationBean registration = 
          new DelegatingFilterProxyRegistrationBean(DEFAULT_FILTER_NAME);
        registration.setOrder(securityProperties.getFilter().getOrder());
        registration.setDispatcherTypes(getDispatcherTypes(securityProperties));
        return registration;
    }

    // ......
}
```

等一下，不是说好了注册 Filter 吗？这是个什么东西？哎，如果小伙伴有这个疑问，那就太不应该了，这很明显是使用 SpringBoot 注册 `Filter` 的方式吧！SpringBoot 不能直接应用 Servlet 的三大组件，需要借助相应的 `RegistrationBean` 来辅助注册！之前在 SpringBoot 的小册中也讲过 `DispatcherServlet` 的注册，如果小伙伴不记得的话一定要回过头去看。

我们先仔细挖掘一下 `DelegatingFilterProxyRegistrationBean` 的注册过程，从中提取一些细节出来。

#### 1）@ConditionalOnBean(name = DEFAULT_FILTER_NAME)

为什么注册 `DelegatingFilterProxyRegistrationBean` 需要依赖一个名为 `DEFAULT_FILTER_NAME` 的 Bean 呢？各位，还记得我们在 24 章中提到的那个基本设计图吗？那里面有一个特殊的过滤器：

![](./img/202303/27sb-security2.png)

还记得这个 `DelegatingFilterProxy` 的作用吗？它的作用是**使用代理的机制，将本来注册到 IOC 容器中的 `Filter` ，挂载到 Servlet 容器的应用上，这样就可以实现 IOC 容器中注册的 `Filter` 也能参与 `Servlet` 访问的过滤**。那既然 `DelegatingFilterProxy` 会引用 IOC 容器中的 Bean ，那它的前提是 IOC 容器里得有这个 Bean 吧！所以这个条件装配的含义也就被解释了吧，IOC 容器里必须得有这个 `DEFAULT_FILTER_NAME` ，这个 `DelegatingFilterProxyRegistrationBean` 才能创建出来，之后初始化，并引用这个名为 `DEFAULT_FILTER_NAME` 的 Bean 。

#### 2）DelegatingFilterProxyRegistrationBean初始化了个谁

好，下一个问题也就紧接着来了，既然 `DelegatingFilterProxyRegistrationBean` 需要依赖名为 `DEFAULT_FILTER_NAME` 的 Bean ，那这个 Bean 是谁呢？借助 IDE 寻找这个常亮的定义位置，发现在 `AbstractSecurityWebApplicationInitializer` 中可以找到：

```java
public abstract class AbstractSecurityWebApplicationInitializer implements WebApplicationInitializer {

    private static final String SERVLET_CONTEXT_PREFIX = 
      "org.springframework.web.servlet.FrameworkServlet.CONTEXT.";

    public static final String DEFAULT_FILTER_NAME = "springSecurityFilterChain";

    // ......
```

OK ，我们已经知道这个过滤器的名叫 `springSecurityFilterChain` 了，暂且先不关心这到底实现类是谁，我们先来看看 `DEFAULT_FILTER_NAME` 这个常量在哪里得以利用。往下找到 `insertSpringSecurityFilterChain` 方法中，我们可以找到一段注册过滤器的逻辑，如下代码所示（有 2 个方法）。

```java
	private void insertSpringSecurityFilterChain(ServletContext servletContext) {
		String filterName = DEFAULT_FILTER_NAME;
		DelegatingFilterProxy springSecurityFilterChain = new DelegatingFilterProxy(filterName);
		String contextAttribute = getWebApplicationContextAttribute();
		if (contextAttribute != null) {
			springSecurityFilterChain.setContextAttribute(contextAttribute);
		}
		registerFilter(servletContext, true, filterName, springSecurityFilterChain);
	}

	private void registerFilters(ServletContext servletContext, boolean insertBeforeOtherFilters, Filter... filters) {
		Assert.notEmpty(filters, "filters cannot be null or empty");
		for (Filter filter : filters) {
			Assert.notNull(filter, () -> "filters cannot contain null values. Got " + Arrays.asList(filters));
			String filterName = Conventions.getVariableName(filter);
			registerFilter(servletContext, insertBeforeOtherFilters, filterName, filter);
		}
	}

```

`insertSpringSecurityFilterChain` 方法的逻辑本身不复杂，它会将过滤器名交给 `DelegatingFilterProxy` ，之后将它注册到 `ServletContext` 中。而注册过滤器的方式，就是 Servlet 3.0 规范中准许的，直接调用 `ServletContext` 的 `addFilter` 方法注册。

另外还有一个小细节哦，注册过滤器的时候，它指定了拦截路径是 `/*` ，这也就解释了 SpringSecurity 默认会拦截所有请求路径的原因。

#### 3）springSecurityFilterChain又是谁

注册过滤器的逻辑也明白了，下面我们就来追究这个 `springSecurityFilterChain` 了。既然显式指定了 Bean 的名称，那必然会有一个 `@Bean` 注解标注的方法引用了它，或者使用编程式构造 `BeanDefinition` 的方式注册了相应的 Bean 。

借助 IDE ，可以发现在 `WebSecurityConfiguration` 中有一个 `@Bean` 注解引用了这个常量（注意这个 `WebSecurityConfiguration` 是 SpringSecurity 内部的，不是我们写的那个！！！），对应的方法如下。

```java
private WebSecurity webSecurity;

@Bean(name = AbstractSecurityWebApplicationInitializer.DEFAULT_FILTER_NAME)
public Filter springSecurityFilterChain() throws Exception {
    boolean hasConfigurers = this.webSecurityConfigurers != null && !this.webSecurityConfigurers.isEmpty();
    boolean hasFilterChain = !this.securityFilterChains.isEmpty();
    Assert.state(!(hasConfigurers && hasFilterChain),
            "Found WebSecurityConfigurerAdapter as well as SecurityFilterChain. Please select just one.");
    if (!hasConfigurers && !hasFilterChain) {
        WebSecurityConfigurerAdapter adapter = this.objectObjectPostProcessor
                .postProcess(new WebSecurityConfigurerAdapter() {
                });
        this.webSecurity.apply(adapter);
    }
    for (SecurityFilterChain securityFilterChain : this.securityFilterChains) {
        this.webSecurity.addSecurityFilterChainBuilder(() -> securityFilterChain);
        for (Filter filter : securityFilterChain.getFilters()) {
            if (filter instanceof FilterSecurityInterceptor) {
                this.webSecurity.securityInterceptor((FilterSecurityInterceptor) filter);
                break;
            }
        }
    }
    for (WebSecurityCustomizer customizer : this.webSecurityCustomizers) {
        customizer.customize(this.webSecurity);
    }
    return this.webSecurity.build();
}
```

从源码中可以大体的看出，整个过滤器的初始化的核心逻辑，是拿 `WebSecurity` 反复摆弄，指定过滤器建造器、添加过滤器等一些操作，最终 `build` 出来一个过滤器，而中间涉及到的一堆过滤器，我们放到下一章中再研究。

下面我们考虑一个问题：既然 `WebSecurityConfiguration` 也是一个配置类，但是上面我们看过那么多自动配置类里面，貌似都没有见到它，那它是如何生效的呢？别急，借助 IDE ，我们看看这个 `WebSecurityConfiguration` 是不是在哪里被 `@Import` 了：

```java
@Import({ WebSecurityConfiguration.class, SpringWebMvcImportSelector.class, 
		  OAuth2ImportSelector.class, HttpSecurityConfiguration.class })
@EnableGlobalAuthentication
@Configuration
public @interface EnableWebSecurity
```

好家伙，合着就是我们手动编写的那个 `@EnableWebSecurity` 注解啊！其实上面我们基本都把配置类过了一遍，唯独这个 `@EnableWebSecurity` 注解没有展开看，刚好就是这个核心注解导入的 SpringSecurity 内部最核心的过滤器。

除此之外，`@EnableWebSecurity` 注解导入的配置类中还包含一些其他组件，不过这些组件大多都不是在核心配置和流程中容易接触到的，小册就不再展开了，感兴趣的小伙伴可以自行结合 IDE 去查看。

【自动装配的部分我们就看到这里吧，理清楚 SpringSecurity 中注册的核心组件后，下一章我们就来梳理和分析 SpringSecurity 底层用来控制安全访问的模型，并结合模型来解析实际运行的机制】

