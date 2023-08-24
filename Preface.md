# 前言

如果近几年你从事过软件工程领域的工作，特别是和服务打交道的后端系统，你可能被大量与数据存储处理有关的流行术语轰炸。NoSQL！大数据-Big Data！网络扩展-Web-scale！分片！最终一致性！ACID！CAP定理！云服务！MapReduce! 即时处理-Real-time！

在过去的十年里，我们在数据库、分布式系统以及基于他们所构建的应用上看到很多有意思的变革。这其中存在一些因素驱动着变革的发生：

* 像谷歌、雅虎，亚马逊，Facebook，领英，微软和推特这些网络公司，他们需要处理海量数据和流量，这就急切的迫使他们寻求一种新的工具来帮助他们有效的处理如此大规模的数据。
* 企业需要敏捷，廉价地测试假设，并通过缩短开发周期和数据模型来快速适应新的市场节奏。
* 免费的开源软件已经日趋成熟，在某些场景下甚至要优于一些商业软件以及定制化软件。
* 尽管CPU的频率增加甚微，但是多核计算已经成为现在的标准，而且网络也越发的快速，这就意味着并行性可以一直叠加。
* 即使你只是在一个小团队中，你也可能会搭建一个涉及多台机器甚至时跨区域的系统，这得益于像亚马逊的Web Service这样的基础架构服务（Iaas）的兴起。
* 很多服务现在要求系统必须要高可用，由于故障中断或维护而导致长时间停服将越来越不被接受。

数据密集型应用程序通过这些科技的发展，正在推动着可能的边界向外延伸。如果一个系统最大的挑战来自于它的数据数量，数据的复杂行或者数据的快速变化，而非受限于CPU执行速度的「计算密集型」系统，那么我们称这样的系统为「数据密集型」应用系统。

用以协助「**数据密集型应用**」来存储和处理数据的工具和技术也在快速这适应着这些变化。除了已经受到广泛关注的新型数据系统（“NoSQL”）外，像消息队列，缓存，搜索索引，流/批处理框架及一些相关技术也备受瞩目。很多应用都结合自己的场景来使用这些技术。

大量的新兴术语填补了这一领域的空白使得人们未来的无限可能充满期待，这是一个创举。然而对于软件工程师或架构师来说，我们仍然要准确透彻的理解这些技术，然后权衡利弊，这样才能构建出好的应用出来。为了能更透彻的理解这些术语，我们必须要深挖他们背后的真正含义。

好在，在技术快速更迭的背景下，无论你使用哪一种技术的哪种版本，都会存在一些亘古不变的准则。如果你清楚这些准则，你便可以用在正确的地方，用对正确的方法，以及怎样防止异常。这就是我们本书的切入点。

本书的目标就是帮助你处在变换莫测的技术更迭中，面对形形色色的场景来帮你理清如果处理和存储数据。本书并非是针对某一领域的工具类用书，也不是一本乏味的理论基础书。相反我们会在本书看到一些比较成功的数据系统案例：来自于主流的应用系统所涉及到的诸如必要的扩展性、高性能以及可靠性这些每天都在我们所使用的产品中使用到的技术。

我们将深入到这些系统的内部结构中，梳理他们的关键算法，讨论他们在实现过程中在权衡各种利弊得失所遵循哪些原则。在这段旅程中，我们并不只是单纯的了解它怎样工作，而会试着去探寻出针对数据系统最有效的处理方式，弄清楚它为什么它要这样工作，以及我们其它的一些疑问。

读完此书，你就会站在一个更高的维度来审视和决策哪种技术适用于什么场景，从而挑选正确的基础组建来构建一个良好的应用架构。你可能会说你工作当中并没有要从头搭建一套自己的数据存储引擎系统的必要。但是，通过本书，你将会对你系统底层所做的操作有一个清楚的认知，能够准确的判断他们的行为走向，从而做出良好的设计决策，及时的追踪到这中间可能产生的问题。

