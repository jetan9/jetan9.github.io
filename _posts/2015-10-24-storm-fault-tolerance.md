---
layout: post
category : Storm
title: Storm容错性
tagline: ""
tags : [storm]
---
{% include JB/setup %}

本文介绍Storm是如何被设计为一个容错的系统的。

## 工作进程异常终止时会发生什么？

工作进程异常终止时，supervisor会重启它。如果它反复启动失败从而无法向Nimbus发送心跳消息，则Nimbus会将工作进程分配到另一台机器。

## 节点异常终止时会发生什么？

分配到这台机器的所有任务都会超时，Nimbus会将这些任务分配到其他机器。

## Nimbus或Supervisor守护进程异常终止时会发生什么？

Nimbus和Supervisor守护进程被设计为速错的（遇到错误情况时进程自动退出）和无状态的（所有状态保存在Zookeeper或本地磁盘中）。如[建立Storm集群](Setting-up-a-Storm-cluster.html)一文所述，Nimbus和Supervisor守护进程需要在类似daemontools或monit的工具监控之下运行。这样一旦它们异常终止，可以立即重启，就像从未发生过异常一样。

值得注意的是，工作进程并不会收到Nimbus或Supervisor异常终止的影响。相反，如果Hadoop的JobTracker异常终止，所有运行中的任务都将丢失。

## Nimbus会单点失败吗？

如果Nimbus节点丢失，工作进程仍将继续工作。而且，supervisor也可以在工作进程失败之后继续重启它们。但是，如果没有Nimbus，工作进程将无法在必要时（如工作机器挂掉）被重新分配到其他机器。

所以Nimbus会引起“某种”单点失败。实际上，由于Nimbus守护进程异常终止并不会造成灾难性的后果，所以这并不是很严重的问题。未来Storm将实现Nimbus的高可用。

## Storm如何确保消息被处理？

Storm提供了即使在节点机器挂掉或消息丢失的情形下也能保证消息被处理的机制。具体请参考[强制消息处理](storm-guaranteeing-message-processing.html)。