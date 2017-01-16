## Bitmap介绍
Bitmap在Android中指的是一张图片，可以是png格式也可以是jpg等其他常见的图片格式。

那么如何加载一个图片呢？     
BitmapFactory类提供了四类方法：decodeFile、decodeResource、decodeStream、decodeByteArray。   
分别用于支持从文件系统、资源、输入流以及字节数组中加载出一个Bitmap对象。 其中decodeFile和decodeResource又间接调用了decodeStream方法。  


## Bitmap高效地加载
高效加载Bitmap的核心思想是：采用BitmapFactory.Options来加载所需尺寸的图片。  
有时候用ImageView显示图片，根本不用显示原图的大小，但是它依然是把整个原图都加载了，所以通过BitmapFactory.Options就可以按一定的采样率来加载缩小后的图片。  
通过BitmapFactory.Options来缩放图片，主要是用到了它的inSampleSize参数，即采样率。  
当inSampleSize为1时，采样后的图片大小为图片的原始大小；当inSampleSize大于1时，比如2，那么采样后的图片宽/高均为原图大小的1/2，那么像素就为原图的1/4，其占有内存大小也为原图的1/4。  
可以发现采样率inSampleSize必须是大于1的整数图片才会有缩小的效果，并且采样率同时作用于宽/高，这将导致缩放后的图片大小以采样率的2次方形式递减，即缩放比例为1/(inSampleSize的2次方)，比如inSampleSize为4，那么缩放比例就是1/16。同时，inSampleSize的取值应该总是2的指数，比如:1、2、4、8、16...  

获取采样率的流程:
```java
public Bitmap decodeSampledBitmapFromResource(Resources res, int resId, int reqWidth, int reqHeight) {
    // First decode with inJustDecodeBounds=true to check dimensions
    // 1. 将inJustDecodeBounds参数设为true，然后加载图片
    final BitmapFactory.Options options = new BitmapFactory.Options();
    options.inJustDecodeBounds = true;
    BitmapFactory.decodeResource(res, resId, options);

    // Calculate inSampleSize
    // 3. 计算出采样率
    options.inSampleSize = calculateInSampleSize(options, reqWidth, reqHeight);

    // Decode bitmap with inSampleSize set
    // 4.将inJustDecodeBounds设置为false，然后重新加载图片
    options.inJustDecodeBounds = false;
    return BitmapFactory.decodeResource(res, resId, options);
}

public int calculateInSampleSize(BitmapFactory.Options options, int reqWidth, int reqHeight) {
    if (reqWidth == 0 || reqHeight == 0) {
        return 1;
    }

    // Raw height and width of image
    // 2. 取出图片的原始宽高信息
    final int height = options.outHeight;
    final int width = options.outWidth;
    Log.d(TAG, "origin, w= " + width + " h=" + height);
    int inSampleSize = 1;

    if (height > reqHeight || width > reqWidth) {
        final int halfHeight = height / 2;
        final int halfWidth = width / 2;

        // Calculate the largest inSampleSize value that is a power of 2 and
        // keeps both height and width larger than the requested height and width.
        while ((halfHeight / inSampleSize) >= reqHeight && (halfWidth / inSampleSize) >= reqWidth) {
            inSampleSize *= 2;
        }
    }

    Log.d(TAG, "sampleSize:" + inSampleSize);
    return inSampleSize;
}
```
将inJustDecodeBounds设置为true的时候，BitmapFactory只会解析图片的原始宽高信息，并不会真正的加载图片，所以这个操作是轻量级的。  
需要注意的是，这个时候BitmapFactory获取的图片宽高信息和图片的位置以及程序运行的设备有关，这都会导致BitmapFactory获取到不同的结果。


## Android中的缓存策略
常见的图片缓存策略:   
1. 内存
2. 存储设备
3. 网络
先看内存中有没有，如果没有再看存储设备中有没有，如果还是没有再从网络中获取。  
因为从内存中加载图片比从存储设备中加载图片要快，所以先看内存中有没有。   
缓存策略主要包含缓存的添加、获取和删除这三类操作。  

### 缓存算法LRU
LRU(Least Recently Used)，近期最少使用算法。它的核心思想是当缓存满时，会优先淘汰那些近期最少使用的缓存对象。   
采用LRU算法的缓存有两种：LruCache(内存缓存)和DiskLruCache(磁盘缓存)。

