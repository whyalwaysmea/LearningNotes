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
Picasso和Glide在磁盘缓存策略上有很大的不同；Picasso缓存的是全尺寸的，而Glide缓存的是跟ImageView尺寸相同的。Glide的这种方式优点是在同样大小的ImageView下加载显示非常快。而Picasso的方式则因为需要在显示之前重新调整大小而导致一些延迟。   

Picasso只缓存一个全尺寸的。Glide则不同，它会为每种大小的ImageView缓存 一次。尽管一张图片已经缓存了一次，但是假如你要在另外一个地方再次以不同尺寸显示，需要重新下载，调整成新尺寸的大小，然后将这个尺寸的也缓存起来。  
当然我们也可以让Glide缓存完整的尺寸：
```java
Glide.with(this)
     .load("http://nuuneoi.com/uploads/source/playstore/cover.jpg")
     .diskCacheStrategy(DiskCacheStrategy.ALL)
     .into(ivImgGlide);
```
下次在任何ImageView中加载图片的时候，全尺寸的图片将从缓存中取出，重新调整大小，然后缓存。

### Glide比Picasso多的功能
Glide可以加载GIF动态图，而Picasso不能。  

还有一个特性是你可以配置图片显示的动画，而Picasso只有一种动画：fading in。

### 包大小
Picasso (v2.5.1)的大小约118kb，而Glide (v3.5.2)的大小约430kb。  
Picasso和Glide的方法个数分别是840和2678个。

## Fresco
### 关于Fresco
Fresco 中设计有一个叫做 Image Pipeline 的模块。它负责从网络，从本地文件系统，本地资源加载图片。为了最大限度节省空间和CPU时间，它含有3级缓存设计（2级内存，1级磁盘）。

Fresco 中设计有一个叫做 Drawees 模块，它会在图片加载完成前显示占位图，加载成功后自动替换为目标图片。当图片不再显示在屏幕上时，它会及时地释放内存和空间占用。

### 特性
#### 内存管理
解压后的图片，即Android中的Bitmap，占用大量的内存。大的内存占用势必引发更加频繁的GC。在5.0以下，GC将会显著地引发界面卡顿。在5.0以下系统，Fresco将图片放到一个特别的内存区域。当然，在图片不显示的时候，占用的内存会自动被释放。这会使得APP更加流畅，减少因图片内存占用而引发的OOM。Fresco 在低端机器上表现一样出色，你再也不用因图片内存占用而思前想后。

#### 图片加载
Fresco的Image Pipeline允许你用很多种方式来自定义图片加载过程，比如：  
- 为同一个图片指定不同的远程路径，或者使用已经存在本地缓存中的图片
- 先显示一个低清晰度的图片，等高清图下载完之后再显示高清图
- 加载完成回调通知
- 对于本地图，如有EXIF缩略图，在大图加载完成之前，可先显示缩略图
- 缩放或者旋转图片
- 对已下载的图片再次处理
- 支持WebP解码，即使在早先对WebP支持不完善的Android系统上也能正常使用！

#### 图片绘制
Fresco 的 Drawees 设计，带来一些有用的特性：  
- 自定义居中焦点
- 圆角图，当然圆圈也行
- 下载失败之后，点击重现下载
- 自定义占位图，自定义overlay, 或者进度条
- 指定用户按压时的overlay

#### 渐进式呈现
渐进式的JPEG图片格式已经流行数年了，渐进式图片格式先呈现大致的图片轮廓，然后随着图片下载的继续，呈现逐渐清晰的图片，这对于移动设备，尤其是慢网络有极大的利好，可带来更好的用户体验。

#### 动图加载
加载Gif图和WebP动图在任何一个Android开发者眼里看来都是一件非常头疼的事情。每一帧都是一张很大的Bitmap，每一个动画都有很多帧。Fresco让你没有这些烦恼，它处理好每一帧并管理好你的内存。

## 相关链接
[Introduction to Glide, Image Loader Library for Android, recommended by Google](https://inthecheesefactory.com/blog/get-to-know-glide-recommended-by-google/en)
