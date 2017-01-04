## Serializable接口
Serializable是Java所提供的一个序列化接口，它是一个空接口，为对象提供标准的序列化和反序列化操作。
其中有以下三点需要注意：
1. 静态成员变量属于类不属于对象，所以不会参与序列化过程
2. 用transient关键字标记的成员变量不参与序列化过程
3. 我们应该手动指定serialVersionUID的值，否则可能导致反序列化的失败。
`private static final long serialVersionUID = 6513246845613L;`

## Parcelable接口
Parcelable也是一个接口，只要实现这个接口，一个类的对象就可以实现序列化并可以通过Intent和Binder传递。



## Serializable和Parcelable区别
Serializable是Java中的序列化接口，其使用起来简单但是开销很大，序列化和反序列化过程需要大量I/O操作。而Parcelable是Android中的序列化方式，因此更适合用在Android平台上，它的缺点就是使用起来稍微麻烦点，但是它的效率很高。


## Binder
直观来说，Binder是Android中的一个类，它实现了IBinder接口。
Android开发中，Binder主要用在Service中，包括AIDL和Messenger，其中普通Service中的Binder不涉及进程间通信，所以较为简单，无法触及Binder的核心，而Messenger的底层其实是AIDL。

所有可以在Binder中传输的接口都需要继承IInterface接口。


aidl工具根据aidl文件自动生成的java接口的解析：首先，它声明了几个接口方法，同时还声明了几个整型的id用于标识这些方法，id用于标识在transact过程中客户端所请求的到底是哪个方法；接着，它声明了一个内部类Stub，这个Stub就是一个Binder类，当客户端和服务端都位于同一个进程时，方法调用不会走跨进程的transact过程，而当两者位于不同进程时，方法调用需要走transact过程，这个逻辑由Stub内部的代理类Proxy来完成。

所以，这个接口的核心就是它的内部类Stub和Stub内部的代理类Proxy。 下面分析其中的方法：

1. asInterface(android.os.IBinder obj)：用于将服务端的Binder对象转换成客户端所需的AIDL接口类型的对象，这种转换过程是区分进程的，如果客户端和服务端是在同一个进程中，那么这个方法返回的是服务端的Stub对象本身，否则返回的是系统封装的Stub.Proxy对象。
2. asBinder：返回当前Binder对象。
3. onTransact：这个方法运行在服务端中的Binder线程池中，当客户端发起跨进程请求时，远程请求会通过系统底层封装后交由此方法来处理。
这个方法的原型是public Boolean onTransact(int code, Parcelable data, Parcelable reply, int flags) 服务端通过code可以知道客户端请求的目标方法，接着从data中取出所需的参数，然后执行目标方法，执行完毕之后，将结果写入到reply中。如果此方法返回false，说明客户端的请求失败，利用这个特性可以做权限验证(即验证是否有权限调用该服务)。
4. Proxy#[Method]：代理类中的接口方法，这些方法运行在客户端，当客户端远程调用此方法时，它的内部实现是：首先创建该方法所需要的参数，然后把方法的参数信息写入到_data中，接着调用transact方法来发起RPC请求，同时当前线程挂起；然后服务端的onTransact方法会被调用，直到RPC过程返回后，当前线程继续执行，并从_reply中取出RPC过程的返回结果，最后返回_reply中的数据。

如果搞清楚了自动生成的接口文件的结构和作用之后，其实是可以不用通过AIDL而直接实现Binder的，


Binder的两个重要方法linkToDeath和unlinkToDeath

Binder运行在服务端，如果由于某种原因服务端异常终止了的话会导致客户端的远程调用失败，所以Binder提供了两个配对的方法linkToDeath和unlinkToDeath，通过linkToDeath方法可以给Binder设置一个死亡代理，当Binder死亡的时候客户端就会收到通知，然后就可以重新发起连接请求从而恢复连接了。
如何给Binder设置死亡代理呢？

1.声明一个DeathRecipient对象，DeathRecipient是一个接口，其内部只有一个方法bindeDied，实现这个方法就可以在Binder死亡的时候收到通知了。
```Java
private IBinder.DeathRecipient mDeathRecipient = new IBinder.DeathRecipient() {
    @Override
    public void binderDied() {
        if (mRemoteBookManager == null) return;
        mRemoteBookManager.asBinder().unlinkToDeath(mDeathRecipient, 0);
        mRemoteBookManager = null;
        // TODO:这里重新绑定远程Service
    }
};
```
2.在客户端绑定远程服务成功之后，给binder设置死亡代理
```Java
mRemoteBookManager.asBinder().linkToDeath(mDeathRecipient, 0);
```
