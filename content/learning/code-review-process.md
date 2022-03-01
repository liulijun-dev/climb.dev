---
title: Code Review之前中后
date: 2020-02-21 18:55:58
tags: 能力建设, code review
---
# 前言
一般我们在做Code Review时希望实现以下几个目的：

- 传播好的编码实践
- 让大家对所做软件更了解
- 避免重复采坑
- 发现方案或代码中功能性和非功能性问题，以及未考虑到的场景

很多团队在做Code Review时只是走了个形式，而没有实现真正目的。这篇博客会给大家介绍Code Review的前中后分别各做什么，从而使得Code Review形成一个闭环过程。

# 一般的代码合入流程
目前大多数公司都使用git来管理代码，一般开发会通过gerrit或pull request（记为PR）来合入代码，因此一个一般的代码提交流程如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/2019062821451838.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2w0NjAxMzM5MjE=,size_16,color_FFFFFF,t_70)
# Code Review之前
在做Code Review之前，至少要完成：
- 开发需要根据需求说明书或验收条件或问题复现方式来完成自检
- 代码通过扫描
- 单元测试通过

我们知道对于修改量比较大的代码，Code Review还是比较占用时间的，因此Code Review不会检查诸如代码格式、if条件是否需要合并、基本功能等这些基本的问题。这些问题需要在Code Review之前由开发、自动化工具和单元测试来保证，比如Lint、Coverity、Findbug、PMD能够帮助我们发现很多潜在的bug、兼容性问题和代码规范问题，我们应该充分利用这些工具。
# Code Review之中
开始Code Review时，开发人员需要确认Reviewers是否了解代码针对的需求或问题，如果不了解可以给Reviewers简单介绍一下，然后<font color=red>按照真实业务的流程来介绍代码</font>，重点介绍可能存在问题的地方，Reviewers在听的时候可以随时提问，确认是问题或需要再确认的需要<font color=red>及时在代码中标注出来</font>。Reviews可以根据如下清单来Review代码：

- 代码是否很容易理解
- 代码中是否有重复的地方
- 代码是否容易测试和调试
- 代码是否考虑到并发
- 代码是否考虑到了非功能性需求，如性能、内存
- 代码是否符合现有的软件架构
- 代码是否符合单一职责原则（SRP)
- 代码是否符合开闭原则（OCP)
- 代码是否符合依赖倒置原则（OIP)

虽然具体清单项可能不同项目会有不同，但上述这些Review项一般是必须的。

# Code Review之后
代码经过Code Review之后，开发人员需要针对Review过程中提出的问题进行分析和修改，修改完成后再提交新的代码，再走一遍上述流程。一般经过一次Review的代码在第二次Review时会特别快。

在这里有一点往往是团队忽略的，如果在Review时经常发现一些同类型的问题，这时就该考虑是不是团队内部对相关知识缺乏了解，是不是应该对团队进行一次相关的培训。

另外，如果我们经常Review出某一类问题，也可以考虑通过配置工具来自动扫描出相关问题。
# 参考
1. Code Review Checklist – To Perform Effective Code Reviews. https://www.evoketechnologies.com/blog/code-review-checklist-perform-effective-code-reviews/

