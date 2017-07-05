## Scopes  
In Dagger, an unscoped component cannot depend on a scoped component. As * {@link edu.com.app.injection.component.ApplicationComponent} is a scoped component ({@code @Singleton}, we create a custom  * scope to be used by all fragment components.   
Additionally, a component with a specific scope * cannot have a sub component with the same scope.    

也就是说：  
1. 一个没有scope的组件component不可以依赖一个有scope的组件component。  
2. 子组件和父组件的scope不能相同。  

[Dagger2 Scope 注解能保证依赖在 component 生命周期内的单例性吗？](https://blog.piasy.com/2016/04/11/Dagger2-Scope-Instance/)

### 可重用的scope-[@Reusable](https://google.github.io/dagger/api/latest/dagger/Reusable.html)  
什么时候使用@Reusable: 有时候你想限制@Inject构造次数或者@Provides的调用次数，但是你又不需要保证单例的情况下。     

当你使用@Reusable注解，这些@Reusable域绑定不像其他的域，不会和任何component联系，相反，每个使用这个绑定component会将返回值或初始化的对象缓存起来。  

这意味着如果你在component中装载了@Reusable绑定的module,但只有一个子component使用了，那么那个子component将会缓存此绑定的对象。如果两个子component都使用了这个绑定但他们不继承同一个component，那么这两个子component的缓存是独立的。如果component已经缓存了对象，其子component会重用该对象。   

并不能保证component只会调用该绑定一次，所以在返回可变对象或者需要使用单例的绑定上使用@Reusable是很危险的。对不关心被分配多少次的不变对象使用@Reusable是安全的。

## 懒加载注入
有时你需要延迟初始化对象。对于任意绑定T，你可以创建Lazy<T>，这样就可以延迟对象初始化直到调用Lazy<T>的get()方法。

## Provider注入


## 组件
### 组件依赖  
两个依赖的组件不能共享作用域，比如两个组件不能共享@Singleton作用域。这个限制产生的[原因看这里](https://github.com/google/dagger/issues/107#issuecomment-71073298)。依赖的组件需要定义自己的作用域。     

尽管Dagger2 有创建作用域实例的能力，你也需要创建和删除引用来满足行为的一致性。Dagger2 不会知道任何底层的实现。可以看看Stack Overflow 的这个[讨论](https://stackoverflow.com/questions/28411352/what-determines-the-lifecycle-of-a-component-object-graph-in-dagger-2)  

当创建依赖组件的时候，父组件需要显示的暴露对象给子组件。比如子组件需要知道Retrofit 对象，也就需要显示的暴露出来。
```java
@Singleton
@Component(modules={AppModule.class, NetModule.class})
public interface NetComponent {
    // downstream components need these exposed with the return type
    // method name does not really matter
    Retrofit retrofit();
}
```
### 子组件  



## 相关链接
[Google官方MVP+Dagger2架构详解【从零开始搭建android框架系列（6）】](http://www.jianshu.com/p/01d3c014b0b1)  
