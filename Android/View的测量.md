# MeasureSpec
系统显示一个View，首先需要通过测量(measure)该View来知晓其长和宽从而确定显示该View时需要多大的空间。在测量的过程中MeasureSpec贯穿全程，发挥着不可或缺的作用。
MeasureSpec在很大程度上决定了一个View的尺寸和规格，之所以说是很大程度上是因为这个过程还受父容器的影响，因为父容器影响View的MeasureSpec的创建过程。
MeasureSpec是一个32位的int值，其中高2位为测量的模式，低30位为测量的大小，在计算中使用位运算的原因是为了提高并优化效率。通过以下两个方法可以得到测量模式和测量大小：
```
int specMode = MeasureSpec.getMode(measureSpec);
int specSize = MeasureSpec.getSize(measureSpec);
```
当然也可以通过size和mode生成measureSpec
```
int measureSpec=MeasureSpec.makeMeasureSpec(size, mode);
```

测量模式有以下三种：
* EXACTLY
精确值模式，当我们将控件的layout_width属性或者layout_height属性指定为具体数值时，比如android:layout_width="100dp",或者指定为match_parent属性时，系统使用的是EXACTLY模式。

* AT_MOST
最大值模式，当控件的layout_width或者layout_height指定为wrap_content时，控件大小一般随着控件的子空间或者内容的变化而变化，此时控件的尺寸只要不超过父控件允许的最大尺寸即可。

* UNSPECIFIED
它不指定其大小测量模式，View想多大就多大，通常情况下在绘制自定义View的时候才会使用。这种情况一般用于系统内部，在此就不做过多讲解。


# MeasureSpec的形成
在ViewGroup中测量子View时会调用到measureChildWithMargins()方法，或者与之类似的方法。源码如下：
```java
/**
 * @param child
 * 子View
 * @param parentWidthMeasureSpec
 * 父容器(比如LinearLayout)的宽的MeasureSpec
 * @param widthUsed
 * 父容器(比如LinearLayout)在水平方向已经占用的空间大小
 * @param parentHeightMeasureSpec
 * 父容器(比如LinearLayout)的高的MeasureSpec
 * @param heightUsed
 * 父容器(比如LinearLayout)在垂直方向已经占用的空间大小
 */
protected void measureChildWithMargins(View child, int parentWidthMeasureSpec, int widthUsed,
                                       int parentHeightMeasureSpec, int heightUsed) {
    // 1.得到子View的LayoutParams                                
    final MarginLayoutParams lp = (ViewGroup.MarginLayoutParams) child.getLayoutParams();
    // 2.得到子View的宽的MeasureSpec
    final int childWidthMeasureSpec =
              getChildMeasureSpec(parentWidthMeasureSpec, mPaddingLeft + mPaddingRight +
                                  lp.leftMargin + lp.rightMargin + widthUsed, lp.width);
    // 3.得到子View的高的MeasureSpec                              
    final int childHeightMeasureSpec =
              getChildMeasureSpec(parentHeightMeasureSpec, mPaddingTop + mPaddingBottom +
                                  lp.topMargin + lp.bottomMargin + heightUsed, lp.height);
    // 4.测量子View                                
    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```
