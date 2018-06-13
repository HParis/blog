---
title: Swift High-Performance Tip 3：@objc 和 dynamic
author: 帕帕
date: 2018-05-24 18:48:01 +0800
categories: 技术
tags: [iOS, Swift]
thumbnail: https://images.unsplash.com/photo-1507358522600-9f71e620c44e?ixlib=rb-0.3.5&ixid=eyJhcHBfaWQiOjEyMDd9&s=9606de2cffd6c619093871ef2d1c0e6f&auto=format&fit=crop&w=160&q=100
---

### @objc

@objc 的作用是为了让 Objective-C 能够调用 Swift 的代码。其中的关键是 @objc 会生成一段 thunk 代码，Objective-C 通过这段 thunk 代码来间接调用 Swift 代码。如果是 Swift 来调用被 @objc 修饰的方法的时候，此时是不需要经过 thunk 代码就能直接调用的。

所以我们可以想象，如果方法变得复杂或者被 @objc 修饰的方法数量变得越来越多会发生什么事？答案就是 thunk 代码变得越来越多，最后会导致我们的包大小也变得越来越大。并且动态链接器（dynamic linker）还需要整理这些 thunk 代码，最后导致加载时间也会变得越来越长。

在 Swift3 的时候，编译器会推断出你的方法不是 Swift 专用的（比如有元组、结构体），就会默认给你的方法增加 @objc 的修饰。这种方式就导致了在 Swift3 的时候，会生成大量的 thunk 代码，并且这其中的大部分代码都不会被使用。所以 Swift4 默认是不做 @objc 的推断，只有我们手动添加了 @objc 之后，Objective-C 才能调用我们的 Swift 代码。 

### dynamic

Swift 的方法是通过 vtable 来调用的，使用 vtable 要比 Objective-C 的 runtime 更高效。

而使用 dynamic 来修饰的方法，代表这个方法是可以被动态调用的。而由于目前 Swfit 还没有实现自己的 runtime 机制，所以动态调用只能够在 Objective-C 去调用。在  Swift4 使用 dynamic 修饰一个方法的时候，编译器会要求你还需要使用 @objc 去修饰。这是为了明确的告诉编译器这个方法是由 Objective-C 的 runtime 来调用的，同时也是为了兼容以后可能会出现的 Swift runtime 机制。

由于目前使用 @objc dynamic 修饰的方法并不在 Swift 实例对象的 vtable 里面，所以 Swift 来调用该方法的时候依旧需要通过 thunk 代码来调用。

### 总结

![此图出自 https://swiftunboxed.com/interop/objc-dynamic/](https://swiftunboxed.com/images/native-objc-dynamic.png)

通过上图我们知道：

> 除非明确的知道会在 Objective-C 中调用这段代码，否则别使用 @objc；
> 除非明确的知道该方法需要被 Objective-C 的 runtime 动态调用，否则别使用 @objc dynamic。


---

**参考文献**

1. https://swiftunboxed.com/interop/objc-dynamic/

2. https://github.com/apple/swift-evolution/blob/master/proposals/0160-objc-inference.md


