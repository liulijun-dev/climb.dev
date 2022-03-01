---
title: 结构化代码-分层封装与按特性封装
date: 2020-03-29 12:05:14
tags: 软件架构, 分层封装, 按特性(domain)封装
---
# 1. 分层封装

在大多数应用中我们都是将代码分层封装（如Clean架构），如下：

```java
com.awesome.project
    .common
        StringUtils
        ExceptionUtils
    .controllers
        LocationController
        PricingController
    .domain
        Address
        Cost
        CostFactory
        Location
        Price
    .repositories
        LocationRepository
        PriceRepository
    .services
        LocationService
```

分层封装比较流行的原因可能是开发人员能够很容易发现功能相似的代码，同时也符合人们“习惯于对事物进行分类”的思想，但是分层封装会带来如下问题：
- 添加或修改业务时需要跨多层修改代码
- 在某一层中封装的代码通常是不相关的（如LocationRepository和PriceRepository)
- 不同的开发人员对不同的代码层应该包含的功能有不同的认知，因此需要时间统一大家的认知，对新人需要进行统一的培训
- 把某一个domain提取成单独的微服务并不容易
- 更多的分层意味着更高的复杂性
- 开发人员需要时间来思考代码应该放到哪个分层
- 随着业务越来越复杂和对架构缺少守护，分层架构会越来越腐化，远远偏离最初的架构设计
# 2. 按特性(或domain)封装

考虑到分层分装的问题，我们可以按特性或domain对代码进行封装，比如：
```java
com.awesome.project.component
    .location
        Address
        Location
        LocationController
        LocationRepository
        LocationService
    .platform
        StringUtils
        ExceptionUtils
    .price
        Cost
        CostFactory
        Distance
        Price
        PriceController
        PriceRepository
```

这种结构化代码的方更符合OO原则，可以做到业务高内聚，不仅能够解决分层封装代码的问题，同时具有如下优势：

- 开发人员更聚焦于业务，而不至于花时间如何组织我们的代码
- 从DDD的角度出发，开发人员可以很容易的找到某一个domain下的聚合根，比如price包中的Price类肯定是聚合根，因为我们可以通过PriceRepository直接获得Price，而Cost很可能被Price使用，因为我们无法直接获得它
使用这种封装方式要解决的首要问题是：**应用业务规则往往是由多个领域的业务规则组合完成的，在哪里完成这些领域业务规则的组合呢？**不同的开发人员可能有不同的解决方法，比如将Controller提出到单独一个层：
```java
com.awesome.project
    .controller
        LocationController
        PriceController
    .component
       .location
           Address
           Location
           LocationRepository
           LocationService
       .platform
           StringUtils
           ExceptionUtils
       .price
           Cost
           CostFactory
           Distance
           Price
           PriceRepository
```
