# rxjava异步框架源码解析

- 10-1 rxjava基本用法和观察者模式：01-传统观察者模式
- 10-2 rxjava观察者模式和基本用法
- 10-3 rxjava如何创建Observable&observer／subscriber
- 10-4 rxjava如何创建subscriber以及如何完成订阅
- 10-5 rxjava操作符之map基本使用
- 10-6 rxjava操作符之map源码探究：lift
- 10-7 rxjava操作符之flatmap
- 10-8 rxjava线程控制：多线程编程准则&Rxjava如何处理多线程&&Schedulers
- 10-9 rxjava线程控制：两个小例子&observeOn和SubscribeOn
- 10-10 rxjava线程控制：SubscribeOn源码剖析
- 10-11 rxjava线程控制：ObserveOn源码剖析&&subscribeOn可以调用几次

##一、rxjava 2.2基本用法和观察者模式

### 1.1、Rxjava四要素

(1)观察者

(2)被观察者

(3)订阅

(4)事件

#### 1.1.2 Rxjava 三部曲

1、创建一个可观察对象Observable发射数据流 

2、通过操作符Operator加工处理数据流 

3、通过线程调度器Scheduler指定操作数据流所在的线程 

4、创建一个观察者Observer接收响应数据流 

![](D:\AndroidFile\Photo\Rxjava\Rxjava1_01.png)

#### 1.1.3 Rxjava 2.x 对比Rxjava 1.x

不过此次更新中，出现了两种观察者模式：

- Observable ( 被观察者 ) / Observer ( 观察者 )

- Flowable （被观察者）/ Subscriber （观察者）

  ![](D:\AndroidFile\Photo\Rxjava\Rxjava1_02.png)



###1.2、Rxjava基本用法解析

```
  /**第一步：创建被观察者 ，create
     */
    Observable observerable=Observable.create(new ObservableOnSubscribe() {
        @Override
        public void subscribe(ObservableEmitter emitter) throws Exception {
            emitter.onNext("hello");
             emitter.onNext("Imooc");
            emitter.onComplete();
        }
    });

    //通过just来创建被观察者
    Observable observableJust = Observable.just("hello", "tim");
    //通过from方法来创建被观察者
    String params []= {};
    Observable observableFrom = Observable.fromArray(params);

    /**
     * 第二步：创建观察者
     */
    Observer observer=new Observer() {
    //rxjava 2.x相对于rxjava 1.x 不同的地方
      private Disposable disposable;
        @Override
        public void onSubscribe(Disposable d) {
           disposable = d;
        }

        @Override
        public void onNext(Object o) {
// 在RxJava 2.x 中，新增的Disposable可以做到切断的操作，让Observer观察者不再接收上游事件
        disposable.dispose(); 
        }

        @Override
        public void onError(Throwable e) {

        }

        @Override
        public void onComplete() {

        }
    };

    /**
     * 第三步 : 订阅
     */
    public void doRxjava() {
        observerable.subscribe(observer);
    }
```

###1.3  rxjava如何如何完成订阅(subscriber) 

##二、 操作符

###2.1 变换

就是将事件序列中的对象或者整个序列进行加工处理

###2.2 map

​        Map基本算是RxJava中一个最简单的操作符了，熟悉RxJava 1.x的知道，它的作用是对发射时间发送的每一个事件应用一个函数，是的每一个事件都按照指定的函数去变化，而在2.x中它的作用几乎一致。 

​         对Observable发射的数据都应用(apply)一个函数，这个函数返回一个Observable，然后合并这些Observables，并且发送（emit）合并的结果。 flatMap和map操作符很相像，flatMap发送的是合并后的Observables，map操作符发送的是应用函数后返回的结果集 

 ![](D:\AndroidFile\Photo\Rxjava\java01.png)



```
Observable.create(new ObservableOnSubscribe<Integer>() {
            @Override
            public void subscribe(@NonNull ObservableEmitter<Integer> e) throws Exception {
                e.onNext(1);
                e.onNext(2);
                e.onNext(3);
            }
        }).map(new Function<Integer, String>() {
            @Override
            public String apply(@NonNull Integer integer) throws Exception {
                return "This is result " + integer;
            }
        }).subscribe(new Consumer<String>() {
            @Override
            public void accept(@NonNull String s) throws Exception {
                mRxOperatorsText.append("accept : " + s +"\n");
                Log.e(TAG, "accept : " + s +"\n" );
            }
        });
```

