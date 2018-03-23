---
title: 如何在 iOS 上自定义 React Native Component
subtitle: 最重要的就是后面三句话
author: 帕帕
date: 2018-03-19 17:22:09
categories: 技术
tags: [iOS, RN]
thumbnail: https://images.unsplash.com/photo-1508921234172-b68ed335b3e6?ixlib=rb-0.3.5&ixid=eyJhcHBfaWQiOjEyMDd9&s=92e40b3819e4c173debf1500f27c9b60&auto=format&fit=crop&w=160&q=60
---

当我们要在 iOS 端实现一个 React Native 可用的 Component，比如：

```ReactNative
<MapView onRegionChange={(event) => {}} zoomLevel={2} />
```

那么我们基本上就是要解决下面这三个问题：

* 如何把 iOS 上的 UI 暴露给 React Native 端？
* 如何在 React Native 给 iOS 的 UI 传值？
* 如何在 React Native 中响应 iOS 的事件？

> 这三个问题可以在[官方文档](https://facebook.github.io/react-native/docs/native-components-ios.html)找到答案。
    
## 如何把 iOS 上的 UI 暴露给 React Native 端

首先你需要创建一个继承自 `RCTViewManager` 的子类：

```iOS
// RNTMapManager.m
#import <MapKit/MapKit.h>
#import <React/RCTViewManager.h>

// 继承 RCTViewManager
@interface RNTMapManager : RCTViewManager
@end


@implementation RNTMapManager

// 调用 RCT_EXPORT_MODULE 暴露该类的名字给 React Native 使用。如果你想自定义
// 暴露给 React Native 的名字时，你需要 RCT_EXPORT_MODULE(YOUR_CUSTOM_NAME)。
RCT_EXPORT_MODULE()

// 由于 RCTViewManager 是 NSObject，所以这里必须需要实现该方法来告诉
// React Native 去使用哪个 UIView
- (UIView *)view
{
  return [MKMapView new];
}

@end
```

这样我们就可以在 React Native 使用 MapView 了：

```ReactNative
// MapView.js

import { requireNativeComponent } from 'react-native';

// requireNativeComponent 会自动把 iOS 上的 RNTMapManager 解析成 RNTMap。
// 如果去掉 iOS 上的 Manager 后缀会有什么影响？嗯，没有任何影响。
module.exports = requireNativeComponent('RNTMap', null);


// MyApp.js
import MapView from './MapView.js';

...

render() {
  return <MapView />;
}
```

## 如何在 React Native 给 iOS 的 UI 传值

如果需要传值给 iOS 上的 UI，那么需要使用另外一个宏：

```iOS
RCT_EXPORT_VIEW_PROPERTY(zoomEnabled, BOOL)
```

这时候就可以在 React Native 上使用了：

```ReactNative
<MapView zoomEnable={true} />
```

这里需要注意的是，`RCT_EXPORT_VIEW_PROPERTY` 所暴露的属性必须是之前我们说的 UIView（即继承于 `RCTViewManager` 并通过 `- (UIView *)view;` 返回的 View）已经存在的属性。

除了上面的宏 `RCT_EXPORT_VIEW_PROPERTY` 可以暴露属性给 React Native 使用之外还有下面 5 种（这里先挖个坑，回头研究一下再说说下面五种的作用和区别）：

* RCT_REMAP_VIEW_PROPERTY
* RCT_CUSTOM_VIEW_PROPERTY
* RCT_EXPORT_SHADOW_PROPERTY
* RCT_REMAP_SHADOW_PROPERTY
* RCT_CUSTOM_SHADOW_PROPERTY

## 如何在 React Native 中响应 iOS 的事件

要想在 React Native 中响应 iOS 的事件，只需要暴露用 `RCTBubblingEventBlock` 或 `RCTDirectEventBlock` 定义的属性即可，代码如下：

```iOS
// RNTMapView.h
#import <MapKit/MapKit.h>
#import <React/RCTComponent.h>

// 由于 MKMapView 没有任何 `RCTBubblingEventBlock` 或 `RCTDirectEventBlock` 所定义的
// 属性，所以这里需要定义 MKMapView 的子类 RNTMapView
@interface RNTMapView: MKMapView

// 需要暴露给 React Native 的事件属性
@property (nonatomic, copy) RCTBubblingEventBlock onRegionChange;

@end


// RNTMapView.m
#import "RNTMapView.h"

@implementation RNTMapView

@end
```

然后我们需要在 `RCTViewManager` 中暴露 `onRegionChange` 给 React Native 使用：

```iOS
// RNTMapManager.m
#import <MapKit/MapKit.h>
#import <React/RCTViewManager.h>

#import "RNTMapView.h"

@interface RNTMapManager : RCTViewManager <MKMapViewDelegate>
@end

@implementation RNTMapManager

RCT_EXPORT_MODULE()
// 暴露 RNTMapView 中的 `onRegionChange` 属性
RCT_EXPORT_VIEW_PROPERTY(onRegionChange, RCTBubblingEventBlock)

- (UIView *)view {
    // 这里需要返回 RNTMapView 而不是 MKMapView
    return [RNTMapView new];
}

@end
```

> 重要的事说三遍：
> 使用 `RCTBubblingEventBlock` 或 `RCTDirectEventBlock` 所定义的事件都必须加上前缀 `on`，否则 React Native 无法接收到事件
> 使用 `RCTBubblingEventBlock` 或 `RCTDirectEventBlock` 所定义的事件都必须加上前缀 `on`，否则 React Native 无法接收到事件
> 使用 `RCTBubblingEventBlock` 或 `RCTDirectEventBlock` 所定义的事件都必须加上前缀 `on`，否则 React Native 无法接收到事件


