---
title: "Sinfonia: a new paradigm for building scalable distributed systems"
toc: true
categories: 技术
date: 2019-07-24 17:22:10
excerpt: 论文笔记
tags: [论文,分布式, 笔记]
---



<!-- toc -->

论文链接: [Sinfonia: a new paradigm for building scalable distributed systems](http://www.sosp2007.org/papers/sosp064-aguilera.pdf)

Sinfornia 像是一个构建分布式系统的 SDK 。目前的市面上分布式系统已经非常多了，比如 etcd, zookeeper, ceph 等等。但他们都是分别实现的，彼此之间的代码并无太多借鉴。Sinfonia 的实践是很难得的，可以说比大部分的分布式软件都更有指导价值一些。



![](/images/sinfonia/arc.png)



这个图中展示了 Sinfonia 中关键的几部分

* user library: SDK 部分，隐藏了分布式系统处理内部细节的 SDK。用户基于 user library 来开发分布式应用
* minitranscations: Sinfonia 的核心概念。一个分布式的事物机制
* application node: 应用节点
* memory node: 维持分布式系统内部状态的节点。

图中没有画出的有

* 管理节点: 用于执行一些定期的 recover 任务
* `directory node`: Application Node 访问 Memory Node 使用的是逻辑 id,   directory node 记录了逻辑 id 与 真实地址的映射。



## Memory Node

Memory Node 存储了应用的状态数据，根据不同情况，可能是存在 RAM 中，也可能是在磁盘上。 user library 封装了操作 Memory Node 中数据的方式。Application Node 可以与 Memory Node 是同台机器也可以是不同机器(有些 Application 出于性能考虑会需要 Applicaiton Node 与 Memory Node 为同台机器，Sinfonia 会告知 Application 此种情况以便让其尽量把数据写在本机)。

Sinfonia 的一个特点是，它的数据结构可以理解为都是字节数组，目的是为了解耦。每个 Memory Node 都只负责管理字节数组，并且可以通过 `(memory-node-id, address)` 这样的序列来定位具体的数据。user library 即封装了操纵 Memory Node 上这些字节数组的细节。



Memory Node是多个，出于数据 locality 的考虑，Sinafonia 并不提供负责均衡策略，而是让 Application 自己选择 Memory Node。Sinfonia 给 Application 提供了 Memory Node 的 load 信息供其参考。





## Minitranscation



Minitranscation 可以说是 Sinfonia 的核心了。Application 可以通过它来访问和修改 Memory Node，并且保证一致性，持久性，隔离性以及可靠性。 它有如下优点:



1. 允许用户将多次操作放到一个 batch 里面
2. 可以直接在 commit protocol 里执行?
3. minitransactions can execute in parallel with a replication scheme ?

 

一个 Minitranscation 包含三类操作, 按执行顺序如下所示:

* compare items: 对比发送的数据与已经存储的数据。如果对比不相等，或者出错，那么整个 transcation 中断。
* read items: 比较成功(相等), 读取数据
* write items: 比较成功(相等), 写入数据

![](/images/sinfonia/minitranscation.png)



这几个 items 的组合能够实现我们普遍需要的一些原子语义:



* `swap`: read -> write
* `compare-and-swap`: compare -> write
* `acquire-lease`: compare 0 -> write id



## 两阶段提交

标准的 two-phase commit 里，如果 coordinator 崩溃，整个系统需等待其恢复后才可用。在 Sinfonia 中，coodinator 就在 application node, 如果它不可用，系统并不会阻塞。而是阻塞在出问题的 Meomory Node 上（因为 Memory Node 保存了系统的核心数据，并且 Memory Node 有 replicate 机制来保证高可用性coodinator 的作用减轻了，它并不需要维持状态。只要在 Meomory Node 处理失败时重试即可。



需要注意的是， 在 phase 1,  participant 在获取锁后，首先对 compare items 进行比较，如果成功，vote commit, 然后读取 read items ，记录 transcations(`redo-log`，只包含 write items).否则就 vote abort,释放锁并反悔了。



在 phase 2, coordinator 汇总结果，告知 participants 是 commit 还是 abort。如果是 commit, partcipants 开始写入 write items。



## Cache

cache 放置在 Application 层，而不是在 Memory Node 上。这样做的好处有:

* 灵活性更好
* Application 对于  cache 的利用率比较高
* Sinfonia 的架构更为简单，从 Sinfonia 获取到的数据永远都是最新的

Minitranscations 提供的 `compare` 机制可以让 Application 方便地校验其本地的 cache 是否有效。



## 容错

Sinfonia 使用四种机制来提供 Meomory Node 容错功能:

* `disk images`: 保存 memory node 的数据的 copy。异步写，数据可能会比较老。reply `repo-log` 可以再 memory 数据丢失的时候进行恢复。
* `logging`: 同步写，保存最近的数据更新
* `replication`: 一个 stand-by 的 node。同步更新
* `backup`: 数据备份。如果 disk images 数据丢失，可以尝试从 backup 恢复。



上面说到 coordinator 的作用在 two-phase commit 中的作用被削弱，因为其不保存状态。在其 crash 时，会使一些 transcation 陷入不确定状态。 Sinfonia 引入了一个 `recovery coordinator` 来解决这个问题。首先，每个 Momory Node 会记录其处理的一些 transcations 的状态，如下图所示:

![](/images/sinfonia/log.png)



recovery coordinator 会循环执行，读取 Memory Node 的记录，从 `in-doubt` list 中读取那些未提交的 transcations 并尝试恢复。首先，在 phase 1, recovery coordinator 要求所有的 participants vote abort。如果一个 participants 已经 vote 过了，那么就使用之前的 vote ，否则就 vote abort。phase 2 就跟普通的流程一样。 所以假设之前的 participants 都 vote 了 commit ，然后 coordinator 不可用，那么 recovery coordinator 就能将这个 transcations commit 了。否则就是 abort 。 



如果一个 Memory Node crash, 上图中的 `decided` list 可以辅助 `disk-image` 用来恢复 Meomory Node 的内存数据。因为它也是异步写的，会有一些 transcations 存在于 `redo-log` 但不在 `decided` 中。这种情况下需要联系其他 particants 看他们的 vote 结果来判定这个 transcations 是否需要 replay。(参与某个 transcations 的particants都记录在 redo-log 中)



如果大部分或者所有 Memory Node crash, 管理节点会启动程序来尝试恢复。关于管理节点，论文中并未着重介绍。目前提到的就两个地方，一个是用于recovery coordinator,一个便是恢复整个系统的。所以看起来更像是一个救火的，并不是常驻服务。在此种情况下，每个 Meomory Node 都会用上面所说的方式进行自我恢复。但是可以通过主动向其他 Meomory Node 发送自己的最近的 transcations 来避免像上面那样的主动询问(交互量上的优化)。



## 垃圾回收

`redo-log`以及其他 log 随着时间的增长肯定是不断增加的，所以需要进行定期的垃圾回收。对于已经 aborted 的 transcation, 可以直接回收。对于已经 commited 的 transcation, 需要确保每个参与的 momory node 都将数据落地了。(因为会有 Node crash 的情况)。其他的 list 也会随着 redo-log 的清理同步回收一些 transcations。



`forced-abort` list 的回收要复杂些。因为 Sinfonia 允许 origin 和 recovery 两个 coordinator 同时运行。如果直接回收，会导致正在运行的 coordinator 出错(它仍想处理这个 transcation)。 Sinfornia 加了一个 `epoch number` 来处理这种情况。epoch number 与时间相关，由participants 定义并返回给 coordinator。每个 transcation 都与一个 `epoch number` 绑定。partipants 可以拒绝` stale` 的 epoch（大于等于两个 epoch 的差距）。那么 `forced-abort` 中早期的 trancations 都可以被回收掉，因为如果要重试肯定会被 abort 掉。 



## 数据备份

transication -> redo-log -> disk-image -> backup 。当然 disk-image 只会更新已经 commited 的 transcations， backup 可以从 disk-image 做整个 copy 或者 snapshot。







## 链接

1. [论文笔记：[SOSP 2007] Sinfonia: a new paradigm for building scalable distributed systems](https://zhuanlan.zhihu.com/p/34421583)

