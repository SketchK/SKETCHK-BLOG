---
title: iOS系统中导航栏的转场解决方案与最佳实践
comments: true
date: 2022-08-29 15:49:21
updated:
tags:
  - iOS
categories:
  - iOS

---

这篇文章是我早年在美团技术博客上发布的一篇文章, 部分内容可能已经过时, 请开发者注意.
将文章同步到个人博客主要是为了同步和备份.

<!-- more -->

## 背景

目前，开源社区和业界内已经存在一些 iOS 导航栏转场的解决方案，但对于历史包袱沉重的美团 App 而言，这些解决方案并不完美。有的方案不能满足复杂的页面跳转场景，有的方案迁移成本较大，为此我们提出了一套解决方案并开发了相应的转场库，目前该转场库已经成为美团点评多个 App 的基础组件之一。

在美团 App 开发的早期，涉及到导航栏样式改变的需求时，经常会遇到转场效果不佳或者与预期样式不符的“小问题”。在业务体量较小的情况下，为了满足快速的业务迭代，通常会使用硬编码的方式来解决这一类“小问题”。但随着美团 App 业务的高速发展，这种硬编码的方式遇到了以下的挑战：

1. 业务模块的不断增加，导致使用硬编码方式编写的代码维护成本增加，代码质量迅速下降。
2. 大型 App 的路由系统使得页面间的跳转变得更加自由和灵活，也使得导航栏相关的问题激增，不但增加了问题的排查难度，还降低了整体的开发效率。
3. App 中的导航栏属于各个业务方的公用资源，由于缺乏相应的约束机制和最佳实践，导致业务方之间的代码耦合程度不断增加。

从各个角度来看，硬编码的方式已经不能很好的解决此类问题，美团 App 需要一个更加合理、更加持久、更加简单易行的解决方案来处理导航栏转场问题。

本文将从导航栏的概念入手，通过讲解转场过程中的状态管理、转换时机和样式变化等内容，引出了在大型应用中导航栏转场的三种常见解决方案，并对美团点评的解决方案进行剖析。

## 重新认识导航栏

### 导航栏里的 MVC

在 iOS 系统中， 苹果公司不仅建议开发者遵循 MVC 开发框架，在它们的代码里也可以看到 MVC 的影子，导航栏组件的构成就是一个类似 MVC 的结构，让我们先看看下面这张图：

![01导航栏组件关系图.png](1.png)

在这张图里，我们可以将 UINavigationController 看做是 C，UINavigationBar 看做是 V，而 UIViewController 和 UINavigationItem 组成的 Stack 可以看做是 M。这里要说明的是，每个 UIViewController 都有一个属于自己的 UINavigationItem，也就是说它们是一一对应的。

UINavigationController 通过驱动 Stack 中的 UIViewController 的变化来实现 View 层级的变化，也就是 UINavigationBar 的改变。而 UINavigationBar 样式的数据就存储在 UIViewController 的 UINavigationItem 中。这也就是为什么我们在代码里只要设置 `self.navigationItem` 的相关属性就可以改变 UINavigationBar 的样式。

很多时候，国内的开发者会将 UINavigationBar 和 UINavigationController 混在一起叫导航栏，这样的做法不仅增加了开发者之间的沟通成本，也容易导致误解。毕竟它们是两个完全不一样的东西。

所以本文为了更好的阐明问题，会采用英文区分不同的概念，当需要描述笼统的导航栏概念时，会使用导航栏组件一词。

通过这一节的回顾，我们应该明确了 NavigationItem、ViewController、NavigationBar 和 NavigationController 在 MVC 框架下的角色。下面我们会重新梳理一下导航栏的生命周期和各个相关方法的调用顺序。

### 导航栏组件的生命周期

大家可以通过下图获得更为直观的感受，进而了解到导航栏组件在 push 过程中各个方法的调用顺序。

![02push过程中的方法调用顺序图.png](2.png)

值得注意的地方有两点：

第一个是 UINavigationController 作为 UINavigationBar 的代理，在没有特殊需求的情况下，不应该修改其代理方法，这里是通过符号断点获取它们的调用顺序。如果我们创建了一个自定义的导航栏组件系统，它的调用顺序可能会与此不同。

第二个是用虚线圈起来的方法，它们也有可能不被调用，这与 ViewController 里的布局代码相关，假设跳转到新页面后，新旧页面中的控件位置会发生变化，或者由于数据改变驱动了控件之间的约束关系发生变化，这就会带来新一轮的布局，进而触发 `viewWillLayoutSubview` 和 `viewDidLayoutSubview` 这两个方法。当然，具体的调用顺序会与业务代码紧密相关，如果我们发现顺序有所不同，也不必惊慌。

下面这张图展示了导航栏在 pop 过程中各个方法的调用顺序：

![03pop过程中的方法调用顺序图.png](3.png)

