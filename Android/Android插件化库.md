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
