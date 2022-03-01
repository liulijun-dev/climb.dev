---
title: CAP定理及其证明
date: 2020-04-15 08:40:34
tags: 分布式系统, CAP
---

CAP定理是分布式系统中的一个基本定理，它指出任何分布式系统最多可以具有以下三个特性中的两个：

- **C**onsistency（一致性）
- **A**vailability（可用性）
- **P**artition tolerance（分区容错）

# 1. 分布式系统

考虑一个非常简单的分布式系统。系统由两台服务器组成G<sub>1</sub>和G<sub>2</sub>，这两台服务器跟踪相同的变量v，v的初始值为v<sub>0</sub>。G<sub>1</sub>和G<sub>2</sub>相互间可以通信，同时也与第三方客户端通信。系统结构如下：



![system_architecture](https://liulijun-dev.github.io/2020/04/15/CAP-Theorem-and-Its-Proof/system_architecture.svg)

客户端可以向任何服务器发起读写请求。当一个服务器收到请求后，它执行计算并向客户端返回响应。请求写的过程如下：

![write_flow](https://liulijun-dev.github.io/2020/04/15/CAP-Theorem-and-Its-Proof/write_flow.png)

请求读的过程如下：

![read_flow](https://liulijun-dev.github.io/2020/04/15/CAP-Theorem-and-Its-Proof/read_flow.png)

# 2. 一致性



一致性的意思是

> 任何写操作之后的读操作，必须返回该值

在一致性系统中，一旦一个客户端成功的向任何服务器写入值后，它期望能够从任何一个服务器获得该值（或最新的值）。

如下图是一个非一致性系统的例子，客户端向G<sub>1</sub>成功写入v<sub>1</sub>，但当客户端从G<sub>2</sub>读取v的值时，其获得的结果是v<sub>0</sub>。

![inconsistent_system](https://liulijun-dev.github.io/2020/04/15/CAP-Theorem-and-Its-Proof/inconsistent_system.png)

一个一致性系统如下图所示，G<sub>1</sub>会先将v值复制给G<sub>2</sub>，再向客户端响应写入结果，当客户端从G<sub>2</sub>读取值时，其获得的是最新的值v<sub>1</sub>。

![consistent_system](https://liulijun-dev.github.io/2020/04/15/CAP-Theorem-and-Its-Proof/consistent_system.png)

# 3. 可用性

可用性的意思是

> 系统中的非故障节点必须为接收到的每个请求产生一个响应

在一个可用系统中，如果客户端向任何一个服务器发送请求，只要服务器没有崩溃，服务器最终必须产生一个响应，而不能忽略该请求。

如用户可以选择向 G<sub>1</sub> 或 G<sub>2</sub> 发起读操作，不管是哪台服务器，只要收到请求，就必须告诉用户，到底是 v<sub>0</sub> 还是 v<sub>1</sub>，否则就不满足可用性。

# 4. 分区容错

分区容错的意思是

> 网络允许丢弃节点间传递的任意多个消息（the network will be allowed to lose arbitrarily many messages sent from one node to another）

大多数分布式系统都分布在多个子网络，每个子网络就叫做一个区（partition）。分区容错的意思是，区间通信可能失败。比如，一台服务器放在中国，另一台服务器放在美国，这就是两个区，它们之间可能无法通信。如果所有的通信都被丢弃，系统如下图所示。

![partition_tolerance](https://liulijun-dev.github.io/2020/04/15/CAP-Theorem-and-Its-Proof/partition_tolerance.svg)

在一个支持分区容错的系统中，我们的系统必须能够在任意网络分区的情况下正常工作。

# 5.证明

接下来证明一个系统不能同时满足这三个特性。

假设存在一个系统同时具有一致性、可用性和分区容错性。首先对系统进行划分，结果如下：

![partition_tolerance](/https://liulijun-dev.github.io/2020/04/15/CAP-Theorem-and-Its-Proof/partition_tolerance.svg)

接下来，客户端向G<sub>2</sub>服务器请求写v<sub>1</sub>，因为系统是可用的，故G<sub>2</sub>会返回响应。但是因为网络被隔离，G<sub>2</sub>无法向G<sub>1</sub>同步更新v<sub>1</sub>。

![proof_step_2](https://liulijun-dev.github.io/2020/04/15/CAP-Theorem-and-Its-Proof/proof_step_2.png)

最后，客户端会向G<sub>1</sub>和G<sub>2</sub>分别请求v的值，因为系统是可用的，G<sub>1</sub>和G<sub>2</sub>会分别返回v<sub>0</sub>和v<sub>1</sub>，导致了**不一致**。

![proof_step_3](https://liulijun-dev.github.io/2020/04/15/CAP-Theorem-and-Its-Proof/proof_step_3.png)



因为我们假设存在一个系统具有一致性、可用性和分区容错性，但是我们证明了对于任何这样的系统都存在一种情况导致系统的不一致性。因此，不存在一个同时满足这三个特性的系统。

# 6. 一致性和可用性间的矛盾

一致性和可用性，为什么不可能同时成立？从上述的证明可以看到因为通信可能会失败（即出现分区容错）。

如果保证 G<sub>2</sub>的一致性，那么 G<sub>1</sub>必须在写操作时，锁定 G<sub>2</sub> 的读操作和写操作。只有数据同步后，才能重新开放 G<sub>2</sub>读写。锁定期间，G2 不能读写，**没有可用性**。

如果保证  G<sub>2</sub> 的可用性，那么势必不能锁定 G<sub>2</sub>，所以**一致性不成立**。

综上所述， G<sub>2</sub> 无法同时做到一致性和可用性。系统设计时只能选择一个目标。如果追求一致性，那么无法保证所有节点的可用性；如果追求所有节点的可用性，那就没法做到一致性。

如果一个系统同时支持可用性和一致性，一般这个系统是非分布式系统。

举例来说，发布一张网页到 CDN，多个服务器有这张网页的副本。后来发现一个错误，需要更新网页，这时只能每个服务器都更新一遍。一般来说，网页的更新不是特别强调一致性。短时期内，一些用户拿到老版本，另一些用户拿到新版本，问题不会特别大。当然，所有人最终都会看到新版本。所以，这个场合就是可用性高于一致性。

![three_ indicators](https://liulijun-dev.github.io/2020/04/15/CAP-Theorem-and-Its-Proof/three_indicators.png)



# 参考

1. https://mwhittaker.github.io/blog/an_illustrated_proof_of_the_cap_theorem/
2. http://www.ruanyifeng.com/blog/2018/07/cap.html

