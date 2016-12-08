我们先直接来看layout()的源码吧：
```java
/*
* l, t, r, b分别表示子View相对于父View的左、上、右、下的坐标
* @param l Left position, relative to parent
* @param t Top position, relative to parent
* @param r Right position, relative to parent
* @param b Bottom position, relative to parent
*/
@SuppressWarnings({"unchecked"})
public void layout(int l, int t, int r, int b) {
 if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
     onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
     mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
 }

 int oldL = mLeft;
 int oldT = mTop;
 int oldB = mBottom;
 int oldR = mRight;

 // 确定该View在其父View中的位置是否发生了变化
 // 调用setFrame()方法，在该方法中把l，t， r， b分别与之前的mLeft，mTop，mRight，mBottom一一作比较，假若其中任意一个值发生了变化，那么就判定该View的位置发生了变化
 boolean changed = isLayoutModeOptical(mParent) ?
         setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);
 // 如果发生了变化，就调用onLayout()方法，所以我们看看onLayout()的源码吧o(︶︿︶)o
 if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
     onLayout(changed, l, t, r, b);
     mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;

     ListenerInfo li = mListenerInfo;
     if (li != null && li.mOnLayoutChangeListeners != null) {
         ArrayList<OnLayoutChangeListener> listenersCopy =
                 (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
         int numListeners = listenersCopy.size();
         for (int i = 0; i < numListeners; ++i) {
             listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
         }
     }
 }

 mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;
 mPrivateFlags3 |= PFLAG3_IS_LAID_OUT;
}
```
可以看出，如果View的位置发生了变化，就会调用onLayout()方法，我们就先看看View中的onLayout():
```java
/**
 * 在layout方法中调用该onLayout()用于指定子View的大小和位置。
 * Called from layout when this view should
 * assign a size and position to each of its children.
 *
 * Derived classes with children should override
 * this method and call layout on each of
 * their children.
 * @param changed This is a new size or position for this view
 * @param left Left position, relative to parent
 * @param top Top position, relative to parent
 * @param right Right position, relative to parent
 * @param bottom Bottom position, relative to parent
 */
protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
}
```
通过文档说明可以知道，在layout方法中调用该onLayout()用于指定子View的大小和位置。 但是谁才有子View呢？当然是ViewGroup了，所以我们应该看看ViewGroup中的源码了。
```java
protected abstract void onLayout(boolean changed,int l, int t, int r, int b);
```
结果发现ViewGroup的onLayout()竟然是一个抽象方法！这就意味着啥呢？
这就是说ViewGroup的子类都必须重写这个方法，实现自己的逻辑。比如：FrameLayou，LinearLayout，RelativeLayout等等布局都需要重写这个方法，在该方法内依据各自的布局规则确定子View的位置。

