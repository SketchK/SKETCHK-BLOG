---
title: Dark Mode 适配工作小指南 
comments: true
date: 2019-06-21 20:08:32
updated:
tags:
    - iOS
    - WWDC
categories:
    - iOS
---

Dark Mode 作为 iOS 13 的新特性之一，为 App 的呈现方式带来了更多的可能，但也给开发者带来了大量的适配工作。

<!-- more -->

## 总览

Dark Mode 作为 iOS 13 的新特性之一，为 App 的呈现方式带来了更多的可能，但也给开发者带来了大量的适配工作。

在适配 Dark Mode 的过程中，开发者只需要思考清楚下面三个问题即可：

1. 如何获取当前的状态和信息 ？（Light Mode 还是 Dark Mode）
2. 明确适配后的效果遵循哪种设计规范？（ Apple 的设计规范，还是自定义的设计规范）
3. 确定哪种元素需要进行适配？（颜色，模糊效果，设计素材，还是其他元素）

## 1. 判断当前的显示状态

在 iOS 13 中，由于 UIView 和 UIViewController 这些基类已经遵循了 UITraitEnvironment 协议，我们可以通过 UITraitCollection 判断 App 当前的 Apparance 状态。代码如下所示：

Swift 的代码如下：

```swift
let isDark = UITraitCollection.current.userInterfaceStyle == .dark
let realColor = UIColor.systemBackground.resolvedColor(with: traitCollection)
let realImage = UIImage(named: "Header")?.imageAsset?.image(with: traitCollection)
```

Objective-C 的代码如下：

```objc
BOOL isDark = [UITraitCollection currentTraitCollection].userInterfaceStyle == UIUserInterfaceStyleDark? YES: NO;
UIColor *realColor = [[UIColor systemBackgroundColor] resolvedColorWithTraitCollection:traitCollection];
UIImage *image = [[[UIImage imageNamed:@"Header"] imageAsset] imageWithConfiguration:traitCollection]
```

需要注意的是，上方代码中我们调用了 [UITraitCollection currentTraitCollection]，但 UITraitCollection 并不是以单例的方式存在，每个 view 和 ViewController 都有一个属于自身的 UITraitCollection 类型的属性。

所以我们也完全可以通过 `traitCollection.` 或者 `self.traitCollection` 的方式获取此属性，

不过，也不是所有方法执行前，UIKit 都会帮我们做把 UITraitCollection.current 设置为 self.traitCollection，只有在以下这些方法中会执行此操作：

| UIView  | UIViewController | UIPresentationController |
| - | - | -- |
| draw() |  |
| layoutSubview()  | viewWillLayoutSubviews()<br />viewDidLayoutSubviews() | containerViewWillLayoutSubviews()<br />containerViewDidLayoutSubviews() |
| traitCollectionDidChange()<br />tintColorDidChange() | traitCollectionDidChange() | traitCollectionDidChange() |

## 2. 适配后的效果遵循哪种设计规范？

在实际写代码前，我们还需更深刻的理解适配 Dark Mode 这项工作背后的含义。

虽然 Apple 给出了了许多好用的 API 帮助开发者迅速完成 Dark Mode 的适配，但这是你应该用的 API 么？例如系统给出了 `systemBlue`, `systemGray` 等 API 到底意味着什么？

这里需要明确的是 `systemBlue` 这些 API 本质上在 Apple 的 Human Interface Guidelines 下制定的 Dark Mode 配色，并不代表这些配色就应该无脑的用在你的 App 上。

Apple 原生的 App 大多数是以白色或浅色系为基调，所以在此设计规范下，它提供的 API 下可以很好的做 Light Mode 和 Dark Mode 的切换，且 App 的视觉效果十分惊艳。

但如果你的 App 不是浅色系，如果像淘宝的橙色系，饿了么的蓝色系，甚至多闪的暗色系，他们如果直接用 Apple 的设计规范，从视觉体验上来说，可能就显得有点不那么匹配了，而且 Apple 提供给你的这些 API 就显得有点多余。

所以在动手前你需要明确自己的需求，Apple 提供的这些设计规范是否符合 App 当前的调性，或者设计师和你的预期么？

