# View对Touch事件的处理
![View的Touch事件](http://img.blog.csdn.net/20160601225357520)

1. View处理Touch事件的总体流程
dispatchTouchEvent()—>onTouch()—>onTouchEvent()—>onClick()
Touch事件最先传入dispatchTouchEvent()中；如果该View存在TouchListener那么会调用该监听器中的onTouch()。在此之后如果Touch事件未被消费，则会执行到View的onTouchEvent()方法，在该方法中处理ACTION_UP事件时若该View存在ClickListener则会调用该监听器中的onClick()
2. onTouch()与onTouchEvent()以及click三者的区别和联系
onTouch()与onTouchEvent()都是处理触摸事件的API
onTouch()属于TouchListener接口中的方法，是View暴露给用户的接口便于处理触摸事件，而onTouchEvent()是Android系统自身对于Touch处理的实现
先调用onTouch()后调用onTouchEvent()。而且只有当onTouch()未消费Touch事件才有可能调用到onTouchEvent()。即onTouch()的优先级比onTouchEvent()的优先级更高。
在onTouchEvent()中处理ACTION_UP时会利用ClickListener执行Click事件。所以Touch的处理是优先于Click的
简单地说三者执行顺序为：onTouch()/setOnTouchListener()–>onTouchEvent()–>onClick()
3. View没有事件的拦截(onInterceptTouchEvent( ))，ViewGroup才有，请勿混淆


# ViewGroup对Touch事件的处理
当一个Touch事件发生后，事件首先由系统传递给当前Activity并且由其dispatchTouchEvent()派发该Touch事件，源码如下：
```java
public boolean dispatchTouchEvent(MotionEvent ev) {
    // 1.处理ACTION_DOWN事件
    if (ev.getAction() == MotionEvent.ACTION_DOWN) {
        // 调用onUserInteraction()该方法在源码中为一个空方法，可依据业务需求在Activity中覆写该方法。
        onUserInteraction();
    }
    // 2.利用PhoneWindow的superDispatchTouchEvent()派发事件
    if (getWindow().superDispatchTouchEvent(ev)) {
        return true;
    }
    // 3.如果没有子View消费Touch事件，那么Activity会调用自身的onTouchEvent()处理Touch.
    return onTouchEvent(ev);
}
```
在以上步骤中第二步是我们关注的重点；它是ViewGroup对于Touch事件分发的核心。 `getWindow().superDispatchTouchEvent(ev)`最终会将Touch事件交给了DecorView进行派发。
DecorView继承自FrameLayout它是整个界面的最外层的ViewGroup。
至此，Touch事件就已经到了顶层的View且由其开始逐级派发。果superDispatchTouchEvent()方法最终true则表示Touch事件被消费；反之，则进入下一步

如果ViewGroup拦截了Touch事件或者子View不能消耗掉Touch事件，那么ViewGroup会在其自身的onTouch()，onTouchEvent()中处理Touch
如果子View消耗了Touch事件父View就不能再处理Touch.

Touch事件的传递顺序为  
Activity–>外层ViewGroup–>内层ViewGroup–>View

Touch事件的消费顺序为  
View–>内层ViewGroup–>外层ViewGroup–>Activity

在Touch事件的传递过程中，如果上一级拦截了Touch那么其下一级就无法在收到Touch事件。  
在Touch事件的消费过程中，如果下一级消费Touch事件那么其上一级就无法处理Touch事件。



# 滑动冲突
在开发中时常遇到一个棘手的问题：Touch事件的滑动冲突。比如ListView嵌套ScrollView，ViewPager嵌套ScrollView，ListView嵌套ScrollView时常常发生。
先啰嗦一下，View 的事件分发机制主要涉及到一下几个方法:
* dispatchTouchEvent ，这个方法主要是用来分发事件的。返回true，表示被自己消费了；返回false，则表示不分发，还给上一级了自己也不处理；返回super才是继续往下走。
* onInterceptTouchEvent，这个方法主要是用来拦截事件的（需要注意的是ViewGroup才有这个方法，View没有onInterceptTouchEvent这个方法)。如果返回true,则表示拦截掉了。返回false，则表示传递给子View
* onTouchEvent这个方法主要是用来处理事件的。返回true，表示处理了。false，表示没有处理，交给上级处理。如果DOWN事件，就返回false，那么就接收不到MOVE,UP事件了。
* requestDisallowInterceptTouchEvent(true)，这个方法能够影响父View是否拦截事件，true表示不拦截事件，false表示拦截事件

