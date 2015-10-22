---
layout: post
category : Storm
title: Storm概念
tagline: ""
tags : [storm]
---
{% include JB/setup %}

本文列出Storm的主要概念以及一些补充资源的链接。主要讨论以下概念：

1. 拓扑
2. 流
3. Spout
4. Bolt
5. 流分组策略
6. 可信性
7. 任务
8. 工作进程

### 拓扑

实时计算应用的逻辑被封装到Storm拓扑中。Storm拓扑类似MapReduce任务。一个关键的不同点是MapReduce任务终将结束，而拓扑将持续运行直到用户手工终止。拓扑是由流分组连接起来的spout和bolt组成的图。这些概念描述如下。

**资源：**

* [TopologyBuilder](/javadoc/apidocs/backtype/storm/topology/TopologyBuilder.html)：Java中通过此类构造拓扑。
* [在生产集群中运行拓扑](storm-running-topologies-on-a-production-cluster.html)
* [本地模式](storm-local-mode.html)：如何在本地模式中开发和测试拓扑。

### 流

流是Storm的核心抽象概念。流是无穷的元组序列，可被分布式地并行创建和处理。流可以通过在模式中指定字段名称来定义。默认情况下，流可以包含integer、long、short、byte、string、double、float、boolean和byte数组。用户也可以通过定义序列化器来在元组中使用自定义类型。

流在声明时需要指定一个id。由于一般spout和bolt只有一个流，[OutputFieldsDeclarer](/javadoc/apidocs/backtype/storm/topology/OutputFieldsDeclarer.html)提供了无需指定id来声明流的简便方法。这时流的默认id为"default"。


**资源：**

