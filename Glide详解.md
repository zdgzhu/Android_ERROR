#Glide详解

## 一、Glide 基本概念

###1.1 Glide中类的作用

####1.1.1 View 

 一般情况下，指Android中的View及其子类控件（包括自定义的），尤其指ImageView。这些控件可在上面绘制Drawable

#### 1.1.2 Target 

Glide中重要的概念，目标。它即可以指封装了一个View的Target（ViewTarget），也可以不包含View（SimpleTarget）。 **Target对象则是用来最终展示图片用的**

#### 1.1.3 Drawable 

指Android中的Drawable类或者它的子类，如BitmapDrawable等。或者Glide中基础Drawable实现的自定义Drawable（如GifDrawable等） 

#### 1.1.4 Request  

加载请求，可以是网络请求或者其他任何下载图片的请求，也是Glide中的一个类。

 Request是用来发出加载图片请求的

####1.1.5 Model 

数据源的提供者，如Url，文件路径等，可以从model中获取InputStream。 

#### 1.1.6 Signature 

签名，可以唯一地标识一个对象。 

#### 1.1.7 recycle() 

Glide中Resource类有此方法，表示该资源不被引用，可以放入池中（此时并没有释放空间）。Android中Bitmap也有此方法，表示释放Bitmap占用的内存。 

#### 1.1.8 RequestManager 

请求管理，每一个Activity都会创建一个RequestManager，根据对应Activity的生命周期管理该Activity上所以的图片请求。 

#### 1.1.9 Engine 

加载图片的引擎，根据Request创建EngineJob和DecodeJob。 

#### 1.1.10 EngineJob 

图片加载。 

#### 1.1.11 DecodeJob 

图片处理 

### 1.2 类分布图



![](D:\AndroidFile\Photo\Glide\glide类分布图.png)

### 1.3 **主要特点** 

(1)支持Memory和Disk图片缓存。

 (2)支持gif和webp格式图片。 

(3)根据Activity/Fragment生命周期自动管理请求。

 (4)使用Bitmap Pool可以使Bitmap复用。

 (5)对于回收的Bitmap会主动调用recycle，减小系统回收压力。 

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
                .asBitmap() //一定要放在load()方法后面
                .asGif() //
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

with()方法是Glide类中的一组静态方法，它有好几个方法重载，我们来看一下Glide类中所有with()方法的方法重载： 

```
public class Glide {

    ...

    public static RequestManager with(Context context) {
        RequestManagerRetriever retriever = RequestManagerRetriever.get();
        return retriever.get(context);
    }

    public static RequestManager with(Activity activity) {
        RequestManagerRetriever retriever = RequestManagerRetriever.get();
        return retriever.get(activity);
    }

    public static RequestManager with(FragmentActivity activity) {
        RequestManagerRetriever retriever = RequestManagerRetriever.get();
        return retriever.get(activity);
    }

    @TargetApi(Build.VERSION_CODES.HONEYCOMB)
    public static RequestManager with(android.app.Fragment fragment) {
        RequestManagerRetriever retriever = RequestManagerRetriever.get();
        return retriever.get(fragment);
    }

    public static RequestManager with(Fragment fragment) {
        RequestManagerRetriever retriever = RequestManagerRetriever.get();
        return retriever.get(fragment);
    }
}
```

可以看到，with()方法的重载种类非常多，既可以传入Activity，也可以传入Fragment或者是Context。每一个with()方法重载的代码都非常简单，都是先调用RequestManagerRetriever的静态get()方法得到一个RequestManagerRetriever对象，这个静态get()方法就是一个单例实现，没什么需要解释的。然后再调用RequestManagerRetriever的实例get()方法，去获取RequestManager对象。

而RequestManagerRetriever的实例get()方法中的逻辑是什么样的呢？我们一起来看一看：

#### 2.3 requestManagerRetriever的get方法

