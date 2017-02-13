## 类的继承结构
![Context类的继承结构](http://upload-images.jianshu.io/upload_images/1187237-1b4c0cd31fd0193f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Context类本身是一个纯abstract类,他有两个具体的实现子类：ContextImpl和ContextWrapper

ContextImpl类则真正实现了Context中的所以函数，应用程序中所调用的各种Context类的方法，其实现均来自于该类。

ContextWrapper下面还有三个子类，分别是：ContextThemeWrapper，Application，Service。ContextWrapper看名字就可以知道，它是一个Context的包装类，所以在它的构造方法中需要传入一个真正的Context。

ContextThemeWrapper类，如其名所言，其内部包含了与主题（Theme）相关的接口，这里所说的主题就是指在AndroidManifest.xml中通过android：theme为Application元素或者Activity元素指定的主题。当然，只有Activity才需要主题，Service是不需要主题的，因为Service是没有界面的后台场景，所以Service直接继承于ContextWrapper，Application同理。

## Context的作用域



## 相关链接

[Android Context 上下文 你必须知道的一切](http://blog.csdn.net/lmj623565791/article/details/40481055)

[Difference between getContext() , getApplicationContext() , getBaseContext() and “this”](http://stackoverflow.com/questions/10641144/difference-between-getcontext-getapplicationcontext-getbasecontext-and)

[Android中Context详解 ---- 你所不知道的Context](http://blog.csdn.net/qinjuning/article/details/7310620)

[android学习—— context 和 getApplicationContext()](http://blog.csdn.net/janronehoo/article/details/7348566)

[Context都没弄明白，还怎么做Android开发？](http://www.jianshu.com/p/94e0f9ab3f1d)

[Android Context完全解析，你所不知道的Context的各种细节](http://blog.csdn.net/guolin_blog/article/details/47028975)
