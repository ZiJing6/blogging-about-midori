# 作为安全能力（Capabilities）的对象

> 原文：[Objects as Secure Capabilities](http://joeduffyblog.com/2015/11/10/objects-as-secure-capabilities/)

[上篇文章中](https://github.com/ZiJing6/blogging-about-midori/blob/master/a_tale_of_three_safeties.md)，我们看到了 Midori 是如何在类型、内存和并发安全性的基础上构建的。这次，我们将看到这如何允许使用一些新的安全的方法。也即是，它让我们的系统消除了[环境权限和访问控制](https://en.wikipedia.org/wiki/Ambient_authority)，支持将[能力（capabilities）](https://en.wikipedia.org/wiki/Capability-based_security)编织到系统和它代码的构造中。与许多我们的其他原则一样，这种保证是通过编程语言和它的类型系统“从根子上”实现的。

## 能力（Capabilities）

首先最重要的是：到底什么是能力（capabilities）？

在我们大多数人都知道和喜爱的安全系统中，譬如 UNIX 和 Windows，允许做某事的权限是基于身份的，通常是以用户和用户组的形式。某些受保护的对象如文件和系统调用有附加到它们的访问控制，限制了哪些用户和组可以使用它们。在运行时，操作系统根据这些访问控制，通过使用环境标记如什么用户正在运行当前的进程，来检查请求的操作是否允许。

为了说明这个概念，请考虑一个对开放 API 的一个简单的 C 调用：

```c
void main() {
	int file = open("filename", O_RDONLY, 0);
	// Interact with `file`...
}
```

在内部，这个调用将查看当前进程的标识，还有对应文件对象的访问控制表，然后看情况允许或拒绝调用。有各种机制来模拟用户，如在 UNIX 中的 su 和 setuid 和 Windows 的 ImpersonateLoggedOnUser。但这里主要的关键在于，open 只“知道”如何检查一些全局状态，以了解所请求的行动的安全影响。这里另一个有意思的方面是 O_RDONLY 标志也被传递进来，请求只读的访问，这些因素也进入了授权过程考量。

那，这有什么问题呢？

它不明确。它依赖程序不可见的环境状态。你无法简单地审核来查看这操作的影响。你只需要知道 open 是如何工作的。但由于它的不明确性，它很容易出问题，而在这里错误就意味着安全漏洞。具体的说，很容易欺骗一个程序代表某用户做一些她永远不打算去做的事情。这被称之为[“混淆代理人问题”](http://c2.com/cgi/wiki?ConfusedDeputyProblem)。你需要做的全部就是欺骗 shell 或程序成冒充的超级用户，然后你就像在家里一样自由了。

而在另一方面，[基于能力的安全（Capability-based security）](https://en.wikipedia.org/wiki/Capability-based_security)，不是同样地依赖于全局的权威。它使用一种称为“不可伪造令牌（unforgeable tokens）”来表示执行特权操作的能力。不管决定是如何做出的 —— 这有一整个复杂的策略管理和涉及社会和人类行为授权的主题 —— 如果软件不打算进行某种操作，它根本就不会收到需要进行操作的令牌。而且因为令牌是不可伪造的，程序甚至不能尝试这个操作。在像 Midori 这样的系统中，类型安全意味着程序不仅不能执行操作，而且在编译时就已经被抓出来。

不安全的操作在编译时就被拒绝了，多么酷！

你可能已经猜到，前面假设的 open API，会看起来很不同：

```csharp
void main(File file) {
	// Interact with `file`...
}
```

好吧，显然我们要面对陌生的情况了。这是*极端的*不同。而且我刚刚推卸了责任。是*其他人*必须给出一个 File 对象吗？他们怎么获得这个对象？

官僚的回答是，管他呢，那是调用者的事！但如果他们*的确*拿出了一个 File 对象，那他们一定得被授权得到它，因为在类型安全系统中的对象引用是不可伪造的。策略和授权的问题现在被推到了源头那里，可以说，它们属于那里。

我有点过于简化这个了，因为这回答很可能会引起更多的额问题，跟它已经回答了的相比。让我们继续深挖下去。

所以，再一次，让我们问这个问题：怎样做才能拿到一个 File 对象？

上面的那段代码既不知道也不关心它是从哪里来的。所有它知道的就是通过一个类似 File 的 API，它被给了一个对象。它可能是调用者刚 new 出来的。更可能的是，它是通过跟一个单独的如 Filesystem 或者 Directory 这样的实体协商获得的，而这二者都是能力（capability）对象：

```csharp
Filesystem fs = ...;
Directory dir = ... something(fs) ...;
File file = ... something(dir) ...;
MyProgram(file);
```

现在你可能对我很恼火了。fs 是从哪来的？我要如何从 fs 中获得一个目录，又如何从 dir 中得到一个文件？我只是将所有有趣的主题兜了个圈子，跟打太极一样，什么都没有回答！

事实是哪些就是所有当你尝试设计一个使用能力（capabilities）的文件系统时会遭遇的有趣的问题。你可能不想允许自由地枚举整个文件系统层级，以为如果你获准访问一个 Filesystem 对象 —— 或文件系统的根目录对象 —— 你就可以访问一切可及的东西。那就是当你开始处理能力（capabilities）时你所做的那种思考。你仔细考虑信息的封装和暴露，因为所有你得到的都是用来保证你系统安全的对象。可能你会有个办法，程序请求访问 Filesystem 某处的一些状态，以声明的方式，然后“能力符咒（capability oracle）”决定是否将这些状态交给你。这是我们的应用程序模型所扮演的角色，以及是 main 如何掌握一个程序的清单请求它需要的能力（capabilities）的。从那点起，它就只是对象。关键是在整个系统中没有一个地方你能找得到经典的那种环境权限，因此也没有这种在它们构造时可以“欺骗”的抽象。

Butler Lampson 的经典论文《[保护](http://research.microsoft.com/en-us/um/people/blampson/08-Protection/Acrobat.pdf)》清楚地阐明了一些关键的根本原则，如不可伪造的令牌。某种意义上，我们系统中的每个对象都是它自己的“保护域”。我也喜欢用[能力迷思批驳（Capability Myths Demolished）](http://srl.cs.jhu.edu/pubs/SRL2003-02.pdf)的方式来比较和对照能力（capabilities）和经典的安全模型，如果你想要更多的细节（或错误地认为二者是同构）的话。

Midori 绝不是第一个在以对象能力（object capabilities）作为核心建立的操作系统。实际上，我们从 [KeyKOS](http://www.cis.upenn.edu/~KeyKOS/NanoKernel/NanoKernel.html) 和它的后继者 [EROS](https://en.wikipedia.org/wiki/EROS_(microkernel)) 和 [Coyotos](http://www.coyotos.org/docs/misc/eros-comparison.html) 中得到了重要的启发。这些系统跟 Midori 一样，使用面向对象的方式来提供能力（capabilities）。我们足够幸运地在团队里得到这些项目的一些原先的设计师。

在继续之前，先给出一个警告：有些系统混淆使用了“能力（capability）”这个术语，虽然它们不是真正的能力（capability）系统。[POSIX 定义了一个这样的系统](http://c2.com/cgi/wiki?PosixCapabilities)，[因此 Linux 和 Android 继承了它](https://www.kernel.org/pub/linux/libs/security/linux-privs/kernel-2.2/capfaq-0.2.txt)。虽然 POSIX 的能力（capabilities）是比典型经典的环境状态和访问控制方法好 —— 跟通常的比较允许细粒度的控制 —— 但它们更接近于传统的那种，而不是我们在这里讨论的这种真正的能力（capability）。

## 对象和状态

能力（capabilities）就是对象的一个好处是你可以将现有的面向对象的知识应用到安全和授权的领域中。

因为对象代表了能力（capabilities），所以它们可以如你所希望的那样粗粒度或细粒度。你可以通过组合来制造个新的，或者通过子类来修改现有的那个。依赖关系是通过就像面向对象系统中的那些依赖一样管理：通过封装、共享和对对象的请求引用。你突然可以再安全的领域使用各种[经典的设计模式](https://en.wikipedia.org/wiki/Design_Patterns)。我不得不承认这个想法的简单性对一些人来说是震惊的。

一个基本的概念是[撤销](http://c2.com/cgi/wiki?RevokableCapabilities)。一个对象有一个类型，而我们的系统让你用一个实现替代另一个。这意味着如果你向我请求一个 Clock，我不需要让你再所有时候都能访问时钟，或者访问真正的时钟。而是，我可以给你我自己的 Clock 的子类，它可以委托到真正的那个，并且再一些事件发生之后拒绝你的访问尝试。你要么相信时钟的来源，或者在你不是很确信的时候明确地保证自己的安全。

另一个概念是状态。在我们的系统中，我们禁止了可变的静态内容，在我们的编程语言，在编译时从根子上禁止了它们。就是这样，不仅静态字段只能被写入一次，而且它所引用的整个对象图在构造之后就被冻结了。事实证明，可变的静态内容实际上只是环境权限的一种形式，这种方法可以防止某人从，假设缓存一个 Filesystem 对象到一个全局的静态变量并自由地共享它，这样会搞成一种很像经典安全模型的东西，而这正式我们力求避免的。这在安全并发方面也有很多好处，甚至给我们带来了性能上的好处，因为静态内容只是简单地就变成了富常量对象图，可以被冻结并在不同的二进制文件间共享。

完全消除可变的静态内容改进了我们系统的可靠性，这难以量化，也很难去泛泛而说。这是我最怀念的东西之一。

回想上面我提及的 Clock。这是一个极端的例子，不过是的，那就是这样的，没有全局的函数来读取时间，像 C 的 localtime 或 C# 的 DateTime.Now。要获取时间，你必须显式请求一个 Clock 能力（capability）。这有从整个类和函数中消除不确定性的效果。一个不进行 IO 的静态函数 —— [这在我们类型系统中是我们可以确定的东西（想一下 Haskell 的 monads）](http://research.microsoft.com/apps/pubs/default.aspx?id=170528)—— 现在变成纯函数式的、可记录的、和甚至有时候我们可以在编译时求值的（有点像 steroids 上的  [constexpr](http://en.cppreference.com/w/cpp/language/constexpr)）。

我首先得承认，开发人员经历了一个成熟的过程，在他们在对象能力（capability）系统中学习设计模式的时候。随着时间推移，“大包装”的能力（capabilities）的增长，和/或能力（capabilities）被在不适当的时候请求，变得很常见。例如，想象一个 Stopwatch API。它可能需要 Clock。你需要将 Clock 传递到每一个需要访问当前时间的操作中吗（像 Start 和 Stop）？或者你事先用一个 Clock 实例构造 Stopwatch，从而封装 Stopwatch 对时间的使用，使它更容易传递给其他人（重要的是要认识到，这实质上赋予了接收者读取时间的能力）。另一个例子是，如果你的抽象需要请求 15 个不同的能力（capabilities）来完成工作，那它的构造方法使用一个 15 个对象的平摊列表吗？多么笨拙和烦人的构造方法啊！相反，更好的方法是逻辑地将这些能力（capabilities）分组到不同的对象中，甚至可能使用上下文相关的存储，像 parent 和 children，来让获取它们变得更加容易。

经典的面向对象系统的弱点也从隐藏中暴露出来。例如，向下类型转换（downcasting），意味着你不能完全相信子类化作为[信息隐藏](https://en.wikipedia.org/wiki/Information_hiding)的手段。如果你请求一个 File，而我给你提供一个我自己的从 File 继承过来的，添加了它自己的公共的云相关的函数的 CloudFile，你可能偷偷向下类型转换到 CloudFile 并且可以做我不打算让你做的事。我们通过严格限制类型转换和把最敏感的能力（capabilities）放置在完全不同的计划来解决这个问题……

## 分布式的能力（Distributed Capabilities）

我将简要介绍一下会在未来帖子中覆盖更多的领域：我们的异步编程模型。这个模型形成了我们如何并发和分布式计算、如何进行 IO、以及与我们现在讨论更相关的，能力（capabilities）如何能够扩展到这些关键领域的基础。

在上面的文件系统示例中，我们的系统经常将 Filesystem 引用背后真正的对象寄宿在一起的另一个不同的进程中。就是那样，调用一个方法实际上是分发一个对另一个服务这个调用的进程的远程调用。所以实际上，绝大多数的，虽然不是所有，能力（capabilities）是异步对象；或者更精确地说，不可伪造的令牌允许一个进程跟它们交互，这是我们称之为“最终”能力（capability）的东西。Clock 是个反例。他是我们称之为“即刻”的能力（capability）：那些包装了系统调用，而不是远程调用的东西。但大多数安全相关的能力（capabilities）往往是远程的，因为大多数需要授权的有意思的东西底下都是某种 IO。你很少会需要授权来进行计算。实际上，文件系统、网络栈、设备驱动、图形表面（graphics surfaces），以及更多的东西都是用最终能力（capabilities）的形式表示。

在操作系统中实现整体安全以及我们如何构建分布式的、高度并发安全系统的统一，是我们最大、最具创新性和最重要的成就之一。

我得提一下，跟一般的能力（capabilities）概念一样，类似的概念在 Midori 之前就已经被提倡了。虽然我们没有直接使用这语言，从 [Joule](https://en.wikipedia.org/wiki/Joule_(programming_language)) 语言和之后的 [E](https://en.wikipedia.org/wiki/E_(programming_language)) 语言，为我们奠定了一些非常强大的基础。[Mark Miller 2006 年的博士论文](http://www.erights.org/talks/thesis/markm-thesis.pdf)是这领域的一个宝库。我们有幸与同我共事过的最聪明的人之一密切合作，他恰巧是这两个系统的首席设计师。

## 总结

有太多关于能力（capabilities）的好处要说。类型安全的基础让我们可以做一些大胆的跃进。这形成了一个跟司空见惯的环境权限和访问控制非常不同的系统架构。这系统将安全的分布式计算带到我之前从未设想过的前沿。出现的设计模式真正地最充分地拥抱了面向对象，使用各种突然变得比以往任何更相关的设计模式。

我们从来也没在这个模型上获得很多真实的曝光。相比架构部分的内容，用户交互方面还没有充分开发，像策略管理这种。例如，我很怀疑我们会想问我妈是否她想让程序使用一个时钟。最有可能的是，我们会想让一些能力（capabilities）是自动授权的（如 Clock），而其他的则可以通过组合分组到相关的地方。幸运的是，作为对象的能力（Capabilities-as-objects）给了我们很多已知的设计模式了来这样做。我们的确有整了一些蜜罐，它们中没有一个被黑掉（好吧，至少我们不知道被黑掉了），但我不能确定生成系统的量化安全性。定性上我可以说在系统构建的各个层面都有双保险的安全让我们感觉良好，但我们没有得到在大规模使用来证明它的机会。

在下一篇文章中，我们将深入了解在整个系统中运行的异步模型。这段日子依赖，异步编程是一个热门话题，伴随着 await 出现在 [C#](https://msdn.microsoft.com/en-us/library/hh156528.aspx)、[ECMAScript7](http://tc39.github.io/ecmascript-asyncawait/)、[Python](https://www.python.org/dev/peps/pep-0492/)、[C++](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2014/n4134.pdf) 和更多语言上。和跟所有那些语言一样容易使用的异步，再加上通过消息传递的细粒度的分解到轻量级进程，能够提供高度并发、可靠和高性能的系统。下次见。