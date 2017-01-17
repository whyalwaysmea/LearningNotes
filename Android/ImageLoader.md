## ImageLoader简介
一个ImageLoader应该具备如下功能：   
* 图片的同步加载
* 图片的异步加载
* 图片压缩
* 内存缓存
* 磁盘缓存
* 网络拉取

## 图片压缩功能的实现
图片压缩在[Bitmap](https://github.com/whyalwaysmea/LearningNotes/blob/master/Android/Bitmap.md)中已经讲过了，主要是应该BitmapFactory.Options参数设置来进行压缩。当然还有其他的压缩算法，以及压缩库。 比如[Luban](https://github.com/Curzibn/Luban)   
但是这里就以讲过Bitmap压缩为例进行编写。
```java
public class ImageResizer {
    private static final String TAG = "ImageResizer";

    public ImageResizer() {
    }

    /**
     * @param res
     * @param resId      图片资源id
     * @param reqWidth   图片需要的宽
     * @param reqHeight  图片需要的高
     * @return
     */
    public Bitmap decodeSampledBitmapFromResource(Resources res,
                                                  int resId, int reqWidth, int reqHeight) {
        // First decode with inJustDecodeBounds=true to check dimensions
        // 1. 先将options.inJustDecodeBounds设置为true
        final BitmapFactory.Options options = new BitmapFactory.Options();
        options.inJustDecodeBounds = true;
        BitmapFactory.decodeResource(res, resId, options);

        // Calculate inSampleSize
        // 2.计算采样率
        options.inSampleSize = calculateInSampleSize(options, reqWidth,
                reqHeight);

        // Decode bitmap with inSampleSize set
        // 3. options.inJustDecodeBounds = false;
        options.inJustDecodeBounds = false;
        return BitmapFactory.decodeResource(res, resId, options);
    }

    public Bitmap decodeSampledBitmapFromFileDescriptor(FileDescriptor fd, int reqWidth, int reqHeight) {
        // First decode with inJustDecodeBounds=true to check dimensions
        final BitmapFactory.Options options = new BitmapFactory.Options();
        options.inJustDecodeBounds = true;
        BitmapFactory.decodeFileDescriptor(fd, null, options);

        // Calculate inSampleSize
        options.inSampleSize = calculateInSampleSize(options, reqWidth,
                reqHeight);

        // Decode bitmap with inSampleSize set
        options.inJustDecodeBounds = false;
        return BitmapFactory.decodeFileDescriptor(fd, null, options);
    }

    /**
     * 计算压缩采样率
     */
    public int calculateInSampleSize(BitmapFactory.Options options,
                                     int reqWidth, int reqHeight) {
        if (reqWidth == 0 || reqHeight == 0) {
            return 1;
        }

        // Raw height and width of image
        // 图片原本的宽/高
        final int height = options.outHeight;
        final int width = options.outWidth;
        Log.d(TAG, "origin, w= " + width + " h=" + height);
        int inSampleSize = 1;

        if (height > reqHeight || width > reqWidth) {
            final int halfHeight = height / 2;
            final int halfWidth = width / 2;

            // Calculate the largest inSampleSize value that is a power of 2 and
            // keeps both
            // height and width larger than the requested height and width.
            while ((halfHeight / inSampleSize) >= reqHeight
                    && (halfWidth / inSampleSize) >= reqWidth) {
                inSampleSize *= 2;
            }
        }

        Log.d(TAG, "sampleSize:" + inSampleSize);
        return inSampleSize;
    }
}
```

## 内存缓存和磁盘缓存实现
这里选择使用LruCache和DiskLruCache来分别完成内存缓存和磁盘缓存的工作。在ImageLoader初始化时，创建LruCache和DiskLruCache。
```java
private LruCache<String, Bitmap> mMemoryCache;
private DiskLruCache mDiskLruCache;

private ImageLoader(Context context) {
    mContext = context.getApplicationContext();
    // 内存缓存可用容量为，进程可用内存的1/8
    int maxMemory = (int) (Runtime.getRuntime().maxMemory() / 1024);
    int cacheSize = maxMemory / 8;
    mMemoryCache = new LruCache<String, Bitmap>(cacheSize) {
        @Override
        protected int sizeOf(String key, Bitmap bitmap) {
            return bitmap.getRowBytes() * bitmap.getHeight() / 1024;
        }
    };
    // 磁盘缓存
    // 磁盘缓存文件夹路径
    File diskCacheDir = getDiskCacheDir(mContext, "bitmap");
    if (!diskCacheDir.exists()) {
        diskCacheDir.mkdirs();
    }
    // 磁盘缓存可用容量为 DISK_CACHE_SIZE = 1024 * 1024 * 50 = 50MB    
    // 先判断剩余空间是否够50MB
    if (getUsableSpace(diskCacheDir) > DISK_CACHE_SIZE) {
        try {
            mDiskLruCache = DiskLruCache.open(diskCacheDir, 1, 1,
                    DISK_CACHE_SIZE);
            mIsDiskLruCacheCreated = true;
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```
初始化完成之后，就需要提供方法来完成缓存的添加和获取功能。   
这里先完成内存缓存的添加和获取功能：  
```java
private void addBitmapToMemoryCache(String key, Bitmap bitmap) {
    if (getBitmapFromMemCache(key) == null) {
        mMemoryCache.put(key, bitmap);
    }
}

private Bitmap getBitmapFromMemCache(String key) {
    return mMemoryCache.get(key);
}
```
磁盘缓存相对于内存缓存的操作就要稍微复杂一点了，具体的方法可以参考[DiskLruCache](https://github.com/whyalwaysmea/LearningNotes/blob/master/Android/Bitmap.md)。这里我们直接来看代码:
```java
private Bitmap loadBitmapFromHttp(String url, int reqWidth, int reqHeight)
            throws IOException {
    if (Looper.myLooper() == Looper.getMainLooper()) {
        throw new RuntimeException("can not visit network from UI Thread.");
    }
    if (mDiskLruCache == null) {
        return null;
    }
    // 对url进行MD5 的转换
    String key = hashKeyFormUrl(url);
    // 如果这个key对应的对象正在被编辑，那么editor就返回null
    DiskLruCache.Editor editor = mDiskLruCache.edit(key);
    if (editor != null) {
        // 因为创建DiskLruCache的时候，第三个参数传的是1，也就是一个key对应一个文件
        // 所以DISK_CACHE_INDEX传0就可以了
        OutputStream outputStream = editor.newOutputStream(DISK_CACHE_INDEX);
        if (downloadUrlToStream(url, outputStream)) {
            editor.commit();
        } else {
            editor.abort();
        }
        mDiskLruCache.flush();
    }
    return loadBitmapFromDiskCache(url, reqWidth, reqHeight);
}

private Bitmap loadBitmapFromDiskCache(String url, int reqWidth,
                                       int reqHeight) throws IOException {
    if (Looper.myLooper() == Looper.getMainLooper()) {
        Log.w(TAG, "load bitmap from UI Thread, it's not recommended!");
    }
    if (mDiskLruCache == null) {
        return null;
    }
    // 缓存的查找
    Bitmap bitmap = null;
    String key = hashKeyFormUrl(url);
    DiskLruCache.Snapshot snapShot = mDiskLruCache.get(key);
    if (snapShot != null) {
        FileInputStream fileInputStream = (FileInputStream)snapShot.getInputStream(DISK_CACHE_INDEX);
        FileDescriptor fileDescriptor = fileInputStream.getFD();
        bitmap = mImageResizer.decodeSampledBitmapFromFileDescriptor(fileDescriptor,
                reqWidth, reqHeight);
        if (bitmap != null) {
            // 添加进内存缓存中
            addBitmapToMemoryCache(key, bitmap);
        }
    }

    return bitmap;
}
```

## 同步加载和异步加载接口的设计
同步加载需要外部在线程中调用，因为同步加载很可能比较耗时。
```java
/**
 * load bitmap from memory cache or disk cache or network.
 * @param uri http url
 * @param reqWidth the width ImageView desired
 * @param reqHeight the height ImageView desired
 * @return bitmap, maybe null.
 */
public Bitmap loadBitmap(String uri, int reqWidth, int reqHeight) {
    // 先从内存缓存中取
    Bitmap bitmap = loadBitmapFromMemCache(uri);
    if (bitmap != null) {
        Log.d(TAG, "loadBitmapFromMemCache,url:" + uri);
        return bitmap;
    }

    try {
        // 然后从磁盘缓存中取
        bitmap = loadBitmapFromDiskCache(uri, reqWidth, reqHeight);
        if (bitmap != null) {
            Log.d(TAG, "loadBitmapFromDisk,url:" + uri);
            return bitmap;
        }
        // 最后从网络中取，并缓存
        bitmap = loadBitmapFromHttp(uri, reqWidth, reqHeight);
        Log.d(TAG, "loadBitmapFromHttp,url:" + uri);
    } catch (IOException e) {
        e.printStackTrace();
    }
    // 最最后，单纯的从网络中取，而且不缓存
    if (bitmap == null && !mIsDiskLruCacheCreated) {
        Log.w(TAG, "encounter error, DiskLruCache is not created.");
        bitmap = downloadBitmapFromUrl(uri);
    }

    return bitmap;
}
```
因为是同步加载，所以在loadBitmapFromHttp中进行了线程的判断。如果不是主线程则抛出异常，
```java
if (Looper.myLooper() == Looper.getMainLooper()) {
    Log.w(TAG, "load bitmap from UI Thread, it's not recommended!");
}
```
接下来是异步加载
```java
public void bindBitmap(final String uri, final ImageView imageView,
                           final int reqWidth, final int reqHeight) {
    imageView.setTag(TAG_KEY_URI, uri);
    // 先在内存中读取缓存
    Bitmap bitmap = loadBitmapFromMemCache(uri);
    if (bitmap != null) {
        imageView.setImageBitmap(bitmap);
        return;
    }
    // 如果内存中没有缓存，那么在线程池中调用loadBitmap
    Runnable loadBitmapTask = new Runnable() {

        @Override
        public void run() {
            // 加载bitmap
            Bitmap bitmap = loadBitmap(uri, reqWidth, reqHeight);
            if (bitmap != null) {
                // 将imageview，url，以及bitmap封装起来
                LoaderResult result = new LoaderResult(imageView, uri, bitmap);
                // 通过handler发送到主线程，在主线程中给imageview设置图片
                mMainHandler.obtainMessage(MESSAGE_POST_RESULT, result).sendToTarget();
            }
        }
    };
    THREAD_POOL_EXECUTOR.execute(loadBitmapTask);
}
```
bindBitmap中用到了线程池和Handler。这里先看看线程池的实现，不了解线程池的可以先看看这里 [线程和线程池](https://github.com/whyalwaysmea/LearningNotes/blob/master/Android/Android%E7%9A%84%E7%BA%BF%E7%A8%8B%E5%92%8C%E7%BA%BF%E7%A8%8B%E6%B1%A0.md)  
```java
private static final int CORE_POOL_SIZE = CPU_COUNT + 1;
private static final int MAXIMUM_POOL_SIZE = CPU_COUNT * 2 + 1;
private static final long KEEP_ALIVE = 10L;

private static final ThreadFactory sThreadFactory = new ThreadFactory() {
    private final AtomicInteger mCount = new AtomicInteger(1);

    public Thread newThread(Runnable r) {
        return new Thread(r, "ImageLoader#" + mCount.getAndIncrement());
    }
};

public static final Executor THREAD_POOL_EXECUTOR = new ThreadPoolExecutor(
            CORE_POOL_SIZE, MAXIMUM_POOL_SIZE,
            KEEP_ALIVE, TimeUnit.SECONDS,
            new LinkedBlockingQueue<Runnable>(), sThreadFactory);
```
这里说一下为什么使用线程池。首先肯定是不能用普通的线程的，因为如果一下加载大量图片的话，那么产生大量的线程，导致整体效率降低；其次不使用AsyncTask是因为在3.0以上的版本，AsyncTask实现并发效果不太自然，同时AsyncTask也有自己的缺陷。   
再来看看Handler在这里的使用
```java
private Handler mMainHandler = new Handler(Looper.getMainLooper()) {
    @Override
    public void handleMessage(Message msg) {
        LoaderResult result = (LoaderResult) msg.obj;
        ImageView imageView = result.imageView;
        String uri = (String) imageView.getTag(TAG_KEY_URI);
        if (uri.equals(result.uri)) {
            imageView.setImageBitmap(result.bitmap);
        } else {
            Log.w(TAG, "set image bitmap,but url has changed, ignored!");
        }
    };
};
```   
可以看出来，Handler是采用主线程的Looper来构造的，同时为了解决View复用的问题，这里进行了url的二次判断。