如果是的，请大胆的使用 Apple 提供的 API 吧。关于这些 API 代表的样式完全可以在 Apple 提供的 [Human Interface Guidelines](https://developer.apple.com/design/human-interface-guidelines/) 和 [Resource](https://developer.apple.com/design/resources/) 中找到。

如果不是，请结合自身 App 的设计规范，抽象出一个中间层或者 SDK 来定义这些变化。

### 2.1 与设计规范相关 API 的解读

与 Dark Mode 相关的 API 有以下几个特点
* 与颜色或者样式相关的 API，大体可以划分为三个类型，adaptable styles, light styles, dark styles，adaptable styles 包含 system 相关的字眼，可以根据系统的变化自动切换颜色，而 light 和 dark 类型的会以自身做结尾，不会根据系统的变化切换颜色
* Activity Indicator 和 Status Bar 的 API 与 Dark Mode 特性 相互照应，更明确了 API 的作用范围。例如 ` UIActivityIndicatorView` 的 style 属性从 `gray` , `white`, `whiteLarge` 变成了 `large` 和 `medium`，`UIStatusBar` 的 style 属性从 `default` 和 `lightContent` 变成了 `default` 和 `lightContent` 和 `darkContent`，其中 `default` 样式会根据当前的模式展示不同的颜色


## 3. 明确哪种元素需要进行适配？

这里的核心问题就是在 Appearance 切换后，哪些元素需要适配？

大部分人的第一直觉会认为是控件颜色，例如控件的背景色和控件的内容颜色，那还有别的元素么？

显然是有的，需要考虑的类型大体有两种，一种是图片素材，一种是模糊效果，我们先简单阐述下这两个元素适配的必要性

* 图片素材的适配必要性显而易见，例如 tab bar 或者 navigation bar 里的 icon，在切换为 dark mode 后，那些原先就是黑色的 icon 在 dark mode 下会变得难以辨识。

* 如果说图片素材要解决的是辨识度问题，那么模糊效果需要解决的就是协调性问题，如果在 dark 模式下，模糊效果整体过于泛白就会显得有些格格不入，进而影响用户体验。

上面说明了必要性，我们来说说针对不同元素的适配方案

### 3.1 背景色类型的适配

如果系统颜色符合你的预期，通过下面的语句就可以实现 dark mode 的适配

```swift
self.view.backgroundColor = .systemBackground
```

如果系统颜色不符合你的预期，通过下面的语句来实现 dark mode 的适配

```swift
self.textView.textColor = .myDynamicColor

extension UIColor {
class var myDynamicColor: UIColor {
get {
.init(dynamicProvider: { (traitCollection) -> UIColor in
if traitCollection.userInterfaceStyle == .dark {
return UIColor(displayP3Red: 0, green: 0, blue: 0, alpha: 1)
} else {
return UIColor(displayP3Red: 255, green: 255, blue: 255, alpha: 1)
}
})
}
}
}
```

### 3.2 模糊效果的适配

通常状态下，App 内的组件都是直接使用系统提供的 UIBlurEffectView 组件，通过下面的代码就可以快速完成 Dark Mode 的匹配，当然 style 里面的 `system` 开头的枚举值需要你根据自身的情况进行选择。

```swift
let blurView = UIVisualEffectView(effect: UIBlurEffect.init(style: .systemMaterial))
```

如果你的模糊控件是自行开发的，那么你就需要根据自身 App 的设计规范重新开发一套 API 来满足使用。


### 3.3 图片素材的适配

我们可以利用 xcassets 中为图片控件新增的 Apperance 属性，分别设置两种模式下所使用到的图片，例如在深色模式下展示日落的照片，而在普通模式下展示日出的照片。

![01](01.jpg)


## 4 总结

恭喜你，至此你已经掌握了适配 Dark Mode 的所有基本知识点，剩下需要做的就是实战了。

如果你还想更深入理解 dark mode 的相关信息，可以观看 [WWDC 2019 Session 214: Implementing Dark Mode on iOS](https://developer.apple.com/videos/play/wwdc2019/214/)