* [Tuple](/javadoc/apidocs/backtype/storm/tuple/Tuple.html)：流由元组组成。
* [OutputFieldsDeclarer](/javadoc/apidocs/backtype/storm/topology/OutputFieldsDeclarer.html)：用于声明流及其模式。
* [序列化](storm-serialization.html)：关于Storm的动态类型及自定义序列化器。
* [ISerialization](/javadoc/apidocs/backtype/storm/serialization/ISerialization.html)：自定义的序列化器需实现此接口。
* [CONFIG.TOPOLOGY_SERIALIZATIONS](/javadoc/apidocs/backtype/storm/Config.html#TOPOLOGY_SERIALIZATIONS)：自定义的序列化器通过此配置进行注册。

### Spout

Spout是拓扑中流的源。一般地，Spout从外部源（如Kestrel队列或Twitter API）读取元组然后发射到拓扑中。Spout可以是__可信的__或__不可信的__。可信的Spout可以在Storm未能成功处理元组时重新发送元组，而不可信的Spout则在元组被发送之后立即清除相关信息。

Spout可以发射多个流。可通过[OutputFieldsDeclarer](/javadoc/apidocs/backtype/storm/topology/OutputFieldsDeclarer.html)的`declareStream`方法声明多个流，并在使用[SpoutOutputCollector](/javadoc/apidocs/backtype/storm/spout/SpoutOutputCollector.html)的`emit`方法时指定要发射的流。

Spout的核心方法是`nextTuple`。`nextTuple`可以向拓扑发射新的元组，也可以在不需要发射元组时直接返回。由于Storm在同一个线程中调用Spout的方法，所以在Spout的实现中`nextTuple`不能阻塞线程。

Spout的其他方法主要有`ack`和`fail`。Storm在检测到从Spout发射出的元组被拓扑完全成功处理或者处理失败时调用这两个方法。Storm只为可信Spout调用`ack`和`fail`方法。参考[API文档](/javadoc/apidocs/backtype/storm/spout/ISpout.html)获得更多信息。

**资源：**

* [IRichSpout](/javadoc/apidocs/backtype/storm/topology/IRichSpout.html)：Spout需要实现的接口。
* [强制消息处理](storm-guaranteeing-message-processing.html)

### Bolt

拓扑的所有处理工作在Bolt中完成。Bolt可以过滤元组、调用方法、聚合元组、连接元组或查询数据库，等等。

Bolt可以进行简单的流变换。复杂的流变换需要多个步骤，从而需要多个Bolt来实现。例如，将推文流变换为热点话题流至少需要两步：首先，一个Bolt负责计算每个话题相关的推文数，其次，另一个Bolt负责选出相关推文最多的X个话题（如果采用3个Bolt则可以更灵活地进行流变换）。

Bolt可以发射多个流。可通过[OutputFieldsDeclarer](/javadoc/apidocs/backtype/storm/topology/OutputFieldsDeclarer.html)的`declareStream`方法声明多个流，并在使用[OutputCollector](/javadoc/apidocs/backtype/storm/task/OutputCollector.html)的`emit`方法时指定要发射的流。

声明Bolt的输入流时，总是需要订阅其他节点的某个流。如果需要订阅某个节点的所有流，只能逐次分别订阅。[InputDeclarer](/javadoc/apidocs/backtype/storm/topology/InputDeclarer.html)提供了定义默认流的简便方法。调用`declarer.shuffleGrouping("1")`即可订阅节点"1"的默认流，这和`declarer.shuffleGrouping("1", DEFAULT_STREAM_ID)`是等价的。

Bolt的核心方法是`execute`，它接受元组作为输入。Bolt通过[OutputCollector](/javadoc/apidocs/backtype/storm/task/OutputCollector.html)对象发射元组。Bolt必须为每个处理的元组调用`OutputCollector`的`ack`方法以通知Storm此元组已被成功处理（从而Storm可以安全确认最初的Spout发出的元组被成功处理）。为实现常规的处理输入元组、发射0个或多个相应元组并确认输入元组的逻辑，Storm提供了[IBasicBolt](/javadoc/apidocs/backtype/storm/topology/IBasicBolt.html)接口以自动完成确认操作。

注意，[OutputCollector](/javadoc/apidocs/backtype/storm/task/OutputCollector.html)不是线程安全的，所有的发射、确认和失败操作都必须在同一个线程中完成。更多细节请参考[故障排查](storm-troubleshooting.html)。

**资源：**

* [IRichBolt](/javadoc/apidocs/backtype/storm/topology/IRichBolt.html)：Bolt的通用接口。
* [IBasicBolt](/javadoc/apidocs/backtype/storm/topology/IBasicBolt.html)：定义实现过滤或简单功能的Bolt的简便接口。
* [OutputCollector](/javadoc/apidocs/backtype/storm/task/OutputCollector.html)：Bolt通过此类实例向其输出流发射元组。
* [强制消息处理](storm-guaranteeing-message-processing.html)

### 流分组策略

定义拓扑时需要为每个Bolt指定其所要读取的流。流分组策略定义了流是如何在Bolt的任务之间进行分组的。

Storm有8种内建分组策略，同时用户可以通过实现[CustomStreamGrouping](/javadoc/apidocs/backtype/storm/grouping/CustomStreamGrouping.html)接口来实现自定义的分组策略。

1. **随机分组**：元组在Bolt的任务之间随机分配，每个Bolt任务将获得相同数量的元组。
2. **字段分组**：流根据指定字段的值进行分组。例如，如果流按照"user-id"字段进行分组，则具有相同"user-id"的元组总是被发送到同一个任务，而不同"user-id"的元组则可能被发送到不同的任务。
3. **部分键分组**：类似字段分组，流根据指定字段的值进行分组，但是在下游的Bolt任务之间会进行负载均衡，这样在数据发生抖动时可以更好地利用资源。[相关论文](https://melmeric.files.wordpress.com/2014/11/the-power-of-both-choices-practical-load-balancing-for-distributed-stream-processing-engines.pdf)详细介绍了其实现机制和带来的好处。
4. **全部分组**：流在Bolt的所有任务上进行复制。慎用此分组策略。
5. **全局分组**：整个流被发送到Bolt的某个特定任务。准确地说，是具有最小id的任务。
6. **无分组**：表明用户并不关心流如何被分组。目前，无分组和随机分组是等价的。未来Storm可能会将指定为无分组的Bolt放到所订阅的Bolt或Spout所在线程中执行。
7. **直接分组**：这是一种特殊的分组策略。这意味着由元组的__生产者__来决定消费的哪个任务来接收元组。直接分组只能被直接流声明。被发射到直接流的元组只能通过[emitDirect](/javadoc/apidocs/backtype/storm/task/OutputCollector.html#emitDirect(int, int, java.util.List)方法之一来发送。Bolt可以通过[TopologyContext](/javadoc/apidocs/backtype/storm/task/TopologyContext.html)来获取其消费者的任务id。通过跟踪[OutputCollector](/javadoc/apidocs/backtype/storm/task/OutputCollector.html)的`emit`方法的返回值（元组所发送到的任务的id）也可以实现此目的。
8. **本地随机分组**：如果目标Bolt在同一个工作进程中有一个或多个任务，元组只会被发送到进程内任务。否则，和普通的随机分组并无区别。

**资源：**

* [TopologyBuilder](/javadoc/apidocs/backtype/storm/topology/TopologyBuilder.html)：用于定义拓扑。
* [InputDeclarer](/javadoc/apidocs/backtype/storm/topology/InputDeclarer.html)：`TopologyBuilder`的`setBolt`方法返回此对象，用于声明Bolt的输入流以及流的分组策略。
* [CoordinatedBolt](/javadoc/apidocs/backtype/storm/task/CoordinatedBolt.html)：此Bolt在分布式RPC拓扑中被用到，并大量使用了直接流和直接分组。

### 可信性

Storm保证每个从Spout发射出的元组都被拓扑充分处理。这是通过跟踪一颗由Spout发出的元组所触发的元组树来实现的。每个拓扑都会设置一个消息超时时间。如果Storm在规定的超时时间内没有检测到Spout发出的元组被成功处理，则认为处理失败并在稍后重新发送元组。

要利用Storm的可信性特性，用户必须在向元组树中添加新边和成功处理某个元组时通知Storm。这是通过Bolt用来发射元组的[OutputCollector](/javadoc/apidocs/backtype/storm/task/OutputCollector.html)对象来完成的。添加新边在`emit`方法中实现，确认元组被成功处理则通过调用`ack`方法实现。

这部分内容在[强制消息处理](storm-guaranteeing-message-processing.html)一文中进行更详细的说明。

### 任务

Spout或Bolt在集群中以任务的形式被执行。每个任务对应一个线程，同时流分组策略定义了如何从一个任务集向另一个任务集发送元组。用户通过[TopologyBuilder](/javadoc/apidocs/backtype/storm/topology/TopologyBuilder.html)的`setSpout`和`setBolt`方法设置每个Spout或Bolt的并行度。

### 工作进程

拓扑在一个或多个工作进程中运行。每个工作进程是一个物理JVM并执行拓扑任务的一个子集。例如，如果拓扑的总体并行度为300且分配了50个工作进程，则每个工作进程将执行6个任务（工作进程内部的线程）。Storm尝试在所有的工作进程之间平均分配任务。

**资源：**

* [Config.TOPOLOGY_WORKERS](/javadoc/apidocs/backtype/storm/Config.html#TOPOLOGY_WORKERS)：设置运行拓扑时分配的工作进程的数量。
