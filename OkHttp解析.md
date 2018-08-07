# OkHttp解析

### 1、OkHttp框架的整体设计思路解析

#### 1.1整体架构

```
    1.Interface：接口层，接收网络访问的请求

    2.Protocol：协议层，处理协议逻辑

    3.Connection：连接层，管理网络连接，发送新的请求，接收服务器访问

    4.Cache：缓存层，管理本地缓存

    5.InterceptChain：拦截器层，拦截网络访问，插入拦截逻辑
```

##### 1.1.1  Interface——接口层（Call、OKHttpClient、Dispatcher）

       接口层接收用户的网络请求（同步、异步），发起实际的网络访问，这里也是我们最常写的代码部分。 

​        **OKHttpClient**，我们在这里对它进行各种设置，实现各种不同形式的网络请求，每个OKHttpClient内部都维护了属于自己的任务队列，连接池，Cache，拦截器等。所以在使用OkHttp作为网络框架时应该全局共享一个OkHttpClient实例。 

### 2、OkHttp使用方法简介

#### 2.1 OkHttp同步方法总结

同步需要注意：发送请求后，就会进入阻塞状态，知道收到响应

#####2.1.1 创建OkHttpClient和Request对象

```
 OkHttpClient client = new OkHttpClient.Builder().readTimeout(5, TimeUnit.SECONDS).build();
```



##### 2.1.2 将Request封装成Call对象

```
 //request 代表请求报文的创建，包括 url,请求头，method等等
        Request request = new Request.Builder()
                .url("http://www.baidu.com")
                .get()
                .build();
        // Call ：可以代表一个事件的Request 的请求，可以当作是request 与response中间的桥梁
        Call call = client.newCall(request);
```



##### 2.1.3 调用Call 的execute()发送同步请求

```
   //同步方法 和异步方法，前三步是一样的。response 响应报文的信息
            Response response = call.execute();
            Log.e("TAG", "synRequest: "+response.body().string() );
```

##### 2.1.4 流程图

![](D:\AndroidFile\Photo\OkHttp\okhttp_流程图01.png)

#### 2.2 OkHttp异步方法总结



##### 2.2.1 调用Call的enqueue方法进行异步请求

```
     call.enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {

            }

            @Override
            public void onResponse(Call call, Response response) throws IOException {

            }
        });

```

##### 2.2.2 需要注意

onResponse 和onFailure 都是在工作线程中执行的，也就是子线程中执行的

#### 2.3 异步和同步的区别

2.3.1：发起请求的方法调用不同

2.3.2：同步和异步是否阻塞线程（同步会阻塞，异步不会，并且，异步会创建新的子线程(AsyncCall)来执行相应的操作）







### 3、OkHttp异步/同步流程和源码分析

####3.1 源码分析

#####3.1.1 OkHttpClient.Builder()的源码

```
    OkHttpClient client = new OkHttpClient.Builder().build();
 public Builder() {
      dispatcher = new Dispatcher();//是OkHttp的一个分发器
      protocols = DEFAULT_PROTOCOLS;
      connectionSpecs = DEFAULT_CONNECTION_SPECS;
      eventListenerFactory = EventListener.factory(EventListener.NONE);
      proxySelector = ProxySelector.getDefault();
      cookieJar = CookieJar.NO_COOKIES;
      socketFactory = SocketFactory.getDefault();
      hostnameVerifier = OkHostnameVerifier.INSTANCE;
      certificatePinner = CertificatePinner.DEFAULT;
      proxyAuthenticator = Authenticator.NONE;
      authenticator = Authenticator.NONE;
      connectionPool = new ConnectionPool();//连接池
      dns = Dns.SYSTEM;
      followSslRedirects = true;
      followRedirects = true;
      retryOnConnectionFailure = true;
      connectTimeout = 10_000;
      readTimeout = 10_000;//读取时间
      writeTimeout = 10_000;
      pingInterval = 0;
    }
```

#####3.1.2 Request.Builder()的源码

