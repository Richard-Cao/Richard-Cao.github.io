title: React-Native With Redux
date: 2016-01-12 00:26:21
categories: hybrid
tags: react-native
---
转载请注明出处http://richard-cao.github.io/
> 经过上次的react-native小模块完成之后，发现不少缺点，而且基本没什么扩展性。这次正好又增加一个react-native模块————我的等级特权，于是动手重构了项目里整个react-native的部分，随着今晚项目发布上线，动手记录下来这次重构的经验。

本文目录
>* 为什么要做这次重构
>* Flux模式与Redux
>* React-Native With Redux
>* 代码规范和语法糖
>* 重构过程中遇到的坑
>* 总结

撰写本文时笔者的相关环境如下
>* 操作系统：OS X 10.11.2
>* npm中react-native版本：0.17.0
>* Android studio中react-native版本：0.17.1

## 为什么要做这次重构
之前的初次踩坑文章是在做第一个react-native需求——通知中心的时候写的，当时为的是功能没问题然后上线，并没有考虑扩展、封装、数据流等问题。当又要添加其他react-native模块的时候，就必须要解决这样的问题了，于是这次重构应运而生。
## Flux模式与Redux
### Flux模式
首先，我们知道，react-native根据什么render UI呢？答案就是state和props。那么可以预料到，当模块增多、代码量增加的话，如果没有一套数据流规范，那么就会遇到state或props不统一导致刷新错乱等问题。react是遵循Flux架构的，那么什么是Flux呢？这里我们看一张图：
![Flux](http://cdn4.infoqstatic.com/statics_s1_20160105-0313u5/resource/news/2014/05/facebook-mvc-flux/zh/resources/0519001.png)
Store包含了应用的所有数据，Dispatcher替换了原来的Controller，当Action触发时，决定了Store如何更新。当Store变化后，View同时被更新，还可以生成一个由Dispatcher处理的Action。这确保了数据在系统组件间单向流动。当系统有多个Store和View时，仍可视为只有一个Store和一个View，因为数据只朝一个方向流动，并且不同的Store和View之间不会直接影响彼此。(这段话引用自[
Facebook：MVC不适合大规模应用，改用Flux](http://www.infoq.com/cn/news/2014/05/facebook-mvc-flux))
### Redux
那么Redux是什么呢？Redux是javascript状态容器，提供可预测化的状态管理，可以构建一致化的应用，除了和React一起用外，还支持其他界面库，体积小（只有2kb）而且没有任何依赖。
Redux由[Flux](http://facebook.github.io/flux/)演变而来，但是避开了Flux的复杂性，上手快，使用简单，而且社区活跃，是目前主流的Flux数据流框架。
关于Redux文档可以看[英文原版](http://rackt.org/redux/)和[中文翻译版](http://camsong.github.io/redux-in-chinese/)。

---
> 从这里开始，默认读者已经阅读过Redux文档，有Redux基础。

## React-Native With Redux
我的`package.json`中引用的模块有：
```
"dependencies": {
     "immutable": "^3.7.5",
     "react": "^0.14.3",
     "react-native": "^0.17.0",
     "react-redux": "^3.1.0",
     "redux": "^3.0.5",
     "redux-thunk": "^1.0.2"
   }
```
redux目前的最新版本3.0.5是基于react 0.14的，所以同时加入`react`和`redux`，`react-redux`是Redux的react绑定库，`redux-thunk`是为了实现异步Action Creator引入的。
下面我以`请求用户等级特权数据并刷新UI`为例梳理一遍整个数据流，包含`Action`，`Store`，`Reducer`三个重要概念。
首先，定义请求用户等级特权数据的ActionType：
`react-native/constants/ActionTypes.js`
```javascript
export const FETCH_RANK_LIST = 'FETCH_RANK_LIST';
```
那么`FETCH_RANK_LIST`就代表了要执行请求等级特权数据的动作类型。然后开始定义Action：
`react-native/actions/rank.js`
```javascript
'use strict';

import * as types from '../constants/ActionTypes';
import {LEVEL_PRIVILEGES} from '../constants/Urls';
import {request} from '../utils/RequestUtils';
import {ToastShort} from '../utils/ToastUtils';

export function fetchLevelPrivileges() {
	return dispatch => {
		dispatch(fetchRankList());
		request(LEVEL_PRIVILEGES, 'get')
			.then((rankList) => {
				dispatch(receiveRankList(rankList));
			})
			.catch((error) => {
				dispatch(receiveRankList([]));
				if (error != null) {
					ToastShort(error.message)
				}
			})
	}
}

function fetchRankList() {
	return {
		type: types.FETCH_RANK_LIST,
	}
}

function receiveRankList(rankList) {
	return {
		type: types.RECEIVE_RANK_LIST,
		rankList: rankList
	}
}
```
这里的Action是异步的，因为请求是异步的。其实意思很简单，通过`fetchLevelPrivileges`请求了后端数据，异步获取了数据之后进行数据的接收，触发了接收数据的Action：`RECEIVE_RANK_LIST`，请求和接收其实是一个连续的动作。
那么定义完Action之后，就需要定义`Reducer`了：
`react-native/reducers/rank.js`
```javascript
'use strict';

import * as types from '../constants/ActionTypes';

const initialState = {
	loading: false,
	rankList: []
}

export default function rank(state = initialState, action) {
	switch (action.type) {
		case types.FETCH_RANK_LIST:
			return Object.assign({}, state, {
				loading: true
			});
		case types.RECEIVE_RANK_LIST:
			return Object.assign({}, state, {
				loading: false,
				rankList: action.rankList
			})
		default:
			return state;
	}
}
```
可以看到`initialState`是初始的状态，然后通过不同的type来更新state。这里state是全新的state，并不是在已有state的引用上改变数据，关于这点Redux的文档中有详细的解释，这里不再赘述。简单的reducer定义好之后，我们要开始定义`Store`了：
`react-native/store/configure-store.js`
```javascript
'use strict';

import {createStore, applyMiddleware} from 'redux';
import thunkMiddleware from 'redux-thunk';
import rootReducer from '../reducers/index';

const createStoreWithMiddleware = applyMiddleware(thunkMiddleware)(createStore);

export default function configureStore(initialState) {
	const store = createStoreWithMiddleware(rootReducer, initialState);

	return store;
}
```
这里使用了`redux-thunk`来支持异步Action，`Middleware`提供的是位于action发起之后，到达reducer之前的扩展点，这是一个比较重要的概念，具体请看redux文档理解。`rootReducer`是最终合并后的reducer：
`react-native/reducers/index.js`
```javascript
'use strict';

import {combineReducers} from 'redux';
import notice from './notice';
import rank from './rank';

const rootReducer = combineReducers({
	notice,
	rank
})

export default rootReducer;
```
这里用到了redux的`combineReducers`函数，将多个模块的reducer合并成一个。
最后我们需要串通整套数据流，我们需要做的是：
`react-native/root.js`
```javascript
import React from 'react-native'
import {Provider} from 'react-redux/native'
import configureStore from './store/configure-store'

import App from './containers/app'

const store = configureStore();

class Root extends React.Component {
	render() {
		return (
			<Provider store={store}>
        {() => <App />}
      </Provider>
		)
	}
}

export default Root;
```
这非常关键，root.js是index.android.js注册的唯一入口，通过`Provider`组件讲store注入进整个app，至此，整套数据流就串通起来了。
那么串通起来是怎么样的呢？我来描述一下：用户点击进入等级特权页面，通过action中`fetchLevelPrivileges`做了请求数据的动作，然后dispatch了`FETCH_RANK_LIST`这个动作，触发了reducer更改state，刷新UI（此时应该是loading界面）；然后当数据请求完成之后dispatch了`RECEIVE_RANK_LIST`这个动作，接收到请求获取的数据，触发了reducer更改state，再刷新UI（此时应该展示完整页面）。这样数据流就非常清晰了：`Action => Dispatcher => Store => View`。当用户进行其他操作时，由View发起Action，继续这个单向的数据流，这样就完成将Flux单向数据流的思想通过Redux融入React-Native项目当中了。
> 将**Flux的思想**应用于项目之中，确实感觉思路清晰，写起来心里踏实。

## 代码规范和语法糖
由于我是菜鸟，所以我在写的时候严格遵循了[Airbnb React/JSX Style Guide](https://github.com/airbnb/javascript/tree/master/react)，相信大厂应该没错的。
语法糖我全部使用了ES6，因为react-native已经使用了Babel完全支持了[ES6语法糖](http://es6.ruanyifeng.com/)，可以使用ES6的新特性，而且我感觉ES6对于我来说更容易理解，因为我是个写Java的Android Developer……
## 重构过程中遇到的坑
这里我要说明一点：**[使用Chrome调试react-native](https://facebook.github.io/react-native/docs/debugging.html#content)**非常重要！在重构的过程中，我都是通过debug来观察数据流，看哪一环出现了问题再去解决。
还有一个**大坑！**
当手机开启`手势触摸`选项之后，在react-native页面，同时用三个或三个以上手指触摸上去你就会发现……crash了。iOS我没测试过，这个是我在Android机器上发现的问题，然而**官方并没有解决办法**，我安装了react-native官方的showcase案例的一个app，发现该问题同样存在。。。所以我只好[提了issue](https://github.com/facebook/react-native/issues/5246)。
还好这种情况很少，目前没有接到线下类似这样的crash反馈，估计是很多手机是不带手势触摸的，而且估计很多用户不会开启手势触摸，其实我用Android手机的时候一直没开过……在我写这篇文章的时候[react-native 0.18.0-rc](https://github.com/facebook/react-native/releases/tag/v0.18.0-rc)已经发布了，但是并没有看到修复类似的bug，不过0.18应该是一个相对较大的更新，`react-native-cli`也更新到了0.1.10，Android里react-native的依赖库也更新到了0.18.0版本，我打算等react-native发布0.18.0 release版本之后进行一次整体更新，继续踩坑……
## 总结
对于我这个菜鸟来说，这次重构+新功能开发确实是有惊无险。总结一下：
所有action的定义都放在action包中，reducer放在reducers包中，store放在store包中，入口依然是index.android.js，只不过注册的时候直接指向root.js，通过root将store注入到app当中，所有的模块都包一层containers，在这里进行connect:
`react-native/containers/RankContainer.js`
```javascript
'use strict';

import React from 'react-native';

import {connect} from 'react-redux/native';

import Rank from '../components/Rank';

class RankContainer extends React.Component{
	render() {
		return (
			<Rank {...this.props} />
		)
	}
}

function mapStateToProps(state){
	const {rank} = state;
	return {
		rank
	}
}

export default connect(mapStateToProps)(RankContainer);
```
这里可以看到，所有的页面都是`组件`，这里的`<Rank />`就是等级特权页面组件，包括自定义控件等组件全部放入components包中，于是整个工程被组件化了，更容易与iOS进行融合。然后在utils包中定义utils，constants包中写了ActionTypes和Urls。在新增模块的时候，思路已经非常清晰了，其实就是做**填空题**：在ActionTypes中添加动作定义，在actions中定义Action，在reducers中定义reducer，然后在containers中写好容器外壳，最后在components中写组件，个人感觉是**可扩展的弹性小架构**，思路、封装、数据流、组件等等都比较清晰，目前这就是我重构之后的样子了。因为这些都是我一个人摸索的，等与公司的web工程师们交流时他们或许会给出更好的建议，期待ing~~

最后附上我的工程目录(IDE: Sublime Text 3)：
![react-native](http://7xr0xq.com1.z0.glb.clouddn.com/react-native.jpg)

至此，本文结束。
欢迎大家互相交流讨论。我只是菜鸡，抛砖引玉~