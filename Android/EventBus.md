## 简单的使用
EventBus是greenrobot在Android平台发布的一款以订阅——发布模式为核心的开源库。EventBus翻译过来是事件总线的意思，可以这样理解：一个个事件(event)发送到总线上，然后EventBus根据已注册的订阅者(subscribers)来匹配相应的事件，进而把事件传递给订阅者，这也是[观察者模式](http://www.jianshu.com/p/dc22f292476e)的一个最佳实践。   

使用流程：
**Step 1.创建事件实体类：**  
所谓的事件实体类，就是传递的事件，一个组件向另一个组件发送的信息可以储存在一个类中，该类就是一个事件，会被EventBus发送给订阅者。新建MessageEvent.java:
```java
public class MessageEvent {

    private String message;

    public MessageEvent(String message){
        this.message = message;
    }

    public String getMessage(){
        return message;
    }
}
```


**Step 2.向EventBus注册，成为订阅者以及解除注册：**
```java
// 将当前类注册，成为订阅者，即对应观察者模式的“观察者”
EventBus.getDefault().register(this);
// 当订阅者不再需要接受事件的时候，我们需要解除注册，释放内存：
EventBus.getDefault().unregister(this);
```

**Step 3.声明订阅方法：**
声明一个订阅方法需要用到@Subscribe注解，因此在订阅者类中添加一个有着@Subscribe注解的方法即可，方法名字可自定义，而且必须是public权限，其方法参数有且只能有一个，另外类型必须为第一步定义好的事件类型(比如上面的MessageEvent)
```java
@Subscribe
public void onEvent(AnyEventType event) {
    /* Do something */
}
```

**Step 4.发送事件：**
```java
EventBus.getDefault().post(EventType eventType);
```

### 线程模式
`@Subscribe(threadMode = ThreadMode.MAIN)`
可以通过以上这种方式来指定订阅者处理事件的线程，EventBus总共有四种线程模式，分别是：  

* ThreadMode.MAIN：表示无论事件是在哪个线程发布出来的，该事件订阅方法onEvent都会在UI线程中执行，这个在Android中是非常有用的，因为在Android中只能在UI线程中更新UI，所有在此模式下的方法是不能执行耗时操作的。
* ThreadMode.POSTING：表示事件在哪个线程中发布出来的，事件订阅函数onEvent就会在这个线程中运行，也就是说发布事件和接收事件在同一个线程。使用这个方法时，在onEvent方法中不能执行耗时操作，如果执行耗时操作容易导致事件分发延迟。
* ThreadMode.BACKGROUND：表示如果事件在UI线程中发布出来的，那么订阅函数onEvent就会在子线程中运行，如果事件本来就是在子线程中发布出来的，那么订阅函数直接在该子线程中执行。
* ThreadMode.ASYNC：使用这个模式的订阅函数，那么无论事件在哪个线程发布，都会创建新的子线程来执行订阅函数。

ASYNC相比前三者不同的地方是可以处理耗时的操作，其采用了线程池，且是一个异步执行的过程，即事件的订阅者可以立即得到执行。

## 3.0源码解析
### 构造方法
一般我们是通过`EventBus.getDefault()`获取到EventBus的，所以我们先看看是如何获取的吧
```java
public static EventBus getDefault() {
    if (defaultInstance == null) {
        synchronized (EventBus.class) {
            if (defaultInstance == null) {
                defaultInstance = new EventBus();
            }
        }
    }
    return defaultInstance;
}
```
就是一个普通的单例，调用的就是默认的构造器。所以我们继续看默认的构造器
```java
public EventBus() {
    this(DEFAULT_BUILDER);
}

EventBus(EventBusBuilder builder) {
    // 以事件类的class对象为键值，记录注册方法信息，值为一个Subscription的列表
    subscriptionsByEventType = new HashMap<>();
    // 以注册的类为键值，记录该类所注册的所有事件类型，值为一个Event的class对象的列表
    typesBySubscriber = new HashMap<>();
    // 记录sticky事件
    stickyEvents = new ConcurrentHashMap<>();
    // 三个Poster, 负责在不同的线程中调用订阅者的方法
    mainThreadPoster = new HandlerPoster(this, Looper.getMainLooper(), 10);
    backgroundPoster = new BackgroundPoster(this);
    asyncPoster = new AsyncPoster(this);

    //方法的查找类，用于查找某个类中有哪些注册的方法
    subscriberMethodFinder = new SubscriberMethodFinder(builder.subscriberInfoIndexes,
        builder.strictMethodVerification, builder.ignoreGeneratedIndex);
    ...
}
```
可以看出来，这里是一个构造者模式的运用。  
subscriptionsByEventType，以Event的class对象为键值，管理注册信息，值为一个处理事件类型为该键值的Subscription的列表。   
typesBySubscriber则相对简单，就是记录一个类注册了哪些事件类型。  
stickyEvents，是一个线程安全的Map,用来记录sticky事件，sticky事件的含义是指即使被观察者发送sticky事件是在订阅者订阅该事件之前，订阅者在订阅之后，EventBus将该事件发送到该订阅者，即调用相应的订阅方法。（如果在之后，那就和普通事件一样）


### 注册
```java
public void register(Object subscriber) {
    Class<?> subscriberClass = subscriber.getClass();
    List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
    synchronized (this) {
        for (SubscriberMethod subscriberMethod : subscriberMethods) {
            subscribe(subscriber, subscriberMethod);
        }
    }
}
```
这里做了两件事：
1. 找出subscriber的所有订阅方法。
2. 调用订阅方法,subscribe(subscriber, subscriberMethod);调用这里有一个subscriber和SubscriberMethod

在看订阅方法之前，我们先了解两个类:
```java
// 这是订阅者和订阅方法类的一个契约关系类。所以类和方法唯一确定一条注册信息
// active表示该注册信息是否有效
final class Subscription {
    final Object subscriber;
    final SubscriberMethod subscriberMethod;
    /**
     * Becomes false as soon as {@link EventBus#unregister(Object)} is called, which is checked by queued event delivery
     * {@link EventBus#invokeSubscriber(PendingPost)} to prevent race conditions.
     */
    volatile boolean active;

    Subscription(Object subscriber, SubscriberMethod subscriberMethod) {
        this.subscriber = subscriber;
        this.subscriberMethod = subscriberMethod;
        active = true;
    }

    ...
}
```
我们这里再来看订阅方法
```java
// Must be called in synchronized block
private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
    Class<?> eventType = subscriberMethod.eventType;
    // 构建Subscription,Subscription
    Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
    // 找到eventType对应的订阅列表中，如果没有则新建一个
    CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
    if (subscriptions == null) {
        subscriptions = new CopyOnWriteArrayList<>();
        subscriptionsByEventType.put(eventType, subscriptions);
    } else {
        if (subscriptions.contains(newSubscription)) {
            throw new EventBusException("Subscriber " + subscriber.getClass() + " already registered to event "
                    + eventType);
        }
    }

    // 将新建的Subscription对象添加到对应的列表中，添加顺序由priority决定
    int size = subscriptions.size();
    for (int i = 0; i <= size; i++) {
       if (i == size || subscriberMethod.priority > subscriptions.get(i).subscriberMethod.priority) {
           subscriptions.add(i, newSubscription);			                                        // 3
           break;
       }
    }
    // 保存subscriber订阅的所有事件类型，方便取消订阅
    List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
    if (subscribedEvents == null) {
       subscribedEvents = new ArrayList<>();
       typesBySubscriber.put(subscriber, subscribedEvents);                                        
    }
    subscribedEvents.add(eventType);

    // 关于sticky
    if (subscriberMethod.sticky) {
        if (eventInheritance) {
            // Existing sticky events of all subclasses of eventType have to be considered.
            // Note: Iterating over all events may be inefficient with lots of sticky events,
            // thus data structure should be changed to allow a more efficient lookup
            // (e.g. an additional map storing sub classes of super classes: Class -> List<Class>).
            Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
            for (Map.Entry<Class<?>, Object> entry : entries) {
                Class<?> candidateEventType = entry.getKey();
                if (eventType.isAssignableFrom(candidateEventType)) {
                    Object stickyEvent = entry.getValue();
                    checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                }
            }
        } else {
            Object stickyEvent = stickyEvents.get(eventType);
            checkPostStickyEventToSubscription(newSubscription, stickyEvent);
        }
    }
}
```
其实也就做了三件事：  
1. 构建Subscription
2. 将该注册信息保存到两个数据结构中。对于subscriptionsByEventType首先获取EventType对应的列表，没有则创建，重复注册则抛异常，正常情况下，则根据priority插入到列表中适合的位置。对于typesBySubscriber，则是更新该subscriber对应的列表即可。
3. 处理sticky事件。在注册时，如果是监听sticky事件，则需要从stickyEvents中取出对应sticky事件，并发送到订阅者。


### 取消注册
```java
/** Unregisters the given subscriber from all event classes. */
public synchronized void unregister(Object subscriber) {
    List<Class<?>> subscribedTypes = typesBySubscriber.get(subscriber);
    if (subscribedTypes != null) {
        for (Class<?> eventType : subscribedTypes) {
            unsubscribeByEventType(subscriber, eventType);
        }
        typesBySubscriber.remove(subscriber);
    } else {
        Log.w(TAG, "Subscriber to unregister was not registered before: " + subscriber.getClass());
    }
}
```
调用unsubscribeByEventType方法,并更新typesBySubscriber数据结构;    
```java
/** Only updates subscriptionsByEventType, not typesBySubscriber! Caller must update typesBySubscriber. */
private void unsubscribeByEventType(Object subscriber, Class<?> eventType) {
    List<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
    if (subscriptions != null) {
        int size = subscriptions.size();
        for (int i = 0; i < size; i++) {
            Subscription subscription = subscriptions.get(i);
            if (subscription.subscriber == subscriber) {
                subscription.active = false;
                subscriptions.remove(i);
                i--;
                size--;
            }
        }
    }
}
```
这里就是更新另一个数据结构subscriptionsByEventType

### 事件分发
我们先提前了解一个类:
```java
/** For ThreadLocal, much faster to set (and get multiple values). */
final static class PostingThreadState {
    // 消息队列
   final List<Object> eventQueue = new ArrayList<Object>();
   boolean isPosting;
   boolean isMainThread;
   Subscription subscription;
   Object event;
   boolean canceled;
}
```  
我们现在只需要注意第一个，事件消息队列即可。这个类的实例对象在EventBus中是一个ThreadLocal变量，即线程本地变量，不同线程之间不会相互影响，而eventQueue则是用来保存当前线程需要发送的事件
```java
private final ThreadLocal<PostingThreadState> currentPostingThreadState = new ThreadLocal<PostingThreadState>() {
    @Override
    protected PostingThreadState initialValue() {
        return new PostingThreadState();
    }
};

/** Posts the given event to the event bus. */
// 将事件发送到EventBus,还没有发送给订阅者
public void post(Object event) {
    PostingThreadState postingState = currentPostingThreadState.get();
    List<Object> eventQueue = postingState.eventQueue;
    // 将要发送的事件添加到事件队列中
    eventQueue.add(event);

    if (!postingState.isPosting) {
        postingState.isMainThread = Looper.getMainLooper() == Looper.myLooper();
        postingState.isPosting = true;
        if (postingState.canceled) {
            throw new EventBusException("Internal error. Abort state was not reset");
        }
        try {
            while (!eventQueue.isEmpty()) {
                // 这里进去了另外一个方法
                postSingleEvent(eventQueue.remove(0), postingState);
            }
        } finally {
            postingState.isPosting = false;
            postingState.isMainThread = false;
        }
    }
}


private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
    Class<?> eventClass = event.getClass();
    boolean subscriptionFound = false;
    // 是否考虑Event事件类型的继承关系
    // 订阅了它的父类或者父类接口的subscriber也会收到消息，默认是true
    if (eventInheritance) {
        // 如果支持，则会找出当前事件的所有父类以及父类接口
        List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);
        int countTypes = eventTypes.size();
        for (int h = 0; h < countTypes; h++) {
            Class<?> clazz = eventTypes.get(h);
            subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);
        }
    } else {
        subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);
    }
    if (!subscriptionFound) {
        if (logNoSubscriberMessages) {
            Log.d(TAG, "No subscribers registered for event " + eventClass);
        }
        if (sendNoSubscriberEvent && eventClass != NoSubscriberEvent.class &&
                eventClass != SubscriberExceptionEvent.class) {
            post(new NoSubscriberEvent(this, event));
        }
    }
}
```


## 相关链接
