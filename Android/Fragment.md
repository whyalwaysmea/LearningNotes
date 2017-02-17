## 生命周期
![Fragment生命周期](https://imgs.babits.top/201701084431fragment_lifecycle.png)

### 单个 Activity + Fragment 的启动？
```java
Activity onCreate
Fragment onAttach
Fragment onCreate
Fragment onCreateView
Fragment onViewCreated

Activity onStart
// 下面两个由 super.onStart 触发
Fragment onActivityCreated
Fragment onStart

Activity onResume

Activity onPostResume
// 下面一个由 super.onPostResume 触发
Activity onResumeFragments
// 下面一个由 super.onResumeFragments 触发
Fragment onResume
```



## replace和add方法
Fragment本身并没有replace和add方法， 这里的理解应该为使用FragmentManager 的replace和 add 两种方法切换 Fragment 时有什么不同。
* add Fragment 并不会导致原本就在 id 指定的 ViewGroup 上的 Fragment onPause；
* replace Fragment 会导致原本就在 id 指定的 ViewGroup 上的 Fragment onPause, onDestroy, onDetach，且这一切都发生在新 Fragment onResume 之后；


show()，hide()最终是让Fragment的View setVisibility(true还是false)，不会调用生命周期；
replace()的话会销毁视图，即调用onDestoryView、onCreateView等一系列生命周期；
使用show()，hide()带来的一个问题就是，如果你不做任何额外处理，在“内存重启”后，Fragment会重叠；（该BUG在support-v4 24.0.0+以上 官方已修复）


## Activity与Fragment通信
当 Fragment 跟 Activity 绑定之后，在 Fragment 中可以直接通过 getActivity（）方法获取到其绑定的Activity 对象， 这样就可以调用 Activity 的方法了。   

如果你Activity中包含自己管理的Fragment的引用，可以通过引用直接访问所有的Fragment的public方法；如果Activity中未保存任何Fragment的引用，那么没关系，每个Fragment都有一个唯一的TAG或者ID,可以通过getFragmentManager.findFragmentByTag()或者findFragmentById()获得任何Fragment实例，然后进行操作。

广播

EventBus

接口

## [Fragment的懒加载](http://www.jianshu.com/p/843088e1ad40)

## Fragment的坑
getActivity()空指针:   
可能你遇到过getActivity()返回null，或者平时运行完好的代码，在“内存重启”之后，调用getActivity()的地方却返回null，报了空指针异常。   
在Fragment基类里设置一个Activity mActivity的全局变量，在onAttach(Activity activity)里赋值，使用mActivity代替getActivity()，保证Fragment即使在onDetach后，仍持有Activity的引用（有引起内存泄露的风险，但是异步任务没停止的情况下，本身就可能已内存泄漏，相比Crash，这种做法“安全”些）