在此以LinearLayout为例，看看ViewGroup对于onLayout()方法的实现。
```java
@Override
protected void onLayout(boolean changed, int l, int t, int r, int b) {
    if (mOrientation == VERTICAL) {
        layoutVertical(l, t, r, b);
    } else {
        layoutHorizontal(l, t, r, b);
    }
}
```
在LinearLayout的onLayout()方法中分别处理了水平线性布局和垂直线性布局。在此，就选择layoutVertical()继续往下看。
```java
/**
 * Position the children during a layout pass if the orientation of this
 * LinearLayout is set to {@link #VERTICAL}.
 *
 * @see #getOrientation()
 * @see #setOrientation(int)
 * @see #onLayout(boolean, int, int, int, int)
 * @param left
 * @param top
 * @param right
 * @param bottom
 */
void layoutVertical(int left, int top, int right, int bottom) {
    final int paddingLeft = mPaddingLeft;

    int childTop;
    int childLeft;

    // Where right end of child should go
    final int width = right - left;
    int childRight = width - mPaddingRight;

    // Space available for child
    // 计算child可以使用的控件大小
    int childSpace = width - paddingLeft - mPaddingRight;

    // 获取子View的个数
    final int count = getVirtualChildCount();

    final int majorGravity = mGravity & Gravity.VERTICAL_GRAVITY_MASK;
    final int minorGravity = mGravity & Gravity.RELATIVE_HORIZONTAL_GRAVITY_MASK;
    // 计算childTop从而确定子View的开始布局位置
    switch (majorGravity) {
       case Gravity.BOTTOM:
           // mTotalLength contains the padding already
           childTop = mPaddingTop + bottom - top - mTotalLength;
           break;

           // mTotalLength contains the padding already
       case Gravity.CENTER_VERTICAL:
           childTop = mPaddingTop + (bottom - top - mTotalLength) / 2;
           break;

       case Gravity.TOP:
       default:
           childTop = mPaddingTop;
           break;
    }
    // 确定每个子View的位置
    for (int i = 0; i < count; i++) {
        final View child = getVirtualChildAt(i);
        if (child == null) {
            childTop += measureNullChild(i);
        } else if (child.getVisibility() != GONE) {
            // 这里获取到的childWidth和childHeight就是在measure阶段所确立的宽和高
            final int childWidth = child.getMeasuredWidth();
            final int childHeight = child.getMeasuredHeight();
            // 得到子View的LayoutParams
            final LinearLayout.LayoutParams lp =
                    (LinearLayout.LayoutParams) child.getLayoutParams();
            // 依据子View的LayoutParams确定子View的位置
            int gravity = lp.gravity;
            if (gravity < 0) {
                gravity = minorGravity;
            }
            final int layoutDirection = getLayoutDirection();
            final int absoluteGravity = Gravity.getAbsoluteGravity(gravity, layoutDirection);
            switch (absoluteGravity & Gravity.HORIZONTAL_GRAVITY_MASK) {
                case Gravity.CENTER_HORIZONTAL:
                    childLeft = paddingLeft + ((childSpace - childWidth) / 2)
                            + lp.leftMargin - lp.rightMargin;
                    break;

                case Gravity.RIGHT:
                    childLeft = childRight - childWidth - lp.rightMargin;
                    break;

                case Gravity.LEFT:
                default:
                    childLeft = paddingLeft + lp.leftMargin;
                    break;
            }

            if (hasDividerBeforeChildAt(i)) {
                childTop += mDividerHeight;
            }

            childTop += lp.topMargin;
            setChildFrame(child, childLeft, childTop + getLocationOffset(child),
                    childWidth, childHeight);
            childTop += childHeight + lp.bottomMargin + getNextLocationOffset(child);

            i += getChildrenSkipCount(child, i);
        }
    }
}
```
通过分析源码到这我们就可以理清楚思路了：

ViewGroup首先调用了layout()确定了自己本身在其父View中的位置，然后调用onLayout()确定每个子View的位置，每个子View又会调用View的layout()方法来确定自己在ViewGroup的位置。

**小结一下：**

>View的layout()方法用于View确定自己本身在其父View的位置
ViewGroup的onLayout()方法用于确定子View的位置

# 总结

1. 获取View的测量大小measuredWidth和measuredHeight的时机。
在某些复杂或者极端的情况下系统会多次执行measure过程，所以在onMeasure()中去获取View的测量大小得到的是一个不准确的值。为了避免该情况，最好在onMeasure()的下一阶段即onLayout()中去获取。
2. getMeasuredWidth()和getWidth()的区别
在绝大多数情况下这两者返回的值都是相同的，但是结果相同并不说明它们是同一个东西。
首先，它们的获取时机是不同的。
在measure()过程结束后就可以调用getMeasuredWidth()方法获取到View的测量大小，而getWidth()方法要在layout()过程结束后才能被调用从而获取View的实际大小。
其次，它们返回值的计算方式不同。
getMeasuredWidth()方法中的返回值是通过setMeasuredDimension()方法得到的，这点我们之前已经分析过，在此不再赘述；而getWidth()方法中的返回值是通过View的右坐标减去其左坐标(right-left)计算出来的。
3. 刚才说到了关于View的坐标，在这就不得不提一下：
view.getLeft()，view.getRight()，view.getBottom()，view.getTop();
这四个方法用于获取子View相对于父View的位置。
但是请注意:
getLeft( )表示子View的左边距离父View的左边的距离
getRight( )表示子View的右边距离父View的左边的距离
getTop( )表示子View的上边距离父View的上边的距离
getBottom( )表示子View的下边距离父View的上边的距离
4. 直接继承自ViewGroup可能带来的复杂处理。
在ViewGroup中包含了多个View，每个View都设置了padding和margin，除此之外还可能包含各种嵌套。在这种情况下，我们在onMeasure()和onLayout()中都要花费大量的精力来处理这些问题。所以在一般情况下，我们可以选择继承自LinearLayout，RelativeLayout等系统已有的布局从而简化这两部分的处理。

# 相关链接
[自定义View系列教程03--onLayout源码详尽分析](http://blog.csdn.net/lfdfhl/article/details/51393131)
