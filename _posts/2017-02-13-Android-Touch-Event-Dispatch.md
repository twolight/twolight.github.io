---
layout:     post
title:      Android Touch Event Dispatch
date:       2017-02-13
summary:    分析Android事件转发机制
categories: jekyll pixyll
---




Android 一个点击操作产生之后，由屏幕捕获，然后交给当前Activity处理，从根布局ViewGroup 的dispatchTouchEvent开始转发。

大体的转发流程，看到网上一个评论说的比较好

> 一次完整的Touch事件，应该是由一个Down、一个Up和若干个Move组成的。Down方式通过dispatchTouchEvent分发，分发的目的是为了找到真正需要处理完整Touch请求的View。当某个View或者ViewGroup的onTouchEvent事件返回true时，便表示它是真正要处理这次请求的View，之后的Aciton_UP和Action_MOVE将由它处理。当所有子View的onTouchEvent都返回false时，这次的Touch请求就由根ViewGroup自己处理了



* dispatchTouchEvent

  起到Touch事件转发、判断的作用

* onTouchEvent

  在这个方法里处理Touch事件（一般会重写这个方法做Touch事件处理）

* onIntercepTouchEvent (ViewGroup)

  起到ViewGroup对Touch事件拦截的作用



下面是关于Touch事件传递的几种情况，图片来自于[eoe](http://www.eoeandroid.com/thread-319301-1-1.html)



* ACTION_DOWN未被消费







![wqwq](http://img.blog.csdn.net/20140226202907812)

​	

* ACTION_DOWN被View消费了



![](http://img.blog.csdn.net/20140226203028734)



* 后续ACTION_MOVE和UP在不被拦截的情况下都会去找VIEW



![](http://img.blog.csdn.net/20140226203125390)





* 后续的被拦截了

  ![](http://img.blog.csdn.net/20140226203214000)





* ACTION_DOWN一开始就被拦截

![](http://img.blog.csdn.net/20140226203301578)





##### 关于ViewParent的requestDisallowInterceptTouchEvent

* ViewParent



> Defines the responsibilities for a class that will be a parent of a View. This is the API that a view sees when it wants to interact with its parent.

定义一个类作为View的parent，这个接口用于子View与其Parent交互

* requestDisallowInterceptTouchEvent

  ​

> Called when a child does not want this parent and its ancestors to intercept touch events with`onInterceptTouchEvent(MotionEvent)`.



让View的所有父节点，都不能通过onInterceptTouchEvent打断Touch Event



参考：

[http://blog.csdn.net/yanzi1225627/article/details/22592831](http://blog.csdn.net/yanzi1225627/article/details/22592831)


















