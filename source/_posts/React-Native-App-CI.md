title: React Native App CI
date: 2016-12-11 22:25:50
categories: hybrid
tags: react-native
---
转载请注明出处：http://richardcao.me/

应一些小伙伴之邀，总结一篇关于react-native app CI配置的文章，分享出来抛砖引玉~

撰写本文时笔者的一些环境版本：

- macOS：10.12.1
- npm：3.10.8
- node：6.9.1

# 背景

我们在开发react-native app的过程中，需要用一些手段来保证我们的代码质量、测试与打包，这里简单列举几个例子：

- 代码lint检查
- 静态检查
- 单元测试
- Android打包apk
- iOS打包ipa
- 测试报告生成
- ...（等等其他很多设施）

这些我列举了一些大家开发过程中比较常用的一些操作。当然，如果我们本机操作的话我相信一般情况不会有人有什么障碍，但是，开发过程中我们需要节约人工时间，提高开发效率，那么我们就需要使用CI系统来帮助我们自动完成这些工作。

# CI选型

众所周知，目前比较主流的CI系统有：Travis，Jenkins，Circle等。其中，Travis，Circle比较适合开源项目和小团队，大公司 or 大团队一般都会选用Jenkins。本篇文章以[开源项目reading](https://github.com/attentiveness/reading)为例，给大家一步步解析reading的CI配置。

由于reading是一个开源项目，所以这里我选择老牌[Travis CI](https://travis-ci.org/)作为reading的CI系统，无缝对接Github。

# Eslint

reading项目中使用了eslint作为js的lint检查工具，我们在配置这个工具的时候，只要本地配置完成，那么CI只需要一行代码就可以跑我们的lint脚本。

本地是如何配置的呢？我们看看`package.json`文件中关于eslint的几个依赖：

```
"devDependencies": {
    ...
    "babel-eslint": "^7.1.1",
    "eslint": "^3.12.0",
    "eslint-config-airbnb": "^13.0.0",
    "eslint-plugin-import": "^2.2.0",
    "eslint-plugin-jsx-a11y": "^2.2.3",
    "eslint-plugin-react": "^6.8.0",
    ...
  }
```

我们可以看出，reading采用了airbnb的eslint方案`eslint-config-airbnb`为蓝本，然后根据作者自己的一些习惯进行了小范围的更改，看看项目根目录下的`.eslintrc`文件：

```
{
    "extends": "airbnb",
    "env": {
        "browser": true,
        "node": true,
        "mocha": true
    },
    "ecmaFeatures": {
        "forOf": true,
        "jsx": true,
        "es6": true
    },
    "rules": {
        "comma-dangle": 0,
        "react/prop-types": 0,
        "no-use-before-define": 0,
        "radix": 0,
        "no-param-reassign": 0,
        "react/jsx-filename-extension": 0,
        "no-mixed-operators": 0,
        "import/prefer-default-export": 0,
        "import/no-extraneous-dependencies": 0,
        "no-plusplus": 0,
        "react/prefer-stateless-function": 0,
        "class-methods-use-this": 0
    }
}
```

那么eslint配置好了之后，怎么去进行lint检查呢？，我们需要在`package.json`文件中添加一句脚本：

```
"scripts": {
    ...
    "lint": "eslint",
    ...
  },
```

最后，我们只需要在项目目录下执行：`npm run lint path/to/js文件或目录`就可以啦。

那么，这句命令也是我们要在CI配置文件中添加的，这个稍后会告诉大家如何写。

# Test

关于测试部分，我之前写过一篇文章[React-Native-APP-With-Testing](http://richardcao.me/2016/05/28/React-Native-APP-With-Testing/)，使用的方案是Mocha+Chai+Enzyme，当时对Jest方案并不是很了解，那时候Jest对React-Native的支持也不是很好，也没有相关的文档可以查阅。但是在最近我发现，Jest对React-Native的支持越来越完善，文档也有了，官方也提供了init命令的Jest相关支持，加上之前的方案中采用了各种开源项目结合，遇到过一两次坑，所以我彻底切换成了Jest来做reading项目的测试。

首先我们还是要看看`package.json`中和Jest相关的依赖：

```
"jest": {
    "preset": "react-native"
  },
  "devDependencies": {
    ...
    "babel-jest": "^17.0.2",
    "babel-preset-react-native": "^1.9.0",
    "jest": "^17.0.3",
    "react-test-renderer": "^15.4.1",
    ...
  }
```

相比之前的组合方案，用Jes之后发现测试部分少了将近一半的依赖，而且切换到Jest，相关的API几乎没有什么大的变化，稍微改一两个地方就可以完全兼容，这让我感到非常开心，所以我几乎没花什么代价就讲测试方案切换为Jest。Jest默认会执行` __tests__`或` __spec__`目录下的测试代码，都不需要像Mocha一样写配置文件，同时Jest官网还支持很多其他的配置比如生成测试报表等，非常方便。

配置好依赖之后，我们还需要配置`package.json`中的Jest执行脚本：

```
"scripts": {
    ...
    "test": "jest",
    ...
  },
```

非常简单的配置，然后我们只需要在项目目录下执行`npm test`命令即可。

# CI配置

在我们做好上面的那些准备之后，我们就要配置Travis的CI脚本文件了。我们再明确一下我们需要达到的目的：

- lint检查（eslint）
- 测试（jest）
- 打包（Android & iOS）

那么我们至少需要Linux环境和osx环境，然后在不同的OS下都要跑eslint和jest。经过reading一段时间的使用，我发现这样的CI效率并不高，每次打出包的时间都比较长，而且假设一不小心lint出了问题，包也打不出来，但是有可能这时候代码是没问题的。于是我做了优化：

- 一个Linux的环境，专门打Android包
- 一个osx环境，专门build iOS
- 再一个osx环境，专门进行js相关的lint和test

这里我做一点说明：Travis目前只有Linux环境支持Android，osx环境支持iOS，但是我本机又是mac，所以js相关的东西我还是用osx环境，这样和本机吻合度高。

经过这样的优化，三个并行环境跑ci脚本，单个环境时间上快了很多，总体时间也减少了，效率也提高了。

讲到这里，我觉得是时候上一波脚本代码了，代码在项目根目录的`.travis.yml`中：

```
# 多os相关配置（包含三个os以及env标识）
matrix:
  include:
  - os: osx
    language: objective-c
    sudo: false
    osx_image: xcode7.3
    env: TEST_TYPE=ios
  - os: linux
    language: android
    sudo: required
    jdk: oraclejdk8
    env: TEST_TYPE=android
    android:
      components:
      - build-tools-23.0.1
      - android-23
      - extra-android-m2repository
      - extra-android-support
  - os: osx
    language: node_js
    sudo: false
    node_js: 6
    env: TEST_TYPE=js
env:
  ...
# code climate配置
addons:
  code_climate:
    repo_token: $CODE_CLIMATE_TOKEN
before_cache:
...
cache:
...
before_install:
...
- rvm get head
- if [[ $TEST_TYPE == 'android' ]]; then gem install fir-cli ; fi
install:
- curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.30.0/install.sh | bash
- source ~/.bashrc
- nvm install 5
...
- travis_wait npm install
branches:
  only:
  - master
script:
- if [[ $TEST_TYPE == 'js' ]]; then npm run lint app ; fi
- if [[ $TEST_TYPE == 'js' ]]; then npm test ; fi
# TODO: We use xcodebuild because xctool would stall when collecting info about
# the tests before running them. Switch back when this issue with xctool has
# been resolved.
- if [[ $TEST_TYPE == 'ios' ]]; then xcodebuild -project ios/$PROJECT_MOBILE.xcodeproj
  -scheme $PROJECT_MOBILE -sdk iphonesimulator9.3 -configuration Debug CODE_SIGN_IDENTITY="" CODE_SIGNING_REQUIRED=NO clean build ; fi
- if [ $TEST_TYPE == 'android' ] && [ $TRAVIS_PULL_REQUEST == 'false' ]; then cd android && ./gradlew clean && ./gradlew resguard --stacktrace ; fi
...
after_success:
- if [ $TEST_TYPE == 'android' ] && [ $TRAVIS_PULL_REQUEST == 'false' ]; then fir p 
  path/to/*.apk
  -T $FIR_TOKEN -c "$TRAVIS_TAG" ; fi
  
```

这里展示的脚本代码做了一点删减，大家如果想看完整的代码可以看[这里](https://github.com/attentiveness/reading/blob/master/.travis.yml)。这个脚本里最难的地方其实就是一开始的matrix相关配置，我摸索了几天才搞定。那么之后的配置就大多按照`TEST_TYPE`进行脚本区分了。这段代码大家其实看字面意思都是可以看懂的，如果一下子看不懂可以看看[travis build的生命周期](https://docs.travis-ci.com/user/customizing-the-build/#The-Build-Lifecycle)之后再看这段代码就一目了然了，其他的命令就是把本地的一些命令挪到了CI中执行，安装好了相关的预备环境。

最后的效果很棒，build jobs的展示也一目了然，大家可以去[reading的travis页面](https://travis-ci.org/attentiveness/reading)看一看效果。

# 最后

我们一步步配置好了适用于react-native app的travis ci，其实circle和Jenkins都可以参考，原理是一样的，只是脚本有些许差别。对于现在的我来说，只要我push代码到master，过几分钟我就可以看看结果，然后[reading的β包](http://fir.im/w7gu)就会出现。我在reading的README中给出的Android β包链接其实就是master分支最新代码打出来的包，ci带来的方便我也切身体会到了。

**抛砖引玉之文**，谢谢大家，有疑问可以讨论起来。