![](D:\AndroidFile\Photo\Rxjava\Rxjava02.png)

是的，map基本作用就是将一个Observable通过某种函数关系，转换为另一种Observable，上面例子中就是把我们的Integer数据变成了String类型。从Log日志显而易见。 



###2.3 flatMap

FlatMap 是一个很有趣的东西，我坚信你在实际开发中会经常用到。它可以把一个发射器Observable 通过某种方法转换为多个Observables，然后再把这些分散的Observables装进一个单一的发射器Observable。但有个需要注意的是，flatMap并不能保证事件的顺序，如果需要保证，需要用到我们下面要讲的ConcatMap。 

1. 将传入的事件对象封装成一个Observable对象
2. 这是不会直接发送这个Observable,而是将这个Observable激活让它自己开始发送事件
3. 每一个创建出来的的Observable发送的事件，都被汇入同一个Observable.

![](D:\AndroidFile\Photo\Rxjava\rxjava07.png)

```
Observable.create(new ObservableOnSubscribe<Integer>() {
            @Override
            public void subscribe(@NonNull ObservableEmitter<Integer> e) throws Exception {
                e.onNext(1);
                e.onNext(2);
                e.onNext(3);
            }
        }).flatMap(new Function<Integer, ObservableSource<String>>() {
            @Override
            public ObservableSource<String> apply(@NonNull Integer integer) throws Exception {
                List<String> list = new ArrayList<>();
                for (int i = 0; i < 3; i++) {
                    list.add("I am value " + integer);
                }
                int delayTime = (int) (1 + Math.random() * 10);
                return Observable.fromIterable(list).delay(delayTime, TimeUnit.MILLISECONDS);
            }
        }).subscribeOn(Schedulers.newThread())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Consumer<String>() {
                    @Override
                    public void accept(@NonNull String s) throws Exception {
                        Log.e(TAG, "flatMap : accept : " + s + "\n");
                        mRxOperatorsText.append("flatMap : accept : " + s + "\n");
                    }
                });
```

![](D:\AndroidFile\Photo\Rxjava\rxjava09.png)

一切都如我们预期中的有意思，为了区分concatMap（下一个会讲），我在代码中特意动了一点小手脚，我采用一个随机数，生成一个时间，然后通过delay（后面会讲）操作符，做一个小延时操作，而查看Log日志也确认验证了我们上面的说法，它是无序的。 





### 2.4 Zip

zip专用于合并事件，该合并不是连接（连接操作符后面会说），而是两两配对，也就意味着，最终配对出的Observable发射事件数目只和少的那个相同。 

![](D:\AndroidFile\Photo\Rxjava\rxjava03.png)



```
Observable.zip(getStringObservable(), getIntegerObservable(), new BiFunction<String, Integer, String>() {
            @Override
            public String apply(@NonNull String s, @NonNull Integer integer) throws Exception {
                return s + integer;
            }
        }).subscribe(new Consumer<String>() {
            @Override
            public void accept(@NonNull String s) throws Exception {
                mRxOperatorsText.append("zip : accept : " + s + "\n");
                Log.e(TAG, "zip : accept : " + s + "\n");
            }
        });
        
private Observable<String> getStringObservable() {
        return Observable.create(new ObservableOnSubscribe<String>() {
            @Override
            public void subscribe(@NonNull ObservableEmitter<String> e) throws Exception {
                if (!e.isDisposed()) {
                    e.onNext("A");
                    mRxOperatorsText.append("String emit : A \n");
                    Log.e(TAG, "String emit : A \n");
                    e.onNext("B");
                    mRxOperatorsText.append("String emit : B \n");
                    Log.e(TAG, "String emit : B \n");
                    e.onNext("C");
                    mRxOperatorsText.append("String emit : C \n");
                    Log.e(TAG, "String emit : C \n");
                }
            }
        });
    }

    private Observable<Integer> getIntegerObservable() {
        return Observable.create(new ObservableOnSubscribe<Integer>() {
            @Override
            public void subscribe(@NonNull ObservableEmitter<Integer> e) throws Exception {
                if (!e.isDisposed()) {
                    e.onNext(1);
                    mRxOperatorsText.append("Integer emit : 1 \n");
                    Log.e(TAG, "Integer emit : 1 \n");
                    e.onNext(2);
                    mRxOperatorsText.append("Integer emit : 2 \n");
                    Log.e(TAG, "Integer emit : 2 \n");
                    e.onNext(3);
                    mRxOperatorsText.append("Integer emit : 3 \n");
                    Log.e(TAG, "Integer emit : 3 \n");
                    e.onNext(4);
                    mRxOperatorsText.append("Integer emit : 4 \n");
                    Log.e(TAG, "Integer emit : 4 \n");
                    e.onNext(5);
                    mRxOperatorsText.append("Integer emit : 5 \n");
                    Log.e(TAG, "Integer emit : 5 \n");
                }
            }
        });
    }
    
    
```

