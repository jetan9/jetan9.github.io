---
layout: post
category : Storm
title: Storm拓扑并行度浅析
tagline: ""
tags : [storm]
---
{% include JB/setup %}

## 运行中拓扑的组成要素：工作进程、执行线程和任务

下面是在Storm集群中用来运行拓扑的三种对象，它们之间有很大的不同。

1. 工作进程
2. 执行线程
3. 任务

下图是三者关系的简单说明：

![Storm中工作进程、执行线程和任务的关系](/images/relationships-worker-processes-executors-tasks.png)

_工作进程_执行拓扑的一个子集。工作进程属于特定的拓扑，可能包含一个或多个执行线程用来执行拓扑的一个或多个节点（Spout或Bolt）。运行中的拓扑由运行在Storm集群中多个机器上的多个这样的进程组成。

_执行线程_是工作进程产生的一个线程。它可能运行同一个节点（Spout或Bolt）的一个或多个任务。

_任务_完成实际的数据处理工作——用户在代码中实现的Spout或Bolt在集群上作为多个任务来运行。节点任务的数目在拓扑的生命周期中是恒定的，但是节点的执行线程的数目可能发生变化。这意味着不等式``#threads ≤ #tasks``成立。默认情况下，任务数被设置为和执行线程数相等，即Storm在每个线程中运行一个任务。

## 配置拓扑的并行度

注意，在Storm术语中，"并行度"用来描述所谓的_并行度提示_，即每个节点最初的执行线程的个数。在本文中，"并行度"不止被用来描述执行线程的个数，还被更一般地用于描述工作进程的个数以及Storm拓扑中任务的个数。如果仅仅是前者，文中会特别指明。

下面简要介绍各种配置项以及如何在代码中设置它们。有多种方法可以设置这些配置项，Storm目前遵从如下的[配置项顺序](Configuration.html): ``defaults.yaml`` < ``storm.yaml`` < 拓扑级别配置 < 节点内部配置 < 节点外部配置。

### 工作进程个数

* 描述：集群机器中用来创建_拓扑_的工作进程的个数。
* 配置项：[TOPOLOGY_WORKERS](/javadoc/apidocs/backtype/storm/Config.html#TOPOLOGY_WORKERS)
* 代码中的设置方法（示例）：
    * [Config#setNumWorkers](/javadoc/apidocs/backtype/storm/Config.html)

### 执行线程个数

* 描述：节点生成的执行线程的个数。
* 配置项：无（向``setSpout``或``setBolt``传递``parallelism_hint``参数）
* 代码中的设置方法（示例）：
    * [TopologyBuilder#setSpout()](/javadoc/apidocs/backtype/storm/topology/TopologyBuilder.html)
    * [TopologyBuilder#setBolt()](/javadoc/apidocs/backtype/storm/topology/TopologyBuilder.html)
    * 自Storm 0.8之后，``parallelism_hint``参数指定Bolt的初始执行线程（非任务）的个数。

### 任务个数

* 描述：节点生成的任务的个数。
* 配置项：[TOPOLOGY_TASKS](/javadoc/apidocs/backtype/storm/Config.html#TOPOLOGY_TASKS)
* 代码中的设置方法（示例）：
    * [ComponentConfigurationDeclarer#setNumTasks()](/javadoc/apidocs/backtype/storm/topology/ComponentConfigurationDeclarer.html)

下面是说明这些配置项的一个示例代码片段：

```java
topologyBuilder.setBolt("green-bolt", new GreenBolt(), 2)
               .setNumTasks(4)
               .shuffleGrouping("blue-spout");
```

上述代码中，我们指定Storm初始使用2个执行线程和4个任务来运行名为``GreenBolt``的Bolt。Storm将会在每个执行线程中运行2个任务。如果不显式指定任务的个数，Storm默认只会在每个执行线程中运行1个任务。

## 运行中拓扑示例

下图是一个简单的拓扑。它由3个节点组成：一个名为``BlueSpout``的Spout和分别名为``GreenBolt``和``YellowBolt``的两个Bolt。节点连接为``BlueSpout``向``GreenBolt``发送输出，后者则向``YellowBolt``发送输出。

![Storm运行中拓扑示例](/images/example-of-a-running-topology.png)

``GreenBolt``按前述代码配置，而``BlueSpout``和``YellowBolt``则只设置了并行度提示（执行线程个数）。代码如下：

```java
Config conf = new Config();
conf.setNumWorkers(2); // 使用2个工作进程

topologyBuilder.setSpout("blue-spout", new BlueSpout(), 2); // 并行度提示设置为2

topologyBuilder.setBolt("green-bolt", new GreenBolt(), 2)
               .setNumTasks(4)
               .shuffleGrouping("blue-spout");

topologyBuilder.setBolt("yellow-bolt", new YellowBolt(), 6)
               .shuffleGrouping("green-bolt");

StormSubmitter.submitTopology(
        "mytopology",
        conf,
        topologyBuilder.createTopology()
    );
```

当然，Storm还有其他的配置项用于控制拓扑的并行度，包括：

* [TOPOLOGY_MAX_TASK_PARALLELISM](/javadoc/apidocs/backtype/storm/Config.html#TOPOLOGY_MAX_TASK_PARALLELISM)：设置单个节点可以产生的执行线程个数的上限。通常用于在本地模式中运行拓扑进行测试时限制产生的线程的个数。可以通过[Config#setMaxTaskParallelism()](/javadoc/apidocs/backtype/storm/Config.html#setMaxTaskParallelism(int))进行设置。

## 如何修改运行中拓扑的并行度

Storm的一个很实用的特性是，用户可以无需重启集群或拓扑即可增减工作进程和/或执行线程的个数。这个过程被称为再平衡。

用户有两种方法再平衡拓扑：

1. 通过Storm UI界面再平衡拓扑。
2. 通过Storm命令行工具再平衡。

下面是使用命令行工具的一个例子：

```
## 重新配置拓扑"mytopology"以使用5个工作进程，
## Spout"blue-spout"使用3个执行线程，
## 并且Bolt"yellow-bolt"使用10个执行线程。

$ storm rebalance mytopology -n 5 -e blue-spout=3 -e yellow-bolt=10
```

## 参考资料

* [概念](storm-concepts.html)
* [配置](storm-configuration.html)
* [在生产集群中运行拓扑](storm-running-topologies-on-a-production-cluster.html)]
* [本地模式](Local-mode.html)
* [教程](storm-tutorial.html)
* [Storm API文档](/javadoc/apidocs/)，主要是类``Config``
