## Q&A
为什么@scope可以保证作用域内单例？生成的代码是什么样的？    

## 生成代码分析

Inject:  
```java
public class Car {
    @Inject
    public Car() {}
}
```

Component:  
```java
@Component
public interface ManComponent {
    void injectMan(ManActivity manActivity);  // 注入 man 所需要的依赖
}
```

```java
public class ManActivity extends AppCompatActivity {

    @Inject
    Car mCar;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_man);
        // DaggerManComponent.builder().build().injectMan(this);
        DaggerManComponent.create().injectMan(this);
        System.out.println(mCar.toString());
    }
}
```
我们这里先从`DaggerManComponent`开始看， `DaggerManComponent`是直接apt生成的代码：  
```java
@Generated(
  value = "dagger.internal.codegen.ComponentProcessor",
  comments = "https://google.github.io/dagger"
)
public final class DaggerManComponent implements ManComponent {
  private MembersInjector<ManActivity> manActivityMembersInjector;

  private DaggerManComponent(Builder builder) {
    assert builder != null;
    initialize(builder);
  }

  public static Builder builder() {
    return new Builder();
  }

  public static ManComponent create() {
    return new Builder().build();
  }

  @SuppressWarnings("unchecked")
  private void initialize(final Builder builder) {

    this.manActivityMembersInjector = ManActivity_MembersInjector.create(Car_Factory.create());
  }

  @Override
  public void injectMan(ManActivity manActivity) {
    manActivityMembersInjector.injectMembers(manActivity);
  }

  public static final class Builder {
    private Builder() {}

    public ManComponent build() {
      return new DaggerManComponent(this);
    }
  }
}
```  
可以看到，这里是有一个Builder类，我们在注入的时候，调用`DaggerManComponent.create();`和`DaggerManComponent.builder().build();`其实是等价的。   
都是调用了`ManComponent`的构造函数，返回了一个`ManComponent`,同时也调用了`initialize()`,initialize就是得到了一个MembersInjector<ManActivity>对象，`injectMan`其实也是调用的`MembersInjector<ManActivity>.injectMembers()`   
所以我们这里先看看initialize：  
```java
private void initialize(final Builder builder) {
    this.manActivityMembersInjector = ManActivity_MembersInjector.create(Car_Factory.create());
}
```   
这里又涉及到了，其他两个由apt生成的java类，分别是`ManActivity_MembersInjector`和`Car_Factory`   
```java
@Generated(
  value = "dagger.internal.codegen.ComponentProcessor",
  comments = "https://google.github.io/dagger"
)
public final class Car_Factory implements Factory<Car> {
  private static final Car_Factory INSTANCE = new Car_Factory();

  @Override
  public Car get() {
    return new Car();
  }

  public static Factory<Car> create() {
    return INSTANCE;
  }
}
```    
首先可以看到Factory<T>继承了Provider<T>接口，里面有一个get方法。  
其次，这里的`Car_Factory.create()`其实就是直接new了一个Car_Factory对象出来。`get`就是new一个Car对象。   
```java
@Generated(
  value = "dagger.internal.codegen.ComponentProcessor",
  comments = "https://google.github.io/dagger"
)
public final class ManActivity_MembersInjector implements MembersInjector<ManActivity> {
  private final Provider<Car> mCarProvider;

  public ManActivity_MembersInjector(Provider<Car> mCarProvider) {
    assert mCarProvider != null;
    this.mCarProvider = mCarProvider;
  }

  public static MembersInjector<ManActivity> create(Provider<Car> mCarProvider) {
    return new ManActivity_MembersInjector(mCarProvider);
  }

  @Override
  public void injectMembers(ManActivity instance) {
    if (instance == null) {
      throw new NullPointerException("Cannot inject members into a null reference");
    }
    instance.mCar = mCarProvider.get();
  }

  public static void injectMCar(ManActivity instance, Provider<Car> mCarProvider) {
    instance.mCar = mCarProvider.get();
  }
}
```  
首先这里是实现了一个`MembersInjector<T>`接口， 里面有一个`injectMembers()`方法，这个方法主要就是返回实例对象的。  
从这里可以得知，我们注入的时候，是不能用`private`来修饰注入对象的，因为那样的话在生成的代码中是无法获取到对象的。   
