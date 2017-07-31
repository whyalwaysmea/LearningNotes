## 简介
当项目越来越庞大的时候，需要通过插件化来减轻应用的内存和CPU占用，还可以实现热插播，即在不发布新版本的情况下更新某些模块。  
不同的插件化方案各有特色，但是它们都必须要解决三个基础性问题：   
* 资源访问
* Activity生命周期的管理
* ClassLoader的管理  

在此之前我们要明白宿主和插件。宿主是指普通的apk，而插件一般是指经过处理的dex或者apk。   


## 基础知识  
### [Java中的ClassLoader](https://www.ibm.com/developerworks/cn/java/j-lo-classloader/)  
#### ClassLoader 中与加载类相关的方法:   

 方法 | 说明 |
 ----|------|
 getParent() | 返回该类加载器的父类加载器
loadClass(String name) | 加载名称为 name的类，返回的结果是 java.lang.Class类的实例。
findClass(String name) | 查找名称为 name的类，返回的结果是 java.lang.Class类的实例。  
findLoadedClass(String name) | 查找名称为 name的已经被加载过的类，返回的结果是 java.lang.Class类的实例。  
defineClass(String name, byte[] b, int off, int len) | 把字节数组 b中的内容转换成 Java 类，返回的结果是 java.lang.Class类的实例。这个方法被声明为 final的。
resolveClass(Class<?> c) | 链接指定的 Java 类。   

#### 类加载器的树状组织结构  
Java 中的类加载器大致可以分成两类，一类是系统提供的，另外一类则是由 Java 应用开发人员编写的。系统提供的类加载器主要有下面三个：   
- 引导类加载器（bootstrap class loader）：它用来加载 Java 的核心库，是用原生代码来实现的，并不继承自 java.lang.ClassLoader。  
- 扩展类加载器（extensions class loader）：它用来加载 Java 的扩展库。Java 虚拟机的实现会提供一个扩展库目录。该类加载器在此目录里面查找并加载 Java 类。  
- 系统类加载器（system class loader）：也称为应用类加载器，它的父加载器为扩展类加载器。它从环境变量classpath或者系统属性java.class.path所指定的目录中加载类，它是用户自定义的类加载器的默认父加载器。系统类加载器是纯Java类，是java.lang.ClassLoader类的子类。  

父子加载器并非继承关系，也就是说子加载器不一定是继承了父加载器。

![类加载器树状组织结构示意图](https://www.ibm.com/developerworks/cn/java/j-lo-classloader/image001.jpg)

#### 双亲委托模式
通俗的讲，就是某个特定的类加载器在接到加载类的请求时，首先将加载任务委托给父类加载器，依次递归，如果父类加载器可以完成类加载任务，就成功返回；只有父类加载器无法完成此加载任务时，才自己去加载。   
1. 因为这样可以避免重复加载，当父亲已经加载了该类的时候，就没有必要子ClassLoader再加载一次。   
2. 考虑到安全因素，我们试想一下，如果不使用这种委托模式，那我们就可以随时使用自定义的String来动态替代java核心api中定义类型，这样会存在非常大的安全隐患，而双亲委托的方式，就可以避免这种情况，因为String已经在启动时被加载，所以用户自定义类是无法加载一个自定义的ClassLoader。  


### [Android中的ClassLoader](http://blog.csdn.net/u012124438/article/details/53235848)   
JVM中ClassLoader通过defineClass方法加载jar里面的Class，而Android中这个方法被弃用了。  
```java
@Deprecated
protected final Class<?> defineClass(byte[] b, int off, int len)
    throws ClassFormatError {
    throw new UnsupportedOperationException("can't load this type of class file");
}
```  
取而代之的是loadClass方法
```java
protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException {
        // First, check if the class has already been loaded
        Class c = findLoadedClass(name);
        if (c == null) {
            long t0 = System.nanoTime();
            try {
                if (parent != null) {
                    c = parent.loadClass(name, false);
                } else {
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // ClassNotFoundException thrown if class not found
                // from the non-null parent class loader
            }

            if (c == null) {
                // If still not found, then invoke findClass in order
                // to find the class.
                long t1 = System.nanoTime();
                c = findClass(name);

                // this is the defining class loader; record the stats
            }
        }
        return c;
}
```
#### DexClassLoader与PathClassLoader   
```java
class DexClassLoader extends BaseDexClassLoader {
    // dexPath:包含dex文件的路径，如apk或者包含dex文件的jar包    
    // optimizedDirectory：这个是从apk中释放出的dex文件的保存路径  
    // librarySearchPath：顾名思义lib搜素路径，一般为null   
    // parent：父加载器  
    public DexClassLoader(String dexPath, String optimizedDirectory, String libraryPath, ClassLoader parent) {
        super(dexPath, new File(optimizedDirectory), libraryPath, parent);
    }
}
```
```java
class PathClassLoader extends BaseDexClassLoader {
    public PathClassLoader(String dexPath, ClassLoader parent) {
        super(dexPath, null, null, parent);
    }

    public PathClassLoader(String dexPath, String libraryPath, ClassLoader parent) {
        super(dexPath, null, libraryPath, parent);
    }
}
```
这两者只是简单的对BaseDexClassLoader做了一下封装，具体的实现还是在父类里。   
但是两者还是有区别的，PathClassLoader的optimizedDirectory只能是null     
optimizedDirectory是用来缓存我们需要加载的dex文件的，并创建一个DexFile对象，如果它为null，那么会直接使用dex文件原有的路径来创建DexFile对象。   
DexClassLoader可以指定自己的optimizedDirectory，所以它可以加载外部的dex，因为这个dex会被复制到内部路径的optimizedDirectory；而PathClassLoader没有optimizedDirectory，所以它只能加载内部的dex，这些大都是存在系统中已经安装过的apk里面的。     

1) DexClassLoader：可以加载jar/apk/dex，可以从SD卡中加载未安装的apk；
2) PathClassLoader：要传入系统中apk的存放Path，所以只能加载已经安装的apk文件；

