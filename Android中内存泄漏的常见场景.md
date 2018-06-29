## Android中内存泄漏的常见场景

### 前言

​        内存管理的目的就是让我们在开发过程中有效避免我们的应用程序出现内存泄露的问题。内存泄露相信大家都不陌生，我们可以这样理解：「没有用的对象无法回收的现象就是内存泄露」。

如果程序发生了内存泄露，则会带来以下这些问题

- 应用可用的内存减少，增加了堆内存的压力
- 降低了应用的性能，比如会触发更频繁的 GC
- 严重的时候可能会导致内存溢出错误，即 OOM Error

OOM 发生在，当我们尝试进行创建对象，但是堆内存无法通过 GC 释放足够的空间，堆内存也无法再继续增长，从而完成对象创建请求的时候，OOM 发生很有可能是内存泄露导致的，但并非所有的 OOM 都是由内存泄露引起的，内存泄露也并不一定引起 OOM。

### 一、引起内存泄漏的情况

#### 1.1、内存泄漏与内存溢出

说到**内存泄露**，就不得不提到**内存溢出**，这两个比较容易混淆的概念，我们来分析一下。

- **内存泄露**：**程序在向系统申请分配内存空间后(new)，在使用完毕后未释放。**结果导致一直占据该内存单元，我们和程序都无法再使用该内存单元，直到程序结束，这是内存泄露。
- **内存溢出**：**程序向系统申请的内存空间超出了系统能给的。**比如内存只能分配一个int类型，我却要塞给他一个long类型，系统就出现oom。又比如一车最多能坐5个人，你却非要塞下10个，车就挤爆了。

大量的内存泄露会导致内存溢出(oom)。

#### 1.2、引起内存泄漏的情况

- 对于使用了BraodcastReceiver，ContentObserver，File，游标 Cursor，Stream，Bitmap等资源的使用，应该在Activity销毁时及时关闭或者注销。
- 静态内部类持有外部成员变量（或context）:可以使用弱引用或使用ApplicationContext。
- 内部类持有外部类引用,异步任务中，持有外部成员变量。
- 集合中没用的对象没有及时remove。
- 不用的对象及时释放，如使用完Bitmap后掉用recycle（），再赋null。
- handler引起的内存泄漏，MessageQueue里的消息如果在activity销毁时没有处理完，就会引起内存的泄漏，可以使用弱引用解决。
- 设置过的监听不用时，及时移除。如在Destroy时及时remove。尤其以addListener开头的，在Destroy中都需要remove。
- activity泄漏可以使用LeakCanary。

### 二、常见场景&解决方案

#### 2.1、单例造成的内存泄漏

​        单例模式是非常常用的设计模式，使用单例模式的类，只会产生一个对象，这个对象看起来像是一直占用着内存，但这并不意味着就是浪费了内存，内存本来就是拿来装东西的，只要这个对象一直都被高效的利用就不能叫做泄露。

​        但是过多的单例会让内存占用过多，而且单例模式由于其 **静态特性**，其生命周期 = 应用程序的生命周期，不正确地使用单例模式也会造成内存泄露。

 举个例子： 

```
public class SingleInstanceTest {

    private static SingleInstanceTest sInstance;
    private Context mContext;

    private SingleInstanceTest(Context context){
        this.mContext = context;
    }

    public static SingleInstanceTest newInstance(Context context){
        if(sInstance == null){
            sInstance = new SingleInstanceTest(context);
        }
        return sInstance;
    }
}
```

​       上面是一个比较简单的单例模式用法，需要外部传入一个 Context 来获取该类的实例，如果此时传入的 Context 是 Activity 的话，此时单例就有持有该 Activity 的强引用（直到整个应用生命周期结束）。这样的话，即使该 Activity 退出，该 Activity 的内存也不会被回收，这样就造成了内存泄露，特别是一些比较大的 Activity，甚至还会导致 OOM（Out Of Memory）。

**解决方法：**单例模式引用的对象的生命周期 = 应用生命周期

```
public class SingleInstanceTest {

    private static SingleInstanceTest sInstance;
    private Context mContext;

    private SingleInstanceTest(Context context){
        this.mContext = context.getApplicationContext();
    }

    public static SingleInstanceTest newInstance(Context context){
        if(sInstance == null){
            sInstance = new SingleInstanceTest(context);
        }
        return sInstance;
    }
}
```

