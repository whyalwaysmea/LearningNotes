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
