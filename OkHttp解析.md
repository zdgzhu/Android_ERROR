

# OkHttp解析

### 一、OkHttp框架的整体设计思路解析

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

### 二、OkHttp使用方法简介

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







### 三、OkHttp异步/同步流程和源码分析

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

###四、Okhttp调度器Dispatcher源码分析

####4.1 okhttp如何实现同步异步的请求？

```
发送的同步/异步请求都会在dispatcher管理器状态
```

####4.2 到底什么是dispatcher ?

```
dispatcher 的作用为维护请求的状态，并维护一个线程池，用于执行请求
```

####4.3 异步请求为什么需要两个队列 ?

```
一个是readyAsyncCalls，runningAsyncCalls，可以理解为生产者，消费者
1、dispatcher :生产者
2、ExecutorService :消费者

```

####4.4 readyAsyncCalls 队列中的线程在什么时候才会被执行呢？

```
问题背景：Call执行完肯定需要在runningAsyncCalls队列中移除这个线程，如果不删除的话，后面等待缓存队列就没法添加进来

```

####4.5 dispatcher 源码分析

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

###五、OkHttp拦截器Interceptor源码分析

####5.1 拦截器定义

```
1、简单回顾同步/异步
(1)同步请求就是执行请求的操作是阻塞式，直到HTTP响应返回
(2)异步请求就类似于非阻塞式的请求，它的执行结果一般都是通过接口回调的方式告知调用者
(3)官网：拦截器是OkHttp中提供一种强大机制，它可以实现网络监听，请求以及响应重写，请求失败重试等功能
```

####5.2 getResponseWithInterceptorChain()分析

```
Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    //应用拦截器
    interceptors.addAll(client.interceptors());
    interceptors.add(retryAndFollowUpInterceptor);
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
    //网络拦截器
      interceptors.addAll(client.networkInterceptors());
    }
    interceptors.add(new CallServerInterceptor(forWebSocket));

    Interceptor.Chain chain = new RealInterceptorChain(interceptors, null, null, null, 0,
        originalRequest, this, eventListener, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());

    return chain.proceed(originalRequest);
  }Response getResponseWithInterceptorChain() throws IOException {

```

注意点：

- 方法的返回值的response，这个就是网络请求的目的，得到的数据都封装在Response对象中。
- 拦截器的使用，在方法的第一行中就创建了interceptors集合，然后紧接着放进去很多拦截器对象
- RealInterceptorChain类的proceed方法，getResponseWithInterceptorChain方法的最后创建了RealInterceptorChain对象，并调用proceed方法。Response 对象就有由RealInterceptorChain类的proceed方法返回的。 

##### 5.2.1 proceed()源码解析

```
public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
      RealConnection connection) throws IOException {
    if (index >= interceptors.size()) throw new AssertionError();
   ...
    RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpCodec,
        connection, index + 1, request, call, eventListener, connectTimeout, readTimeout,
        writeTimeout);
    Interceptor interceptor = interceptors.get(index);
    Response response = interceptor.intercept(next);
    ...
    }
```

##### 5.2.2  StreamAllocation 源码解析

```
 public StreamAllocation(ConnectionPool connectionPool, Address address, Call call,
      EventListener eventListener, Object callStackTrace) {
    this.connectionPool = connectionPool;
    this.address = address;
    this.call = call;
    this.eventListener = eventListener;
    this.routeSelector = new RouteSelector(address, routeDatabase(), call, eventListener);
    this.callStackTrace = callStackTrace;
  }
```





####5.3 拦截器流程图

#####5.3.1 六大拦截器运行图

![](D:\AndroidFile\Photo\OkHttp\okhttp_拦截器02.png)



##### 5.3.2 整个网络访问的核心 流程图

![](D:\AndroidFile\Photo\OkHttp\okhttp_拦截器03.png)

####5.4 RetryAndFollowUpInterceptor

用来实现连接失败的重试和重定向;负责两个部分的逻辑：

(1)在网络请求失败后进行重试

(2)当服务器返回当前请求需要进行重定向时直接发起新的请求， 并在条件允许情况下复用当前连接

**源码主要的操作：**

(1)创建StreamAllocation对象

(2)调用RealInterceptorChain.proceed(...)进行网络请求

(3)根据异常结果或者响应结果判断是否要进行重新请求

(4)调用下一个拦截器，对response进行处理，返回给上一个拦截器

####5.5 BridgeInterceptor

