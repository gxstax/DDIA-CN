![ch5](../img/chapter5.png)

# 第五章：备份

> 一件事情出错和不出错最大的区别在于当一件不可能出错的事情出错时，通常已经难以挽救了。
>
> — Douglas Adams, *Mostly Harmless* (1992)



备份意味着要在被网络连接的多个不同的机器之间存储相同的数据。正如在 [第二部分](README.md) 引言中所描述的，有以下几点原因是为什么需要进行备份：

* 使数据在地理位置上离用户更近（减少延迟）。
* 当部分数据存储服务宕机后，系统仍然可以继续提供服务（提高可用性）。
* 自由伸缩提供读取服务的机器数量（提高读取吞吐量）。

在本章我们假设你的数据量足够小，每台机器都可以完整的保存整个数据集。在[第六章](chapter6.md)会抛开这个假设来谈谈对于单台机器来说数据量过大时怎么来对数据集进行分区（分片）。在后续的章节，我们还会讨论在备份数据系统中可能产生的一些错误或者缺陷，以及如何来解决它们。

如果你的副本数据不随着时间而发生变化，那复制是比较容易的：你只需要一次性的把数据复制到各个节点就可以了。复制机制中所有的难点都在于对更改的数据处理上，这也是我们本章所探讨的重点。我们会谈到节点之间对于变更数据复制的3种主流的算法：单主（*single-leader*）复制，多主（*multi-leader*）复制以及无主（*leaderless*）复制。所有对于分布式数据的处理无外乎这3种方式。他们之间各有利弊，接下来我们将会详细的阐述。

复制（replication）涉及许多权衡考虑，例如是否使用同步或异步复制，以及如何处理失败的副本。这些通常是数据库中的配置选项，尽管具体细节因数据库而异，但原则上他们的实现是相似的。我们将在本章中讨论此他们做出这些选择的原因。

数据库的复制是一个比较久远的话题—自从19世纪70年代以来，它的原理性的东西并没有太大变化[[1](#ch5References1)]，因为底层网络的「**约束规范性条件（fundamental constraints ）**」也一直保持没变。然而，在研究之外，很多开发人员长时间以来一直认为数据库始终都只有一个节点。对于分布式数据的主流应用是近期才出现的。由于很多应用开发人员都是刚接触到这一领域，所以他们对于像「最终一致性（*eventual consistency*）」这些概念存在很多误解。在[延迟同步问题](#延迟同步问题)我们将更深入的叙述「最终一致性」，并且会讨论诸如「**读己写-*read-your-writes***」和「**单调读 -*monotonic reads***」等概念。

## 领导者和追随者

每一个存储数据库拷贝的节点我们称之为一个「**副本-*replica***」。随着多副本的产生，问题也会随之而来：我们怎样确保每个副本中都保存有完整的数据？

每一次对数据库的写请求必须能被所有副本接收并且处理；不然，数据库副本之间就可能会包含不同的数据。对于此类问题最常用的解决方案是如[<font color="#A7535A">**图5-1**</font>](#figure5-1)所示的「**基于领导者复制-*leader-based replication***」（也被称作「**主动/被动-*active/passive***」 或者「**主从复制-*master–slave replication***」）。它的工作机制如下：

1. 把其中一个副本设定为「**领导者**」（也被称为「**主-*master***」或「**首要-*primary***」）。如果客户端对数据库写入数据，则必须将他们的请求发送给领导者，领导者会最先把新写入的数据写入到本地。

2. 其它的副本就是我们所说的「**追随者**」（**只读-*read replicas***，***slaves***，***secondaries***或 ***hot standbys***）。[^i]当领导者把新数据写入本地存储后，还会把变更的数据以变更日志或变更流的形式发送给它的追随者。追随者接受到领导者发送的变更日志后便会更新它本地的数据库备份数据，它会按照领导者的执行顺序来执行本地的变更。

   ---

   [^i]:Different people have different definitions for *hot*, *warm*, and *cold* standby servers. In PostgreSQL, for example, *hot standby* is used to refer to a replica that accepts reads from clients, whereas a *warm standby*processes changes from the leader but doesn’t process any queries from clients. For purposes of this book, the difference isn’t important.

3. 当客户端读取时，它既可以从领导者副本读，也可以从追随者副本读取。但是写入请求只能由领导者副本接收（从客户端的角度来看，追随者副本是只读的）。

![图5-1](../img/figure5-1.png)

<a id="figure5-1"><font color="#A7535A">**图5-1.**</font></a> **基于领导者（主从）复制**

这种复制模型时很多诸如（PostgreSQL (9.0以后版本), MySQL, Oracle Data Guard [[2]()], 以及 SQL Server的

AlwaysOn Availability Groups [[3]()]）等关系型数据库内建特性。

### 同步复制与异步复制

### 增加新的追随者

### 处理节点异常

### 同步日志实现



## <a id="ReplicationLag">延迟同步问题</a>

### 读己写

### 单调读

### 一致性前缀读

### 同步延时解决方案



## 多主复制

### 多主复制用例

### 写冲突处理

### 多领导者副本拓扑



## 无主复制

### 写时节点故障

### 限定法定数一致性

### Sloppy Quorums and Hinted Handoff



## <a id="detectiongConcurrentWrites">并发写入检测</a>
