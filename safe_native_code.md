# 安全的本机代码（Safe Native Code）

> 原文：[Safe Native Code](http://joeduffyblog.com/2015/12/19/safe-native-code/)


在我的[第一篇 Midori 文章](https://github.com/ZiJing6/blogging-about-midori/blob/master/a_tale_of_three_safeties.md)中，我描述了安全如何成为我们做的所有东西的基础。我提到我们用安全代码搭建了一个操作系统，而且与用 C 和 C++ 写的操作系统如 Windows 和 Linux 相比仍然保持竞争力。在许多方面，系统架构扮演了一个关键的角色，我会在以后的文章中继续讨论怎么做到的。但是，在基础上，一个经常能从原本“托管”的、类型和内存安全的代码获得本机代码性能的优化的编译器，是我们最重要的武器之一。在这篇文章中，我会描述一些对我们成功至关重要的关键的理解和技术。

## 概述

当人们想到 C#、Java 以及相关的语言，他们通常想到的是[即时（Just-In-Time(JIT)）编译](https://en.wikipedia.org/wiki/Just-in-time_compilation)。特别是在 Midori 开始时的 2000 年代中期。但 Midori 是不同的，从一开始就使用了更加类似 C++ 的[提前（Ahead-Of-Time(AOT)）编译](https://en.wikipedia.org/wiki/Ahead-of-time_compilation)。

跟 C 和 C++ 相比，AOT 编译托管的[垃圾收集代码](https://en.wikipedia.org/wiki/Garbage_collection_(computer_science))提出了一些独特的挑战。因此，很多 AOT 的努力并没有达到跟本机同行对等的水平。.NET 的 NGEN 技术就是一个很好的例子。实际上，.NET 中的大多数工作都是专门针对启动时间这个目标；这当然是一个关键的指标，但当你正在构建一个操作系统和之上的所有一切，启动时间只是勉强触及到了表面而已。

