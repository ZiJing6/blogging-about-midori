# 并发 15 年

在[三种安全](https://github.com/ZiJing6/blogging-about-midori/blob/master/a_tale_of_three_safeties.md)中，我们讨论了三个种类的安全：类型、内存和并发。在这篇后续的文章中,我们将深入最后一个,也许也是最难的一个。首先是并发安全性将我引导到 [Midori](https://github.com/ZiJing6/blogging-about-midori/blob/master/README.md) 项目的，之前我在 .NET 和 c++ 的并发模型上花了多年的时间，让我最终加入进去。在这期间我们构建了一些伟大的的东西，让我非常自豪。也许某种程度更有趣的是，在该项目已经过去几年之后，对这一段经历的反思。

在今年的早些时间，我尝试了大概 6 次来写这篇文章，我很激动最后终于能分享它。我希望这对对这个领域感兴趣的每个人都有用，特别是在这方面进行积极创新的人。虽然代码示例和经验教训都深深根植于 C#、.NET 和 Midori 项目，但我已经试图将这些想法一般化，以便它们不受编程语言的影响。希望你们喜欢。

## 背景

在 21 世纪的大多数时间里，我的工作都是找出如何将并发交到开发者手中，这工作从微软 [CLR 团队](https://en.wikipedia.org/wiki/Common_Language_Runtime)的一个相关的适逢其会的工作开始。

### 适逢其会的开始

回到那个时候，这很大程度是需要构建一个经典线程、锁和同步原语的更好版本，并尽可能尝试将它们凝聚成最佳实践。例如，我们给 .NET 1.1 带来了一个线程池，并利用这个经验改善 Windows 内核、调度器以及它自己线程池的可伸缩性。我们有这个疯狂的 128 个处理器的 [NUMA（非均匀内存访问/非统一内存访问架构）](https://en.wikipedia.org/wiki/Non-uniform_memory_access) 机器，让我们忙于各种深奥的性能挑战。我们开发了[如何将并发弄妥当](http://joeduffyblog.com/2006/10/26/concurrency-and-the-impact-on-reusable-libraries/)的规则 —— 锁级别等等 —— 并进行了[静态分析](https://www.microsoft.com/en-us/research/wp-content/uploads/2008/08/tr-2008-108.pdf)实验。我甚至还写了关于它的[一本书](https://www.amazon.com/Concurrent-Programming-Windows-Joe-Duffy/dp/032143482X)。

为什么并发是第一位的？

一句话，它具有极大的挑战性，完全技术话事，所以充满了乐趣。

我一直都是语言的沉迷者。因此我就自然地被学术界几十年深入的研究深深吸引，包括编程语言与运行时共生（特别是 [Cilk](https://en.wikipedia.org/wiki/Cilk) 和 [NESL](https://en.wikipedia.org/wiki/NESL)）、高级类型系统、以及甚至专门的并行硬件体系结构（特别激进的像[连接机（The Connection Machine）](https://en.wikipedia.org/wiki/Connection_Machine)，以及 [MIMD](https://en.wikipedia.org/wiki/MIMD) 超级计算机，那创新超越了我们可信赖的老伙伴[冯·诺依曼体系](https://en.wikipedia.org/wiki/Von_Neumann_architecture)）。

虽然一些非常大的客户真的跑了[对称多处理器（SMP）](https://en.wikipedia.org/wiki/Symmetric_multiprocessing)服务器 —— 是的，我们事实上习惯这样叫它们 —— 我不会说并发是一个非常流行的专门领域。因此当然任何提及那些酷的带研究味的来源都会让我的同事和经理感觉怪怪的。尽管如此，我还是坚持了下去。

尽管有乐趣，我不会说这个时期我们做的工作对随意的观察人员产生了巨大的影响。我们将抽象提高了一点 —— 以便开发者可以调度逻辑的工作项，考虑同步的更高级别等等 —— 但没有任何改变游戏规则的地方。然而，这段时期有助于为之后到来的那些奠定基础，无论是基础上还是社区上，尽管我那时候还不知道。

### 不再有免费的午餐，进入多核

然后一些重大的事情发生了。

2004 年，Intel 和 AMD 告诉我们[摩尔定律](https://en.wikipedia.org/wiki/Moore's_law)，尤其是它的[接近终结](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.87.8775&rep=rep1&type=pdf)。[能量墙挑战](https://www.quora.com/Why-havent-CPU-clock-speeds-increased-in-the-last-5-years)会[严重削弱曾经一直增长的时钟速度改进](http://www.economist.com/technology-quarterly/2016-03-12/after-moores-law)，而那是工业界早已经习惯了的。

突然，管理人员都变得非常关心并发问题。Herb Sutter 在 2005 年的[免费午餐结束了文章](http://www.gotw.ca/publications/concurrency-ddj.htm)中记录了这狂热的投入。如果我们不能让开发者可以编写大量的并行软件 —— 这在历史上是非常困难而且几乎并且不太可能在没有显著降低进入门槛的情况下发生的事 —— 微软和英特尔的业务以及互利互惠的业务模式，都会遇到麻烦。如果硬件不会像往常那样变得更快，软件也不会自动变得更好，人们就没有理由去购买新的硬件和软件。[Wintel](https://en.wikipedia.org/wiki/Wintel) 时代和[安迪·比尔盖茨定律](http://www.forbes.com/2005/04/19/cz_rk_0419karlgaard.html)就结束了，*“安迪给多少，比尔盖茨就拿走多少”*。

或者，顺着这样想下去。

这是“多核”这个词闯入主流的时候，而我们开始设想一个拥有 1024 核处理器或者更前瞻的从[数字信号处理器（DSP）](https://en.wikipedia.org/wiki/Digital_signal_processor)借鉴而来的[“非常多核”架构](https://en.wikipedia.org/wiki/Manycore_processor)的世界，混合了通用和专用的核心，能够承载非常重负荷的功能，例如加密、压缩等等。

另一方面，过了 10 年再回来看，事情并没有完全按照我们设想的那样演变。我们并没有用 1024 个传统的核来跑 PC，虽然[我们的 GPU 已经远超过了这个数](http://www.geforce.com/hardware/10series/titan-x-pascal)，而且我们的确看到了跟以前更多的异构性，特别是数据中心中，在那里 [FPGA 现在负担起大量的关键任务，像加密和压缩](https://www.wired.com/2016/09/microsoft-bets-future-chip-reprogram-fly/)。

在我看来，真正大的错失是移动端。恰恰是当思考能量曲线、密度和异构时，应该能告知我们移动是迫在眉睫的，而且是很大的一块。与其寻求更强大的 PC，我们应该去寻求我们口袋里的 PC。相反，我们自然的本能是抓住过去，“拯救” PC 业务。这是一个经典的[创新者的困境](https://en.wikipedia.org/wiki/The_Innovator's_Dilemma)，虽然在当时当然看起来不像。当然 PC 并没有在一夜间死亡，所以这里的创新也没有浪费，只是在历史的背景下让人感到不平衡。不好意思，我跑题了。

## 使并发更容易

作为一个并发 geek，这就是我一直等待的时刻。几乎一夜之间，为所有这些我一直梦想的创新工作寻找赞助商变得相当容易，因为现在它有了一个真正的非常迫切的业务需要。

简短地说，我们需要：

* 使编写并行代码变得更容易
* 使避开并发陷阱变得更容易
* 使以上两件事“出乎意料”地发生。

我们已经有了线程、线程池、锁以及基本的事件。现在我们应该去向何方呢？

三个特定的项目围绕这个点孵化了出来，并受到了兴趣和人员配置的激励。

### 软件事务内存

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
