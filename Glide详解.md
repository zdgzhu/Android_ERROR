#Glide详解

## 一、Glide 基本概念

### 1.1 Model

表示的是我们数据的来源，可以是url类型，本地图片类型等等

### 1.2 Data



### 1.3 Resource

对获得的原始数据进行解码，对解码之后的资源，称之为Resource

### 1.4 TransformedResource

就是把Resource进行转换，对转换后的资源，称之为TransformedResource

### 1.5 TranscodeResource

负责转码的是Transcode ,解码之后的称之为TranscodeResource

### 1.6 Target

### 1.7 示意图

![](D:\AndroidFile\Photo\Glide\glide02.png)



## 二、基本用法及源码解析 

### 2.1 Glide常用配置参数(3.7.0)

```
 Glide
                .with(getApplicationContext())
                .load(url)
                .into(imageView);
```

```
 public void loadImage(View view) {
        Glide.with(getApplicationContext()) // 指定Context
                .load(url)// 指定图片的URL
                .placeholder(R.mipmap.ic_launcher)// 指定图片未成功加载前显示的图片
                .error(R.mipmap.ic_launcher)// 指定图片加载失败显示的图片
                .override(300, 300)//指定图片的尺寸
                .fitCenter()//指定图片缩放类型为
                .centerCrop()// 指定图片缩放类型为
                .skipMemoryCache(true)// 跳过内存缓存
                .diskCacheStrategy(DiskCacheStrategy.NONE)//跳过磁盘缓存
                .diskCacheStrategy(DiskCacheStrategy.SOURCE)//仅仅只缓存原来的全分辨率的图像
                .diskCacheStrategy(DiskCacheStrategy.RESULT)//仅仅缓存最终的图像
                .diskCacheStrategy(DiskCacheStrategy.ALL)//缓存所有版本的图像
                .priority(Priority.HIGH)//指定优先级.Glide将会用他们作为一个准则，并尽可能的处理这些请求，但是它不能保证所有的图片都会按照所要求的顺序加载。优先级排序:IMMEDIATE > HIGH > NORMAL >　LOW
                .into(imageView);//指定显示图片的Imageview
    }
```

### 2.2 with方法 

####2.2.1 with方法的功能

​        首先，调用Glide.with()方法用于创建一个加载图片的实例。with()方法可以接收Context、Activity或者Fragment类型的参数。也就是说我们选择的范围非常广，不管是在Activity还是Fragment中调用with()方法，都可以直接传this。那如果调用的地方既不在Activity中也不在Fragment中呢？也没关系，我们可以获取当前应用程序的ApplicationContext，传入到with()方法当中。注意with()方法中传入的实例会决定Glide加载图片的生命周期，如果传入的是Activity或者Fragment的实例，那么当这个Activity或Fragment被销毁的时候，图片加载也会停止。如果传入的是ApplicationContext，那么只有当应用程序被杀掉的时候，图片加载才会停止。 

#### 2.2.2  requestManager获取

#### 2.3 requestManagerRetriever的get方法



###2.3 load方法 

#### 2.3.1 load的功能 

​        接下来看一下load()方法，这个方法用于指定待加载的图片资源。Glide支持加载各种各样的图片资源，包括网络图片、本地图片、应用资源、二进制流、Uri对象等等。因此load()方法也有很多个方法重载，除了我们刚才使用的加载一个字符串网址之外，你还可以这样使用load()方法： 

```
// 加载本地图片
File file = new File(getExternalCacheDir() + "/image.jpg");
Glide.with(this).load(file).into(imageView);

// 加载应用资源
int resource = R.drawable.image;
Glide.with(this).load(resource).into(imageView);

// 加载二进制流
byte[] image = getImageBytes();
Glide.with(this).load(image).into(imageView);

// 加载Uri对象
Uri imageUri = getImageUri();
Glide.with(this).load(imageUri).into(imageView);
```

### 2.3.2 源码解析



###2.4 into方法 

####2.4.1 （buildTarget）

####2.4.2 request建立和begin方法  

#### 2.4.3 Loadprovider 

#### 2.4.4 硬盘缓存／内存缓存 

####2.4.5 内存缓存的读取 

####2.4.6 内存缓存的写入 

## 三、API扩展

### 3.1 占位图placeholder

​       观察刚才加载网络图片的效果，你会发现，点击了Load Image按钮之后，要稍微等一会图片才会显示出来。这其实很容易理解，因为从网络上下载图片本来就是需要时间的。那么

​        我们有没有办法再优化一下用户体验呢？当然可以，Glide提供了各种各样非常丰富的API支持，其中就包括了占位图功能。

​        顾名思义，占位图就是指在图片的加载过程中，我们先显示一张临时的图片，等图片加载出来了再替换成要加载的图片。

​        下面我们就来学习一下Glide占位图功能的使用方法，首先我事先准备好了一张loading.jpg图片，用来作为占位图显示。然后修改Glide加载部分的代码，如下所示：

```
Glide.with(this)
     .load(url)
     .placeholder(R.drawable.loading)
     .into(imageView);
```

​       没错，就是这么简单。我们只是在刚才的三步走之间插入了一个placeholder()方法，然后将占位图片的资源id传入到这个方法中即可。另外，这个占位图的用法其实也演示了Glide当中绝大多数API的用法，其实就是在load()和into()方法之间串接任意想添加的功能就可以了。

​       不过如果你现在重新运行一下代码并点击Load Image，很可能是根本看不到占位图效果的。因为Glide有非常强大的缓存机制，我们刚才加载那张必应美图的时候Glide自动就已经将它缓存下来了，下次加载的时候将会直接从缓存中读取，不会再去网络下载了，因而加载的速度非常快，所以占位图可能根本来不及显示。

