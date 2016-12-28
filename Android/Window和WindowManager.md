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

#### 关于Toast在子线程中使用报错
在子线程中使用Toast抛出异常，提示错误显示:Can't create handler inside thread that has not called Looper.prepare()

其实问题没那么复杂，直接从代码分析原因即可。
先看Toast.makeText(Context,CharSequence,int)的源码:
```java
public static Toast makeText(Context context, CharSequence text, @Duration int duration) {
    Toast result = new Toast(context);

    LayoutInflater inflate = (LayoutInflater)
            context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
    View v = inflate.inflate(com.android.internal.R.layout.transient_notification, null);
    TextView tv = (TextView)v.findViewById(com.android.internal.R.id.message);
    tv.setText(text);

    result.mNextView = v;
    result.mDuration = duration;

    return result;
}
```
这里就是初始化View并给Toast赋值，但是这里并没有涉及Handler,为什么会出现“Can't create handler inside thread that has not called Looper.prepare()”这样的错误呢？

其实是在Toast的构造方法中:
```java
public Toast(Context context) {
    mContext = context;
    mTN = new TN();
    mTN.mY = context.getResources().getDimensionPixelSize(
            com.android.internal.R.dimen.toast_y_offset);
    mTN.mGravity = context.getResources().getInteger(
            com.android.internal.R.integer.config_toastDefaultGravity);
}
```
接着看看TN这个类
```java
private static class TN extends ITransientNotification.Stub {
    final Runnable mShow = new Runnable() {
        @Override
        public void run() {
            handleShow();
        }
    };

    final Runnable mHide = new Runnable() {
        @Override
        public void run() {
            handleHide();
            // Don't do this in handleHide() because it is also invoked by handleShow()
            mNextView = null;
        }
    };

    private final WindowManager.LayoutParams mParams = new WindowManager.LayoutParams();
    // 会在TN的构造方法之前执行，从而导致在Toast()中抛出异常。
    final Handler mHandler = new Handler();

    int mGravity;
    int mX, mY;
    float mHorizontalMargin;
    float mVerticalMargin;


    View mView;
    View mNextView;
    int mDuration;

    WindowManager mWM;

    static final long SHORT_DURATION_TIMEOUT = 5000;
    static final long LONG_DURATION_TIMEOUT = 1000;

    TN() {
        ...
    }

    /**
     * schedule handleShow into the right thread
     */
    @Override
    public void show() {
        if (localLOGV) Log.v(TAG, "SHOW: " + this);
        mHandler.post(mShow);
    }

    /**
     * schedule handleHide into the right thread
     */
    @Override
    public void hide() {
        if (localLOGV) Log.v(TAG, "HIDE: " + this);
        mHandler.post(mHide);
    }

    ...
}
```
所以到这里就可以看出为什么会崩溃了，因为在show()之前，Toast的初始化方法中，TN初始化之前，new了一个Handler从而导致异常。
Toast本质上是一个window，跟activity是平级的，checkThread只是Activity维护的View树的行为。

Toast使用的无所谓是不是主线程Handler，吐司操作的是window，不属于checkThread抛主线程不能更新UI异常的管理范畴。它用Handler只是为了用队列和时间控制排队显示吐司。所以Toast无法在没有Looper的线程中弹出，这是因为Handler需要使用Looper才能完成切换线程的功能。


