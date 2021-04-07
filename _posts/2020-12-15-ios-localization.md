---
title: iOS 项目国际化 Localization
date: 2020-12-15 20:00:00 +0800
categories: [iOS]
tags: [localization]     # TAG names should always be lowercase
image:
  src: /assets/img/post/localization/header.png
---

随着 Apple 对多语言用户使用体验的不断提升，从 iOS13 开始用户可以在应用设置中单独设置区别于系统的语言，同时对于开发者来说，对于多语言的适配体验也在不断的得到提升，本文将逐步说明如何对 iOS 项目做国际化支持。

## 添加支持的语言

支持国际化的第一步是添加我们想要支持的语言，选中 Project - Info ，在 Localizations 下面可以添加语言。

![](/assets/img/post/localization/add-language.png)

如果项目中存在可国际化的文件，比如 Storyboard / xib，下一步会让我们选择需要创建的文件，这时我们选择 `Localizable Strings`，如果选择 `Interface Builder Storyboard` 将会新建额外的 Storyboard ，意味着我们在支持不同语言的同时，还要为不同的语言创建不同的界面，这一般不是我们想要的。

![](/assets/img/post/localization/choose-files.png)

完成之后，将会为每一个 Storyboard / xib 新建一个  `strings` 文件，并统一放在该语言的文件下，我们暂时不去翻译它们。

![](/assets/img/post/localization/iproj.png)

## 为你的资源添加国际化支持

项目中使用的图片、颜色、音频、视频等资源文件，可能都需要支持国际化，选中资源，在 Inspector - Localization 下面，可以添加我们想要支持的语言。

![](/assets/img/post/localization/inspector-localization.png)

添加对应语言的资源文件，到这我们完成了项目中所有资源的国际化支持。

![](/assets/img/post/localization/add-resource.png)

## 导出需要支持国际化的文件

在添加完资源文件之后，接下来就是对字符串进行国际化支持了。选中 Project - Editor - Export for Localization，导出 `.xcloc` (Xcode Localization Catalog) 文件。

![](/assets/img/post/localization/export-localization.png)

导出的 `zh-Hans.xcloc` 包含以下内容：

| 文件               | 描述                                                         |
| ------------------ | ------------------------------------------------------------ |
| contents.json      | 一个包含目录元数据的 json 文件，例如开发工具信息，目录版本等 |
| Localized Contents | 包含需要国际化支持资源的文件夹，重要的 `xliff` 文件就在这里  |
| Source Contents    | 包含生成国际化资源的源文件，主要是为翻译人员提供上下文       |
| Notes              | 给翻译人员提供的附加信息，例如截屏等                         |

Xcode 将我们源代码中传给 [SwiftUI Text](https://developer.apple.com/documentation/swiftui/text) , [NSLocalizedString](https://developer.apple.com/documentation/foundation/nslocalizedstring) 以及类似 API 中的字符串，storyboard, XIB, strings 等中的字符串提取出来，放在了一个 `xliff` (XML Localization Interchange File Format) 文件中。下一步就是对这个文件中的字符串进行翻译了。

## 翻译字符

这一步我们使用 [Poedit](https://poedit.net/) 进行翻译，一个开源的跨平台翻译编辑器。
可以看到这个文件包含了我们在代码中用到的字符串、在 Storyboard 中用到的字符串、应用的名称以及获取权限时的说明，非常的全面，我们对它们逐一进行翻译。

![](/assets/img/post/localization/poedit.png)

## 导入支持国际化之后的文件

选中 Project - Editor - Import Localizations，导入 `.xcloc` 文件。
如果存在没有国际化的信息，Xcode 会有 Warning 提示。如果我们想要在导入之前查看所有改动，点击文件图标，选中对应的文件就能看到，左边是我们即将导入的文件，右边是当前项目的文件。

![](/assets/img/post/localization/import-localizations.png)

导入之后 Xcode 为我们新建了 
- `InfoPlist.strings` 文件，包含了 `CFBundleName` 以及 `NSPhotoLibraryUsageDescription`
- `Localizable.strings` 文件，包含了我们代码中用到的字符

我们以后有新的字符串需要支持国际化时，也可以直接修改以上的文件，在文件中定义 key 和相应的文字。

## App Store 信息

我们还应该为我们应用在 App Store 中展示的信息做国际化支持，如果我们只提供一种语言支持，那么在所有应用可销售区域，将只显示一种语言。

### 添加语言
在 App Store 中选中 App，点击右上角的首选语言，可以添加我们想要支持的语言

![](/assets/img/post/localization/choose-language.png)

### 编辑信息
在相应的语言对应的界面中，编辑应用信息，包含不同的 Screenshots。

### 修改首选语言
在添加过语言之后，可以在 App 的 General Information 信息中，修改首选语言。

![](/assets/img/post/localization/app-info.png)

## Demo

![](/assets/img/post/localization/demo.png)

[![](/assets/img/post/download-demo.png){: width="200"}](https://github.com/zhiying-fan/Demo-Localization.git)

## 参考

- [Creating Great Localized Experiences with Xcode 11](https://developer.apple.com/videos/play/wwdc2019/403/)
- [Localizing Your App](https://developer.apple.com/documentation/xcode/localizing_your_app)
- [Xcode Help: Localize your app](https://help.apple.com/xcode/mac/current/#/deve2bc11fab)