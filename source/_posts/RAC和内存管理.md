---
layout: post 
title: RAC 和内存管理
subtitle: 最近在用 RAC 的时候发现自己对内存管理还是有些困惑...
author: 帕帕
date: 2017-09-05 15:30:56 +0800
categories: 技术
tags: [iOS, RAC]
---

最近在用 RAC 的时候发现自己对内存管理还是有些困惑，于是自己写了一些代码来验证自己的一些理解。
在一开始接触 RAC 的时候，我们知道 RAC 对于 block 都是 copy 赋值的。

```Swift
@implementation RACSignal

#pragma mark Lifecycle

+ (RACSignal *)createSignal:(RACDisposable * (^)(id<RACSubscriber> subscriber))didSubscribe {
    return [RACDynamicSignal createSignal:didSubscribe];
}
```

```Swift
@implementation RACDynamicSignal

#pragma mark Lifecycle

+ (RACSignal *)createSignal:(RACDisposable * (^)(id<RACSubscriber> subscriber))didSubscribe {
    RACDynamicSignal *signal = [[self alloc] init];
    signal->_didSubscribe = [didSubscribe copy];
    return [signal setNameWithFormat:@"+createSignal:"];
}
```

在创建 RACSingal 的时候会调用其子类 RACDynamicSignal 去创建，我们也看到 RACDynamicSignal 对 didSuscribe 这个 block 是进行了 copy。所以大家可能会被要求注意循环引用的问题，于是大家都用 @weakify(target) 和 @strongify(target) 来避免循环引用的问题。那是不是所有用到 RAC 的地方都需要使用这些宏来避免循环引用的问题，不尽然。比如下面这个：

```Swift
// 场景1
[RACObserve(self, title) subscribeNext:^(id x) {
    NSLog(@"%@", x);
}];
```

接下来，我们来对比以下的几种用法：

```Swift
@interface ViewController()
@property (strong, nonatomic) ViewModel * viewModel;
@end

@implementation ViewController

- (void)viewDidiLoad {
    [super viewDidLoad];

    self.viewModel = [ViewModel new];

    // 场景2
    dispatch_async(dispatch_get_main_queue(), ^{
        self.title = @"你好";
    });

    // 场景3
    [self.viewModel.titleSignal subscribeNext:^(NSString * title) {
        self.title = title;
    }];

    // 场景4
    [RACObserve(self.viewModel, title) subscribeNext:^(NSString * title)     {
        self.title = title;
    }]; 
}

@end
```

场景2是我们平常都会用到的，而且我们也没有在这种场景下去考虑循环引用的问题，这是因为 dispatch 的 block 不是属于 self 的（至于这个 block 是属于谁的，回头我再查点资料或者请各位指教），所以即使你在 block 使用了 self 也不会有循环应用的问题。

场景3很明显是有循环引用的问题：**self->viewModel->titleSignal->block->self**，这个时候如果我们不做处理的话，那么 self 就永远不会被释放。正确的做法应该是使用 @weakify(self) 和 @strongify(self)：

```Swift
// 场景3
@weakify(self);
[self.viewModel.titleSignal subscribeNext:^(NSString * title) {
    @strongify(self);
    self.title = title;
}];
```

场景4在我们看来是没有问题的，因为这里看起来只有 **singal->block->self** 的引用，它们之间并没有造成循环引用的问题。我们先来看看 RACObserve 的实现：

```Swift
#define RACObserve(TARGET, KEYPATH) \
({ \
_Pragma("clang diagnostic push") \
_Pragma("clang diagnostic ignored \"-Wreceiver-is-weak\"") \
__weak id target_ = (TARGET); \
[target_ rac_valuesForKeyPath:@keypath(TARGET, KEYPATH) observer:self]; \
_Pragma("clang diagnostic pop") \
})

- (RACSignal *)rac_valuesForKeyPath:(NSString *)keyPath observer:(__weak NSObject *)observer;
```

其实，看到这里你会认为这里只是调用了一个方法创建了一个 Signal，而且这个 Signal 也并不属于任何对象。我们再来看看具体的实现是怎么样的？