这些滑动冲突的产生，一般而言都具有以下特点：

1. 子View和父View都有滑动的需求
2. 滑动事件不能准确地传递给合适的View
那么，有哪些方法可以解决滑动冲突呢？
1. 子View禁止父View拦截Touch事件
我们知道：Touch事件是由父View分发的。如果一个Touch事件是子View需要的，但是被其父View拦截了，子View就无法处理该Touch事件了。在此情形下，子View可以调用requestDisallowInterceptTouchEvent( )禁止父View对Touch的拦截
2. 在父View中准确地进行事件分发和拦截
我们可以重写父View中与Touch事件分发相关的方法，比如onInterceptTouchEvent( )。这些方法中摒弃系统默认的流程，结合自身的业务逻辑重写该部分代码，从而使父View放行子View需要的Touch。

解决滑动冲突的两种常见套路：外部拦截和内部拦截；
既然是套路，那么就是有固定的招数的：
*外部拦截*: 所谓的外部拦截法是指点击事件都先经过父容器的拦截处理，如果父容器需要此事件就拦截，如果不需要此事件就不拦截，这种处理方式也是符合事件分发机制的流程的。外部拦截法需要重写父View的onInterceptTouchEvent（）方法在内部做相应的处理即可：
```java
@Override
public boolean onInterceptTouchEvent(MotionEvent ev) {
   boolean isIntercepted = false;
   int x = (int) ev.getX();
   int y = (int) ev.getY();
   switch (ev.getAction()) {
   case MotionEvent.ACTION_DOWN:
       isIntercepted = false;
       break;
   case MotionEvent.ACTION_MOVE:
       if(父容器需要当前点击事件){
           isIntercepted = true;
       }else{
           isIntercepted = false;
       }
       break;
   case MotionEvent.ACTION_UP:
       isIntercepted = false;
       break;

   default:
       break;
   }
   mLastX = x;
   mLastY = y;
   return isIntercepted;
}
```
*内部拦截：* 内部拦截法是指父容器不拦截任何事件，所有的事件都传递给子元素，如果子元素需要就直接消费掉，否则交给父容器进行处理。这种方式和事件分发流程不太一致，需要父容器的onInterceptTouchEvent方法做相应处理，才能完成！使用起来比外部拦截法稍显复杂，我们需要重写子元素的dispatchTouchEvent方法：
```java
// 内部view拦截
@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
    int x = (int) ev.getX();
    int y = (int) ev.getY();

    switch (ev.getAction()) {
    case MotionEvent.ACTION_DOWN:
        getParent().requestDisallowInterceptTouchEvent(true);
        break;
    case MotionEvent.ACTION_MOVE:
        int dx = x - mLastX;
        int dy = y - mLastY;
        if(父View需要当前点击事件){
            getParent().requestDisallowInterceptTouchEvent(false);
        }
        break;
    case MotionEvent.ACTION_UP:

        break;
    default:
        break;
    }

    mLastX = x;
    mLastY = y;

    return super.dispatchTouchEvent(ev);
}

// 外部view继续拦截
@Override
public boolean onInterceptTouchEvent(MotionEvent ev) {
    int action = ev.getAction();
    if (action == MotionEvent.ACTION_DOWN) {
        return false;
    }else{
        return true;
    }
}
```

# 相关链接
[图解 Android 事件分发机制](http://www.jianshu.com/p/e99b5e8bd67b#)
[详解Touch事件](http://blog.csdn.net/lfdfhl/article/details/51603088)
