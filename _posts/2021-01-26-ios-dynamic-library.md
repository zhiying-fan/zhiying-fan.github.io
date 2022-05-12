---
title: iOS 动态库 Dynamic library
date: 2021-01-26 20:00:00 +0800
categories: [iOS]
tags: [library]     # TAG names should always be lowercase
image:
  src: /assets/img/post/library/header.png
---

动态库可以在 app launch time 或者 runtime 被加载，也就是说代码可以只在需要的时候被加载，这就减少了可执行文件的大小和 App 启动时的内存消耗。

App 的功能本质是通过可执行代码来实现的，在使用静态库时，代码是被全部复制到了生成的可执行代码中去，并且在 App 启动的时候，一次性全部加载到内存中去，因此会使得文件变大，也更消耗内存。下图展示了静态库是如何被链接到 App 的：

![static libraries](/assets/img/post/library/app-using-static-libraries.png)

相比之下，动态库只加载了对其的引用而非库本身：

![dynamic libraries](/assets/img/post/library/app-using-dynamic-libraries.png)

| 区别                                                         |                             |
| ------------------------------------------------------------ | --------------------------- |
| 动态库可以在运行时再加载                                     | 静态库不可以                |
| 动态库更新时 App 不必升级也能使用最新的功能                  | 使用静态库的话 App 必须升级 |
| 动态库可以在加载时初始化，在 App 终止前做一些 clean-up 的工作 | 静态库不可以                |
| 动态库在更新时必须保证向下兼容性                             | 静态库不需要                |

## 动态库是怎么被加载的

在 App 启动时，系统会把代码和数据加载到一个新进程的地址空间中，同时也会把动态加载器 ( `/usr/lib/dyld` ) 加载到进程中。App 记录每个动态库的文件名，动态加载器会在需要的时候根据文件名加载动态库。

动态库会被分成必备的和可选的，如果必备的动态库在初始化的时候加载失败了，dyld 将会杀死进程，终止程序的运行；如果可选库加载失败了，程序会继续运行，只是不会执行到相应库的代码。其实大家在开发中进行 iOS 版本判断，进而选择性的使用某些新的框架（比如 Combine, SwiftUI）时，就是加载了可选库。

下面用例子详细看一下：

新建一个工程，添加一个必备库和一个可选库：

![](/assets/img/post/library/link-libraries.png)

编译一下工程，然后在 Products 下面找到编译好的 .app 包，右键选择 Show Package Contents，找到可执行文件。

![](/assets/img/post/library/show-contents.png)

下一步我们使用 `otool ` 来查看包引用的动态库：

```zsh
➜  ~ otool -L /Users/**/Debug-iphonesimulator/Demo-Framework.app/Demo-Framework
/Users/**/Debug-iphonesimulator/Demo-Framework.app/Demo-Framework:
	/System/Library/Frameworks/CoreLocation.framework/CoreLocation (compatibility version 1.0.0, current version 2420.12.14)
	/System/Library/Frameworks/CallKit.framework/CallKit (compatibility version 1.0.0, current version 1.0.0, weak)
	/System/Library/Frameworks/Foundation.framework/Foundation (compatibility version 300.0.0, current version 1770.255.0)
	/usr/lib/libobjc.A.dylib (compatibility version 1.0.0, current version 228.0.0)
	/usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 1292.60.1)
	/System/Library/Frameworks/UIKit.framework/UIKit (compatibility version 1.0.0, current version 4003.1.101)
	/usr/lib/swift/libswiftCore.dylib (compatibility version 1.0.0, current version 1200.2.41)
	/usr/lib/swift/libswiftCoreFoundation.dylib (compatibility version 1.0.0, current version 1.6.0, weak)
	/usr/lib/swift/libswiftCoreGraphics.dylib (compatibility version 1.0.0, current version 2.0.0, weak)
	/usr/lib/swift/libswiftCoreImage.dylib (compatibility version 1.0.0, current version 1.0.0, weak)
	/usr/lib/swift/libswiftCoreLocation.dylib (compatibility version 1.0.0, current version 5.0.0, weak)
	/usr/lib/swift/libswiftDarwin.dylib (compatibility version 1.0.0, current version 0.0.0, weak)
	/usr/lib/swift/libswiftDispatch.dylib (compatibility version 1.0.0, current version 4.40.2, weak)
	/usr/lib/swift/libswiftFoundation.dylib (compatibility version 1.0.0, current version 20.0.0)
	/usr/lib/swift/libswiftMapKit.dylib (compatibility version 1.0.0, current version 2.0.0, weak)
	/usr/lib/swift/libswiftMetal.dylib (compatibility version 1.0.0, current version 1.3.1, weak)
	/usr/lib/swift/libswiftObjectiveC.dylib (compatibility version 1.0.0, current version 1.0.0)
	/usr/lib/swift/libswiftQuartzCore.dylib (compatibility version 1.0.0, current version 1.0.0, weak)
	/usr/lib/swift/libswiftUIKit.dylib (compatibility version 1.0.0, current version 19.0.0)
```

