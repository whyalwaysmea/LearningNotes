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


### 资源访问   
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
