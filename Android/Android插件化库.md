### [dynamic-load-apk](https://github.com/singwhatiwanna/dynamic-load-apk)  
#### 源码分析  
[Dynamic-Load-Apk源码解析](http://www.jianshu.com/p/30114b7176a3)    
[Android插件化学习之路（八）之DynamicLoadApk 源码解析（上）](http://blog.csdn.net/u012124438/article/details/53241755)

#### 主要思想  
主要是通过**代理**来完成Activity,Service的相关操作     
![总体设计](https://raw.githubusercontent.com/android-cn/android-open-project-analysis/master/tool-lib/plugin/dynamic-load-apk/image/overall-design.png)



#### 缺点  
* 不支持IntentService，不支持 Provider，静态广播；         
* 插件编写规范上有一定的限制，比如无法直接使用this，需要继承指定的类      
* 不支持LaunchMode  
* 虽然启动了Activity，也通过接口的方式来跟随了生命周期，但是它也只能算是一个普通的类那样。  
* 主题管理，需要给每个Activity单独设置主题    
* 插件和宿主资源 id 可能重复的问题没有解决，需要修改 aapt 中资源 id 的生成规则；  

---

### [DroidPlugin](https://github.com/DroidPluginTeam/DroidPlugin)    
#### 源码分析  
[插件开发之360 DroidPlugin源码分析](https://github.com/DroidPluginTeam/DroidPlugin/tree/master/DOC)    
[Android插件化原理解析](http://weishu.me/2016/01/28/understand-plugin-framework-overview/)  

#### 基本原理  
1. 共享进程：为android提供一个进程运行多个apk的机制，通过API欺骗机制瞒过系统  
2. 占坑：通过预先占坑的方式实现不用在mainfest注册，通过一带多的方式实现服务管理   
3. Hook机制：动态代理实现函数hook，Binder代理绕过部分系统服务限制，IO重定向（先获取原始Object-->Read,然后动态代理Hook Object后-->Write回去，达到瞒天过海的目的）  

#### 特点
1. 支持Androd 2.3以上系统  
2. 插件APK完全不需做任何修改，可以独立安装运行、也可以做插件运行。要以插件模式运行某个APK，你无需重新编译、无需知道其源码。  
3. 插件的四大组件完全不需要在Host程序中注册，支持Service、Activity、BroadcastReceiver、ContentProvider四大组件  
4. 插件之间、Host程序与插件之间会互相认为对方已经"安装"在系统上了。  
5. API低侵入性：极少的API。HOST程序只是需要一行代码即可集成Droid Plugin  
6. 超强隔离：插件之间、插件与Host之间完全的代码级别的隔离：不能互相调用对方的代码。通讯只能使用Android系统级别的通讯方法。
7. 支持所有系统API  
8. 资源完全隔离：插件之间、与Host之间实现了资源完全隔离，不会出现资源窜用的情况。  
9. 实现了进程管理，插件的空进程会被及时回收，占用内存低。  
10. 插件的静态广播会被当作动态处理，如果插件没有运行（即没有插件进程运行），其静态广播也永远不会被触发。

#### 缺点
1. 通知栏限制（无法在插件中发送具有自定义资源的Notification，例如： 1. 带自定义RemoteLayout的Notification 2. 图标通过R.drawable.XXX指定的通知（插件系统会自动将其转化为Bitmap）   
2. 安全性担忧（可以修改，hook一些重要信息）  
3. 机型适配（不是所有机器上都能行，因为大量用反射相关，如果rom厂商深度定制了framework层，反射的方法或者类不在，容易插件运用失败）  
4. 无法在插件中注册一些具有特殊Intent Filter的Service、Activity、BroadcastReceiver、ContentProvider等组件以供Android系统、已经安装的其他APP调用。  
5. 缺乏对Native层的Hook，对某些带native代码的apk支持不好，可能无法运行。比如一部分游戏无法当作插件运行。  

---
### [RePlugin](https://github.com/Qihoo360/RePlugin)  

#### 特点  
1. 极其灵活：主程序无需升级（无需在Manifest中预埋组件），即可支持新增的四大组件，甚至全新的插件   
2. 非常稳定：Hook点仅有一处（ClassLoader），无任何Binder Hook！如此可做到其崩溃率仅为“万分之一”，并完美兼容市面上近乎所有的Android ROM  
3. 特性丰富：支持近乎所有在“单品”开发时的特性。包括静态Receiver、Task-Affinity坑位、自定义Theme、进程坑位、AppCompat、DataBinding等  

#### 缺点  


----

### [VirtualAPK](https://github.com/didi/VirtualAPK)
#### 使用接入
[VirtualAPK插件框架介绍（一）----框架接入](http://www.jianshu.com/p/013510c19391)

#### 源码分析
[滴滴插件化方案 VirtualApk 源码解析](http://blog.csdn.net/lmj623565791/article/details/75000580)  
[VirtualAPK 资源篇](https://www.notion.so/VirtualAPK-1fce1a910c424937acde9528d2acd537)  

#### 特性
[VirtualAPK的特性](https://github.com/didi/VirtualAPK/wiki#virtualapk%E7%9A%84%E7%89%B9%E6%80%A7)  
