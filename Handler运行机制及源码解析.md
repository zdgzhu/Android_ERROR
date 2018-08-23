# Handler运行机制及源码解析

## 一、概述

### 1.1 Handler 、 Looper 、Message  之间的关系

Handler 、 Looper 、Message 这三者都与Android异步消息处理线程相关的概念。那么什么叫异步消息处理线程呢？

 

异步消息处理线程启动后会进入一个无限的循环体之中，每循环一次，从其内部的消息队列中取出一个消息，然后回调相应的消息处理函数，执行完成一个消息后则继续循环。若消息队列为空，线程则会阻塞等待。

 

说了这一堆，那么和Handler 、 Looper 、Message有啥关系？其实Looper负责的就是创建一个MessageQueue，然后进入一个无限循环体不断从该MessageQueue中读取消息，而消息的创建者就是一个或多个Handler 。

## 二、源码解析





























## 参考资料

[Android 异步消息处理机制](https://blog.csdn.net/lmj623565791/article/details/38377229)



