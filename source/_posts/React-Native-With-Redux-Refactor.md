title: 再谈React-Native With Redux
date: 2016-09-07 18:55:27
categories: hybrid
tags: react-native
---

> 有一段时间没写react-native redux相关文章了，使用中多多少少有些感悟，总结了一些，这次把它写出来。

本文目录
>* redux三大原则
>* 重新认识redux三要素
>* react-redux的最新用法
>* redux性能优化
>* redux周边设施
>* 总结 & 其他

撰写本文时笔者的相关环境如下：
>* 操作系统：OS X 10.11.6
>* react-native版本：0.32.0
>* npm版本：3.9.5
>* node版本：4.4.5

# redux三大原则
首先，我们明确一点：redux的核心思想就是FLux架构，单向数据流。

## 单一数据源
在redux中，所有的state最后都会被combine起来，变为一个大的state树，这个state树只存在于唯一的一个store中。这么做的好处是开发和调试都变得很方便，也很好理解。

## state只读
改变state唯一的办法就是通过触发action，别无他法。这么做非常纯粹，确保没其他副作用，state不会被任意修改。

## 纯函数修改state
想要修改state，在action被触发之后，需要编写reducer来进行修改，reducer只是一些纯函数，它会接受到action和state，返回**全新的state**。

# 重新认识redux三要素

## action
执行动作，把数据传递到store，触发它是用来改变state的，它是store的唯一数据来源。

## reducer
纯函数，用来修改/更新state，action只是说明有动作发生了，但是具体产生什么样的作用（state如何变更），还是reducer说了算。在修改完state之后，通过Object.assign，或者...state或者immutable返回**全新的state**

## store
整个应用唯一维护state树的地方，**是单一的**。可以获取、更新state，还可以注册监听。

这三个元素像齿轮一样运转起来，形成了FLux架构的单向数据流。

# react-redux最新用法
react-redux这个库，在redux和react结合使用的时候几乎是必用的。在经历过某次升级后，它的Provider组件用法发生了一点变化，最新的用法如下：

```JavaScript
import { Provider } from 'react-redux';

const Root = () => (
  <Provider store={store}>
    <App />
  </Provider>
);
```

# redux性能优化
性能优化第一步，就是搞清楚action和reducer的关系。如果搞不清关系，就会像我之前一样，大量页面被过度渲染。它俩是什么关系呢？实际上它俩是多对多的关系。也就是说，一个action操作可能会造成state树中的一个或者某几个部分发生改变，就对应了一个或多个reducer，多个action操作也同理，这是我们必须明确的。

如何避免引入redux之后造成页面过度渲染？

首先我们要搞清楚，为什么会过度渲染。在一个引用了redux的rn app中，state树是唯一的。但是在某个页面中，控制它的状态只是整棵状态树的子集，也就是state树中的一部分。如果没搞清action和reducer的关系随便乱用，很容易造成一个reducer控制着多个页面的情况（也就是说页面组件和reducer的状态子集形成了多对多的关系），那就妥妥的过度渲染了。举个简单的例子：reducer A控制着页面C和D的状态，当发起了一个action之后，A改变了，那么C和D就会调用render方法进行无用渲染。但是这时候很可能C或D并不是正在展示的页面，这里就出现了过度渲染，如果页面多，就会造成卡顿情况。

如何避免过度渲染？action是用来发起修改state操作的，具体怎么改是由reducer决定的，但是最终所有的reducer都会被combine成一个，那么reducer的拆分就是我们需要考虑的点。每一个reducer都控制了状态树的一个子集的变化，我们通过react-redux库的`connect`函数向组件注入状态，那么我们就需要把reducer拆分到较小的单元，使得一个页面的状态集合是由一个或多个reducer来控制，这里是一对多的关系，能做到这一点，就能避免过度渲染的问题。还是用刚才的例子，拆分后的reducer A只控制页面C，页面D由另一个reducer控制着，那么当A改变了，只会引起C的渲染，就能保证只渲染正在向用户展示的那个页面了。

# redux周边设施

## react-redux
这个应该是首当其冲的了，在redux官方文档中都有提及，只要用react/react-native，引入redux，就会引入这个库打配合，简单好用。

## redux-logger
这个库的用法很简单，它的作用是一次配置好，以后所有action触发了state的变化，都会用比较鲜艳的log日志打印出来，用chrome调试就可以很明显的看到，这样就能一清二楚每一个action最终都导致了state前后发生了什么样的变化。

## redux-thunk
这个库的作用主要是异步action。在我们引入redux之后，肯定会有很多异步操作，我们想把这些异步操作和组件剥离开，那么最直观的就是放入action中进行处理，通过thunk实现异步action。使用简单，理解起来也方便，很多刚入坑的同学处理异步action一般都会选用这个库。

## redux-saga
这个库本质上也是处理异步action的，但是相对thunk来说，它就没那么直观没那么好理解了。这个库主要思想就是通过es6的generator，把异步promise操作变成了类似async和await的同步写法，但是saga更强大。saga主要运用了协程的思想，在异步操作变为同步写法的时候，提供了非常便利的可测试性，而且简化了action。在引入saga之后，action里的代码就变得非常纯粹了，什么都不干，只触发action，具体的处理，不管是同步还是异步统统挪到saga中，写法也变得非常简洁。相对测试来说，写UT也变得清晰了很多，因为saga天生就是支持步进的，那么action对应的测试就写在saga包中就可以，因为真正处理action逻辑的是saga。相比之下，thunk就不好写测试了，一般用thunk的，测试都是根据reducer来写的，thunk中的操作就变得不可控了。

## immutable.js
这个库是Facebook的，主要是做deep clone，比较高效，和redux结合的地方就是reducer返回全新state的地方。

其实还有很多其他周边设施，我这里列举的是目前比较火或者比较主流的一些，当然了我也不一定全部都清楚，如果大家有更好的欢迎留言交流。

# 总结 & 其他
对于我这枚菜鸡来说，还有很长的路要走，文中提到的点大多都在我自己的开源项目[reading](https://github.com/attentiveness/reading)中有体现，也非常欢迎大家关注。最近收到了几个高质量的PR非常开心，在学习的同时把reading做的更好也是我的愿望。只有实战、写代码、踩坑，才能总结、重构、进阶，这条路没有捷径。只要方向没错，那就努力！