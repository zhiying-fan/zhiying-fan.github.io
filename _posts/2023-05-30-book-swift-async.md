---
title: 读《Swift 异步与并发编程》
date: 2023-08-12 20:00:00 +0800
categories: [读书]
tags: [books]     # TAG names should always be lowercase
image:
  src: /assets/img/post/swift-async/header.png
---

Swift 在 5.5 版本中加入了 async/await 语法来简化异步编程，他们的基础用法不难，但是如果要深入理解并且结合 Actor 来更好的使用，还是需要对其深入学习。这本王巍写的[《Swift 异步与并发编程》](https://objccn.io/products/async-swift)，在[《Swift 进阶》](https://objccn.io/products/advanced-swift/)书籍的基础上，对异步编程进行了更详细的讲解，非常值得一读。

## 对一些基本概念的图文澄清

### 同步

同步操作意味着在操作完成之前，运行这个操作的线程都将被占用，直到函数最终被抛出或者返回。在函数返回之前，运行它的线程无法执行其他操作。

![](/assets/img/post/swift-async/sync.png)

### 异步

我们将运行在后台线程加载数据的行为称为异步操作，区别于同步，它不会阻塞运行这个函数的线程，带来的好处就是不会阻塞 UI 的更新。

![](/assets/img/post/swift-async/async.png)

### 串行

串行严格按照函数被调用的先后顺序发生，同步和异步都可能会有串行的运行方式。

![同步串行](/assets/img/post/swift-async/serial-sync.png)
_同步串行_

![异步串行](/assets/img/post/swift-async/serial-async.png)
_异步串行_

### 并行

可以在不同的线程中同时执行多个函数的方式，我们称之为并行。

![](/assets/img/post/swift-async/parallel.png)

### Swift 并发

Swift 并发指的是异步和并行代码的组合，这个定义去掉了在同一个线程中多个操作交替运行的并发方式，更易于我们理解。

### 异步函数 Async

`func loadSignature() async -> String`

异步函数的引入，在语法上确保了其使用的正确性。

`async` 会帮助编译器确保两件事情：

1. 它允许我们在函数体内部使用 await 关键字；
2. 它要求其他人在调用这个函数时，使用 await 关键字。

`await` 代表了函数在此处可能会放弃当前线程，它是程序的潜在暂停点。

### 结构化并发

对于同步函数来说，线程决定了它的执行环境。而对于异步函数，线程的概念被弱化，异步函数的执行环境交由任务 (Task) 决定。

使用 Task.init 可以创建一个执行异步函数的上下文环境。

```swift
func someSyncMethod() {
  Task {
    await loadFromDatabase()
    await loadSignature()
    print("Done")
  }
}
```

上面函数中的 `loadFromDatabase` 和 `loadSignature` 其实是被串行执行的。如果 `loadSignature` 不依赖上一个函数的运算结果，我们当然希望他们可以并行，来加快获取结果的速度。所用到的方法就是结构化并发，有两种方式可以实现，`async let` 和 `task group`。

`async let` 的使用例子如下

```swift
func someSyncMethod() {
  Task {
    async let loadStrings = loadFromDatabase()
    async let loadSignature = loadSignature()

    await loadStrings
    await loadSignature
    print("Done")
  }
}
```

### Actor

Actor 是用来提供封装良好的数据隔离，确保并发代码的安全的。更加详细的介绍在后面的章节会有。

## 异步函数

### 使用 Async 对已有的闭包进行封装

对于项目中已有的大量闭包函数，我们可能不能很快的将它们都重写为异步函数，那么一个切实有效的迁移方法就是先对闭包进行封装，让它得以在其他异步函数中使用。
封装方法如下：

```swift
func withUnsafeContinuation<T>(
_ fn: (UnsafeContinuation<T, Never>) -> Void
) async -> T

func withUnsafeThrowingContinuation<T>(
_ fn: (UnsafeContinuation<T, Error>) -> Void
) async throws -> T
```

比如我们要对一个闭包函数 `func load(completion: @escaping ([String]?, Error?) -> Void)` 进行封装：

```swift
func load() async throws -> [String] {
  try await withUnsafeThrowingContinuation { continuation in
    load { values, error in
      if let error {
        continuation.resume(throwing: error)
      } else if let values {
        continuation.resume(returning: values)
      } else {
        assertFailure("Both parameters are nil")
      }
    }
  }
}
```

需要确保 `resume` 方法被调用且只被调用了一次。

### 运行环境

和可抛出错误的函数一样，异步函数也具有“传染性”：由于运行一个异步函数可能会带来潜在的暂停点，因此它必须要用 await 明确标记。而 await 又只能在 async 标记的异步函数中使用。于是，将一个函数转换为异步函数时，往往也意味着它的所有调用者也需要变成异步函数。

在最上层提供运行环境的是 `Task`，它有几种不同的形式，常用的两个 API 如下：

1. 在同步函数中使用 Task 创建异步环境

```swift
func syncMethod() throws {
  Task {
    try await asyncMethod()
  }
}
```

2. 在 SwiftUI 的 View 上调用 `.task` modifier，它的调用时机和 `onAppear` 相同，并且受 View 生命周期的影响，会在 View 的生命周期结束时，取消对应的 Task。

```swift
var body: some View {
  ProgressView()
    .task {
      try? await load()
    }
}
```

## 结构化并发

在介绍结构化并发之前，我们先来看看什么是结构化编程，即代码拥有单一的入口和出口，我们可以期望代码会在出口处将控制权交还给我们。

![](/assets/img/post/swift-async/structured.png)

相对于结构化编程，最典型的是 goto 语句，它是非结构化的，它可以让代码无条件的跳转到某个标签，在多个跳转之后，我们可能就会失去对代码的控制了。

结构化并发即是在进行并发操作时，也保证了控制流路径的单一入口和单一出口，得益于异步函数，我们才能更好的实现结构化并发。下面是使用回调导致的非结构化并发 与 使用异步函数的结构化并发的对比：

![](/assets/img/post/swift-async/structured-concurrency.png)

作为对比，先看一个非结构化并发的例子：

```swift
Task {
  let t1 = Task {
    print("t1: \(Task.isCancelled)")
  }

  let t2 = Task {
    print("t2: \(Task.isCancelled)")
  }

  t1.cancel()
  print("t: \(Task.isCancelled)")
}

// 输出：
// t: false
// t1: true
// t2: false
```

上例中 t1 和 t2 的执行都是在外层 Task 闭包结束后才进行的，它们逃逸出去了。

创建结构化并发有两种方式：`withTaskGroup` 和 `async let`

### Task Group

在某个异步函数中，我们可以通过 withTaskGroup 为当前的任务添加一组结构化的并发子任务：

```swift
struct TaskGroupSample {
  func start() async {
    print("Start")
    // 1
    await withTaskGroup(of: Int.self) { group in
      for i in 0 ..< 3 {
        // 2
        group.addTask {
          await work(i)
        }
      }
      print("Task added")

      // 4
      for await result in group {
        print("Get result: \(result)")
      }

      // 5
      print("Task ended")
    }
    print("End")
  }

  private func work(_ value: Int) async -> Int {
    // 3
    print("Start work \(value)")
    try? await Task.sleep(nanoseconds: UInt64(value) * NSEC_PER_SEC)
    print("Work \(value) done")
    return value
  }
}

Task {
  await TaskGroupSample().start()
}

// 输出：
// Start
// Task added
// Start work 0
// Start work 1
// Start work 2
// Work 0 done
// Get result: 0
// Work 1 done
// Get result: 1
// Work 2 done
// Get result: 2
// Task ended
// End
```

由 work 定义的三个异步操作并发执行，它们各自运行在独自的子任务空间中。这些子任务在被添加后即刻开始执行，并最终在离开 group 作用域时再汇集到一起。

### Async let

这是一种简化版的创建结构化并发的方式，不需要像 withTaskGroup 一样的模板代码。

上面的例子使用 `async let` 可以改写为：

```swift
func start() async {
  print("Start")
  async let v0 = work(0)
  async let v1 = work(1)
  async let v2 = work(2)
  print("Task added")

  let result = await v0 + v1 + v2
  print("Task ended")
  print("End. Result: \(result)")
}
```

async let 赋值后，子任务会立即开始执行，即在 await 调用之前，任务已经开始了。

### Task Group 与 Async let 对比

既然 Async let 更加简洁，那是不是可以完全不使用 withTaskGroup 了呢？大多数情况是的。
withTaskGroup 有它的独特之处，它可以动态的创建任务，例如上面的例子，我们使用了 for 循环动态地添加了 3 个任务到任务组。它的 API 看起来也更加的严肃，使得并发的结构会更加清晰。

## 协作式任务取消

我们知道对于结构化并发可以通过调用 `task.cancel()` 来取消任务，但是要注意的是任务本身在取消后并不会自动停止，需要我们在任务的实现时通过对 `isCancelled` 的检查，来自己处理任务取消之后的响应。

`task.cancel()` 做的事情只有两件：

- 将自身任务的 isCancelled 标识置为 true。
- 在结构化并发中，如果该任务有子任务，那么取消子任务。

### 自己处理任务取消

既然任务自己不会取消，那我们就得负责它的取消行为。可以在任务开始前检查 isCancelled 并返回空值或者抛出错误，来正确的响应对任务的调用。

比如在发现任务被取消之后抛出错误：

```swift
func work() async throws -> String {
  var s = ""
  for c in "Hello" {
    // 检查取消状态
    guard !Task.isCancelled else {
      throw CancellationError()
    }
    // ...
  }
}
```

或者使用简化版的 API，这是推荐的做法：

```swift
func work() async throws -> String {
  var s = ""
  for c in "Hello" {
    // 检查取消状态
    try Task.checkCancellation()
    // ...
  }
}
```

### 内建 API 的取消

一些标准库中的 API 已经做了对任务取消的支持，比如

```
extension Task where Success == Never, Failure == Never {
  public static func sleep(nanoseconds duration: UInt64) async throws
}
```

它们通过抛出错误 `CancellationError` 来告知调用者任务已被取消。

## Actor 模型和数据隔离

Actor 模型可以用来保证线程安全，避免数据竞争。在多线程并发的操作中，为了保证同一块内存不会被多个线程同时访问，我们可以使用线程锁，但是锁的设计较为复杂，且容易出错。
书中举了一个进入房间写正字的例子，通过对比锁和 Actor 模型，我们还可以很明显的发现 Actor 对性能的提升，调用者不需要花更多的时间去等待：

![](/assets/img/post/swift-async/lock.png)
_线程锁_

![](/assets/img/post/swift-async/actor.png)
_Actor_

### actor 的创建和访问

`actor` 和 `class` 在结构上是非常相似的，下面是一个 actor 的例子：

```swift
actor Room {
  let roomNumber = "101"
  var visitorCount: Int = 0

  init() {}

  func visit() -> Int {
    visitorCount += 1
    return visitorCount
  }
}
```

可以看到，在 actor 内部访问和修改属性和类是一样的，主要的不同是在外部的调用。例如如下的外部访问，都是不被允许的

```swift
let room = Room()
room.visitorCount += 1

room.visit()
```

因为对于 room 这个实例来说，是被 actor 隔离的，外部对它的访问属于跨 actor 访问，只能在异步方法中才可以进行。使用 await 表明这里是一个潜在的暂停点，消息被转发给了 actor 之后，需要等到 actor 可用时才会执行，所以这个异步线程可能需要等待，在此期间它可以去执行别的任务。

```swift
let room = Room()
let visitCount = await room.visit()
print(visitCount)
print(await room.visitorCount)
```

### isolated 和 nonisolated

- 在 actor 中的声明，默认处于 actor 的隔离域中。
- 在 actor 外的声明，默认处于 actor 隔离域外。

使用关键字 isolated 和 nonisolated，可以做到：

- 对于 actor 之外的声明，使用 isolated 关键字来明确表示函数体应该运行在该 actor 的隔离域中。
- 在 actor 中的声明，如果加上 nonisolated，那么它将被置于隔离域外。

例如被 `isolated` 标记的方法，会将方法的实现运行在参数 room 的 actor 中。

```
func reportRoom(room: isolated Room) {
  print("Room: \(room.visitorCount)")
}
```

下面的例子是在一个 actor 中调用该方法：

```swift
actor Room {
  func doSometing() async {
  // `self` 在自身的隔离域中
  reportRoom(room: self)

  let anotherRoom = Room()
  // anotherRoom 不在 `self` 隔离域中。需要切换隔离域。
  await reportRoom(room: anotherRoom)
  }
}
```

由于 actor 隔离域的不同，会发生 actor hopping 的行为。方法会继承上下文的 actor 隔离域，所以第一个 reportRoom 的方法中传入的 self 是在同隔离域的，不需要 await 进行跨 actor 执行。第二个 reportRoom 的方法，由于 anotherRoom 需要的隔离域不同，所以需要使用 await 跨 actor 执行。在方法执行完之后，会重新跳回 self 的隔离域，保持同样的上下文环境。
隔离域是根据实例划分的，每一个实例都是一个单独的隔离域。

### MainActor

MainActor 是全局 actor 之一，可以理解为主线程，主要用来隔离 UI 相关的代码。@MainActor 的属性包装，可以做来标记类、属性、方法或者全局属性。对于 MainActor 的切换，同样可以使用 await 来进行跳转，或者使用 @MainActor 来标记 Task，来将整个 Task 闭包切换到 MainActor。

```swift
Task { @MainActor in
  ...
}
```

## 并发线程模型

在使用异步函数时，我们没见到过自己去调度线程的情况，取而代之的是使用 Task 和 Actor。那么异步函数的线程调度是怎么完成的呢？
异步函数与一直在同一线程上运行的同步函数不同，异步函数可能会被 await 分割，并由多个不同的线程协同运行。Swift 并发在底层采用了一种新的调度方式，称为协同式线程池：它使用一个串行队列来调度工作，将函数中剩余的执行内容抽象为轻量级的续体，然后进行调度。实际的工作运行则由一个全局的并行队列来处理，具体的任务可能会被分配到最多与 CPU 核心数量相等的线程中去。
书中使用的异步线程模型图示特别的形象，这里直接搬过来：

> Excerpt From
Swift 异步与并发编程
王 巍
https://itunes.apple.com/WebObjects/MZStore.woa/wa/viewBook?id=0
This material may be protected by copyright.

假设我们有下面的代码：

```swift
func bar1() {}
func bar2() async {}
func bar3() async {
  await baz()
}
func baz() async {
  bar1()
}

func foo() async {
  bar1()
  await bar2()
  await bar3()
}

func method() {
  Task {
    await foo()
  }
}
```

1. 当某个线程执行 method 时，Task.init 首先被入栈，它是一个普通的初始化方法，在执行完毕后立即出栈，method 函数随之结束。通过 Task.init 参数闭包传入的 await foo()，被提交给协同式线程池，如果协同式调度队列正在执行其他工作，那么它被存放在堆上，等待空闲线程：

![](/assets/img/post/swift-async/cooperative-1.png)

2. 当有适合的线程可以运行协同式调度队列中的工作时，执行器读取 foo 并将它推入这个线程的栈上，开始执行。需要注意的是，这里的“适合线程”和 method 所在的线程并不需要一致，它可能是另外的空闲线程 (因此我们这里使用的颜色和上面的栈有所不同)：

![](/assets/img/post/swift-async/cooperative-2.png)

3. foo 中的第一个调用是一个同步函数 bar1。在异步函数中调用同步函数并没有什么不同，bar1 将被作为新的 frame 被推入栈中执行：

![](/assets/img/post/swift-async/cooperative-3.png)

4. 当 bar1 执行完毕后，它对应的 frame 被出栈，控制权回到 foo，准备执行其中的第二个调用 await bar2()：

![](/assets/img/post/swift-async/cooperative-4.png)

5. 接下来我们会在这个线程中执行到 await bar2()，它是一个异步函数调用。为了不阻塞当前线程，异步函数 foo 可能会在此处暂停并放弃线程。当前的执行环境 (如 bar2 和 foo 的关系) 会被记录到堆中，以便之后它在调度栈上继续运行。此时，执行器有机会到堆中寻找下一个需要执行的工作。在这里，我们假设它找到的就是 bar2。它将被装载到栈上，替换掉当前的栈空间，当前线程就可以继续执行，而不至于阻塞了：

![](/assets/img/post/swift-async/cooperative-5.png)

当然，执行器也有可能寻找到其他的工作 (比如最近有优先级更高的任务被加入)，这种情况下 bar2 就将被挂起一段时间，直到调度栈有机会再次寻找下一个工作。不过不论如何，串行调度队列都不会停止工作。它要么会去执行 bar2，要么会去执行其他找到的工作，唯独不会傻傻等待。

6. 当 bar2 执行完毕后，它被从堆上移除。因为在执行 bar2 前，我们在堆上保持了 foo 和 bar2 的关系，因此在 await     bar2() 结束后，执行器可以从堆中装载 foo，并发现接下来需要运行的指令。在我们的例子中，await bar3() 将被运行。和 bar2 时的情况类似，底层调度使用 bar3 替换掉栈的内容，并继续在堆上维护返回时要继续执行的续体：

![](/assets/img/post/swift-async/cooperative-6.png)

需要注意，await bar2() 前后代码可能会运行在不同线程上，除非指定了 MainActor，否则协作式调度队列并不会对具体运行的线程作出保证。

7. bar3 中的第一个调用是 await baz()。这是一个在异步函数中调用其他的异步函数的情况，实质上它的情况和 foo 中调用 await bar2() 或 await bar3() 是相同的。baz 会替换调度队列所对应的线程的栈：

![](/assets/img/post/swift-async/cooperative-7.png)

8. 在这个栈中，同步方法 bar1 的调用依然在当前栈上进行普通的入栈和出栈：

![](/assets/img/post/swift-async/cooperative-8-1.png)
![](/assets/img/post/swift-async/cooperative-8-2.png)

在异步函数定义的栈上调用同步函数，所执行的是普通的出入栈操作。因此在 Swift 异步函数中，我们是可以透明地调用 C 或者 Objective-C 的同步函数的。在底层，这种调用就是普通的同步调用。

9. 当 baz 完成后，执行器从堆中找到接下来的续体部分，也就是 bar3，并将它替换到某个线程的栈中。虽然已经多次说明，但笔者依然想再强调一次，此时 bar3 的执行线程可能会和 baz 不同，也可能和 bar3 最早的执行线程不同，(虽然大部分情况下是一致的，但这是一个实现细节) 我们不应该对具体的执行线程进行任何假设：

![](/assets/img/post/swift-async/cooperative-9.png)

10. 最后，bar3 的执行也结束了，执行器最终寻找到一开始的 foo，并最终完成整个 Task 的执行：

![](/assets/img/post/swift-async/cooperative-10.png)

这节内容是本书的最后一章，这里也是本文的最后了。通过阅读本书，对 Swift 异步函数的使用，以及并发编程有了更加深入的认识，也意识到了一些能对已有代码进行重构的点，还是收获颇丰的！
