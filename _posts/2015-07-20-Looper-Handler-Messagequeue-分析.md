---
layout:     post
title:      Looper Handler MessageQueue源码分析
date:       2015-07-20 21:10:00
summary:    一般情况，在Service做耗时操作，会在Service中另开子线程，做这部分耗时操作。幸运的是，Android提供了IntenService这个类，满足了我们在Service中做耗时操作的要求
categories: jekyll pixyll
---

## Looper Handler MessageQueue分析

我所有的文章都会提交到github上，喜欢的同学可以来github关注。欢迎提交你们的文章

Github [https://github.com/twolight/LearnNote.git](https://github.com/twolight/LearnNote.git)

邮箱 twolight88@gmail.com

### 背景

在Android，线程之间的通信，通过Hanlder发送消息，具体Hanlder是向那个线程发送消息，是根据Hanlder在哪个线程中创建的。这同时决定了Hanlder的handleMessage方法运行在哪个线程。

```
	public class MyActivity extends Activity{
		public void onCreate(...){
			//在主线程中创建，Handler发送的消息是向主线程发送，主线程处理消息
			Handler hanlder = new Hanlder();
			
		}
	}
	Thread thread = new Thread(new Runnable(){
		public void run(){
			//创建Looper
			Looper.prepare();
			//在子线程中创建，Handler发送的消息是向子线程发送，子线程处理消息
			Handler hanlder = new Hanlder();
			//Looper开始循环，从消息队列中开始取出消息处理
			Looper.loop();
		}
	});
```
了解清楚之后，再来分析消息是如何发送到线程的，在线程中又是如何如何处理消息的。

### 分析

在上面的代码中，之所以在Activity创建Hanlder之前没有调用Looper.prepare()和Looper.loop()，是因为主线程在启动时，已经自己创建Looper,并调用了loop。

所以我们知道，线程之间的通信，要经历着几个步骤。

1. 创建Looper

2. 创建Hanlder

3. Looper开始循环（如果此时并没有消息队列并没有消息，就线程就阻塞，直到有消息）

4. Hanlder发送消息到消息队列

5. Looper开始处理消息（调用handler的dispatchMessage方法）


下面分析这几个步骤

### 步骤分析
* #### 创建Looper


	```
	public static void prepare() {
        prepare(true);
    }

    private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException(...);
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
    private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }

    /**
     * Initialize the current thread as a looper, marking it as an
     * application's main looper. The main looper for your application
     * is created by the Android environment, so you should never need
     * to call this function yourself.  See also: {@link #prepare()}
     */
     
     /**
     这是Andorid主线程，自己调用的，我们不应该在代码中调用
     */
    public static void prepareMainLooper() {
        prepare(false);
        synchronized (Looper.class) {
            if (sMainLooper != null) {
                throw new IllegalStateException(...);
            }
            sMainLooper = myLooper();
        }
    }
	```
	涉及到ThreadLocal的概念，如果想要深入了解ThreadLocal机制，可以自己去了解下，这里简要说下。
	
	>Implements a thread-local storage, that is, a variable for which each thread
	 has its own value. All threads share the same {@code ThreadLocal} object,
	 but each sees a different value when accessing it, and changes made by one
	 thread do not affect the other threads. The implementation supports
	 {@code null} values.
	 
	ThreadLocal实现线程的本地变量存储。通俗的来讲，在一个Thread中ThreadLocal.set(变量)，只有这个线程能通过ThreadLocal.get拿出来这个变量。
	
	Looper.prepare()的作用是，创建一个Looper对象，并将它放到当前线程的ThreadLocal中去。并且创建一个MessageQueue.

* #### 创建Hanlder

	```
	public Handler(Callback callback, boolean async) {
		...
        mLooper = Looper.myLooper();
        if (mLooper == null) {
            ...
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
   	}
   	
   	public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }
	```
	
	由于创建Handler和创建Looper的线程是同一个线程，Handler能拿到之前创建的Looper对象，以及在Looper中创建的消息队列。
	所以这就是Threadlocal的作用，只要在同一个线程内，对于不同的对象，都能通过Threadlocal拿到共享的变量。Looper，就是这样的共享的变量。
	
* #### Looper消息循环
	关键代码
	
	```
	public static void loop() {
        final Looper me = myLooper();
        ...
        final MessageQueue queue = me.mQueue;

        ...

        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }
           ...

            msg.target.dispatchMessage(msg);

           ...
        }
        //next 方法代码多，下面简化成伪代码
        Message next(){
        	for(;;){
        		if(有消息){
        			return msg;
        		}else{
        			
        		}
        		if(需要退出){
        			return null； 
        		}
        	}
        	
        }
    }
	```
	next()方法内是一个死循环，如果消息队列没有消息，那么该方法不会返回Message对象，代码一直在其内部执行。如果MessageQueue正在退出，那么返回null。
	
	loop方法，在死循环中，不停的从消息队列取消息，没有消息，代码就一直在next方法中执行，不会有返回值。代码不会往下执行。next()如果返回null，表示消息队列正在退出，那么Looper也需要退出，就跳出死循环停止对消息队列取消息。next()返回消息，那么调用的msg.target.dispatchMessage(msg)，也就是调用Handler的dispatchMessage方法，去分发msg。
	
* #### Hanlder发送消息

	Handler发送消息，最终会调用一下代码
	
	```
	public class Handler{
		...
		private boolean enqueueMessage(MessageQueue queue, Message msg, long 
					uptimeMillis) {
			//所以知道msg的target，就是发送这个消息的Handler。
        	msg.target = this;
        	if (mAsynchronous) {
            	msg.setAsynchronous(true);
        	}
        	//将消息方法消息队列中
        	return queue.enqueueMessage(msg, uptimeMillis);
    	}
    	...
	}
	
	```
	queue.enqueueMessage(msg, uptimeMillis);将消息方法消息队列中，之前说的消息队列的next方法，会一个死循环，不停地检测是否有消息。只要成功执行了queue.enqueueMessage，那么Looper就能拿到发送的消息，并转发消息。
	
* **消息处理**

	```
	public class Handler{
		/**
     	* Subclasses must implement this to receive messages.
     	*/
     	//
		public void handleMessage(Message msg) {
	    }
	    
	    /**
	     * Handle system messages here.
	     */
	    public void dispatchMessage(Message msg) {
	        if (msg.callback != null) {
	            handleCallback(msg);
	        } else {
	            if (mCallback != null) {
	                if (mCallback.handleMessage(msg)) {
	                    return;
	                }
	            }
	            handleMessage(msg);
	        }
	    }
    }
	```
	上面的步骤，说了Looper会调用msg.target.dispatchMessage(msg);让Handler转发消息，handler会调用handleMessage方法，这是一个空实现，我们可以在子类或者匿名类中重写此方法。
	

### 总结


至此，线程间的消息机制，调用流程，就分析完了。




	
	
	

	

	
	
	
	

	
	
	

	




