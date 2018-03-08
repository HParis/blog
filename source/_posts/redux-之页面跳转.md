---
layout: post 
title: redux 之页面跳转
subtitle: 关于 React Native 的页面跳转逻辑的一点思考
author: 帕帕
date: 2018-02-26 17:48:56 +0800
categories: React
tag: React
---

最近正在用 React Native 重构整个项目，我们用了 **[react-native-navigation](https://github.com/krystofcelba/react-native-navigation#rn52)** 这个库来作为项目的导航控制器。
所以，我们平常会把页面跳转逻辑的时候放在 Screen 里面的，比如:

```Javascript
class FirstScreen extends React.Component {
    
    // 点击事件
    _someAction = () => {
        this.props.navigator.push({
          screen: 'example.SecondScreen',
        });
    }
    
    render = () => {
        ...
    }
}
```

一般情况下，上面的写法没有问题。但是直到我们碰到这样一个需求的时候就抓瞎了：点击一个 PDF 文件，如果 PDF 文件没有下载就先去下载，下载完成之后自动跳转到 PDF 阅读器。由于用了 redux 之后，我们就增加一个 finished 的 state 来判断是否已经下载完成。示例代码如下：

```Javascript
class ExampleScreen extends React.Component {

    componentWillReceiveProps = (nextProps) => {
        // 这里判断下载状态是否已完成，完成的话就去跳转
        if (nextProps.finished === true) {
            // 这里需要重置一下状态，不然其他 state 发生变化会多次触发页面的跳转
            this.props.dispatch(resetFinished());
            this.props.navigator.push({
              screen: 'example.PDFScreen',
            });
        }
    }
    
    // 点击事件
    _someAction = () => {
        // openPDF() 这个 action 会自动去下载 PDF 文件，然后修改 finished 的状态
        this.props.dispatch(openPDF());
    }
    
    render = () => {
        ...
    }
}

const mapStateToProps = state => {
  return {
    finished: state.finished
  }
};

export default connect(mapStateToProps)(ExampleScreen);
```

上面的做法是可以实现我们的需求，但是这种写法很蛋疼。因为当你在调用用 openPDF() 的时候，你以为后面的事不需要你操心，然后这个时候有人告诉你还需要在其他地方增加一个中间状态去补充 openPDF() 的后续逻辑处理。

经过讨论之后，我们决定改成用 callback 的方式来实现：

```Javascript
class ExampleScreen extends React.Component {

    // 点击事件
    _someAction = () => {
        // openPDF() 是一个异步 action
        this.props.dispatch(openPDF(callback: () => {
            this.props.navigator.push({
              screen: 'example.PDFScreen',
            });
        }));
    }
    
    render = () => {
        ...
    }
}
```

使用 callback 的好处就是去掉了一个烦人的中间状态，并且从阅读体验来说很容易让读者明白这个点击事件在干什么。但是在 redux 的 action 方法中增加一个 callback 的调用，看起来也有点不伦不类的。虽然我认为 callback 和其他参数具有相同的法律地位。

其实最好的实现是，这个点击事件应该连页面的跳转逻辑也不需要处理：

```Javascript
class ExampleScreen extends React.Component {

    // 点击事件，这个事件只做一件事就是去 dispatch 一个 openPDF() 的 action
    _someAction = () => {
        this.props.dispatch(openPDF());
    }
    
    render = () => {
        ...
    }
}
```

像上面这种实现，我们也就只能在 openPDF() 里动手脚了：

```Javascript
// action.js
export const openPDF = await () => {
    return dispatch => {
        // 异步下载 PDF
        async downloadPDF();
        // 完成之后通过 router 去实现页面跳转
        dispatch(openRouter('PDFScreen'));
    };
}
```

> 这里就不再详细说 router 的实现细节了，因为网上有很多现成的资料。（PS: 主要是我也还没看到这一块）

从页面（Screen）的角度来说，我认为这样的处理是最合适的。因为 Screen 只需要关注本页面的 state 和 action，至于跳转的逻辑交给后面的 action 来处理是最好的。