​        可以看到在 SingleInstanceTest 的构造函数中，将 context.getApplicationContext() 赋值给 mContext，此时单例引用的对象是 Application，而 Application 的生命周期本来就跟应用程序是一样的，也就不存在内存泄露。

​        这里再拓展一点，很多时候我们在需要用到 Activity 或者 Context 的地方，会直接将 Activity 的实例作为参数传给对应的类，就像这样：

 

```
public class Sample {
    
    private Context mContext;
    
    public Sample(Context context){
        this.mContext = context;
    }

    public Context getContext() {
        return mContext;
    }
}

// 外部调用
Sample sample = new Sample(MainActivity.this);
```

​        这种情况如果不注意的话，很容易就会造成内存泄露，比较好的写法是使用弱引用（WeakReference）来进行改进。 

 

```
public class Sample {

    private WeakReference<Context> mWeakReference;

    public Sample(Context context){
        this.mWeakReference = new WeakReference<>(context);
    }

    public Context getContext() {
        if(mWeakReference.get() != null){
            return mWeakReference.get();
        }
        return null;
    }
}

// 外部调用
Sample sample = new Sample(MainActivity.this);
```

​         被弱引用关联的对象只能存活到下一次垃圾回收之前，也就是说即使 Sample 持有 Activity 的引用，但由于 GC 会帮我们回收相关的引用，被销毁的 Activity 也会被回收内存，这样我们就不用担心会发生内存泄露了。 

#### 2.2、非静态内部类 / 匿名类

我们先来看看非静态内部类（non static inner class）和 静态内部类（static inner class）之间的区别。 

| class 对比                        | static inner class               | non static inner class         |
| --------------------------------- | -------------------------------- | ------------------------------ |
| 与外部 class 引用关系             | 如果没有传入参数，就没有引用关系 | 自动获得强引用                 |
| 被调用时需要外部实例              | 不需要                           | 需要                           |
| 能否调用外部 class 中的变量和方法 | 不能                             | 能                             |
| 生命周期                          | 自主的生命周期                   | 依赖于外部类，甚至比外部类更长 |

​        可以看到非静态内部类自动获得外部类的强引用，而且它的生命周期甚至比外部类更长，这便埋下了内存泄露的隐患。如果一个 Activity 的非静态内部类的生命周期比 Activity 更长，那么 Activity 的内存便无法被回收，也就是发生了内存泄露，而且还有可能发生难以预防的空指针问题。

举个例子：

 

```
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        new MyAscnyTask().execute();
    }

    class MyAscnyTask extends AsyncTask<Void, Integer, String>{
        @Override
        protected String doInBackground(Void... params) {
            try {
                Thread.sleep(5000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return "";
        }
    }
}
```

 可以看到我们在 Activity 中继承 AsyncTask 自定义了一个非静态内部类，在 doInbackground() 方法中做了耗时的操作，然后在 onCreate() 中启动 MyAsyncTask。如果在耗时操作结束之前，Activity 被销毁了，这时候因为 MyAsyncTask 持有 Activity 的强引用，便会导致 Activity 的内存无法被回收，这时候便会产生内存泄露。

**解决方法：**将 MyAsyncTask 变成静态内部类

```
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        new MyAscnyTask().execute();
    }

    static class MyAscnyTask extends AsyncTask<Void, Integer, String>{
        @Override
        protected String doInBackground(Void... params) {
            try {
                Thread.sleep(50000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return "";
        }
    }
}
```

**这时候 MyAsyncTask 不再持有 Activity 的强引用**，即使 AsyncTask 的耗时操作还在继续，Activity 的内存也能顺利地被回收。

匿名类和非静态内部类最大的共同点就是 **都持有外部类的引用**，因此，匿名类造成内存泄露的原因也跟静态内部类基本是一样的，下面举个几个比较常见的例子：

```
public class MainActivity extends AppCompatActivity {

    private Handler mHandler = new Handler(){
        @Override
        public void handleMessage(Message msg) {
            super.handleMessage(msg);
        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        // ① 匿名线程持有 Activity 的引用，进行耗时操作
        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(50000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }).start();

        // ② 使用匿名 Handler 发送耗时消息
        Message message = Message.obtain();
        mHandler.sendMessageDelayed(message, 60000);
    }
```

 上面举出了两个比较常见的例子

