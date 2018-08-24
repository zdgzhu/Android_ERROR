# Handler运行机制及源码解析

## 一、概述

### 1.1 Handler 、 Looper 、Message  之间的关系

Handler 、 Looper 、Message 这三者都与Android异步消息处理线程相关的概念。那么什么叫异步消息处理线程呢？

 

异步消息处理线程启动后会进入一个无限的循环体之中，每循环一次，从其内部的消息队列中取出一个消息，然后回调相应的消息处理函数，执行完成一个消息后则继续循环。若消息队列为空，线程则会阻塞等待。

 

说了这一堆，那么和Handler 、 Looper 、Message有啥关系？其实Looper负责的就是创建一个MessageQueue，然后进入一个无限循环体不断从该MessageQueue中读取消息，而消息的创建者就是一个或多个Handler 。

## 二、 Loopr 源码解析

对于Looper主要是prepare()和loop()两个方法。 首先看prepare()方法 

### 2.1 prepare() 

```
public static final void prepare() {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(true));
```

sThreadLocal是一个ThreadLocal对象，可以在一个线程中存储变量。可以看到，在第5行，将一个Looper的实例放入了ThreadLocal，并且2-4行判断了sThreadLocal是否为null，否则抛出异常。**这也就说明了Looper.prepare()方法不能被调用两次**，同时也保证了一个线程中只有一个Looper实例~相信有些哥们一定遇到这个错误。 下面看Looper的构造方法： 

###2.2 Looper的构造方法 

```
private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread(); //获取当前的线程
}
```

Looper的构造函数初始化了一个MessageQueue，从上面的结论可以得知，不同线程调用Looper.prepare()创建looper会将looper对象保存在对应的线程当中，保证了不同线程之前的looper实例单独保存，互不影响。同理，不同线程之间的MessageQueue也是单独保存，互不影响。

因此我们可以得知，Handler在当前线程初始化，则它对应的Looper和MessageQueue都保存在当前的线程当中，与其它的线程是隔离开来的。再来看看Looper中的loop函数，它负责从MessageQueue中取出message。

在构造方法中，创建了一个MessageQueue（消息队列），和一个线程。 然后我们看loop()方法： 

###2.3 loop()方法 

```
public static void loop() {
        final Looper me = myLooper();
        if (me == null) {
            throw new RuntimeException("No Looper; Looper.prepare() wasn't called on this thread.");
        }
        final MessageQueue queue = me.mQueue;
 
        // Make sure the identity of this thread is that of the local process,
        // and keep track of what that identity token actually is.
        Binder.clearCallingIdentity();
        final long ident = Binder.clearCallingIdentity();
 
        for (;;) { //代表无线循环的意思
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }
 
            // This must be in a local variable, in case a UI event sets the logger
            Printer logging = me.mLogging;
            if (logging != null) {
                logging.println(">>>>> Dispatching to " + msg.target + " " +
                        msg.callback + ": " + msg.what);
            }
 
            msg.target.dispatchMessage(msg);
 
            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }
 
            // Make sure that during the course of dispatching the
            // identity of the thread wasn't corrupted.
            final long newIdent = Binder.clearCallingIdentity();
            if (ident != newIdent) {
                Log.wtf(TAG, "Thread identity changed from 0x"
                        + Long.toHexString(ident) + " to 0x"
                        + Long.toHexString(newIdent) + " while dispatching to "
                        + msg.target.getClass().getName() + " "
                        + msg.callback + " what=" + msg.what);
            }
 
            msg.recycle();
        }
}

```

**for(;;){} 代表无线循环的意思 **

第2行： public static Looper myLooper() { return sThreadLocal.get(); } 方法直接返回了sThreadLocal存储的Looper实例，如果me为null则抛出异常，也就是说looper方法必须在prepare方法之后运行。

 第6行：拿到该looper实例中的mQueue（消息队列） 

13到45行：就进入了我们所说的无限循环。 14行：取出一条消息，如果没有消息则阻塞。 

