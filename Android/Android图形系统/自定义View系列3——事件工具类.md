# 自定义View系列——工具类

## Scroller

### 概述

Scroller是一个帮助计算scroll偏移值的工具类。它的2个经典应用场景：

1. ViewPager拖拽松手后scroll回弹效果；

2. RecyclerView滑动松手后的fling效果。

注意：

1. Scroller并非只能对View的内容滚动，若作用与View的坐标之上可以实现对View的拖拽。看具体适用场景。

2. Scroller和属性动画一样，只计算当前时刻的scroll偏移值，真正摆放View的操作一般通过重写`View.computeScroll`去根据这一帧Scroller计算的scroll偏移值去自行实现。

### 使用

#### Scroll

```kotlin
class ScrollViewGroup @JvmOverloads constructor(
    context: Context, attrs: AttributeSet? = null, defStyleAttr: Int = 0
) : FrameLayout(context, attrs, defStyleAttr) {
    private var downX = 0f
    private var downY = 0f
    private var lastX = downX
    private var lastY = downY
    //1.初始化Scroller
    private var mScroller = Scroller(context)

    override fun onTouchEvent(e: MotionEvent): Boolean {
        when (e.action) {
            MotionEvent.ACTION_DOWN -> {
                downX = e.x
                downY = e.y
                lastX = downX
                lastY = downY
                return true
            }
            MotionEvent.ACTION_MOVE -> {
                val x = (e.x - lastX).toInt()
                val y = (e.y - lastY).toInt()
                //2.拖拽
                scrollBy(-x, -y)
                lastX = e.x
                lastY = e.y
            }
            MotionEvent.ACTION_CANCEL,
            MotionEvent.ACTION_UP -> {
                //3. 初始化scroll，设置起始值、偏移值、时间(默认为250ms)
                mScroller.startScroll(scrollX, scrollY, -scrollX, -scrollY)
                //4.启动scroll，类似动画start。触发View.computeScroll
                invalidate()
            }
        }
        return super.onTouchEvent(e)
    }
    
    override fun computeScroll() {
        //5.判断滚动是否结束，true表示还未结束
        if (mScroller.computeScrollOffset()) {
            //6.得到此时scroller计算出的scroll值，并调用scrollTo来滚动内容
            scrollTo(mScroller.currX, mScroller.currY)
            //7.记得刷新，虽然scrollTo自带刷新，但是有可能下一帧scroller计算的scroll值和上一次一样，导致提前中断滚动。
            invalidate()
        }
    }

}
```

#### Fling

