# OkHttp解析

### 1、OkHttp框架的整体设计思路解析



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





### 4、OkHttp核心类OkhttpClient/call 解析



### 5、Okhttp 连接池ConnectionPool原理解析



### 6、Okhttp调度器Dispatcher源码分析



### 7、OkHttp任务调度和调度模型分析 





### 8、OkHttp拦截器Interceptor源码分析 





### 9、OkHttp缓存策略源码分析 



### 10、OkHttp链接复用原理分析 



### 11、Okhttp网络底层详解（Address/StreamAllocation/httpCodec） 





























