```
  Request request = new Request.Builder().build();
 public static class Builder {
    HttpUrl url;
    String method;
    Headers.Builder headers;
    RequestBody body;

    /** A mutable map of tags, or an immutable empty map if we don't have any. */
    Map<Class<?>, Object> tags = Collections.emptyMap();

    public Builder() {
      this.method = "GET";
      this.headers = new Headers.Builder();
    }
```

##### 3.1.3  execute()的源码 

```
 client.newCall(request).execute();
@Override public Response execute() throws IOException {
    synchronized (this) {
    //executed ：标志位，判断是否为ture，表示call是否执行过（同一个http请求，只能执行一次）
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    //捕捉一些http请求时的异常堆栈信息，和主流程没有什么关系，知道这个就行
    captureCallStackTrace();
    //开启了监听事件，每当我们的Call调用了execute(),enqueue()方法时，就会开启这个listener
    eventListener.callStart(this);
    try {
    //主要是将call添加到同步请求队列中
      client.dispatcher().executed(this);
      //这个是拦截器链的方法，在这里依次调用连接器进行相应的操作
      Response result = getResponseWithInterceptorChain();
      if (result == null) throw new IOException("Canceled");
      return result;
    } catch (IOException e) {
      eventListener.callFailed(this, e);
      throw e;
    } finally {
    //请看3.1.4
      client.dispatcher().finished(this);
    }
  }
```

#####3.1.4 dispatcher.finished()的源码分析 

```


void finished(RealCall call) {
    finished(runningSyncCalls, call, false);
  }

/*主要做了三件事情 
* 1、把请求从当前执行的请求队列中删除
* 2、(异步时的操作)调用了promoteCalls()，作用是调整我们整个异步任务队列，（因为我们的队列是非线程安全
*的）主要是线程安全的考虑
* 3、重新计算正在执行的请求的数量
*/

private <T> void finished(Deque<T> calls, T call, boolean promoteCalls) {
    int runningCallsCount;
    Runnable idleCallback;
    synchronized (this) {
    //从当前队列中，移除这个同步请求，不能移除就爆出异常（注意：是同步请求）
      if (!calls.remove(call)) throw new AssertionError("Call wasn't in-flight!");
      // client.dispatcher().finished(this)，同步的时候，传入的promoteCalls=false，所以这个方法不
      //会执行，异步的时候，会执行这个方法
      if (promoteCalls) promoteCalls();
      //
      runningCallsCount = runningCallsCount();
      idleCallback = this.idleCallback;
    }
    //目前正在执行的请求数为0的时候，表示整个dispatcher分发器中没有可执行的请求，
    if (runningCallsCount == 0 && idleCallback != null) {
      idleCallback.run();
    }
  }
  
  //计算目前正在执行的请求数：返回的是异步执行请求和同步执行请求的总数
   public synchronized int runningCallsCount() {
    return runningAsyncCalls.size() + runningSyncCalls.size();
  }
```

##### 3.1.5  dispatcher().enqueue 异步请求源码分析

```
/*AsyncCall 就是runnable
*Callback responseCallback:请求结束之后，用于方法的回调的
*/
client.dispatcher().enqueue(new AsyncCall(responseCallback));

synchronized void enqueue(AsyncCall call) {
/*
*runningAsyncCalls.size():异步请求数
*maxRequests:最大的请求数
*runningCallsForHost：每一个主机正在运行的请求数
*maxRequestsPerHost：我们设定好的最大主机请求值
*runningAsyncCalls：正在执行的异步请求队列(作用：判断并发请求的数量)
*/
//当你发起一个请求的时候，它首先会判断，当前的请求数，是否小于最大的请求数
    if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
    //将call添加到正在执行的异步请求队列中
      runningAsyncCalls.add(call);
      //通过线程池去执行这个call
      executorService().execute(call);
    } else {
    //加载到等待队列当中
      readyAsyncCalls.add(call);
    }
  }
  
  //线程池
   public synchronized ExecutorService executorService() {
    if (executorService == null) {
      executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
          new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
    }
    return executorService;
  }
  
```

