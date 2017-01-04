# Android IPC简介
IPC是Inter-Process Communication的缩写，含义为进程间通信或者跨进程通信。

## 进程和线程
按照操作系统中的描述，
>线程是CPU调度的最小单元，同时线程是一种有限的系统资源。
而进程一般指一个执行单元，在PC和移动设备上指一个程序或者一个应用
一个进程可以包含多个线程，因为进程和线程是包含与被包含的关系。

## 比较
IPC不是Android中所独有的，任何一个操作系统都需要有相应的IPC机制。

Windows上可以通过剪贴板、管道和邮槽来进行进程间通信；

Linux上可以通过命名管道、共享内存、信号量等来进行进程间通信；

对于Android来说，它是一种基于Linux内核的移动操作系统，它的进程间通信方式并不能完全继承自Linux，相反，它有自己的进程间通信方式：
>在Android中最有特色的进程间通信方式就是Binder了，除了Binder，Android还支持Socket，文件共享，AIDL，Messenger、ContentProvider、Bundle。


# Android中的多进程模式
我们必须要先理解Android中的多进程模式。通过给四大组件指定`android:process`属性，我们可以轻易地开启多进程模式。
也就是说我们无法给一个线程或者一个实体类指定其运行时所在的进程。当然，我们也可以通过JNI在native层去fork一个新的进程。
```xml
<activity
    android:name=".MainActivity"
    android:configChanges="orientation|screenSize"
    android:label="@string/app_name"
    android:launchMode="standard" >
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
    </intent-filter>
</activity>
<activity
    android:name=".SecondActivity"
    android:configChanges="screenLayout"
    android:label="@string/app_name"
    android:process=":remote" />
<activity
    android:name=".ThirdActivity"
    android:configChanges="screenLayout"
    android:label="@string/app_name"
    android:process="com.haha.remote" />
```
我们可以在AndroidMenifest中指定android:process属性。在此可以看出SecondActivity和ThirdActivity的android:process属性分别为：“:remote”和"com.haha.remote"。
那么这两种方式的区别是什么呢？

1. ":"的含义是指要在当前的进程名前面附加上当面的属性值，这是一种简写的方法。 假如我们完整的包名是`com.haha`，那么对于SecondActivity来说，它完整的进程名为`"com.haha:remote"`

2. 对于ThirdActivity中的声明方式，它是一种完整的命名方式。
进程名以":"开头的进程属于当前应用的私有进程，其他应用的组件不可以和它跑在同一个进程中，而进程名不以":"开头的进程属于全局进程，其他应用通过ShareUID方式可以和它跑在同一个进程中。

我们知道Android系统会为每个应用分配一个唯一的UID，具有相同UID的应用才能共享数据。
这里要说明的是，两个应用通过ShareUID跑在同一个进程中是有要求的，需要这两个应用有相同的ShareUID并且签名相同才可以。在这种情况下，它们可以互相访问对方的私有数据，比如data目录、组件信息等，不管它们是否跑在同一个进程中。当然如果它们跑在同一个进程中，那么除了能共享data目录、组件信息，还可以共享内存数据，或者说它们看起来就像是一个应用的两个部分。

## 多进程运行模式
一般来说，使用多进程会造成如下几方面的问题：

1. 静态成员和单例模式完全失效
Android为每一个应用分配了一个独立的虚拟机，或者说为每个进程都分配一个独立的虚拟机，不同的虚拟机在内存分配上有不同的地址控件，这就导致在不同的虚拟机中访问同一个类的对象会产生多份副本。正常情况下，四大组件中间不可能不通过一些中间层来共享数据。

2. 线程同步机制完全失效
本质上如果1.既然都不是一块内存了，那么不管是锁对象还是锁全局类都无法保证线程同步，因为不同进程锁的不是同一个对象。

3. SharedPreferences的可靠性下降
因为SharedPreferences不支持两个进程同时去执行写操作，SharedPreferences底层是通过读/写xml文件实现的，并发显然是可能出问题的。

4. Application会多次创建
不同进程的组件会拥有独立的虚拟机、Application以及内存空间。
