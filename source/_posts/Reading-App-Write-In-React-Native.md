title: Reading App Write In React-Native
date: 2016-02-06 14:56:57
categories: reading
tags: react-native
---
转载请注明出处：http://richardcao.me/

**reading项目开源地址：https://github.com/attentiveness/reading**
## Reading App Write In React-Native
> No Profit, No Advertisement, Only Feelings

这是一个产品级的开源项目（请允许我这么说）。
## Init
打开我的git log看了看，创建reading项目的时间是**2015年12月11日上午1:00**，我依然记得那时候我的初衷是想做一个react-native为主导开发的APP。从刚开始学js到做公司android项目中react-native模块的开发还不到一个月，我想，能否用react-native为主导开发一个自己的APP来慢慢学习呢？于是我就init了reading项目放到了我的github上，还专门建了一个organization叫[attentiveness](https://github.com/attentiveness)，愉快的装了一逼。但是到**2015年12月13日下午3:15**的时候，我更新了一发commit就暂停了。因为当时我还不知道应该怎么去写代码，我都不知道js的代码规范是怎么样的，我也不知道做reading到底需要做什么需求，在迷茫之中，我停止了reading项目的开发。
## Restart
当我重启reading项目的时候，记录显示时间是**2016年1月25日上午1:24**。为什么重启reading项目？其实很简单，之前搁置的原因就是我不知道到底要做什么需求，到底应该如何去做，也不知道代码怎样去写。截止16年1月25日的时候，正好是我做公司的react-native项目刚满一个月的日子，我已经知道了如何去写代码，如何去做一个项目，所以我重启了reading项目。需求哪里来？我的一个小伙伴想做产品经理，但是他以前根本没有经验，那就一起学。需求就这么来了，于是我们就说干就干了。
## 0.1.2
**0.1.2是Reading app发布市场的第一个版本**。为什么是0.1.2？因为0.1.1是reading的内测版本，内测平台我选择了[蒲公英](http://www.pgyer.com/)。说实话，这是我第一次打react-native开发的app的release包，第一次把一个自己认为称得上是`项目`的app发布到了应用市场。
>* 360市场下载地址：[Download Reading](http://zhushou.360.cn/detail/index/soft_id/3217938?recrefer=SE_D_Reading)
>* 豌豆荚下载地址：[Download Reading](http://www.wandoujia.com/apps/com.reading)

## React-Native Android类似弹窗效果的实现
在react-native ios中，可以用[Modal](https://facebook.github.io/react-native/docs/modal.html#content)组件实现这个效果，但是官方文档中明确注明了这个组件是
> This component is only available in iOS at this time.

OK，那android怎么办？从官方的[Known Issues](https://facebook.github.io/react-native/docs/known-issues.html#content)中可以看到，官方有让Modal组件支持android的计划，但是这个版本是没有的，现在最新版本[0.19.0的release note](https://github.com/facebook/react-native/releases)中也是没有的，但是reading app要上线第一个版本，就必须实现分享的弹窗。当时考虑有两种方案：`dialog`和`spinner`。官方组件是有dialog的，这个组件是[Alert](https://facebook.github.io/react-native/docs/alert.html#content)，但是这个组件并不支持自定义UI，只是固定的样式模板，那么无法实现我这个需求。`spinner`组件官方还没支持，并且明确了以后会支持，所以我想到了能不能有android可以使用的类似Modal的组件呢？于是我找来找去，还真有，官方文档中并未提及，这个组件的名字叫[Portal](https://github.com/facebook/react-native/blob/master/Libraries/Portal/Portal.js)。组件找到就好办了，于是我开始动手实现。

首先引入Portal：

```
import Portal from 'react-native/Libraries/Portal/Portal.js';
```

引入之后，我看了下Portal，使用起来也挺简单，虽然没有文档，但是可以看到，我能用到的有这么几个方法
>* allocateTag
>* showModal
>* closeModal
>* getOpenModals

那么就简单了，引入之后我需要

```
componentWillMount() {
    if (Platform.OS === 'android') {
      tag = Portal.allocateTag();
    }
  }
```
拿到`tag`之后

```
onActionSelected() {
    Portal.showModal(tag, this.renderSpinner());
  }
```
这里的`onActionSelected`是[ToolbarAndroid](https://facebook.github.io/react-native/docs/toolbarandroid.html#content)的一个属性，可以触发点击action的事件，我的分享按钮就在右上角，那么点击之后，分享的弹窗就出现了。

那么如何关闭？我这里做的比较简单，是这样的

```
goBack() {
    if (Portal.getOpenModals().length != 0) {
      Portal.closeModal(tag);
      return true;
    } else if (canGoBack) {
      this.refs.webview.goBack();
      return true;
    }
    return NaviGoBack(this.props.navigator);
  }
```
然后

```
componentDidMount() {
    BackAndroid.addEventListener('hardwareBackPress', this.goBack);
  }

  componentWillUnmount() {
    BackAndroid.removeEventListener('hardwareBackPress', this.goBack);
  }
```

这个UI怎么像是一个弹窗呢？其实技巧就在于背景颜色上面。我的分享View是居中的，背景颜色我用了rgba设置了透明度，就像是这样

```
backgroundColor: 'rgba(0, 0, 0, 0.65)'
```
那么就非常像一个弹窗啦~具体效果可以看[这里](https://github.com/attentiveness/reading/blob/master/screenshot/Reading_Share.jpg)
## Release Note与线上事故复盘
针对这部分，我也在项目中专门写了这两个文档，具体可以看
>* [Reading Release Note](https://github.com/attentiveness/reading/releases)
>* [Release线上事故复盘](https://github.com/attentiveness/reading/blob/master/Reading_OnLine_Accident.md)

文档会根据实际情况保持更新，这是reading自己的积累。

## 敏捷开发
git工作流我就不在这里赘述了，大家可以参照[A successful Git branching model](http://nvie.com/posts/a-successful-git-branching-model/)。

在0.1.3版本上线之后，我们发现我们需要开发流程，之前流程的不规范导致我们工作效率不高并且容易遗漏很多东西，所以我采用了最简单的敏捷开发流程，我们使用[Tower](https://tower.im/)进行协同，为什么选择Tower？
>* 与微信绑定，使用方便
>* 有敏捷开发板，可以构建简易的敏捷开发，防止需求遗漏，还可以进行问题讨论等，管理非常方便，也是现在reading正需要的
>* 没有多余的冗杂功能，reading现在还很小，这就足够了，简约而不简单

这是我们目前的Reading敏捷开发板展示，可以看到我们已经在计划做0.1.4版本的需求了

![Reading敏捷开发展示](http://7xr0xq.com1.z0.glb.clouddn.com/Reading%E6%95%8F%E6%8D%B7%E5%BC%80%E5%8F%91%E5%B1%95%E7%A4%BA.jpg)

## 最后再说几句
欢迎大家[star，fork，pr，issue](https://github.com/attentiveness/reading)，和reading、和我们一起成长~

目前只发布了360市场和豌豆荚，360市场审核快一些，apk版本更新之后我会及时发布市场提交审核的，当然reading中也是有自己的热更新哒，react-native你懂得。针对reading项目我采用的**热更新方案**是[Microsoft/react-native-code-push](https://github.com/Microsoft/react-native-code-push)，因为reading是没有后端的，所以code-push项目完美的帮我解决了热更新的jsbundle版本管理发布问题。

**目前Reading线上只有android版本，功能稳定之后我会做ios的兼容，并且有发布到AppStore的计划。**

**Reading会持续更新下去的，希望能得到大家的支持和鼓励。你们的鼓励是我们前进的动力！**

![Reading in Facebook react-native showcase](http://7xr0xq.com1.z0.glb.clouddn.com/reading_in_react-native_showcase.jpg)