![](D:\AndroidFile\Photo\Rxjava\rxjava04.png)

需要注意的是：

1) zip 组合事件的过程就是分别从发射器A和发射器B各取出一个事件来组合，并且一个事件只能被使用一次，组合的顺序是严格按照事件发送的顺序来进行的，所以上面截图中，可以看到，1永远是和A 结合的，2永远是和B结合的。

2) 最终接收器收到的事件数量是和发送器发送事件最少的那个发送器的发送事件数目相同，所以如截图中，5很孤单，没有人愿意和它交往，孤独终老的单身狗。

### 2.5 Concat

对于单一的把两个发射器连接成一个发射器，虽然 zip 不能完成，但我们还是可以自力更生，官方提供的 concat 让我们的问题得到了完美解决。

![](D:\AndroidFile\Photo\Rxjava\rxjava05.png)

```
Observable.concat(Observable.just(1,2,3), Observable.just(4,5,6))
                .subscribe(new Consumer<Integer>() {
                    @Override
                    public void accept(@NonNull Integer integer) throws Exception {
                        mRxOperatorsText.append("concat : "+ integer + "\n");
                        Log.e(TAG, "concat : "+ integer + "\n" );
                    }
                });
```

![](D:\AndroidFile\Photo\Rxjava\rxjava06.png)

如图，可以看到。发射器B把自己的三个孩子送给了发射器A，让他们组合成了一个新的发射器，非常懂事的孩子，有条不紊的排序接收。 

###2.6 concatMap 

上面其实就说了，concatMap 与 FlatMap 的唯一区别就是 concatMap 保证了顺序，所以，我们就直接把 flatMap 替换为 concatMap 验证吧。 

```
Observable.create(new ObservableOnSubscribe<Integer>() {
            @Override
            public void subscribe(@NonNull ObservableEmitter<Integer> e) throws Exception {
                e.onNext(1);
                e.onNext(2);
                e.onNext(3);
            }
        }).concatMap(new Function<Integer, ObservableSource<String>>() {
            @Override
            public ObservableSource<String> apply(@NonNull Integer integer) throws Exception {
                List<String> list = new ArrayList<>();
                for (int i = 0; i < 3; i++) {
                    list.add("I am value " + integer);
                }
                int delayTime = (int) (1 + Math.random() * 10);
                return Observable.fromIterable(list).delay(delayTime, TimeUnit.MILLISECONDS);
            }
        }).subscribeOn(Schedulers.newThread())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Consumer<String>() {
                    @Override
                    public void accept(@NonNull String s) throws Exception {
                        Log.e(TAG, "flatMap : accept : " + s + "\n");
                        mRxOperatorsText.append("flatMap : accept : " + s + "\n");
                    }
                });
```

![](D:\AndroidFile\Photo\Rxjava\rxjava10.png)

结果的确和我们预想的一样。 

### 2.7 lift



### 2.8 interval、takeWhile









##三、 Rxjava 线程Schedulers

### 3.1 Schedulers种类

####3.1.1 Schedulers.immediate()

直接在当前线程运行，相当于不指定线程。这是默认的 `Scheduler`。 Rxjava2中已经取消了

####3.1.2 Schedulers.newThread()

总是启用新线程，并在新线程执行操作。 

####3.1.3 Schedulers.io()