```kotlin
class ScrollFlingViewGroup @JvmOverloads constructor(
    context: Context, attrs: AttributeSet? = null, defStyleAttr: Int = 0
) : FrameLayout(context, attrs, defStyleAttr) {
    private var downX = 0f
    private var downY = 0f
    private var lastX = downX
    private var lastY = downY
    //1.初始化Scroller
    private var mScroller = Scroller(context)
    private val mMinimumFlingVelocity =
        ViewConfiguration.get(context).scaledMinimumFlingVelocity  //最小触发fling速度
    private val mMaxFlingVelocity =
        ViewConfiguration.get(context).scaledMaximumFlingVelocity  //最大允许fling速度
    //2.速度检测
    private var mVelocityTracker: VelocityTracker? = null
    
    override fun onTouchEvent(e: MotionEvent): Boolean {
        //3.初始化速度检测
        if (mVelocityTracker == null) {
            mVelocityTracker = VelocityTracker.obtain()
        }
        //4.追踪事件
        mVelocityTracker?.addMovement(e)  
        when (e.action) {
            MotionEvent.ACTION_DOWN -> {
                downX = e.x
                downY = e.y
                lastX = downX
                lastY = downY
                //5.结束还在fling的动画
                if (!mScroller.isFinished) {
                    mScroller.abortAnimation()
                }
                return true
            }
            MotionEvent.ACTION_MOVE -> {
                val x = (e.x - lastX).toInt()
                val y = (e.y - lastY).toInt()
                //6.平移child
                ViewCompat.offsetLeftAndRight(getChildAt(0), x)
                ViewCompat.offsetTopAndBottom(getChildAt(0), y)
                lastX = e.x
                lastY = e.y
            }
            MotionEvent.ACTION_CANCEL,
            MotionEvent.ACTION_UP -> {
                //7.计算此刻滑动速率。参数1000表示单位为 像素/秒
                mVelocityTracker?.computeCurrentVelocity(1000, mMaxFlingVelocity.toFloat())
                //8.获取计算后得到的X、y轴滑动速率
                val initVelocityX = mVelocityTracker!!.xVelocity.toInt()
                val initVelocityY = mVelocityTracker!!.yVelocity.toInt()
                if (abs(initVelocityX) > mMinimumFlingVelocity || abs(initVelocityY) > mMinimumFlingVelocity) {
                    //9.初始化fling，设置一系列参数
                    mScroller.fling(
                        getChildAt(0).left,getChildAt(0).top,//初始坐标
                        initVelocityX,initVelocityY,//离手速度
                        0,width - getChildAt(0).width,//左右边界
                        0,height - getChildAt(0).height//上下边界
                    )
                }
                //10.启动fling，类似动画start。触发View.computeScroll
                invalidate()
                //11.回收VelocityTracker
                if (mVelocityTracker != null) {
                    mVelocityTracker!!.recycle()
                    mVelocityTracker = null
                }
            }
        }
        return super.onTouchEvent(e)
    }

    override fun computeScroll() {
         //12.判断滚动是否结束，true表示还未结束
        if (mScroller.computeScrollOffset()) {
            //13.得到Scroller计算的scroll值，平移child。
            ViewCompat.offsetLeftAndRight(getChildAt(0), mScroller.currX - getChildAt(0).left)
            ViewCompat.offsetTopAndBottom(getChildAt(0), mScroller.currY - getChildAt(0).top)
            //14.持续刷新
            invalidate()
        }
    }
}
```

### 源码

#### Scroll

scroll的源码极其简单，无非就用到两个函数`startScroll`和`computeScrollOffset`：

```java
//初始化scroll参数
public void startScroll(int startX, int startY, int dx, int dy, int duration) {
    mMode = SCROLL_MODE;//scroll模式
    mFinished = false;
    mDuration = duration;
    mStartTime = AnimationUtils.currentAnimationTimeMillis();//当前时刻
    mStartX = startX;
    mStartY = startY;
    mFinalX = startX + dx;
    mFinalY = startY + dy;
    mDeltaX = dx;
    mDeltaY = dy;
    mDurationReciprocal = 1.0f / (float) mDuration;//时长倒数
}
//计算scroll偏移值
 public boolean computeScrollOffset() {
        if (mFinished) {
            return false;
        }
		//1.当前时刻距离初始时刻过去的时间
        int timePassed = (int)(AnimationUtils.currentAnimationTimeMillis() - mStartTime);
    	
        if (timePassed < mDuration) {//还未结束
            switch (mMode) {
            case SCROLL_MODE:
                //2.根据时间完成度得到动画完成度
                final float x = mInterpolator.getInterpolation(timePassed * mDurationReciprocal);
                //3.计算出当前时刻scroll偏移值
                mCurrX = mStartX + Math.round(x * mDeltaX);
                mCurrY = mStartY + Math.round(x * mDeltaY);
                break;
            case FLING_MODE:
                //...
                break;
            }
        }else {
            mCurrX = mFinalX;
            mCurrY = mFinalY;
            mFinished = true;
        }
        return true;
    }
```

#### Fling

Fling的源码比较复杂，不过和Scroll是同样的原理。

### Scroller和OverScroller

OverScroller基本和Scroller一样，只不过在Scroller的基础上做了优化和“回弹”效果。OverScroller的fling效果比Scroller好。

