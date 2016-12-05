# 什么是View
View是一种界面层的控件的一种抽象，它代表了一个控件。除了View，还有ViewGroup。
ViewGroup也继承了View，这就意味着View本身就可以是单个控件也可以是由多个控件组成的一组控件。

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

# Scroller
