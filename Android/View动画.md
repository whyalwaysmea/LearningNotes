View动画的作用对象是View，它支持4种动画效果，分别是平移动画，缩放动画 ，旋转动画，透明度动画。
帧动画也属于View动画，但是帧动画的表现形式和上面的四种变化效果不太一样。所以这里分开讲解

## View动画
View动画的四种变化效果对应着Animation的四个子类：TranslateAnimation、ScaleAnimation、RotateAnimation和AlphaAnimation。
这四种动画既可以通过XML来定义，也可以通过代码来动态创建，对于View动画来说，建议采用XML来定义动画，因为XML格式的动画可读性更好。
```XML
<?xml version="1.0" encoding="utf-8"?>
<set xmlns:android="http://schemas.android.com/apk/res/android"
    android:duration="1000" // 动画的持续时间
    android:fillAfter="true"// 动画结束后是否停留在结束位置
    android:interpolator="" // 插值器
    android:shareInterpolator="true" // 表示集合中的动画是否和集合共享一个插值器>

    // 透明度动画
    <alpha
        android:fromAlpha="float"
        android:toAlpha="float"/>
    // 缩放动画
    <scale
        android:fromXScale="float"
        android:toXScale="float"
        android:fromYScale="float"
        android:toYScale="float"
        // 缩放的轴点的x坐标，会影响缩放的效果，默认情况是在中心
        android:pivotX="float"
        android:pivotY="float"/>
    // 平移动画
    <translate
        android:fromXDelta="float"
        android:toXDelta="float"
        android:fromYDelta="float"
        android:toYDelta="float"/>
    // 旋转动画
    <rotate
        android:fromDegrees="float"
        android:toDegrees="float"
        // 旋转的轴点的X左边
        android:pivotY="float"
        android:pivotX="float"/>

</set>
```
如何应用上面的代码呢?
```Java
TextView tv = (TextView) findViewById(R.id.tv);
Animation animation = AnimationUtils.loadAnimation(this, R.anim.view);
tv.startAnimation(animation);
```
当然这些动画也能直接通过TranslateAnimation、ScaleAnimation、RotateAnimation和AlphaAnimation类来创建

## View动画的特殊使用场景
除了上面说的View的四种动画效果以外，View动画还可以在一些特殊的场景下使用，比如在ViewGroup中可以控制子元素的出场效果，在Activity中可以实现不同的Activity之间的切换效果。
#### LayoutAnimation
LayoutAnimation作用于ViewGroup，为ViewGroup指定一个动画。这种效果我们常用在ListView上面
1.首先定义LayoutAnimation
```XML
// res/anim/anim/anim_layout
<?xml version="1.0" encoding="utf-8"?>
<layoutAnimation xmlns:android="http://schemas.android.com/apk/res/android"
    // 子元素开始动画的时间延迟
    android:delay="0.5"
    // 子元素动画的顺序
    android:animationOrder="normal"
    // 子元素的动画
    android:animation="@anim/anim_item" />
```
2.然后设置子元素的动画
3.为ViewGroup指定android:layoutAnimation属性`android:layoutAnimation="@anim/anim_layout"`
当然还可以直接通过代码来指定：
```Java
Animation animation = AnimationUtils.loadAnimation(this, R.anim.anim_item);
LayoutAnimationController controller = new LayoutAnimationController(animation);
controller.setDelay(0.5f);
controller.setOrder(LayoutAnimationController.ORDER_NORMAL);
listView.setLayoutAnimation(controller);
```

## Activity的切换效果
可以通过如下方式添加自定义的切换效果:
```Java
Intent intent = new Intent(MainActivity.this, SecondActivity.class);
startActivity(intent);
overridePendingTransition(R.anim.enter_anim, R.anim.exit_anim);
```
需要注意的是overridePendingTransition这个方法必须位置startActivity或者finish的后面调用，否则动画效果将不起作用。

Fragment也可以添加切换动画，但是Fragment是在API 11中新引入的类，因此为了兼容性我们需要使用support-v4兼容包，在这种情况下我们可以通过如下方式设置动画
```Java
FragmentTransaction fragmentTransaction = getSupportFragmentManager().beginTransaction();
fragmentTransaction.setCustomAnimations(android.R.anim.slide_in_left, android.R.anim.slide_out_right);
```
