---
layout: post
category : Storm
title: Trident教程
tagline: ""
tags : [storm]
---
{% include JB/setup %}

Trident是基于Storm的用于实时计算的高层抽象。它使得用户可以无缝集成高吞吐量（每秒数百万条消息）、有状态的流处理以及低延迟的分布式查询。如果用户对诸如Pig或Cascading的高层批处理工具熟悉的话，也会对Trident的概念相当熟悉——Trident提供了连接、聚合、分组、函数和过滤器。除此之外，Trident还增加了用于执行基于任意数据库或其他可持久化存储的有状态的增量处理的原语。Trident具有一致的、恰好一次的语义，所以很容易预测Trident拓扑的行为。

## 范例

接下来看一个Trident的例子。这个例子完成两件事情：

1. 从输入的句子流计算流中单词的次数
2. 实现给定单词列表次数之和的查询

作为示范，本例从如下源中读入无穷句子流：

```java
FixedBatchSpout spout = new FixedBatchSpout(new Fields("sentence"), 3,
               new Values("the cow jumped over the moon"),
               new Values("the man went to the store and bought some candy"),
               new Values("four score and seven years ago"),
               new Values("how many apples can you eat"));
spout.setCycle(true);
```

这个Spout从句子集合中不断循环来产生句子流。下面是计算单词次数的代码：

```java
TridentTopology topology = new TridentTopology();        
TridentState wordCounts =
     topology.newStream("spout1", spout)
       .each(new Fields("sentence"), new Split(), new Fields("word"))
       .groupBy(new Fields("word"))
       .persistentAggregate(new MemoryMapState.Factory(), new Count(), new Fields("count"))                
       .parallelismHint(6);
```

下面来逐行解释代码。首先创建一个TridentTopology对象，它暴露了构建Trident计算的接口。TridentTopology提供了一个名为newStream的方法用于在拓扑中创建从输入源读取的数据流。本例中，输入源即之前定义的FixedBatchSpout。输入源也可以试类似Kestrel或Kafka之类的队列中间件。Trident在Kafka中为每个输入源保存了少量状态记录（关于消费情况的元数据），而"spout1"字符串则指定了Trident应该在Zookeeper的哪个节点下保存元数据。

Trident将流作为小的元组批来处理。例如，输入的句子流可能被划分为如下的批次：

![分批后的流](/images/batched-stream.png)

一般来说，这些批数据的容量约为几千到几百万个元组，具体取决于输入的流量。

Trident提供了一套成熟的批处理API用于处理这些小规模的批数据。这些API和Hadoop的高层抽象如Pig或Cascading提供的很相似：用户可以执行分组、连接、聚合、运行函数、运行过滤器，等等。当然，单独处理每批数据并没有太大意义，所以Trident提供了用于跨批聚合数据并且存储聚合结果的功能，聚合结果可被存储于内存、Memcached、Cassandra或其他存储中间件。最后，Trident提供了用于查询实时状态源的一级函数。状态可以被Trident更新（如本例），也可以是一个独立的状态源。

回到这个例子，Spout发射了一个包含单个域"sentence"的流。拓扑代码的下一行为流中的每个元组应用了Split函数，它提取"sentence"域并将其分割成单词。每个句子元组生成了多个单词元组——例如，句子"the cow jumped over the moon"生成了6个单词元组。下面是Split的定义：

```java
public class Split extends BaseFunction {
   public void execute(TridentTuple tuple, TridentCollector collector) {
       String sentence = tuple.getString(0);
       for(String word: sentence.split(" ")) {
           collector.emit(new Values(word));
       }
   }
}
```

可见其实现非常简单，只是接收句子、按空格分割然后为每个单词发射一个元组。

