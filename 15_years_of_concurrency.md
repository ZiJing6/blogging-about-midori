# 并发 15 年

在[三种安全](https://github.com/ZiJing6/blogging-about-midori/blob/master/a_tale_of_three_safeties.md)中，我们讨论了三个种类的安全：类型、内存和并发。在这篇后续的文章中,我们将深入最后一个,也许也是最难的一个。首先是并发安全性将我引导到 [Midori](https://github.com/ZiJing6/blogging-about-midori/blob/master/README.md) 项目的，之前我在 .NET 和 c++ 的并发模型上花了多年的时间，让我最终加入进去。在这期间我们构建了一些伟大的的东西，让我非常自豪。也许某种程度更有趣的是，在该项目已经过去几年之后，对这一段经历的反思。

在今年的早些时间，我尝试了大概 6 次来写这篇文章，我很激动最后终于能分享它。我希望这对对这个领域感兴趣的每个人都有用，特别是在这方面进行积极创新的人。虽然代码示例和经验教训都深深根植于 C#、.NET 和 Midori 项目，但我已经试图将这些想法一般化，以便它们不受编程语言的影响。希望你们喜欢。

## 背景

在 21 世纪的大多数时间里，我的工作都是找出如何将并发交到开发者手中，这工作从微软 [CLR 团队](https://en.wikipedia.org/wiki/Common_Language_Runtime)的一个相对小众的工作开始。

### 小众的开始

回到那个时候，这很大程度是需要构建一个经典线程、锁和同步原语的更好的版本，并尽可能尝试将它们凝聚成最佳实践。例如，我们给 .NET 1.1 带来了一个线程池，并利用这个经验改善 Windows 内核、调度器以及它自己线程池的可伸缩性。我们有这个疯狂的 128 个处理器的 [NUMA（非均匀内存访问/非统一内存访问架构）](https://en.wikipedia.org/wiki/Non-uniform_memory_access) 机器，让我们忙于各种深奥的性能挑战。我们开发了[如何将并发弄妥当](http://joeduffyblog.com/2006/10/26/concurrency-and-the-impact-on-reusable-libraries/)的规则 —— 锁级别等等 —— 并进行了[静态分析](https://www.microsoft.com/en-us/research/wp-content/uploads/2008/08/tr-2008-108.pdf)实验。我甚至还写了关于它的[一本书](https://www.amazon.com/Concurrent-Programming-Windows-Joe-Duffy/dp/032143482X)。

为什么并发是第一位的？

一句话，它具有极大的挑战性，完全技术话事，所以充满了乐趣。

我一直都是语言的沉迷者。因此我就自然地被学术界几十年深入的研究深深吸引，包括编程语言与运行时共生（特别是 [Cilk](https://en.wikipedia.org/wiki/Cilk) 和 [NESL](https://en.wikipedia.org/wiki/NESL)）、高级类型系统、以及甚至专门的并行硬件体系结构（特别激进的像[连接机（The Connection Machine）](https://en.wikipedia.org/wiki/Connection_Machine)，以及 [MIMD](https://en.wikipedia.org/wiki/MIMD) 超级计算机，那创新超越了我们可信赖的老伙伴[冯·诺依曼体系](https://en.wikipedia.org/wiki/Von_Neumann_architecture)）。

虽然一些非常大的客户真的跑了[对称多处理器（SMP）](https://en.wikipedia.org/wiki/Symmetric_multiprocessing)服务器 —— 是的，我们事实上习惯这样叫它们 —— 我不会说并发是一个非常流行的专门领域。因此当然任何提及那些酷的带研究味的来源都会让我的同事和经理感觉怪怪的。尽管如此，我还是坚持了下去。

尽管很有趣，但我不会说这个时期我们做的工作对随意的观察人员产生了巨大的影响。我们是将抽象提高了一点 —— 以便开发者可以调度逻辑的工作项，考虑同步的更高级别等等 —— 但没有任何改变游戏规则的地方。但这段时期为之后到来的那些奠定基础，无论是基础上还是社区上，尽管我那时候还不知道。

### 不再有免费的午餐，进入多核

然后一些重大的事情发生了。

2004 年，Intel 和 AMD 跟我们谈[摩尔定律](https://en.wikipedia.org/wiki/Moore's_law)，尤其是它[就要终结](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.87.8775&rep=rep1&type=pdf)了。[能量墙困境](https://www.quora.com/Why-havent-CPU-clock-speeds-increased-in-the-last-5-years)会[严重减缓了曾经一直增长的时钟速度改进](http://www.economist.com/technology-quarterly/2016-03-12/after-moores-law)，而那是工业界早已经习惯了的。

突然，管理人员都变得非常关心并发问题。Herb Sutter 在 2005 年的[免费午餐结束了一文](http://www.gotw.ca/publications/concurrency-ddj.htm)中记录了这种狂热的投入。如果我们不能让开发者可以编写大量的并行软件 —— 这在历史上是非常困难而且几乎并且不太可能在没有显著降低进入门槛的情况下发生的事 —— 微软和英特尔的业务以及互利互惠的业务模式，都会遇到麻烦。如果硬件不会像往常那样变得更快，软件就不会自动变得更快，人们就没理由去购买新的硬件和软件。[Wintel](https://en.wikipedia.org/wiki/Wintel) 时代和[安迪·比尔盖茨定律](http://www.forbes.com/2005/04/19/cz_rk_0419karlgaard.html)就玩完了，*“安迪给多少，比尔盖茨就拿走多少”*。

或者，顺着这样想。

这是“多核”这个词闯入主流的时刻，而我们开始设想一个拥有 1024 核处理器或者更前瞻的从[数字信号处理器（DSP）](https://en.wikipedia.org/wiki/Digital_signal_processor)借鉴而来的[“非常多核”架构](https://en.wikipedia.org/wiki/Manycore_processor)的世界，混合了通用和专用的核心，能够承载非常重负荷的功能，例如加密、压缩等等。

另一方面，过了 10 年再回来看，事情并没有完全按照我们设想的那样演变。我们并没有用 1024 个传统的核来跑 PC，虽然[我们的 GPU 已经远超过了这个数](http://www.geforce.com/hardware/10series/titan-x-pascal)，而且我们的确看到了跟以前更多的异构性，特别是数据中心中，在那里 [FPGA 现在负担起大量的关键任务，像加密和压缩](https://www.wired.com/2016/09/microsoft-bets-future-chip-reprogram-fly/)。

在我看来，真正大的错失是移动端。恰恰是当思考能量曲线、密度和异构时，应该能告知我们移动是迫在眉睫的，而且是很大的一块。与其寻求更强大的 PC，我们应该去寻求我们口袋里的 PC。相反，我们自然的本能是抓住过去，“拯救” PC 业务。这是一个经典的[创新者的困境](https://en.wikipedia.org/wiki/The_Innovator's_Dilemma)，虽然在当时当然看起来不像。当然 PC 并没有在一夜间死亡，所以这里的创新也没有浪费，只是在历史的背景下让人感到不平衡。不好意思，我跑题了。

### 使并发更容易

作为一个并发 geek，这就是我一直等待的时刻。几乎一夜之间，为所有这些我一直梦想的创新工作寻找赞助商变得相当容易，因为现在它有了一个真正的非常迫切的业务需要。

简短地说，我们需要：

* 使编写并行代码变得更容易
* 使避开并发陷阱变得更容易
* 使以上两件事“出乎意料”地发生。

我们已经有了线程、线程池、锁以及基本的事件。现在我们应该去向何方呢？

三个特定的项目围绕这个点孵化了出来，并受到了兴趣和人员配置的激励。

#### 软件事务内存

具有讽刺意味的是，我们首先从安全开始。这预示着后面的故事，因为通常来说，安全是非常靠后的东西，知道我在 Midori 的背景下将它重新捡回来。

开发者已经有好几种引入并发的机制，并且仍然在努力地想写出正确的代码。因此我们寻求那些更高层次的好像能够意外地使正确性成为可能的抽象。

进入[软件事务内存(STM)](https://en.wikipedia.org/wiki/Transactional_memory)。自从 [Herlihy 和 Moss 开创性的 1993 年的论文](https://ocw.mit.edu/courses/electrical-engineering-and-computer-science/6-895-theory-of-parallel-systems-sma-5509-fall-2003/readings/herlihy_mo93.pdf)以来，大量有希望的研究已经问世，尽管这不是万能妙药，但我们中的很多人已经迷上了它的能够提高抽象层次的能力。

STM 让你像下面这样写东西，就能得到自动的安全：

```csharp
void Transfer(Account from, Account to, int amt) {
    atomic {
        from.Withdraw(amt);
        to.Deposit(amt);
    }
}
```

看哪！没有锁！

STM 能透明地处理所有的决策，例如了解使用多粗或多细粒度的同步、针对同步的竞争策略、死锁检测及预防、并保证当你访问共享数据结构时不会忘记锁定。所有的这些都藏在一个简单的关键字背后：atomic。

STM 还附带了易用的，更声明式的协调机制，例如 [orElse](https://hackage.haskell.org/package/stm-2.4.4.1/docs/Control-Monad-STM.html#v:orElse)。因此，尽管重点在于消除手动管理锁定的需要，它还有助于在线程间实现同步。

不幸的是，经过几年的深度运行时、操作系统甚至硬件支持原型之后，我们放弃了这方面的努力。我简要的总结是，鼓励良好的并发体系比糟糕的“仅仅能用”的并发体系更重要，虽然我已经在[这里](http://joeduffyblog.com/2010/01/03/a-brief-retrospective-on-transactional-memory/)和[这里](http://joeduffyblog.com/2010/05/16/more-thoughts-on-transactional-memory/)写了更多的细节。我们应该首先关注和解决一个更高层次的架构，在尘埃落定之后，再回来看看剩下的差距在那里。一旦我们达成了那个目标，STM 是否还是正确的工具仍未可知。（事后诸葛亮：我的确认为它是工具架中很多非常合理工具中的一个，尽管随着更多的分布式应用架构在增加，将[它交给人们是危险的](http://wiki.c2.com/?DistributedTransactionsAreEvil)。）

然而，我们的 STM 努力也并不是完全失败。正式这段时间，我开始尝试使用类型系统来实现安全并发。此外，这些零碎的东西最终被整合进了 Intel 的 Haswell 处理器，作为[事务同步扩展(Transactional Synchronization Extensions(TSX))](https://en.wikipedia.org/wiki/Transactional_Synchronization_Extensions)指令集，带来了使用[投机锁省略（speculative lock elision）](http://citeseer.ist.psu.edu/viewdoc/download;jsessionid=496F867855F76185B4C1EA3195D42F8C?doi=10.1.1.136.1312&rep=rep1&type=pdf)方法实现超低成本的同步和锁操作的能力。而且再一次，这段时期，我又跟一些神奇的了不起的人在一起工作。

#### 并发语言集成查询(PLINQ)

在 STM 之余，我还在晚上和周末搞着一个“臭鼬工程”数据并行框架，利用了我们最近在[语言集成查询(LINQ)](https://en.wikipedia.org/wiki/Language_Integrated_Query)中的工作成果。

并行 LINQ(PLINQ) 背后的想法是从三个研究得很好的领域偷了一页过来：

1. [并行数据库](https://en.wikipedia.org/wiki/Parallel_database)，它们已经能够[基于用户的行为并行 SQL 查询](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.21.2197&rep=rep1&type=pdf)，并且用户不需要知道发生了什么，经常带来非常惊人的结果。

2. 声明式和函数式语言，它们经常使用 [列表推导式(list comprehension)](https://en.wikipedia.org/wiki/List_comprehension) 来表达更高级别的能被更积极地优化的语言操作，包括并行。为此，我加深了我对 [Haskell](https://wiki.haskell.org/GHC/Data_Parallel_Haskell) 的痴迷，并从 [APL](https://en.wikipedia.org/wiki/APL_(programming_language)) 中得到了启发。

3. 数据并行，在[学术界有相当长的历史](https://en.wikipedia.org/wiki/Data_parallelism)了，甚至有更多的主流典型，最著名的是 [OpenMP](https://en.wikipedia.org/wiki/OpenMP)

这想法相当直接。将已有的 LINQ 查询拿过来，它们都已经包含了映射、过滤和聚集之类的操作 —— 这些在语言里和数据库里都是经典的可并行的东西 —— 并自动并行它们。当然，这不能是隐式的，因为有副作用。但所有的这些只需要一个小小的 AsParallel 就能启用起来了。

```csharp
// Sequential:
var q = (from x in xs
         where p(x)
         select f(x)).Sum();

// Parallel:
var q = (from x in xs.AsParallel()
         where p(x)
         select f(x)).Sum();
```

这 demo 了数据并行的一个很棒的方面，它能根据你输入的规模进行扩展，不管是数据量还是操作数据的花销，即使是两者一起，都行。当用足够高层次的语言，像 LINQ 来表达时，开发人员不必操心调度、选择正确的任务数量、同步之类的东西。

这本质上是在同一台机器上，跨越了多个处理器的 [MapReduce](https://en.wikipedia.org/wiki/MapReduce)。事实上，我们后来跟微软研究中心（MSR）在一个名为 [DryadLINQ](https://www.microsoft.com/en-us/research/project/dryadlinq/) 的项目上协作，它不仅在多个处理器上运行这些查询，还可以在很多机器上分布式执行它们。（最终我们甚至使用了 SIMD 和 GPGPU 实现了更细粒度的调度。）最终生成了微软自家的跟 MapReduce 一样的东西 —— [Cosmos](https://www.quora.com/Distributed-Systems-What-is-Microsofts-Cosmos)，这是一个今时今日还为微软提供很多大数据创新的系统。

开发 PLINQ 是我职业生涯的一段美好时光，也是一个真正的转折点。我跟一些了不起的人建立了合作关系。盖茨给这想法写了一整页的评论，以“我们必须特别为这项工作投入更多的资源”来总结。由于发展资金的投入，这种强烈的激励言辞并没带来什么伤害。它也引起了一些难以置信的人的注意。例如，[Jim Gray](https://en.wikipedia.org/wiki/Jim_Gray_(computer_scientist)) 就注意到了，我不得不亲身体验他臭名昭著的慷慨相助，就在他不幸消失的两个月前。

不用说，这是个激动人心的时期！

#### 插曲：形成 PFX 团队

这个时候，我决定扩大我们的工作范围，不仅仅是数据并行，还要处理任务并行和其他的并发抽象。因此我四处推销成立一个新团队的想法。

令我惊奇的是，开发平台事业部正在创建一个新的并行计算组，以响应不断变化的技术环境，他们希望赞助这些项目。这是一个机会，在一个很棒的业务主题下将所有这些整合起来，统一招募努力，将东西推得更远，最终散入 C++、GPGPU 和其他东西中。

所以，很明显地，我说了 yes。

我将团队命名为“[PFX](https://en.wikipedia.org/wiki/Parallel_Extensions)”，最初是“parallel frameworks”的缩写，虽然在我们发布的时候，市场展示了它的魔力，将它重命名为“Parallel Extensions to .NET”。团队的初始交付包含了 PLINQ、任务并行（task parallelism）和一批新的协作数据结构（Coordination Data Structures（CDS）），旨在处理高级同步工作，像[屏障风格（barrier-style）的同步](https://en.wikipedia.org/wiki/Barrier_(computer_science))、来源于[许多优秀研究论文](http://cs.rochester.edu/u/scott/papers/1995_TR600.pdf)的[无锁并发集合](https://github.com/dotnet/corefx/tree/master/src/System.Collections.Concurrent/src/System/Collections/Concurrent)等等。

#### 任务并行库（Task Parallel Library）

这将我带到了任务并行。

作为 PLINQ 的一部分，我们需要创建自己的并行“任务”的概念。我们需要一个精巧的调度器，能够根据给出的机器可用资源自动扩展。大多数已有的调度器是类线程池的，在里面它们要求每个任务跑在单独的线程上面，即使那样做是无益的。并且在其中将任务映射到线程是相当初级的，虽然[我们在那几年中已经对这做了改进](http://www.sigmetrics.org/conferences/sigmetrics/2009/workshops/papers_hotmetrics/session2_2.pdf)。

鉴于我对 Cilk 的热爱，还有调度大量存在递归可能的细粒度任务的需要，为我们的调度架构选择一个[工作偷取调度器](https://en.wikipedia.org/wiki/Work_stealing)是不假思索的事。

起初，我们的目光完全被锁在 PLINQ 的一亩三分地上，所以我们没有花多少精力在抽象上。然后 MSR 开始探索一个独立的任务并行库会是什么样的。这是一个完美的合作机会，所以我们开始一起构建一些东西。Task&lt;T> 抽象诞生了，我们重写了 PLINQ 来使用它，并为常见的模式创建了一套[并行 API](https://msdn.microsoft.com/en-us/library/system.threading.tasks.parallel(v=vs.110).aspx)，例如 fork/join 和 并行的 for 和 foreach 循环。

在发布之前，我们将线程池的核心替换成了我们全新闪亮的工作窃取调度器，在进程中引入统一的资源管理，一边多个调度器不会互相争斗。直到现在，代码几乎还跟我当初支持 PLINQ 的实现一样（当然还有很多的 bug 修复和改进）。

我们长期以来真的对相对小的 API 的易用性非常着紧。虽然我们犯过错误，但事后看来我还是很满意我们这样做了。我们直觉觉得 Task&lt;T> 会是我们在并行空间做的一切的核心，但没人料到如此广泛的使用，作为这些年来非常流行的异步编程。现在，它给 async 和 await 充满了力量，而我再也无法想象没有 Task&lt;T> 的生活。

#### 大声说出来：来自 Java 的启示

如果我没有提到 Java，以及它对我自己想法的影响，那将是我的疏忽。

为此，我们在 Java 社区的邻居也开始做一些创新工作，Doug Lea 领队，也收到了同样的学术来源的启发。Doug 在 1999 年的书[《Java 并发编程》](http://gee.cs.oswego.edu/dl/cpj/index.html)帮助在主流中推广这些思想，最终引致将 [JSR 166](https://jcp.org/en/jsr/detail?id=166) 纳入到 JDK 5.0 中。同时，Java 的内存模型也根据 [JSR 133](https://jcp.org/en/jsr/detail?id=133) 形式化，这是无锁数据结构的一个关键的基础，这是扩展到大量处理器时必需的。

他做的是我见到的第一次主流的尝试，在线程、锁和事件之上提升抽象级别成为更容易接受的东西：并发集合、[fork/join](http://gee.cs.oswego.edu/dl/papers/fj.pdf) 等等。这也使工业界更接近学术界的漂亮的并发编程语言。这些努力对我们产生了巨大的影响。我特别赞赏学术界和工业界如何密切合作，将数十年来的知识财富带到桌面上，并明确寻求在今后的几年中都[效仿](http://www.cs.washington.edu/events/colloquia/search/details?id=768)这种做法。

不用说，考虑到 .NET 和 Java 之间的相似性，以及竞争的级别，我们深受启发。

### 啊！安全，你在哪里？

所有的这些都有个大问题，它们都是不安全的。我们之前几乎只关注引入并发的机制，但没有一点使用它们是安全的保证。

这里有个好理由：这很难，真他妈难！特别是横跨开发人员能用的多种多张的并发方式时。值得庆幸的是，学术界在这个领域也有了几十年的经验，虽然相对于并行方面的研究，它显得格外“神秘”。我开始夜以继日上下而求索。

我的转折点是另一篇 BillG ThinkWeek 文章，我在 2008 年写的《驯服副作用》。其中，我描述了一种新的类型系统，我当时还不知道，它会构成我未来 5 年工作的基础。它不是很正确，而且还过于束缚于我在 STM 的经历，但它是一个不错的起点。

Bill 再次下结论：“我们需要搞这个。”所以我滚去开工了！

## 你好，Midori

但仍然有一个巨大的问题。我无法想象在现有的语言和运行时中逐步地完成这项工作。我不是在寻找一个温暖舒适的近似安全，而是一种这样的东西：如果你程序被编译通过了，你就知道它是没有数据竞争的。它是要刀枪不入的。

事实上，我试过了。我用 C# 的自定义 attribute 和静态分析做了这个系统的变体的原型，但很快就能做出推论，问题深深植根在语言中，要让这想法的任何部分正确工作起来，必须将它集成到类型系统里。而且它们还可能被远程调用。虽然当时我们有一些有趣的孵化项目（像 [Axum](https://en.wikipedia.org/wiki/Axum_(programming_language))），但考虑到愿景的范围，以及文化和技术原因，我知道这项工作需要一个新的家。

因此我加入了 Midori。

### 一种架构，以及一种设想

一群并发的专家也在 Midori 团队，好几年以来我一直跟他们探讨这个，这也导致了我的加入。在根子上，我们知道基于现有的基础工作是个错误的赌注。共享内存多线程真的不是未来，我们认为，特别是我之前的工作完全没有解决这个问题。Midori 团队正是成立来解决这个巨大挑战，做下豪赌的。

因此，我们做了一些工作：

* 隔离（Isolation）是最最重要的，我们会尽可能地拥抱它。

* 消息传递会连接很多这种隔离的部分，通过强类型的 RPC 接口。

* 也就是说，在进程中，存在一个消息循环，并且默认情况下，不会有额外的并行。

* 一种“promises-like”的编程模式是头等的，以使：
    * 同步阻塞是不允许的。
    * 所有在系统中的异步活动是显式的。
    * 复杂的协调模式是可行的，无需凭借锁和事件。

为了得到这些结论，我们深受 [Hoare 的通信顺序进程（CSP）](https://en.wikipedia.org/wiki/Communicating_sequential_processes)，Gul Agha 和 Carl Hewitt 在 [Actors](https://en.wikipedia.org/wiki/Actor_model)、[E](https://en.wikipedia.org/wiki/E_(programming_language))、[π](https://en.wikipedia.org/wiki/%CE%A0-calculus) 和 [Erlang](https://en.wikipedia.org/wiki/Erlang_(programming_language)) 上的工作，还有我们自己这些年来在开发并发、分布式的各种基于 RPC 的系统的共同经验的启发。

之前我没说过，但在我之前在 PFX 的工作中，消息传递明显是缺席的。有多方面的原因。首先，是有很多与之竞赛的成果，但没有一个“感觉”对头的。例如，[并发和协调运行时（Concurrency and Coordination Runtime(CCR)）](https://en.wikipedia.org/wiki/Concurrency_and_Coordination_Runtime)非常复杂（但有很多满意的客户）；Axum 语言，呃，一种新的语言；MSR 的 [Cω](http://research.microsoft.com/en-us/um/cambridge/projects/comega/) 很强大，但需要语言的变化，让人不禁犹豫要不要追一下（虽然派生出了只需要库的工作，还有[牛人加入](http://research.microsoft.com/en-us/um/people/crusso/joins/)，有了一些保障）；等等。另外，每个人似乎对基本的概念应该是怎么样的都有不同的想法，它没有带来什么帮助。

但这归根到底都是隔离。对于我们认为很有必要用来提供安全、无处不在且容易的消息传递的细粒度隔离，Windows 进程太重量级了。而且 Windows 上没有适合这种任务的进程内隔离技术：[COM apartments](https://en.wikipedia.org/wiki/Component_Object_Model#Threading)、CLR AppDomain …… 许多有缺陷的尝试立刻涌上心头；坦白说，我真不愿挂在那座山上。

（在那之后，我得说明，有了一些很不错的成果，像 [Orleans](https://github.com/dotnet/orleans) —— 某种程度是由一些前 Midori 成员构建的 —— [TPL Dataflow](https://msdn.microsoft.com/en-us/library/hh228603(v=vs.110).aspx)，还有 [Akka.NET](http://getakka.net/)，如果你现在想在 .NET 搞 actor 和/或 消息传递，我推荐你去尝试下它们。）

另一方面，Midori 拥抱多级别的隔离，从由于软件隔离，比 Windows 的线程还要轻量的进程开始。更粗粒度的隔离也是可行的，以域（domains）的形式，再加上对不受信任或逻辑上分离的代码托管的硬件保护双保险。在早期的日子里，我们当然也希望能达到更细的粒度 —— 受 [E 的 vats 概念](http://www.erights.org/elib/concurrency/vat.html)的启发，对于进程消息泵我们已经开始这种抽象了 —— 但不确信如何才能安全地实现它。所以我们在这里停下来等待。但这正好给了我们对于鲁棒、高性能和安全的消息传递基础所需要的。

对这种架构的讨论中重要的是[无共享（shared nothing）](https://en.wikipedia.org/wiki/Shared_nothing_architecture)的概念，这是 Midori 作为核心的操作原则。无共享架构对可靠性、消除单点故障非常有用，但它们对并发安全也同样有用。如果你不共享任何东西，那就不会有机会具备竞争条件！（这有点撒谎，通常情况下还是不充分的，我们迟点会看到。）

有意思的是，在我们跟这些扭打在一起的时候，Node.js 也正在开发。异步、无阻塞、进程范围的单个事件循环的核心思想是非常相似的。也许 2007 到 2009 之间空气中飘散着甜美的味道。事实上，许多这样的特性在[事件循环并发](https://en.wikipedia.org/wiki/Event_loop)中是很常见的。

这形成了整个并发模型绘制于其上的画卷。我已经在[异步一切](https://github.com/ZiJing6/blogging-about-midori/blob/master/asynchronous_everything.md)文章中讨论了这点。但这里还有更多……

### 为什么不在这儿停下来？

这是一个很合理的问题。不需要更多的东西只基于以上的工作已经足以建立一个非常健壮的系统了，而且我得说，系统经过多年的冲击，上述的基础经受住了时间的考验，并且相对于下面到来的东西（语法方面的）远远没有什么变化。就在这时离开它是值得称赞的。实际上，事后看来，我相信在这停下来将是一个合理的“V1”故事。

然而，一些东西促使我们推进更多：

* 没有进程内的并行。尤其是任务和数据并行的缺失。这对刚从 .NET 的 Task 和 PLINQ 编程模型过来的人来说很痛苦。我们有大量的地方潜在等待被解锁的并发性，如图像解码、多媒体管道、FRP（函数式响应式编程？）渲染栈、浏览器、语音识别等等。Midori 的一个顶层目标就是解决掉并发怪兽，虽然许多并行由于进程模型是“免费”的，但缺乏任务和数据并行是很痛苦的。

* 进程间的所有消息都需要 RPC 封送处理，因此不能共享大对象。缺乏任务并行的一个解决方案是将所有东西都建模成进程。需要一个任务吗？那就加一个进程。在 Midori 中，进程能用来干这事，它的成本是足够低的。但这样做需要封送数据。这不仅是一个昂贵的操作，不是所有的类型都是可封送的，这严重限制了可并行的操作。

* 事实上，一个现有的[“交换堆”](http://read.seas.harvard.edu/cs261/2011/singularity.html)已经为了缓冲区被开发了出来，松散地基于线性的概念。为了避免封送大缓冲区，我们已经有了一个在进程间交换它们的系统，无需作为 RPC 协议的一部分进行复制。这个想法似乎足够有用，足以泛化并提供更高层的数据结构。

* 甚至尽管由于上述的单一消息循环模型没有了数据竞争，由于多个异步活动飞来飞去并交织在一起，进程内的竞争条件还是存在的。await 模型的好处是至少在源代码中，交织是可见可审计的；但它们还仍然可以触发并发错误。我们看到了用语言和框架来帮助开发人员正确实现的机会。

* 最后，我们也有一个模糊的愿望，希望在系统中有更多的不变性。当然这样可以对并发安全有帮助，但我们认为语言也应该帮助开发人员将现有的常见模式通过构建获得正确性。我们也看到如果编译器能信任不变性，存在性能优化的机会。

我们回到学术界和 ThinkWeek 的论文中寻找灵感。这些方法，如果以一种雅致的方式结合起来，似乎能给我们提供必要的工具，不仅提供安全的任务和数据并行，而且还提供更细粒度的隔离、不变性和工具，以至有可能解决一些进程内的竞争条件。

所以我们 fork 了一份 C# 的编译器，来到了工地。

### 模型

在这节，我会重新有点失序地（适当地）安排故事。首先我会介绍我们花了几年功夫最终完成的系统的样子，用“教程风格”，而不是从我们如何完成的少混乱的历史开始。我希望这能给出对系统更加简明的了解。然后，我会给出完整的历史账目，包括在深深触动我们的那个出现之前的几十个系统，

我们从 C# 的类型系统开始，并加入了两个关键的慨念：许可（permission）和所有权（ownership）。

#### 许可

第一个关键的概念是*许可*。

任何一个引用都可以有一个许可，而它决定了你能对这个引用对象干什么：

* mutable（可变）：目标对象（图）能够通过平常的方式变更。
* readonly（只读）：目标对象（图）能被读取但不能被变更。
* immutable（不可变）：目标对象（图）能被读取并且*永远都不会*被变更。

一个[子类型关系](https://en.wikipedia.org/wiki/Subtyping)意味着你能够隐式将 mutable 或 immutable 转换为 readonly。换句话说，mutable &lt;: readonly 和 immutable &lt;: readonly。

例如：

```csharp
Foo m = new Foo(); // mutable by default.

immutable Foo i = new Foo(); // cannot ever be mutated.
i.Field++; // error: cannot mutate an immutable object.

readonly Foo r1 = m; // ok; cannot be mutated by this reference.
r1.Field++; // error: cannot mutate a readonly object.

readonly Foo r2 = i; // ok; still cannot be mutated by this reference.
r2.Field++; // error: cannot mutate a readonly object.
```

这些保证是由编译器强制执行的，且必须经过[验证](https://en.wikipedia.org/wiki/Typed_assembly_language)。

默认情况下，如果没有特别声明，int、string 等基元类型是不可变的，而所有的其他类型是可变的。这在几乎所有的情况下保留了现有的 C# 语义。（也就是，C# 编译器没有做什么特别的改变。）这是有争议的，但实际上是系统的一个很酷的方面。它有争议，是因为最小授权原则将导致你应该选择 readonly 作为默认。它酷，是因为你能够采用任何 C# 代码，并开始在有价值的地方点滴增加许可。如果我们决定更激进地与 C# 决裂 —— 回头看来我们应该这样做 —— 那么打破兼容性，选择更安全的默认项是个正确的选择；但对于我们规定的 C# 兼容性目标，我想我们做出了正确的决定。

这些许可也能作用在方法上，以指示这参数该如何使用：

```csharp
class List<T> {
    void Add(T e);
    int IndexOf(T e) readonly;
    T this[int i] { readonly get; set; }
}
```

调用者需要足够的许可来调用一个方法：

```csharp
readonly List<Foo> foos = ...;
foos[0] = new Foo(); // error: cannot mutate a readonly object.
```

可以使用委托类型和 lambda 来声明类似的东西，例如：

```csharp
delegate void PureFunc<T>() immutable;
```

这意味着符合 PureFunc 接口的 lambda 表达式只能闭合（close over）不变的状态。

请注意这突然变得多么强大！这个 PureFunc 正是我们并行任务想要的。正如我们一会儿会看到的，仅仅这些简单的概念就足以使很多那些 PFX 抽象变得安全。

默认情况下，许可是“深（deep）”的，这样它们就可以应用到能达到的整个对象图。这与泛型的相互作用是显而易见的，然而，为了你能够，举个例子，结合深和浅的许可：

```csharp
readonly List<Foo> foos = ...;             // a readonly list of mutable Foos.
readonly List<readonly Foo> foos = ...;    // a readonly list of readonly Foos.
immutable List<Foo> foos = ...;            // an immutable list of mutable Foos.
immutable List<immutable Foo> foos = ...;  // an immutable list of immutable Foos.
// and so on...
```

尽管这也能正常工作，而且看起来很显而易见，但人类就是很难将事情做对！

对于高级用户，我们也有一个方法来写参数化许可的泛型类型。这绝对需要对高度泛型代码内部的深入把握，否则会被系统 90% 的用户所忽略：

```csharp
delegate void PermFunc<T, U, V, permission P>(P T, P U, P V);

// Used elsewhere; expands to `void(immutable Foo, immutable Bar, immutable Baz)`:
PermFunc<Foo, Bar, Baz, immutable> func = ...;
```

我还得说明，为了方便起见，你可以将一个类型标记为 immutable，以指示所有这个类型的实例都是不可变的。这确实是最流行的特性之一。在系统开发结束时，我估计系统中大概所有类型中的 1/4 - 1/3 是被标记为不可变的。

```csharp
immutable class Foo {...}
immutable struct Bar {...}
```

有一个有趣的扭转。正如我们将看到的，readonly 习惯性被称为 readable（可读），而完全是另一个东西。但在我们离开 Midori 后并努力尝试将这些概念纳入 C# 时，我们决定尝试并统一它们。这就是我在这里介绍的。唯一的障碍是，readonly 被给予了一个稍微不同的含义。作用在字段上时，readonly 现在的含义是“值不能被改变”；当值是指针时，readonly 现在并不会影响引用的对象图。而在这个新的模型中，它会。考虑到我们预计引入一个可选的标志：----strict-mutability，这是可以接受的，只需用 readonly mutable，有点绕，就能回到了旧的行为上。这对我来说不是什么血色交易 —— 特别由于 C# 中开发人员以为 readonly 是深度的（现在它会是了）是很常见的一个 bug，很明显在脑袋中跟 const 混淆了。

#### 所有权

第二个关键的概念是*所有权*。

一个引用能给它一个所有权声明，就像能给一个许可一样：
* isolated：目标对象（图）形成一个状态（state）的无别名（unaliased）可达的闭包。

例如：

```csharp
isolated List<int> builder = new List<int>();
```

不像许可，许可指出对一个引用执行什么操作是合法的，所有权声明告诉我们关于给出的对象图的重要的别名使用属性。一个隔离的图只有单单一个“指入引用”，指向对象图中的根对象，而没有“指出引用”（除了不可变对象引用之外，那是被允许的。=）。

这幅图可能能帮你理解这个概念：  
![隔离图](http://joeduffyblog.com/assets/img/2016-11-30-15-years-of-concurrency.isolated-bubble.jpg)

针对一个隔离的对象，我们可以原位（in-place）更改它：

```csharp
for (int i = 0; i < 42; i++) {
    builder.Add(i);
}
```

且/或摧毁原先的引用并将所有权转让给一个新的引用：

```csharp
isolated List<int> builder2 = consume(builder);
```

在这里编译器会将 builder 标记为未初始化的，尽管如果它存储在堆中有可能存在多个别名会指向它，因此这样的分析永远都做不到刀枪不入。在这种情况下，原先的引用会被置为 null，以免有安全陷阱。（这是为了更自然地集成到现有的 C# 的类型系统中作出的许多妥协中的一个例子。）

将隔离拆除也是可能的，就是拿回一个原先的 List&lt;int>：

```csharp
List<int> built = consume(builder);
```

这就实现了一种对安全并发有用的线性形式 —— 对象可以安全地切换，包括buffer 的交换堆这样的特例在内 —— 也支持像 builder 这样的模式，为强不变性奠定了基础。

要想知道为什么这跟不变性扯上关系，要注意到我们之前正好跳过了一个不可变的对象是怎样创建的。为了安全起见，类型系统需要证明不存在当时指向那个对象的其他的可变引用，而且之后也永远不会存在。谢天谢地，那就正是 isolated 能为我们做的！

```csharp
immutable List<int> frozen = consume(builder);
```

或者更简洁点，你倾向于这样做：

```csharp
immutable List<int> frozen = new List<int>(new[] { 0, ..., 9 });
```

在某种意义上，我们已经将我们的隔离泡（见上文）整个都变成绿色的了。  
![全绿隔离图](http://joeduffyblog.com/assets/img/2016-11-30-15-years-of-concurrency.immutable-bubble.jpg)

在幕后，对类型系统的增强是 isolated 和所有权分析。我们很快会见到实践中的更多的这种形式，不过首先有一个简单的印象：这个 List&lt;int> 构造函数的所有输入都是 isolated —— 指的是在这个例子中用 new[] 构造出来的数组 —— 因此得出的 List&lt;int> 也是如此。

事实上，任一个只使用 isolated 和/或 immutable 输入并被推断为 readonly 类型的表达式可被隐式升级到 immutable 的；又，一个相似的表达式，推断为 mutable 类型的，可被升级为 isolated。这意味着使用原有的表达式创建新的 isolated 和 immutable 是很直接的。

这里的**安全**也依赖于环境权限（ambient authority）和泄露构造（leaky construction）的消除。

#### 无环境权限

Midori 的一个原则是消除[环境权限](https://en.wikipedia.org/wiki/Ambient_authority)，这允许[基于能力的安全](https://github.com/ZiJing6/blogging-about-midori/blob/master/objects_as_secure_capabilities.md)，然而以一种微妙的方式也对不变性和下面要提到的安全并发抽象是很必要的。

要知道为什么，让我们看看之前提到的 PureFunc 例子。这给了我们一种局部推断 lambda 表达式捕获的状态的方法。一种我们渴望的特性是函数只接受 immutable 的输入并形成[引用透明（Referential transparency）](https://en.wikipedia.org/wiki/Referential_transparency)的结果，这解锁了许多[创新的编译器优化](https://github.com/ZiJing6/blogging-about-midori/blob/master/safe_native_code.md)并更容易推断代码。

然而，如果可变的静态东西还存在，就会存在 PureFunc 实际上不 pure 的诅咒！

例如：

```csharp
static int x = 42;

PureFunc<int> func = () => x++;
```

从类型系统的观点来看，这个 PureFunc 函数没捕获到状态，因此它遵循不可变捕获需求。（说：我们能“看到” x++，所以拒用这个 lambda，这个说法很有诱惑力，但这个 x++ 可能深藏在一系列虚调用中发生，那对我们是不可见的。）

所有的副作用需要暴露给类型系统。过去的几年里，我们探索额外的声明来表示“这个方法会可变地访问静态变量”；然而，mutable 许可已经是我们处理这种情况的方法了，而且感觉上跟 Midori 采用的对于环境权限整体的立场是一致的。

因此，我们排除了所有的环境副作用操作，替代为利用能力对象。这明显覆盖了I/O操作 —— 所有I/O在我们的系统中都是异步的RPC —— 同时甚至 —— 某种程度是从根本上 —— 意味着即使只是获取当前时间，或者生成一个随机数，都需要一个能力对象。这让我们以类型系统能看到的方式来对副作用进行建模，并同时收获能力对象带来的其他好处。

这意味着所有的静态变量必须是不可变的。 这本质上将 C# 的 const 关键字带给了所有的静态变量：

```csharp
const Map<string, int> lookupTable = new Map<string, int>(...);
```

在 C# 中，const 只限制于基元数据常量，像 int、bool 和 string。我们的系统扩展了相同的能力到任意类型，像 list、map、……、真正所有的类型。

这正是有趣的地方。正如 C# 现在 const 的概念一样，我们的编译器在编译时推断所有的这些对象，并将它们冻结到生成的二进制镜像的只读段中。感谢类型系统的保证，保证了不可变真的是不可变，这样干就不存在运行时出问题的风险了。

冻结有两个吸引人的性能影响。首先，我们可以跨多个进程共享页面（page），减少了总体内存使用量和 TLB 压力。（例如，作为 map 的 lookup table 被自动共享给跨所有使用这个库的程序。）第二，我们能够消除所有类的构造方法的访问，用常量偏移来代替它，这使整个操作系统的代码大小减少了超过了 10%，以及相关的速度改进，特别是启动时间。

可变的静态变量毫无疑问是昂贵的！

#### 无泄漏构造

这带来了我们要修补的第二个“洞”：泄露的构造。

泄露构造指的是任一个构造函数在构造完成之前就将 this 共享了出来。即使它是自己构造函数里“非常后面”的地方共享也是如此，因为继承和构造链，这样不能保证是安全的。

那么，为什么构造泄露很危险？主要因为它们给其他相关方暴露了[部分构造的对象](http://joeduffyblog.com/2010/06/27/on-partiallyconstructed-objects/)，不仅那些对象的不变性值得怀疑，特别是构造可能会失败，而且它们还造成了不变性的风险。

在我们这个特定的情况下，我们如何能知道在创建一个新的可能是不可变的对象之后，没有别的人隐秘地拥有一个可变的引用呢？在这种情况下，将这个对象打上 immutable 的标签是一个类型漏洞。

我们完全禁止了所有的泄露构造。秘密是什么呢？一种特别的许可，init，这意味着目标对象正在进行初始化，从而不服从常规的规则。例如，它意味着字段还不能保证已经被赋值，非空性还未确保，并且对该对象的引用也不能转换为所谓的“顶级”权限， readonly 。任何构造函数默认都有这个权限，并且你不能覆盖它。我们还自动在特定区域使用 init 机制，以保证语言能够更加无缝地工作，就像在对象初始化器中一样。

这会导致一个不好的后果：默认情况下，你不能从构造函数中调用其他实例方法。（说实话，这在我看来是件好事，因为这意味着你不用顾虑还未完全构造的对象，不会意外地[从构造函数中调用其他虚函数](https://www.securecoding.cert.org/confluence/display/cplusplus/OOP50-CPP.+Do+not+invoke+virtual+functions+from+constructors+or+destructors)，等等）。在大多数情况下，这个问题都能变通解决。但是，对于那些真正需要在构造函数中调用实例方法的情形，我们允许将方法标记为 init 来让它们则拥有该许可。

#### 形式化及许可

尽管上面这些直觉上是合理的，但在这些场景背后有一个形式化的类型系统。

在即将将它作为系统核心时，我们跟 MSR 合作来证明这种方式的完整性，特别是 isolated，并在 [OOPSLA' 12](http://dl.acm.org/citation.cfm?id=2384619) 中发表了这篇论文（也在 [MSR 技术报告](http://research-srv.microsoft.com/pubs/170528/msr-tr-2012-79.pdf)中免费提供）。虽然论文是在最终模型固化下来的前些年发表的，但那时大多数关键的设想已经成型并且进行得很顺利。

然而，作为一个简单的思维模型，我总是从子类型和替代的角度思考问题。

事实上，一旦通过这种方式建模，对于类型系统的大多数启示就很自然地“瓜熟蒂落”。readonly 是 “头等许可”， mutable 和 immutable 都可以隐式地转换过去。转换到 immutable 是一个微妙的过程，需要 isolated 状态来保证遵守不变性需求。从那里起，所有的常见的启示开始出现，包括[替代（substitution）](https://en.wikipedia.org/wiki/Liskov_substitution_principle)，[协变（variance）](https://en.wikipedia.org/wiki/Covariance_and_contravariance_(computer_science))，以及它们对于转换、覆盖和子类型的各种影响。

这形成了一个二维方阵，一个维度是经典观念中的“类型”，另一个是“许可”，这样所有的类型能够转换为 readonly 对象。如下图所示：

![许可方阵](http://joeduffyblog.com/assets/img/2016-11-30-15-years-of-concurrency.lattice.jpg)

没有这些形式化知识的帮助，系统显然也能跑。然而，我已经受够了[这些年来因为类型系统陷阱带来的让人提心吊胆又诡异的安全问题](https://www.microsoft.com/en-us/research/wp-content/uploads/2007/01/appsem-tcs.pdf)，所以走得更远点并做形式化不仅能帮助我们更好地理解我们的系统，还能让我们夜里能睡个好觉。

#### 这如何带来安全并发呢？

掌握了新的类型系统，我们现在可以回头重新探访 PFX 抽象，并把它们都弄成安全的。

我们必须建立的核心特性是，当一个 activity（活动） 对一个给定的对象有 mutable 权，这个对象必须不能同时被任何其他的 activity 访问。请注意我正谨慎地在用“activity”这个术语。现在，可以想象它等同于“task（任务）”，尽管我们随时会回头看这个细微的地方。也请注意我说了“对象”；那也是一种粗暴的简化，因为对于某些像数组这样的数据结构，简单地保证 activity 对重叠的区域没有 mutable 权就足够了。

超越这些不允许的东西，它事实上允许一些有趣的模式。举例说，任意数目的并发活动可以共享对同一个对象的 readonly 的访问。（这有点像读写锁，只是没有任何的锁，也没有运行时开销。）记住我们可以将 mutable 转换为 readonly，这意味着，对于给定的有 mutable 访问的的活动，我们可以捕获了一个有 readonly 许可的对象来做 fork/join 并行，只要在这个 fork/join 操作过程中，修改器（mutator）被临时暂停了。

或者，看代码：

```csharp
int[] arr = ...;
int[] results = await Parallel.Fork(
    () => await arr.Reduce((x, y) => x+y),
    () => await arr.Reduce((x, y) => x*y)
);
```

仅仅读一下这段代码，我们就能知道它并行计算一个数组的求和以及乘积。这段代码是没有数据竞争的。

怎么做到的呢？这个例子中的 Fork API 使用了许可来强制了所需要的安全性：

```csharp
public static async T[] Fork<T>(params ForkFunc<T>[] funcs);

public async delegate T ForkFunc<T>() readonly;
```

让我们一块块分开看这段代码。Fork 简单地使用一个 ForkFunc 数组。因为 Fork 是静态的，我们无需担心它会危险地捕获状态。但 ForkFunc 是个委托，可以通过实例方法和 lambda 表达式满足，这两者都可以闭合（close over）状态。通过将 this 位置标记为 readonly，我们将捕获限制为 readonly；因此，虽然在上面的例子中 lambda 表达式能捕获 arr，但它们不能改变它。就是如此。

也要注意内部的 Reduce 方法也能并行地运行，感谢 ForkFunc！显然，所有熟悉的 Parallel.For，Parallel.ForEach 以及它们伙伴们，能享受类似的待遇，类似的安全。

结果是大多数 fork/join 模式，我们可以保证改变状态的方法被暂停的，也这样工作。例如，所有的 PLINQ 可以这样表现，具有完全的无数据竞争。这是我一直以来的用例。

实际上，我们现在能引入了[自动并行](https://en.wikipedia.org/wiki/Automatic_parallelization)！有几种方法可以这样做。一种是永不提供不用 readonly 声明提供保护的 LINQ 操作，这是我倾向的办法，查询操作会带来改动是荒谬的。不过另外的方法也是可能的。一种是提供重载 —— 一组供给 mutable 操作，另一组供给 readonly 操作 —— 然后编译器的重载裁定会根据类型检查选择最小许可的那个。

如先前所述，任务甚至比这样还简单：

```csharp
public static Task<T> Run<T>(PureFunc<T> func);
```

这接受了我们前面的老朋友，被保证引用透明的 PureFunc。因为任务没有类似 fork/join 和数据并行伙伴那样的结构化的生命周期，我们甚至不能捕获 readonly 的状态。记住，上述例子能工作的一个技巧是修改器（mutator）被临时暂停了，而这在非结构化的任务并行中是不能保证的。

那么，如果一个任务要处理可变的状态改怎么办呢？

