## Android消息机制概括
Android的消息机制主要是指Handler的运行机制，Handler的运行需要底层的MessageQueue和Looper的支撑。

**Message** 是在线程之间传递的消息，它可以在内部携带少量的信息，用于在不同线程之间交换数据。

**Handler** 顾名思义也就是处理者的意思，它主要用于发送和处理消息的。发送消息一般会使用Handler的sendMessage()方法，而发出的消息经过一些列地辗转处理后，最终会传递到Handler的handleMessage()方法中。

**MessageQueue** 是消息队列的意思，它主要用于存放所有通过Handler发送的消息。这部分消息会一直存在于消息队列中，等待被处理。每个线程中只会有一个MessageQueue对象。（MessageQueue是在Looper的构造函数中创建的，因此一个MessageQueue对应一个Looper）

**Looper** 是每个线程中MessageQueue的管家，调用Looper的loop()方法后，就会进入到一个无限循环当中，然后每当发现MessageQueue中存在一条消息，就会将它取出，并传递到Handler的handleMessage()方法中。每个线程中也会只有一个Looper对象。

MessageQueue是以**单链表**的数据结构存储消息列表，但是以队列的形式对外提供插入和删除消息操作的消息队列。
MessageQueue只是消息的存储单元，它不能去处理消息。而Looper则是以无限循环的形式去查找是否有新消息，如果有的话就去处理消息，否则就一直等待着。

Looper中还有一个特殊的概念那就是ThreadLocal，ThreadLocal并不是线程，它的作用是可以在每个线程中存储数据。
Handler创建的时候会采用当前线程的Looper来构造消息循环系统，那么Handler内部如何获取到当前线程的Looper呢？这就要使用ThreadLocal了，ThreadLocal可以在不同的线程中互不干扰地存储并提供数据，通过ThreadLocal可以轻松获取 每个线程的Looper。线程是默认没有Looper的，如果需要使用Handler就必须为线程创建Looper。主线程就是ActivityThread，ActivityThread被创建的时候就会初始化Looper。

Handler的创建会采用当前线程的Looper来构建内部的消息循环系统，如果当前线程中不存在Looper的话就会报错。Handler可以用post方法将一个Runnable投递到消息队列中，也可以用send方法发送一个消息投递到消息队列中，其实post最终还是调用了send方法。当Handler的send方法被调用时，它会调用MessageQueue的enqueueMessage方法将这个消息放入消息队列中，然后Looper发现有新消息到来时，就会处理这个消息，最终消息中的Runnable或者Handler的handleMessage方法就会被调用。

`为什么不允许子线程访问UI呢？`
Android规定UI操作只能在主线程中进行，ViewRootImpl的checkThread方法会验证当前线程是否可以进行UI操作。
这是因为UI组件不是线程安全的，如果在多线程中并发访问可能会导致UI组件处于不可预期的状态。另外，如果对UI组件的访问进行加锁机制的话又会降低UI访问的效率，所以还是采用单线程模型来处理UI事件。
如果在其它线程访问UI线程，Android提供了以下的方式：

1. Activity.runOnUiThread(Runnable)
2. View.post(Runnable)
3. View.postDelayed(Runnable, long)
4. Handler


## Android消息机制分析
### ThreadLocal的工作原理
ThreadLocal是一个线程内部的数据存储类，通过它可以在指定的线程中存储数据，数据存储以后，只有在指定线程中可以获取到存储的数据，对于其他线程来说则无法获取到数据。
对于Handler来说，它需要获取当前线程的Looper，而Looper的作用域就是线程并且不同线程具有不同的Looper，这个时候通过ThreadLocal就可以实现Looper在线程中的存取了。
**ThreadLocal的原理：** 不同线程访问同一个ThreadLocal的get方法时，ThreadLocal内部会从各自的线程中取出一个数组，然后再从数组中根据当前ThreadLocal的索引去查找出对应的value值，不同线程中的数组是不同的，这就是为什么通过ThreadLocal可以在不同线程中维护一套数据的副本并且彼此互不干扰。
下面分析ThreadLocal的内部实现。ThreadLocal是一个泛型类`public class ThreadLocal<T>`，我们只要弄清楚它的get和set方法就好。
```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```
ThreadLocalMap是Thread类内部专门用来存储线程的ThreadLocal数据的，它内部有一个数组private Entry[] table，ThreadLocal的值就存在这个table数组中。
```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null)
            return (T)e.value;
    }
    return setInitialValue();
}
```
从get和set方法可以看出，它们所操作的对象都是用当前线程所进行了区分的，这就是为什么ThreadLocal可以在多个线程中互不干扰的存储和修改数据。

### MessageQueue的工作原理
MessageQueue其实是通过单链表来维护消息列表的，它包含两个主要操作`enqueueMessage`和`next`，前者是插入消息，后者是读取出一条消息并移除。
单链表在插入和删除上比较有优势。
next方法是一个无限循环的方法，如果消息队列中没有消息，那么next方法会一直阻塞在这里。当有新消息到来时，next方法会返回这条消息并将它从链表中移除。

