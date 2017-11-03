title: Jenkins中使用Appium的最佳姿势
date: 2017-11-03 12:48:33
categories: testing
tags: appium
---

> 最近在公司内部申请了一台云主机当做我们组自己的slave机器，于是把之前从北京侧同事那边借的Job迁移过来，appium相关的环境要全部重新配置。上次同事帮了我很多忙，所以我并不太清楚Jenkins这块儿到底应该怎么配置，这次是完全自己走了一遍坑，记录一下遇到的问题。

撰写本文时笔者的云主机上一些环境版本：

- Linux：Ubuntu 14.04
- npm：5.4.2
- node：6.11.5
- appium：1.7.1

# 环境变量

在配好slave机器之后，已经有了一些默认的环境变量（我的机器上是Android环境），要配置appium，我们需要npm、node套件。我是一个比较保守的人，于是选择了node 6和npm 5，在Ubuntu上安装了这些，然后

```shell
npm install appium -g
```
等待安装完毕之后，基本的appium环境就具备了。于是我跑了一遍脚本，GG。看了报错发现appium、ruby等都没识别出来，对比之前的机器我发现因为环境变量没有新增，我的机器上Jenkins相关的环境变量配置在/data/config/config.properties文件中，我添加了node、ruby路径然后配置了PATH变量，**重启了机器**，再跑一遍Job，发现appium、ruby等识别出来了。

注：网络环境不好的话，大概率可能会出来类似这样的log：

```
info UiAutomator2 downloading UiAutomator2 Server APK v0.1.9 : https://github.com/appium/appium-uiautomator2-server/releases/download/v0.1.9/appium-uiautomator2-server-v0.1.9.apk
info UiAutomator2 downloading UiAutomator2 Server test APK v0.1.9 : https://github.com/appium/appium-uiautomator2-server/releases/download/v0.1.9/appium-uiautomator2-server-debug-androidTest.apk
```

然后一直卡在这里不动了。那解决方式就简单了，手动下载这俩app，然后放到对应的目录~/.nvm/versions/node/v6.11.5/lib/node_modules/appium/node_modules/appium-uiautomator2-driver/uiautomator2/中就行。如果没有uiautomator2子目录，就新建一个。

# 进程守护

在配置好appium环境之后，运行

```shell
appium
```

就可以启动appium进程。本地这么做当然没什么问题，但是在Jenkins上这么做就有问题了。首先，我们要在一个Job里跑脚本，当运行appium之后，整个会话就被堵塞了，后面的脚本没法执行了，所以我们一定要让appium server跑在后台，同时还要管理appium的日志已经启动/停止appium server。

## 方法一

我们都知道，Linux系统中有一个Upstart服务管理程序，它可以让一些服务在后台运行并且管理，相关的服务我们可以在/etc/init/目录下找到。既然它可以让服务在后台运行，我们也可以使用它来管理我们的appium服务。有了这个思路，我们只需要在/etc/init/目录下新增一个配置文件appium.conf：

```shell
stop on shutdown
respawn

script
  . /data/config/config.properties
  cd /data/log/appium
  exec appium -a 127.0.0.1 --session-override > appium.log
end script
```

这里提一点，在Jenkins机器上跑appium，要想远程连接真机，需要指定ip为127.0.0.1，本机直接运行appium默认的ip是0.0.0.0，这个在Jenkins机器上是无法连接远程真机的。
这个配置文件写好保存好之后，我们运行起来就简单了，这里只需要了解三个命令：

```shell
# 启动appium服务
start appium
```

```shell
# 停止appium服务
stop appium
```

```shell
# 查看appium服务状态
status appium
```

然后appium的log被重定向到一个自定义的log文件中，就可以在Jenkins Job里输出相关的log内容了。在运行start appium之后，服务在后台运行，Job就可以继续执行后续脚本，达到了一个Job运行appium测试的目的。我目前使用的就是这个方法。

## 方法二

appium服务是一个node server，既然这样，那可以选取node应用的相关工具来管理。这里推荐pm2，一个非常强大的node服务管理工具，安装也非常简单：

