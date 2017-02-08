---
layout:     post
title:      IntentService源码分析
date:       2016-07-20 14:10:00
summary:    一般情况，在Service做耗时操作，会在Service中另开子线程，做这部分耗时操作。幸运的是，Android提供了IntenService这个类，满足了我们在Service中做耗时操作的要求
categories: jekyll pixyll
---

##IntentService分析

我所有的文章都会提交到github上，喜欢的同学可以来github关注。欢迎提交你们的文章

Github [https://github.com/twolight/LearnNote.git](https://github.com/twolight/LearnNote.git)

邮箱 twolight88@gmail.com

###背景###

Service运行在调用它的进程和线程中，在开发当中，在Activity中启动Service，意味着Service将运行在主线程中。所以，它的onXxx方法，并不能做耗时的操作的，不然将阻塞主线程，可能导致ARN。

一般情况，在Service做耗时操作，会在Service中另开子线程，做这部分耗时操作。幸运的是，Android提供了IntenService这个类，满足了我们在Service中做耗时操作的要求。

###分析##

下面将分析IntenService内部是如何做到的。下面是主要的代码

``` 
	public abstract class IntentService extends Service{
		private final class ServiceHandler extends Handler {
	        public ServiceHandler(Looper looper) {
	            super(looper);
	        }
	
	        @Override
	        public void handleMessage(Message msg) {
	            onHandleIntent((Intent)msg.obj);
	            stopSelf(msg.arg1);
	        }
	    }
		@Override
	    public void onCreate() {
	        // TODO: It would be nice to have an option to hold a partial wakelock
	        // during processing, and to have a static startService(Context, Intent)
	        // method that would launch the service & hand off a wakelock.
	
	        super.onCreate();
	        HandlerThread thread = new HandlerThread("IntentService[" + mName + "]");
	        thread.start();
	
	        mServiceLooper = thread.getLooper();
	        mServiceHandler = new ServiceHandler(mServiceLooper);
	    }
	    @Override
	    public void onStart(Intent intent, int startId) {
	        Message msg = mServiceHandler.obtainMessage();
	        msg.arg1 = startId;
	        msg.obj = intent;
	        mServiceHandler.sendMessage(msg);
	    }
	    @Override
	    public void onDestroy() {
	        mServiceLooper.quit();
	    }
	    @WorkerThread
	    protected abstract void onHandleIntent(Intent intent);
	}
```


在IntenService onCreate方法中，创建一个HandlerThread 线程对象并启动起来。

看看HandlerThread的实现

```
	public class HandlerThread extends Thread{
		...
		
		@Override
	    public void run() {
	        mTid = Process.myTid();
	        Looper.prepare();
	        synchronized (this) {
	            mLooper = Looper.myLooper();
	            notifyAll();
	        }
	        Process.setThreadPriority(mPriority);
	        onLooperPrepared();
	        Looper.loop();
	        mTid = -1;
	    }
	    ...
	}
```
HandlerThread 确实是一个线程，果然IntentService实现也是另外开一个线程。
在这个子线程中启动、运行的时候，创建一个Looper，并让Loop开始从消息队列取出消息进行处理。

```
	public static void loop() {
        ...
        final MessageQueue queue = me.mQueue;
        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }
            msg.target.dispatchMessage(msg);
        }
    }

```
Looper的loop方法中，for无限循环的调用消息，通过注释//might block，如果消息队列为空，那么线程会阻塞在这里，直到有消息队列不为空。取出来的消息如果为null，通过注释// No message indicates that the message queue is quitting.。知道message queue正在退出，Looper就跳出循环，不在处理消息。不为null，则调用Hander的dispatchMessage方法。

Looper/Handler知识，请看我的另外一篇文章
[Looper Handler MessageQueue分析](https://github.com/twolight/LearnNote/blob/master/Android/Looper_Handler_Messagequeue%E5%88%86%E6%9E%90.md)



IntentService onCreate方法中，启动启动线程之后，获得这个子线程的Looper，并且将它与ServiceHandler这个Handler绑定。


那我们就知道了，ServiceHandler其实是用来向子线程发送消息的。子线程通过Looper，去遍历消息队列，取出消息，回调ServiceHandler的handleMessage方法，进而调用onHandleIntent方法。onHandleIntent方法，就是需要我们自己实现的耗时操作。

###总结###

* IntentService创建时

	创建一个Hander和子线程，handler与子线程的Looper绑定。线程run方法中,Looper一直循环等待处理消息。所以子线程不会退出，一直等待消息的到来。

* IntentService启动时

	通过Handler向子线程发送消息，子线程遍历消息队列，发现有消息，并处理消息，调用handleMessage方法，handleMessage方法调用onHandleIntent方法。onHandleIntent方法中我们实现自己的耗时操作。也就是说耗时操作，是在子线程运行的，并不会阻塞主线程。另外，onHandleIntent调用完接着会调用stopSelf(msg.arg1)方法，停止IntentService。系统回调onDestroy方法，让子线程的Looper退出，不在循环处理消息。子线程的run方法也就调用完毕，子线程自己结束释放线程资源。
	
	

		
	



















