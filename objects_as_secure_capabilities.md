# 作为安全能力（Capabilities）的对象

> 原文：[Objects as Secure Capabilities](http://joeduffyblog.com/2015/11/10/objects-as-secure-capabilities/)

[上篇文章中](https://github.com/ZiJing6/blogging-about-midori/blob/master/a_tale_of_three_safeties.md)，我们看到了 Midori 是如何在类型、内存和并发安全性的基础上构建的。这次，我们将看到这如何允许使用一些新的安全的方法。也即是，它让我们的系统消除了[环境权限和访问控制](https://en.wikipedia.org/wiki/Ambient_authority)，支持将[能力（capabilities）](https://en.wikipedia.org/wiki/Capability-based_security)编织到系统和它代码的构造中。与许多我们的其他原则一样，这种保证是通过编程语言和它的类型系统“从根子上”实现的。

## 能力（Capabilities）

首先最重要的是：到底什么是能力？

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

而在另一方面，[基于能力的安全](https://en.wikipedia.org/wiki/Capability-based_security)，不是同样地依赖于全局的权威。它使用一种称为“不可伪造令牌（unforgeable tokens）”来表示执行特权操作的能力。不管决定是如何做出的 —— 这有一整个复杂的策略管理和涉及社会和人类行为授权的主题 —— 如果软件不打算进行某种操作，它根本就不会收到需要进行操作的令牌。而且因为令牌是不可伪造的，程序甚至不能尝试这个操作。在像 Midori 这样的系统中，类型安全意味着程序不仅不能执行操作，而且在编译时就已经被抓出来。

