# 65536异常
在Android中单个dex文件所能够包含的最大方法数为65536，这包含Android FrameWork、依赖的jar包以及应用本身的代码中的所有方法。
当应用的方法数达到65536后，编译器就无法完成编译工作并抛出异常。

1. 生成的apk在android 2.3或之前的机器上无法安装，提示INSTALL_FAILED_DEXOPT
2. 方法数量过多，编译时出错，提示：

> Conversion to Dalvik format failed:Unable to execute dex: method ID not in [0, 0xffff]: 65536

而问题的产生原因如下：

1. 无法安装（Android 2.3 INSTALL_FAILED_DEXOPT）问题，是由dexopt的LinearAlloc限制引起的，在Android版本不同分别经历了4M/5M/8M/16M限制，目前主流4.2.x系统上可能都已到16M， 在Gingerbread或者以下系统LinearAllocHdr分配空间只有5M大小的， 高于Gingerbread的系统提升到了8M。Dalvik linearAlloc是一个固定大小的缓冲区。在应用的安装过程中，系统会运行一个名为dexopt的程序为该应用在当前机型中运行做准备。dexopt使用LinearAlloc来存储应用的方法信息。Android 2.2和2.3的缓冲区只有5MB，Android 4.x提高到了8MB或16MB。当方法数量过多导致超出缓冲区大小时，会造成dexopt崩溃。
2. 超过最大方法数限制的问题，是由于DEX文件格式限制，一个DEX文件中method个数采用使用原生类型short来索引文件中的方法，也就是4个字节共计最多表达65536个method，field/class的个数也均有此限制。对于DEX文件，则是将工程所需全部class文件合并且压缩到一个DEX文件期间，也就是Android打包的DEX过程中， 单个DEX文件可被引用的方法总数（自己开发的代码以及所引用的Android框架、类库的代码）被限制为65536；

# 可行的解决办法
1. 加大Proguard的力度来减小DEX的大小和方法数，但这是治标不治本的方案，随着业务代码的添加，方法数终究会到达这个限制
2. 插件化方案，但是插件化是一套重量级的技术方案，并且其兼容性问题往往较多，从单纯的解决方法数越界的角度来说，插件化并不是一个非常适合的方案。
3. 采用google提供的MultiDex方案

# MultiDex的使用

# MultiDex的坑
### 启动时间过长
由于应用启动时会加载额外的dex文件，这将导致应用的启动速度降低。
Application.attachBaseContext是我们能控制的最早执行的代码，在这个方法里面执行MultiDex.install()无疑是最佳时机。
还有一点我们需要了解，首次启动时Dalvik虚拟机会对classes.dex执行dexopt操作，生成ODEX文件，这个过程非常耗时，而执行MultiDex.install()必然会再次对classes2.dex执行dexopt等操作，所有这些操作必须在5秒内完成，否则就ANR给你看！因此应该避免其他的dex文件过大
非首次启动则直接从cache中读取已经执行过dexopt的ODEX文件，这个过程对启动并无太大影响。
基于此，对attachBaseContext稍作改动：
```java
@Override
protected void attachBaseContext(final Context base) {
    super.attachBaseContext(base);
    initBeforeDex2Installed();

    if (isFirstLaunch()) {
        // 首次启动
        new Thread(new Runnable() {

            @Override
            public void run() {
                MultiDex.install(base);
                initAfterDex2Installed();
            }
        }).start();
    } else {
        // 非首次启动
        MultiDex.install(base);
        initAfterDex2Installed();
    }
}
```
首次启动开启一个线程来加载classes2.dex，防止阻塞UI线程，非首次启动则同步执行。

initAfterDex2Installed()方法是根据Classes2.dex中结果，将涉及到的相关初始化工作移到classes2.dex加载完之后执行，避免启动问题。

建议在classes2.dex加载完成前，设置一个启动等待界面，之后再进入主界面，确保用户体验。

### ANR/Crash
在冷启动时因为需要安装DEX文件，如果DEX文件过大时，处理时间过长，很容易引发ANR（Application Not Responding）；