- new 出一个匿名的 Thread，进行耗时的操作，如果 MainActivity 被销毁而 Thread 中的耗时操作没有结束的话，便会产生内存泄露
- new 出一个匿名的 Handler，这里我采用了 sendMessageDelayed() 方法来发送消息，这时如果 MainActivity 被销毁，而 Handler 里面的消息还没发送完毕的话，Activity 的内存也不会被回收

 **解决方法：**

- 继承 Thread 实现静态内部类
- 继承 Handler 实现静态内部类，以及在 Activity 的 onDestroy() 方法中，移除所有的消息 mHandler.removeCallbacksAndMessages(null);

 #### 2.3、集合类

集合类添加元素后，仍引用着集合元素对象，导致该集合中的元素对象无法被回收，从而导致内存泄露，举个例子： 

```
static List<Object> objectList = new ArrayList<>();
   for (int i = 0; i < 10; i++) {
       Object obj = new Object();
       objectList.add(obj);
       obj = null;
    }
```

 在这个例子中，循环多次将 new 出来的对象放入一个静态的集合中，因为静态变量的生命周期和应用程序一致，而且他们所引用的对象 Object 也不能释放，这样便造成了内存泄露。

**解决方法：**在集合元素使用之后从集合中删除，等所有元素都使用完之后，将集合置空。

 

```
 objectList.clear();
 objectList = null;
```



#### 2.4、静态Activity和View

静态变量Activity和View会导致内存泄漏，在下面这段代码中对Activity的Context和TextView设置为静态对象，从而产生内存泄漏。 

```
public class MainActivity extends AppCompatActivity {

    private static Context context;
    private static TextView textView;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        context = this;
        textView = new TextView(this);
    }
}
```

因为，context，textView实例的生命周期与应用的生命周期一样，而他们都持有当前Activity的（MainActivity ）引用，一旦MainActivity 销毁，而他的引用一直被持有，就不会被回收。所以，内存泄漏就产出了。 

#### 2.5、线程造成的内存泄漏

对于线程造成的内存泄漏，也是平时比较常见的，如下这两个示例可能每个人都这样写过： 

```
//——————test1
        new AsyncTask<Void, Void, Void>() {
            @Override
            protected Void doInBackground(Void... params) {
                SystemClock.sleep(10000);
                return null;
            }
        }.execute();
//——————test2
        new Thread(new Runnable() {
            @Override
            public void run() {
                SystemClock.sleep(10000);
            }
        }).start();
```

上面的异步任务和Runnable都是一个匿名内部类，因此它们对当前Activity都有一个隐式引用。如果Activity在销毁之前，任务还未完成， 那么将导致Activity的内存资源无法回收，造成内存泄漏。正确的做法还是使用静态内部类的方式，如下： 

```
static class MyAsyncTask extends AsyncTask<Void, Void, Void> {
        private WeakReference<Context> weakReference;

        public MyAsyncTask(Context context) {
            weakReference = new WeakReference<>(context);
        }

        @Override
        protected Void doInBackground(Void... params) {
            SystemClock.sleep(10000);
            return null;
        }

        @Override
        protected void onPostExecute(Void aVoid) {
            super.onPostExecute(aVoid);
            MainActivity activity = (MainActivity) weakReference.get();
            if (activity != null) {
                //...
            }
        }
    }
    
    // test2----
    static class MyRunnable implements Runnable{
        @Override
        public void run() {
            SystemClock.sleep(10000);
        }
    }
//——————
    new Thread(new MyRunnable()).start();
    new MyAsyncTask(this).execute();
```

这样就避免了Activity的内存资源泄漏，当然在Activity销毁时候也应该取消相应的任务AsyncTask::cancel()，避免任务在后台执行浪费资源。

如果你还是不太清楚，再举个例子： 

在下面这段代码中存在一个非静态的匿名类对象Thread，会隐式持有一个外部类的引用MainActivity 。同理，若是这个Thread作为MainActivity的内部类而不是匿名内部类，他同样会持有外部类的引用。 

```
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.main);
        leakFun();
    }

    private void leakFun(){
        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    Thread.sleep(10*1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
    }
}
```

在线程休眠的这10s内，会一直隐式持有外部类的引用MainActivity，如果在10s之前，销毁MainActivity，就会报内存泄漏。同理，若是这个Thread作为MainActivity的内部类而不是匿名内部类，也会内存泄漏。总而言之：如果Activity在销毁之前，任务还未完成， 那么将导致Activity的内存资源无法回收，造成内存泄漏。  解决办法：在这里只需要将为Thread匿名类定义成静态的内部类即可（静态的内部类不会持有外部类的一个隐式引用）。或保证在Activity在销毁之前，完成任务！ 