27行：使用调用 msg.target.dispatchMessage(msg);把消息交给msg的target的dispatchMessage方法去处理。Msg的target是什么呢？其实就是handler对象，下面会进行分析。

 44行：释放消息占据的资源。 

 ### 2.4 Looper主要作用

1、	与当前线程绑定，保证一个线程只会有一个Looper实例，同时一个Looper实例也只有一个MessageQueue。     2、	loop()方法，不断从MessageQueue中去取消息，交给消息的target属性的dispatchMessage去处理。 好了，我们的异步 消息处理线程已经有了消息队列（MessageQueue），也有了在无限循环体中取出消息的哥们，现在缺的就是发送消息的对象了，于是乎：Handler登场了。 

##三、Handler 源码解析

使用Handler之前，我们都是初始化一个实例，比如用于更新UI线程，我们会在声明的时候直接初始化，或者在onCreate中初始化Handler实例。所以我们首先看Handler的构造方法，看其如何与MessageQueue联系上的，它在子线程中发送的消息（一般发送消息都在非UI线程）怎么发送到MessageQueue中的。 

###3.1 Handler()

```
public Handler() {
        this(null, false);
}
public Handler(Callback callback, boolean async) {
        if (FIND_POTENTIAL_LEAKS) {
            final Class<? extends Handler> klass = getClass();
            if ((klass.isAnonymousClass() || klass.isMemberClass() || klass.isLocalClass()) &&
                    (klass.getModifiers() & Modifier.STATIC) == 0) {
                Log.w(TAG, "The following Handler class should be static or leaks might occur: " +
                    klass.getCanonicalName());
            }
        }
 
        mLooper = Looper.myLooper();
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }

```

14行：通过Looper.myLooper()获取了当前线程保存的Looper实例，然后在19行又获取了这个Looper实例中保存的MessageQueue（消息队列），这样就保证了handler的实例与我们Looper实例中MessageQueue关联上了。

然后看我们最常用的sendMessage方法

###3.2 sendMessage() 

```
   public final boolean sendMessage(Message msg)
    {
        return sendMessageDelayed(msg, 0);
    }

```

### 3.3 sendEmptyMessageDelayed()

```
   public final boolean sendEmptyMessageDelayed(int what, long delayMillis) {
        Message msg = Message.obtain();
        msg.what = what;
        return sendMessageDelayed(msg, delayMillis);
    }

```

###3.4 sendMessageDelayed() 

```
 public final boolean sendMessageDelayed(Message msg, long delayMillis)
    {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }

```

###3.5 sendMessageAtTime() 

```
 public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }

```

###3.6 enqueueMessage()

辗转反则最后调用了sendMessageAtTime，在此方法内部有直接获取MessageQueue然后调用了enqueueMessage方法，我们再来看看此方法： 

```
 private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
        msg.target = this;
        if (mAsynchronous) {
            msg.setAsynchronous(true);
        }
        //将消息保存到消息队列中去
        return queue.enqueueMessage(msg, uptimeMillis);
    }

```

enqueueMessage中首先为meg.target赋值为this，【如果大家还记得Looper的loop方法会取出每个msg然后交给msg,target.dispatchMessage(msg)去处理消息】，也就是把当前的handler作为msg的target属性。最终会调用queue的enqueueMessage的方法，也就是说handler发出的消息，最终会保存到消息队列中去。

现在已经很清楚了Looper会调用prepare()和loop()方法，在当前执行的线程中保存一个Looper实例，这个实例会保存一个MessageQueue对象，然后当前线程进入一个无限循环中去，不断从MessageQueue中读取Handler发来的消息。然后再回调创建这个消息的handler中的dispathMessage方法，下面我们赶快去看一看这个方法：

### 3.7 dispatchMessage() 

```
public void dispatchMessage(Message msg) {
//这里msg的callback和target都有值，那么会执行哪个呢？
如果不为null，则执行callback回调，也就是我们的Runnable对象。
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

```

可以看到，第10行，调用了handleMessage方法，下面我们去看这个方法： 

```
  /**
     * Subclasses must implement this to receive messages.
     */
    public void handleMessage(Message msg) {
    }

```