可以看到，这里传入了父View的一些参数。所以这里就可以说明，父View影响着子View的MeasureSpec生成
第一步，没啥好说的；第二步和第三步都调用到了getChildMeasureSpec( )，在该方法内部又做了哪些操作呢？
```java
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
    //获取父容器的specMode,specSize
   int specMode = View.MeasureSpec.getMode(spec);
   int specSize = View.MeasureSpec.getSize(spec);
   // 得到父容器在水平方向或垂直方向可用的最大空间值
   int size = Math.max(0, specSize - padding);

   int resultSize = 0;
   int resultMode = 0;
   // 确定子View的specMode和specSize
   switch (specMode) {
       case View.MeasureSpec.EXACTLY:
            // 当子View的宽或高为一个具体值，比如：100dp
           if (childDimension >= 0) {
               resultSize = childDimension;
               resultMode = View.MeasureSpec.EXACTLY;
           } else if (childDimension == LayoutParams.MATCH_PARENT) {
               // 当子View的宽或高为MATCH_PARENT
               resultSize = size;
               resultMode = View.MeasureSpec.EXACTLY;
           } else if (childDimension == LayoutParams.WRAP_CONTENT) {
               // 当子View的宽或高为WRAP_CONTENT
               resultSize = size;
               resultMode = View.MeasureSpec.AT_MOST;
           }
           break;

       case View.MeasureSpec.AT_MOST:
           if (childDimension >= 0) {
               resultSize = childDimension;
               resultMode = View.MeasureSpec.EXACTLY;
           } else if (childDimension == LayoutParams.MATCH_PARENT) {
               resultSize = size;
               resultMode = View.MeasureSpec.AT_MOST;
           } else if (childDimension == LayoutParams.WRAP_CONTENT) {
               resultSize = size;
               resultMode = View.MeasureSpec.AT_MOST;
           }
           break;

       case View.MeasureSpec.UNSPECIFIED:
           if (childDimension >= 0) {
               resultSize = childDimension;
               resultMode = View.MeasureSpec.EXACTLY;
           } else if (childDimension == LayoutParams.MATCH_PARENT) {
               resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
               resultMode = View.MeasureSpec.UNSPECIFIED;
           } else if (childDimension == LayoutParams.WRAP_CONTENT) {
               resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
               resultMode = View.MeasureSpec.UNSPECIFIED;
           }
           break;
   }
   return View.MeasureSpec.makeMeasureSpec(resultSize, resultMode);
}
```
该方法就是确定子View的MeasureSpec的具体实现。
![](http://img.blog.csdn.net/20160510112048981)

刚才我们讨论的是：measureChildWithMargins( )调用getChildMeasureSpec( )
除此以外还有一种常见的操作：
measureChild( )调用getChildMeasureSpec( )
那么，measureChildWithMargins( )和measureChild( )有什么相同点和区别呢？
1. measureChildWithMargins( )和measureChild( )均用来测量子View的大小
2. 两者在调用getChildMeasureSpec( )均要计算父View已占空间
3. 在measureChild( )计算父View所占空间为mPaddingLeft + mPaddingRight，即父容器左右两侧的padding值之和
4. measureChildWithMargins( )计算父View所占空间为mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin+ widthUsed。此处，除了父容器左右两侧的padding值之和还包括了子View左右的margin值之和( lp.leftMargin + lp.rightMargin)，因为这两部分也是不能用来摆放子View的应算作父View已经占用的空间。这点从方法名measureChildWithMargins也可以看出来它是考虑了子View的margin所占空间的。


# onMeasure()
```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {  
   setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),  
                        getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));  
}
```
onMeasure( )源码流程如下:
1. 在onMeasure调用setMeasuredDimension( )设置View的宽和高.
2. 在setMeasuredDimension()中调用getDefaultSize()获取View的宽和高.
3. 在getDefaultSize()方法中又会调用到getSuggestedMinimumWidth()或者getSuggestedMinimumHeight()获取到View宽和高的最小值.

先来看getSuggestedMinimumWidth():
```java
protected int getSuggestedMinimumWidth() {  
    // 如果没有背景，返回View本身的最小宽度。 该宽度可以在XML中设置，也可以调用View的setMinimumWidth()方法设置
    // 如果有背景，返回View本身最小宽度mMinWidth和View背景的最小宽度的最大值
    return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());  
}
```
接下来看看getDefaultSize( )的源码
```java
// 该方法用于获取View的宽或者高的大小。
//该方法的第一个输入参数size就是调用getSuggestedMinimumWidth()方法获得的View的宽或高的最小值。
public static int getDefaultSize(int size, int measureSpec) {
    int result = size;
    int specMode = MeasureSpec.getMode(measureSpec);
    int specSize = MeasureSpec.getSize(measureSpec);

    switch (specMode) {
        // 在此情况下该方法的返回值就是View的宽或者高最小值.
        case MeasureSpec.UNSPECIFIED:
            result = size;
            break;
        // 在此情况下getDefaultSize()的返回值就是该子View的measureSpec中的specSize。    
        case MeasureSpec.AT_MOST:
        case MeasureSpec.EXACTLY:
            result = specSize;
            break;
    }
    return result;
}
```
因为MeasureSpec.UNSPECIFIED这种情况可以不考虑，所以在measure阶段View的宽和高由其measureSpec中的specSize决定！！
结合刚才的图发现一个问题：在该图的最后一行，如果子View在XML布局文件中对于大小的设置采用wrap_content，那么不管父View的specMode是MeasureSpec.AT_MOST还是MeasureSpec.EXACTLY对于子View而言系统给它设置的specMode都是MeasureSpec.AT_MOST，并且其大小都是parentLeftSize即父View目前剩余的可用空间。这时wrap_content就失去了原本的意义，变成了match_parent一样了.
所以自定义View在重写onMeasure()的过程中应该手动处理View的宽或高为wrap_content的情况。
这也在《Android群英传》中有提到，“View类默认的onMeasure()方法只支持EXACTLY模式，所以如果在自定义控件的时候，想让自定义View支持wrap_content属性，那么就必须重写onMeasure()方法来指定wrap_content时的大小”。可以查看系统中的其他组件，它们大都是重写了onMeasure()方法的

最后通过调用setMeasuredDimension()方法，来设置View的宽和高的测量值。

# 如何重写onMeasure()
刚才有说到：“View类默认的onMeasure()方法只支持EXACTLY模式，所以如果在自定义控件的时候，想让自定义View支持wrap_content属性，那么就必须重写onMeasure()方法来指定wrap_content时的大小”，那么如何重写onMeasure()方法来指定wrap_content时的大小呢？
以下是一个简单的例子
```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    super.onMeasure(widthMeasureSpec , heightMeasureSpec);  
    int widthSpecMode = MeasureSpec.getMode(widthMeasureSpec);  
    int widthSpceSize = MeasureSpec.getSize(widthMeasureSpec);  
    int heightSpecMode = MeasureSpec.getMode(heightMeasureSpec);  
    int heightSpceSize = MeasureSpec.getSize(heightMeasureSpec);  
    // 此处涉及到的mWidth和mHeight均为一个默认值；应根据具体情况而设值。
    if(widthSpecMode==MeasureSpec.AT_MOST&&heightSpecMode==MeasureSpec.AT_MOST){          
        setMeasuredDimension(mWidth, mHeight);  
    }else if(widthSpecMode==MeasureSpec.AT_MOST){  
        setMeasuredDimension(mWidth, heightSpceSize);  
    }else if(heightSpecMode==MeasureSpec.AT_MOST){  
        setMeasuredDimension(widthSpceSize, mHeight);  
    }  
 }
```

# 相关链接
[自定义View系列教程02--onMeasure源码详尽分析](http://blog.csdn.net/lfdfhl/article/details/51347818)
