# 异步一切

Midori 是由很多超轻量级、细粒度进程构成的，它们通过强类型的消息传递接口连接彼此。通常见到的经典程序是那种单个的、庞大的进程 —— 也许有内部的多线程。但我们取替之以一大堆的小进程，形成了自然的、安全的和大量的自动并行。同步阻塞直接就是不允许的，这意味着不夸张地说所有的东西都是异步的：所有的文件和网络 IO，所有的消息传递，和所有的像跟其他异步工作项会合的“同步”活动项。结果造就系统超高并发，用户输入响应良好，并且非常容易扩展。但就如同你想象的那样，这当然也带来了令人着迷的挑战。

## 异步编程模型

Midori 的异步编程模型表面上很像 C# 的 async/await。

这当然不是巧合。我是 [.NET 任务库](https://en.wikipedia.org/wiki/Parallel_Extensions)的架构师和主程。作为 Midori 并发架构师，我不得不承认我对这个眼前这个 .NET Release 的成功有些偏见。然而即使是我也清楚我们之前的工作不能照样在 Midori 上奏效，所以开始着手这段经年的旅程。但在我们前进的同时，我们也与 C# 团队紧密合作，以将一些 Midori 的做法带回到这个行进中的语言。并且早在 C# 着眼之前的一年多，就已经开始使用一种 async/await 的变体。我们并没有将所有 Midori 的优势都带给 .NET，有些劣势已经显露出来了，大多数都是关于性能的方面。一想到我再也回不到过去及时将 .NET 的 Task 弄成 struct，我就想死。

但我得超越自己。要进行一个很长的旅程才能接触到现在这点，让我们从最开始的地方启程吧。

## Promises

在我们异步模型的核心是一个叫 [promises](https://en.wikipedia.org/wiki/Futures_and_promises) 的技术。现在，这个概念已经相当普遍了。但就像等会你会看到的那样，我们使用 promises 的方式更有趣些。我们受 [E 系统](https://en.wikipedia.org/wiki/E_(programming_language))的影响很深。也许跟现在流行的大多数异步框架最大的不同是我们完全不玩虚的，在我们系统里面连一个同步的 API 都没有。

异步模型的第一版用的是显式的回调（callbacks）。这东西用过 Node.js 的都知道。主要想法就是你得到的任何一个操作的 Promise&lt;T>，最终会给你一个结果 T （或出错了）。这个 T 可能由内部的异步处理甚至在远程的某个地方跑出来的，不过使用它的就不需要关心了。它们只是把 Promise&lt;T> 当作普通的类型处理，去索取的时候，必须将 T 交出来。

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

[几乎](https://msdn.microsoft.com/en-us/library/hh156528.aspx)[每一个](http://tc39.github.io/ecmascript-asyncawait/)[重要的](https://www.python.org/dev/peps/pep-0492/)[语言](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2014/n4134.pdf)现在都具备类似 async 和/或 await 特性。而我们是从 2009 年开始大范围使用的。我说的大范围，那是真的大范围。

async/await 让我们保持系统的非阻塞本性并且消除了上述的一些可用性混乱。事后看来，这是很显然的，但请记住当时使用 await 上规模的最主流的的语言也不过是 F#，用它的[异步工作流](http://blogs.msdn.com/b/dsyme/archive/2007/10/11/introducing-f-asynchronous-workflows.aspx)（也看看[这篇论文](http://research.microsoft.com/apps/pubs/default.aspx?id=147194)）。尽管在可用性和生产力上很有好处，但在队伍里也有很大的争议，后来更多。

我们搞的这套跟 C# 和 .NET 里的有点不同。让我们跟随从上面说的 promises 到新的基于 async/await 模型的脚步。我们一路走，一路我会指出它们的不同之处。

我们首先将 Promise&lt;T> 重命名为 AsyncResult&lt;T>，将它设计成 struct。（这跟 .NET 的 Task&lt;T> 很相似，但比起“计算”，更关注“数据”。）这样一个相关类型的家族就诞生了：

* T： 不会失败的同步计算即刻返回的结果。
* Async&lt;T>： 不会失败的异步计算的结果。
* Result&lt;T>： 可能会失败的同步计算即刻返回的结果。
* AsyncResult&lt;T>： 可能会失败的异步计算的结果。

最后一个其实只是 Async&lt;Result&lt;T>> 的简写。

会失败的和不会失败的区别是以后会提到的另一个课题。总而言之，不管怎样，我们的类型为我们保证了这些属性。

顺着下去，我们加入了 await 和 async 关键字。一个方法能被标记为 async：

```csharp
async int Foo() { ... }
```

这意味着在它里面允许 await:

```csharp
async int Bar() {
    int x = await Foo();
    ...
    return x * x;
}
```

开始这只不过是解决回调麻烦的语法糖，像 C# 里一样。但在最终，我们更进一步，为了性能，加入了轻量级的协程（coroutine）和链接栈（linked stack）。下面会提及更多。

要调用 async 方法的调用者必须做出选择：是用 await 并等待方法的结果呢？还是用 async 并执行异步的操作？所有的异步性在系统因此显式为：

```csharp
int x = await Bar();        // Invoke Bar, but wait for its result.
Async<int> y = async Bar(); // Invoke Bar asynchronously; I'll wait later.
int z = await y;            // ...like now.  This waits for Bar to finish.
```

这也给我们带来了一个非常重要但微妙的特性，这是我们很后面才认识到的。因为在 Midori 里面“等待”某样东西的唯一的办法是使用异步模型，而且没有隐藏的阻塞。我们的类型系统能告诉我们所有的能“等待”的东西。更重要的是，它能告诉我们所有不能等待的东西，这就是纯粹的同步计算！这就能用来保证没有代码能够阻塞 UI 绘画，以及其他的，我们下面能看到，非常多很强的能力。

因为系统中异步代码的绝对量太大，我们在语言里修饰了很多 C# 还没有支持的模式。例如迭代器，for 循环和 LINQ 查询：

```csharp
IAsyncEnumerable<Movie> GetMovies(string url) {
    foreach (await var movie in http.Get(url)) {
        yield return movie;
    }
}
```

或者，用 LINQ 的风格：

```csharp
IAsyncEnumerable<Movie> GetMovies(string url) {
    return
        from await movie in http.Get(url)
        ... filters ...
        select ... movie ...;
}
```

整个 LINQ 体系是流式处理的，包括资源管理和[背压](https://en.wikipedia.org/wiki/Back_pressure)。

我们将数百万行的代码从旧的 callback style 转换成新的 async/await 方式。这个过程中发现了大量 bug，由原先显式回调模型的复杂的控制流造成的。特别是在循环和错误处理逻辑中，相比原来的愚蠢的 API 版本，现在能用熟悉的编程语言结构了。

我之前提到这是有争议的。队伍里大多数人喜欢这可用性的改善，但并不是全部都如此。

可能最大的问题在于它鼓励了一种拉模式（pull-style）的并发。拉，指的是调用者在处理它自己操作之前需要等待被调用者。在新的模型里，你需要花点力气才能不这样做。当然，由于 async 关键字的存在，不这样做是一直允许的，但就是比旧的模型多了那么一点摩擦。只需将  await 关键字拿掉，就变成了旧的、熟悉的、阻塞的等待模型。

我们在拉和推之间提供了桥梁，用一种[响应式](https://rx.codeplex.com/)的 IObservable&lt;T>/IObserver&lt;T> 适配器的形式。我不会宣称他们非常成功，然而对于不采用数据流的有边际影响的操作，他们是很有用的。实际上，我们的整个 UI 框架是基于[函数式反应型编程](https://en.wikipedia.org/wiki/Functional_reactive_programming)这个概念的，这需要跟 Reactive Framework 有点不同，为了性能。我们就不继续展开来说了。

一个有趣的影响是带来了一种新的不同，在先 await 再返回一个 T 的方法，和直接返回一个 Async&lt;T> 的方法之间。以前类型系统并不存在这种区别。坦白地说，这让我们很郁闷，现在也还是。例如：

```csharp
async int Bar()  { return await Foo(); }
Async<int> Bar() { return async Foo(); }
```

我们很想说这两种方式性能上是一样的，但很可惜，并不是。前一种拦截并保持一个栈帧（stack frame）活动，后一种不会。有些编译器的小机灵能帮助处理常见的模式 —— 道义上跟异步尾调用是等价的 —— 但并不什么时候都那么凑效。

在它本身，并不是一个大问题。虽然它在一些重要的地方例如流中导致了一些反模式。开发人员倾向于在以前直接返回 Async&lt;T> 的地方 await，导致一堆没必要的暂停的栈帧堆积。我们对大部分模式有很好的解决方案，但一直到项目的结束我们仍然为它挣扎，特别在需要跑满 10GB 网卡线速的网络栈。下面我们会讨论我们应用的一些技术。

但在旅程的结束往回看，这个改变还是值得的，在于模型的简单性和可用性，也在于它给我们打开了一些优化的大门。

## 执行模型 

这就引出了执行模型。我们经历了也许五个不同的模型，但最后着陆在一个不错的地方。

完成异步一切的一个关键是超轻量级进程。这是可能做到的，多亏在[上一篇提到的安全基础](https://github.com/ZiJing6/blogging-about-midori/blob/master/a_tale_of_three_safeties.md)上构建的[软件隔离进程（SIPs）](http://research.microsoft.com/apps/pubs/default.aspx?id=71996)。

去除共享的、静态 (static) 的可变状态帮助我们保持进程纤小。在传统程序中，拥有一大堆可变静态变量，能烧掉多少地址空间是让人震惊的。我之前提到，我们冻结了绝大部分静态变量成常量，将它们在很多进程之间共享。执行模型还导致了更低成本的栈（详见下文），也是一个关键因素。最终的这些受益于不仅仅是技术，还有文化。每晚在实验室，我们测量进程的启动时间和占用内存，并有一个“棘轮”过程保证我们每个 sprint 都比上一个好。我们中的一个组每周找个地方，仔细看这些数字并尝试回答为什么升了、降了或者为什么还是一样。我们普遍地有这种性能文化，但正因如此，它保持了系统轻巧快速的基准。

进程中运行的代码不能阻塞。在内核里，阻塞在特定的地方是允许的，但要记得没有用户代码能够在内核里跑，所以这只是一个实现细节。当我说“非阻塞”，我是说真的：Midori 没有[虚拟内存](https://en.wikipedia.org/wiki/Demand_paging)，这在传统的操作系统中，意味着接触一块内存有可能会物理阻塞以进行 IO 操作。我得说，页面颠簸的缺少是非常受欢迎的，一直以来，我对新的 Windows 系统做的第一件事就是关掉虚拟内存。我宁愿操作系统因为内存不足杀掉程序，然后稳定地运行下去，也不愿意它像疯了一样处理虚拟内存。

C# 的 async/await 实现完全是一个前端编译器的把戏。如果你反编译过生成的程序集（assembly），你就知道：它提升捕获到的变量到一个对象的的字段里，将方法体重写为一个状态机，然后用 Task 的继续传递机制来保持类似迭代器的对象在不同的状态中推进。

我们也是这样开始的，还向 C# 和 .NET 组分享了一些我们关键的优化。不幸的是，在 Midori 这种规模上，结果就是根本没法跑。

首先，要记得，Midori 是一整个操作系统都是用垃圾回收内存写的。我们学到了一些重要的经验教训，有必要充分这样做。但我得说主要的指示是避免像灾难一样的过多的分配，包括那些短命的对象。早期在 .NET 有一种这样的称赞：Gen0 回收是免费的。不幸的是，它塑造了很多的 .NET 类库代码，但完全是胡说。Gen0 回收带来暂停、弄脏缓存、还给高并发的系统带来[频率节拍问题（beat frequency issues）](https://en.wikipedia.org/wiki/Beat_(acoustics))。然而我会指出，在 Midori 这样的规模上运行垃圾回收的一个窍门恰恰是细粒度的进程模型，每一个进程都有单独的可以独立回收的堆。我以后会写一整篇文章来说我们是怎样让我们的垃圾回收器获得好的表现的，但这是最重要的架构特性。

因此，第一个关键的优化，如果一个 async 方法不进行 await ，就不应该为它做任何的分配。

我们给 .NET 分享了我们的经验，在实现 C# 的 await 时。可惜的是，那时候， .NET 的 Task 已经实现成类（ class ）了。因为 .NET 需要 async 方法返回 Task 类型，所以做不到零分配，除非你自己山寨一种模式，如缓存一个 singleton 的 Task 对象。

第二个关键的优化是确保为进行了 await 的 async 的方法只做尽可能少的分配。

在 Midori 中，很常见一个 async 方法调用另外一个，这个又调用另外一个，一直下去。你设想一下在状态机模型下会发生什么，那块的一个叶子方法会触发了一连串的 O(K) 内存分配，K 是 await 时栈的深度。这是非常不划算的。

我们最终达成的是一个只会在 await 发生时才会分配的模型，而且在整个调用链中只会分配一次。我们管这调用链叫“活动”。最上层的 async 区分了活动的分界。结果， async 可能还是有点花销，但 await 回收是免费的。

是的，这需要一个额外的步骤，而且这一步很重要。

最后的关键优化是确保 async 方法遭受尽可能少的损失。这意味着要消除状态机重写模型的一些不够好的方面。实际上，我们最终放弃了状态机重写：

1. 它完全摧毁了代码质量。它阻碍了内联（inlining）这样的简单优化，因为很少内联器会认为一个有多个状态变量的切换语句，加上堆分配的显示帧，有大量的局部变量赋值，会是一个“简单的方法”。我们在和用本地代码开发的操作系统竞争，所以这很重要。

2. 它要求改变调用约定。就是说，返回值必须是 Async*&lt;T> 对象，跟 .NET 的 Task&lt;T> 类似。这是非首选的。虽然我们的是 struct —— 消除了分配的方面 —— 它们仍然是多余的，并且要求代码进行状态和类型测试来获取值。如果我的异步方法返回一个 int，我希望生成的机器码是一个该死的返回 int 的方法！

3. 最后，经常有太多的堆状态需要捕获了。我们希望一个等待的活动使用的总空间尽可能的小。很经常有些进程有上百上千个这样的活动，加上有些进程不断在它们间切换。由于内存占用和缓存理由，让它们尽可能地保持跟非常小心的手写的状态机一样小是非常重要的。

我们创建的模型是一个异步活动跑在[链接栈（linked stack）](https://gcc.gnu.org/wiki/SplitStacks)上的模型。这些链接开始只有 128 字节，并按需要增长。多次试验之后，我们形成了个链接每次大小翻倍的模型。即一开始 128 byte，然后 256 byte，最多到 8k 的分块。实现这个需要很深入的编译器支持。编译器知道提升链接检查，特别是在循环中，并在它能预测栈帧的大小的时候探测更大的数量（为了内联）。用链接存在一个常见问题是会频繁结束并重新链接，特别是在循环里进行函数调用的边界，然而上述的绝大部分优化避免了这种情况发生。而且即使发生了，我们的链接代码是手写的汇编 —— 如果我没记错的话，链接是用三个指令 —— 而且我们保持一个可重用的热连接段的后备。  【这段翻译得超烂，我也不是很懂。】

还有另一个关键的创新。还记得吧，我之前提过的，我们从类型系统中知道一个方法是异步与否，就简单地看是不是有 async 关键字。这让我们在编译器中可以将所有非异步的代码执行在经典的栈上。结果就是所有的同步代码仍然是无需探测的！另一个效果是操作系统内核能在一堆池化的栈上调度所有的同步代码。它们总是热的，更类似一个经典的线程池，而不是普通的 OS 调度器。因为这些代码从不阻塞，你不需要 O(T) 个栈，这里 T 是整个系统中活动线程的数量。作为替代，你最后只需要 O(P) 个栈，这里 P 是机器的处理器数目。记住，去掉了虚拟内存也是实现这个成果的关键。所以这是真的一堆大赌注加起来最终剧烈的改变了游戏的规则。

## 消息传递

系统中一个基础的部分在之前的讲述中被无视了：消息传递。

进程不仅是超轻量级的，他们还是单线程的。每个上面跑一个[事件循环](https://en.wikipedia.org/wiki/Event-driven_programming)而且事件循环不能被阻塞。由于系统的非阻塞特性，它的工作是执行一块非阻塞的工作指导它完成或者进行等待，然后获取下一块工作，如此下去。之前等待并满足了的等待被调度为触发下一轮运作的曲柄。

每一个这样的曲柄轮转被贴切地称为：一个“轮换”。

这意味着轮换会在异步活动之间发生，不在其他地方，只在等待点上。因此，并发只会在定义好的点发生交错。这是推测并发状态的一个巨大的福利，然而也随之有些坑，我们后面会探索这些。

最好的部分就是，进程再也不会遭受共享内存争用了。
