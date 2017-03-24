title: 聊聊Code Review
date: 2016-09-30 11:26:27
categories: 杂谈
tags: code review tools
---

> 最近思考一个问题，如何进行高效的codereview，有没有好的工具可以使用，于是花了两三天时间在Google里淘了一番，这里留下记录。

# [Phabricator](https://www.phacility.com/)

首屈一指的codereview工具，当然并不限于codereview，这个工具我体验了下，功能很强大。除了codereview之外，还有task，bug的管理，wiki管理，项目管理等功能，而且还有自定义的功能，界面也很清爽。个人觉得几乎没什么可以挑剔的地方，如果正好需要一套工具互相配合的团队，选这个就没错了。（如果是我，我就比较倾向于这个工具的）

* [体验Phabricator](https://secure.phabricator.com/)

# [ReviewNinja](https://app.review.ninja/)

刚开始体验这个工具，纯粹是因为好奇，被这名字吸引住了：英文+日文的读音，再加上我又是火影迷……体验之后感觉还真的不错。

这个工具非常轻量级，而且只支持github，很适合个人、小团队使用。专注于codereview这个功能，界面什么的也很清爽，可以通过一些特殊的comment符号让github的merge按钮产生响应的变化，同时还会改变github pr的checks。如果我的场景只有github，那我会选择用这个工具，接入也非常简单，开源免费。

* [ReviewNinja的review效果](https://github.com/Richard-Cao/ReviewNinja-Welcome/pull/1)

# [Codacy](https://www.codacy.com)

这个工具有点像Phabricator，不过这个工具有代码质量的统计和建议，还有分析，codereview功能也很全，还有dashboard可以一览项目的各种指标，非常赞，关键是这个工具可以对接github、bitbucket、jira和Jenkins，还可以对接hipchat和slack等，功能很强大，值得好好挖掘一下。个人觉得这个工具适用范围挺广的，一些对项目质量有追求的不仅仅限于codereview的可以尝试用一用，对github上的public仓库是免费的。

* [Codacy Features](https://www.codacy.com/features)
* [Codacy体验效果](https://www.codacy.com/app/403164405/reading/dashboard)

# [RhodeCode](https://rhodecode.com/)

支持git，svn，多仓库管理，界面体验也比较清爽，功能和codacy有不少重合的地方。

* [RhodeCode Features](https://rhodecode.com/features)
* [RhodeCode demo](https://try.rhodecode.com/)

# [Gerrit](https://www.gerritcodereview.com/)

这是Google开源的codereview工具，和Phabricator并驾齐驱，也很强大，只不过我个人不太喜欢这个界面风格……这个我没有自己去搭过，只是看了官网的一些信息，功能上不输Phabricator。喜欢的朋友可以去体验一把。

# 总结

以上列举了我这两三天着重看的一些codereview工具，适用场景也大概总结了一下。高效的codereview非常重要，如果有好的工具帮助我们进行codereview，往往会达到事半功倍的效果。