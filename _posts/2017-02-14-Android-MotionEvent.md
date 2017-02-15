---
layout:     post
title:      Android MotionEvent
date:       2017-02-13
summary:    分析Android MotionEvent
categories: jekyll pixyll
---

Github:[https://github.com/twolight](https://github.com/twolight)

Blog:[https://twolight.github.io/](https://twolight.github.io/)

Email: [twolight88@gmail.com](twolight88@gmail.com)



Android MotionEvent 对象描述了一个触摸屏幕的动作

* #### 触摸位置

  ``getX``和``getY``：Touch点的相对坐标，相对于消费此事件的View的左上角的坐标

  `getRawX()`和`getRawY()`：Touch点的绝对坐标，相对于屏幕的左上角

* #### 触摸类型

  ##### 单个手指

  1. ``MotionEvent.ACTION_DOWN`` 第一个手指按下屏幕
  2. ``MotionEvent.ACTION_MOVE``
  3. ``MotionEvent.ACTION_UP`` 最后一个手指离开屏幕


  ##### 多个手指

  1. ``MotionEvent.ACTION_POINTER_DOWN`` 第二个手指按下屏幕
  2. ``MotionEvent.ACTION_POINTER_UP`` 多个手指中的一个离开屏幕


* #### Point

  上面提到触摸类型，有单个手指和多个手指的情况， 用``Point``来标示一个触摸点。

  ##### Point的属性：

  1. id  在整个事件流中是不会发生变化的
  2. x,y
  3. index  在整个事件流中会发生变化

  ``MotionEvent``实际上课理解为对一个或多个``Point``进行了封装。通过它，可以获取上面的``Point``的属性。

  ##### getAction：

  获取的int值是由pointer的index值和事件类型值组合。例如`` index`` 0x01类型 0x0015，``getAction``获取的int值死0x0115

  ##### getActionMasked：

  获取事件的类型值

  ##### getActionInde：

  获取Point index

  ##### getPointerId(int index)：

  通过Point index获取Point id 

* #### 实践

  ##### 例子1：RecyclerView 嵌套横竖滑动冲突

  ````
  @Override
  public boolean onInterceptTouchEvent(MotionEvent e) {  
    ...

    switch (action) {
      case MotionEvent.ACTION_DOWN:
          ...

      case MotionEvent.ACTION_MOVE: {
          ...

          if (mScrollState != SCROLL_STATE_DRAGGING) {
            boolean startScroll = false;
            if (canScrollHorizontally && Math.abs(dx) > mTouchSlop) {
              ...
              startScroll = true;
            }
            if (canScrollVertically && Math.abs(dy) > mTouchSlop) {
              ...
              startScroll = true;
            }
            if (startScroll) {
              setScrollState(SCROLL_STATE_DRAGGING);
            }
         }
      } break;
        ...

    }
    return mScrollState == SCROLL_STATE_DRAGGING;
  }
  ````

  ​

  上面是``RecyclerView``源码。在外层上下滑动``RecyclerView``嵌套左右滑动的 ``RecyclerView``，当用户左右滑动的时候，外层的``RecyclerView``将拦截了事件的传递，内层``RecyclerView``是获取不到滑动事件的。内层``RecyclerView``的滑动就会出现问题。事件传递相关链接[Android事件转发机制](https://twolight.github.io/jekyll/pixyll/2017/02/13/Android-Touch-Event-Dispatch/)

  ​

  ````
  @Override
  public boolean onInterceptTouchEvent(MotionEvent e) {
          final int action = MotionEventCompat.getActionMasked(e);
          
          switch (action) {
              case MotionEvent.ACTION_DOWN:
              	//当前只有一个触摸点，getPointerId(0)即可获得Point id
                  mScrollPointerId = e.getPointerId(0);
                  mInitialTouchX = (int) (e.getX() + 0.5f);
                  mInitialTouchY = (int) (e.getY() + 0.5f);
                  return super.onInterceptTouchEvent(e);

              case MotionEventCompat.ACTION_POINTER_DOWN:
              	//当前不只一个触摸点，必须得通过当前的index获得Point id
              	final int actionIndex = MotionEventCompat.getActionIndex(e);
                  mScrollPointerId = e.getPointerId(actionIndex);
                  mInitialTouchX = (int) (e.getX(actionIndex) + 0.5f);
                  mInitialTouchY = (int) (e.getY(actionIndex) + 0.5f);
                  return super.onInterceptTouchEvent(e);

              case MotionEvent.ACTION_MOVE: {
              	//Point id 在Touch事件流中不会变，所以通过id获取index，再获得x,y坐标
              	
                  final int index = e.findPointerIndex(mScrollPointerId);
                  if (index < 0) {
                      return false;
                  }

                  final int x = (int) (e.getX(index) + 0.5f);
                  final int y = (int) (e.getY(index) + 0.5f);
                  if (getScrollState() != SCROLL_STATE_DRAGGING) {
                      final int dx = x - mInitialTouchX;
                      final int dy = y - mInitialTouchY;
                      final boolean canScrollHorizontally = getLayoutManager().canScrollHorizontally();
                      final boolean canScrollVertically = getLayoutManager().canScrollVertically();
                      boolean startScroll = false;
                      if (canScrollHorizontally && Math.abs(dx) > mTouchSlop && (Math.abs(dx) >= Math.abs(dy) || canScrollVertically)) {
                          startScroll = true;
                      }
                      if (canScrollVertically && Math.abs(dy) > mTouchSlop && (Math.abs(dy) >= Math.abs(dx) || canScrollHorizontally)) {
                          startScroll = true;
                      }
                      return startScroll && super.onInterceptTouchEvent(e);
                  }
                  return super.onInterceptTouchEvent(e);
              }

              default:
                  return super.onInterceptTouchEvent(e);
          }
      }
  ````

  解决方法就是重写``RecyclerView`` ``onInterceptTouchEvent``方法，增加拦截条件。使得左右滑动时，外层的``RecyclerView`` 不拦截事件。上面的代码，基本覆盖了``MotionEvent``的用法。

  ​

  ​




