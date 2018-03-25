---
title: 说说 Objective-C 中的 Copy 操作
date: 2018-03-23 16:35:55 +0800
author: 帕帕
categories: 技术 
tags: [iOS, Objective-C]
---

## 浅拷贝（Shallow copies）和深拷贝（Deep copies）

我们都知道 Objective-C 中把 Copy 操作分成两种：`浅拷贝（Shallow copies）`和`深拷贝（Deep copies）`。学过 C 语言的同学应该知道区分这两种操作的区别其实很简单：

> 浅拷贝（Shallow copies）: 指针拷贝，指向的还是同一块内容的地址
> 深拷贝（Deep copies）: 内容拷贝


但是在 Objective-C 里面对于 Copy 的实现还是跟 C 语言的有点差别。我们先来看看 Apple 的官方文档给出的一张图：

![Collections Programming Topics](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Collections/Art/CopyingCollections_2x.png)

通过上图可以看出`浅拷贝`过后，Array 1 和 Array 2 的元素都是相同的指针地址，指向相同的内容；`深拷贝`过后，内容被拷贝一份新的出来，Array 2 的元素的指针地址都和 Array 1 不一样，因为 Array2 的元素的指针地址都指向新的内容。

## immutable 和 mutable 对象的拷贝

在 Objective-C 中一般会用 copy 或 mutableCopy 进行拷贝操作，我们可以通过观察指针变化来确定这两种拷贝操作是`浅复制`还是`深复制`。
    
* immutable 对象的复制操作

    ```Objective-C
    NSString * aName = @"帕帕";
    NSString * bName = [aName copy];
    NSMutableString * cName = [aName mutableCopy];
    
    NSLog(@"aName 的指针：%p", aName);
    NSLog(@"bName 的指针：%p", bName);
    NSLog(@"cName 的指针：%p", cName);
    
    ```

    输出的结果：
    
    ```
    aName 的指针：0x103d34070
    bName 的指针：0x103d34070
    cName 的指针：0x600000250dd0
    ```

* mutable 对象的复制操作

    ```Objective-C
    NSMutableString * aName = [NSMutableString stringWithString:@"帕帕"];
    NSString * bName = [aName copy];
    NSMutableString * cName = [aName mutableCopy];
    
    NSLog(@"aName 的指针：%p", aName);
    NSLog(@"bName 的指针：%p", bName);
    NSLog(@"cName 的指针：%p", cName);
    
    ```

    输出的结果：
    
    ```
    aName 的指针：0x60000025e150
    bName 的指针：0x600000222900
    cName 的指针：0x60000025e450
    ```

通过上面两个例子以及它们的输出结果，我们可以得出下面这个表格：

|  | imutable 对象 | mutable 对象 |
| :---: | :---: | :---: |
| copy | 浅复制 | 深复制 |
| mutableCopy | 深复制| 深复制 |

上面的规则对集合对象也是一样的：NSArray 和 NSMutableArray，NSDictionary 和 NSMutableDictionary，NSSet 和 NSMutableSet


## 单层深复制（one-level-deep）

```Objective-C
NSMutableString * aString = [NSMutableString stringWithString:@"Hello"]

NSMutableArray * aArray = [NSMutableArray arrayWithObjects:aString, nil];
NSArray * bArray = [aArray copy];

NSMutableString * bString = bArray[0];
[bString appendString:@" 帕帕"];
    
NSLog(@"aArray 的指针：%p", aName);
NSLog(@"bArray 的指针：%p", bName);
NSLog(@"aArray 第一个元素的指针: %p，内容：%@", aArray[0], aArray[0]);
NSLog(@"bArray 第一个元素的指针: %p，内容：%@", bArray[0], bArray[0]);
```

输出结果：

```
aArray 的指针：0x60000025d9a0
bArray 的指针：0x60000002cf60
aArray 第一个元素的指针: 0x60000025d880，内容：Hello 帕帕
bArray 第一个元素的指针: 0x60000025d880，内容：Hello 帕帕
```

从 aArray 到 bArray 的 copy 操作之后，它们的指针地址发生了变化，按照我们之前的理解这是`深拷贝`。`深拷贝`会把 aArray 的元素都拷贝一份，那为什么改变 bArray 的元素的值会导致 aArray 的元素的值也发生了变化呢？