从上面能够发现，`Foundation` `UIKit` 等系统必备的库已经自动引用了，我们自己添加的 `CoreLocation` `CallKit` 也被引用了，不同的是 `CallKit` 后面有一个 `weak ` ，表明是可选库。系统库是被所有应用所共用的，在 App 运行时动态加载。

## CocoaPods 引入的动态库和静态库

首先怎么看一个库是静态库还是动态库呢，后缀为 `.a` 的是静态库，`.dylib` 是动态库，`.framework ` 可以是动态库也可以是静态库，可以通过 `file ` 命令查看可执行文件获取信息：

```zsh
➜  GoogleMaps.framework git:(main) ✗ cd /Users/zhiying.fan/Library/Developer/Xcode/DerivedData/Demo-Framework-cqyqoilkxvbzuwcjdiccbzbutijf/Build/Products/Debug-iphonesimulator/Demo-Framework.app/Frameworks/Alamofire.framework
➜  Alamofire.framework file Alamofire
Alamofire: Mach-O 64-bit dynamically linked shared library x86_64
➜  Alamofire.framework cd /Users/zhiying.fan/Desktop/repos/demos/Demo-Framework/Pods/GoogleMaps/Maps/Frameworks/GoogleMaps.framework
➜  GoogleMaps.framework git:(main) ✗ file GoogleMaps
GoogleMaps: Mach-O universal binary with 4 architectures: [arm_v7:Mach-O object arm_v7] [i386] [x86_64] [arm64]
GoogleMaps (for architecture armv7):	Mach-O object arm_v7
GoogleMaps (for architecture i386):	Mach-O object i386
GoogleMaps (for architecture x86_64):	Mach-O 64-bit object x86_64
GoogleMaps (for architecture arm64):	Mach-O 64-bit object arm64
```

- 动态库会显示 `dynamically linked shared library`
- 静态库则会显示是基于什么架构生成的

下面看一下使用 CocoaPods 引入动态库和静态库有什么区别：

作为例子，引入 `Alamofire` （动态库）和 `GoogleMaps` （静态库），编译之后来看一下包内容。

![](/assets/img/post/library/add-frameworks.png)

然后再用命令看一下链接的动态库

