# Dagger2注解
Dagger2是基于Java注解来实现依赖注入的，那么在正式使用之前我们需要先了解下Dagger2中的注解。Dagger2使用过程中我们通常接触到的注解主要包括：
@Inject, @Module, @Provides, @Component, @Qulifier, @Scope, @Singleten。

* @Component: @Component用于标注接口，是依赖需求方和依赖提供方之间的桥梁。被Component标注的接口在编译时会生成该接口的实现类（如果@Component标注的接口为CarComponent，则编译期生成的实现类为DaggerCarComponent）,我们通过调用这个实现类的方法完成注入；


* @Inject: @Inject有两个作用，一是用来标记需要依赖的变量，以此告诉Dagger2为它提供依赖；二是用来标记构造函数，Dagger2通过@Inject注解可以在需要这个类实例的时候来找到这个构造函数并把相关实例构造出来，以此来为被@Inject标记了的变量提供依赖；

* @Module: @Module用于标注提供依赖的类。你可能会有点困惑，上面不是提到用@Inject标记构造函数就可以提供依赖了么，为什么还需要@Module？很多时候我们需要提供依赖的构造函数是第三方库的，我们没法给它加上@Inject注解，又比如说提供以来的构造函数是带参数的，如果我们之所简单的使用@Inject标记它，那么他的参数又怎么来呢？@Module正是帮我们解决这些问题的。


* @Provides：@Provides用于标注Module所标注的类中的方法，该方法在需要提供依赖时被调用，从而把预先提供好的对象当做依赖给标注了@Inject的变量赋值；

* @Qulifier：@Qulifier用于自定义注解

* @Scope：@Scope同样用于自定义注解，我能可以通过@Scope自定义的注解来限定注解作用域，实现局部的单例；

* @Singleton：@Singleton其实就是一个通过@Scope定义的注解，我们一般通过它来实现全局单例。但实际上它并不能提前全局单例，是否能提供全局单例还要取决于对应的Component是否为一个全局对象。

我们提到@Inject和@Module都可以提供依赖，那如果我们即在构造函数上通过标记@Inject提供依赖，有通过@Module提供依赖Dagger2会如何选择呢？具体规则如下：


* 步骤3：若不存在提供依赖的方法，则查找@Inject标注的构造函数，看构造函数是否存在参数。     
 * a：若存在参数，则从步骤1开始依次初始化每一个参数
 * b：若不存在，则直接初始化该类实例，完成一次依赖注入。

* 步骤1：首先查找@Module标注的类中是否存在提供依赖的方法。
* 步骤2：若存在提供依赖的方法，查看该方法是否存在参数。   
    * a:若存在参数，则按从步骤1开始依次初始化每个参数；
    * b:若不存在，则直接初始化该类实例，完成一次依赖注入。

 ## 简单案例1 @Inject
 Shoe是依赖提供方，所以我们需要在它的构造函数上添加@Inject
 ```Java
 public class Shoe {
     @Inject
     public Shoe() {

     }
     @Override
     public String toString() {
         return "鞋子";
     }
 }
 ```
 MainActivity是需求依赖方，依赖了Shoe类；因此我们需要在类变量Shoe上添加@Inject,
 ```Java
 @Inject
 Shoe mShoe;
 ```
 接下来我们需要创建一个用@Component标注的接口MainComponent，这个MainComponent其实就是一个注入器，这里用来将Shoe注入到MainActivity中。
 ```Java
 @Component
 public interface MainComponent {
     void inject(MainActivity mainActivity);
 }
 ```
 这个时候编译一下项目，会在生成app/build/generated/source/apt..中生成DaggerMainComponent.java，然后我们就可以在MainActivity中进行依赖注入了
 ```Java
 public class MainActivity extends AppCompatActivity {

     @Inject
     Shoe mShoe;

     @Override
     protected void onCreate(Bundle savedInstanceState) {
         super.onCreate(savedInstanceState);
         setContentView(R.layout.activity_main);

         TextView tv = (TextView) findViewById(R.id.haha);
         DaggerMainComponent
                 .builder()
                 .build()
                 .inject(this);
         tv.setText(mShoe.toString() + "");
     }
 }
 ```

 ## 简单案例2 @Module
如果创建Shoe的构造函数是带参数的呢？比如说制造一双鞋子是需要布料(cloth)的。或者Shoe是第三方库的类，我们无法直接在他的构造函数中添加@Inject。这时候就需要@Module和@Provides
```Java
public class Cloth {

    private String color;

    public String getColor() {
        return color;
    }

    public void setColor(String color) {
        this.color = color;
    }

    @Override
    public String toString() {
        return color + "布料";
    }
}
```

