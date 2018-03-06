---
layout: post 
title: #include、#import和@import的区别
subtitle: 
author: 帕帕
date: 2017-09-05 15:30:56 +0800
categories: iOS 
tag: iOS 
---
 
今天我们来了解下面这几种包含文件的方式有什么特点和区别：

```
#include "fiel"
#include <file>
#import "file"
#import <file>
@import Module
```

---

## 一、#include

学过 C 语言的人都知道，#include 其实是一个预处理命令。它会在预处理的时候简单的把被#include 包含的文件内容进行复制粘贴。我们来看看下面的代码：

```
// A.h
void sampleA() {
  // A code
}

```

```
// B.h
#include "A.h"

void sampleB() {
  // B code
}
```

我们使用 gcc -E B.h 命令来看看经过预处理后的文件内容大概如下：

```
# 1 "B.h"
# 1 "<built-in>" 1
# 1 "<built-in>" 3
# 329 "<built-in>" 3
# 1 "<command line>" 1
# 1 "<built-in>" 2
# 1 "B.h" 2
# 1 "./A.h" 1
void sampleA() {

}
# 2 "B.h" 2

void sampleB() {

}
```

我们可以看到经过预处理之后，A.h 文件中的内容被直接复制并粘贴到 B.h 文件中来。如果我们在 B.h 文件中多次包含了 A.h 文件，会出现什么情况？比如：

```
// A.h
void sampleA() {
  // A code
}
```

```
// B.h
#include "A.h"
#include "A.h"

void sampleB() {
  // B code
}
```

经过预处理之后的内容大概如下：

```
# 1 "B.h"
# 1 "<built-in>" 1
# 1 "<built-in>" 3
# 329 "<built-in>" 3
# 1 "<command line>" 1
# 1 "<built-in>" 2
# 1 "B.h" 2
# 1 "./A.h" 1
void sampleA() {

}
# 2 "B.h" 2
# 1 "./A.h" 1
void sampleA() {

}
# 3 "B.h" 2

void sampleB() {

}
```

A.h 文件中的 sampleA() 函数出现了两次，所以我们需要利用其他的一些预处理命令来规避这种情况，看看下面的代码：

```
// A.h
#ifndef FILE_A
#define FILE_A

void sampleA() {
  // A code
}
#endif
```

```
// B.h
#include "A.h"
#include "A.h"

void sampleB() {
  // B code
}
```

我们再来看看增加了这些预处理命令之后的预处理文件内容：

```
# 1 "B.h"
# 1 "<built-in>" 1
# 1 "<built-in>" 3
# 329 "<built-in>" 3
# 1 "<command line>" 1
# 1 "<built-in>" 2
# 1 "B.h" 2
# 1 "./A.h" 1



void sampleA() {

}
# 2 "B.h" 2


void sampleB() {

}
```

OK，这就正常了。如果我们在 A.h 中包含 B.h，然后又在 B.h 中包含 A.h，具体代码如下：

```
// A.h
#include "B.h"

void sampleA() {
  // A code
}
#endif
```

```
// B.h
#include "A.h"

void sampleB() {
  // B code
}
```

我们再来看看经过 gcc -E B.h 处理之后的文件内容：

```
# 1 "B.h"
# 1 "<built-in>" 1
# 1 "<built-in>" 3
# 329 "<built-in>" 3
# 1 "<command line>" 1
# 1 "<built-in>" 2
# 1 "B.h" 2
# 1 "./A.h" 1
# 1 "./B.h" 1
# 1 "./A.h" 1
# 1 "./B.h" 1
...
...
# 1 "./A.h" 1
# 1 "./B.h" 1
In file included from ./B.h:1:
In file included from ./A.h:1:
In file included from ./B.h:1:
In file included from ./A.h:1:
...
...
In file included from ./B.h:1:
In file included from ./A.h:1:
./A.h:1:10: error: #include nested too deeply
#include "B.h"
         ^


void sampleA() {

}
# 2 "./B.h" 2

void sampleB() {

}
# 2 "./A.h" 2

void sampleA() {

}
# 2 "./B.h" 2

void sampleB() {

}
# 2 "./A.h" 2

void sampleA() {

}
...
...
# 2 "./A.h" 2

void sampleA() {

}
# 2 "./B.h" 2

void sampleB() {

}
1 error generated.
```

我们发现 A.h 和 B.h 重复出现，这是因为这个时候 A.h 和 B.h 文件互相引用导致的。从理论上来讲，这个时候会无限循环下去，直至世界终结。在这里最后会出现一句 *1 error generated.*的提示是 gcc 强行中断了这个预处理的过程，所以我们才能看到这样的结果。那我们可以怎么做？当然是利用前面说的预处理命令来避免循环引用的问题。看下面的代码：

```
// A.h
#ifndef FILE_A
#define FILE_A

#include "B.h"

void sampleA() {
  // A code
}
#endif
```

```
// B.h
#ifndef FILE_B
#define FILE_B

#include "A.h"

void sampleB() {
  // B code
}
#endif
```

这个时候使用 gcc -E B.h 就可以正常的进行预处理，最后的结果如下：