#### 2.6、非静态内部类创建静态实例造成的内存泄漏

有的时候我们可能会在启动频繁的Activity中，为了避免重复创建相同的数据资源，会出现这种写法： 

```
public class MainActivity extends AppCompatActivity {
    private static TestResource mResource = null;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        if(mResource == null){
            mResource = new TestResource();
        }
        //...
    }
    class TestResource {
        //...
    }
}
```

这样就在Activity内部创建了一个非静态内部类的单例，每次启动Activity时都会使用该单例的数据，这样虽然避免了资源的重复创建，不过这种写法却会造成内存泄漏，**因为非静态内部类默认会持有外部类的引用，而又使用了该非静态内部类创建了一个静态的实例，该实例的生命周期和应用的一样长**，这就导致了该静态实例一直会持有该Activity的引用，导致Activity的内存资源不能正常回收。正确的做法为：

将该内部类设为静态内部类或将该内部类抽取出来封装成一个单例，如果需要使用Context，请使用ApplicationContext 。

#### 2.7、Handler造成的内存泄漏

Handler的使用造成的内存泄漏问题应该说最为常见了，平时在处理网络任务或者封装一些请求回调等api都应该会借助Handler来处理，对于Handler的使用代码编写一不规范即有可能造成内存泄漏，如下示例： 

```
public class MainActivity extends AppCompatActivity {
    private Handler mHandler = new Handler() {
        @Override
        public void handleMessage(Message msg) {
            //...
        }
    };
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        loadData();
    }
   //loadData()方法是在子线程中，执行
    private void loadData(){
        //...request
        Message message = Message.obtain();
        mHandler.sendMessage(message);
    }
}
```

这种创建Handler的方式会造成内存泄漏，由于mHandler是Handler的非静态匿名内部类的实例，所以它持有外部类Activity的引用，我们知道消息队列是在一个Looper线程中不断轮询处理消息，*那么当这个Activity退出时，消息队列中还有未处理的消息或者正在处理消息（例如上面的例子中，子线程中的耗时任务，还没有执行完毕，activity就退出，销毁了。）*，而消息队列中的Message持有mHandler实例的引用，mHandler又持有Activity的引用，所以导致该Activity的内存资源无法及时回收，引发内存泄漏。哪有什么处理方案呢？  处理方案一：： 

```
public class MainActivity extends AppCompatActivity {
    private MyHandler mHandler = new MyHandler(this);
    private TextView mTextView ;
    private static class MyHandler extends Handler {
        private WeakReference<Context> reference;
        public MyHandler(Context context) {
            reference = new WeakReference<>(context);
        }
        @Override
        public void handleMessage(Message msg) {
            MainActivity activity = (MainActivity) reference.get();
            if(activity != null){
                activity.mTextView.setText("");
            }
        }
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        mTextView = (TextView)findViewById(R.id.textview);
        loadData();
    }
   //loadData()方法是在子线程中，执行
    private void loadData() {
        //...request
        Message message = Message.obtain();
        mHandler.sendMessage(message);
    }
}
```

创建一个静态Handler内部类，然后对Handler持有的对象使用弱引用，这样在回收时也可以回收Handler持有的对象，这样虽然避免了Activity泄漏，如果你的Handler被delay（延时了），在Activity的Destroy时或者Stop时应该移除消息队列中的消息： 

```
public class MainActivity extends AppCompatActivity {
    private MyHandler mHandler = new MyHandler(this);
    private TextView mTextView ;
    private static class MyHandler extends Handler {
        private WeakReference<Context> reference;
        public MyHandler(Context context) {
            reference = new WeakReference<>(context);
        }
        @Override
        public void handleMessage(Message msg) {
            MainActivity activity = (MainActivity) reference.get();
            if(activity != null){
                activity.mTextView.setText("");
            }
        }
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        mTextView = (TextView)findViewById(R.id.textview);
        loadData();
    }
   //loadData()方法是在子线程中，执行
    private void loadData() {
        //...request
        Message message = Message.obtain();
        mHandler.sendMessage(message);
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        //如果你的Handler被delay（延时了）,可以做如下的处理
        mHandler.removeCallbacksAndMessages(null);
    }
}
```

使用mHandler.removeCallbacksAndMessages(null);是移除消息队列中所有消息和所有的Runnable。当然也可以使用mHandler.removeCallbacks();或mHandler.removeMessages()；来移除指定的Runnable和Message。 

