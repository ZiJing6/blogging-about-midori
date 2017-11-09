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

你需要一个很棒的[内联处理器（inliner）](https://en.wikipedia.org/wiki/Inline_expansion)。你想要[公共子表达式消除（common subexpression elimination (CSE)）](https://en.wikipedia.org/wiki/Common_subexpression_elimination)，[常量传播和折叠（constant propagation and folding）](https://en.wikipedia.org/wiki/Constant_folding)，[强度降低（strength reduction）](https://en.wikipedia.org/wiki/Strength_reduction)，以及一个优秀的[循环优化器（loop optimizer）](https://en.wikipedia.org/wiki/Loop_optimization)。现在，你可能想要使用[静态单赋值形式（static single assignment form (SSA)）](https://en.wikipedia.org/wiki/Static_single_assignment_form)，以及一些独特的 SSA 优化如[全局值编号（global value numbering）](https://en.wikipedia.org/wiki/Global_value_numbering)（虽然在到处使用 SSA 时你需要小心工作集和编译器吞吐量）。你需要对你很重要的目标架构的特定机器依赖的优化器，包括[寄存器分配器（register allocators）](https://en.wikipedia.org/wiki/Register_allocation)。最终，你将需要一个全局分析器来做过程间优化，链接时代码生成以扩展跨过程的过程间优化，一个应对现代处理器（SSE、NEON、AVX 等等）的[矢量化器（vectorizer）](https://en.wikipedia.org/wiki/Automatic_vectorization)，以及定义良好的[剖析引导优化（profile guided optimizations (PGO)）](https://en.wikipedia.org/wiki/Profile-guided_optimization)来在真实世界的场景中应用上面说的这些技术。

虽然面对安全语言会在你的行进路上扔出一些独特有趣的香蕉球 —— 我下面会提及 —— 你仍需要所有的标准的优化编译器的东西。

我不想这样说，但将所有这些东西都做好是“桌面筹码”。早在 2000 年代中，我们不得不手写所有的东西。谢天谢地，现在你可以得到一个极好的现成的优化编译器，如 [LLVM](http://llvm.org/)，其中大部分已经经过实战测试，准备就绪，而且并为你的改进做好了准备。

### 不同之处是什么？

不过当然，有不同之处，很多。要不然这篇文章就没啥意思了。

不同之处更多在于，在抛给优化器的代码和数据结构中，你所期望的“形状（shape）”会有什么不一样。这些形状来自于不同的指令序列、代码中不存在 C++ 等效的逻辑操作（如更多的边界检查）、数据结构布局差异（如额外的对象头或接口表），以及在大多数情况下，更多的支持 runtime 数据结构。

相比在譬如 C 中的朴素的数据类型，在大多数托管语言中，对象有“更多的东西”。（注意 C++ 的数据结构并没有你想象的那么朴素，可能跟你直觉不一样，更加接近 C#。）在 Java 中，每个对象都在它对象头有个 vtable 指针。在 C# 中，大多数也如此，虽然 struct 不是。GC 可以强加额外的布局限制，例如填充和几个字节来完成它的簿记。注意这些没有一个是限定在托管语言的 —— C 和 C++ 分配器也可以插入它们自己的字节，而且当然，很多 C++ 对象也带有 vtable —— 然而可以公平地说，大多数 C 和 C++ 实现在这些方面往往是倾向于更经济的。在大多数情况下，是出于文化因素而不是硬性的技术因素。在堆中添加了好几千个对象，特别是当你的系统是像 Midori 那样用许多使用隔离堆的小进程搭建起来时，对象增加很快。

在 Java 中，你有更多的虚拟调用，因为默认方法就是虚拟的。在 C# 中，谢天谢地方法默认是非虚拟的。（我们甚至将类默认设置成 sealed。）太多的虚拟调用可以全部拧起来内联，这是一个对小函数的关键优化。在托管语言中，你倾向于使用更多的小函数，基于两个理由：1) 属性，和 2) 高级程序员倾向于过度抽象。

虽然很少正式地提及，有一个“ABI”（[应用程序二进制接口（Application Binary Interface）](https://en.wikipedia.org/wiki/Application_binary_interface)）用于组织代码和 runtime 的交互。ABI 就是轮胎接触路面的地方。它是诸如调用约定、异常处理、以及最突出的，机器码中的 GC 清单这样的东西。这*不是*托管代码独有的！C++ 有一个“runtime”因此也有一个 ABI。它只是主要由 header、像分配器的库等的组合，相比于经典的运行时是不可妥协的（而且在 JIT 情形下是相当笨重的） C# 和 Java 虚拟机，能够更加透明地链接到程序中。这样想对我很有帮助，因为这跟 C++ 的同构性立刻变得明显起来。

真正的大头是数组边界检查。传统的方式在访问前，不管是加载还是存储，检查索引是在数组的边界范围内。这是一个额外的字段读取、比较、以及条件分支。[分支预测](https://en.wikipedia.org/wiki/Branch_predictor)现在可以做得很好了，然而如果你干了更多的活，那么你一定要付出更多，这是纯物理定律。有意思的是，我们使用 C++ 的 array_view&lt;T> 来干这个，因此开销是相同的。

与此相关，也有在 C++ 中不存在的 null 检查。举个例子，如果你在 C++ 的 null 对象指针上执行方法调用，你最终会运行那个函数。如果函数尝试访问 this，它被绑定到 [AV](https://en.wikipedia.org/wiki/Segmentation_fault)，但在 Java 和 .NET 中，在这种情况下，编译器被要求（每个规范）显式检查并抛出一个异常，在调用发生之前。这些小的分支也会累加起来。我们在优化的构建中消除了这种检查，转而支持 C++ 语义。

在 Midori 中，我们默认使用溢出检查选项编译。这跟现在的 C# 不同，在 C# 中你必须显式传入 /checked 标记来达成这个行为。在我们的经验中，意料不到的溢出被捕获的数目和出乎意料的程度，是非常值得这种不方便和成本的。但这也意味着我们的编译器需要在理解如何消除不必要的溢出检查上做的非常好。

静态变量在 .NET 和 Java 中是非常昂贵的。比你想象的更厉害。它们是可变的，因此不能保存在只读的镜像段中以在不同的进程中共享。而且注入到结果代码中的惰性初始化检查的数量是超出天际突破想象的。在 .NET 中从 preciseinit 切换到 beforefieldinit 语义有点帮助，因为初始化检查不需要在每次访问一个静态成员时都需要发生 —— 只需要访问正在使用中的静态变量 —— 但与使用常量和用心的全局初始化结合的精心编写的 C 程序相比，还是令人不爽的。

最后一个主要的领域特定于 .NET 的：struct。虽然 struct 有助于缓解 GC 压力，因此对于大多数程序是一件好事，但它们也有一些微妙的问题。例如，CLI 在它们初始化时指定了令人奇怪的行为。即如果在构造过程中一个异常发生了，整个结构槽必须保持初始化为 0。结果是大多数编译器都使用了防御性的复制。另一个例子是编译器必须在任何时候你在一个只读的 struct 上调用函数时都执行防御性复制。struct 到处被复制是相当常见的，对你的计算时间周期损害特别厉害，因为通常时间都花在 memcpy 上。我们有很多技术来解决这个问题，而且有趣的是，我很肯定当所有的这些被说出来并搞定，我们的代码质量会比 C++ 的*更好*，考虑到它的 RAII，复制构造函数，构析函数等等的惩罚。

## 编译架构

我们的架构涉及三大主要组件：

* [C# 编译器](https://github.com/dotnet/roslyn)：执行词法分析、语句分析和语义分析。最终将 C# 文本源代码转化为基于 [CIL](https://en.wikipedia.org/wiki/Common_Intermediate_Language) 的[中间表示（intermediate representation (IR)）](https://en.wikipedia.org/wiki/Intermediate_language)。
* [Bartok](https://en.wikipedia.org/wiki/Bartok_(compiler))：接受给定的 IR 进行高层次的基于 MSIL 的分析、转换和优化，最终将 IR 降低到更接近于更具体的机器表示。例如，使用 Bartok 处理完 IR 后，泛型就消失了。
* [Phoenix](https://en.wikipedia.org/wiki/Phoenix_(compiler_framework))：接受上面的更低层次的 IR，并尽可能地处理它。这是大部分“油门到底”的优化发生的地方。输出是机器代码。

这里跟 Swift 的编译器设计，尤其是 [SIL](http://llvm.org/devmtg/2015-10/slides/GroffLattner-SILHighLevelIR.pdf) 的相似之处是显而易见的。.NET Native 项目也多少照抄了这个架构。坦率地说，大多数针对高层次语言的 AOT 编译器都这样做。

在大多数地方，编译器的内部表示使用了[静态单赋值形式（SSA）](https://en.wikipedia.org/wiki/Static_single_assignment_form)。SSA 一直保留直到编译的很后期。这促进并改善了前面提到的很多经典编译器优化的使用。

这个架构的目标包括：

* 促进快速的原型和实验。
* 生成跟商业 C/C++ 编译器同等水平的高质量机器代码。
* 支持调试优化的机器代码以提高生产力。
* 促进基于采样或/和检测的剖析引导优化代码。
* 适合自托管（self-host）：
    * 编译出来的编译器是足够快的。
    * 足够快以让编译器开发人员乐于使用它。
    * 当编译器出现情况时容易调试问题。

最后，一个简短的警告。我们尝试过很多东西，我没法把它们都记起来。早在我参与其中前 Bartok 和 Phoenix 就已经存在好几年了。Bartok 是托管语言研究的沃土 —— 从优化到 GC 到软件事务内存 —— 而 Phoenix 是为了取代 Visual C++ 编译器的。所以不管怎样，我是没法讲出全部完整的故事的。但我会尝试做到最好。

## 优化

让我们深入探讨一些特定的经典编译器优化领域，扩展到覆盖安全代码。

### 消除边界检查

C# 数组是带边界检查的。Midori 的也是。虽然在常规的 C# 代码中消除多余的边界检查非常重，在我们的情况下就更是如此了，因为即使系统的最底层也使用边界检查的数组。例如，在 Windows 和 Linux 内核内部，你会见到的是 int*，在 Midori 中你会见到的是 int[]。

要看看边界检查是什么样子，考虑这个简单的例子：

```csharp
var a = new int[100];
for (int i = 0; i < 100; i++) {
    ... a[i] ...;
}
```

这里是作为存在边界检查的循环内部访问生成的机器码的例子：

```asm
; 首先，将数组的长度放入 EAX：
3B15: 8B 41 08        mov         eax,dword ptr [rcx+8]
; 如果 EDX >= EAX，访问越界；跳到 error：
3B18: 3B D0           cmp         edx,eax
3B1A: 73 0C           jae         3B28
; 否则，访问正常；计算元素地址并赋值：
3B1C: 48 63 C2        movsxd      rax,edx
3B1F: 8B 44 81 10     mov         dword ptr [rcx+rax*4+10h],r8d
; ...
; 错误处理；只是调用 runtime helper 进行 throws:
3B28: E8 03 E5 FF FF  call        2030
```

若你在每个循环迭代中都做这个簿记（bookkeeping），你不会得到非常紧凑的循环代码。那你当然不会有任何向量化它的希望。所以，我们花了很多时间和精力来尝试消除这样的检查。

在上面的例子中，人类一眼就能看出，没有必要进行边界检查。然而，对于编译器来说分析并不是那么简单。它需要证明关于范围的各种事实。它还需要知道 a 不是可能在循环体中某刻被修改的别名。这个问题很快就会变得令人惊奇的如此困难。

我们系统有多层的边界检查消除。

首先要注意的是，CIL 严格约束了优化器在某些领域的精确性。例如，越界访问数组会抛出一个 IndexOutOfRangeException，跟 Java 的 ArrayOutOfBoundsException 类似。而且 CIL 规范它应该精确地在抛出异常的地方这样做。我们稍后会看到，我们的错误模型更加宽松。它是基于快速失败（fail-fast）的并允许导致不可避免的失败比较其他情况“更快”地发生的代码移动。没有这个，我们的手就会被束缚在我要讨论的很多事情上。

在 Bartok 的最高层上，IR 仍然是相对接近程序输入的。所以一些简单的模式可以被匹配和消除。在更进一步底层化之前，[ABCD 算法](http://www.cs.virginia.edu/kim/courses/cs771/papers/bodik00abcd.pdf) —— 一种基于 SSA 的直接值范围分析 —— 开始运行来消除更常见的模式，使用比模式匹配更好的更原则性的方法。由于过程内的长度和控制流事实传播，我们也能够在全局分析阶段使用 ABCD。

接下来，Phoenix 循环优化器开始着手处理东西。这层执行各种循环优化，以及跟本节内容最相关的范围分析。例如：

* 循环实体化：这种分析实际上创建循环。它识别出若表示为循环会更理想的代码的重复模式，并在有收益时重写它们为循环。这包括展开（ unroll）手工（hand-rolled）的循环以便向量器能够处理它们，即使它们可能稍后会被重新展开（re-unroll）。
* 循环克隆、展开以及版本化：这种分析创建循环的多个复制以进行专门化。包括循环展开、创建向量化的循环的特定架构的版本，等等。
* [归纳（Induction）](https://en.wikipedia.org/wiki/Induction_variable)范围优化：这是我们在这节最关注的阶段。除了做经典的例如拓宽（widening）的归纳变量优化之外，的它使用归纳范围分析来消除不必要的检查。作为这一阶段的副产品，边界检查被消除和将它们提升到循环之外来合并。

这些原则性的分析比前面所展示的更有威力。例如，有一些方法可以编写可以很容易“欺骗”前面所讨论的更基本的技术的早期循环代码：

```csharp
var a = new int[100];

// Trick #1: use the length instead of constant.
for (int i = 0; i < a.length; i++) {
    a[i] = i;
}

// Trick #2: start counting at 1.
for (int i = 1; i <= a.length; i++) {
    a[i-1] = i-1;
}

// Trick #3: count backwards.
for (int i = a.length - 1; i >= 0; i--) {
    a[i] = i;
}

// Trick #4: don't use a for loop at all.
int i = 0;
next:
if (i < a.length) {
    a[i] = i;
    i++;
    goto next;
}
```

你了解了这点。显然在某种程度上，你可以压榨优化器的能力来做任何事情，特别是当你开始在循环内部进行虚拟调用时，其中会丢掉别名信息。显然，当数组的长度不能静态地获知的时候，情况就变得更加困难，如上面例子的 100 所示。然而，如果你能证明循环范围和数组的关系，则所有的这些都不会丢失。这些分析的大部分都需要C# 的数组长度是不可变的事实的特别知识。

在这些日子的最终，优化做得很好，这里是区别：

```asm
; Initialize induction variable to 0:
3D45: 33 C0           xor         eax,eax
; Put bounds into EDX:
3D58: 8B 51 08        mov         edx,dword ptr [rcx+8]
; Check that EAX is still within bounds; jump if not:
3D5B: 3B C2           cmp         eax,edx
3D5D: 73 13           jae         3D72
; Compute the element address and store into it:
3D5F: 48 63 D0        movsxd      rdx,eax
3D62: 89 44 91 10     mov         dword ptr [rcx+rdx*4+10h],eax
; Increment the loop induction variable:
3D66: FF C0           inc         eax
; If still < 100, then jump back to the loop beginning:
3D68: 83 F8 64        cmp         eax,64h
3D6B: 7C EB           jl          3D58
; ...
; Error routine:
3D72: E8 B9 E2 FF FF  call        2030
```

以及下面的，完全优化的，没有边界检查的，循环：

```asm
; Initialize induction variable to 0:
3D95: 33 C0           xor         eax,eax
; Compute the element address and store into it:
3D97: 48 63 D0        movsxd      rdx,eax
3D9A: 89 04 91        mov         dword ptr [rcx+rdx*4],eax
; Increment the loop induction variable:
3D9D: FF C0           inc         eax
; If still < 100, then jump back to the loop beginning:
3D9F: 83 F8 64        cmp         eax,64h
3DA2: 7C F3           jl          3D97
```

有趣的是，当我们使用 C++ 新的 array_view&lt;T> 类型进行同样的运用时，我感到似曾相识。有时候我和我的前 Midori 同事开玩笑，我们注定要在接下来的 10 年里慢慢地、耐心地重复自己的人生了。我知道这听起来很傲慢。但我几乎每天都有这种感觉。

### 溢出检查

如前所述，我们在 Midori 中默认使用 checked 运算（通过 C# 中的 /checked 标志）。这消除了开发人员没有预料到的那些错误，因此代码从溢出中更正过来。当然我们保留了显示的 checked 和 unchecked 作用域构造，以在适当的时候覆盖默认情况，不过这是更可取的，因为程序员声明了他的意图。

不管如何，如你所料，这同时也会降低代码质量。

作为对比，假设我们将两个变量加起来：

```csharp
int x = ...;
int y = ...;
int z = x + y;
```

现在假设 x 在 ECX 中，而 y 在 EDX 中。下面是标准的 unchecked 加操作：

```asm
03 C2              add         ecx,edx
```

或者你想要更花哨一些，那个使用同一 LEA 指令也将结果存储在 EAX 寄存器中，跟很多现代编译器可能会做的那样：

```asm
8D 04 11           lea         eax,[rcx+rdx]
```

嗯，下面是插入了一个边界检查的等效代码：

```asm
3A65: 8B C1              mov         eax,ecx
3A67: 03 C2              add         eax,edx
3A69: 70 05              jo          3A70
; ...
3A70: E8 B3 E5 FF FF     call        2028
```

多出了那些可恶的条件跳转（JO）跟错误处理例程（CALL 2028）。

事实证明，前面提到的许多证明边界检查多余的分析也适用于证明溢出检查是多余的。这都是关于证明关于范围的事实。例如，如果你可以证明[某些检查是由之前的某些检查控制](https://en.wikipedia.org/wiki/Dominator_(graph_theory))的，而且更早的检查是后面检查的超集，那么后面的检查就是不必要的。如果恰好相反 —— 即前面的检查是后面检查的子集，如果后面块的超出之前的那个，你可能要将更强的检查移到程序的前面。

另一个常见的模式是一样或者相似的，算术运算一而再多次出现：

```csharp
int p = r * 32 + 64;
int q = r * 32 + 64 - 16;
```

很显然如果 p 赋值不会溢出，那么 q 肯定也不会。

在现实世界的代码中还有一个神奇的现象经常出现。在同样相邻的代码中同时有边界检查和算术检查是很常见的。设想一些代码从一个数组中读取一组值：

```csharp
int data0 = data[dataOffset + (DATA_SIZE * 0)];
int data1 = data[dataOffset + (DATA_SIZE * 1)];
int data2 = data[dataOffset + (DATA_SIZE * 2)];
int data3 = data[dataOffset + (DATA_SIZE * 3)];
.. and so on ...
```

C# 数组不能有负的边界。如果编译器知道 DATA_SIZE 是足够小的，溢出的计算不会小于 0，那么它可以为边界检查消除范围检查。

还有许多你可以覆盖的其他的模式和特殊情况。但上面示例了一个真正好的与循环优化器集成的范围优化器的力量。它可以覆盖一系列的场景，包括数组边界和算术运算。它需要大量的工作，但最后它是值得的。

### 内联

在大多数情况下，[内联](https://en.wikipedia.org/wiki/Inline_expansion)跟真正的本机代码是相同的。而且是同样的重要。经常是更为重要，因为 C# 开发人员倾向于编写大量小的方法（像属性访问器）。由于本文中的许多主题，要得到小的代码会比在 C++ 中更难 —— 更多的分支、更多的检查等等 —— 因此，实际上大多数托管代码的编译器比本机代码的编译器内联得少得多，或者至少需要做相当不同的调整。这回实际上严重影响和破坏性能。

也有一些习惯性的膨胀的方面。在 MSIL 中 lambda 编码的方式对于一个本机后端编译器是无法理解的，除非这个事实使工程师幡然醒悟。例如，我们有一个优化接受这样的代码：

```csharp
void A(Action a) {
    a();
}

void B() {
    int x = 42;
    A(() => x++);
    ...
}
```

而在内联之后，可以将 B 转化为这样：

```csharp
void B() {
    int x = 43;
    ...
}
```

Action 参数是个 lambda 表达式，如果你知道 C# 编译器是如何将 lambda 编码成 MSIL，就会了解这个技巧是多么困难。作为例子，下面是 B 生成的代码：

```msil
.method private hidebysig instance void
    B() cil managed
{
    // Code size       36 (0x24)
    .maxstack  3
    .locals init (class P/'<>c__DisplayClass1' V_0)
    IL_0000:  newobj     instance void P/'<>c__DisplayClass1'::.ctor()
    IL_0005:  stloc.0
    IL_0006:  nop
    IL_0007:  ldloc.0
    IL_0008:  ldc.i4.s   42
    IL_000a:  stfld      int32 P/'<>c__DisplayClass1'::x
    IL_000f:  ldarg.0
    IL_0010:  ldloc.0
    IL_0011:  ldftn      instance void P/'<>c__DisplayClass1'::'<B>b__0'()
    IL_0017:  newobj     instance void [mscorlib]System.Action::.ctor(object,
                                                                  native int)
    IL_001c:  call       instance void P::A(class [mscorlib]System.Action)
    IL_0021:  nop
    IL_0022:  nop
    IL_0023:  ret
}
```

要获得这个魔术般的结果，需要常量传播 ldftn，识别出委托构造是如何工作的（IL_0017），利用这信息来内联 B 并一起消除 lambda/delegate，然后再主要通过常量传播，折叠算术到用常数 42 做 x 的初始化。这种“掉到”多个不同关注点的优化的组合的情况，我总是发现的它的优雅。

与本机代码一样，剖析引导优化（PGO）方式使我们的内联决策更加有效。

### 结构（Struct）

CLI 结构几乎就跟 C 的结构一样，除了它们不是。CLI 强制了一些引起开销的语义。这些开销几乎总是表现为过度复制。更糟糕的是，这些复制通常在你的程序中是隐藏起来的。这是毫无意义的。因为复制构造和构析，C++ 也有一些真正的问题，通常比我要描述的更糟。

也许最烦人的是，用 CLI 的方式初始化一个结构需要一个防卫性的复制。例如，考虑这个程序，S 的初始函数抛出了一个异常：

```csharp
class Program {
    static void Main() {
        S s = new S();
        try {
            s = new S(42);
        }
        catch {
            System.Console.WriteLine(s.value);
        }
    }
}

struct S {
    public int value;
    public S(int value) {
        this.value = value;
        throw new System.Exception("Boom");
    }
}
```

这个程序的行为必须是值 0 被写到控制台中。在实践中，这意味着赋值操作 s = new S(42) 必须首先在栈上创建一个新的 S 类型的槽，构造它，并且*然后*而且只有在然后才将这个值复制回 s 变量中。对于这样的只有一个 int 的结构，这没什么大不了的。对于大的结构，这意味着要诉诸于 memcpy。在 Midori 中，由于我们的错误模型（以后会讲及更多），我们知道什么方法会 throw，而哪些不会，意味着我们可以在几乎所有情况下避免这种开销。

另一个烦人的情况像下面这样：

```csharp
struct S {
    // ...
    public int Value { get { return this.value; } }
}

static readonly S s = new S();
```

每一次我们读取 s.Value：

```csharp
int x = s.Value;
```

我们都会得到一个本地的副本。这项实际上只会在 MSIL 中才能看出来。这是没有 readonly 的：

```csharp
ldsflda    valuetype S Program::s
call       instance int32 S::get_Value()
```

而这个是有 readonly 的：

```csharp
ldsfld     valuetype S Program::s
stloc.0
ldloca.s   V_0
call       instance int32 S::get_Value()
```

请注意编译器选择使用 ldsfld，再跟一个 ldloca.s，而不是像第一个示例那样用 ldsflda 直接加载地址。由此产生的机器代码甚至更加糟糕。正如我后面会提及的那样，我也不能通过引用来传递结构，必须复制它，也可能出现问题。

在 Midori 中我们解决了这个问题，因为我们的编译器知道不会改变成员的方法。所有的静态变量首先就是不可变的，所以上面的 s 不会需要防御性复制。或者，除此之外，这个 struct 可以被声明为 immutable 的，如下：

```csharp
immutable struct S {
    // As above ...
}
```

或者反正所有的静态值都是不可变的。或者相关的属性或者方法可以被声明为 readable，即它们不能触发变更所以不需要防御性复制。

我提到了通过引用传递。在 C++ 中，开发人员知道通过引用传递大的结构体，用 * 或者 &，来避免过多的复制。我们也同样养成了这样做的习惯。作为例子，我们有 in 参数，如下：

```csharp
void M(in ReallyBigStruct s) {
    // 读取, 但不赋值到, s ...
}
```

我承认我们可能将这个做到了极限，到了那个我们 API 难以忍受的点。如果我能够重来一次，我会回到过去消除 C# 中 class 和 struct 的根本区别。事实证明，指针并没有那么糟糕，对于系统代码，你确实希望深入了解“近”（值）和“远”（指针）之间的区别。我们的确在 C# 中实现了相当于 C++ 中的引用，但这还不够。在我即将到来的深入挖掘我们的编程语言中会有更多相关内容。

### 代码大小（Code Size）

我们在代码大小上压制得很狠。甚至比我知道的 C++ 编译器狠得多。

泛型实例化知识一些花哨的带一些替代的代码的复制和粘贴。很显然，与开发人员实际编写的代码相比，这意味着编译器要处理的代码激增。[我之前已经写过很多关于泛型的性能挑战](http://joeduffyblog.com/2011/10/23/on-generics-and-some-of-the-associated-overheads/)。一个重要的问题是传递闭包问题。.NET 中看起来简单直接的 List&lt;T> 实际上在它的传递闭包种创建了 28 种类型！还没说每个类型种的所有方法。泛型是代码大小快速激增的原因。

我忘记不了我重构我们 LINQ 实现的那天。不像在 .NET 中使用扩展方法的方式，我们在我们集合类型层次的最基类上将所有的 LINQ 操作实现为实例方法。这意味着大约 100 个内嵌类，每个 LINQ 操作都有一个，_对于每一个实例化的集合_！重构这个是在整个 Midori “工作站”操作系统镜像中节省超过 100MB 代码大小的简单办法。是的，100MB！

我们学会了更周到地使用泛型。例如，嵌套在外部泛型中的类型通常不是个好主意。我们还积极共享泛型实例化，甚至比 CLR 所做的还多。即我们还共享 GC 指针在相同位置的值类型泛型。因此，例如考虑一个结构 S：

```csharp
struct S {
    int Field;
}
```

我们会在 List&lt;S> 跟 List&lt;int> 共享相同的代码表示。以及相似的，考虑：

```csharp
struct S {
    object A;
    int B;
    object C;
}

struct T {
    object D;
    int E;
    object F;
}
```

我们会在 List&lt;S> 和 List&lt;T> 中共享实例化。

你也许没有意识到这一点，但 C# 生成保证 struct 有 sequential 布局的 IL：

```msil
.class private sequential ansi sealed beforefieldinit S
    extends [mscorlib]System.ValueType
{
    ...
}
```

因此，我们不能让 List&lt;S> 和 List&lt;T> 与某些例如的 List&lt;U> 共享实例化：

```csharp
struct U {
    int G;
    object H;
    object I;
}
```

因为这个，除了其他原因之外 —— 像给编译器在填充、缓存对齐等等方面更多的灵活性 —— 我们在我们的语言中将 struct 默认设为 auto。事实上，sequential 只在如果你在用 unsafe 代码时才有关系，而在我们的编程模型中，这是不允许的。

在 Midori 中我们不支持反射。原则上，我们有最终完成它的计划，作为一个纯可选的特性。在实践中，我们从来就不需要它。我们发现代码生成从来都是更为适合的方案。通过这样我们最好情况下比 C# 的镜像大小剔掉至少 30%。如果你在系统中跟大多数情况一样保留所有的 MSIL，还会显著更多，即使是在 NGen 和 .NET AOT 方案中。

实际上，我们还删掉了 System.Type 中的一大块。没有 Assembly，没有 BaseType，而且甚至没有 FullName。.NET Framework 的 mscorlib.dll 包含了约 100KB 的净的类型名称。当然，名称是有用，但我们的事件框架利用代码生成来生成在运行时实际需要的那些东西。

在某个时刻，我们意识到我们镜像大小的 40% 是 [vtable](https://en.wikipedia.org/wiki/Virtual_method_table)。我们不停地努力琢磨这个，在这之后，我们仍然有很多改进的余地。

每个 vtable 都使用镜像的空间来保存在调用时使用的虚函数的指针，当然还有运行时的表示。每个拥有 vtable 的对象同时还有一个嵌入它的 vtable 指针。所以，如果你对大小（镜像的和运行时的）很在意，你得注意 vtable。

在 C++ 中，你只会在你的类型是[多态](http://www.cplusplus.com/doc/tutorial/typecasting/)时才会有 vtable。在 C# 和 Java 这样的语言中，即使你不想、不需要或者不去使用它，也还是会有一个 vtable。在 C# 中，至少你可以用一个 struct 类型来去掉它们。我真的很喜欢 Go 的这方面，其中你通过接口来得到类似虚拟调用的东西，而不需要每个类型都有 vtable 的花销；你只需要为你使用的东西付出，在将某个东西硬变成 interface 的时候。

C# 中另一个 vtable 问题是所有的对象都从 System.Object 中继承了三个虚拟方法：Equals、GetHashCode 和 ToString。除了这点，这些方法通常不会用正确的方法干正确的事情 —— Equals 需要反射来用在值类型上，GetHashCode 是不确定的并且标记对象头（或者同步块；迟点会谈更多），而 ToString 没有提供格式化和本地化控制 —— 它们也给每个 vtable 膨胀了三个槽。这可能听起来好像不是太多，但它肯定比没有这种开销的 C++ 多。

我们还剩下的主要苦恼来源是 C# 中的假定，坦率地说大多数 OOP 语言如 C++ 和 Java，[RTTI](https://en.wikipedia.org/wiki/Run-time_type_information) 对向下转换总是可用的。因为上面的那些原因，这在泛型中就特别痛苦。虽然我们积极地共享实例化，但我们不可能完全折叠这些家伙的类型结构，即使不同的实例化往往是相同的，或者至少是非常相似的。如果我可以从头全部再来一次，我会干掉 RTTI。在 90% 情况下，可区分类型联合（type discriminated union）或模式匹配（pattern matching）是更合适的解决方案。

### 剖析引导优化（Profile guided optimizations (PGO)）

我已经提到过[剖析引导优化](https://en.wikipedia.org/wiki/Profile-guided_optimization)（PGO）。这是在几乎所有其他这篇文章中的东西都完全做了之后的最后一公里的关键因素。这让我们的浏览器程序在像 [SunSpider](https://webkit.org/perf/sunspider/sunspider.html) 和 [Octane](https://developers.google.com/octane/) 之类的基准测试中获得了将近 30-40% 的提升。

大多数使用 PGO 的方式跟经典的本机 profiler 一样，不过有两个大的不同。

首先，我们教会了 PGO 关于本文中列出的许多独特的优化，譬如异步栈探查、泛型实例化、lambda、以及更多。跟其他很多事情一样，这里我们可以永远干下去。

其次，除了普通的检查分析（instrumented profiling）外，我们试验了采样分析（sample profiling）。从开发人员的角度来看，这就好得多 —— 它们不需要两次 build —— 而且也让你从真实的、活生生在数据中心跑着的系统上收集数据。一个关于能达到什么样的可能性的好例子来自[这篇 Google-Wide Profiling(GWP) 论文](http://static.googleusercontent.com/media/research.google.com/en/us/pubs/archive/36575.pdf)。

## 系统架构

上面描述的基础都很重要。但一些更具影响力的领域需要更深层次的跟语言、runtime、framework 以及操作系统自身的架构的协同设计和协同进化。我[曾经写过这种“整个系统”方法的巨大好处](http://joeduffyblog.com/2014/09/10/software-leadership-7-codevelopment-is-a-powerful-thing/)。这是一种魔法般的不可思议。

### GC

Midori 是完全垃圾收集的。这是我们整个模型的安全以及生产力的关键因素。实际上，在某个时候，我们有 11 种不同的收集器，每个有它独特的特点。（例如，看[这个研究](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.353.9594&rep=rep1&type=pdf)）。我们有了一些方法来克服常见的问题，如长时间的停顿。我会在将来的文章种详细讨论这个。现在，让我们先坚持代码质量的领域。

一个 top-level 的决定是：_保守_还是_精确_？一个保守的收集齐更容易切入现有的系统，但它会在某些方面造成麻烦。它经常需要扫描更多的堆来完成同样的工作。而且它会错误地保持对象存活。我们觉得这两点对于系统编程环境是都不能接受的。这是一个简单快速的决定：我们追求精确。

然而，精确会让你在代码生成器中付出一些代价。一个精确的收集器需要得到指示去哪里找到它的根集。这根集包括堆中数据结构的字段偏移量，还有栈上的位置，或者还有某些情况下的寄存器。它需要找到这些以便它不会错过一个对象，错误地回收它或者在重新定位期间无法调整指针，这两种情况都会导致内存安全问题。除了 runtime 和代码生成器之间的紧密，没有什么神奇的技巧可以使这变得高效。