**参考链接：**    
[深入探讨 Java 类加载器](https://www.ibm.com/developerworks/cn/java/j-lo-classloader/)    
[Android插件化学习之路（二）之ClassLoader完全解析](http://blog.csdn.net/u012124438/article/details/53235848)  
[Android插件化框架系列之类加载器](http://www.jianshu.com/p/57fc356b9093)   

### 加载类的过程   
java.lang.Object    
   ↳	java.lang.ClassLoader   
 	   ↳	dalvik.system.BaseDexClassLoader   
 	 	   ↳	dalvik.system.DexClassLoader / dalvik.system.PathClassLoader     

1. Android中，ClassLoader用loadClass方法来加载我们需要的类  
2. loadClass方法调用了findClass方法，而BaseDexClassLoader重载了这个方法   
3. 结果还是调用了DexPathList的findClass       
4. 最后调用了Native方法defineClass加载类   


### 调用.dex中的代码  
Java程序中，JVM虚拟机是通过类加载器ClassLoader加载.jar文件里面的类的。Android也类似，不过android用的是Dalvik/ART虚拟机，不是JVM，也不能直接加载.jar文件，而是加载dex文件。  

先要通过Android SDK提供的DX工具把.jar文件优化成.dex文件，然后Android的虚拟机才能加载。注意，有的Android应用能直接加载.jar文件，那是因为这个.jar文件已经经过优化，只不过后缀名没改（其实已经是.dex文件）。   


调用普通的逻辑代码可以通过下面两种方式：   
1. 使用DexClassLoader加载进来的类，我们本地并没有这些类的源码，所以无法直接调用，不过可以通过反射的方法调用。  
2. 毕竟.dex文件也是我们自己维护的，所以可以把方法抽象成公共接口，把这些接口也复制到主项目里面去，就可以通过这些接口调用动态加载得到的实例的方法了。

**参考链接：**   
[Android动态加载dex技术初探](http://blog.csdn.net/u013478336/article/details/50734108)   
[Android插件化学习之路（三）之调用外部.dex文件中的代码](http://blog.csdn.net/u012124438/article/details/53236472)  


## 资源访问   
res里的每一个资源都会在R.Java里生成一个对应的Integer类型的id，APP启动时会先把R.java注册到当前的上下文环境，我们在代码里以R文件的方式使用资源时正是通过使用这些id访问res资源，然而插件的R.java并没有注册到当前的上下文环境，所以插件的res资源也就无法通过id使用了。   

平时我们所访问资源一般是通过`getResources().getXXX()`的方式来获取的。  
因为宿主程序中并没有插件的资源，所以通过R来加载插件的资源是行不通的，程序会抛出异常：无法找到某某id所对应的资源。     
所以我们需要先获取到插件的Resources:   
```java
private Resources getPlugResources() {
    // 获取插件包中的资源
    AssetManager assetManager = null;
    try {
        assetManager = AssetManager.class.newInstance();
        Method addAssetPath = assetManager.getClass().getMethod("addAssetPath", String.class);
        addAssetPath.invoke(assetManager, mDexPath);
    } catch (Exception e) {
        e.printStackTrace();
    }
    if (assetManager == null) {
        return null;
    }
    Resources superRes = super.getResources();
    mPlugResources = new Resources(assetManager, superRes.getDisplayMetrics(),
            superRes.getConfiguration());

    return mPlugResources;
}

public int getColor(String colorName) {
    if (mPlugResources == null) {
        getPlugResources();
    }
    try {
        return mPlugResources.getColor(mPlugResources.getIdentifier(colorName, COLOR, PLUG_NAME));

    } catch (Resources.NotFoundException e) {
        e.printStackTrace();
        return -1;
    }
}
```    

[插件化知识梳理(9) - 资源的动态加载示例及源码分析](http://www.jianshu.com/p/86dbf0360348)    

##  调用插件中的Activity    
apk被宿主程序调起以后，apk中的activity其实就是一个普通的对象，不具有activity的性质，因为系统启动activity是要做很多初始化工作的，而我们在应用层通过反射去启动activity是很难完成系统所做的初始化工作的，所以activity的大部分特性都无法使用包括activity的生命周期管理，这就需要我们自己去管理。当然还需要我们常说的上下文Context   

需要先简单的了解一下Activity的[启动流程](http://www.jianshu.com/p/1035ffd9e9cf)。  
在此就直接记录重要的结论：  
>在`ActivityThread.java`的`performLaunchActivity`方法中，该方法通过Instrumentation的newActivity创建了activity类，接着完成了application的创建（没有创建Application的情况下，在该方法中就创建了Application的context），接着通过createBaseContextForActivity方法为该activity创建context，再调用attach方法进行绑定。  

正常的的Activity被AMS反射调用，在attach后就有了Context，那我们自己反射的Activity要想有Context，就要模拟AMS调用方式，构造Context，但是这相当于再写个系统，不可实现，那怎么办？

遇到问题，解决问题。   
插件中被反射的activity没有了Context，我们可以把主apk的Acitvity的Context传递给插件Acitivity。  
在宿主APK注册一个ProxyActivity（代理Activity），就是作为占坑使用。每次打开插件APK里的某一个Activity的时候，都是在宿主里使用启动ProxyActivity，然后在ProxyActivity的生命周期里方法中，调用插件中的Activity实例的生命周期方法，从而执行插件APK的业务逻辑。所以思路就来了：  
第一、ProxyActivity中需要保存一个Activity实例，该实例记录着当前需要调用插件中哪个Activity的生命周期方法。   
第二、ProxyActivity如何调用插件apk中Activity的所有生命周期的方法,使用反射呢？还是其他方式（接口）。  

缺点：  
1. 插件Activity不能使用this关键字，比如this.finish()方法是无效的，真正掌管生命周期的是proxy应该调用proxy.finish()，所以百度开源框架 dynamic-load-apk使用that指向proxy，约定插件中使用that来代替this。  
2. 插件Activity无法深度演绎真正的Activity组件，可能有些高级特性无法使用。  
3. 启动新activity的约束：启动外部activity不受限制，启动apk内部的activity有限制，首先由于apk中的activity没注册，所以不支持隐式调用，其次必须通过BaseActivity中定义的新方法startActivityByProxy和startActivityForResultByProxy，还有就是不支持LaunchMode。


### 生命周期管理   
1. 反射  
2. 接口  


## 动态创建Activity  
使用代理Activity有一些限制:  
1. 实际运行的Activity实例其实都是ProxyActivity，并不是真正想要启动的Activity；  
2. ProxyActivity只能指定一种LaunchMode，所以插件里的Activity无法自定义LaunchMode；  
3. 不支持静态注册的BroadcastReceiver；  
4. 往往不是所有的apk都可作为插件被加载，插件项目需要依赖特定的框架，还有需要遵循一定的”开发规范”；  

解决对策就是，在需要启动插件的某一个Activity（比如PlugActivity）的时候，动态创建一个TargetActivity，新创建的TargetActivity会继承PlugActivity的所有共有行为，而这个TargetActivity的包名与类名刚好与我们事先注册的TargetActivity一致，我们就能以标准的方式启动这个Activity。   

启动Activity是一个复杂的过程，有很多环节：Activity.startActivity()->Activity.startActivityForResult()->Instrument.excuteStartActivity()->ASM.startActivity()。大概又这么几个环节，详细了解可以参考文章：[《深入理解Activity的启动过程》](http://www.cloudchou.com/android/post-788.html)。 所谓“占坑”在宿主端的AndroidManifest.xml注册一个不存在的Activity，可以取名为StubActivity，同样启动插件的Activity都是启动StubActivity，然后在启动Activity的某个环节，我们找个“临时”演员来代替StubActivity，这个临时演员就是插件中定义的Activity，这叫“瞒天过海”。如何找“临时”演员？本章要讲的重点：使用 dexmaker 临时改造插件Activity。


[插件化研究之dexmaker动态生成Activity](http://www.jianshu.com/p/7a9d52e73d05)  
[Android插件化学习之路（六）之动态创建Activity](http://blog.csdn.net/u012124438/article/details/53239497)  

### HOOK Activity  
在了解了Activity的启动流程之后，我们得知是`mInstrumentation.newActivity()`创建的Activity。   
我们要做的就是通过替换掉Instrumentation类，达到定制插件运行环境的目的。  

#### 简单的HOOK
1. 替换Instrumentation类  
```java
// 先获取到当前的ActivityThread对象
Class<?> activityThreadClass = Class.forName("android.app.ActivityThread");
Method currentActivityThreadMethod = activityThreadClass.getDeclaredMethod("currentActivityThread");
currentActivityThreadMethod.setAccessible(true);
Object currentActivityThread = currentActivityThreadMethod.invoke(null);

// 拿到原始的 mInstrumentation字段
Field mInstrumentationField = activityThreadClass.getDeclaredField("mInstrumentation");
mInstrumentationField.setAccessible(true);
Instrumentation mInstrumentation = (Instrumentation) mInstrumentationField.get(currentActivityThread);

//如果没有注入过，就执行替换
if (!(mInstrumentation instanceof PluginInstrumentation)) {
    PluginInstrumentation pluginInstrumentation = new PluginInstrumentation(mInstrumentation);
    mInstrumentationField.set(currentActivityThread, pluginInstrumentation);
}
```

2. 替换newActivity()  
```java
@Override
public Activity newActivity(ClassLoader cl, String className, Intent intent)
        throws InstantiationException, IllegalAccessException, ClassNotFoundException {
    if (intent != null) {
        isPlugin = intent.getBooleanExtra(PluginCons.FLAG_ACTIVITY_FROM_PLUGIN, false);
    }
    if (isPlugin && intent != null) {
        className = intent.getStringExtra(PluginCons.FLAG_ACTIVITY_CLASS_NAME);
    }
    return super.newActivity(cl, className, intent);
}
```  

HOOK：   
[8个类搞定插件化——Activity实现方案](https://kymjs.com/code/2016/05/15/01/)   
[Android 插件化原理解析——Activity生命周期管理](http://weishu.me/2016/03/21/understand-plugin-framework-activity-management/)  

动态代理：  
[知识总结 插件化学习 Activity加载分析](http://www.jianshu.com/p/127ecc0c7567)   
[Android插件化系列第（五）篇---Activity的插件化方案（代理模式）](http://www.jianshu.com/p/7b2cc534d097)    

## 各类分析文章  
[插件化框架android-pluginmgr全解析](http://www.jianshu.com/p/b8ef0a92c060)  
[Dynamic-Load-Apk源码解析](http://www.jianshu.com/p/30114b7176a3)  
[Android 全面插件化 RePlugin 流程与源码解析](https://juejin.im/post/59752eb1f265da6c3f70eed9)  
[Android插件化快速入门与实例解析（VirtualApk）](https://juejin.im/post/596b80ebf265da6c4d1bdfbe)  


## 系列文章   
[Android动态加载技术 系列索引](https://segmentfault.com/a/1190000004086213)   
[Android插件化原理解析——概要](http://weishu.me/2016/01/28/understand-plugin-framework-overview/)  
[Android插件化入门指南](http://lruheng.com/2017/07/01/Android%E6%8F%92%E4%BB%B6%E5%8C%96%E5%85%A5%E9%97%A8%E6%8C%87%E5%8D%97/)  
