# 简介
View的滑动有三种实现方式：
1. 通过View本身提供的scrollTo/scrollBy方法;
2. 通过动画给View施加平移效果来实现滑动;
3. 通过改变View的LayoutParams使得View重新布局从而实现滑动。


# 使用scrollTo/scrollBy
```java
public void scrollTo(int x, int y) {
    if (mScrollX != x || mScrollY != y) {
        int oldX = mScrollX;
        int oldY = mScrollY;
        mScrollX = x;
        mScrollY = y;
        invalidateParentCaches();
        onScrollChanged(mScrollX, mScrollY, oldX, oldY);
        if (!awakenScrollBars()) {
            postInvalidateOnAnimation();
        }
    }
}

public void scrollBy(int x, int y) {
    scrollTo(mScrollX + x, mScrollY + y);
}

```
通过源码可以看出，scrollBy实际上也是调用了scrollTo的方法，它实现了基于当前位置的相对滑动，而scrollTo则实现了基于所传递参数的绝对滑动。

mScrollX的值总是等于View左边缘和View内容左边缘在水平方向的距离，当View左边缘在View内容左边缘的右边时，mScrollX为正值。

```java
@Override
public boolean onTouchEvent(MotionEvent event) {
    int x = (int) event.getX();
    int y = (int) event.getY();
    switch (event.getAction()) {
        case MotionEvent.ACTION_DOWN:
            lastX = (int) event.getX();
            lastY = (int) event.getY();
            break;
        case MotionEvent.ACTION_MOVE:
            int offsetX = x - lastX;
            int offsetY = y - lastY;
            ((View) getParent()).scrollBy(-offsetX, -offsetY);
            break;
    }
    return true;
}
```

# 使用动画
既可以采用传统的View动画，也可以采用属性动画
```xml
// 使用View动画
<?xml version="1.0" encoding="utf-8"?>
<set xmlns:android="http://schemas.android.com/apk/res/android"
    android:fillAfter="true"
    android:zAdjustment="normal" >

    <translate
        android:duration="100"
        android:fromXDelta="0"
        android:fromYDelta="0"
        android:interpolator="@android:anim/linear_interpolator"
        android:toXDelta="100"
        android:toYDelta="100" />
</set>
// 属性动画
ObjectAnimator.ofFloat(targetView, "translationX, 0, 100").setDuration(100).start();
```

如果采用属性动画的话，为了能够兼容3.0以下的版本，需要采用开源库[nineoldandroids](http://nineoldandroids.com)
```java
// 使用属性动画
@Override
public boolean onTouchEvent(MotionEvent event) {
   int x = (int) event.getRawX();
   int y = (int) event.getRawY();
   switch (event.getAction()) {
   case MotionEvent.ACTION_DOWN: {
       break;
   }
   case MotionEvent.ACTION_MOVE: {
       int deltaX = x - mLastX;
       int deltaY = y - mLastY;
       Log.d(TAG, "move, deltaX:" + deltaX + " deltaY:" + deltaY);
       int translationX = (int)ViewHelper.getTranslationX(this) + deltaX;
       int translationY = (int)ViewHelper.getTranslationY(this) + deltaY;
       // ViewHelper是动画兼容库中的一个类
       ViewHelper.setTranslationX(this, translationX);
       ViewHelper.setTranslationY(this, translationY);
       break;
   }
   case MotionEvent.ACTION_UP: {
       break;
   }
   default:
       break;
   }

   mLastX = x;
   mLastY = y;
   return true;
}
```

# 改变布局参数
通过改变LayoutParams。
```
@Override
public boolean onTouchEvent(MotionEvent event) {
    int x = (int) event.getX();
    int y = (int) event.getY();
    switch (event.getAction()) {
        case MotionEvent.ACTION_DOWN:
            // 记录触摸点坐标
            lastX = (int) event.getX();
            lastY = (int) event.getY();
            break;
        case MotionEvent.ACTION_MOVE:
            // 计算偏移量
            int offsetX = x - lastX;
            int offsetY = y - lastY;
            ViewGroup.MarginLayoutParams layoutParams = (ViewGroup.MarginLayoutParams) getLayoutParams();
            layoutParams.leftMargin = getLeft() + offsetX;
            layoutParams.topMargin = getTop() + offsetY;
            setLayoutParams(layoutParams);
            break;
    }
    return true;
}
```

# 区别
scrollTo/scrollBy：操作简单，适合对view内容的滑动；
动画：操作简单，主要适用于没有交互的view和实现复杂的动画效果
改变布局参数：操作稍微复杂，适用于有交互的view


# 弹性滑动
先了解一下Scroller，弹性滑动对象，用于实现View的弹性滑动。刚才上面讲的使View滑动的方式其过程是瞬间完成的，没有过度效果用户体验不好。这个时候就可以使用Scroller来实现有过度效果的滑动。
```java
Scroller scroller = new Scroller(mContext);

// 缓慢滚动到指定位置
private void smoothScrollTo(int destX, int destY) {
    int scrollX = getScrollX();
    int delta = destX - scrollX;
    // 100ms内滑动destX
    scroller.startScroll(scrollX, 0, delta, 0, 1000);
    invalidate();
}

public void computeScoll() {
    if(scroller.computeScrollOffset()) {
        scrollTo(scroller.getCurrX(), scroller.getCurrY());
        postInvalidate();
    }
}
```

# Scroller原理
其实在scroller.startScroll(int startX, int startY, int dx, int dy, int duration);中，是没有做滑动相关的事，那么Scroller到底如何让View弹性滑动的呢？
其实是通过*invalidate();* invalidate()会导致View重绘，在View的draw方法中又会去调用computeScoll();computeScoll方法会向Scroller获取当前的scrollX和scrollY,然后通过scrollTo方法实现滑动；接着又重绘，又调用。

[站在源码的肩膀上全解Scroller工作机制](http://blog.csdn.net/lfdfhl/article/details/53143114)
