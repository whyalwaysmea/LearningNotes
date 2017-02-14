# Service

## Service的启动方式
1. startService();开启服务
开启服务后，服务就会长期的后台运行，即使调用者退出了，服务仍然在后台继续运行。服务和调用者没有什么关系，调用者是不可以访问服务里面的方法。
2. bindService();绑定服务
服务开启后，生命周期与调用者相关联。调用者挂了，服务也会跟着挂掉，不求同时生，但求同时死。调用者和服务绑定在一起，调用者可以间接的调用到服务里面的方法。


## Service和Thread的关系
不少Android初学者都可能会有这样的疑惑，Service和Thread到底有什么关系呢？什么时候应该用Service，什么时候又应该用Thread？答案可能会有点让你吃惊，因为Service和Thread之间没有任何关系！

之所以有不少人会把它们联系起来，主要就是因为Service的后台概念。Thread我们大家都知道，是用于开启一个子线程，在这里去执行一些耗时操作就不会阻塞主线程的运行。而Service我们最初理解的时候，总会觉得它是用来处理一些后台任务的，一些比较耗时的操作也可以放在这里运行，这就会让人产生混淆了。但是，如果我告诉你Service其实是运行在主线程里的，你还会觉得它和Thread有什么关系吗？让我们看一下这个残酷的事实吧。