```
public class RequestManagerRetriever implements Handler.Callback {

    private static final RequestManagerRetriever INSTANCE = new RequestManagerRetriever();

    private volatile RequestManager applicationManager;

    ...

    /**
     * Retrieves and returns the RequestManagerRetriever singleton.
     */
    public static RequestManagerRetriever get() {
        return INSTANCE;
    }

    private RequestManager getApplicationManager(Context context) {
        // Either an application context or we're on a background thread.
        if (applicationManager == null) {
            synchronized (this) {
                if (applicationManager == null) {
                    applicationManager = new RequestManager(context.getApplicationContext(),
                            new ApplicationLifecycle(), new EmptyRequestManagerTreeNode());
                }
            }
        }
        return applicationManager;
    }

    public RequestManager get(Context context) {
        if (context == null) {
            throw new IllegalArgumentException("You cannot start a load on a null Context");
        } else if (Util.isOnMainThread() && !(context instanceof Application)) {
            if (context instanceof FragmentActivity) {
                return get((FragmentActivity) context);
            } else if (context instanceof Activity) {
                return get((Activity) context);
            } else if (context instanceof ContextWrapper) {
                return get(((ContextWrapper) context).getBaseContext());
            }
        }
44        return getApplicationManager(context);
    }

    public RequestManager get(FragmentActivity activity) {
 48      if (Util.isOnBackgroundThread()) {
            return get(activity.getApplicationContext());
        } else {
            assertNotDestroyed(activity);
            FragmentManager fm = activity.getSupportFragmentManager();
            return supportFragmentGet(activity, fm);
        }
    }

    public RequestManager get(Fragment fragment) {
        if (fragment.getActivity() == null) {
            throw new IllegalArgumentException("You cannot start a load on a fragment before it is attached");
        }
        if (Util.isOnBackgroundThread()) {
            return get(fragment.getActivity().getApplicationContext());
        } else {
            FragmentManager fm = fragment.getChildFragmentManager();
            return supportFragmentGet(fragment.getActivity(), fm);
        }
    }

    @TargetApi(Build.VERSION_CODES.HONEYCOMB)
    public RequestManager get(Activity activity) {
        if (Util.isOnBackgroundThread() || Build.VERSION.SDK_INT < Build.VERSION_CODES.HONEYCOMB) {
            return get(activity.getApplicationContext());
        } else {
            assertNotDestroyed(activity);
            android.app.FragmentManager fm = activity.getFragmentManager();
            return fragmentGet(activity, fm);
        }
    }

    @TargetApi(Build.VERSION_CODES.JELLY_BEAN_MR1)
    private static void assertNotDestroyed(Activity activity) {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.JELLY_BEAN_MR1 && activity.isDestroyed()) {
            throw new IllegalArgumentException("You cannot start a load for a destroyed activity");
        }
    }

    @TargetApi(Build.VERSION_CODES.JELLY_BEAN_MR1)
    public RequestManager get(android.app.Fragment fragment) {
        if (fragment.getActivity() == null) {
            throw new IllegalArgumentException("You cannot start a load on a fragment before it is attached");
        }
        if (Util.isOnBackgroundThread() || Build.VERSION.SDK_INT < Build.VERSION_CODES.JELLY_BEAN_MR1) {
            return get(fragment.getActivity().getApplicationContext());
        } else {
            android.app.FragmentManager fm = fragment.getChildFragmentManager();
            return fragmentGet(fragment.getActivity(), fm);
        }
    }

    @TargetApi(Build.VERSION_CODES.JELLY_BEAN_MR1)
    RequestManagerFragment getRequestManagerFragment(final android.app.FragmentManager fm) {
        RequestManagerFragment current = (RequestManagerFragment) fm.findFragmentByTag(FRAGMENT_TAG);
        if (current == null) {
            current = pendingRequestManagerFragments.get(fm);
            if (current == null) {
                current = new RequestManagerFragment();
                pendingRequestManagerFragments.put(fm, current);
                fm.beginTransaction().add(current, FRAGMENT_TAG).commitAllowingStateLoss();
                handler.obtainMessage(ID_REMOVE_FRAGMENT_MANAGER, fm).sendToTarget();
            }
        }
        return current;
    }

    @TargetApi(Build.VERSION_CODES.HONEYCOMB)
    RequestManager fragmentGet(Context context, android.app.FragmentManager fm) {
     //向当前的activity添加隐藏的fragment
 117       RequestManagerFragment current = getRequestManagerFragment(fm);
        RequestManager requestManager = current.getRequestManager();
        if (requestManager == null) {
            requestManager = new RequestManager(context, current.getLifecycle(), current.getRequestManagerTreeNode());
            current.setRequestManager(requestManager);
        }
        return requestManager;
    }

    SupportRequestManagerFragment getSupportRequestManagerFragment(final FragmentManager fm) {
        SupportRequestManagerFragment current = (SupportRequestManagerFragment) fm.findFragmentByTag(FRAGMENT_TAG);
        if (current == null) {
            current = pendingSupportRequestManagerFragments.get(fm);
            if (current == null) {
                current = new SupportRequestManagerFragment();
                pendingSupportRequestManagerFragments.put(fm, current);
                fm.beginTransaction().add(current, FRAGMENT_TAG).commitAllowingStateLoss();
                handler.obtainMessage(ID_REMOVE_SUPPORT_FRAGMENT_MANAGER, fm).sendToTarget();
            }
        }
        return current;
    }

    RequestManager supportFragmentGet(Context context, FragmentManager fm) {
  //向当前的activity添加隐藏的fragment
 141      SupportRequestManagerFragment current = getSupportRequestManagerFragment(fm);
        RequestManager requestManager = current.getRequestManager();
        if (requestManager == null) {
            requestManager = new RequestManager(context, current.getLifecycle(), current.getRequestManagerTreeNode());
            current.setRequestManager(requestManager);
        }
        return requestManager;
    }

    ...
}
```

上述代码虽然看上去逻辑有点复杂，但是将它们梳理清楚后还是很简单的。RequestManagerRetriever类中看似有很多个get()方法的重载，什么Context参数，Activity参数，Fragment参数等等，实际上只有两种情况而已，即传入Application类型的参数，和传入非Application类型的参数。

我们先来看传入Application参数的情况。如果在Glide.with()方法中传入的是一个Application对象，那么这里就会调用带有Context参数的get()方法重载，然后会在第44行调用getApplicationManager()方法来获取一个RequestManager对象。其实这是最简单的一种情况，因为Application对象的生命周期即应用程序的生命周期，因此Glide并不需要做什么特殊的处理，它自动就是和应用程序的生命周期是同步的，如果应用程序关闭的话，Glide的加载也会同时终止。

