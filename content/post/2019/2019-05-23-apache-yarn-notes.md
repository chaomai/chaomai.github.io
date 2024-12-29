---
title: Apache Hadoop YARN笔记
date: 2019-05-23 21:49:24
categories:
    - yarn
tags:
    - yarn
    - hadoop
---

# 介绍
YARN（Yet Another Resource Negotiator）是Hadoop的资源管理系统。

YARN把资源管理和任务的调度/监控拆分到了独立的进程，即ResourceManager（RM）和每个程序的ApplicationMaster，一个程序要么是一个单独的job或者是由DAG表示的多个job。

RM和NodeManager（NM）构成了数据计算的框架。RM拥有最大的权利来决断系统中每个程序所需的资源。NM是每个机器上的一个代理，负责container的管理，监控它们资源使用（cpu、内存、磁盘、网络），并汇报给RM。

每个程序的ApplicationMaster是框架的特定库，负责与RM协商资源，并与NMs一起工作来执行和监控任务。

RM有两个组成部分：Scheduler和ApplicationsManager（AM）。

Scheduler，在容量和队列的限制下，负责为各种程序分配资源。Scheduler只负责调度，不负责监控和跟踪程序的状态；也不为由于程序错误或硬件错误导致的任务失败提供重启保证。Scheduler基于程序的资源需求来执行调度功能；进一步说，是基于对资源的抽象，即Container（cpu、内存、磁盘、网络）来进行调度的。

Scheduler有一个可插拔策略，负责在各种队列和程序之间对集群资源进行划分。例如当前的scheduler有CapacityScheduler和FairScheduler。

AM负责接收job的提交、协商第一个container来运行程序的ApplicationMaster，以及为出错的ApplicationMaster container提供重启服务。每个程序的ApplicationMaster负责与Scheduler协商资源适当的的container，并追踪它们的状态和监控进度。

通过ReservationSystem，YARN还支持资源预定。

# YARN应用的运行
![](images/2019/15585937581379.jpg)

## 资源请求
YARN的资源请求模型会考虑，
* 每个容器需要的资源。
* 局部性（主要指数据的局部性）。
    例如如果使用了HDFS的数据，会优先使用存放副本的结点，其次是存有这些副本的机架，最后才是集群的任意结点。

YARN应用可以在任意时刻提出资源的申请，
* 在一开始就申请所有的资源。
* 以动态的方式，在需要更多资源的时候提出。

## 应用生命周期
按照应用的类型，应用的生命周期会有较大差异，主要分为以下3个模型，
1. 一个应用对应一个用户的job，例如MR任务。
2. 一个应用对应一个工作流或用户jobs的session，container可以在job之间复用，并cache数据，例如Spark。
3. 一个长期运行的应用被多个用户共享。这样的应用一般作为协调者的角色存在。

# YARN优势
* 可扩展性（Scalability）
    每个应用都有一个专门的application master，分离了资源调度和task管理。就MR任务而言，这模型与Google MapReduce论文中所述的模型更加接近，即，一个master协调worker上的map和reduce任务。
* 可用性（Availability）
    拆分RM和application master简化了高可用的实现。先为RM提供高可用，再为YARN应用提供高可用。
* 利用率（Utilization）
    相比MapReduce 1，精细化了资源的管理，应用可以按需请求资源。
* 多租户（Multitenancy）
    YARN支持除MapReduce外的其他分布式计算框架。

# 调度
YARN有3中调度器：FIFO调度器、容量调度器和公平调度器。

## 关于container
vcore是一个host的cpu核心占用比例。

container是，
* cpu（vcore）、内存、磁盘、网络的抽象。
* 在有task或ApplicationMaster运行的时候，表示一个已分配的资源。
* *不同*于docker中的container概念。

```java
public abstract class ContainerLaunchContext {
  // ...
  /**
   * Add the list of <em>commands</em> for launching the container. All
   * pre-existing List entries are cleared before adding the new List
   * @param commands the list of <em>commands</em> for launching the container
   */
  @Public
  @Stable
  public abstract void setCommands(List<String> commands);
  // ...
}
```

