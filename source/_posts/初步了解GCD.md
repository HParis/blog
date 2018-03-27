---
title: 初步了解 GCD
author: 帕帕
date: 2017-09-05 15:30:56 +0800
categories: 技术 
tags: [iOS, GCD]
---

## GCD简介

GCD(Grand Central Dispatch) 是苹果提供的一套多线程编程技术。想象一下，如果让你编写一个可以高效的跑在不同计算机、不同内核的应用程序，你会怎么做呢？你要看看硬件是什么，看看有有多少个内核，想想用什么算法，想想在什么时候去切换线程...总之，你要做的东西多了去了。而 GCD 帮我们屏蔽了这些技术细节，但是如果要用好 GCD 的话，还是要多了解一些知识点。

## Dispatch对象和内存管理

在 Objective-C 里面，所有的 dispatch 对象都是 Objective-C 对象，所以他们同样适用引用技术的内存管理。如果你是使用 ARC 的话，dispatch 对象会向普通的 Objective-C 对象一样自动进行 retain 和 release 操作；如果你是使用 MRC，要记住使用 dispatch_retain 和 dispatch_release 来进行管理。

## 常用API

### dispatch_queue_t（调度队列）

```Swift
public func dispatch_queue_create(label: UnsafePointer<Int8>, _ attr: dispatch_queue_attr_t!) -> dispatch_queue_t!
```

在 GCD 中只能通过上面的 API 来创建调度队列，我们可以通过创建各种各样的 Block 形式的任务并由该调度队列来决定如何去执行这些 Block 任务。上面创建调度队列的函数需要两个参数：

* label: 这个参数是用来给你创建的调度队列进行命名的，特别是在调试的时候你可以通过该参数来判断是哪个调度队列的任务在执行。
* attr: 这个参数只有 DISPATCH_QUEUE_SERIAL 和 DISPATCH_QUEUE_CONCURRENT 两种值（在 Objective-C 中这个参数可以为 NULL，这个时候默认是 DISPATCH_QUEUE_SERIAL）。DISPATCH_QUEUE_SERIAL 是告诉调度队列以串行的方式去执行任务，DISPATCH_QUEUE_CONCURRENT 是告诉调度队列以并发的方式去执行任务。

当然我们还可以通过下面的方法来获取系统已经创建好的调度队列：

```Swift
// 获取全局队列
public func dispatch_get_global_queue(identifier: Int, _ flags: UInt) -> dispatch_queue_t!
```
```Swift
// 获取主线程的com.apple.main-thread (serial)队列
public func dispatch_get_main_queue() -> dispatch_queue_t!
```

注意，所有 pending 状态的 Block 任务都会持有该调度队列的引用，所以我们不需要显示的去持有调度队列，而调度队列会在所有的 Block 任务都从 pending 变为 finished 之后才会被释放。

总之，现在大家要知道的是我们可以把不同的 Block任务提交到调度队列，具体的细节和实现看看后面内容。

### dispatch_sync 和 dispatch_async（同步和异步）

```Swift
let queue = dispatch_queue_create("com.PS.Queue", DISPATCH_QUEUE_SERIAL)  // 创建调度队列
print("Begin Sync")
// 同步调用
dispatch_sync(queue) {
    // Block任务
    print("Execute Block Task1")   
}
dispatch_sync(queue) {
    // Block任务
    print("Execute Block Task2")   
}
print("After Sync")
```

这段代码的输出结果如下：

```Swift
Begin Sync
Execute Block Task1
Execute Block Task2
After Sync
```

上面的例子就是我们平常对 dispatch_sync 的用法，并且我们可以看到第一个 Block 任务执行之后才会执行第二个 Block 任务。dispatch_sync 需要等待 Block的任务执行完成之后，才能继续往后执行。但是使用 dispatch_sync 的时候，有几点是需要注意的：

1. 当调用 dispatch_sync 方法的时候，系统默认情况下会在当前线程去执行调度队列里的任务，只有在一些特殊情况下才会把调度队列的任务分配到其他线程去执行。所以我们就知道，线程和调度队列并不是一对一的关系。至于为什么默认情况下会在当前线程去执行调度队列里的任务，我的猜测是为了性能。大家想一想，dispatch_sync 会同步执行 Block任务， Block任务没有结束的情况下，后面的代码是无法执行的。基于这样一个同步的机制，GCD 还有必要先把当前线程挂起，然后去创建新线程，然后切换到新的线程去执行调度队列里的任务，然后再把线程切换到当前线程，然后再让当前线程恢复么？结论是没有必要。

