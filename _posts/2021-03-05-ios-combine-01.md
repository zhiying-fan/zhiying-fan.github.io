---
title: iOS Combine：核心概念
date: 2021-07-05 20:00:00 +0800
categories: [iOS]
tags: [combine]     # TAG names should always be lowercase
image:
  src: /assets/img/post/combine/header.jpeg
---

[Combine](https://developer.apple.com/documentation/combine) 是 iOS13 新加入的库，它的出现主要是为了解决当前我们在进行异步编程时遇到的痛点，各种 Delegate, Callback, Notification 混杂在我们的 App 中，导致异步事件混乱，难以调试代码，下图展示了当前常用的异步 API：

![async-event](/assets/img/post/combine/async-event.png)
_本图片来自 Combine 书籍_

Combine 和 RxSwift 中的核心思想和概念是一致的，所以如果你使用过 RxSwift，那么学习并使用 Combine 对你来说将是轻而易举的。

## 核心

Combine 的核心是 Publisher, Operator 和 Subscriber。

Publisher：负责发送消息

Operator：负责对消息进行处理，比如 map, filter 等

Subscriber：接收消息进行消费

大概流程如下图：

![process](/assets/img/post/combine/process.jpg)
_本图中的素材来自 Combine 书籍_

从 Publisher 到 Operator 再到 Subscriber 的调用是链式的，Operator 的输入是 Publisher，输出仍然是 Publisher，得益于 Operator 的高度解耦，我们可以对数据进行多次处理，并且对其的测试也是比较容易的。

下面分别对 Combine 中的几个核心概念进行说明，以下内容翻译自 [Using Combine](https://heckj.github.io/swiftui-notes/#coreconcepts)

## Publisher and Subscriber

两个关键概念，[publisher](https://developer.apple.com/documentation/combine/publisher) 和 [subscriber](https://developer.apple.com/documentation/combine/subscriber)，在 Swift 中被描述为协议。

当你谈论编程（尤其是 Swift 和 Combine）时，类型描述了很多。 当你说一个函数或方法返回一个值时，该值通常被描述为“此类型之一”。

Combine 就是定义随着时间的推移使用许多可能的值进行操作的过程。 Combine 还不仅仅是定义结果，它还定义了我们如何处理失败。 它不仅讨论可以返回的类型，还讨论可能发生的失败。

现在我们要引入的第一个核心概念是发布者。 当其被订阅之后，根据请求会提供数据， 没有任何订阅请求的发布者不会提供任何数据。 当你表达一个 Combine 的发布者时，应该用两种相关的类型来描述它：一种用于输出，一种用于失败。

![basic-types](/assets/img/post/combine/basic-types.svg)

这些通常使用泛型语法编写，该语法在描述类型的文本周围使用`<` 和 `>` 符号。这表示我们正在谈论这种类型的值的通用实例。例如，如果发布者返回了一个 `String` 类型的实例，并且可能以 `URLError` 实例的形式返回失败，那么发布者可能会用 `<String, URLError>` 来描述。

与发布者匹配的对应概念是订阅者，是第二个要介绍的核心概念。

订阅者负责请求数据并接受发布者提供的数据（和可能的失败）。 订阅者同样被描述为两种关联类型，一种用于输入，一种用于失败。 订阅者发起数据请求，并控制它接收的数据量。 它可以被认为是在 Combine 中起“驱动作用”的，因为如果没有订阅者，其他组件将保持闲置状态，没有数据会流动起来。

发布者和订阅者是相互连接的，它们构成了 Combine 的核心。 当你将订阅者连接到发布者时，两种类型都必须匹配：发布者的输出和订阅者的输入以及它们的失败类型。 将其可视化的一种方法是对两种类型进行一系列并行操作，其中两种类型都需要匹配才能将组件插入在一起。

![input-output](/assets/img/post/combine/input-output.svg)

第三个核心概念是操作符——一个既像订阅者又像发布者的对象。操作符是同时实现了[订阅者协议](https://developer.apple.com/documentation/combine/subscriber)和[发布者协议](https://developer.apple.com/documentation/combine/publisher)的类。 它们支持订阅发布者，并将结果发送给任何订阅者。

你可以用这些创建成链，用于处理和转换发布者提供的数据和订阅者请求的数据。我称这些组合序列为**管道**。

![pipeline](/assets/img/post/combine/pipeline.svg)

操作符可用于转换值或类型 - 输出和失败类型都可以。 操作符还可以拆分或复制流，或将流合并在一起。 操作符必须始终按输出/失败这样的类型组合对齐。 编译器将强制执行匹配类型，因此错误将导致编译器错误（如果幸运的话，会有一个有用的 *fixit* 片段建议给你解决方案）。

用 swift 编写的简单的 Combine 管道如下所示：

```swift
let _ = Just(5) (1)
    .map { value -> String in (2)
        // do something with the incoming value here
        // and return a string
        return "a string"
    }
    .sink { receivedValue in (3)
        // sink is the subscriber and terminates the pipeline
        print("The end result was \(receivedValue)")
    }
```

1. 管道从发布者 `Just` 开始，它用它定义的值（在本例中为整数 `5`）进行响应。输出类型为 `<Integer>`，失败类型为 `<Never>`。
2. 然后管道有一个 `map` 操作符，它在转换值及其类型。 在此示例中，它忽略了发布者发出的输入并返回了一个字符串。 这也将输出类型转换为 `<String>`，并将失败类型仍然保持为 `<Never>`。
3. 然后管道以 `sink` 订阅者结束。

当你去尝试理解管道时，你可以将其视为由输出和失败类型链接的一系列操作。 当你开始构建自己的管道时，这种模式就会派上用场。 创建管道时，你可以选择操作符来帮助你转换数据、类型或两者同时使用以实现最终目的。 最终目标可能是启用或禁用用户界面的某个元素，或者可能是得到某些数据用来显示。 许多 Combine 的操作符专门设计用来做这些转换。

有许多操作符是以 `try` 为前缀的，这表示它们返回一个 `<Error>` 的失败类型。 例如 [map](https://heckj.github.io/swiftui-notes/#reference-map) 和 [tryMap](https://heckj.github.io/swiftui-notes/#reference-trymap)。 `map` 操作符可以转换输出和失败类型的任意组合。 `tryMap` 接受任何输入和失败类型，并允许输出任何类型，但始终会输出 `<Error>` 的失败类型。

像 `map` 这样的操作符，你在定义返回的输出类型时，允许你通过基于你在提供给操作符的闭包中返回的内容推断输出类型。 在上面的例子中，`map` 操作符返回一个 `String` 的输出类型，因为这正是闭包返回的类型。

为了更具体地说明更改类型的示例，我们扩展了值在传输过程中的转换逻辑。此示例仍然以提供类型 `<Int, Never>` 的发布者开始，并以订阅类型为 `<String, Never>` 的值结束。

[SwiftUI-NotesTests/CombinePatternTests.swift](https://github.com/heckj/swiftui-notes/blob/master/SwiftUI-NotesTests/CombinePatternTests.swift)

```swift
let _ = Just(5) <1>
    .map { value -> String in <2>
        switch value {
        case _ where value < 1:
            return "none"
        case _ where value == 1:
            return "one"
        case _ where value == 2:
            return "couple"
        case _ where value == 3:
            return "few"
        case _ where value > 8:
            return "many"
        default:
            return "some"
        }
    }
    .sink { receivedValue in <3>
        print("The end result was \(receivedValue)")
    }
```

1. Just 是创建一个 `<Int, Never>` 类型组合的发布者，提供单个值然后完成。
2. 提供给 `.map()` 函数的闭包接受一个 `<Int>` 并将其转换为一个 `<String>`。由于 `<Never>` 的失败类型没有被改变，所以就直接输出了。
3. `sink` 作为订阅者，接受 `<String, Never>` 类型的组合数据。

> 当你在 Xcode 中创建管道，类型不匹配时，Xcode 中的错误消息可能包含一个有用的修复建议 *fixit*。 在某些情况下，例如上个例子，当提供给 `map` 的闭包中不指定特定的返回类型时，编译器就无法推断其返回值类型。 Xcode (11 beta 2 and beta 3) 显示此为错误消息： `Unable to infer complex closure return type; add explicit type to disambiguate`。 在上面示例中，我们用 `value → String in` 明确指定了返回的类型。

你可以将 Combine 的发布者、操作符和订阅者视为具有两种需要对齐的平行类型 —— 一种用于成功的有用值，另一种用于错误处理。 设计管道时经常会选择如何转换其中一种或两种类型以及与之相关的数据。

## 发布者和订阅者的生命周期

订阅者和发布者以明确定义的顺序进行通信，因此使得它们具有从开始到结束的生命周期：

![lifecycle-diagram](/assets/img/post/combine/lifecycle-diagram.svg)

_一个 Combine 管道的生命周期_

1. 当调用 `.subscribe(_: Subscriber)` 时，订阅者被连接到了发布者。
2. 发布者随后调用 `receive(subscription: Subscription)` 来确认该订阅。
3. 在订阅被确认后，订阅者请求值 *N*，此时调用 `request(_: Demand)`。
4. 发布者可能随后（当它有值时）发送 *N* 或者更少的值，通过调用 `receive(_: Input)`。 发布者不会发送**超过**需求量的值。
5. 订阅确认后的任何时间，订阅者都可能调用 `.cancel()` 来发送 [cancellation](https://developer.apple.com/documentation/combine/subscribers/completion)
6. 发布者可以选择性地发送 [completion](https://developer.apple.com/documentation/combine/subscribers/completion)：`receive(completion:)`。 完成可以是正常终止，也可以是通过 `.failure` 完成，可选地传递一个错误类型。 已取消的管道不会发送任何完成事件。

在上述图表中包含了一组堆积起来的弹珠图， 这是为了突出 Combine 的弹珠图在管道的整体生命周期中的重点。 通常，图表推断所有的连接配置都已完成并已发送了数据请求。 Combine 的弹珠图的核心是从请求数据到触发任何完成或取消之间的一系列事件。

## 发布者

发布者是数据的提供者。 当订阅者请求数据时， [publisher protocol](https://developer.apple.com/documentation/combine/publisher) 有严格的返回值类型约定，并有一系列明确的完成信号可能会终止它。

你可以从 [Just](https://heckj.github.io/swiftui-notes/#reference-just) 和 [Future](https://heckj.github.io/swiftui-notes/#reference-future) 开始使用发布者，它们分别作为单一数据源和异步函数来使用。

当订阅者发出请求时，许多发布者会立即提供数据。 在某些情况下，发布者可能有一个单独的机制，使其能够在订阅后返回数据。 这是由协议 [ConnectablePublisher](https://developer.apple.com/documentation/combine/connectablepublisher) 来约定实现的。 遵循 `ConnectablePublisher` 的发布者将有一个额外的机制，在订阅者发出请求后才启动数据流。 这可能是对发布者单独的调用 `.connect()` 来完成。 另一种可能是 `.autoconnect()`，一旦订阅者请求，它将立即启动数据流。

Combine 提供了一些额外的便捷的发布者：

| Convenience publishers                                       |                                                              |                                                              |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| [Just](https://heckj.github.io/swiftui-notes/#reference-just) | [Future](https://heckj.github.io/swiftui-notes/#reference-future) | [Deferred](https://heckj.github.io/swiftui-notes/#reference-deferred) |
| [Empty](https://heckj.github.io/swiftui-notes/#reference-empty) | [Sequence](https://heckj.github.io/swiftui-notes/#reference-sequence) | [Fail](https://heckj.github.io/swiftui-notes/#reference-fail) |
| [Record](https://heckj.github.io/swiftui-notes/#reference-record) | [Share](https://heckj.github.io/swiftui-notes/#reference-share) | [Multicast](https://heckj.github.io/swiftui-notes/#reference-multicast) |
| [ObservableObject](https://heckj.github.io/swiftui-notes/#reference-observableobject) | [@Published](https://heckj.github.io/swiftui-notes/#reference-published) |                                                              |

Combine 之外的一些苹果的 API 也提供发布者。

- [SwiftUI](https://heckj.github.io/swiftui-notes/#reference-swiftui) 使用 `@Published` 和 `@ObservedObject` 属性包装，由 Combine 提供，含蓄地创建了一个发布者，用来支持它的声明式 UI 的机制。
- Foundation
  - [URLSession.dataTaskPublisher](https://heckj.github.io/swiftui-notes/#reference-datataskpublisher)
  - [.publisher on KVO instance](https://heckj.github.io/swiftui-notes/#reference-kvo-publisher)
  - [NotificationCenter](https://heckj.github.io/swiftui-notes/#reference-notificationcenter)
  - [Timer](https://heckj.github.io/swiftui-notes/#reference-timer)
  - [Result](https://heckj.github.io/swiftui-notes/#reference-result)

## 操作符

操作符是苹果参考文档中包含的一些预构建功能的便捷名称。 操作符用来组合成管道。 许多操作符会接受开发人员的一个或多个闭包，以定义业务逻辑，同时保持并持有发布者/订阅者的生命周期。

一些操作符支持合并来自不同管道的输出、更改数据的时序或过滤所提供的数据。 操作符可能还会对操作类型有限制， 还可用于定义错误处理和重试逻辑、缓冲和预先载入以及支持调试。

|                       Mapping elements                       |                                                              |                                                              |
| :----------------------------------------------------------: | ------------------------------------------------------------ | ------------------------------------------------------------ |
| [scan](https://heckj.github.io/swiftui-notes/#reference-scan) | [tryScan](https://heckj.github.io/swiftui-notes/#reference-tryscan) | [setFailureType](https://heckj.github.io/swiftui-notes/#reference-setfailuretype) |
| [map](https://heckj.github.io/swiftui-notes/#reference-map) | [tryMap](https://heckj.github.io/swiftui-notes/#reference-trymap) | [flatMap](https://heckj.github.io/swiftui-notes/#reference-flatmap) |

|                      Filtering elements                      |                                                              |                                                              |
| :----------------------------------------------------------: | ------------------------------------------------------------ | ------------------------------------------------------------ |
| [compactMap](https://heckj.github.io/swiftui-notes/#reference-compactmap) | [tryCompactMap](https://heckj.github.io/swiftui-notes/#reference-trycompactmap) | [replaceEmpty](https://heckj.github.io/swiftui-notes/#reference-replaceempty) |
| [filter](https://heckj.github.io/swiftui-notes/#reference-filter) | [tryFilter](https://heckj.github.io/swiftui-notes/#reference-tryfilter) | [replaceError](https://heckj.github.io/swiftui-notes/#reference-replaceerror) |
| [removeDuplicates](https://heckj.github.io/swiftui-notes/#reference-removeduplicates) | [tryRemoveDuplicates](https://heckj.github.io/swiftui-notes/#reference-tryremoveduplicates) |                                                              |

|                      Reducing elements                       |                                                              |                                                              |
| :----------------------------------------------------------: | ------------------------------------------------------------ | ------------------------------------------------------------ |
| [collect](https://heckj.github.io/swiftui-notes/#reference-collect) | [reduce](https://heckj.github.io/swiftui-notes/#reference-reduce) | [tryReduce](https://heckj.github.io/swiftui-notes/#reference-tryreduce) |
| [ignoreOutput](https://heckj.github.io/swiftui-notes/#reference-ignoreoutput) |                                                              |                                                              |

|              Mathematic operations on elements               |                                                              |                                                              |
| :----------------------------------------------------------: | ------------------------------------------------------------ | ------------------------------------------------------------ |
| [max](https://heckj.github.io/swiftui-notes/#reference-max) | [tryMax](https://heckj.github.io/swiftui-notes/#reference-trymax) | [count](https://heckj.github.io/swiftui-notes/#reference-count) |
| [min](https://heckj.github.io/swiftui-notes/#reference-min) | [tryMin](https://heckj.github.io/swiftui-notes/#reference-min) |                                                              |

|            Applying matching criteria to elements            |                                                              |                                                              |
| :----------------------------------------------------------: | ------------------------------------------------------------ | ------------------------------------------------------------ |
| [allSatisfy](https://heckj.github.io/swiftui-notes/#reference-allsatisfy) | [tryAllSatisfy](https://heckj.github.io/swiftui-notes/#reference-tryallsatisfy) | [contains](https://heckj.github.io/swiftui-notes/#reference-contains) |
| [containsWhere](https://heckj.github.io/swiftui-notes/#reference-containswhere) | [tryContainsWhere](https://heckj.github.io/swiftui-notes/#reference-trycontainswhere) |                                                              |

|           Applying sequence operations to elements           |                                                              |                                                              |
| :----------------------------------------------------------: | ------------------------------------------------------------ | ------------------------------------------------------------ |
| [firstWhere](https://heckj.github.io/swiftui-notes/#reference-firstwhere) | [tryFirstWhere](https://heckj.github.io/swiftui-notes/#reference-tryfirstwhere) | [first](https://heckj.github.io/swiftui-notes/#reference-first) |
| [lastWhere](https://heckj.github.io/swiftui-notes/#reference-lastwhere) | [tryLastWhere](https://heckj.github.io/swiftui-notes/#reference-trylastwhere) | [last](https://heckj.github.io/swiftui-notes/#reference-last) |
| [dropWhile](https://heckj.github.io/swiftui-notes/#reference-dropwhile) | [tryDropWhile](https://heckj.github.io/swiftui-notes/#reference-trydropwhile) | [dropUntilOutput](https://heckj.github.io/swiftui-notes/#reference-dropuntiloutput) |
| [prepend](https://heckj.github.io/swiftui-notes/#reference-prepend) | [drop](https://heckj.github.io/swiftui-notes/#reference-drop) | [prefixUntilOutput](https://heckj.github.io/swiftui-notes/#reference-prefixuntiloutput) |
| [prefixWhile](https://heckj.github.io/swiftui-notes/#reference-prefixwhile) | [tryPrefixWhile](https://heckj.github.io/swiftui-notes/#reference-tryprefixwhile) | [output](https://heckj.github.io/swiftui-notes/#reference-output) |

|         Combining elements from multiple publishers          |                                                              |                                                              |
| :----------------------------------------------------------: | ------------------------------------------------------------ | ------------------------------------------------------------ |
| [combineLatest](https://heckj.github.io/swiftui-notes/#reference-combinelatest) | [merge](https://heckj.github.io/swiftui-notes/#reference-merge) | [zip](https://heckj.github.io/swiftui-notes/#reference-zip) |

|                       Handling errors                        |                                                              |                                                              |
| :----------------------------------------------------------: | ------------------------------------------------------------ | ------------------------------------------------------------ |
| [catch](https://heckj.github.io/swiftui-notes/#reference-catch) | [tryCatch](https://heckj.github.io/swiftui-notes/#reference-trycatch) | [assertNoFailure](https://heckj.github.io/swiftui-notes/#reference-assertnofailure) |
| [retry](https://heckj.github.io/swiftui-notes/#reference-retry) | [mapError](https://heckj.github.io/swiftui-notes/#reference-maperror) |                                                              |

|                   Adapting publisher types                   |                                                              |      |
| :----------------------------------------------------------: | ------------------------------------------------------------ | ---- |
| [switchToLatest](https://heckj.github.io/swiftui-notes/#reference-switchtolatest) | [eraseToAnyPublisher](https://heckj.github.io/swiftui-notes/#reference-erasetoanypublisher) |      |

|                      Controlling timing                      |                                                              |                                                              |
| :----------------------------------------------------------: | ------------------------------------------------------------ | ------------------------------------------------------------ |
| [debounce](https://heckj.github.io/swiftui-notes/#reference-debounce) | [delay](https://heckj.github.io/swiftui-notes/#reference-delay) | [measureInterval](https://heckj.github.io/swiftui-notes/#reference-measureinterval) |
| [throttle](https://heckj.github.io/swiftui-notes/#reference-throttle) | [timeout](https://heckj.github.io/swiftui-notes/#reference-timeout) |                                                              |

|                    Encoding and decoding                     |                                                              |      |
| :----------------------------------------------------------: | ------------------------------------------------------------ | ---- |
| [encode](https://heckj.github.io/swiftui-notes/#reference-encode) | [decode](https://heckj.github.io/swiftui-notes/#reference-decode) |      |

|              Working with multiple subscribers               |      |      |
| :----------------------------------------------------------: | ---- | ---- |
| [multicast](https://heckj.github.io/swiftui-notes/#reference-multicast) |      |      |

|                          Debugging                           |                                                              |                                                              |
| :----------------------------------------------------------: | ------------------------------------------------------------ | ------------------------------------------------------------ |
| [breakpoint](https://heckj.github.io/swiftui-notes/#reference-breakpoint) | [handleEvents](https://heckj.github.io/swiftui-notes/#reference-handleevents) | [print](https://heckj.github.io/swiftui-notes/#reference-print) |

## Subjects

Subjects 是一种遵循 [`Subject`](https://developer.apple.com/documentation/combine/subject) 协议的特殊的发布者。 这个协议要求 subjects 有一个 `.send(_:)` 方法，来允许开发者发送特定的值给订阅者或管道。

Subjects 可以通过调用 `.send(_:)` 方法来将值“注入”到流中， 这对于将现有的命令式的代码与 Combine 集成非常有用。

一个 subject 还可以向多个订阅者广播消息。 如果多个订阅者连接到一个 subject，它将在调用 `send(_:)` 时向多个订阅者发送值。 一个 subject 还经常用于连接或串联多个管道，特别是同时给多个管道发送值时。

Subject 不会盲目地传递其订阅者的需求。 相反，它为需求提供了一个聚合点。 在没有收到订阅消息之前，一个 subject 不会向其连接的发布者发出需求信号。 当它收到订阅者的需求时，它会向它连接的发布者发出 `unlimited` 需求信号。 虽然 subject 支持多个订阅者，但任何未请求数据的订阅者，在请求之前均不会给它们提供数据。

Combine 中有两种内建的 subject : [CurrentValueSubject](https://heckj.github.io/swiftui-notes/#reference-currentvaluesubject) 和 [PassthroughSubject](https://heckj.github.io/swiftui-notes/#reference-passthroughsubject)。 它们的行为类似，但不同的是 `CurrentValueSubject` 需要一个初始值并记住它当前的值，`PassthroughSubject` 则不会。 当调用 `.send()` 时，两者都将向它们的订阅者提供更新的值。

在给遵循 [`ObservableObject`](https://developer.apple.com/documentation/combine/observableobject) 协议的对象创建发布者时，`CurrentValueSubject` 和 `PassthroughSubject` 也很有用。 SwiftUI 中的多个声明式组件都遵循这个协议。

## 订阅者

虽然 [`Subscriber`](https://developer.apple.com/documentation/combine/subscriber) 是用于接收整个管道数据的协议，但通常 *the subscriber* 指的是管道的末端。

Combine 中有两个内建的订阅者： [Assign](https://heckj.github.io/swiftui-notes/#reference-assign) 和 [Sink](https://heckj.github.io/swiftui-notes/#reference-sink)。 SwiftUI 中有一个订阅者： [onReceive](https://heckj.github.io/swiftui-notes/#reference-onreceive)。

订阅者支持取消操作，取消时将终止订阅关系以及所有流完成之前，由发布者发送的数据。 `Assign` 和 `Sink` 都遵循 [Cancellable 协议](https://developer.apple.com/documentation/combine/cancellable).

当你存储和自己订阅者的引用以便稍后清理时，你通常希望引用销毁时能自己取消订阅。 [AnyCancellable](https://heckj.github.io/swiftui-notes/#reference-anycancellable) 提供类型擦除的引用，可以将任何订阅者转换为 `AnyCancellable` 类型，允许在该引用上使用 `.cancel()`，但无法访问订阅者本身（对于实例来说可以，但是需要更多数据）。 存储对订阅者的引用非常重要，因为当引用被释放销毁时，它将隐含地取消其操作。

[`Assign`](https://developer.apple.com/documentation/combine/subscribers/assign) 将从发布者传下来的值应用到由 keypath 定义的对象， keypath 在创建管道时被设置。 一个在 Swift 中这样的例子：

```swift
.assign(to: \.isEnabled, on: signupButton)
```

[`Sink`](https://developer.apple.com/documentation/combine/subscribers/sink) 接受一个闭包，该闭包接收从发布者发送的任何结果值。 这允许开发人员使用自己的代码终止管道。 此订阅者在编写单元测试以验证发布者或管道时也非常有帮助。 一个在 Swift 中这样的例子：

```swift
.sink { receivedValue in
    print("The end result was \(String(describing: receivedValue))")
}
```

其他订阅者是其他 Apple 框架的一部分。 例如，SwiftUI 中的几乎每个 `control` 都可以充当订阅者。 SwiftUI 中的 [View 协议](https://developer.apple.com/documentation/swiftui/view/) 定义了一个 `.onReceive(publisher)` 函数，可以把视图当作订阅者使用。 `onReceive` 函数接受一个类似于 `sink` 接受的闭包，可以操纵 SwiftUI 中的 `@State` 或 `@Bindings`。

一个在 SwiftUI 中这样的例子：

```swift
struct MyView : View {

    @State private var currentStatusValue = "ok"
    var body: some View {
        Text("Current status: \(currentStatusValue)")
            .onReceive(MyPublisher.currentStatusPublisher) { newStatus in
                self.currentStatusValue = newStatus
            }
    }
}
```

对于任何类型的 UI 对象 (UIKit、AppKit 或者 SwiftUI)， [Assign](https://heckj.github.io/swiftui-notes/#reference-assign) 可以在管道中使用来更新其属性。

## 参考

- [Combine: Asynchronous Programming with Swift](https://www.raywenderlich.com/books/combine-asynchronous-programming-with-swift/v2.0)

- [Using Combine](https://heckj.github.io/swiftui-notes/)

