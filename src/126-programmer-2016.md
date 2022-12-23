## Android 自定义控件：如何使 View 动起来？

文/郭莉莉（子墨）

Android 中的许多控件都有滑动功能，但是当原生控件满足不了需求时，就需要自定义控件。那么如何能让控件滑动起来呢？本文主要总结几种可以使控件滑动起来的方法。

其实让 View 动起来，要么就是 View 本身具备滑动功能，像 ListView 那样可以上下滑动；要么就是布局实现滑动功能，像 ScrollView 那样使子 View 滑动；要么就直接借助动画或者工具类实现 View 滑动，下面从这几方面给出 View 滑动的方法。

### View 本身实现移动

#### offsetLeftAndRight(offsetX) or offsetTopAndBottom(offsetY)

见名知意，下面先看一下源码，了解其实现原理：

```
public void offsetLeftAndRight(int offset) {
    if (offset != 0) {
        final boolean matrixIsIdentity = hasIdentityMatrix();
        if (matrixIsIdentity) {
            if (isHardwareAccelerated()) {
                invalidateViewProperty(false, false);
        } else {
            final ViewParent p = mParent;
            if (p != null && mAttachInfo != null) {
                final Rect r = mAttachInfo.mTmpInvalRect;
                int minLeft;
                int maxRight;
               if (offset < 0) {
              minLeft = mLeft + offset;
              maxRight = mRight;
           } else {
              minLeft = mLeft;
              maxRight = mRight + offset;
              }
          r.set(0, 0, maxRight - minLeft, mBottom - mTop);
                    p.invalidateChild(this, r);
                }
            }
        } else {
            invalidateViewProperty(false, false);
        }
       mLeft += offset;
       mRight += offset;
       mRenderNode.offsetLeftAndRight(offset);
       ......
}
```

代码1

判断 offset 是否为0，也就是说是否存在滑动距离。不为0的情况下，根据是否在矩阵中做过标记来操作。如果做过标记，没有开启硬件加速则开始计算坐标。先获取到父 View，如果父 View 不为空，在 offset<0时，计算出左侧的最小边距，在 offset>0时，计算出右侧的最大值。其实分析了这么多，主要的实现代码就那一句 mRenderNode.offsetLeftAndRight(offset)，由 native 实现的左右滑动，以上分析的部分主要计算 View 显示的区域。

最后总结一下，offsetLeftAndRight(int offset)就是通过 offset 值改变了 View 的 getLeft()和 getRight()实现了 View 的水平移动。

offsetTopAndBottom(int offset)方法实现原理与offsetLeftAndRight(int offset)相同，offsetTopAndBottom(int offset)通过 offset 值改变 View 的getTop()、getBottom()值，同样给出核心代码 mRenderNode.offsetTopAndBottom(offset)，这个方法也是有 native 实现。

大概了解了 offsetLeftAndRight(int offset)和 offsetTopAndBottom(int offset)原理之后，怎么用呢？view.offsetLeftAndRight(offsetX)，view.offsetTopAndBottom(offsetY)，如果实现自定义 view 中使用这两个方法，那就在 onTouchEvent 方法的 MOVE 中直接调用。

这两个方法在自定义控件中使用较多，让 View 滑动的场景也是可以的，直接调用，简单、方便，需要注意的是这两个方法会使得 View 划出屏幕，所以在使用这两个方法时，一定要注意对屏幕宽高的判断。

#### layout 方法

看到 layout 方法实现 View 的滑动，是不是通过改变布局实现动画呢？Talk is cheap, show me the code（Linus 语）！

