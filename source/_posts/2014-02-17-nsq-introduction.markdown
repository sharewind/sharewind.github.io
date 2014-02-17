---
layout: post
title: "nsq introduction"
date: 2014-02-17 22:24
comments: true
categories: NSQ golang
---

NSQ

a realtime distributed messaging platform  
一个实时的分布式消息平台

####Introduction 介绍
NSQ is a realtime distributed messaging platform designed to operate at bitly’s scale, handling billions of messages per day (current peak of 90k+ messages per second).  
NSQ是一个实时的分布式消息平台，被设计用来承载 bitly 处理每天数十亿的消息（每秒90k+的峰值）。

It promotes distributed and decentralized topologies without single points of failure, enabling fault tolerance and high availability coupled with a reliable message delivery guarantee. See features & guarantees.  
它提升了分布式和分散式拓扑结构，避免了单点故障。通过容错性和高可用性来保障消息的传达。请参阅特性和保证。

Operationally, NSQ is easy to configure and deploy (all parameters are specified on the command line and compiled binaries have no runtime dependencies). For maximum flexibility, it is agnostic to data format (messages can be JSON, MsgPack, Protocol Buffers, or anything else). Official Go and Python libraries are available out of the box and, if you’re interested in building your own client, there’s a protocol spec (see client libraries).  
使用上，NSQ易于配置和部署（所有参数都指定在命令行和编译的二进制文件没有运行时依赖）。为了获得最大的灵活性，它是不可知的数据格式（消息可以是JSON，MsgPack，Protocol Buffers，或其他任何东西）。官方的go和Python库可开箱即用，如果你有兴趣建立自己的客户端，这里有一个协议规范（参见客户端库）。



####Features & Guarantees  特性与保证

Features特性    

- support distributed topologies with no SPOF  
 支持分布式拓扑，没有单点故障  
- horizontally scalable (no brokers, seamlessly add more nodes to the cluster)  
横向拓展（没有brokers, 无缝地往集群里添加更多节点）  
- low-latency push based message delivery (performance)  
基于推送的低延迟消息传递（性能）  
- combination load-balanced and multicast style message routing  
结合负载平衡和组播方式的消息路由  
- excel at both streaming (high-throughput) and job oriented (low-throughput) workloads  
同时擅长于流（高流量）与工作导向（低流量）工作负载  
- primarily in-memory (beyond a high-water mark messages are transparently kept on disk)  
主要是在内存中（超过上限的消息将被透明地保存在磁盘上）  
- runtime discovery service for consumers to find producers (nsqlookupd)   
运行发现服务，为消费者找到生产者（nsqlookupd）  
- transport layer security (TLS)   
传输层安全协议（TLS）  
- data format agnostic    
数据格式无关  
- few dependencies (easy to deploy) and a sane, bounded, default configuration    
没有依赖（易于部署），附有一个健全的，限定的默认配置  
- simple TCP protocol supporting client libraries in any language  
支持任何语言客户端库的简单TCP协议  
- HTTP interface for stats, admin actions, and producers (no client library needed to publish)   
HTTP接口进行统计，管理操作和生产（无需客户端库发布）  
- integrates with statsd for realtime instrumentation    
集成了statsd用于实时监控  
- robust cluster administration interface (nsqadmin)  
强大的集群管理界面（nsqadmin）  

####Guarantees保证

As with any distributed system, achieving your goal is a matter of making intelligent tradeoffs. By being transparent about the reality of these tradeoffs we hope to set expectations about how NSQ will behave when deployed in production.  
正如任何分布式系统，你的目标是实现智能权衡。透过高透明度有关这些权衡，我们希望设置NSQ在生产环境部署的的预期行为。

messages are not durable (by default)  
默认情况下消息不进行持久化。

Although the system supports a “release valve” (--mem-queue-size) after which messages will be transparently kept on disk, it is primarily an in-memory messaging platform.  
虽然该系统支持超过“阈值”（--mem-queue-size）的值之后，信息将被透明地保存在磁盘上，但它主要是一个内存中的消息传递平台。

--mem-queue-size can be set to 0 to to ensure that all incoming messages are persisted to disk. In this case, if a node failed, you are susceptible to a reduced failure surface (i.e. did the OS or underlying IO subsystem fail).  
--mem-queue-size值可以设置为0，以确保所有传入的消息被保存到磁盘上。在这种情况下，如果一个节点失败，则很容易减小遭受的损失（即没操作系统或底层IO子系统失败）。

