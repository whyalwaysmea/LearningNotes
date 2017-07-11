## 说明
本次示例是基于：   
>compile 'com.google.dagger:dagger:2.11'
annotationProcessor 'com.google.dagger:dagger-compiler:2.11'   


## 注入代码  
这里我们先从最基础的注入方式开始：  
Inject:  
```java
public class Engine {
    @Inject
    public Engine() {}
}

public class Car {
  @Inject
  public Car(Engine engine) {}
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
public final class DaggerManComponent implements ManComponent {
  private Provider<Car> carProvider;

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

    this.carProvider = Car_Factory.create(Engine_Factory.create());

    this.manActivityMembersInjector = ManActivity_MembersInjector.create(carProvider);
  }

  @Override
  public void inject(ManActivity mainActivity) {
    manActivityMembersInjector.injectMembers(mainActivity);
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
  this.carProvider = Car_Factory.create(Engine_Factory.create());

  this.manActivityMembersInjector = ManActivity_MembersInjector.create(carProvider);
}
```   
这里又涉及到了，其他三个由apt生成的java类，分别是`ManActivity_MembersInjector`和`Car_Factory`， `Engine_Factory`       
```java
public final class Engine_Factory implements Factory<Engine> {
  private static final Engine_Factory INSTANCE = new Engine_Factory();

  @Override
  public Engine get() {
    return new Engine();
  }

  public static Factory<Engine> create() {
    return INSTANCE;
  }
}

public final class Car_Factory implements Factory<Car> {
  private final Provider<Engine> engineProvider;

  public Car_Factory(Provider<Engine> engineProvider) {
    assert engineProvider != null;
    this.engineProvider = engineProvider;
  }

  @Override
  public Car get() {
    return new Car(engineProvider.get());
  }

  public static Factory<Car> create(Provider<Engine> engineProvider) {
    return new Car_Factory(engineProvider);
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

-----
## 前言  
之前介绍了最简单的注入生成代码，使用的注入方式是使用`@Inject`， 今天我们解析另外一种注入方式。   


## Module  
使用@Inject标注构造函数来提供依赖的对象实例的方法，不是万能的，在以下几种场景中无法使用：  
1. 第三方库的类不能被标注  
2. 构造函数中的参数必须配置   
3. 抽象的类  

所以这里我们先简单的使用Module：
```java
@Module
public class CarModule {
    @Provides
    Car carProvide(Engine engine) {
        return new Car(engine);
    }
}
```
当然Component也需要跟着做一些调整：  
```java
@Component(modules = CarModule.class)
public interface ManComponent {
    void inject(ManActivity mainActivity);
}
```  
接着我们来分析生成的代码，这里最大的不同就是XXModule中有几个@Provides，就会生成几个对应的类：XXModule_ProvideYYFactory   
因为这里我们只有一个@Provides，所以就生成了一个CarModule_ProvideCarFactory：   
```java
public final class CarModule_CarProvideFactory implements Factory<Car> {
  private final CarModule module;

  private final Provider<Engine> engineProvider;

  public CarModule_CarProvideFactory(CarModule module, Provider<Engine> engineProvider) {
    assert module != null;
    this.module = module;
    assert engineProvider != null;
    this.engineProvider = engineProvider;
  }

  @Override
  public Car get() {
    return Preconditions.checkNotNull(
        module.carProvide(engineProvider.get()),
        "Cannot return null from a non-@Nullable @Provides method");
  }

  public static Factory<Car> create(CarModule module, Provider<Engine> engineProvider) {
    return new CarModule_CarProvideFactory(module, engineProvider);
  }

  /** Proxies {@link CarModule#carProvide(Engine)}. */
  public static Car proxyCarProvide(CarModule instance, Engine engine) {
    return instance.carProvide(engine);
  }
}
```  
同样这个类也是实现了`Factory<T>`接口，不同于上一节的Car_Factory，这个类中多了一个构造方法和一个`proxyProvideCar`方法，同时`get`方法的实现也有了一点不一样。    
原来的`get()`是直接调用了构造函数来返回实例的，而现在是通过module.carProvide来获取的实例， 这正是@Proxies起的作用。   
`proxyCarProvide`所起的作用和`get`的作用一样。    

## 小结
在使用@Module的时候，里面会有对应的@Providers来提供需要生成的对象。   
每一个@Provides，都会生成对应的XXModule_ProvideYYFactory， 该类就类似于最基础的Factory<T>，用于提供实例。  

-----

## @Scope   
>When a binding uses a scope annotation, that means that the component object holds a reference to the bound object until the component object itself is garbage-collected.  

当 Component 与 Module、目标类（需要被注入依赖）使用 Scope 注解绑定时，意味着 Component 对象持有绑定的依赖实例的一个引用直到 Component 对象本身被回收。  
也就是作用域的原理，其实是让生成的依赖实例的生命周期与 Component 绑定，Scope 注解并不能保证生命周期，要想保证赖实例的生命周期，需要确保 Component 的生命周期。   

Scope 是用来确定注入的实例的生命周期的，如果没有使用 Scope 注解，Component 每次调用 Module 中的 provide 方法或 Inject 构造函数生成的工厂时都会创建一个新的实例，而使用 Scope 后可以复用之前的依赖实例。      

Scope 注解只能标注目标类、@provide 方法和 Component。  
Scope 注解要生效的话，需要同时标注在 Component 和提供依赖实例的 Module 或目标类上。  
Module 中 provide 方法中的 Scope 注解必须和 与之绑定的 Component 的 Scope 注解一样，否则作用域不同会导致编译时会报错。    

## @Singleton   
@Singleton顾名思义保证单例，那么它又是如何实现的呢，实现了单例模式那样只返回一个实例吗？   
我们先看看怎么使用：
```java
@Singleton
public class Engine {
    @Inject
    public Engine() {}
}