### Looper的工作原理
Looper在Android的消息机制中扮演中消息循环的角色，它会不停地 从MessageQueue中查看是否有新消息，如果有新消息就会立刻处理，否则就一直阻塞在那里。
```java
// Looper的构造方法。
private Looper(boolean quitAllowed) {
    // 创建一个MessageQueue
    mQueue = new MessageQueue(quitAllowed);
    // 保存当前线程的对象
    mThread = Thread.currentThread();
}
```
```java
// 为一个线程创建Looper的方法
new Thread("test"){
    @Override
    public void run() {
        //创建looper
        Looper.prepare();
        //可以创建handler了
        Handler handler = new Handler();
        //开始looper循环
        Looper.loop();
    }
}.start();
```

1. Looper的prepareMainLooper方法主要是给主线程也就是ActivityThread创建Looper使用的，本质也是调用了prepare方法。
2. Looper还提供了getMainLooper方法，通过它可以在任何地方获取到主线程的looper。
3. Looper提供了quit和quitSafely来退出一个Looper,区别是：前者会直接退出Looper，后者只是设定一个退出标记，然后把消息队列中的已有消息处理完毕后才安全地退出。Looper退出之后，通过Handler发送的消息就会失败，这个时候Handler的send方法会返回false。在子线程中，如果手动为其创建了Looper，那么在所有的事情完成以后应该调用quit方法来终止消息循环，否则这个子线程就会一直处于等待的状态，而如果退出Looper以后，这个线程就会立刻终止，因此建议不需要的时候终止Looper。
4. Looper最重要的方法是loop，只有调用了loop后，消息循环系统才会真正的起作用。Looper的loop方法会调用MessageQueue的next方法来获取新消息，而next是一个阻塞操作，当没有消息时，next方法会一直阻塞着在那里，这也导致了loop方法一直阻塞在那里。如果MessageQueue的next方法返回了新消息，Looper就会处理这条消息:msg.target.dispatchMessage(msg),其中的msg.target就是发送这条消息的Handler对象。


### Handler的工作原理
Handler就是处理消息的发送和接收之后的处理；消息的发送可以通过post的一系列方法以及send的一系列方法来实现，post方法最终是通过send方法来实现的。
```java
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
                this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}

private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```
可以发现，Handler发送消息的过程仅仅是向消息队列中插入了一条消息，MessageQueue的next方法就会返回这条消息给Looper，Looper收到消息后就开始处理了，最终消息由Looper交给Handler处理，即Handler的dispatchMessage方法会被调用。
```java
public void dispatchMessage(Message msg) {
    if (msg.callback != null) {
        //当message是runnable的情况，也就是Handler的post方法传递的参数，这种情况下直接执行runnable的run方法
        handleCallback(msg);
    } else {
        if (mCallback != null) {
            //如果创建Handler的时候是给Handler设置了Callback接口的实现，那么此时调用该实现的handleMessage方法
            if (mCallback.handleMessage(msg)) {
                return;
            }
        }
        //如果是派生Handler的子类，就要重写handleMessage方法，那么此时就是调用子类实现的handleMessage方法
        handleMessage(msg);
    }
}
```

## 主线程的消息循环
Android的主线程就是ActivityThread，主线程的入口方法就是main，其中调用了Looper.prepareMainLooper()来创建主线程的Looper以及MessageQueue，并通过Looper.loop()方法来开启主线程的消息循环。

主线程内有一个Handler，即ActivityThread.H，它定义了一组消息类型，主要包含了四大组件的启动和停止等过程，例如LAUNCH_ACTIVITY等。 ActivityThread通过ApplicationThread和AMS进行进程间通信，AMS以进程间通信的方法完成ActivityThread的请求后会回调ApplicationThread中的Binder方法，然后ApplicationThread会向H发送消息，H收到消息后会将ApplicationThread中的逻辑切换到ActivityThread中去执行，即切换到主线程中去执行，这个过程就是主线程的消息循环模型。

### Looper.loop 为什么不会阻塞住主线程？
主线程被阻塞的定义是什么？其实就是主线程的 MessageQueue 中，某个消息的处理时间过长，导致后面的消息不能及时处理。

那 Looper.loop 会不会阻塞主线程？看在哪里调用，如果在 Activity.onCreate 中或者其他任何在主线程中执行的代码中调用，当然会，因为主线程进入死循环了。但如果在 ActivityThread 的 main 函数中（这也是这个问题被提出的场景），当然不会。

正是这个无限循环不断地从 MessageQueue 中取出消息，并进行处理，安卓系统的事件循环模型才得以运行。它会导致某个消息的处理时间过长，后面的消息不能及时处理吗？当然不会，没有它都谈不上消息处理！

## Handler所导致的内存泄漏问题
如果使用了非静态的内部 Handler 子类、匿名 Handler 子类，或者把非静态的内部 Runnable 子类、匿名 Runnable 子类 post 到任意 Handler 上，就很可能发生内存泄漏，而如果这些类都在 Activity 内部，那就泄露了 Activity。
