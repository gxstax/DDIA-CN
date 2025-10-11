![第十一章](../img/chapter11.png)

# 第十一章：流处理

> 复杂的系统都是从简单的系统演进而来的，从另一个角度来说，开始就很复杂的系统是根本不会被设计出来的。
>
> — 约翰·加尔, *Systemantics* (1975)

在[第十章](chapter10.md)中我们讨论了批处理-它是将一组文件作为输出然后在产出一组新的文件的技术。输出是**派生数据**（*derived data*）的一种形式；运行在批处理流程中的数据在必要的时候是可以再次生成的。我们了解到了这一简单却强大的理念是如何被应用于构建搜索索引、推荐系统、分析工具等领域。

但是，在[第十章](chapter10.md)中我们始终假设的一个命题是：输入是有边界的-即已知且有限的大小-所以批处理程序知道它何时完成了输入信息的读取。例如MapReduce的核心排序操作必须完整的读取所有的输入后才能开始输出结果：这就有可能会发生一个很小的键值数据记录很晚才被输入，因为这个小键值记录需要我们第一个输出，所以我们的输出不能很早进行（要等待这个最小键值输入后才能开始输出操作）。

实际上，很多数据之所以无边界是因为它们是随着时间的推移慢慢到达的：你的用户昨天和今天产生了数据，它明天还会继续产生更多的数据。除非你歇业，否者这个过程永远不会停止，所以在一定意义上，数据集从来不会有“完成（complete）”态\[[1](#ch11References1)]。因此，批处理程序必须人工的将数据划分为固定区间的数据块：例如：在每天结束的时候处理一天的有效数据，或者在每个小时结束后处理这一小时的有效数据。

以天为维度的批处理问题在于输入数据的变化只有在一天后才会对输出的结果有影响，这对很多急性子的用户来说太慢了。为了减少延迟，我们可以更频繁的执行批处理程序-例如每一秒都执行一次来处理上一秒的有效数据，甚至于说连续的，完全不划分固定区间，每个事件来临就立即处理。这便是「流处理」（*stream processing*）背后的思想。

通常，“流”指的是随着时间推移而逐渐可用的数据。这个概念出现在很多地方：在Unix的标准输入（stdin）和标准输出（stdout）中，编程语言（lazy lists）\[[](#ch11References2)],文件系统API（例如Java的FileInputStream），TCP连接，通过互联网传输的音频和视频等等。

本章，我们将把事件流（event streams）视为一种数据管理机制：一种对应于我们上一章所介绍的，一种无界的，持续增量处理的批数据处理方式。我们首先会讨论流在网络中怎样表示、存储和传输。在“[数据库与流](#数据库与流)”中我们会讨论流和数据库之间的联系。最后，在“[流处理](#流处理)”中,我们会继续探讨处理这些流的方法和工具，以及它们用来构建应用的方式。

## 传输事件流

在批处理的世界，任务的输入输出一般是文件（可能是分布式的文件系统）。那么流又是怎样的呢？

当输入是文件（字节序列）时，首先要做的处理步骤通常是把它解析成为记录序列。在流处理的上下文中，一条记录通常会被称为**事件**（event），本质上它们是同一件事物：一种小的、独立的、不可变的含有事物发生时间节点信息的对象。一个事件通常会包含一个表明在一天中发生的时钟节点一致的时间戳（详见[单调和实时时钟](#单调和实时时钟)“Monotonic Versus Time-of-Day Clocks”）。

举个例子，这个发生的事件既可能是来自于用户执行的一个操作，如浏览网页或者支付一笔账款。也可能是来自于机器，如对温度进行的周期性测量或者是CPU的利用率指标。在[使用Unix工具的批处理](#使用Unix工具的批处理)这个例子中，web服务器的每一行日志便是一个事件。


事件的编码方式可能是字符串，或者是JSON，也可能是我们在[第五章](#chapter5)中讨论过的二进制表格。怎样编码决定了你可以怎样存储你的事件，例如你可以把它追加到文件中，也可以把它插入到关系型数据库表，又或者是把它写入到文档数据库中。另外你还可以通过网络把事件发送给其他节点来处理。

在批处理中，一次写入文件可能会被多个任务读取。类似的，在流处理的语境下，一个事件也是由生产者一次生成（也可称为发布者或发送者），然后可能被多个消费者消费（订阅者或者是收件人）\[[3](#anchor-3)]。在文件系统中，文件名用以标识一组相关记录；在「流处理」系统中，相关的事件通常会组合到**主题**（topic）或者**数据流**（stream）中。

原则上，文件或数据库足以连接生产者和消费者：生产者将它生产的每个事件写入到数据存储，然后每个消费者定期的从数据存储中轮询数据来检查从上次拉取后产生的新事件。实际上这正是批处理在每天结束时处理这一天有效数据的过程。

但是，若数据存储不是专门为这种用途而设计的，要在低延迟的情况下实现持续处理数据，轮询的代价是非常大的。你轮询操作越频繁，返回新事件的请求比例越低，系统开销也就越大。因此，当新事件产生时，我们最好主动通知消费者（而不是消费者轮询）。

传统数据库通常不能很好的支持这种通知机制：关系型数据库一般使用**触发器**（*triggers*），来承接这种变化（比如：往表里插入了一条数据），但是它们的能力非常有限，这更像是数据库事后的补偿设计\[[4](#ch11References4)]。所以，它是一种味了支撑事件通知而开发的特殊工具。

### 消息系统

向消费者通知新事件的通用方法就是使用消息系统（messaging system）：生产者发送一条包含事件的消息，然后推送给消费者。在前面我们在[“基于消息传递数据流”](#message-passing_dataflow)[page 136](#page136)介绍过，现在我们会进一步详细的介绍。

像Unix管道或者TCP链接这种建立在生产者和消费者之间的直接通信通道是实现消息系统的最简单方式。其实，大部分消息系统都是在基于这个模型演进的。不同的是，Unix管道和TCP链接是一个发送者对应一个接收者，而消息系统允许多个生产者节点往同一个topic发送消息且允许多个消费者节点从一个topic中接收消息。

在这种**发布/订阅**模型中，不同的系统采用不同的方法，并没有一个标准的答案可以同时满足不同的目的。想要区分这些系统，接下来的这两个问题将会对我们很有帮助：

1. **当生产者发送消息过快超过了消费者的接收范围会发生什么？**一般来说，有三种选择：丢弃消息，把消息缓存到队列，或者使用**背压**（也就是我们通常所说的流量控制-*flow control*；比如阻塞生产者来防止生产更多的消息）。例如，Unix管道和TCP链接就是使用了**背压**：他们使用了一个固定大小的缓冲区，当缓冲区满的时候，发送者会被阻塞直到有接收者从缓冲区消费数据（参见：[网络拥塞与排队](../part2/chapter7.md#网络拥塞与排队) 第282页）。

如果消息被缓存到队列，那么了解队列增长时会发生什么将会很重要。如果消息队列长度达到内存最大容量时系统会不回崩溃，或者消息是否需要写入到磁盘？如果写磁盘，那么磁盘访问会对消息系统\[[6](#ch11References6)]的性能产生什么影响？

2. **如果节点崩溃或者离线，是否会发生消息丢失？**

和数据库一样，持久化可能需要写磁盘/复制这些组合操作（详见：[复制与持久化](../part2/chapter7.md#复制与持久化)），这些操作都是有成本的。如果你可以容忍偶发的消息丢失，那么在相同的硬件配置下你或许可以获得更高的吞吐量以及更低的延时。

消息丢失是否可以被接受在很大程度上取决于应用。例如，像传感器的读数和统计这种周期性的传输场景，偶尔丢失消息是没有那么重要的，因为不管怎样，下一个新的数据很快就会再次发送出来。但是，需要特别注意的是如果大量数据丢失，而这些丢失的数据可能不会被立即觉察到，那么就会导致统计的错误\[[7](#ch11References7)].如果你是在对事件计数，那么交付的可靠性将会变得非常重要，因为在每次丢失消息之后意味着你后面的统计数都是错误的。

在[第十章](#chapter10.md)批处理系统中有一个很好的特性是它提供了强可靠性的保证：失败的任务是可以自动重试的，并且失败任务中输出部分也会自动被丢弃。这也就是说没有失败的情况下输出内容都是相同的，这利于我们简化程序模型。稍后在本章中我们会探讨如何在流处理情景下怎么来实现类似的保证。

#### 生产者消费者之间的直接消息传递

一部分消息系统在消费者和生产者之间采用无中间节点的直接网络通信：
* UDP 组播广泛的应用于像股票市场这种需要低延迟要求的金融行业\[[8](#ch111References8)]。尽管UDP本身是不可靠的，但应用层协议可以重新请求丢失的包（生产者必须记录它所发送包以便于在需要的时候可以重新发送）。
* 像[ZeroMQ\[[4](#ch11-References4)]无代理的消息库采用的方法类似，使用TCP或IP多播来实现消息的发布/订阅。
* StatsD\[[10](#ch11References10)]和Brubeck\[[7](#ch11References7)]使用不可靠的UDP消息来统计网络中机器的指标并对它们进行监控。（在StatsD协议中，计数器指标只有在接收到所有消息的情况下才正确；使用UDP只能是统计指标近似正确\[[11](#ch11References11)],令请参阅第283页 [TCP与UDP](../part2/chapter7#TPC与UDP)）。
* 如果消费者在网络上暴漏服务，那么生产者可以直接发送HTTP或RPC请求（参见：[基于服务的数据流：REST和RPC](../part1/chapter4.md#基于服务的数据流：REST和RPC)）来推送消息给消费者。这也正是[webhook](#ch11References12)背后的思想，一个服务的回调URL被注册到另一个服务中，并且每当事件发生时都会向该URL发送请求。

尽管这些直接消息系统在适配当前设计的目标下可以很好的工作，它们通常需要应用代码能意识到消息丢失。缺点是它们容错十分有限：尽管这个协议能够感知到网络中包的丢失并重新转发，前提是它们始终认为生产者和消费者一直处于在线状态。

如果有消费者离线，它可能会丢失掉发送出去还未到达的消息。一些协议允许生产者对失败的消息重新投送，但是当生产者崩溃是，这种方法可能就会失效了，这会导致本来需要重新投送的消息缓冲区丢失。

#### 消息代理

一种广泛的做法是使用**消息代理**（也称之为**消息队列**）来发送消息，本质上是一种优化消息流\[[13](#ch11References13)]处理方式的一类数据库。它作为服务器运行，生产者和消费者作为客户端与它连接。生产者写消息到代理服务器，消费者通过从代理服务器读取的方式来接收消息。

通过把数据集中在代理中的方式，这些系统可以从容的应对客户端的变化（连接、断开和故障），并且持久化的问题也同样的转移到了代理服务器上。某些消息代理仅仅将消息放到内存中，而另一些（取决于配置）会写入到磁盘，以此来保证代理服务器故障的时候不会丢失消息。对于速度比较慢的消费者，它们一般允许无限队列（而不是丢弃消息或者**背压**），当然这种选择也取决于配置。

排队的结果通常也使得消费者通常采用异步消费（*asynchronous*）：当生产者发送一条消息，通常只需要等待代理服务器确认消息已经进入到缓冲区，而不需要关心消费者处理消息的过程。向消费者交付通常发生在未来的某一个不确定的时间点——一般在几分之一秒内，但如果遇到队列积压，有时也会出现明显的延迟。


#### 消息代理和数据库对比

一些消息代理服务有时会使用XA或者JTA来实现两阶段提交（two-phase）协议（参见：[分布式事务实践](#分布式事务实践)）。这个特性使得它本质上与数据库非常相似，尽管消息代理与数据库之间仍然存在着明显的差异：

* 数据库通常会保留数据直到主动的删除，而大多数消息代理在把消息投递给消费者后就会自动的删除数据。这样的消息代理服务是不适合做数据的长期存储的。
* 因为消息会被很快删除的原因，可以认定大部分消息代理服务器的工作集相当的小——也就是说队列很短。如果是因为消费者速度变慢而代理服务需要缓存很多消息数据的话（可能没有足够的内存，那么多出来的消息会被写到磁盘），那么每个独立的消息被处理的时间都会拉长，从而导致整体的吞吐量下降\[[6](#ch11References6)]。
* 数据库通常支持**二级索引**和多种查找数据的方式，而消息代理通常支持订阅适配了某些正则表达式的topic数据子集。这些机制虽然不同，但本质上都是为客户端选择它们需要的部分数据。
* 当查询数据库时，结果集通常是基于数据的某一个时间点的快照；如果其他的客户端随后又写入一些数据到数据库改变了之前的查询结果集，第一个客户端是发现不了它的部分结果是过期数据的（除非它重新查询或者轮询查看更改内容）。相比之下，消息代理虽不支持随时查询，但是当数据变化时（即有新的有效消息）它会通知客户端；

这是消息代理服务的传统观念，像JMS\[[14](#ch11References14)] 和 AMQP \[[15](#ch11References15)]标准中也有所体现，并且像RabbitMQ,ActiveMQ, HornetQ, Qpid, TIBCO Enterprise Message Service, IBM MQ, Azure Service Bus, and Google Cloud Pub/Sub \[[16](#ch11References16)]等系统中都有实现.

#### 多消费者

当多个消费者从同一个主题中读取消息时，就会出现像<font color="#A7535A"> **[图11-1](#figure11-1)**</font>表示的那样，有两种主流的消息投递模式会被用到：

- 负载均衡模式
	每个消息投递给其中一个消费者，从而消费者可以共享主题下的消息处理任务。消息代理可以任意的指定消费消息的消费者。这种模式适合消息处理代价比较高的场景，并且你可以通过增加消费者来水平扩展消息的处理能力。（在 AMQP 中，你可以通过让多个客户端从同一个队列中消费来实现负载均衡，在 JMS 中则被称为**共享订阅**。）

- 分发模式（扇出）
	每条消息都会投递给所有的消费者。分发模式允许多个独立的消费者各自收听到同一个消息的广播，而不会相互影响——流处理等价于几个不同的批处理任务读取同一个输入文件。（JMS的主题订阅和AMQP的交换机绑定都提供了这种特性。）

![图11-1](../img/figure11-1.png)
<a id="figure11-1"><font color="#A7535A"> **图 11-1.**</font> </a> (a)负载均衡：在消费者中间共享同一个主题下的消费任务；（b）分发：每个消息都投递给多个消费者。

这两种模式可以组合：比入两组相互独立的消费者组可能都订阅了同一个topic，这样每组可以共同接收所有的消息，但是在组内每条消息只能有一个节点接收。

#### 确认和重传

消费者可能会随时崩溃，所以可能你会发生代理投递一个消息给消费者，但消费者从来没有处理它，或者在奔溃时只处理了一部分。为了确保消息不会丢失，消息代理使用**确认**（acknowledgments）：当客户端处理完一条消息后必须明确的告知代理服务，然后代理服务就可以把它从队列中移除。

如果接收确认消息时和客户端的连接关闭或超时，那么代理服务会认为消息未被处理，然后会把这条消息再次投递给另一个消费者。（要注意，消息实际上可能完全处理完成，但是确认消息在网络传输中丢失了。为了处理这种场景就需要一个原子性的提交协议，正如[“分布式事务实践”](../part2/chapter9.md#分布式事务实践)。）

当与**负载均衡**结合时，重新投递的行为在对消息的排序中会产生一个有趣的影响。在<font color="#A7535A"> **[图11-2]**</font>中，消费者通常是根据生产者发送的顺序来处理消息。但是，当消费者1处理消息m4的同时，消费者 2处理消息m3发生故障。那么未被确认的消息m3随后被投递给了消费者1，结果就是消费者1按照m4，m3，m5的顺序来处理消息。因此，m3和m不是按照生产者1的发送顺序来处理的。

![图11-2](../img/figure11-2.png)
<a id="figure11-2"><font color="#A7535A"> **图 11-2.**</font></a> 消费者2在处理消息m3的时候发生了故障，随后消息被投递给了消费者1。

尽管消息代理尝试保持消息的顺序（正如JMS和AMQP协议要求的那样），但在组合负载均衡的时候，消息投递不可避免的导致消息的重排序。为了解决这个问题，你可以让每个消费者单独的使用一个消息队列（即不使用负载均衡特性）。如果消息之间是完全相互独立那么消息重排序就不算是一个问题，但是消息顺序对消息之间存在依赖关系的情况下是非常重要的，我们会在本章后面继续来讨论。

### 分区日志

通过网络发送的数据包或者发送给网络服务的请求通常是不会持久化操作痕迹的瞬时操作。尽管是可以永久记录（使用抓包和日志），我们通常不会考虑使用这种方式。尽管消息代理服务会把消息持久化写入到磁盘，但当消息投递给消费者后又会很快的被删除，因为消息代理是构建在**瞬时消息思维模式**之上的。

数据库和文件系统则采用完全相反的方式：任何写入数据库或文件的数据通常都是期望被永久保存，至少在显式的删除之前是这样。

这种思维方式上差异对于如何创建派生数据有很大影响。在[第十章](#chapter10.md)讨论中，批处理的一个关键特性就是你可以重复的执行并尝试处理步骤，但是并不会有损害输入数据的风险（因为输入是**只读的**）。这并不是AMQP/JMS风格的消息：接收了一个确认的消息对数据是有损坏性的，因为被消费者确认过的消息会被消息代理删除，所以你不能通过重复消费者操作来获得相同的数据。

如果你在消息系统中增加一个新的消费者，它通常只能接收到从它注册的时间点往后发送的消息；所有之前的消息都已经丢失并且不能被回复。相对于文件和数据库，则可以随时增加客户端，并且可以读取到以前任意写入的数据（是要应用程序没有明确的覆盖或删除数据）。

那为什么我们不能结合数据库的持久化和消息数据的低延迟混合起来使用呢？这正是**消息日志代理**背后的思想。


#### 基于日志的消息存储

日志仅仅是磁盘上的一个支持追加的记录序列。我们之前在[第三章](../part1/chapter3.md)的**日志结构存储引擎**和**预写日志**（WAL技术：write-ahead logs）以及[第五张](../part2/chapter5.md)的副本中讨论过日志。

我们可以使用同样的结构来实现消息代理：生产者通过将消息追加到日志的末尾来发送消息，消费者通过依次读取日志来接收消息。如果消费者读取到了日志的末尾，那么它将会等待消息追加的通知。UNIX的tail -f 工具，正是基于这种工作思路来实现监听是否有数据追加到文件。

为了打破单个磁盘所能提供的高吞吐量的上限，日志支持分区（[第六章](../part2/chapter6.md)的内容）。不同的分区可以分布在不同的机器上，每一个分区的日志都是隔离的，从而可以实现独立于其他分区的读取和写入。一个主题可以定义成带有相同类型数据的一组分区。如<font color="#A7535A">[图11-3](#figure11-3)</font>所示：

在每个分区中，代理服务为每条消息会分配一个单调递增的序列号或偏移量-*offset*(在图11-3，方框中的数字便是偏移量）。


![图11-3](../img/figure11-3.png)
<a id="figure11-3"><font color="#A7535A"> **图 11-3.**</font></a>生产者通过追加到主题分区文件的方式发送消息，然后消费者依次读取。

Apache Kafka\[[17](#ch11References17),[18](#ch11References18)],Amazon Kinesis Steams\[[19](#ch11References19)],和Twitter的DistributedLog\[[20](#ch11References20),[21](#ch11References21)] 是这种基于日志的消息代理实现方式。Google Cloud Pub/Sub 的架构也是类似，但是它是通过暴漏JMS风格的API的方式而非通过抽象日志的方式\[[16](#ch11References16)].尽管这些消息代理把所有的消息都写入磁盘，但是他们依然能够通过多服务器分区的方式实现百万级别的消息吞吐量，并且可以通过消息复制来实现容错性\[[22](#ch11References22),[23](#ch11References23)].


#### 日志与传统消息系统对比

基于日志的方式可以很自然的支持消息的分发/扇出(fan-out)，因为多个消费者可以做到不相互影响独立的读取日志-读取消息不回将它从日志中删除。为了在一组消费者之间实现负载均衡，代理可以将整个分区分配给消费者组的节点，而不是单独的将消息分配给消费者客户端。

每个客户端都会消费到分配给它的分区的所有消息。通常，当消费者被指定一个日志分区后，它将直接以单线程的方式依次从分区中读取消息。这种粗粒度的负载均衡方式有以下缺陷：
* 一个主题的消费的节点数最多等于该主题下分区数，因为同一个分区内的消息被投递给了同一个节点。[^注i]
* 如果单个消息处理缓慢，它会阻塞分区内后续的消息（这是一种对头阻塞的方式；参见：第13页的[性能描述](../part1/chapter1.md#性能描述)）。

---
 <font size=2 color=#42426F>注i： 可以创建一个负载均衡方案，在这个方案中，两个消费者通过读取全部消息来共享处理分区的工作，单其中一个只考虑偶数偏移量的消息，而另一个处理奇数偏移量的消息。或者，可以将消息处理扩展到线程池，但这种方法会使消费者偏移管理变得复杂。通常，单线程处理分区是优选的，并且可以通过使用更多的分区来增加并行性。</font>

---

因此，在消息处理成本高昂，你希望在消息一条一条的基础上并行处理，并且消息没有有序性要求的场景中，JMS/AMQP类型消息代理会更合适。另一方面，当需要高吞吐量，每条消息都需要快速处理并且要保证消息顺序的场景下，基于日志的消息存储更适合。

#### 消费者偏移

顺序的消费分区中的消息可以很容易的知道哪些消息是已经被处理的：所有位移小于消费者当前便宜量的消息都是已经被处理过的消息，所有大于消费者当前便宜量的消息都是还未处理的。所以消息代理不再需要为每条消息再标记确认信息——只需要周期性的记录消费者的偏移量即可。记录开销的减少以及批量处理和流水线操作有助于提高基于日志的系统的吞吐量。

偏移量实际上和主从复制数据库中常见的日志序列号非常相似，我们在第五章中[**配置新的从节点**](../part2/chapter5.md#配置新的从节点)讨论过这种这种场景。在数据库复制中，日志序列号可以能够让追随者在和领导者断连后再次接入，而不会中断这断连期间的任何写入数据。同样这里也使用了相同的原则：消息代理的行为就像数据库领导者一样，消费者就像追随者。

如果消费者节点故障，那么这个故障的消费分区将会分配给消费者组中的另外一个节点，它会接着最后一次记录的偏移量继续消费。如果消费者消费了部分数据后还未来得及记录消费偏移量，那么这些消息在重新启动后将会在一次被消费处理。我们会在本章的后续部分来讨论这个问题的处理方式。

#### 磁盘空间利用

如果你只是往日志里面追加写入，那么最后会造成磁盘空间溢出。为了回收磁盘空间，日志实际上是分段的，老的日志段会不定期的被删除或者被存档。（稍后我们会讨论更多更复杂的释放磁盘空间的方法。）

这就意味着如果一个很慢的消费者不能够维持消息消费的速率，那么它的消费偏移量将远远落后甚至偏移位置指向了一个删除后的日志片段，这就会导致一些消息的丢失。实际上，日志实现了一个有限大小的缓冲区也就是我们所熟知的**圆形缓冲区**或**环形缓冲区**，当它满的时候就会把旧的消息丢弃（覆盖掉）。但是，因为这个缓冲区在磁盘上，所有它的空间可能会非常大。

我们做一个粗略的计算。在撰写本文的时候，典型的硬盘容量一般是6TB，顺序写的吞吐量是150M/s。如果按照磁盘最快的速率写入消息，写满硬盘大约花费11个小时。所以，磁盘可以保留11个小时内的有效数据，接着后续写入的新消息便会覆盖旧消息。保即使你加再多的磁盘和机器，这个比率仍然不会发生改变。但实际中，很少能够达以磁盘的满带宽速度写入，所以日志通常可以保留几天甚至几周的有效数据。

无论你保留的时间有多长，日志的吞吐量基本保持恒定，因为所有的消息最终都会写入到磁盘\[[18](#ch11References18)]。这种操作方式和默认把消息写入缓存，只有当消息队列增长过快时才会将它们写入磁盘的方式形成鲜明的对比：当消息队列很短时系统很快，当队列过长以致于开始写磁盘时速度就会变的很慢，所以吞吐量取决于要保留历史数据的大小。

#### 消费者落后生产者

#### 消息重新处理

## 数据库与流

### 保持系统同步

## 流处理

## 流处理的适用场景

## 小结



<a id="message-passing_dataflow"></a>

<a id="netWork-congestion-and-queueing"></a>






---
## 引用

[<a id="ch11References1">1</a>] Tyler Akidau, Robert Bradshaw, Craig Chambers, et al.: [“The Dataflow Model: A Practical Approach to Balancing Correctness, Latency, and Cost in Massive-Scale,Unbounded, Out-of-Order Data Processing,”](https://www.vldb.org/pvldb/vol8/p1792-Akidau.pdf) Proceedings of the VLDB Endowment,volume 8, number 12, pages 1792–1803, August 2015. doi:10.14778/2824032.2824076

[<a id="ch11References2">2</a>] Harold Abelson, Gerald Jay Sussman, and Julie Sussman: [Structure and Interpretation of Computer Programs](https://mitpress.mit.edu/sicp/), 2nd edition. MIT Press, 1996. ISBN:978-0-262-51087-5, available online at mitpress.mit.edu

[<a id="ch11References3">3</a>] Patrick Th. Eugster, Pascal A. Felber, Rachid Guerraoui, and Anne-Marie Kermarrec: “[The Many Faces of Publish/Subscribe](https://www.cs.ru.nl/~pieter/oss/manyfaces.pdf),” ACM Computing Surveys, volume 35, number 2, pages 114–131, June 2003. doi:10.1145/857076.857078

[<a id="ch11References4">4</a>] Joseph M. Hellerstein and Michael Stonebraker: [Readings in Database Systems](http://redbook.cs.berkeley.edu/),4th edition. MIT Press, 2005. ISBN: 978-0-262-69314-1, available online at redbook.cs.berkeley.edu

[<a id="ch11References5">5</a>] Don Carney, Uğur Çetintemel, Mitch Cherniack, et al.: “[Monitoring Streams – A New Class of Data Management Applications](https://www.vldb.org/conf/2002/S07P02.pdf),” at 28th International Conference on Very Large Data Bases (VLDB), August 2002.

[<a id="ch11References6">6</a>] Matthew Sackman: “[Pushing Back](lshift.net/blog/2016/05/05/pushing-back/),” lshift.net, May 5, 2016.

[<a id="ch11References7">7</a>] Vicent Martí: “[Brubeck, a statsd-Compatible Metrics Aggregator](https://githubengineering.com/brubeck/),” githubengineering.com, June 15, 2015.

[<a id="ch11References8">8</a>] Seth Lowenberger: “[MoldUDP64 Protocol Specification V 1.00](https://www.nasdaqtrader.com/content/technicalsupport/specifications/dataproducts/moldudp64.pdf),” nasdaqtrader.com, July 2009.

[<a id="ch11References9">9</a>] Pieter Hintjens: [ZeroMQ – The Guide](https://zguide.zeromq.org/page:all). O’Reilly Media, 2013. ISBN: 978-1-449-33404-8

[<a id="ch11References10">10</a>] Ian Malpass: “[Measure Anything, Measure Everything](https://codeascraft.com/2011/02/15/measure-anything-measure-everything/),” codeascraft.com, February 15, 2011.

[<a id="ch11References11">11</a>] Dieter Plaetinck: “[25 Graphite, Grafana and statsd Gotchas](https://blog.raintank.io/25-graphite-grafana-and-statsd-gotchas/),” blog.raintank.io,March 3, 2016.

[<a id="ch11References12">12</a>] Jeff Lindsay: “[Web Hooks to Revolutionize the Web](https://progrium.com/blog/2007/05/03/web-hooks-to-revolutionize-the-web/),” progrium.com, May 3,2007.

[<a id="ch11References13">13</a>] Jim N. Gray: “[Queues Are Databases](http://research.microsoft.com/pubs/69641/tr-95-56.pdf),” Microsoft Research Technical Report MSR-TR-95-56, December 1995.

[<a id="ch11References14">14</a>] Mark Hapner, Rich Burridge, Rahul Sharma, et al.: “[JSR-343 Java Message Service (JMS) 2.0 Specification](https://jcp.org/en/jsr/detail?id=343),” jms-spec.java.net, March 2013.

[<a id="ch11References15">15</a>] Sanjay Aiyagari, Matthew Arrott, Mark Atwell, et al.: “[AMQP: Advanced Message Queuing Protocol Specification](https://www.rabbitmq.com/resources/specs/amqp0-9-1.pdf),” Version 0-9-1, November 2008.

[<a id="ch11References16">16</a>] “[Google Cloud Pub/Sub: A Google-Scale Messaging Service](https://cloud.google.com/pubsub/architecture),” cloud.google.com,2016.

[<a id="ch11References17">17</a>] “[Apache Kafka 0.9 Documentation](https://kafka.apache.org/documentation.html),” kafka.apache.org, November 2015.

[<a id="ch11References18">18</a>] Jay Kreps, Neha Narkhede, and Jun Rao: “[Kafka: A Distributed Messaging System for Log Processing](http://www.longyu23.com/doc/Kafka.pdf),” at 6th International Workshop on Networking Meets Databases (NetDB), June 2011.

[<a id="ch11References19">19</a>] “[Amazon Kinesis Streams Developer Guide](https://docs.aws.amazon.com/streams/latest/dev/introduction.html),” docs.aws.amazon.com, April 2016.

[<a id="ch11References20">20</a>] Leigh Stewart and Sijie Guo: “[Building DistributedLog: Twitter’s HighPerformance Replicated Log Service](https://blog.twitter.com/2015/building-distributedlog-twitter-s-high-performance-replicated-log-service),” blog.twitter.com, September 16, 2015.

[<a id="ch11References21">21</a>] “[DistributedLog Documentation](https://distributedlog.incubator.apache.org/docs/latest/),” Twitter, Inc., distributedlog.io, May 2016.

[<a id="ch11References22">22</a>] Jay Kreps: “[Benchmarking Apache Kafka: 2 Million Writes Per Second (On Three Cheap Machines)](https://engineering.linkedin.com/kafka/benchmarking-apache-kafka-2-million-writes-second-three-cheap-machines),” engineering.linkedin.com, April 27, 2014.

[<a id="ch11References23">23</a>] Kartik Paramasivam: “[How We’re Improving and Advancing Kafka at LinkedIn](https://engineering.linkedin.com/apache-kafka/how-we_re-improving-and-advancing-kafka-linkedin),” engineering.linkedin.com, September 2, 2015.

[<a id="ch11References24">24</a>] Jay Kreps: “[The Log: What Every Software Engineer Should Know About RealTime Data’s Unifying Abstraction](https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying),” engineering.linkedin.com, December 16, 2013.

[<a id="ch11References25">25</a>] Shirshanka Das, Chavdar Botev, Kapil Surlaker, et al.: “[All Aboard the Databus](http://www.socc2012.org/s18-das.pdf)!,” at 3rd ACM Symposium on Cloud Computing (SoCC), October 2012.

[<a id="ch11References26">26</a>] Yogeshwer Sharma, Philippe Ajoux, Petchean Ang, et al.: “[Wormhole: Reliable Pub-Sub to Support Geo-Replicated Internet Services](https://www.usenix.org/system/files/conference/nsdi15/nsdi15-paper-sharma.pdf),” at 12th USENIX Symposium on Networked Systems Design and Implementation (NSDI), May 2015.

[<a id="ch11References27">27</a>] P. P. S. Narayan: “[Sherpa Update](http://web.archive.org/web/20160801221400/https://developer.yahoo.com/blogs/ydn/sherpa-7992.html),” developer.yahoo.com, June 8, .

[<a id="ch11References28">28</a>] Martin Kleppmann: “[Bottled Water: Real-Time Integration of PostgreSQL and Kafka](https://martin.kleppmann.com/2015/04/23/bottled-water-real-time-postgresql-kafka.html),” martin.kleppmann.com, April 23, 2015.

[<a id="ch11References29">29</a>] Ben Osheroff: “[Introducing Maxwell, a mysql-to-kafka Binlog Processor](https://developer.zendesk.com/blog/introducing-maxwell-a-mysql-to-kafka-binlog-processor),” developer.zendesk.com, August 20, 2015.

[<a id="ch11References24">30</a>] Randall Hauch: “[Debezium 0.2.1 Released](http://debezium.io/blog/2016/06/10/Debezium-0/),” debezium.io, June 10, 2016.

[<a id="ch11References31">31</a>] Prem Santosh Udaya Shankar: “[Streaming MySQL Tables in Real-Time to Kafka](https://engineeringblog.yelp.com/2016/08/streaming-mysql-tables-in-real-time-to-kafka.html),” engineeringblog.yelp.com, August 1, 2016.

[<a id="ch11References32">32</a>] “[Mongoriver](https://github.com/stripe/mongoriver),” Stripe, Inc., github.com, September 2014.

[<a id="ch11References33">33</a>] Dan Harvey: “[Change Data Capture with Mongo + Kafka](https://www.slideshare.net/slideshow/change-data-capture-with-mongodb-and-kafka/51345480),” at Hadoop Users Group UK, August 2015.

[<a id="ch11References34">34</a>] “[Oracle GoldenGate 12c: Real-Time Access to Real-Time Information](https://www.oracle.com/us/products/middleware/data-integration/oracle-goldengate-realtime-access-2031152.pdf),” Oracle White Paper, March 2015.

[<a id="ch11References35">35</a>] “[Oracle GoldenGate Fundamentals: How Oracle GoldenGate Works](https://www.youtube.com/watch?v=6H9NibIiPQE),” Oracle Corporation, youtube.com, November 2012.

[<a id="ch11References36">36</a>] Slava Akhmechet: “[Advancing the Realtime Web](https://rethinkdb.com/blog/realtime-web/),” rethinkdb.com, January 27,
2015.

[<a id="ch11References37">37</a>] “[Firebase Realtime Database Documentation](https://firebase.google.com/docs/database?hl=zh-cn),” Google, Inc., firebase.google.com,May 2016.

[<a id="ch11References38">38</a>] “[Apache CouchDB 1.6 Documentation](http://docs.couchdb.org/en/latest/),” docs.couchdb.org, 2014.

[<a id="ch11References39">39</a>] Matt DeBergalis: “[Meteor 0.7.0: Scalable Database Queries Using MongoDB Oplog Instead of Poll-and-Diff](http://info.meteor.com/blog/meteor-070-scalable-database-queries-using-mongodb-oplog-instead-of-poll-and-diff),” info.meteor.com, December 17, 2013.

[<a id="ch11References40">40</a>] “[Chapter 15. Importing and Exporting Live Data](https://docs.voltdb.com/UsingVoltDB/ChapExport.php),” VoltDB 6.4 User Manual,docs.voltdb.com, June 2016.

[<a id="ch11References41">41</a>] Neha Narkhede: “[Announcing Kafka Connect: Building Large-Scale LowLatency Data Pipelines](confluent.io/blog/announcing-kafka-connect-building-large-scale-low-latency-data-pipelines),” confluent.io, February 18, 2016.

[<a id="ch11References42">42</a>] Greg Young: “[CQRS and Event Sourcing](https://www.youtube.com/watch?v=JHGkaShoyNs),” at Code on the Beach, August 2014.

[<a id="ch11References43">43</a>] Martin Fowler: “[Event Sourcing](http://martinfowler.com/eaaDev/EventSourcing.html),” martinfowler.com, December 12, 2005.

[<a id="ch11References44">44</a>] Vaughn Vernon: [Implementing Domain-Driven Design](https://vaughnvernon.co/?page_id=168). Addison-Wesley Professional, 2013. ISBN: 978-0-321-83457-7

[<a id="ch11References45">45</a>] H. V. Jagadish, Inderpal Singh Mumick, and Abraham Silberschatz: “[View Maintenance Issues for the Chronicle Data Model](http://www.mathcs.emory.edu/~cheung/papers/StreamDB/Histogram/1995-Jagadish-Histo.pdf),” at 14th ACM SIGACT-SIGMODSIGART Symposium on Principles of Database Systems (PODS), May 1995. doi:10.1145/212433.220201

[<a id="ch11References46">46</a>] “[Event Store 3.5.0 Documentation](http://docs.geteventstore.com/),” Event Store LLP, docs.geteventstore.com,
February 2016.

[<a id="ch11References47">47</a>] Martin Kleppmann: [Making Sense of Stream Processing](http://www.oreilly.com/data/free/stream-processing.csp). Report, O’Reilly Media,
May 2016.

[<a id="ch11References48">48</a>] Sander Mak: “[Event-Sourced Architectures with Akka](http://www.slideshare.net/SanderMak/eventsourced-architectures-with-akka),” at JavaOne, September 2014.

[<a id="ch11References49">49</a>] Julian Hyde: [personal communication](https://twitter.com/julianhyde/status/743374145006641153), June 2016.

[<a id="ch11References50">50</a>] Ashish Gupta and Inderpal Singh Mumick: Materialized Views: Techniques,Implementations, and Applications. MIT Press, 1999. ISBN: 978-0-262-57122-7

[<a id="ch11References51">51</a>] Timothy Griffin and Leonid Libkin: “[Incremental Maintenance of Views with Duplicates](https://homepages.inf.ed.ac.uk/libkin/papers/sigmod95.pdf),” at ACM International Conference on Management of Data (SIGMOD),May 1995. doi:10.1145/223784.223849

[<a id="ch11References52">52</a>] Pat Helland: “[Immutability Changes Everything](http://www.cidrdb.org/cidr2015/Papers/CIDR15_Paper16.pdf),” at 7th Biennial Conference on Innovative Data Systems Research (CIDR), January 2015.

[<a id="ch11References53">53</a>] Martin Kleppmann: “[Accounting for Computer Scientists](https://martin.kleppmann.com/2011/03/07/accounting-for-computer-scientists.html),” martin.kleppmann.com, March 7, 2011.

[<a id="ch11References54">54</a>] Pat Helland: “[Accountants Don’t Use Erasers](https://blogs.msdn.microsoft.com/pathelland/2007/06/14/accountants-dont-use-erasers/),” blogs.msdn.com, June 14, 2007.

[<a id="ch11References55">55</a>] Fangjin Yang: “[Dogfooding with Druid, Samza, and Kafka: Metametrics at Metamarkets](https://metamarkets.com/2015/dogfooding-with-druid-samza-and-kafka-metametrics-at-metamarkets/),” metamarkets.com, June 3, 2015.

[<a id="ch11References56">56</a>] Gavin Li, Jianqiu Lv, and Hang Qi: “[Pistachio: Co-Locate the Data and Compute for Fastest Cloud Compute](https://yahoohadoop.tumblr.com/post/116365275781/pistachio-co-locate-the-data-and-compute-for),” yahoohadoop.tumblr.com, April 13, 2015.

[<a id="ch11References57">57</a>] Kartik Paramasivam: “[Stream Processing Hard Problems – Part 1: Killing Lambda](https://engineering.linkedin.com/blog/2016/06/stream-processing-hard-problems-part-1-killing-lambda),” engineering.linkedin.com, June 27, 2016.

[<a id="ch11References58">58</a>] Martin Fowler: “[CQRS](https://martinfowler.com/bliki/CQRS.html),” martinfowler.com, July 14, 2011.

[<a id="ch11References59">59</a>] Greg Young: “[CQRS Documents](cqrs.files.wordpress.com/2010/11/cqrs_documents.pdf),” cqrs.files.wordpress.com, November 2010.

[<a id="ch11References60">60</a>] Baron Schwartz: “[Immutability, MVCC, and Garbage Collection](http://www.xaprb.com/blog/2013/12/28/immutability-mvcc-and-garbage-collection/),” xaprb.com,December 28, 2013.

[<a id="ch11References61">61</a>] Daniel Eloff, Slava Akhmechet, Jay Kreps, et al.: “[Re: Turning the Database Inside-out with Apache Samza](https://news.ycombinator.com/item?id=9145197),” Hacker News discussion, news.ycombinator.com,March 4, 2015.

[<a id="ch11References62">62</a>] “[Datomic Development Resources: Excision](https://docs.datomic.com/excision.html),” Cognitect, Inc., docs.datomic.com.

[<a id="ch11References63">63</a>] “[Fossil Documentation: Deleting Content from Fossil](http://fossil-scm.org/index.html/doc/trunk/www/shunning.wiki),” fossil-scm.org, 2016.

[<a id="ch11References64">64</a>] Jay Kreps: “[The irony of distributed systems is that data loss is really easy but deleting data is surprisingly hard](twitter.com/jaykreps/status/582580836425330688),” twitter.com, March 30, 2015.

[<a id="ch11References65">65</a>] David C. Luckham: “[What’s the Difference Between ESP and CEP](http://www.complexevents.com/2006/08/01/what%E2%80%99s-the-difference-between-esp-and-cep/)?,” complexevents.com, August 1, 2006.

[<a id="ch11References66">66</a>] Srinath Perera: “[How Is Stream Processing and Complex Event Processing (CEP) Different](https://www.quora.com/How-is-stream-processing-and-complex-event-processing-CEP-different)?,” quora.com, December 3, 2015.

[<a id="ch11References67">67</a>] Arvind Arasu, Shivnath Babu, and Jennifer Widom: “[The CQL Continuous Query Language: Semantic Foundations and Query Execution](http://research.microsoft.com/pubs/77607/cql.pdf),” The VLDB Journal,volume 15, number 2, pages 121–142, June 2006. doi:10.1007/s00778-004-0147-z

[<a id="ch11References68">68</a>] Julian Hyde: “[Data in Flight: How Streaming SQL Technology Can Help Solve the Web 2.0 Data Crunch](http://queue.acm.org/detail.cfm?id=1667562),” ACM Queue, volume 7, number 11, December 2009. doi:10.1145/1661785.1667562

[<a id="ch11References69">69</a>] “[Esper Reference, Version 5.4.0](http://www.espertech.com/esper/release-5.4.0/esper-reference/html_single/index.html),” EsperTech, Inc., espertech.com, April 2016.

[<a id="ch11References70">70</a>] Zubair Nabi, Eric Bouillet, Andrew Bainbridge, and Chris Thomas: “[Of Streams and Storms](https://developer.ibm.com/streamsdev/wp-content/uploads/sites/15/2014/04/Streams-and-Storm-April-2014-Final.pdf),” IBM technical report, developer.ibm.com, April 2014.

[<a id="ch11References71">71</a>] Milinda Pathirage, Julian Hyde, Yi Pan, and Beth Plale: “[SamzaSQL: Scalable Fast Data Management with Streaming SQL](https://github.com/milinda/samzasql-hpbdc2016/blob/master/samzasql-hpbdc2016.pdf),” at IEEE International Workshop on High-Performance Big Data Computing (HPBDC), May 2016. doi:10.1109/IPDPSW.2016.141

[<a id="ch11References72">72</a>] Philippe Flajolet, Éric Fusy, Olivier Gandouet, and Frédéric Meunier: “[HyperLog Log: The Analysis of a Near-Optimal Cardinality Estimation Algorithm](http://algo.inria.fr/flajolet/Publications/FlFuGaMe07.pdf),” at Conference on Analysis of Algorithms (AofA), June 2007.

[<a id="ch11References73">73</a>] Jay Kreps: “[Questioning the Lambda Architecture](oreilly.com/ideas/questioning-the-lambda-architecture),” oreilly.com, July 2, 2014.

[<a id="ch11References74">74</a>] Ian Hellström: “[An Overview of Apache Streaming Technologies](https://databaseline.wordpress.com/2016/03/12/an-overview-of-apache-streaming-technologies/),” databaseline.wordpress.com, March 12, 2016.

[<a id="ch11References75">75</a>] Jay Kreps: “[Why Local State Is a Fundamental Primitive in Stream Processing](https://www.oreilly.com/content/why-local-state-is-a-fundamental-primitive-in-stream-processing/),”oreilly.com, July 31, 2014.

[<a id="ch11References76">76</a>] Shay Banon: “[Percolator](https://www.elastic.co/blog/percolator),” elastic.co, February 8, 2011.

[<a id="ch11References77">77</a>] Alan Woodward and Martin Kleppmann: “[Real-Time Full-Text Search with Luwak and Samza](https://martin.kleppmann.com/2015/04/13/real-time-full-text-search-luwak-samza.html),” martin.kleppmann.com, April 13, 2015.

[<a id="ch11References78">78</a>] “[Apache Storm 1.0.1 Documentation](https://storm.apache.org/releases/1.0.1/index.html),” storm.apache.org, May 2016.

[<a id="ch11References79">79</a>] Tyler Akidau: “[The World Beyond Batch: Streaming 102](oreilly.com/ideas/the-world-beyond-batch-streaming-102),” oreilly.com, January 20, 2016.

[<a id="ch11References80">80</a>] Stephan Ewen: “[Streaming Analytics with Apache Flink](https://www.confluent.io/kafka-summit-2016-systems-advanced-streaming-analytics-with-apache-flink-and-apache-kafka),” at Kafka Summit, April 2016.

[<a id="ch11References81">81</a>] Tyler Akidau, Alex Balikov, Kaya Bekiroğlu, et al.: “[MillWheel: Fault-Tolerant Stream Processing at Internet Scale](research.google.com/pubs/pub41378.html),” at 39th International Conference on Very Large
Data Bases (VLDB), August 2013.

[<a id="ch11References82">82</a>] Alex Dean: “[Improving Snowplow’s Understanding of Time](http://snowplowanalytics.com/blog/2015/09/15/improving-snowplows-understanding-of-time/),” snowplowanalytics.com, September 15, 2015.

[<a id="ch11References83">83</a>] “[Windowing (Azure Stream Analytics)](https://msdn.microsoft.com/en-us/library/azure/dn835019.aspx),” Microsoft Azure Reference,msdn.microsoft.com, April 2016.

[<a id="ch11References84">84</a>] “[State Management](https://samza.apache.org/learn/documentation/0.10/container/state-management.html),” Apache Samza 0.10 Documentation, samza.apache.org,December 2015.

[<a id="ch11References85">85</a>] Rajagopal Ananthanarayanan, Venkatesh Basker, Sumit Das, et al.: “[Photon:Fault-Tolerant and Scalable Joining of Continuous Data Streams](http://research.google.com/pubs/pub41318.html),” at ACM International Conference on Management of Data (SIGMOD), June 2013. doi:10.1145/2463676.2465272

[<a id="ch11References86">86</a>] Martin Kleppmann: “[Samza Newsfeed Demo](https://github.com/ept/newsfeed),” github.com, September 2014.

[<a id="ch11References87">87</a>] Ben Kirwin: “[Doing the Impossible: Exactly-Once Messaging Patterns in Kafka](https://ben.kirw.in/2014/11/28/kafka-patterns/),”ben.kirw.in, November 28, 2014.

[<a id="ch11References88">88</a>] Pat Helland: “[Data on the Outside Versus Data on the Inside](http://cidrdb.org/cidr2005/papers/P12.pdf),” at 2nd Biennial Conference on Innovative Data Systems Research (CIDR), January 2005.

[<a id="ch11References89">89</a>] Ralph Kimball and Margy Ross: The Data Warehouse Toolkit: The Definitive Guide to Dimensional Modeling, 3rd edition. John Wiley & Sons, 2013. ISBN:
978-1-118-53080-1

[<a id="ch11References90">90</a>] Viktor Klang: “[I’m coining the phrase ‘effectively-once’ for message processing with at-least-once + idempotent operations](twitter.com/viktorklang/status/789036133434978304),” twitter.com, October 20, 2016.

[<a id="ch11References91">91</a>] Matei Zaharia, Tathagata Das, Haoyuan Li, et al.: “[Discretized Streams: An Efficient and Fault-Tolerant Model for Stream Processing on Large Clusters](https://www.usenix.org/system/files/conference/hotcloud12/hotcloud12-final28.pdf),” at 4th USENIX Conference in Hot Topics in Cloud Computing (HotCloud), June 2012.

[<a id="ch11References92">92</a>] Kostas Tzoumas, Stephan Ewen, and Robert Metzger: “[High-Throughput, LowLatency, and Exactly-Once Stream Processing with Apache Flink](http://data-artisans.com/high-throughput-low-latency-and-exactly-once-stream-processing-with-apache-flink/),” data-artisans.com,August 5, 2015.

[<a id="ch11References93">93</a>] Paris Carbone, Gyula Fóra, Stephan Ewen, et al.: “[Lightweight Asynchronous Snapshots for Distributed Dataflows](https://arxiv.org/abs/1506.08603),” arXiv:1506.08603 [cs.DC], June 29, 2015.

[<a id="ch11References94">94</a>] Ryan Betts and John Hugg: [Fast Data: Smart and at Scale](http://www.oreilly.com/data/free/fast-data-smart-and-at-scale.csp). Report, O’Reilly Media, October 2015.

[<a id="ch11References95">95</a>] Flavio Junqueira: “[Making Sense of Exactly-Once Semantics](http://conferences.oreilly.com/strata/hadoop-big-data-eu/public/schedule/detail/49690),” at Strata+Hadoop World London, June 2016.

[<a id="ch11References96">96</a>] Jason Gustafson, Flavio Junqueira, Apurva Mehta, Sriram Subramanian, and Guozhang Wang: “[KIP-98 – Exactly Once Delivery and Transactional Messaging](https://cwiki.apache.org/confluence/display/KAFKA/KIP-98+-+Exactly+Once+Delivery+and+Transactional+Messaging),”cwiki.apache.org, November 2016.

[<a id="ch11References97">97</a>] Pat Helland: “[Idempotence Is Not a Medical Condition](https://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.401.1539&rep=rep1&type=pdf),” Communications of the ACM, volume 55, number 5, page 56, May 2012. doi:10.1145/2160718.2160734

[<a id="ch11References98">98</a>] Jay Kreps: “[Re: Trying to Achieve Deterministic Behavior on Recovery/Rewind](http://mail-archives.apache.org/mod_mbox/samza-dev/201409.mbox/%3CCAOeJiJg%2Bc7Ei%3DgzCuOz30DD3G5Hm9yFY%3DUJ6SafdNUFbvRgorg%40mail.gmail.com%3E),”email to samza-dev mailing list, September 9, 2014.

[<a id="ch11References99">99</a>] E. N. (Mootaz) Elnozahy, Lorenzo Alvisi, Yi-Min Wang, and David B. Johnson:“[A Survey of Rollback-Recovery Protocols in Message-Passing Systems](https://www.cs.utexas.edu/~lorenzo/papers/SurveyFinal.pdf),” ACM Computing Surveys, volume 34, number 3, pages 375–408, September 2002. doi:10.1145/568522.568525

[<a id="ch11References100">100</a>] Adam Warski: “[Kafka Streams – How Does It Fit the Stream Processing Landscape?](https://softwaremill.com/kafka-streams-how-does-it-fit-stream-landscape/),” softwaremill.com, June 1, 2016.