```shell
npm install pm2 -g
```

使用的话，如果我们要达到上面方法一的效果，启动appium的命令应该这么写：

```shell
pm2 start appium -- -a 127.0.0.1
```

其中--代表了后面跟着appium识别的参数，这里并没有重定向log文件，因为pm2已经具备了比较强大的日志管理功能。在~/.pm2/logs/目录下，启动appium之后会生成2个日志文件，一个error的日志，一个普通的log日志，这个就非常强大了，同时提供了

```shell
pm2 flush
```

可以用来清空日志，重新收集。停止服务也非常简单：

```shell
pm2 stop appium
```

其他相关用法请自行Google。**记得使用pm2启动服务，一定要设置相应的环境变量，否则GG！**

# 环境检查

运行appium脚本之前，我们需要通过appium-doctor工具检查运行环境是否都具备了。安装appium-doctor很简单：

```shell
npm install appium-doctor -g
```

然后检查一下环境：

```shell
appium-doctor
```

# 踩坑集锦

在Jenkins机器上，Job运行起来之后，报了一个大大的ERROR：

```
An unknown server-side error occurred while processing the command. Original error: JAVA_HOME is not set currently. Please set JAVA_HOME. (Selenium::WebDriver::Error::UnknownError)
```

我就纳闷了，appium-doctor都没问题，怎么跑起来有问题了呢？我又观察了log输出，发现了存在问题的log：

```
[UiAutomator2] Unable to remove port forward 'Error executing adbExec. Original error: 'Command '/data/tools/android_sdk/platform-tools//adb -P 5037 -s 172.22.36.116\:7401 forward --remove tcp\:8200' exited with code 1'; Stderr: 'error: listener 'tcp:8200' not found'; Code: '1''
```

没看懂什么意思，于是我去Google搜了一波，搜到的信息大多都和我实际遇到的问题无关，包括JAVA_HOME没设置的问题，也没收获。于是我又仔细观察了log，发现了一个疑点：

```
[debug] [UiAutomator2] Deleting UiAutomator2 session
[debug] [UiAutomator2] Deleting UiAutomator2 server session
[UiAutomator2] Did not get confirmation UiAutomator2 deleteSession worked; Error was: Error: Trying to proxy a session command without session id
```

这个log看起来是有问题的，于是我又去Google搜了一波，依然没什么收获，我以为我的机器有毒，我就回到自己电脑上更新了一下appium，然后本地跑了一波，居然报了同样的错误！这我就不能忍了，本机为什么报这样的错误？既然搜不到，也只能在日志里寻找蛛丝马迹。本机报错的时候，我发现多了一些日志输出：