因此这里我们还需要稍微做一点修改，来让占位图能有机会显示出来，修改代码如下所示： 

```
Glide.with(this)
     .load(url)
     .placeholder(R.drawable.loading)
     .diskCacheStrategy(DiskCacheStrategy.NONE)//跳过磁盘缓存
     .into(imageView);
```



### 3.3 异常占位图error

异常占位图就是指，如果因为某些异常情况导致图片加载失败，比如说手机网络信号不好，这个时候就显示这张异常占位图。

异常占位图的用法相信你已经可以猜到了，首先准备一张error.jpg图片，然后修改Glide加载部分的代码，如下所示：

```
Glide.with(this)
     .load(url)
     .placeholder(R.drawable.loading)
     .error(R.drawable.error)
     .diskCacheStrategy(DiskCacheStrategy.NONE)
     .into(imageView);
```

很简单，这里又串接了一个error()方法就可以指定异常占位图了。

现在你可以将图片的url地址修改成一个不存在的图片地址，或者干脆直接将手机的网络给关了，然后重新运行程序

### 3.4  指定图片格式

####3.4.1 加载静态图(asBitmap) 

我们还需要再了解一下Glide另外一个强大的功能，那就是Glide是支持加载GIF图片的。这一点确实非常牛逼，因为相比之下Jake Warton曾经明确表示过，Picasso是不会支持加载GIF图片的。

而使用Glide加载GIF图并不需要编写什么额外的代码，Glide内部会自动判断图片格式。比如这是一张GIF图片的URL地址：

```
http://p1.pstatp.com/large/166200019850062839d3
```

我们只需要将刚才那段加载图片代码中的URL地址替换成上面的地址就可以了 ;

也就是说，不管我们传入的是一张普通图片，还是一张GIF图片，Glide都会自动进行判断，并且可以正确地把它解析并展示出来。

但是如果我想指定图片的格式该怎么办呢？就比如说，**我希望加载的这张图必须是一张静态图片**，我不需要Glide自动帮我判断它到底是静图还是GIF图。

想实现这个功能仍然非常简单，我们只需要再串接一个新的方法就可以了，如下所示：

```
Glide.with(this)
     .load(url)
     .asBitmap() //只允许加载静态图片，不需要Glide去帮我们自动进行图片格式的判断了。
     .placeholder(R.drawable.loading)
     .error(R.drawable.error)
     .diskCacheStrategy(DiskCacheStrategy.NONE)
     .into(imageView);
```

由于调用了asBitmap()方法，现在GIF图就无法正常播放了，而是会在界面上显示第一帧的图片。 

#### 3.4.2 加载动态图(asGif)

那么类似地，既然我们能强制指定加载静态图片，就也能强制指定加载动态图片。比如说我们想要实现必须加载动态图片的功能，就可以这样写： 

```
Glide.with(this)
     .load(url)
     .asGif()
     .placeholder(R.drawable.loading)
     .error(R.drawable.error)
     .diskCacheStrategy(DiskCacheStrategy.NONE)
     .into(imageView);
```

这里调用了asGif()方法替代了asBitmap()方法，很好理解，相信不用我多做什么解释了。

那么既然指定了只允许加载动态图片，如果我们传入了一张静态图片的URL地址又会怎么样呢？试一下就知道了，将图片的URL地址改成刚才的必应美图，

**没错，如果指定了只能加载动态图片，而传入的图片却是一张静图的话，那么结果自然就只有加载失败喽。** 

### 3.5 指定图片的大小（override）

实际上，使用Glide在绝大多数情况下我们都是不需要指定图片大小的。

在学习本节内容之前，你可能还需要先了解一个概念，就是我们平时在加载图片的时候很容易会造成内存浪费。什么叫内存浪费呢？比如说一张图片的尺寸是1000*1000像素，但是我们界面上的ImageView可能只有200*200像素，这个时候如果你不对图片进行任何压缩就直接读取到内存中，这就属于内存浪费了，因为程序中根本就用不到这么高像素的图片。

关于图片压缩这方面，我之前也翻译过Android官方的一篇文章，感兴趣的朋友可以去阅读一下 [Android高效加载大图、多图解决方案，有效避免程序OOM](http://blog.csdn.net/guolin_blog/article/details/9316683) 。

而使用Glide，我们就完全不用担心图片内存浪费，甚至是内存溢出的问题。因为Glide从来都不会直接将图片的完整尺寸全部加载到内存中，而是用多少加载多少。Glide会自动判断ImageView的大小，然后只将这么大的图片像素加载到内存当中，帮助我们节省内存开支。

当然，Glide也并没有使用什么神奇的魔法，它内部的实现原理其实就是上面那篇文章当中介绍的技术，因此掌握了最基本的实现原理，你也可以自己实现一套这样的图片压缩机制。

也正是因为Glide是如此的智能，所以刚才在开始的时候我就说了，在绝大多数情况下我们都是不需要指定图片大小的，因为Glide会自动根据ImageView的大小来决定图片的大小。

不过，如果你真的有这样的需求，必须给图片指定一个固定的大小，Glide仍然是支持这个功能的。修改Glide加载部分的代码，如下所示：

```
Glide.with(this)
     .load(url)
     .placeholder(R.drawable.loading)
     .error(R.drawable.error)
     .diskCacheStrategy(DiskCacheStrategy.NONE)
     .override(100, 100)//指定显示的图片的宽和高是100*100像素
     .into(imageView);
```

仍然非常简单，这里使用override()方法指定了一个图片的尺寸，也就是说，Glide现在只会将图片加载成100*100像素的尺寸，而不会管你的ImageView的大小是多少了。 







##四、试题

### 3.1 bitmap&oom&优化bitmap、

### 3.2 三级缓存&lrucache 
