拓扑代码的其他部分计算单词次数并持久化。首先，流按照"word"域进行分组。然后，每个组使用Count聚合器进行持久化聚合。persistentAggregate函数知道如何存储并更新状态源的聚合结果。本例中，单词计数结果位于内存中，但是这可以很轻松地迁移到Memcached、Cassandra或者其他持久化存储中。例如，要使用Memcached来存储计数结果只需使用如下代码（使用了[trident-memcached](https://github.com/nathanmarz/trident-memcached)），其中"serverLocations"是Memcached集群的主机/端口列表：

```java
.persistentAggregate(MemcachedState.transactional(serverLocations), new Count(), new Fields("count"))        
MemcachedState.transactional()
```

persistentAggregate所存储的值表示流所发射的所有批的聚合结果。

Trident最酷的一点是它具有完全容错、恰好处理一次的语义。这对推导实时处理的结果很有用。Trident保存状态的方式保证了即使发生了错误需要重试，也不会为同一个源数据进行多次更新操作。

persistentAggregate方法将Stream转化为TridentState对象。本例中TridentState对象代表了所有单词出现的次数。下面将使用这个TridentState对象来实现分布式查询的功能。

拓扑的下一部分实现了一个低延迟的分布式查询单词出现次数的模块。该模块接受空格分隔的单词列表，返回其中所有单词出现次数之和。这些查询的执行方式和普通的RPC调用类似，只是它们在后台是并行的。下面是一个调用示例：

```java
DRPCClient client = new DRPCClient("drpc.server.location", 3772);
System.out.println(client.execute("words", "cat dog the man");
// prints the JSON-encoded result, e.g.: "[[5078]]"
```

如上所示，除了在Storm集群中被并行执行外，此查询和普通的RPC调用并无区别。上例中所示的小规模查询的延迟一般在10ms左右。更密集的DRPC查询可能需要更长时间，不过这也取决于用户为计算程序分配了多少资源。

分布式查询具体实现如下：

```java
topology.newDRPCStream("words")
       .each(new Fields("args"), new Split(), new Fields("word"))
       .groupBy(new Fields("word"))
       .stateQuery(wordCounts, new Fields("word"), new MapGet(), new Fields("count"))
       .each(new Fields("count"), new FilterNull())
       .aggregate(new Fields("count"), new Sum(), new Fields("sum"));
```

The same TridentTopology object is used to create the DRPC stream, and the function is named "words". The function name corresponds to the function name given in the first argument of execute when using a DRPCClient.

Each DRPC request is treated as its own little batch processing job that takes as input a single tuple representing the request. The tuple contains one field called "args" that contains the argument provided by the client. In this case, the argument is a whitespace separated list of words.

First, the Split function is used to split the arguments for the request into its constituent words. The stream is grouped by "word", and the stateQuery operator is used to query the TridentState object that the first part of the topology generated. stateQuery takes in a source of state – in this case, the word counts computed by the other portion of the topology – and a function for querying that state. In this case, the MapGet function is invoked, which gets the count for each word. Since the DRPC stream is grouped the exact same way as the TridentState was (by the "word" field), each word query is routed to the exact partition of the TridentState object that manages updates for that word.

Next, words that didn't have a count are filtered out via the FilterNull filter and the counts are summed using the Sum aggregator to get the result. Then, Trident automatically sends the result back to the waiting client.

Trident is intelligent about how it executes a topology to maximize performance. There's two interesting things happening automatically in this topology:

1. Operations that read from or write to state (like persistentAggregate and stateQuery) automatically batch operations to that state. So if there's 20 updates that need to be made to the database for the current batch of processing, rather than do 20 read requests and 20 writes requests to the database, Trident will automatically batch up the reads and writes, doing only 1 read request and 1 write request (and in many cases, you can use caching in your State implementation to eliminate the read request). So you get the best of both words of convenience – being able to express your computation in terms of what should be done with each tuple – and performance.
2. Trident aggregators are heavily optimized. Rather than transfer all tuples for a group to the same machine and then run the aggregator, Trident will do partial aggregations when possible before sending tuples over the network. For example, the Count aggregator computes the count on each partition, sends the partial count over the network, and then sums together all the partial counts to get the total count. This technique is similar to the use of combiners in MapReduce.

Let's look at another example of Trident.

## Reach

The next example is a pure DRPC topology that computes the reach of a URL on demand. Reach is the number of unique people exposed to a URL on Twitter. To compute reach, you need to fetch all the people who ever tweeted a URL, fetch all the followers of all those people, unique that set of followers, and that count that uniqued set. Computing reach is too intense for a single machine – it can require thousands of database calls and tens of millions of tuples. With Storm and Trident, you can parallelize the computation of each step across a cluster.

This topology will read from two sources of state. One database maps URLs to a list of people who tweeted that URL. The other database maps a person to a list of followers for that person. The topology definition looks like this:

```java
TridentState urlToTweeters =
       topology.newStaticState(getUrlToTweetersState());
TridentState tweetersToFollowers =
       topology.newStaticState(getTweeterToFollowersState());

topology.newDRPCStream("reach")
       .stateQuery(urlToTweeters, new Fields("args"), new MapGet(), new Fields("tweeters"))
       .each(new Fields("tweeters"), new ExpandList(), new Fields("tweeter"))
       .shuffle()
       .stateQuery(tweetersToFollowers, new Fields("tweeter"), new MapGet(), new Fields("followers"))
       .parallelismHint(200)
       .each(new Fields("followers"), new ExpandList(), new Fields("follower"))
       .groupBy(new Fields("follower"))
       .aggregate(new One(), new Fields("one"))
       .parallelismHint(20)
       .aggregate(new Count(), new Fields("reach"));
```

The topology creates TridentState objects representing each external database using the newStaticState method. These can then be queried in the topology. Like all sources of state, queries to these databases will be automatically batched for maximum efficiency.

The topology definition is straightforward – it's just a simple batch processing job. First, the urlToTweeters database is queried to get the list of people who tweeted the URL for this request. That returns a list, so the ExpandList function is invoked to create a tuple for each tweeter.

Next, the followers for each tweeter must be fetched. It's important that this step be parallelized, so shuffle is invoked to evenly distribute the tweeters among all workers for the topology. Then, the followers database is queried to get the list of followers for each tweeter. You can see that this portion of the topology is given a large parallelism since this is the most intense portion of the computation.

Next, the set of followers is uniqued and counted. This is done in two steps. First a "group by" is done on the batch by "follower", running the "One" aggregator on each group. The "One" aggregator simply emits a single tuple containing the number one for each group. Then, the ones are summed together to get the unique count of the followers set. Here's the definition of the "One" aggregator:

```java
public class One implements CombinerAggregator<Integer> {
   public Integer init(TridentTuple tuple) {
       return 1;
   }

   public Integer combine(Integer val1, Integer val2) {
       return 1;
   }

   public Integer zero() {
       return 1;
   }        
}
```

This is a "combiner aggregator", which knows how to do partial aggregations before transferring tuples over the network to maximize efficiency. Sum is also defined as a combiner aggregator, so the global sum done at the end of the topology will be very efficient.

Let's now look at Trident in more detail.

## Fields and tuples

The Trident data model is the TridentTuple which is a named list of values. During a topology, tuples are incrementally built up through a sequence of operations. Operations generally take in a set of input fields and emit a set of "function fields". The input fields are used to select a subset of the tuple as input to the operation, while the "function fields" name the fields the operation emits.

Consider this example. Suppose you have a stream called "stream" that contains the fields "x", "y", and "z". To run a filter MyFilter that takes in "y" as input, you would say:

```java
stream.each(new Fields("y"), new MyFilter())
```

Suppose the implementation of MyFilter is this:

```java
public class MyFilter extends BaseFilter {
   public boolean isKeep(TridentTuple tuple) {
       return tuple.getInteger(0) < 10;
   }
}
```

This will keep all tuples whose "y" field is less than 10. The TridentTuple given as input to MyFilter will only contain the "y" field. Note that Trident is able to project a subset of a tuple extremely efficiently when selecting the input fields: the projection is essentially free.

Let's now look at how "function fields" work. Suppose you had this function:

```java
public class AddAndMultiply extends BaseFunction {
   public void execute(TridentTuple tuple, TridentCollector collector) {
       int i1 = tuple.getInteger(0);
       int i2 = tuple.getInteger(1);
       collector.emit(new Values(i1 + i2, i1 * i2));
   }
}
```

This function takes two numbers as input and emits two new values: the addition of the numbers and the multiplication of the numbers. Suppose you had a stream with the fields "x", "y", and "z". You would use this function like this:

```java
stream.each(new Fields("x", "y"), new AddAndMultiply(), new Fields("added", "multiplied"));
```

The output of functions is additive: the fields are added to the input tuple. So the output of this each call would contain tuples with the five fields "x", "y", "z", "added", and "multiplied". "added" corresponds to the first value emitted by AddAndMultiply, while "multiplied" corresponds to the second value.

With aggregators, on the other hand, the function fields replace the input tuples. So if you had a stream containing the fields "val1" and "val2", and you did this:

```java
stream.aggregate(new Fields("val2"), new Sum(), new Fields("sum"))
```

The output stream would only contain a single tuple with a single field called "sum", representing the sum of all "val2" fields in that batch.

With grouped streams, the output will contain the grouping fields followed by the fields emitted by the aggregator. For example:

```java
stream.groupBy(new Fields("val1"))
     .aggregate(new Fields("val2"), new Sum(), new Fields("sum"))
```

In this example, the output will contain the fields "val1" and "sum".

## 状态

实时计算的一个关键问题是如何处理状态以确保在发生错误需要重试时更新操作是幂等的。错误是不可能被避免的，所以当节点失败或发生其他错误时，批数据需要被重新处理。问题是，如何进行状态更新（不管是在外部数据库还是拓扑内部）才能使得看起来每条消息恰好只被处理了一次？

这个复杂的问题可用如下例子来说明。假设用户正在进行计数聚合并且希望将计数值存储在数据库中。如果数据库中只存储了计数值，那么当需要更新计数值时，将无法得知此前是否已经进行了更新。批数据可能之前已经进行了处理并成功更新了数据库中的值，只是在稍后的其他步骤中失败了。当然也可能根本没有成功更新数据库中的值。用户此时无法确定具体是哪一种情况。

Trident通过下面两步解决了这个问题：

1. 每一批数据都被指定了一个叫做"事务id"的唯一id。如果批数据被重试，事务id不变。
2. 批数据的状态更新是有序的。即，第2批被更新完毕才会进行第3批的更新。

使用这两个原语，用户即可在自定义的状态处理中实现恰好一次的语义。与之前只在数据库中存储计数值不同，用户可以将事务id和计数值一并作为一个原子值存入数据库。这样，在更新计数值时，只需要将当前批的事务id和数据库中的事务id进行比较即可。如果是相同的，则跳过更新操作——因为根据有序原语，可以确定数据库中的计数值对应于当前的批数据。如果不同，则可以增加计数值。

当然，用户并不需要在拓扑中手动实现此逻辑。这部分内容被包含在State的抽象类中以自动完成。用户自定义的State对象也不需要实现事务id。如果用户不希望在数据库中存储id，也可以不存储。这时State具有在发生错误时保证至少处理一次的语义（对用户应用来说应该没有什么问题）。关于如何实现State对象以及权衡各种容错措施，请参阅[文档](/documentation/Trident-state)。

A State is allowed to use whatever strategy it wants to store state. So it could store state in an external database or it could keep the state in-memory but backed by HDFS (like how HBase works). State's are not required to hold onto state forever. For example, you could have an in-memory State implementation that only keeps the last X hours of data available and drops anything older. Take a look at the implementation of the [Memcached integration](https://github.com/nathanmarz/trident-memcached/blob/master/src/jvm/trident/memcached/MemcachedState.java) for an example State implementation.

## Trident拓扑的执行

Trident拓扑会尽可能高效地编译为Storm拓扑。只有在需要对数据重新分区的时候（如执行groupBy或shuffle操作），元组才会被在整个网络上发送。如果Trident拓扑如下：

![Trident编译为Storm 1](/images/trident-to-storm1.png)

它将被编译为如下所示的Storm spout/bolt：

![Trident编译为Storm 2](/images/trident-to-storm2.png)

## 总结

Trident使得实时计算更加优雅。本文已经展示了如何利用Trident API来无缝组合高吞吐量的流处理、状态控制和低延迟的查询。Trident可以使用户更加自然地表达实时计算的逻辑，同时获得最佳的性能。