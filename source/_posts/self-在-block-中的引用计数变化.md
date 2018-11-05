---
title: self 在 block 中的引用计数变化
author: 帕帕
date: 2018-04-19 11:34:51 +0800
categories: 技术 
tags: [iOS, Objective-C, Block]
thumbnail: https://images.unsplash.com/photo-1462303966430-8a4708fd729e?ixlib=rb-0.3.5&ixid=eyJhcHBfaWQiOjEyMDd9&s=c9dd0952e673c518403fb8d4c28f93b5&auto=format&fit=crop&w=160&q=60
---


相信大家在 Objective-C 中都会通过 `__waek` 的修饰符来保证 block 和 self 不会互相引用，代码如下:

```Objective-C
__weak typeof(self) weakSelf = self;
self.block = ^{
    __strong typeof(self) strongSelf = self;
    ...
}
```

但是你思考过 self 在这一段旅程中的引用计数变化么，接下来我会通过三个例子来展示这一段旅程是怎样的？


```Objective-C
// 🌰1
NSLog(@"Before block：%ld", CFGetRetainCount((__bridge CFTypeRef)(self)));
self.block = ^{
    self;
    NSLog(@"Within block：%ld", CFGetRetainCount((__bridge CFTypeRef)(self)));
};
self.block();
NSLog(@"After block：%ld", CFGetRetainCount((__bridge CFTypeRef)(self)));


// 🌰2
__weak typeof(self) weakSelf = self;
NSLog(@"Before block：%ld", CFGetRetainCount((__bridge CFTypeRef)(self)));
self.block = ^{
    weakSelf;
    NSLog(@"Within block：%ld", CFGetRetainCount((__bridge CFTypeRef)(weakSelf)));
};
self.block();
NSLog(@"After block：%ld", CFGetRetainCount((__bridge CFTypeRef)(self)));


// 🌰3
__weak typeof(self) weakSelf = self;
NSLog(@"Before block：%ld", CFGetRetainCount((__bridge CFTypeRef)(self)));
self.block = ^{
    __strong typeof(self) strongSelf = weakSelf;
    NSLog(@"Within block：%ld", CFGetRetainCount((__bridge CFTypeRef)(weakSelf)));
};
self.block();
NSLog(@"After block：%ld", CFGetRetainCount((__bridge CFTypeRef)(self)));
```

我们可以通过 Clang 对上面的三个例子做一下编译，通过编译后的 C 代码（接下来所展示代码都是经过简化），我们可以推导出 self 的引用计数变化。

---

🌰1 的 C 代码如下：

```Objective-C
// Block 结构体。这个大家可以通过其他的资料去看看，我们今天主要是来探寻一下 self 的旅程，这里就不对 Block 本身做更详细的介绍
struct __block_impl {
  void *isa;
  int Flags;
  int Reserved;
  void *FuncPtr;
};

// ^{} 的实现
struct __BlockTest__test_block_impl_0 {
    struct __block_impl impl;
    struct __BlockTest__test_block_desc_0* Desc;
    BlockTest *const __strong self;
    __BlockTest__test_block_impl_0(void *fp, struct __BlockTest__test_block_desc_0 *desc, BlockTest *const __strong _self, int flags=0) : self(_self) {
        impl.isa = &_NSConcreteStackBlock;
        impl.Flags = flags;
        impl.FuncPtr = fp;
        Desc = desc;
    }
};

// Block 方法
static void __BlockTest__test_block_func_0(struct __BlockTest__test_block_impl_0 *__cself) {
    BlockTest *const __strong self = __cself->self; // bound by copy
    self;
}

// Block 的 copy 操作
static void __BlockTest__test_block_copy_0(struct __BlockTest__test_block_impl_0*dst, struct __BlockTest__test_block_impl_0*src) {_Block_object_assign((void*)&dst->self, (void*)src->self, 3/*BLOCK_FIELD_IS_OBJECT*/);}

// Block 的 dispose 操作
static void __BlockTest__test_block_dispose_0(struct __BlockTest__test_block_impl_0*src) {_Block_object_dispose((void*)src->self, 3/*BLOCK_FIELD_IS_OBJECT*/);}

// 描述 Block 的 copy 和 dispose
static struct __BlockTest__test_block_desc_0 {
    size_t reserved;
    size_t Block_size;
    void (*copy)(struct __BlockTest__test_block_impl_0*, struct __BlockTest__test_block_impl_0*);
    void (*dispose)(struct __BlockTest__test_block_impl_0*);
} __BlockTest__test_block_desc_0_DATA = { 0, sizeof(struct __BlockTest__test_block_impl_0), __BlockTest__test_block_copy_0, __BlockTest__test_block_dispose_0};

// 方法主体
static void _I_BlockTest_test(BlockTest * self, SEL _cmd) {
    ((void (*)(id, SEL, void (*)()))(void *)objc_msgSend)((id)self, sel_registerName("setBlock:"), ((void (*)())&__BlockTest__test_block_impl_0((void *)__BlockTest__test_block_func_0, &__BlockTest__test_block_desc_0_DATA, self, 570425344)));
    ((void (*(*)(id, SEL))())(void *)objc_msgSend)((id)self, sel_registerName("block"))();

}
```

