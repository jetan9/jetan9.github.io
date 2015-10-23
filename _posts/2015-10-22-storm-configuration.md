---
layout: post
category : Storm
title: Storm配置
tagline: ""
tags : [storm]
---
{% include JB/setup %}

Storm提供了很多配置项 用于调整Nimbus、Supervisor和拓扑的行为。一些配置项是系统级的，无法在拓扑层面进行修改，而另一些配置项可以为每个拓扑进行设置。

每个配置项都在Storm代码的[defaults.yaml](https://github.com/apache/storm/blob/master/conf/defaults.yaml)中定义了默认值。用户可以通过在Nimbus和Supervisor的类路径中定义storm.yaml文件来覆盖默认值。拓扑级别的配置则需要在使用[StormSubmitter](/javadoc/apidocs/backtype/storm/StormSubmitter.html)提交拓扑时进行设置。拓扑级别的配置只能覆盖以"TOPOLOGY"开头的配置项。

Storm 0.7.0及后续版本可以在Bolt/Spout级别覆盖配置值。可设置的配置只有如下几种：

1. "topology.debug"
2. "topology.max.spout.pending"
3. "topology.max.task.parallelism"
4. "topology.kryo.register"：和其他配置项略有不同，因为序列化可被拓扑中的所有节点使用。细节请参考[序列化](storm-serialization.html)。

Java API允许用户以两种方式设置配置Bolt/Spout级别的配置：

1. *内部方式：* 在任意Spout或Bolt中重写`getComponentConfiguration`方法并返回该节点特定的配置map。
2. *外部方式：* `TopologyBuilder`的`setSpout`和`setBolt`方法返回一个带`addConfiguration`方法的对象，`addConfigurations`方法可以用来覆盖节点的配置。

配置值的优先级为defaults.yaml < storm.yaml < 拓扑级别的配置 < 节点内部配置 < 节点外部配置。 

**资源：**

* [Config](/javadoc/apidocs/backtype/storm/Config.html)：所有配置项及配置拓扑级别配置项的工具类。
* [defaults.yaml](https://github.com/apache/storm/blob/master/conf/defaults.yaml)：定义了所有配置项的默认值。
* [建立Storm集群](storm-setting-up-a-storm-cluster.html)：说明了如何创建和配置Storm集群。
* [在生产集群中运行拓扑](storm-running-topologies-on-a-production-cluster.html)：列出了在集群中运行拓扑可用的配置项。
* [本地模式](storm-local-mode.html)：列出了本地模式可用的配置项。