------

## ViewDragHelper

### 概述

ViewDragHelper is a utility class for writing custom ViewGroups. It offers a number of useful operations and state tracking for allowing a user to drag and reposition views within their parent ViewGroup.

简单来说，该类就是作用与ViewGoup之上。封装了对ViewGroup中child的各种**Drag、边界拖拽、Scroll、Fling、状态跟踪**等操作，大大简化了开发过程。其中的Scroll、Fling是依靠OverScroller实现的。

### 使用

ViewDragHelper 基本的四个操作

1. Drag——对child的拖拽
2. Edge Drag——在Parent的边缘拖拽，类似DrawerLayout的侧边菜单呼出效果
3. Scroll——Scroller的scroll效果
4. Fling——Scroller的Fling效果

#### Drag

注意：当对**消耗事件**的`Child`进行拖拽时需要额外实现`getViewVerticalDragRange`、`getViewHorizontalDragRange`两个函数，具体为什么源码分析中会解释

```kotlin
//ViewGroup中
private lateinit var mDragHelper: ViewDragHelper

init {
        mDragHelper = ViewDragHelper.create(this, object : ViewDragHelper.Callback() {
            override fun tryCaptureView(child: View, pointerId: Int): Boolean {
                //1.true表示对该child进行捕获
                return when (child.id) {
                    R.id.canscroll,
                    R.id.view -> true
                    else -> false
                }
            }
		
            override fun clampViewPositionHorizontal(child: View, left: Int, dx: Int): Int {
                return left//2.该值是helper计算出child的left应当跟随手指滑动到的最新值
            }

            override fun clampViewPositionVertical(child: View, top: Int, dy: Int): Int {
                return top
            }
		   
            override fun getViewHorizontalDragRange(child: View): Int {
                return 1//3.返回值 > 0，则表示shouldInterceptTouchEvent返回true，开启事件拦截。防止child消费事件，导致processTouchEvent无法执行（注意getViewVerticalDragRange返回值也要>0）
            }

            override fun getViewVerticalDragRange(child: View): Int {
                return 1
            }
        })
    }

    override fun onInterceptTouchEvent(ev: MotionEvent): Boolean {
        return mDragHelper.shouldInterceptTouchEvent(ev)//4.拦截，只有在getViewXXXDragRange返回值 > 0时触发。
    }

    override fun onTouchEvent(event: MotionEvent): Boolean {
        mDragHelper.processTouchEvent(event)////5.处理事件
        return true//6.一定要返回true
    }
```

#### Edge Drage

```kotlin
private lateinit var mDragHelper: ViewDragHelper

init {
    mDragHelper = ViewDragHelper.create(this, object : ViewDragHelper.Callback() {
        override fun tryCaptureView(child: View, pointerId: Int): Boolean {
            return true
        }

        override fun clampViewPositionHorizontal(child: View, left: Int, dx: Int): Int {
            return left
        }

        override fun clampViewPositionVertical(child: View, top: Int, dy: Int): Int {
            return top
        }

        override fun onEdgeTouched(edgeFlags: Int, pointerId: Int) {
            //当手指触碰ViewGroup边界
        }

        override fun onEdgeLock(edgeFlags: Int): Boolean {
            //锁定指定边界——手指拖拽EDGE_LEFT，上下滑动会被禁止传递给onEdgeDragStarted
            return edgeFlags == ViewDragHelper.EDGE_LEFT
        }

        override fun onEdgeDragStarted(edgeFlags: Int, pointerId: Int) {
            //当手指拖拽边界时触发,此时可以捕获需要相应事件的child
            mDragHelper.captureChildView(this@ViewDragHelperEdgeView.getChildAt(0), pointerId)
        }
    })
    //指定允许跟踪的边界
    mDragHelper.setEdgeTrackingEnabled(ViewDragHelper.EDGE_ALL)
}


override fun onTouchEvent(event: MotionEvent): Boolean {
    mDragHelper.processTouchEvent(event)
    return true
}
```