#### LruCache
LruCache是Android 3.1才有的，通过support-v4兼容包可以兼容到早期的Android版本。  
LruCache类是一个线程安全的泛型类，它内部采用一个LinkedHashMap以强引用的方式存储外界的缓存对象，其提供了get和put方法来完成缓存的获取和添加操作，当缓存满时，LruCache会移除较早使用的缓存对象，然后再添加新的缓存对象。   
* 强引用：直接的对象引用
* 软引用：当一个对象只有软引用存在时，系统内存不足时此对象会被gc回收
* 弱引用：当一个对象只有弱引用存在时，此对象会随便被gc回收。
LruCache的初始化:
```java
// 获取到可用内存的最大值，使用内存超出这个值会引起OutOfMemory异常。  
// LruCache通过构造函数传入缓存值，以KB为单位。  
int maxMemory = (int)(Runtime.getRuntime().maxMemory() / 1024);
// 总容量为当前进程可用内存的1/8
int cacheSize = maxMemory / 8;
mMemoryCache = new LruCache<String,Bitmap>(cacheSize) {
    protected int sizeOf(String key, Bitmap bitmap) {
        // 重写此方法来衡量每张图片的大小，默认返回图片数量。  
        return bitmap.getByteCount() / 1024;  
    }
}
// 添加一个缓存对象
public void addBitmapToMemoryCache(String key, Bitmap bitmap) {  
    if (getBitmapFromMemCache(key) == null) {  
        mMemoryCache.put(key, bitmap);  
    }  
}  
// 取出一个缓存对象
public Bitmap getBitmapFromMemCache(String key) {  
    return mMemoryCache.get(key);  
}  
```

