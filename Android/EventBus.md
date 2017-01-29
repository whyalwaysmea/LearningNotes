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