接下来我们看传入非Application参数的情况。不管你在Glide.with()方法中传入的是Activity、FragmentActivity、v4包下的Fragment、还是app包下的Fragment，最终的流程都是一样的，那就是会向当前的Activity当中添加一个隐藏的Fragment。具体添加的逻辑是在上述代码的第117行和第141行，分别对应的app包和v4包下的两种Fragment的情况。那么这里为什么要添加一个隐藏的Fragment呢？因为Glide需要知道加载的生命周期。很简单的一个道理，如果你在某个Activity上正在加载着一张图片，结果图片还没加载出来，Activity就被用户关掉了，那么图片还应该继续加载吗？当然不应该。可是Glide并没有办法知道Activity的生命周期，于是Glide就使用了添加隐藏Fragment的这种小技巧，因为Fragment的生命周期和Activity是同步的，如果Activity被销毁了，Fragment是可以监听到的，这样Glide就可以捕获这个事件并停止图片加载了。

这里额外再提一句，从第48行代码可以看出，**如果我们是在非主线程当中使用的Glide，那么不管你是传入的Activity还是Fragment，都会被强制当成Application来处理。**不过其实这就属于是在分析代码的细节了，本篇文章我们将会把目光主要放在Glide的主线工作流程上面，后面不会过多去分析这些细节方面的内容。

总体来说，第一个with()方法的源码还是比较好理解的。其实就是为了得到一个RequestManager对象而已，然后Glide会根据我们传入with()方法的参数来确定图片加载的生命周期，并没有什么特别复杂的逻辑。不过复杂的逻辑还在后面等着我们呢，接下来我们开始分析第二步，load()方法。

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

####2.3.2 RequestManager源码解析

由于with()方法返回的是一个RequestManager对象，那么很容易就能想到，load()方法是在RequestManager类当中的，所以说我们首先要看的就是RequestManager这个类。不过在上一篇文章中我们学过，Glide是支持图片URL字符串、图片本地路径等等加载形式的，因此RequestManager中也有很多个load()方法的重载。但是这里我们不可能把每个load()方法的重载都看一遍，因此我们就只选其中一个加载图片URL字符串的load()方法来进行研究吧。 

```
public class RequestManager implements LifecycleListener {

    ...
    
    public DrawableTypeRequest<String> load(String string) {
        return (DrawableTypeRequest<String>) fromString().load(string);
    }


    public DrawableTypeRequest<String> fromString() {
        return loadGeneric(String.class);
    }

    private <T> DrawableTypeRequest<T> loadGeneric(Class<T> modelClass) {
        ModelLoader<T, InputStream> streamModelLoader = Glide.buildStreamModelLoader(modelClass, context);
        ModelLoader<T, ParcelFileDescriptor> fileDescriptorModelLoader =
                Glide.buildFileDescriptorModelLoader(modelClass, context);
        if (modelClass != null && streamModelLoader == null && fileDescriptorModelLoader == null) {
            throw new IllegalArgumentException("Unknown type " + modelClass + ". You must provide a Model of a type for"
                    + " which there is a registered ModelLoader, if you are using a custom model, you must first call"
                    + " Glide#register with a ModelLoaderFactory for your custom model class");
        }
        return optionsApplier.apply(
                new DrawableTypeRequest<T>(modelClass, streamModelLoader, fileDescriptorModelLoader, context,
                        glide, requestTracker, lifecycle, optionsApplier));
    }

    ...

}
```

RequestManager类的代码是非常多的，但是经过我这样简化之后，看上去就比较清爽了。在我们只探究加载图片URL字符串这一个load()方法的情况下，那么比较重要的方法就只剩下上述代码中的这三个方法。

那么我们先来看load()方法，这个方法中的逻辑是非常简单的，只有一行代码，就是先调用了fromString()方法，再调用load()方法，然后把传入的图片URL地址传进去。而fromString()方法也极为简单，就是调用了loadGeneric()方法，并且指定参数为String.class，因为load()方法传入的是一个字符串参数。那么看上去，好像主要的工作都是在loadGeneric()方法中进行的了。

其实loadGeneric()方法也没几行代码，这里分别调用了Glide.buildStreamModelLoader()方法和Glide.buildFileDescriptorModelLoader()方法来获得ModelLoader对象。**ModelLoader对象是用于加载图片的**，而我们给load()方法传入不同类型的参数，这里也会得到不同的ModelLoader对象。不过buildStreamModelLoader()方法内部的逻辑还是蛮复杂的，这里就不展开介绍了，要不然篇幅实在收不住，感兴趣的话你可以自己研究。由于我们刚才传入的参数是String.class，因此最终得到的是StreamStringLoader对象，它是实现了ModelLoader接口的。

最后我们可以看到，loadGeneric()方法是要返回一个DrawableTypeRequest对象的，因此在loadGeneric()方法的最后又去new了一个DrawableTypeRequest对象，然后把刚才获得的ModelLoader对象，还有一大堆杂七杂八的东西都传了进去。具体每个参数的含义和作用就不解释了，我们只看主线流程。

那么这个DrawableTypeRequest的作用是什么呢？我们来看下它的源码，如下所示：

####2.3.3 DrawableTypeRequest解析 

