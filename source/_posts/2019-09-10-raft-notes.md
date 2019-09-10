---
title: Raft 协议笔记
toc: true
excerpt:  学习学习
categories: 技术
date: 2019-09-10 14:51:05
tags:  [论文,分布式]
---

关注于 Raft 可能更多是因为 kubernetes/etcd 的原因，二者似乎变成了 raft 最大的`客户`，时不时地重新理解一下这个协议还是很有必要的。

之所以这样讲，是因为分布式协议本身就很难理解。即使 raft 论文中一再声称它的主要目的之一便是容易理解，比 paxos 更好懂，但对于一般从业者来说，要想完整地理解它还是有困难的。所以经常需要温故而知新。并且，在网上一搜，关于 raft 协议的 blog 也是汗牛充栋，每个人都在试着理解，记录。可能后续还需要参考下别人的笔记，互相印证。

> 论文地址: [In Search of an Understandable Consensus Algorithm](https://raft.github.io/raft.pdf)

## 基本知识

`leader` 的概念是核心之一。raft 作为一个分布式系统，各个 server 通过固定的算法选举出 leader , 然后 leader 负责状态的维护，处理 client 的请求，维护整个集群的状态等等。跟民主制度里面多党选举有点像， 一旦大多数投票通过，总统就拥有至高权力。



除了 leader 之外，还有另外两个角色是 `follower`以及`candidate`。选举状态下，大家都是`candidate`,正常工作情况下，就分成了`leader`和`follower`。



Raft 并没有使用系统时钟来作为同步机制，因为太不可靠了，而且维护起来很麻烦。raft 使用了一个 `term` 的概念来代表时序。



## Leader election

一般的流程如下所示:

* server 启动时，成为 follower
* 如果 follower 在 election timeout 的时间内，没有收到 leader 发来的 heartbeat， 那么它就假定没有 leader, 准备开始选举
* follower 将自己变为 candidate， 然后给自己投一票， 然后发信息给其他 server 问其他 server 是否可以给自己投票。结果要么是大多数投了它，它成为 leader, 或者另外一个 candidate 成为了 leader, 或者没有 leader (splite vote,没有 server 获取到大多数 vote)，那么就进行一下次投票。
* 在 election 的过程中，有可能其他的 candidate 已经成为了 leader, 并给当前的 candidate 发来了同步数据的请求。这时候就可以通过 term 的比较来判定，如果发过来的 term 数不小于自己的，那么就承认发送者的 leader 地位，并且把自己切换回 follower 状态。否则的话拒绝请求并且继续选举。

这里面有一个问题，所有人都在给所有人发请求要求 vote 自己，怎么能达到有一个 candidate 获取到 大多数 vote 呢? raft 采用了 randomized election timeout 来解决这个问题。不同 server 的 lection timout 不一样，避免了大部分情况下的 splite vote 情况。如果发生 splite vote, 重来一次。这种机制跟 TCP 的拥塞控制有些类似。



## Log replication

log replication 有几个概念，如下图所示:

![](/images/raft/log.png)

Entry 代表了一次操作，一条记录，每个 term 可能包含一条或者多条 entry, entry分为 commited entries 和 非 commited entries.  commited entries 是 leader 确定那些可以提交到 state machines 里的 entries。这些 entries 能确保一直可用(比如 leader crash)，并且被大部分的 server 同步。Entry 有个前置性推断， commited entries 之前的 entries 也是 commited entries。


可以理解， raft 系统的目的就是尽量保证所有的 server 有相同的 entry 序列。这中间要经历选主，leader crash, follower crash ,各种 timeout 等等。在 raft 中，如果 follower 与 leader 的 entries 有不一致的地方，会用 leader 的来覆盖 follower 的。而 leader 自己的 log, 不会覆盖也不会删除。这个逻辑是挺简单的，但实现起来略微繁琐。 leader 需要找到各个 follower 与 自己一致的地方，删除 之后follower 中`错误`的数据，并且把自己 发过去进行覆盖。在这个过程中，leader 给每个 follower 维护了一个 `nextIndex` 序列，用做要发给各个 follower 的数据的指示。如果被 follower 拒绝了，那么就继续往回找，直到找到二者的相同序列。



## Log compaction

如果没有 compaction, server 的 log entries 就会无限制的增长。所以一般需要某种 snapshot 技术来压缩已经 commited 的 entries。如下图所示:

![](/images/raft/compaction.png)



每个 server 各自做各自的 snapshots., 并且添加一些 index 来指明位置。同时增加了 leader 向 follower 同步 snapshot 的功能。



## Client 交互

因为所有的情况下 client 的请求都由 leader 处理，但 client 又知道 leader 是谁。所以会随机发给一个 server，由 server 告知 client leader 相关信息。网络请求都要考虑重试以及可重入性的情况。假设 leader处理了一个请求，在 response 发出去之前 crash 了，那么client 会重试，这个操作就又会被执行第二遍。解决办法是给每个 client 的每个 command 一个序列号， raft 追踪下每个 client已经执行过 的最新的序列号,如果发现新发过来的已经执行过，就跳过了。

普通的 read 操作也会有问题，虽然不涉及到数据改动，但可能会读取到 stale data。有可能在读的时候有了新的 leader 并且当前数据已经过期了，所以 raft 在返回 read-only 的操作的 response 之前又增加了一个 leader exchange heartbeat 步骤来同步 leader 状态。

