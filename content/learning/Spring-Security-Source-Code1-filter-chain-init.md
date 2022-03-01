---
title: Spring Security源码阅读2-Spring Security过滤器链初始化1
date: 2020-02-18 20:00:33
tags: Spring Security, 过滤器, filter
---
Spring Security的核心实现是通过一条过滤器链来确定用户的每一个请求应该得到什么样的反馈。

# 1. 使用@EnableWebSecurity注解开启Spring Security

在使用Spring Security时首先要通过@EnableWebSecurity注解开启Spring Security的默认行为。

```java
@Retention(value = java.lang.annotation.RetentionPolicy.RUNTIME)
@Target(value = { java.lang.annotation.ElementType.TYPE})
@Documented
@Import({ WebSecurityConfiguration.class,
   SpringWebMvcImportSelector.class,
   OAuth2ImportSelector.class })
@EnableGlobalAuthentication
@Configuration
public @interface EnableWebSecurity {
    boolean debug() default false;
}
```

该注解通过@Import注解将WebSecurityConfiguration类导入到Spring 的IoC容器，从而对Spring Security进行初始化。同时，@EnableWebSecurity可以通过配置debug = true开启调试模式，能够打印出Spring Security运行时的详细信息，如下：

![img](https://liulijun-dev.github.io/2020/02/18/Spring-Security-Source-Code1-filter-chain-init/enablewesecurity_debug_output.jpg) 

接下来，我们看看上图中的过滤器是如何加载到过滤器链中的。

# 2. WebSecurityConfiguration

首先看WebSecurityConfiguration类中的setFilterChainProxySecurityConfigurer函数，该函数用来初始化SecurityConfigurer列表：

```java
@Autowired(required = false)
public void setFilterChainProxySecurityConfigurer(
   ObjectPostProcessor<Object> objectPostProcessor,
  （1）@Value("#{@autowiredWebSecurityConfigurersIgnoreParents.getWebSecurityConfigurers()}") List<SecurityConfigurer<Filter, WebSecurity>> webSecurityConfigurers)
   throws Exception {
  (2)webSecurity = objectPostProcessor
     .postProcess(new WebSecurity(objectPostProcessor));
  if (debugEnabled != null) {
   webSecurity.debug(debugEnabled);
  }

  (3)webSecurityConfigurers.sort(AnnotationAwareOrderComparator.INSTANCE);

  Integer previousOrder = null;
  Object previousConfig = null;
  （4）for (SecurityConfigurer<Filter, WebSecurity> config : webSecurityConfigurers) {
   Integer order = AnnotationAwareOrderComparator.lookupOrder**(config);
   if (previousOrder != null && previousOrder.equals(order)) {
     throw new IllegalStateException(
        "@Order on WebSecurityConfigurers must be unique. Order of "
           \+ order + " was already used on " + previousConfig + ", so it cannot be used on "
           \+ config + " too.");
   }
   previousOrder = order;
   previousConfig = config;
  }
  （5）for (SecurityConfigurer<Filter, WebSecurity> webSecurityConfigurer : webSecurityConfigurers) {
   webSecurity.apply(webSecurityConfigurer);
  }
  this.webSecurityConfigurers = webSecurityConfigurers;
}
```

(1）在传入的参数中，objectPostProcess用于初始化对象，暂时不用关注。webSecurityConfigurers是通过SpEL调用Bean的方法获得的值，其获得的是我们在配置Spring Security时继承自WebSecurityConfigurerAdapter的配置类（具体代码见：AutowiredWebSecurityConfigurersIgnoreParents类的getWebSecurityConfigurers方法）。

(2）初始化WebSecurity。

(3）对webSecurityConfigurers按升序进行排序（排序算法是稳定的），如果一个应用中有多个SecurityConfigurer，可通过@Order注解指定其顺序，注解中的值越大，SecurityConfigurer排序后越靠后。继承自WebSecurityConfigurerAdapter的配置类其@Order注解的默认值为100。该步骤的作用是为第（4）步检查重复的@Order注解值做准备。

(4）检查多个SecurityConfigurer配置的@Order注解值是否相同，如果相同则报错。这就说明如果代码中通过继承自WebSecurityConfigurerAdapter配置了多个SecurityConfigurer，则必须为每个SecurityConfigurer设置@Order注解，并且注解值不能相同。

(5）将配置的每一个SecurityConfigurer应用到WebSecurity。这里我们顺便看下WebSecurity类的apply函数(WebSecurity继承自AbstractConfiguredSecurityBuilder，apply函数位于AbstractConfiguredSecurityBuilder中。)，因为后面的代码分析会用到。

```java
public <C extends SecurityConfigurer<O, B>> C apply(C configurer) throws Exception {
  add(configurer);
  return configurer;
}

private <C extends SecurityConfigurer<O, B>> void add(C configurer) {
  Assert.notNull(configurer, "configurer cannot be null");

  Class<? extends SecurityConfigurer<O, B>> clazz = (Class<? extends SecurityConfigurer<O, B>>) configurer
     .getClass();
  synchronized (configurers) {
   if (buildState.isConfigured()) {
     throw new IllegalStateException("Cannot apply " + configurer
        \+ " to already built object");
   }
   List<SecurityConfigurer<O, B>> configs = allowConfigurersOfSameType ? this.configurers
      .get(clazz) : null;
   if (configs == null) {
     configs = new ArrayList<>(1);
   }
   configs.add(configurer);
   （6）this.configurers.put(clazz, configs);
   if (buildState.isInitializing()) {
     this.configurersAddedInInitializing.add(configurer);
   }
  }
}
```

可以看到apply函数主要调用add函数，并在add函数中将SecurityConfigurer添加到WebSecurity类的configures属性，键值为clazz（第（6）步）。注意此时WebSecurity中的buildState的状态为UNBUILT。

回到WebSecurityConfiguration中，我们再看其另一个函数：springSecurityFilterChain，该函数返回Filter，用于创建Spring Security过滤器链，代码如下：

```java
@Bean(name = AbstractSecurityWebApplicationInitializer.DEFAULT_FILTER_NAME)
public Filter springSecurityFilterChain() throws Exception {
  boolean hasConfigurers = webSecurityConfigurers != null
     && !webSecurityConfigurers.isEmpty();
  (7)if (!hasConfigurers) {
   WebSecurityConfigurerAdapter adapter = objectObjectPostProcessor
      .postProcess(new WebSecurityConfigurerAdapter() {
      });
   webSecurity.apply(adapter);
  }
  (8)return webSecurity.build();
}
```

(7) 如果没有通过继承WebSecurityConfigurerAdatper类配置过Spring Security，则会以WebSecurityConfigurerAdatper中的配置默认行为。

(8) 调用WebSecurity类的build方法构建过滤器链。

接下来将讲到最核心的过滤器链的构建，首先看下WebSecurity类中的build函数（WebSecurity继承自AbstractConfiguredSecurityBuilder类，而后者又继承自AbstractSecurityBuilder类，build函数位于AbstractSecurityBuilder类中），代码为：

```java
public final O build() throws Exception {
  if (this.building.compareAndSet(false, true)) {
   （9）this.object = doBuild();
   return this.object;
  }
  throw new AlreadyBuiltException("This object has already been built");
}
```

可以看到最终调用的是doBuild函数（位于AbstractConfiguredSecurityBuilder类中），代码如下：

```java
@Override
protected final O doBuild() throws Exception {
  synchronized (configurers) {
   buildState = BuildState.INITIALIZING;

   (10)beforeInit();
   (11)init();

   buildState = BuildState.CONFIGURING;

   (12)beforeConfigure();
   (13)configure();

   buildState = BuildState.BUILDING;

   (14)O result = performBuild();

   buildState = BuildState.BUILT;

   return result;
  }
}
```

(10) 默认该函数为空。

(11) 初始化SecurityConfigurer，初始化的SecurityConfigurer为前面讲的setFilterChainProxySecurityConfigurer函数调用apply函数时设置的，最终调用的是WebSecurityConfigurerAdapter的init函数，将所有的HttpSecurity添加到WebSecurity中。

(12) 默认该函数为空。

(13) 调用WebSecurityConfigurerAdapter中的configure(WebSecurity web)函数，默认为空。

(14) 完成过滤器链的构建。

从上面的步骤可以看出，只有第（11）和第（14）步才真正做了操作，而第（14）步才是真正构建过滤器链的操作。因此，接下来将看performBuild函数（位于WebSecurity类中），代码如下：

```java
@Override
protected Filter performBuild() throws Exception {
  Assert.state(
     !securityFilterChainBuilders.isEmpty(),
     () -> "At least one SecurityBuilder<? extends SecurityFilterChain> needs to be specified. "
        \+ "Typically this done by adding a @Configuration that extends WebSecurityConfigurerAdapter. "
        \+ "More advanced users can invoke "
        \+ WebSecurity.class.getSimpleName()
        \+ ".addSecurityFilterChainBuilder directly");
  int chainSize = ignoredRequests.size() + securityFilterChainBuilders.size();
  List<SecurityFilterChain> securityFilterChains = new ArrayList<>(
     chainSize);
  for (RequestMatcher ignoredRequest : ignoredRequests) {
   securityFilterChains.add(new DefaultSecurityFilterChain(ignoredRequest));
  }
  （15）for (SecurityBuilder<? extends SecurityFilterChain> securityFilterChainBuilder : securityFilterChainBuilders) {
   securityFilterChains.add(securityFilterChainBuilder.build());
  }
  （16）FilterChainProxy filterChainProxy = new FilterChainProxy(securityFilterChains);
  if (httpFirewall != null) {
   filterChainProxy.setFirewall(httpFirewall);
  }
  filterChainProxy.afterPropertiesSet();

  Filter result = filterChainProxy;
  if (debugEnabled) {
   logger.warn("\n\n"
      \+ "********************************************************************\n"
      \+ "**********     Security debugging is enabled.    *************\n"
      \+ "**********   This may include sensitive information.  *************\n"
      \+ "**********    Do not use in a production system!   *************\n"
      \+ "********************************************************************\n\n");
   result = new DebugFilter(filterChainProxy);
  }
  postBuildAction.run();
  (17)return result;
}
```

(15) 为每一个securityFilterChainBuilder生成过滤器链，securityFilterChainBuilders集合中的内容是我们在第（11）步中设置的HttpSecurity。这里请注意一点，即WebSecurity和HttpSecurity有相同的继承结构，如下：

![img](https://liulijun-dev.github.io/2020/02/18/Spring-Security-Source-Code1-filter-chain-init/uml_for_websecurity_and_httpsecurity.jpg) 

因此，参考步骤（10）~（14），可以确定最终调用的是HttpSecurity下的performBuild函数，如下：

```java
@Override
protected DefaultSecurityFilterChain performBuild() {
  filters.sort(comparator);
  return new DefaultSecurityFilterChain(requestMatcher, filters);
}
```

可以看到最终返回的是DefaultSecurityFilterChain类，该类中包含了过滤器链中的所有过滤器。

(16) 将生成的过滤器链securityFilterChains由FilterChainProxy来代理，FilterChainProxy间接实现了Filter接口。

(17) 将FilterChainProxy对象返回

以上就是是过滤器链的生成过成，目前遗留了两个问题在接下来的文章中分析：

1. 在步骤(15)中，我们说HttpSecurity类中的performBuild函数返回了DefaultSecurityFilterChain类，而该类中封装了该链的所有过滤器，那么这些过滤器是如何添加进来的呢？

2. 在步骤(17)中生成的FilterChainProxy对象也是Filter实例，那么在收到客户端请求后是如何完成过滤器链式调用的呢？