```
public class DrawableTypeRequest<ModelType> extends DrawableRequestBuilder<ModelType> implements DownloadOptions {
    private final ModelLoader<ModelType, InputStream> streamModelLoader;
    private final ModelLoader<ModelType, ParcelFileDescriptor> fileDescriptorModelLoader;
    private final RequestManager.OptionsApplier optionsApplier;

    private static <A, Z, R> FixedLoadProvider<A, ImageVideoWrapper, Z, R> buildProvider(Glide glide,
            ModelLoader<A, InputStream> streamModelLoader,
            ModelLoader<A, ParcelFileDescriptor> fileDescriptorModelLoader, Class<Z> resourceClass,
            Class<R> transcodedClass,
            ResourceTranscoder<Z, R> transcoder) {
        if (streamModelLoader == null && fileDescriptorModelLoader == null) {
            return null;
        }

        if (transcoder == null) {
            transcoder = glide.buildTranscoder(resourceClass, transcodedClass);
        }
        DataLoadProvider<ImageVideoWrapper, Z> dataLoadProvider = glide.buildDataProvider(ImageVideoWrapper.class,
                resourceClass);
        ImageVideoModelLoader<A> modelLoader = new ImageVideoModelLoader<A>(streamModelLoader,
                fileDescriptorModelLoader);
        return new FixedLoadProvider<A, ImageVideoWrapper, Z, R>(modelLoader, transcoder, dataLoadProvider);
    }

    DrawableTypeRequest(Class<ModelType> modelClass, ModelLoader<ModelType, InputStream> streamModelLoader,
            ModelLoader<ModelType, ParcelFileDescriptor> fileDescriptorModelLoader, Context context, Glide glide,
            RequestTracker requestTracker, Lifecycle lifecycle, RequestManager.OptionsApplier optionsApplier) {
        super(context, modelClass,
                buildProvider(glide, streamModelLoader, fileDescriptorModelLoader, GifBitmapWrapper.class,
                        GlideDrawable.class, null),
                glide, requestTracker, lifecycle);
        this.streamModelLoader = streamModelLoader;
        this.fileDescriptorModelLoader = fileDescriptorModelLoader;
        this.optionsApplier = optionsApplier;
    }


    public BitmapTypeRequest<ModelType> asBitmap() {
        return optionsApplier.apply(new BitmapTypeRequest<ModelType>(this, streamModelLoader,
                fileDescriptorModelLoader, optionsApplier));
    }

    public GifTypeRequest<ModelType> asGif() {
        return optionsApplier.apply(new GifTypeRequest<ModelType>(this, streamModelLoader, optionsApplier));
    }

    ...
}
```

这个类中的代码本身就不多，我只是稍微做了一点简化。可以看到，最主要的就是它提供了**asBitmap()**和**asGif()**这两个方法。这两个方法我们在上一篇文章当中都是学过的，分别是用于强制指定加载静态图片和动态图片。而从源码中可以看出，它们分别又创建了一个BitmapTypeRequest和GifTypeRequest，如果没有进行强制指定的话，那默认就是使用DrawableTypeRequest。

好的，那么我们再回到RequestManager的load()方法中。刚才已经分析过了，fromString()方法会返回一个DrawableTypeRequest对象，接下来会调用这个对象的load()方法，把图片的URL地址传进去。但是我们刚才看到了，DrawableTypeRequest中并没有load()方法，那么很容易就能猜想到，load()方法是在父类当中的。

DrawableTypeRequest的父类是DrawableRequestBuilder，我们来看下这个类的源码：

####2.3.4 DrawableRequestBuilder解析

```
public class DrawableRequestBuilder<ModelType>
        extends GenericRequestBuilder<ModelType, ImageVideoWrapper, GifBitmapWrapper, GlideDrawable>
        implements BitmapOptions, DrawableOptions {

    DrawableRequestBuilder(Context context, Class<ModelType> modelClass,
            LoadProvider<ModelType, ImageVideoWrapper, GifBitmapWrapper, GlideDrawable> loadProvider, Glide glide,
            RequestTracker requestTracker, Lifecycle lifecycle) {
        super(context, modelClass, loadProvider, GlideDrawable.class, glide, requestTracker, lifecycle);
        // Default to animating.
        crossFade();
    }

    public DrawableRequestBuilder<ModelType> thumbnail(
            DrawableRequestBuilder<?> thumbnailRequest) {
        super.thumbnail(thumbnailRequest);
        return this;
    }


    @Override
    public DrawableRequestBuilder<ModelType> placeholder(int resourceId) {
        super.placeholder(resourceId);
        return this;
    }

  

    @Override
    public DrawableRequestBuilder<ModelType> listener(
            RequestListener<? super ModelType, GlideDrawable> requestListener) {
        super.listener(requestListener);
        return this;
    }
   

    @Override
    public DrawableRequestBuilder<ModelType> override(int width, int height) {
        super.override(width, height);
        return this;
    }

 

    @Override
    public DrawableRequestBuilder<ModelType> load(ModelType model) {
        super.load(model);
        return this;
    }

  
    @Override
    public Target<GlideDrawable> into(ImageView view) {
        return super.into(view);
    }

}
```

DrawableRequestBuilder中有很多个方法，这些方法其实就是Glide绝大多数的API了。里面有不少我们在上篇文章中已经用过了，比如说placeholder()方法、error()方法、diskCacheStrategy()方法、override()方法等。当然还有很多暂时还没用到的API，我们会在后面的文章当中学习。

