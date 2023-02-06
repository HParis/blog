---
title: 如何避免 App 被重签名（奇怪的用法）
date: 2023-02-05 16:01:06
author: 帕帕
categories: 技术
tags: [iOS, Swift]
thumbnail: https://images.unsplash.com/photo-1507289872412-523fc6b2db5f?ixlib=rb-0.3.5&ixid=eyJhcHBfaWQiOjEyMDd9&s=20ca9d0eba2016344894aec7bb453a2d&auto=format&fit=crop&w=160&q=100
---

由于业务线的调整，之前负责的产品不打算上架 AppStore 了，需要调整为只提供给公司内部的员工使用。当时考虑的解决方案有两种：

1. 使用新的 BundleID 和企业证书打包，采用此种方案需要重新申请部分第三方 SDK 的权限
2. 直接在现在 AppStore 的包的基础之上使用企业证书**重签名**，这种方式会导致推送功能无法使用，还有一些依赖于证书的特性（Capabilities）无法使用

后来经过团队的考虑决定采用第二种方案，但是在采用第二种方案的之后出现了一个崩溃问题，而这个问题就是可以被我们利用来防止 App 被重签名的关键了。

写过 Swift 的同学都知道，在 Swift 中有一种 optional 类型的数据，它可以通过 **!** 进行强制解包，而当它强制解包不成功的时候就会导致程序出现崩溃问题。

（说到这里可能会有同学认为在代码中就不应该使用强制解包的语法，那请问 Swift 为什么要推出这种语法呢？其实当我们能够百分比确认某一个变量或业务的状态，那我们就应该使用强制解包，这个时候强制解包的好处就是能够帮助我们在开发阶段提前把问题暴露出来，并且在代码的使用上也更方便。）

我们的应用由于需要在 Extension 和 Host App 之间进行数据共享，所以我们开启了 **App Groups** 这个特性。而在 Swift 代码中我们是这样去使用：

```Swift
let store = UserDefaults(suiteName: "group.com.yourcompany.product")!
// or
let url = FileManager.default.containerURL(forSecurityApplicationGroupIdentifier: "group.com.yourcompany.product")!
```

由于使用重签名的企业证书并没有包含这个的 **App Group** 的特性，所以最后会导致当我们手机安装重签名的包之后并且在启动后运行到这段代码的时候因为强制解包失败导致应用直接崩溃。
所以一般的应用即使在业务上没有相关的需求，我依旧建议大家也应该增加这样的特性功能，并且利用这个特性来让其他人没有那么容易就能对你的应用进行重签名之后拿去使用。
