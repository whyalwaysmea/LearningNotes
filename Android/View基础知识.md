# 什么是View
View是一种界面层的控件的一种抽象，它代表了一个控件。除了View，还有ViewGroup。
ViewGroup也继承了View，这就意味着View本身就可以是单个控件也可以是由多个控件组成的一组控件。

# 初始ViewRoot和DecorView
ViewRoot对应于ViewRootImpl类，它是连接WindowManager和DecorView的纽带，View的三大流程均是通过ViewRoot来完成的。
在ActivityThread中，当Activity对象被创建完毕后，会将DecorView添加到Window中，同时会创建ViewRootImpl对象，并将ViewRootImpl对象和DecorView建立关联。

# View的位置参数
View的位置主要由它的四个顶点来决定，分别对应于View的四个属性：top、left、right、bottom。
这些坐标都是相当于View的父容器来说的，因此它是一种相对坐标。
从Android3.0开始，View增加了额外的几个参数：x、y、translationX和translationY，其中x和y是View左上角的坐标，而translationX和translationY是View左上角相当于父容器的偏移量。
**注意：** View在平移的过程中，top和left表示的是原始左上角的位置信息，其值并不会发生改变，此时发生改变的是x、y、translationX和translationY。

# MotionEvent和TouchSlop
## MotionEvent：
典型的事件类型分为：
* ACTION_DOWN -- 手指刚接触屏幕
* ACTION_MOVE -- 手指在屏幕上移动
* ACTION_UP -- 手指从屏幕上松开的一瞬间

MotionEvent.getX/getY：返回的是相当于当前View左上角的x和y坐标
MotionEvent.getRawX/getRawY:返回的是相当于手机屏幕左上角的x和y坐标。

## TouchSlop
TouchSlop是系统所能识别出的被认为是滑动的最小距离。
`ViewConfiguration.get(getContext()).getScaledTouchSlop();`

# VelocityTracker
速度追踪，用于追踪手指在滑动过程中的速度，包括水平和竖直方向的速度。
```java
// 在View的onTouchEvent方法中追踪当前单击事件的速度
VelocityTracker velocityTracker = VelocityTracker.obtain();
velocityTracker.addMovement(event);
// 获取速度
velocityTracker.computeCurrentVelocity(1000);
float xVelocity = velocityTracker.getXVelocity();
float yVelocity = velocityTracker.getYVelocity();

// 不需要的时候再进行重置并回收内存
velocityTracker.clear();
velocityTracker.recycle();
```

# GestureDetector
手势检测，用于辅助检测用户的单机、滑动、长按、双击等行为。
*建议：* 如果只是监听滑动相关的，建议自己在onTouchEvent中实现，如果要监听双击这种行为的话，那么就使用GestureDetector。

```java
// 1. 先写一个类用于继承GestureDetector.SimpleOnGestureListener，为什么继承这个类？因为这个类中包含了双击事件
class MyOnGestureListener extends GestureDetector.SimpleOnGestureListener {
    ...
}
// 2.实例化GestureDetector
GestureDetector mGestureDetector = new GestureDetector(new MyOnGestureListener());
// 解决长按屏幕后无法拖动的现象
mGestureDetector.setIsLongPressEnabled(false);

// 3.接管目标View的onTouchEvent方法
boolean consume = mGestureDetector.onTouchEvent(event);
return consume;
```

# 相关方法
## inflate
我们经常会使用到如下的方法
```java
LayoutInflater.from(mContext).inflate(int resource, ViewGroup root, boolean attachToRoot);
```
* 如果root为null，attachToRoot将失去作用，设置任何值都没有意义。返回resource对应的View
* 如果root不为null，attachToRoot设为true，则会给加载的布局文件的指定一个父布局，即root。然后返回root
* 如果root不为null，attachToRoot设为false，则会将布局文件最外层的所有layout属性进行设置，当该view被添加到父view当中时，这些layout属性会自动生效。

[Android LayoutInflater原理分析，带你一步步深入了解View(一)](http://blog.csdn.net/guolin_blog/article/details/12921889)   
[Android LayoutInflater深度解析 给你带来全新的认识](http://blog.csdn.net/lmj623565791/article/details/38171465)

## onFinishInflate
当View中所有的子控件均被映射成xml后触发   

## onSizeChanged
当view的大小发生变化时触发

## requestLayout
会导致调用measure()过程 和 layout()过程 。

## invalidate
invalidate使用的非常频繁，它会触发View的重新绘制，也就是绘制流程的draw过程，但不会调用测量和布局过程。需要在主线程中调用

## postInvalidate
在子线程更新UI，postInvalidate内部实现也是使用handler来发送msg到主线程然后调用invalidate。
