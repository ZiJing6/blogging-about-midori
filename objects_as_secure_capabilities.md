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

它不精确。它依赖程序不可见的环境状态。你无法简单地审核来查看这操作的影响。你只需要知道 open 是如何工作的。但由于它的不精确性，它很容易出错，而在这里错误就因为这安全漏洞。具体的说，很容易欺骗一个程序代表某用户做一些她永远不打算去做的事情。这被称之为[“混淆代理人问题”](http://c2.com/cgi/wiki?ConfusedDeputyProblem)。你需要做的全部就是欺骗 shell 或程序成冒充的超级用户，然后你就像在家里一样自由了。

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