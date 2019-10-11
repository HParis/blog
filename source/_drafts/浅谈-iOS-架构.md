---
title: 浅谈 iOS 架构
date: 2019-03-04 16:58:32 +0800
author: 帕帕
categories: 技术
tags: [iOS]
thumbnail: https://images.unsplash.com/photo-1514464750060-00e6e34c8b8c?ixlib=rb-0.3.5&ixid=eyJhcHBfaWQiOjEyMDd9&s=3749c47dd7beec20102c6b32fc19833a&auto=format&fit=crop&w=160&q=100
---

## 架构还是框架
老实说，一开始我对这两个概念还是很模糊的。这段时间通过不断的思考和学习，从我个人的理解来说架构和框架有一个根本性的区别（当然，对架构和框架的理解会随着不停的深入而有所不同）：de

> 架构：原则指导
> 框架：规则约束

架构设计可以通过一些原则（比如单一职责原则、开放封闭原则、接口分离原则等等）来指导我们进行设计，设计的过程中需要考虑业务和团队的因素。

框架其实是一套规则，它约束着你应该怎么去做实现。比如我们常见的 MVC、MVVM、MVP、VIPER 等等，这些都是属于框架的范畴。框架是可以被具现化的，这是它和架构的一个不同点。比如我们常说的 redux 框架，它就有具体的实现 [redux.js](https://redux.js.org/)，我将这成为框架的具现化。

## 架构设计
我们都知道用户对客户端的最重要和最直接的感受就是 UI，而 UI 的内容是需要用数据来填充的。所以，对于客户端的架构设计，我们大体上可以从这两个维度来设计。

### UI 设计
UI 的架构设计基本上没有什么好说的。这里我们需要注意的就是把控件分成两种类型，基础控件和业务组件。业务组件是指跟业务强相关的控件，一旦换了业务就代表这个控件可能不再可用；基础控件基本上就跟业务无关了，它主要是确定了整个 UI 的风格和基调，并且它可以比较方便的被移植。

在 iOS 上还有一个需要注意的是，不应该在 UITableViewCell、UICollectionViewCell、UICollectionReusableView 做具体的 UI 实现，而是应该在 UIView 或 UIResponse 的子类中做具体的 UI 实现，然后在 UITableViewCell、UICollectionViewCell、UICollectionReusableView **「贴上」**具体的 UI 实现。假设你用 UITableViewCell 直接做 UI 细节的实现，一旦当前页面要换成 UICollectionView 实现的时候，我们就很难直接复用 UITableViewCell 的 UI 实现了。

### 数据设计

这里需要注意的是，网络层和持久层所处理的数据都是原始数据，数据层需要将原始数据 Model 化，交给上层使用。



## 框架设计
在 iOS 开发中，主要的框架有 MVC、MVP、MVVM、VIPER 等等，至于这几个架构的优缺点可以看看「[iOS Architecture Patterns Demystifying MVC, MVP, MVVM and VIPER](https://medium.com/ios-os-x-development/ios-architecture-patterns-ecba4c38de52#.tliwdfd60)」这篇文章。

其中 MVC(Model-View-Controller) 是最为开发者所熟悉的，同时也是用的最多的。但随着时间的推移和业务的发展，MVC 框架已经不能满足我们的需求。因为所有的业务逻辑都围着 C 转，C 变得越来越臃肿，越来越难以维护。于是广大的开发者又探讨出来几种其他的框架，有些是基于 MVC 框架改造的，有些是借鉴了其他平台的主流框架（比如 redux）。

在框架选择上，我主要选择 MVVM 框架。主要是考虑到团队成员都有开发 iOS 的背景，所以大家都比较熟悉原来的 MVC 框架，而 MVVM 是在 MVC 的框架上做改进，所以团队的成员要上手 MVVM 也会比较容易，推广起来也会容易点。如果有一天，当我们使用跨平台的技术栈来实现我们的业务，那我可能就不会选择 MVVM 了，有可能就选在 redux。所以选择框架的时候还是要根据自身的业务和团队情况选择合适的框架。

接下来我会主要讲讲 MVVM，包括对 MVVM 的一些改变吧。

### MVVM

![图一](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Collections/Art/CopyingCollections_2x.png)

在图一这个经典的 MVVM 框架图中，总共有四个角色。

* Model:
Model 只是简单的做数据展示和通知数据发生变化的工作，当然你也可以用 KVO 或者 RX 框架来主动监听 Model 数据的变化。

* View:
View 也只是简单的处理 UI 的布局和变化。

* ViewModel:
ViewModel 基本上承接了以前 Controller 的大部分工作，它需要处理各种业务逻辑（数据缓存、网络请求、事件处理等等）。

* Controller:
Controller 现在同时拥有 View 和 ViewModel，并且它的主要工作基本上就是绑定。
ViewModel 会有一系列 States 和 Actions，当在 Controller 中完成了绑定的工作，那么 ViewModel 的 State 发生了变化就会引起 View 的变化，View 的事件都会交给 ViewModel 的 Action 进行处理。

现在假设我们有两个按钮「收藏」和「保存」可以对图片进行操作，其中收藏按钮通过点击之后会向服务器发送请求，请求成功之后收藏按钮的文案变成「已收藏」；点击保存按钮之后就直接把图片保存到本地相册。以前我们会把这一系列逻辑代码都写到 Controller，那现在如果用 MVVM 可以会变成什么样呢？看下面代码：

```Swift
class FavViewModel {
    init(id: String) {
        // init
    }
    
    // MARK: - States
    private var title: String = "收藏" {
        didSet(oldValue) {
            guard oldValue != title else {
                return
            }
            DispatchQueue.main.async {
                self.titleChange?(self.title)
            }
        }
    }
    var titleChange: ((String) -> Void)? {
        didSet {
            DispatchQueue.main.async {
                self.titleChange?(self.title)
            }
        }
    }
    
    // MARK: - Actions
    @objc func favAction() {
        // Async Request
        title = "已收藏"
    }
}
class SaveViewModel {
    init(image: UIImage) {
        // init
    }
    
    // MARK: - States
    let title = "保存" 
        
    // MARK: - Actions
    @objc func saveAction() {
        // Async save image
    }
}

class Controller: UIViewController {  
    override func viewDidLoad() {
        view.addSubview(favButton)
        view.addSubview(saveButton)
        
        // 绑定 FavButton 和 FavViewModel
        favButton.addTarget(favViewModel, action: #selector(FavViewModel.favAction), for: .touchUpInside)
        favViewModel.titleChange = { [unowned self] title in
            self.favButton.setTitle(title, for: .normal)
        }
        
        // 绑定 SaveButton 和 SaveViewModel
        saveButton.setTitle(saveViewModel.title, for: .normal)
        saveButton.addTarget(saveViewModel, action: #selector(SaveViewModel.saveAction), for: .touchUpInside)
    }
    
     // MARK: - Lazy
    lazy var favButton = { () -> UIButton in
        return UIButton()
    }()
    lazy var favViewModel = { () -> FavViewModel in
        return FavViewModel.init(id: self.id)
    }()
    
    lazy var saveButton = { () -> UIButton in
        return UIButton()
    }()
    lazy var saveViewModel = { () -> SaveViewModel in
        return SaveViewModel.init(image: self.image)
    }()
}
```

👆的代码基本上展示了如何在 Swift 中实现经典的 MVVM 框架，其中有几个需要注意的地方：
1. 当给 titleChange 赋值的时候需要立即调用 titleChange(title)
2. 需要在主线程调用 titleChange(title)

> 上面的代码没有用到任何第三方框架就可以实现 MVVM，当然如果使用了 Rx 之类的框架可以让我们在代码上和绑定逻辑上可以写得更简洁。

上面的例子在实际的实践过程中还是有一些问题的：
1. 如果有多个 Controller 都用到相同的 View，那在每个 Controller 中都要做一遍绑定的工作么？
2. 如果有不同的 ViewModel 用到相同的 Model，那从网络请求到转换成 Model 到本地缓存这一些逻辑都要再写一遍么？

我的处理方法是在这里加一层 Extension(⚠️，这里的 Extension 只是为了表明对原有功能的扩展，它和 Objective-C 中 Extension 的概念不一样哦。在 Objective-C 中可以用 Category 来实现这一层扩展，当然在 Swfit 中可以是 Extension)，看下面的图二：

![图二](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Collections/Art/CopyingCollections_2x.png)

首先我们注意到 Controller 和 ViewMode 的关系从原来的「实线 Owns」变成「虚线 Create」，这表明了 Controller 不再持有 ViewModel，而是创建 ViewModel 然后分配给不同的 View。从图中可以看到 View 持有了 ViewModel，同时在 View 的 Extension 层完成了绑定的工作。首先，这样就解决了我们之前的问题，实现在不同的 Controller 中重复完成绑定的工作。

为什么要在 View 的 Extension 中做这个绑定并且持有 ViewModel 的操作？首先，我们要明白其实这个绑定工作是和 View 有强联系的。而且 View 和不同 ViewModel 的绑定都可以统一到这个扩展层，方便管理。

所以我们之前的例子可以稍微做一下修改：
```Swift
extension UIButton {
    func bind(viewModel: FavViewModel) {
        addTarget(viewModel, action: #selector(FavViewModel.favAction), for: .touchUpInside)
        viewModel.titleChange = { [unowned self] title in
            self.setTitle(title, for: .normal)
        }
    }
    
    func bind(viewMode: SaveViewModel) {
        setTitle(viewMode.title, for: .normal)
        addTarget(viewMode, action: #selector(SaveViewModel.saveAction), for: .touchUpInside)
    }
}

class ControllerA: UIViewController {
    override func viewDidLoad() {
        view.addSubview(favButton)
        view.addSubview(saveButton)
        
        favButton.bind(viewMode: FavViewModel.init(id: self.id))
        saveButton.bind(viewMode: SaveViewModel.init(image: self.image))
    }
}

class ControllerB: UIViewController {
    override func viewDidLoad() {
        view.addSubview(favButton)
        
        favButton.bind(viewMode: FavViewModel.init(id: self.id))
    }
}
```

我们可以看到，ControllerA 和 ControllerB 的工作变得异常简单，创建 ViewModel 然后传给 View 的 Extension 层去做绑定。

类似的，Model 这一边也可以增加 Extension 层来统一管理 Model 的缓存、数据请求、JSON 转 Model 还有一些其他的弱业务逻辑处理。但是如果这样做，本来一个纯数据展示的「瘦 Model」就变成一个带逻辑的「胖 Model」（关于「胖 Model」和「瘦 Model」的讨论，推荐看一下这篇文章「[iOS应用架构谈 view层的组织和调用方案](https://casatwy.com/iosying-yong-jia-gou-tan-viewceng-de-zu-zhi-he-diao-yong-fang-an.html)」）。不过因为我们是用 Extension 机制来实现的，不会对 Model 的数据造成额外的影响。当我们需要迁移 Model 的时候只需要把 Extension 这一层舍弃掉就可以了。至于是「胖 Model」还是「瘦 Model」，我觉得问题都不大，只要这些工作都是围绕 Model 本身来实现的，然后在整个架构中取得一个平衡就可以了。

所以修改后的 MVVM 框架图如下：

![图三](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/Collections/Art/CopyingCollections_2x.png)

> 注意：图三中还有一个 Router，它主要是来实现模块和模块之间的跳转，后面我们会继续深入了解。这里就先放着，暂不做详细的说明。

这里我们以一个稍微复杂点的例子来说明一下，假设我们有一个新闻列表页面，新闻列表只需要显示标题；点击某条新闻之后进入新闻详情页面，在新闻详情页面可以点赞这条新闻。



目的：
1. 指导代码的实现
2. 代码可单元测试
3. 高内聚，低耦合
4. 提高代码的可复用性


MVC框架：

优点：
1. 其概念简单，易于理解，iOS 开发附赠
2. 其中 View 和 Model 的可维护性和可测试性较好

缺点：
1. Controller 职责过于复杂，导致其可维护性和可测试性特别差
2. Model 和 View 不一定能一一对应起来，需要在 Controller 中
3. 任务分配极其不合理，View 和 Model 的任务过于轻松
4. 代码可读性差


MVVM 框架：

优点：
1. 基于 MVC 改进而来，上手难度不大
2. Controller 变得轻量级，维护方便
3. Model 不和 View 直接对应，而是通过 ViewModel 做了转换之后可以一一对应
4. 开发人员可以专注于业务逻辑和数据的开发 ViewModel，设计人员可以专注于页面设计

缺点：
1. ViewModel 容易变得跟 Controller 一样臃肿
2. 如何把一个页面拆分成不同的 ViewModel 来实现，会增加复杂性
3. 数据存储可能会不一致
4. 过于简单的图形界面不适用


MVVM + Extension 框架

优点：
1. 任务拆分相对合理，每个角色都有自己的任务
2. Extension 跟 View 和 Model 形成强绑定关系，一旦 View 或 Model 被移除掉，相对应的 Extension 也可以不要
3. 每个角色的功能比较清晰，在开发相应角色时只需要完成该角色本身的功能即可，开发流程相对固定
4. 代码重用度高

缺点：
1. 代码文件多增多，增加切换文件的成本
2. 上手难度较大，需要额外理解 Extension 的作用
3. 过于简单的图形界面不适用


架构方案 A：三层结构是由于之前的 MVC 框架导致的，因为在 MVC 框架中会不小心就把所有代码都放到 Controller 实现。

优点：
1. 业务开发快速
2. 层级简单，开发基本上只需要关心业务层的开发
3. 开发周期快

缺点：
1. 代码重用度不高，导致相同的功能在不同模块可能会出现实现逻辑不一致的问题
2. 测试困难，业务层涵盖的功能和任务太多
3. 项目可理解性差

架构方案 B：
优点：
1. 代码重用度高
2. 把业务（变化快）和服务（变化慢）分层实现，有利于项目的维护以及在后续开发中节省时间
3. 模块与模块间的耦合性低
4. 代码可以单元测试
5. 容易理解（万岁）

缺点：
1. 增加了把哪些内容下沉到服务以及如何拆分服务的复杂度
2. 模块跳转时需要知道目标模块的地址（字符串），需要单独维护一个表来实现跳转的功能
3. 