可以看到这是一个空方法，为什么呢，因为消息的最终回调是由我们控制的，我们在创建handler的时候都是复写handleMessage方法，然后根据msg.what进行消息处理。

###3.8 handleMessage() 

例如：

```
private Handler mHandler = new Handler()
	{
		public void handleMessage(android.os.Message msg)
		{
			switch (msg.what)
			{
			case value:
				
				break;
 
			default:
				break;
			}
		};
	};

```

到此，这个流程已经解释完毕，让我们首先总结一下

###3.9 总结

1、首先Looper.prepare()在本线程中保存一个Looper实例，然后该实例中保存一个MessageQueue对象；因为Looper.prepare()在一个线程中只能调用一次，所以MessageQueue在一个线程中只会存在一个。

2、Looper.loop()会让当前线程进入一个无限循环，不端从MessageQueue的实例中读取消息，然后回调msg.target.dispatchMessage(msg)方法。

3、Handler的构造方法，会首先得到当前线程中保存的Looper实例，进而与Looper实例中的MessageQueue想关联。

4、Handler的sendMessage方法，会给msg的target赋值为handler自身，然后加入MessageQueue中。

5、在构造Handler实例时，我们会重写handleMessage方法，也就是msg.target.dispatchMessage(msg)最终调用的方法。

好了，总结完成，大家可能还会问，那么在Activity中，我们并没有显示的调用Looper.prepare()和Looper.loop()方法，为啥Handler可以成功创建呢，这是因为在Activity的启动代码中，已经在当前UI线程调用了Looper.prepare()和Looper.loop()方法。

## 四、Handler.post()

今天有人问我，你说Handler的post方法创建的线程和UI线程有什么关系？

其实这个问题也是出现这篇博客的原因之一；这里需要说明，有时候为了方便，我们会直接写如下代码：

###4.1 mHandler.post()

```
mHandler.post(new Runnable()
		{
			@Override
			public void run()
			{
				Log.e("TAG", Thread.currentThread().getName());
				mTxt.setText("yoxi");
			}
		});

```

然后run方法中可以写更新UI的代码，其实这个Runnable并没有创建什么线程，而是发送了一条消息，下面看源码：

```
 public final boolean post(Runnable r)
    {
       return  sendMessageDelayed(getPostMessage(r), 0);
    }

```

###4.2 getPostMessage()

```
  private static Message getPostMessage(Runnable r) {
        Message m = Message.obtain();
        m.callback = r;
        return m;
    }

```

可以看到，在getPostMessage中，得到了一个Message对象，然后将我们创建的Runable对象作为callback属性，赋值给了此message.

注：**产生一个Message对象，可以new  ，也可以使用Message.obtain()方法；两者都可以，但是更建议使用obtain方法，因为Message内部维护了一个Message池用于Message的复用，避免使用new 重新分配内存。**

###4.3 sendMessageDelayed()

```
 public final boolean sendMessageDelayed(Message msg, long delayMillis)
    {
        if (delayMillis < 0) {
            delayMillis = 0;
        }
        return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
    }

```

### 4.4 sendMessageAtTime() 

```
 public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
        MessageQueue queue = mQueue;
        if (queue == null) {
            RuntimeException e = new RuntimeException(
                    this + " sendMessageAtTime() called with no mQueue");
            Log.w("Looper", e.getMessage(), e);
            return false;
        }
        return enqueueMessage(queue, msg, uptimeMillis);
    }

```

最终和handler.sendMessage一样，调用了sendMessageAtTime，然后调用了enqueueMessage方法，给msg.target赋值为handler，最终加入MessagQueue.

 ### 4.5 msg的callback和target都有值，那么会执行哪个呢？

可以看到，这里msg的callback和target都有值，那么会执行哪个呢？

其实上面已经贴过代码，就是dispatchMessage方法：

```
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

```

第2行，如果不为null，则执行callback回调，也就是我们的Runnable对象。

好了，关于Looper , Handler , Message 这三者关系上面已经叙述的非常清楚了。

##五、图解

![](D:\AndroidFile\Photo\Handler\handler_01.png)