public class Car {
    public Engine mEngine;
    public Car(Engine engine) {
        this.mEngine = engine;
    }

    public Engine getEngine() {
        return mEngine;
    }
}
```
```java
@Module
public class CarModule {
    @Provides
    @Singleton
    Car carProvide(Engine engine) {
        return new Car(engine);
    }
}
```
```java
@Component(modules = CarModule.class)
@Singleton
public interface ManComponent {
    void inject(ManActivity mainActivity);
}
```
```java
public class ManActivity extends AppCompatActivity {

    @Inject
    Car mCar;

    @Inject
    Car mCar2;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        DaggerManComponent.create().inject(this);
        Log.d("Car", mCar.toString());
        Log.d("Car", mCar2.toString());
    }
}
```  
最后我们观察日志：
>07-11 11:20:47.788 31539-31539/com.whyalwaysmea.dagger2 D/Car: com.whyalwaysmea.dagger2.bean.Car@eda5747
07-11 11:20:47.789 31539-31539/com.whyalwaysmea.dagger2 D/Car: com.whyalwaysmea.dagger2.bean.Car@eda5747   

可以发现这两个变量其实是同一个。     
对比生成的代码，最大的不一样是`DaggerManComponent`，其他几个类都几乎没有什么变化。   
```java
public final class DaggerManComponent implements ManComponent {
  ...

  @SuppressWarnings("unchecked")
  private void initialize(final Builder builder) {
    //
    this.engineProvider = DoubleCheck.provider(Engine_Factory.create());

    this.carProvideProvider =
        DoubleCheck.provider(CarModule_CarProvideFactory.create(builder.carModule, engineProvider));

    this.manActivityMembersInjector = ManActivity_MembersInjector.create(carProvideProvider);
  }

  ...
}
```   
可以看到这里最大的不同就是多了一个`DoubleCheck.provider`
```java
public final class DoubleCheck<T> implements Provider<T>, Lazy<T> {
  private static final Object UNINITIALIZED = new Object();
  private volatile Provider<T> provider;
  private volatile Object instance = UNINITIALIZED; // instance 就是依赖实例的引用
  ...
  @SuppressWarnings("unchecked") // cast only happens when result comes from the provider
  @Override
  public T get() {
    Object result = instance;
    if (result == UNINITIALIZED) {  // 只生成一次实例，之后调用的话直接复用
      synchronized (this) {
        result = instance;
        if (result == UNINITIALIZED) {
          result = provider.get();  // 生成实例
          /* Get the current instance and test to see if the call to provider.get() has resulted
           * in a recursive call.  If it returns the same instance, we'll allow it, but if the
           * instances differ, throw. */
          Object currentInstance = instance;
          if (currentInstance != UNINITIALIZED && currentInstance != result) {
            throw new IllegalStateException("Scoped provider was invoked recursively returning "
                + "different results: " + currentInstance + " & " + result + ". This is likely "
                + "due to a circular dependency.");
          }
          instance = result;
          /* Null out the reference to the provider. We are never going to need it again, so we
           * can make it eligible for GC. */
          provider = null;
        }
      }
    }
    return (T) result;
  }
  ...
}
```  
可以看到`DoubleCheck<T>`实现了`Factory<T>`和`Lazy<T>`，主要的逻辑还是在`get()`方法中。  
内部方法中有点像[Double CheckLock实现单例](https://github.com/whyalwaysmea/LearningNotes/blob/master/Design%20pattern/%E5%8D%95%E4%BE%8B%E6%A8%A1%E5%BC%8F.md#double-checklock实现单例)的方法，只是每次都会有一个新的局部变量，而真正的单例方法是使用的static。    
