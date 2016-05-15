title: React-Native Android 热更新
date: 2015-12-03 21:58:00
categories: hybrid
tags: react-native
---
转载请注明出处：http://richardcao.me/

继上次[React-native Android 初次踩坑之旅](http://richard-cao.github.io/2015/11/24/React-native-Android-%E5%88%9D%E6%AC%A1%E8%B8%A9%E5%9D%91%E4%B9%8B%E6%97%85/)的分享之后，这次分享的内容是React-native Android 热更新实现。

本文目录：

>*  网上已知方案
>*  分析与发现
>*  新的热更新方案

撰写本文基于的开发环境：
>*  操作系统：OS X 10.11.1
>*  react-native Android版本：0.16.0
>*  npm中react-native版本： 0.16.0-rc

## 网上已知方案
首先说下网上已有的方案，这是我找到的方案，并且测试确实可以热更新的：[React-Native-Remote-Update](https://github.com/fengjundev/React-Native-Remote-Update)。简述下这个方案实现热更新原理是反射调用了`ReactInstanceManagerImpl`中的如下方法：
```java
private void recreateReactContextInBackground(
      JavaScriptExecutor jsExecutor,
      JSBundleLoader jsBundleLoader) {
    UiThreadUtil.assertOnUiThread();

    ReactContextInitParams initParams = new ReactContextInitParams(jsExecutor, jsBundleLoader);
    if (!mIsContextInitAsyncTaskRunning) {
      // No background task to create react context is currently running, create and execute one.
      ReactContextInitAsyncTask initTask = new ReactContextInitAsyncTask();
      initTask.execute(initParams);
      mIsContextInitAsyncTaskRunning = true;
    } else {
      // Background task is currently running, queue up most recent init params to recreate context
      // once task completes.
      mPendingReactContextInitParams = initParams;
    }
  }
```
然后通过自定义`JSBundleLoader`将bundle指向的文件重定向，反射调用这个方法就可以实现热更新，重新加载重定向之后的bundle文件，这个bundle文件就是从服务端下载好的。

这个方案是通过反射调用private方法实现的热更新，在我看来还是有些不安全的，Facebook没把这个方法public应该是有原因的，可能他们没想用这种方法去公开的实现热更新，那么也许在迭代的过程中，可能这个反射调用的方法就失效了，那么我认为用这个方案做线上的热更新是不太安全的。
## 分析与发现
在更新react-native Android版本之后，我发现在`ReactInstanceManager.Builder`中有这么一个方法可以使用：
```java
/**
     * Path to the JS bundle file to be loaded from the file system.
     *
     * Example: {@code "assets://index.android.js" or "/sdcard/main.jsbundle}
     */
    public Builder setJSBundleFile(String jsBundleFile) {
      mJSBundleFile = jsBundleFile;
      return this;
    }
```
看这个注释，意思就是可以通过这个方法实现bundle文件的重定向。也就是说，我们可以通过这个方法来实现热更新。具体思路继续往下看，其实挺简单。
## 新的热更新方案
首先，看更改之后的`ReactInstanceManager`单例变成什么样子了：
```java
/**
 * Created by caolicheng on 15/11/12.
 */
public class CrowdReactInstanceManager {
    public final static String MODULE_NAME = "CrowdReactApp";

    private CrowdReactInstanceManager() {

    }

    public static ReactInstanceManager getInstance() {
        return Holder.sInstance;
    }

    private static class Holder {
        private static ReactInstanceManager sInstance = ReactInstanceManager.builder()
                .setApplication((Application) ElemeApplicationContext.getContext())
                .setJSMainModuleName("react-native/index.android")
                .addPackage(new MainReactPackage())
                .addPackage(new CrowdReactPackage())
                .setUseDeveloperSupport(BuildConfig.DEBUG)
                .setInitialLifecycleState(LifecycleState.RESUMED)
                .setJSBundleFile(ReactJsBundleInstanceManager.getInstance().getJSBundleFile())
                .build();
    }

}
```
可以很清晰的看到，之前的`setBundleAssetName`方法被删除了，取而代之的是`setJSBundleFile`方法，里面是`ReactJsBundleInstanceManager`这个单例，从这个单例中直接拿出bundle文件路径，相当于bundle被重定向了。
我们再看看这个`ReactJsBundleInstanceManager`到底是个什么：
```java
/**
 * Created by caolicheng on 15/12/2.
 */
public class ReactJsBundleInstanceManager {
    public static final String BUNDLE_NAME = "index.android.bundle";

    private ReactJsBundleInstanceManager() {

    }

    public static JSBundleManager getInstance() {
        return Holder.sInstance;
    }

    private static class Holder {
        private static JSBundleManager sInstance = new JSBundleManager.Builder()
                .setBundleAssetName(BUNDLE_NAME)
                .setAssetDir(ElemeApplicationContext.getContext().getFilesDir())
                .setEnabled(!BuildConfig.DEBUG)
                .build();
    }

}
```
我们很明显的看到了这是一个叫`JSBundleManager`这个东西的单例，这个东西设置了bundle的名字、bundle的父文件路径、在release时候启动更新。也就是说，`JSBundleManager`中完成了react-native热更新的逻辑，说白了就是：下载新的bundle，替换旧的bundle。

那么来了一个问题：因为`ReactInstanceManager`是个单例，也就是说，`setJSBundleFile`的路径一开始就已经固定了，那么如果我们把bundle文件打包在assets文件夹下的话，就要在一开始的时候把assets文件目录下的bundle文件copy一份到我们的热更新bundle的路径下，类似这样：
```java
public void initReactNative() {
        if (PreferenceManager.isFirstStart()) {
            try {
                File bundle = new File(ReactJsBundleInstanceManager.getInstance().getJSBundleFile());
                IOHelpers.saveStream(getAssets().open("index.android.bundle"), bundle);
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
```
react-native热更新就完成了。
这里我把我的`JSBundleManager`代码贴出来，这部分其实是最关键的，但是我的代码没法直接使用，只是告诉大家一个思路：
```java
public class JSBundleManager {
    public static final String BUNDLE_VERSION = "bundle_version";
    private static final String DEFAULT_BUNDLE_VERSION = "0.0.0";
    private final String mBundleAssetName;
    private final Callback mCallback;
    private final Boolean mEnabled;
    private final File assetDir;
    private Upgrader upgrader;

    JSBundleManager(@NonNull String bundleAssetName, @NonNull File bundleDir,
                    @Nullable Callback callback, @Nullable Boolean enabled) {
        mBundleAssetName = bundleAssetName;
        mCallback = callback;
        mEnabled = enabled;
        assetDir = new File(bundleDir, "assets");
        upgrader = new Upgrader();
    }

    public String getJSBundleFile() {
        File assetFile = new File(assetDir, mBundleAssetName);
        if (assetFile.exists()) {
            return assetFile.getAbsolutePath();
        }
        return "assets://" + mBundleAssetName;
    }

    //获取bundle版本号
    public String getBundleVersion() {
        return SharedManageUtils.getString(BUNDLE_VERSION, DEFAULT_BUNDLE_VERSION);
    }

    public JSBundleManager checkUpdate(AppVersion appVersion) {
        if (mEnabled == null || mEnabled) {
            checkAndDownloadUpdate(appVersion);
        }
        return this;
    }

    private void downloadBundle(final AppVersion appVersion) {
        ReactUpdateInfo reactUpdateInfo = new ReactUpdateInfo(ElemeApplicationContext.getContext());
        reactUpdateInfo.setDownloadUrl(appVersion.getDownloadUrl());
        upgrader.upgrade(reactUpdateInfo, new DownloadProgressListener() {
            @Override
            public void onProgressChanged(int progress) {
                if (mCallback != null) {
                    mCallback.onDownloading();
                }
            }
        }, new DownloadResultListener() {
            @Override
            public void downloadSuccess(DownloadFile file) {
                try {
                    File bundle = new File(getJSBundleFile());
                    if (bundle.exists()) {
                        bundle.delete();
                    }
                    FileUtil.copyFile(file.getFile(), new File(assetDir.getPath(), mBundleAssetName));
                    SharedManageUtils.set(BUNDLE_VERSION, appVersion.getLatestVersion());
                } catch (IOException e) {
                    if (mCallback != null) {
                        mCallback.onError(e);
                    }
                    e.printStackTrace();
                } finally {
                    FileUtil.deleteFile(file.getFile());
                }
                if (mCallback != null) {
                    mCallback.onUpdateReady();
                }
            }

            @Override
            public void downloadFail(Exception e) {
                if (mCallback != null) {
                    mCallback.onError(e);
                }
            }
        });
    }

    private void checkAndDownloadUpdate(AppVersion appVersion) {
        if (appVersion.isUpdate()) {
            downloadBundle(appVersion);
        } else {
            if (mCallback != null) {
                mCallback.onNoUpdate();
            }
        }
    }

    public interface Callback {
        void onDownloading();

        void onError(Exception e);

        void onNoUpdate();

        void onUpdateReady();
    }

    public static class Builder {

        private String mBundleAssetName;
        private File mAssetDir;
        private Callback mCallback;
        private Boolean mEnabled;

        public Builder setBundleAssetName(@NonNull final String bundleAssetName) {
            mBundleAssetName = bundleAssetName;
            return this;
        }

        public Builder setAssetDir(@NonNull final File assetDir) {
            mAssetDir = assetDir;
            return this;
        }

        public Builder setCallback(@Nullable final Callback callback) {
            mCallback = callback;
            return this;
        }

        public Builder setEnabled(@Nullable final Boolean enabled) {
            mEnabled = enabled;
            return this;
        }

        public JSBundleManager build() {
            return new JSBundleManager(mBundleAssetName, mAssetDir, mCallback, mEnabled);
        }
    }
}
```
简单来说，思路就是：当外部调用`checkUpdate`方法的时候，传进来的`AppVersion`是从服务端获取到的数据，包含bundle最新的版本号、是否需要更新和下载链接等信息。判断之后如果需要更新，那么就下载bundle到缓存中，如果成功下载，就把bundle复制到我这里自己定义热更新bundle的`assetDir`文件夹中，最后删除缓存中的bundle文件。

至此，我的react-native热更新方案就结束了。

欢迎大家互相讨论交流。