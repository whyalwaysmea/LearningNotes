
# Activity启动模式

## 任务栈
任务栈是一种后进先出的结构。位于栈顶的Activity处于焦点状态,当按下back按钮的时候,栈内的Activity会一个一个的出栈,并且调用其onDestory()方法。如果栈内没有Activity,那么系统就会回收这个栈,每个APP默认只有一个栈,以APP的包名来命名.


## LaunchMode
* standard标准模式：

  系统默认模式，每次启动一个activity都会创建一个新的实例压在栈顶，不管这个activity是否已经存在。被创建的实例的生命周期符合典型情况下Activity的生命周期。在这种模式下，谁启动了这个Activity，那么这个Activity就运行在启动它的那个Activity所在的栈中。

  当我们用ApplicationContext去启动standard模式的Activity的时候会报错，

```java
android.util.AndroidRuntimeException: Calling startActivity() from outside of an
 Activity  context requires the FLAG_ACTIVITY_NEW_TASK flag. Is this really what you want?
```
  这是因为standard模式的Activity默认会进入启动它的Activity所属的任务栈中，但是由于非Activity类型的Context(如ApplicationContext)并没有所谓的任务栈。解决这个问题的方法就是为待启动的Activity指定FLAG_ACTIVITY_NEW_TASK标记位，这样启动的时候就会为它创建一个新的任务栈，这个时候待启动Activity实际上是以singleTask模式启动的。

* singleTop栈顶复用模式：
  如果新Activity已经位于任务栈的栈顶，那么Activity不会被重新创建，所以它的启动三回调(onCreate, onStart, onResume)就不会执行，同时Activity的onNewIntent()方法会被回调。如果Activity已经存在，但是不再栈顶，那么作用和standard模式一样。

* singleTask栈内单例：
activity只会在任务栈里面存在一个实例。如果要激活的activity，在任务栈里面已经存在，就不会创建新的activity，而是复用这个已经存在的activity，调用 onNewIntent() 方法，并且清空这个activity任务栈上面所有的activity。应用场景：大多数App的主页。对于大部分应用，当我们在主界面点击回退按钮的时候都是退出应用，那么当我们第一次进入主界面之后，主界面位于栈底，以后不管我们打开了多少个Activity，只要我们再次回到主界面，都应该使用将主界面Activity上所有的Activity移除的方式来让主界面Activity处于栈顶，而不是往栈顶新加一个主界面Activity的实例，通过这种方式能够保证退出应用时所有的Activity都能报销毁。  
在启动 singleTask 的 Activity时，系统会先检查是否有 Task 的 affinity 值与该 Activity 的 taskAffinity 相同，如果有，Activity 就会在这个 Task 中启动，否则就会在新的 Task 中启动。因此，如果我们想要设置了 singleTask 启动模式的 Activity 在新的 Task 中启动，就要为它设置一个独立的 taskAffinity 属性值。如果不是在新的 Task 中启动，它会在已有的 Task 中查看是否已经存在相应的 Activity 实例，如果存在，就会把位于这个 Activity 实例上面的 Activity 全部结束掉，即最终这个 Activity 实例会位于该 Task 的栈顶，并调用该实例的 onNewIntent() 方法。

* singleInstance 单实例模式：
这是一种加强的singleTask。此种模式的Activity只能单独地位于一个任务栈中，换句话说，比如Activity A是singleInstance模式，当A启动后，系统会为它创建一个新的任务栈，然后A独自在这个新的任务栈中，由于栈的复用性，后续的请求均不会创建新的Activity A。

## TaskAffinity
在singleTask启动模式中，多次提到某个Activity所需的任务栈，什么是Activity所需要的任务栈的？
这要从一个参数说起：**TaskAffinity**，可以翻译为任务相关性。这个参数标识了一个Activity所需要的任务栈的名字，默认情况下，所有Activity所需的任务栈的名字为应用的包名。当然，我们可以为每个Activity都单独指定TaskAffinity属性，这个属性值必须不能和包名相同，否则就相当于没有指定。
TaskAffinity属性主要和singleTask启动模式或者allowTaskReparenting属性配对使用。

### 案例
情况1：新建两个app。
app1什么都不改，自带一个MainActivity（布局文件标识一下是app1的Mainactivity）。
在app2的manifest给app2的MainAcivity添加android:taskAffinity属性，指定包名为 app1的包名。

我们把两个程序安装一下，清掉其他正在运行的程序
运行app1，然后按下home键（让app1变成后台程序），接着打开app2，我们会发现，这时候进入的不是app2的界面，而是app1的。

## allowTaskReparenting
 allowTaskReparenting用来标记Activity能否从启动的Task移动到taskAffinity指定的Task，默认是继承至application中的allowTaskReparenting=false，如果为true，则表示可以更换；false表示不可以。

