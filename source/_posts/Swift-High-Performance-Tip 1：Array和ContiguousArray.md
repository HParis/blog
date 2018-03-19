---
layout: post 
title: Swift High-Performance Tip 1：Array 和 ContiguousArray
subtitle: 简单了解一下 Swift 中的 Array 的性能
author: 帕帕
date: 2017-09-05 15:30:56 +0800
categories: 技术
tags: [iOS, Swift]
---

> Array 是随机存储的（random-access）集合类型。

> ContiguousArray 是连续存储（contiguously stored）的数组，并且不允许和 NSArray 进行桥接的。

当我们的数组元素是 Class 或 @objc protocol 类型的话，并且我们不需要在 Objective-C 中使用该数组的话，那么我们最好使用 ContiguousArray。这是因为 Array 需要额外的资源来处理跟 NSArray 的桥接功能，但是 ContiguousArray 则不需要，所以 ContiguousArray 比 Array 的效率要高。

```Swift
class A {

}

// 不要用Array: let array = Array<A>()

let contiguousArray = ContiguousArray<A>()
```

另外需要注意的是官方文档说如果数组元素不是 Class 和 @objc protocol 类型的话，Array 和 ContiguousArray 的效率是一样的。（我猜测是因为如果 Array 的元素都是 Struct 类型的话，它就不需要消耗资源来处理桥接的问题了。）

> Efficiency is equivalent to that of Array, unless Element is a class or @objc protocol type, in which case using ContiguousArray may be more efficient.

但是 [@Paul Hudson](https://twitter.com/twostraws) 在他的[《Pro Swift》](https://gumroad.com/l/proswift)中说他发现即使数组元素是 Struct 类型的话，ContiguousArray 也要比 Array 更快。我们来看看他给出的例子：

```Swift
let array2 = Array<Int>(1...1000000)
let array3 = ContiguousArray<Int>(1...1000000)

var start = CFAbsoluteTimeGetCurrent()
array2.reduce(0, combine: +)
var end = CFAbsoluteTimeGetCurrent() - start
print("Took \(end) seconds")

start = CFAbsoluteTimeGetCurrent()
array3.reduce(0, combine: +)
end = CFAbsoluteTimeGetCurrent() - start
print("Took \(end) seconds")
```

经过我的测试，上面的代码中 ContiguousArray 只用了0.19秒而 Array 用了0.38秒，所以 ContiguousArray 确实要比 Array 快。

如果大家想在性能上有所提升的话，建议大家可以用 ContiguousArray 试一试。

