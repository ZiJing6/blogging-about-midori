# 作为安全能力（Capability）的对象

> 原文：[Objects as Secure Capabilities](http://joeduffyblog.com/2015/11/10/objects-as-secure-capabilities/)

[上篇文章中](https://github.com/ZiJing6/blogging-about-midori/blob/master/a_tale_of_three_safeties.md)，我们看到了 Midori 是如何在类型、内存和并发安全性的基础上构建的。这次，我们将看到这如何允许使用一些新的安全的方法。也即是，它让我们的系统消除了[环境权限和访问控制](https://en.wikipedia.org/wiki/Ambient_authority)，支持将[能力（capability）](https://en.wikipedia.org/wiki/Capability-based_security)编织到系统和它代码的构造中。与许多我们的其他原则一样，这种保证是通过编程语言和它的类型系统“从根子上”实现的。