# 安全的本机代码（Safe Native Code）

> 原文：[Safe Native Code](http://joeduffyblog.com/2015/12/19/safe-native-code/)


在我的[第一篇 Midori 文章](https://github.com/ZiJing6/blogging-about-midori/blob/master/a_tale_of_three_safeties.md)中，我描述了安全如何成为我们做的所有东西的基础。我提到我们用安全代码搭建了一个操作系统，而且与用 C 和 C++ 写的操作系统如 Windows 和 Linux 相比仍然保持竞争力。在许多方面，系统架构扮演了一个关键的角色，我会在以后的文章中继续讨论怎么做到的。但是，在基础上，一个经常能从原本“托管”的、类型和内存安全的代码获得本机代码性能的优化的编译器，是我们最重要的武器之一。在这篇文章中，我会描述一些对我们成功至关重要的关键的理解和技术。

## 概述

当人们想到 C#、Java 以及相关的语言，他们通常想到的是[即时（Just-In-Time(JIT)）编译](https://en.wikipedia.org/wiki/Just-in-time_compilation)。特别是在 Midori 开始时的 2000 年代中期。但 Midori 是不同的，从一开始就使用了更加类似 C++ 的[提前（Ahead-Of-Time(AOT)）编译](https://en.wikipedia.org/wiki/Ahead-of-time_compilation)。

跟 C 和 C++ 相比，AOT 编译托管的[垃圾收集代码](https://en.wikipedia.org/wiki/Garbage_collection_(computer_science))提出了一些独特的挑战。因此，很多 AOT 的努力并没有达到跟本机同行对等的水平。.NET 的 NGEN 技术就是一个很好的例子。实际上，.NET 中的大多数工作都是专门针对启动时间这个目标；这当然是一个关键的指标，但当你正在构建一个操作系统和之上的所有一切，启动时间只是勉强触及到了表面而已。

在 8 年的过程中，我们已经可以显著缩小我们的 C# 版本的系统和经典的 C/C++ 系统之间的差距，在基本代码的质量上，不管是大小还是速度，将 Midori 的性能与现有负载比较，已经很少称为决定的因素。事实上，还有些反直觉的事情出现了。语言、运行时、框架和编译器一同设计的能力 —— 在一个领域的折衷以获得其他领域的优势 —— 在关于程序的语义上给编译器提供了比以往任何时候更多的符号信息，所以，我敢说，能在不少的情形下超过 C 和 C++ 的性能。

在深入之前，我得提醒你。架构决定 —— 像[异步一切](https://github.com/ZiJing6/blogging-about-midori/blob/master/asynchronous_everything.md)和零拷贝 IO（很快会提到）—— 更多在“整个系统”的层面缩小差距。特别是我们更少 GC 饥饿的写系统代码的方式。但高度优化的编译器的基础，知道和利用安全的优点，是我们成果中至关紧要的东西。

我得坦率地指出，与我们同时，外部世界在这一领域也取得了相当大的进展。[Go](https://golang.org/) 在系统性能和安全性之间有一个优雅的界限。[Rust](http://rust-lang.org/) 就是纯粹的超赞。[.NET Native](https://msdn.microsoft.com/en-us/vstudio/dotnetnative.aspx) 和相关的 [Android Runtime](https://en.wikipedia.org/wiki/Android_Runtime) 项目已经用一种更多限制的方式给 C# 和 Java 带来良好的 AOT 体验，作为一种“静默的”优化技术，以避免移动应用因为 JIT 时造成的卡顿。最近，我们正在致力于通过 [CoreRT 项目](https://github.com/dotnet/corert)将 AOT 带到更广泛的 .NET 环境中。通过这一努力，我希望我们能够将下面的一些经验教训带到现实世界中。由于打破了微妙的平衡，我们能走得多远还需要观望。我们花了好几年才让一切东西和谐工作起来，以数十人年为单位，然而，将知识转授也需要花费很多时间。