##### 3.1.7 AsyncCall 的 execute()异步源码分析

```
final class AsyncCall extends NamedRunnable {
    private final Callback responseCallback;

    @Override protected void execute() {
      boolean signalledCallback = false;
      try {
      //拦截链(重点)，这个方法就是构建了一个拦截器的链，每一个拦截器都有不同的作用
        Response response = getResponseWithInterceptorChain();
        /*判断  这个拦截器是否取消了
        *retryAndFollowUpInterceptor:重定向拦截器和重试拦截器
        */
        if (retryAndFollowUpInterceptor.isCanceled()) {
          signalledCallback = true;
          responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
        } else {
          signalledCallback = true;
          //真正的网络操作时在onResponse中进行的
          responseCallback.onResponse(RealCall.this, response);
        }
      } catch (IOException e) { //主要做网络请求失败的操作
        if (signalledCallback) {
          // Do not signal the callback twice!
          Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
        } else {
          eventListener.callFailed(RealCall.this, e);
          responseCallback.onFailure(RealCall.this, e);
        }
      } finally {
        client.dispatcher().finished(this);
      }
    }
  }
```







#####3.1.8 Enqueue方法总结

```
1：判断当前的call，（也就是http请求）是否只被执行了一次，如果不是的话，就会抛出异常
2：封装成了一个AsyncCall对象，
3：client.dispatcher().enqueue()
```

#### 3.2 Dispatcher 总体分析(任务调度核心)

##### 3.2.1 okhttp如何实现同步异步的请求？

```
发送的同步/异步请求都会在dispatcher管理器状态
```

##### 3.2.2 到底什么是dispatcher ?

```
dispatcher 的作用为维护请求的状态，并维护一个线程池，用于执行请求
```

##### 3.2.3异步请求为什么需要两个队列 ?

```
一个是readyAsyncCalls，runningAsyncCalls，可以理解为生产者，消费者
1、dispatcher :生产者
2、ExecutorService :消费者

```

##### 3.2.4 readyAsyncCalls 队列中的线程在什么时候才会被执行呢？ 

```
问题背景：Call执行完肯定需要在runningAsyncCalls队列中移除这个线程，如果不删除的话，后面等待缓存队列就没法添加进来

```



##### 3.2.5 dispatcher 源码分析

