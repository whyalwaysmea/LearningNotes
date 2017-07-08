## 说明
本次示例是基于：   
>compile 'com.google.dagger:dagger:2.11'
annotationProcessor 'com.google.dagger:dagger-compiler:2.11'   


## 注入代码  
这里我们先从最基础的注入方式开始：  
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
        DaggerManComponent.create().inject(this);
        Log.d("Car", mCar.toString());

        DaggerManComponent.builder().build().inject(this);
        Log.d("Car", mCar.toString());
    }
}
```  
这里我们先运行起来，看看打印出来的日志：
>07-08 11:00:32.461 9953-9953/com.whyalwaysmea.dagger2 D/Car: com.whyalwaysmea.dagger2.bean.Car@850a3f9
07-08 11:00:32.461 9953-9953/com.whyalwaysmea.dagger2 D/Car: com.whyalwaysmea.dagger2.bean.Car@27e1df3e

可以发现，Car的地址变了，说明两次注入是生成的不同的对象。  

## 生成代码分析
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

## 小结   
通过分析生成的代码，我们知道了三个接口`Factory<T>`和`MembersInjector<T>`，`Factory<T>`又是继承于`Provider<T>`。   

`Factory<T>`可以理解成是生产实例对象的地方：      
构造方法由`@Inject`修饰过的类XX，会生成一个实现了Factory的类XX_Factory，该类中主要是通过`get()`方法来获取对象。   

`MembersInjector<T>`可以理解成是需要注入对象的成员：  
在示例中，是ManActivity需要Car,所以这里是生成了`ManActivity_MembersInjector implements MembersInjector<ManActivity>`       
该类中主要是通过`injectMembers(ManActivity instance)`方法，将Provider<T>.get()获取到的对象赋值给了ManActivity.T   

最后再来小结一下`DaggerXXXComponent`，该类是直接实现了XXXComponent接口的。  
Component一直被认为是Dagger2中依赖注入的桥梁。这里由于没有涉及到Module，所以Component也没有起到很大的作用。  
所以也就有了第三种注入的方式：
```java
ManActivity_MembersInjector.create(Car_Factory.create()).injectMembers(this);
```    
