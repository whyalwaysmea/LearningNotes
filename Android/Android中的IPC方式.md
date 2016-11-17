## 使用Bundle
四大组件中的三大组件(Activity、Service、Receiver)都是支持在Intent中传递Bundle数据的。
所以我们在一个进程中启动了另外一个进程的Activity、Service、Receiver，我们就可以在Bundle中附加我们需要传输给远程进程的信息并通过Intent发送出去。
```java
Intent intent = new Intent(BundleFirstActivity.this, BundleSecondActivity.class);
intent.putExtra("User", new User("haha", 222));
startActivity(intent);
```


特殊场景：
当A进程计算出来了一个结果，需要传输给B进程，但是传输的数据不支持放入Bundle中，因此无法通过Intent来传输。这个时候我们可以通过Intent启动进程B的一个Service组件(比如IntentService)，让Service在后台计算，计算完成后再启动B进程中真要需要的目标组件。

## 使用文件共享
共享文件也是一种不错的进程间通信方式，两个进程通过读/写同一个文件来交换数据，比如A进程把数据写入文件，B进程通过读取这个文件来获取数据。

需要注意的是并发情况下的读/写，可能造成数据错乱。
可能有人会想到SharedPreferences，但是SharedPreferences是个特例。从本质上来说，SharedPreferences也属于文件的一种，但是由于系统对它的读/写有一定的缓存策略，即在内存中会有一个SharedPreferences文件的缓存，因此在多进程模式下，系统对它的读/写就变得不可靠，当面对高并发的读/写访问，SharedPreferences有很大几率会丢失数据，因此，不建议在进程间通信中使用SharedPreferences。

## 使用Messenger
Messenger是一种轻量级的IPC方案，它的底层实现是AIDL。

由于它一次处理一个请求，因此在服务端我们不用考虑线程同步的问题，这是因为服务端中不存在并发执行的情形。正是因为Messenger是以串行的方式处理客户端发来的消息，如果大量的消息同时发送到服务端，服务端仍然只能一个个处理，如果有大量的并发请求，那么用Messenger就不太合适 了。同时，Messenger的作用主要是为了传递消息，很多时候我们可能需要跨进程调用服务端的方法，这种情形用Messenger就无法做到了。

## 使用AIDL


## 使用ContentProvider
ContentProvider是Android中提供的专门用于不同应用间进行数据共享的方式，它的底层实现同样也是Binder。

系统预置了许多ContentProvider，比如通讯录、日程表等，要跨进程访问这些信息，只需要通过ContentProvider的query、update、insert和delete方法即可。

关于自定义ContentProvider的过程：

1. 创建一个自定义的ContentProvider，名字就叫BookProvider，只需要继承ContentProvider类并实现六个抽象方法即可：onCreate、query、update、insert、delete和geType。除了onCreate由系统回调并运行在主线程里，其他五个方法均由外界回调并运行在Binder线程池中。

2. 注册ContentProvider。

3. 通过外部应用访问BookProvider。



## 使用Socket