There is no built in replication. However, there are a variety of ways this tradeoff is managed such as deployment topology and techniques which actively slave and persist topics to disk in a fault-tolerant fashion.  
没有内置的复制。但是，也有各种折衷的管理方式，诸如配置从节点，并把topics持久化到硬盘上的部署与拓扑方式。

messages are delivered at least once  
消息至少传递一次  

Closely related to above, this assumes that the given nsqd node does not fail.  
与上述密切相关的是，这里假设给定nsqd节点不会失败。

This means, for a variety of reasons, messages can be delivered multiple times (client timeouts, disconnections, requeues, etc.). It is the client’s responsibility to perform idempotent operations or de-dupe.  
这意味着，由于各种原因，消息可能会被传送多次（客户端超时，断线，对其重新排队，等等）。客户端承担起履行幂等操作或重复数据删除的责任。

messages received are un-ordered  
消息的接收是无序的  

You cannot rely on the order of messages being delivered to consumers.  
你不能依赖于消息传递到消费者的顺序。  

Similar to message delivery semantics, this is the result of requeues, the combination of in-memory and on disk storage, and the fact that each nsqd node shares nothing.  
类似的消息传递语义，这是对其重新排队的结果，结合了内存和磁盘上的存储，每个nsqd节点间不共享任何内容。

It is relatively straightforward to achieve loose ordering (i.e. for a given consumer its messages are ordered but not across the cluster as a whole) by introducing a window of latency in your consumer to accept messages and order them before processing (although, in order to preserve this invariant one must drop messages falling outside that window).  
这里有一个比较简单的实现松散的顺序（即对于一个给定的消费者它的消息是有序的，但不是以整个集群作为一个整体）通过引入一个延迟窗口，在您的消费者接受信息，并在处理之前排序（虽然，为了保持顺序不变必须抛弃在窗口之外的消息）。

consumers eventually find all topic producers  
消费者最终会发现所有topic的生产者。  

The discovery service (nsqlookupd) is designed to be eventually consistent. nsqlookupd nodes do not coordinate to maintain state or answer queries.  
发现服务（nsqlookupd）被设计为最终一致。nsqlookupd节点并不需要互相协调来保持状态或回复查询。

Network partitions do not affect availability in the sense that both sides of the partition can still answer queries. Deployment topology has the most significant effect of mitigating these types of issues.  
网络分区不影响可用性，分区双方仍然可以回复查询。部署拓扑结构能在很大程度减轻这类问题。


###FAQ
####Deployment部署

What is the recommended topology for nsqd?  
推荐的nsqd拓扑结构是什么样的？

We strongly recommend running an nsqd alongside any service(s) that produce messages.  
我们强烈建议nsqd与任何产生消息的服务一起运行。  

nsqd is a relatively lightweight process with a bounded memory footprint, which makes it well suited to “playing nice with others”.  
nsqd是一个占用有限内存的相对轻量级进程，这使得它非常适合于“playing nice with others”。

This pattern aids in structuring message flow as a consumption problem rather than a production one.
这种模式构建消息流有利于消费问题，而不是一个生产问题。

Another benefit is that it essentially forms an independent, sharded, silo of data for that topic on a given host.
另一个好处是，它本质上形成一个独立，分片，筒仓的数据对于一个给定主机上的topic。

NOTE: this isn’t an absolute requirement though, it’s just simpler (see question below).  
注意：这不是一个绝对的要求，虽然，它只是简单的（见下面的问题）。

Why can’t nsqlookupd be used by producers to find where to publish to?  
为什么不能nsqlookupd使用由生产者来找到在哪里发布到？

NSQ promotes a consumer-side discovery model that alleviates the upfront configuration burden of having to tell consumers where to find the topic(s) they need.  
NSQ提供了消费者发现模型，降低了消息者找到所需 topic 之前的前期配置负担。  

However, it does not provide any means to solve the problem of where a service should publish to. This is a chicken and egg problem, the topic does not exist prior to the first publish.  
然而，它不提供任何手段来解决一个服务应发布到哪的问题。这是一个鸡和蛋的问题，topic 不会在第一个发布之前存在。

By co-locating nsqd (see question above), you sidestep this problem entirely (your service simply publishes to the local nsqd) and allow NSQ’s runtime discovery system to work naturally.  
通过协同定位nsqd（见上面的问题），你完全回避这个问题（您的服务只是发布到本地nsqd），并允许NSQ运行时发现系统自然工作。

I just want to use nsqd as a work queue on a single node, is that a suitable use case?
我只是想用nsqd作为单个节点的工作队列，是一个合适的用例？  

