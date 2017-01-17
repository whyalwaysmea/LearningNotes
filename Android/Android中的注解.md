## 准备工作
默认情况下,Android中的注解包并没有包括在framework中,它独立成一个单独的包,通常我们需要引入这个包.
```
dependencies {
    compile 'com.android.support:support-annotations:22.2.0'
}
```
但是如果我们已经引入了appcompat则没有必要再次引用support-annotations,因为appcompat默认包含了对其引用.

## 替代枚举
在最早的时候,当我们想要做一些值得限定实现枚举的效果,通常是

* 定义几个常量用于限定
* 从上面的常量选取值进行使用

```java
public static final int COLOR_RED = 0;
public static final int COLOR_GREEN = 1;
public static final int COLOR_YELLOW = 2;

public void setColor(int color) {
    //some code here
}
//调用
setColor(COLOR_RED)
```
然而上面的还是有不尽完美的地方

* setColor(COLOR_RED)与setColor(0)效果一样,而后者可读性很差,但却可以正常运行
* setColor方法可以接受枚举之外的值,比如setColor(3),这种情况下程序可能出问题

一个相对较优的解决方法就是使用Java中的Enum.使用枚举实现的效果如下
```Java
// ColorEnum.java
public enum ColorEmun {
    RED,
    GREEN,
    YELLOW
}

public void setColorEnum(ColorEmun colorEnum) {
    //some code here
}

setColorEnum(ColorEmun.GREEN);
```
然而Enum也并非最佳,Enum因为其相比方案一的常量来说,占用内存相对大很多而受到曾经被Google列为不建议使用,为此Google特意引入了一些相关的注解来替代枚举.
```java
public class Colors {
    // 1.声明必要的int常量
    public static final int RED = 0;
    public static final int GREEN = 1;
    public static final int YELLOW = 2;
    public static final int BLACK = 3;

    // 2.声明一个注解为LightColors
    // 3.使用@IntDef修饰LightColors,参数设置为待枚举的集合
    // 4.使用@Retention(RetentionPolicy.SOURCE)指定注解仅存在与源码中,不加入到class文件中
    @IntDef({RED, GREEN, YELLOW})
    @Retention(RetentionPolicy.SOURCE)
    public @interface LightColors{}

    // 5. 方法限制
    public void setColor(@LightColors int lightColors) {

    }
}
// 调用
Colors colors = new Colors();
// 如果这里传入数值会报错，传入未设置的值(如Colors.BLACK)也会报错
colors.setColor(Colors.YELLOW);
```

## Null相关的注解
和Null相关的注解有两个

* @Nullable 注解的元素可以是Null
* @NonNull 注解的元素不能是Null

上面的两个可以修饰如下的元素
* 成员属性
* 方法参数
* 方法的返回值

## 区间范围注解
Android中的IntRange和FloatRange是两个用来限定区间范围的注解,
```java
private float setProgress(@FloatRange(from=0.0f, to=1.0f) float progress) {
    return progress;
}
```
如果`setProgress(5)` 就会报错

## 长度以及数组大小限制
限制字符串的长度
```java
private void setKey(@Size(6) String key) {
}
```
限定数组集合的大小
```java
private void setData(@Size(max = 1) String[] data) {
}
setData(new String[]{"b", "a"});//error occurs
```
限定特殊的数组长度,比如3的倍数
```java
private void setItemData(@Size(multiple = 3) String[] data) {
}
```

## 权限相关
在Android中,有很多场景都需要使用权限,无论是Marshmallow之前还是之后的动态权限管理.都需要在manifest中进行声明,如果忘记了,则会导致程序崩溃. 好在有一个注解能辅助我们避免这个问题.使用RequiresPermission注解即可.
```java
@RequiresPermission(Manifest.permission.SET_WALLPAPER)
public void changeWallpaper(Bitmap bitmap) throws IOException {}
```

## 资源注解
在Android中几乎所有的资源都可以有对应的资源id.比如获取定义的字符串,我们可以通过下面的方法
```java
public String getStringById(int stringResId) {
    return getResources().getString(stringResId);
}
```
使用这个方法,我们可以很容易的获取到定义的字符串,但是这样的写法也存在着风险.
如果我们在不知情或者疏忽情况下,传入错误值,就会出现问题. 但是如果我们使用资源相关的注解修饰了参数,就能很大程度上避免错误的情况.
```java
public String getStringById(@StringRes int stringResId) {
    return getResources().getString(stringResId);
}
```

## CheckResult
这是一个关于返回结果的注解，用来注解方法，如果一个方法得到了结果，却没有使用这个结果，就会有错误出现，一旦出现这种错误，就说明你没有正确使用该方法。
```java
@CheckResult
public String trim(String s) {
    return s.trim();
}
```

## CallSuper
重写的方法必须要调用super方法
使用这个注解,我们可以强制方法在重写时必须调用父类的方法 比如Application的onCreate,onConfigurationChanged等.

## Keep
在Android编译生成APK的环节,我们通常需要设置minifyEnabled为true实现下面的两个效果

* 混淆代码
* 删除没有用的代码

但是出于某一些目的,我们需要不混淆某部分代码或者不删除某处代码,除了配置复杂的Proguard文件之外,我们还可以使用@Keep注解 .
```java
@Keep
public static int getBitmapWidth(Bitmap bitmap) {
    return bitmap.getWidth();
}
```

## 相关链接

[探究Android中的注解](http://droidyue.com/blog/2016/08/14/android-annnotation/)
