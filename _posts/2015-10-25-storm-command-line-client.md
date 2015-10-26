---
layout: post
category : Storm
title: Storm命令行客户端
tagline: ""
tags : [storm]
---
{% include JB/setup %}

本文介绍"storm"命令行客户端的所有命令。要了解如何设置"storm"客户端来和远程集群通信，请参考[设置开发环境](setting-up-a-development-environment.html)中的指令。

这些命令包括：

1. jar
1. kill
1. activate
1. deactivate
1. rebalance
1. repl
1. classpath
1. localconfvalue
1. remoteconfvalue
1. nimbus
1. supervisor
1. ui
1. drpc

### jar

语法：`storm jar topology-jar-path class ...`

使用指定参数运行`class`的main方法。`~/.storm`路径下的Storm相关jar包和配置文件会被加入类路径。进程会进行设置，这样[StormSubmitter](/javadoc/apidocs/backtype/storm/StormSubmitter.html)在提交拓扑时会上传`topology-jar-path`下的jar包。

### kill

语法：`storm kill topology-name [-w wait-time-secs]`

终止名为`topology-name`的拓扑。Storm首先停用拓扑的Spout并等待拓扑的超时时间，以便所有处理中的消息有机会被处理完毕。然后Storm会终止工作线程并清理状态。用户可以通过-w参数设置Storm在停用Spout和终止工作线程之间的等待时间。

### activate

语法：`storm activate topology-name`

激活指定的拓扑的Spout。

### deactivate

语法：`storm deactivate topology-name`

停用指定的拓扑的Spout。

### rebalance

语法：`storm rebalance topology-name [-w wait-time-secs]`

有时用户可能希望迁移运行拓扑的工作进程。例如，用户的集群有10个节点，每个节点上有4个工作进程，这时用户向集群新增了10个节点，并且希望工作进程能够迁移到新的节点上，从而每个节点只有2个工作进程。一种方法是终止进程然后重新提交，但是Storm提供了"rebalance"命令来更轻松地完成这个任务。

rebalance首先指定消息超时时间（使用-w参数）并停用拓扑，然后将工作进程平均分配到集群中的节点上。拓扑之后将会恢复到之前的状态（停用的拓扑仍然停用，激活的拓扑将自动激活）。

### repl

语法：`storm repl`

使用Storm类路径内的jar包和配置打开一个Clojure REPL。调试时非常有用。

### classpath

语法：`storm classpath`

打印Storm客户端运行命令时使用的类路径。

### localconfvalue

语法：`storm localconfvalue conf-name`

打印本地Storm配置中名为`conf-name`的配置项的值。本地Storm配置为`~/.storm/storm.yaml`和`defaults.yaml`合并之后的结果。

### remoteconfvalue

语法：`storm remoteconfvalue conf-name`

打印远程Storm配置中名为`conf-name`的配置项的值。远程Storm配置为`$STORM-PATH/conf/storm.yaml`和`defaults.yaml`合并之后的结果。此命令必须在集群中的机器上运行。

### nimbus

语法：`storm nimbus`

启动nimbus守护进程。此命令应在类似[daemontools](http://cr.yp.to/daemontools.html)或[monit](http://mmonit.com/monit/)的工具的监控之下运行。详情请参考[建立Storm集群](Setting-up-a-Storm-cluster.html)。

### supervisor

语法：`storm supervisor`

启动supervisor守护进程。此命令应在类似[daemontools](http://cr.yp.to/daemontools.html)或[monit](http://mmonit.com/monit/)的工具的监控之下运行。详情请参考[建立Storm集群](Setting-up-a-Storm-cluster.html)。

### ui

语法：`storm ui`

启动UI守护进程。UI提供了一个Web界面用户展示运行中的拓扑的各种详细信息。此命令应在类似[daemontools](http://cr.yp.to/daemontools.html)或[monit](http://mmonit.com/monit/)的工具的监控之下运行。详情请参考[建立Storm集群](Setting-up-a-Storm-cluster.html)。

### drpc

语法：`storm drpc`

启动DRPC守护进程。此命令应在类似[daemontools](http://cr.yp.to/daemontools.html)或[monit](http://mmonit.com/monit/)的工具的监控之下运行。详情请参考[建立Storm集群](Setting-up-a-Storm-cluster.html)。
