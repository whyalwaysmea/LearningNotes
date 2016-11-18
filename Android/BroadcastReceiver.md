# 广播机制简介
BroadcastReceiver也就是“广播接收者”的意思，顾名思义，它就是用来接收来自系统和应用中的广播。

在Android系统中，广播体现在方方面面，例如当开机完成后系统会产生一条广播，接收到这条广播就能实现开机启动服务的功能；当网络状态改变时系统会产生一条广播，接收到这条广播就能及时地做出提示和保存数据等操作；当电池电量改变时，系统会产生一条广播，接收到这条广播就能在电量低时告知用户及时保存进度，等等。

# 广播分类
Android中的广播主要可以分为两种类型：标准广播和有序广播。

* 标准广播是一种完全异步执行的广播，在广播发出之后，所有的广播接收器几乎都会在同一时刻接收到这条广播信息。

* 有序广播则是一种同步执行的广播，在广播发出之后，同一时刻只会有一个广播接收器能够收到这条广播消息，当这个广播接收器中的逻辑执行完毕之后，广播才会继续传播。所以此时的广播接收器是有先后顺序的，优先级高的广播接收器就可以先收到广播消息，并且前面的广播接收器还可以截断正在传播的广播。

# 注册方式
注册广播的方式一般有两种，在代码中注册和在AndroidManifest.xml中注册。其中前者被称为动态注册，后者也被成为静态注册。

在此我们以监听网络变化为例，分别使用一下这两种注册方式。
## 动态注册
1.创建一个类，继承BroadcastReceiver

```java
class NetworkChangeReceiver extends BroadcastReceiver {
    @Override
    public void onReceive(Context context, Intent intent) {
        Log.e("NetworkChangeReceiver", "onReceive");
    }
}
```

2.代码中注册广播

```java
IntentFilter intentFilter = new IntentFilter();
intentFilter.addAction("android.net.conn.CONNECTIVITY_CHANGE");
NetworkChangeReceiver networkChangeReceiver = new NetworkChangeReceiver();
registerReceiver(networkChangeReceiver, intentFilter);
```
我们创建了一个IntentFilter的实例，并给它添加了一个值为android.net.conn.CONNECTIVITY_CHANGE的action。因为当网络状态发生变化的时候，系统发出一条值为android.net.conn.CONNECTIVITY_CHANGE的广播，也就是说我们的广播接收器想要监听什么广播，就在这里添加相应的action就行了。
最后要记得，动态注册的广播接收器一定都要调用`unregisterReceiver()`取消注册才行。