```
Response.Builder responseBuilder = networkResponse.newBuilder()
        .request(userRequest);
    //判断服务器是否支持gzip压缩格式,如果支持则交给kio压缩
    if (transparentGzip&& "gzip".equalsIgnoreCase(networkResponse.header("Content-Encoding"))
        && HttpHeaders.hasBody(networkResponse)) {
      GzipSource responseBody = new GzipSource(networkResponse.body().source());
      Headers strippedHeaders = networkResponse.headers().newBuilder()
          .removeAll("Content-Encoding")
          .removeAll("Content-Length")
          .build();
      responseBuilder.headers(strippedHeaders);
      responseBuilder.body(new RealResponseBody(strippedHeaders, Okio.buffer(responseBody)));
    }

    return responseBuilder.build();
```

​     BridgeInterceptor拿到CacheInterceoptor返回的response之后根据请求头的Content-Encoding类型来决定返回的类型，如果是gzip,则将response经过GzipSource 处理后将其Response的body设置为RealResponseBody对象，否则就简单的返回response。  简单流程如下：  

![](D:\AndroidFile\Photo\OkHttp\okhttp_拦截器04.png)

   这个Interceptor做的事情比较简单。可以分为发送请求和收到响应两个阶段来看。在发送请求阶段，BridgeInterceptor补全一些http header，这主要包括`Content-Type`、`Content-Length`、`Transfer-Encoding`、`Host`、`Connection`、`Accept-Encoding`、`User-Agent`，还加载`Cookie`，随后创建新的Request，并交给后续的Interceptor处理，以获取响应。

而在从后续的Interceptor获取响应之后，会首先保存`Cookie`。如果服务器返回的响应的content是以gzip压缩过的，则会先进行解压缩，移除响应中的header `Content-Encoding`和`Content-Length`，构造新的响应并返回；否则直接返回响应。

`CookieJar`来自于`OkHttpClient`，它是OkHttp的`Cookie`管理器，负责`Cookie`的存取：

 

用来修改请求和响应的header信息；主要负责以下几部分内容

(1)设置内容长度，内容编码

(2)设置gzip压缩，并在接收到内容后进行解压，省去了应用层处理数据解压的麻烦

(3)添加cookie

(4)设置其他抱头，如User-Agent,Host,keep-alive等，其中keep-alive是实现多路复用的必要步骤。

**源码的核心步骤**

(1)是负责将用户构建的一个Request请求转化成能够进行网络访问的请求

(2)将这个符合网络请求的Request进行网络请求

(3)将网络请求回来的响应Response转化为用户可用的Response.

####5.6 CacheInterceptor

##### 5.6.1 OkHttp缓存策略源码分析



用来实现响应缓存。比如获取到的 Response 带有 Date，Expires，Last-Modified，Etag 等 header，表示该 Response 可以缓存一定的时间，下次请求就可以不需要发往服务端，直接拿缓存的 .

**主要负责以下几部分内容 ： **

(1)当网络请求有符合要求的Cache时，直接返回Cache管理

(2)当服务器返回内容有改变时，更新当前cache

(3)如果当前cache失效，删除

```
Response intercept(Chain chain) throws IOException {
    //如果配置了缓存：优先从缓存中读取Response
    Response cacheCandidate = cache != null
        ? cache.get(chain.request())
        : null;
    long now = System.currentTimeMillis();
    //缓存策略，该策略通过某种规则来判断缓存是否有效
   CacheStrategy strategy = new CacheStrategy.Factory(now, chain.request(), cacheCandidate).get();
    Request networkRequest = strategy.networkRequest;
    Response cacheResponse = strategy.cacheResponse;
    。。。。
    //如果根据缓存策略strategy禁止使用网络，并且缓存无效，直接返回空的Response
    if (networkRequest == null && cacheResponse == null) {
      return new Response.Builder()
          。。。
          .code(504)
          .message("Unsatisfiable Request (only-if-cached)")
          .body(Util.EMPTY_RESPONSE)//空的body
          。。。
          .build();
    }

    //如果根据缓存策略strategy禁止使用网络，且有缓存则直接使用缓存
    if (networkRequest == null) {
      return cacheResponse.newBuilder()
          .cacheResponse(stripBody(cacheResponse))
          .build();
    }

    //需要网络
    Response networkResponse = null;
    try {//执行下一个拦截器，发起网路请求
      networkResponse = chain.proceed(networkRequest);
    } finally {
      。。。
    }

    //本地有缓存，
    if (cacheResponse != null) {
      //并且服务器返回304状态码（说明缓存还没过期或服务器资源没修改）
      if (networkResponse.code() == HTTP_NOT_MODIFIED) {
        //使用缓存数据
        Response response = cacheResponse.newBuilder()
            。。。
            .build();
          。。。。
         //返回缓存 
        return response;
      } else {
        closeQuietly(cacheResponse.body());
      }
    }

    //如果网络资源已经修改：使用网络响应返回的最新数据
    Response response = networkResponse.newBuilder()
        .cacheResponse(stripBody(cacheResponse))
        .networkResponse(stripBody(networkResponse))
        .build();

    //将最新的数据缓存起来
    if (cache != null) {
      if (HttpHeaders.hasBody(response) && CacheStrategy.isCacheable(response, networkRequest)) {

        CacheRequest cacheRequest = cache.put(response);
        return cacheWritingResponse(cacheRequest, response);
      }
      。。。。
   //返回最新的数据
    return response;
  }
```

