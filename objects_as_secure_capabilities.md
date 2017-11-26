# 作为安全能力（Capabilities）的对象

> 原文：[Objects as Secure Capabilities](http://joeduffyblog.com/2015/11/10/objects-as-secure-capabilities/)

[上篇文章中](https://github.com/ZiJing6/blogging-about-midori/blob/master/a_tale_of_three_safeties.md)，我们看到了 Midori 是如何在类型、内存和并发安全性的基础上构建的。这次，我们将看到这如何允许使用一些新的安全的方法。也即是，它让我们的系统消除了[环境权限和访问控制](https://en.wikipedia.org/wiki/Ambient_authority)，支持将[能力（capabilities）](https://en.wikipedia.org/wiki/Capability-based_security)编织到系统和它代码的构造中。与许多我们的其他原则一样，这种保证是通过编程语言和它的类型系统“从根子上”实现的。

## 能力（Capabilities）

首先最重要的是：到底什么是能力？

在我们大多数人都知道和喜爱的安全系统中，譬如 UNIX 和 Windows 中，允许做某事的权限是基于身份的，通常是以用户和用户组的形式。某些受保护的对象如文件和系统调用有附加到它们的访问控制，限制了哪些用户和组可以使用它们。在运行时，操作系统根据这些访问控制，通过使用环境标记如什么用户正在运行当前的进程，来检查请求的操作是否允许。

为了说明这个概念，请考虑一个对开放 API 的一个简单的 C 调用：

```c
void main() {
	int file = open("filename", O_RDONLY, 0);
	// Interact with `file`...
}
```