首先我们创建Module类：
```Java
@Module
public class MainModule {

    public MainModule() {

    }

    @Provides
    public Shoe proiveShoe(Cloth cloth) {
        return new Shoe(cloth);
    }

    @Provides
    private Cloth provideCloth() {
        Cloth cloth = new Cloth();
        cloth.setColor("红色");
        return cloth;
    }
}
```
接下来我们还需要对MainComponent做一些修改:
```Java
@Component(modules = MainModule.class)
public interface MainComponent {
    void inject(MainActivity mainActivity);
}
```
之前的@Component注解是不带参数的，现在我们需要加上`modules = MainModule.class`,用来告诉Dagger2提供依赖的是MainModule这个类。
最后编译一下，然后在MainActivity中进行一些简单的修改就可以直接使用了
```Java
MainActivity:
DaggerMainComponent
                .builder()
                .mainModule(new MainModule())
                .build()
                .inject(this);
```

## 简单案例3 @Named/@Qulifier
如果一双Shoe需要两种颜色不一样的布料怎么办呢？很容易想到就是在module中写两个提供Cloth的方法:
```java
@Provides
public Cloth provideCloth() {
    Cloth cloth = new Cloth();
    cloth.setColor("红色");
    return cloth;
}

@Provides
public Cloth provideAnotherCloth() {
    Cloth cloth = new Cloth();
    cloth.setColor("蓝色");
    return cloth;
}
```
可问题就来了,Dagger2是通过返回值类型来确定的,当你需要红布料时,它又怎么知道哪个是红布料呢?所以Dagger2为我们提供`@Named`注解,它怎么使用呢?它有一个value值,用来标识这个方法是给谁用的.修改我们的代码:
```java
@Module
public class MainModule {

    public MainModule() {

    }

    @Provides
    public Shoe proiveShoe(@Named("blue") Cloth cloth) {
        return new Shoe(cloth);
    }

    @Provides
    @Named("red")
    public Cloth provideCloth() {
        Cloth cloth = new Cloth();
        cloth.setColor("红色");
        return cloth;
    }

    @Provides
    @Named("blue")
    public Cloth provideAnotherCloth() {
        Cloth cloth = new Cloth();
        cloth.setColor("蓝色");
        return cloth;
    }
}
```
我们在getRedCloth方法上使用`@Named("red")`表明此方法返回的是红布料,同理,在getBlueCloth方法上使用`@Named("blue")`表明此方法返回的是蓝布料。在使用的时候同样也是标注`@Named()`就好了。

**@Qulifier**
当然我们也可以使用`@Qulifier`,@Qulifier功能和@Named一样,并且@Named就是继承@Qulifier的,我们要怎么使用@Qulifier注解呢?答案就是自定义一个注解:
```java
@Qualifier
@Retention(RetentionPolicy.RUNTIME)
    public @interface RedCloth {
}
```
使用
```java
@Provides
public Shoe proiveShoe(@RedCloth Cloth cloth) {
    return new Shoe(cloth);
}

@Provides
@RedCloth
public Cloth provideCloth() {
    Cloth cloth = new Cloth();
    cloth.setColor("红色");
    return cloth;
}
```

## 简单案例4 @Singleton/@Scope
我们先在MainActivity中进行如下测试：
```java
public class MainActivity extends AppCompatActivity {

    @Inject
    Shoe mShoe;

    @Inject
    @RedCloth
    Cloth mCloth;


    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        TextView tv = (TextView) findViewById(R.id.haha);
        tv.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                Intent intent = new Intent(MainActivity.this, OldActivity.class);
                startActivity(intent);
            }
        });

        DaggerMainComponent
                .builder()
                .mainModule(new MainModule())
                .build()
                .inject(this);
        tv.setText((mCloth == mShoe.getCloth()) + "");
    }
}
```
最后的打印结果为false。mCloth和mShoe中的cloth不是同一个对象。按照道理来说，我们这个时候应该是希望cloth是一个对象的，所以我们就可以使用@Singleton
首先,对于module的方法添加注解
```java
@Provides
@RedCloth
@Singleton
public Cloth provideCloth() {
    Cloth cloth = new Cloth();
    cloth.setColor("红色");
    return cloth;
}
```
然后，再对MainComponent接口添加注解
```java
@Component(modules = MainModule.class)
@Singleton
public interface MainComponent {

    void inject(MainActivity mainActivity);
}
```
最后再次打印结果为：true。这就可以说明mCloth和mShoe中的cloth是同一个对象了。