到这里，第二步load()方法也就分析结束了。为什么呢？因为你会发现DrawableRequestBuilder类中有一个into()方法（上述代码第220行），也就是说，最终load()方法返回的其实就是一个DrawableTypeRequest对象。那么接下来我们就要进行第三步了，分析into()方法中的逻辑。

###2.4 into(ImageView view)方法 

####2.4.1 GenericRequestBuilder解析

```
   Glide.with(getApplicationContext())//指定Context
                .load(url)//指定图片的URL
                .into(image)       ;
```

点击查看into()方法的源码，会跳转到DrawableRequestBuilder.class中的into()方法，

```
  @Override
    public Target<GlideDrawable> into(ImageView view) {
        return super.into(view);
    }
```

显而易见，super.into(view)；调用的是DrawableRequestBuilder的父类中的into()方法，我们进去看一下，

```
public class GenericRequestBuilder<ModelType, DataType, ResourceType, TranscodeType> implements Cloneable {
 
 public Target<TranscodeType> into(ImageView view) {
        Util.assertMainThread();
        if (view == null) {
            throw new IllegalArgumentException("You must pass in a non null View");
        }

        if (!isTransformationSet && view.getScaleType() != null) {
            switch (view.getScaleType()) {
                case CENTER_CROP:
                    applyCenterCrop();
                    break;
                case FIT_CENTER:
                case FIT_START:
                case FIT_END:
                    applyFitCenter();
                    break;
                //$CASES-OMITTED$
                default:
                    // Do nothing.
            }
        }

        return into(glide.buildImageViewTarget(view, transcodeClass));
    }
    }
```

​        

 这里前面一大堆的判断逻辑我们都可以先不用管，等到后面文章讲transform的时候会再进行解释，现在我们只需要关注最后一行代码。最后一行代码先是调用了glide.buildImageViewTarget()方法，这个方法会构建出一个Target对象，**Target对象则是用来最终展示图片用的**，如果我们跟进去的话会看到如下代码： 

###2.5 Target创建

####2.5.1 buildImageViewTarget()

```
public class Glide {
...
<R> Target<R> buildImageViewTarget(ImageView imageView, Class<R> transcodedClass) {
    return imageViewTargetFactory.buildTarget(imageView, transcodedClass);
}
...
}
```

这里其实又是调用了ImageViewTargetFactory的buildTarget()方法，我们继续跟进去，代码如下所示： 

#### 2.5.2 ImageViewTargetFactory解析

#####2.5.2.1 buildTarget()

```
public class ImageViewTargetFactory {

    @SuppressWarnings("unchecked")
    public <Z> Target<Z> buildTarget(ImageView view, Class<Z> clazz) {
        if (GlideDrawable.class.isAssignableFrom(clazz)) {
            return (Target<Z>) new GlideDrawableImageViewTarget(view);
        } else if (Bitmap.class.equals(clazz)) {
         //如果调用的是asBitmap()这个方法就返回BitmapImageViewTarget,如果没有调用asBitmp()这个方法，
        //就返回GlideDrawableImageVeiwTarget的对象
            return (Target<Z>) new BitmapImageViewTarget(view);
        } else if (Drawable.class.isAssignableFrom(clazz)) {
            return (Target<Z>) new DrawableImageViewTarget(view);
        } else {
            throw new IllegalArgumentException("Unhandled class: " + clazz
                    + ", try .as*(Class).transcode(ResourceTranscoder)");
        }
    }
}
buildTRarget这个方法很简单，就是传入两个参数，主要是根据Class<Z>class,来判断我们到底是加载哪一张图片
```

 这个class参数其实基本上只有两种情况：

#####2.5.2.2 BitmapImageViewTarget对象

如果你在使用Glide加载图片的时候调用了asBitmap()方法，那么这里就会构建出BitmapImageViewTarget对象，

#####2.5.2.3 GlideDrawableImageViewTarget对象

否则的话构建的都是GlideDrawableImageViewTarget对象。

#####2.5.2.4 DrawableImageViewTarget对象

至于上述代码中的DrawableImageViewTarget对象，这个通常都是用不到的，我们可以暂时不用管它。 

  也就是说，通过glide.buildImageViewTarget()方法，我们构建出了一个GlideDrawableImageViewTarget对象。

###2.6 into(Y target)

​       那现在回到刚才into()方法的最后一行，可以看到，这里又将这个参数传入到了GenericRequestBuilder另一个接收Target对象的into()方法当中了。我们来看一下这个into()方法的源码： 

```
public <Y extends Target<TranscodeType>> Y into(Y target) {
//只能在主线程中运行，
    Util.assertMainThread();
    if (target == null) {
        throw new IllegalArgumentException("You must pass in a non null Target");
    }
    if (!isModelSet) {
        throw new IllegalArgumentException("You must first set a model (try #load())");
    }
    //获取当前target对象所绑定的Request对象，就是前一个(旧的)Request,只要将旧的Request删除了，才能绑定新的Request, 才能进行新的操作.
    Request previous = target.getRequest();
    if (previous != null) {
        previous.clear();
        requestTracker.removeRequest(previous);
        previous.recycle();
    }
    //创建了一个Request对象
    Request request = buildRequest(target);
    target.setRequest(request);
    lifecycle.addListener(target);
    //执行了这个Request
    requestTracker.runRequest(request);
    return target;
}
```