在MainActivity的onCreate()方法里加入一行打印当前线程id的语句：
```java
Log.d("MyService", "MainActivity thread id is " + Thread.currentThread().getId());
```
然后在MyService的onCreate()方法里也加入一行打印当前线程id的语句：
```java
Log.d("MyService", "MyService thread id is " + Thread.currentThread().getId());
```
会看到如下打印日志：
![](http://img.blog.csdn.net/20131029231358812?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZ3VvbGluX2Jsb2c=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

可以看到，它们的线程id完全是一样的，由此证实了Service确实是运行在主线程里的，也就是说如果你在Service里编写了非常耗时的代码，程序必定会出现ANR的。

你可能会惊呼，这不是坑爹么！？那我要Service又有何用呢？其实大家不要把后台和子线程联系在一起就行了，这是两个完全不同的概念。Android的后台就是指，它的运行是完全不依赖UI的。即使Activity被销毁，或者程序被关闭，只要进程还在，Service就可以继续运行。比如说一些应用程序，始终需要与服务器之间始终保持着心跳连接，就可以使用Service来实现。你可能又会问，前面不是刚刚验证过Service是运行在主线程里的么？在这里一直执行着心跳连接，难道就不会阻塞主线程的运行吗？当然会，但是我们可以在Service中再创建一个子线程，然后在这里去处理耗时逻辑就没问题了。

额，既然在Service里也要创建一个子线程，那为什么不直接在Activity里创建呢？这是因为Activity很难对Thread进行控制，当Activity被销毁之后，就没有任何其它的办法可以再重新获取到之前创建的子线程的实例。而且在一个Activity中创建的子线程，另一个Activity无法对其进行操作。但是Service就不同了，所有的Activity都可以与Service进行关联，然后可以很方便地操作其中的方法，即使Activity被销毁了，之后只要重新与Service建立关联，就又能够获取到原有的Service中Binder的实例。因此，使用Service来处理后台任务，Activity就可以放心地finish，完全不需要担心无法对后台任务进行控制的情况。

## Service的生命周期
一旦在项目的任何位置调用了context的`startService();`方法，相应的服务就会启动起来，并回调`onStartCommand()`方法。如果这个服务之前还没有创建过，`onCreate()`方法会先于`onStartCommand()`方法执行。直到`stopService()`或`stopSelf()`方法被调用。
注意：虽然每调用用一次`startService()`方法，`onStartCommand()`就会执行一次，但实际上每个服务都只会存在一个实例。所以不管调用了多少次startService()方法，只需要调用一次`stopService()`或`stopSelf()`方法，服务就会停止下来了。

还可以调用context的`bindService()`来获取一个服务的持久连接，这时就会回调服务中的`onBind()`方法。

当调用了`startService()`方法后，又去调用`stopService()`方法，这时服务中的`onDestroy()`方法就会执行，表示服务已经销毁了。
类似地，当调用了`bindService()`方法后，又去调用`unbindService()`方法，`onDestroy()`方法也会执行。
如果既调用了`startService()`方法，又调用了`bindService()`方法的，需要同时调用`stopService()`和`unbindService()`方法，`onDestroy()`才会执行。

![](http://upload-images.jianshu.io/upload_images/2243690-fe9f0af59a4c59a9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## Q&A
### Android Service与Activity之间通信的几种方式?
* 通过Binder对象。Activity中调用bindService()方法将Activity和Service进行绑定，并new一个ServiceConnection。Service中
* 通过broadcast(广播)的形式


### 如何保证service在后台不被Kill?
一、onStartCommand方法，返回START_STICKY。当service因内存不足被kill，当内存又有的时候，service又被重新创建，比较不错，但是不能保证任何情况下都被重建，比如进程被干掉了....
* START_STICKY 在运行onStartCommand后service进程被kill后，那将保留在开始状态，但是不保留那些传入的intent。不久后service就会再次尝试重新创建，因为保留在开始状态，在创建     service后将保证调用onstartCommand。如果没有传递任何开始命令给service，那将获取到null的intent。

* START_NOT_STICKY 在运行onStartCommand后service进程被kill后，并且没有新的intent传递给它。Service将移出开始状态，并且直到新的明显的方法（startService）调用才重新创建。因为如果没有传递任何未决定的intent那么service是不会启动，也就是期间onstartCommand不会接收到任何null的intent。

* START_REDELIVER_INTENT 在运行onStartCommand后service进程被kill后，系统将会再次启动service，并传入最后一个intent给onstartCommand。直到调用stopSelf(int)才停止传递intent。如果在被kill后还有未处理好的intent，那被kill后服务还是会自动启动。因此onstartCommand不会接收到任何null的intent。

二、提升service优先级
在AndroidManifest.xml文件中对于intent-filter可以通过android:priority = "1000"这个属性设置最高优先级，1000是最高值，如果数字越小则优先级越低，同时适用于广播。

三、提升service进程优先级
Android中的进程是托管的，当系统进程空间紧张的时候，会依照优先级自动进行进程的回收。Android将进程分为6个等级,它们按优先级顺序由高到低依次是:

1. 前台进程( FOREGROUND_APP)
2. 可视进程(VISIBLE_APP )
3. 次要服务进程(SECONDARY_SERVER )
4. 后台进程 (HIDDEN_APP)
5. 内容供应节点(CONTENT_PROVIDER)
6. 空进程(EMPTY_APP)

当service运行在低内存的环境时，将会kill掉一些存在的进程。因此进程的优先级将会很重要，可以使用startForeground 将service放到前台状态。这样在低内存时被kill的几率会低一些。
```java
public void onCreate() {  
    super.onCreate();  
    Notification notification = new Notification(R.drawable.ic_launcher,  
            "有通知到来", System.currentTimeMillis());  
    Intent notificationIntent = new Intent(this, MainActivity.class);  
    PendingIntent pendingIntent = PendingIntent.getActivity(this, 0,  
            notificationIntent, 0);  
    notification.setLatestEventInfo(this, "这是通知的标题", "这是通知的内容",  
            pendingIntent);  
    startForeground(1, notification);  
    Log.d(TAG, "onCreate() executed");  
}  
```

四、onDestroy方法里重启service
service +broadcast  方式，就是当service走ondestory的时候，发送一个自定义的广播，当收到广播的时候，重新启动service；

五、监听系统广播判断Service状态
通过系统的一些广播，比如：手机重启、界面唤醒、应用状态改变等等监听并捕获到，然后判断我们的Service是否还存活，别忘记加权限啊。