**@Scope**
@Singleton是怎么实现的呢?我们先看看@Scope注解,弄懂它,@Singleton你也就会明白了,下面我们就来分析分析
顾名思义,@Scope就是用来声明作用范围的.@Scope和@Qulifier一样,需要我们自定义注解才能使用,我们先自定义一个注解:
```java
@Scope
@Retention(RetentionPolicy.RUNTIME)
public @interface PerActivity {
}
```
这个注解有什么用呢?答案就是声明作用范围,当我们将这个注解使用在Module类中的Provide方法上时,就是声明这个Provide方法是在PerActivity作用范围内的,并且当一个Component要引用这个Module时,必须也要声明这个Component是PerActivity作用范围内的,否则就会报错,声明方法也很简单,就是在Component接口上使用这个注解.

但是我们声明这个作用范围又有什么用呢?原来Dagger2有这样一个机制:在同一个作用范围内,Provide方法提供的依赖对象就会变成单例,也就是说依赖需求方不管依赖几次Provide方法提供的依赖对象,Dagger2都只会调用一次这个方法.

*注意：* 单例是在同一个Component实例提供依赖的前提下才有效的,不同的Component实例只能通过Component依赖才能实现单例.也就是说,你虽然在两个Component接口上都添加了PerActivity注解,但是这两个Component提供依赖时是没有联系的,他们只能在各自的范围内实现单例

## 简单案例5 #dependencies
在实际开发中,我们经常会使用到工具类,工具类一般在整个App的生命周期内都是单例的,我们现在给我们的Demo添加一个工具类ClothHandler:
```java
public class ClothHandler {
    public Shoe handle(Cloth cloth){
        return new Shoe(cloth);
    }
}
```

它的功能就是将cloth加工成clothes,假设我们现在有两个Activity中都要使用该工具类,我们要怎么使用Dagger2帮我们注入呢?
可能有人会想到在module中提供ClothHandler就可以了，现在只有两个Activity使用到了ClothHandler，如果有20个Activity都用到了ClothHandler，难道不成要写20个？况且我们更希望这个工具类是以单例的形式存在
事实上是当然不用的
在面向对象的思想中,我们碰到这种情况一般都要抽取父类,Dagger2也是用的这种思想,我们先创建一个BaseModule,用来提供工具类:

```java
@Module
public class BaseModule {
    @Provides
    @Singleton
    public ClothHandler getClothHandler() {
        return new ClothHandler();
    }
}
```
然后创建一个BaseComponent:
```java
@Component(dependencies = BaseModule.class)
@Singleton
public interface BaseComponent {
    ClothHandler getClothHandler();
}
```
嗯?这个Component怎么有点不一样,怎么没有inject方法呢?
上面讲过,我们通过inject方法依赖需求方实例送到Component中,从而帮助依赖需求方实现依赖,但是我们这个BaseComponent是给其他Component提供依赖的,所以我们就可以不用inject方法,但是BaseComponent中多了一个getClothHandler方法,它的返回值是ClothHandler对象,这个方法有什么用呢?它的作用就是告诉依赖于BaseComponent的Component,BaseComponent能为你们提供ClothHandler对象
再修改一下MainComponent和MainModule:
```java
@Component(modules = MainModule.class, dependencies = BaseComponent.class)
@PerActivity
public interface MainComponent {

    void inject(MainActivity mainActivity);
}
```
```java
@Provides
public Shoe handleShoe(@RedCloth Cloth cloth, ClothHandler clothHandler) {
    return  clothHandler.handle(cloth);
}
```
最后就是使用了:
```java
public class MyApplication extends Application {

    private BaseComponent mBaseComponent;

    @Override
    public void onCreate() {
        super.onCreate();
        mBaseComponent = DaggerBaseComponent.builder()
                .baseModule(new BaseModule())
                .build();

    }

    public BaseComponent getBaseComponent() {
        return mBaseComponent;
    }
}
// MainActivity中：
DaggerMainComponent
                .builder()
                .baseComponent(((MyApplication)getApplication()).getBaseComponent())
                .mainModule(new MainModule())
                .build()
                .inject(this);
```

# 相关链接
[Dagger2 入门,以初学者角度.](http://www.jianshu.com/p/1d84ba23f4d2#)

[Android：dagger2让你爱不释手-终结篇](http://www.jianshu.com/p/65737ac39c44#)

[都是套路——Dagger2没有想象的那么难](http://www.jianshu.com/p/47c7306b2994#)