### 案例
应用A中两个Activity，LaunchMode1Activity的模式定义为singleTask，allowTaskReparenting设置为true，为了别的应用可以访问该页面，exportd属性设置为true.
```xml
<activity android:name=".MainActivity">
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />

        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity>
<activity
    android:name="._1activity.LaunchMode1Activity"
    android:allowTaskReparenting="true"
    android:exported="true">
</activity>
```
另一个应用B中启动应用A中的LaunchMode1Activity。
```Java
Intent intent = new Intent(Intent.ACTION_VIEW);
intent.setClassName(packageName, LaunchMode1Activity);
startActivity(intent);
```
应用A的LaunchMode1Activity启动后，点击home键回到桌面，然后点应用A的启动图标，发现没有进入应用A的MainActivity，而是进入应用A的LaunchMode1Activity。
这是为什么呐？
正常情况下，如果LaunchMode1Activity没有设置allowTaskReparenting为true，则应用B启动LaunchMode1Activity时，LaunchMode1Activity会进入应用B所在的任务栈中。此时点击home回到桌面，再点应用A的图标，一定会到应用A的MainActivity.

但是，如果LaunchMode1Activity的allowTaskReparenting属性设为true时，LaunchMode1Activity就可以改变从属任务栈，也就是当应用B启动他时，他会从B的任务栈自动移到他自己的任务栈，也就是应用A的包名的名称的任务栈。
当从桌面启动应用A时，发现A的任务栈已经存在，就认为上次A没有退出去，最后一个页面是栈中的LaunchMode1Activity，所以直接显示LaunchMode1Activity。


### 案例singleTask测试
singleTask 不设置 taskAffinity：
![singleTask 不设置 taskAffinity](https://imgs.babits.top/2017010852activity_single_task_without_task_affinity.png)   
只有一个任务栈  
-----

singleTask 设置 taskAffinity 为包名：
![singleTask 设置 taskAffinity 为包名](https://imgs.babits.top/2017010884648activity_single_task_with_same_task_affinity.png)   
只有一个任务栈
-----


singleTask 设置 taskAffinity 为包名以外的值：
![singleTask 设置 taskAffinity 为包名以外的值](https://imgs.babits.top/2017010879551activity_single_task_with_different_task_affinity.png)   
有两个任务栈
-----


Activity A为standart模式， ActivityB 和 ActivityC为singleTask模式，taskAffinity相同，为包名以外的值(假设为:com.haha)。  
启动顺序 A->B->C->A->B   
这个时候返回，就显示A，再返回就到桌面了。

解析：
一开始A在包名的栈下面，启动B，这个时候创建com.haha栈和B实例，启动C，创建C实例进入com.haha栈中。    
C启动A，创建一个A的实例，进入com.haha栈中。     
A启动B，B回到栈顶，CA出栈。  
在B的时候按返回，B出栈， 后台的包名栈到前台，显示A。


### 案例singleInstance测试
singleInstance 不设置 taskAffinity：
![singleInstance 不设置 taskAffinity](https://imgs.babits.top/2017011786410activity_single_instance_without_task_affinity.png)   
有两个任务栈
-----


singleInstance 设置 taskAffinity 为包名：
![singleInstance 设置 taskAffinity 为包名](https://imgs.babits.top/2017011752729activity_single_instance_with_same_task_affinity.png)  
有两个任务栈
-----


singleInstance 设置 taskAffinity 为包名以外的值：
![singleInstance 设置 taskAffinity 为包名以外的值](https://imgs.babits.top/2017011751645activity_single_instance_with_different_task_affinity.png)
有两个任务栈
-----


singleInstance（设置 taskAffinity 为包名以外的值）启动 standard（不设置 taskAffinity）：
![singleInstance（设置 taskAffinity 为包名以外的值）启动 standard（不设置 taskAffinity）](https://imgs.babits.top/2017011751555single_instance_difftaskaff_launch_standard_notaskaff.png)
有两个任务栈,新启动的standard在最开始的那个任务栈中


## 设置LaunchMode
给Activity指定启动模式，有两种方法：第一种是通过AndroidMenifest为Activity指定启动模式；另一种是通过在Intent中设置标志位来为Activity指定启动模式。

区别：
1. 优先级上第二种方式的优先级要高于第一种，当两种同时存在时，以第二种方式为准；
2. 两种方式在限制范围上有所不同：第一种方式无法直接为Activity设定FLAG_ACTIVITY_CLEAR_TOP标识，而第二种方式无法为Activity指定singleInstance模式


## LaunchMode与StartActivityForResult

我们在开发过程中经常会用到StartActivityForResult方法启动一个Activity，然后在onActivityResult()方法中可以接收到上个页面的回传值，但你有可能遇到过拿不到返回值的情况，那有可能是因为Activity的LaunchMode设置为了singleTask。5.0之后，android的LaunchMode与StartActivityForResult的关系发生了一些改变。两个Activity，A和B，现在由A页面跳转到B页面，看一下LaunchMode与StartActivityForResult之间的关系：


![before 5.0](http://upload-images.jianshu.io/upload_images/1187237-144638fbf8298061.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![after5.0](http://upload-images.jianshu.io/upload_images/1187237-864d6df150cf2142.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 相关链接
[我打赌你一定没搞明白的Activity启动模式](http://www.jianshu.com/p/2a9fcf3c11e4#)

[安卓基础：task, launchMode, Intent flag](https://blog.piasy.com/2017/01/16/Android-Basics-Task-and-LaunchMode/)
