# Activity 的 Flags

Activity的Flags有很多，作用也各不一样。有的标记位可以设定Activity的启动模式，有的标记位可以影响Activity的运行状态。

* FLAG_ACTIVITY_NEW_TASK

单独使用并没有什么作用。

配合FLAG_ACTIVITY_CLEAR_TASK使用，栈内就会只剩一个Activity

如果配合Task Affinity使用：
>1.如果D这个Activity在Manifest.xml中的声明中添加了Task Affinity，系统首先会查找有没有和D的Task Affinity相同的Task栈存在，如果有存在，将D压入那个栈.如果没有就创建一个新栈，并入栈
2.如果D这个Activity在Manifest.xml中的Task Affinity默认没有设置，则会把其压入栈1，变成：A B C D，这样就和标准模式效果是一样的了。

* FLAG_ACTIVITY_SINGLE_TOP

作用是为Activity指定“singleTop”启动模式，其效果和在XML中指定该启动模式相同。

* FLAG_ACTIVITY_CLEAR_TOP

具有此标记位的Activity，当它启动时，在同一个任务栈中所有位于它上面的Activity都要出栈。
这个模式一般需要和FLAG_ACTIVITY_NEW_TASK配合使用，在这种情况下，被启动Activity的实例如果已经存在，那么系统就会调用它的onNewIntent。如果呗启动的Activity采用standard模式启动，那么它连同它之上的Activity都要出栈。
singleTask启动模式默认就具有此标记位的效果。

* FLAG_ACTIVITY_NO_HISTORY

具有这个标记的Activity不会出现在历史Activity的列表中，当某些情况下我们不希望用户通过历史列表回到我们的Activity的时候这个标记比较有用。
例如A-B,B中以这种模式启动C，C再启动D，则当前Activity的栈为ABCD。但是退出D的时候，C也跟着执行了onDestroy
它等同于在XML中指定Activity的属性android:noHistory="true"

## 相关链接
[Android Intent的FLAG标志详解](http://www.jianshu.com/p/537aa221eec4)