```
[MJSONWP] Encountered internal error running command: Error: JAVA_HOME is not set currently. Please set JAVA_HOME.
    at getJavaHome (../../lib/helpers.js:110:9)
    at getJavaForOs (../../lib/helpers.js:99:17)
    at ADB.callee$0$0$ (../../../lib/tools/apk-signing.js:106:16)
    at tryCatch (/usr/local/lib/node_modules/appium/node_modules/.5.8.24@babel-runtime/regenerator/runtime.js:67:40)
    at GeneratorFunctionPrototype.invoke [as _invoke] (/usr/local/lib/node_modules/appium/node_modules/.5.8.24@babel-runtime/regenerator/runtime.js:315:22)
    at GeneratorFunctionPrototype.prototype.(anonymous function) [as next] (/usr/local/lib/node_modules/appium/node_modules/.5.8.24@babel-runtime/regenerator/runtime.js:100:21)
    at invoke (/usr/local/lib/node_modules/appium/node_modules/.5.8.24@babel-runtime/regenerator/runtime.js:136:37)
    at enqueueResult (/usr/local/lib/node_modules/appium/node_modules/.5.8.24@babel-runtime/regenerator/runtime.js:185:17)
    at Promise.F (/usr/local/lib/node_modules/appium/node_modules/.1.2.7@core-js/library/modules/$.export.js:30:36)
    at AsyncIterator.enqueue (/usr/local/lib/node_modules/appium/node_modules/.5.8.24@babel-runtime/regenerator/runtime.js:184:12)
    at AsyncIterator.prototype.(anonymous function) [as next] (/usr/local/lib/node_modules/appium/node_modules/.5.8.24@babel-runtime/regenerator/runtime.js:100:21)
    at Object.runtime.async (/usr/local/lib/node_modules/appium/node_modules/.5.8.24@babel-runtime/regenerator/runtime.js:209:12)
    at ADB.callee$0$0 [as checkApkCert] (../../../lib/tools/apk-signing.js:106:13)
    at UiAutomator2Server.checkAndSignCert$ (../../lib/uiautomator2.js:114:33)
    at tryCatch (/usr/local/lib/node_modules/appium/node_modules/.5.8.24@babel-runtime/regenerator/runtime.js:67:40)
    at GeneratorFunctionPrototype.invoke [as _invoke] (/usr/local/lib/node_modules/appium/node_modules/.5.8.24@babel-runtime/regenerator/runtime.js:315:22)
    at GeneratorFunctionPrototype.prototype.(anonymous function) [as next] (/usr/local/lib/node_modules/appium/node_modules/.5.8.24@babel-runtime/regenerator/runtime.js:100:21)
    at invoke (/usr/local/lib/node_modules/appium/node_modules/.5.8.24@babel-runtime/regenerator/runtime.js:136:37)
    at enqueueResult (/usr/local/lib/node_modules/appium/node_modules/.5.8.24@babel-runtime/regenerator/runtime.js:185:17)
    at Promise.F (/usr/local/lib/node_modules/appium/node_modules/.1.2.7@core-js/library/modules/$.export.js:30:36)
    at AsyncIterator.enqueue (/usr/local/lib/node_modules/appium/node_modules/.5.8.24@babel-runtime/regenerator/runtime.js:184:12)
    at AsyncIterator.prototype.(anonymous function) [as next] (/usr/local/lib/node_modules/appium/node_modules/.5.8.24@babel-runtime/regenerator/runtime.js:100:21)
    at Object.runtime.async (/usr/local/lib/node_modules/appium/node_modules/.5.8.24@babel-runtime/regenerator/runtime.js:209:12)
    at UiAutomator2Server.checkAndSignCert (../../lib/uiautomator2.js:114:15)
    at UiAutomator2Server.signAndInstall$ (../../lib/uiautomator2.js:108:16)
    at tryCatch (/usr/local/lib/node_modules/appium/node_modules/.5.8.24@babel-runtime/regenerator/runtime.js:67:40)
    at GeneratorFunctionPrototype.invoke [as _invoke] (/usr/local/lib/node_modules/appium/node_modules/.5.8.24@babel-runtime/regenerator/runtime.js:315:22)
    at GeneratorFunctionPrototype.prototype.(anonymous function) [as next] (/usr/local/lib/node_modules/appium/node_modules/.5.8.24@babel-runtime/regenerator/runtime.js:100:21)
    at invoke (/usr/local/lib/node_modules/appium/node_modules/.5.8.24@babel-runtime/regenerator/runtime.js:136:37)
    at enqueueResult (/usr/local/lib/node_modules/appium/node_modules/.5.8.24@babel-runtime/regenerator/runtime.js:185:17)
    at Promise.F (/usr/local/lib/node_modules/appium/node_modules/.1.2.7@core-js/library/modules/$.export.js:30:36)
    at AsyncIterator.enqueue (/usr/local/lib/node_modules/appium/node_modules/.5.8.24@babel-runtime/regenerator/runtime.js:184:12)
    at AsyncIterator.prototype.(anonymous function) [as next] (/usr/local/lib/node_modules/appium/node_modules/.5.8.24@babel-runtime/regenerator/runtime.js:100:21)
    at Object.runtime.async (/usr/local/lib/node_modules/appium/node_modules/.5.8.24@babel-runtime/regenerator/runtime.js:209:12)
```

这段日志显然是非常有用的，我摘取了日志中非常重要的几条信息：

