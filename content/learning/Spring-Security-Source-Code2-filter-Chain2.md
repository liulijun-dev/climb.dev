---
title: Spring Security源码阅读3 - Spring Security过滤器链初始化2
date: 2020-02-20 13:04:46
tags: Spring Security, Filter, 过滤器链调用实现原理，过滤器向Servlet容器注册
---
在[Spring Security源码阅读2-Spring Security过滤器链的初始化1](https://liulijun-dev.github.io/2020/02/18/Spring-Security-Source-Code1-filter-chain-init/)文章中，遗留了如下两个问题：

1. 在步骤(15)中，我们说HttpSecurity类中的performBuild函数返回了DefaultSecurityFilterChain类，而该类中封装了该链的所有过滤器，那么这些过滤器是如何添加进来的呢？

2. 在步骤(17)中生成的FilterChainProxy对象也是Filter实例，那么在收到客户端请求后是如何完成过滤器链式调用的呢？

接下来我们就逐步回答上面两个问题。本文先回答问题2，因为回答问题2后，读者就可以从全视角了解Spring Security过滤器的创建和调用流程。本文中会将该问题拆的更细，按由简到繁的顺序来解答。

# 1. FilterChainProxy如何实现过滤器链调用

根据Filter接口的定义，可以确定每次客户端请求在经过过滤器时调用的都是doFilter函数，FilterChainProxy对doFilter函数的实现如下：

```java
@Override
	public void doFilter(ServletRequest request, ServletResponse response,
			FilterChain chain) throws IOException, ServletException {
		boolean clearContext = request.getAttribute(FILTER_APPLIED) == null;
		if (clearContext) {
			try {
				request.setAttribute(FILTER_APPLIED, Boolean.TRUE);
				(1)doFilterInternal(request, response, chain);
			}
			finally {
				SecurityContextHolder.clearContext();
				request.removeAttribute(FILTER_APPLIED);
			}
		}
		else {
			doFilterInternal(request, response, chain);
		}
	}
```

(1) 派发到过滤器链上执行，可以看到doFilter调用的是FilterChainProxy实例的内部函数doFilterInternal，其代码如下：

```
private void doFilterInternal(ServletRequest request, ServletResponse response,
      FilterChain chain) throws IOException, ServletException {

   FirewalledRequest fwRequest = firewall
         .getFirewalledRequest((HttpServletRequest) request);
   HttpServletResponse fwResponse = firewall
         .getFirewalledResponse((HttpServletResponse) response);

   （2)List<Filter> filters = getFilters(fwRequest);

   if (filters == null || filters.size() == 0) {
      if (logger.isDebugEnabled()) {
         logger.debug(UrlUtils.buildRequestUrl(fwRequest)
               + (filters == null ? " has no matching filters"
                     : " has an empty filter list"));
      }

      fwRequest.reset();

      (3)chain.doFilter(fwRequest, fwResponse);

      return;
   }

   (4)VirtualFilterChain vfc = new VirtualFilterChain(fwRequest, chain, filters);
   (5)vfc.doFilter(fwRequest, fwResponse);
}
```

(2) 根据每个过滤器链配置的RequestMathcher，决定每一个请求要经过哪些过滤器。(参考[Spring Security源码阅读2-Spring Security过滤器链的初始化1](https://liulijun-dev.github.io/2020/02/18/Spring-Security-Source-Code1-filter-chain-init/)第(15)步，HttpSecurity在返回performBuild中返回的是DefaultSecurityFilterChain类的实例，而在创造DefaultSecurityFilterChain实例时传递的RequestMatcher实例是AnyRequestMatcher.INSTANCE，其matches函数默认返回true，请参考附录代码一)。

(3) 如果当前过滤器链没有匹配的过滤器，则执行下一条过滤器链。

(4) 将所有的过滤器合并成一个虚拟过滤器链。

(5) 执行虚拟过滤器链。

可以看到最终执行的是虚拟过滤器链VirtualFilterChain类的实例，接下来看看VirtualFilterChain类doFilter的实现：

```java
@Override
		public void doFilter(ServletRequest request, ServletResponse response)
				throws IOException, ServletException {
			if (currentPosition == size) {
				if (logger.isDebugEnabled()) {
					logger.debug(UrlUtils.buildRequestUrl(firewalledRequest)
							+ " reached end of additional filter chain; proceeding with original chain");
				}

				// Deactivate path stripping as we exit the security filter chain
				this.firewalledRequest.reset();

				(6)originalChain.doFilter(request, response);
			}
			else {
				currentPosition++;

				(7)Filter nextFilter = additionalFilters.get(currentPosition - 1);

				if (logger.isDebugEnabled()) {
					logger.debug(UrlUtils.buildRequestUrl(firewalledRequest)
							+ " at position " + currentPosition + " of " + size
							+ " in additional filter chain; firing Filter: '"
							+ nextFilter.getClass().getSimpleName() + "'");
				}

				(8)nextFilter.doFilter(request, response, this);
			}
		}
	}
```

(6) 如果当前虚拟过滤器链上的所有过滤器都已经执行完毕，则执行原生过滤器链上的剩余逻辑。

(7) 获得当前虚拟过滤器链上的下一个过滤器。

(8) 执行过滤器。doFilter的函数原型定义如下：

```java
public void doFilter(ServletRequest request, ServletResponse response,
            FilterChain chain) throws IOException, ServletException;
```

我们可以看到第三个参数传递的是this，即当前虚拟过滤器链实例。因此，当在nextFilter的doFilter函数中再次通过chain参数调用doFilter函数时，则会再次调用到当前虚拟过滤器链实例（伪代码如下)，从而完成虚拟过滤器链上的所有过滤器的调用 。

```java
public void doFilter(ServletRequest request, ServletResponse response,
FilterChain chain) throws IOException, ServletException {
   ......
   chain.doFilter(request, response, chain); //这里的chain是传进来的VirtualFilterChain实例
   ......
}
```

以上就是FilterChainProxy实现过滤器链调用的全过程。

接下来讨论一下客户端请求是如何传递到过滤器的，但是要了解请求传递到过滤器的过程，就必须得清楚过滤器在Servlet容器中注册的是什么。

# 2. Spring Security过滤器如何注册到Servlet容器

这里先说明一下，为什么本节的标题用的是过滤器而非过滤器链，因为本文第一节已经分析了FilterChainProxy实现过滤器链调用的原理，而FilterChainProxy本质上也是一个Filter实例。

在IDE中启动Spring应用默认使用的是Spring Boot内嵌的Tomcat容器，因此这里讨论的Servlet容器指的是Tomcat的Servlet容器。在启动Spring应用时会向IoC注册securityFilterChainRegistration Bean：

```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnWebApplication(type = Type.SERVLET)
@EnableConfigurationProperties(SecurityProperties.class)
@ConditionalOnClass({ AbstractSecurityWebApplicationInitializer.class, SessionCreationPolicy.class })
@AutoConfigureAfter(SecurityAutoConfiguration.class)
public class SecurityFilterAutoConfiguration {

	private static final String DEFAULT_FILTER_NAME = AbstractSecurityWebApplicationInitializer.DEFAULT_FILTER_NAME;

	@Bean
	@ConditionalOnBean(name = DEFAULT_FILTER_NAME)
	public DelegatingFilterProxyRegistrationBean securityFilterChainRegistration(
			SecurityProperties securityProperties) {
		(9)DelegatingFilterProxyRegistrationBean registration = new DelegatingFilterProxyRegistrationBean(
				DEFAULT_FILTER_NAME);
		registration.setOrder(securityProperties.getFilter().getOrder());
		registration.setDispatcherTypes(getDispatcherTypes(securityProperties));
		return registration;
	}
	......
}
```

(9) DelegatingFilterProxyRegistrationBean实现了ServletContextInitializer接口，该接口用于配置Servlet上下文。DelegatingFilterProxyRegistrationBean目的是向Servlet容器中注册一个过滤器：实现类为 DelegatingFilterProxy 的一个 Servlet Filter。DelegatingFilterProxy  其实是一个代理过滤器，它被 Servlet 容器用于匹配特定URL模式的请求，而它会将任务委托给Spring管理的Bean，即名字为 springSecurityFilterChain 的Bean，而springSecurityFilterChain正是[Spring Security源码阅读2-Spring Security过滤器链的初始化1](https://liulijun-dev.github.io/2020/02/18/Spring-Security-Source-Code1-filter-chain-init/)中介绍的，<font color='red'>从而实现了Servlet容器管理的DelegatingFilterProxy与Spring容器创建的springSecurityFilterChain Bean的关联</font>。关于这一段结论的代码分析，请参考附录代码二。

为了加深对DelegatingFilterProxy的理解，我们可以看下其注释：

```java
/*Proxy for a standard Servlet Filter, delegating to a Spring-managed bean that
 * implements the Filter interface. Supports a "targetBeanName" filter init-param
 * in {@code web.xml}, specifying the name of the target bean in the Spring
 * application context.
 */
```

清楚了Servlet容器管理的DelegatingFilterProxy与Spring容器创建的springSecurityFilterChain Bean的关联，接下来就可以分析客户端请求如何传递到过滤器。

# 3. 客户端请求如何传递到过滤器

当用户请求到来并且与过滤器的URL模式匹配后，会调用DelegatingFilterProxy的doFilter函数：

```java
  @Override
	public void doFilter(ServletRequest request, ServletResponse response, FilterChain filterChain)
			throws ServletException, IOException {

		// Lazily initialize the delegate if necessary.
		Filter delegateToUse = this.delegate;
		if (delegateToUse == null) {
			synchronized (this.delegateMonitor) {
				delegateToUse = this.delegate;
				if (delegateToUse == null) {
					WebApplicationContext wac = findWebApplicationContext();
					if (wac == null) {
						throw new IllegalStateException("No WebApplicationContext found: " +
								"no ContextLoaderListener or DispatcherServlet registered?");
					}
					(10)delegateToUse = initDelegate(wac);
				}
				this.delegate = delegateToUse;
			}
		}

		// Let the delegate perform the actual doFilter operation.
		(11)invokeDelegate(delegateToUse, request, response, filterChain);
	}
```

(10) 获得Spring容器管理的springSecurityFilterChain Bean.

(11) 调用FilterChainProxy的doFilter函数，代码如下：

```java
protected void invokeDelegate(
			Filter delegate, ServletRequest request, ServletResponse response, FilterChain filterChain)
			throws ServletException, IOException {

		delegate.doFilter(request, response, filterChain);
	}
```



以上就是Spring Security过滤器的调用和执行流程。在整个分析中，为了分析的简单和文章不至于太长，我并没有详细分析Servlet容器和Spring IoC容器的交互过程，有兴趣的读者可以自行分析。

# 附录

## 1. 代码一

HttpSecurity.java

```java
	private RequestMatcher requestMatcher = AnyRequestMatcher.INSTANCE;
 
  @Override
	protected DefaultSecurityFilterChain performBuild() {
		filters.sort(comparator);
		return new DefaultSecurityFilterChain(requestMatcher, filters);
	}
```

## 2. 代码二

我们直接从ServletWebServerApplicationContext类开始分析，因为在Servlet类型应用中，实际实例化的应用上下文为ServletWebServerApplicationContext。为什么会如此，读者可以分析一下Spring Boot的启动流程。

```java
   /**
	 * Returns the {@link ServletContextInitializer} that will be used to complete the
	 * setup of this {@link WebApplicationContext}.
	 * @return the self initializer
	 * @see #prepareWebApplicationContext(ServletContext)
	 */
	private org.springframework.boot.web.servlet.ServletContextInitializer getSelfInitializer() {
		return this::selfInitialize;
	}

	private void selfInitialize(ServletContext servletContext) throws ServletException {
		prepareWebApplicationContext(servletContext);
		registerApplicationScope(servletContext);
		WebApplicationContextUtils.registerEnvironmentBeans(getBeanFactory(), servletContext);
		(1)for (ServletContextInitializer beans : getServletContextInitializerBeans()) {
			beans.onStartup(servletContext);
		}
	}
```

(1) 回调所有ServletContextInitializer的onStart函数。通过调试，ServletWebServerApplicationContext中有如下这些ServletContextInitializer：

![image-20200220112615616](https://liulijun-dev.github.io/2020/02/20/Spring-Security-Source-Code2-filter-Chain2/ServletContextInitializer.png)

可以看到DelegatingFilterProxyRegistrationBean类的实例，再看看其onStart回调函数，其最终回调的是addRegistration函数(DelegatingFilterProxyRegistrationBean继承自AbstractFilterRegistrationBean，该函数位于其中)：

```java
  @Override
	protected Dynamic addRegistration(String description, ServletContext servletContext) {
		Filter filter = getFilter();
		(2)return servletContext.addFilter(getOrDeduceName(filter), filter);
	}
```

(2) 通过调用DelegatingFilterProxyRegistrationBean中的getFilter函数获得DelegatingFilterProxy类的实例，并将基添加到Servlet上下文中，最终将过滤器添加到StandardContext（这个就是Tomcat的上下文了，从而建立了Tomcat容器与Spring的关系）的filterDefs属性中。
