---
layout: post
title:  "DDIA学习笔记(1)之系统概述"
categories: DDIA
tags: DDIA
author: ShuxiaoW
---

* content
{:toc}

最近开始学习DDIA(Designing Data-Intensive Applications)，这里记录下每一章节的学习心得，以作备忘。

DDIA应该可以作为所有互联网后台开发工程师的必读书目。这本书从系统整体描述，再到组成系统的各个具体主题，基本涵盖了如何开发一个健壮的、可扩展的、易维护的数据处理系统所涉及的方方面面。

## 1 什么是Data-Intensive应用

Data-Intensive是和Computing-Intensive相对的概念：一个偏重如何更好地为业务或用户提供数据服务（读、写、流处理等）；另一个则偏重考虑如何针对特定的任务提高计算力。

在互联网行业，特别是后台开发这块儿，几乎所有的系统都属于Data-Intensive：

- 数据库。提供数据的存储和查询（磁盘）。
- 缓存。提高数据的读效率，包括速度和吞吐量。
- 搜索引擎。
- 流处理系统。实现数据在上下游异构系统间的流动。
- 批处理系统。实现数据分析功能。

## 2 如何定义“好的”系统

主要有3个方面：

- 可靠性。
- 可扩展性。
- 可维护性。

### 可靠性

主要表现在以下方面：

1. 对于合法的输入，总是能够按照预期输出结果。
2. 对于不合法的输入，仍然能够正常运行。
3. 提供一定的访问认证机制，抵抗非法用户。
4. 在不超过系统负载的情况下，总是能够稳定的提供业务预期的服务。例如，对于数据查询，其响应时间应该稳定在可接受范围内。

这里主要总结下，可靠的系统应该能够抵御哪些故障，做到`fault-tolerant`：

- 硬件故障。具有随机性、不可避免性、相互独立性等特点。
- 软件故障。某个边界条件触发，可能是全局性的；可能会吃光机器资源，影响同机器其它服务；可能引起连锁效应和雪崩效应。
- 人为故障。人总是会犯错的，例如配置错误、操作失误等等。

对于硬件故障，我们没法提前预知，但是可以在系统设计时就考虑在某台机器故障时仍然有其它机器可提供服务，达到系统可以允许多少台机器同时出现故障。

对于软件故障，只能做到：1）开发时充分考虑各种边界条件；2）测试时覆盖各种边界条件；3）运行时做好关键指标监控和告警；4）灰度升级服务。

对于人为故障，只能尽量降低需要人为操作的场景，例如提供接口来实现特定功能；提供“测试环境”来验证各种人为操作安全后，再上线操作；使用配置中心服务管理配置项；需要保证服务在出现错误时能够尽快恢复，例如快速回滚配置或进程；核心指标监控和告警。

### 可扩展性

可扩展性（*scalability*）可以用来衡量系统在负载增长时的表现：

- 在不扩容的情况下，负载超过当前系统最大容量时，会怎样。
- 负载超过最大容量时，如何扩容以保证系统重新回到正常状态。

** 确定影响系统性能的负载参数 **

在研究如何提高系统的可扩展性之前，我们需要先确定特定的系统的负载参数有哪些。这个是因系统而异的。

例如，对于web服务器，QPS（每秒收到的请求数）就是核心负载参数；对于数据库，读写比是核心负载参数；对于聊天室，同时在线用户数；对于缓存系统，缓存命中率。

这里DDIA拿Twiiter作例子：每个用户的首页，需要显示他关注的所有人发的推特，并按时间排序。

在DB层面，这些数据分别存在了3张表：用户信息表（users）; 推特表（tweets）；用户关注表（follows）。

这个系统的负载指标除了用户发推的QPS之外，还跟一个指标有关：每个用户的关注数。这个指标属于`fan-out`：输入一个uid，对应到db层面是放大了N倍的。

Twitter给出了两种方案：

