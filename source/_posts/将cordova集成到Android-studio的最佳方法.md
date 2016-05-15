title: 将cordova集成到Android studio的最佳方法
date: 2015-11-22 00:21:52
categories: hybrid
tags: cordova
---
转载请注明出处：http://richardcao.me/

>网上有很多集成cordova到Android studio中进行Android开发的方法，这里我给大家介绍一种比较简单的方法，亲测有效。

### 目的

首先我们明确目的，我们希望把cordova快速集成到Android studio中，为了以后的复用，我们希望做成jar或者aar，可以用于以后的项目，目的明确了，后面就一步一步来吧。

### cordova相关代码

代码哪里来？大家不需要找，请看[这里](https://github.com/apache/cordova-android)，将代码clone下来后发现有这么几个我们要用的文件夹：

- **cordova-js-src**：这部分是cordova-android对应的js代码，混合开发的时候H5需要将这个文件夹导入web工程，并做相应的引用和配置
- **framework**：这部分是cordova-android部分代码，需要全部拿过来
- **test**：这部分是测试工程，我们在集成好cordova后，还要根据测试工程实例进行相关配置，才能完全集成cordova到Android studio中进行hybrid开发。

### 开始集成

1. 直接把framework模块打一个aar或jar包（我打的是aar）；
2. 把cordova-js-src复制到web工程中；
3. 结束。
卧槽？这么简单？别激动，还没完，继续往下看。

### 融合cordova到自己的工程中

首先，我们建一个自己的Android工程，然后我们复制test工程中的/res/xml/config.xml文件到我们自己工程的/res/xml/config.xml中，不要改路径，然后修改config.xml文件：

``` xml
<?xml version='1.0' encoding='utf-8'?>
<widget
    id="your package name"    //包名
    version="0.0.1">          //版本号
    <content src="index.html" />
    <feature name="xxxxx">    //插件名
        <param
            name="android-package"
            value="your package name.xxxxx" />  //插件路径
        <param
            name="onload"
            value="true" />
    </feature>
    <preference
        name="loglevel"
        value="DEBUG" />
    <preference
        name="useBrowserHistory"
        value="true" />
    <preference
        name="exit-on-suspend"
        value="false" />
    <preference
        name="showTitle"
        value="true" />
</widget>
```
极其重要的信息我已经做了注释，我想大家都能看懂，那就没问题了。**这个配置文件是极其重要的，必须要有，切记切记！**

至此，集成就结束了。那么如何开发？我简单讲一下native这边需要做什么。

- **Plugin**：大家在上文中可以看到插件的配置，那么插件是什么？其实就是cordova提供给native和js通信的管道，我们需要自己实现一个插件类，参考[这里](http://my.oschina.net/Cphone/blog/491003)
- **AllowBridgeAccess**：在插件类中，我们需要：

``` java
@Override
    public Boolean shouldAllowBridgeAccess(String url) {
        return true;
    }
```
这样我们的插件才可以被允许作为bridge（生效）。
- **Lifecycle**：为了后续布局的扩展，我没有选择extend CordovaActivity，那么我需要模仿CordovaActivity处理相关生命周期，这里我的布局很简单：

``` xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/exam_home"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <org.apache.cordova.engine.SystemWebView
        android:id="@+id/cordovaWebView"
        android:layout_width="match_parent"
        android:layout_height="match_parent" />

</LinearLayout>
```
然后我们看activity中的代码：

``` java
@ContentView(R.layout.activity_exam)
public class ExamActivity extends BaseFragmentActivity {
    private static final String EXAM_CENTER_URL = "";
    //测试环境url
    private static final String EXAM_CENTER_URL_TEST = "";
    private static final String ERROR_URL = "file:///android_asset/404.html";
    @InjectView(R.id.cordovaWebView)
    protected SystemWebView webView;
    @InjectView(R.id.exam_home)
    protected LinearLayout examHome;
    private CordovaWebView cordovaWebView;
    protected CordovaInterfaceImpl cordovaInterface;
    private LoadingDialogFragment loading;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        init();
    }

    private void init() {
        setTitle(R.string.drawer_examination);
        loading = LoadingDialogFragment.newInstance(false, getString(R.string.loading));
        loading.setCancelable(false);
        initWebView();
        loadExamCenterUrl();
    }

    private void initWebView() {
        ConfigXmlParser parser = new ConfigXmlParser();
        parser.parse(getActivity());
        cordovaInterface = new CordovaInterfaceImpl(getActivity()) {
            @Override
            public Object onMessage(String id, Object data) {
                if ("onPageStarted".equals(id)) {
                    showRequestLoading();
                    return true;
                }
                if ("onPageFinished".equals(id)) {
                    hideRequestLoading();
                    return true;
                }
                if ("onReceivedError".equals(id)) {
                    cordovaWebView.loadUrl(ERROR_URL);
                    return true;
                }
                return super.onMessage(id, data);
            }
        };
        if (NetworkUtils.isNetworkConnected(getActivity())) {
            webView.getSettings().setCacheMode(WebSettings.LOAD_DEFAULT);
        } else {
            webView.getSettings().setCacheMode(WebSettings.LOAD_CACHE_ELSE_NETWORK);
        }
        cordovaWebView = new CordovaWebViewImpl(new SystemWebViewEngine(webView));
        if (!cordovaWebView.isInitialized()) {
            cordovaWebView.init(cordovaInterface, parser.getPluginEntries(), parser.getPreferences());
        }
    }

    private void loadExamCenterUrl() {
        if (AppConfig.isApkInDebug()) {
            cordovaWebView.loadUrl(getExamCenterUrl(EXAM_CENTER_URL_TEST));
        } else {
            cordovaWebView.loadUrl(getExamCenterUrl(EXAM_CENTER_URL));
        }
    }

    private String getExamCenterUrl(String initUrl) {
        return "http://www.baidu.com";
    }

    private void hideRequestLoading() {
        synchronized (loading) {
            loading.dismiss();
        }
    }

    private void showRequestLoading() {
        synchronized (loading) {
            loading.show(getSupportFragmentManager());
        }
    }

    @Override
    protected void onResume() {
        super.onResume();
        if (cordovaWebView != null) {
            cordovaWebView.handleResume(true);
        }
    }

    @Override
    protected void onPause() {
        super.onPause();
        if (cordovaWebView != null) {
            cordovaWebView.handlePause(true);
        }
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        examHome.removeView(webView);
        webView.removeAllViews();
        if (cordovaWebView != null) {
            cordovaWebView.handleDestroy();
        }
    }

    @Override
    protected void onStart() {
        super.onStart();
        if (cordovaWebView != null) {
            cordovaWebView.handleStart();
        }
    }

    @Override
    protected void onStop() {
        super.onStop();
        if (cordovaWebView != null) {
            cordovaWebView.handleStop();
        }
    }
}
```

可以看出，这里对声明周期的处理模仿了CordovaActivity的生命周期，同时对基本native加载流程做了简单处理。尤其onDestory中的处理可以避免报webview.destory()的错误。这段代码适用于任何一个没有extend CordovaActivity进行cordova开发的activity。

还有最最重要的一点：**在正式打包apk的时候，一定要记得，在proguard-rules.pro文件中，去掉插件类的混淆**，不能混淆插件类，否则打出来release包之后，进入混合开发的activity，会让你崩到爽……

OK，就是这样了，有什么疑问欢迎大家交流。
