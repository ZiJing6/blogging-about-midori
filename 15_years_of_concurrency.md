# 并发 15 年

在[三种安全](https://github.com/ZiJing6/blogging-about-midori/blob/master/a_tale_of_three_safeties.md)中，我们讨论了三个种类的安全：类型、内存和并发。在这篇后续的文章中,我们将深入最后一个,也许也是最难的一个。首先是并发安全性将我引导到 [Midori](https://github.com/ZiJing6/blogging-about-midori/blob/master/README.md) 项目的，之前我在 .NET 和 c++ 的并发模型上花了多年的时间，让我最终加入进去。在这期间我们构建了一些伟大的的东西，让我非常自豪。也许某种程度更有趣的是，在该项目已经过去几年之后，对这一段经历的反思。

在今年的早些时间，我尝试了大概 6 次来写这篇文章，我很激动最后终于能分享它。我希望这对对这个领域感兴趣的每个人都有用，特别是在这方面进行积极创新的人。虽然代码示例和经验教训都深深根植于 C#、.NET 和 Midori 项目，但我已经试图将这些想法一般化，以便它们不受编程语言的影响。希望你们喜欢。

## 背景

在 21 世纪的大多数时间里，我的工作都是找出如何将并发交到开发者手中，这工作从微软 [CLR 团队](https://en.wikipedia.org/wiki/Common_Language_Runtime)的一个相关的适逢其会的工作开始。

### 适逢其会的开始

回到那个时候，这很大程度是需要构建一个经典线程、锁和同步原语的更好版本，并尽可能尝试将它们凝聚成最佳实践。例如，我们给 .NET 1.1 带来了一个线程池，并利用这个经验改善 Windows 内核、调度器以及它自己线程池的可伸缩性。我们有这个疯狂的 128 个处理器的 [NUMA（非均匀内存访问/非统一内存访问架构）](https://en.wikipedia.org/wiki/Non-uniform_memory_access) 机器，让我们忙于各种深奥的性能挑战。我们开发了[如何将并发弄妥当](http://joeduffyblog.com/2006/10/26/concurrency-and-the-impact-on-reusable-libraries/)的规则 —— 锁级别等等 —— 并进行了[静态分析](https://www.microsoft.com/en-us/research/wp-content/uploads/2008/08/tr-2008-108.pdf)实验。我甚至还写了关于它的[一本书](https://www.amazon.com/Concurrent-Programming-Windows-Joe-Duffy/dp/032143482X)。

为什么并发是第一位的？

一句话，它具有极大的挑战性，完全技术话事，所以充满了乐趣。