```zsh
➜  ~ otool -L /Users/zhiying.fan/Library/Developer/Xcode/DerivedData/Demo-Framework-cqyqoilkxvbzuwcjdiccbzbutijf/Build/Products/Debug-iphonesimulator/Demo-Framework.app/Demo-Framework
/Users/zhiying.fan/Library/Developer/Xcode/DerivedData/Demo-Framework-cqyqoilkxvbzuwcjdiccbzbutijf/Build/Products/Debug-iphonesimulator/Demo-Framework.app/Demo-Framework:
	/usr/lib/libc++.1.dylib (compatibility version 1.0.0, current version 904.4.0)
	/usr/lib/libz.1.dylib (compatibility version 1.0.0, current version 1.2.11)
	/System/Library/Frameworks/Accelerate.framework/Accelerate (compatibility version 1.0.0, current version 4.0.0)
	@rpath/Alamofire.framework/Alamofire (compatibility version 1.0.0, current version 1.0.0)
	/System/Library/Frameworks/CFNetwork.framework/CFNetwork (compatibility version 1.0.0, current version 1209.0.0)
	/System/Library/Frameworks/CoreData.framework/CoreData (compatibility version 1.0.0, current version 1044.3.0)
	/System/Library/Frameworks/CoreGraphics.framework/CoreGraphics (compatibility version 64.0.0, current version 1463.2.1)
	/System/Library/Frameworks/CoreImage.framework/CoreImage (compatibility version 1.0.0, current version 5.0.0)
	/System/Library/Frameworks/CoreLocation.framework/CoreLocation (compatibility version 1.0.0, current version 2420.12.14)
	/System/Library/Frameworks/CoreTelephony.framework/CoreTelephony (compatibility version 1.0.0, current version 0.0.0)
	/System/Library/Frameworks/CoreText.framework/CoreText (compatibility version 1.0.0, current version 1.0.0)
	/System/Library/Frameworks/GLKit.framework/GLKit (compatibility version 1.0.0, current version 124.0.0)
	/System/Library/Frameworks/ImageIO.framework/ImageIO (compatibility version 1.0.0, current version 1.0.0)
	/System/Library/Frameworks/Metal.framework/Metal (compatibility version 1.0.0, current version 244.40.1)
	/System/Library/Frameworks/OpenGLES.framework/OpenGLES (compatibility version 1.0.0, current version 1.0.0)
	/System/Library/Frameworks/QuartzCore.framework/QuartzCore (compatibility version 1.2.0, current version 1.11.0)
	/System/Library/Frameworks/SystemConfiguration.framework/SystemConfiguration (compatibility version 1.0.0, current version 1109.60.2)
	/System/Library/Frameworks/UIKit.framework/UIKit (compatibility version 1.0.0, current version 4003.1.101)
	/System/Library/Frameworks/CallKit.framework/CallKit (compatibility version 1.0.0, current version 1.0.0, weak)
	/System/Library/Frameworks/Foundation.framework/Foundation (compatibility version 300.0.0, current version 1770.255.0)
	/usr/lib/libobjc.A.dylib (compatibility version 1.0.0, current version 228.0.0)
	/usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 1292.60.1)
	/System/Library/Frameworks/CoreFoundation.framework/CoreFoundation (compatibility version 150.0.0, current version 1770.255.0)
	/System/Library/Frameworks/Security.framework/Security (compatibility version 1.0.0, current version 59754.62.1)
	/usr/lib/libc++abi.dylib (compatibility version 1.0.0, current version 904.4.0)
	/usr/lib/swift/libswiftCore.dylib (compatibility version 1.0.0, current version 1200.2.41)
	/usr/lib/swift/libswiftCoreFoundation.dylib (compatibility version 1.0.0, current version 1.6.0, weak)
	/usr/lib/swift/libswiftCoreGraphics.dylib (compatibility version 1.0.0, current version 2.0.0, weak)
	/usr/lib/swift/libswiftCoreImage.dylib (compatibility version 1.0.0, current version 1.0.0, weak)
	/usr/lib/swift/libswiftCoreLocation.dylib (compatibility version 1.0.0, current version 5.0.0, weak)
	/usr/lib/swift/libswiftDarwin.dylib (compatibility version 1.0.0, current version 0.0.0, weak)
	/usr/lib/swift/libswiftDispatch.dylib (compatibility version 1.0.0, current version 4.40.2, weak)
	/usr/lib/swift/libswiftFoundation.dylib (compatibility version 1.0.0, current version 20.0.0)
	/usr/lib/swift/libswiftMapKit.dylib (compatibility version 1.0.0, current version 2.0.0, weak)
	/usr/lib/swift/libswiftMetal.dylib (compatibility version 1.0.0, current version 1.3.1, weak)
	/usr/lib/swift/libswiftObjectiveC.dylib (compatibility version 1.0.0, current version 1.0.0)
	/usr/lib/swift/libswiftQuartzCore.dylib (compatibility version 1.0.0, current version 1.0.0, weak)
	/usr/lib/swift/libswiftUIKit.dylib (compatibility version 1.0.0, current version 19.0.0)
```

