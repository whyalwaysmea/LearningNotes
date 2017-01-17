## Glide的简单使用
Glide是一个优秀的图片加载工具库。它可以支持多种图片数据源，在对图片加载并显示时，能较好的处理好缓存、保持较低的内存占用。目前已经被Google用于其官方应用中。  
Glide的使用还是很简单的，但是由于它的功能强大，也有很多可以自定义的地方，这些就需要通过参考其他文章来进行学习了，这里就只介绍简单的使用。
```java
Glide.with(context).load(url). placeholder(R.drawable.placeholder).into(imageView)。
```
很简单的，就可以进行图片的加载了。
我们就先从简单的入手，一步一步的分析Glide。

## Glide.with(context)
```java
/**
* Activity
* FragmentActivity
* android.app.Fragment
* Fragment
* Context
*/
public static RequestManager with(FragmentActivity activity) {
    RequestManagerRetriever retriever = RequestManagerRetriever.get();
    return retriever.get(activity);
}
```
Glide有五个静态的重载方法with()，其内部都通过RequestManagerRetriever相应的get重载方法获取一个RequestManager对象。  
RequestManagerRetriever提供各种重载方法的好处就是可以将Glide的加载请求与Activity/Fragment的生命周期绑定而自动执行请求，暂停操作。  

接下来我们拿Activity参数分析Glide请求如何和绑定生命周期自动请求，暂停，以及销毁。
```java
public RequestManager get(Activity activity) {
    if (Util.isOnBackgroundThread() || Build.VERSION.SDK_INT < Build.VERSION_CODES.HONEYCOMB) {
        return get(activity.getApplicationContext());
    } else {
        //判断activity是否已经是销毁状态
        assertNotDestroyed(activity);
        //获取FragmentManager 对象
        android.app.FragmentManager fm = activity.getFragmentManager();
         //创建Fragment，RequestManager并将其绑定
        return fragmentGet(activity, fm);
    }
}

@TargetApi(Build.VERSION_CODES.HONEYCOMB)
RequestManager fragmentGet(Context context, android.app.FragmentManager fm) {
    //*获取RequestManagerFragment，主要利用Frament进行请求的生命周期管理
    RequestManagerFragment current = getRequestManagerFragment(fm);
    RequestManager requestManager = current.getRequestManager();
    //requestManager 为空，即首次加载初始化requestManager ，
    //并调用setRequestManager设置到RequestManagerFragment
    if (requestManager == null) {
        //创建RequestManager传入Lifecycle实现类，如ActivityFragmentLifecycle
        requestManager = new RequestManager(context, current.getLifecycle(), current.getRequestManagerTreeNode());
        current.setRequestManager(requestManager);
    }
    return requestManager;
}

//获取Fragment对象
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

// 这里涉及到了RequestManagerFragment
// 此时我们看到RequestManagerFragment 继承了Fragment.
// 并且在其生命周期onStart(),onStop(),onDestory()，调用了ActivityFragmentLifecycle 相应的方法，
// ActivityFragmentLifecycle实现了Lifecycle 接口，
// 在其中通过addListener（LifecycleListener listener）回调相应（LifecycleListener的 onStart(),onStop(),onDestory()）周期方法。
// LifecycleListener是监听生命周期时间接口。
public class RequestManagerFragment extends Fragment {
    private final ActivityFragmentLifecycle lifecycle;
    //省略部分代码...
    @Override
    public void onStart() {
        super.onStart();
        //关联lifecycle相应onStart方法
        lifecycle.onStart();
    }

    @Override
    public void onStop() {
        super.onStop();
         //关联lifecycle相应onStop方法
        lifecycle.onStop();
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
         //关联lifecycle相应onDestroy方法
        lifecycle.onDestroy();
    }
}
```
通过对`Glide.with(Activity activity)`的分析，可以知道，最后是通过`RequestManager `的构造方法，得到了一个RequestManager对象。  
其他的几个`with`方法其实也大同小异，都是会通过RequestManager的构造方法得到一个RequestManager对象。  
那接下来我们看看RequestManager
```java
/**
 * A class for managing and starting requests for Glide. Can use activity, fragment and connectivity lifecycle events to
 * intelligently stop, start, and restart requests. Retrieve either by instantiating a new object, or to take advantage
 * built in Activity and Fragment lifecycle handling, use the static Glide.load methods with your Fragment or Activity.
 */
public class RequestManager implements LifecycleListener {
  //An interface for listening to Activity/Fragment lifecycle events.
    private final Lifecycle lifecycle;
 public RequestManager(Context context, Lifecycle lifecycle, RequestManagerTreeNode treeNode) {
        this(context, lifecycle, treeNode, new RequestTracker(), new ConnectivityMonitorFactory());
    }

    RequestManager(Context context, final Lifecycle lifecycle, RequestManagerTreeNode treeNode,
            RequestTracker requestTracker, ConnectivityMonitorFactory factory) {
        this.context = context.getApplicationContext();
        this.lifecycle = lifecycle;
        this.treeNode = treeNode;
        //A class for tracking, canceling, and restarting in progress, completed, and failed requests.
        // 该对象就是跟踪请求取消，重启，完成，失败
        this.requestTracker = requestTracker;
        //通过Glide的静态方法获取Glide实例。单例模式
        this.glide = Glide.get(context);
        this.optionsApplier = new OptionsApplier();

        //通过工厂类ConnectivityMonitorFactory的build方法获取ConnectivityMonitor （一个用于监控网络连接事件的接口）
        ConnectivityMonitor connectivityMonitor = factory.build(context,
                new RequestManagerConnectivityListener(requestTracker));

        // If we're the application level request manager, we may be created on a background thread. In that case we
        // cannot risk synchronously pausing or resuming requests, so we hack around the issue by delaying adding
        // ourselves as a lifecycle listener by posting to the main thread. This should be entirely safe.
        if (Util.isOnBackgroundThread()) {
            new Handler(Looper.getMainLooper()).post(new Runnable() {
                @Override
                public void run() {
                    lifecycle.addListener(RequestManager.this);
                }
            });
        } else {
        //设置监听
            lifecycle.addListener(this);
        }
        lifecycle.addListener(connectivityMonitor);
    }
    /**
     * Lifecycle callback that registers for connectivity events (if the android.permission.ACCESS_NETWORK_STATE
     * permission is present) and restarts failed or paused requests.
     */
    @Override
    public void onStart() {
        // onStart might not be called because this object may be created after the fragment/activity's onStart method.
        resumeRequests();
    }

    /**
     * Lifecycle callback that unregisters for connectivity events (if the android.permission.ACCESS_NETWORK_STATE
     * permission is present) and pauses in progress loads.
     */
    @Override
    public void onStop() {
        pauseRequests();
    }

    /**
     * Lifecycle callback that cancels all in progress requests and clears and recycles resources for all completed
     * requests.
     */
    @Override
    public void onDestroy() {
        requestTracker.clearRequests();
    }
}
```

