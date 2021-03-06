---
title: 分层软件架构及其数据解耦
date: 2020-02-11 15:24:58
tags: 软件架构,分层，数据解耦
author: 刘利军
---
#  1. 分层软件架构

分层架构是软件的软件中最常用的架构设计方法，如clean架构、MVP架构等。

![Clean架构](https://liulijun-dev.github.io/2020/02/11/Layered-architechture-and-its-data-decoupling/clean-architecture.png)
![MVP架构](https://liulijun-dev.github.io/2020/02/11/Layered-architechture-and-its-data-decoupling/mvp-architecture.png)

分层的实质是隔离关注点，使得每一层具有一致的行为，这样不同的开发才有可能关注不同的软件层。如WEB开发中常用的前后端分离，前端关注的是用户体验，后端关注的是稳定可靠的服务。再比如DDD中主张将领域和应用进行分离，从而能够获得一个比较稳定的领域能力层。

解耦的本质是分离变化点，将不同的变化点分离到不同的层次或模块中，使得其职责更单一，从而有利于软件的开发、维护和扩展。

因为用户对其完成某一系列业务case的完整性并没有随着解耦而消失，因此在分层的软件架构中除了分和解，还要有合。通过聚合层、各类中间件sdk等来完成用户的具体业务case。

分、解、合中，分和解是软件研发组织内部的诉求，不是用户诉求，主要完成用户界面、业务逻辑和数据存储几大类任务的分层和解耦。合是基于分和解的结果，实际是其难度更大，否则会造成合的结果耦合过重导致难于维护。因此，分和解时要考虑到合，只有同时考虑到分、解、合的架构才是一个完整的架构。

# 2. 各软件层间的数据解耦和转换

采用分层的软件架构后，在各个软件层上都要有自己的数据模型，但由于用户业务的完整性，各个软件层的数据又需要相互转换，从而完成软件的“分”与“合”。各个软件层的数据模型一般要满足如下约束：

- 每个软件层只能使用自己的数据模型

- 软件层间的数据模型通过转换器相互转换

分层架构中常用的数据模型是VO、DTO、 DO和PO，解析如下：

- VO（View Object）：视图对象，用于表示层，它的作用是封装页面（或组件）的数据

- DTO（Data Transfer Object）：数据传输对象，用于表示层与服务层之间的数据传输对象

- DO（Domain Object）：领域对象，从现实世界中抽象出来的有形或无形的业务实体

- PO（Persistent Object）：持久化对象，它跟持久层（通常是数据库）的数据结构形成一一对应的映射关系，如果持久层是关系型数据库，那么，数据表中的每个字段（或若干个）就对应PO的一个（或若干个）属性

其中，各个数据模型的转换关系如下：
![VO DTO DO PO转换关系](https://liulijun-dev.github.io/2020/02/11/Layered-architechture-and-its-data-decoupling/vo-dto-do-po.png)

- 用户发出请求，其请求中的数据在UI层表示为VO

- UI层把VO转换为业务层对应方法所要求的DTO并传送给服务层（在WEB开发中，DTO为API接收到的参数）

- 业务层首先根据DTO的数据构造（或重建）一个DO，调用DO的业务方法完成具体业务

- 业务层把DO转换为存储层对应的PO，调用相应的存储层的持久化方法，把PO传递给它，完成存储操作
- 对于逆向操作，如读取，也采取类似的方法进行转换和传递