##六、子线程中创建Handler

###6.1 例子

Looper的作用是不断的轮询当前线程MessageQueue消息队列中的消息，然后交给Handler去处理。要使handler消息机制开始运作，就必须使当前线程生成一个Looper(一个线程只能有一个Looper与之关联) 

```
         new Thread(new Runnable() {
                    @Override
                    public void run() {
                    //在当前线程中初始化一个Looper轮询器
                        Looper.prepare();
                        mHandler = new Handler(Looper.myLooper(), new Handler.Callback() {
                            @Override
                            public boolean handleMessage(Message msg) {
                                Log.e(TAG, "handleMessage: " );
                                return false;
                            }
                        });
                      //  使Looper轮询器开始轮询
                        Looper.loop();
                    }
                }).start();

//发送消息
mHandler.sendEmptyMessage(1); 
```



## 七、HandlerThread

###7.1 工作原理

内部原理 = `Thread`类 + `Handler`类机制，即：

- 通过继承`Thread`类，**快速地创建1个带有Looper对象的新工作线程**
- 通过封装`Handler`类，**快速创建Handler & 与其他线程进行通信**



### 7.2 `HandlerThread`的使用步骤 

```
// 步骤1：创建HandlerThread实例对象
// 传入参数 = 线程名字，作用 = 标记该线程
   HandlerThread mHandlerThread = new HandlerThread("handlerThread");

// 步骤2：启动线程
   mHandlerThread.start();

// 步骤3：创建工作线程Handler & 复写handleMessage（）
// 作用：关联HandlerThread的Looper对象、实现消息处理操作 & 与其他线程进行通信
// 注：消息处理操作（HandlerMessage（））的执行线程 = mHandlerThread所创建的工作线程中执行
  Handler workHandler = new Handler( handlerThread.getLooper() ) {
            @Override
            public boolean handleMessage(Message msg) {
                ...//消息处理
                return true;
            }
        });

// 步骤4：使用工作线程Handler向工作线程的消息队列发送消息
// 在工作线程中，当消息循环时取出对应消息 & 在工作线程执行相关操作
  // a. 定义要发送的消息
  Message msg = Message.obtain();
  msg.what = 2; //消息的标识
  msg.obj = "B"; // 消息的存放
  // b. 通过Handler发送消息到其绑定的消息队列
  workHandler.sendMessage(msg);

// 步骤5：结束线程，即停止线程的消息循环
  mHandlerThread.quit();
```

###7.3 步骤1：创建HandlerThread的实例对象

```
/**
  * 具体使用
  * 传入参数 = 线程名字，作用 = 标记该线程
  */ 
   HandlerThread mHandlerThread = new HandlerThread("handlerThread");

/**
  * 源码分析：HandlerThread类的构造方法
  */ 
    public class HandlerThread extends Thread {
    // 继承自Thread类

        int mPriority; // 线程优先级
        int mTid = -1; // 当前线程id
        Looper mLooper; // 当前线程持有的Looper对象

       // HandlerThread类有2个构造方法
       // 区别在于：设置当前线程的优先级参数，即可自定义设置 or 使用默认优先级

            // 方式1. 默认优先级
            public HandlerThread(String name) {
                // 通过调用父类默认的方法创建线程
                super(name);
                mPriority = Process.THREAD_PRIORITY_DEFAULT;
            }

            // 方法2. 自定义设置优先级
            public HandlerThread(String name, int priority) {
                super(name);
                mPriority = priority;
            }
            ...
     }
```

总结

- `HandlerThread`类继承自`Thread`类
- 创建`HandlerThread`类对象 = 创建`Thread`类对象 + 设置线程优先级 = **新开1个工作线程 + 设置线程优先级**

### 7.4 步骤2：启动线程

