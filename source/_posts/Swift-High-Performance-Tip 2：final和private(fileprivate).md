---
title: Swift High-Performance Tip 2：final 和 private(fileprivate)
author: 帕帕
date: 2017-09-05 15:30:56 +0800
categories: 技术
tags: [iOS, Swift]
---

> Dynamic dispatch means that program has to determine at run time which method or property is being referred to and then perform an indirect call or indirect access.

我们都知道 Swift 的 class 是可以被继承，function 和 property 是可以被重写的，而这就意味着 Swift 需要 dynamic dispatch 这种机制来完成这些功能。Swift 的 dynamic dispatch 首先会再 method table 查找方法，然后间接调用。很明显这种方式要比直接调用的效率慢，并且用间接调用的方式还会阻止编译器的一些优化无法实现。

**那么应该怎么优化呢？**

当我们明确的知道 class、function、property 是不需要 overridden，我们可以通过使用 final 和 private(fileprivate) 这些关键字减少动态派发的发生，从而有效的提高效率。

在 Swift 中，如果被 final 或 private(fileprivate) 修饰的 class、function、property 是不能 overridden，并且调用这些 class、function、property 的时候不再通过 dynamic dispatch 去间接调用，而是直接调用。

所以，通过在必要的代码中使用 final 或 private(fileprivate) 这些关键字进行优化的话，将可以有效提高的效率。

**Whole Module Optimization**

Swift 的 class、function、property 的默认权限都是 internal ，除非我们明确的加上 public 或 private(fileprivate) 关键字才能改变它们的默认权限。

编译器在编译 Module 的时候都是对里面的源文件进行单独编译，这样的话编译器就无法确切的知道被 internal 修饰的 class、function、property 究竟有没有被 overridden。一旦我们开启 Whole Module Optimization 的优化选项，编译器就会同时对整个 Module 的所有源文件进行编译，这个时候编译器就可以知道哪些被 internal 修饰的 class、function、property 没有被 overridden，从而把它们的权限从 internal 修改为 final。这样的话，就可以减少 dynamic dispatch 的发生从而提高效率。

开启编译优化选项的步骤如下：Xcode -> Build Settings -> Swift Compiler -> Optimization Level。

![](http://i.imgur.com/0AxWEVA.jpg)

---

**参考文献**

1. https://www.reddit.com/r/iOSProgramming/comments/3atu5w/does_swift_use_dynamic_method_dispatch_or_a/

2. https://developer.apple.com/swift/blog/?id=27

3. https://github.com/apple/swift/blob/3ef6c79e3c591cf31b8a853b1357e1b8c5771252/docs/OptimizationTips.rst#whole-module-optimizations

