title: React-Native Android 初次踩坑之旅
date: 2015-11-24 21:37:27
categories: hybrid
tags: react-native
---

> 本文背景：项目要上线一个app内部通知中心的功能模块，UI比较简单，ListView为主。之前关注react-native一段时间，所以打算使用react-native踩坑，激进的来一把。

本文目录
> * 如何把react-native集成到已经存在的Android studio工程中
> * 如何调试
> * 开发过程中踩过的那些坑
> * Android打包过程中踩过的那些坑

本文一切操作均在OS X系统上执行，调试手机为Android手机。

## 如何把react-native集成到已经存在的Android studio工程中

这部分主要参考官方文档[Intergrating with Existing Apps](https://facebook.github.io/react-native/docs/embedded-app-android.html#content)内容，这里简述一下：
### 导入react-native相关引用和权限
```java
compile 'com.facebook.react:react-native:0.15.1'
```
我开发的时候最新版本是0.15.1，如果想查看最新版本请戳[Maven Central](http://search.maven.org/#search%7Cgav%7C1%7Cg%3A%22com.facebook.react%22%20AND%20a%3A%22react-native%22)。然后
在studio工程中的`AndroidManifest.xml`中加入
```xml
<uses-permission android:name="android.permission.INTERNET" />
```
在Android中支持晃动手机或点击菜单键打开react-native的调试页面，需要在`AndroidManifest.xml`中加入
```xml
<activity android:name="com.facebook.react.devsupport.DevSettingsActivity" />
```
react-natice需要app的`build.gradle`文件中配置`compileSdkVersion`为23，`minSdkVersion`为16，但是我们项目的app`minSdkVersion`为15，所以为了支持15，要修改app的`build.gradle`文件添加如下内容
```
defaultConfig {
   ndk {
      abiFilters "armeabi-v7a"
   }
}
```
在`AndroidManifext.xml`中添加
```
<uses-sdk tools:overrideLibrary="com.facebook.react" />
```
这时候可能会报一个ndk的错误，只要在`gradle.properties`中添加
```
android.useDeprecatedNdk=true
```
即可。
### 写好基础的Android原生和js代码
这部分参照官方文档[Intergrating with Existing Apps](https://facebook.github.io/react-native/docs/embedded-app-android.html#content)中的Add native code和Add JS to your app部分内容。最后官方还提出了如果想要在多个activity或fragment中使用react，需要把`ReactInstanceManager`用单例实现，这里我的实现很简单：
```java
public class ReactInstanceManager {
    public final static String MODULE_NAME = "CrowdReactApp";

    private static class Holder {
        private static ReactInstanceManager sInstance = ReactInstanceManager.builder()
                    .setApplication((Application) ElemeApplicationContext.getContext())
                    .setBundleAssetName("index.android.bundle")
                    .setJSMainModuleName("react-native/index.android")
                    .addPackage(new MainReactPackage())
                    .addPackage(new CrowdReactPackage())
                    .setUseDeveloperSupport(BuildConfig.DEBUG)
                    .setInitialLifecycleState(LifecycleState.RESUMED)
                    .build();
    }

    private ReactInstanceManager() {

    }

    public static ReactInstanceManager getInstance() {
        return Holder.sInstance;
    }
}

```
这里的`index.android.bundle`就是react部分打包生成好的文件，Android打包之后react部分就是根据这个文件来生成代码，热更新也是更换这个文件。`react-native/index.android`就是react-native目录下的`index.android.js`文件。这里我在app工程中新建了react-native文件夹，react代码都放入该文件夹中。
### Android原生模块和js部分拆分开发
js部分我使用Sublime Text 3进行开发。这里简单讲讲我配置的简易插件：
- 首先毫无疑问的就是[Babel](https://babeljs.io/)，支持es6语法高亮，在Sublime Text 3中安装请看[babel-sublime](https://github.com/babel/babel-sublime)。
- jsx的语法检查插件，参考[esformatter-jsx](https://github.com/royriojas/esformatter-jsx)，具体安装配置文档已经说的很清楚了，这里不再赘述。若想了解jsx请查看[JSX in Depth](https://facebook.github.io/react/docs/jsx-in-depth.html)。

至此Android工程中已经集成好react-native模块了。
## 如何调试
在看这一部分之前，要保证上面一部分的内容已经非常仔细的执行完毕，否则一定会报错。尤其是对上面提到的官方文档部分的仔细研读。

关于react-native的调试我们知道，实际上就是在本地起一个node server，然后当js文件有改动或debug模式下手动选择reload js时候会自动更新bundle文件，达到改动js文件后即时显示的调试效果。

这里我没有用虚拟机调试，我直接使用真机调试。主要参考官方文档[Running On Device](https://facebook.github.io/react-native/docs/running-on-device-android.html#content)部分，对于Android 5.0以上的手机，在USB调试模式下连接电脑run`adb reverse tcp:8081 tcp:8081`命令即可，晃动手机弹出调试窗口，选择reload js就可以看到效果。如果是Android5.0以下的手机，在晃动手机或者点击menu键弹出的react-native调试菜单中选择`Dev Settings`然后配置`Debug server host & port for device`中配置你当前PC的ip地址加端口号即可，例如
```
127.0.0.1:8081
```
端口号必须是8081，ip地址根据当前pc的ip地址填写，此时手机和PC必须在同一wifi下，如果用这种方式调试react-native，可以不插USB。
在这个页面我们还可以看到一个选项叫`Auto reload on JS change`，如果我们选择它，就会在选项后面的小方框打勾，启动js改边自动更新的模式，这时候需要安装`watchman`才可以生效，如何安装
```shell
brew install watchman
```
这里我是安装过homebrew的，如果没有安装的可以看[Homebrew](http://brew.sh/)进行安装配置。

调试部分内容到此结束。
## 开发过程中踩的那些坑
在开发之前，请务必仔细研读一篇非常优秀的博文[React Native中组件的生命周期](http://www.race604.com/react-native-component-lifecycle/?hmsr=toutiao.io&utm_medium=toutiao.io&utm_source=toutiao.io)和官方文档中[Native Modules](https://facebook.github.io/react-native/docs/native-modules-android.html#content)，读懂之后再进行开发。

我们知道，react的核心就是虚拟DOM，关于这部分网上介绍的文章太多了，在此不再赘述。这里主要以我用到的[ListView](https://facebook.github.io/react-native/docs/listview.html#content)控件和[Image](https://facebook.github.io/react-native/docs/image.html#content)控件两大坑为主进行介绍，满满都是泪。

首先是ListView的常规写法：
```JavaScript
getInitialState: function() {
        return {
            dataSource: new ListView.DataSource({
                rowHasChanged: (row1, row2) => row1 !== row2,
            }),
            loaded: false,
        };
    },
```
我们需要了解的是，`rowHasChanged`中代码的意思是，当row1和row2不同时刷新Listview。这里的不同，是指**引用不同**，只是数据不同是不满足这个条件的。这是ListView的坑。说到引用不同，自然而然想到deepclone，Facebook自家提供了一个deepclone的高效解决方案[Immutable](https://facebook.github.io/immutable-js/)。我项目中就用到了它，后面我会讲我是怎么用的。写代码的时候要注意一下，**render中代码如果注释掉会报错的，其他部分的代码可以注释，但是render中的UI部分代码不能注释，要么就删掉，要么就保留**。

我们的逻辑应当是由数据来控制UI的显示，数据通过HTTP请求得到，那么免不了需要封装网络请求模块，这里我分享一下我是如何封装的
```JavaScript
'use strict';

var NativeManagerAndroid = require('./NativeManagerAndroid');

function RequestService() { // Singleton pattern
    if (typeof RequestService.instance === 'object') {
        return RequestService.instance;
    }
    RequestService.instance = this;
}

RequestService.prototype._httpHeader = function() {
return new Promise((resolve, reject) => {
    NativeManagerAndroid.header((header) => {
        resolve(header);
    });
})
}

RequestService.prototype.request = function(url, method, body) {
var isOk;
return new Promise((resolve, reject) => {
    this._httpHeader().then((header) => {
        fetch(header.Host + url, {
            method: method,
            headers: {
                'Content-Type': header.Content_Type,
                'User-Agent': header.User_Agent,
                'X-VERSION': header.X_VERSION,
                'X-DEVICE': 1, //表示Android设备，iOS为2
                'API-TIME': header.API_TIME,
                'API-DEBUG': header.API_DEBUG,
                'X-TOKEN': header.X_TOKEN,
                'X-ID': header.X_ID,
            },
            body: body,
        })
            .then((response) => {
                if (response.ok) {
                    isOk = true;
                } else {
                    isOk = false;
                }
                return response.json();
            })
            .then((responseData) => {
                if (isOk) {
                    resolve(responseData);
                } else {
                    reject(responseData.message);
                }
            })
            .catch((error) => {
                reject(error);
            });
    });
})
};

module.exports = RequestService;
```
简单解释一下，这里的`NativeManagerAndroid`是我封装的Android原生模块的js部分，`NativeManagerAndroid.header`方法执行的是Android原生事先写好的header方法，关于这部分请再次仔细研读[Native Modules](https://facebook.github.io/react-native/docs/native-modules-ios.html#content)，我这个方法主要是从原生代码中获取http的header，然后使用react-native自带的`fetch`进行http网络请求的封装。`fetch`返回的是`promise`，所以我利用了promise的特性进行封装，关于`fetch and promise`请仔细研读官方文档[Network](https://facebook.github.io/react-native/docs/network.html#content)部分。注意：`response.ok`是HTTP CODE在200到300之间。`module.exprots = RequestService`是指把当前RequestService.js作为一个module，可以在其他js文件中引用。

注意一下，如果用fetch执行put请求，必须有body，否则会crash，如果没有的话也要传一个空的body，像这样
```JavaScript
var body = JSON.stringify({
});
```

关于react-native的StyleSheet部分，采用的是flexbox的样式，在开发之前请先阅读[A Complete Guide to Flexbox](https://css-tricks.com/snippets/css/a-guide-to-flexbox/)。关于点击反馈，请查看官方文档`COMPONENTS`的TouchableHighlight，TouchableNativeFeedback，TouchableOpacity，TouchableWithoutFeedback。

关于Immutable.js的使用，我代码中是这么用的
```JavaScript
var newDs = [];
newDs = this.state.ds.slice();
notice.status = 1;
var changeNotice = Immutable.Map(notice);
newDs[rowID] = changeNotice.toObject();
this.setState({
    dataSource: this.state.dataSource.cloneWithRows(newDs),
});
```
这段代码已经很好的解释了deepclone在这个模块里的用法，这时候`dataSource`的引用已经改变了。Immutable.js的运用远不止这么一点，具体还是要在项目中慢慢体会，仔细研读Immutable的官方文档。注意：这里的`this.state.ds`是每次从网上拉取新数据是本地的备份，参见
```JavaScript
Request.request(NOTICE_LIST_URL, 'get')
    .then((noticeList) => {
        this.setState({
            dataSource: this.state.dataSource.cloneWithRows(noticeList.notice_list),
            loaded: true,
            ds: noticeList.notice_list,
        });
    })
```
`load`状态我是用来判断当前view是否展示loading界面还是listview界面。react的render机制，要么是state改变，要么是props改变，就会执行render进行UI刷新，我主要用到的是通过state来控制是否刷新UI。这里有个优化小技巧，我添加了这么一行代码
```JavaScript
mixins: [React.addons.PureRenderMixin],
```
这样可以减少不必要的render次数，具体参见React的官方文档[PureRenderMixin](https://facebook.github.io/react/docs/pure-render-mixin.html)。

下面讲讲Image的坑。

根据react-native的官方文档Image部分，如果需要加载本地静态资源（与Android原生共用图片资源），需要
```JavaScript
<Image
  source={require('image!myIcon')}
/>
```
但是，当工程跑起来之后，你会发现报了错：提示image!myIcon找不到。这里我反复google才发现2种解决方案：

1.加载静态图片资源的方法与官网写法一致，启动node server时执行如下命令
```shell
react-native start --assetRoots ./android/app/src/main/res/
```
在debug模式下就可以正常加载本地静态资源并进行调试了，但是在release打包后会crash，原因就是静态资源找不到，暂时没找到解决方案。
2.加载静态图片资源的写法改为
```JavaScript
<Image
  source={ { uri: "myIcon", isStatic: true} }
/>
```
启动node server时只要执行
```shell
react-native start
```
即可。这种解决方案在**Android release打包时同样有效**，我目前就采用的这个解决方案。

这是个大坑，花了好长时间才解决。
##  在Android打包过程中踩过的那些坑
首当其冲的是混淆，原生代码中所有自定义的`ReactPackage`和`ReactContextBaseJavaModule`等和reactjs部分配合使用的原生模块都必须keep掉，否则会crash，找不到原生的方法或类。这点我没在官方文档上找到说明，估计是Facebook觉得常识就应该这么做吧……

Image静态资源问题上面已经讲到了，在打包过程中最后选取的解决方案，这是个坑。

最后推荐开发过程中可以参考的一个项目[ZhiHuDaily-React-Native](https://github.com/race604/ZhiHuDaily-React-Native)，这个项目的作者也写了相关博客，很有用。

踩坑还要继续，毕竟以前几乎没有写过js，也是小白，慢慢学习中，比较喜欢React，欢迎与我交流。