```
/**
  * 具体使用
  */ 
   mHandlerThread.start();

/**
  * 源码分析：此处调用的是父类（Thread类）的start()，最终回调HandlerThread的run（）
  */ 
  @Override
    public void run() {
        // 1. 获得当前线程的id
        mTid = Process.myTid();

        // 2. 创建1个Looper对象 & MessageQueue对象
        Looper.prepare();

        // 3. 通过持有锁机制来获得当前线程的Looper对象
        synchronized (this) {
            mLooper = Looper.myLooper();

            // 发出通知：当前线程已经创建mLooper对象成功
            // 此处主要是通知getLooper（）中的wait（）
            notifyAll();

            // 此处使用持有锁机制 + notifyAll() 是为了保证后面获得Looper对象前就已创建好Looper对象
        }

        // 4. 设置当前线程的优先级
        Process.setThreadPriority(mPriority);

        // 5. 在线程循环前做一些准备工作 ->>分析1
        // 该方法实现体是空的，子类可实现 / 不实现该方法
        onLooperPrepared();

        // 6. 进行消息循环，即不断从MessageQueue中取消息 & 派发消息
        Looper.loop();

        mTid = -1;
    }
}

/**
  * 分析1：onLooperPrepared();
  * 说明：该方法实现体是空的，子类可实现 / 不实现该方法
  */ 
    protected void onLooperPrepared() {

    }
```

总结  

1. 为当前工作线程（即步骤1创建的线程）创建1个`Looper`对象 & `MessageQueue`对象  

2. 通过持有锁机制来获得当前线程的`Looper`对象  
3. 发出通知：当前线程已经创建mLooper对象成功  4. 工作线程进行消息循环，即不断从MessageQueue中取消息 & 派发消息 

###7.5 步骤3：创建工作线程Handler & 复写handleMessage（） 

```
/**
  * 具体使用
  * 作用：将Handler关联HandlerThread的Looper对象、实现消息处理操作 & 与其他线程进行通信
  * 注：消息处理操作（HandlerMessage（））的执行线程 = mHandlerThread所创建的工作线程中执行
  */ 
   Handler workHandler = new Handler( handlerThread.getLooper() ) {
            @Override
            public boolean handleMessage(Message msg) {
                ...//消息处理
                return true;
            }
        });

/**
  * 源码分析：handlerThread.getLooper()
  * 作用：获得当前HandlerThread线程中的Looper对象
  */ 
    public Looper getLooper() {
        // 若线程不是存活的，则直接返回null
        if (!isAlive()) {
            return null;
        } 
        // 若当前线程存活，再判断线程的成员变量mLooper是否为null
        // 直到线程创建完Looper对象后才能获得Looper对象，若Looper对象未创建成功，则阻塞
        synchronized (this) {


            while (isAlive() && mLooper == null) {
                try {
                    // 此处会调用wait方法去等待
                    wait();
                } catch (InterruptedException e) {
                }
            }
        }
        // 上述步骤run（）使用 持有锁机制 + notifyAll()  获得Looper对象后
        // 则通知当前线程的wait（）结束等待 & 跳出循环
        // 最终getLooper()返回的是在run（）中创建的mLooper对象
        return mLooper;
    }
```

总结

- 在获得`HandlerThread`工作线程的`Looper`对象时存在一个同步的问题：只有当线程创建成功 & 其对应的`Looper`对象也创建成功后才能获得`Looper`的值，才能将创建的`Handler` 与 工作线程的`Looper`对象绑定，从而将`Handler`绑定工作线程
- 解决方案：即保证同步的解决方案 = 同步锁、`wait（）` 和 `notifyAll（）`，即 在`run()`中成功创建`Looper`对象后，立即调用`notifyAll（）`通知 `getLooper()`中的`wait（）`结束等待 & 返回`run()`中成功创建的`Looper`对象，使得`Handler`与该`Looper`对象绑定

###7.6 步骤4：使用工作线程`Handler`向工作线程的消息队列发送消息

```
/**
  * 具体使用
  * 作用：在工作线程中，当消息循环时取出对应消息 & 在工作线程执行相关操作
  * 注：消息处理操作（HandlerMessage（））的执行线程 = mHandlerThread所创建的工作线程中执行
  */ 
  // a. 定义要发送的消息
  Message msg = Message.obtain();
  msg.what = 2; //消息的标识
  msg.obj = "B"; // 消息的存放
  // b. 通过Handler发送消息到其绑定的消息队列
  workHandler.sendMessage(msg);

/**
  * 源码分析：workHandler.sendMessage(msg)
  * 此处的源码即Handler的源码，故不作过多描述
  */ 
```

