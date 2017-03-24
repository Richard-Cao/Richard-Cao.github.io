title: React-Native-APP-With-Testing
date: 2016-05-28 23:53:25
categories:	testing
tags: react-native
---

这两天，我尝试了一下在开发react-native app的过程中如何去组建一套比较容易理解和使用的测试框架进行UT和component UT，这里做一下整理。
PS：最终代码在reading项目中, [欢迎关注](https://github.com/attentiveness/reading)。

## 为何要尝试UT
其实我接触UT部分，也就是这两天。如今很多人都知道TDD的开发模式，由测试来驱动开发，优秀的代码是离不开优秀的测试的。在产品快速迭代的过程中，我们很容易忽略测试这一重要的环节，目前公司里我负责的app正好就处在这个状态。很多问题是测试很难测出来的，尤其在资源紧张的情况下，我认为，需要研发工程师使用UT来尽可能保证质量，并且在已有的功能重构和回归方面，通过各种工具手段来减少测试资源的占用。

注：本文只谈react-native app的测试。在对蜂鸟众包app中react-native部分添加UT之前，我使用了reading项目先进行一些了解和踩坑。

## Jest
我对js方面的测试框架不太了解，对react-native相关的就更不了解了。[Jest](https://facebook.github.io/jest/)这个框架是我看到react-native源码中有使用，所以按照react-native源码中的配置，我进行了尝试。具体细节在reading的[try jest](https://github.com/attentiveness/reading/commit/b38b3f325ec504bf41971f2085311ac7b56ef9c9)这次提交中，整个配置我也是各种尝试，因为引用react-native作为框架开发的项目毕竟和依赖react-native源码中测试的配置部分有不少不同的地方，经过各种尝试，我按照fb的方式跑起来了一个测试用例，感觉对我来说有点晦涩……Jest我看了下是个挺全面的框架，包括断言、mock等部分，而且在做component UT的时候，我感觉有些麻烦，用到了好多react-dom的东西。也许是博大精深我不太会用，之前也没有怎么了解过这个，就想寻求一种相对更简单交互更好的方式。

## Mocha & Enzyme & Chai
[Mocha](https://mochajs.org/)是我之前有稍微看过一下下的（其实就是看过官方主页），感觉很简洁，很方便的样子。[Chai](http://chaijs.com/)是一个很简洁的断言库，可以和Mocha结合做非UI代码的UT。关于react-native component UI部分的UT，我发现用[Enzyme](https://github.com/airbnb/enzyme)的很多，这是airbnb开源出来的一个react的js测试库，可以结合Mocha，同时可以结合react-native一起用，看起来也挺简洁的，所以我就想把这三个结合到一起用。下面就详细讲一下这种方式吧，以reading项目为例。

这里我在写测试代码的时候，为了和项目代码保持一致，统一采用ES6.

### 首先需要安装以下依赖

```shell
npm i babel-core babel-preset-es2015 babel-preset-react-native chai enzyme mocha react-addons-test-utils react-dom react-native-mock --save-dev
```
### 添加.babelrc文件

```
{
  "presets": ["es2015", "react-native"]
}

```

### 配置测试脚本

```
"scripts": {
		...
    "test": "mocha --require react-native-mock/mock.js --compilers js:babel-core/register --recursive your_app/**/*/*.spec.js"
  },
```
这种配置路径`your_app/**/*/*.spec.js"`是表示在app目录下的两层目录下后缀为`.spec.js`的均为测试代码，这是根据我reading的路径来走的，你可以根据你自己项目的具体路径来配置。

### 编写第一个测试代码
这里我第一个写的非常简单的测试用例是给reading中`Button`组件写的，代码如下：

```JavaScript
'use strict';

import React from 'react';
import {View, Text, StyleSheet} from 'react-native';
import {shallow} from 'enzyme';
import {expect} from 'chai';
import Button from '../Button';

describe('<Button />', () => {
  it('it should render 1 Text component', () => {
    const wrapper = shallow(<Button />);
    expect(wrapper.find(Text)).to.have.length(1);
  });

});
```
这里例子非常简单了，其实就是检测一下`Button`这个组件中是否渲染了一个`Text`组件。然后执行脚本：

```
npm test
```
你就会看到非常人性化的输出了：

```
~/workspace/reading  master ✔                                                                                23h37m
▶ npm test

> reading@0.1.5-rc test /Users/richardcao/workspace/reading
> mocha --require react-native-mock/mock.js --compilers js:babel-core/register --recursive app/**/*/*.spec.js



  <Button />
    ✓ it should render 1 Text component


  1 passing (393ms)
```
到这里，本文就结束了。关于TDD和自动化测试这部分，我基本了解的也不多，所以本文也只是针对入门级想尝试对react-native做UI的同学参考。

## 后续
首先，我是一个连测试用例都不会写的菜鸡，但是我一直觉得开发写UT是挺有必要的，也是结合我目前遇到的情况总结出来的，慢慢的我会学一些自动化测试相关的内容，从reading开始，尽量慢慢向TDD模式转化，也会在公司内部用起来。抛砖引玉，有什么好想法的同学，欢迎交流。

本文相关详细代码请看[reading项目](https://github.com/attentiveness/reading)。