1. 在方法主体里面首先会构造一个 `__BlockTest__test_block_impl_0` 的结构体，该结构体捕获了 self；
2. `__BlockTest__test_block_impl_0` 的构造函数中使用了 `__strong` 来捕获 self，所以我们知道在构造的时候默认是使用 `__strong` 来捕获外部的对象变量，此时 self 的引用计数应该要 +1；
3. Block 被构造出来之后需要被赋值给 self，我们知道在 ARC 模式下此时的 Block 会执行 Copy 操作，从 `_NSConcreteStackBlock` 变成 `_NSMallocBlock`；
4. Block 通过 `__BlockTest__test_block_desc_0_DATA` 找到 Copy 方法的具体实现 `__BlockTest__test_block_copy_0`，从上面的代码中我们知道该方法的实现是通过 `_Block_object_assign` 来实现的（对于这个方法的实现细节暂时还没有找到更相信的资料，有知道的可以麻烦告诉一下），通过名字我们可以猜测出该方法只是把捕获的变量地址直接拷贝一份到堆内存中，但是不会引起引用计数的变化；
5. 当 Block 被真正执行的时候会通过 `__block_impl` 的 `FuncPtr` 找到真正的实现代码 `__BlockTest__test_block_func_0`，我们观察到在这个方法里面有这样一句代码 `BlockTest *const __strong self = __cself->self`，很明显此时 self 的引用计数会 +1，当该 `__BlockTest__test_block_func_0` 执行完毕之后还是会释放 self 的，此时引用计数会 -1；

从上面的分析过程中，我们知道由于 Block 在构造的时候默认就对捕获的 self 进行了强引用，导致 self 的引用计数 +1；而又由于 self 持有了 Block，所以这里就造成了循环引用的问题。

我们来看 🌰2 能不能解决这个问题？

---

🌰2 的 C 代码如下：

```Objective-C
// ^{} 结构体
struct __BlockTest__test_block_impl_0 {
    struct __block_impl impl;
    struct __BlockTest__test_block_desc_0* Desc;
    BlockTest *const __weak weakSelf;
    __BlockTest__test_block_impl_0(void *fp, struct __BlockTest__test_block_desc_0 *desc, BlockTest *const __weak _weakSelf, int flags=0) : weakSelf(_weakSelf) {
        impl.isa = &_NSConcreteStackBlock;
        impl.Flags = flags;
        impl.FuncPtr = fp;
        Desc = desc;
    }
};

// Block 方法
static void __BlockTest__test_block_func_0(struct __BlockTest__test_block_impl_0 *__cself) {
    BlockTest *const __weak weakSelf = __cself->weakSelf; // bound by copy
    weakSelf;
}

// Block 的 copy 操作
static void __BlockTest__test_block_copy_0(struct __BlockTest__test_block_impl_0*dst, struct __BlockTest__test_block_impl_0*src) {_Block_object_assign((void*)&dst->weakSelf, (void*)src->weakSelf, 3/*BLOCK_FIELD_IS_OBJECT*/);}

// Block 的 dispose 操作
static void __BlockTest__test_block_dispose_0(struct __BlockTest__test_block_impl_0*src) {_Block_object_dispose((void*)src->weakSelf, 3/*BLOCK_FIELD_IS_OBJECT*/);}

// 描述 Block 的 copy 和 dispose
static struct __BlockTest__test_block_desc_0 {
    size_t reserved;
    size_t Block_size;
    void (*copy)(struct __BlockTest__test_block_impl_0*, struct __BlockTest__test_block_impl_0*);
    void (*dispose)(struct __BlockTest__test_block_impl_0*);
} __BlockTest__test_block_desc_0_DATA = { 0, sizeof(struct __BlockTest__test_block_impl_0), __BlockTest__test_block_copy_0, __BlockTest__test_block_dispose_0};

// 方法主体
static void _I_BlockTest_test(BlockTest * self, SEL _cmd) {
    __attribute__((objc_ownership(weak))) typeof(self) weakSelf = self;
    ((void (*)(id, SEL, void (*)()))(void *)objc_msgSend)((id)self, sel_registerName("setBlock:"), ((void (*)())&__BlockTest__test_block_impl_0((void *)__BlockTest__test_block_func_0, &__BlockTest__test_block_desc_0_DATA, weakSelf, 570425344)));
    ((void (*(*)(id, SEL))())(void *)objc_msgSend)((id)self, sel_registerName("block"))();
}
```

