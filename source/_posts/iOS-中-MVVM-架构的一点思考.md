---
title: iOS MVVM 架构的一点思考
date: 2023-4-2 16:58:32 +0800
author: 帕帕
categories: 技术
tags: [iOS]
thumbnail: https://images.unsplash.com/photo-1514464750060-00e6e34c8b8c?ixlib=rb-0.3.5&ixid=eyJhcHBfaWQiOjEyMDd9&s=3749c47dd7beec20102c6b32fc19833a&auto=format&fit=crop&w=160&q=100
---

## MVVM 的基本介绍

MVVM（Model-View-ViewModel）是一种基于 MVC（Model-View-Controller）模式的衍生设计模式，它将应用程序分为三个部分：模型（Model）、视图（View）和视图模型（ViewModel）。

1. 模型（Model）
   模型是应用程序中的数据和业务逻辑层，它是一个独立的组件，负责数据的存储和处理。模型通常是被动的，也就是说它不知道视图和视图模型的存在，只负责提供数据和业务逻辑的处理。 **模型可以包含数据模型、网络模型、数据库模型等等，它们都负责对数据的获取、处理和存储**。
2. 视图（View）
   视图是应用程序的用户界面，它负责将模型中的数据展示给用户，并接收用户的操作反馈。视图负责应用程序的外观和交互，并且需要与用户进行直接的交互。视图通常由图形用户界面（GUI）组成，它包含了各种控件和布局信息，例如按钮、文本框、列表等等。
3. 视图模型（ViewModel）
   视图模型是 MVVM 模式的核心部分，它充当了视图和模型之间的中间层，将模型中的数据转换成视图可以使用的数据，并将视图反馈转换成模型可以处理的操作。视图模型还负责处理用户交互逻辑，例如校验输入、调用模型方法等。 **视图模型通过绑定机制将数据和视图绑定起来，当数据发生变化时，视图会自动更新**。

总之，MVVM 模式通过将应用程序分为三个部分来分离用户界面和业务逻辑，使得应用程序更加容易维护和扩展。模型和视图模型之间使用双向数据绑定机制，使得视图模型和视图之间的通信变得更加轻松和高效。在实际开发中，MVVM 模式已经被广泛应用于 iOS、Android 和 Web 等平台的应用程序中。

## MVVM 的正常架构