除了上面说到的两点，pop 过程中还需要注意一点，那就是从 B 返回到 A 的过程中，A 视图控制器的 `viewDidLoad` 方法并不会被调用。关于这个问题，只要提醒一下，大多数人都会反应过来是为什么。不过在实际开发过程中，总会有人忘记这一点。

通过这两个图，我们已经基本了解了导航栏组件的生命周期和相关方法的调用顺序，这也是后面章节的理论基础。

### 导航栏组件的改变与革新

导航栏组件在 iOS 11 发布时，获得了重大更新，这个更新可不是增加了一个大标题样式（Large Title Display Mode）那么简单，需要注意的地方大概有两点：

1. 导航栏全面支持 Auto Layout 且 NavigationBar 的层级发生了明显的改变，关于这一点可以阅读 [UIBarButtonItem 在 iOS 11 上的改变及应对方案](http://sketchk.xyz/2018/02/23/How-to-make-your-UIBarButtonItem-perfect-match-in-iOS/) 。

2. 由于引进了 Safe Area 等概念，`topLayoutGuide` 和 `bottomLayoutGuide` 等属性会逐渐废弃，虽然变化不大，但如果我们的导航栏在转场过程中总是出现视图上下移动的现象，不妨从这个方面思考一下，如果想深究可以查看 [WWDC 2017 Session 412](https://developer.apple.com/videos/play/wwdc2017/412/)。

## 导航栏组件到底怎么了？

经常有人说 iOS 的原生导航栏组件不好使用，抱怨主要集中在导航栏组件的状态管理和控件的布局问题上。

控件的布局问题随着 iOS 11 的到来已经变得相对容易处理了不少，但导航栏组件的状态管理仍然让开发者头疼不已。

可能已经有朋友在思考导航栏组件的状态管理到底是什么东西？不要着急，下面的章节就会做相关的介绍。

### 导航栏的状态管理

虽然导航栏组件的 push 和 pop 动画给人一种每次操作后都会创建一遍导航栏组件的错觉，但实际上这些 ViewController 都是由一个 NavigationController 所管理，所以你看到的 NavigationBar 是唯一的。

![04导航栏示例图.png](4.png)

在 NavigationController 的 Stack 存储结构下，每当 Stack 中的 ViewController 修改了导航栏，势必会影响其他 ViewController 展示的效果。

例如下图所示的场景，如果 NavigationBar 原先的颜色是绿色，但之后进入 Stack 里的 ViewController 将 NavigationBar 颜色修改为紫色后，在此之后 push 的 ViewController 会从默认的绿色变为紫色，直到有新的 ViewController 修改导航栏颜色才会发生变化。

![05导航栏push状态.png](5.png)

虽然在 push 过程中，NavigationBar 的变化听起来合情合理，但如果你在 NavigationBar 为绿色的 ViewController 里设置不当的话，那么当你 pop 回这个 ViewController 时，NavigationBar 可就不一定是绿色了，它还会保持为紫色的状态。

![06导航栏pop状态.png](6.png)

通过这个例子，我们大概会意识到在导航栏里的 Stack 中，每个 ViewController 都可以永久的影响导航栏样式，这种全局性的变化要求我们在实际开发中必须坚持“谁修改，谁复原”的原则，否则就会造成导航栏状态的混乱。这不仅仅是样式上的混乱，在一些极端状况下，还有可能会引起 Stack 混乱，进而造成 Crash 的情况。

### 导航栏样式转换的时机

我们刚才提到了“谁修改，谁复原”的原则，但何时修改，何时复原呢？

对于那些存储在 Stack 中的 ViewController 而言，它其实就是在不断的经历 appear 和 disappear 的过程，结合 ViewController 的生命周期来看，`viewWillAppear:` 和 `viewWillDisappear:` 是两个完美的时间节点，但很多人却对这两个方法的调用存在疑惑。

苹果公司在它的 API 文档中专门用了一段文字来解答大家的疑惑，这段文字的标题为《Handling View-Related Notifications》，在这里我们直接引用原文：

> When the visibility of its views changes, a view controller automatically calls its own methods so that subclasses can respond to the change. Use a method like viewWillAppear: to prepare your views to appear onscreen, and use the viewWillDisappear: to save changes or other state information. Use other methods to make appropriate changes.
> Figure 1 shows the possible visible states for a view controller’s views and the state transitions that can occur. Not all ‘will’ callback methods are paired with only a ‘did’ callback method. You need to ensure that if you start a process in a ‘will’ callback method, you end the process in both the corresponding ‘did’ and the opposite ‘will’ callback method.

![07视图管理器的状态转换.jpg](7.jpg)

这里很好的解释了所有的 will 系列方法和 did 系列方法的对应关系，同时也给我们吃了一个定心丸，那就是在 appearing 和 disappearing 状态之间会由 will 系列方法进行衔接，避免了状态中断。这对于连续 push 或者连续 pop 的情况是及其重要的，否则我们无法做到 “谁修改，谁复原”的原则。

通常来说，如果只是一个简单的导航栏样式变化，我们的代码结构大体会如下所示：

```objc
- (void)viewWillAppear:(BOOL)animated{
    [super viewWillAppear:animated];
    // MARK: change the navigationbar style 
}

- (void)viewWillDisappear:(BOOL)animated{
    [super viewWillDisappear:animated];
    // MARK: restore the navigationbar style
}
```

现在，我们明确了修改时机，接下来要明确的就是导航栏的样式会进行怎样的变化。

### 导航栏的样式变化

对于不同 ViewController 之间的导航栏样式变化，大多可以总结为两种情况：

1. 导航栏的显示与否
2. 导航栏的颜色变化

#### 导航栏的显示与否

对于显示与否的问题，可以在上一节提到的两个方法里调用 `setNavigationBarHidden:animated:` 方法，这里需要提醒的有两点：

1. 在导航栏转场的过程中，不要天真的以为 `setNavigationBarHidden:` 和 `setNavigationBarHidden:animated:` 的效果是一样的，直接使用 `setNavigationBarHidden:` 会造成导航栏转场过程中的闪现、背景错乱等问题，这一现象在使用手势驱动转场的场景中十分常见，所以正确的方式是使用带有 animated 参数的 API。
2. 在 push 和 pop 的方法里也会带有 animated 参数，尽量保证与 `setNavigationBarHidden:animated:` 中的 animated 参数一致。

#### 导航栏的颜色变化

颜色变化的问题就稍微复杂一些，在 iOS 7 后，导航栏增加了 `translucent` 效果，这使得导航栏背景色的变化出现了两种情况：

1. `translucent` 属性值为 YES 的前提下，更改导航栏的背景色。
2. `translucent` 属性值为 NO 的前提下，更改导航栏的背景色。

对于第一种情况，我们需要调用 UINavigationBar 的 `setBackgroundColor:` 方法。

对于第二种情况我们需要调用 UINavigationBar 的 `setBackgroundImage:forBarMetrics:` 方法。

对于第二种情况，这里有三点需要提示：

1. 在设置透明效果时，我们通常可以直接设置一个 `[UIImage new]` 创建的对象，无须创建一个颜色为透明色的图片。
2. 在使用 `setBackgroundImage:forBarMetrics:` 方法的过程中，如果图像里存在 `alpha` 值小于 1.0 的像素点，则 `translucent` 的值为 YES，反之为 NO。也就是说，如果我们真的想让导航栏变成纯色且没有 `translucent` 效果，请保证所有像素点的 `alpha` 值等于 1。
3. 如果设置了一个完全不透明的图片且强行将 NavigationBar 的 `translucent` 属性设置为 YES 的话，系统会自动修正这个图片并为它添加一个透明度，用于模拟 `translucent` 效果。
4. 如果我们使用了一个带有透明效果的图片且导航栏的 `translucent` 效果为 NO 的话，那么系统会在这个带有透明效果的图片背后，添加一个不透明的纯色图片用于整体效果的合成。这个纯色图片的颜色取决于 `barStyle` 属性，当属性为 `UIBarStyleBlack` 时为黑色，当属性为 `UIBarStyleDefault` 时为白色，如果我们设置了 `barTintColor`，则以设置的颜色为基准。

#### 分清楚 `transparent`，`translucent`，`opaque`，`alpha` 和 `opacity` 也挺重要

在刚接触导航栏 API 时，许多人经常会把文档里的这些英文词搞混，也不太明白带有这些词的变量为什么有的是布尔型，有的是浮点型，总之一切都让人很困惑。

在这里将做了一个总结，这对于理解 Apple 的 API 设计原则十分有帮助。

`transparent`， `translucent`， `opaque` 三个词经常会用在一起，它用于描述物体的透光强度，为了让大家更好的理解这三个词，这里做了三个比喻：

* `transparent` 是指透明，就好比我们可以透过一面干净的玻璃清楚的看到外面的风景。
* `translucent` 是指半透明，就好比我们可以透过一面有点磨砂效果的塑料墙看外面的风景，不能说看不见，但我们肯定看不清。
* `opaque` 是指不透明，就好比我们透过一个堵石墙是看不见任何外面的东西，眼前看到的只有这面墙。

这三个词更多的是用来表述一种状态，不需要量化，所以这与这三个词相关的属性，一般都是 BOOL 类型。

![08transparent-translucent-opaque的区别.png](8.png)

`alpha` 和 `opacity` 经常会在一起使用，它要表示的就是透明度，在 Web 端这两个属性有着明显的区别。

在 Web 端里，`opacity` 是设定整个元素的透明值，而 `alpha` 一般是放在颜色设置里面，所以我们可以做到对特定对元素的某个属性设定 `alpha`，比如背景、边框、文字等。

```css
div {
  width: 100px;
  height: 100px;
  background: rgba(0,0,0,0.5);
  border: 1px solid #000000;
  opacity: 0.5;
}
```

这一概念同样适用于 iOS 里的概念，比如我们可以通过 `alpha` 通道单独的去设置 `backgroudColor`、`borderColor`，它们互不影响，且有着独立的 `alpha` 通道，我们也可以通过 `opacity` 统一设置整个 view 的透明度。

但与 Web 端不一致的是，iOS 里面的 view 不光拥有独立的 `alpha` 属性，同时也是基于 CALayer，所以我们可以看到任意 UIView 对象下面都会有一个 layer 的属性，用于表明 CALayer 对象。view 的 `alpha` 属性与 layer 里面的 `opacity` 属性是一个相等的关系，需要注意的是 view 上的 `alpha` 属性是 Web 端并不具备的一个能力，所以笔者认为：在 iOS 中去说 `alpha` 时，要区分是在说 view 上的属性，还是在说颜色通道里的 `alpha`。

由于这两个词都是在描述程度，所以我们看到它们都是 CGFloat 类型：

![9alpha-opacity的区别.png](9.png)

### 转场过程中需要注意的问题和细节

说完了导航栏的转场时机和转场方式，其实大体上你已经能处理好不同样式间的转换，但还有一些细节需要你去考虑，下面我们来说说其中需要你关注的两点。

#### translucent 属性带来的布局改变

translucent 会影响导航栏组件里 ViewController 的 View 布局，这里需要大家理清 5 个 API 的使用场景：

1. `edgesForExtendedLayout`
2. `extendedLayoutIncluedsOpaqueBars`
3. `automaticallyAdjustScrollViewInsets`
4. `contentInsetAdjustmentBehavior`
5. `additionalSafeAreaInsets`

前三个 API 是 iOS 11 之前的 API，它们之间的区别和联系在 Stack Overflow 上有一个比较精彩的回答 - [Explaining difference between automaticallyAdjustsScrollViewInsets, extendedLayoutIncludesOpaqueBars, edgesForExtendedLayout in iOS7](https://stackoverflow.com/questions/18798792/explaining-difference-between-automaticallyadjustsscrollviewinsets-extendedlayo)，我在这里就不做详细阐述，总结一下它的观点就是:

如果我们先定义一个 UINavigationController，它里面包含了多个 UIViewController，每个 UIViewController 里面包含一个 UIView 对象：

* 那么 `edgesForExtendedLayout` 是为了解决 UIViewController 与 UINavigationController 的对齐问题，它会影响 UIViewController 的实际大小，例如 `edgesForExtendedLayout` 的值为 `UIRectEdgeAll` 时，UIViewController 会占据整个屏幕的大小。
* 当 UIView 是一个 UIScrollView 类或者子类时，`automaticallyAdjustsScrollViewInsets` 是为了调整这个 UIScrollView 与 UINavigationController 的对齐问题，这个属性并不会调整  UIViewController 的大小。
* 对于 UIView 是一个 UIScrollView 类或者子类且导航栏的背景色是不透明的状态时，我们会发现使用 `edgesForExtendedLayout` 来调整 UIViewController 的大小是无效的，这时候你必须使用 `extendedLayoutIncludesOpaqueBars` 来调整 UIViewController 的大小，可以认为 `extendedLayoutIncludesOpaqueBars` 是基于 `automaticallyAdjustsScrollViewInsets` 诞生的，这也是为什么经常会看到这两个 API 会同时使用。

这些调整布局的 API 背后是一套基于 `topLayoutGuide` 和 `bottomLayoutGuide` 的计算而已，在 iOS 11 后，Apple 提出了 Safe Area 的概念，将原先分裂开来的 `topLayoutGuide` 和 `bottomLayoutGuide` 整合到一个统一的 LayoutGuide 中，也就是所谓的 Safe Area，这个改变看起来似乎不是很大，但它的出现确实方便了开发者。

![10safe-area示例图.jpg](10.jpg)

如果想对 Safe Area 带来的改变有更全面的认识，十分推荐阅读 Rosberry 的工程师 Evgeny Mikhaylov 在 Medium 上的文章 [iOS Safe Area](https://medium.com/rosberryapps/ios-safe-area-ca10e919526f)，这篇文章基本涵盖了 iOS 11 中所有与 Safe Area 相关的 API 并给出了真正合理的解释。

这里只说一下 `contentInsetAdjustmentBehavior` 和 `additionalSafeAreaInsets` 两个 API。

对于 `contentInsetAdjustmentBehavior` 属性而言，它的诞生也意味着 `automaticallyAdjustsScrollViewInsets` 属性的失效，所以我们在那些已经适配了 iOS 11 的工程里能看到如下类似的代码：

```objc
if (@available(iOS 11.0, *)) {
    self.tableView.contentInsetAdjustmentBehavior = UIScrollViewContentInsetAdjustmentNever;
} else {
    self.automaticallyAdjustsScrollViewInsets = NO;
}
```

此处的代码片段只是一个示例，并不适用所有的业务场景，这里需要着重说明几个问题：

1. 关于 `contentInsetAdjustmentBehavior` 中的 `UIScrollViewContentInsetAdjustmentAutomatic` 的说明一直很“模糊”，通过 Evgeny Mikhaylov 的文章，我们可以了解到他在大多数情况下会与 `UIScrollViewContentInsetAdjustmentScrollableAxes` 一致，当且仅当满足以下所有条件时才会与 `UIScrollViewContentInsetAdjustmentAlways` 相似：
    * UIScrollView 类型的视图在水平轴方向是可滚动的，垂直轴是不可滚动的。
    * ViewController 视图里的第一个子控件是 UIScrollView 类型的视图。
    * ViewController 是 navigation 或者 tab 类型控制器的子视图控制器。
    * 启用 `automaticallyAdjustsScrollViewInsets`。

2. iOS 11 后，通过 `contentInset` 属性获取的偏移量与 iOS 10 之前的表现形式并不一致，需要获取 `adjustedContentInset` 属性才能保证与之前的 `contentInset` 属性一致，这样的改变需要我们在代码里对不同的版本进行适配。

对于 `additionalSafeAreaInsets` 而言，如果系统提供的这几种行为并不能满足我们的布局要求，开发者还可以考虑使用 `additionalSafeAreaInsets` 属性做调整，这样的设定使得开发者可以更加灵活，更加自由的调整视图的布局。

#### backIndicator 上的动画

苹果提供了许多修改导航栏组件样式的 API，有关于布局的，有关于样式的，也有关于动画的。`backIndicatorImage` 和 `backIndicatorTransitionMaskImage` 就是其中的两个 API。

`backIndicatorImage` 和 `backIndicatorTransitionMaskImage` 操作的是 NavigationBar 里返回按钮的图片，也就是下图红色圆圈所标注的区域。

![11backIndicator示例图.png](11.png)

想要成功的自定义返回按钮的图标样式，我们需要同时设置这两个 API ，从字面上来看，它们一个是返回图片本身，另一个是返回图片在转场时用到的 mask 图片，看起来不怎么难，我们写一段代码试试效果：

```objc
self.navigationController.navigationBar.backIndicatorImage = [UIImage imageNamed:@"backArrow"];
self.navigationController.navigationBar.backIndicatorTransitionMaskImage = [UIImage imageNamed:@"backArrowMask"];
```

代码里的图片如下所示：

![12mask图片示例图1.png](12.png)

也许大多数人在这里会都认为，mask 图片会遮挡住文字使其在遇到返回按钮右边缘的时候就消失。但实际的运行效果是怎么样子的呢？我们来看一下：

![13mask动态效果图1.gif](13.gif)

在上面的图片中，我们可以看到返回按钮的文字从返回按钮的图片下面穿过并且文字被图片所遮挡，这种动画看起来十分奇怪，这是无法接受的。我们需要做点修改：

```objc
self.navigationController.navigationBar.backIndicatorImage = [UIImage imageNamed:@"backArrow"];
self.navigationController.navigationBar.backIndicatorTransitionMaskImage = [UIImage imageNamed:@"backArrow"];
```

这一次我们将 `backIndicatorTransitionMaskImage` 改为 indicatorImage 所用的图片。

![14mask图片示例图2.png](14.png)

到这里，可能大多数人都会好奇，这代码也能行？让我们看下它实际的效果：

![15mask动态效果图2.gif](15.gif)

在上面的图中，我们看到文字在到达图片的右边缘时就从下方穿过并被完全遮盖住了，这种动画效果虽然比上面好一些，但仍然有改进的空间，不过这里我们先不继续优化了，我们先来讨论一下它们背后的运作原理。

iOS 系统会将 indicatorImage 中不透明的颜色绘制成返回按钮的图标， indicatorTransitionMaskImage 与 indicatorImage 的作用不同。indicatorTransitionMaskImage 将自身不透明的区域像 mask 一样作用在 indicatorImage 上，这样就保证了返回按钮中的文字像左移动时，文字只出现在被 mask 的区域，也就是 indicatorTransitionMaskImage 中不透明的区域。

掌握了原理，我们来解释下刚才的两种现象：

在第一种实现中，我们提供的 indicatorTransitionMaskImage 覆盖了整个返回按钮的图标，所以我们在转场过程中可以清晰的看到返回按钮的文字。

在第二种实现中，我们使用 indicatorImage 作为 indicatorTransitionMaskImage，记住文字是只能出现在 indicatorTransitionMaskImage 里不透明的区域，所以显然返回按钮中的文字会在图标的最右边就已经被遮挡住了，因为那片区域是透明的。

那么前面提到的进一步优化指的是什么呢？

让我们来看一下下面这个示例图，为了更好的区分，我们将 indicatorTransitionMaskImage 用红色进行标注。黑色仍然是 indicatorImage。

![16mask图片示例图3.jpg](16.jpg)

按照刚才介绍的原理，我们应该可以理解，现在文字只会出现在红色区域，那么它的实际效果是什么样子的呢，我们可以看下图：

![17mask动态效果图3.gif](17.gif)

现在，一个完美的返回动画，诞生啦！

> 此节所用的部分效果图出自 Ray Wenderlich 的文章 [UIAppearance Tutorial: Getting Started](https://www.raywenderlich.com/1625-uiappearance-tutorial-getting-started)

## 导航栏的跳转或许可以这么玩儿

前两章的铺垫就是为了这一章的内容，所以现在让我们开始今天的大餐吧。

### 这样真的好么？

刚才我们说了两个页面间 NavigationBar 的样式变化需要在各自的 `viewWillAppear:` 和 `viewWillDisappear:` 中进行设置。那么问题就来了：这样的设置会带来什么问题呢？

试想一下，当我们的页面会跳到不同的地方时，我们是不是要在 `viewWillAppear:` 和 `viewWillDisappear:` 方法里面写上一堆的判断呢？如果应用里还有 router 系统的话，那么页面间的跳转将变得更加不可预知，这时候又该如何在 `viewWillAppear:` 和 `viewWillDisappear:` 里做判断呢？

现在我们的问题就来了，如何让导航栏的转场更加灵活且相互独立呢？

常见的解决方案如下所示：

1. 重新实现一个类似 UINavigationController 的容器类视图管理器，这个容器类视图管理器做好不同 ViewController 间的导航栏样式转换工作，而每个 ViewController 只需要关心自身的样式即可。
![18常见的导航栏转场方案1示例图.png](18.png)

2. 将系统原有导航栏的背景设置为透明色，同时在每个 ViewController 上添加一个 View 或者 NavigationBar 来充当我们实际看到的导航栏，每个 ViewController 同样只需要关心自身的样式即可。
![19常见的导航栏转场方案2示例图.png](19.png)

3. 在转场的过程中隐藏原有的导航栏并添加假的 NavigationBar，当转场结束后删除假的 NavigationBar 并恢复原有的导航栏，这一过程可以通过 Swizzle 的方式完成，而每个 ViewController 只需要关心自身的样式即可。
![20常见的导航栏转场方案3示例图.png](20.png)

这三种方案各有优劣，我们在网上也可以看到很多关于它们的讨论。

例如方案一，虽然看起来工作量大且难度高，但是这个工作一旦完成，我们就会将处理导航栏转场的主动权牢牢抓在手里。但这个方案的一个弊端就是，如果苹果修改了导航栏的整体风格，就好比 iOS 11 的大标题特效，那么工作量就来了。

对于方案二而言，虽然看起来简单易用，但这需要一个良好的继承关系，如果整个工程里的继承关系混乱或者是历史包袱比较重，后续的维护就像“打补丁”一样，另外这个方案也需要良好的团队代码规范和完善的技术文档来做辅助。

对于方案三而言，它不需要所谓的继承关系，使用起来也相对简单，这对于那些继承关系和历史包袱比较重的工程而言，这一个不错的解决方案，但在解决 Bug 的时候，Swizzle 这种方式无疑会增加解决问题的时间成本和学习成本。

### 我们的解决方案

在美团 App 的早期，各个业务方都想充分利用导航栏的能力，但对于导航栏的状态维护缺乏理解与关注，随着业务方的增加和代码量的上升，与导航栏相关的问题逐渐暴露出来，此时我们才意识到这个问题的严重性。

大型 App 的导航栏问题就像一个典型的“公地悲剧”问题。在软件行业，公用代码的所有权可以被视作“公地”，因为不注重长期需求而容易遭到消耗。如果开发人员倾向于交付“价值”，而以可维护性和可理解性为代价，那么这个问题就特别普遍了。如果是这种情况，每次代码修改将大大减少其总体质量，最终导致软件的不可维护。

所以解决这个问题的核心在于：明确公用代码的所有权，并在开发期施加约束。

明确公用代码的所有权，可以理解为将导航栏相关的组件抽离成一个单独的组件，并交由特定的团队维护。而在开发期施加约束，则意味着我们要提供一套完整的解决方案让各个业务方遵守。

这一节我们会以美团内部的解决方案为例，讲解如何实现一个流畅的导航栏跳转过程和相关使用方法。

#### 设计理念

使用者只用关心当前 ViewController 的 NavigationBar 样式，而不用在 push 或者 pop 的时候去处理 NavigationBar 样式。

举个例子来说，当从 A 页面 push 到 B 页面的时候，转场库会保存 A 页面的导航栏样式，当 pop 回去后就会还原成以前的样式，因此我们不用考虑 pop 后导航栏样式会改变的情况，同时我们也不必考虑 push 后的情况，因为这个是页面 B 本身需要考虑的。

#### 使用方法

转场库的使用十分简单，我们不需要 import 任何头文件，因为它在底层通过 Method Swizzling 进行了处理，只需要在使用的时候遵循下面 4 点即可：

* 当需要改变导航栏样式的时候，在视图控制器的 `viewDidLoad` 或者 `viewWillAppear:` 方法里去设置导航栏样式。
* 用 `setBackgroundImage:forBarMetrics:` 方法和 `shadowImage` 属性去修改导航栏的背景样式。
* 不要在 `viewWillDisappear:` 里添加针对导航栏样式修改的代码。
* 不要随意修改 translucent 属性，包括隐式的修改和显示的修改。

> 隐式修改是指使用 `setBackgroundImage:forBarMetrics:` 方法时，如果 image 里的像素点没有 `alpha` 通道或者 `alpha` 全部等于 1 会使得 `translucent` 变为 NO 或者 nil。

### 基本原理

以上，我们讲完了设计理念和使用方法，那么我们来看看美团的转场库到底做了什么？

从大方向上来看，美团使用的是前面所说的第三种方案，不过它也有一些自己独特的地方，为了更好的让大家理解整个过程，我们设计这样一个场景，从页面 A push 到页面 B，结合之前探讨过的方法调用顺序，我们可以知道几个核心方法的调用顺序大致如下：

1. 页面 A 的 `pushViewController:animated:`
2. 页面 B 的 `viewDidLoad` or `viewWillAppear:`
3. 页面 B 的 `viewWillLayoutSubviews`
4. 页面 B 的 `viewDidAppear:`

在 push 过程的开始，转场库会在页面 A 自身的 view 上添加一个与导航栏一模一样的 NavigationBar 并将真的导航栏隐藏。之后这个假的导航栏会一直存在页面 A 上，用于保留 A 离开时的导航栏样式。

等到页面 B 调用 `viewDidLoad` 或者 `viewWillAppear:` 的时候，开发者在这里自行设置真的导航栏样式。转场库在这里会对页面布局做一些修正和辅助操作，但不会影响导航栏的样式。

等到页面 B 调用 `viewWillLayoutSubviews` 的时候，转场库会在页面 B 自身的 view 上添加一个与真的导航栏一模一样的 NavigationBar，同时将真的导航栏隐藏。此时不论真的导航栏，还是假的导航栏都已经与 `viewDidLoad` 或者 `viewWillAppear:` 里设置的一样的。

> 当然，这一步也可以放在 `viewWillAppear:` 里并在 dispatch main queue 的下一个 runloop 中处理。

等到页面 B 调用 `viewDidAppear:` 的时候，转场库会将假的导航栏样式设置到真的导航栏中，并将假的导航栏从视图层级中移除，最终将真的导航栏显示出来。

为了让大家更好地理解上面的内容，请参考下图：

![21KMNavigationBarTransiton的原理图-push流程.png](21.png)

说完了 push 过程，我们再来说一下从页面 B pop 回页面 A 的过程，几个核心方法的调用顺序如下：

1. 页面 B 的 `popViewControllerAnimated:`
2. 页面 A 的 `viewWillAppear:`
3. 页面 A 的 `viewDidAppear:`

在 pop 过程的开始，转场库会在页面 B 自身的 view 上添加一个与导航栏一模一样的 NavigationBar 并将真的导航栏隐藏，虽然这个假的导航栏会一直存在于页面 B 上，但它自身会随着页面 B 的 `dealloc` 而消亡。

等到页面 A 调用 `viewWillAppear:` 的时候，开发者在这里自行设置真的导航栏样式。当然我们也可以不设置，因为这时候页面 A 还持有一个假的导航栏，这里还保留着我们之前在 `viewDidLoad` 里写的导航栏样式。

等到页面 A 调用 `viewDidAppear:` 的时候，转场库会将假的导航栏样式设置到真的导航栏中，并将假的导航栏从视图层级中移除，最终将真的导航栏显示出来。

同样，我们可以参考下面的图来理解上面所说的内容：

![22KMNavigationBarTransiton的原理图-pop流程.png](22.png)

现在，大家应该对我们美团的解决方案有了一定的认识，但在实际开发过程中，还需要考虑一些布局和适配的问题。

## 最佳实践

在维护这套转场方案的时间里，我们总结了一些此类方案的最佳实践。

### 判断导航栏问题的基本准则

如果发现导航栏在转场过程中出现了样式错乱，可以遵循以下几点基本原则：

* 检查相应 ViewController 里是否有修改其他 ViewController 导航栏样式的行为，如果有，请做调整。
* 保证所有对导航栏样式变化的操作出现在 `viewDidLoad` 和 `viewWillAppear:` 中，如果在 `viewWillDisappear:` 等方法里出现了对导航栏的样式修改的操作，如果有，请做调整。
* 检查是否有改动 `translucent` 属性，包括显示修改和隐式修改，如果有，请做调整。

### 只关心当前页面的样式

永远记住每个 ViewController 只用关心自己的样式，设置的时机点在 `viewWillAppear:` 或者 `viewDidLoad` 里。

### 透明样式导航栏的正确设置方法

如果需要一个透明效果的导航栏，可以使用如下代码实现：

```objc
[self.navigationController.navigationBar setBackgroundImage:[UIImage new] forBarMetrics:UIBarMetricsDefault];
self.navigationController.navigationBar.shadowImage = [UIImage new]; 
```

### 导航栏的颜色渐变效果

如果需要导航栏实现随滚动改变整体 `alpha` 值的效果，可以通过改变 `setBackgroundImage:forBarMetrics:` 方法里 image 的 `alpha` 值来达到目标，这里一般是使用监听 `scrollView.contentOffset` 的手段来做。请避免直接修改 NavigationBar 的 `alpha` 值。

还有一点需要注意的是，在页面转场的过程中，也会触发 `contentOffset` 的变化，所以请尽量在 disappear 的时候取消监听。否则会容易出现导航栏透明度的变化。

### 导航栏背景图片的规范

请避免背景图里的像素点没有 `alpha` 通道或者 `alpha` 全部等于 1，容易触发 `translucent` 的隐式改变。

### 如果真的要隐藏导航栏

如果我们需要隐藏导航栏，请保证所有的 ViewController 能坚持如下原则：

1. 每个 ViewController 只需要关心当前页面下的导航栏是否被隐藏。
2. 在 `viewWillAppear:` 中，统一设置导航栏的隐藏状态。
3. 使用 `setNavigationBarHidden:animated:` 方法，而不是 `setNavigationBarHidden:`。

### 转场动画与导航栏隐藏动画的一致性

如果在转场的过程中还会显示或者隐藏导航栏的话，请保证两个方法的动画参数一致。

```objc
- (void)viewWillAppear:(BOOL)animated{
    [self.navigationController setNavigationBarHidden:YES animated:animated];
}
```

> `viewWillAppear:` 里的 animated 参数是受 push 和 pop 方法里 animated 参数影响。

### 导航栏固有的系统问题

目前已知的有两个系统问题如下：

1. 当前后两个 ViewController 的导航栏都处于隐藏状态，然后在后一个 ViewController 中使用返回手势 pop 到一半时取消，再连续 push 多个页面时会造成导航栏的 Stack 混乱或者 Crash。
2. 当页面的层级结构大体如下所示时，在红色导航栏的 Stack 中，返回手势会大概率的出现跨层级的跳转，多次后会导致整个导航栏的 Stack 错乱或者 Crash。

![23引发导航栏栈错乱的视图层级.png](23.png)

### 导航栏内置组件的布局规范

导航栏里的组件布局在 iOS 11 后发生了改变，原有的一些解决方案已经失效，这些内容不在本篇文章的讨论范围之内，推荐阅读[UIBarButtonItem 在 iOS 11 上的改变及应对方案](http://sketchk.xyz/2018/02/23/How-to-make-your-UIBarButtonItem-perfect-match-in-iOS/)，这篇文章详细的解释了 iOS 11 里的变化和可行的应对方案。

## 总结

本文涉及内容较多，从 iOS 系统下的导航栏概念到大型应用里的最佳实践，这里我们总结一下整篇文章的核心内容：

* 理解导航栏组件的结构和相关方法的生命周期。
  * 导航栏组件的结构留有 MVC 架构的影子，在解决问题时，要去相应的层级处理。
  * 转场问题的关键点是方法的调用顺序，所以了解生命周期是解决此类问题的基础。
* 状态管理，转换时机和样式变化是导航栏里常见问题的三种表现形式，遇到实际问题时需要区分清楚。
  * 状态管理要坚持“谁修改，谁复原”的原则。
  * 转换时机的设定要做到连续可执行。
  * 样式变化的核心点是导航栏的显示与否与颜色变化。
* 为了更好的配合大型应用里的路由系统，导航栏转场的常见解决方案有三种，各有利弊，需要根据自身的业务场景和历史包袱做取舍。
  * 解决方案1：自定义导航栏组件。
  * 解决方案2：在原有导航栏组件里添加 Fake Bar。
  * 解决方案3：在导航栏转场过程中添加 Fake Bar。
* 美团在实际开发过程中采用了第三种方案，并给出了适合美团 App 的最佳实践。

> 特别感谢[莫洲骐](https://github.com/MoZhouqi)在此项目里的贡献与付出。

## 参考链接

* [UIAppearance Tutorial: Getting Started](https://www.raywenderlich.com/1625-uiappearance-tutorial-getting-started)
* [KMNavigationBarTransition](https://github.com/MoZhouqi/KMNavigationBarTransition)

## 作者简介

思琦，美团点评 iOS 工程师。2016 年加入美团，负责美团平台的业务开发及 UI 组件的维护工作。

## 招聘

美团平台诚招 iOS、Android、FE 高级/资深工程师和技术专家，Base 北京、上海、成都，欢迎有兴趣的同学投递简历到zhangsiqi04@meituan.com