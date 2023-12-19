---
title: 1.2 关键类型
weight: 2
---

Rx 功能强大，可以极大地简化响应事件的代码。但要写好响应式代码，您必须了解一些基本概念。 Rx 的基本构建块是一个名为 `IObservable<T>` 的接口，与其相对应的是名为 `IObserver<T>` 的接口。理解这两个接口是成功使用 Rx 的关键。

前面的章节将这个 LINQ 查询表达式作为第一个示例展示给读者：

```C#
var bigTrades =
    from trade in trades
    where trade.Volume > 1_000_000;
```

大多数 .NET 开发人员至少会熟悉 [LINQ](https://learn.microsoft.com/en-us/dotnet/csharp/linq/) 的多种流行形式之一，例如 [LINQ to Objects](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/concepts/linq/linq-to-objects) 或 [Entity Framework Core 查询](https://learn.microsoft.com/en-us/ef/core/querying/)。大多数 LINQ 实现允许您查询静态数据。LINQ to Objects 适用于数组或其他集合，Entity Framework Core 中的 LINQ 查询针对数据库中的数据，但 Rx 不同：它提供了查询实时事件流（您可能称之为动态数据）的能力。

如果您不喜欢查询表达式语法，也可以直接调用 LINQ 运算符来编写完全相同的代码：

```C#
var bigTrades = trades.Where(trade => trade.Volume > 1_000_000);
```

它们都是 LINQ 的表达方式，表示我们希望 bigTrades 仅包含 trades 中 Volume 属性大于一百万的项。

我们无法准确说明这些示例的作用，因为我们看不到 trades 或 bigTrades 变量的类型。类型不同，代码的含义也会有很大差异。如果我们使用 LINQ to objects，这两个变量的类型可能是 `IEnumerable<Trade>` 。也就是说这两个变量都引用了表示集合的对象，而我们可以使用 `foreach` 循环枚举其内容；同时它们表示静态数据，即我们的代码可以直接检查的数据。

所以我们需要通过明确类型来明确代码的含义：

```C#
IObservable<Trade> bigTrades = trades.Where(trade => trade.Volume > 1_000_000);
```

这样就消除了歧义。现在很明显，我们处理的不是静态数据。我们正在使用 `IObservable<Trade>` 。但这到底是个啥？

## IObservable<T>

[`IObservable<T>`](https://learn.microsoft.com/en-us/dotnet/api/system.iobservable-1) 接口代表了 Rx 的基本抽象：若干 T 类型的值组成的序列。从非常抽象的意义上来说，这意味着它和 `IEnumerable<T>` 代表着相同的东西。

区别在于代码如何使用这些值。 `IEnumerable<T>` 使代码能够（主动地）检索值（通常使用 `foreach` 循环），而 `IObservable<T>` 在值可用时提供值。这种区别有时被描述为“推送式（push）”与“拉取式（pull）”。我们可以通过执行 `foreach` 循环从 `IEnumerable<T>` 中提取值，但 `IObservable<T>` 会将值推送到我们的代码中。

`IObservable<T>` 如何将值推送到我们的代码中？如果我们想要这些值，我们的代码必须订阅 `IObservable<T>` ，也就是说为其提供一些可以调用的方法。事实上，“订阅（Subscribe）”是 `IObservable<T>` 直接支持的唯一操作。这是接口的完整定义：

```C#
public interface IObservable<out T>
{
    IDisposable Subscribe(IObserver<T> observer);
}
```

您可以在 GitHub 上查看 `IObservable<T>` 的[源代码](https://github.com/dotnet/runtime/blob/b4008aefaf8e3b262fbb764070ea1dd1abe7d97c/src/libraries/System.Private.CoreLib/src/System/IObservable.cs)。请注意，它是 .NET 运行时库的一部分，而不是 System.Reactive NuGet 包的一部分。 `IObservable<T>` 代表了一个极其重要的抽象，因此它被融入到 .NET 中。 （您可能想知道 System.Reactive NuGet 包的用途。.NET 运行时库只定义了 `IObservable<T>` 和 `IObserver<T>` 接口，而不包括 LINQ 实现。 System.Reactive NuGet 包为我们提供了 LINQ 支持，同时还能处理多线程。）

这个接口唯一的方法清楚地表明了我们可以用 `IObservable<T>` 做什么：如果我们想接收它提供的事件，我们需要订阅它。我们也可以取消订阅： `Subscribe` 方法返回 `IDisposable` ，如果我们调用 `Dispose` ，它就会取消我们的订阅。`Subscribe` 方法要求我们传入 `IObserver<T>` 的实现，这点我们待会儿再说。

细心的读者会发现前一章中的示例似乎不符合 `Subscribe` 方法的要求。该代码创建了一个每秒生成一次事件的 `IObservable<long>` ，然后使用以下代码订阅它：

```C#
ticks.Subscribe(
    tick => Console.WriteLine($"Tick {tick}"));
```

这是在传递一个委托，而不是 `IObservable<T>.Subscribe` 所需的 `IObserver<T>` 。我们很快就会提到 `IObserver<T>` ，但这里示例使用的是 System.Reactive NuGet 包中的一个扩展方法：

```C#
// From the System.Reactive library's ObservableExtensions class
public static IDisposable Subscribe<T>(this IObservable<T> source, Action<T> onNext)
```

这是一个辅助方法，它将委托包装在一个 `IObserver<T>` 的实现中，然后将其传递给 `IObservable<T>.Subscribe`。于是我们可以只编写一个简单的方法（而不是 `IObserver<T>` 的完整实现），并且可观察源每次想要提供值时都会调用我们提供的回调。使用这种辅助方法比我们自己实现 Rx 的接口更常见。

## 热源和冷源

Since an IObservable<T> cannot supply us with values until we subscribe, the time at which we subscribe can be important. Imagine an IObservable<Trade> describing trades occurring in some market. If the information it supplies is live, it's not going to tell you about any trades that occurred before you subscribed. In Rx, sources of this kind are described as being hot.
由于 IObservable<T> 在我们订阅之前无法为我们提供值，因此我们订阅的时间可能很重要。想象一下 IObservable<Trade> 描述某个市场中发生的交易。如果它提供的信息是实时的，它不会告诉您在您订阅之前发生的任何交易。在 Rx 中，此类来源被描述为热门来源。

Not all sources are hot. There's nothing stopping an IObservable<T> always supplying the exact same sequence of events to any subscriber no matter when the call to Subscribe occurs. (Imagine an IObservable<Trade> which, instead of reporting live information, generates notifications based on recorded historical trade data.) Sources where it doesn't matter at all when you subscribe are known as cold sources.
并非所有来源都是热门的。无论何时发生对 Subscribe 的调用，都无法阻止 IObservable<T> 始终向任何订阅者提供完全相同的事件序列。 （想象一下 IObservable<Trade> ，它不是报告实时信息，而是根据记录的历史交易数据生成通知。）当您订阅时根本不重要的来源称为冷来源。

Here are some sources that might be represented as hot observables:
以下是一些可能表示为热可观察量的来源：

Measurements from a sensor
传感器测量
Price ticks from a trading exchange
来自交易交易所的价格变动
An event source that distributes events immediately such as Azure Event Grid
立即分发事件的事件源，例如 Azure 事件网格
mouse movements 鼠标移动
timer events 定时器事件
broadcasts like ESB channels or UDP network packets
广播，如 ESB 通道或 UDP 网络数据包
And some examples of some sources that might make good cold observables:
以及一些可能成为良好的冷可观测值的来源的示例：

the contents of a collection (such as is returned by the ToObservable extension method for IEnumerable<T>)
集合的内容（例如 IEnumerable<T> 的 ToObservable 扩展方法返回的内容）
a fixed range of values, such as Observable.Range produces
固定范围的值，例如 Observable.Range 产生
events generated based on an algorithm, such as Observable.Generate produces
基于算法生成的事件，例如 Observable.Generate 产生
a factory for an asynchronous operation, such as FromAsync returns
异步操作的工厂，例如 FromAsync 返回
events produced by running conventional code such as a loop; you can create such sources with Observable.Create
运行常规代码（例如循环）产生的事件；您可以使用 Observable.Create 创建此类源
a streaming event provides such as Azure Event Hub or Kafka (or any other streaming-style source which holds onto events from the past to be able to deliver events from a particular moment in the stream; so not an event source in the Azure Event Grid style)
流事件提供，例如 Azure 事件中心或 Kafka（或任何其他流式源，它们保留过去的事件，以便能够传递流中特定时刻的事件；因此不是 Azure 事件网格中的事件源风格）
Not all sources are strictly completely hot or cold. For example, you could imagine a slight variation on a live IObservable<Trade> where the source always reports the most recent trade to new subscribers. Subscribers can count on immediately receiving something, and will then be kept up to date as new information arrives. The fact that new subscribers will always receive (potentially quite old) information is a cold-like characteristic, but it's only that first event that is cold. It's still likely that a brand new subscriber will have missed lots of information that would have been available to earlier subscribers, making this source more hot than cold.
并非所有来源都是严格意义上的完全热或冷。例如，您可以想象实时 IObservable<Trade> 的细微变化，其中源始终向新订阅者报告最新交易。订阅者可以立即收到信息，并在新信息到达时及时了解最新情况。事实上，新订阅者总是会收到（可能很旧的）信息，这是一个类似冷的特征，但只有第一个事件是冷的。一个全新的订阅者仍然有可能错过许多早期订阅者可以获得的信息，从而使这个来源变得更热而不是冷。

There's an interesting special case in which a source of events has been designed to enable applications to receive every single event in order, exactly once. Event streaming systems such as Kafka or Azure Event Hub have this characteristic—they retain events for a while, to ensure that consumers don't miss out even if they fall behind from time to time. The standard input (stdin) for a process also has this characteristic: if you run a command line tool and start typing input before it is ready to process it, the operating system will hold that input in a buffer, to ensure that nothing is lost. Windows does something similar for desktop applications: each application thread gets a message queue so that if you click or type when it's not able to respond, the input will eventually be processed. We might think of these sources as cold-then-hot. They're like cold sources in that we won't miss anything just because it took us some time to start receiving events, but once we start retrieving the data, then we can't generally rewind back to the start. So once we're up and running they are more like hot events.
有一个有趣的特殊情况，其中事件源被设计为使应用程序能够按顺序接收每个事件，并且只接收一次。诸如 Kafka 或 Azure Event Hub 之类的事件流系统就具有这样的特点——它们将事件保留一段时间，以确保消费者即使不时落后也不会错过。进程的标准输入 (stdin) 也具有此特征：如果您运行命令行工具并在准备好处理输入之前开始键入输入，操作系统会将该输入保存在缓冲区中，以确保不会丢失任何内容。 Windows 对桌面应用程序做了类似的事情：每个应用程序线程都有一个消息队列，这样，如果您在无法响应时单击或键入，输入最终将被处理。我们可能认为这些来源先冷后热。它们就像冷源一样，我们不会因为花了一些时间才开始接收事件而错过任何东西，但一旦我们开始检索数据，我们通常就无法倒回到开始处。因此，一旦我们启动并运行，它们就更像是热门事件。

This kind of cold-then-hot source can present a problem if we want to attach multiple subscribers. If the source starts providing events as soon as subscription occurs, then that's fine for the very first subscriber: it will receive any events that were backed up waiting for us to start. But if we wanted to attach multiple subscribers, we've got a problem: that first subscriber might receive all the notifications that were sitting waiting in some buffer before we manage to attach the second subscriber. The second subscriber will miss out.
如果我们想要附加多个订阅者，这种先冷后热的源可能会出现问题。如果订阅发生后源就开始提供事件，那么这对于第一个订阅者来说没问题：它将接收所有已备份等待我们开始的事件。但是，如果我们想要附加多个订阅者，就会遇到一个问题：在我们设法附加第二个订阅者之前，第一个订阅者可能会收到在某个缓冲区中等待的所有通知。第二个订阅者将会错过。

In these cases, we really want some way to rig up all our subscribers before kicking things off. We want subscription to be separate from the act of starting. By default, subscribing to a source implies that we want it to start, but Rx defines a specialised interface that can give us more control: IConnectableObservable<T>. This derives from IObservable<T>, and adds just a single method, Connect:
在这些情况下，我们确实希望在开始之前以某种方式来配置所有订阅者。我们希望订阅与启动行为分开。默认情况下，订阅源意味着我们希望它启动，但 Rx 定义了一个专门的接口，可以为我们提供更多控制： IConnectableObservable<T> 。它派生自 IObservable<T> ，并仅添加一个方法 Connect ：

public interface IConnectableObservable<out T> : IObservable<T>
{
    IDisposable Connect();
}
This is useful in these scenarios where there will be some process that fetches or generates events and we need to make sure we're prepared before that starts. Because an IConnectableObservable<T> won't start until you call Connect, it provides you with a way to attach however many subscribers you need before events begin to flow.
这在这些场景中非常有用，其中将有一些获取或生成事件的进程，并且我们需要确保在开始之前做好准备。由于 IConnectableObservable<T> 在您调用 Connect 之前不会启动，因此它为您提供了一种在事件开始流动之前附加所需数量的订阅者的方法。

The 'temperature' of a source is not necessarily evident from its type. Even when the underlying source is an IConnectableObservable<T>, that can often be hidden behind layers of code. So whether a source is hot, cold, or something in between, most of the time we just see an IObservable<T>. Since IObservable<T> defines just one method, Subscribe, you might be wondering how we can do anything interesting with it. The power comes from the LINQ operators that the System.Reactive NuGet library supplies.
源的“温度”不一定从其类型中明显看出。即使底层源是 IConnectableObservable<T> ，它通常也隐藏在代码层后面。因此，无论源是热源、冷源还是介于两者之间的源，大多数时候我们只看到一个 IObservable<T> 。由于 IObservable<T> 只定义了一个方法 Subscribe ，您可能想知道我们如何用它做任何有趣的事情。其功能来自 System.Reactive NuGet 库提供的 LINQ 运算符。

## LINQ 运算符及其组合

So far I've shown only a very simple LINQ example, using the Where operator to filter events down to ones that meet certain criteria. To give you a flavour of how we can build more advanced functionality through composition, I'm going to introduce an example scenario.
到目前为止，我仅展示了一个非常简单的 LINQ 示例，使用 Where 运算符将事件筛选为满足特定条件的事件。为了让您了解如何通过组合构建更高级的功能，我将介绍一个示例场景。

Suppose you want to write a program that watches some folder on a filesystem, and performs automatic processing any time something in that folder changes. For example, web developers often want to trigger automatic rebuilds of their client side code when they save changes in the editor so they can quickly see the effect of their changes. Filesystem changes often come in bursts. Text editors might perform a few distinct operations when saving a file. (Some save modifications to a new file, then perform a couple of renames once this is complete, because this avoids data loss if a power failure or system crash happens to occur at the moment you save the file.) So you typically won't want to take action as soon as you detect file activity. It would be better to give it a moment to see if any more activity occurs, and take action only after everything has settled down.
假设您想编写一个程序来监视文件系统上的某个文件夹，并在该文件夹中的某些内容发生更改时执行自动处理。例如，Web 开发人员通常希望在编辑器中保存更改时触发客户端代码的自动重建，以便快速查看更改的效果。文件系统更改通常会突然发生。保存文件时，文本编辑器可能会执行一些不同的操作。 （有些将修改保存到新文件，然后在完成后执行几次重命名，因为这样可以避免在保存文件时发生电源故障或系统崩溃时丢失数据。）因此您通常不会希望在检测到文件活动后立即采取行动。最好给它一点时间，看看是否有更多活动发生，并在一切稳定下来后才采取行动。

So we should not react directly to filesystem activity. We want take action at those moments when everything goes quiet after a flurry of activity. Rx does not offer this functionality directly, but it's possible for us to create a custom operator by combing some of the built-in operators. The following code defines an Rx operator that detects and reports such things. If you're new to Rx (which seems likely if you're reading this) it probably won't be instantly obvious how this works. This is a significant step up in complexity from the examples I've shown so far because this came from a real application. But I'll walk through it step by step, so all will become clear.
因此我们不应该直接对文件系统活动做出反应。我们希望在一系列活动过后一切归于平静的时刻采取行动。 Rx 不直接提供此功能，但我们可以通过组合一些内置运算符来创建自定义运算符。以下代码定义了一个 Rx 运算符来检测并报告此类事件。如果您是 Rx 新手（如果您正在阅读本文，这似乎很可能），那么它可能不会立即明显看出它是如何工作的。与我迄今为止所展示的示例相比，这在复杂性上有了显着的进步，因为它来自真实的应用程序。但我会一步一步地进行，这样一切都会变得清晰。

static class RxExt
{
    public static IObservable<IList<T>> Quiescent<T>(
        this IObservable<T> src,
        TimeSpan minimumInactivityPeriod,
        IScheduler scheduler)
    {
        IObservable<int> onoffs =
            from _ in src
            from delta in 
               Observable.Return(1, scheduler)
                         .Concat(Observable.Return(-1, scheduler)
                                           .Delay(minimumInactivityPeriod, scheduler))
            select delta;
        IObservable<int> outstanding = onoffs.Scan(0, (total, delta) => total + delta);
        IObservable<int> zeroCrossings = outstanding.Where(total => total == 0);
        return src.Buffer(zeroCrossings);
    }
}
The first thing to say about this is that we are effectively defining a custom LINQ-style operator: this is an extension method which, like all of the LINQ operators Rx supplies, takes an IObservable<T> as its implicit argument, and produces another observable source as its result. The return type is slightly different: it's IObservable<IList<T>>. That's because once we return to a state of inactivity, we will want to process everything that just happened, so this operator will produce a list containing every value that the source reported in its most recent flurry of activity.
首先要说的是，我们正在有效地定义一个自定义 LINQ 样式运算符：这是一个扩展方法，与 Rx 提供的所有 LINQ 运算符一样，它采用 IObservable<T> 作为其隐式参数，并产生另一个可观察的源作为其结果。返回类型略有不同：它是 IObservable<IList<T>> 。这是因为一旦我们返回到不活动状态，我们将希望处理刚刚发生的所有事情，因此该运算符将生成一个列表，其中包含源在最近的一系列活动中报告的每个值。

When we want to show how an Rx operator behaves, we typically draw a 'marble' diagram. This is a diagram showing one or more IObservable<T> event sources, with each one being illustrated by a horizontal line. Each event that a source produces is illustrated by a circle (or 'marble') on that line, with the horizontal position representing timing. Typically, the line has a vertical bar on its left indicating the instant at which the application subscribed to the source, unless it happens to produce events immediately, in which case it will start with a marble. If the line has an arrowhead on the right, that indicates that the observable's lifetime extends beyond the diagram. Here's a diagram showing how the Quiescent operator above response to a particular input:
当我们想要展示 Rx 运算符的行为方式时，我们通常会绘制“大理石”图。这是一张显示一个或多个 IObservable<T> 事件源的图，每个事件源都用一条水平线表示。源产生的每个事件都用该线上的圆圈（或“大理石”）来说明，水平位置表示时间。通常，该线的左侧有一个垂直条，指示应用程序订阅源的时刻，除非它恰好立即产生事件，在这种情况下它将以弹珠开始。如果该线的右侧有箭头，则表明可观察量的生命周期超出了图表。下面的图表显示了上面的 Quiescent 运算符如何响应特定输入：

An Rx marble diagram illustrating two observables. The first is labelled 'source', and it shows six events, labelled numerically. These fall into three groups: events 1 and 2 occur close together, and are followed by a gap. Then events 3, 4, and 5 are close together. And then after another gap event 6 occurs, not close to any. The second observable is labelled 'source.Quiescent(TimeSpan.FromSeconds(2), Scheduler.Default)'. It shows three events. The first is labelled '1, 2', and its horizontal position shows that it occurs a little bit after the '2' event on the 'source' observable. The second event on the second observable is labelled '3,4,5' and occurs a bit after the '5' event on the 'source' observable. The third event from on the second observable is labelled '6', and occurs a bit after the '6' event on the 'source' observable. The image conveys the idea that each time the source produces some events and then stops, the second observable will produce an event shortly after the source stops, which will contain a list with all of the events from the source's most recent burst of activity.

This shows that the source (the top line) produced a couple of events (the values 1 and 2, in this example), and then stopped for a bit. A little while after it stopped, the observable returned by the Quiescent operator (the lower line) produced a single event with a list containing both of those events ([1,2]). Then the source started up again, producing the values 3, 4, and 5 in fairly quick succession, and then going quiet for a bit. Again, once the quiet spell had gone on for long enough, the source returned by Quiescent produced a single event containing all of the source events from this second burst of activity ([3,4,5]). And then the final bit of source activity shown in this diagram consists of a single event, 6, followed by more inactivity, and again, once the inactivity has gone on for long enough the Quiescent source produces a single event to report this. And since that last 'burst' of activity from the source contained only a single event, the list reported by this final output from the Quiescent observable is a list with a single value: [6].
这表明源（顶行）产生了几个事件（在本例中为值 1 和 2 ），然后停止了一会儿。停止后不久， Quiescent 运算符返回的可观察对象（下面一行）生成了一个事件，其中包含包含这两个事件的列表 ( [1,2] )。然后源再次启动，以相当快的速度连续生成值 3 、 4 和 5 ，然后安静一会儿。同样，一旦安静期持续足够长的时间， Quiescent 返回的源就会生成一个事件，其中包含第二次活动突发 ( [3,4,5] ) 中的所有源事件。然后，此图中显示的源活动的最后一位由单个事件 6 组成，随后是更多的不活动，再次，一旦不活动持续足够长的时间， Quiescent 源产生一个事件来报告这一点。由于来自源的最后一次“突发”活动仅包含一个事件，因此 Quiescent 可观察对象的最终输出报告的列表是具有单个值的列表： [6] 。

So how does the code shown achieve this? The first thing to notice about the Quiescent method is that it's just using other Rx LINQ operators (the Return, Scan, Where, and Buffer operators are explicitly visible, and the query expression will be using the SelectMany operator, because that's what C# query expressions do when they contain two from clauses in a row) in a combination that produces the final IObservable<IList<T>> output.
那么所示的代码是如何实现这一目标的呢？关于 Quiescent 方法首先要注意的是，它只是使用其他 Rx LINQ 运算符（ Return 、 Scan 、 Where 、和 Buffer 运算符是显式可见的，并且查询表达式将使用 SelectMany 运算符，因为这就是 C# 查询表达式在包含两个 from 子句时所做的操作行）的组合，产生最终的 IObservable<IList<T>> 输出。

This is Rx's compositional approach, and it is how we normally use Rx. We use a mixture of operators, combined (composed) in a way that produces the effect we want.
这就是Rx的组合方式，也是我们通常使用Rx的方式。我们使用混合运算符，以产生我们想要的效果的方式组合（组合）。

But how does this particular combination produce the effect we want? There are a few ways we could get the behaviour that we're looking for from a Quiescent operator, but the basic idea of this particular implementation is that it keeps count of how many events have happened recently, and then produces a result every time that number drops back to zero. The outstanding variable refers to the IObservable<int> that tracks the number of recent events, and this marble diagram shows what it produces in response to the same source events as were shown on the preceding diagram:
但这种特殊的组合如何产生我们想要的效果呢？我们可以通过几种方法从 Quiescent 运算符获取我们正在寻找的行为，但是这个特定实现的基本思想是它记录最近发生的事件数量，然后每当该数字回落到零时就会产生一个结果。 outstanding 变量引用跟踪最近事件数量的 IObservable<int> ，此弹珠图显示了它响应与之前相同的 source 事件而生成的内容如上图所示：

How the Quiescent operator counts the number of outstanding events. An Rx marble diagram illustrating two observables. The first is labelled 'source', and it shows the same six events as the preceding figure, labelled numerically, but this time also color-coded so that each event has a different color. As before, these events fall into three groups: events 1 and 2 occur close together, and are followed by a gap. Then events 3, 4, and 5 are close together. And then after another gap event 6 occurs, not close to any. The second observable is labelled 'outstanding' and for each of the events on the 'source' observable, it shows two events. Each such pair has the same color as on the 'source' line; the coloring is just to make it easier to see how events on this line are associated with events on the 'source' line. The first of each pair appears directly below its corresponding event on the 'source' line, and has a number that is always one higher than its immediate predecessor; the very first item shows a number of 1. The first item from the second pair is the next to appear on this line, and therefore has a number of 2. But then the second item from the first pair appears, and this lowers the number back to 1, and it's followed by the second item from the second pair, which shows 0. Since the second batch of events on the first line appear fairly close together, we see values of 1, 2, 1, 2, 1, and then 0 for these. The final event on the first line, labelled 6, has a corresponding pair on the second line reporting values of 1 and then 0. The overall effect is that each value on the second, 'outstanding' line tells us how many items have emerged from the 'source' line in the last 2 seconds.

I've colour coded the events this time so that I can show the relationship between source events and corresponding events produced by outstanding. Each time source produces an event, outstanding produces an event at the same time, in which the value is one higher than the preceding value produced by outstanding. But each such source event also causes outstanding to produce another event two seconds later. (It's two seconds because in these examples, I've presumed that the first argument to Quiescent is TimeSpan.FromSeconds(2), as shown on the first marble diagram.) That second event always produces a value that is one lower than whatever the preceding value was.
这次我对事件进行了颜色编码，以便可以显示 source 事件与 outstanding 生成的相应事件之间的关系。每次 source 产生一个事件时， outstanding 同时产生一个事件，其中的值比 outstanding 产生的前一个值高 1。但每个此类 source 事件也会导致 outstanding 两秒后产生另一个事件。 （这是两秒，因为在这些示例中，我假设 Quiescent 的第一个参数是 TimeSpan.FromSeconds(2) ，如第一个弹珠图所示。）第二个事件总是产生一个值该值比之前的值低一个。

This means that each event to emerge from outstanding tells us how many events source produced within the last two seconds. This diagram shows that same information in a slightly different form: it shows the most recent value produced by outstanding as a graph. You can see the value goes up by one each time source produces a new value. And two seconds after each value produced by source, it drops back down by one.
这意味着 outstanding 中出现的每个事件都会告诉我们过去两秒内产生了多少个 source 事件。该图以略有不同的形式显示了相同的信息：它以图表的形式显示了 outstanding 生成的最新值。您可以看到，每次 source 产生新值时，该值都会增加 1。在 source 生成每个值两秒后，它会回落一。

The number of outstanding events as a graph. An Rx marble diagram illustrating the 'source' observables, and the second observable from the preceding diagram this time illustrated as a bar graph showing the latest value. This makes it easier to see that the 'outstanding' value goes up each time a new value emerges from 'source', and then goes down again two seconds later, and that when values emerge close together this running total goes higher. It also makes it clear that the value drops to zero between the 'bursts' of activity.

In simple cases like the final event 6, in which it's the only event that happens at around that time, the outstanding value goes up by one when the event happens, and drops down again two seconds later. Over on the left of the picture it's a little more complex: we get two events in fairly quick succession, so the outstanding value goes up to one and then up to two, before falling back down to one and then down to zero again. The middle section looks a little more messy—the count goes up by one when the source produces event 3, and then up to two when event 4 comes in. It then drops down to one again once two seconds have passed since the 3 event, but then another event, 5, comes in taking the total back up to two. Shortly after that it drops back to one again because it has now been two seconds since the 4 event happened. And then a bit later, two seconds after the 5 event it drops back to zero again.
在简单的情况下，例如最终事件 6 ，它是在该时间附近发生的唯一事件，当事件发生时 outstanding 值会增加 1，然后再次下降两秒钟后。在图片的左侧，情况稍微复杂一些：我们以相当快的速度连续获得两个事件，因此 outstanding 值上升到 1，然后上升到 2，然后回落到 1，然后再次降至零。中间部分看起来有点混乱——当 source 产生事件 3 时，计数增加 1，当事件 4 到来时，计数增加 2一旦 3 事件过去两秒，它就会再次下降到 1，但随后另一个事件 5 出现，使总数回到 2。不久之后，它再次回落到 1，因为距离 4 事件发生已经过去了两秒。然后稍后，在 5 事件发生两秒后，它再次回落到零。

That middle section is the messiest, but it's also most representative of the kind of activity this operator is designed to deal with. Remember, the whole point here is that we're expecting to see flurries of activity, and if those represents filesystem activity, they will tend to be slightly chaotic in nature, because storage devices don't always have entirely predictable performance characteristics (especially if it's a magnetic storage device with moving parts, or remote storage in which variable networking delays might come into play).
中间部分是最混乱的，但它也最能代表该操作员要处理的活动类型。请记住，这里的要点是，我们期望看到大量的活动，如果这些活动代表文件系统活动，那么它们本质上往往会稍微混乱，因为存储设备并不总是具有完全可预测的性能特征（特别是如果它是一种带有移动部件的磁性存储设备，或者是远程存储，其中可能会出现可变的网络延迟）。

With this measure of recent activity in hand, we can spot the end of bursts of activity by watching for when outstanding drops back to zero, which is what the observable referred to by zeroCrossing in the code above does. (That's just using the Where operator to filter out everything except the events where outstanding's current value returns to zero.)
有了对近期活动的测量，我们可以通过观察 outstanding 何时回落到零来发现活动爆发的结束，这就是 zeroCrossing 中的可观察到的内容。上面的代码确实如此。 （这只是使用 Where 运算符过滤掉除 outstanding 的当前值返回零的事件之外的所有内容。）

But how does outstanding itself work? The basic approach here is that every time source produces a value, we actually create a brand new IObservable<int>, which produces exactly two values. It immediately produces the value 1, and then after the specified timespan (2 seconds in these examples) it produces the value -1. That's what's going in in this clause of the query expression:
但是 outstanding 本身是如何工作的呢？这里的基本方法是，每次 source 产生一个值时，我们实际上创建一个全新的 IObservable<int> ，它恰好产生两个值。它立即生成值 1，然后在指定的时间跨度（这些示例中为 2 秒）后生成值 -1。这就是查询表达式的子句中的内容：

from delta in Observable
    .Return(1, scheduler)
    .Concat(Observable
        .Return(-1, scheduler)
        .Delay(minimumInactivityPeriod, scheduler))
I said Rx is all about composition, and that's certainly the case here. We are using the very simple Return operator to create an IObservable<int> that immediately produces just a single value and then terminates. This code calls that twice, once to produce the value 1 and again to produce the value -1. It uses the Delay operator so that instead of getting that -1 value immediately, we get an observable that waits for the specified time period (2 seconds in these examples, but whatever minimumInactivityPeriod is in general) before producing the value. And then we use Concat to stitch those two together into a single IObservable<int> that produces the value 1, and then two seconds later produces the value -1.
我说过 Rx 就是关于组合，这里的情况当然也是如此。我们使用非常简单的 Return 运算符来创建一个 IObservable<int> ，它立即生成一个值，然后终止。此代码调用两次，一次生成值 1 ，再次生成值 -1 。它使用 Delay 运算符，这样我们就不会立即获取 -1 值，而是得到一个等待指定时间段（在这些示例中为 2 秒，但无论 minimumInactivityPeriod 通常是在产生值之前。然后我们使用 Concat 将这两个拼接在一起形成一个 IObservable<int> ，生成值 1 ，然后两秒后生成值 -1

Although this produces a brand new IObservable<int> for each source event, the from clause shown above is part of a query expression of the form from ... from .. select, which the C# compiler turns into a call to SelectMany, which has the effect of flattening those all back into a single observable, which is what the onoffs variable refers to. This marble diagram illustrates that:
尽管这会为每个 source 事件生成一个全新的 IObservable<int> ，但上面显示的 from 子句是 from ... from .. select ，C# 编译器将其转换为对 SelectMany 的调用，其效果是将它们全部展平为单个可观察量，这就是 onoffs 变量所引用的内容。这个弹珠图说明了这一点：

The number of outstanding events as a graph. Several Rx marble diagrams, starting with the 'source' observable from earlier figures, followed by one labelled with the LINQ query expression in the preceding example, which shows 6 separate marble diagrams, one for each of the elements produced by 'source'. Each consists of two events: one with value 1, positioned directly beneath the corresponding event on 'source' to indicate that they happen simultaneously, and then one with the value -1 two seconds later. Beneath this is a marble diagram labelled 'onoffs' which contains all the same events from the preceding 6 diagrams, but merged into a single sequence. These are all colour coded ot make it easier to see how these events correspond to the original events on 'source'. Finally, we have the 'outstanding' marble diagram which is exactly the same as in the preceding figure.

This also shows the outstanding observable again, but we can now see where that comes from: it is just the running total of the values emitted by the onoffs observable. This running total observable is created with this code:
这也再次显示了 outstanding 可观察值，但我们现在可以看到它的来源：它只是 onoffs 可观察值发出的值的运行总计。这个运行总计可观察值是使用以下代码创建的：

IObservable<int> outstanding = onoffs.Scan(0, (total, delta) => total + delta);
Rx's Scan operator works much like the standard LINQ Aggregate operator, in that it cumulatively applies an operation (addition, in this case) to every single item in a sequence. The different is that whereas Aggregate produces just the final result once it reaches the end of the sequence, Scan shows all of its working, producing the accumulated value so far after each input. So this means that outstanding will produce an event every time onoffs produces one, and that event's value will be the running total—the sum total of every value from onoffs so far.
Rx 的 Scan 运算符的工作方式与标准 LINQ Aggregate 运算符非常相似，因为它会将操作（在本例中为加法）累积地应用于序列中的每个项目。不同之处在于， Aggregate 在到达序列末尾时仅生成最终结果，而 Scan 显示其所有工作情况，在每次输入后生成到目前为止的累积值。因此，这意味着每次 onoffs 生成一个事件时， outstanding 都会生成一个事件，并且该事件的值将是运行总计 - 来自 onoffs 到目前为止。

So that's how outstanding comes to tell us how many events source produced within the last two seconds (or whatever minimumActivityPeriod has been specified).
这就是 outstanding 告诉我们过去两秒内产生了多少个 source 事件（或指定的任何 minimumActivityPeriod 事件）。

The final piece of the puzzle is how we go from the zeroCrossings (which produces an event every time the source has gone quiescent) to the output IObservable<IList<T>>, which provides all of the events that happened in the most recent burst of activity. Here we're just using Rx's Buffer operator, which is designed for exactly this scenario: it slices its input into chunks, producing an event for each chunk, the value of which is an IList<T> containing the items for the chunk. Buffer can slice things up a few ways, but in this case we're using the form that starts a new slice each time some IObservable<T> produces an item. Specifically, we're telling Buffer to slice up the source by creating a new chunk every time zeroCrossings produces a new event.
难题的最后一部分是我们如何从 zeroCrossings （每次源静止时生成一个事件）到输出 IObservable<IList<T>> ，它提供了所有事件发生在最近的一次活动爆发中。这里我们只使用 Rx 的 Buffer 运算符，它是专门为这种场景而设计的：它将输入切成块，为每个块生成一个事件，其值为 IList<T> 包含该块的项目。 Buffer 可以通过几种方式对事物进行切片，但在本例中，我们使用的形式是在每次 IObservable<T> 生成一个项目时启动一个新切片。具体来说，我们告诉 Buffer 通过在每次 zeroCrossings 产生新事件时创建一个新块来分割 source 。

(One last detail, just in case you saw it and were wondering, is that this method requires an IScheduler. This is an Rx abstraction for dealing with timing and concurrency. We need it because we need to be able to generate events after a one second delay, and that sort of time-driven activity requires a scheduler.)
（最后一个细节，以防万一您看到它并想知道，该方法需要一个 IScheduler 。这是一个用于处理时序和并发性的 Rx 抽象。我们需要它，因为我们需要能够一秒延迟后生成事件，这种时间驱动的活动需要调度程序。）

We'll get into all of these operators and the workings of schedulers in more detail in later chapters. For now, the key point is that we typically use Rx by creating a combination of LINQ operators that process and combine IObservable<T> sources to define the logic that we require.
我们将在后面的章节中更详细地介绍所有这些运算符和调度程序的工作原理。目前，关键点是我们通常通过创建 LINQ 运算符的组合来使用 Rx，这些运算符处理和组合 IObservable<T> 源来定义我们所需的逻辑。

Notice that nothing in that example actually called the one and only method that IObservable<T> defines (Subscribe). There will always be something somewhere that ultimately consumes the events, but most of the work of using Rx tends to entail declaratively defining the IObservable<T>s we need.
请注意，该示例中没有任何内容实际调用 IObservable<T> 定义的唯一方法 ( Subscribe )。总会有一些东西最终会消耗事件，但是使用 Rx 的大部分工作往往需要声明性地定义我们需要的 IObservable<T> 。

Now that you've seen an example of what Rx programming looks like, we can address some obvious questions about why Rx exists at all.
现在您已经看到了 Rx 编程的示例，我们可以解决一些关于 Rx 为何存在的明显问题。

## 使用.NET 事件会出什么问题？

.NET has had built-in support for events from the very first version that shipped over two decades ago—events are part of .NET's type system. The C# language has intrinsic support for this in the form of the event keyword, along with specialized syntax for subscribing to events. So why, when Rx turned up some 10 years later, did it feel the need to invent its own representation for streams of events? What was wrong with the event keyword?
从二十多年前发布的第一个版本开始，.NET 就内置了对事件的支持——事件是 .NET 类型系统的一部分。 C# 语言以 event 关键字的形式对此提供内在支持，并提供用于订阅事件的专用语法。那么，为什么当 Rx 大约 10 年后出现时，它觉得有必要发明自己的事件流表示形式呢？ event 关键字出了什么问题？

The basic problem with .NET events is that they get special handling from the .NET type system. Ironically, this makes them less flexible than if there had been no built-in support for the idea of events. Without .NET events, we would have needed some sort of object-based representation of events, at which point you can do all the same things with events that you can do with any other objects: you could store them in fields, pass them as arguments to methods, define methods on them and so on.
.NET 事件的基本问题是它们需要从 .NET 类型系统进行特殊处理。具有讽刺意味的是，这使得它们比没有对事件想法的内置支持更不灵活。如果没有 .NET 事件，我们将需要某种基于对象的事件表示，此时您可以对事件执行与对任何其他对象执行的所有相同操作：您可以将它们存储在字段中，将它们传递为方法的参数、定义方法的方法等等。

To be fair to .NET version 1, it wasn't really possible to define a good object-based representation of events without generics, and .NET didn't get those until version 2 (three and a half years after .NET 1.0 shipped). Different event sources need to be able to report different data, and .NET events provided a way to parameterize events by type. But once generics came along, it became possible to define types such as IObservable<T>, and the main advantage that events offered went away. (The other benefit was some language support for implementing and subscribing to events, but in principle that's something that could have been done for Rx if Microsoft had chosen to. It's not a feature that required events to be fundamentally different from other features of the type system.)
公平地说，.NET 版本 1 确实不可能在没有泛型的情况下定义良好的基于​​对象的事件表示，并且 .NET 直到版本 2 才实现这些（.NET 1.0 发布三年半后） ）。不同的事件源需要能够报告不同的数据，而.NET事件提供了一种按类型参数化事件的方法。但是，一旦泛型出现，就可以定义诸如 IObservable<T> 之类的类型，并且事件提供的主要优势就消失了。 （另一个好处是对实现和订阅事件的一些语言支持，但原则上，如果 Microsoft 选择的话，这是可以为 Rx 做的事情。这不是一个要求事件与该类型的其他功能有根本不同的功能系统。）

Consider the example we've just worked through. It was possible to define our own custom LINQ operator, Quiescent, because IObservable<T> is just an interface like any other, meaning that we're free to write extension methods for it. You can't write an extension method for an event.
考虑一下我们刚刚完成的例子。可以定义我们自己的自定义 LINQ 运算符 Quiescent ，因为 IObservable<T> 只是一个与其他接口一样的接口，这意味着我们可以自由地为其编写扩展方法。您无法为事件编写扩展方法。

Also, we are able to wrap or adapt IObservable<T> sources. Quiescent took an IObservable<T> as an input, and combined various Rx operators to produce another observable as an output. Its input was a source of events that could be subscribed to, and its output was also a source of events that could be subscribed to. You can't do this with .NET events—you can't write a method that accepts an event as an argument, or that returns an event.
此外，我们还能够包装或改编 IObservable<T> 源。 Quiescent 将 IObservable<T> 作为输入，并组合各种 Rx 运算符以生成另一个可观察量作为输出。它的输入是可以订阅的事件源，它的输出也是可以订阅的事件源。您无法使用 .NET 事件执行此操作 — 您无法编写接受事件作为参数或返回事件的方法。

These limitations are sometimes described by saying that .NET events are not first class citizens. There are things you can do with values or references in .NET that you can't do with events.
这些限制有时被描述为“.NET 事件不是一等公民”。有些事情可以用 .NET 中的值或引用来完成，而不能用事件来完成。

If we represent an event source as a plain old interface, then it is a first class citizen: it can use all of the functionality we expect with other objects and values precisely because it's not something special.
如果我们将事件源表示为普通的旧接口，那么它就是一等公民：它可以使用我们期望的其他对象和值的所有功能，正是因为它不是什么特殊的东西。

## 流（Stream）呢？
I've described IObservable<T> as representing a stream of events. This raises an obvious question: .NET already has System.IO.Stream, so why not just use that?
我将 IObservable<T> 描述为代表事件流。这提出了一个明显的问题：.NET 已经有了 System.IO.Stream ，那么为什么不直接使用它呢？

The short answer is that streams are weird because they represent an ancient concept in computing dating back long before the first ever Windows operating system shipped, and as such they have quite a lot of historical baggage. This means that even a scenario as simple as "I have some data, and want to make that available immediately to all interested parties" is surprisingly complex to implement though the Stream type.
简而言之，流很奇怪，因为它们代表了一个古老的计算概念，可以追溯到第一个 Windows 操作系统发布之前很久，因此它们有相当多的历史包袱。这意味着，即使是像“我有一些数据，并且希望立即将其提供给所有感兴趣的各方”这样简单的场景，通过 Stream 类型实现起来也非常复杂。

Moreover, Stream doesn't provide any way to indicate what type of data will emerge—it only knows about bytes. Since .NET's type system supports generics, it is natural to want the types that represent event streams to indicate the event type through a type parameter.
此外， Stream 不提供任何方法来指示将出现什么类型的数据——它只知道字节。由于.NET的类型系统支持泛型，因此很自然地希望表示事件流的类型通过类型参数来指示事件类型。

So even if you did use Stream as part of your implementation, you'd want to introduce some sort of wrapper abstraction. If IObservable<T> didn't exist, you'd need to invent it.
因此，即使您确实使用 Stream 作为实现的一部分，您也需要引入某种包装器抽象。如果 IObservable<T> 不存在，您就需要发明它。

It's certainly possible to use IO streams in Rx, but they are not the right primary abstraction.
当然可以在 Rx 中使用 IO 流，但它们不是正确的主要抽象。

(If you are unconvinced, see Appendix A: What's Wrong with Classic IO Streams for a far more detailed explanation of exactly why Stream is not well suited to this task.)
（如果您不相信，请参阅附录 A：经典 IO 流有何问题，以获取关于 Stream 为何不太适合此任务的更详细解释。）

Now that we've seen why IObservable<T> needs to exist, we need to look at its counterpart, IObserver<T>.
现在我们已经了解了为什么 IObservable<T> 需要存在，我们需要看看它的对应项 IObserver<T> 。

## IObserver<T>

Earlier, I showed the definition of IObservable<T>. As you saw, it has just one method, Subscribe. And this method takes just one argument, of type IObserver<T>. So if you want to observe the events that an IObservable<T> has to offer, you must supply it with an IObserver<T>. In the examples so far, we've just supplied a simple callback, and Rx has wrapped that in an implementation of IObserver<T> for us, but even though this is very often the way we will receive notifications in practice, you still need to understand IObserver<T> to use Rx effectively. It is not a complex interface:
之前，我展示了 IObservable<T> 的定义。正如您所看到的，它只有一个方法 Subscribe 。该方法只接受一个 IObserver<T> 类型的参数。因此，如果您想观察 IObservable<T> 必须提供的事件，则必须为其提供 IObserver<T> 。在到目前为止的示例中，我们只是提供了一个简单的回调，Rx 已将其包装在 IObserver<T> 的实现中，但即使这通常是我们在实践中接收通知的方式，您仍然需要了解 IObserver<T> 才能有效地使用 Rx。它不是一个复杂的接口：

public interface IObserver<in T>
{
    void OnNext(T value);
    void OnError(Exception error);
    void OnCompleted();
}
As with IObservable<T>, you can find the source for IObserver<T> in the .NET runtime GitHub repository, because both of these interfaces are built into the runtime libraries.
与 IObservable<T> 一样，您可以在 .NET 运行时 GitHub 存储库中找到 IObserver<T> 的源代码，因为这两个接口都内置于运行时库中。

If we wanted to create an observer that printed values to the console it would be as easy as this:
如果我们想创建一个将值打印到控制台的观察者，那么就像这样简单：

public class MyConsoleObserver<T> : IObserver<T>
{
    public void OnNext(T value)
    {
        Console.WriteLine($"Received value {value}");
    }

    public void OnError(Exception error)
    {
        Console.WriteLine($"Sequence faulted with {error}");
    }

    public void OnCompleted()
    {
        Console.WriteLine("Sequence terminated");
    }
}
In the preceding chapter, I used a Subscribe extension method that accepted a delegate which it invoked each time the source produced an item. This method is defined by Rx's ObservableExtensions class, which also defines various other extension methods for IObservable<T>. It includes overloads of Subscribe that enable me to write code that has the same effect as the preceding example, without needing to provide my own implementation of IObserver<T>:
在前一章中，我使用了 Subscribe 扩展方法，该方法接受委托，每次源生成项目时都会调用该委托。该方法由 Rx 的 ObservableExtensions 类定义，该类还为 IObservable<T> 定义了各种其他扩展方法。它包含 Subscribe 的重载，使我能够编写与前面的示例具有相同效果的代码，而无需提供我自己的 IObserver<T> 实现：

source.Subscribe(
    value => Console.WriteLine($"Received value {value}"),
    error => Console.WriteLine($"Sequence faulted with {error}"),
    () => Console.WriteLine("Sequence terminated")
);
The overloads of Subscribe where we don't pass all three methods (e.g., my earlier example just supplied a single callback corresponding to OnNext) are equivalent to writing an IObserver<T> implementation where one or more of the methods simply has an empty body. Whether we find it more convenient to write our own type that implements IObserver<T>, or just supply callbacks for some or all of its OnNext, OnError and OnCompleted method, the basic behaviour is the same: an IObservable<T> source reports each event with a call to OnNext, and tells us that the events have come to an end either by calling OnError or OnCompleted.
我们不传递所有三个方法的 Subscribe 重载（例如，我之前的示例仅提供了与 OnNext 相对应的单个回调）相当于编写 IObserver<T> 的类型更方便，或者只是为其部分或全部 OnNext 、 OnError 和 OnCompleted 方法，基本行为是相同的： IObservable<T> 源通过调用 OnNext 报告每个事件，并告诉我们事件已经结束调用 OnError 或 OnCompleted 。

If you're wondering whether the relationship between IObservable<T> and IObserver<T> is similar to the relationship between IEnumerable<T> and IEnumerator<T>, then you're onto something. Both IEnumerator<T> and IObservable<T> represent potential sequences. With both of these interfaces, they will only supply data if we ask them for it. To get values out of an IEnumerable<T>, an IEnumerator<T> needs to come into existence, and similarly, to get values out of an IObservable<T> requires an IObserver<T>.
如果您想知道 IObservable<T> 和 IObserver<T> 之间的关系是否类似于 IEnumerable<T> 和 IEnumerator<T> 之间的关系，那么您是到某物上。 IEnumerator<T> 和 IObservable<T> 都代表潜在的序列。对于这两个接口，它们只会在我们要求时提供数据。要从 IEnumerable<T> 中获取值，需要存在 IEnumerator<T> ，同样，要从 IObservable<T> 中获取值需要 IObserver<T>

The difference reflects the fundamental pull vs push difference between IEnumerable<T> and IObservable<T>. Whereas with IEnumerable<T> we ask the source to create an IEnumerator<T> for us which we can then use to retrieve items (which is what a C# foreach loop does), with IObservable<T>, the source does not implement IObserver<T>: it expects us to supply an IObserver<T> and it will then push its values into that observer.
这种差异反映了 IEnumerable<T> 和 IObservable<T> 之间基本的拉动与推动差异。而对于 IEnumerable<T> ，我们要求源为我们创建一个 IEnumerator<T> ，然后我们可以使用它来检索项目（这就是 C# foreach 循环的作用），对于 IObservable<T> ，源不会实现 IObserver<T> ：它期望我们提供一个 IObserver<T> ，然后它将其值推送到该观察者中。

So why does IObserver<T> have these three methods? Remember when I said that in an abstract sense, IObserver<T> represents the same thing as IEnumerable<T>? I meant it. It might be an abstract sense, but it is precise: IObservable<T> and IObserver<T> were designed to preserve the exact meaning of IEnumerable<T> and IEnumerator<T>, changing only the detailed mechanism of consumption.
那么为什么 IObserver<T> 有这三个方法呢？还记得我说过，在抽象意义上， IObserver<T> 与 IEnumerable<T> 表示相同的东西吗？我是认真的。这可能是一个抽象的含义，但它是精确的： IObservable<T> 和 IObserver<T> 旨在保留 IEnumerable<T> 和 IEnumerator<T> 的确切含义，仅改变具体的消费机制。

To see what that means, think about what happens when you iterate over an IEnumerable<T> (with, say, a foreach loop). With each iteration (and more precisely, on each call to the enumerator's MoveNext method) there are three things that could happen:
要了解这意味着什么，请考虑当您迭代 IEnumerable<T> （例如，使用 foreach 循环）时会发生什么。每次迭代（更准确地说，每次调用枚举器的 MoveNext 方法）都可能发生三件事：

MoveNext could return true to indicate that a value is available in the enumerator's Current property
MoveNext 可以返回 true 以指示枚举器的 Current 属性中存在可用值
MoveNext could throw an exception
MoveNext 可能会引发异常
MoveNext could return false to indicate that you've reached the end of the collection
MoveNext 可能会返回 false 以指示您已到达集合末尾
These three outcomes correspond precisely to the three methods defined by IObserver<T>. We could describe these in slightly more abstract terms:
这三个结果恰好对应于 IObserver<T> 定义的三种方法。我们可以用稍微更抽象的术语来描述这些：

Here's another item 这是另一项
It has all gone wrong
一切都出了问题
There are no more items
没有更多商品了
That describes the three things that either can happen next when consuming either an IEnumerable<T> or an IObservable<T>. The only difference is the means by which consumers discover this. With an IEnumerable<T> source, each call to MoveNext will tell us which of these three applies. And with an IObservable<T> source, it will tell you one of these three things with a call to the corresponding member of your IObserver<T> implementation.
这描述了在使用 IEnumerable<T> 或 IObservable<T> 时接下来可能发生的三件事。唯一的区别在于消费者发现这一点的方式。对于 IEnumerable<T> 源，每次调用 MoveNext 都会告诉我们这三个中哪一个适用。对于 IObservable<T> 源，它会通过调用 IObserver<T> 实现的相应成员来告诉您这三件事之一。

## Rx 序列的基本规则

Notice that two of the three outcomes in the list above are terminal. If you're iterating through an IEnumerable<T> with a foreach loop, and it throws an exception, the foreach loop will terminate. The C# compiler understands that if MoveNext throws, the IEnumerator<T> is now done, so it disposes it and then allows the exception to propagate. Likewise, if you get to the end of a sequence, then you're done, and the compiler understands that too: the code it generates for a foreach loop detects when MoveNext returns false and when that happens it disposes the enumerator and then moves onto the code after the loop.
请注意，上面列表中的三个结果中有两个是最终结果。如果您使用 foreach 循环迭代 IEnumerable<T> ，并且它引发异常，则 foreach 循环将终止。 C# 编译器知道，如果 MoveNext 抛出，则 IEnumerator<T> 现在已完成，因此它会处理它，然后允许异常传播。同样，如果到达序列的末尾，那么就完成了，编译器也理解这一点：它为 foreach 循环生成的代码会检测 MoveNext 何时返回 false当发生这种情况时，它会处理枚举器，然后移至循环后的代码。

These rules might seem so obvious that we might never even think about them when iterating over IEnumerable<T> sequences. What might be less immediately obvious is that exactly the same rules apply for an IObservable<T> sequence. If an observable source either tells an observer that the sequence has finished, or reports an error, then in either case, that is the last thing the source is allowed to do to the observer.
这些规则可能看起来非常明显，以至于我们在迭代 IEnumerable<T> 序列时可能永远不会考虑它们。可能不太明显的是，完全相同的规则适用于 IObservable<T> 序列。如果可观察源告诉观察者序列已完成，或者报告错误，那么无论哪种情况，这都是允许源对观察者做的最后一件事。

That means these examples would be breaking the rules:
这意味着这些示例将违反规则：

public static void WrongOnError(IObserver<int> obs)
{
    obs.OnNext(1);
    obs.OnError(new ArgumentException("This isn't an argument!"));
    obs.OnNext(2);  // Against the rules! We already reported failure, so iteration must stop
}

public static void WrongOnCompleted(IObserver<int> obs)
{
    obs.OnNext(1);
    obs.OnCompleted();
    obs.OnNext(2);  // Against the rules! We already said we were done, so iteration must stop
}

public static void WrongOnErrorAndOnCompleted(IObserver<int> obs)
{
    obs.OnNext(1);
    obs.OnError(new ArgumentException("A connected series of statements was not supplied"));

    // This next call is against the rules because we reported an error, and you're not
    // allowed to make any further calls after you did that.
    obs.OnCompleted();
}

public static void WrongOnCompletedAndOnError(IObserver<int> obs)
{
    obs.OnNext(1);
    obs.OnCompleted();

    // This next call is against the rule because already said we were done.
    // When you terminate a sequence you have to pick between OnCompleted or OnError
    obs.OnError(new ArgumentException("Definite proposition not established"));
}
These correspond in a pretty straightforward way to things we already know about IEnumerable<T>:
这些以非常简单的方式对应于我们已经了解的 IEnumerable<T> 的内容：

WrongOnError: if an enumerator throws from MoveNext, it's done and you mustn't call MoveNext again, so you won't be getting any more items out of it
WrongOnError ：如果枚举器从 MoveNext 抛出异常，则它已完成，您不能再次调用 MoveNext ，因此您不会再从中获取任何项目它
WrongOnCompleted: if an enumerator returns false from MoveNext, it's done and you mustn't call MoveNext again, so you won't be getting any more items out of it
WrongOnCompleted ：如果枚举器从 MoveNext 返回 false ，则完成，您不能再次调用 MoveNext ，因此您不会从中获取更多物品
WrongOnErrorAndOnCompleted: if an enumerator throws from MoveNext, that means its done, it's done and you mustn't call MoveNext again, meaning it won't have any opportunity to tell that it's done by returning false from MoveNext
WrongOnErrorAndOnCompleted ：如果枚举器从 MoveNext 抛出异常，则意味着它已完成，它已经完成，并且您不能再次调用 MoveNext ，这意味着它不会有任何有机会通过从 MoveNext 返回 false 来告诉它已完成
WrongOnCompletedAndOnError: if an enumerator returns false from MoveNext, it's done and you mustn't call MoveNext again, meaning it won't have any opportunity to also throw an exception
WrongOnCompletedAndOnError ：如果枚举器从 MoveNext 返回 false ，则已完成，您不能再次调用 MoveNext ，这意味着它不会有机会也抛出异常
Because IObservable<T> is push-based, the onus for obeying all of these rules fall on the observable source. With IEnumerable<T>, which is pull-based, it's up to the code using the IEnumerator<T> (e.g. a foreach loop) to obey these rules. But they are essentially the same rules.
因为 IObservable<T> 是基于推送的，所以遵守所有这些规则的责任就落在了可观察的源身上。对于基于拉的 IEnumerable<T> ，由使用 IEnumerator<T> （例如 foreach 循环）的代码来遵守这些规则。但它们本质上是相同的规则。

There's an additional rule for IObserver<T>: if you call OnNext you must wait for it to return before making any more method calls into the same IObserver<T>. That means this code breaks the rules:
IObserver<T> 还有一条附加规则：如果调用 OnNext ，则必须等待它返回，然后才能对同一个 IObserver<T> 进行更多方法调用。这意味着这段代码违反了规则：

public static void EverythingEverywhereAllAtOnce(IEnumerable<int> obs)
{
    Random r = new();
    for (int i = 0; i < 10000; ++i)
    {
        int v = r.Next();
        Task.Run(() => obs.OnNext(v)); // Against the rules!
    }}
This calls obs.OnNext 10,000 times, but it executes these calls as individual tasks to be run on the thread pool. The thread pool is designed to be able to execute work in parallel, and that's a a problem here because nothing here ensures that one call to OnNext completes before the next begins. We've broken the rule that says we must wait for each call to OnNext to return before calling either OnNext, OnError, or OnComplete on the same observer. (Note: this assumes that the caller won't subscribe the same observer to multiple different sources. If you do that, you can't assume that all calls to its OnNext will obey the rules, because the different sources won't have any way of knowing they're talking to the same observer.)
这会调用 obs.OnNext 10,000 次，但它将这些调用作为要在线程池上运行的单独任务来执行。线程池被设计为能够并行执行工作，这是一个问题，因为这里没有任何内容可以确保对 OnNext 的一次调用在下一次调用开始之前完成。我们打破了规则，即在调用 OnNext 、 OnError 或 OnComplete 的每次调用返回> 在同一个观察者上。 （注意：这假设调用者不会将同一个观察者订阅到多个不同的源。如果这样做，则不能假设对其 OnNext 的所有调用都会遵守规则，因为不同的源消息来源无法知道他们正在与同一个观察者交谈。）

This rule is the only form of back pressure built into Rx.NET: since the rules forbid calling OnNext if a previous call to OnNext is still in progress, this enables an IObserver<T> to limit the rate at which items arrive. If you just don't return from OnNext until you're ready, the source is obliged to wait. However, there are some issues with this. Once schedulers get involved, the underlying source might not be connected directly to the final observer. If you use something like ObserveOn it's possible that the IObserver<T> subscribed directly to the source just puts items on a queue and immediately returns, and those items will then be delivered to the real observer on a different thread. In these cases, the 'back pressure' caused by taking a long time to return from OnNext only propagates as far as the code pulling items off the queue.
此规则是 Rx.NET 中内置的唯一背压形式：由于如果先前对 OnNext 的调用仍在进行中，则规则禁止调用 OnNext ，因此这会启用 IObserver<T> 限制物品到达的速率。如果您在准备好之前才从 OnNext 返回，则源必须等待。然而，这存在一些问题。一旦调度程序介入，底层源可能不会直接连接到最终观察者。如果您使用类似 ObserveOn 的内容，则直接订阅源的 IObserver<T> 可能只是将项目放入队列并立即返回，然后这些项目将被传递给真正的观察者一个不同的线程。在这些情况下，由于从 OnNext 返回需要很长时间而导致的“背压”仅传播到从队列中拉出项目的代码。

It may be possible to use certain Rx operators (such as Buffer or Sample) to mitigate this, but there are no built-in mechanisms for cross-thread propagation of back pressure. Some Rx implementations on other platforms have attempted to provide integrated solutions for this; in the past when the Rx.NET development community has looked into this, some thought that these solutions were problematic, and there is no consensus on what a good solution looks like. So with Rx.NET, if you need to arrange for sources to slow down when you are struggling to keep up, you will need to introduce some mechanism of your own. (Even with Rx platforms that do offer built-in back pressure, they can't provide a general-purpose answer to the question: how do we make this source provide events more slowly? How (or even whether) you can do that will depend on the nature of the source. So some bespoke adaptation is likely to be necessary in any case.)
可以使用某些 Rx 运算符（例如 Buffer 或 Sample ）来缓解这种情况，但没有用于背压跨线程传播的内置机制。其他平台上的一些 Rx 实现已尝试为此提供集成解决方案；过去，当 Rx.NET 开发社区研究这个问题时，有些人认为这些解决方案是有问题的，并且对于什么是好的解决方案还没有达成共识。因此，对于 Rx.NET，如果您在努力跟上时需要安排源放慢速度，则需要引入一些您自己的机制。 （即使使用确实提供内置背压的 Rx 平台，它们也无法提供以下问题的通用答案：我们如何使该源更慢地提供事件？您如何（甚至是否）可以做到这一点取决于来源的性质。因此在任何情况下都可能需要进行一些定制的调整。）

This rule in which we must wait for OnNext to return is tricky and subtle. It's perhaps less obvious than the others, because there's no equivalent rule for IEnumerable<T>—the opportunity to break this rule only arises when the source pushes data into the application. You might look at the example above and think "well who would do that?" However, multithreading is just an easy way to show that it is technically possible to break the rule. The harder cases are where single-threaded re-entrancy occurs. Take this code:
我们必须等待 OnNext 返回的这条规则很棘手且微妙。它可能不如其他规则那么明显，因为 IEnumerable<T> 没有等效的规则 - 仅当源将数据推送到应用程序时才会出现打破此规则的机会。看到上面的例子，您可能会想“谁会这么做呢？”然而，多线程只是证明技术上可以打破规则的一种简单方法。更困难的情况是发生单线程重入的情况。采取这个代码：

public class GoUntilStopped
{
    private readonly IObserver<int> observer;
    private bool running;

    public GoUntilStopped(IObserver<int> observer)
    {
        this.observer = observer;
    }

    public void Go()
    {
        this.running = true;
        for (int i = 0; this.running; ++i)
        {
            this.observer.OnNext(i);
        }
    }

    public void Stop()
    {
        this.running = false;
        this.observer.OnCompleted();
    }
}
This class takes an IObserver<int> as a constructor argument. When you call its Go method, it repeatedly calls the observer's OnNext until something calls its Stop method.
该类采用 IObserver<int> 作为构造函数参数。当您调用其 Go 方法时，它会重复调用观察者的 OnNext 直到有东西调用其 Stop 方法。

Can you see the bug?
你能看到这个错误吗？

We can take a look at what happens by supplying an IObserver<int> implementation:
我们可以通过提供 IObserver<int> 实现来看看会发生什么：

public class MyObserver : IObserver<int>
{
    private GoUntilStopped? runner;

    public void Run()
    {
        this.runner = new(this);
        Console.WriteLine("Starting...");
        this.runner.Go();
        Console.WriteLine("Finished");
    }

    public void OnCompleted()
    {
        Console.WriteLine("OnCompleted");
    }

    public void OnError(Exception error) { }

    public void OnNext(int value)
    {
        Console.WriteLine($"OnNext {value}");
        if (value > 3)
        {
            Console.WriteLine($"OnNext calling Stop");
            this.runner?.Stop();
        }
        Console.WriteLine($"OnNext returning");
    }
}
Notice that the OnNext method looks at its input, and if it's greater than 3, it tells the GoUntilStopped object to stop.
请注意， OnNext 方法查看其输入，如果它大于 3，它会告诉 GoUntilStopped 对象停止。

Let's look at the output:
让我们看看输出：

Starting...
OnNext 0
OnNext returning
OnNext 1
OnNext returning
OnNext 2
OnNext returning
OnNext 3
OnNext returning
OnNext 4
OnNext calling Stop
OnCompleted
OnNext returning
Finished
The problem is right near the end. Specifically, these two lines:
问题已经临近尾声了。具体来说，这两行：

OnCompleted
OnNext returning
This tells us that the call to our observer's OnCompleted happened before a call in progress to OnNext returned. It didn't take multiple threads to make this occur. It happened because the code in OnNext decides whether it wants to keep receiving events, and when it wants to stop, it immediately calls the GoUntilStopped object's Stop method. There's nothing wrong with that. Observers are allowed to make outbound calls to other objects inside OnNext, and it's actually quite common for an observer to inspect an incoming event and decide that it wants to stop.
这告诉我们，对观察者的 OnCompleted 的调用发生在对 OnNext 的调用返回之前。不需要多个线程来实现这一点。发生这种情况是因为 OnNext 中的代码决定是否要继续接收事件，当它想要停止时，它会立即调用 GoUntilStopped 对象的 Stop 方法。这没有什么问题。观察者可以对 OnNext 内的其他对象进行出站调用，实际上观察者检查传入事件并决定要停止是很常见的。

The problem is in the GoUntilStopped.Stop method. This calls OnCompleted but it makes no attempt to determine whether a call to OnNext is in progress.
问题出在 GoUntilStopped.Stop 方法中。这会调用 OnCompleted ，但不会尝试确定是否正在进行对 OnNext 的调用。

This can be a surprisingly tricky problem to solve. Suppose GoUntilStopped did detect that there was a call in progress to OnNext. What then? In the multithreaded case, we could have solved this by using lock or some other synchronization primitive to ensure that calls into the observer happened one at at time, but that won't work here: the call to Stop has happened on the same thread that called OnNext. The call stack will look something like this at the moment where Stop has been called and it wants to call OnCompleted:
这可能是一个非常棘手的问题。假设 GoUntilStopped 确实检测到正在进行对 OnNext 的调用。然后怎样呢？在多线程情况下，我们可以通过使用 lock 或其他一些同步原语来解决这个问题，以确保对观察者的调用一次发生一次，但这在这里不起作用：对 Stop 发生在与 OnNext 相同的线程上。在调用 Stop 并想要调用 OnCompleted 时，调用堆栈将如下所示：

`GoUntilStopped.Go`
  `MyObserver.OnNext`
    `GoUntilStopped.Stop`
Our GoUntilStopped.Stop method needs to wait for OnNext to return before calling OnCompleted. But notice that the OnNext method can't return until our Stop method returns. We've managed to create a deadlock with single-threaded code!
我们的 GoUntilStopped.Stop 方法需要等待 OnNext 返回，然后再调用 OnCompleted 。但请注意，在 Stop 方法返回之前， OnNext 方法无法返回。我们已经成功地用单线程代码造成了死锁！

In this case it's not all that hard to fix: we could modify Stop so it just sets the running field to false, and then move the call to OnComplete into the Go method, after the for loop. But more generally this can be a hard problem to fix, and it's one of the reasons for using the System.Reactive library instead of just attempting to implement IObservable<T> and IObserver<T> directly. Rx has general purpose mechanisms for solving exactly this kind of problem. (We'll see these when we look at Scheduling.) Moreover, all of the implementations Rx provides take advantage of these mechanisms for you.
在这种情况下，修复起来并不难：我们可以修改 Stop ，这样它只需将 running 字段设置为 false ，然后将调用移动到 < b3> 进入 Go 方法，在 for 循环之后。但更一般地说，这可能是一个很难解决的问题，这是使用 System.Reactive 库而不是仅仅尝试实现 IObservable<T> 和 IObserver<T> 的原因之一直接地。 Rx 具有解决此类问题的通用机制。 （当我们查看调度时，我们将看到这些。）此外，Rx 提供的所有实现都为您利用了这些机制。

If you're using Rx by composing its built-in operators in a declarative way, you never have to think about these rules. You get to depend on these rules in your callbacks that receive the events, and it's mostly Rx's problem to keep to the rules. So the main effect of these rules is that it makes life simpler for code that consumes events.
如果您通过以声明方式组合内置运算符来使用 Rx，则您永远不必考虑这些规则。您必须在接收事件的回调中依赖这些规则，而遵守这些规则主要是 Rx 的问题。因此，这些规则的主要作用是，它使使用事件的代码变得更简单。

These rules are sometimes expressed as a grammar. For example, consider this regular expression:
这些规则有时被表达为语法。例如，考虑这个正则表达式：

(OnNext)*(OnError|OnComplete)
This formally captures the basic idea: there can be any number of calls to OnNext (maybe even zero calls), that occur in sequence, followed by either an OnError or an OnComplete, but not both, and there must be nothing after either of these.
这正式捕获了基本思想：可以有任意数量的对 OnNext 的调用（甚至可能是零个调用），这些调用按顺序发生，后跟 OnError 或 OnComplete ，但不能同时存在，并且其中任何一个之后都不能有任何内容。

One last point: sequences may be infinite. This is true for IEnumerable<T>. It's perfectly possible for an enumerator to return true every time MoveNext is returned, in which case a foreach loop iterating over it will never reach the end. It might choose to stop (with a break or return), or some exception that did not originate from the enumerator might cause the loop to terminate, but it's absolutely acceptable for an IEnumerable<T> to produce items for as long as you keep asking for them. The same is true of a IObservable<T>. If you subscribe to an observable source, and by the time your program exits you've not received a call to either OnComplete or OnError, that's not a bug.
最后一点：序列可能是无限的。对于 IEnumerable<T> 来说也是如此。每次返回 MoveNext 时，枚举器完全有可能返回 true ，在这种情况下，对其进行迭代的 foreach 循环将永远不会到达末尾。它可能会选择停止（使用 break 或 return ），或者某些并非源自枚举器的异常可能会导致循环终止，但对于 IEnumerable<T> 只要你不断要求就可以生产物品。 IObservable<T> 也是如此。如果您订阅了可观察源，并且当您的程序退出时，您还没有收到对 OnComplete 或 OnError 的调用，那么这不是一个错误。

So you might argue that this is a slightly better way to describe the rules formally:
因此，您可能会认为这是正式描述规则的稍微更好的方式：

(OnNext)*(OnError|OnComplete)?
More subtly, observable sources are allowed to do nothing at all. In fact there's a built-in implementation to save developers from the effort of writing a source that does nothing: if you call Observable.Never<int>() it will return an IObservable<int>, and if you subscribe to that, it will never call any methods on your observer. This might not look immediately useful—it is logically equivalent to an IEnumerable<T> in which the enumerator's MoveNext method never returns, which might not be usefully distinguishable from crashing. It's slightly different with Rx, because when we model this "no items emerge ever" behaviour, we don't need to block a thread forever to do it. We can just decide never to call any methods on the observer. This may seem daft, but as you've seen with the Quiescent example, sometimes we create observable sources not because we want the actual items that emerge from it, but because we're interested in the instants when interesting things happen. It can sometimes be useful to be able to model "nothing interesting ever happens" cases. For example, if you have written some code to detect unexpected inactivity (e.g., a sensor that stops producing values), and wanted to test that code, your test could use a Never source instead of a real one, to simulate a broken sensor.
更巧妙的是，可观察源被允许什么都不做。事实上，有一个内置的实现可以使开发人员免于编写不执行任何操作的源代码：如果您调用 Observable.Never<int>() ，它将返回 IObservable<int> ，并且如果您订阅该源，它永远不会在你的观察者上调用任何方法。这看起来可能不会立即有用 - 它在逻辑上相当于 IEnumerable<T> ，其中枚举器的 MoveNext 方法永远不会返回，这可能无法有效地区分崩溃。它与 Rx 略有不同，因为当我们对这种“永远不会出现任何项目”行为进行建模时，我们不需要永远阻塞线程来执行此操作。我们可以决定永远不对观察者调用任何方法。这可能看起来很愚蠢，但正如您在 Quiescent 示例中看到的那样，有时我们创建可观察源并不是因为我们想要从中出现的实际项目，而是因为我们对有趣的时刻感兴趣事情发生。有时，能够对“没有发生任何有趣的事情”的情况进行建模会很有用。例如，如果您编写了一些代码来检测意外的不活动（例如，停止产生值的传感器），并且想要测试该代码，您的测试可以使用 Never 源而不是真实的源，模拟损坏的传感器。

We're not quite done with the Rx's rules, but the last one applies only when we choose to unsubscribe from a source before it comes to a natural end.
我们还没有完全了解 Rx 的规则，但最后一个规则仅适用于我们在自然结束之前选择取消订阅源的情况。

## 订阅的生命周期

There's one more aspect of the relationship between observers and observables to understand: the lifetime of a subscription.
观察者和可观察对象之间的关系还有一个方面需要理解：订阅的生命周期。

You already know from the rules of IObserver<T> that a call to either OnComplete or OnError denotes the end of a sequence. We passed an IObserver<T> to IObservable<T>.Subscribe, and now the subscription is over. But what if we want to stop the subscription earlier?
您已经从 IObserver<T> 的规则知道，对 OnComplete 或 OnError 的调用表示序列的结束。我们将 IObserver<T> 传递给 IObservable<T>.Subscribe ，现在订阅结束了。但是如果我们想提前停止订阅怎么办？

I mentioned earlier that the Subscribe method returns an IDisposable, which enables us to cancel our subscription. Perhaps we only subscribed to a source because our application opened some window showing the status of some process, and we wanted to update the window to reflect that's process's progress. If the user closes that window, we no longer have any use for the notifications. And although we could just ignore all further notifications, that could be a problem if the thing we're monitoring never reaches a natural end. Our observer would continue to receive notifications for the lifetime of the application. This is a waste of CPU power (and thus power consumption, with corresponding implications for battery life and environmental impact) and it can also prevent the garbage collector from reclaiming memory that should have become free.
我之前提到过 Subscribe 方法返回 IDisposable ，这使我们能够取消订阅。也许我们只订阅了一个源，因为我们的应用程序打开了一些显示某个进程状态的窗口，并且我们希望更新该窗口以反映该进程的进度。如果用户关闭该窗口，我们就不再使用通知。尽管我们可以忽略所有进一步的通知，但如果我们正在监视的事情永远不会自然结束，这可能会成为一个问题。我们的观察者将在应用程序的生命周期内继续收到通知。这是对 CPU 功率的浪费（因此会造成功耗，相应地影响电池寿命和环境影响），并且还会阻止垃圾收集器回收本应空闲的内存。

So we are free to indicate that we no longer wish to receive notifications by calling Dispose on the object returned by Subscribe. There are, however, a few non-obvious details.
因此，我们可以通过对 Subscribe 返回的对象调用 Dispose 来表明我们不再希望接收通知。然而，还有一些不明显的细节。

## 订阅的处理（Disposal）是可选的

You are not required to call Dispose on the object returned by Subscribe. Obviously if you want to remain subscribed to events for the lifetime of your process, this makes sense: you never stop using the object, so of course you don't dispose it. But what might be less obvious is that if you subscribe to an IObservable<T> that does come to an end, it automatically tidies up after itself.
您不需要对 Subscribe 返回的对象调用 Dispose 。显然，如果您想在进程的生命周期内保持订阅事件，这是有道理的：您永远不会停止使用该对象，因此您当然不会处置它。但可能不太明显的是，如果您订阅的 IObservable<T> 确实结束了，它会自动清理完毕。

IObservable<T> implementations are not allowed to assume that you will definitely call Dispose, so they are required to perform any necessary cleanup if they stop by calling the observer's OnCompleted or OnError. This is unusual. In most cases where a .NET API returns a brand new object created on your behalf that implements IDisposable, it's an error not to dispose it. But IDisposable objects representing Rx subscriptions are an exception to this rule. You only need to dispose them if you want them to stop earlier than they otherwise would.
IObservable<T> 实现不允许假设您肯定会调用 Dispose ，因此如果它们通过调用观察者的 OnCompleted 或 OnError 。这很不寻常。在大多数情况下，.NET API 返回代表您创建的实现 IDisposable 的全新对象，不处置它是错误的。但表示 Rx 订阅的 IDisposable 对象是此规则的一个例外。仅当您希望它们比其他情况更早停止时，您才需要处理它们。

## 取消订阅可能会很慢甚至无效

Dispose won't necessarily take effect instantly. Obviously it will take some non-zero amount of time in between your code calling into Dispose, and the Dispose implementation reaching the point where it actually does something. Less obviously, some observable sources may need to do non-trivial work to shut things down.
Dispose 不一定立即生效。显然，在代码调用 Dispose 和 Dispose 实现达到实际执行某些操作之间需要一些非零的时间。不太明显的是，一些可观察的来源可能需要做一些重要的工作才能关闭。

A source might create a thread to be able to monitor for and report whatever events it represents. (That would happen with the filesystem source shown above when running on Linux on .NET 8, because the FileSystemWatcher class itself creates its own thread on Linux.) It might take a while for the thread to detect that it is supposed to shut down.
源可能会创建一个线程，以便能够监视和报告它所代表的任何事件。 （当在 .NET 8 上的 Linux 上运行时，上面显示的文件系统源会发生这种情况，因为 FileSystemWatcher 类本身在 Linux 上创建自己的线程。）线程可能需要一段时间才能检测到它应该关闭。

It is fairly common practice for an IObservable<T> to represent some underlying work. For example, Rx can take any factory method that returns a Task<T> and wrap it as an IObservable<T>. It will invoke the factory once for each call to Subscribe, so if there are multiple subscribers to a single IObservable<T> of this kind, each one effectively gets its own Task<T>. This wrapper is able to supply the factory with a CancellationToken, and if an observer unsubscribes by calling Dispose before the task naturally runs to completion, it will put that CancellationToken into a cancelled state. This might have the effect of bringing the task to a halt, but that will work only if the task happens to be monitoring the CancellationToken. Even if it is, it might take some time to bring things to a complete halt. Crucially, the Dispose call doesn't wait for that to happen. It will attempt to initiate cancellation but it may return before cancellation is complete.
IObservable<T> 代表一些基础工作是相当常见的做法。例如，Rx 可以采用任何返回 Task<T> 的工厂方法并将其包装为 IObservable<T> 。每次调用 Subscribe 时，它都会调用工厂一次，因此，如果此类 IObservable<T> 有多个订阅者，则每个订阅者实际上都会获得自己的 Task<T> .这个包装器能够为工厂提供 CancellationToken ，如果观察者在任务自然运行完成之前通过调用 Dispose 取消订阅，它将把 CancellationToken 进入取消状态。这可能会导致任务停止，但只有当任务恰好正在监视 CancellationToken 时才会起作用。即使是这样，也可能需要一些时间才能让事情完全停止。至关重要的是， Dispose 调用不会等待这种情况发生。它将尝试启动取消，但可能会在取消完成之前返回。

## 取消订阅 Rx 序列时的规则

The fundamental rules of Rx sequences described earlier only considered sources that decided when (or whether) to come to a halt. What if a subscriber unsubscribes early? There is only one rule:
前面描述的 Rx 序列的基本规则仅考虑决定何时（或是否）停止的源。如果订阅者提前取消订阅怎么办？只有一条规则：

Once the call to Dispose has returned, the source will make no further calls to the relevant observer. If you call Dispose on the object returned by Subscribe, then once that call returns you can be certain that the observer you passed in will receive no further calls to any of its three methods (OnNext, OnError, or OnComplete).
一旦对 Dispose 的调用返回，源将不再对相关观察者进行调用。如果您对 Subscribe 返回的对象调用 Dispose ，那么一旦该调用返回，您就可以确定您传入的观察者将不会再收到对其三个方法中任何一个的调用（ OnNext 、 OnError 或 OnComplete ）。

That might seem clear enough, but it leaves a grey area: what happens when you've called Dispose but it hasn't returned yet? The rules permit sources to continue to emit events in this case. In fact they couldn't very well require otherwise: it will invariably take some non-zero length of time for the Dispose implementation to make enough progress to have any effect, so in a multi-threaded world it it's always going to be possible that an event gets delivered in between the call to Dispose starting, and the call having any effect. The only situation in which you could depend on no further events emerging would be if your call to Dispose happened inside the OnNext handler. In this case the source will already have noted a call to OnNext is in progress so further calls were already blocked before the call to Dispose started.
这看起来似乎很清楚，但它留下了一个灰色区域：当您调用 Dispose 但它尚未返回时会发生什么？在这种情况下，规则允许源继续发出事件。事实上，他们不能很好地要求其他： Dispose 实现总是需要一些非零长度的时间才能取得足够的进展以产生任何效果，所以在多线程世界中它是在对 Dispose 的调用开始和调用产生任何效果之间总是有可能传递事件。唯一可以保证不再出现任何事件的情况是，对 Dispose 的调用发生在 OnNext 处理程序内部。在这种情况下，源已经注意到对 OnNext 的调用正在进行中，因此在对 Dispose 的调用开始之前，进一步的调用已经被阻止。

But assuming that your observer wasn't already in the middle of an OnNext call, any of the following would be legal:
但假设您的观察者尚未处于 OnNext 调用过程中，则以下任何操作都是合法的：

stopping calls to IObserver<T> almost immediately after Dispose begins, even when it takes a relatively long time to bring any relevant underlying processes to a halt, in which case your observer will never receive an OnCompleted or OnError
在 Dispose 开始后几乎立即停止对 IObserver<T> 的调用，即使需要相对较长的时间才能停止任何相关的底层进程，在这种情况下，您的观察者将永远不会收到 < b2> 或 OnError
producing notifications that reflect the process of shutting down (including calling OnError if an error occurs while trying to bring things to a neat halt, or OnCompleted if it halted without problems)
生成反映关闭过程的通知（包括如果在尝试完全停止时发生错误则调用 OnError ，或者如果没有问题停止则调用 OnCompleted ）
producing a few more notifications for some time after the call to Dispose begins, but cutting them off at some arbitrary point, potentially losing track even of important things like errors that occurred while trying to bring things to a halt
在调用 Dispose 开始后一段时间内再生成一些通知，但会在某个任意点切断它们，甚至可能会丢失重要的事情，例如尝试停止时发生的错误
As it happens, Rx has a preference for the first option. If you're using an IObservable<T> implemented by the System.Reactive library (e.g., one returned by a LINQ operator) it is highly likely to have this characteristic. This is partly to avoid tricky situations in which observers try to do things to their sources inside their notification callbacks. Re-entrancy tends to be awkward to deal with, and Rx avoids ever having to deal with this particular form of re-entrancy by ensuring that it has already stopped delivering notifications to the observer before it begins the work of shutting down a subscription.
碰巧的是，Rx 更倾向于第一个选项。如果您使用的是由 System.Reactive 库实现的 IObservable<T> （例如，由 LINQ 运算符返回的库），则很可能具有此特征。这在一定程度上是为了避免观察者尝试在通知回调中对其源执行操作的棘手情况。重入往往很难处理，Rx 通过确保在开始关闭订阅的工作之前已经停止向观察者发送通知，避免了必须处理这种特定形式的重入。

This sometimes catches people out. If you need to be able to cancel some process that you are observing but you need to be able to observe everything it does up until the point that it stops, then you can't use unsubscription as the shutdown mechanism. As soon as you've called Dispose, the IObservable<T> that returned that IDisposable is no longer under any obligation to tell you anything. This can be frustrating, because the IDisposable returned by Subscribe can sometimes seem like such a natural and easy way to shut something down. But basic truth is this: once you've initiated unsubscription, you can't rely on getting any further notifications associated with that subscription. You might receive some—the source is allowed to carry on supplying items until the call to Dispose returns. But you can't rely on it—the source is also allowed to silence itself immediately, and that's what most Rx-implemented sources will do.
这有时会让人们感到困惑。如果您需要能够取消正在观察的某些进程，但需要能够观察它所做的一切直到它停止为止，那么您不能使用取消订阅作为关闭机制。一旦您调用了 Dispose ，返回 IDisposable 的 IObservable<T> 就不再有义务告诉您任何信息。这可能会令人沮丧，因为 Subscribe 返回的 IDisposable 有时看起来像是一种自然而简单的关闭某些东西的方法。但基本事实是：一旦您开始取消订阅，您就不能依赖于获取与该订阅相关的任何进一步通知。您可能会收到一些 — 允许源继续提供项目，直到对 Dispose 的调用返回。但你不能依赖它——源也可以立即让自己安静下来，这就是大多数 Rx 实现的源会做的事情。

One subtle consequence of this is that if an observable source reports an error after a subscriber has unsubscribed, that error might be lost. A source might call OnError on its observer, but if that's a wrapper provided by Rx relating to a subscription that has already been disposed, it just ignores the exception. So it's best to think of early unsubscription as inherently messy, a bit like aborting a thread: it can be done but information can be lost, and there are race conditions that will disrupt normal exception handling.
这样做的一个微妙后果是，如果可观察源在订阅者取消订阅后报告错误，则该错误可能会丢失。源可能会在其观察者上调用 OnError ，但如果这是 Rx 提供的与已释放的订阅相关的包装器，则它只会忽略异常。因此，最好将早期取消订阅视为本质上混乱的，有点像中止线程：它可以完成，但信息可能会丢失，并且存在会破坏正常异常处理的竞争条件。

In short, if you unsubscribe, then a source is not obliged to tell you when things stop, and in most cases it definitely won't tell you.
简而言之，如果您取消订阅，那么消息来源没有义务在事情停止时告诉您，而且在大多数情况下它肯定不会告诉您。

## 订阅的生命周期和组合

We typically combine multiple LINQ operators to express our processing requirements in Rx. What does this mean for subscription lifetime?
我们通常组合多个 LINQ 运算符来表达 Rx 中的处理要求。这对于订阅生命周期意味着什么？

For example, consider this:
例如，考虑一下：

IObservable<int> source = GetSource();
IObservable<int> filtered = source.Where(i => i % 2 == 0);
IDisposable subscription = filtered.Subscribe(
    i => Console.WriteLine(i),
    error => Console.WriteLine($"OnError: {error}"),
    () => Console.WriteLine("OnCompleted"));
We're calling Subscribe on the observable returned by Where. When we do that, it will in turn call Subscribe on the IObservable<int> returned by GetSource (stored in the source variable). So there is in effect a chain of subscriptions here. (We only have access to the IDisposable returned by filtered.Subscribe but the object that returns will be storing the IDisposable that it received when it called source.Subscribe.)
我们对 Where 返回的可观察对象调用 Subscribe 。当我们这样做时，它将依次对 GetSource 返回的 IObservable<int> （存储在 source 变量中）调用 Subscribe 。所以这里实际上存在一个订阅链。 （我们只能访问 filtered.Subscribe 返回的 IDisposable ，但返回的对象将存储调用 source.Subscribe 时收到的 IDisposable b9> .)

If the source comes to an end all by itself (by calling either OnCompleted or OnError), this cascades through the chain. So source will call OnCompleted on the IObserver<int> that was supplied by the Where operator. And that in turn will call OnCompleted on the IObserver<int> that was passed to filtered.Subscribe, and that will have references to the three methods we passed, so it will call our completion handler. So you could look at this by saying that source completes, it tells filtered that it has completed, which invokes our completion handler. (In reality this is a very slight oversimplification, because source doesn't tell filtered anything; it's actually talking to the IObserver<T> that filtered supplied. This distinction matters if you have multiple subscriptions active simultaneously for the same chain of observables. But in this case, the simpler way of describing it is good enough even if it's not absolutely precise.)
如果源完全自行结束（通过调用 OnCompleted 或 OnError ），则会通过链级联。因此， source 将在 Where 运算符提供的 IObserver<int> 上调用 OnCompleted 。反过来，它将在传递给 filtered.Subscribe 的 IObserver<int> 上调用 OnCompleted ，并且它将引用我们传递的三个方法，因此它将调用我们的完成处理程序。因此，您可以通过说 source 完成来看待这一点，它告诉 filtered 它已完成，这会调用我们的完成处理程序。 （实际上，这是一个非常轻微的过度简化，因为 source 没有告诉 filtered 任何东西；它实际上是在与 IObserver<T> 对话， filtered 提供。如果您对同一个可观察量链有多个同时活动的订阅，那么这种区别很重要。但在这种情况下，描述它的更简单的方法就足够好了，即使它不是绝对精确。）

In short, completion bubbles up from the source, through all the operators, and arrives at our handler.
简而言之，完成从源头冒出，通过所有操作符，到达我们的处理程序。

What if we unsubscribe early by calling subscription.Dispose()? In that case it all happens the other way round. The subscription returned by filtered.Subscribe is the first to know that we're unsubscribing, but it will then call Dispose on the object that was returned when it called source.Subscribe for us.
如果我们通过调用 subscription.Dispose() 提前取消订阅怎么办？在这种情况下，一切都会以相反的方式发生。 filtered.Subscribe 返回的 subscription 是第一个知道我们要取消订阅的，但它随后会在调用 < 时返回的对象上调用 Dispose 。 b4> 对我们来说。

Either way, everything from the source to the observer, including any operators that were sitting in between, gets shut down.
无论哪种方式，从源头到观察者的一切，包括中间的任何操作员，都会被关闭。

Now that we understand the relationship between an IObservable<T> source and the IObserver<T> interface that received event notifications, we can look at how we might create an IObservable<T> instance to represent events of interest in our application.
现在我们了解了 IObservable<T> 源和接收事件通知的 IObserver<T> 接口之间的关系，我们可以看看如何创建一个 IObservable<T> 实例来表示我们的应用程序中感兴趣的事件。