### 4.0之前的bug
由于Dalvik linearAlloc的bug，这可能导致使用MultiDex的应用无法在Android4.0以前的手机上运行，因此需要做大量的兼容性测试。同时由于Dalvik linearAlloc的bug，有可能出现应用在运行中采用了MultiDex方案从而产生大量的内存消耗情况，这会导致应用崩溃。但是这种现象目前极少遇到。


# MultiDex的一种改进实现
美团的 Dex自动拆包及动态加载方案

我们发现MultiDex在冷启动时容易导致ANR的瓶颈， 在2.1版本之前的Dalvik的VM版本中， MultiDex的安装大概分为几步，第一步打开apk这个zip包，第二步把MultiDex的dex解压出来（除去Classes.dex之外的其他DEX，例如：classes2.dex， classes3.dex等等)，因为android系统在启动app时只加载了第一个Classes.dex，其他的DEX需要我们人工进行安装，第三步通过反射进行安装，这三步其实都比较耗时， 为了解决这个问题我们考虑是否可以把DEX的加载放到一个异步线程中，这样冷启动速度能提高不少，同时能够减少冷启动过程中的ANR，对于Dalvik linearAlloc的一个缺陷(Issue 22586)和限制(Issue 78035)，我们考虑是否可以人工对DEX的拆分进行干预，使每个DEX的大小在一定的合理范围内，这样就减少触发Dalvik linearAlloc的缺陷和限制； 为了实现这几个目的，我们需要解决下面三个问题：

1. 在打包过程中如何产生多个的DEX包？
2. 如果做到动态加载，怎么决定哪些DEX动态加载呢？
3. 如果启动后在工作线程中做动态加载，如果没有加载完而用户进行页面操作需要使用到动态加载DEX中的class怎么办？

### 解决办法
1. 为了实现产生多个DEX包，我们可以在生成DEX文件的这一步中， 在Ant或gradle中自定义一个Task来干预DEX产生的过程，从而产生多个DEX
2. 寻找启动类.这个需要分析出放到Main DEX中的class依赖，需要确保把Main DEX中class所有的依赖都要放进来，否则在启动时会发生ClassNotFoundException。
3. 如果我们在后台加载Secondary DEX过程中，用户点击界面将要跳转到使用了在Secondary DEX中class的界面， 那此时必然发生ClassNotFoundException, 那怎么解决这个问题呢，在所有的Activity跳转代码处添加判断Secondary DEX是否加载完成？这个方法可行，但工作量非常大;那有没有更好的解决方案呢？我们通过分析Activity的启动过程，发现Activity是由ActivityThread 通过Instrumentation来启动的，我们是否可以在Instrumentation中做一定的手脚呢？通过分析代码ActivityThread和Instrumentation发现，Instrumentation有关Activity启动相关的方法大概有：execStartActivity、newActivity等等，这样我们就可以在这些方法中添加代码逻辑进行判断这个Class是否加载了，如果加载则直接启动这个Activity，如果没有加载完成则启动一个等待的Activity显示给用户，然后在这个Activity中等待后台Secondary DEX加载完成，完成后自动跳转到用户实际要跳转的Activity

# 小结
* 提高设计与代码质量应该可以不必被方法数限制困扰
* 多试验首次启动App，以观察启动log是必须的，除了测试MultiDex是否会对首次启动时间产生明显影响，最重要的还是查看启动过程中是否有找不到的类。
* 通常多次云测也是必须的，毕竟测试能覆盖到的机型有限，云测也节省了测试工作量。

# 相关链接
[Android MultiDex实践：如何绕过那些坑？](http://mp.weixin.qq.com/s?__biz=MzA4MjU5NTY0NA==&mid=405574783&idx=1&sn=6ff49fda8a7229bf6b2692fddcf23e04#rd)

[美团Android DEX自动拆包及动态加载简介](http://tech.meituan.com/mt-android-auto-split-dex.html)