###7.7 步骤5：结束线程，即停止线程的消息循环

```
/**
  * 具体使用
  */ 
  mHandlerThread.quit();

/**
  * 源码分析：mHandlerThread.quit()
  * 说明：
  *     a. 该方法属于HandlerThread类
  *     b. HandlerThread有2种让当前线程退出消息循环的方法：quit() 、quitSafely()
  */ 

  // 方式1：quit() 
  // 特点：效率高，但线程不安全
  public boolean quit() {
        Looper looper = getLooper();
        if (looper != null) {
            looper.quit(); 
            return true;
        }
        return false;
    }

  // 方式2：quitSafely()
  // 特点：效率低，但线程安全
  public boolean quitSafely() {
        Looper looper = getLooper();
        if (looper != null) {
            looper.quitSafely();
            return true;
        }
        return false;
    }

  // 注：上述2个方法最终都会调用MessageQueue.quit（boolean safe）->>分析1

/**
  * 分析1：MessageQueue.quit（boolean safe）
  */ 
    void quit(boolean safe) {
        if (!mQuitAllowed) {
            throw new IllegalStateException("Main thread not allowed to quit.");
        }
        synchronized (this) {
            if (mQuitting) {
                return;
            }
            mQuitting = true;


            if (safe) {
                removeAllFutureMessagesLocked(); // 方式1（不安全）会调用该方法 ->>分析2
            } else {
                removeAllMessagesLocked(); // 方式2（安全）会调用该方法 ->>分析3
            }
            // We can assume mPtr != 0 because mQuitting was previously false.
            nativeWake(mPtr);
        }
    }
/**
  * 分析2：removeAllMessagesLocked()
  * 原理：遍历Message链表、移除所有信息的回调 & 重置为null
  */ 
  private void removeAllMessagesLocked() {
    Message p = mMessages;
    while (p != null) {
        Message n = p.next;
        p.recycleUnchecked();
        p = n;
    }
    mMessages = null;
}
/**
  * 分析3：removeAllFutureMessagesLocked() 
  * 原理：先判断当前消息队列是否正在处理消息
  *      a. 若不是,则类似分析2移除消息
  *      b. 若是，则等待该消息处理处理完毕再使用分析2中的方式移除消息退出循环
  * 结论：退出方法安全与否（quitSafe（） 或 quit（）），在于该方法移除消息、退出循环时是否在意当前队列是否正在处理消息
  */ 
  private void removeAllFutureMessagesLocked() {

    final long now = SystemClock.uptimeMillis();
    Message p = mMessages;

    if (p != null) {
        // 判断当前消息队列是否正在处理消息
        // a. 若不是，则直接移除所有回调
        if (p.when > now) {
            removeAllMessagesLocked();
        } else {
        // b. 若是正在处理，则等待该消息处理处理完毕再退出该循环
            Message n;
            for (;;) {
                n = p.next;
                if (n == null) {
                    return;
                }
                if (n.when > now) {
                    break;
                }
                p = n;
            }
            p.next = null;
            do {
                p = n;
                n = p.next;
                p.recycleUnchecked();
            } while (n != null);
        }
    }
}
```

至此，关于`HandlerThread`源码的分析完毕。 



###7.8 HandlerThread源码总体解析 

