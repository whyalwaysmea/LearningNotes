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
一旦在项目的任何位置调用了context的`startService();`方法，相应的服务就会启动起来，并回调`onStartCommand()`方法。如果这个服务之前还没有创建过，`onCreate()`方法会先于`onStartCommand()`方法执行。当`stopService()`或`stopSelf()`方法被调用。
注意：虽然每调用用一次`startService()`方法，`onStartCommand()`就会执行一次，但实际上每个服务都只会存在一个实例。所以不管调用了多少次startService()方法，只需要调用一次`stopService()`或`stopSelf()`方法，服务就会停止下来了。

还可以调用context的`bindService()`来获取一个服务的持久连接，这时就会回调服务中的`onBind()`方法。

当调用了`startService()`方法后，又去调用`stopService()`方法，这时服务中的`onDestroy()`方法就会执行，表示服务已经销毁了。
类似地，当调用了`bindService()`方法后，又去调用`unbindService()`方法，`onDestroy()`方法也会执行。
如果既调用了`startService()`方法，又调用了`bindService()`方法的，需要同时调用`stopService()`和`unbindService()`方法，`onDestroy()`才会执行。

![](http://upload-images.jianshu.io/upload_images/2243690-fe9f0af59a4c59a9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