2. 你不能够在当前的串行调度队列的任务里面去添加新的任务到当前的调度队列里面，否则会造成死锁。这句话怎么理解呢，我们来来看看下面的例子：
    
```Swift
// 例1
let queue = dispatch_queue_create("com.PS.Queue", DISPATCH_QUEUE_SERIAL)  // 创建串行的调度队列
// 同步调用
dispatch_sync(queue) {
    // Block1
    print("Begin Execute Block Task1")
    dispatch_sync(queue) {
        // Block2
        print("Execute Block Task2")   
    }
    print("End Execute Block Task1")
}

// 例1的结果
Begin Execute Block Task1
    
```

为什么 Block1 后面的 print 和 Block2 的 print 都不执行了呢？首先我们要知道被 DISPATCH_QUEUE_SERIAL 声明的调度队列是串行调度队列，串行调度队列里的任务是同时只能有一个任务在执行，并且当前任务没有执行完成，下一个任务也无法执行。上面的例子中会先输出 Block1 中的 *Begin Execute Block Task1*，然后这个时候再把 Block2 添加到同一个串行调度队列中去。这个时候的 Block1 还没有执行完成，它需要等 dispatch_sync 的 Block2 执行完成之后才能继续执行，而 Block2 又必须等待 Block1 执行完成之后才能执行，所以这个时候就造成 Block1 等着 Block2，Block2 等着 Block1 的死锁。

我们再把调度队列属性改为 DISPAT_QUEUE_CONCURRENT，然后再看看执行结果是什么：


```Swift
// 例2
let queue = dispatch_queue_create("com.PS.Queue", DISPATCH_QUEUE_SERIAL)  // 创建串行的调度队列
// 同步调用
dispatch_sync(queue) {
    // Block1
    print("Begin Execute Block Task1")
    dispatch_sync(queue) {
        // Block2
        print("Execute Block Task2")   
    }
    print("End Execute Block Task1")
}
```
    
```Swift
// 例2的结果
Begin Execute Block Task1
Execute Block Task2
End Execute Block Task1
```

被 DISPATCH_QUEUE_CONCURRENT 声明的并行调度队列就没有这种死锁的问题。并行调度队列里的任务是不会霸占资源不放的，每一个任务执行一个时间片段之后会把资源交出来给别的任务去执行。所以例2中的 Block1 虽然需要等待 Block2 执行完成之后才能继续执行，但是当 Block1 在等待的过程中，是可以把资源释放出来交给 Block2 去执行，Block2 执行完成之后 Block1 就可以继续执行了。所以，这个时候就不会造成死锁来。

再来看看下面的例子会不会造成死锁：

```Swift
override func viewDidLoad() {
    dispatch_sync(dispatch_get_main_queue()) {
        print("Excute Block Task")
    }
}
```
    
答案是会的。给大家一点提示，主线程的默认调度队列是串行（DISPATCH_QUEUE_SERIAL）的，viewDidLoad() 是在主线程的调度队列 com.apple.main-thread (serial) 执行的。

上面的例子主要是希望大家理解串行和并行的概念，同时要明白造成死锁的原因。而要解决死锁一般可以用 DISPATCH_QUEUE_CONCURRENT 或接下来我们要讲的 dispatch_async 来解决。

通过对 dispatch_sync 的了解，我们可以利用 dispatch_async 很快的写出异步代码：

```Swift
let queue = dispatch_queue_create("com.PS.Queue", DISPATCH_QUEUE_SERIAL)  // 创建调度队列
print("Begin Async")
// 异步调用
dispatch_async(queue) {
    // Block1
    print("Execute Block Task1")   
}
dispatch_async(queue) {
    // Block2
    print("Execute Block Task2")   
}
print("After Async")
```

这个例子的结果有好几种：

```Swift
// 结果1
Begin Async
After Async
Execute Block Task1
ExEcute Block Task2
```
```Swift
// 结果2
Begin Async
Execute Block Task1
ExEcute Block Task2
After Async
```

