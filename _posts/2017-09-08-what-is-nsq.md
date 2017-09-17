---
layout: article-detail
category: nsq
author: 龙飞 
title: NSQ是什么
---

这是NSQ代码阅读笔记的第一遍。之前在如何阅读代码一文中提到了阅读代码之前要先了解项目的用途和功能模块划分。本文试图记录这些信息。

## 我们为什么需要NSQ 
我们在开发过程中多次用到了NSQ，所以我认为应该更加深入地了解它。
那么为什么需要用到NSQ呢？可能不同的人有不同的用法。我们使用了它的最主要的两个特性：解耦和缓冲。
比如工作流每执行一步之后发送到调用者的调用状态（成功or失败），转码后的消息回调，以及转码过程中更新完DB后更新Redis的操作，都是用NSQ来完成的。
至于为什么用NSQ而不是Kafka或者RabbitMQ之类，主要是因为我们这边都是go的代码，使用NSQ接入别的如监控等服务比较方便，同时让生态显得比较统一。

如果你使用一个消息队列，你最关心的特性是什么呢？
我们最关注的有这几点：
1. 消息即时性，反映在数据上就是pct99。
2. 消息完备性，即无论生产者发送了多少消息，消费者都能完全读到，不会丢失消息或者其中的某一部分。 
3. 消息可重读，即多个消费者都能读出相同的数据（不特别过滤的条件下） 
4. 容灾性能，即没有单点故障，能快速扩容，便于监控，支持降级，故障中能迅速恢复，不丢失数据。

查看NSQ的主页（http://nsq.io/overview/features_and_guarantees.html），我们能找到官方对于这些要求的描述。
```
Features
• support distributed topologies with no SPOF
• horizontally scalable (no brokers, seamlessly add more nodes to the cluster)
• low-latency push based message delivery (performance)
• combination load-balanced and multicast style message routing
• excel at both streaming (high-throughput) and job oriented (low-throughput) workloads
• primarily in-memory (beyond a high-water mark messages are transparently kept on disk)
• runtime discovery service for consumers to find producers (nsqlookupd)
• transport layer security (TLS)
• data format agnostic
• few dependencies (easy to deploy) and a sane, bounded, default configuration
• simple TCP protocol supporting client libraries in any language
• HTTP interface for stats, admin actions, and producers (no client library needed to publish)
• integrates with statsd for realtime instrumentation
• robust cluster administration interface (nsqadmin)

Guarantees
• messages are not durable (by default)
• messages are delivered at least once
• messages received are un-ordered
• consumers eventually find all topic producers
```
可见NSQ都能很好地满足我们的需求，同时还将稳定性放在了一个很重要的位置。在以后的若干篇文章内，我会根据代码来分析这些特性的实现。

## 你想象中的NSQ实现
一个典型的消息队列如何实现呢？如果你熟悉golang，一定会马上想到channel。它同样是一个生产者+消费者的结构，只要channel有数据，就能一直读取，只要channel未满，就能一直写入。其他情况都会阻塞。
事实上，NSQ最核心的数据结构确实是用channel来实现的。不过，需要增加许多额外的手段来保证上面提到的各种特性。
比如：为了实现多个消费者读到同样的数据，引入了单个topic下包含多个channel（此channel非go中的chan）的结构；为了保证消息不丢失，引入了diskQueue将数据保存在磁盘上；为了保证消息一定能被消费者完全接受，引入了inFlightQueue；为了实现延时消息，引入了deferedQueue。

阅读别人的代码最好从一个写代码的人的角度来思考：你要了解的这个功能是不是最基础的特性，如果不是，它依赖了哪些特性。就好像你在实现功能的时候先要找到别人提供的API一样。遇到看不懂的代码怎么办？先放在一边，了解了最基础的特性（函数，类之类）之后，再加上Google，一般来说就能很轻松地理解了。
## 小结
所以NSQ是什么呢？NSQ就是一个消息队列，它能保证消息迅速传达，能保证所有消息不会丢失(除非磁盘满了），至少被传递（到消费者）一次，它能保证多个消费者都能读到同样的数据，它没有单点故障，能快速从错误中恢复。还有一点，它的代码足够简单，而且很gopher范。