方案一：直接查数据库。用户发推时，只用写到tweets表；每个用户登录后，都去DB做一次三表联结的查询。

方案二：每个用户维护一个主页时间线的缓存。用户发推时，不光要写tweets表，而且还要写到所有关注了他的用户的主页时间线缓存。但是用户查询主页时间线时只需要查询一次自己的缓存就可以。

当表越来越大时，方案一在查DB时耗时越来越长，只能切换到方案二。但是方案二，对名人（很多人关注他）来说，他发一次推文需要写百万千万用户的时间线缓存。

最终，Twitter将两种方案结合起来：对于名人，仍然采用方案一；对于普通人，采用方案二。

**确定衡量系统性能的指标参数**

确定了负载参数后，我们还需要确定如何去衡量我们系统的服务质量。

例如，对于批处理系统（Hadoop），我们可以用每秒处理记录数或每条记录处理时间来衡量；对于线上服务系统，我们可以用请求响应时间来衡量。

还是以线上服务来说，它可能提供各种功能的API，这条请求和那条请求，其响应时间差别很大。所以我们一般用统计学上的一些指标来衡量：

- 平均数。将所有请求的响应时间加上，然后除以请求总数。
- 百分率。例如，p99表示99%的请求响应时间都小于多少；p999表示99.9%的请求响应时间都小于多少。

一般我们用百分率特别是p999来衡量系统的质量：1000个用户只有1个用户的响应时间高于这个。对于大部分系统来说，要保证p9999有点困难和得不偿失：1/10000的失败可能是一些随机的不可控的因素导致的。

有了百分率作为衡量系统的质量指标之后，我们就可以定SLO（Service Level Objectives）和SLA（Service Level Agreement）了：我们可以商定我们的系统的目标是提供p999的响应时间在多少以下。如果没有达到这个指标，那就需要优化系统。

此外，在经验上，如果系统响应时间过大，一般是由于请求在系统内部排队过长：如果系统同时只能处理N条请求；那么当请求超过时就需要排队等待。排队时间加上请求处理时间，再加上网络时延，才是客户端感知到的响应时间。

PS：这点我在实际项目中是深有体会，所以在我的服务中，当请求处理线程从队列取出请求开始处理前，要先检查下它放入队列的时间，如果已经排队很长时间了，那就不需要处理了，因为这时候等你处理完再返回给客户端，从客户端角度来看，它已经认定请求失败了，甚至已经重新发起请求。所以，我们需要把这种过期的请求给丢弃，只处理较新的请求，以保证对资源的有效利用。特别是队列越堵，说明资源相对于输入是越匮乏的，越要用在刀刃上。

**扩展系统**

系统达到瓶颈后（质量指标开始下降），我们就要对系统进行扩容了：

- 垂直扩容。提升单台机器的处理能力。
- 水平扩容。提升机器数量，降低单台机器需要处理的请求数。

系统要支持水平扩容，那就是要设计好分布式负载均衡策略：相比于单机模式，复杂度上去了，也就更容易出问题。所以除非是单机真的搞不定，我们才需要去搞分布式。

系统架构设计是因项目而异的，没有万金油的架构可以适用所有的业务场景。但是组成系统的各个模块，却是可以借鉴的，这也是我们研究分布式系统的目的：只要你的工具库中的工具越多越全，在设计具体某个系统时，才能游刃有余。

### 可维护性

可维护性，更多的是说项目越大，越容易出现混乱：代码负载难懂；模块耦合严重，牵一发而动全身；越来越多的`if-else`用来处理越来越多的需求。

提高可维护性的核心，就是做好“抽象”。这个说起来容易，但是做起来就非常难了，也没有什么可以列出1,2,3的指导条款，只能根据自身的经验总结了。

一个具备良好的可维护性的系统应该：

- 容易理解。只要抽象的好，各个模块各司其职，耦合程度低，方便理解，也就方便维护了。
- 容易运维。这点不多说。
- 容易演进。没有项目说上线之后，需求就不变了。