###2.7 Request对象的创建 

####2.7.1 buildRequest()

**创建新的Request对象；Request是用来发出加载图片请求的**，它是Glide中非常关键的一个组件。我们先来看buildRequest()方法是如何构建Request对象的： 

```
public class GenericRequestBuilder<ModelType, DataType, ResourceType, TranscodeType> implements Cloneable {

...
//创建新的 Request 对象
private Request buildRequest(Target<TranscodeType> target) {
    if (priority == null) {
        priority = Priority.NORMAL;
    }
    return buildRequestRecursive(target, null);
}

private Request buildRequestRecursive(Target<TranscodeType> target, ThumbnailRequestCoordinator parentCoordinator) {
    if (thumbnailRequestBuilder != null) {
    //处理缩略图
        if (isThumbnailBuilt) {
            throw new IllegalStateException("You cannot use a request as both the main request and a thumbnail, "
                    + "consider using clone() on the request(s) passed to thumbnail()");
        }
        // Recursive case: contains a potentially recursive thumbnail request builder.
        if (thumbnailRequestBuilder.animationFactory.equals(NoAnimation.getFactory())) {
            thumbnailRequestBuilder.animationFactory = animationFactory;
        }

        if (thumbnailRequestBuilder.priority == null) {
            thumbnailRequestBuilder.priority = getThumbnailPriority();
        }

        if (Util.isValidDimensions(overrideWidth, overrideHeight)
                && !Util.isValidDimensions(thumbnailRequestBuilder.overrideWidth,
                        thumbnailRequestBuilder.overrideHeight)) {
          thumbnailRequestBuilder.override(overrideWidth, overrideHeight);
        }

        ThumbnailRequestCoordinator coordinator = new ThumbnailRequestCoordinator(parentCoordinator);
        Request fullRequest = obtainRequest(target, sizeMultiplier, priority, coordinator);
        // Guard against infinite recursion.
        isThumbnailBuilt = true;
        // Recursively generate thumbnail requests.
        Request thumbRequest = thumbnailRequestBuilder.buildRequestRecursive(target, coordinator);
        isThumbnailBuilt = false;
        coordinator.setRequests(fullRequest, thumbRequest);
        return coordinator;
    } else if (thumbSizeMultiplier != null) {
        // Base case: thumbnail multiplier generates a thumbnail request, but cannot recurse.
        ThumbnailRequestCoordinator coordinator = new ThumbnailRequestCoordinator(parentCoordinator);
        Request fullRequest = obtainRequest(target, sizeMultiplier, priority, coordinator);
        Request thumbnailRequest = obtainRequest(target, thumbSizeMultiplier, getThumbnailPriority(), coordinator);
        coordinator.setRequests(fullRequest, thumbnailRequest);
        return coordinator;
    } else {
        // Base case: no thumbnail.通过obtainRequest获取一个Request对象
        return obtainRequest(target, sizeMultiplier, priority, parentCoordinator);
    }
}

private Request obtainRequest(Target<TranscodeType> target, float sizeMultiplier, Priority priority,
        RequestCoordinator requestCoordinator) {
        //这里调用了GenericRequest的obtain()方法
    return GenericRequest.obtain(
            loadProvider,
            model,
            signature,
            context,
            priority,
            target,
            sizeMultiplier,
            placeholderDrawable,
            placeholderId,
            errorPlaceholder,
            errorId,
            fallbackDrawable,
            fallbackResource,
            requestListener,
            requestCoordinator,
            glide.getEngine(),
            transformation,
            transcodeClass,
            isCacheable,
            animationFactory,
            overrideWidth,
            overrideHeight,
            diskCacheStrategy);
}
...
}
```

​       可以看到，buildRequest()方法的内部其实又调用了buildRequestRecursive()方法，而**buildRequestRecursive()**方法中的代码虽然有点长，**但是其中90%的代码都是在处理缩略图的。**如果我们只追主线流程的话，那么只需要看第47行代码就可以了。这里调用了obtainRequest()方法来获取一个Request对象，而obtainRequest()方法中又去调用了GenericRequest的obtain()方法。注意这个obtain()方法需要传入非常多的参数，而其中很多的参数我们都是比较熟悉的，像什么placeholderId、errorPlaceholder、diskCacheStrategy等等。因此，我们就有理由猜测，刚才在load()方法中调用的所有API，其实都是在这里组装到Request对象当中的。

#### 2.7.2 GenericRequest 

#####2.7.2.1 GenericRequest.obtain()

obtain()方法实际上就是创建了一个GenericRequest对象，并且初始化了这个对象;那么我们进入到这个GenericRequest的obtain()方法瞧一瞧： 

