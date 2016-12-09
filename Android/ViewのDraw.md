Draw过程就相对比较简单了，它的作用是将View绘制到屏幕上面。View的绘制过程遵循如下几步：

1. 绘制背景drawBackground(canvas);
2. 绘制自己onDraw(canvas);
3. 绘制children dispatchDraw(canvas);
4. 绘制装饰 onDrawForeground(canvas);

一般情况下，我们的重点是在第二步上面，`onDraw(canvas)`
因为这里传入了一个canvas参数，我们首先来看看什么是canvas
>The Canvas class holds the “draw” calls. To draw something, you need 4 basic components: A Bitmap to hold the pixels, a Canvas to host the draw calls (writing into the bitmap), a drawing primitive (e.g. Rect,Path, text, Bitmap), and a paint (to describe the colors and styles for the drawing).

在绘图时需要明确四个核心的东西(basic components)：

1.用什么工具画？
这个小问题很简单，我们需要用一支画笔(Paint)来绘图。
当然，我们可以选择不同颜色的笔，不同大小的笔。
2.把图画在哪里呢？
我们把图画在了Bitmap上，它保存了所绘图像的各个像素(pixel)。
也就是说Bitmap承载和呈现了画的各种图形。
3.画的内容？
根据自己的需求画圆，画直线，画路径。
4.怎么画？
调用canvas执行绘图操作。
比如，canvas.drawCircle()，canvas.drawLine()，canvas.drawPath()将我们需要的图像画出来。

# canvas
在此依次分析canvas的两个构造方法Canvas( )和Canvas(Bitmap bitmap)

```java
/**
 * Construct an empty raster canvas. Use setBitmap() to specify a bitmap to
 * draw into.  The initial target density is {@link Bitmap#DENSITY_NONE};
 * this will typically be replaced when a target bitmap is set for the
 * canvas.
 */
public Canvas() {
    if (!isHardwareAccelerated()) {
        // 0 means no native bitmap
        mNativeCanvasWrapper = initRaster(null);
        mFinalizer = NoImagePreloadHolder.sRegistry.registerNativeAllocation(
                this, mNativeCanvasWrapper);
    } else {
        mFinalizer = null;
    }
}
```
请注意该构造的第一句注释。官方不推荐通过该无参的构造方法生成一个canvas。如果要这么做那就需要调用setBitmap( )为其设置一个Bitmap。为什么Canvas非要一个Bitmap对象呢？原因很简单：Canvas需要一个Bitmap对象来保存像素，如果画的东西没有地方可以保存，又还有什么意义呢？既然不推荐这么做，那就接着有参的构造方法。

```java
/**
 * Construct a canvas with the specified bitmap to draw into. The bitmap
 * must be mutable.
 *
 * The initial target density of the canvas is the same as the given
 * bitmap's density.
 *
 * @param bitmap Specifies a mutable bitmap for the canvas to draw into.
 */
public Canvas(Bitmap bitmap) {
    if (!bitmap.isMutable()) {
        throw new IllegalStateException("Immutable bitmap passed to Canvas constructor");
    }
    throwIfCannotDraw(bitmap);
    mNativeCanvasWrapper = initRaster(bitmap);
    mFinalizer = new CanvasFinalizer(mNativeCanvasWrapper);
    mBitmap = bitmap;
    mDensity = bitmap.mDensity;
}
```
通过该构造方法为Canvas设置了一个Bitmap来保存所绘图像的像素信息。
