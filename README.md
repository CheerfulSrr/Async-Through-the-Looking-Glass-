# Async Through the Looking Glass翻译

> 原文地址: <https://medium.com/@nhumrich/async-through-the-looking-glass-d69a0a88b661>
>
> 原作者: [Nick Humrich](https://medium.com/@nhumrich?source=post_page-----d69a0a88b661----------------------)
>
> 创作时间: [Nov 19, 2016](https://medium.com/@nhumrich/async-through-the-looking-glass-d69a0a88b661?source=post_page-----d69a0a88b661----------------------)
>

### 翻译

* 张成琪

### note

* 阅读时间: 15分钟

![avatar](/img/python.png)

​		A lot of talk has been happening in the python community about *asyncio.* Asynchronous programming is the new hot thing in python. In [my last post](https://hackernoon.com/asynchronous-python-45df84b82434) I talked about how to do asynchronous programming and what it is, but in this post we will cover whether or not it’s worth even bothering with this whole async thing. Let’s talk about some common use cases in python, and decide whether asyncio will serve you well.

​		很多谈话都发生生在关于异步的python社区，异步编程是新的热点在巨蟒中兴高采烈。在我上一篇文章中我谈到关于如何进行异步编程和这是什么，但在这篇文章中我们不管它是否值得整个异步的事情。我们来谈谈python中的常见用例，并决定asyncio是否会为您服务。



*Disclaimer: every point in this article assumes Python as the programming language. If you are using*
*another language, some of these points might not be completely accurate.*

*免责声明：本文的要点，假设python作为编程语言。如果你在用另一种语言，其中一些可能不是完全准确。*



## **The Pool of Tears**

​		The first thing you need to understand is that python has a [Global Interpreter Lock](https://wiki.python.org/moin/GlobalInterpreterLock). This basically means that the python interpreter uses a lock to make sure only one thread is executing code at any given time. In other words, python multi-threaded applications still only use one CPU core, and are not any faster than standard single-threaded applications. In fact, in cpu-bound cases, a multi-threaded application can take longer than a single threaded one because of thread-contention. (Proof at bottom of the post).

​		首先需要了解的是python有一个全局解释器锁。这基本上意味着python解释器使用一个锁来确保在任何给定时间只有一个线程在执行代码。换句话说，python多线程应用程序仍然只使用一个CPU内核，并不比标准的单线程。应用程序快多少。实际上，在CPU受限的情况下，由于线程争用，多线程应用程序可能比单线程应用程序花费更长的时间。（证据在文章的底部）

​		This means that a single asynchronous thread will inherently have the same performance as a multi-threaded app, except for the fact that the async version won’t have any thread contention.

​		这意味着单个异步线程将具有与多线程应用程序相同的性能。除了异步版本不会有任和线程争用之外。

​		In order to do [true parallelism](https://blog.golang.org/concurrency-is-not-parallelism) in python you need to use multiple processes. Each process in python has its own GIL and they do not share resources at all. If you want to maximize your CPU cores, typically the rule is to have the number of processes equal the number of cores you have plus one. This rule applies to both  hreads and async programming.

​		为了在python中实现真正的并行，您需要使用多个进程。Python中的每个进程都有自己的GIL，它们根本不共享资源。如果您想最大化您的CPU核心，通常的规则是进程的数量等于核的数量加1。此规则同时适用于线程和异步编程.



## 	**The Mock Turtle’s Story**	

​		There is [no silver bullet](http://worrydream.com/refs/Brooks-NoSilverBullet.pdf) in programming, and asynchronous programming is no exception. There are certain cases where asynchronous programming might benefit you, and several others where it won’t. There are even some areas where async could hurt you. In order to know if async will provide benefits, we have to consider the use cases. The most common use case would be a web-application backend (web server). This is one of the most common types of programming I come across in python, so I will try to cover it in depth. If you don’t deal with web programming, feel free to skip to the **Queens Croquet Ground** section.

​		在编程中没有什么灵丹妙药，异步编程也不例外。在某些情况下，异步编程可能会使您受益，而在其它一些情况下，异步编程则不会。甚至在某些领域可能会伤害您。为了知道异步是否会带来好处，我们必须考虑用例。最常见的用例是web应用程序后端（web服务器 ）。这是我在python中遇到的最常见的编程类型之一，所以我将尝试深入讨论它。如果您不处理网络编程，请随意跳到Queens Croquet Ground部分。

​		Remember the days before online banking, when we would actually go to a bank, stand in line, and wait for them to accept our money? Since you might be too young to remember such dark days, here is a brief sum up of how banks work. It’s a cold day, and you’re in a hurry to cash your check. You walk into the bank and see a line of people standing in a single queue. You notice that there are several tellers helping customers. You wait in line, getting antsy because you left your car outside with the engine on. You get anxious when you see a slow, troublesome customer over at teller #4. Hoping that everyone in front of you only needs to simply cash a check, you stay in line. You know that teller #4 won’t be freed up anytime soon, so you start cheering in your head for teller #2 to work quickly. When you finally get up to the teller, you hand them your check, and get the cash equivalent. How long you waited in line was dependent on how long it took the bankers to handle the people in front of you.

​		还记得网上银行出现之前的日子吗？那时我们会去银行排队，等着他们来取钱。由于你可能还太年轻，不记得如此黑暗的日子，这里有一个简短的总结银行是如何运作的。今天很冷，你急着去兑现支票。你走进银行，看到一排人排着队。你注意到有几个出纳员在帮助顾客。你排着队等着，因为你把车停在外面，引擎还开着。当你在4号柜台看到一个慢吞吞、麻烦不断的顾客时，你会感到焦虑。希望你前面的每个人只需要兑现一张支票，你就可以排队了。你知道4号出纳员不会很快被释放，所以你开始为2号出纳员快速工作而欢呼。当你终于走到出纳员面前时，你举手示意。他们给你支票，并得到等值的现金。你排了多长时间的队取决于你前面的人需要多长时间才能被银行家搞定。

​		The fast-food industry, however, has learned the art of maximizing throughput. After all, a synonym for
speed is in the name. When you go to a fast-food place, there is usually only one person taking orders. The person takes the order, you complete your transaction, then you wait for your food to arrive. The difference here is that while you are waiting for your food, the same teller that took your order is now taking the order of everyone else in line. Because you only wait to order, you go through the line a lot quicker.

​		然而，快餐业已经学会了最大化产量的艺术。毕竟，速度的同义词就在名字里。当你去快餐店时，通常只有一个人在点餐。服务员点了菜，你完成了交易，然后你就等着你的食物上来。这里的不同之处在于，当你在等待食物的时候，为你点单的那个出纳员现在正在为排队的其他人点单。因为你只需要等着点餐，所以你通过快了很多。



## Lobster Quadrille

​		The bank example is a synchronous web server. A typical python web server works by having thousands of threads (the tellers) ready to handle requests. The teller completes the entire transaction himself. If the eller has to wait on a manager or someone to come over, he will wait with you rather than moving to nother customer. Just like the bank, your webserver cannot handle more requests (customers) at a time than threads (tellers) you have in your pool (bank). However, because python can only have one thread running at a time, imagine that it’s more like the bank has only one cash box, and it can only  accommodate one teller at a time. No two tellers can actually be counting cash (their main task) in parallel.

​		银行示例是一个同步web服务器。典型的python web服务器的工作方式是准备好处理请求的数千个线程（出纳员）。出纳员自己完成整个交易。如果出纳员不得不等经理或其他人过来，他会和你一起等，而不是去找其他客户。与银行一样，您的web服务器一次处理的请求（客户）不能超过池（银行）中的线程（出纳员）。但是，因为python一次只能运行一个线程，所以可以想象一下，它更像是银行只有一个现金箱，一次只能容纳一个出纳员。没有两个出纳员可以同时数钱（他们的主要任务）。

​		The fast-food example is the asynchronous example. In this example there is only one clerk who is accepting incoming requests as fast as possible. Since you are limited to a single cash-register, it makes sense to only have a single person (thread) handling requests. This way you don’t waste time “waiting for the cash register” (cpu scheduling). The fast-food place instead has waiting points; waiting for another person to complete their part of the order. When a wait happens, the employee can help another customer while waiting. For example imagine the person assembling your food as the database. While you are waiting for your food, the clerk is accepting new orders. But once your food is ready, the clerk puts it all
together and hands it to you.

​		快餐的例子是异步的例子。在这个示例中，只有一个职员接受尽可能快的传入请求。由于您只能使用一台收银机，所以只有一个人（线程）处理请求是有意义的。这样你就不会浪费时间“等待收银机”（CPU调度）。相反，快餐店有等候点；等待另一个人完成他们的部分订单。当等待发生时，员工可以在等待时帮助其他客户。例如，假设某人将您的食物组合为数据库。当你在等食物时，店员正在接受新订单。但是一旦你的食物准备好了，店员就会把一切都准备好把它放在一起，交给你。

​		The astute reader might realize that the overall time waiting “to be finished” is still the same in
both scenarios, the only difference is that in the fast-food example, you are waiting twice instead of all at once. This is true in the naive example, but your average wait times decrease as your tasks become denormalized. Imagine every person ordered the same meal at the restaurant. If this was the case, the
bank style and the fast-food style of getting your food would yield the same results. The reason why fast-food chains prefer an asynchronous flow is because it yields better results with varying sizes of orders. Imagine it’s dinner time, and everyone is ordering with their families, but all you want to do is order a shake. In a synchronous example, you have to wait for a family’s meal to be ready before you can even place your order. But in the asynchronous example, you get to place your order right away, and because it doesn’t require the food assemblers (database), you can get your shake right away, rather than waiting on others. This increases your overall throughput and lowers average response times. It also means that large tasks don’t make other non-large tasks more latent.

​		精明的读者可能会意识到，在这两种情况下，等待"待完成"的总时间仍然相同，唯一的区别是，在快餐示例中，您等待两次，而不是一次等待所有时间。在天真的例子中，这是事实，但随着任务非规范化，平均等待时间会减少。想象一下，每个人在餐厅都订了同样的饭。如果是这样的话，银行风格和快餐式得到你的食物会产生相同的结果。快餐连锁店之所以喜欢异步流程，是因为它能产生更好的结果，订单大小也不同。想象一下，现在是晚餐时间，每个人都在和家人一起点菜，但你想做的只是点一个奶昔。在同步示例中，您必须等待一个家庭的膳食准备好，然后才能下订单。但在异步示例中，您可以马上下订单，并且因为它不需要食品组装器（数据库），因此您可以马上获得您的摇动，而不是等待其他人。这会增加您的总吞吐量并降低平均响应时间。这也意味着大型任务不会使其他非大型任务更加潜在.

​	One thing I didn’t discuss ist he cost of context switching with threads. I won’t waste too much time
discussing context switching, as [I have already done so](https://hackernoon.com/asynchronous-python-45df84b82434).Needless to say, asynchronous web servers are theoretically faster than threaded web servers even in the naive case, because you have no wasted context
switches. A very simple asynchronous web server **will** be faster than a simple threaded version. However, this does not mean **yours** will be faster simply by moving to async. It depends greatly on what your web server is doing and to whether you are actually using asynchronous paradigms.

​		有一件事我没有讨论，那就是使用线程切换上下文的成本。我不会浪费太多的时间讨论上下文切换，因为我已经这样做了。不用说，异步 Web 服务器理论上比线程 Web 服务器快，即使在天真的情况下也是如此，因为您没有浪费的上下文切换。非常简单的异步 Web 服务器将比简单的线程版本更快。但是，这并不意味着只需移动到异步，您的速度就会更快。这在很大程度上取决于 Web 服务器正在执行的操作以及您是否实际使用异步范例。



## **Who Stole the Tarts?**

​		The main contingency with whether or not async will work for you, is if you have a bottleneck of some sort. I am going to make a huge generalization and say that almost every web server has a bottleneck and simply switching to an asynchronous framework will not buy much, if anything. And as we learn from [The Goal](https://amzn.com/0884271951) *(*at least that’s where I think the quote originally comes from):Any improvements made anywhere besides the bottleneck are an illusion.

​		异步是否适用于您的主要偶然性是，您是否存在某种瓶颈。我将做一个巨大的概括，并说，几乎每一个Web服务器都有一个瓶颈，简单地切换到一个异步框架不会买很多，如果有的话。正如我们从《目标》中学到的（至少我认为这句话最初来自）：除了瓶颈之外，在任何地方所做的任何改进都是一种错觉.

​		In other words, if you have a large bottleneck, you will only see nano improvements by tweaking any other part of the system. Most servers have a hard dependency on a database. That database could be a bottleneck or [perhaps the code that abstracts the database is the bottleneck](http://techspot.zzzeek.org/2015/02/15/asynchronous-python-and-databases/). Async can still provide
benefits despite a bottleneck if 1) the bottleneck is a remote service (i.e.database, downstream service, redis, etc.), and 2) you can do other things for that same request while waiting on a database/service.

​		换句话说，如果您有一个很大的瓶颈，你只会看到纳米改进通过调整系统的任何其他部分。大多数服务器都依赖于数据库。该数据库可能是一个瓶颈，或者抽象数据库的代码可能是瓶颈。Async 仍然可以提供好处，尽管存在瓶颈，但如果 1） 瓶颈是远程服务（即数据库、下游服务、redis 等），2） 在等待数据库/服务时，您可以为同一请求执行其他操作。

1. Async programming switches context on I/O. That means that your process will be doing other things while its waiting on your database. If we assume you are using a connection pool for your database, this means that when your connection pool is 100% in use, your server is sitting there waiting because the threads are blocked on the connection. In async, your server will keep working even if your connection pool is completely utilized.

   异步编程在 I/O 上切换上下文。这意味着，您的进程将在数据库等待时执行其他操作。如果我们假定您正在为数据库使用连接池，这意味着当连接池使用 1000/0 时，您的服务器将坐在那里等待，因为连接上的线程被阻止。在异步中，即使连接池被完全利用，您的服务器也会继续工作
   
2. Number 1 can have varying results, but what will really help is if you can do work for a specific request
while waiting on the database for the same request. This is the async paradigm, and it’s where improvements really come from. Let’s say you are caching your large database queries with a redis cluster, and you also need to call a 3rd party service. Typically you would have three synchronous operations; calling the service, then redis to see if the data is cached, then finally call the database. Returning after all three are done. This means your response time can not be any faster then the response times of all three (*a + b + c*). In other words, you have three bottlenecks. With asyncio you can call all three at the same time, then process the results. For example, you would start the call on all three, but only wait for the service and redis. If the response from redis comes back and you don’t have the query cached, then you continue to wait for the database. Your response time is now just *max(a, b, c)* because you handled *a* and *b* while waiting on *c.* You’ve made improvements by making your main bottleneck the only bottleneck. You’ve completely removed 2 other bottlenecks!
   
   号码我可以有不同的结果，但真正有帮助的是，如果你能做一个特定的请求，而等待相同的请求的数据库工作。这是异步模式，也是改进的真正所在。假设您正在用 redis 群集缓存大型数据库查询，并且还需要调用第三方服务。通常，您将有三个同步操作;调用服务，然后 redis 查看数据是否缓存，最后调用数据库。完成所有三个工作后返回。这意味着您的响应时间不能比所有三个 （a + b + c） 的响应时间更快。换句话说，您有三个瓶颈。使用异syncio，您可以同时调用所有三个，然后处理结果。例如，您将对所有三个调用启动呼叫，但仅等待服务和 redis。如果来自 redis 的响应返回，并且没有缓存查询，则继续等待数据库。您的响应时间现在只是最大值（a、b、c），因为您在等待 c 时处理了 a 和 b。通过使主要瓶颈成为唯一瓶颈，您取得了改进。您完全消除了其他 2 个瓶颈！
   
   

​	   The next contingency is whether your web server is CPU bound. If your web server has only a couple processes that do CPU-bound tasks, async will slow you down. However, if you have a standard web server (lots of threads) and your app is mostly doing calculations (not much I/O), then threads will actually slow you down more than async. Async doesn’t really lend itself to CPU bound programs well, and it would be complicated to write, but if done correctly, would be faster than many threads due to lack of thread contention (see benchmarks at bottom of post).

​		下一个意外情况是 Web 服务器是否受 CPU 限制。如果您的 Web 服务器只有几个进程执行 CPU 绑定的任务，则异步会减慢您的速度。但是，如果您有一个标准的 Web 服务器（大量线程），并且你的应用主要在进行计算（I/O 不多），那么线程实际上会比异步更慢。Async 并不真正适合 CPU 绑定的程序，而且编写起来会很复杂，但如果操作正确，由于缺少线程争用，将比许多线程快（请参阅帖子底部的基准测试）

​		The last contingency is longr unning connections. One fairly hot technology these days is websockets.
Websockets allows a server to maintain a long-running connection to a client, and send them information without the need to poll the server. Websockets are also often used to improve performance since the client can use a single, already established connection. The problem with websockets, however, is that they must remain open. In a thread based server this means that every client is holding a thread open. Every thread held open is a potential hit to your throughput, since throughput is directly correlated to number of available threads. Async, however, can use a single process to handle all these open connections, which means the number of long running opened connections it can handle is essentially unlimited (OS socket limits are typically in the millions). If you are doing anything with long running open connections, async **will**
benefit you regardless of performance, because you won’t need to worry about resource exhaustion. This also means that async servers are far less susceptible to slow-client DoS attacks.

​		最后一个意外事件是长时间运行的连接。如今，一个相当热门的技术是Websocket 。Websocket 允许服务器保持与客户端的长期运行连接，并发送信息而无需轮询服务器。Websocket 还经常用于提高性能，因为客户端可以使用单个已建立的连接。然而，网络套接字的问题在于它们必须保持打开状态。在基于线程的服务器中，这意味着每个客户端都保持线程打开。每个保持打开的线程都会对吞吐量造成潜在冲击，因为吞吐量与可用线程数直接相关。但是，Async 可以使用单个进程来处理所有这些打开的连接，这意味着它可以处理的长时间运行的打开连接的数量基本上是无限的（OS  socket限制通常以百万计）。如果您正在执行任何具有长时间运行的打开连接操作，则无论性能如何，异步都将使您受益，因为您无需担心资源耗尽。这也意味着异步服务器远不会受到慢速客户端 DOS 攻击的影响。



## **The Queen’s Croquet Ground**

​		For those writing non web server applications in python, it’s a little easier to reason about whether or
not async will help you. It all comes down to some simple questions.

​		对于那些用python编写非网络服务器应用程序的人来说，对于异步是否对你有帮助，要推理起来要容易一些。这一切都归结为一些简单的问题

1. **Is your program CPU-Bound?** Are you primarily doing calculations such as data science and machine learning data fitting? Async only switches contexts at defined points, which is typically I/O type things, so async will buy you nothing. That being said, you are even worse off using threads and thread contention on CPU bound things is fairly problematic. I would shy away from threads and async in this case and only use a process pool.

   您的程序是否为 CPU 绑定？您是否主要进行计算，如数据科学和机器学习数据拟合？异步只在定义的oints处切换上下文，这通常是 I/O 类型，因此异步不会为您带来任何好处。也就是说，在 CPU 绑定的事情上使用线程和线程争用的情况更糟，这相当成问题。在这种情况下，我会回避线程和异步，只使用进程池。
   
2. **Does your program require doing things concurrently?** A typical use case for doing things concurrently is having a GUI and a controller portion of the app. A simple way is for each to be on its own thread. If you only have a couple threads and you don’t spawn any, then threading isn’t too bad here, async won’t really buy you anything unless you just don’t like dealing with threads. Async could, however, help you prevent common threading issues such as deadlocks. But, if you very often spin up new, short lived, threads — or your thread pool is very large — async will most likely give you speed and resource benefits

   你的程序需要同时做一些事情吗 ?并发操作的一个典型用例是具有应用的 GUI 和控制器部分。一个简单的方法是让每个线程都位于自己的线程上。如果你只有几个线程，你不生成任何，那么线程在这里不是太糟糕，异步不会真的为你买任何东西，除非你只是不喜欢处理线程。但是，Async 可以帮助您防止常见的线程问题，如死锁。但是，如果您经常启动新的、短生存的线程（或者您的线程池非常大），则异步很可能会为您提供速度和资源优势

3. **Does your program work in a single process?** A lot of scripts and CLI’s are intended for scripting simple user actions, and any “waiting” they do is to be expected. These programs typically run on a single process, and never spin up any threads. Don’t use async for these. Not only is async slower than a single thread, but it’s also not as easy to reason about
   
   您的程序在单个进程中工作吗？许多脚本和 CLI 都用于编写简单用户操作的脚本，它们所做的任何"等待"都是可以预期的。这些程序通常在单个进程上运行，从不启动任何线程。不要对这些使用异步。异步不仅比单个线程慢，而且也不容易推理。
   
4.  **Are you working with anyone fairly new to programming/python (including yourself)?** I am just going to be honest —  async code is much harder to reason about than synchronous code. [Python’s new *async* and *await*](https://www.python.org/dev/peps/pep-0492/) makes it a lot easier to reason about, but it’s still a lot more confusing than standard code. If you have anyone newer to programming, or you dont want to spend time learning how async paradigms work, you should probably shy away from asynchronous programming for now.
   
   您是否与对编程/python（包括您自己）相当陌生的人一起工作？我只想说实话-异步代码比同步代码更难解释。使它更容易推理，但它仍然比标准代码混乱得多。如果你有新的编程人员，或者你不想花时间学习异步模式是如何工作的，那么你现在应该回避异步编程。 

5. **Are you doing a lot of network calls?** If you do a lot of network calls (maybe uploading/downloading files for example) and you think you could make your program faster by doing some of them at the same time (or using *AWS S3* multipart upload for example), than asyncio can benefit you. You will probably have the same performance as doing things  threaded, but you won’t have to deal with those pesky threads and the race conditions they bring with them.

   你经常打网络电话吗？如果您进行了大量的网络调用（例如上载/下载文件），并且您认为通过同时执行其中一些调用（例如使用AWS S3 multipart upload），可以使您的程序更快，那么异步将为您带来好处。您可能会拥有与线程处理相同的性能，但您不必处理这些讨厌的线程及其带来的竞争条件。



## **Down the Rabbit Hole**

​		I have mostly been comparing async against sync as if the two can not coexist. They can, however, coexist. Making them coexist might or might not be the right choice though. If you have an application that is currently synchronous, or even threaded, and you want to do one or two things asynchronously, you can. All you need to do is start an event loop for that one given thing. However, if you are also using threads you
need to be careful as not everything asyncio does is thread safe.

​		我主要比较了异步和同步，好像两者不能共存。然而，它们可以共存。不过，让它们共存可能是正确的选择，也可能不是正确的选择。如果您的应用程序当前是同步的，甚至是线程化的，并且您希望异步执行一个或两个操作，则可以。你所需要做的就是为一件给定的事情启动一个事件循环。但是，如果同时使用线程，则需要小心，因为异步所做的一切并不是线程安全的。

​		If you have an asynchronous module, you need to be careful. The basic rule of thumb is, once you go full async, **everything** has to go async. If you accidentally write or use one non-async function that takes too long, or does some waiting, you have just blocked your entire event loop. In the case of a web-server, that means you are not accepting any requests while your event loop is blocked. The most common mistake is using the standard *time.sleep(10)* instead of the asynchronous version *await* *asyncio.sleep(10).* The former will make your entire event loop sleep for 10 seconds. That sounds fun, right?

​		如果您有异步模块，则需要小心。基本的经验法则是，一旦你完全异步，一切都必须去异步。如果您不小心编写或使用一个非异步函数，但该函数花费过长，或者需要一些等待，则只需阻止整个事件循环。对于 Web 服务器，这意味着在事件循环被阻止时，您不接受任何请求。最常见的错误是使用标准的 time.sIeep(10) 而不是异步版本等待 asyncio.sIeep(10) 。前者将使您的整个事件循环睡眠 10秒。听起来很有趣，对吧？



## **A Mad Tea-Party**

​		Since everything an asynchronous library does needs to be asynchronous, this leads to an interesting problem. That problem is 3rd party libraries. If you decide to go asynchronous, say goodbye to all the libraries you have learned to love. Libraries such as *requests, sqlalchemy, and boto3* are all off-limits. They use blocking IO and will therefore halt your event loop. You need to instead find a similar asynchronous version of the library. For example there is *aiohttp, asyncpgsa,* and *aiobotocore* respectively. Their libraries are typically newer, and do not have as many features. This also leads to a division in the python community. Almost everything that is popular in the python community is now being re-done for async support. Rather than a single library supporting both paradigms, you end up with a separate library for each paradigm.

​		由于异步库的所有内容都需要异步，这导致了一个有趣的问题。这个问题是第三方图书馆。如果你决定采用异步方式，那么请与您所喜爱的所有库说再见。*requests*、*sqlalchemy* 和 *boto3* 等库都是禁止的。它们使用阻塞 IO，因此将停止事件循环。相反，您需要查找类似的异步版本的库。例如，分别有 *aiohttp*、*asyncpgsa* 和 *aiobotocore* 。它们的库通常较新，并且没有那么多功能。这也导致了python社区中的分裂。在 python 社区中流行的几乎所有内容现在都被重新完成，以便获得异步支持。而不是一个库支持这两种范例，你最终为每个范例有一个单独的库。



## **Advice from a Caterpillar**

​		My recommendation is that you try the asyncio library for a simple task and see what you think. It takes a while to get used to and understand how everything works. Also, if you haven’t done so yet, you should [read my previous article](https://hackernoon.com/asynchronous-python-45df84b82434) where I explain how async works in python.

​		我的建议是，对于一个简单的任务，您可以尝试使用*asyncio* 库，并了解您的想法。它需要一段时间来适应和理解一切是如何工作的

​		I really think async is fun, and can help you if you truly switch to an async paradigm, but it might not be
worth giving up certain libraries to do it. I have already experienced much frustration when I can’t find an asynchronous library doesn’t do things the way the synchronous version does, and it leaves me confused. Also, halting your event loop can be disastrous, but unfortunately all too easy to do.

​		我真的认为异步很有趣，如果你真的切换到异步模式，可以帮你，但可能不值得放弃某些库来做到这一点。当我找不到异步库不能以同步版本的方式做事时，我已经经历了很多挫折，这让我感到困惑。此外，停止事件循环可能是灾难性的，但不幸的是，太容易做到。

​		That being said, Canopy runs asyncio in production and it is working very well for us. Our python apps are getting the same throughput and latency as our Java apps. Asyncio is stable enough for production, and a good number of libraries exist. Try it out and see what you think!

​		话虽如此，Canopy 在生产中运行异步，它对我们非常有效。我们的 python 应用获得与 Java 应用相同的吞吐量和延迟。异步对于生产来说已经足够稳定，可以进行生产，并且存在大量库。试试看，看看你的想法！

​		

## **Alice’s Evidence**

​		That’s pretty much the end, but I know your sitting here thinking, “what about some benchmarks to show how blazing fast this thing is?” Pretty much any benchmark you get online is useless because it’s usually some arbitrary test that has nothing to do with your current application. The test is normally designed to show something faster in a specific way. Which either means mileage will vary, or you will have something else as a bottleneck, and the difference won’t matter. However, I know you are all going to ask for some benchmarks, so here you go:

​		差不多结束了，但我知道你坐在这里想，"用一些基准，以显示这东西有多快怎么样？”你在网上获得的任何基准测试几乎都是无用的，因为它通常是一些与当前应用程序无关的任意测试。测试通常设计为以特定方式更快地显示某些内容。要么意味着里程会有所不同，要么你会有别的东西作为瓶颈，差异并不重要。然而，我知道你们都要要求一些基准，所以你可以这么做

1. What happens if we call google.com a bunch of times in a row? In this example we call them all one
   after the other (normal) as a control group. We then call them all concurrently using asyncio and threaded paradigms and compare the total time to complete the entire task (in seconds). Code at: <https://gist.github.com/nhumrich/b1ec570d1b6226f169c4b42a2905ee88>
   
   如果我们连续多次调用google.com会怎么样？在这个例子中，我们把它们一个接一个地称为对照组。然后，我们使用异步和线程模式同时调用它们，并比较完成整个任务的总时间（以秒为单位）
   
   normal: 18.08
   asyncio: 1.734
   threaded: 2.55
   
   ```python
   from timeit import timeit
   import asyncio
   import requests
   from threading import Thread
   
   import aiohttp
   
   client = aiohttp.ClientSession()
   
   
   def call_threaded():
       threads = [Thread(target=call_google_normal) for i in range(10)]
       for t in threads:
           t.start()
       for t in threads:
           t.join()
   
   
   def call_async():
       responses = [call_google() for i in range(10)]
       loop = asyncio.get_event_loop()
       loop.run_until_complete(asyncio.wait(responses))
   
   
   async def call_google():
   
       async with client.get('http://google.com') as resp:
           return resp.status
   
   
   def call_google_normal():
       resp = requests.get('http://google.com')
       return resp.status_code
   
   
   def call_normal():
       responses = [call_google_normal() for i in range(10)]
   
   
   def myfunc(a):
       a = ' '.join(['hello', a])
   
   
   num = 5
   a = timeit(call_normal, number=num)
   b = timeit(call_async, number=num)
   c = timeit(call_threaded, number=num)
   
   print('normal:', a)
   print('async:', b)
   print('threaded:', c)
   
   client.close()
   ```
   

​		To be honest, in this example we are splitting hairs. I am sure if you ran the test enough times you would see the threaded example win over asyncio a good amount as well. This really boils down to what the application code itself is doing. I use two different libraries *requests* and *aiohttp* and this code is really just about which one of those two is faster. I think asyncio and threaded are close to tied in this example because we have a very low volume of threads

​		老实说，在本例中，我们是在吹毛求疵。我确信，如果您运行测试足够多次，您也会看到线程示例在很大程度上战胜了异步。这实际上可以归结为应用程序代码本身在做什么。我使用两个不同的库请求和aiohttp，这段代码实际上是关于这两个库中哪一个更快。我认为在这个例子中，异步和线程是紧密相连的，因为我们的线程数量非常少

2. What happens if we do something really stupid and use asyncio/threads for a CPU-bound tasks? We see exactly why you shouldn’t do that, that’s what. In this example we will calculate the Fibonacci sequence in two ways. First, the normal way (our control), where we just try to calculate it flat out. Then we will try to calculate it in a way where every step is a new thread/co-routine. This means we should be able to calculate a lot faster as we are using map/reduce right? Wrong. This isn’t parallelism, its
   concurrency. Example code: 

   如果我们做了一些非常愚蠢的事情，并使用异步/线程执行CPU绑定的任务，会发生什么？我们明白你不该那样做的原因，就是这样。在这个例子中，我们将用两种方法计算斐波那契序列。首先，正常的方式（我们的控制），我们只是试图计算出它的平面。然后我们将尝试以一种每一步都是新的线程/协同例程的方式计算它。这意味着我们在使用map/reduce时应该能够更快地计算，对吧？错了。

   Normal: 0.01143600800060085
   Asyncio: 1.130632936998154
   Threaded: 2.099454802002583

   ```python
   import asyncio
   from multiprocessing.pool import ThreadPool
from timeit import timeit
   from functools import partial

   pool = ThreadPool(processes=400)
loop = asyncio.get_event_loop()
   
   
   def fib(x):
       if x in (0, 1):
           return 1
       a = fib(x-2)
       b = fib(x-1)
       return a + b
   
   
   def fib_threaded(x):
       if x in (0, 1):
           return 1
       a, b = pool.map(fib_threaded, (x-2, x-1))
       return a + b
   
   async def fib_async(x):
       if x in (0, 1):
           return 1
       a, b = await asyncio.gather(fib_async(x-2),fib_async(x-1))
       return a + b
   
   y = 14
   
   
   def normal():
       return fib(y)
   
   
   def threaded():
       return fib_threaded(y)
   
   
   def async_():
       return loop.run_until_complete(fib_async(y))
   
   
   num=100
   a = timeit(normal, number=num)
   b = timeit(async_, number=num)
   c = timeit(threaded, number=num)
   
   print('Normal:', a)
   print('Async:', b)
   print('Threaded:', c)
   
   
   concurrently = 10
   
   async def fib_async_wrap(x):
       return fib(x)
   
   def multiple_times():
       for i in range(concurrently):
           fib(y)
   
   
   def threaded_at_once():
       threads = [y for i in range(concurrently)]
       results = pool.map(fib, threads)
   
   
   def async_at_once():
       coros = [fib_async_wrap(y) for i in range(concurrently)]
       loop.run_until_complete(asyncio.gather(*coros))
   
   
   num=100
   a = timeit(multiple_times, number=num)
   b = timeit(async_at_once, number=num)
   c = timeit(threaded_at_once, number=num)
   
   print()
   print('--concurrently--')
   print('Normal:', a)
   print('Async:', b)
   print('Threaded:', c)
   ```
   

​		As you can see, there is a lot of overhead when creating new threads and dealing with context switching. The async version is 100 times as slow, and the threaded version is 200 times as slow

​		如您所见，在创建新线程和处理上下文切换时有大量开销。异步版本是慢的100倍，线程版本是的慢速是200倍。

​		So what if we instead calculate the Fibonacci sequence a couple times, and instead of breaking up per
calculation, we just calculate it 10 times, concurrently? See below! (same code snippet)

​		那么，如果我们改为计算斐波那契序列几次，而不是按计算分解，我们只是同时计算10次呢？见下面！（相同的代码段）

​		Normal: 0.09980539800017141
 		Asyncio: 0.12089022299915086
 		hreaded: 0.12662064500182169

​		So asyncio is about 1.2 times slower than normal, and threaded is about 1.25 times slower. In this example, all threads already existed, so you are only seeing the overhead of context switching on only 10 threads. Shows the effect of context switching.

​		异步比正常情况慢 1.2 倍，线程大约慢 1.25 倍。在此示例中，所有线程都已存在，因此您只看到上下文仅在 10 个线程上切换的开销。显示上下文切换的效果。