### Glide.get(Context);
```java
public static Glide get(Context context) {
    if (glide == null) {
        synchronized (Glide.class) {
            if (glide == null) {
                Context applicationContext = context.getApplicationContext();
                // 获取GlideModule集合
                List<GlideModule> modules = new ManifestParser(applicationContext).parse();

                GlideBuilder builder = new GlideBuilder(applicationContext);
                for (GlideModule module : modules) {
                    module.applyOptions(applicationContext, builder);
                }
                glide = builder.createGlide();
                for (GlideModule module : modules) {
                    module.registerComponents(applicationContext, glide);
                }
            }
        }
    }

    return glide;
}
```
Glide.get(Context)通过单例方式获取到了Glide的实例，同时在初始化的时候实现了GlideModule配置功能。  
接着看看这个GlideModule是怎么配置的
```java
// new ManifestParser(applicationContext).parse()
public List<GlideModule> parse() {
    List<GlideModule> modules = new ArrayList<GlideModule>();
    try {
        // 获取metaData信息
        ApplicationInfo appInfo = context.getPackageManager().getApplicationInfo(
                context.getPackageName(), PackageManager.GET_META_DATA);
        if (appInfo.metaData != null) {
            for (String key : appInfo.metaData.keySet()) {
                // 如果metaData的key为GlideModule
                if (GLIDE_MODULE_VALUE.equals(appInfo.metaData.get(key))) {
                    // 将GlideModule添加进入List
                    modules.add(parseModule(key));
                }
            }
        }
    } catch (PackageManager.NameNotFoundException e) {
        throw new RuntimeException("Unable to find metadata to parse GlideModules", e);
    }

    return modules;
}

// 再次获取到我们在AndroidManifest.xml中声明的自定义GlideModule
private static GlideModule parseModule(String className) {
    Class<?> clazz;
    try {
        clazz = Class.forName(className);
    } catch (ClassNotFoundException e) {
        throw new IllegalArgumentException("Unable to find GlideModule implementation", e);
    }

    Object module;
    try {
        module = clazz.newInstance();
    } catch (InstantiationException e) {
        throw new RuntimeException("Unable to instantiate GlideModule implementation for " + clazz, e);
    } catch (IllegalAccessException e) {
        throw new RuntimeException("Unable to instantiate GlideModule implementation for " + clazz, e);
    }

    if (!(module instanceof GlideModule)) {
        throw new RuntimeException("Expected instanceof GlideModule, but found: " + module);
    }
    return (GlideModule) module;
}
```
当获取到我们在AndroidManifest.xml中声明的自定义GlideModule之后，遍历了集合并调用相应的applyOptions和registerComponents方法
同时看到glide对象是通过`builder.createGlide()`获取的
```java
Glide createGlide() {
    if (sourceService == null) {
        final int cores = Math.max(1, Runtime.getRuntime().availableProcessors());
        // 初始化线程池
        sourceService = new FifoPriorityThreadPoolExecutor(cores);
    }
    if (diskCacheService == null) {
        diskCacheService = new FifoPriorityThreadPoolExecutor(1);
    }

    MemorySizeCalculator calculator = new MemorySizeCalculator(context);
     //设置Bitmap池
    if (bitmapPool == null) {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.HONEYCOMB) {
            int size = calculator.getBitmapPoolSize();
            bitmapPool = new LruBitmapPool(size);
        } else {
            bitmapPool = new BitmapPoolAdapter();
        }
    }

    if (memoryCache == null) {
        //内存缓存
        memoryCache = new LruResourceCache(calculator.getMemoryCacheSize());
    }

    if (diskCacheFactory == null) {
        //内部磁盘缓存
        diskCacheFactory = new InternalCacheDiskCacheFactory(context);
    }

    if (engine == null) {
        //初始化引擎类
        engine = new Engine(memoryCache, diskCacheFactory, diskCacheService, sourceService);
    }

    if (decodeFormat == null) {
        decodeFormat = DecodeFormat.DEFAULT;
    }

    return new Glide(engine, memoryCache, bitmapPool, context, decodeFormat);
}
```
可以看到，这里都是对Glide进行的一些初始化操作。
通过上面的分析，你就会理解，为什么之前提到配置信息只需要实现GlideModule接口，重写其中的方法，并再清单文件配置metaData，并且metaData的key是自定义GlideModule的全路径名，value值必须是GlideModule.会明白当我们不想让自定义的GlideModule生效时只需要删除相应的GlideModule。当使用了混淆时为什么要配置...  
>-keep public class * implements com.bumptech.glide.module.GlideModule