```
# 1 "B.h"
# 1 "<built-in>" 1
# 1 "<built-in>" 3
# 329 "<built-in>" 3
# 1 "<command line>" 1
# 1 "<built-in>" 2
# 1 "B.h" 2



# 1 "./A.h" 1



# 1 "./B.h" 1
# 5 "./A.h" 2

void sampleA() {

}
# 5 "./B.h" 2

void sampleB() {

}
```

所以C程序员总是需要通过各种手段（比如：[#pragma once](https://en.wikipedia.org/wiki/Pragma_once)）来防范此类事件的发生。


## 二、#import

我们在文件中通过#import来导入 iAd Framework：
![](http://i.imgur.com/nLPSsNN.jpg)


编译报错：
![](http://i.imgur.com/XBXD8wu.jpg)

需要重新导入和链接 Framework：
![](http://i.imgur.com/rUnKJGb.jpg)
![](http://i.imgur.com/XuxVI6b.jpg)

编译成功：
![](http://i.imgur.com/QvyQunr.jpg)

从上面的过程中我们就知道在 Objective-C 项目中使用 #import 需要注意导入和链接 Framework，否则是会报错的。

预处理器在碰到 #import 命令的时候，它会采用递归的方式把被所有头文件的内容复制并粘贴到当前文件中，如果文件依赖层次比较深就会造成预处理后的文件内容体积大幅度变大。

比如导入 UIKit 的时候只需要一行代码：

```Objective-C
#import <UIKit/UIKit.h>
```

预处理之后会变成200多行（UIKit.h 文件有200多行代码）：

```Objective-C
#import <UIKit/UIKitDefines.h>

#if __has_include(<UIKit/UIAccelerometer.h>)
#import <UIKit/UIAccelerometer.h>
.....
#import <UIKit/UIRegion.h>
#endif
```

接下来还需要递归的把每个头文件的内容展开，最后的结果就是一行代码变成超过11000行代码。如果有多个文件都包含来 UIKit 的头文件，这样就会让每个文件的体积都会变得很大，编译过程也会变得越来越慢。这种递归的方式会让项目的编译时间变成：*M source files + N headers => M x N compile time*。

所以这个时候有一个优化方法就是把项目中频繁被引用的文件放到 PCH（Pre-Compile Header）文件中。PCH 会被编译一次并且会被缓存，这就可以缩短编译时间，我们也不需要在不同的文件里面添加import语法。

当然，PCH 也有自己的缺点：

* 维护负担：随着项目变得越来越复杂，我们就会不停的往PCH文件加入内容，内容一旦变多就会变得不好维护。（这也是我们平常在项目中要避免在 ViewController 做太多事情的，要研究 MVVM的缘故。）

* 命名空间污染


最后，给大家提供一个例子看看 #import 编译出来之后的文件内容：

```
// A.h
#import "B.h"
 
void sampleA() {
  // A code
}
#endif
```

```
// B.h
#import "A.h"
#import "A.h"

void sampleB() {
  // B code
}
```

使用 gcc -E B.h 进行预处理之后的内容如下：

```

# 1 "B.h"
# 1 "<built-in>" 1
# 1 "<built-in>" 3
# 329 "<built-in>" 3
# 1 "<command line>" 1
# 1 "<built-in>" 2
# 1 "B.h" 2
# 1 "./A.h" 1


void sampleA() {

}
# 2 "./B.h" 2

void sampleB() {

}
```

我们在B.h中有两个 #import "A.h"，但是这些内容跟我们之前在 A.h 和 B.h 文件中使用 #include 和其他预处理命令之后的处理结果很相似，所以我们就明白了 #import 大概做了什么事。

## 三、@import

在2012年的 LLVM 大会上，苹果的 Doug Gregor 首次提出了 Objective-C 中的 Module。使用 @import 方式导入有几个好处：

* 不需要像 #import 一样得手动去链接 Framework，@import会自动去链接

* @import 工作方式和 PCH 很像，但是 @import 要比 PCH 的效率高出许多

* @import 导入 Modul 优化文件体积变大、编译速度变慢的问题

* 可以部分导入（@import Framework.A）或全部导入（@import Framework）

所以，建议大家尽量使用 @import 来导入文件。如果你以前的项目用的是 #import，那么你也不需要担心，我们只通过 Build Settings 开启 Modules 选项（看下图），#import 和 #include 会自动被映射成 @import，所以你不需要更改原来的代码也能享受 @import带来的好处。

![](http://i.imgur.com/l7ZMUy6.jpg)

详细内容可以看看苹果2013年的 [Advances in Objective-C](https://developer.apple.com/videos/play/wwdc2013/404/)，里面就详细介绍了 Module。

## 四、文件路径

接下来我们来了解一下 *#include <file>* 和 *#include "file"*：

* \#include \<file>: 表示编译器会直接到系统设定的目录下寻找指定的文件。
 
* \#include "file": 表示编译器会到当前的目录下寻找指定的文件，如果找不到，则会去系统设定的目录下寻找指定的文件。

---
参考文献：

1. https://gcc.gnu.org/onlinedocs/cpp/Include-Syntax.html

2. http://stackoverflow.com/questions/18947516/import-vs-import-ios-7

3. https://www.raywenderlich.com/49850/whats-new-in-objective-c-and-foundation-in-ios-7

