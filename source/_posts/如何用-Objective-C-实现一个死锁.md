---
title: 如何用 Objective-C 实现一个死锁
date: 2018-04-01 20:29:42 +0800
author: 帕帕
categories: 技术
tags: [iOS, Objective-C, GCD]
thumbnail: https://images.unsplash.com/photo-1509655172625-03265ba919a1?ixlib=rb-0.3.5&ixid=eyJhcHBfaWQiOjEyMDd9&s=24b01fba0d86b9fa95814c692429379f&auto=format&fit=crop&w=160&q=100
---

> 当系统存在两个线程及以上的时候，双方都在等待对方停止执行，以获得系统资源，但是没有一方提前退出的时候就叫做死锁。


那在 Objective-C 里面如何实现死锁呢：

```Objective-C 
self.lock1 = [NSLock new];
self.lock2 = [NSLock new];
    
dispatch_async(dispatch_queue_create("com.papa.task1", DISPATCH_QUEUE_SERIAL), ^{
    [self.lock1 lock];
    NSLog(@"task1 获得 lock1");
    sleep(2);
    [self.lock2 lock];
    NSLog(@"task1 获得 lock2");
    [self.lock2 unlock];
    [self.lock1 unlock];
});

dispatch_async(dispatch_queue_create("com.papa.task2", DISPATCH_QUEUE_SERIAL), ^{
    [self.lock2 lock];
    NSLog(@"task2 获得 lock2");
    sleep(2);
    [self.lock1 lock];
    NSLog(@"task2 获得 lock1");
    [self.lock1 unlock];
    [self.lock2 unlock];
});
```
我们可以看到最后控制台输出的结果是：

```
task2 获得 lock2
task1 获得 lock1
```

或

```
task2 获得 lock2
task1 获得 lock1
```

为什么是两种结果呢，可以参考我的[「初步了解 GCD」](https://hparis.github.io/blog/2017/09/05/%E5%88%9D%E6%AD%A5%E4%BA%86%E8%A7%A3GCD/)。

但是不管如何，这两种结果都没有输出 `task1 获得 lock2` 或 `task2 获得 lock1`。我们来分析一下：

    * task1 获得 lock1，然后沉睡 2s
    * task2 获得 lock2，然后沉睡 2s
    * task1 经过 2s 的沉睡之后想要去获取 lock2，此时发现 lock2 已经被使用，那就继续等待
    * task2 经过 2s 的沉睡之后想要去获取 lock1，此时发现 lock1 已经被使用，那就继续等待
    * task1 又被唤醒想要去获取 lock2，此时 lock2 依旧没有被 task2 释放，只能继续等待
    * task2 又被唤醒想要去获取 lock1，此时 lock1 依旧没有被 task1 释放，只能继续等待
    * ...
    * ...
    
于是 task1 和 task2 都在等待对方释放资源，但是自己也不退出也不释放资源，最终导致死锁的产生。


接下来，我们来讨论另外一个例子是不是死锁：

```Objective-C
// 某个按钮的点击事件
- (void)onClick:(UIEvent *)event {
    dispatch_sync(dispatch_get_main_queue(), ^{
        ...    
    });
}
```

相信大家都知道上面这个例子会导致主线程发生阻塞的现象，但是这是因为死锁造成的么？

> Submits a block to a dispatch queue for synchronous execution. Unlike dispatch_async, this function does not return until the block has finished. Calling this function and targeting the current queue results in deadlock.

在官方文档里面明确的说了，这就是死锁。我们可以把上面的例子“翻译”一下：

```Objective-C
// 首先进入 onClick 的时候，我们可以认为此时是需要加锁的
// 某个按钮的点击事件
- (void)onClick:(UIEvent *)event {
    [self.lock1 lock];
    
    // 此时我们可以认为 dispatch_sync 是在获取 block 里面的 lock2 
    {
        [self.lock2 lock];
        
        // 放在主线程执行，那么它也需要获得 lock1 
        [self.lock1 lock];
        [self.lock1 unlock];
        
        [self.lock2 unlock];
    }
    
    [self.lock1 unlock];
}
```

我们可以看到其实上面的情况就是两个任务都在同时竞争主线程的资源，并且谁都没有退出最终导致死锁的产生。但是这两个任务并不是普通的两个线程在竞争资源，而是都在主线程上，一个嵌套另外一个。而且这种特殊的情况，在运行的时候会直接导致奔溃，而不像我们一开始的例子一样只是在互相等待。但是既然苹果把这种情况也称为死锁，那我们就当做死锁来看待，毕竟他们都是在竞争系统资源。

