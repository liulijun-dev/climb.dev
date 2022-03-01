---
title: Spring Security源码阅读1 - 核心组件和服务
date: 2020-02-24 15:42:28
tags: Spring Security, 核心组件, 核心服务
---
Spring Security 是一个能够为基于 Spring 的企业应用系统提供声明式的安全访问控制解决方案的安全框架。它提供了一组可以在 Spring 应用上下文中配置的 Bean，充分利用了 Spring IoC（Inversion of Control 控制反转），DI（Dependency Injection 依赖注入）和 AOP（面向切面编程）功能，为应用系统提供声明式的安全访问控制功能，减少了为企业系统安全控制编写大量重复代码的工作。

Spring Security 拥有以下特性：

- 对身份认证和授权的全面且可扩展的支持
- 防御会话固定、点击劫持，跨站请求伪造等攻击
- 支持 Servlet API 集成
- 支持与 Spring Web MVC 集成
- 其他的特性

Spring Security与Spring和Spring Boot的关系如下：

![image-20200224100517215](https://liulijun-dev.github.io/2020/02/24/Spring-Security-Source-Code1_Core-Compoinets-And-Services/springframework_springboot_springsecurity.png)

目前Spring Security提供以下安全技术或支持与现有技术集成：

- In-Memory认证
- JDBC认证
- LDAP认证
- Active Directory认证
- Remember-Me认证
- OpenID
- 匿名认证
- JAAS(Java Authentication and Authorization) Provider
- CAS认证
- X.509认证
- Basic And Digest认证
- OAuth 2.0
- SAML2

接下来先介绍Spring Security的核心组件开始。

# 1. 核心组件 - SecurityContextHolder, SecurityContext and Authentication

最基本的对象是` SecurityContextHolder`，它存储当前应用程序安全上下文的详细信息，其中包括当前使用应用程序的主体（通常是用户）的详细信息。如当前操作的用户是谁，该用户是否已经被认证，他拥有哪些角色权限等。默认情况下，`SecurityContextHolder`使用 `ThreadLocal` 来存储这些详细信息，这意味着 Security Context 始终可用于同一执行线程中的方法，即使 Security Context 未作为这些方法的参数显式传递。考虑到在用户请求被处理后，Spring Security会自动清除线程，因此使用`ThreadLocal` 是线程安全的。

SecurityContextHolder支持三种安全策略：

- `SecurityContextHolder.MODE_THREADLOCAL`: 每个线程有其自己的`SecurityContextHolder`
- `SecurityContextHolder.MODE_INHERITABLETHREADLOCAL`: 继承自安全线程的线程与安全线程有相同的安全标识
- `SecurityContextHolder.MODE_GLOBAL`：所有线程共享相同的`SecurityContextHolder`

可能通过配置`spring.security.strategy`系统属性来设置`SecurityContextHolder`的安全策略，但是大多数应用程序不需要修改`SecurityContextHolder`安全策略。

## 1.1 获取当前用户信息

`SecurityContextHolder`存储了当前与应用程序交互的用户信息，并且用户信息与当前执行线程已绑定。 在Spring Security中使用`Authentication`类代表用户信息，并且可以使用如下代码块在代码的任意处获得当前已验证用户的用户名：

```java
(1)Object principal = SecurityContextHolder.getContext().getAuthentication().getPrincipal();

if (principal instanceof UserDetails) {
String username = ((UserDetails)principal).getUsername();
} else {
String username = principal.toString();
}
```

(1) `getContext()`获得的是`SecurityContext`接口的实例，而该实例存储在`ThreadLocal`中(分析见附录-代码1)，代表当前线程需要的最少的安全信息。`SecurityContext`接口中定义了两个方法，代码如下：

```java
public interface SecurityContext extends Serializable {
   Authentication getAuthentication();
   void setAuthentication(Authentication authentication);
}
```

并且通过跟踪代码可以确`getPrincipal()`返回的是`UserDetails`实例(分析见附录-代码1)。

## 1.2 Authentication

 `SecurityContext.getAuthentication()`返回的`Authentication`也是一个接口，它的定义如下：

```java
// org.springframework.security.core.Authentication.java
public interface Authentication extends Principal, Serializable {
    // 权限信息列表，默认是GrantedAuthority接口的一些实现类，通常是代表权限信息的一系列字符串
    Collection<? extends GrantedAuthority> getAuthorities();
    // 凭证信息以证明主体的正确性，如用户在前端输入的密码
    Object getCredentials();
    // 其他信息，如IP地址，证书序列号等
    Object getDetails();
    // 主体的标识，如用户名，大部分情况下返回的是UserDetails接口的实例
    Object getPrincipal();
    boolean isAuthenticated();
    void setAuthenticated(boolean isAuthenticated) throws IllegalArgumentException;
}
```

 `Authentication` 直接继承自 `Principal` 类，而` Principal `是位于 java.security 包中。**通过`Authentication` 接口的实现类，我们可以得到用户拥有的权限信息列表，密码，用户细节信息，用户身份信息，认证信息等。**

### 1.2.1 UserDetailService

从`Authentication`代码可知，可以通过其中的`getPrincipal()`方法获得安全主体，虽然返回的是Object对象，**但大多数情况下我们可以将其转为`UserDetails`对象**。`UserDetails`是Spring Security的核心类，代表一个安全主体并且是高度可扩展的，代码如下：

```java
public interface UserDetails extends Serializable {
   Collection<? extends GrantedAuthority> getAuthorities();
   String getPassword();
   String getUsername();
   boolean isAccountNonExpired();
   boolean isAccountNonLocked();
   boolean isCredentialsNonExpired();
   boolean isEnabled();
}
```

这里需要注意的是`UserDetails`的`getPassword()`和`Authentication`的 `getCredentials()`的不同：前者是用户正确的密码，后者是用户提交的密码凭证。 

在`SecurityContextHolder`中用`UserDetails`存储安全主体信息，但是在应用程序中我们需要的安全主体信息可能更多（如需要email, empolyeeNumber等），此时我们可以通过继承`UserDetails`接口实现自定义安全主体并存储在`SecurityContextHolder`中，从而在使用时可以将`SecurityContextHolder`中获得的`UserDetails`实例转换为自定义的实例。**因此我们可以将`UserDetails`认为是应用程序和Spring Security框架之间的适配器**。

为了向`SecurityContextHolder`中提供自定义的`UserDetails`，只需要实向Spring容器中注册一个实现了`UserDetailsService`接口的Bean即可，模板代码如下：

```java
@Service
publiic class AuthUserDetailsService implements UserDetailsService {
  @Override  
  (2)public UserDetails loadUserByUsername(userName: String) {
    	//你的逻辑
  }
}
```

（2）只需要在`loadUserByUsername`接口中添加定制的业务逻辑即可

Spring Security也提供了一些`UserDetailsService`的实现，如`InMemoryDaoImpl`)和`JdbcDaoImpl`。**但是不管如何提供`UserDetailsService`的实现，都可以通过`SecurityContextHolder`获得`UserDetailsService`返回的数据**。