![Imgur](https://i.imgur.com/svn3AbQ.png)

## 完全深复制

那我们要如何做到真正的深复制呢？我们可以简单的把上面的代码改一下：

```Objective-C
NSMutableString * aString = [NSMutableString stringWithString:@"Hello"]

NSMutableArray * aArray = [NSMutableArray arrayWithObjects:aString, nil];

// 只需要改动这一行代码
NSArray *bArray = [NSKeyedUnarchiver unarchiveObjectWithData:[NSKeyedArchiver archivedDataWithRootObject:aArray]];

NSMutableString * bString = bArray[0];
[bString appendString:@" 帕帕"];
    
NSLog(@"aArray 的指针：%p", aName);
NSLog(@"bArray 的指针：%p", bName);
NSLog(@"aArray 第一个元素的指针: %p，内容：%@", aArray[0], aArray[0]);
NSLog(@"bArray 第一个元素的指针: %p，内容：%@", bArray[0], bArray[0]);
```

输出结果：

```
aArray 的指针：0x600000259cb0
bArray 的指针：0x600000030ac0
aArray 第一个元素的指针: 0x604000452120，内容：Hello
bArray 第一个元素的指针: 0x604000452780，内容：Hello 帕帕
```

只要先对集合对象分别用 NSKeyedArchiver 和 NSKeyedUnarchiver 就可以真正完成对一个集合对象的深复制。

## Copy 和 内存管理

之前我们说过 Objective-C 里面对于 Copy 的实现还是跟 C 语言的有点差别，那差别在什么地方呢？
内存中做复制操作是很耗费资源的，而我们都知道 Objective-C 高效的一个原因在于它的内存管理机制是`引用计数`。我们前面分析的`深拷贝`是对内容的拷贝，这一点跟 C 语言的一样。C 语言的`浅拷贝`是指针的拷贝，它依旧做了一次复制操作。而在 Objective-C 中，`浅拷贝`其实只是引用计数的增加，不信的话，我们可以看看下面的例子：

```Objective-C
NSArray * aArray = [NSArray arrayWithObjects:@"帕帕", nil];
NSLog(@"aArray 的指针：%p，引用计数：%ld", aName, CFGetRetainCount((__bridge CFTypeRef)(aArray)));
NSArray * bArray = [aArray copy];
NSLog(@"aArray 的指针：%p，引用计数：%ld", aName, CFGetRetainCount((__bridge CFTypeRef)(aArray)));
NSMutableArray * cArray = [aArray mutableCopy];
NSLog(@"aArray 的指针：%p，引用计数：%ld", aName, CFGetRetainCount((__bridge CFTypeRef)(aArray)));
```

输出结果：

```
aArray 的指针：0x604000443ba0，引用计数：2
aArray 的指针：0x604000443ba0，引用计数：3
aArray 的指针：0x604000443ba0，引用计数：3
```

为什么 aArray 刚出来的时候的引用计数是 2？因为 `[NSArray arrayWithObjects:@"帕帕", nil]` 本身就是一个对象，它的引用计数就是 1；然后我们又定义了 aArray 来引用这个对象，此时它的引用计数就增加了 1，变成了 2；之后我们对 aArray 进行了 copy 操作，发现它的引用计数变成了 3，所以这里的 copy 操作其实相当于 retaion；最有我们对 aArray 进行了 mutableCopy 操作，此时它的引用计数还是 3，没有发生变化，因为这个时候进行了内容复制。

所以在 Objective-C 中对一个 imutable 对象进行的 copy（浅复制）操作，其实都只会引起引用计数的变化，而不会在内存中做出任何拷贝操作，包括指针拷贝。

## NSCopying 和 NSMutableCopying

如果我们有一个自定义的对象，并且对其进行 copy 操作的话，会发生什么：

```Objective-C
// Person
@interface Person: NSObject
@property (nonatomic, copy) NSString * name;
@end
@implementation Person
@end

Person * aPerson = [Person new];
Person * bPerson = [aPerson copy];
```

Xcode 直接奔溃了：

```
// 崩溃
*** Terminating app due to uncaught exception 'NSInvalidArgumentException', reason: '-[Person copyWithZone:]: unrecognized selector sent to instance 0x60000000d5f0'
```

为什么我们对一个 Person 对象使用了 copy，Xcode 确报的是找不到 `copyWithZone:` 这个 selector 的错误。

这是因为 Objective-C 中规定，一个对象如果想要使用 copy 或 mutableCopy 的操作，必须要实现 `NSCopying` 或 `NSMutableCopying` 这两个协议。这两个协议规定了对象需要实现 `copyWithZone:` 或 `mutableCopyWithZone:` 这两个方法，因为对一个对象做 copy 或 mutableCopy 最后都会去调用这两个方法来做最终的实现。
上面例子中的集合对象能够使用 copy 和 mutableCopy 操作是也因为它们都实现了 NSCopying 和 NSMutableCopying 协议。

我们来看看如何对一个普通的对象实现 NSCopying 协议：

```Objective-C
@interface Person: NSObject <NSCopying>
@property (nonatomic, copy) NSString * name;
@property (nonatomic, strong) NSMutableArray * mArray;
@end

@implementation Person
- (instancetype)copyWithZone:(NSZone *)zone {
    Person * person = [[self class] new];
    person.name = [self.name copy];
    person.mArray = [self.mArray mutableCopy];
    return person;
}
@end
```
 
这样，我们就可以愉快的使用 `[Person copy]` 了。当然，这里 Person 的 mArray 也只是`单层深复制`，如果想要实现`完全深复制`的话，我们可以用 NSKeyedArchiver 和 NSKeyedUnarchiver 来完成对 mArray 的`完全深复制`。

参考资料：
1. https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Collections/Articles/Copying.html
2. https://www.zybuluo.com/MicroCai/note/50592