解决这个问题，我们先通过源码来具体探究一下：
我们先看Toast的显示过程
```java
public void show() {
    if (mNextView == null) {
        throw new RuntimeException("setView must have been called");
    }

    INotificationManager service = getService();
    String pkg = mContext.getOpPackageName();
    TN tn = mTN;
    tn.mNextView = mNextView;

    try {        
        service.enqueueToast(pkg, tn, mDuration);
    } catch (RemoteException e) {
        // Empty
    }
}

static private INotificationManager getService() {
    if (sService != null) {
        return sService;
    }
    sService = INotificationManager.Stub.asInterface(ServiceManager.getService("notification"));
    return sService;
}
```
其中的INotificationManager是NotificationManagerService, 而且getService是一个非常典型的Binder通信
```java
// 第一个参数表示当前应用的包名
// 第二个参数表示远程回调
// 第三个参数表示toast的时长
public void enqueueToast(String pkg, ITransientNotification callback, int duration) {
    if (pkg == null || callback == null) {
        return ;
    }

    synchronized (mToastQueue) {
        int callingPid = Binder.getCallingPid();
        long callingId = Binder.clearCallingIdentity();
        try {
            ToastRecord record;
            int index = indexOfToastLocked(pkg, callback);
            // If it's already in the queue, we update it in place, we don't
            // move it to the end of the queue.
            if (index >= 0) {
                record = mToastQueue.get(index);
                record.update(duration);
            } else {
                record = new ToastRecord(callingPid, pkg, callback, duration);
                mToastQueue.add(record);
                index = mToastQueue.size() - 1;
                keepProcessAliveLocked(callingPid);
            }
            // If it's at index 0, it's the current toast.  It doesn't matter if it's
            // new or just been updated.  Call back and tell it to show itself.
            // If the callback fails, this will remove it from the list, so don't
            // assume that it's valid after this.
            if (index == 0) {
                showNextToastLocked();
            }
        } finally {
            Binder.restoreCallingIdentity(callingId);
        }
    }
}
```
enqueueToast首先将Toast请求封装为ToastRecord对象并将其添加到一个名为mToastQueue的队列中，mToastQueue其实是一个ArrayList。对于非系统应用来说，mToastQueue中最多能同时存在50个ToastRecord，这样做是为了防止DOS。
当ToastRecord被添加到mToastQueue中后，NotificationManagerService就会通过showNextToastLocked方法来显示当前的Toast。
```java
void showNextToastLocked() {
    ToastRecord record = mToastQueue.get(0);
    while (record != null) {
        if (DBG) Slog.d(TAG, "Show pkg=" + record.pkg + " callback=" + record.callback);
        try {
            // 这里就对应了远程回调的show
            // 其实也就是TN类中的show()
            record.callback.show();
            // 显示之后，会调用该方法发送一个延时消息
            scheduleTimeoutLocked(record);
            return;
        } catch (RemoteException e) {
            Slog.w(TAG, "Object died trying to show notification " + record.callback
                    + " in package " + record.pkg);
            // remove it from the list and let the process die
            int index = mToastQueue.indexOf(record);
            if (index >= 0) {
                mToastQueue.remove(index);
            }
            keepProcessAliveLocked(record.pid);
            if (mToastQueue.size() > 0) {
                record = mToastQueue.get(0);
            } else {
                record = null;
            }
        }
    }
}
```
我们先看看record.callback.show();也就是TN中的show()
```java
final Runnable mShow = new Runnable() {
    @Override
    public void run() {
        handleShow();
    }
};
// 这里才是真正显示toast的地方。这里才真正涉及到了更新UI操作
public void handleShow() {
    if (localLOGV) Log.v(TAG, "HANDLE SHOW: " + this + " mView=" + mView
            + " mNextView=" + mNextView);
    if (mView != mNextView) {
        // remove the old view if necessary
        handleHide();
        mView = mNextView;
        Context context = mView.getContext().getApplicationContext();
        String packageName = mView.getContext().getOpPackageName();
        if (context == null) {
            context = mView.getContext();
        }
        mWM = (WindowManager)context.getSystemService(Context.WINDOW_SERVICE);
        // We can resolve the Gravity here by using the Locale for getting
        // the layout direction
        final Configuration config = mView.getContext().getResources().getConfiguration();
        final int gravity = Gravity.getAbsoluteGravity(mGravity, config.getLayoutDirection());
        mParams.gravity = gravity;
        if ((gravity & Gravity.HORIZONTAL_GRAVITY_MASK) == Gravity.FILL_HORIZONTAL) {
            mParams.horizontalWeight = 1.0f;
        }
        if ((gravity & Gravity.VERTICAL_GRAVITY_MASK) == Gravity.FILL_VERTICAL) {
            mParams.verticalWeight = 1.0f;
        }
        mParams.x = mX;
        mParams.y = mY;
        mParams.verticalMargin = mVerticalMargin;
        mParams.horizontalMargin = mHorizontalMargin;
        mParams.packageName = packageName;
        mParams.removeTimeoutMilliseconds = mDuration ==
            Toast.LENGTH_LONG ? LONG_DURATION_TIMEOUT : SHORT_DURATION_TIMEOUT;
        if (mView.getParent() != null) {
            if (localLOGV) Log.v(TAG, "REMOVE! " + mView + " in " + this);
            mWM.removeView(mView);
        }
        if (localLOGV) Log.v(TAG, "ADD! " + mView + " in " + this);
        // 在真正真正的将Toast添加上去
        mWM.addView(mView, mParams);
        trySendAccessibilityEvent();
    }
}
```
接下来再看看,NMS发送的一个延时消息
```java
private void scheduleTimeoutLocked(ToastRecord r)
{
    mHandler.removeCallbacksAndMessages(r);
    Message m = Message.obtain(mHandler, MESSAGE_TIMEOUT, r);
    // 在这里控制Toast的显示时长，LONG_DELAY是3.5s，SHORT_DELAY是2s
    long delay = r.duration == Toast.LENGTH_LONG ? LONG_DELAY : SHORT_DELAY;
    mHandler.sendMessageDelayed(m, delay);
}
```
延迟相应的时间后，NMS会通过cancelToastLocked方法来隐藏Toast，并将其从mToastQueue中移除，这个时候如果mToastQueue中还有其他Toast，那么NMS就继续显示其他Toast。

##### 解决办法
最简单的方法：
```java
new Thread(){
  public void run(){
    Looper.prepare();//给当前线程初始化Looper
    Toast.makeText(getApplicationContext(),"你猜我能不能弹出来～～",0).show();//Toast初始化的时候会new Handler();无参构造默认获取当前线程的Looper，如果没有prepare过，则抛出题主描述的异常。上一句代码初始化过了，就不会出错。
    Looper.loop();//这句执行，Toast排队show所依赖的Handler发出的消息就有人处理了，Toast就可以吐出来了。但是，这个Thread也阻塞这里了，因为loop()是个for (;;) ...
  }
}.start();
```

##### 总结
Toast本质上是一个window，跟activity是平级的，checkThread只是Activity维护的View树的行为。

Toast使用的无所谓是不是主线程Handler，吐司操作的是window，不属于checkThread抛主线程不能更新UI异常的管理范畴。它用Handler只是为了用队列和时间控制排队显示吐司。

同时这里还涉及到两处IPC的知识，一个是NotificationManagerService的创建，一个是显示和隐藏Toast方法的调用。