上面只是列出来两种可能，但实际上还有其他的可能。当我们调用 dispatch_async 的时候，它总是会在 Block 任务被提交之后马上返回，而不会傻傻的等待 Block 任务执行完成。由于上面创建的是串行调度队列，所以我们可以保证 Block1 要比 Block2 优先执行，但是 After Async 就无法确定是在 Block1 的前后还是 Block2 的前后。

如果我们把上面的 DISPATCH_QUEUE_SERIAL 改成 DISPATCH_QUEUE_CONCURRENT，那我们就无法确定 After Async、Block1 和 Block2 这三者的执行顺序了。

我们刚才说到用 dispatch_async 可以解决死锁的问题，那它是怎么解决的呢？

```Swift
let queue = dispatch_queue_create("com.PS.Queue", DISPATCH_QUEUE_SERIAL)  // 创建串行的调度队列
// 异步调用
dispatch_async(queue) {
    // Block1
    print("Begin Execute Block Task1")
    dispatch_async(queue) {
        // Block2
        print("Execute Block Task2")   
    }
    print("End Execute Block Task1")
}
```

上面的例子会优先输出 Block1 的 *Begin Execute Block Task1* 之后，通过 dispatch_async 把 Block2 提交到串行队列里面，然后又马上返回到 Block1 去输出 *End Execute Block Task1*，这个时候的 Block1 就结束了，接下来就开始执行 Block2。所以上面的代码是不会造成死锁的，虽然上面的例子也是创建了一个串行调度队列，但是该调度队列只是保证了 Block1 要比 Block2 优先执行。

### dispatch_once

写过 Objective-C 的人都知道，dispatch_once 一般会被用来创建单例对象：

```Swift
@implementation Single
+ (Single *)sharedInstance {
    static Single * _single = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        _single = [[Single alloc] init];
    });
    return _single; 
}
@end
```

这是由于 dispatch_once 是线程安全且只会执行一次，所以才会被用来作为单例的实现。这里需要注意的是 dispatch_once_t 必须是静态的或全局的才能保证 dispatch_once 的 Block 只会被执行一次，所以上面的代码用了 static 来修饰 dispatch_once_t。

### dispatch_apply

```Swift
public func dispatch_apply(iterations: Int, _ queue: dispatch_queue_t!, _ block: (Int) -> Void)
```

其中的 interations 是表明要执行多少次 block，block 中的 Int 是该 Block 被执行的序号。调用这个方法的时候要注意该方法跟 dispatch_sync 一样会阻塞当前线程，所以我们需要注意在主线程中调用该方法。

### dispatch_after

```Swift
public func dispatch_after(when: dispatch_time_t, _ queue: dispatch_queue_t, _ block: dispatch_block_t)
```

调用这个方法的时候需要注意的是 when 这个参数，你需要通过 dispatch_time 或 dispatch_walltime 来创建。并且该方法是异步执行的，并不会阻塞当前线程。

一般的写法如下：

```Swift
dispatch_after(dispatch_time(DISPATCH_TIME_NOW, Int64(5 * NSEC_PER_SEC)), queue) {
    print("5s \(NSThread.currentThread())")
}
```

### dispatch_group_t

dispatch_group_t 是用来做聚合同步的，它可以用来跟踪你提交的所有任务（即使是在不同的调度队列也可以）的完成状态。

接下来我们来看看 dispatch group 的一些常见用法：

```Swift
// 创建 dispatch_group_t 对象
let group = dispatch_group_create()

// 创建串行队列
let serialQueue = dispatch_queue_create("Serial Queue", DISPATCH_QUEUE_SERIAL)

// 提交两个 Block 任务到 serialQueue，同时关联 serialQueue 和 group 的关系
dispatch_group_async(group, serialQueue) {
    print("Execute Block1 within Serial Queue")
}
dispatch_group_async(group, serialQueue) {
    print("Execute Block2 within Serial Queue")
}

// 创建并行队列，并提交 Block 任务，同时关联该并行队列和 group 的关系
dispatch_group_async(group, dispatch_queue_create("Concurrent Queue", DISPATCH_QUEUE_CONCURRENT)) {
    print("Execute Block within Concurrent Queue")
}

// 下面的代码只有当前面被关联到 group 的所有任务完成之后才会被触发
dispatch_group_notify(group, dispatch_queue_create("Finished")) {
    print("Finished")
}
```

