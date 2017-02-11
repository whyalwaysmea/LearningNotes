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
Android的Activity什么时候会调用onCreate()而不调用onStart()？
>直接oncreate里就ondestroy就行


onStart(),与onResume()有什么区别?
>onStart()：在活动由不可见变为可见的时候调用;  onResume()在活动准备好和用户进行交互的时候调用，此时的活动一定位于返回栈的栈顶。


onStart和onResume,onPause和onStop区别？
>onStart和onStop是从Activity是否可见这个角度来回调的，而onResume和onPause是从Activity是否位于前台这个角度来回调

假设当前Activity A启动Activity B，那么两者的生命周期是什么样?
>A: onPause ->  B onCreate onStart onResume ->A onStop