操作（读写文件、读写数据库、网络信息交互等）所使用的 `Scheduler`。行为模式和 `newThread()` 差不多，区别在于 `io()` 的内部实现是是用一个无数量上限的线程池，可以重用空闲的线程，因此多数情况下 `io()` 比 `newThread()` 更有效率。不要把计算工作放在 `io()` 中，可以避免创建不必要的线程。 

####3.1.4 Schedulers.computation()

计算所使用的 `Scheduler`。这个计算指的是 CPU 密集型计算，即不会被 I/O 等操作限制性能的操作，例如图形的计算。这个 `Scheduler` 使用的固定的线程池，大小为 CPU 核数。不要把 I/O 操作放在 `computation()` 中，否则 I/O 操作的等待时间会浪费 CPU。 

####3.1.5 AndroidSchedulers.mainThread()

它指定的操作将在 Android 主线程运行。 

###3.2  Rxjava 线程调度

####3.2.1 Rxjava 如何进行线程控制?

有了这几个 `Scheduler` ，就可以使用 `subscribeOn()` 和 `observeOn()` 两个方法来对线程进行控制了。 * `subscribeOn()`: 指定 `subscribe()` 所发生的线程，即 `Observable.OnSubscribe` 被激活时所处的线程。或者叫做事件产生的线程。 * `observeOn()`: 指定 `Subscriber` 所运行在的线程。或者叫做事件消费的线程。 

```
Observable.just(1, 2, 3, 4)
    .subscribeOn(Schedulers.io()) // 指定 subscribe() 发生在 IO 线程
    .observeOn(AndroidSchedulers.mainThread()) // 指定 Subscriber 的回调发生在主线程
    .subscribe(new Action1<Integer>() {
        @Override
        public void call(Integer number) {
            Log.d(tag, "number:" + number);
        }
    });
```



####3.2.2 subscribeOn()

只能调用一次

####3.2.3 observeOn()

可以调用多次

#### 3.2.4 线程切换注意点

RxJava 内置的线程调度器的确可以让我们的线程切换得心应手，但其中也有些需要注意的地方。

- 简单地说，`subscribeOn()` 指定的就是发射事件的线程，`observerOn` 指定的就是订阅者接收事件的线程。
- 多次指定发射事件的线程只有第一次指定的有效，也就是说多次调用 `subscribeOn()` 只有第一次的有效，其余的会被忽略。
- 但多次指定订阅者接收线程是可以的，也就是说每调用一次 `observerOn()`，下游的线程就会切换一次。

 

```
Observable.create(new ObservableOnSubscribe<Integer>() {
            @Override
            public void subscribe(@NonNull ObservableEmitter<Integer> e) throws Exception {
                Log.e(TAG, "Observable thread is : " + Thread.currentThread().getName());
                e.onNext(1);
                e.onComplete();
            }
        }).subscribeOn(Schedulers.newThread())
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .doOnNext(new Consumer<Integer>() {
                    @Override
                    public void accept(@NonNull Integer integer) throws Exception {
                        Log.e(TAG, "After observeOn(mainThread)，Current thread is " + Thread.currentThread().getName());
                    }
                })
                .observeOn(Schedulers.io())
                .subscribe(new Consumer<Integer>() {
                    @Override
                    public void accept(@NonNull Integer integer) throws Exception {
                        Log.e(TAG, "After observeOn(io)，Current thread is " + Thread.currentThread().getName());
                    }
                });

//输出结果
07-03 14:54:01.177 15121-15438/com.nanchen.rxjava2examples E/RxThreadActivity: Observable thread is : RxNewThreadScheduler-1
07-03 14:54:01.178 15121-15121/com.nanchen.rxjava2examples E/RxThreadActivity: After observeOn(mainThread)，Current thread is main
07-03 14:54:01.179 15121-15439/com.nanchen.rxjava2examples E/RxThreadActivity: After observeOn(io)，Current thread is RxCachedThreadScheduler-2

```

 实例代码中，分别用 `Schedulers.newThread()` 和 `Schedulers.io()` 对发射线程进行切换，并采用 `observeOn(AndroidSchedulers.mainThread()` 和 `Schedulers.io()` 进行了接收线程的切换。可以看到输出中发射线程仅仅响应了第一个 `newThread`，但每调用一次 `observeOn()` ，线程便会切换一次，因此如果我们有类似的需求时，便知道如何处理了。

 

 

 

 

 

 

 

 