1. 方法主体会先用 `__weak` 初始化一个 weakSelf，此时 self 的引用计数是不会发生变化的；之后会构造一个`__BlockTest__test_block_impl_0` 的结构体，该结构体捕获了 weakSelf；
2. `__BlockTest__test_block_impl_0` 的构造函数中使用了 `__weak` 来捕获 weakSelf，所以我们知道此时 self 的引用计数应该要也是不会发生变化的；
3. 然后把该结构体赋值给 self.block，block 结构体被从栈复制到堆的时候使用了 `_Block_object_assign`，所以此时 self 的引用计数不会发生变化
4. 然后 block 在被执行的时候做了一下 `__weak` 的操作 `BlockTest *const __weak weakSelf = __cself->weakSelf`，这时候 self 的引用计数也不会发生变化
5. 由于 block 对 weakSelf 没有强引用，所以在 block 执行完成之后也不需要做释放 weakSelf 的工作

所以，在该例子中 block 无法强引用 weakSelf，weakSelf 的引用计数没有发生任何变化。由于 self 没有被 block 强应用，所以当 self 要被释放的时候，block 也会被释放，这就解决了我们 🌰1 中的循环引用的问题。但是在 block 方法执行的过程中，self 对象有可能已经被释放了，此时如果你还去使用 weakSelf 就有可能造成奔溃的情况。

---

🌰3 的 C 代码如下：

```Objective-C
struct __BlockTest__test_block_impl_0 {
    struct __block_impl impl;
    struct __BlockTest__test_block_desc_0* Desc;
    BlockTest *const __weak weakSelf;
    __BlockTest__test_block_impl_0(void *fp, struct __BlockTest__test_block_desc_0 *desc, BlockTest *const __weak _weakSelf, int flags=0) : weakSelf(_weakSelf) {
        impl.isa = &_NSConcreteStackBlock;
        impl.Flags = flags;
        impl.FuncPtr = fp;
        Desc = desc;
    }
};

static void __BlockTest__test_block_func_0(struct __BlockTest__test_block_impl_0 *__cself) {
    BlockTest *const __weak weakSelf = __cself->weakSelf; // bound by copy
    __attribute__((objc_ownership(strong))) typeof(self) strongSelf = weakSelf;    
}

static void __BlockTest__test_block_copy_0(struct __BlockTest__test_block_impl_0*dst, struct __BlockTest__test_block_impl_0*src) {_Block_object_assign((void*)&dst->weakSelf, (void*)src->weakSelf, 3/*BLOCK_FIELD_IS_OBJECT*/);}

static void __BlockTest__test_block_dispose_0(struct __BlockTest__test_block_impl_0*src) {_Block_object_dispose((void*)src->weakSelf, 3/*BLOCK_FIELD_IS_OBJECT*/);}

static struct __BlockTest__test_block_desc_0 {
    size_t reserved;
    size_t Block_size;
    void (*copy)(struct __BlockTest__test_block_impl_0*, struct __BlockTest__test_block_impl_0*);
    void (*dispose)(struct __BlockTest__test_block_impl_0*);
} __BlockTest__test_block_desc_0_DATA = { 0, sizeof(struct __BlockTest__test_block_impl_0), __BlockTest__test_block_copy_0, __BlockTest__test_block_dispose_0};

static void _I_BlockTest_test(BlockTest * self, SEL _cmd) {
    __attribute__((objc_ownership(weak))) typeof(self) weakSelf = self;
    ((void (*)(id, SEL, void (*)()))(void *)objc_msgSend)((id)self, sel_registerName("setBlock:"), ((void (*)())&__BlockTest__test_block_impl_0((void *)__BlockTest__test_block_func_0, &__BlockTest__test_block_desc_0_DATA, weakSelf, 570425344)));
    ((void (*(*)(id, SEL))())(void *)objc_msgSend)((id)self, sel_registerName("block"))();  
}
```


前面的步骤都跟 🌰2 中的一样，关键是在 Block 的方法实现里面有点不一样。我们来看看 `__BlockTest__test_block_func_0`，它首先调用了 `BlockTest *const __weak weakSelf = __cself->weakSelf`， 所以它此时的引用计数不会发生变化；但是接下来又用 `objc_ownership(strong)` 来强引用 weakSelf，所以此时 self 的引用计数 +1。这就保证了在函数执行的过程中，Block 会一直持有 self，知道 Block 执行完毕之后会释放 weakSelf。

所以 🌰3 完美的解决了循环应用和直接使用 `__weak` 可能导致奔溃的问题。

---

最后，说一下关于 _Block_object_assign 的猜想：

```
_Block_object_assign((void*)&dst->weakSelf, (void*)src->weakSelf, 3/*BLOCK_FIELD_IS_OBJECT*/);
```

通过上面的例子，我们知道 Block 在构造的时候就会对捕获的变量进行内存管理（强引用和弱引用），所以当 Block 在做 Copy 操作的时候其实没有必要对它捕获的变量再做一遍内存管理了。这也应该是 Block 的 Copy 操作使用了 `_Block_object_assign` 这种不会导致引用计数发生变化的方式来实现的原因。




