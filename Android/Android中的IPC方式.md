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
每个应用的SharedPreferences文件都可以在当前包所在的data目录下查看到。一般来说，它的目录位于/data/data/package name/share_prefs目录下。

## 使用Messenger
Messenger是一种轻量级的IPC方案，它的底层实现是AIDL。

由于它一次处理一个请求，因此在服务端我们不用考虑线程同步的问题，这是因为服务端中不存在并发执行的情形。正是因为Messenger是以串行的方式处理客户端发来的消息，如果大量的消息同时发送到服务端，服务端仍然只能一个个处理，如果有大量的并发请求，那么用Messenger就不太合适 了。同时，Messenger的作用主要是为了传递消息，很多时候我们可能需要跨进程调用服务端的方法，这种情形用Messenger就无法做到了。


## 使用AIDL（Android Interface Defined Language（安卓接口定义语言））
如上所说Messenger无法完成跨进程调用服务端的方法，而且不能处理并发请求。但是我们可以使用AIDL来实现跨进程的方法调用。AIDL也是Messenger的底层实现，因此Messenger本质上也是AIDL，只不过系统为我们做了封装从而方便上层的调用而已。

1.服务端
服务端首先要创建一个Service用来监听客户端的连接请求，然后创建一个AIDL文件，将暴露给客户端的接口在这个AIDL文件中声明，最后在Service中实现这个AIDL接口即可。

2.客户端
首先需要绑定服务端的Service，然后将服务端返回的Binder对象转成AIDL接口所属类型，最后调用AIDL中的方法。

3.AIDL接口的创建
默认情况下，AIDL 支持下列数据类型：
* Java 编程语言中的所有原语类型（如 int、long、char、boolean 等等）
* String
* CharSequence
* List
List 中的所有元素都必须是以上列表中支持的数据类型、其他 AIDL 生成的接口或您声明的可打包类型。 可选择将 List 用作“通用”类（例如，List<String>）。另一端实际接收的具体类始终是 ArrayList，但生成的方法使用的是 List 接口。
* Map
Map 中的所有元素都必须是以上列表中支持的数据类型、其他 AIDL 生成的接口或您声明的可打包类型。 不支持通用 Map（如 Map<String,Integer> 形式的 Map）。 另一端实际接收的具体类始终是 HashMap，但生成的方法使用的是 Map 接口。
* Parcelable
所有实现了Parcelable接口的对象。

*注意：* 如果AIDL文件中用到了自定义的Parcelable对象，那么必须新建一个和它同名的AIDL文件，并在其中声明它为Parcelable类型。
```java
// Book.aidl
package com.whyalwaysmea.ipc;

parcelable Book;
```

4.远程服务端Service的实现
```java
public class BookManagerService extends Service {

    // CopyOnWriteArrayList支持并发读/写
    // 因为AIDL方法是在服务端的Binder线程池中执行的，因此当多个客户端同时连接的时候，会存在多个线程同时访问的情形
    // 所以我们要在AIDL方法中处理线程同步
    private CopyOnWriteArrayList<Book> mBookList = new CopyOnWriteArrayList<>();

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return mBinder;
    }

    private Binder mBinder = new IBookManager.Stub(){

        @Override
        public List<Book> getBookList() throws RemoteException {
            return mBookList;
        }

        @Override
        public void addBook(Book book) throws RemoteException {
            mBookList.add(book);
        }
    };

    @Override
    public void onCreate() {
        super.onCreate();
    }
}
```

5.客户端的实现
首先要绑定远程服务，绑定成功后将服务端返回的Binder对象转换成AIDL接口，然后就可以通过这个接口去调用服务端的远程方法了。
```java
public class BookManagerActivity extends AppCompatActivity {

    private ServiceConnection mConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            IBookManager bookManager = IBookManager.Stub.asInterface(service);
            try {
                List<Book> bookList = bookManager.getBookList();
                System.out.println("book  list type : " + bookList.getClass().getCanonicalName());
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {

        }
    };

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_book_manager);
        Intent intent = new Intent(this, BookManagerService.class);
        bindService(intent, mConnection, Context.BIND_AUTO_CREATE);
    }

    @Override
    protected void onDestroy() {
        unbindService(mConnection);
        super.onDestroy();
    }
}
```
可能有人会发现，在Service中，我们返回的数据类型是CopyOnWriteArrayList，但是到了Activity中，接收到了却是List。这是为什么呢？