```
public void layout(int l, int t, int r, int b) {
    if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {
        onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);
        mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
    }
    int oldL = mLeft;
    int oldT = mTop;
    int oldB = mBottom;
    int oldR = mRight;
    boolean changed = isLayoutModeOptical(mParent) ?
            setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);
    if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
        onLayout(changed, l, t, r, b);
        mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;
        ListenerInfo li = mListenerInfo;
        if (li != null && li.mOnLayoutChangeListeners != null) {
            ArrayList<onlayoutchangelistener> listenersCopy =
            (ArrayList<onlayoutchangelistener>)li.mOnLayoutChangeListeners.clone();
            int numListeners = listenersCopy.size();
            for (int i = 0; i < numListeners; ++i) {
                listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
            }
        }
    }
    mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;
    mPrivateFlags3 |= PFLAG3_IS_LAID_OUT;
}</onlayoutchangelistener></onlayoutchangelistener>
```

代码2

计算 mPrivateFlags3 和 PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT 的与运算，先来看一下 mPrivateFlags3 赋值的过程：

```
if (cacheIndex < 0 || sIgnoreMeasureCache) {
         // measure ourselves, this should set the measured dimension flag back
          onMeasure(widthMeasureSpec, heightMeasureSpec);
          mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
  } else {
          long value = mMeasureCache.valueAt(cacheIndex);
          // Casting a long to int drops the high 32 bits, no mask needed
           setMeasuredDimensionRaw((int) (value >> 32), (int) value);
           mPrivateFlags3 |= PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;
            }
```

代码3

以上代码摘自 measure 方法中，如果当前的 if 条件成立，就走 onMeasure 方法，给 mPrivateFlags3 赋值，跟 PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT 与运算为0。也就是说 layout 方法的第一个 if 不成立，不执行 onMeasure 方法。如果 measure 方法中的 if 条件不成立，那个 mPrivateFlags3 和 PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT 作与运算时就不为0，在 layout 方法中的第一个 if 成立，执行 onMeasure 方法。

如果左上右下的任何一个值发生改变，都会触发 onLayout(changed, l, t, r, b)方法，到这里应该明白 View 是如何移动的，通过 Layout 方法给的 l,t,r,b 改变 View 的位置。

layout(int l, int t, int r, int b)：

- 第一个参数为 View 左侧到父布局的距离；

- 第二个参数为 View 顶部到父布局的距离；

- 第三个参数为 View 右侧到父布局的距离；

- 第四个参数为 View 底端到父布局的距离。

layout 方法一般在自定义 View 时使用，通过 layout 方法调用 onLayout 方法改变 View 的位置，layout 方法使用起来更加直观、明了。

### 通过改变父布局实现 View 移动

#### scrollTo or scrollBy

先看一下 scrollTo 的源码：

```
public void scrollTo(int x, int y) {
    if (mScrollX != x || mScrollY != y) {
        int oldX = mScrollX;
        int oldY = mScrollY;
        mScrollX = x;
        mScrollY = y;
        invalidateParentCaches();
        onScrollChanged(mScrollX, mScrollY, oldX, oldY);
        if (!awakenScrollBars()) {
            postInvalidateOnAnimation();
        }
    }
}
```

代码4

判断当前的坐标是否是同一个坐标。如果不是，将当前坐标点赋值给旧的坐标点，把即将移动到的坐标点赋值给当前坐标点，通过 onScrollChanged(mScrollX, mScrollY, oldX, oldY)方法移动到坐标点(x,y)处。

```
public void scrollBy(int x, int y) {
    scrollTo(mScrollX + x, mScrollY + y);
}
```

代码5

scrollBy 方法简单粗暴，调用 scrollTo 方法，在当前的位置继续偏移(x , y)。

这里把它归类到通过改变父布局实现 View 移动是有原因的，如果在 View 中使用这个方法改变的是内容，而非 View 本身。

这两个方法一般在自定义 ViewGroup 的时候使用，通过这种方式改变子 View 的位置。当然也可以在自定义 View 中使用，在其中改变的是 View 的内容，而非 View 本身。所以使用这两个方法要先确定移动内容还是移动 View 本身的位置，这两个方法可应用于普通的 View，但是比较生硬，单纯地使 View 移动的话，就不如直接使用动画了。