## requestManager.load
通过上面的分析，我们知道了，Glide.with()获取到的就是一个requestManager对象。  
接下来继续看Glide.with(context).load(url).. 其实也就是requestManager.load(url);  
这里的url可以是String、Uri、Integer等几种类型。这里我们就以String类型为例进行分析：   
```java
public DrawableTypeRequest<String> load(String string) {
    return (DrawableTypeRequest<String>) fromString().load(string);
}
public DrawableTypeRequest<String> fromString() {
    return loadGeneric(String.class);
}

private <T> DrawableTypeRequest<T> loadGeneric(Class<T> modelClass) {
    // 省略一段代码
    ...
    return optionsApplier.apply(
    // 创建DrawableTypeRequest，它是GenericRequestBuilder的子类
    new DrawableTypeRequest<T>(modelClass, streamModelLoader, fileDescriptorModelLoader, context,
            glide, requestTracker, lifecycle, optionsApplier));
}
```
这里最后是有了一个DrawableTypeRequest对象，我们先了解一下这个DrawableTypeRequest  

![DrawableTypeRequest](https://user-gold-cdn.xitu.io/2016/12/21/6b739c4d3e79a4806acbb765a1d654eb)

所以最终的load方法，其实是调用的`GenericRequestBuilder.load`。对于常用函数placeholder（），error（），transform等设置都是在此设置
```java
public GenericRequestBuilder<ModelType, DataType, ResourceType, TranscodeType> load(ModelType model) {
    this.model = model;
    isModelSet = true;
    return this;
}

public GenericRequestBuilder<ModelType, DataType, ResourceType, TranscodeType> error(
            Drawable drawable) {
    this.errorPlaceholder = drawable;

    return this;
}

```
首先我们通过GenericRequestBuilder的名字就可以看出来，它是一个构造者模式。然后可以发现这些设置方法都是进行变量的赋值。

## into(ImageView)
经过一系列操作后，最终调用into(imageView)方法来完成图片的最终加载
```java
// GenericRequestBuilder.java
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

public <Y extends Target<TranscodeType>> Y into(Y target) {
    // 判断只能在主线程中执行
    Util.assertMainThread();
    if (target == null) {
        throw new IllegalArgumentException("You must pass in a non null Target");
    }
    if (!isModelSet) {
        throw new IllegalArgumentException("You must first set a model (try #load())");
    }
    // 获取Request对象
    Request previous = target.getRequest();

    if (previous != null) {
        previous.clear();
        requestTracker.removeRequest(previous);
        previous.recycle();
    }
     //创建请求对象
    Request request = buildRequest(target);
    target.setRequest(request);
    //将target加入lifecycle    
    lifecycle.addListener(target);
    //执行请求
    requestTracker.runRequest(request);

    return target;
}
```
这里的target我们可以理解成View，,只是Glide对我们的View做了一层封装。  
之后通过buildRequest创建请求对象。
```java
// 创建请求对象
private Request buildRequest(Target<TranscodeType> target) {
    if (priority == null) {
        // 默认加载优先级 NORMAL
        priority = Priority.NORMAL;
    }
    return buildRequestRecursive(target, null);
}

private Request buildRequestRecursive(Target<TranscodeType> target, ThumbnailRequestCoordinator parentCoordinator) {
    if (thumbnailRequestBuilder != null) {
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
        // Base case: no thumbnail.
        return obtainRequest(target, sizeMultiplier, priority, parentCoordinator);
    }
}

private Request obtainRequest(Target<TranscodeType> target, float sizeMultiplier, Priority priority,
            RequestCoordinator requestCoordinator) {
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
// 最终通过GenericRequest.obtain方法创建了
// GenericRequest.java
public static <A, T, Z, R> GenericRequest<A, T, Z, R> obtain(...) {
    @SuppressWarnings("unchecked")
    GenericRequest<A, T, Z, R> request = (GenericRequest<A, T, Z, R>) REQUEST_POOL.poll();
    if (request == null) {
        request = new GenericRequest<A, T, Z, R>();
    }
    //利用设置的参数初始化Request对象
    request.init(...);
    //返回Request对象
    return request;
}
```
至此请求对象创建成功，在通过buildRequest创建请求成功后，使用了target.setRequest(request);  
将请求设置到target,并通过addListener将target加入到lifecycle。  
上面执行了那么多都只是请求创建，请求的执行时通过requestTracker.runRequest(request);开始的。

### 发送请求
```java
/**
* Starts tracking the given request.
*/
public void runRequest(Request request) {
    //添加request对象到集合中
   requests.add(request);
   if (!isPaused) {
       //如果当前状态是非暂停的，调用begin方法发送请求
       request.begin();
   } else {
       //将请求加入到挂起的请求集合
       pendingRequests.add(request);
   }
}
```
每次提交请求都将请求加入了一个set中，用它来管理请求，然后通过request的实现类GenericRequest查看begin方法执行的内容
```java
public void begin() {
    startTime = LogTime.getLogTime();
    if (model == null) {
        //加载错误占位图设置
        onException(null);
        return;
    }

    status = Status.WAITING_FOR_SIZE;
    //验证宽高是否合法
    if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
         //发送请求
        onSizeReady(overrideWidth, overrideHeight);
    } else {
        target.getSize(this);
    }

    if (!isComplete() && !isFailed() && canNotifyStatusChanged()) {
        //加载前默认占位图设置回调
        target.onLoadStarted(getPlaceholderDrawable());
    }
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
        logV("finished run method in " + LogTime.getElapsedMillis(startTime));
    }
}
//获取设置加载开始时占位图片的Drawable 对象
private Drawable getPlaceholderDrawable() {
    if (placeholderDrawable == null && placeholderResourceId > 0) {
        placeholderDrawable = context.getResources().getDrawable(placeholderResourceId);
    }
    return placeholderDrawable;
}

```
上面有一句!isComplete() && !isFailed() && canNotifyStatusChanged()判断，
如果都为真会回调target.onLoadStarted(getPlaceholderDrawable());  
我们可以看到Target的实现类ImageViewTarget中onLoadStarted的回调执行语句  
```java
// ImageViewTarget.java
//给ImageView设置Drawable
public void onLoadStarted(Drawable placeholder) {
    view.setImageDrawable(placeholder);
}
```
接下来看看发送请求的
```java
public void onSizeReady(int width, int height) {
    //省略部分代码
    ...

    status = Status.RUNNING;//将请求状态更新为运行状态
    //省略部分代码
    ...

    // 进入Engine的入口，请求执行的核心方法
    loadStatus = engine.load(signature, width, height, dataFetcher, loadProvider, transformation, transcoder,
            priority, isMemoryCacheable, diskCacheStrategy, this);
    loadedFromMemoryCache = resource != null;
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
        logV("finished onSizeReady in " + LogTime.getElapsedMillis(startTime));
    }
}
```
Engine类封装了数据获取的重要入口方法，向request层提供这些API，比如load(), release(), clearDiskCache()等方法
```java
// Engine.java
public <T, Z, R> LoadStatus load(Key signature, int width, int height, DataFetcher<T> fetcher,
            DataLoadProvider<T, Z> loadProvider, Transformation<Z> transformation, ResourceTranscoder<Z, R> transcoder,
            Priority priority, boolean isMemoryCacheable, DiskCacheStrategy diskCacheStrategy, ResourceCallback cb) {
    //断言是否在主线程
    Util.assertMainThread();
    long startTime = LogTime.getLogTime();

    final String id = fetcher.getId();
    //创建Enginekey
    EngineKey key = keyFactory.buildKey(id, signature, width, height, loadProvider.getCacheDecoder(),
            loadProvider.getSourceDecoder(), transformation, loadProvider.getEncoder(),
            transcoder, loadProvider.getSourceEncoder());
    //从缓存加载图片
    EngineResource<?> cached = loadFromCache(key, isMemoryCacheable);
    if (cached != null) {
        // 获取数据成功，会回调target的onResourceReady()
        cb.onResourceReady(cached);
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            logWithTimeAndKey("Loaded resource from cache", startTime, key);
        }
        return null;
    }
    // 尝试从活动Resources 中获取，它表示的是当前正在使用的Resources，与内存缓存不同之处是clear缓存时不会clear它。
    EngineResource<?> active = loadFromActiveResources(key, isMemoryCacheable);
    if (active != null) {
       //获取成功回调
        cb.onResourceReady(active);
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            logWithTimeAndKey("Loaded resource from active resources", startTime, key);
        }
        return null;
    }

    EngineJob current = jobs.get(key);
    //判断jobs中是否已经存在任务，如果存在说明任务之前已经提交了
    if (current != null) {
        current.addCallback(cb);
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            logWithTimeAndKey("Added to existing load", startTime, key);
        }
        return new LoadStatus(cb, current);
    }

    //缓存没有获取到，创建EngineJob 对象
    EngineJob engineJob = engineJobFactory.build(key, isMemoryCacheable);
    DecodeJob<T, Z, R> decodeJob = new DecodeJob<T, Z, R>(key, width, height, fetcher, loadProvider, transformation,
            transcoder, diskCacheProvider, diskCacheStrategy, priority);
     //EngineRunnable 是任务执行阶段的入口
    EngineRunnable runnable = new EngineRunnable(engineJob, decodeJob, priority);
    jobs.put(key, engineJob);
    engineJob.addCallback(cb);
    // 开始提交job
    engineJob.start(runnable);

    if (Log.isLoggable(TAG, Log.VERBOSE)) {
        logWithTimeAndKey("Started new load", startTime, key);
    }
    return new LoadStatus(cb, engineJob);
}
```
我们看到先根据调用loadFromCache从内存加载，若返回值为空再次从活动的资源中加载，若再次为空查看jobs是否提交过任务，若没有提交则创建EngineRunnable，并将任务提交到engineJob中。我们先看下EngineJob中的start方法  
```java
// EngineJob
//提交任务，将任务加入到线程池
public void start(EngineRunnable engineRunnable) {
    this.engineRunnable = engineRunnable;
    //提交任务到diskCacheService线程池
    future = diskCacheService.submit(engineRunnable);
}
```
接下来看线程类EngineRunnable的run方法，它是任务执行的入口
```java
// EngineRunnable.java
@Override
public void run() {
    if (isCancelled) {
        return;
    }

    Exception exception = null;
    Resource<?> resource = null;
    try {
        //数据的获取，编解码
        resource = decode();
    } catch (Exception e) {
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            Log.v(TAG, "Exception decoding", e);
        }
        exception = e;
    }
    //如果当前状态是取消，则回收各种资源防止内存泄露
    if (isCancelled) {
        if (resource != null) {
            resource.recycle();
        }
        return;
    }

    if (resource == null) {
        //加载失败回调
        onLoadFailed(exception);
    } else {
         //加载成功回调
        onLoadComplete(resource);
    }
}

private Resource<?> decode() throws Exception {
    if (isDecodingFromCache()) {
        // 从DiskLruCache中获取数据并解码
        return decodeFromCache();
    } else {
        // 从其他途径获取数据并解码，如网络，本地File，数据流等
        return decodeFromSource();
    }
}
```
这里就开始真正的获取数据了：分为从缓存中获取数据和从其他途径获取数据
```java
// 从缓存中获取数据
private Resource<?> decodeFromCache() throws Exception {
    Resource<?> result = null;
    try {
        result = decodeJob.decodeResultFromCache();
    } catch (Exception e) {
        if (Log.isLoggable(TAG, Log.DEBUG)) {
            Log.d(TAG, "Exception decoding result from cache: " + e);
        }
    }

    if (result == null) {
        result = decodeJob.decodeSourceFromCache();
    }
    return result;
}
// DecodeJob.java
public Resource<Z> decodeSourceFromCache() throws Exception {
    if (!diskCacheStrategy.cacheSource()) {
        return null;
    }

    long startTime = LogTime.getLogTime();
    //从DiskCache中获取资源
    Resource<T> decoded = loadFromCache(resultKey.getOriginalKey());
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
        logWithTimeAndKey("Decoded source from cache", startTime);
    }
    return transformEncodeAndTranscode(decoded);
}
//从DiskCache中获取资源
private Resource<T> loadFromCache(Key key) throws IOException {
    //根据key从DiskCache获取文件
    File cacheFile = diskCacheProvider.getDiskCache().get(key);
    if (cacheFile == null) {
        return null;
    }

    Resource<T> result = null;
    try {
        result = loadProvider.getCacheDecoder().decode(cacheFile, width, height);
    } finally {
        if (result == null) {
            diskCacheProvider.getDiskCache().delete(key);
        }
    }
    return result;
}
```
```java
// 从其他途径获取数据
// 调用decodeJob来完成数据获取和编解码
private Resource<?> decodeFromSource() throws Exception {
    return decodeJob.decodeFromSource();
}
public Resource<Z> decodeFromSource() throws Exception {
    // 获取数据，解码
    Resource<T> decoded = decodeSource();
    //编码并保存
    return transformEncodeAndTranscode(decoded);
}
// 获取数据，解码
private Resource<T> decodeSource() throws Exception {
    Resource<T> decoded = null;
    try {
        long startTime = LogTime.getLogTime();
        //数据拉取
        final A data = fetcher.loadData(priority);
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            logWithTimeAndKey("Fetched data", startTime);
        }
        if (isCancelled) {
            return null;
        }
        //编码
        decoded = decodeFromSourceData(data);
    } finally {
        fetcher.cleanup();
    }
    return decoded;
}
```
在数据获取时先调用DataFetcher的loadData()拉取数据，  
对于DataFetcher的实现类有好几个，我们拿从url拉取数据为例，也就是HttpUrlFetcher类
```java
// HttpUrlFetcher.java
public InputStream loadData(Priority priority) throws Exception {
    return loadDataWithRedirects(glideUrl.toURL(), 0 /*redirects*/, null /*lastUrl*/, glideUrl.getHeaders());
}
//返回InputStream 对象
private InputStream loadDataWithRedirects(URL url, int redirects, URL lastUrl, Map<String, String> headers)
        throws IOException {
    if (redirects >= MAXIMUM_REDIRECTS) {
        throw new IOException("Too many (> " + MAXIMUM_REDIRECTS + ") redirects!");
    } else {
        // Comparing the URLs using .equals performs additional network I/O and is generally broken.
        // See http://michaelscharf.blogspot.com/2006/11/javaneturlequals-and-hashcode-make.html.
        try {
            if (lastUrl != null && url.toURI().equals(lastUrl.toURI())) {
                throw new IOException("In re-direct loop");
            }
        } catch (URISyntaxException e) {
            // Do nothing, this is best effort.
        }
    }
    // 静态工厂模式创建HttpURLConnection对象
    urlConnection = connectionFactory.build(url);
    for (Map.Entry<String, String> headerEntry : headers.entrySet()) {
      urlConnection.addRequestProperty(headerEntry.getKey(), headerEntry.getValue());
    }
    //设置请求参数
    //设置连接超时时间2500ms
    urlConnection.setConnectTimeout(2500);
    //设置读取超时时间2500ms
    urlConnection.setReadTimeout(2500);
    //不使用http缓存
    urlConnection.setUseCaches(false);
    urlConnection.setDoInput(true);

    // Connect explicitly to avoid errors in decoders if connection fails.
    urlConnection.connect();
    if (isCancelled) {
        return null;
    }
    final int statusCode = urlConnection.getResponseCode();
    if (statusCode / 100 == 2) {
         //请求成功
        return getStreamForSuccessfulRequest(urlConnection);
    } else if (statusCode / 100 == 3) {
        String redirectUrlString = urlConnection.getHeaderField("Location");
        if (TextUtils.isEmpty(redirectUrlString)) {
            throw new IOException("Received empty or null redirect url");
        }
        URL redirectUrl = new URL(url, redirectUrlString);
        return loadDataWithRedirects(redirectUrl, redirects + 1, url, headers);
    } else {
        if (statusCode == -1) {
            throw new IOException("Unable to retrieve response code from HttpUrlConnection.");
        }
        throw new IOException("Request failed " + statusCode + ": " + urlConnection.getResponseMessage());
    }
}
```
看到这终于看到了网络加载请求，我们也可以自定义DataFetcher，从而使用其他网络库，如OkHttp，Volley.    
最后我们再看下transformEncodeAndTranscode方法
```java
private Resource<Z> transformEncodeAndTranscode(Resource<T> decoded) {
    long startTime = LogTime.getLogTime();
    // 根据ImageView的scaleType等参数计算真正被ImageView使用的图片宽高，并保存真正宽高的图片。
    Resource<T> transformed = transform(decoded);
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
        logWithTimeAndKey("Transformed resource from source", startTime);
    }
    // 写入到DiskLruCache中，下次就可以直接从DiskLruCache获取使用
    writeTransformedToCache(transformed);

    startTime = LogTime.getLogTime();
    // 转码，将源图片转码为ImageView所需的图片格式
    Resource<Z> result = transcode(transformed);
    if (Log.isLoggable(TAG, Log.VERBOSE)) {
        logWithTimeAndKey("Transcoded transformed from source", startTime);
    }
    return result;
}
```
到此图片加载的基本流程就走了一遍了。

## 相关链接

[详谈高大上的图片加载框架Glide](https://gold.xitu.io/post/585a091a1b69e6006cb79079)

[Glide入门教程](http://www.jianshu.com/p/7610bdbbad17)