#### Scroll

其中有两种实现方式：1. smoothSlideViewTo ；2. settleCapturedViewAt。两者的区别在于后者会在松手那一刻，以自身初速度进行滚动，且只允许在`onViewReleased`函数中调用！（后续源码会解释）。

```kotlin
private lateinit var mDragHelper: ViewDragHelper

init {
    mDragHelper = ViewDragHelper.create(this, object : ViewDragHelper.Callback() {
        override fun tryCaptureView(child: View, pointerId: Int): Boolean {
            return true
        }

        override fun clampViewPositionHorizontal(child: View, left: Int, dx: Int): Int {
            return left
        }

        override fun clampViewPositionVertical(child: View, top: Int, dy: Int): Int {
            return top
        }

        override fun onViewReleased(releasedChild: View, xvel: Float, yvel: Float) {
            if (releasedChild.id == R.id.scroll1) {
                //平滑滚动
                mDragHelper.smoothSlideViewTo(
                    releasedChild,
                    width - releasedChild.width,
                    height - releasedChild.height
                )
            } else {
                //以一定初速度滚动
                mDragHelper.settleCapturedViewAt(
                    width - releasedChild.width * 2,
                    height - releasedChild.height
                )
            }
            invalidate()//触发computeScroll
        }

    })
}

override fun onTouchEvent(event: MotionEvent): Boolean {
    mDragHelper.processTouchEvent(event)
    return true
}

override fun computeScroll() {
    if (mDragHelper.continueSettling(true)) {//判断是否滚动结束，注意该函数已经实现了释放滚动的效果哦
        invalidate()
    }
}
```

#### Fling

对child进行“投掷(fling)”：

注意：`flingCapturedView`和`settleCapturedViewAt`一样，必需在`onViewReleased`函数中调用！。

```kotlin
private lateinit var mDragHelper: ViewDragHelper

    init {
        mDragHelper = ViewDragHelper.create(this, object : ViewDragHelper.Callback() {
            override fun tryCaptureView(child: View, pointerId: Int): Boolean {
                return true
            }

            override fun clampViewPositionHorizontal(child: View, left: Int, dx: Int): Int {
                return left
            }

            override fun clampViewPositionVertical(child: View, top: Int, dy: Int): Int {
                return top
            }

            override fun onViewReleased(releasedChild: View, xvel: Float, yvel: Float) {
                //fling，参数为不允许滑出的边界
                mDragHelper.flingCapturedView(
                    0,
                    0,
                    width - releasedChild.width,
                    height - releasedChild.height
                )
                invalidate()//触发computeScroll
            }

        })
    }

    override fun onTouchEvent(event: MotionEvent): Boolean {
        mDragHelper.processTouchEvent(event)
        return true
    }

    override fun computeScroll() {
        if (mDragHelper.continueSettling(true)) {//判断是否滚动结束，注意该函数已经实现了释放滚动的效果哦
            invalidate()
        }
    }
```

------

### 源码

以下的源码分析基本都是从`ViewDragHelper.Callback`的回调函数开始分析。

#### 捕获

捕获对应的是`mCallback.tryCaptureView(toCapture, pointerId)`。

源码从`processTouchEvent`消费down事件开始：

```java
public void processTouchEvent(@NonNull MotionEvent ev) {
        switch (action) {
            case MotionEvent.ACTION_DOWN: {
                final float x = ev.getX();
                final float y = ev.getY();
                final int pointerId = ev.getPointerId(0);
                final View toCapture = findTopChildUnder((int) x, (int) y);//找到手指按下对应的最顶层child
                saveInitialMotion(x, y, pointerId);
                tryCaptureViewForDrag(toCapture, pointerId);//1.试图捕获并拖拽该child
            }
        }
}

 boolean tryCaptureViewForDrag(View toCapture, int pointerId) {
        if (toCapture != null && mCallback.tryCaptureView(toCapture, pointerId)) {//2.回调mCallback.tryCaptureView。该返回值是一个分歧点，会导致captureChildView函数是否执行
            mActivePointerId = pointerId;
            captureChildView(toCapture, pointerId);//3.执行到这里说明成功捕获child
            return true;
        }
        return false;
    }

 public void captureChildView(@NonNull View childView, int activePointerId) {
        mCapturedView = childView;//4.记录捕获的child
        mActivePointerId = activePointerId;
        mCallback.onViewCaptured(childView, activePointerId);//回调
        setDragState(STATE_DRAGGING);//5.将state设置成STATE_DRAGGING
    }
```