从包内容可以发现，`Alamofire` 被放在了 `Frameworks` 文件夹下，动态库加载器会从 `@rpath/Alamofire.framework/Alamofire` 去链接它。`@rpath` 代表的是已经被嵌入到应用程序的本地列表。

而对于静态库 `GoogleMaps` ，在包内容中只能看到它的资源文件 `GoogleMaps.bundle` ，在动态链接库中也看不到它。因为它是被拷贝到可执行文件中去了。

## 在自己的库中引用动态库和静态库

1. 参考 [Using Pod Lib Create](https://guides.cocoapods.org/making/using-pod-lib-create.html) 创建自己的库 `Maps`

2. 在自己的框架中引入 `Alamofire` 和 `GoogleMaps`

3. 将新创建的框架引入到 Demo 工程中去

在上面三步做完并且执行 `pod install` 时，遇到了报错：

```zsh
➜  Demo-Framework git:(main) ✗ pod install
Analyzing dependencies
Downloading dependencies
Installing Alamofire (5.4.1)
Installing GoogleMaps (4.1.0)
Installing Maps 0.1.0
[!] The 'Pods-Demo-Framework' target has transitive dependencies that include statically linked binaries: (/Users/zhiying.fan/Desktop/repos/demos/Demo-Framework/Pods/GoogleMaps/Base/Frameworks/GoogleMapsBase.framework, /Users/zhiying.fan/Desktop/repos/demos/Demo-Framework/Pods/GoogleMaps/Maps/Frameworks/GoogleMaps.framework, and /Users/zhiying.fan/Desktop/repos/demos/Demo-Framework/Pods/GoogleMaps/Maps/Frameworks/GoogleMapsCore.framework)
```

因为我们引用的 `GoogleMaps` 是静态库，所以自己的库也必须是静态库，[CocoaPods 指定静态库的语法](https://guides.cocoapods.org/syntax/podspec.html#static_framework)为：

```ruby
spec.static_framework = true
```

加上之后再执行 `pod install` ，然后去看动态库链接，会发现和之前直接在工程中引用并没有什么不同。

## 静态库中的资源文件

静态库在打包时，资源文件是不会被一起打包进去的，CocoaPods 会将包的资源文件拷贝到项目中去。CocoaPods 在对包的资源文件进行拷贝时，有两种选择可以配置：[resources](https://guides.cocoapods.org/syntax/podspec.html#resources) 和 [resource_bundles](https://guides.cocoapods.org/syntax/podspec.html#resource_bundles)

![resources](/assets/img/post/library/resources.jpg)

- resources: 把所有库的资源文件直接拷贝到项目中。
- resource_bundles: 把所有资源文件放在一个 `.bundle` 的文件夹下，然后再拷贝到项目中，使用这种方式可以避免文件名冲突的问题。

### 获取资源文件

因为 `resources` 和 `resource_bundles` 对资源文件的不同处理方式，我们在获取这些资源的时候，也需要区别对待：

```swift
  var mapsResourceBundle: Bundle {
    let codeBundle = Bundle(for: MapView.self)
    guard
      let resourceBundlePath = codeBundle.path(forResource: "Maps", ofType: "bundle"),
      let resourceBundle = Bundle(path: resourceBundlePath)
    else {
      return codeBundle
    }
    return resourceBundle
  }
```

- 如果使用的是 resources 方式进行资源文件的打包，那么用 `codeBundle ` 就可以获取到 `mainBundle`，资源文件就直接放置在其中。
- 如果使用的是 resource_bundles 方式进行资源文件的打包，那么获取到 `mainBundle ` 之后，还需要找到自己库的 `bundle` 文件，然后才能在其中找到想要的资源。

## Demo

[![](/assets/img/post/download-demo.png){: width="200"}](https://github.com/zhiying-fan/Demo-Framework.git)

## 参考

- [Dynamic Library Programming Topics](https://developer.apple.com/library/archive/documentation/DeveloperTools/Conceptual/DynamicLibraries/000-Introduction/Introduction.html#//apple_ref/doc/uid/TP40001869)
- [Dynamic Frameworks](https://www.raywenderlich.com/books/advanced-apple-debugging-reverse-engineering/v3.0/chapters/15-dynamic-frameworks)