Yep, nsqd can run standalone just fine.  
是的，独立运行nsqd就好了。

nsqlookupd is beneficial in larger distributed environments.  
nsqlookupd 有利于在更大的分布式环境。

How many nsqlookupd should I run?  
我需要启动多少个nsqlookupd？

Typically only a few depending on your cluster size, number of nsqd nodes and consumers, and your desired fault tolerance.  
通常只要几个，取决于你集群大小，nsqd节点和消费者的数目，以及您需要的容错性。

3 or 5 works really well for deployments involving up to several hundred hosts and thousands of consumers.  
3或5个就能够为部署上百的主机与数千的消费者工作很好。

nsqlookupd nodes do not require coordination to answer queries. The metadata in the cluster is eventually consistent.  
nsqlookupd节点并没有需要协调来回答查询。集群中的元数据是最终一致。

####Publishing发布消息

Do I need a client library to publish messages?  
是否需要一个客户端库发布消息？

NO! Just use the HTTP endpoints for publishing (/pub and /mpub). It’s simple, it’s easy, and it’s ubiquitous in almost any programming environment.

In fact, the overwhelming majority of NSQ deployments use HTTP to publish.  
事实上，绝大多数NSQ部署使用HTTP进行发布。  

Why force a client to handle responses to the TCP protocol’s PUB and MPUB commands?  
为什么要强制客户端处理响应TCP协议的PUB和MPUB命令？

We believe NSQ’s default mode of operation should prioritize safety and we wanted the protocol to be simple and consistent.
我们认为NSQ的默认模式应优先考虑安全性，我们希望协议是简单和一致。

When can a PUB or MPUB fail?  
何时PUT 或 MPU 会失败？

1. The topic name is not formatted correctly (to character/length restrictions). See the topic and channel name spec.  
topic名称格式不正确（以字符/长度的限制）。请参阅topic和channel名称规范。  
2. The message is too large (this limit is exposed as a parameter to nsqd).  
该消息是太大（这个限制是作为参数传递给nsqd）  
3. The topic is in the middle of being deleted.  
主题是在被删除中。  
4. nsqd is in the middle of cleanly exiting.  
nsqd正在退出中。  
5. Any client connection-related failures during the publish.   
发布时任何客户端连接相关的故障。  

(1) and (2) should be considered programming errors. (3) and (4) are rare and (5) is a natural part of any TCP based protocol.  
（1）和（2）应当考虑的编程错误。（3）和（4）是罕见的，（5）是任何基于TCP协议的一个自然组成部分。  

How can I mitigate scenario (3) above?  
我怎样才能减轻上述场景（3）？  

Deleting topics is a relatively infrequent operation. If you need to delete a topic, orchestrate the timing such that publishes eliciting topic creations will never be performed until a sufficient amount of time has elapsed since deletion.  
删除topic是一个相对较少的操作。如果你需要删除一个主题，编排，使得publish引出话题的作品将永远，直到时间足够量的自删除已经过去执行的时机。

####Design and Theory设计与理念

How do you recommend naming topics and channels?  
你推荐如何命令topics和channels?  

A topic name should describe the data in the stream.  
一个 topic name 应该描述流里的数据。  

A channel name should describe the work performed by its consumers.  
一个channel name 应该描述消费者执行的工作。

For example, good topic names are encodes, decodes, api_requests, page_views and good channel names are archive, analytics_increment, spam_analysis.

Are there any limitations to the number of topics and channels a single nsqd can support?  
是否有单个nsqd结点上支持topics和channels的数量上限？

There are no built-in limits imposed. It is only limited by the memory and CPU of the host nsqd is running on (per-client CPU usage was greatly reduced in issue #236).
有没有内置强加的限制。它仅受限于运行nsqd 的主机内存和CPU（每客户端CPU占用率大大降低在问题＃236）。

How are new topics announced to the cluster?
如何新topic发布到集群？

The first PUB or SUB to a topic will create the topic on an nsqd. Topic metadata will then propagate to the configured nsqlookupd. Other readers will discover this topic by periodically querying the nsqlookupd.  
在topic 上的第一个 PUB或SUB 将在nsqd上创建 topic。然后topic的元数据将传播到配置nsqlookupd。其它readers 将通过定期查询nsqlookupd发现这个topic。

Can NSQ do RPC?  
NSQ可以用来做RPC？

Yes, it’s possible, but NSQ was not designed with this use case in mind.
是的，这是可以的，但NSQ最初并没有设计这个用例。

We intend to publish some docs on how this could be structured but in the meantime reach out if you’re interested.





