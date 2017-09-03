title: 五步让Hexo博客支持PWA
date: 2017-09-03 23:52:33
categories: hexo
tags: pwa
---

> 某天，突发奇想看了看PWA，觉得很有意思。正好自己有个小博客，遂实验：Hexo博客极速支持PWA。本文不讲PWA的技术原理，只给出让Hexo博客支持PWA的最快方法。

# 第一步

支持PWA的第一步，便是全站HTTPS。由于我的博客采用Hexo+Gitpage搭建，完全静态，没有任何后端服务，那怎么支持全站HTTPS呢？Gitpage本身是不能用自定义域名HTTPS的。于是我另辟蹊径，发现了[Clowdflare](https://www.cloudflare.com/)服务。这个服务可以让我的静态博客也支持HTTPS，只要更换DNS就可以，而且免费套餐就足够我用，这对于贫穷的我来说简直再好不过了。

具体过程这里就不赘述了，可以参考[这篇文章](https://liaokun.me/2017/07/10/buildBlog/)或者自行Google。全站HTTPS完成后，打开你的博客，会看到一个小绿锁，那就没啥问题了。

如果你想检测安全性，可以使用[SSL Labs](https://www.ssllabs.com/ssltest/analyze.html)进行分析，生成报告。

# 第二步

你需要`cd`到你的博客工程目录下，敲一行命令：

```
npm install hexo-offline --save
```

（其实这个工具是用来直接生成ServiceWorker代码的，Vue.js官网就使用了这个插件，感兴趣的小伙伴可以直接看它的仓库）

然后在站点**_config.yml**中配置如下内容：

```
# Offline
## Config passed to sw-precache
## https://github.com/JLHwung/hexo-offline
offline:
  maximumFileSizeToCacheInBytes: 10485760
  staticFileGlobs:
    - public/**/*.{js,html,css,png,jpg,jpeg,gif,svg,json,xml}
  stripPrefix: public
  verbose: true
  runtimeCaching:
    # CDNs - should be cacheFirst, since they should be used specific versions so should not change
    - urlPattern: /*
      handler: cacheFirst
      options:
        origin: cdnjs.cloudflare.com
```

这是我的配置，仅供参考。

# 第三步

PWA需要有一个manifest.json文件，你需要将创建这个文件到**source**目录下。如何快速生成manifest文件呢？推荐一款工具：[App Manifest Generator](https://app-manifest.firebaseapp.com/)，直接生成无烦恼。

相关的icon图片放在博客的**source/images/icons**目录下就可以。没有这个目录就创建一个，记得要和manifest.json中的icon路径匹配。

# 第四步

Manifest.json文件需要在head标签里引用，我使用的是next主题，所以我在**layout/_partials/head.swing**文件中添加一句代码：

```
<link rel="manifest" href="/manifest.json">
```

如果不是next主题，你可以直接打开你的博客，然后用过chrome的开发者工具找到**head**标签，再从你的代码里找相关的代码，只要找到这么一个文件就可以了。

# 第五步

部署你的博客！

# 检测

当你完成上面五步操作之后，你的Hexo博客就已经支持PWA啦！如果你想生成一份你的博客对于PWA的相关检测报表，可以安装一个chrome插件：Lighthouse。打开你的博客，直接运行它，稍等片刻，就会生成一份报告。