简单的总结下上面的代码都做了些神马： 

1、如果在初始化OkhttpClient的时候配置缓存，则从缓存中取caceResponse 

2、将当前请求request和caceResponse 构建一个CacheStrategy对象  

3、CacheStrategy这个策略对象将根据相关规则来决定caceResponse和Request是否有效，如果无效则分别将caceResponse和request设置为null  

4、经过CacheStrategy的处理(步骤3），如果request和caceResponse都置空，直接返回一个状态码为504，且body为Util.EMPTY_RESPONSE的空Respone对象  

5、经过CacheStrategy的处理(步骤3），resquest 为null而cacheResponse不为null，则直接返回cacheResponse对象  

6、执行下一个拦截器发起网路请求，  

7、如果服务器资源没有过期（状态码304）且存在缓存，则返回缓存  

8、将网络返回的最新的资源（networkResponse）缓存到本地，然后返回networkResponse.  

####5.7 ConectInterceptor

#####5.7.1 源码解析

(1)用来打开到服务端的连接。其实是调用了 StreamAllocation 的`newStream` 方法来打开连接的。关联的 TCP 握手，TLS 握手都发生该阶段。过了这个阶段，和服务端的 socket 连接打通 

(2) ConectInterceptor源码解析

这个类的定义看上去倒是蛮简洁的。ConnectInterceptor的主要职责是建立与服务器之间的连接，但这个事情它主要是委托给StreamAllocation来完成的。如我们前面看到的，StreamAllocation对象是在RetryAndFollowUpInterceptor中分配的。

ConnectInterceptor通过StreamAllocation创建了HttpStream对象和RealConnection对象，随后便调用了realChain.proceed()，向连接中写入HTTP请求，并从服务器读回响应。

```
 @Override public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Request request = realChain.request();
    //从拦截器中得到StreamAllocation对象
    StreamAllocation streamAllocation = realChain.streamAllocation();
    //
    boolean doExtensiveHealthChecks = !request.method().equals("GET");
    HttpCodec httpCodec = streamAllocation.newStream(client, chain, doExtensiveHealthChecks);
   //关键代码
   //即为当前请求找到合适的连接，可能复用已有连接也可能是重新创建的连接，返回的连接由连接池负责决定。
   RealConnection connection = streamAllocation.connection();
   //执行下一个拦截器
    return realChain.proceed(request, streamAllocation, httpCodec, connection);
  }
```

(3)ConnectInterceptor获取Interceptor传过来的StreamAllocation,streamAllocation.newStream()

(4)将刚才创建的用于网络IO的RealConnection对象，以及对于与服务器交互最为关键的HttpCodec等对象传递给后面的拦截器

(5)**总结**

```
1、弄一个RealConnection对象
2、选择不同的连接方式
3、CallServerInterceptor
```

##### 5.7.2 Okhttp 连接池ConnectionPool原理解析

```

```



####5.8 CallServerInterceptor

(1)用来发起请求并且得到响应。上一个阶段已经握手成功，HttpStream 流已经打开，所以这个阶段把 Request 的请求信息传入流中，并且从流中读取数据封装成 Response 返回

(2)CallServerInterceptor负责向服务器发起真正的访问请求，并在接收到服务器返回后读取响应返回。 

(3)`CallServerInterceptor`首先将http请求头部发给服务器，如果http请求有body的话，会再将body发送给服务器，继而通过`httpStream.finishRequest()`结束http请求的发送。

随后便是从连接中读取服务器返回的http响应，并构造Response。

如果请求的header或服务器响应的header中，`Connection`值为`close`，`CallServerInterceptor`还会关闭连接。

最后便是返回Response。

####5.9 拦截器总结

#####5.9.1 OkHttp拦截器--总结1

- 1、创建 一系列拦截器，并将其放入其中一个拦截器list中（包含了应用拦截器和网络拦截器）
- 2、创建一个拦截器链RealInterceptorChain,并执行拦截器链的proceed方法，这个proceed方法的核心就是创建下一个拦截器链。

 ##### 5.9.2 OkHttp拦截器--总结2

- 1、在发起请求钱，对request进行处理

- 2、调用下一个拦截器，获取response

- 3、对response进行处理，返回给上一个拦截






### 六、Okhttp网络底层详解

####6.1 Address源码分析 

####6.2 StreamAllocation源码分析 

####6.3 httpCodec 源码分析

### 七、OkHttp的get()&post()

#### 7.1 get()方法的使用

```
  public static void getOkHttp(final Activity  context, final TextView textView) {

        OkHttpClient client = new OkHttpClient();
        /**
         * 构造Request 对象
         * 采用建造者模式，链式调用指明进行get请求，传入get的请求地址
         *
         */
        final Request request = new Request.Builder()
                .get()
                .url(URL_GET_PATH1)
                .build();
        Call call = client.newCall(request);

        //以异步的方式去执行请求,调用的是call.enqueue，将call加入调度队列，
        // 然后等待任务执行完成，我们在Callback中即可得到结果。
        call.enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {
                //请求失败的处理
            }

            @Override
            public void onResponse(Call call, Response response) throws IOException {
                //请求成功返回结果
                //如果希望返回的是字符串
                final String responseData=response.body().string();
                //如果希望返回的是二进制字节数组
                byte[] responseBytes=response.body().bytes();
                //如果希望返回的是inputStream,有inputStream我们就可以通过IO的方式写文件.
                InputStream responseStream=response.body().byteStream();
                //注意，此时的线程不是ui线程，
                // 如果此时我们要用返回的数据进行ui更新，操作控件，就要使用相关方法
                context.runOnUiThread(new Runnable() {
                    @Override
                    public void run() {
                        // 更新UI的操作
                        textView.setText(responseData);
                    }
                });

            }
        });
    }

```



#### 7.2 post()方法的使用

##### 7.2.1  RequestBody--json数据提交



```

    public static void post1(String url, String json) {
       //上传数据的类型，Content-Type 指明了上传数据的后缀格式
        MediaType jsonType = MediaType.parse("application/json;charset=utf-8");
        OkHttpClient client = new OkHttpClient();
        //封装数据格式
        RequestBody formBody = RequestBody.create(jsonType, json);
        final Request request = new Request.Builder()
                .url(url)
                .post(formBody)
                .build();
        Call call = client.newCall(request);
        call.enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {
                Log.e("TAG", "onFailure: " + e.getMessage());
            }

            @Override
            public void onResponse(Call call, Response response) throws IOException {
                Log.e("TAG", "onResponse: " + response.body().string());
            }
        });
    }

```

#####7.3 FromBody---表单提交

​        这种能满足大部分的需求，FromBody用于提交表单键值对,key-value,其作用类似于HTML中的<form>标记。比如username="LHX",age="21"等类似的键值对；这个是用HashMap

```
  public static void post2(String url) {
        //把参数 传进map中
        HashMap<String, String> paramsMap = new HashMap<>();
        paramsMap.put("name", "zhuzhuzhu");
        paramsMap.put("client", "Android");
        paramsMap.put("id", "32564");
        FormBody.Builder builder = new FormBody.Builder();
        for (String key : paramsMap.keySet()) {
            //追加表单信息
            builder.add(key, paramsMap.get(key));
        }

        OkHttpClient okHttpClient = new OkHttpClient();
        RequestBody formBody = builder.build();
        Request request = new Request.Builder()
                .url(url)
                .post(formBody)
                .build();
        Call call = okHttpClient.newCall(request);

        call.enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {
                Log.e(TAG, "onFailure: e : " + e.getMessage());
            }

            @Override
            public void onResponse(Call call, Response response) throws IOException {
                Log.e(TAG, "onResponse: " + response.body().string());
            }
        });
    }
```

FromBody---表单提交 另一种提交数据的方式 list

```
 public static void post3(String url) {
        List<RequestParameter> parameter = new ArrayList<>();
        RequestParameter rp1 = new RequestParameter("name", "wang");
        parameter.add(rp1);

        RequestParameter rp2 = new RequestParameter("client", "Android");
        parameter.add(rp2);

        RequestParameter rp3 = new RequestParameter("id", "369852");
        parameter.add(rp3);

        //创建一个FormBody.Builder
        FormBody.Builder builder = new FormBody.Builder();

        if (parameter != null && parameter.size() > 0) {
            for (RequestParameter p : parameter) {
                builder.add(p.getName(), p.getValue());
            }

        }
        RequestBody formbody = builder.build();
        OkHttpClient okHttpClient = new OkHttpClient();
        Request request = new Request.Builder()
                .url(url)
                .post(formbody)
                .build();

        Call call = okHttpClient.newCall(request);
        call.enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {

                Log.e(TAG, "onFailure: " + e.getMessage());
            }

            @Override
            public void onResponse(Call call, Response response) throws IOException {
                Log.e(TAG, "onResponse: " + response.body().string());
            }
        });

    }
```



##### 7.4 MultipartBody---文件上传

​          MultipartBody可以构建与HTML文件上传格式兼容的复杂请求体。构建MultipartBody这个类时，需要手动设置type为multipart/form-data，不然无法上传<name, value>类型的form数据。 原因是MultipartBody默认的type是mixed；

​        通过第二节boundary的分析，我们知道需要给图片设置Content-Type为image/png，给form类型的数据 设置Content-type为multipart/form-data

[OkHttp之MultipartBody上传文件](http://www.liziyang.top/2016/12/05/OkHttp%E4%B9%8BMultipartBody%E4%B8%8A%E4%BC%A0%E6%96%87%E4%BB%B6/)

```
  public static void post5(String url,File file) {
        OkHttpClient client = new OkHttpClient();
        // 创建一个RequestBody，文件的类型是image/png
        RequestBody requestBody = RequestBody.create(MediaType.parse("image/png"), file);
        // 初始化请求体对象，设置Content-Type以及文件数据流
        MultipartBody multipartBody = new MultipartBody.Builder()
                // 设置type为"multipart/form-data"，不然无法上传参数
                .setType(MultipartBody.FORM)
                .addFormDataPart("filename", "xxx.png", requestBody)
                .addFormDataPart("comment", "上传一个图片哈哈哈哈")
                .build();
        Request request = new Request.Builder()
                .url(url)
                .post(multipartBody)
                .build();
        client.newCall(request).enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {
            }
            @Override
            public void onResponse(Call call, Response response) throws IOException {
                System.out.println("上传返回：\n" + response.body().string());
            }
        });
    }
```

另外一种写法：

```
  public static void post4(String url) {
        File file = new File(Environment.getExternalStorageDirectory(), "balabala.png");
       //  上传的文件类型，Content-Type指明了上传文件的后缀格式
        MediaType media_type_png = MediaType.parse("image/png");
        //根据文件格式，封装文件
        RequestBody filebody = MultipartBody.create(media_type_png, file);
        // 初始化请求体对象，设置Content-Type以及文件数据流
        MultipartBody.Builder multiBuilder = new MultipartBody.Builder();

        //参数以添加header 方式 将参数封装，否则上传参数为空
        //设置请求体
        multiBuilder.setType(MultipartBody.FORM);

        /**
         * 这里是封装上传图片参数
         *
         * 参数分别为， 请求key ，文件名称 ， RequestBody
         *
         * 是 MultipartBody.Builder 的 addFormDataPart 方法，是对于之前的 addPart 方法做了
         * 做了一个封装，所以，不需要再去配置 Header 之类的。
         */
        multiBuilder.addFormDataPart("file", file.getName(), filebody);
        //封装请求参数，这里最重要
        HashMap<String, String> params = new HashMap<>();
        params.put("client", "Android");
        params.put("uid", "1061");
        params.put("token", "1911173227afe098143caf4d315a436d");
        params.put("uuid", "A000005566DA77");
        //参数以添加header方式将参数封装，否则上传参数为空
        if (params != null && !params.isEmpty()) {
            for (String key : params.keySet()) {
                multiBuilder.addPart(Headers.of("Content-Disposition", "form-data; name=\"" + key + "\""), RequestBody.create(null, params.get(key)));


            }
        }
        RequestBody multiBody = multiBuilder.build();
        OkHttpClient okHttpClient = new OkHttpClient();
        Request request = new Request.Builder()
                .url(url)
                .post(multiBody)
                .build();

        Call call = okHttpClient.newCall(request);
        call.enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {
                Log.e(TAG, "onFailure: " );
            }

            @Override
            public void onResponse(Call call, Response response) throws IOException {
                Log.e(TAG, "onResponse: " );
            }
        });

    }
```





### 九、 经典试题

#### 9.1 HttpClient&HttpUrlConnection 

#### 9.2 OkHttp来实现WebSocket连接 

#### 9.3 WebSocket&轮询相关 

#### 9.4 Http缓存、Etag等标示作用 

#### 9.5  断点续传原理&Okhttp如何实现 

#### 9.6 多线程下载 

#### 9.7 文件上传&Okhttp如何处理文件上传 

#### 9.8 如何解析Json类型数据 

#### 9.9 Https／对称加密&不对称加密 