## 谁应该读这本书？

如果你开发的应用程序涉及到后端服务数据的处理和存储，并且你的应用是对互联网开放的（如：web应用，APP或 有网络交互场景），那么这本书适合你读。

本书面**向软件工程师**，**软件架构师**以及热爱编码的**技术经理**。特别是你工作中需要对你现有的系统架构的治理做决策性相关的工作，这本书尤其适用。例如，你需要选用一个技术工具来解决现有业务上的一些问题，并能够分析出利用该工具能很好的解决问题的观点做支撑。即使工作中没有选择工具这种场景，这本书同样能帮助你很好的了解他们的优势以及弱点。

你必须在构建web服务或网络服务领域有一定的开发经验，并且要数据关系型数据库和SQL。如果对非关系型数据库或者其他的一些数据相关的工具有了解就更好了，当然这不做强制要求。对一些基础的TCP和HTTP网络协议有一定的理解更有利于你理解。至于使用什么语言或着框架对读懂本书没有任何影响。

如果接下来列举的这些选项都是你想达到的目的，那么本书对你来说绝对意义非凡：

* 想学习如果搭建可伸缩的数据系统，例如可支撑数百万用户的web服务或移动应用。
* 需要构建高可用（最小停机时间）和易维护的应用系统。
* 正在寻求在系统规模不断扩大，需求和技术不断迭代的情况下也能够保持长期易维护的方法。
* 对主流的网站或在线服务的运行原理有着天然的好奇心，并想要探索他们内部本质的原理。本书对剖析了各种数据库以及数据处理系统的内部结构，探索他们的设计思想会为我们带来极大的乐趣。

有时，当我们在讨论「**可扩展性**」数据系统时，人们就会说：“你并不是谷歌或亚马逊，不需要担心可扩展性，只需要关系型数据库就已经够用了。“ 这句话在这份声明中时成立的：在你不需要「**可扩展性**」的场景下构建「**可扩展性**」的系统是在浪费精力，这会很容易让你陷入到一种教条主义的困境当中。实际上，它是一种过早的设计形式。但是，选择正确解决问题的工作是很重要的，因为每一种不同的技术都有它的优势和短板。也正如我们看到的，关系型数据库至关重要，但也不是解决数据问题的最终解。

## 本书涵盖的范围

本书不会去详细的介绍怎样去安装或使用特定的软件或API，因为已经有足够多的文档去教你怎么去做这些事了。而是讨论数据系统的各种基本原则已经调度规则，并探讨不同的产品之间有什么不同的设计决策。

在电子书版本中，我们已经把在线资源链接放到了原文中去了。所有的链接在发表过程中都得到了验证，但是链接可能因为网络自身的原因会失效，如果你遇到链接打不开或者你读的是印刷本，你可以使用搜素引擎来查找相关文献。对于**学术论文**，你可以在谷歌论文网站上根据标题来搜索到相关的开源PDF文件。或者，你可以在 *https://github.com/ept/ddia-references*这个网站上去查找所有的参考文献，我们会在这个工程上维护最新的文献链接。

我们主要关注于数据系统的架构以及它是如何继承到数据密集型系统中的。本书并没有去涵盖**部署**，**运维**，**安全**，**管理**—这些复杂且重要的话题，也不会浅显的来介绍这些话题的内容。他们应该在他们该属于的领域书籍中存在。

本书介绍的很多技术点大多数都可以归属到「**大数据**」的范畴。但是，由于「**大数据**」这个术语的过度使用以及并没有一个确切的定义来真正描述它，使得在一些演进的工程讨论中显得毫无意义。本书则使用了更为模糊的术语，比如单节点相对于多节点，在线交互式系统相对于离线批处理系统。

本书更倾向于免费的开源软件（FOSS），因为阅读、修改、并执行源代码是深入理解它如何工作的最好方式。开放的平台同时也降低了来自供应商的限制。但是，在特定的场合，我们也会讨论商业软件（闭源软件，服务软件，以及公司内部未开放的软件）。

