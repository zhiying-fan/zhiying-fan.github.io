---
title: iOS 地图 MapKit - 搜索与路径
date: 2020-12-25 20:00:00 +0800
categories: [iOS]
tags: [map]     # TAG names should always be lowercase
image:
  src: /assets/img/post/map/header.png
---

使用地图来搜索感兴趣的地方，并且获取到达该地方的路径，想必是我们经常使用到的功能，那么在 MapKit 中如何实现呢，下面一起来看看吧。

**写下该文章的时候，计算路径的 API 在国内还是不可用的，具体的可用区域可以查看 [iOS and iPadOS Feature Availability](https://www.apple.com/ios/feature-availability/#maps-directions)**

## 搜索

MapKit 中实现搜索功能的类是 `MKLocalSearch`，主要有三个方法：

```swift
// 初始化
init(request: MKLocalSearch.Request)
init(request: MKLocalPointsOfInterestRequest)

// 开始搜索
func start(completionHandler: @escaping MKLocalSearch.CompletionHandler)

// 取消搜索
func cancel()
```

我们分别来看下初始化和搜索方法

### 初始化

初始化的时候传入 `MKLocalSearch.Request`，可以配置以下信息：

| 配置信息              | 说明                                                       |
| --------------------- | ---------------------------------------------------------- |
| naturalLanguageQuery  | 搜索关键字                                                 |
| region                | 搜索区域                                                   |
| resultTypes           | 想要搜索的结果类型，可以选择 `address` , `pointOfInterest` |
| pointOfInterestFilter | POI的类型筛选，比如 `airport ` , `bank` , `cafe` 等等      |

`MKLocalPointsOfInterestRequest` 是针对只搜索 POI 的情况，和 `Request` 比较类似。

### 开始搜索

初始化配置完成之后，就可以开始搜索了，搜索回调返回的是 `MKLocalSearch.Response` 或者 `Error` 。

Response 中包含了结果数组 `[MKMapItem]` 和边界区域 `MKCoordinateRegion` ，我们就可以根据结果展示在地图上了。

这里需要注意的是，如果是在搜索中，那么调用 `start` 方法将会返回失败信息，可以使用 `isSearching` 属性判断是否正在搜索中。

## 搜索自动补全

![diagram](/assets/img/post/map/search-completer.gif)

自动补全也是比较实用的功能，实现类是 `MKLocalSearchCompleter` 

配置信息和 `MKLocalSearch.Request` 非常类似，配置好之后，每次设置 `queryFragment` 值，都可以在 `MKLocalSearchCompleterDelegate` 中得到结果。

```swift
func completerDidUpdateResults(_ completer: MKLocalSearchCompleter) {
  // results 中是 MKLocalSearchCompletion，只包含 title, subtitle 和相应的 Ranges 信息
  results = completer.results.map { $0.title }
  tableView.reloadData()
}
```

有了自动补全的 title 之后，就可以调用前面介绍过的搜索方法，拿到具体的地理位置信息了。

## 计算路径

在搜索到我们感兴趣的地方之后，下一步就是计算到达路径了，用到的类是 `MKDirections` ，主要方法有：

```swift
// 初始化
init(request: MKDirections.Request)

// 开始计算
func calculate(completionHandler: @escaping MKDirections.DirectionsHandler)
func calculateETA(completionHandler: @escaping MKDirections.ETAHandler)

// 取消计算
func cancel()
```

还是分别来看下初始化和计算方法

### 初始化

初始化的时候传入 `MKDirections.Request`，可以配置以下信息：

| 配置信息                | 说明                                                         |
| ----------------------- | ------------------------------------------------------------ |
| source                  | 出发地                                                       |
| destination             | 目的地                                                       |
| transportType           | 交通方式，可选的有 `automobile` , `walking` , `transit` , `any`  目前还没有 Cycling |
| requestsAlternateRoutes | 是否允许多条路径，默认为 false                               |
| departureDate           | 出发时间                                                     |
| arrivalDate             | 到达时间                                                     |

### 开始计算

初始化配置完成之后，就可以开始计算路径了，回调返回的是 `MKDirections.Response` 或者 `Error` 。

Response 中包含了结果数组 `[MKRoute]` 和起点终点信息，来看看 `MKRoute` 都包含了哪些信息：

| 字段               | 说明                                            |
| ------------------ | ----------------------------------------------- |
| name               | 路径名字                                        |
| advisoryNotices    | 重要的公告，比如修路、封路等信息                |
| distance           | 距离                                            |
| expectedTravelTime | 预计花费时间                                    |
| transportType      | 交通方式                                        |
| polyline           | 用来在地图上绘制路径信息                        |
| steps              | `[MKRoute.Step]` 详细的路径步骤，比如到路口左转 |

拿到以上信息，就可以根据需要展示给用户了，使用 `polyline` 绘制在地图上，或者用列表展示 `steps` 信息。

如果只想要获取预计花费时间，那么使用 `calculateETA` 方法，将不包含 `polyline` , `steps` 等信息。

这里需要注意的是，如果是在计算中，那么调用 `calculate` 方法将会返回失败信息，可以使用 `isCalculating` 属性判断是否正在计算中。

## 地理编码

地理编码即根据地址名获取经纬度等信息，反地理编码则反之，常用在获取到用户定位之后进行反地理编码。

### 编码

```swift
CLGeocoder().geocodeAddressString("故宫博物院", in: region, preferredLocale: locale) { (placemarks, error) in

}
```

### 反编码

```swift
CLGeocoder().reverseGeocodeLocation(location, preferredLocale: locale) { (placemarks, error) in

}
```

## 参考

- [Routing With MapKit and Core Location](https://www.raywenderlich.com/10028489-routing-with-mapkit-and-core-location)

