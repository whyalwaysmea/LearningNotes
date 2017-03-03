## Glide & Picasso
因为Glide和Picasso的api差不多，所以比较起来更容易一点，就先比较这两个好了。
### 依赖
**Glide**
```java
dependencies {
  compile 'com.github.bumptech.glide:glide:3.7.0'
  compile 'com.android.support:support-v4:19.1.0'
}
```
**Picasso**
```java
dependencies {
  compile 'com.squareup.picasso:picasso:2.5.2'
}
```
别忘了Glide需要依赖Support Library v4。其实Support Library v4已经是应用程序的标配了，这不是什么问题。  

### 使用
**Glide**
```java
Glide.with(context)
    .load("http://inthecheesefactory.com/uploads/source/glidepicasso/cover.jpg")
    .into(ivImg);
```
**Picasso**
```java
Picasso.with(context)
    .load("http://inthecheesefactory.com/uploads/source/glidepicasso/cover.jpg")
    .into(ivImg);
```
尽管它们看起来差不多，但是设计上Glide做得更好：因为Glide的with方法不光接受Context，还接受Activity 和 Fragment，Context会自动的从他们获取。

![比较](http://upload-images.jianshu.io/upload_images/1635594-89c02ddab63a80b8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
将Activity/Fragment传给Glide的好处是：图片加载会和Activity/Fragment的生命周期保持一致，比如Paused状态在暂停加载，在Resumed的时候又自动重新加载。所以我建议传参的时候传递Activity 和 Fragment给Glide，而不是Context。  

### 默认加载格式 & 内存消耗
下面是将1920x1080的图片加载进入768x432大小的ImageView的对比图
![Btmiap Format](https://inthecheesefactory.com/uploads/source/glidepicasso/firstload.jpg)
Glide默认的Bitmap格式是RGB_565，比ARGB_8888的内存开销要少50%。   

下图是Picasso和Glide的内存消耗比较图（基础的application就消耗了大概8MB）
![memory](https://inthecheesefactory.com/uploads/source/glidepicasso/ram1_1.png)

当然Glide也是可以指定加载格式的：
```java
public class GlideConfiguration implements GlideModule {

    @Override
    public void applyOptions(Context context, GlideBuilder builder) {
        // Apply options to the builder here.
        builder.setDecodeFormat(DecodeFormat.PREFER_ARGB_8888);
    }

    @Override
    public void registerComponents(Context context, Glide glide) {
        // register ModelLoaders here.
    }
}
```
同时在AndroidManifest.xml中将GlideModule定义为meta-data
```xml
<meta-data android:name="com.name.GlideConfiguration"
            android:value="GlideModule"/>
```
在指定了Glide的加载格式之后，我们再来看看内存消耗的对比图
![memory](https://inthecheesefactory.com/uploads/source/glidepicasso/ram2_1.png)
可以看到，Picasso占用的内存依然比Glide大。这是因为Picasso加载的是完整的图像(1920x1080)进入内存的。  
而Glide是加载的真实ImageView(768x432)的大小进入内存的。  
当然Picasso是可以调整加载图像的大小:
```java
Picasso.with(this)
    .load("http://nuuneoi.com/uploads/source/playstore/cover.jpg")
    .resize(768, 432)
    .into(ivImgPicasso);
```
当然，有时候你还无法明确的知道ImageView的大小，所以你还可以这么做：
```java
Picasso.with(this)
    .load("http://nuuneoi.com/uploads/source/playstore/cover.jpg")
    .fit()
    .centerCrop()
    .into(ivImgPicasso);
```
![memory](https://inthecheesefactory.com/uploads/source/glidepicasso/memory3.png)

### 缓存
Picasso和Glide在磁盘缓存策略上有很大的不同；Picasso缓存的是全尺寸的，而Glide缓存的是跟ImageView尺寸相同的。


## 相关链接
[Introduction to Glide, Image Loader Library for Android, recommended by Google](https://inthecheesefactory.com/blog/get-to-know-glide-recommended-by-google/en)