## 内容纲要

本书分为3个部分：

1. **第一部分**，我们将讨论支撑数据密集型应用程序设计的基本思想。从[第一章](part2/chapter1.md)开始我们将会讨论我们真正想要达到的目标：可靠性，可扩展性以及可维护性；我们应该从哪种维度来思考以及如何具体的实现。在[第二章](part1/chapter2.md)我们会比较集中不同的数据模型以及查询语言，以及他们是怎么在不同的场景下来使用的。在[第三章](part1/chapter3.md)我们讨论存储引擎，数据如何在磁盘上排布，才能方便我们下一次很容易的再查询到它。[第四章](part1/chapter4.md)则会转向数据编码（序列化）及它的演化过程。
2. **第二部分**，我们从在一台机器上存储数据到多台机器上存储数据，这对「可伸缩性」来说通常是很有必要的，但也会带来一系列特殊的挑战。这部分我们会讨论**副本**（[第五章](part2/chapter5.md)），**分区/分片**（[第六章](part2/chapter6.md)），以及**事务**（[第七章](part2/chapter7.md)），接下来我们会渗入到分布式系统所涉及的问题中来讨论（[第八章](part2/chapter8.md)）和分布式系统中达成一致性共识的意义所在（[第九章](part2/chapter9.md)）。
3. **第三部分**，我们将讨论从一个数据集到另一个数据集的派生数据系统。派生数据经常出现在异构系统中：当单个数据库不能很好的完成任务时，应用需要结合多个数据库、缓存或者索引等共同完成。我们首先会在[第十章](#part2/chapter10.md)中讨论派生数据的批处理，在这基础上我们接在会在[第十一章](#part3/chapter11.md)讨论流处理。最后在[第十二章](#part2/chapter12.md)我们将结合的来讨论关于构建可靠的、易扩展的以及可维护应用的可行性方案。

## 参考文献和延伸阅读

本书所涉及的大部分内容都已经在其它的某些地方以某种如会议演示文稿，研究论文，博客文章，源码，BUG跟踪器，邮件表等形式出现过了。本书从这些来源中间总结出了他们中最重要的设计思想，其中包括原文件的链接。本书每一章结尾的文献也是一些能让你要更深入了解这些领域知识的重要资料，并且他们几乎都可以在网上免费获取。

## 奥莱利狩猎系列图书

> Safari（原Safari Books Online）是一个为企业、政府、教育机构以及个人提供会员制培训以及指导的平台。

会员有权限对超过250家出版社所出版的数千本图书，培训视频，学习途径，交互式教程以及精选视频集免费访问。他们包括O’Reilly媒体，Harvard Business Review，Prentice Hall Professional，Addison-Wesley Professional, Microsoft Press, Sams, Que, Peachpit Press, Adobe, Focal Press, Cisco Press, John Wiley & Sons, Syngress, Morgan Kaufmann, IBM Redbooks, Packt,Adobe Press, FT Press, Apress, Manning, New Riders, McGraw-Hill, Jones & Bartlett等课程技术。

更多信息可以访问 [http://oreilly.com/safari](http://oreilly.com/safari)。

## 联系我们

任何对本书的意见和建议都可以通过以下地址来反馈给出版商：

O’Reilly Media, Inc.

1005 Gravenstein Highway North

Sebastopol, CA 95472

800-998-9938 (美国或加拿大)

707-829-0515 (国际或国内)

707-829-0104 (传真)

本书还有一个web网页，我们会列举出相关勘误、示例以及其它的信息。你可以通过这个页面：[*http://bit.ly/designing-data-intensive-apps*](http://bit.ly/designing-data-intensive-apps)来访问。

如果想对本书提出建议或咨询技术问题，请发送邮件到邮箱： [*bookquestions@oreilly.com*](bookquestions@oreilly.com)。

有关本书的更多信息，课程，议题和新闻，可以访问我们的网站：[*http://www.oreilly.com*](http://www.oreilly.com).

Facebook: *http://facebook.com/oreilly*

Twitter: *http://twitter.com/oreillymedia*

YouTube: *http://www.youtube.com/oreillymedia*

## 鸣谢

本书把大量其它人积累下来的想法和知识结合学术研究和工程实践经验来融合并系统性的概括。在计算机领域，人们往往会被新的事物所吸引，但是我认为我们从以往沉淀下来的技术中汲取知识将具有重大的价值。本书将有超过800多篇的参考文献，包括论文，博文，演讲以及文档等，他们对我来说都是非常宝贵的学习资源。在此，我也很感谢这些相关文献的作者能够无私的分享他们的成果。

我也从和他们的交谈中受益颇丰，感谢这么多朋友能够花时间来与我共同探讨问题，交换想法，并耐心讲解，在此，特别要感谢：Joe Adler, Ross Anderson, Peter Bailis, Márton Balassi, Alastair Beresford, Mark Callaghan, Mat Clayton, Patrick Collison, Sean Cribbs, Shirshanka Das, Niklas Ekström, Stephan Ewen, Alan Fekete, Gyula Fóra,Camille Fournier, Andres Freund, John Garbutt, Seth Gilbert, Tom Haggett, Pat Helland, Joe Hellerstein, Jakob Homan, Heidi Howard, John Hugg, Julian Hyde, Conrad Irwin, Evan Jones, Flavio Junqueira, Jessica Kerr, Kyle Kingsbury, Jay Kreps, Carl Lerche, Nicolas Liochon, Steve Loughran, Lee Mallabone,Nathan Marz, CaitieMcCaffrey, Josie McLellan, Christopher Meiklejohn, Ian Meyers, Neha Narkhede,Neha Narula, Cathy O’Neil, Onora O’Neill, Ludovic Orban, Zoran Perkov, Julia Powles, Chris Riccomini, Henry Robinson, David Rosenthal, Jennifer Rullmann,Matthew Sackman, Martin Scholl, Amit Sela, Gwen Shapira, Greg Spurrier, Sam Stokes, Ben Stopford, Tom Stuart, Diana Vasile, Rahul Vohra, Pete Warden, Brett Wooldridge.

另外还有一些朋友对本书进行审阅和反馈，这对我本书的写作也是无比宝贵的。对于这些贡献，我要感谢Raul Agepati, Tyler Akidau, Mattias Andersson, Sasha Baranov, Veena Basavaraj,David Beyer, Jim Brikman, Paul Carey, Raul Castro Fernandez, Joseph Chow, Derek Elkins, Sam Elliott, Alexander Gallego, Mark Grover, Stu Halloway, Heidi Howard,Nicola Kleppmann, Stefan Kruppa, Bjorn Madsen, Sander Mak, Stefan Podkowinski, Phil Potter, Hamid Ramazani, Sam Stokes, Ben Summers。当然，对于本书存在的错误和问题，本人将承担一切责任。

还要特别感谢我的编辑 Marie Beaugureau, Mike Loukides, Ann Spencer, 以及O’Reilly的团队成员，有了他们本书才能能以出版，感谢他们包容我缓慢的写作进度，以及满足我提出的各种要求。感谢 Rachel Head，帮忙为本书找出最精确的用语。感谢Alastair Beresford, Susan Goodhue, Neha Narkhede和 Kevin Scott给了我大量的时间能让我自由写作，以及有时间做一些其它的工作。

特别感谢Shabbir Diwan 和 Edie Freedman，感谢他们为每一章绘制的精美导图，他们这个创造性的想法真的太棒了，这使得本书最终效果更佳酷炫。

最后，我挚爱的家人和朋友，没有你们的支持，我不可能完成这将近4年的写作历程，永远爱你们。