可见如果`mCallback.tryCaptureView`返回false，则会导致`state!=STATE_DRAGGING`,`mCapturedView=null`。这样会什么后果呢？。来看看move事件：

```java
public void processTouchEvent(@NonNull MotionEvent ev) {
        switch (action) {
           if (mDragState == STATE_DRAGGING) {//判断 mDragState==STATE_DRAGGING
                    final int index = ev.findPointerIndex(mActivePointerId);
                    final float x = ev.getX(index);
                    final float y = ev.getY(index);
                    final int idx = (int) (x - mLastMotionX[mActivePointerId]);
                    final int idy = (int) (y - mLastMotionY[mActivePointerId]);
               
                    dragTo(mCapturedView.getLeft() + idx, mCapturedView.getTop() + idy, idx, idy);//1.拖拽

                    saveLastMotion(ev);
                }else{
               //...
           }
        }
}
```

可见`mDragState！=STATE_DRAGGING`所以就无法响应dragTo方法了（该方法能拖拽child实现跟随移动）。

**小结**：要想使得child能够被拖拽，必须要能被Parent“捕获”到，即`mCallback.tryCaptureView`对应的点击的child要返回true

思考：child不能被拖拽的原因会有几个？

1. `mCallback.tryCaptureView`返回false导致`mDragState != STATE_DRAGGING`，上面分析过

2. child自己消费了事件，导致父View的`onTouchEvent`根本就没执行。因此就需要`shouldInterceptTouchEvent`来派上用场啦！

#### 开启Move拦截

开启Move拦截对应的是`mCallback.getViewHorizontalDragRange(child)|mCallback.getViewVerticalDragRange(child)`。

想让`shouldInterceptTouchEvent`起到拦截作用一定发生在move事件，Parent可以根据move的距离来判断是否拦截。因此找到`shouldInterceptTouchEvent` Move事件处理：