```Swift
- (RACSignal *)rac_valuesAndChangesForKeyPath:(NSString *)keyPath options:(NSKeyValueObservingOptions)options observer:(__weak NSObject *)weakObserver {
    NSObject *strongObserver = weakObserver;
    keyPath = [keyPath copy];

    NSRecursiveLock *objectLock = [[NSRecursiveLock alloc] init];
    objectLock.name = @"org.reactivecocoa.ReactiveCocoa.NSObjectRACPropertySubscribing";

    __weak NSObject *weakSelf = self;

    RACSignal *deallocSignal = [[RACSignal zip:@[
                            self.rac_willDeallocSignal,
                            strongObserver.rac_willDeallocSignal ?: [RACSignal never]
    ]] doCompleted:^{
        // Forces deallocation to wait if the object variables are currently
        // being read on another thread.
        [objectLock lock];
        @onExit {
            [objectLock unlock];
        };
    }];

    return [[[RACSignal createSignal:^ RACDisposable * (id<RACSubscriber> subscriber) {
        // Hold onto the lock the whole time we're setting up the KVO
        // observation, because any resurrection that might be caused by our
        // retaining below must be balanced out by the time -dealloc returns
        // (if another thread is waiting on the lock above).
        [objectLock lock];
        @onExit {
            [objectLock unlock];
        };
    
        __strong NSObject *observer __attribute__((objc_precise_lifetime)) = weakObserver;
        __strong NSObject *self __attribute__((objc_precise_lifetime)) = weakSelf;
    
        if (self == nil) {
            [subscriber sendCompleted];
            return nil;
        }
    
        return [self rac_observeKeyPath:keyPath options:options observer:observer block:^(id value, NSDictionary *change, BOOL causedByDealloc, BOOL affectedOnlyLastComponent) {
                [subscriber sendNext:RACTuplePack(value, change)];
        }];
    }] takeUntil:deallocSignal] setNameWithFormat:@"%@ -rac_valueAndChangesForKeyPath: %@ options: %lu observer: %@", self.rac_description, keyPath, (unsigned long)options, strongObserver.rac_description];
}
```

重点观察 **deallocSignal** 和 **[signal takeUntile:deallocSignal]**，我们把 deallocSignal 单独拿出来看看：

```Swift
RACSignal *deallocSignal = [[RACSignal zip:@[
                        self.rac_willDeallocSignal,
                        strongObserver.rac_willDeallocSignal ?: [RACSignal never]
                        ]] doCompleted:^{
    // Forces deallocation to wait if the object variables are currently
    // being read on another thread.
    [objectLock lock];
    @onExit {
    [objectLock unlock];
    };
}];
```

这里的 deallocSignal 是只有在 self 和 strongObserve 都将要发生 dealloc 的时候才会触发的。即用 RACObserve 创建的信号只有在其 target 和 observe 都发生 dealloc 的时候才会被 disposable (这个好像是 RAC 用来销毁自己资源的东西)。不明白的童鞋，我们回头来分析一下场景4的代码：

```Swift
// 场景4
[RACObserve(self.viewModel, title) subscribeNext:^(NSString * title) {
    self.title = title;
}];
```

用 RACObserve 创建的信号看起来只要出了函数体其资源应该就会被回收，但是这个信号其实是只有在 self.viewModel.rac_willDeallocSignal 和 self.rac_willDeallocSignal 都发生的情况下才会被释放。所以场景4的引用关系看起来只有 signal->block->self，但是这个 signal 只有在 self.rac_willDeallocSignal 的时候才会被释放。所以这里如果不打断这种关系的话就会造成循环引用的问题，正确做法应该是：

```Swift
// 场景4
@weakify(self);
[RACObserve(self.viewModel, title) subscribeNext:^(NSString * title) {
    @strongify(self);
    self.title = title;
}];
```

最后，在说一个特别需要注意的，就是 UITableViewCell 和 UICollectionViewCell 复用和 RAC 的问题。

```Swift
- (NSInteger)tableView:(nonnull UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
    return 1000;
}

- (UITableViewCell *)tableView:(nonnull UITableView *)tableView cellForRowAtIndexPath:(nonnull NSIndexPath *)indexPath {
    UITableViewCell * cell = [tableView dequeueReusableCellWithIdentifier:@"TableViewCell"];

    @weakify(self);
    [RACObserve(cell.textLabel, text) subscribeNext:^(id x) {
        @strongify(self);
        NSLog(@"%@", self);
    }];

    return cell;
}
```

我们看到这里的 RACObserve 创建的 Signal 和 self 之间已经去掉了循环引用的问题，所以应该是没有什么问题的。但是结合之前我们对 RACObserve 的理解再仔细分析一下，这里的 Signal 只要 self 没有被 dealloc 的话就不会被释放。虽然每次 UITableViewCell 都会被重用，但是每次重用过程中创建的信号确实无法被 disposable。那我们该怎么做呢？

```Swift
- (NSInteger)tableView:(nonnull UITableView *)tableView numberOfRowsInSection:(NSInteger)section {
    return 1000;
}

- (UITableViewCell *)tableView:(nonnull UITableView *)tableView cellForRowAtIndexPath:(nonnull NSIndexPath *)indexPath {
    UITableViewCell * cell = [tableView dequeueReusableCellWithIdentifier:@"TableViewCell"];

    @weakify(self);
    [[RACObserve(cell.textLabel, text) takeUntil:cell.rac_prepareForReuseSignal] subscribeNext:^(id x) {
        @strongify(self);
        NSLog(@"%@", self);
    }];

    return cell;
}
```

注意，我们在cell里面创建的信号加上 takeUntil:cell.rac_prepareForReuseSignal，这个是让 cell 在每次重用的时候都去 disposable 创建的信号。

以上所说的关于内存的东西我都用 Instrument 的 Allocations 验证过了，但是依旧建议大家自己也去试试。


