# 安全的本机代码（Safe Native Code）

> 原文：[Safe Native Code](http://joeduffyblog.com/2015/12/19/safe-native-code/)


在我的[第一篇 Midori 文章](https://github.com/ZiJing6/blogging-about-midori/blob/master/a_tale_of_three_safeties.md)中，我描述了安全如何成为我们做的所有东西的基础。我提到我们用安全代码搭建了一个操作系统，而且与用 C 和 C++ 写的操作系统如 Windows 和 Linux 相比仍然保持竞争力。在许多方面，系统架构扮演了一个关键的角色，我会在以后的文章中继续讨论怎么做到的。但是，在基础上，一个经常能从原本“托管”的、类型和内存安全的代码获得本机代码性能的优化的编译器，是我们最重要的武器之一。在这篇文章中，我会描述一些对我们成功至关重要的关键的理解和技术。

## 概述

当人们想到 C#、Java 以及相关的语言，他们通常想到的是[即时（Just-In-Time(JIT)）编译](https://en.wikipedia.org/wiki/Just-in-time_compilation)。特别是在 Midori 开始时的 2000 年代中期。但 Midori 是不同的，从一开始就使用了更加类似 C++ 的[提前（Ahead-Of-Time(AOT)）编译](https://en.wikipedia.org/wiki/Ahead-of-time_compilation)。

跟 C 和 C++ 相比，AOT 编译托管的[垃圾收集代码](https://en.wikipedia.org/wiki/Garbage_collection_(computer_science))提出了一些独特的挑战。因此，很多 AOT 的努力并没有达到跟本机同行对等的水平。.NET 的 NGEN 技术就是一个很好的例子。实际上，.NET 中的大多数工作都是专门针对启动时间这个目标；这当然是一个关键的指标，但当你正在构建一个操作系统和之上的所有一切，启动时间只是勉强触及到了表面而已。

在 8 年的过程中，我们已经可以显著缩小我们的 C# 版本的系统和经典的 C/C++ 系统之间的差距，在基本代码的质量上，不管是大小还是速度，将 Midori 的性能与现有负载比较，已经很少称为决定的因素。事实上，还有些反直觉的事情出现了。语言、runtime、框架和编译器一同设计的能力 —— 在一个领域的折衷以获得其他领域的优势 —— 在关于程序的语义上给编译器提供了比以往任何时候更多的符号信息，所以，我敢说，能在不少的情形下超过 C 和 C++ 的性能。

在深入之前，我得提醒你。架构决定 —— 像[异步一切](https://github.com/ZiJing6/blogging-about-midori/blob/master/asynchronous_everything.md)和零拷贝 IO（很快会提到）—— 更多在“整个系统”的层面缩小差距。特别是我们更少 GC 饥饿的写系统代码的方式。但高度优化的编译器的基础，知道和利用安全的优点，是我们成果中至关紧要的东西。

我得坦率地指出，与我们同时，外部世界在这一领域也取得了相当大的进展。[Go](https://golang.org/) 在系统性能和安全性之间有一个优雅的界限。[Rust](http://rust-lang.org/) 就是纯粹的超赞。[.NET Native](https://msdn.microsoft.com/en-us/vstudio/dotnetnative.aspx) 和相关的 [Android Runtime](https://en.wikipedia.org/wiki/Android_Runtime) 项目已经用一种限制更多的方式给 C# 和 Java 带来良好的 AOT 体验，作为一种“静默的”优化技术，以避免移动应用因为 JIT 时造成的卡顿。最近，我们正在致力于通过 [CoreRT 项目](https://github.com/dotnet/corert)将 AOT 带到更广泛的 .NET 环境中。通过这一努力，我希望我们能够将下面的一些经验教训带到现实世界中。由于打破了微妙的平衡，我们能走得多远还需要观望。我们花了好几年才让一切东西和谐工作起来，以数十人年为单位，然而，将知识转授也需要花费很多时间。

首先的首先，让我们先快速回顾一下：本机语言和托管语言的区别到底是什么？

### 相同之处是什么？

我鄙视“本机和托管”这种错误的二分法，所以我得为使用了这种说法而道歉。读了这篇文章之后，我希望能让你相信这是一个连续的东西。C++ 现在比以往变得更安全，与之对应的是 C# 的性能。有趣的是到底有多少经验教训被直接应用到我们团队这些天正在致力的安全 C++ 工作上。

因此，让我们从考虑哪些是一样的开始。

所有[龙书](https://en.wikipedia.org/wiki/Principles_of_Compiler_Design)的基本主题跟本机代码一样适用于托管代码。

基本上，编译代码是一种平衡行为，一方面为目标架构平台生成最有效率的指令序列，以快速地执行程序；另一方面，为目标架构平台生成最小的指令编码，以将程序紧凑地存储并有效地使用目标设备的内存系统。你最喜爱的编译器上有无数的旋钮，它们根据你的场景在两方面中拨动。在移动设备，你可能想要更小的代码，而在多媒体工作站上，你可能想要最快的代码。

选择托管代码不会改变这里的任何一点。你仍然需要同样的灵活性。你在 C 和 C++ 编译器中使用来达成这个目的的技术绝大程度上跟使用在安全代码上的一样。

你需要一个很棒的[内联处理器（inliner）](https://en.wikipedia.org/wiki/Inline_expansion)。你想要[公共子表达式消除（common subexpression elimination (CSE)）](https://en.wikipedia.org/wiki/Common_subexpression_elimination)，[常量传播和折叠（constant propagation and folding）](https://en.wikipedia.org/wiki/Constant_folding)，[强度降低（strength reduction）](https://en.wikipedia.org/wiki/Strength_reduction)，以及一个优秀的[循环优化器（loop optimizer）](https://en.wikipedia.org/wiki/Loop_optimization)。现在，你可能想要使用[静态单一赋值形式（static single assignment form (SSA)）](https://en.wikipedia.org/wiki/Static_single_assignment_form)，以及一些独特的 SSA 优化如[全局值编号（global value numbering）](https://en.wikipedia.org/wiki/Global_value_numbering)（虽然在到处使用 SSA 时你需要小心工作集和编译器吞吐量）。你需要对你很重要的目标架构的特定机器依赖的优化器，包括[寄存器分配器（register allocators）](https://en.wikipedia.org/wiki/Register_allocation)。最终，你将需要一个全局分析器来做过程间优化，链接时代码生成以扩展跨过程的过程间优化，一个应对现代处理器（SSE、NEON、AVX 等等）的[矢量化器（vectorizer）](https://en.wikipedia.org/wiki/Automatic_vectorization)，以及定义良好的[剖析引导优化（profile guided optimizations (PGO)）](https://en.wikipedia.org/wiki/Profile-guided_optimization)来在真实世界的场景中应用上面说的这些技术。

虽然面对安全语言会在你的行进路上扔出一些独特有趣的香蕉球 —— 我下面会提及 —— 你仍需要所有的标准的优化编译器的东西。

我不想这样说，但将所有这些东西都做好是“桌面筹码”。早在 2000 年代中，我们不得不手写所有的东西。谢天谢地，现在你可以得到一个极好的现成的优化编译器，如 [LLVM](http://llvm.org/)，其中大部分已经经过实战测试，准备就绪，而且并为你的改进做好了准备。

### 不同之处是什么？

不过当然，有不同之处，很多。要不然这篇文章就没啥意思了。

不同之处更多在于，在抛给优化器的代码和数据结构中，你所期望的“形状（shape）”会有什么不一样。这些形状来自于不同的指令序列、代码中不存在 C++ 等效的逻辑操作（如更多的边界检查）、数据结构布局差异（如额外的对象头或接口表），以及在大多数情况下，更多的支持 runtime 数据结构。

相比在譬如 C 中的朴素的数据类型，在大多数托管语言中，对象有“更多的东西”。（注意 C++ 的数据结构并没有你想象的那么朴素，可能跟你直觉不一样，更加接近 C#。）在 Java 中，每个对象都在它对象头有个 vtable 指针。在 C# 中，大多数也如此，虽然 struct 不是。GC 可以强加额外的布局限制，例如填充和几个字节来完成它的簿记。注意这些没有一个是限定在托管语言的 —— C 和 C++ 分配器也可以插入它们自己的字节，而且当然，很多 C++ 对象也带有 vtable —— 然而可以公平地说，大多数 C 和 C++ 实现在这些方面往往是倾向于更经济的。在大多数情况下，是出于文化因素而不是硬性的技术因素。在堆中添加了好几千个对象，特别是当你的系统是像 Midori 那样用许多使用隔离堆的小进程搭建起来时，对象增加很快。

在 Java 中，你有更多的虚拟调用，因为默认方法就是虚拟的。在 C# 中，谢天谢地方法默认是非虚拟的。（我们甚至将类默认设置成 sealed。）太多的虚拟调用可以全部拧起来内联，这是一个对小函数的关键优化。在托管语言中，你倾向于使用更多的小函数，基于两个理由：1) 属性，和 2) 高级程序员倾向于过度抽象。

虽然很少正式地提及，有一个“ABI”（[应用程序二进制接口（Application Binary Interface）](https://en.wikipedia.org/wiki/Application_binary_interface)）用于组织代码和 runtime 的交互。ABI 就是轮胎接触路面的地方。它是诸如调用约定、异常处理、以及最突出的，机器码中的 GC 清单这样的东西。这_不是_托管代码独有的！C++ 有一个“runtime”因此也有一个 ABI。它只是主要由 header、像分配器的库等的组合，相比于经典的运行时是不可妥协的（而且在 JIT 情形下是相当笨重的） C# 和 Java 虚拟机，能够更加透明地链接到程序中。这样想对我很有帮助，因为这跟 C++ 的同构性立刻变得明显起来。