**解决方案总结**

1： 通过程序逻辑来进行保护

- 在关闭Activity的时候停掉你的后台线程。线程停掉了，就相当于切断了Handler和外部连接的线，Activity自然会在合适的时候被回收。
- 如果你的Handler是被delay的Message持有了引用，那么使用相应的Handler的removeCallbacks()方法，把消息对象从消息队列移除就行了。

2： 将Handler声明为静态类

- 在Java 中，非静态的内部类和匿名内部类都会隐式地持有其外部类的引用，静态的内部类不会持有外部类的引用。静态类不持有外部类的对象，所以你的Activity可以随意被回收。由于Handler不再持有外部类对象的引用，导致程序不允许你在Handler中操作Activity中的对象了。所以你需要在Handler中增加一个对Activity的弱引用（WeakReference）。

#### 2.8、动画

在属性动画中有一类无限循环动画，如果在Activity中播放这类动画并且在onDestroy中没有去停止动画，那么这个动画将会一直播放下去，这时候Activity会被View所持有，从而导致Activity无法被释放。解决此类问题则是需要早Activity中onDestroy去调用objectAnimator.cancel()来停止动画。 

```
public class LeakActivity extends AppCompatActivity {

    private TextView textView;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_leak);
        textView = (TextView)findViewById(R.id.text_view);
        ObjectAnimator objectAnimator = ObjectAnimator.ofFloat(textView,"rotation",0,360);
        objectAnimator.setRepeatCount(ValueAnimator.INFINITE);
        objectAnimator.start();
    }
}
```

#### 2.9、第三方库使用不当

对于EventBus，RxJava等一些第三开源框架的使用，若是在Activity销毁之前没有进行解除订阅将会导致内存泄漏。 

#### 2.10、资源未关闭造成的内存泄漏

对于使用了BraodcastReceiver，ContentObserver，File，Cursor，Stream，Bitmap等资源的使用，应该在Activity销毁时及时关闭或者注销，否则这些资源将不会被回收，造成内存泄漏。 

 #### 2.x、其他情况

除了上述 3 种常见情况外，还有其他的一些情况

- 1、需要手动关闭的对象没有关闭
  - 网络、文件等流忘记关闭
  - 手动注册广播时，退出时忘记 unregisterReceiver()
  - Service 执行完后忘记 stopSelf()
  - EventBus 等观察者模式的框架忘记手动解除注册
- 2、static 关键字修饰的成员变量
- 3、ListView 的 Item 泄露

 ### 三、利用工具进行内存泄露的排查

除了必须了解常见的内存泄露场景以及相应的解决方法之外，掌握一些好用的工具，能让我们更有效率地解决内存泄露的问题。

####3.1、Android Lint

Lint 是 Android Studio 提供的 **代码扫描分析工具**，它可以帮助我们发现代码机构 / 质量问题，同时提供一些解决方案，检测内存泄露当然也不在话下，使用也是非常的简单，可以参考下这篇文章：

 [Android 性能优化：使用 Lint 优化代码、去除多余资源](https://blog.csdn.net/u011240877/article/details/54141714)

 #### 3.2、leakcanary

LeakCanary 是 Square 公司开源的「Android 和 Java 的内存泄漏检测库」，Square 出品，必属精品，功能很强大，使用也很简单。建议直接看 Github 上的说明：[leakcanary](https://link.jianshu.com?t=https%3A%2F%2Fgithub.com%2Fsquare%2Fleakcanary)，也可以参考这篇文章：[Android内存优化（六）LeakCanary使用详解](https://link.jianshu.com?t=http%3A%2F%2Fblog.csdn.net%2Fitachi85%2Farticle%2Fdetails%2F77826112)

 

###参考资料：

- [Android泄漏及解决方案](https://www.jianshu.com/p/65f914e6a2f8)
- [Android 性能优化：手把手带你全面了解内存泄露](https://link.jianshu.com/?t=https%3A%2F%2Fjuejin.im%2Fpost%2F5a652d31518825734108080d)
- [系统剖析Android中的内存泄漏](https://link.jianshu.com/?t=https%3A%2F%2Fdroidyue.com%2Fblog%2F2016%2F11%2F23%2Fmemory-leaks-in-android%2F)
- [Android 内存泄露分析](https://link.jianshu.com/?t=https%3A%2F%2Fjuejin.im%2Fentry%2F58805842b123db0061cdb82b)