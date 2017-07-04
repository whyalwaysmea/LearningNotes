## Scopes  
In Dagger, an unscoped component cannot depend on a scoped component. As * {@link edu.com.app.injection.component.ApplicationComponent} is a scoped component ({@code @Singleton}, we create a custom  * scope to be used by all fragment components.   
Additionally, a component with a specific scope * cannot have a sub component with the same scope.    

也就是说：  
1. 一个没有scope的组件component不可以依赖一个有scope的组件component。  
2. 子组件和父组件的scope不能相同。  

[Dagger2 Scope 注解能保证依赖在 component 生命周期内的单例性吗？](https://blog.piasy.com/2016/04/11/Dagger2-Scope-Instance/)



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
