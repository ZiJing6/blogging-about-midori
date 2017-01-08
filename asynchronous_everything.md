# 异步一切

Midori 是由很多超轻量级、细粒度进程构成的，它们通过强类型的消息传递接口连接彼此。通常见到的经典程序是那种单个的、庞大的进程 —— 也许有内部的多线程。
但我们取替之以一大堆的小进程，形成了自然的、安全的和大量的自动并行。同步阻塞直接就是不允许的，
这意味着不夸张地说所有的东西都是异步的：所有的文件和网络 IO，所有的消息传递，和所有的像跟其他异步工作项会合的“同步”活动项。
结果造就系统超高并发，用户输入响应良好，并且非常容易扩展。但就如同你想象的那样，这当然也带来了令人着迷的挑战。

## 异步编程模型

Midori 的异步编程模型表面上很像 C# 的 async/await。

这当然不是巧合。我是 [.NET 任务库](https://en.wikipedia.org/wiki/Parallel_Extensions)的架构师和主程。作为 Midori 并发架构师，
我不得不承认我对这个眼前这个 .NET Release 的成功有些偏见。然而即使是我也清楚我们之前的工作不能照样在 Midori 上奏效，所以开始着手这段经年的旅程。
但在我们前进的同时，我们也与 C# 团队紧密合作，以将一些 Midori 的做法带回到这个行进中的语言。并且早在 C# 着眼之前的一年多，就已经开始使用
一种 async/await 的变体。我们并没有将所有 Midori 的优势都带给 .NET，有些劣势已经显露出来了，大多数都是关于性能的方面。一想到我再也回不到过去
及时将 .NET 的 Task 弄成 struct，我就想死。

但我得超越自己。要进行一个很长的旅程才能接触到现在这点，让我们从最开始的地方启程吧。

## Promises

在我们异步模型的核心是一个叫 [promises](https://en.wikipedia.org/wiki/Futures_and_promises) 的技术。现在，这个概念已经相当普遍了。
但就像等会你会看到的那样，我们使用 promises 的方式更有趣些。我们受 [E 系统](https://en.wikipedia.org/wiki/E_(programming_language))的影响很深。
也许跟现在流行的大多数异步框架最大的不同是我们完全不玩虚的，在我们系统里面连一个同步的 API 都没有。

异步模型的第一版用的是显式的回调（callbacks）。这东西用过 Node.js 的都知道。主要想法就是你得到的任何一个操作的 Promise<T>，最终会给你一个结果 T （或出错了）。
这个 T 可能由内部的异步处理甚至在远程的某个地方跑出来的，不过使用它的就不需要关心了。它们只是把 Promise<T> 当作普通的类型处理，去索取的时候，必须将 T 交出来。

基本的 callback 模型开始像下面一样：

```csharp
Promise<T> p = ... some operation ...;

... 可选地跟 p 里面的操作并发地做一些事情 ...;

Promise<U> u = Promise.When(
    p,
    (T t) => { ... the T is available ... },
    (Exception e) => { ... a failure occurred ... }
);
```

后来我们将静态方法改成了实例方法：

```csharp
Promise<U> u = p.WhenResolved(
    (T t) => { ... the T is available ... },
    (Exception e) => { ... a failure occurred ... }
);
```

注意这个 promise 链。操作的 callback 要不返回一个类型为 U 的值，要不根据情况扔出一个异常。然后 u promise 的使用者再干同样的活，一而再，再而三。


