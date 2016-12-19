#Drawable简介
Drawable有很多种，它们都表示一种图像的概念，但是它们又不全是图片，通过颜色也可以构造出各式各样的图像的效果。在实际开发中，Drawable常被用来作为View的背景使用。Drawable一般都是通过XML来定义的，当然我们也可以通过代码来创建具体的Drawable对象，只是用代码创建会稍显复杂。

## BitmapDrawable
这几乎是最简单的Drawable了，它表示的就是一张图片。
```XML
<?xml version="1.0" encoding="utf-8"?>
<bitmap xmlns:android="http://schemas.android.com/apk/res/android"
    图片的资源id
    android:src="@drawable/resource"
    是否开启图片抗锯齿功能。开启后会让图片变得平滑，但是一定程度上降低图片的清晰度，不过影响不大。因此应该开启。
    android:antialias="true"
    // 是否开启抖动效果。可以让高质量的图片在低质量的屏幕上还保持较好的显示效果，应该开启。
    android:dither="true"
    // 是否开启过滤效果。当图片尺寸被拉伸或者压缩，开启过滤效果可以保持较好的显示效果，应该开启。
    android:filter="true"
    // 当图片小于容器的尺寸时，设置此选项可以对图片进行定位。
    android:gravity="top"
    // 一种图像相关的处理技术。日常开发中为false
    android:mipMap="false"
    // 平铺模式
    android:tileMode="disabled" />
```

## ShapeDrawable
ShapeDrawable是一种很常见的Drawable，可以理解为通过颜色来构造的图形，它既可以是纯色的图形，也可以是具有渐变效果的图形。
```xml
<?xml version="1.0" encoding="utf-8"?>
<shape xmlns:android="http://schemas.android.com/apk/res/android"
    // 表示图形的形状。line - 横线； oval - 椭圆； rectangle - 矩形； ring - 圆环
    android:shape="line" | "oval" | "rectangle" | "ring">

    <!-- 只适用于矩形的shape， 表示shape四个角的圆角程度 -->
    <corners
        android:radius=""
        android:topLeftRadius=""
        android:topRightRadius=""
        android:bottomLeftRadius=""
        android:bottomRightRadius="" />
    <!-- 渐变色填充，和solid是互相排斥的 -->
    <gradient
        // 渐变的角度， 0表示从左到右， 90表示从下到上
        android:angle=""
        // 渐变的中心点的横坐标
        android:centerX=""
        android:centerY=""
        // 渐变的开始颜色
        android:startColor=""
        android:centerColor=""
        android:endColor=""
        // 渐变半径， 仅当android:type="radial"时有效
        android:gradientRadius=""
        // 渐变的类别，有linear - 线性渐变； radial - 径向渐变； sweep - 扫描线渐变
        android:type=""
        // 一般为false
        android:useLevel="" />
    // 纯色填充
    <solid
        android:color="" />

    <padding
        android:left=""
        android:top=""
        android:right=""
        android:bottom="" />

    <size
        android:width=""
        android:height="" />
    // shape的描边
    <stroke
        android:width=""
        android:color=""
        android:dashWidth=""
        android:dashGap="" />

</shape>
```

## LayerDrawable
LayerDrawable对应的XML标签是<layer-list>，它表示一种层次化的Drawable集合，通过将不同的Drawable放置在不同的层上面从而达到一种叠加后的效果。
一个layer-list中可以包含多个item，每个item表示一个Drawable。Item的结构也比简单，常用属性有android:top/bottom/left/right，它们分别表示Drawable相当于View的上下左右偏移量。

## StateListDrawable
StateListDrawable对应于<selector>标签，它也是表示Drawable集合，每个Drawable都对应着View的一种状态，这样系统会根据View的状态来选择合适的Drawable。

## LevelListDrawable
LevelListDrawable对应于<level-list>标签，它同样表示一个Drawable集合，集合中的每个Drawable都有一个等级的概念。根据不同的等级，LevelListDrawable会切换为对应的Drawable。
```xml
<?xml version="1.0" encoding="utf-8"?>
<level-list xmlns:android="http://schemas.android.com/apk/res/android">

    <item
        android:drawable="@drawable/status_off"
        android:maxLevel="0" />

    <item
        android:drawable="@drawable/status_on"
        android:maxLevel="1" />

</level-list>
```
当它作为View的背景时，可以通过Drawable的setLevel方法来设置不同的等级从而切换具体的Drawable。
如果它被用来作为ImageView的前景Drawable，那么还可以通过ImageView的setImageLevel方法来切换Drawable。
Drawable是有等级范围的，即0~10000。

## TransitionDrawable
TransitionDrawable 对应标签<transition>，它用于是吸纳两个Drawable之间的淡入淡出效果。
```
<transition xmlns:android="http://schemas.android.com/apk/res/android" >
    <item android:drawable="@drawable/shape_drawable_gradient_linear"/>
    <item android:drawable="@drawable/shape_drawable_gradient_radius"/>
</transition>

TransitionDrawable drawable = (TransitionDrawable) v.getBackground();
drawable.startTransition(5000);
```

## InsetDrawable
InsetDrawable 对应标签<inset>，它可以将其他drawable内嵌到自己当中，并可以在四周留出一定的间距。当一个view希望自己的背景比自己的实际区域小的时候，可以采用InsetDrawable来实现。
```XML
<inset xmlns:android="http://schemas.android.com/apk/res/android"
    android:insetBottom="15dp"
    android:insetLeft="15dp"
    android:insetRight="15dp"
    android:insetTop="15dp" >

    <shape android:shape="rectangle" >
        <solid android:color="#ff0000" />
    </shape>

</inset>
```

## ScaleDrawable
ScaleDrawable 对应标签<scale>，它可以根据自己的level将指定的Drawable缩放到一定比例。如果level越大，那么内部的drawable看起来就越大。

## ClipDrawable
ClipDrawable 对应标签<clip>，它可以根据自己当前的level来裁剪另一个drawable，裁剪方向由android:clipOrientation和andoid:gravity属性来共同控制。level越大，表示裁剪的区域越小。
```xml
<clip xmlns:android="http://schemas.android.com/apk/res/android"
    android:clipOrientation="vertical"
    android:drawable="@drawable/image1"
    android:gravity="bottom" />
```

# 自定义Drawable
Drawable的使用范围很单一，一个是作为ImageView中的图像来显示，另外一个就是作为View的背景，大多数情况下Drawable都是以View的背景这种形式出现的。
Drawable的工作原理很简单，其核心就是draw方法。
通常情况下，我们没有必要去自定义Drawable，这是因为自定义的Drawable无法在XML中使用。
1. Drawable的工作核心就是draw方法，所以自定义drawable就是重写draw方法，当然还有setAlpha、setColorFilter和getOpacity这几个方法。当自定义Drawable有固有大小的时候最好重写getIntrinsicWidth和getIntrinsicHeight方法。
2. Drawable的内部大小不等于Drawable的实际区域大小，Drawable的实际区域大小可以通过它的getBounds方法来得到，一般来说它和view的尺寸相同。