```
public class HandlerThread extends Thread {
    int mPriority; // 线程优先级
    int mTid = -1; // 当前线程id
    Looper mLooper; // 当前线程持有的Looper对象
    
       // HandlerThread类有2个构造方法
       // 区别在于：设置当前线程的优先级参数，即可自定义设置 or 使用默认优先级
    public HandlerThread(String name) {
    // 通过调用父类默认的方法创建线程
        super(name);
        mPriority = Process.THREAD_PRIORITY_DEFAULT;
    }

    //也可以指定线程的优先级，注意使用的是 android.os.Process 而不是 java.lang.Thread 的优先级！
    // 方法2. 自定义设置优先级
    public HandlerThread(String name, int priority) {
        super(name);
        mPriority = priority;
    }

    // 子类需要重写的方法，在这里做一些执行前的初始化工作
    protected void onLooperPrepared() {
    }

    //获取当前线程的 Looper
    //如果线程不是正常运行的就返回 null
    //如果线程启动后，Looper 还没创建，就 wait() 等待 创建 Looper 后 notify
    public Looper getLooper() {
        if (!isAlive()) {
            return null;
        }

        synchronized (this) {
            while (isAlive() && mLooper == null) {    //循环等待
                try {
                    wait();
                } catch (InterruptedException e) {
                }
            }
        }
        return mLooper;
    }

    //调用 start() 后就会执行的 run()
    @Override
    public void run() {
      // 1. 获得当前线程的id
        mTid = Process.myTid();
        Looper.prepare();            //帮我们创建了 Looepr
        // 3. 通过持有锁机制来获得当前线程的Looper对象
        synchronized (this) {
            mLooper = Looper.myLooper();
             //此处主要是通知getLooper（）中的wait（）
              //Looper 已经创建，唤醒阻塞在获取 Looper 的线程
            notifyAll();   
            // 此处使用持有锁机制 + notifyAll() 是为了保证后面获得Looper对象前就已创建好Looper对象
        }
         // 4. 设置当前线程的优先级
        Process.setThreadPriority(mPriority);
         // 5. 在线程循环前做一些准备工作 ->>分析1
        // 该方法实现体是空的，子类可实现 / 不实现该方法
        onLooperPrepared();    
        // 6. 进行消息循环，即不断从MessageQueue中取消息 & 派发消息
        Looper.loop();        //开始循环
        mTid = -1;
    }

    public boolean quit() {
        Looper looper = getLooper();
        if (looper != null) {
            looper.quit();
            return true;
        }
        return false;
    }

    public boolean quitSafely() {
        Looper looper = getLooper();
        if (looper != null) {
            looper.quitSafely();
            return true;
        }
        return false;
    }

    public int getThreadId() {
        return mTid;
    }
}
```

可以看到，①HandlerThread 本质还是个 Thread，创建后别忘了调用 `start()`。

②在 `run()` 方法中创建了 Looper，调用 `onLooperPrepared` 后开启了循环

③我们要做的就是在子类中重写 `onLooperPrepared`，做一些初始化工作

④在创建 HandlerThread 时可以指定优先级，注意这里的参数是 `Process.XXX` 而不是 `Thread.XXX`

###7.9 设置优先级

```
Process.setThreadPriority(int) 
A Linux priority level, from -20 for highest scheduling priority to 19 for lowest scheduling priority. Linux优先级，从最高调度优先级的20到最低调度优先级的19。
```

可选的值如下 

```
public static final int THREAD_PRIORITY_DEFAULT = 0;
public static final int THREAD_PRIORITY_LOWEST = 19;
public static final int THREAD_PRIORITY_BACKGROUND = 10;
public static final int THREAD_PRIORITY_FOREGROUND = -2;
public static final int THREAD_PRIORITY_DISPLAY = -4;
public static final int THREAD_PRIORITY_URGENT_DISPLAY = -8;
public static final int THREAD_PRIORITY_AUDIO = -16;
```

### 7.10 HandlerThread 的使用场景

我们知道，HandlerThread 所做的就是在新开的子线程中创建了 Looper，那它的使用场景就是 Thread + Looper 使用场景的结合，即：**在子线程中执行耗时的、可能有多个任务的操作**。

比如说多个网络请求操作，或者多文件 I/O 等等。

使用 HandlerThread 的典型例子就是 `IntentService`







## 参考资料

[Android 异步消息处理机制](https://blog.csdn.net/lmj623565791/article/details/38377229)

[HandlerThread 使用场景及源码解析](https://blog.csdn.net/u011240877/article/details/72905631)

[ Android多线程：一步步带你源码解析HandlerThread ](https://blog.csdn.net/carson_ho/article/details/52693418)