#### LayoutParams

LayoutParams 通过改变局部参数里面的值 View 的位置保存布局参数。如果布局中有多个 View，那么它们的位置整体移动。

LinearLayout.LayoutParams params = (LinearLayout.LayoutParams) getLayoutParams();

params.leftMargin = getLeft() + offsetX;

params.topMargin = getTop() + offsetY;

setLayoutParams(params);

这个方法应用的场景不是很多，例如动态改变 ViewGroup 的布局，它通过改变 ViewGroup 的 params 属性改变布局。

### 借助 Android 提供的工具实现移动

#### 动画

说到借助工具实现 View 的移动，相信第一个出现在脑海中的就是动画，动画有好几种，包括属性动画、帧动画、补间动画等。这里只给出属性动画的实例，它能实现以上几种动画的所有效果。

直接在代码中写属性动画或者写入 XML 文件，这里给出一个 XML 文件的属性动画。

```
<!--?xml version="1.0" encoding="utf-8"?-->
<set xmlns:android="http://schemas.android.com/apk/res/android">
    <objectanimator android:duration="5000" android:propertyname="translationX" android:valuefrom="100dp" android:valueto="200dp">
    <objectanimator android:duration="5000" android:propertyname="translationY" android:valuefrom="100dp" android:valueto="200dp">
</objectanimator></objectanimator></set>
```

代码6

然后在代码中读取 XML 文件：

```
animator = AnimatorInflater.loadAnimator(MainActivity.this,R.animator.translation);
animator.setTarget(image);
animator.start();
```

代码7

动画的使用范围很广，合理使用动画增加 UI 体验的舒适度，不仅可以单纯地使 View 动起来，还可以在自定义 View 中实现很多极炫的效果，当然不合理地使用动画可能使得内存溢出，甚至界面卡顿。

#### Scroller

Android 中的 Scroller 类封装了滚动操作，记录滚动的位置，下面看一下 Scroller 的源码：

```
public Scroller(Context context, Interpolator interpolator, boolean flywheel) {
    mFinished = true;
    if (interpolator == null) {
        mInterpolator = new ViscousFluidInterpolator();
    } else {
        mInterpolator = interpolator;
    }
    mPpi = context.getResources().getDisplayMetrics().density * 160.0f;
    mDeceleration = computeDeceleration(ViewConfiguration.getScrollFriction());
    mFlywheel = flywheel;
    mPhysicalCoeff = computeDeceleration(0.84f); // look and feel tuning
}
```

代码8

interpolator 默认创建一个 ViscousFluidInterpolator，主要就是初始化参数。

```
public void startScroll(int startX, int startY, int dx, int dy) {
    startScroll(startX, startY, dx, dy, DEFAULT_DURATION);
public void startScroll(int startX, int startY, int dx, int dy, int duration) {
    mMode = SCROLL_MODE;
    mFinished = false;
    mDuration = duration;
    mStartTime = AnimationUtils.currentAnimationTimeMillis();
    mStartX = startX;
    mStartY = startY;
    mFinalX = startX + dx;
    mFinalY = startY + dy;
    mDeltaX = dx;
    mDeltaY = dy;
    mDurationReciprocal = 1.0f / (float) mDuration;
}
```

代码9

使用过 Scroller 的开发者都知道要调用这个方法，它主要起到记录参数的作用。记录下当前滑动模式、是否滑动结束、滑动时间、开始时间、开始滑动的坐标点、滑动结束的坐标点、滑动时的偏移量、插值器的值。看方法名字会造成一个错觉，View 要开始滑动了，其实这是不正确的，这个方法仅仅是记录而已，其他什么事都没做。