### 1.2.2 GrantedAuthority

除了主体，另一个`Authentication`提供的重要方法是`getAuthorities()`。这个方法提供了`GrantedAuthority`对象数组。`GrantedAuthority`是赋予到主体的权限，这些权限通常使用角色表示，比如`ROLE_ADMINISTRATOR`或`ROLE_HR_SUPERVISOR`。这些角色会用于web验证，方法验证和领域对象验证。`GrantedAuthority`对象通常使用`UserDetailsService`读取，即在`loadUserByUsername()`方法中返回的`UserDetails`实例时设置。

## 1.3 总结

上面介绍的Spring Security中使用的核心组件及其功能如下：

- `SecurityContextHolder`：提供几种保存 `SecurityContext`的方式
- `SecurityContext`：保存`Authentication`信息
- `Authentication`：代表Spring Security中的主体
- `GrantedAuthority`：主体的权限
- `UserDetails`：代表主体信息
- `UserDetailsService`：加载`UserDetails`

# 2. 核心服务 - AuthenticationManager, ProviderManager 和 AuthenticationProvider

`AuthenticationManager`接口是认证相关的核心接口，也是认证发起的出发点，在实际需求中，应用可能即允许用户使用用户名 + 密码登录，又允许用户使用邮箱 + 密码，手机号码 + 密码等形式登录，所以要求认证系统要支持多种认证方式，因此需要一个接口定义认证的基本功能。`AuthenticationManager`的定义如下：

```java
public interface AuthenticationManager {
   Authentication authenticate(Authentication authentication)
         throws AuthenticationException;
}
```

Spring Security 中 `AuthenticationManager`接口的默认实现是 **`ProviderManager`**, 其对`Authentication authenticate(Authentication authentication)`方法实现如下：