AIDL中能够使用的List只有ArrayList，但是我们这里却使用了CopyOnWriteArrayList（注意它并不是继承自ArrayList）。这是因为AIDL中所支持的是抽象的List，而List只是一个接口，因此虽然服务端返回的是CopyOnWriteArrayList，但是在Binder中会按照List的规范去访问数据并最终形成一个ArrayList传递给客户端。


### RemoteCallbackList
RemoteCallbackList是系统专门提供的用于删除跨进程Listener的接口。RemoteCallbackList是一个泛型，支持管理任意的AIDL接口，因为所有的AIDL接口都继承自IInterface接口。

### 注意
客户端调用远程服务的方法，被调用的方法运行在服务端的Binder线程池中，同时客户端线程会被挂起，这个时候如果服务端方法执行比较耗时，就会导致客户端线程长时间地阻塞，如果这个客户端线程是UI线程的话，就会导致客户端ANR。所以，如果我们明确知道某个远程方法是耗时的，那么就要避免在客户端的UI线程中去访问远程的耗时方法。

同理，当远程服务端需要调用客户端的方法时，被调用的方法也运行在Binder线程池中，只不过是客户端的线程池。所以，我们同样不可以在服务端中调用客户端的耗时方法。

Binder是可能意外死亡的，由于服务端进程意外停止了，这时我们需要重新连接服务。

1. 给Binder设备DeathRecipient监听
2. 在onServiceDisconnected中重连远程服务。

两者的区别是：onServiceDisconnected在客户端的UI线程中被回调，而binderDied在客户端的Binder线程池中被回调。

### 权限验证
默认情况下，我们的远程服务任何人都可以连接，但这以你哥哥不是我们愿意看到的，所以我们必须加入权限验证功能。

1.在onBind中进行验证，验证不通过就直接返回null。比如使用permission验证
```
<permission
        android:name="com.whyalwaysmea.permission.ACCESS_BOOK_SERVICE"
        android:protectionLevel="normal" />

public IBinder onbind(Intent intent) {
    int check = checkCallingOrSelfPermission("com.whyalwaysmea.permission.ACCESS_BOOK_SERVICE");
    if(check == PackageManager.PERMISSION_DENIED) {
        return null;
    }
    return mBinder;
}        
```
如果我们自己的应用想绑定到服务上，只需要在它的AndroidMenifest文件中采用如下方式使用permission
```
<uses-permission android:name="com.whyalwaysmea.permission.ACCESS_BOOK_SERVICE" />
```

2.我们可以在服务端的onTransact方法中进行权限验证，如果验证失败就返回false，这样服务端就不会终止执行AIDL中的方法从而达到保护客户端的效果。这里可以验证的方式很多，比如permission，Uid，Pid等验证。

## 使用ContentProvider
ContentProvider是Android中提供的专门用于不同应用间进行数据共享的方式，它的底层实现同样也是Binder。

系统预置了许多ContentProvider，比如通讯录、日程表等，要跨进程访问这些信息，只需要通过ContentProvider的query、update、insert和delete方法即可。

关于自定义ContentProvider的过程：

1. 创建一个自定义的ContentProvider，名字就叫BookProvider，只需要继承ContentProvider类并实现六个抽象方法即可：onCreate、query、update、insert、delete和geType。除了onCreate由系统回调并运行在主线程里，其他五个方法均由外界回调并运行在Binder线程池中。

2. 注册ContentProvider。

3. 通过外部应用访问BookProvider。



## 使用Socket