![图一](https://i.imgur.com/nAFi2ds.png)

这是一个很经典的 MVVM 架构图，但是当我们细化到 iOS 开发的时候这个架构图就有简单了。我们可以思考下面的几个问题：

1. Controller 在这里被当成一个特殊的 View 对待的，合理么？
2. View 层持有 ViewModel，方便测试么？
3. Model 究竟是 Fat (胖) 还是 Thin (瘦) ？

## MVVM 的 iOS 架构

![图二](https://i.imgur.com/d9IeLxf.png)

### Controller（控制器）

在 iOS 中，Controller（控制器）通常被放在 View 层中处理，但是这种做法可能会导致代码的混乱和耦合度的增加。

在传统的 iOS 开发中，Controller 负责将数据从模型传递给视图，并响应视图的事件和用户交互。然而，随着应用程序变得越来越复杂，Controller 所承担的责任也变得越来越多，它可能包含大量的业务逻辑和用户交互代码，导致代码难以维护和测试。此外，Controller 与特定的视图耦合度较高，使得重用和测试变得更加困难。

因此，为了避免这些问题，我们可以将 Controller 的职责进行分离，将视图控制和业务逻辑分别放置在不同的层中。在这种情况下，Controller 可以变得更加轻量级，并将其主要职责放在视图控制上，而将业务逻辑和用户交互放在其他层中。

一种常见的做法是采用 MVVM 模式，将视图控制和视图模型分离开来。视图模型负责管理视图所需的数据和逻辑，并提供双向绑定机制，将视图与模型解耦。Controller 则只需要负责处理视图的生命周期和响应路由，将视图和视图模型连接起来。

因此，虽然 Controller 在 iOS 中通常被放在 View 层中处理，但是为了避免代码的混乱和耦合度的增加，我们应该将其职责进行分离，并采用合适的模式来进行架构设计。

```Swift
final class ViewController: UIViewController {

    override func viewDidLoad() {
        super.viewDidLoad()
        // Do any additional setup after loading the view.
        setupView()
        setupViewModel()
        setupBuriedPoint()
    }
    // MARK: View & ViewModel
    func setupView() {
        view.addSubview(aView)
    }

    func setupViewModel() {
        aView.bind(viewModel: aViewModel)
    }

    // MARK: 埋点 & 日志
    func setupBuriedPoint() {
        aViewModel.searchEvent.subscribe {
        }
        aViewModel.likeEvent.subscribe {
        }
    }

    // MARK: Life Cycle
    override func viewDidAppear(_ animated: Bool) {
        super.viewDidAppear(animated)
    }

    override func viewDidDisappear(_ animated: Bool) {
        super.viewDidDisappear(animated)
    }

    // MARK: - View
    private let aView = AView()

    // MARK: - ViewModel
    private let aViewModel = AViewModel()

    // MARK: - Business Logic / Network Request / Event Handle 统统都不见了
}
```

### View 和 ViewModel 数据绑定

双向绑定：双向数据绑定是指在数据模型和视图控件之间建立双向的关联，当数据模型发生变化时，自动更新关联的视图控件；当视图控件发生变化时，自动更新关联的数据模型。实现双向数据绑定可以提高应用程序的开发效率和用户体验，降低代码的复杂度和维护成本。

单向绑定：单向数据绑定是指在数据模型和视图控件之间建立单向的关联，当数据模型发生变化时，自动更新关联的视图控件，但是当视图控件发生变化时，不会自动更新关联的数据模型。

![图三](https://i.imgur.com/mPKsPUN.png)

### Model 应该是 Fat 还是 Thin

Fat Model 是指将所有的业务逻辑和数据操作都放在 Model 层中实现。在这种设计模式下，Model 层会变得比较庞大，甚至会包含一些与业务逻辑无关的代码，因此被称为“Fat Model”。

Thin Model 是指将 Model 层中的业务逻辑和数据操作进行拆分，只保留最基本的数据存储和数据操作功能。在这种设计模式下，业务逻辑和数据操作通常被放在其他层中实现，比如 Controller 层或者 Service 层，因此 Model 层相对来说比较“瘦”，被称为“Thin Model”。

在过去的项目中，我通常会将 Model 层定义为介于 Fat 和 Thin 之间，因为我会将跟 Model 相关的网络请求和弱业务逻辑的代码放在这一层。

我们来看看一般情况下在 ViewModel 层如何处理数据和网络相关的代码：

```Swift
// ViewModel 层处理网络和数据的伪代码
do {
    let response = try await Network.request(method, path, parameter)
    if response.code == 200 {
        do {
            let model = try AModel.decode(from: response.data)
            AModel.saveToCache()
        } catch {
            // 处理 decode 的异常情况
            if let model = AModel.getFromCache() {

            } else {
                // 没有缓存返回 error
            }
        }
    } else {
        // 处理非 200 的业务代码
    }
} catch {
    // 处理网络请求的 error
}
```

实际上，上述的网络请求、业务代码处理以及数据解析在每个 Model 中都大致相同（缓存处理属于弱业务逻辑的一部分，因为每个 Model 的数据缓存策略可能会有一些不同的业务需求）。因此，我们希望将这部分代码封装起来以便复用。有些项目可能会新建工具类来处理这部分代码，但我通常会将这些代码放在 Model 的 Extension/Category 中处理。

```Swift
// AModel.Extension.Swift
extesion AModel {
    static fetch() async throws -> AModel {
        do {
            let response = try await Network.request(method, path, parameter)
            if response.code == 200 {
                do {
                    let model = try AModel.decode(from: response.data)
                    AModel.saveToCache()
                } catch {
                    // 处理 decode 的异常情况
                    if let model = AModel.getFromCache() {

                    } else {
                        // 没有缓存返回 error
                        throw Error.cache
                    }
                }
            } else {
                // 处理非 200 的业务代码
                throw Error.data
            }
        } catch {
            // 处理网络请求的 error
            throw Error.network
        }
    }
}


// AViewModel.Swift
final class AViewModel {
    func fetch() {
        do {
        	let model = try await AModel.fetch()
        } catch {
            // 处理不同类型的 error
        }
    }
}
```

> 弱业务逻辑（Weak Business Logic）是指应用程序中的业务逻辑缺乏严谨性或者可扩展性，并且散落在项目中的各个地方，导致应用程序难以维护和扩展。现在我们通过上面的优化相当把很多弱业务逻辑的代码放到统一的层级来进行管理，有利于我们的代码维护。

对于图形界面来说，最重要的是拿到数据进行页面渲染，并响应用户的输入并交给数据去处理。它们并不需要关心数据的来源和如何处理数据。通过我们之前的优化，ViewModel 基本上不需要知道网络和缓存的存在。Model 层的数据可能是从网络系统、文件系统、本地缓存等方式获取的，但是这些细节 ViewModel 不需要关心。

**Model 可以包含数据模型、网络模型、数据库模型等等，它们都负责对数据的获取、处理和存储。**

---

参考文献：

1. [iOS Architecture Patterns Demystifying MVC, MVP, MVVM and VIPER](https://medium.com/ios-os-x-development/ios-architecture-patterns-ecba4c38de52#.tliwdfd60)
2. [iOS 应用架构谈 view 层的组织和调用方案](https://casatwy.com/iosying-yong-jia-gou-tan-viewceng-de-zu-zhi-he-diao-yong-fang-an.html)
