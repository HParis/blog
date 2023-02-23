---
title: 关于 iOS 中 View 和 Cell 的思考
date: 2023-02-23 14:35:06
author: 帕帕
categories: 技术
tags: [iOS]
thumbnail: https://images.unsplash.com/photo-1507289872412-523fc6b2db5f?ixlib=rb-0.3.5&ixid=eyJhcHBfaWQiOjEyMDd9&s=20ca9d0eba2016344894aec7bb453a2d&auto=format&fit=crop&w=160&q=100
---

## Cell 组件

相信很多 iOS 开发同学在拿到一个列表的需求的时候，会习惯性的写出一个 Cell 的组件，比如下面的课程组件。

![列表组件](https://i.imgur.com/Z96M8LW.jpg)

```Swift
// 定义课程的 Cell 组件
final class CourseCell: UITableCell {
    // ...

    let coverView = CourseCoverView()
    let infoView = CourseInfoView()
}

// 定义课程的封面组件
final class CourseCoverView: UIView {
    // ...
}

// 定义课程的详细信息组件
final class CourseInfoView: UIView {
    // ...
}

```

如果现在又来一个新的需求，需要在列表的最上面放一排可以横向滚动的封面图，下面依旧还是课程列表。这里使用 UICollectionView 来实现我们的这个需求，那么需要修改的代码如下：

```Swift
// 定义课程的 Cell 组件
final class CourseCell: UICollectionViewCell {
    // ...

    let coverView = CourseCoverView()
    let infoView = CourseInfoView()
}

// 定义课程封面的 Cell 组件
final class CourseCoverCell: UICollectionViewCell {
    // ...

    let coverView = CourseCoverView()
}

// 定义课程的封面组件
final class CourseCoverView: UIView {
    // ...
}

// 定义课程的详细信息组件
final class CourseInfoView: UIView {
    // ...
}

```

通过上面的两个例子，我们就能够清楚的知道当我们以 Cell 的方式来实现组件的时候，一旦列表的实现方式在 UITableView 和 UICollectionView 之前切换的话，我们的 Cell 组件就需要做修改了，这就不够灵活了。甚至于当这个 Cell 组件不仅仅用在 UITableView 或 UICollectionView 的时候，我们还需要把这个组件使用 UIView 的方式重新实现一遍。

## View 组件

我们现在以 View 的方式来实现组件，参考下面的代码：

```Swift
// MARK: - Cell 组件
// 定义课程的 UICollectionViewCell 组件
final class CourseCollectionCell: UICollectionViewCell {
    // ...

    let courseView = CourseView()
}

// 定义课程的 UITableViewCell 组件
final class CourseTableCell: UITableViewCell {
    // ...

    let courseView = CourseView()
}

// MARK: -  View 组件
// 定义课程组件
final class CourseView: UIView {
    // ...

    let coverView = CourseCoverView()
    let infoView = CourseInfoView()
}

// 定义课程的封面组件
final class CourseCoverView: UIView {
    // ...
}

// 定义课程的详细信息组件
final class CourseInfoView: UIView {
    // ...
}

```

这种以 View 的方式实现的组件就要比以 Cell 实现的组件要灵活了，因为在这里 Cell 的作用其实就只是一个容器而已，只是把 View 放到 Cell 上去，并且负责做一些注册和重用的机制。

但是这种方式也有一个问题，就是只要这个 View 被用在 UITableView 或 UICollectionView 的时候，我们就要去写对应的 Cell 容器代码，而且这部分代码的工作基本上都是一样的（添加 View，添加约束）。

## [CheapCell](https://github.com/HParis/CheapCell)

CheapCell 是我们之前的项目重构成 Swift 之后实现的一套能够让你不用写 Cell 容器的库，这个库当时帮助我们减少了上千行的 Cell 容器代码。它的核心代码很简单，最重要的是利用了 Swift 的面相协议编程的方式来实现的。具体的实现代码可以看我的 Github 仓库 https://github.com/HParis/CheapCell。

下面是集成了 CheapCell 之后的使用方式：

第一步：让我们的 View 组件遵循我们定义的 CheapCell 协议

```Swift
// MARK: - CheapCell 协议
extension CourseView: CheapCell {}
extension CourseCoverView: CheapCell {}
```

第二步：在 UICollectionView 中的使用方式（UITableView 的使用方式也是类似的）

```Swift
// Register cheap cell
collectionView.registerCheapCell(CourseView.self)
collectionView.registerCheapCell(CourseCoverView.self)

// Reuse cheep cell
let cell = collectionView.dequeueReusableCheapCell(for: indexPath) as CollectionCell<CourseView>
let cell = collectionView.dequeueReusableCheapCell(for: indexPath) as CollectionCell<CourseCoverView>
```

### 配合第三方库使用

如果你使用了一些第三方的 Cell 的话，比如 [SwipeCellKit](https://github.com/SwipeCellKit/SwipeCellKit)，那么你可以按照下面的方式来实现你的 CheapCell：

```Swift
import SwipeCellKit

public extension UICollectionView {
    func registerCheapSwipeCell<T: CheapCell>(_ : T.Type) {
        register(CollectionSwipeCell<T>.self, forCellWithReuseIdentifier: CollectionSwipeCell<T>.identifier)
    }

    func dequeueReusableCheapSwaipeCell<T: CheapCell>(for indexPath: IndexPath) -> CollectionSwipeCell<T> {
        dequeueReusableCell(withReuseIdentifier: CollectionSwipeCell<T>.identifier, for: indexPath) as! CollectionSwipeCell<T>
    }
}


public final class CollectionSwipeCell<T: CheapCell>: SwipeCollectionViewCell {
    public static var identifier: String {
        return T.identifier
    }

    public let itemView = T()
    override init(frame: CGRect) {
        super.init(frame: frame)
        setupView()
    }

    required init?(coder aDecoder: NSCoder) {
        super.init(coder: aDecoder)
        setupView()
    }

    private func setupView() {
        contentView.addSubview(itemView)

        itemView.translatesAutoresizingMaskIntoConstraints = false
        itemView.leadingAnchor.constraint(equalTo: contentView.leadingAnchor).isActive = true
        itemView.trailingAnchor.constraint(equalTo: contentView.trailingAnchor).isActive = true
        itemView.topAnchor.constraint(equalTo: contentView.topAnchor).isActive = true
        itemView.bottomAnchor.constraint(equalTo: contentView.bottomAnchor).isActive = true
    }
}
```

现在你就可以直接通过 CheapCell 的方式来使用了：

```Swift
// Register cheap swipe cell
collectionView.registerCheapSwipeCell(CourseView.self)

// Reuse cheep swipe cell
let cell = collectionView.dequeueReusableCheapCell(for: indexPath) as CollectionSwipeCell<CourseView>
cell.delegate = self
```
