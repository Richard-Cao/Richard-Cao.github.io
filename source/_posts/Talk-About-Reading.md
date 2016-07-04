title: 聊聊我的处女作：reading
date: 2016-07-05 00:36:54
categories: open source
tags: react-native
---
转载请注明出处：http://richardcao.me/

**[戳我关注reading](https://github.com/attentiveness/reading)**

[![GitHub stars](https://img.shields.io/github/stars/attentiveness/reading.svg)](https://github.com/attentiveness/reading/stargazers)

>* 不知不觉reading项目已经有500+star了，作为自己的处女作，感到非常开心。感谢关注reading的人，我会坚持继续维护下去。
>* No Profit, No Advertisement, Only Feelings

---

## reading初衷
其实做reading项目的初衷很简单，就是我想自己尝试着用react-native写一个app。真正开始写的时候，大概是五、六个月前的样子，想把自己在实战中的一些经验用代码的方式展现出来，抛砖引玉，和大家一起交流进步。那时候github上国内的react-native开源app还很少，而且大多都没继续维护了，于是自己下决心好好做一个，好好维护下去。不知不觉半年了，每次打开reading项目代码，还是感到一阵亲切。开源reading，可以让reading成长的更快更好，同时对我而言也是一样。对待自己的处女作开源项目，我格外认真，目标是把reading打造成一个高质量的开源项目。

## reading架构演进
再小的APP也有自己的架构，reading也不例外。在项目的readme中，我简单的提到了[reading的架构](https://github.com/attentiveness/reading#application-architecture)，这里我分享下reading从一开始到现在发生了哪些变化。

> reading是基于react-native开发的Android & iOS双平台的APP，数据流和状态通过redux进行管理，并且在摸索着进行UT。

reading现在的架构是这样的：

>* actions：redux中action部分都在这里
>* components：reading中用到的通用控件，全部抽离出来在这里，同时任何一个react-native app，都可以直接拿去用，支持Android & iOS双平台
>* constants：这里主要是一些常量，比如action的type，一些字符常量，url常量，还可以有颜色常量等各种
>* containers：容器层，需要注入state的地方，我会用容器包一层。如果有需要页面中用到多个组件拼成，也通过容器包一层，达到复用。app入口也在这里
>* img：这里主要放图片资源
>* pages：这里主要放页面，其实就是组件。目前来看，这里面的组件其实都是reading的单个页面。其实页面是可以组合的，在外面包一层容器的话，就可以进行组合。一些复杂的页面，完全可以通过这种方式去做，复用性更高，而且还能解耦
>* reducers：redux中reducer部分都在这里，reducer和action不一定是一一对应的，数量也不一定是相等的。state可以进行各种颗粒度的细分，最后通过`combine`函数合为一个。拆分state非常重要，它涉及到性能优化，不同的state通过不同的reducer进行处理，然后渲染不同的页面，甚至是渲染同一个页面中的不同组件，这样就避免了过渡渲染。如果不拆分，当一个大state改变的时候，所有的页面都会重新render，这是很浪费性能的
>* store：redux的store部分，通过reducer和各种各样的middleware创建store。例如redux-thunk，redux-logger等
>* utils：工具包
>* root.js：统一index.android.js和index.ios.js入口，通过react-redux中的`Provider`组件包裹整个APP

以上均为reading的js部分。目前reading中除了引用的第三方库含有原生代码，我几乎没写什么原生代码，我想尝试用js的方式去写。不排除如果后面功能复杂的话，会加一些原生（暂时并不会写OC……）。reading项目是我主打以rn为主原生为辅的开发方式构建跨平台的开源APP。

### 起初
一开始，reading是没有这些东西的，所有代码都是一坨……完全没有模块划分的东西。当时我还在饿了么尝试在蜂鸟众包app中加入react-native模块，也没有任何架构，完全不具备可扩展性。第一次重构发生在我想要加功能的时候，根本没法加，于是我本能的感觉代码是有问题的。当时react-native资料除了官方文档其他的少之又少，中文资料根本没有，于是我找到了一个老外写的app，感觉很6，clone了代码下来读。读完了之后，照猫画虎的重构了一下，有了简单的分层，这就是reading架构的雏形。在这个时候，我才明白redux是个什么东西，怎么用，分层是什么样的。

### 第一次里程碑
由于根本没法加功能，于是我进行了第一次重构。我的记忆中，经历过第一次重构，reading的代码从一坨变成了这样：

>* actions：redux中action部分都在这里
>* components：reading中用到的通用控件，并且在当时所有的页面也在这里，并不支持ios平台
>* constants：这里主要是一些常量，当时只有action的type
>* containers：容器层，app入口也在这里，当时我所有的页面都会有一个容器，存在资源浪费和过渡渲染的问题
>* img：这里主要放图片资源
>* reducers：redux中reducer部分都在这里，当时我误认为reducer和action必须一一对应，于是我按照一一对应去写的，这样的思路是有问题的
>* store：redux的store部分，通过reducer和各种各样的middleware创建store。当时还不太清楚异步action和同步action的问题，所以redux-thunk都没用上，更别提redux-logger了
>* utils：工具包
>* root.js：统一index.android.js和index.ios.js入口，通过react-redux中的`Provider`组件包裹整个APP

可以看出，这就是reading现有架构的雏形。它已经初步规定了数据流向，规定了系统分层与组件间的关系等。在这个架构的基础上，我添加了reading最初也是最基础的几个功能。

### 再一次改变
写着写着，也发了几个版本，感觉性能有点问题，也看了更多的一些代码，跟着官方更新的脚步思考，又看了一些文章，反反复复，我觉得我的代码存在一些问题，需要进一步修改。当时我意识到的问题有：

>* redux使用存在过度渲染问题，数据流管理有点乱，需要重点优化
>* 组件和页面的概念有点模糊，需要进一步界定
>* action和reducer的概念有点模糊，需要更清晰的理解
>* 组件化思想愈发明确，action中需要异步与同步共同处理

首先，关于redux的问题，其实是一连串的问题，也是我之前没有搞清楚的概念。我一直认为action和reducer是一一对应的，其实完全不是的，更形象的说，是多对多的。redux的state最后会合并成一个，所以为了避免每次改变state都刷新所有的页面，应该拆分state，局部刷新，这样可以避免过度渲染。只要我们根据实际情况，把reducer拆开，action可以根据一个页面定义一个，然后一个页面完全可以对应多个reducer去操作，这样不光解决了性能问题，数据流管理也更加清晰了。其实这里涉及到2个框架：`redux`和`react-redux`，拆分了之后需要用react-redux的`connect`函数按需注入`dispatch`和`state`，这时候拆分的reducer就起作用啦，模块中containers部分也得到了优化，从一个页面对应一个容器，优化到需要注入的页面才有容器，不需要的就不要容器。概念区分清楚了，定义就完全自由了。
其次，关于组件和页面的问题，我干脆加了一个pages包，专门放页面，相当于页面级的组件，和业务相关，components里只放通用模块组件，保证我拿到别的项目分分钟一样可以用起来，这样一拆分，瞬间清爽多了。
组件化思想驱使我在action中处理同步+异步的问题，之前的代码中，网络请求等明显的异步action是放在页面组件中的，现在全部抽走拿到action中，也就是说V这一层被解耦了，只和action、reducer相关，这个action我感觉有点像Android MVP架构中的P层。

### Android & iOS统一
架构暂时稳定了一段时间，我考虑兼容iOS。因为我起初是只做了android平台，所以在兼容ios的时候，主要还是在UI的部分，其实并不难，在做android的时候，我就考虑到了后续兼容ios的问题，组件方法函数等都考虑到了ios，这部分我做的最多的还是UI层的统一。在代码改动最少的情况下兼容ios，而且reading的业务逻辑并不复杂，于是很容易就做到啦。

reading发展到现在，大致上经历了这四个过程，后面还有很长的路要走，要学习的东西还有很多。

## reading未来发展
下一步，reading会基于Android & iOS双平台的基础上，继续做一些事情，在这里我简单列举一下：

>* redux还有很多更高级的玩法，还需要再琢磨琢磨[redux的官方文档和react-redux的API文档](http://redux.js.org/)，翻译的很棒的中文版[在这里](http://cn.redux.js.org/)
>* 逻辑+UI的测试框架我已经跑通，如何编写有价值的测试用例也是我需要学习的
>* 目前reading项目中Navigator机制用的是官方提供的[Navigator](http://facebook.github.io/react-native/docs/navigator.html)实现双平台的统一，但是关注react-native动态的小伙伴都知道，官方正在做一款新的Navigator机制：`NavigationExperimental`。最近的版本release note里关于这个的内容很多，暂时还不稳定，所以我还没去看，官方在issue里表示过，新的Navigator很快就会release，到时候我会在reading中进行切换
>* 我会时刻关注react-native动态，会在reading的代码中体现出来

## 最后说几句
reading还有很长的路要走，我的目标是把它打造成**国内优质的react-native APP开源项目**，让想学习rn快速搭建APP的人通过学习reading的代码，系统性的搭建起属于自己的APP，并且进行各种各样的架构优化与演进，用于开源项目或者商业项目，体现价值。欢迎大家与我交流，提issue，pr给我。

## 特别感谢
感谢[鬼道](https://github.com/luics)大哥star了reading项目，鼓励我让我继续加油。刚开始接触react-native的时候，什么都不会，很多问题都是请教了鬼道大哥才明白，经常交流，受益匪浅。同时，热烈庆祝[weex](https://github.com/alibaba/weex)开源，阿里巴巴对技术的执着追求让我敬仰，对开源社区的贡献让我钦佩。虽然我现在还非常弱小，但我也非常愿意在开源社区贡献自己的力量，伴随我的成长，相信这股力量会越来越强大。