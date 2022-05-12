---
title: 读《Fullstack React Native》（二）
date: 2022-03-24 20:00:00 +0800
categories: [读书]
tags: [books]     # TAG names should always be lowercase
image:
  src: /assets/img/post/rn/header.png
---

## 七步法构建 RN 应用

这本书的第二章，用了一个新的示例应用：计时器，介绍了构建 RN 应用的七步法，分别是：

1. 把应用分解为不同的组件。
2. 开始构建一个静态的应用。
3. 决定哪些数据需要使用 state 来保存状态。
4. 决定这些 state 应该存在在哪个组件中。
5. 先用 hardcode 的初始状态。
6. 添加数据流动逻辑，包括向上的数据流动。
7. 添加和服务器的通信。

这个计时器应用长这样子：

![](/assets/img/post/rn/timer.jpg)

整章主要是采用该七步法，去实际的写代码来实现这个计时器，使我们能够真正理解这个方法，没有介绍新的关于 RN 的核心概念，这里我对自认为有用的两点做了笔记。

### Class components vs Functional components

在上一章中，我们创建组件使用的都是 `Class components`, 本章介绍了另外一种方式 `Functional components` 。下面通过一个简单的例子对比一下：

```jsx
class ClassComponent extends React.Component {
  render() {
    const { name } = this.props;
    return <Text>Hello, { name }</Text>;
 }
}

const FunctionalComponent = ({ name }) => {
 return <Text>Hello, {name}</Text>;
};
```

`Functional components` 可以被认为是只需要实现 `render()` 方法的组件，他们不管理 `state` ，也不需要关注 React 特殊的生命周期。`props` 被作为第一个参数传入并解构。

书中比较推荐的是 `Functional components`，有两个主要原因：

1. 它能促使程序员在更少的地方管理 state，使程序更易懂。
2. 它能使我们创建高可复用的组件，因为它所有的配置都需要从外面传进去。

### 需要使用 State 的判定标准

