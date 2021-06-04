

# 嵌套滑动

 参考：[浅析NestedScrolling嵌套滑动机制之基础篇](https://juejin.cn/post/6844904184911773709#heading-5)

## NestedScrollView源码

### Scroll事件

主要分析Down事件及Move事件

```java
public boolean onTouchEvent(MotionEvent ev) {
    initVelocityTrackerIfNotExists();

    final int actionMasked = ev.getActionMasked();

    if (actionMasked == MotionEvent.ACTION_DOWN) {
        mNestedYOffset = 0;
    }

    MotionEvent vtev = MotionEvent.obtain(ev);
    vtev.offsetLocation(0, mNestedYOffset);

    switch (actionMasked) {
        case MotionEvent.ACTION_DOWN: {
            if (getChildCount() == 0) {
                return false;
            }
            // mScroller.isFinished=false 说明Scroller还在fling
            if ((mIsBeingDragged = !mScroller.isFinished())) {
                final ViewParent parent = getParent();
                if (parent != null) {
                    parent.requestDisallowInterceptTouchEvent(true);
                }
            }

            
            if (!mScroller.isFinished()) {
                abortAnimatedScroll();//停止fling
            }

            mLastMotionY = (int) ev.getY();// Remember where the motion event started
            mActivePointerId = ev.getPointerId(0);
            //1.startNestedScroll🚀
            startNestedScroll(ViewCompat.SCROLL_AXIS_VERTICAL, ViewCompat.TYPE_TOUCH);
            break;
        }
        case MotionEvent.ACTION_MOVE:
            final int activePointerIndex = ev.findPointerIndex(mActivePointerId);

            final int y = (int) ev.getY(activePointerIndex);
            //2.deltaY=手指滑动距离
            int deltaY = mLastMotionY - y;
            if (!mIsBeingDragged && Math.abs(deltaY) > mTouchSlop) {
                final ViewParent parent = getParent();
                if (parent != null) {
                    parent.requestDisallowInterceptTouchEvent(true);
                }
                mIsBeingDragged = true;
                if (deltaY > 0) {
                    deltaY -= mTouchSlop;
                } else {
                    deltaY += mTouchSlop;
                }
            }
            if (mIsBeingDragged) {
                //3.dispatchNestedPreScroll🚀
                if (dispatchNestedPreScroll(0, deltaY, mScrollConsumed, mScrollOffset,
                        ViewCompat.TYPE_TOUCH)) {
                    //4.减去parent消费的距离。如果parent全部消费了，则deltaY=0；如果parent消费了一些，则剩下的一些需要自己去消费⭐
                    deltaY -= mScrollConsumed[1];
                    //5.累加每次当前View在Window中的偏移值
                    mNestedYOffset += mScrollOffset[1];
                }

                
                mLastMotionY = y - mScrollOffset[1];

                final int oldY = getScrollY();
                final int range = getScrollRange();
                final int overscrollMode = getOverScrollMode();
                boolean canOverscroll = overscrollMode == View.OVER_SCROLL_ALWAYS
                        || (overscrollMode == View.OVER_SCROLL_IF_CONTENT_SCROLLS && range > 0);

                //6.轮到自己去消费剩余的deltaY，剩余的的deltaY就是第[4]步计算出的
                if (overScrollByCompat(0, deltaY, 0, getScrollY(), 0, range, 0,
                        0, true) && !hasNestedScrollingParent(ViewCompat.TYPE_TOUCH)) {
                    mVelocityTracker.clear();
                }
				
                //7.计算自己最终消费了、滑动了的scrolledDeltaY偏移量
                final int scrolledDeltaY = getScrollY() - oldY;
                //8.计算未消费的偏移量。例如划了一段距离deltaY，其中滑到一半的时候就已经滑到边界了，此时还有一半是没有消费的⭐
                final int unconsumedY = deltaY - scrolledDeltaY;

                mScrollConsumed[1] = 0;

                //9.和dispatchNestedPreScroll逻辑一样。将未消费的偏移量unconsumedY丢给parent去消费。
                dispatchNestedScroll(0, scrolledDeltaY, 0, unconsumedY, mScrollOffset,
                        ViewCompat.TYPE_TOUCH, mScrollConsumed);

                mLastMotionY -= mScrollOffset[1];
                mNestedYOffset += mScrollOffset[1];
            }
            break;
    }

    if (mVelocityTracker != null) {
        mVelocityTracker.addMovement(vtev);
    }
    vtev.recycle();

    return true;
}
```

*startNestedScroll🚀*

```java
public boolean startNestedScroll(@ScrollAxis int axes, @NestedScrollType int type) {
    if (hasNestedScrollingParent(type)) {
        // Already in progress
        return true;
    }
    if (isNestedScrollingEnabled()) {
        ViewParent p = mView.getParent();
        View child = mView;
        while (p != null) {
            //1.如果 parent.onStartNestedScroll 返回true⭐
            if (ViewParentCompat.onStartNestedScroll(p, child, mView, axes, type)) {
                //2.
                setNestedScrollingParentForType(type, p);
                //4.调用parent.onNestedScrollAccepted⭐
                ViewParentCompat.onNestedScrollAccepted(p, child, mView, axes, type);
                return true;
            }
            if (p instanceof View) {
                child = (View) p;
            }
            p = p.getParent();
        }
    }
    return false;
}

private void setNestedScrollingParentForType(@NestedScrollType int type, ViewParent p) {
        switch (type) {
            case TYPE_TOUCH:
                //2.记录嵌套滑动的parent
                mNestedScrollingParentTouch = p;
                break;
            case TYPE_NON_TOUCH:
                mNestedScrollingParentNonTouch = p;
                break;
        }
    }
```

*dispatchNestedPreScroll🚀*

```java
public boolean dispatchNestedPreScroll(int dx, int dy, @Nullable int[] consumed,
        @Nullable int[] offsetInWindow, @NestedScrollType int type) {
    if (isNestedScrollingEnabled()) {
        final ViewParent parent = getNestedScrollingParentForType(type);//获取嵌套滑动的parent
        if (parent == null) {
            return false;
        }

        if (dx != 0 || dy != 0) {
            int startX = 0;
            int startY = 0;
            //1.记录mView当前在Window的顶点坐标
            if (offsetInWindow != null) {
                mView.getLocationInWindow(offsetInWindow);
                startX = offsetInWindow[0];
                startY = offsetInWindow[1];
            }
		   //2.初始化一个空的consumed数组
            if (consumed == null) {
                consumed = getTempNestedScrollConsumed();
            }
            consumed[0] = 0;
            consumed[1] = 0;
            //3.调用parent.onNestedPreScroll⭐
            ViewParentCompat.onNestedPreScroll(parent, mView, dx, dy, consumed, type);

            //4.重新计算mView顶点位置，将初始值替换和上一次顶点位置的偏移值。如果得到的偏移值不等于0，说明当前View位置发生变动了。这就说明调用parent.onNestedPreScroll，parent发生了scroll！
            if (offsetInWindow != null) {
                mView.getLocationInWindow(offsetInWindow);
                offsetInWindow[0] -= startX;
                offsetInWindow[1] -= startY;
            }
            //5.如果调用parent.onNestedPreScroll之后consumed有值则返回true，代表parent消费了滑动
            return consumed[0] != 0 || consumed[1] != 0;
        } else if (offsetInWindow != null) {
            offsetInWindow[0] = 0;
            offsetInWindow[1] = 0;
        }
    }
    return false;
}
```

#### 小结

Down事件：Child先询问Parent是否要处理嵌套滑动，如果存在则记录该Parent

<img src="pic\image-20210526212425892.png" alt="image-20210526212425892" style="zoom:80%;" />

Move事件：Move产生的Delta值会先传给Parent消费，之后Child再捡Parent剩下为消费的值消费，如果Child还是没消费完，则继续丢给Parent消费。这中间会传递Parent两次。

<img src="pic\image-20210526213057975.png" alt="image-20210526213057975" style="zoom:80%;" />

### Fling事件

```java
  case MotionEvent.ACTION_UP:
                final VelocityTracker velocityTracker = mVelocityTracker;
                velocityTracker.computeCurrentVelocity(1000, mMaximumVelocity);
                int initialVelocity = (int) velocityTracker.getYVelocity(mActivePointerId);
                if ((Math.abs(initialVelocity) >= mMinimumVelocity)) {
                    //1.dispatchNestedPreFling🚀（false）
                    if (!dispatchNestedPreFling(0, -initialVelocity)) {
                        //2.dispatchNestedFling🚀（未处理）
                        dispatchNestedFling(0, -initialVelocity, true);
                        //3.fling🚀
                        fling(-initialVelocity);
                    }
                } else if (mScroller.springBack(getScrollX(), getScrollY(), 0, 0, 0,
                        getScrollRange())) {
                    ViewCompat.postInvalidateOnAnimation(this);
                }
                mActivePointerId = INVALID_POINTER;
                endDrag();
                break;
```

`dispatchNestedPreFling🚀`

```java
public boolean dispatchNestedPreFling(float velocityX, float velocityY) {
    if (isNestedScrollingEnabled()) {
        ViewParent parent = getNestedScrollingParentForType(TYPE_TOUCH);
        if (parent != null) {
            return ViewParentCompat.onNestedPreFling(parent, mView, velocityX,
                    velocityY);
        }
    }
    return false;
}

//Parent(NestedScrollView)也是递归调用其parent.dispatchNestedPreFling,最终会返回false
public boolean onNestedPreFling(@NonNull View target, float velocityX, float velocityY) {
        return dispatchNestedPreFling(velocityX, velocityY);
    }
```

`dispatchNestedFling🚀`

```java
public boolean dispatchNestedFling(float velocityX, float velocityY, boolean consumed) {
        if (isNestedScrollingEnabled()) {
            ViewParent parent = getNestedScrollingParentForType(TYPE_TOUCH);
            if (parent != null) {
                return ViewParentCompat.onNestedFling(parent, mView, velocityX,
                        velocityY, consumed);
            }
        }
        return false;
    }

  public boolean onNestedFling(
            @NonNull View target, float velocityX, float velocityY, boolean consumed) {
        if (!consumed) {//consumed=true
            dispatchNestedFling(0, velocityY, true);
            fling((int) velocityY);
            return true;
        }
        return false;
    }
```

fling🚀

```java
public void fling(int velocityY) {
        if (getChildCount() > 0) {
            //调用Scroller去Fling
            mScroller.fling(getScrollX(), getScrollY(), 
                    0, velocityY, 
                    0, 0, 
                    Integer.MIN_VALUE, Integer.MAX_VALUE, 
                    0, 0); 
            runAnimatedScroll(true);
        }
    }
```

在Up事件中和处理Move有些相似，也是秉着Parent优先的原则，将速度丢给Parent，让它先Fling。只不过在NestedScrollView充当的Parent角色中并未做出处理。`dispatchNestedPreFling`和`dispatchNestedFling`这两个接口并未有实际作用。

<img src="pic\image-20210526230217276.png" alt="image-20210526230217276" style="zoom:80%;" />



#### Fling的传递

通过不断的从每一帧拿到Scroller计算的scroll值去偏移当前View：

```java
public void fling(int velocityY) {
        if (getChildCount() > 0) {
            mScroller.fling(getScrollX(), getScrollY(), //调用Scroller去Fling
                    0, velocityY, 
                    0, 0, 
                    Integer.MIN_VALUE, Integer.MAX_VALUE, 
                    0, 0); 
            runAnimatedScroll(true);
        }
    }


 private void runAnimatedScroll(boolean participateInNestedScrolling) {
        if (participateInNestedScrolling) {//true
            //1.记录消费TYPE_NON_TOUCH类型的Parent
            startNestedScroll(ViewCompat.SCROLL_AXIS_VERTICAL, ViewCompat.TYPE_NON_TOUCH);
        } else {
            stopNestedScroll(ViewCompat.TYPE_NON_TOUCH);
        }
        //2.记录当前scroll值
        mLastScrollerY = getScrollY();
        //3.触发computeScroll()
        ViewCompat.postInvalidateOnAnimation(this);
    }


public void computeScroll() {
    if (mScroller.isFinished()) {
        return;
    }
    mScroller.computeScrollOffset();//计算当前时刻scroll值
    final int y = mScroller.getCurrY();
    int unconsumed = y - mLastScrollerY;//计算偏移值
    mLastScrollerY = y;
    mScrollConsumed[1] = 0;//初始化mScrollConsumed
    
    //1.和Move中的逻辑一样，先丢给Parent消费，注意此时的类型为TYPE_NON_TOUCH，表示Fling
    dispatchNestedPreScroll(0, unconsumed, mScrollConsumed, null,
            ViewCompat.TYPE_NON_TOUCH);
    
    //2.减掉Parent消费的scroll值
    unconsumed -= mScrollConsumed[1];
    final int range = getScrollRange();
    if (unconsumed != 0) {
        final int oldScrollY = getScrollY();
        
        //3.轮到自己消费
        overScrollByCompat(0, unconsumed, getScrollX(), oldScrollY, 0, range, 0, 0, false);
        final int scrolledByMe = getScrollY() - oldScrollY;
        
        //4.计算自己也没消费完的scroll
        unconsumed -= scrolledByMe;
        mScrollConsumed[1] = 0;//再次置空
        
        //5.又丢给Parent去消费
        dispatchNestedScroll(0, scrolledByMe, 0, unconsumed, mScrollOffset,
                ViewCompat.TYPE_NON_TOUCH, mScrollConsumed);
        unconsumed -= mScrollConsumed[1];
    }

    //6.最终剩余的scroll表现为边缘阴影
    if (unconsumed != 0) {
        final int mode = getOverScrollMode();
        final boolean canOverscroll = mode == OVER_SCROLL_ALWAYS
                || (mode == OVER_SCROLL_IF_CONTENT_SCROLLS && range > 0);
        if (canOverscroll) {
            ensureGlows();
            if (unconsumed < 0) {
                if (mEdgeGlowTop.isFinished()) {
                    mEdgeGlowTop.onAbsorb((int) mScroller.getCurrVelocity());
                }
            } else {
                if (mEdgeGlowBottom.isFinished()) {
                    mEdgeGlowBottom.onAbsorb((int) mScroller.getCurrVelocity());
                }
            }
        }
        abortAnimatedScroll();
    }

    if (!mScroller.isFinished()) {
        ViewCompat.postInvalidateOnAnimation(this);//再次触发computeScroll
    } else {
        stopNestedScroll(ViewCompat.TYPE_NON_TOUCH);
    }
}
```

#### 小结

Fling的处理和Scroll的逻辑一摸一样，并且调用的dispatch接口也是一样。

