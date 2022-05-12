---
title: 使用 Page Object Model 来写 UI Test
date: 2022-05-12 20:00:00 +0800
categories: [iOS]
tags: [test]     # TAG names should always be lowercase
image:
  src: /assets/img/post/test/header.png
---

随着 SwiftUI 的使用，使得现在写 UI Test 也更为容易了一些，一般的视图都能够被分离出来，单独拿出来进行测试。如果再加上 Page Object Model，那么将能够给写 UI Test 锦上添花。下面就来一探究竟吧。

本文的示例代码来自于 [UI Testing using Page Object pattern in Swift](https://swiftwithmajid.com/2021/03/24/ui-testing-using-page-object-pattern-in-swift/)

例子使用一个简单的登录界面，大家可以把它简单地想象成这样子：

![](/assets/img/post/test/login.png){: w="200"}

## 基本的测试写法

我们希望测试这个界面的界面元素和登录交互，代码可以长这个样子：

```swift
final class LoginTests: XCTestCase {
    var app: XCUIApplication!

    override func setUp() {
        continueAfterFailure = false
        app = XCUIApplication()
        app.launchArguments = ["testing"]
        app.launch()
    }

    func testLoginFlow() {
        let email = app.textFields["email"]
        email.tap()
        email.typeText("cmecid@gmail.com")

        let pwd = app.secureTextFields["password"]
        pwd.tap()
        pwd.typeText("pwd")

        app.buttons["login"].tap()

        let message = app.staticTexts["Hello World!"]
        XCTAssertTrue(message.waitForExistence(timeout: 5))
    }
}
```

在测试 case 执行前，获取到 app 实例，然后启动 app。在测试方法中，通过查找界面元素，获取到输入 email 和 password 的 `TextField`，并且点击他们输入测试文本，然后找到 login 的 `Button` 点击它，期望是在按钮点击之后，切换到一个新的界面，显示 Hello World! 的文本。这是一个基本的登录操作的流程。

上面的测试代码，是有可优化的点的。比如 `setUp` 方法中的 app 环境，我们几乎在每个 UI 测试中都会用到它，因此它是可以被抽取的；还有假使这个登录流程需要作为某一个 UI 测试的前置操作，会被复用到，那么势必会出现重复代码。下面就通过两次优化，来引出 Page Object Model。

## 抽取 UI Test 的基类

通过把 `setUp` 和 `tearDown` 的代码抽取出来，让子类继承，来简化每次测试前后的代码书写。

```swift
class UITestCase: XCTestCase {
    var app: XCUIApplication!

    override func setUp() {
        continueAfterFailure = false
        app = XCUIApplication()
        app.launchArguments = ["testing"]
        app.launch()
    }

    override func tearDown() {
        let screenshot = XCUIScreen.main.screenshot()
        let attachment = XCTAttachment(screenshot: screenshot)
        attachment.lifetime = .deleteOnSuccess
        add(attachment)
        app.terminate()
    }
}
```

在 `setUp` 阶段启动 app，在 `tearDown` 阶段，如果有测试失败的话，保存失败场景的截图，便于 Debug。

通过继承 `UITestCase` ，`LoginTests` 可以简化为：

```swift
final class LoginTests: UITestCase {
    func testLoginFlow() {
        let email = app.textFields["email"]
        email.tap()
        email.typeText("cmecid@gmail.com")

        let pwd = app.secureTextFields["password"]
        pwd.tap()
        pwd.typeText("pwd")

        app.switches["rememberMe"].tap()
        app.buttons["login"].doubleTap()
        app.buttons["login"].twoFingerTap()

        let message = app.staticTexts["Hello World!"]
        XCTAssertTrue(message.waitForExistence(timeout: 5))
    }
}
```

## 使用 Page Object Model

Page Object 是按照界面来进行封装，封装的类只提供当前界面的一些元素和交互方法，而测试用例来复杂消费他们，然后构建整个交互流程。

```swift
protocol Screen {
    var app: XCUIApplication { get }
}

struct LoginScreen: Screen {
    let app: XCUIApplication

    private enum Identifiers {
        static let email = "email"
        static let password = "password"
        static let login = "login"
        static let error = "error"
    }

    func typeEmail(_ email1: String) -> Self {
        let email = app.textFields[Identifiers.email]
        email.tap()
        email.typeText(email1)
        return self
    }

    func tapLoginExpectingError() -> Self {
        app.buttons[Identifiers.login].tap()
        let error = app.staticTexts[Identifiers.error]
        XCTAssertTrue(error.waitForExistence(timeout: 5))
        return self
    }

    func tapLogin() -> MessageScreen {
        app.buttons[Identifiers.login].tap()
        return MessageScreen(app: app)
    }

    func typePassword(_ password: String) -> Self {
        let pwd = app.secureTextFields[Identifiers.password]
        pwd.tap()
        pwd.typeText(password)
        return self
    }
}
```

`LoginScreen` 这个结构体，只包含登录界面的元素和一些交互方法，它把每个交互都单独定义了方法，这样不论测试用例想如何进行验证，都是比较灵活的。然后我们还有一个登录成功之后的 `MessageScreen`。

```swift
struct MessageScreen: Screen {
    let app: XCUIApplication

    func verifyMessage(_ message: String) -> Self {
        let message = app.staticTexts[message]
        XCTAssertTrue(message.waitForExistence(timeout: 5))
        return self
    }
}
```

以上所有的方法，都返回了自己或者其他的 `Screen`，这样方便我们进行链式调用来组合一些元素和交互，例如 `tapLogin` 方法返回了 `MessageScreen`，这也是符合交互逻辑的。正常情况下，点击完登录应该跳转到 Message 界面。

最终我们的 UI Test 就变成了这样：

```swift
final class LoginTests: UITestCase {
    func testLoginFlow() {
        LoginScreen(app: app)
            .typeEmail("Cmecid@gmail.com")
            .typePassword("password")
            .tapLoginExpectingError()
            .typeEmail("Cmecid@gmail.com")
            .typePassword("pwd")
            .tapLogin()
            .verifyMessage("Hello World!")
    }
}
```

这样的调用逻辑是非常清晰的，并且各个页面都能被复用，每个页面里面的具体细节都被隐藏了起来。

例子中使用的是 Screen 而非 Page，这是因为 Page Object 这个概念是从 Web 来的，所以示例的作者使用了 Screen 代替了 Page，其实在项目实践中，并不一定每一个类都要代表一个 Screen，它可以是对一个 UI Component 的测试，因为我们往往会把一些可复用组件进行拆分，所以对小组件单独进行测试也是不错的实践。

## 是否在 Page 中进行断言

大家会发现，在文中的示例其实是在 Page 的方法中加入了断言语句，而在测试用例中只是调用了相应的方法。还有一种观点是 Page 应该只负责提供界面元素和交互方法，而不加断言，让测试方法自己去决定想要怎么断言。[Martin Fowler](https://martinfowler.com/) 在他的文章中指出了这两种方法的优缺点：

- 在 Page 中写断言，优点是可以避免重复的断言语句，提供更符合上下文的错误信息，以及更符合 [TellDontAsk](https://martinfowler.com/bliki/TellDontAsk.html) 风格的 API。
- 而缺点则是，使得 Page 的职责不再单一，混杂了断言语句的逻辑，会使得 Page 变得更大更复杂。

因为示例的场景比较简单，因此看起来在 Page 中写断言还是挺清晰的，但是随着页面和逻辑变得更复杂，它的弊端会不会展现出来呢？

Martin Fowler 个人更偏向于不在 Page 中写断言，大家可以根据自己的想法和项目情况，来进行选择。目前个人觉得在页面不复杂的情况下，在 Page 中写断言是更清晰的。

## 参考

- [PageObject](https://martinfowler.com/bliki/PageObject.html)
- [UI Testing using Page Object pattern in Swift](https://swiftwithmajid.com/2021/03/24/ui-testing-using-page-object-pattern-in-swift/)