```
public final class GenericRequest<A, T, Z, R> implements Request, SizeReadyCallback,
        ResourceCallback {

    ...

    public static <A, T, Z, R> GenericRequest<A, T, Z, R> obtain(
            LoadProvider<A, T, Z, R> loadProvider,
            A model,
            Key signature,
            Context context,
            Priority priority,
            Target<R> target,
            float sizeMultiplier,
            Drawable placeholderDrawable,
            int placeholderResourceId,
            Drawable errorDrawable,
            int errorResourceId,
            Drawable fallbackDrawable,
            int fallbackResourceId,
            RequestListener<? super A, R> requestListener,
            RequestCoordinator requestCoordinator,
            Engine engine,
            Transformation<Z> transformation,
            Class<R> transcodeClass,
            boolean isMemoryCacheable,
            GlideAnimationFactory<R> animationFactory,
            int overrideWidth,
            int overrideHeight,
            DiskCacheStrategy diskCacheStrategy) {
        @SuppressWarnings("unchecked")
        GenericRequest<A, T, Z, R> request = (GenericRequest<A, T, Z, R>) REQUEST_POOL.poll();
        if (request == null) {
      //  new了一个GenericRequest对象，并在最后一行返回
            request = new GenericRequest<A, T, Z, R>();
        }
     //   调用了GenericRequest的init()
        request.init(loadProvider,
                model,
                signature,
                context,
                priority,
                target,
                sizeMultiplier,
                placeholderDrawable,
                placeholderResourceId,
                errorDrawable,
                errorResourceId,
                fallbackDrawable,
                fallbackResourceId,
                requestListener,
                requestCoordinator,
                engine,
                transformation,
                transcodeClass,
                isMemoryCacheable,
                animationFactory,
                overrideWidth,
                overrideHeight,
                diskCacheStrategy);
        return request;
    }

    ...
}
```

​      可以看到，这里在第33行去new了一个GenericRequest对象，并在最后一行返回，也就是说，obtain()方法实际上获得的就是一个GenericRequest对象。另外这里又在第35行调用了GenericRequest的init()，里面主要就是一些赋值的代码，将传入的这些参数赋值到GenericRequest的成员变量当中，我们就不再跟进去看了。

好，那现在解决了构建Request对象的问题，接下来我们看一下这个Request对象又是怎么执行的。回到刚才的into()方法，你会发现在第18行调用了requestTracker.runRequest()方法来去执行这个Request.

###2.8 Request对象的执行

####2.8.1 requestTracker.runRequest()

那么我们跟进去瞧一瞧，如下所示：

```
/**
 * Starts tracking the given request.
 */
public void runRequest(Request request) {
    requests.add(request);
    if (!isPaused) {
        request.begin();
    } else {
        pendingRequests.add(request);
    }
}
```

这里有一个简单的逻辑判断，就是先判断Glide当前是不是处理暂停状态，如果不是暂停状态就调用Request的begin()方法来执行Request，否则的话就先将Request添加到待执行队列里面，等暂停状态解除了之后再执行。

暂停请求的功能仍然不是这篇文章所关心的，这里就直接忽略了，我们重点来看这个begin()方法。

####2.8.2 GenericRequest.begin()

由于当前的Request对象是一个GenericRequest，因此这里就需要看GenericRequest中的begin()方法了，如下所示：

```
@Override
public void begin() {
    startTime = LogTime.getLogTime();
    if (model == null) {
        onException(null);
        return;
    }
    status = Status.WAITING_FOR_SIZE;
    if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
        onSizeReady(overrideWidth, overrideHeight);
    } else {
        target.getSize(this);
    }
    if (!isComplete() && !isFailed() && canNotifyStatusChanged()) {
        target.onLoadStarted(getPlaceholderDrawable());
    }
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
        logV("finished run method in " + LogTime.getElapsedMillis(startTime));
    }
}
```

这里我们来注意几个细节，首先如果model等于null，model也就是我们在第二步load()方法中传入的图片URL地址，这个时候会调用onException()方法。如果你跟到onException()方法里面去看看，你会发现它最终会调用到一个setErrorPlaceholder()当中

####2.8.3 异常占位图被使用

#####2.8.3.1 setErrorPlaceholder()

如下所示： 

```
private void setErrorPlaceholder(Exception e) {
    if (!canNotifyStatusChanged()) {
        return;
    }
    Drawable error = model == null ? getFallbackDrawable() : null;
    if (error == null) {
      error = getErrorDrawable();
    }
    if (error == null) {
        error = getPlaceholderDrawable();
    }
    target.onLoadFailed(e, error);
}
```

这个方法中会先去获取一个error的占位图，如果获取不到的话会再去获取一个loading占位图，然后调用target.onLoadFailed()方法并将占位图传入。那么onLoadFailed()方法中做了什么呢？

#####2.8.3.2 onLoadFailed()

我们看一下： 

```
public abstract class ImageViewTarget<Z> extends ViewTarget<ImageView, Z> implements GlideAnimation.ViewAdapter {

    ...

    @Override
    public void onLoadStarted(Drawable placeholder) {
    //在图片请求开始之前，会先使用这张占位图代替最终的图片显示
        view.setImageDrawable(placeholder);
    }

    @Override
    public void onLoadFailed(Exception e, Drawable errorDrawable) {
    //error占位图显示到ImageView上
        view.setImageDrawable(errorDrawable);
    }

    ...
}
```

很简单，其实就是将这张error占位图显示到ImageView上而已，因为现在出现了异常，没办法展示正常的图片了。而如果你仔细看下刚才begin()方法的第15行，你会发现它又调用了一个target.onLoadStarted()方法，并传入了一个loading占位图，在也就说，在图片请求开始之前，会先使用这张占位图代替最终的图片显示。这也是我们在上一篇文章中学过的placeholder()和error()这两个占位图API底层的实现原理。

