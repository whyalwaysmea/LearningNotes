# Window和WindowManager
Window是抽象类，具体实现是PhoneWindow，通过WindowManager就可以创建Window。WindowManager是外界访问Window的入口，但是Window的具体实现是在WindowManagerService中，WindowManager和WindowManagerService的交互是一个IPC过程。所有的视图例如Activity、Dialog、Toast都是附加在Window上的。

#### 通过WindowManager添加View的过程
将一个Button添加到屏幕坐标为(100,300)的位置上
```java
mFloatingButton = new Button(this);
mFloatingButton.setText("test button");
mLayoutParams = new WindowManager.LayoutParams(
        LayoutParams.WRAP_CONTENT, LayoutParams.WRAP_CONTENT, 0, 0,
        PixelFormat.TRANSPARENT);//0,0 分别是type和flags参数，在后面分别配置了
mLayoutParams.flags = LayoutParams.FLAG_NOT_TOUCH_MODAL
        | LayoutParams.FLAG_NOT_FOCUSABLE
        | LayoutParams.FLAG_SHOW_WHEN_LOCKED;
mLayoutParams.type = LayoutParams.TYPE_SYSTEM_ERROR;
mLayoutParams.gravity = Gravity.LEFT | Gravity.TOP;
mLayoutParams.x = 100;
mLayoutParams.y = 300;
mFloatingButton.setOnTouchListener(this);
mWindowManager.addView(mFloatingButton, mLayoutParams);
```
flags参数解析：
FLAG_NOT_FOCUSABLE：表示window不需要获取焦点，也不需要接收各种输入事件。此标记会同时启用FLAG_NOT_TOUCH_MODAL，最终事件会直接传递给下层的具有焦点的window；

FLAG_NOT_TOUCH_MODAL：在此模式下，系统会将window区域外的单击事件传递给底层的window，当前window区域内的单击事件则自己处理，一般都需要开启这个标记；

FLAG_SHOW_WHEN_LOCKED：开启此模式可以让Window显示在锁屏的界面上。

type参数表示window的类型:
window共有三种类型：应用window，子window和系统window。应用window对应着一个Activity，子window不能独立存在，需要附属在特定的父window之上，比如Dialog就是子window。系统window是需要声明权限才能创建的window，比如Toast和系统状态栏这些都是系统window，需要声明的权限是`<uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW" />`。

window是分层的，每个window都对应着z-ordered，层级大的会覆盖在层级小的上面，应用window的层级范围是1~99，子window的层级范围是1000~1999，系统window的层级范围是2000~2999。

WindowManager所提供的功能很简单，常用的只有三个方法，即添加View、更新View和删除View，这三个方法定义在ViewManager中，而WindowManager继承了ViewManager。

#### Window的内部机制

Window是一个抽象的概念，不是实际存在的，它也是以View的形式存在。在实际使用中无法直接访问Window，只能通过WindowManager才能访问Window。每个Window都对应着一个View和一个ViewRootImpl，Window和View通过ViewRootImpl来建立联系。

Window的添加、删除和更新过程都是IPC过程，以Window的添加为例，WindowManager的实现类对于addView、updateView和removeView方法都是委托给WindowManagerGlobal类，该类保存了很多数据列表，例如所有window对应的view集合mViews、所有window对应的ViewRootImpl的集合mRoots等，之后添加操作交给了ViewRootImpl来处理，接着会通过WindowSession来完成Window的添加过程，这个过程是一个IPC调用，因为最终是通过WindowManagerService来完成window的添加的。

# Window的创建过程
#### Activity的Window创建过程
要分析Activity中的Window的创建过程，就必须要了解[Activity的启动过程]。Activity的启动过程很复杂，最终会由ActivityThread中的performLaunchActivity来完成整个启动过程，在这个方法内部会通过类加载器创建Activity的实例对象，并调用它的attach方法为其关联运行过程中所依赖的一系列上下文环境变量；

在Activity的attach方法里，系统会创建Activity所属的Window对象并为其设置回调接口。当window接收到外界的状态变化时就会回调Activity的方法，例如onAttachedToWindow、onDetachedFromWindow、dispatchTouchEvent等；

Activity的Window是由PolicyManager来创建的，它的真正实现是Policy类，它会新建一个PhoneWindow对象，Activity的setContentView的实现是由PhoneWindow来实现的；

Activity的视图是由`setContentView`方法提供的，
```java
public void setContentView(int layoutResID) {
    getWindow().setContentView(layoutResID);
    initWindowDecorActionBar();
}
```
可以看出，Activity将具体的实现交给了Window处理，而Window的具体实现是PhoneWindow。所以只需要看PhoneWindow的相关处理就可以
1.如果没有DecorView，那么就创建它
DecorView是一个FrameLayout，是Activity中的顶级View，它的内部包含标题栏和内部栏。在Activity中我们通过setContentView所设置的布局文件其实就是被加到内容栏之中的，内容栏的完整id是android.R.id.content。DecorView的创建过程由installDecor方法完成

2.将View添加到DecorView的mContentParent中。

3.回调Activity的onContentChanged方法通知Activity视图已经发生改变。但是这个时候DecorView还没有被WindowManager正式添加到Window中。

4.在ActivityThread调用handleResumeActivity方法时，首先会调用Activity的onResume方法，然后会调用makeVisible方法，这个方法中DecorView真正地完成了添加和显示过程。

# Dialog的Window创建过程
过程与Activity的Window创建过程类似，*普通的Dialog* 有一个特别之处，即它必须采用Activity的Context，如果采用Application的Context会报错。原因是Application没有应用token，`应用token`一般是只有Activity才拥有[service也没有token，所以也不能在service中弹出dialog]
另外，系统Window比较特殊，它可以不需要token。因为可以修改Window的类型，然后也是可以显示的。
```java
Dialog dialog = new Dialog(MainActivity.this.getApplication());
TextView tv = new TextView(MainActivity.this);
tv.setText("哈哈");
dialog.setContentView(tv);
dialog.getWindow().setType(WindowManager.LayoutParams.TYPE_SYSTEM_ERROR);
dialog.show();
```

# Toast的Window创建过程
Toast属于系统Window，它内部的视图由两种方式指定：一种是系统默认的演示；另一种是通过setView方法来指定一个自定义的View。

Toast具有定时取消功能，所以系统采用了Handler。Toast的显示和隐藏是IPC过程，都需要NotificationManagerService来实现。在Toast和NMS进行IPC过程时，NMS会跨进程回调Toast中的TN类中的方法，TN类是一个Binder类，运行在Binder线程池中，所以需要通过Handler将其切换到当前发送Toast请求所在的线程，所以Toast无法在没有Looper的线程中弹出。

对于非系统应用来说，mToastQueue最多能同时存在50个ToastRecord，这样做是为了防止DOS(Denial of Service，拒绝服务)。因为如果某个应用弹出太多的Toast会导致其他应用没有机会弹出Toast。