```java
public boolean shouldInterceptTouchEvent(@NonNull MotionEvent ev) {
        switch (action) {
              case MotionEvent.ACTION_MOVE: {
                final int pointerCount = ev.getPointerCount();
                for (int i = 0; i < pointerCount; i++) {
                    final int pointerId = ev.getPointerId(i);

                    final float x = ev.getX(i);
                    final float y = ev.getY(i);
                    final float dx = x - mInitialMotionX[pointerId];
                    final float dy = y - mInitialMotionY[pointerId];

                    final View toCapture = findTopChildUnder((int) x, (int) y);//找到顶层child
                    //1.注意checkTouchSlop方法决定pastSlop的值
                    final boolean pastSlop = toCapture != null && checkTouchSlop(toCapture, dx, dy);
                    if (pastSlop) {
                        final int oldLeft = toCapture.getLeft();
                        final int targetLeft = oldLeft + (int) dx;
                        final int newLeft = mCallback.clampViewPositionHorizontal(toCapture,
                                targetLeft, (int) dx);
                        final int oldTop = toCapture.getTop();
                        final int targetTop = oldTop + (int) dy;
                        final int newTop = mCallback.clampViewPositionVertical(toCapture, targetTop,
                                (int) dy);
                        final int hDragRange = mCallback.getViewHorizontalDragRange(toCapture);
                        final int vDragRange = mCallback.getViewVerticalDragRange(toCapture);
                        if ((hDragRange == 0 || (hDragRange > 0 && newLeft == oldLeft))
                                && (vDragRange == 0 || (vDragRange > 0 && newTop == oldTop))) {
                            break;
                        }
                    }
                    reportNewEdgeDrags(dx, dy, pointerId);
                    //3.由于child消费事件导致 mDragState！=STATE_DRAGGING
                    if (mDragState == STATE_DRAGGING) {
                        break;
                    }
				  //4.如果 pastSlop=true 则执行tryCaptureViewForDrag，尝试捕获child，捕获到child则mDragState=STATE_DRAGGING，child就可以被拖拽啦！
                    if (pastSlop && tryCaptureViewForDrag(toCapture, pointerId)) {
                        break;
                    }
                }
                saveLastMotion(ev);
                break;
            }
        }
     //5.这里需要注意，当mDragState=STATE_DRAGGING变为true时会拦截Move事件。
     return mDragState == STATE_DRAGGING;
}

//当滑动超过mTouchSlop返回true
private boolean checkTouchSlop(View child, float dx, float dy) {
   	    //2.checkTouchSlop的返回值依赖于getViewHorizontalDragRange|getViewHorizontalDragRange返回是否大于0
        final boolean checkHorizontal = mCallback.getViewHorizontalDragRange(child) > 0;
        final boolean checkVertical = mCallback.getViewVerticalDragRange(child) > 0;
        if (checkHorizontal && checkVertical) {
            return dx * dx + dy * dy > mTouchSlop * mTouchSlop;
        } else if (checkHorizontal) {
            return Math.abs(dx) > mTouchSlop;
        } else if (checkVertical) {
            return Math.abs(dy) > mTouchSlop;
        }
        return false;
    }
```

**小结**：看到这里就很简单了，`getViewHorizontalDragRange`|`getViewVerticalDragRange`决定了`shouldInterceptTouchEvent`是否开启拦截Move事件。即使Child能够消费事件也会被Parent捕获到并拦截处理。

#### 拖拽

拖拽对应的是`mCallback.clampViewPositionHorizontal(mCapturedView, left, dx)|mCallback.clampViewPositionVertical(mCapturedView, top, dy)`

进入`拖拽`状态时，child就需要依赖这两个函数来响应手指来移动了。

```java
public void processTouchEvent(@NonNull MotionEvent ev) {
        switch (action) {
           if (mDragState == STATE_DRAGGING) {//判断 mDragState
                    final int index = ev.findPointerIndex(mActivePointerId);
                    final float x = ev.getX(index);
                    final float y = ev.getY(index);
                    final int idx = (int) (x - mLastMotionX[mActivePointerId]);//和上次触摸点的偏移
                    final int idy = (int) (y - mLastMotionY[mActivePointerId]);
                    //1.拖拽，将计算出新的left|top、偏移值传入
                    dragTo(mCapturedView.getLeft() + idx, mCapturedView.getTop() + idy, idx, idy);
                    saveLastMotion(ev);
                }else{
               //...
           }
        }
}

 private void dragTo(int left, int top, int dx, int dy) {
        int clampedX = left;
        int clampedY = top;
        final int oldLeft = mCapturedView.getLeft();
        final int oldTop = mCapturedView.getTop();
        if (dx != 0) {
            //2.调用clampViewPositionHorizontal得到clampedX
            clampedX = mCallback.clampViewPositionHorizontal(mCapturedView, left, dx);
            //3.根据clampedX摆放child
            ViewCompat.offsetLeftAndRight(mCapturedView, clampedX - oldLeft);
        }
        if (dy != 0) {
            clampedY = mCallback.clampViewPositionVertical(mCapturedView, top, dy);
            ViewCompat.offsetTopAndBottom(mCapturedView, clampedY - oldTop);
        }

        if (dx != 0 || dy != 0) {
            final int clampedDx = clampedX - oldLeft;
            final int clampedDy = clampedY - oldTop;
            mCallback.onViewPositionChanged(mCapturedView, clampedX, clampedY,//onViewPositionChanged
                    clampedDx, clampedDy);
        }
    }
```