```
public final class Dispatcher {
  private int maxRequests = 64;
  private int maxRequestsPerHost = 5;
  private @Nullable Runnable idleCallback;
  /*
  * dispatcher 正是通过了这个线程池来维护了整个异步的请求，还有进行了高效的网络操作
  *
  */
  private @Nullable ExecutorService executorService;
  /**
  *readyAsyncCalls；就绪等待缓存异步请求的队列;当我们的异步请求不满足执行条件的   
  *时候，就添加到就绪异步请求队列当中，来进行缓存，如果当条件再满足之后，就把这里面
  *的队列放在正在执行的runningAsyncCalls 队列中 
  *
  */
  private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();
//正在执行的异步请求队列(作用：判断并发请求的数量)
  private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();
//正在执行的同步请求的队列
  private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();

  public Dispatcher(ExecutorService executorService) {
    this.executorService = executorService;
  }

  public Dispatcher() {
  }

  public synchronized ExecutorService executorService() {
    if (executorService == null) {
    /*第一个参数表示核心线程池的数量，这里设定为0。
    *为什么设定为0？如果空闲一段时间之后，我们就会将所有的线程全部销毁
    */
      executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
          new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
    }
    return executorService;
  }


  public synchronized void setMaxRequests(int maxRequests) {
    if (maxRequests < 1) {
      throw new IllegalArgumentException("max < 1: " + maxRequests);
    }
    this.maxRequests = maxRequests;
    promoteCalls();
  }

  public synchronized int getMaxRequests() {
    return maxRequests;
  }

 
  public synchronized void setMaxRequestsPerHost(int maxRequestsPerHost) {
    if (maxRequestsPerHost < 1) {
      throw new IllegalArgumentException("max < 1: " + maxRequestsPerHost);
    }
    this.maxRequestsPerHost = maxRequestsPerHost;
    promoteCalls();
  }

  public synchronized int getMaxRequestsPerHost() {
    return maxRequestsPerHost;
  }


  public synchronized void setIdleCallback(@Nullable Runnable idleCallback) {
    this.idleCallback = idleCallback;
  }

  synchronized void enqueue(AsyncCall call) {
    if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
      runningAsyncCalls.add(call);
      executorService().execute(call);
    } else {
      readyAsyncCalls.add(call);
    }
  }

  public synchronized void cancelAll() {
    for (AsyncCall call : readyAsyncCalls) {
      call.get().cancel();
    }

    for (AsyncCall call : runningAsyncCalls) {
      call.get().cancel();
    }

    for (RealCall call : runningSyncCalls) {
      call.cancel();
    }
  }

/**
*
*/
  private void promoteCalls() {
    if (runningAsyncCalls.size() >= maxRequests) return; // Already running max capacity.
    if (readyAsyncCalls.isEmpty()) return; // No ready calls to promote.

    for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
      AsyncCall call = i.next();

      if (runningCallsForHost(call) < maxRequestsPerHost) {
        i.remove();
        runningAsyncCalls.add(call);
        executorService().execute(call);
      }

      if (runningAsyncCalls.size() >= maxRequests) return; // Reached max capacity.
    }
  }

  /** Returns the number of running calls that share a host with {@code call}. */
  private int runningCallsForHost(AsyncCall call) {
    int result = 0;
    for (AsyncCall c : runningAsyncCalls) {
      if (c.get().forWebSocket) continue;
      if (c.host().equals(call.host())) result++;
    }
    return result;
  }

  /** Used by {@code Call#execute} to signal it is in-flight. */
  synchronized void executed(RealCall call) {
    runningSyncCalls.add(call);
  }

  /** Used by {@code AsyncCall#run} to signal completion. */
  void finished(AsyncCall call) {
    finished(runningAsyncCalls, call, true);
  }

  /** Used by {@code Call#execute} to signal completion. */
  void finished(RealCall call) {
    finished(runningSyncCalls, call, false);
  }

  private <T> void finished(Deque<T> calls, T call, boolean promoteCalls) {
    int runningCallsCount;
    Runnable idleCallback;
    synchronized (this) {
      if (!calls.remove(call)) throw new AssertionError("Call wasn't in-flight!");
      /*promoteCalls():
      *
      */
      if (promoteCalls) promoteCalls();
      runningCallsCount = runningCallsCount();
      idleCallback = this.idleCallback;
    }

    if (runningCallsCount == 0 && idleCallback != null) {
      idleCallback.run();
    }
  }

  /** Returns a snapshot of the calls currently awaiting execution. */
  public synchronized List<Call> queuedCalls() {
    List<Call> result = new ArrayList<>();
    for (AsyncCall asyncCall : readyAsyncCalls) {
      result.add(asyncCall.get());
    }
    return Collections.unmodifiableList(result);
  }

  /** Returns a snapshot of the calls currently being executed. */
  public synchronized List<Call> runningCalls() {
    List<Call> result = new ArrayList<>();
    result.addAll(runningSyncCalls);
    for (AsyncCall asyncCall : runningAsyncCalls) {
      result.add(asyncCall.get());
    }
    return Collections.unmodifiableList(result);
  }

  public synchronized int queuedCallsCount() {
    return readyAsyncCalls.size();
  }

  public synchronized int runningCallsCount() {
    return runningAsyncCalls.size() + runningSyncCalls.size();
  }
}

```





### 4、OkHttp核心类OkhttpClient/call 解析



### 5、Okhttp 连接池ConnectionPool原理解析



### 6、Okhttp调度器Dispatcher源码分析



### 7、OkHttp任务调度和调度模型分析 





### 8、OkHttp拦截器Interceptor源码分析 





### 9、OkHttp缓存策略源码分析 



### 10、OkHttp链接复用原理分析 



### 11、Okhttp网络底层详解（Address/StreamAllocation/httpCodec） 





























