- 我发现出现这个报错的场景是重新安装UiAutomator2相关的2个apk时候报的，大概猜测在重新安装的时候对这两个apk进行了签名相关的校验
- 报错主要的信息应该在uiautomator2.js、tools/apk-signing.js和helpers.js中，这些文件都在appium的node_modules中

获得了这些非常有用的信息之后，我打算查一下代码，看看到底是什么问题。首先看uiautomator2.js，这个文件在appium的appium-uiautomator2-driver依赖库中，定位到出问题的代码：

```javascript
  async signAndInstall (apk, apkPackage, timeout = SERVER_INSTALL_RETRIES * 1000, test = false) {
    await this.checkAndSignCert(apk, apkPackage);
    await this.adb.install(apk, true, timeout);
    logger.info(`Installed UiAutomator2 server${test ? ' test' : ''} apk`);
  }

  async checkAndSignCert (apk, apkPackage) {
    let signed = await this.adb.checkApkCert(apk, apkPackage);
    if (!signed) {
      await this.adb.sign(apk);
    }
    return !signed;
  }
```

checkAndSignCert就是检查并且签名的操作，这部分代码定位到之后，下一步寻找的就是tools/apk-signing.js，这个文件在appium的appium-adb依赖库中，定位到出问题的代码：

```javascript
apkSigningMethods.checkApkCert = async function (apk, pkg) {
  const java = getJavaForOs();
  if (!(await fs.exists(apk))) {
    log.debug(`APK doesn't exist. ${apk}`);
    return false;
  }
  if (this.useKeystore) {
    return await this.checkCustomApkCert(apk, pkg);
  }
  log.debug(`Checking app cert for ${apk}.`);
  try {
    await exec(java, ['-jar', path.resolve(this.helperJarPath, 'verify.jar'), apk]);
    log.debug("App already signed.");
    await this.zipAlignApk(apk);
    return true;
  } catch (e) {
    log.debug("App not signed with debug cert.");
    return false;
  }
};
```

继续定位，根据log发现是getJavaForOs方法出了问题，这个方法在helpers.js中，正好与log中下一步的报错信息吻合：

```javascript
function getJavaForOs () {
  const sep = path.sep;
  let java = `${getJavaHome()}${sep}bin${sep}java`;
  if (system.isWindows()) {
    java = java + '.exe';
  }
  return java;
}
```

通过日志我们可以看到是getJavaHome方法报错了，继续定位：

```javascript
function getJavaHome () {
  if (process.env.JAVA_HOME) {
    return process.env.JAVA_HOME;
  }
  throw new Error("JAVA_HOME is not set currently. Please set JAVA_HOME.");
}
```

到这里，精准定位问题了，throw的就是最开始的报错信息。原来appium-adb拿到java环境变量是通过process.env获取的，而process.env是node获取环境变量的手段，也就是说，如果没有export java home到环境变量中，这里拿到的肯定是undefined，就会抛出异常。我发现我的mac电脑还真没有export JAVA_HOME，于是我加上了这个变量，重新运行appium，完美运行。

为什么升级了一下appium就会出现问题了呢？显然是appium的这些依赖库代码更新了。我在appium的package.json中发现了这么一句依赖：

```
"appium-uiautomator2-driver": "0.x",
```

本机问题解决了，Jenkins机器上报的错是一模一样的，肯定是同样的问题。我仔细检查了Jenkins机器上环境变量的配置，突然发现，环境变量配置文件里只是配置了java home，并没有export！于是我在.bashrc中加了export操作，然后重启了机器，运行了一次Job……

失败！

求助了一波同事，发现其实我这种start appium的方式，服务运行的环境不一样，所以需要在appium.conf里加一句话：

```shell
script
  . /data/config/config.properties
  # 加的就是这一句
  export JAVA_HOME=$JAVA_HOME
  cd /data/log/appium
  exec appium -a 127.0.0.1 --session-override > appium.log
end script
```

**果然成功了！**

# 总结

其实这些问题都不是什么大问题，但是在出现问题之后，我通过各种手段解决了它。最后Job开开心心的跑了起来，我重新配置上了定时任务，让它在迁移机器之后更好的发挥作用，做出更多的贡献！