在第三步中，需要决定哪些数据需要使用 state，这里 Facebook 在 [Thinking In React](https://facebook.github.io/react/docs/thinking-in-react.html) 这篇文章中，给了三个评判标准：

1. 该数据是通过 props 从父组件中传入的吗？如果是，那它可能不是一个 state。
2. 该数据会在后续被改变吗？如果不会，那它可能不是一个 state。
3. 该数据可以根据组件中其他的 state 或者 props 计算得出吗？如果是，那它可能不是一个 state。

## Core Components

### View

`View` 在 RN 中是被广泛使用的，它通常在以下两种情况下使用：

- 使用 `View` 来进行布局，对其中的子视图进行横向或纵向排列。
- 使用 `View` 来添加样式，例如添加边框、背景色等等。

#### 布局

##### Flex

RN 中的布局方式和前端类似，主要采用 `flex` 进行布局。在父视图上，有三个属性可供我们使用：

- `flexDirection`
- `justifyContent`
- `alignItems`

`flexDirection` 控制子组件的主轴方向，可选的值如下：

![](/assets/img/post/rn/flex-direction.jpg)

`justifyContent` 控制子组件沿着主轴的排列方式，因此会受到主轴方向的影响，它可选的值如下：

![](/assets/img/post/rn/justify-content.jpg)

`alignItems` 控制子组件沿着交叉轴的排列方式，因此也会受到主轴方向的影响，它可选择的值如下：

![](/assets/img/post/rn/align-items.jpg)

关于 `flex` 的使用，更详细的用法可以参考官方文档 [Layout with Flexbox](https://reactnative.dev/docs/flexbox)

##### Position

`Position` 这种布局方式是无视它的兄弟视图的，可能会使得视图重叠，它有两个值可以选择：

- `relative`：在视图被布局好之后，对其进行位置的调整
- `absolute`：完全无视其他布局对它的影响，只相对于父视图进行布局

比如如下的布局，会使得视图填充满父视图

```
const absoluteFillStyle = {
  position: 'absolute',
  top: 0,
  right: 0,
  bottom: 0,
  left: 0,
};
```

#### Box Model

RN 中对于间距的设置和 Web 中是一样的，都是用 Box Model, 一般通过设置 `margin` 和 `padding` 就能满足我们的需求。

![](/assets/img/post/rn/box-model.jpg)

当我们使用 `width` 和 `height` 时，指的是 content 的宽高加上 padding 和 border，不包括 margin。

### Text

`Text` 组件是有它的固有尺寸的，这和 iOS 开发中类似，渲染所需的文本所需要的宽高就是它默认的尺寸。

Text 是比较简单的，那些常见的属性它都有，比如 `color` `fontFamily` `fontSize` `textAlign` `numberOfLines` 。

### TouchableOpacity

这个组件和 `View` 类似，不同的是它可以接受点击事件，并且会在按下和释放的时候附带有动画效果，可以修改 `activeOpacity` 来调整动画对透明度的改变量。

如果想要背景色改变的动画，可以使用 `TouchableHighlight`。这两个组件都只能有一个子组件，这是它们的一点限制。

### Image

它可以通过 `source` 来渲染本地或者来自 URI 的图片。如果要设置图片的填充模式，可以使用 `resizeMode`。

### ActivityIndicator

它会渲染 iOS 和 Android 平台上默认的 Loading 动画。

### FlatList

这是用来渲染长列表的组件，它接受一个 `data` 的数据列表还有每个列表元素组件的构建方法 `renderItem`。它每次只会渲染一屏的组件来提升性能。

### TextInput

文本输入框，主要用到的属性有 `value`，`onChangeText`，`onSubmitEditing`。


### ScrollView

在渲染少量的需要滑动展示的视图时，可以使用 `ScrollView`，它不像 `FlatList` 那样每次只渲染一屏的组件，而是会将所有的组件全部渲染，所以只能用于少量数据的展示，否则会有性能问题。

### Modal

用来展示模态视图的容器，用法如下，通过 `visible` 来控制其显示：

```jsx
<Modal
  visible={showModal}
  animationType="slide"
  onRequestClose={this.closeCommentScreen}
>
  <Comments/>
</Modal>

```

## Core APIs

除了内置的组件之外，还有一些功能需要调用 API 来实现，比如存储、获取网络状况等。我们可以遵循以下方法来调用 RN 的 API：

1. 先根据需求确定我们需要调用哪个 API
1. 确定调用的位置，例如在某个生命周期的方法或者工具类中
1. 调用 API，要注意是异步还是同步
1. 在组件的 `state` 中存储返回的结果
1. 根据最新的 `state` 来重新渲染 UI

### AsyncStorage

RN 提供了 `AsyncStorage` 来做少量的字符串存储，它通过键值对然后序列化成 Json 去存储，基础的 API 是：

```
AsyncStorage.getItem(key)
AsyncStorage.setItem(key, value)
```

如果要存储对象，需要先转成 Json 字符串，取出之后也需要解析成对象去使用。

```
JSON.stringify(object)
JSON.parse(jsonString)
```

### StatusBar

RN 提供了两种方式来调整状态栏的 UI。

一种是通过组件。这个组件可以放在任意的组件树中：

```jsx
const statusBar = (
  <StatusBar
    backgroundColor={backgroundColor}
    barStyle={isConnected ? 'dark-content' : 'light-content'}
    animated={false}
  />
);
```

第二种方法是调用 API：

```js
StatusBar.setBarStyle(info === 'none' ? 'light-content' : 'dark-content');
```

### NetInfo

这是用来获取网络状态的 API

下面这个 API 能够获取到当前的网络状态，返回的值可能为 `wifi`，`cellular` 或者 `none`

```js
const info = await NetInfo.getConnectionInfo();
```

同时也可以监听网络的变化

```js
async componentWillMount() {
  this.subscription = NetInfo.addEventListener('connectionChange', this.handleChange);
}

componentWillUnmount() {
  this.subscription.remove();
}

handleChange = (info) => {
  this.setState({ info });
};
```

### Alert

下面是 RN 中显示 Alert 的 API，会分别在 iOS 和 Android 上渲染 Alert 组件。

```js
handlePressMessage = () => {
  Alert.alert(
    'Delete message?',
    'Are you sure you want to permanently delete this message?',
    [
      {
        text: 'Cancel',
        style: 'cancel',
      },
      {
        text: 'Delete',
        style: 'destructive',
        onPress: () => {
          const { messages } = this.state;
          this.setState({
            messages: messages.filter(message => message.id !== id),
          });
        },
      },
    ],
  );
};
```

### Geolocation

获取位置的 API：

```js
navigator.geolocation.getCurrentPosition(position => {
  const { coords: { latitude, longitude } } = position;
});
```

### Dimensions

获取设备屏幕的宽高可以使用这个 API，要注意获取到的值是当前的屏幕宽高，如果旋转屏幕之后需要重新调用获取。

```js
const windowWidth = Dimensions.get('window').width;
const windowHeight = Dimensions.get('window').height;
```

## 组件生命周期

![](/assets/img/post/rn/lifecycle.png)

## 参考

- [React Native Lifecycle](https://github.com/yasinugrl/react-native-lifecycle)
