---
title: UIScrollView 的偏移问题
tags:
  - iOS
author: 帕帕
categories: 技术
thumbnail: >-
  https://images.unsplash.com/photo-1514464750060-00e6e34c8b8c?ixlib=rb-0.3.5&ixid=eyJhcHBfaWQiOjEyMDd9&s=3749c47dd7beec20102c6b32fc19833a&auto=format&fit=crop&w=160&q=100
date: 2019-09-27 17:40:01
---


## 被控制的 UIScrollView

在 UIViewController 中有个属性：`automaticallyAdjustsScrollViewInsets`，这个属性是用来控制 UIScrollView 的偏移行为的。

> |                                                              |
> | ------------------------------------------------------------ |
> | The default value of this property is `true`, which lets container view controllers know that they should adjust the scroll view insets of this view controller’s view to account for screen areas consumed by a status bar, search bar, navigation bar, toolbar, or tab bar. Set this property to `false` if your view controller implementation manages its own scroll view inset adjustments. |
> |                                                              |

官方文档的意思当在 UIViewController 上添加  UIScrollView 的时候，会根据当前页面的 status bar、 search bar、navigation bar、toolbar 或 tab bar 来修改 UIScrollView 的内容区域。但是这个阶段的 UIViewController 比较蠢，不管任何情况下都会修改 UIScrollView 的偏移量。

比如我们现在有个 UINavigationController，然后添加一个 UIScrollView，然后在 UIScrollView 上面添加一个红色的方块，代码如下：

```Swift
let scrollView = UIScrollView()
scrollView.backgroundColor = .blue
scrollView.translatesAutoresizingMaskIntoConstraints = false
// 这里强制设置 contentSize 只是为了让 scrollView 能滚动起来
scrollView.contentSize = CGSize.init(width: view.frame.size.width, height: 1000)
view.addSubview(scrollView)
// ⚠️ 这里是直接跟 view 的 topAnchor 产生约束
scrollView.topAnchor.constraint(equalTo: view.topAnchor).isActive = true
scrollView.bottomAnchor.constraint(equalTo: view.bottomAnchor).isActive = true
scrollView.leftAnchor.constraint(equalTo: view.leftAnchor).isActive = true
scrollView.rightAnchor.constraint(equalTo: view.rightAnchor).isActive = true

let redView = UIView()
redView.backgroundColor = .red
redView.translatesAutoresizingMaskIntoConstraints = false
scrollView.addSubview(redView)
redView.centerXAnchor.constraint(equalTo: scrollView.centerXAnchor).isActive = true
redView.topAnchor.constraint(equalTo: scrollView.topAnchor).isActive = true
redView.widthAnchor.constraint(equalToConstant: 100).isActive = true
redView.heightAnchor.constraint(equalToConstant: 100).isActive = true
```

<img src="https://i.imgur.com/OnQDx7E.png" alt="图一：iOS 10 模拟器效果" style="zoom:50%;" />



当我们把 UIScrollView 的 topAnchor 修改为跟 UIViewController 的 topLayoutGuide 发生约束：

```Swift
scrollView.topAnchor.constraint(equalTo: topLayoutGuide.bottomAnchor).isActive = true
```



<img src="https://i.imgur.com/dwklB0X.png" alt="图二：iOS 10 模拟器效果" style="zoom:50%;" />

我们发现最终的效果是 UIScrollView 也发生了偏移，而且这个偏移是根据你顶部的 status bar 和 navigation bar 的高度来决定的。所以在 iOS 10 及以下的版本的时候，添加到 UIViewController 的 UIScrollView 总是会发生偏移。但是你可以通过把刚才说的那个属性 `automaticallyAdjustsScrollViewInsets`设置成 false，UIViewController 就不会让你的 UIScrollView 发生偏移。但是这个属性会影响到所有添加到 UIViewController 上的 UIScrollView，如果有些想要发生偏移，有些不想发生偏移的时候就需要把 `automaticallyAdjustsScrollViewInsets`设置成 false，然后通过代码单独去为每个 UIScrollView 设置不同的 contentInset。

这种被控制的生活很不是滋味，于是随着 iOS 系统来到 11 之后，UIScrollView 终于夺回了自己的偏移控制权。UIViewController 的`automaticallyAdjustsScrollViewInsets`终于被废弃了，取而代之的是 UIScrollView 自己的`contentInsetAdjustmentBehavior`。

## 自由的 UIScrollView

>This property specifies how the safe area insets are used to modify the content area of the scroll view. The default value of this property is [UIScrollView.ContentInsetAdjustmentBehavior.automatic](apple-reference-documentation://hs7dxiWRRh).

UIScrollView 的`contentInsetAdjustmentBehavior`的默认行为是`automatic`，这和 iOS 10 默认行为的最大区别就是它会判断 UIScrollView 是被添加到哪个位置，然后根据这个位置来判断是否需要修改 UIScrollView 的偏移量。

还是拿上面图二的情况来讲，在 iOS 11 及 iOS 11 之后，我们还是照样只修改  UIScrollView 的 topAnchor：

```swift
scrollView.topAnchor.constraint(equalTo: view.safeAreaLayoutGuide.topAnchor).isActive = true
```

<img src="https://i.imgur.com/co1D7xr.png" alt="图三：iOS 12 模拟器效果" style="zoom:50%;" />

此时我们发现 UIScrollView 并没有发生偏移，这也是因为 iOS 11 之后引入来 safeArea 的概念之后带来的 UI 方面的优化。

`contentInsetAdjustmentBehavior`还有两个值，其中`always`对应了`automaticallyAdjustsScrollViewInsets`的`true`,`never`对应了`automaticallyAdjustsScrollViewInsets`的`false`。

至于`scrollableAxes`，它其实就是根据 UIScrollView 的滚动方向来决定在哪个轴上使用 sa feArea。

通过`contentInsetAdjustmentBehavior`我们就可以为 UIViewController 上的每一个 UIScrollView 定制它们的偏移行为。