```
public boolean computeScrollOffset() {
    if (mFinished) {
        return false;
    }
    int timePassed = (int)(AnimationUtils.currentAnimationTimeMillis() - mStartTime);
    if (timePassed < mDuration) {
        switch (mMode) {
        case SCROLL_MODE:
            final float x = mInterpolator.getInterpolation(timePassed * mDurationReciprocal);
            mCurrX = mStartX + Math.round(x * mDeltaX);
            mCurrY = mStartY + Math.round(x * mDeltaY);
            break;
        case FLING_MODE:
        final float t = (float) timePassed / mDuration;
        final int index = (int) (NB_SAMPLES * t);
        float distanceCoef = 1.f;
        float velocityCoef = 0.f;
        if (index < NB_SAMPLES) {
            final float t_inf = (float) index / NB_SAMPLES;
            final float t_sup = (float) (index + 1) / NB_SAMPLES;
            final float d_inf = SPLINE_POSITION[index];
            final float d_sup = SPLINE_POSITION[index + 1];
            velocityCoef = (d_sup - d_inf) / (t_sup - t_inf);
            distanceCoef = d_inf + (t - t_inf) * velocityCoef;
        }
        mCurrVelocity = velocityCoef * mDistance / mDuration * 1000.0f;
        mCurrX = mStartX + Math.round(distanceCoef * (mFinalX - mStartX));
        // Pin to mMinX <= mCurrX <= mMaxX
        mCurrX = Math.min(mCurrX, mMaxX);
        mCurrX = Math.max(mCurrX, mMinX);
        mCurrY = mStartY + Math.round(distanceCoef * (mFinalY - mStartY));
        //Pin to mMinY <= mCurrY <= mMaxY
        mCurrY = Math.min(mCurrY, mMaxY);
        mCurrY = Math.max(mCurrY, mMinY);
        if (mCurrX == mFinalX && mCurrY == mFinalY) {
            mFinished = true;
        }
        break;
      }
  }
  else {
        mCurrX = mFinalX;
        mCurrY = mFinalY;
        mFinished = true;
       }
    return true;
}
```

代码10

Scroller 还有一个重要的方法就是 computeScrollOffset()，它的职责是计算当前的坐标点。Scroller 方法使用比较广泛，既可应用于自定义 View，又可以应用于自定义 ViewGroup，配合着 computeScroll()方法实现滑动。其实 Scroller 并不会使 View 动起来，它起到的作用就是记录和计算的作用，通过 invalidate()刷新界面调用 onDraw 方法，进而调用 computeScroll()方法完成实际的滑动。

#### ViewDragHelper

ViewDragHelper 封装了滚动操作，内部使用了 Scroller 滑动，所以使用 ViewDragHelper 也要实现 computeScroll()方法。这里不再给出实例，最好的实例就是 Android 的源码，笔者最近有看 DrawerLayout 源码，DrawerLayout 滑动部分就是使用 ViewDragHelper 实现的，想要了解更多关于 ViewDragHelper 的内容请看《DrawerLayout 源码分析》（http://blog.csdn.net/elinavampire/article/details/51935425）。

ViewDragHelper 比较多的应用在自定义 ViewGroup 中，其中比较重要的两点，一是 ViewDragHelper.callback 方法，这里面的方法比较多，可以按照需要重写。另一个就是要把事件拦截和事件处理留给 ViewDragHelper，否则写的这一堆代码，都没什么价值了。

### 总结

熟练掌握以上这几种方法以及应用场景，完美地使 View 动起来，在 onDraw 方法中画出 View，在 onMeasure 方法中对 wrap_content 做相应处理，完美的自定义 View 就出自你手了！在 onMeasure 方法中准确地去计算子 View 的宽高，onLayout 方法根据需要去布局子 View，自定义 ViewGroup 也就熟练掌握了。

当然，自定义 View 或者自定义 ViewGroup 写得越多越熟练，当不知道如何下手时，可以先按照前辈的好控件去造轮子，造几次慢慢就成创造者了。本文如果有不正确的地方，欢迎指正。