小结：这两个函数的返回值决定child被拖拽的位置，一般只需要返回helper已经计算出的值就可以了。

#### 释放

释放对应的是`onViewReleased(View releasedChild, float xvel, float yvel)`

很显然代码是从Up事件开始：

```java
public void processTouchEvent(@NonNull MotionEvent ev) {
    final int action = ev.getActionMasked();
    if (action == MotionEvent.ACTION_DOWN) {
        cancel();
    }
    if (mVelocityTracker == null) {
        mVelocityTracker = VelocityTracker.obtain();
    }
    mVelocityTracker.addMovement(ev);
    switch (action) {
        case MotionEvent.ACTION_UP: {
            if (mDragState == STATE_DRAGGING) {
                //1.release
                releaseViewForPointerUp();
            }
            cancel();
            break;
        }
        case MotionEvent.ACTION_CANCEL: {
            if (mDragState == STATE_DRAGGING) {
                //2.release 初速度为0
                dispatchViewReleased(0, 0);
            }
            cancel();
            break;
        }
    }
}

private void releaseViewForPointerUp() {
        mVelocityTracker.computeCurrentVelocity(1000, mMaxVelocity);
    	//3.计算手指离屏时刻初速度
        final float xvel = clampMag(
                mVelocityTracker.getXVelocity(mActivePointerId),
                mMinVelocity, mMaxVelocity);
        final float yvel = clampMag(
                mVelocityTracker.getYVelocity(mActivePointerId),
                mMinVelocity, mMaxVelocity);
        dispatchViewReleased(xvel, yvel);//传递vx、vy
    }

private void dispatchViewReleased(float xvel, float yvel) {
    	//4.注意这个变量，其他时刻都为false，只有调用onViewReleased前才会改为true。
        mReleaseInProgress = true;
    	//5.调用onViewReleased
        mCallback.onViewReleased(mCapturedView, xvel, yvel);//xvel，yvel为手指离屏的初速度
        mReleaseInProgress = false;
        if (mDragState == STATE_DRAGGING) {
            setDragState(STATE_IDLE);
        }
    }
```

**小结**：释放的过程很简单，仅仅把释放的逻辑传给`mCallback.onViewReleased(mCapturedView, xvel, yvel)`让我们自行处理。无非利用VelocityTracker帮助计算了一个松手时刻的初速度。

#### 平滑滚动(smoothSlideViewTo)

之前说过`settleCapturedViewAt`和`flingCapturedView`只能在`mCallback.onViewReleased(mCapturedView, xvel, yvel)`中调用，下面来分析为什么以及这两个函数的源码，顺便分析`smoothSlideViewTo`的源码：

```java
public boolean smoothSlideViewTo(@NonNull View child, int finalLeft, int finalTop) {
    mCapturedView = child;
    mActivePointerId = INVALID_POINTER;
    //1.掉用forceSettleCapturedViewAt，注意最后两个参数——x、y初速度为0,表示平滑滚动
    boolean continueSliding = forceSettleCapturedViewAt(finalLeft, finalTop, 0, 0);
    if (!continueSliding && mDragState == STATE_IDLE && mCapturedView != null) {
        mCapturedView = null;
    }
    return continueSliding;
}

    private boolean forceSettleCapturedViewAt(int finalLeft, int finalTop, int xvel, int yvel) {
        final int startLeft = mCapturedView.getLeft();
        final int startTop = mCapturedView.getTop();
        final int dx = finalLeft - startLeft;
        final int dy = finalTop - startTop;
        if (dx == 0 && dy == 0) {
            mScroller.abortAnimation();
            setDragState(STATE_IDLE);
            return false;
        }
        //2.计算出滚动时长，源码比较复杂
        final int duration = computeSettleDuration(mCapturedView, dx, dy, xvel, yvel);
        //3.调用OverScroller.startScroll
        mScroller.startScroll(startLeft, startTop, dx, dy, duration);
        //4.设置为STATE_SETTLING状态
        setDragState(STATE_SETTLING);
        return true;
    }
```

