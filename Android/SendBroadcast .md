# 发送自定义广播
广播主要分为两种类型，标准广播和有序广播。

1.定义一个广播接收器
```Java
class MyBroadcastReceiver extends BroadcastReceiver {
    @Override
    public void onReceive(Context context, Intent intent) {
        Log.e("MyBroadcastReceiver", "onReceive");
    }
}
```
2.对广播进行注册
```xml
<receiver android:name=".MyBroadcastReceiver">
    <intent-filter>
        <action android:name="com.whyalwaysmea.broadcasttest" />
    </intent-filter>
</receiver>
```
3.发送广播
```Java
Intent intent = new Intent("com.whyalwaysmea.broadcasttest");
// 无序广播
sendBroadcast(intent);
// 有序广播
sendOrderedBroadcast(intent, null);
```
可以看到，发送有序广播和无序广播只有一行代码不一样。sendOrderedBroadcast()方法传入了两个参数，第一个参数仍然是Intent，第二个参数是一个与权限相关的字符串，这里传入null就可以了。
对于有序广播而言，广播接收器是有先后顺序的，而且前面的广播接收器还可以将广播截断，以阻止其继续传播。

设置广播接收器优先级:
```xml
<receiver android:name=".MyBroadcastReceiver">
    <intent-filter android:priority="100">
        <action android:name="com.whyalwaysmea.broadcasttest" />
    </intent-filter>
</receiver>
```
我们通过android:priority属性给广播接收器设置了优先级，数值越大的优先级越高，优先级比较高的广播接收器就可以先收到广播。动态注册优先级高于静态注册

如果在onReceive()方法中调用了`abortBroadcast()`方法，就表示将这条广播截断，后面的广播接收器将无法接收到这条广播。

# 发送本地广播
普通情况下所发送的广播都是属于系统全局广播，即发出去的广播可以被其他任何的任何应用程序接收到，并且我们也可以接收到来自于其他任何应用程序的广播。这样就很容易引发安全性的问题。
为了能够简单地解决广播的安全性问题，Android引入了一套本地广播机制，使用这个机制发出的广播只能够在应用程序的内部进行传递，并且广播接收器也只能接收来自本应用程序发出的广播。

1.定义一个广播接收器
```Java
class LocalBroadcastReceiver extends BroadcastReceiver {
    @Override
    public void onReceive(Context context, Intent intent) {
        Log.e("LocalBroadcastReceiver", "onReceive");
    }
}
```

2.使用LocalBroadcastManager来注册本地广播监听器
```java
LocalBroadcastManager localBroadcastManager = LocalBroadcastManager.getInstance(this);
IntentFilter intentFilter = new IntentFilter();
intentFilter.addAction("com.whyalwaysmea.localbroadcasttest");

LocalBroadcastReceiver localReceiver = new LocalBroadcastReceiver();
localBroadcastManager.registerReceiver(localReceiver, intentFilter);
```

3.使用LocalBroadcastManager发送本地广播
```Java
Intent intent = new Intent("com.whyalwaysmea.localbroadcasttest");
localBroadcastManager.sendBroadcast(intent);
```

*注意：* 本地广播是无法通过静态注册的方式来接收的。

使用本地广播的优点：
1. 可以明确地知道正在发送的广播不会离开我们的程序，因此不需要担心机密数据泄漏的问题
2. 其他的程序无法将广播发送到我们程序的内部，因此不需要担心会有安全漏洞的隐患
3. 发送本地广播比起发送全局广播将会更加高效。