## FIFO调度器（FIFO Scheduler）
按提交的顺序运行应用，首先为第一个应用分配资源，如果可以满足，再依次为其他应用服务。

## 容量调度器（Capacity Scheduler）
为每个组织分配一个专门的队列，每个队列可配置为使用一定量的集群资源，队列可以再进行划分。同一个队列内使用FIFO策略进行调度。

关于资源的使用，
* 队列中单个任务使用的资源不会超过队列的容量。
* 如果队列满，且集群有空闲的资源，调度器可以把资源分配给此队列（可配置），弹性队列。
* 正常情况下，容量调度器不会抢占容器，因此如果一个队列随着使用，资源不够时，只能等待其他队列释放资源。
    容量调度器也可以执行work-preserving preemption，RM会请求应用返回容器。

## 公平调度器（Fair Scheduler）
* 每个队列有权重元素，用于fair share的计算。
* 默认队列和动态创建的队列，权重为1（默认队列的可配置）。
* 调度器会使用最小资源数量来进行资源分配进行优先排序。如果两个队列的资源都低于fair share额度，那么远低于最小资源数量的队列，会被有限分配资源。

### 队列放置
公平调度器使用一个规则的系统来判断应用所属队列。

### 饥饿和抢占
FairShare的计算会被用于判断饥饿以及是否进行抢占。在计算FairShare时，有两种：
* Steady FairShare，按照配置文件中所有queue的weight，计算出的。
* Instantaneous FairShare，，按照配置文件中所有queue的weight，仅对包含活动应用程序的queue计算出的。

在配置`yarn.scheduler.fair.preemption`和`yarn.scheduler.fair.preemption.cluster-utilization-threshold`后，抢占会启用。

**饥饿**有两种：
* FairShare Starvation
    判定条件为：
    1. 未获得所要求的资源。
    2. 应用程序资源使用低于Instantaneous FairShare。
    3. 应用程序的资源使用低于fairSharePreemptionThreshold，并持续fairSharePreemptionTimeout。

    要注意的是，在同一个队列里面，如果存在多个应用程序，它们会平均的分摊Instantaneous FairShare。因此可能存在队列整体不是饥饿状态，但是每个应用程序是。
* MinShare Starvation
    判定条件为：
    1. 未获得所要求的资源。
    2. 应用程序资源使用低于MinShare。
    3. 应用程序的资源使用低于MinShare，并持续MinSharePreemptionTimeout。

决定需要进行抢占的时候，可能在多个队列中都有可抢占的container，决定container是否可以被抢占，需要满足：
* 所在队列是可抢占的。
* 杀死container以后不会导致应用程序的资源低于Instantaneous FairShare。

启用抢占**并不能**保证队列或应用程序能够获得所有的Instantaneous FairShare。只能最终保证脱离饥饿的状态，即获得fairSharePreemptionThreshold份额的资源。

FairShare Starvation、MinShare Starvation以及抢占的关系如下：

![](images/2019/15598134204968.jpg)

### Best Practice
* 一般**不建议**配置MinShare Starvation或minimum resources。
    增加复杂性的同时，并不能带来多少好处。
* 如果配置minimum resources，所有minimum resources的加和不能超出总的资源数。

## 延迟调度
局部性是YARN调度时优先考虑的，但如果发现所请求的节点资源不够，那么任务可能就会被调度到其他节点上了。此时如果等待几秒，能够增加在所请求节点上分配到container的机会。

# References
1. [Apache Hadoop YARN](http://hadoop.apache.org/docs/current/hadoop-yarn/hadoop-yarn-site/YARN.html)
2. [Untangling Apache Hadoop YARN](https://blog.cloudera.com/blog/2015/09/untangling-apache-hadoop-yarn-part-1/)
3. [YARN FairScheduler Preemption Deep Dive](https://blog.cloudera.com/blog/2018/06/yarn-fairscheduler-preemption-deep-dive/)
4. Hadoop - The Definitive Guide