注意，关联到 group 的方法只有 dispatch_group_async 而没有 dispatch_group_sync。

但是还有另外一种方法可以让我们关联一个普通的任务：

```Swift
// 创建 dispatch_group_t 对象
let group = dispatch_group_create()

// 使用 dispatch_group_enter 和 dispatch_group_leave 的话，我们不需要调用
// dispatch_group_async 也能关联一个任务到 group 上
dispatch_group_enter(group)
self.executeTask {
    // 执行代码
    
    dispatch_group_leave(group)
}

// 下面的代码只有当前面被关联到 group 的所有任务完成之后才会被触发
dispatch_group_notify(group, dispatch_queue_create("Finished")) {
    print("Finished")
}
```

使用 dispatch_group_enter 和 dispatch_group_leave 的时候，它们必须成双成对出现，否则 dispatch_group_notify 是不会被调用的。

接下来我们还要了解一下 dispatch_group_wait：

```Swift
public func dispatch_group_wait(group: dispatch_group_t, _ timeout: dispatch_time_t) -> Int
```

dispatch_group_wait 可以指定一个 timeout 的参数，当 group 的任务没有在规定的时间内完成，它会返回一个非零的值，当 group 的任务能够在规定的时间内完成就返回0。同时，大家要注意这个方法会挂起当前线程，所以在主线程的时候要慎重使用该方法。

### dispatch_barrier_t

我们先来试想一个场景，假如现在有多个线程要去读取一份文件的内容，同时又有其他线程想要去更新该文件的内容，那么就有可能会发生你读错文件内容的现象。这个时候我们可以把所有读写操作都放到我们之前学习的串行队列去执行，但是我们都知道同时有多个线程去读取一份文件内容是没有问题的。

使用 dispatch barrier 可以解决上面的问题：

```Swift
// 创建操作文件的并发队列
let queue = dispatch_queue_create("File", DISPATCH_QUEUE_CONCURRENT)
dispatch_async(queue) {
    // Read1
}
dispatch_async(queue) {
    // Read2
}
dispatch_barrier_async(queue) {
    // Write
}
dispatch_async(queue) {
    // Read3
}
```

通过 dispatch_barrier_async 或 dispatch_barrier_sync 提交的任务会等待当前队列里正在执行的任务执行完毕才会执行，并且其他还没有执行的任务都必须等待提交到 dispatch barrier 的任务执行完毕之后才会开始执行。所以上面的代码中，当 Write 任务被提交的时候，如果当前队列中只有 Read1 在执行，那么 Write 会等待 Read1 执行完成之后才会执行，Read2  和 Read3 都必须等待 Write 执行完之后才会执行。另外，上面的代码中创建的是并发队列，因为如果是串行队列的话就没有必要用 dispatch barrier 了。

### dispatch_semaphore_t

dispatch semaphore 是一个效率非常高的传统计数信号量，所以我们一般可以用这个来控制最大的并发数量。

```Swift
// 创建初始值为2的信号量，最大并发数量为2
let semaphore = dispatch_semaphore_create(2)
// 创建并发队列
let queue = dispatch_queue_create("Semaphore", DISPATCH_QUEUE_CONCURRENT)
// 创建100个并发任务
for index in 1...100 {
    // 这个方法会进行信号量减1的操作，并且如果信号量减1之后的结果小于0的话，该方法会造成线程的挂起直
    // 到该信号量进行加1操作才会恢复，所以在主线程要注意该方法的使用。
    // 注意：这个方法要放在 dispatch_async 外面，否则系统依旧会创建超过2个线程同时来处理该调度队列
    // 的任务
    dispatch_semaphore_wait(semaphore, DISPATCH_TIME_FOREVER)
    dispatch_async(queue) {
        
        // 释放资源，信号量增加1
        dispatch_semaphore_signal(semaphore)
    }
}
```

## 其他

GCD 在 Swift3 的语法跟现在的语法不太一样了，有兴趣的可以自行去了解。在未来可能会考虑把本文章的代码都用 Swift3 的语法来重新写一下。