更详情的LruCache使用方法： [Android高效加载大图、多图解决方案，有效避免程序OOM](http://blog.csdn.net/guolin_blog/article/details/9316683)

#### DiskLruCache
DiskLruCache磁盘缓存，它不属于Android sdk的一部分

**DiskLruCache的创建：**
DiskLruCache并不能通过构造方法来创建，它提供了open方法用于创建自身
```java
/**
*  第一个参数指定的是数据的缓存地址,
*  第二个参数指定当前应用程序的版本号
*  第三个参数指定同一个key可以对应多少个缓存文件，通常是1
*  第四个参数指定最多可以缓存多少字节的数据
*/
public static DiskLruCache open(File directory, int appVersion, int valueCount, long maxSize)  
```
关于缓存路径这里再多讲解一下，通常都会存放在 /sdcard/Android/data/<application package>/cache 这个路径下面，其中application package表示当前应用的包名，当应用被卸载后，此目录会被一并删除。但同时我们又需要考虑如果这个手机没有SD卡，或者SD正好被移除了的情况，因此比较优秀的程序都会专门写一个方法来获取缓存地址，如下所示：
```java
public File getDiskCacheDir(Context context, String uniqueName) {  
    String cachePath;  
    if (Environment.MEDIA_MOUNTED.equals(Environment.getExternalStorageState())  
            || !Environment.isExternalStorageRemovable()) {  
        // 获取到的就是 /sdcard/Android/data/<application package>/cache 这个路径    
        cachePath = context.getExternalCacheDir().getPath();  
    } else {  
        // /data/data/<application package>/cache
        cachePath = context.getCacheDir().getPath();  
    }  
    return new File(cachePath + File.separator + uniqueName);  
}
```

-----

**DiskLruCache缓存添加：**
DiskLruCache的缓存添加是通过Editor完成的，Editor表示一个缓存对象的编辑对象。
这里以图片缓存为例，首先需要获取图片url所对应的key，然后根据key就可以通过edit()来获取Editor对象，如果这个缓存正在被编辑，那么edit()会返回null，即DiskLruCache不允许同时编辑一个缓存对象。
```java
// 将图片的URL进行MD5编码，编码后的字符串肯定是唯一的，并且只会包含0-F这样的字符，完全符合文件的命名规则。
public String hashKeyForUrl(String key) {  
    String cacheKey;  
    try {  
        final MessageDigest mDigest = MessageDigest.getInstance("MD5");  
        mDigest.update(key.getBytes());  
        cacheKey = bytesToHexString(mDigest.digest());  
    } catch (NoSuchAlgorithmException e) {  
        cacheKey = String.valueOf(key.hashCode());  
    }  
    return cacheKey;  
}  

private String bytesToHexString(byte[] bytes) {  
    StringBuilder sb = new StringBuilder();  
    for (int i = 0; i < bytes.length; i++) {  
        String hex = Integer.toHexString(0xFF & bytes[i]);  
        if (hex.length() == 1) {  
            sb.append('0');  
        }  
        sb.append(hex);  
    }  
    return sb.toString();  
}
```
将url转成key之后，就可以获取Editor对象了。
```java
String key = hashKeyForUrl(url);
DiskLruCache.Editor editor = mDiskLruCache.edit(key);
if(edit != null) {
    // 由于前面调用open方法的时候，第三个参数传1，所以这里DISK_CACHE_INDEX设置为0就好
    OutputStream outputStream = editor.newOutputStream(DISK_CACHE_INDEX);
}
```
有了文件输出流，就可以从网络下载图片，图片就可以通过这个文件输出流写入文件系统上:
```java
private boolean downloadUrlToStream(String urlString, OutputStream outputStream) {  
    HttpURLConnection urlConnection = null;  
    BufferedOutputStream out = null;  
    BufferedInputStream in = null;  
    try {  
        final URL url = new URL(urlString);  
        urlConnection = (HttpURLConnection) url.openConnection();  
        in = new BufferedInputStream(urlConnection.getInputStream(), 8 * 1024);  
        out = new BufferedOutputStream(outputStream, 8 * 1024);  
        int b;  
        while ((b = in.read()) != -1) {  
            out.write(b);  
        }  
        return true;  
    } catch (final IOException e) {  
        e.printStackTrace();  
    } finally {  
        if (urlConnection != null) {  
            urlConnection.disconnect();  
        }  
        try {  
            if (out != null) {  
                out.close();  
            }  
            if (in != null) {  
                in.close();  
            }  
        } catch (final IOException e) {  
            e.printStackTrace();  
        }  
    }  
    return false;  
}
```
这段代码相当基础，相信大家都看得懂，就是访问urlString中传入的网址，并通过outputStream写入到本地。但是并没有真正的把图片写入文件系统，还需要通过Editor的commit()来提交写入操作
因此，一次完整写入操作的代码如下所示：
```java
new Thread(new Runnable() {  
    @Override  
    public void run() {  
        try {  
            String imageUrl = "http://img.my.csdn.net/uploads/201309/01/1378037235_7476.jpg";  
            String key = hashKeyForUrl(imageUrl);  
            DiskLruCache.Editor editor = mDiskLruCache.edit(key);  
            if (editor != null) {  
                OutputStream outputStream = editor.newOutputStream(DISK_CACHE_INDEX);  
                if (downloadUrlToStream(imageUrl, outputStream)) {  
                    editor.commit();  
                } else {  
                    editor.abort();  
                }  
            }  
            mDiskLruCache.flush();  
        } catch (IOException e) {  
            e.printStackTrace();  
        }  
    }  
}).start();  
```

-----

**DiskLruCache的缓存查找：**
缓存查找过程也需要先将url转换成key，然后通过DiskLruCache的get方法得到一个Snapshot对象，接着再通过Snapshot对象即可得到缓存的文件输入流，有了文件输入流，自然就可以得到Bitmap对象了。
```java
try {  
    String imageUrl = "http://img.my.csdn.net/uploads/201309/01/1378037235_7476.jpg";  
    String key = hashKeyForUrl(imageUrl);  
    DiskLruCache.Snapshot snapShot = mDiskLruCache.get(key);  
    if (snapShot != null) {  
        InputStream is = snapShot.getInputStream(0);  
        Bitmap bitmap = BitmapFactory.decodeStream(is);  
        mImage.setImageBitmap(bitmap);  
    }  
} catch (IOException e) {  
    e.printStackTrace();  
}
```

更多DiskLruCache的使用方法可以查看 [Android DiskLruCache完全解析，硬盘缓存的最佳方案](http://blog.csdn.net/guolin_blog/article/details/28863651)
