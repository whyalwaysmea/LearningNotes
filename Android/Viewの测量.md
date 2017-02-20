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

# Measure过程
Measure的过程要分情况来看，如果只是一个原始的View，那么通过measure方法就完成了其测量过程，如果是一个ViewGroup，除了完成自己的测量过程外，还会遍历去调用所有子元素的measure方法，各个子元素再递归去执行这个流程。


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
/**
 * @param spec 父view的详细测量值(MeasureSpec)
 * @param padding view当前尺寸的的内边距和外边距(padding,margin)
 * @param childDimension 子视图的布局参数（宽/高）
 */
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
    //获取父容器的specMode,specSize
   int specMode = MeasureSpec.getMode(spec);
   int specSize = MeasureSpec.getSize(spec);
   // 得到父容器在水平方向或垂直方向可用的最大空间值
   int size = Math.max(0, specSize - padding);

   //子view想要的实际大小和模式（需要计算）
   int resultSize = 0;
   int resultMode = 0;
   // 确定子View的specMode和specSize
   switch (specMode) {
       case View.MeasureSpec.EXACTLY:
            // 当子View的宽或高为一个具体值，比如：100dp
           if (childDimension >= 0) {
               // 实际大小就是想要的数值大小，
               resultSize = childDimension;
               // 模式就为EXACTLY
               resultMode = MeasureSpec.EXACTLY;
           } else if (childDimension == LayoutParams.MATCH_PARENT) {
               // 当子View的宽或高为MATCH_PARENT
               // 大小就是可用的最大空间
               resultSize = size;
               resultMode = MeasureSpec.EXACTLY;
           } else if (childDimension == LayoutParams.WRAP_CONTENT) {
               // 当子View的宽或高为WRAP_CONTENT
               resultSize = size;
               resultMode = MeasureSpec.AT_MOST;
           }
           break;

       case View.MeasureSpec.AT_MOST:
           if (childDimension >= 0) {
               resultSize = childDimension;
               resultMode = MeasureSpec.EXACTLY;
           } else if (childDimension == LayoutParams.MATCH_PARENT) {
               resultSize = size;
               resultMode = MeasureSpec.AT_MOST;
           } else if (childDimension == LayoutParams.WRAP_CONTENT) {
               resultSize = size;
               resultMode = MeasureSpec.AT_MOST;
           }
           break;

       case MeasureSpec.UNSPECIFIED:
            if (childDimension >= 0) {
                // Child wants a specific size... let him have it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size... find out how big it should
                // be
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size.... find out how
                // big it should be
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            }
            break;
        }
   }
   return View.MeasureSpec.makeMeasureSpec(resultSize, resultMode);
}
```
上述方法不难理解，它的主要作用是根据父容器的MeasureSpec同时结合View本身的LayoutParams来确定子元素的MeasureSpec.
该方法就是确定子View的MeasureSpec的具体实现。
![](http://upload-images.jianshu.io/upload_images/944365-6caa765e896a02a2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

刚才我们讨论的是：measureChildWithMargins( )调用getChildMeasureSpec( )
除此以外还有一种常见的操作：
measureChild( )调用getChildMeasureSpec( )
那么，measureChildWithMargins( )和measureChild( )有什么相同点和区别呢？

1. measureChildWithMargins( )和measureChild( )均用来测量子View的大小
2. 两者在调用getChildMeasureSpec( )均要计算父View已占空间
3. 在measureChild( )计算父View所占空间为mPaddingLeft + mPaddingRight，即父容器左右两侧的padding值之和
4. measureChildWithMargins( )计算父View所占空间为mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin+ widthUsed。此处，除了父容器左右两侧的padding值之和还包括了子View左右的margin值之和( lp.leftMargin + lp.rightMargin)，因为这两部分也是不能用来摆放子View的应算作父View已经占用的空间。这点从方法名measureChildWithMargins也可以看出来它是考虑了子View的margin所占空间的。

# measure
measure过程要分情况来看：
* 原始的View，那么通过measure方法就完成了其测量过程
* ViewGroup除了完成自己的测量过程外，还会遍历去调用所有字元素的measure方法，各个子元素再递归去执行这个流程

## View#onMeasure()
View的measure过程由其measure方法来完成，measure方法是一个final类型的方法，这意味着子类不能重写此方法，在View的measure方法中会去调用View的onMeasure方法，因此只需要看onMeasure的实现即可。
```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {  
   setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),  
                        getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));  
}
```
onMeasure( )源码流程如下:

1. 在onMeasure中调用setMeasuredDimension( )设置View的宽和高.
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

### 如何重写onMeasure()
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

## ViewGroup#measure

# Q&A
## 获取View的宽/高
当我们想在Activity已启动的时候做一件任务，但是这一件任务需要获取某个View的宽/高。但是因为View的measure过程和Activity生命周期方法不是同步执行的，所以无法保证Activity执行了onCreate(),onStart(),onResume()时某个View已经测量完毕了，如果View还没有测量完毕，那么获取到的宽/高就是0.
那么我们来看看到底有哪些方法来在Activity中获取宽/高

1.Activity/View#onWindowFocusChanged
这个方法的含义是：View已经初始化完毕了，宽/高已经准备好了，这个时候去获取宽/高是没有问题的。需要注意的是，onWindowFocusChanged会被调用多次，当Activity的窗口得到焦点和失去焦点的时候均会被调用一次。
```java
@Override
public void onWindowFocusChanged(boolean hasFocus) {
    super.onWindowFocusChanged(hasFocus);
    if(hasFocus) {
        int width = view.getMeasuredWidth();
        int height = view.getMeasuredHeight();
    }
}
```

2.view.post(runnable)
通过post可以将一个runnable投递到消息队列的尾部，然后等待Looper调用此runnable的时候，View已经初始化好了
```java
view.post(new Runnable() {
    @Override
    public void run() {
        int measuredWidth = view.getMeasuredWidth();
        int measuredHeight = view.getMeasuredHeight();
    }
});
```

3.ViewTreeObserver
```java
final ViewTreeObserver viewTreeObserver = view.getViewTreeObserver();
viewTreeObserver.addOnGlobalLayoutListener(new ViewTreeObserver.OnGlobalLayoutListener() {
    @Override
    public void onGlobalLayout() {
        viewTreeObserver.removeGlobalOnLayoutListener(this);
        int measuredWidth = view.getMeasuredWidth();
        int measuredHeight = view.getMeasuredHeight();
    }
});
```

4.view.measure(int widthMeasureSpec, int heightMeasureSpec);
通过手动对View进行measure来得到View的宽/高。

## ScrollView嵌套ListView问题解析
在ScrollView中嵌套ListView显示是不正常的，确切地说是只会显示ListView的第一个项。
那么为什么会造成这样的显示不正常呢？ 我们应该能够想到，是ListView在测量自己高度的时候出现了问题，但是我们知道子View的测量模式是由父View决定的，那么我们就先来查看一下ScrollView的源码
```java
// ScrollView.java
@Override
protected void measureChildWithMargins(View child, int parentWidthMeasureSpec, int widthUsed,
        int parentHeightMeasureSpec, int heightUsed) {
    final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

    final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
            mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                    + widthUsed, lp.width);
    final int usedTotal = mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin +
            heightUsed;
    // 在这里可以看出来，ScrollView将子View的测量模式指定为了MeasureSpec.UNSPECIFIED        
    final int childHeightMeasureSpec = MeasureSpec.makeSafeMeasureSpec(
            Math.max(0, MeasureSpec.getSize(parentHeightMeasureSpec) - usedTotal),
            MeasureSpec.UNSPECIFIED);

    child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
}
```
现在我们知道了ListView的测量模式了，所以就来看看ListView的测量方法吧
```java
@Override
// ListView.java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    // Sets up mListPadding
    super.onMeasure(widthMeasureSpec, heightMeasureSpec);

    final int widthMode = MeasureSpec.getMode(widthMeasureSpec);
    final int heightMode = MeasureSpec.getMode(heightMeasureSpec);
    int widthSize = MeasureSpec.getSize(widthMeasureSpec);
    int heightSize = MeasureSpec.getSize(heightMeasureSpec);

    int childWidth = 0;
    int childHeight = 0;
    int childState = 0;

    mItemCount = mAdapter == null ? 0 : mAdapter.getCount();
    if (mItemCount > 0 && (widthMode == MeasureSpec.UNSPECIFIED
            || heightMode == MeasureSpec.UNSPECIFIED)) {
        final View child = obtainView(0, mIsScrap);

        // Lay out child directly against the parent measure spec so that
        // we can obtain exected minimum width and height.
        measureScrapChild(child, 0, widthMeasureSpec, heightSize);

        childWidth = child.getMeasuredWidth();
        childHeight = child.getMeasuredHeight();
        childState = combineMeasuredStates(childState, child.getMeasuredState());

        if (recycleOnMeasure() && mRecycler.shouldRecycleViewType(
                ((LayoutParams) child.getLayoutParams()).viewType)) {
            mRecycler.addScrapView(child, 0);
        }
    }

    if (widthMode == MeasureSpec.UNSPECIFIED) {
        widthSize = mListPadding.left + mListPadding.right + childWidth +
                getVerticalScrollbarWidth();
    } else {
        widthSize |= (childState & MEASURED_STATE_MASK);
    }
    // 通过这句话可以看出来，当ListView高的测量模式为MeasureSpec.UNSPECIFIED时，
    // 高的值为padding + childHeight + getVerticalFadingEdgeLength() * 2
    // 所以就导致了ScrollView中的ListView显示出现了问题
    if (heightMode == MeasureSpec.UNSPECIFIED) {
        heightSize = mListPadding.top + mListPadding.bottom + childHeight +
                getVerticalFadingEdgeLength() * 2;
    }

    if (heightMode == MeasureSpec.AT_MOST) {
        // TODO: after first layout we should maybe start at the first visible position, not 0
        heightSize = measureHeightOfChildren(widthMeasureSpec, 0, NO_POSITION, heightSize, -1);
    }

    setMeasuredDimension(widthSize, heightSize);

    mWidthMeasureSpec = widthMeasureSpec;
}
```
那至于怎么解决这个问题就很明显了，我们只用重写一下ListView的onMeasure方法就好了
```java
@Override
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
    int heightSpec = MeasureSpec.makeMeasureSpec(Integer.MAX_VALUE >> 2, MeasureSpec.AT_MOST);
    super.onMeasure(widthMeasureSpec, heightSpec);
}
```

# 相关链接
[自定义View系列教程02--onMeasure源码详尽分析](http://blog.csdn.net/lfdfhl/article/details/51347818)

[自定义View Measure过程 - 最易懂的自定义View原理系列（2）](http://www.jianshu.com/p/1dab927b2f36#)