可以看到，这不就是封装了一下`Scroller.startScroll`的用法吗？因此再找到封装`Scroller.computeScrollOffset()`的代码：

```java
public boolean continueSettling(boolean deferCallbacks) {
    if (mDragState == STATE_SETTLING) {
        //1.Scroller.computeScrollOffset()
        boolean keepGoing = mScroller.computeScrollOffset();
        final int x = mScroller.getCurrX();
        final int y = mScroller.getCurrY();
        final int dx = x - mCapturedView.getLeft();
        final int dy = y - mCapturedView.getTop();
        //2.摆放CapturedView
        if (dx != 0) {
            ViewCompat.offsetLeftAndRight(mCapturedView, dx);
        }
        if (dy != 0) {
            ViewCompat.offsetTopAndBottom(mCapturedView, dy);
        }
        if (dx != 0 || dy != 0) {
            mCallback.onViewPositionChanged(mCapturedView, x, y, dx, dy);//onViewPositionChanged
        }
        //3.是否结束
        if (keepGoing && x == mScroller.getFinalX() && y == mScroller.getFinalY()) {
            mScroller.abortAnimation();
            keepGoing = false;
        }
        //4.根据是否结束来设置DragState状态
        if (!keepGoing) {
            if (deferCallbacks) {
                mParentView.post(mSetIdleRunnable);
            } else {
                setDragState(STATE_IDLE);
            }
        }
    }
    return mDragState == STATE_SETTLING;
}
```

可见Helper把`Scroller.computeScrollOffset()`后续我们拖拽child的逻辑也封装了。

**小结**：ViewDragHelper不过就是利用Scroller来封装并实现scroll效果。

#### 具有初速度的滚动(settleCapturedViewAt)

```java
public boolean settleCapturedViewAt(int finalLeft, int finalTop) {
      //1.在释放中说过这个变量只有在mCallback.onViewRelease调用前会改为true，因此settleCapturedViewAt函数只能在onViewRelease调用，否则抛异常
    if (!mReleaseInProgress) {
        throw new IllegalStateException("Cannot settleCapturedViewAt outside of a call to "
                + "Callback#onViewReleased");
    }
     //2.掉用forceSettleCapturedViewAt，注意最后两个参数——x、y初速度不为为0,表示以一定初速度滚动
    return forceSettleCapturedViewAt(finalLeft, finalTop,
            (int) mVelocityTracker.getXVelocity(mActivePointerId),
            (int) mVelocityTracker.getYVelocity(mActivePointerId));
}
```

**小结**：这就是为什么`settleCapturedViewAt`只能在`Callback.onViewReleased`中调用，且该函数与`smoothSlideViewTo`区别也只是是否以初速度scroll。

#### 投掷(flingCapturedView)

```java
public void flingCapturedView(int minLeft, int minTop, int maxLeft, int maxTop) {
    //1.在释放中说过这个变量只有在mCallback.onViewRelease调用前会改为true，因此flingCapturedView函数只能在onViewRelease调用，否则抛异常
    if (!mReleaseInProgress) {
        throw new IllegalStateException("Cannot flingCapturedView outside of a call to "
                + "Callback#onViewReleased");
    }
    mScroller.fling(mCapturedView.getLeft(), mCapturedView.getTop(),
            (int) mVelocityTracker.getXVelocity(mActivePointerId),
            (int) mVelocityTracker.getYVelocity(mActivePointerId),
            minLeft, maxLeft, minTop, maxTop);
    setDragState(STATE_SETTLING);
}
```

**小结**：这就是为什么`flingCapturedView`只能在`Callback.onViewReleased`中调用。该函数





