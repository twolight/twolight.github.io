---
layout:     post
title:      Android Launch Mode
date:       2017-02-8
summary:    Android launch mode分析
categories: jekyll pixyll
---

在Google官方网站的关于Launch Mode的文档也是模拟两可,细节讲的不够完全，有很多case需要自己去手动的测试，才能明白。官方地址：[launchMode](https://developer.android.com/guide/topics/manifest/activity-element.html#lmode)

	
一个Activity组件通过四种启动模式配合Intent的`FLAG_ACTIVITY_`标志启动.

下面一张官方的图

![sdsd](http://twolight.github.io/images/439590A8-7AD1-4EA0-AE94-17D24E5C3D67.png)

使用standard和singleTop，Activity会被实例化多次，并且实例可以属于任何Task，存在于Activity栈的任何地方。singleTask 和 singleInstance只有一个实例。


* standard

系统每次都会在目标task创建一个activity实例，并且把Intent传递给实例

* singleTop

如果目标task的顶部存在一个实例，系统将通过调用onNewIntent()把Intent传递给已存在的实例，否则就创建一个实例。

* singleTask

当前activity栈中是否存在一个实例，如果存在就将实例之前的activity结束，然后重用这个实例并且回调onNewIntent()方法，否则创建一个新的task，并且把实例放到其底部。

* singleInstance

和singleTask基本相同，区别在与，task在放置了singleInstance模式的activity实例之后，将不再放置其他activity实例。




下面通过几张流程图来说明就更清楚了：


![](http://twolight.github.io/images/7CA174ED-F146-45F6-9F59-13EA5ED60E0F.png)

![](http://twolight.github.io/images/7C19C14E-420D-4A7C-B365-E1B59FFC8365.png)

![](http://twolight.github.io/images/B2EF1E62-A19C-4FA0-B902-32EEF047F7ED.png)