好，那么我们继续回到begin()方法。刚才讲了占位图的实现，

####2.8.4  那么具体的图片加载又是从哪里开始的呢？

​        是在begin()方法的第10行和第12行。这里要分两种情况，一种是你使用了override() API为图片指定了一个固定的宽高，一种是没有指定。如果指定了的话，就会执行第10行代码，调用onSizeReady()方法。如果没指定的话，就会执行第12行代码，调用target.getSize()方法。这个target.getSize()方法的内部会根据ImageView的layout_width和layout_height值做一系列的计算，来算出图片应该的宽高。具体的计算细节我就不带着大家分析了，总之在计算完之后，它也会调用onSizeReady()方法。也就是说，不管是哪种情况，最终都会调用到onSizeReady()方法，在这里进行下一步操作。那么我们跟到这个方法里面来：







我们再来看一下into(glide.buildImageViewTarget(view, transcodeClass))中的buildImageViewTarget

```
//通过imageViewTargetFactory这个工厂来构建我们的target
<R> Target<R> buildImageViewTarget(ImageView imageView, Class<R> transcodedClass) {
        return imageViewTargetFactory.buildTarget(imageView, transcodedClass);
    }
 
 //我们看一下buildeTarget是怎么做的
public class ImageViewTargetFactory {

    @SuppressWarnings("unchecked")
    public <Z> Target<Z> buildTarget(ImageView view, Class<Z> clazz) {
        if (GlideDrawable.class.isAssignableFrom(clazz)) {
            return (Target<Z>) new GlideDrawableImageViewTarget(view);
        } else if (Bitmap.class.equals(clazz)) {
        //如果调用的是asBitmap()这个方法就返回BitmapImageViewTarget,如果没有调用asBitmp()这个方法，
        //就返回GlideDrawableImageVeiwTarget的对象
            return (Target<Z>) new BitmapImageViewTarget(view);
        } else if (Drawable.class.isAssignableFrom(clazz)) {
            return (Target<Z>) new DrawableImageViewTarget(view);
        } else {
            throw new IllegalArgumentException("Unhandled class: " + clazz
                    + ", try .as*(Class).transcode(ResourceTranscoder)");
        }
    }
}
buildTRarget这个方法很简单，就是传入两个参数，主要是根据Class<Z>class,来判断我们到底是加载哪一张图片

```



到这里，Targer差不多就常见完成了



####2.4.2 request建立和begin方法  

```
从DrawableRequestBuilder 中的into()
@Override
    public Target<GlideDrawable> into(ImageView view) {
        return super.into(view);
    }
    方法中，跳转过来的
    GenericRequestBuilder.class
public <Y extends Target<TranscodeType>> Y into(Y target) {
//对我们是不是主线程做一个判断，图片显示是在UI线程中执行的
        Util.assertMainThread();
        if (target == null) {
            throw new IllegalArgumentException("You must pass in a non null Target");
        }
        if (!isModelSet) {
            throw new IllegalArgumentException("You must first set a model (try #load())");
        }
//获取当前target对象所绑定的Request对象，就是前一个(旧的)Request,只要将旧的Request删除了，才能绑定新
//的Request, 才能进行新的操作.
        Request previous = target.getRequest();

        if (previous != null) {
            previous.clear();
            //将这个Request这个动作暂停
            requestTracker.removeRequest(previous);
            previous.recycle();
        }
        //创建新的Request
        Request request = buildRequest(target);
        //进行相应的绑定
        target.setRequest(request);
        lifecycle.addListener(target);
        requestTracker.runRequest(request);

        return target;
    }
```

这个方法主要做了三件事情：

1、创建一个我们需要的加载图片的Reuqst,

2、创建好request之前，它需要对我们旧的Target绑定的Request进行删除，然后再另一个Target绑定到Request

3、发送我们的request，交给我们的requestTracker，这个requestTracker有管理跟踪进行相应的request处理。









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

### 3.5 指定图片的大小 override

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

### 3.6 bitmapTransform

###3.7 指定优先级priority

```
.priority(Priority.HIGH) //指定优先级
```

### 3.8 磁盘缓存 diskCacheStrategy

####3.8.1 跳过磁盘缓存 NONE

```
.diskCacheStrategy(DiskCacheStrategy.NONE)//跳过磁盘缓存
```

#### 3.8.2 缓存原来的全分辨率的图像 SOURCE

```
.diskCacheStrategy(DiskCacheStrategy.SOURCE)//仅仅只缓存原来的全分辨率的图像
```

#### 3.8.3 缓存最终的图像 RESULT

```
.diskCacheStrategy(DiskCacheStrategy.RESULT)//仅仅缓存最终的图像
```

#### 3.8.4 缓存所有版本的图像 ALL

```
.diskCacheStrategy(DiskCacheStrategy.ALL)//缓存所有版本的图像
```

### 3.9 图片缩放的类型

####3.9.1 fitCenter

```
.fitCenter()//指定图片缩放的类型
```

####3.9.2 centerCrop

```
.centerCrop() //指定图片缩放的类型 
```

### 3.10 内存缓存 skipMemoryCache

```
.skipMemoryCache(true)//跳过内存缓存
```



##四、试题

### 3.1 bitmap&oom&优化bitmap、

### 3.2 三级缓存&lrucache 
















