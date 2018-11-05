---
title: NSNotification 的一些知识点
author: 帕帕
date: 2018-11-05 17:48:56 +0800
categories: 技术
tags: [iOS]
thumbnail: https://images.unsplash.com/photo-1514464750060-00e6e34c8b8c?ixlib=rb-0.3.5&ixid=eyJhcHBfaWQiOjEyMDd9&s=3749c47dd7beec20102c6b32fc19833a&auto=format&fit=crop&w=160&q=100
---

## 重复添加相同观察者

我们先来看看日常开发中我们对 NSNotification 的正常用法，如下：
```swift
// 定义通知
let TestNotification = NSNotification.Name.init("com.papa.test")

// 测试类
class Test {

    init() {
        NotificationCenter.default.addObserver(self, selector: #selector(Test.test(notification:)), name: TestNotification, object: nil)
    }

    // 注意
    deinit {
        NotificationCenter.default.removeObserver(self)
    }

    @objc func test(notification: Notification) {
        print("Test")
    }
}
```

但是如果我们在刚才代码中的 `init` 方法里面对同一个通知多次添加同一个观察者的话，会发生什么？
```swift
init() {
    NotificationCenter.default.addObserver(self, selector: #selector(Test.test(notification:)), name: TestNotification, object: nil)
    NotificationCenter.default.addObserver(self, selector: #selector(Test.test(notification:)), name: TestNotification, object: nil)
}

// 发送 TestNotification 通知
NotificationCenter.default.post(name: TestNotification, object: nil)
```

答案是会输出：
```swift
Test
Test
```

所以我们要尽量避免重复添加观察者，因为这有可能会造成一些未知现象的发生。

## 通知中的线程问题
 
```swift
// 定义通知
let ThreadNotification = NSNotification.Name.init("com.papa.thread")

// 测试类
class Test {

    init() {
        print("Add Observer: \(Thread.current)")
        NotificationCenter.default.addObserver(self, selector: #selector(Test.test(notification:)), name: ThreadNotification, object: nil)
    }

    // 注意
    deinit {
        NotificationCenter.default.removeObserver(self)
    }

    @objc func test(notification: Notification) {
        print("Receive: \(Thread.current)")
    }
}

DispatchQueue.init(label: "com.ps.test.queue").async {
    print("Post: \(Thread.current)")
    NotificationCenter.default.post(name: ThreadNotification, object: nil)
}

```

我们来看看观察者是在什么线程上接受到通知的:
```swift
Add Observer: <NSThread: 0x60000147d1c0>{number = 1, name = main}
Post: <NSThread: 0x600001462640>{number = 3, name = (null)}
Receive: <NSThread: 0x600001462640>{number = 3, name = (null)}
```

虽然我们是在主线程中去添加观察者，但是因为我们是在其他线程中去发送通知的，所以最后我们也是在其他线程中接收到通知的。

## 通知中的阻塞问题

```swift
// 定义通知
let SleepNotification = NSNotification.Name.init("com.papa.sleep")

// 测试类
class Test {

    init() {
        NotificationCenter.default.addObserver(self, selector: #selector(Test.test(notification:)), name: SleepNotification, object: nil)
    }

    // 注意
    deinit {
        NotificationCenter.default.removeObserver(self)
    }

    @objc func test(notification: Notification) {
        sleep(3)
    }
}

let start = Date()
NotificationCenter.default.post(name: SleepNotification, object: nil)
let end = Date()
print("相差：\(end.timeIntervalSince(start))")
```

我们可以看到最后相差时间大概是 `3s` ，通过上面的代码我们就知道单 NotificationCenter 去 post 一个通知的时候，它会等待观察者处理完改通知之后才会继续往后执行。所以平常使用过程中我们要注意 post 有可能会阻塞当前线程，特别是在主线程中。


