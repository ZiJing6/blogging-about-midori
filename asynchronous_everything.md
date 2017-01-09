# 异步一切

Midori 是由很多超轻量级、细粒度进程构成的，它们通过强类型的消息传递接口连接彼此。通常见到的经典程序是那种单个的、庞大的进程 —— 也许有内部的多线程。但我们取替之以一大堆的小进程，形成了自然的、安全的和大量的自动并行。同步阻塞直接就是不允许的，这意味着不夸张地说所有的东西都是异步的：所有的文件和网络 IO，所有的消息传递，和所有的像跟其他异步工作项会合的“同步”活动项。结果造就系统超高并发，用户输入响应良好，并且非常容易扩展。但就如同你想象的那样，这当然也带来了令人着迷的挑战。

## 异步编程模型

Midori 的异步编程模型表面上很像 C# 的 async/await。

这当然不是巧合。我是 [.NET 任务库](https://en.wikipedia.org/wiki/Parallel_Extensions)的架构师和主程。作为 Midori 并发架构师，我不得不承认我对这个眼前这个 .NET Release 的成功有些偏见。然而即使是我也清楚我们之前的工作不能照样在 Midori 上奏效，所以开始着手这段经年的旅程。但在我们前进的同时，我们也与 C# 团队紧密合作，以将一些 Midori 的做法带回到这个行进中的语言。并且早在 C# 着眼之前的一年多，就已经开始使用一种 async/await 的变体。我们并没有将所有 Midori 的优势都带给 .NET，有些劣势已经显露出来了，大多数都是关于性能的方面。一想到我再也回不到过去及时将 .NET 的 Task 弄成 struct，我就想死。

但我得超越自己。要进行一个很长的旅程才能接触到现在这点，让我们从最开始的地方启程吧。

## Promises

在我们异步模型的核心是一个叫 [promises](https://en.wikipedia.org/wiki/Futures_and_promises) 的技术。现在，这个概念已经相当普遍了。但就像等会你会看到的那样，我们使用 promises 的方式更有趣些。我们受 [E 系统](https://en.wikipedia.org/wiki/E_(programming_language))的影响很深。也许跟现在流行的大多数异步框架最大的不同是我们完全不玩虚的，在我们系统里面连一个同步的 API 都没有。

异步模型的第一版用的是显式的回调（callbacks）。这东西用过 Node.js 的都知道。主要想法就是你得到的任何一个操作的 Promise<T>，最终会给你一个结果 T （或出错了）。这个 T 可能由内部的异步处理甚至在远程的某个地方跑出来的，不过使用它的就不需要关心了。它们只是把 Promise<T> 当作普通的类型处理，去索取的时候，必须将 T 交出来。

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

这就是[并发](https://en.wikipedia.org/wiki/Concurrent_computing#Concurrent_programming_languages)[数据流](https://en.wikipedia.org/wiki/Dataflow_programming)编程。它很好用，因为一系列操作的真值依赖（true dependencies）组织起系统活动的调度。传统的系统不是因为真值依赖，而是因为[假值依赖](https://en.wikipedia.org/wiki/Data_dependency)经常发生故障，像程序突然在深层的异步 IO 调用中出了问题，但调用者却不知情。

实际上，你 Windows 的屏幕经常卡住变白可能就是因为这个引起的。我永远不会忘记几年前找到导致 Outlook 挂起的主要原因之一的那份报告。一个经常使用的 API 偶尔会尝试访问网络上的打印机以枚举附加的字体。它将这些字体缓存下来，这样隔一段时间才需访问打印机一次，隔一段无法预测的时间。结果，这种“好”的表现让开发人员觉得从 UI 线程调用它是没问题的。测试的时候也没出什么
问题（嗯，可能开发人员用很值钱的电脑，在接近完美的网络下工作）。悲催的是，当网络卡爆时，结果就出现一个 10 秒的挂起白屏跟一个旋转圆圈鼠标。一直到现在，在我用过的每一个操作系统中，还是会有这种问题。

上面例子的问题在于开发者调用 API 时，高延迟有没显现出来的可能性。当调用被深深地埋在调用栈深处，标记位虚函数调用等，那就更难显现出来了。在 Midori 里，所有的异步性是在类型系统表现出来的，这种问题不可能会发生，因为类似的 API 必然会返回一个 promise 。当然，开发者仍然可以干些荒唐事（像在 UI 线程里面搞个无限循环），但要搞砸已经难得多了。特别是跟 IO 相关的话。

你不想数据流链继续下去了？没问题。

```csharp
p.WhenResolved(
    ... as above ...
).Ignore();

```

这被证明有一点反模式，它通常是你正对共享状态作修改的一个信号。

这个 Ignore 得稍微解释一下。我们的语言不许你忽略返回值，除非你显式表明要这样做。若你意外地忽略一些或很多重要的东西（例如：一个异常），这个特定的 Ignore 方法还会加入一些诊断信息来帮助调试。

最后我们为常用的模式加入了一堆帮助方法重载和 API：

```csharp
// Just respond to success, and propagate the error automatically:
Promise<U> u = p.WhenResolved((T t) => { ... the T is available ... });

// Use a finally-like construct:
Promise<U> u = p.WhenResolved(
    (T t) => { ... the T is available ... },
    (Exception e) => { ... a failure occurred ... },
    () => { ... unconditionally executes ... }
);

// Perform various kinds of loops:
Promise<U> u = Async.For(0, 10, (int i) => { ... the loop body ... });
Promise<U> u = Async.While(() => ... predicate, () => { ... the loop body ... });

// And so on.
```

这当然远不是新点子了。[Joule](https://en.wikipedia.org/wiki/Joule_(programming_language)) 和 [Alice](https://en.wikipedia.org/wiki/Alice_(programming_language)) 语言甚至有内置的语法让类似上面的笨重的回调传递变得让人更好受点。

但它还是让人没法忍。这编程模型扔掉了一大堆熟悉的编程语言构造，像循环。

这非常非常非常差！它会导致回调困境（callback soup）、非常多深层次的嵌套，而且经常在需要保证正确性的真正重要的代码里出现。想象一下你在一个磁盘驱动程序里面，看到这样的的代码：

```csharp
Promise<void> DoSomething(Promise<string> cmd) {
    return cmd.WhenResolved(
        s => {
            if (s == "...") {
                return DoSomethingElse(...).WhenResolved(
                    v => {
                        return ...;
                    },
                    e => {
                        Log(e);
                        throw e;
                    }
                );
            }
            else {
                return ...;
            }
        },
        e => {
            Log(e);
            throw e;
        }
    );
}
```

根本不可能搞清楚会发生什么。很难知道这各种各样的 return 到底 return 到哪里，什么异常没有被处理，而且非常容易导致重复代码（例如上面的 error 分支），因为经典的代码块范围已经不适用了。上帝保佑，你需要的只是一个循环。而这正是一个磁盘驱动 —— 最需要可靠性的那块！

## 进入 Async 和 Await