```java
public Authentication authenticate(Authentication authentication)
      throws AuthenticationException {
   Class<? extends Authentication> toTest = authentication.getClass();
   AuthenticationException lastException = null;
   AuthenticationException parentException = null;
   Authentication result = null;
   Authentication parentResult = null;
   boolean debug = logger.isDebugEnabled();

   (3)for (AuthenticationProvider provider : getProviders()) {
      if (!provider.supports(toTest)) {
         continue;
      }
     
      try {
         result = provider.authenticate(authentication);

         if (result != null) {
            (4)copyDetails(authentication, result);
            break;
         }
      }
      catch (AccountStatusException | InternalAuthenticationServiceException e) {
         prepareException(e, authentication);
         throw e;
      } catch (AuthenticationException e) {
         lastException = e;
      }
   }

   if (result == null && parent != null) {
      // Allow the parent to try.
      try {
         (5)result = parentResult = parent.authenticate(authentication);
      }
      catch (ProviderNotFoundException e) {
      }
      catch (AuthenticationException e) {
         lastException = parentException = e;
      }
   }

   if (result != null) {
      if (eraseCredentialsAfterAuthentication
            && (result instanceof CredentialsContainer)) {
         (6)((CredentialsContainer) result).eraseCredentials();
      }

      // If the parent AuthenticationManager was attempted and successful than it will publish an AuthenticationSuccessEvent
      // This check prevents a duplicate AuthenticationSuccessEvent if the parent AuthenticationManager already published it
      if (parentResult == null) {
         eventPublisher.publishAuthenticationSuccess(result);
      }
      return result;
   }

   if (lastException == null) {
      lastException = new ProviderNotFoundException(messages.getMessage(
            "ProviderManager.providerNotFound",
            new Object[] { toTest.getName() },
            "No AuthenticationProvider found for {0}"));
   }

   if (parentException == null) {
      prepareException(lastException, authentication);
   }

   (7)throw lastException;
}
```

(3) `ProviderManager`本身并不直接处理身份认证请求，而是将认证委托给`AuthenticationProvider`，依次查询每个列表项是否可以执行身份认证。每个 Provider 要么抛出异常要么返回一个完全填充的 `Authentication`对象

(4) 认证成功后，会将原始的认证信息拷贝到Provider的返回结果中

(5) 若当前ProviderManager无法完成认证操作，且其包含父级认证器，则转交给父级认证器尝试进行认证

(6) 完成认证，从authentication对象中删除私密数据，防止一些机密数据（如用户密码）过长时间保留在内存中

(7) 如果认证失败，则抛出`AuthenticationException`

Spring Security提供了很多认证Provider，如：

- `DaoAuthenticationProvider`
- `AnonymousAuthenticationProvider`
- `RememberMeAuthenticationProvider`

所有的Provider都继承自`AuthenticationProvider`接口，代码如下：

```java
public interface AuthenticationProvider {
   // 验证请求
   Authentication authenticate(Authentication authentication)
         throws AuthenticationException;

   // 判断是否支持对authentication的认证
   boolean supports(Class<?> authentication);
}
```

比如在`DaoAuthenticationProvider`中使用`UserDetailsService`根据用户名获得`UserDetails`，再通过比对用户密码判断用户是否合法。

但是需要注意的是，在使用相应的认证机制时，必须为其提供相应的认证Provider，否则会导致认证失败。如JA-SIG CAS认证，其必须使用`CasAuthenticationProvider`。

# 3. 总结

- 应用中可以通过`SecurityContextHolder`获得认证信息`Authentication`
- 应用可以从获得的`Authentication`实例中获得认证后的用户信息，如用户名和权限
- 应用通过继承`UserDetails`接口定制合适的UserDetails（如新增getEmail()函数）
- 应用通过继承`UserDetailsService`接口将系统中存储的用户信息转换为Spring Security的UserDetails实例
- Spring Security中`ProviderManager`为认证的入口
- `ProviderManager`通过`AuthenticationProvider`完成认证

# 参考

1. [Spring Security架构简介](https://segmentfault.com/a/1190000020886438 "Spring Security架构简介")
2. [Spring Security 5.2官方文档](https://docs.spring.io/spring-security/site/docs/5.2.2.RELEASE/reference/htmlsingle/#core-services "Spring security 5.2官方文档")

# 附录

# 代码1

`SecurityContextHolder.getContext()`代码如下：

```java
public static SecurityContext getContext() {
   （1)return strategy.getContext();
}
```

(1) 默认情况下strategy的指向的是ThreadLocalSecurityContextHolderStrategy实例，代码如下：

```java
private static void initialize() {
   if (!StringUtils.hasText(strategyName)) {
      // Set default
      strategyName = MODE_THREADLOCAL;
   }

   if (strategyName.equals(MODE_THREADLOCAL)) {
      (2) strategy = new ThreadLocalSecurityContextHolderStrategy();
   }
   ......
}
```

(2) 可以确定最终调用的是ThreadLocalSecurityContextHolderStrategy实例中的`getContexxt()`方法，代码如下：

```java
public SecurityContext getContext() {
   (3)SecurityContext ctx = contextHolder.get();

   if (ctx == null) {
      ctx = createEmptyContext();
      contextHolder.set(ctx);
   }

   return ctx;
}
```

(3) `contextHolder`的定义如下：

```java
private static final ThreadLocal<SecurityContext> contextHolder = new ThreadLocal<>();
```

因此默认获得的SecurityContext实例是存放在ThreadLocal中的。

