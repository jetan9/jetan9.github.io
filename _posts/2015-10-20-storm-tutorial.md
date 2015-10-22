---
layout: post
category : Storm
title: Storm教程
tagline: ""
tags : [storm]
---
{% include JB/setup %}

本文介绍如何创建Storm拓扑（topology）并将其部署到Storm集群上。使用的主要语言为Java，在介绍Storm的多语言特性时也会用Python作为示范。

## 预备知识

本文使用的例子来自于[storm-starter](https://github.com/apache/storm/blob/master/examples/storm-starter)。推荐将此工程复制到本地。进一步的操作请参考[建立开发环境](storm-setting-up-development-environment.html)和[创建新的Storm工程](storm-creating-a-new-storm-project.html)。

## Storm集群的组成

Storm集群和Hadoop集群看起来很相似。在Hadoop上运行的是MapReduce任务，而在Storm上运行的是拓扑。任务与拓扑差别很大，其中一个关键的不同是MapReduce任务终将结束运行，而拓扑进程将不停地处理消息（除非被手动终止）。

Storm集群中有两种节点，即主节点（master node）和工作节点（worker node）。类似Hadoop的JobTracker，主节点运行一个叫做nimbus的守护进程。nimbus负责分发代码、分配任务和监控故障。

每个工作节点运行一个叫做supervisor的守护进程。supervisor监听分配给宿主机的任务，并按照nimbus的指令启动或终止工作进程。每个工作进程运行的是一个拓扑的子集，每个运行中的拓扑由分布在多台机器上的工作进程组成。

![Storm集群](/images/storm-cluster.png)

nimbus和supervisor之间的配合通过[Zookeeper](http://zookeeper.apache.org/)集群来完成。另外，nimbus和supervisor守护进程都是速错（fail-fast）和无状态的，所有的状态都保存在zookeeper或者本地磁盘中。这意味着nimbus或supervisor进程可以轻松重启。这样的设计使得Storm集群非常稳定。

## 拓扑

要在Storm上进行实时计算，需要创建拓扑。拓扑即计算的图结构。拓扑中的每个节点上都有处理逻辑，节点间的边定义了数据的流向。

运行拓扑非常简单。首先，将所有代码及依赖项打包为一个jar包，然后运行如下命令：

```
storm jar all-my-code.jar backtype.storm.MyTopology arg1 arg2
```

此命令运行类`backtype.storm.MyTopology`，参数为`arg1`和`arg2`。类中的main方法定义了拓扑并提交到nimbus。`storm jar`负责连接nimbus并上传jar包。

由于拓扑定义实际上是thrift结构体，而且nimbus是一个thrift服务，所以可以使用任意语言来创建和提交拓扑。上面的例子是使用基于JVM的语言的最简单的方法。关于启动/停止拓扑的更多信息请参考[在生产集群中运行拓扑](storm-running-topologies-on-a-production-cluster.html)。

## 流

Storm中最核心的抽象概念是流。流是无穷的元组序列。Storm提供了一组原语用于变换流，这些变换是分布式且可靠的。例如，可以将推文流转化为热点话题流。

Storm提供的用于变换流的最基本的原语是spout和bolt。spout和bolt提供了一组接口，需要实现这些接口以运行应用程序逻辑。

spout是流的源。例如，spout可以从[Kestrel](http://github.com/nathanmarz/storm-kestrel)队列读取元组，然后作为流发射出去。spout也可以连接到Twitter API然后发射推文流。

bolt接收流的输入、进行处理并可能发射新的流。诸如通过推文流计算热点话题流的复杂流变换需要多步来完成，所以需要通过多个bolt来实现。bolt可以运行方法、过滤元组、聚合流、连接流或者查询数据库，等等。

spout和bolt组成的网状结构被打包为最顶层的抽象概念——拓扑，并被提交到Storm集群进行执行。拓扑是一个流变换图，其中的节点是spout或bolt之一。图中的边说明了bolt和流之间的订阅关系。当spout或bolt向流中发射元组时，该元组被发送到所有订阅该流的bolt。

![Storm拓扑](/images/topology.png)

拓扑中节点之间的链接定义了元组如何被传递。例如，spout A和bolt B之间、spout A和bolt C之间以及bolt B和bolt C之间各有一条链接，则每次spout A发射元组时，该元组都会被发送到bolt B和bolt C。bolt B的输出元组则被发送到bolt C。

Storm拓扑中的每个节点都是并行执行的。在拓扑中，可以为每个节点指定并行度，Storm会在集群中创建指定数目的线程来执行任务。

拓扑持续运行，直到被手工终止。Storm会自动重新分配失败的任务。另外，即使机器停机或者消息被丢弃，Storm也可以保证数据不丢失。

## 数据模型

Storm使用元组作为数据模型。元组是一个命名值列表，其中的每个字段可以是任意类型的对象。Storm本身支持所有的基本类型用作元组的值字段，包括string和byte数组。要使用其他类型，需要实现该类型对应的[序列化器](storm-serialization.html)。

拓扑中的每个节点必须声明所要发射的元组的字段。例如，下面的bolt声明它将发送一个二元组，其中字段为“double”和“triple”：

```java
public class DoubleAndTripleBolt extends BaseRichBolt {
    private OutputCollectorBase _collector;

    @Override
    public void prepare(Map conf, TopologyContext context, OutputCollectorBase collector) {
        _collector = collector;
    }

    @Override
    public void execute(Tuple input) {
        int val = input.getInteger(0);        
        _collector.emit(input, new Values(val*2, val*3));
        _collector.ack(input);
    }

    @Override
    public void declareOutputFields(OutputFieldsDeclarer declarer) {
        declarer.declare(new Fields("double", "triple"));
    }    
}
```

`declareOutputFields`方法为组件声明了输出字段`["double", "triple"]`。其余部分将在下面的小节中进行解释。

## 一个简单的拓扑

我们通过一个简单的拓扑来介绍一下基本概念和代码结构。下面是storm-starter中的`ExclamationTopology`：

```java
TopologyBuilder builder = new TopologyBuilder();        
builder.setSpout("words", new TestWordSpout(), 10);        
builder.setBolt("exclaim1", new ExclamationBolt(), 3)
        .shuffleGrouping("words");
builder.setBolt("exclaim2", new ExclamationBolt(), 2)
        .shuffleGrouping("exclaim1");
```

这个拓扑包含一个spout和两个bolt。spout发射单词，bolt则向读取的单词末尾添加字符串"!!!"。节点按线性排列，即spout发射到第一个bolt，后者则发射到第二个bolt。如果spout发射了元组["bob"]和["john"]，则第二个bolt将发射单词["bob!!!!!!"]和["john!!!!!!"]。

这段代码使用`setSpout`和`setBolt`方法来定义节点。这些方法接受一个用户定义的id、一个包含处理逻辑的对象和节点的并行度作为输入。在本例中，spout的id为"words"，bolt的id则分别为"exclaim1"和"exclaim2"。

包含处理逻辑的对象实现了spout的[IRichSpout](/javadoc/apidocs/backtype/storm/topology/IRichSpout.html)接口和bolt的[IRichBolt](/javadoc/apidocs/backtype/storm/topology/IRichBolt.html)接口。

最后一个定义节点并行度的参数是可选的。它指定了集群中应该有多少个线程用于执行此组件的任务。如果不指定这个参数，Storm只会为此节点分配一个线程。

`setBolt`返回一个[InputDeclarer](/javadoc/apidocs/backtype/storm/topology/InputDeclarer.html)对象，该对象用于定义bolt的输入。本例中，节点"exclaim1"声明希望读取节点"words"所发射的全部元组且分组策略为随机分组（shuffle grouping），而节点"exclaim2"声明希望读取节点"exclaim1"所发射的全部元组且分组策略同样为随机分组。随机分组意味着元组从输入任务传送到bolt的任务时是随机分布的。节点间的数据分组策略有很多种，将在稍后进行说明。

如果需要节点"exclaim2"同时读取从节点"words"和节点"exclaim1"发射的全部元组，则应将节点"exclaim2"定义为：

```java
builder.setBolt("exclaim2", new ExclamationBolt(), 5)
            .shuffleGrouping("words")
            .shuffleGrouping("exclaim1");
```

如上所示，可以通过链接输入声明来为bolt指定多个源。

接下来看看这个拓扑中spout和bolt的实现。spout负责向拓扑中发射新消息。这个拓扑中的`TestWordSpout`每100ms从列表["nathan", "mike", "jackson", "golda", "bertels"]中随机选择一个单词作为一元组进行发射。`TestWordSpout`的`nextTuple()`方法实现如下：

```java
public void nextTuple() {
    Utils.sleep(100);
    final String[] words = new String[] {"nathan", "mike", "jackson", "golda", "bertels"};
    final Random rand = new Random();
    final String word = words[rand.nextInt(words.length)];
    _collector.emit(new Values(word));
}
```

可以看到实现非常简单。

`ExclamationBolt`在输入之后添加字符串"!!!"。下面来看一下`ExclamationBolt`的完整实现：

```java
public static class ExclamationBolt implements IRichBolt {
    OutputCollector _collector;

    @Override
    public void prepare(Map conf, TopologyContext context, OutputCollector collector) {
        _collector = collector;
    }

    @Override
    public void execute(Tuple tuple) {
        _collector.emit(tuple, new Values(tuple.getString(0) + "!!!"));
        _collector.ack(tuple);
    }

    @Override
    public void cleanup() {
    }

    @Override
    public void declareOutputFields(OutputFieldsDeclarer declarer) {
        declarer.declare(new Fields("word"));
    }
    
    @Override
    public Map<String, Object> getComponentConfiguration() {
        return null;
    }
}
```

`prepare`方法为bolt提供了一个`OutputCollector`对象用于发射元组。元组可在任意时刻被发射，无论是在`prepare`，`execute`还是`cleanup`方法中，甚至在其他线程中异步发射也可以。本例中，`prepare`只是将`OutputCollector`保存为一个变量，稍后在`execute`方法中使用。

`execute`方法接受来自bolt的一个输入的元组。`ExclamationBolt`提取元组的第一个字段，添加后缀字符串"!!!"之后放入输出流。如果实现的bolt订阅了多个输入源，可以通过`Tuple#getSourceComponent`方法获得[元组](/javadoc/apidocs/backtype/storm/tuple/Tuple.html)的来源。

`execute`方法还完成了一些其他任务，例如输入元组被作为第一个参数传递给了`emit`方法，并且在最后一行输入元组被确认。这是Storm的可靠性API，用来确保不会发生数据丢失。本文稍后将予以说明。

bolt关闭时应该释放任何已打开的资源，这通过调用`cleanup`方法来完成。在集群中并不保证此方法被调用。例如运行任务的机器停机，则没有机会来调用此方法。`cleanup`方法主要在[本地模式](storm-local-mode.html)（其中Storm集群通过进程来模拟）中被调用，这样就可以任意启停拓扑而不用担心资源泄露。

`declareOutputFields`方法声明`ExclamationBolt`发射的是只包含一个字段"word"的一元组。

`getComponentConfiguration`方法允许用户配置节点运行时的各种属性。这是一个高级主题，将在[配置](Configuration.html)一文中进行讲解。

诸如`cleanup`和`getComponentConfiguration`的方法一般不需要实现。必要时，用户可以通过继承提供了这些方法的缺省实现的类来更简洁地定义bolt。`ExclamationBolt`可以通过继承`BaseRichBolt`进行重写：

```java
public static class ExclamationBolt extends BaseRichBolt {
    OutputCollector _collector;

    @Override
    public void prepare(Map conf, TopologyContext context, OutputCollector collector) {
        _collector = collector;
    }

    @Override
    public void execute(Tuple tuple) {
        _collector.emit(tuple, new Values(tuple.getString(0) + "!!!"));
        _collector.ack(tuple);
    }

    @Override
    public void declareOutputFields(OutputFieldsDeclarer declarer) {
        declarer.declare(new Fields("word"));
    }    
}
```

## 本地模式运行ExclamationTopology

下面介绍如何以本地模式运行`ExclamationTopology`并且观察它是如何工作的。

Storm有两种模式：本地模式和分布式模式。在本地模式中，Storm完全在单个进程中运行，并通过线程来模拟工作节点。本地模式在开发和测试拓扑时非常有用。storm-starter中的拓扑可以以本地模式运行，可以通过控制台观察每个节点所发射的信息。关于在本地模式中运行拓扑的其他信息请参考[本地模式](storm-local-mode.html)。

在分布式模式中，Storm作为集群来运行。用户向master提交拓扑时，运行拓扑所需的所有代码被一并提交。master负责分发代码和分配工作线程来运行拓扑。如果工作线程终止，master会重新分配任务。关于在集群中运行拓扑的更多信息可以参考[在生产集群中运行拓扑](storm-running-topologies-on-a-production-cluster.html)]. 

下面是在本地模式中运行`ExclamationTopology`的代码：

```java
Config conf = new Config();
conf.setDebug(true);
conf.setNumWorkers(2);

LocalCluster cluster = new LocalCluster();
cluster.submitTopology("test", conf, builder.createTopology());
Utils.sleep(10000);
cluster.killTopology("test");
cluster.shutdown();
```

首先，通过创建`LocalCluster`对象，代码定义了一个进程内集群。向这个虚拟集群提交拓扑和向分布式集群提交拓扑是等价的。代码调用`submitTopology`方法向`LocalCluster`提交拓扑，该方法接受的参数为拓扑名称、拓扑配置和拓扑对象本身。

拓扑名称用于标识拓扑，稍后用于终止拓扑。拓扑将持续运行，直到用户手工终止。

配置用来调整运行中的拓扑的各种属性。这里列出两类常用的配置：

1. **TOPOLOGY_WORKERS**（通过`setNumWorkers`设置）指定集群中用来运行拓扑的_进程_的数目。拓扑中的每个节点都会作为_线程_来运行。给定节点的线程数通过`setBolt`和`setSpout`方法来设置。这些_线程_存在于工作_进程_之中。每个工作_进程_中都包含若干节点的一些_线程_。例如，用户为所有节点在配置文件中指定了300个线程和50个工作进程。每个工作进程将执行6个线程，每个线程可能属于不同的节点。用户可以通过调整节点的并行度和线程所属的工作进程的数目来优化Storm拓扑的性能。
2. **TOPOLOGY_DEBUG**（通过`setDebug`设置）被设置为true时，Storm将会为每个节点所发射的每条消息生成日志。在本地模式中调试拓扑时打开这个选项会很有用，但在集群中应该将其关闭。

拓扑还有很多其他的配置可以设置。详细内容可以参考[Config](/javadoc/apidocs/backtype/storm/Config.html)的文档。

关于如何建立开发环境并以本地模式运行拓扑（例如在Eclipse中），请参考[新建Storm工程](storm-creating-a-new-storm-project.html)。

## 流分组策略

流分组策略告诉拓扑如何在节点之间发送元组。请记住，spout和bolt在集群中是以多个任务的形式并行运行的。从任务层面来观察，拓扑是这样运行的：

![拓扑中的任务](/images/topology-tasks.png)

当bolt A的一个任务向bolt B发射元组时，元组会被发送到哪个任务呢？

流分组策略通过告诉Storm如何在任务集合之间发送元组来解决这个问题。在深入探讨不同流分组策略之间的差异之前，我们来看一看[storm-starter](http://github.com/apache/storm/blob/master/examples/storm-starter)中的另一个拓扑。这个名为[WordCountTopology](https://github.com/apache/storm/blob/master/examples/storm-starter/src/jvm/storm/starter/WordCountTopology.java)的拓扑从spout读取一系列句子，然后通过`WordCountBolt`发送截止某一时刻它所观察到的某个单词的总数：

```java
TopologyBuilder builder = new TopologyBuilder();
        
builder.setSpout("sentences", new RandomSentenceSpout(), 5);        
builder.setBolt("split", new SplitSentence(), 8)
        .shuffleGrouping("sentences");
builder.setBolt("count", new WordCount(), 12)
        .fieldsGrouping("split", new Fields("word"));
```

`SplitSentence`为接收到的每个句子中的每个单词发射一个元组，`WordCount`则在内存中保存一个单词计数的map。每当`WordCount`接收到一个单词，它就会更新map并且发射新的单词数目。

有多种不同的分组策略。

最简单的分组策略是随机分组，即每个元组被发送给随机的任务。`WordCountTopology`使用随机分组将从`RandomSentenceSpout`接收到的元组发送给`SplitSentence`。它可以将处理元组的工作量平均分配到`SplitSentence`的所有任务上。

A另一种更加有趣的分组策略是字段分组。`SplitSentence`和`WordCount`之间使用的就是字段分组。确保同一个单词总是被发送到同一个任务对于`WordCount`的正确运行很重要。否则，多个不同的任务将接收到同样的单词，由于信息不完整将会发射错误的计数值。字段分组允许用户通过字段对流进行分组，从而保证具有相同字段值的元组被发送到同一个任务。由于`WordCount`通过基于"word"字段的字段分组订阅了`SplitSentence`的输出流，同一个单词总是被发送到同一个任务，从而保证了结果的正确性。

字段分组是流连接和聚合以及很多其他应用的基础。字段分组通过哈希值的模运算来实现。

除此之外还有一些其他的分组策略，请参考[概念](storm-concepts.html)。

## 使用其它语言定义bolt

bolt可以通过任意语言定义。使用其他语言编写的bolt以子进程的形式运行，Storm通过stdin/stdout流中的JSON消息和这些子进程进行通信。通信协议需要借助一个约100行的适配库完成，Storm为Ruby、Python和Fancy提供了适配库。

下面是`WordCountTopology`中`SplitSentence`的实现：

```java
public static class SplitSentence extends ShellBolt implements IRichBolt {
    public SplitSentence() {
        super("python", "splitsentence.py");
    }

    public void declareOutputFields(OutputFieldsDeclarer declarer) {
        declarer.declare(new Fields("word"));
    }
}
```

`SplitSentence`继承了`ShellBolt`并且声明它将使用参数`splitsentence.py`来运行`python`。下面是`splitsentence.py`的实现：

```python
import storm

class SplitSentenceBolt(storm.BasicBolt):
    def process(self, tup):
        words = tup.values[0].split(" ")
        for word in words:
          storm.emit([word])

SplitSentenceBolt().run()
```

关于如何使用其他语言编写spout和bolt以及使用其它语言创建拓扑，请参考[在Storm中使用非JVM语言](storm-using-non-jvm-languages-with-storm.html)。

## 强制消息处理

本文前面省略了元组发射过程中的一些细节。这些细节是Storm的可靠性API的一部分，即Storm如何确保spout发出的每条消息都被充分处理。具体细节以及作为用户如何有效利用Storm的可靠性API可以参考[这里](Guaranteeing-message-processing.html)。

## 事务拓扑

Storm保证每条消息在拓扑中至少被处理一次。一个常见的问题是“如何在Storm中进行计数？如何避免重复计数？”Storm提供了一种叫做事务拓扑的特性用于保证每条消息被且仅被处理一次。关于事务拓扑可以参考[这里](storm-transactional-topologies.html)。

## 分布式RPC

本文只展示了如何在Storm之上进行基本的流处理。借助Storm原语，用户还可以做更多的事情。其中一个最有趣的应用就是分布式RPC，它可以将高计算量的方法调用动态地并行化。详细介绍请参考[分布式 RPC](storm-distributed-rpc.html)。

## 结论

本文简要介绍了如何开发、测试和部署Storm拓扑。其余的文章将更深入地探讨Storm使用过程中的各方面问题。