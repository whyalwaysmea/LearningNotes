# Activity生命周期
* Active/Running
这时候，Activity处于Activity栈的最顶层，可见，并与用户进行交互。
>启动Activity: onCreate()—>onStart()—>onResume()，Activity进入运行状态。
* Paused
当Activity失去焦点，被一个新的非全屏的Activity或者一个透明的Activity放置在栈顶时，Activity就转化为Paused形态。但它只是失去了与用户交互的能力，所有状态信息、成员变量都还保持着，只有在系统内存极地的情况下，才会被系统回收掉。
>Activity退居后台: 当前Activity转到新的Activity界面或按Home键回到主屏： onPause()—>onStop()，进入停滞状态。





* Activity返回前台: onRestart()—>onStart()—>onResume()，再次回到运行状态。
* Activity退居后台，且系统内存不足， 系统会杀死这个后台状态的Activity（此时这个Activity引用仍然处在任务栈中，只是这个时候引用指向的对象已经为null），若再次回到这个Activity,则会走onCreate()–>onStart()—>onResume()(将重新走一次Activity的初始化生命周期)

![Activity生命周期](https://camo.githubusercontent.com/1523c46db6fb3e46f13db6effb0b27e972dd85d8/687474703a2f2f696d672e626c6f672e6373646e2e6e65742f3230313330383238313431393032383132)

-------
### 屏幕旋转的生命周期
在未对Activity进行属性设置的时候：
```Java
onPause -> onSaveInstanceState -> onStop -> onDestroy -> onCreate -> onStart -> onResume
```
防止屏幕旋转Activity重建的方法：
1. 禁止旋转屏幕
```Java
android:screenOrientation="portrait"
```
2. 手工处理旋转
```Java
android:configChanges="orientation|keyboardHidden|screenSize"
```
但是会调用onConfigurationChanged

当系统配置发生改变后，Activity会被销毁，其onPause, onStop, onDestroy均会被调用，同时由于Activity是在异常情况下终止的，系统会调用onSaveInstanceState来保存当前Activity的状态。这个方法的调用时机是在onStop之前，它和onPause没有既定的时序关系，它既可能在onPause之前调用，也可能在onPause之后调用。

onRestoreInstanceState的调用时机在onStart之后。

同时，我们要知道，在onSaveInstanceState和onRestoreInstanceState方法中，系统自动为我们做了一定的恢复工作。当Activity在异常情况下需要重新创建时，系统会默认为我们保存当前Activity的视图结构，并且在Activity重启后为我们恢复这些数据，比如文本框中用户输入的数据，ListView滚动的位置等，这些都是View自带的onSaveInstanceState和onRestoreInstanceState实现的。

关于保存和恢复View层次结构，系统的工作流程是这样的：首先Activity被意外终止时，Activity会调用onSaveInstanceState去保存数据，然后Activity会委托Window去保存数据，接着Window再委托它上面的顶级容易去保存数据。最后顶级容器再去一一通知它的子元素来保存数据。这是一种典型的委托思想，上层委托下层，比如View的绘制过程，事件分发都是类似思想。


## Activity生命周期的妙用
### 应用被系统回收后的处理
应用处于后台运行时，当android系统内存不足时会销毁后台的应用。
>一个应用，先后打开3个Activity，A->B->C，当该应用处于后台运行并被系统回收后，发生了什么？同时Android的进程回收机制是什么样的？

#### Android内存回收机制
当系统内存不足时，系统将激活内存回收过程。
回收优先级：
>IMPORTANCE_FOREGROUND:前台进程，目前正在屏幕上显示的进程和一些系统进程。

>IMPORTANCE_VISIBLE:可见进程，可见进程是一些不再前台，但用户依然可见的进程，比如输入法、天气、时钟等。

>IMPORTANCE_SERVICE:服务进程，拨号、邮件存储之类的。

>IMPORTANCE_BACKGROUND:后台进程，启动后被切换到后台的进程。

>IMPORTANCE_EMPTY:没有任何东西在内运行的进程，有些程序，比如BTE，在程序退出后，依然会在进程中驻留一个空进程，这个进程里没有任何数据在运行，作用往往是　　提高该程序下次的启动速度或者记录程序的一些历史信息。


## Q&A
### Android的Activity什么时候会调用onCreate()而不调用onStart()？
直接oncreate里就finish()就行


### onStart(),与onResume()有什么区别?
onStart()：在活动由不可见变为可见的时候调用;  onResume()在活动准备好和用户进行交互的时候调用，此时的活动一定位于返回栈的栈顶。


### onStart和onResume,onPause和onStop区别？
onStart和onStop是从Activity是否可见这个角度来回调的，而onResume和onPause是从Activity是否位于前台这个角度来回调

### 假设当前Activity A启动Activity B，那么两者的生命周期是什么样?
A: onPause ->  B onCreate onStart onResume ->A onStop


### 理解Activity，View,Window三者关系?
Activity像一个工匠（控制单元），Window像窗户（承载模型），View像窗花（显示视图）LayoutInflater像剪刀，Xml配置像窗花图纸。

1：Activity构造的时候会初始化一个Window，准确的说是PhoneWindow。   
2：这个PhoneWindow有一个“ViewRoot”，这个“ViewRoot”是一个View或者说ViewGroup，是最初始的根视图。   
3：“ViewRoot”通过addView方法来一个个的添加View。比如TextView，Button等   
4：这些View的事件监听，是由WindowManagerService来接受消息，并且回调Activity函数。比如onClickListener，onKeyDown等。  

Activity在onCreate之前调用attach方法，在attach方法中会创建window对象。window对象创建时并没有创建 DocerView 对象。用户在Activity中调用setContentView,然后调用window的setContentView，这时会检查DecorView是否存在，如果不存在则创建DecorView对象，然后把用户自己的 View  添加到 DecorView 中。

### 两个Activity之间传递数据，除了intent，广播接收者，contentprovider还有啥？
1. 利用 static 静态数据，public static 成员变量
2. 利用外部存储的传输: file, SharedPreferences, 数据库（外部存储需要注意读写冲突）
3. 事件总线

### Intent可以传递什么类型的参数
Intent 可以传递的数据类型非常的丰富， java的基本数据类型和 String以及他们的数组形式都可以，除此之外还可以传递实现了 Serializable 和 Parcelable 接口的对象。

Serializable和Parcelable的区别:   
这两个方法是用来序列化数据的。  
实现方式上不一样：一个只需要实现Serializable接口，并提供一个序列化版本id(serialVersionUID，避免反序列化失败)即可。另一个实现Parcelable接口之外还需要写几个方法。  
1. 在使用内存的时候，Parcelable 类比Serializable性能高，所以推荐使用Parcelable类。
2. Serializable在序列化的时候会产生大量的临时变量，从而引起频繁的GC。
3. Parcelable不能使用在要将数据存储在磁盘上的情况，因为Parcelable不能很好的保证数据的持续性在外界有变化的情况下。尽管Serializable效率低点， 也不提倡用，但在这种情况下，还是建议你用Serializable 。
4. Serializable是Java中就有的接口,Parcelable是Android中提供的新的序列化方式

### [关于获取当前Activity的一些思考](https://zhuanlan.zhihu.com/p/25221428?utm_source=qq&utm_medium=social)
**反射：** 反射是我们经常会想到的方法，思路大概为

1. 获取ActivityThread中所有的ActivityRecord
2. 从ActivityRecord中获取状态不是pause的Activity并返回

一个使用反射来实现的代码大致如下
```Java
public static Activity getActivity() {
    Class activityThreadClass = null;
    try {
        activityThreadClass = Class.forName("android.app.ActivityThread");
        Object activityThread = activityThreadClass.getMethod("currentActivityThread").invoke(null);
        Field activitiesField = activityThreadClass.getDeclaredField("mActivities");
        activitiesField.setAccessible(true);
        Map activities = (Map) activitiesField.get(activityThread);
        for (Object activityRecord : activities.values()) {
            Class activityRecordClass = activityRecord.getClass();
            Field pausedField = activityRecordClass.getDeclaredField("paused");
            pausedField.setAccessible(true);
            if (!pausedField.getBoolean(activityRecord)) {
                Field activityField = activityRecordClass.getDeclaredField("activity");
                activityField.setAccessible(true);
                Activity activity = (Activity) activityField.get(activityRecord);
                return activity;
            }
        }
    } catch (ClassNotFoundException e) {
        e.printStackTrace();
    } catch (NoSuchMethodException e) {
        e.printStackTrace();
    } catch (IllegalAccessException e) {
        e.printStackTrace();
    } catch (InvocationTargetException e) {
        e.printStackTrace();
    } catch (NoSuchFieldException e) {
        e.printStackTrace();
    }
    return null;
}
```
然而这种方法并不是很推荐，主要是有以下的不足：

* 反射通常会比较慢
* 不稳定性，这个才是不推荐的原因，Android框架代码存在修改的可能性，谁要无法100%保证mActivities，paused固定不变。所以可靠性不是完全可靠。

----


**Activity基类:**
既然反射不是很可靠，那么有一种比较可靠的方式，就是使用Activity基类。   
在Activity的onResume方法中，将当前的Activity实例保存到一个变量中。    
```Java
public class BaseActivity extends Activity{

    @Override
    protected void onResume() {
        super.onResume();
        MyActivityManager.getInstance().setCurrentActivity(this);
    }
}
```
然而，这一种方法也不仅完美，因为这种方法是基于约定的，所以必须每个Activity都继承BaseActivity，如果一旦出现没有继承BaseActivity的就可能有问题。

---
**回调方法:**
```Java
public class MyApplication extends Application {
    @Override
    public void onCreate() {
        super.onCreate();
        registerActivityLifecycleCallbacks(new ActivityLifecycleCallbacks() {
            @Override
            public void onActivityCreated(Activity activity, Bundle savedInstanceState) {

            }

            @Override
            public void onActivityStarted(Activity activity) {

            }

            @Override
            public void onActivityResumed(Activity activity) {
                MyActivityManager.getInstance().setCurrentActivity(activity);
            }

            @Override
            public void onActivityPaused(Activity activity) {

            }

            @Override
            public void onActivityStopped(Activity activity) {

            }

            @Override
            public void onActivitySaveInstanceState(Activity activity, Bundle outState) {

            }

            @Override
            public void onActivityDestroyed(Activity activity) {

            }
        });
    }
}
```
这种方法唯一的遗憾就是只支持API 14即其以上。
```Java
public class MyActivityManager {
    private static MyActivityManager sInstance = new MyActivityManager();
    private WeakReference<Activity> sCurrentActivityWeakRef;

    private MyActivityManager() {

    }

    public static MyActivityManager getInstance() {
        return sInstance;
    }

    public Activity getCurrentActivity() {
        Activity currentActivity = null;
        if (sCurrentActivityWeakRef != null) {
            currentActivity = sCurrentActivityWeakRef.get();
        }
        return currentActivity;
    }

    public void setCurrentActivity(Activity activity) {
        sCurrentActivityWeakRef = new WeakReference<Activity>(activity);
    }
}
```  
那么为什么要使用弱引用持有Activity实例呢？    
其实最主要的目的就是避免内存泄露，因为使用默认的强引用会导致Activity实例无法释放，导致内存泄露的出现。
