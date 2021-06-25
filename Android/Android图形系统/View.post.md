参考：

[Android 之 ViewTreeObserver 全面解析](https://www.jianshu.com/p/59d636695d42)

# ViewTreeObserver源码

## 概述

> A view tree observer is used to register listeners that can be notified of global changes in the view tree. Such global events include, but are not limited to，layout of the whole tree, beginning of the drawing pass, touch mode change....
>
> ViewTreeObserver用于侦听全局View Tree的事件，如焦点、布局、绘制

## 使用

```kotlin
class UIGlobalLayoutActivity : BaseActivity() {
    private lateinit var listener: ViewTreeObserver.OnGlobalLayoutListener
    init {
        listener= ViewTreeObserver.OnGlobalLayoutListener{
            //2.回调
            CmdUtil.v("中心TextView宽高：${text.width}|${text.height}")
            //3.移除（即只触发一次）
            window.decorView.viewTreeObserver.removeOnGlobalLayoutListener(listener)
        }
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        //1.注册GlobalLayoutListener
        window.decorView.viewTreeObserver.addOnGlobalLayoutListener (listener)
    }
}
```

## 源码

### 注册监听器

```java
// View.java
public ViewTreeObserver getViewTreeObserver() {
    //1.mAttachInfo指向ViewRootImpl的AttachInfo。由于Activity resume后才初始化ViewRootImpl，而mAttachInfo的初始化就在创建ViewRootImpl过程中，所以该方法的调用时机决定该值是否为空
    if (mAttachInfo != null) {
        return mAttachInfo.mTreeObserver;
    }
    //2.当mAttachInfo为空时，获取的是View的成员变量mFloatingTreeObserver
    if (mFloatingTreeObserver == null) {
        mFloatingTreeObserver = new ViewTreeObserver(mContext);
    }
    return mFloatingTreeObserver;
}
```

```java
// ViewTreeObserver.java
public void addOnWindowAttachListener(OnWindowAttachListener listener) {
    checkIsAlive();

    if (mOnWindowAttachListeners == null) {
        mOnWindowAttachListeners
                = new CopyOnWriteArrayList<OnWindowAttachListener>();
    }
	//3.将listener添加到ViewTreeObserver中的CopyOnWriteArrayList集合中（此时的ViewTreeObserver不确定是AttachInfo中还是View中的）
    mOnWindowAttachListeners.add(listener);
}
```

### 合并所有监听器

ViewRootImpl开始执行View的绘制流程时：

```cpp
// ViewRootImpl.java
private void performTranversals(){
     if (mFirst) {
         //...
            //1. host表示Decor，参数0表示View.VISIABLE
            host.dispatchAttachedToWindow(mAttachInfo, 0);
         
           //会回调OnWindowAttachListener
            mAttachInfo.mTreeObserver.dispatchOnWindowAttachedChange(true);
        }
}
```

```java
// ViewGroup.java
void dispatchAttachedToWindow(AttachInfo info, int visibility) {
        mGroupFlags |= FLAG_PREVENT_DISPATCH_ATTACHED_TO_WINDOW;
    	//1.调用View.dispatchAttachedToWindow
        super.dispatchAttachedToWindow(info, visibility);
        mGroupFlags &= ~FLAG_PREVENT_DISPATCH_ATTACHED_TO_WINDOW;

        final int count = mChildrenCount;
        final View[] children = mChildren;
        for (int i = 0; i < count; i++) {
            final View child = children[i];
            //5.遍历Children，所有child都进行合并
            child.dispatchAttachedToWindow(info,
                    combineVisibility(visibility, child.getVisibility()));
        }
  }

// View.java
void dispatchAttachedToWindow(AttachInfo info, int visibility) {
    //2.View保存ViewRootImpl的mAttachInfo
    mAttachInfo = info;
    if (mFloatingTreeObserver != null) {
        //3.AttachInfo合并当前View的ViewTreeObserver
        info.mTreeObserver.merge(mFloatingTreeObserver);
        mFloatingTreeObserver = null;
    }

    if (mRunQueue != null) {
        mRunQueue.executeActions(info.mHandler);
        mRunQueue = null;
    }
   
    onAttachedToWindow();

    ListenerInfo li = mListenerInfo;
    final CopyOnWriteArrayList<OnAttachStateChangeListener> listeners =
            li != null ? li.mOnAttachStateChangeListeners : null;
    if (listeners != null && listeners.size() > 0) {
        for (OnAttachStateChangeListener listener : listeners) {
            listener.onViewAttachedToWindow(this);
        }
    }

    int vis = info.mWindowVisibility;
    if (vis != GONE) {
        onWindowVisibilityChanged(vis);
        if (isShown()) {
            onVisibilityAggregated(vis == VISIBLE);
        }
    }
    
    onVisibilityChanged(this, visibility);
}

// ViewTreeObserver.java
void merge(ViewTreeObserver observer) {
    if (observer.mOnWindowAttachListeners != null) {
        if (mOnWindowAttachListeners != null) {
            //4.⭐最终将当前View中ViewTreeObserver的Listener添加至mAttachInfo中的ViewTreeObserver
            mOnWindowAttachListeners.addAll(observer.mOnWindowAttachListeners);
        } else {
            mOnWindowAttachListeners = observer.mOnWindowAttachListeners;
        }
    }

    if (observer.mOnWindowFocusListeners != null) {
        if (mOnWindowFocusListeners != null) {
            mOnWindowFocusListeners.addAll(observer.mOnWindowFocusListeners);
        } else {
            mOnWindowFocusListeners = observer.mOnWindowFocusListeners;
        }
    }
    //...
}
```

## 各监听器调用时机

### OnWindowAttachListener

调用时机：

attach—— performTraversal方法中，在View Tree三大绘制流程前调用 ；

detached—— Activity 销毁时，ViewRootImpl.doDie()中调用；

```java

public interface OnWindowAttachListener {
    /**
     * Callback method to be invoked when the view hierarchy is attached to a window
 
     */
    public void onWindowAttached();

    /**
     * Callback method to be invoked when the view hierarchy is detached from a window

     */
    public void onWindowDetached();
}
```

### OnWindowFocusChangeListener

调用时机：Window焦点改变。在ViewRootImpl中的W内部类（W作为一个WMS的客户端保存在ViewRootImpl中）中触发。

```java
public interface OnWindowFocusChangeListener {
    /**
     * Callback method to be invoked when the window focus changes in the view tree.
     *
     * @param hasFocus Set to true if the window is gaining focus, false if it is
     * losing focus.
     */
    public void onWindowFocusChanged(boolean hasFocus);
}
```

### OnGlobalFocusChangeListener

调用时机：View Tree中的View焦点发生改变（注意参数可能为空）

```java
/**
 * Interface definition for a callback to be invoked when the focus state within
 * the view tree changes.
 */
public interface OnGlobalFocusChangeListener {
    /**
     * Callback method to be invoked when the focus changes in the view tree. When
     * the view tree transitions from touch mode to non-touch mode, oldFocus is null.
     * When the view tree transitions from non-touch mode to touch mode, newFocus is
     * null. When focus changes in non-touch mode (without transition from or to
     * touch mode) either oldFocus or newFocus can be null.
     *
     * @param oldFocus The previously focused view, if any.
     * @param newFocus The newly focused View, if any.
     */
    public void onGlobalFocusChanged(View oldFocus, View newFocus);
}
```

### OnGlobalLayoutListener

调用时机： view tree 测量、布局之后，绘制之前。

 当全局处于布局阶段或者view tree可见性改变时触发回调（此时可以获取View的宽高，注意该方法会调用多次，不一定每次的宽高都相等）

```java
/**
 * Interface definition for a callback to be invoked when the global layout state
 * or the visibility of views within the view tree changes.
 */
public interface OnGlobalLayoutListener {
 
    public void onGlobalLayout();
}
```

### OnPreDrawListener

调用时机：view tree 测量、布局之后，绘制之前。

注意返回值为true表示执行绘制，否则重新测量、布局再回调判断。

```java
public interface OnPreDrawListener {
    /**
    Return true to proceed with the current drawing pass, or false to cancel.
    */
    public boolean onPreDraw();
}
```

**源码**

```java
public final boolean dispatchOnPreDraw() {
        boolean cancelDraw = false;
        final CopyOnWriteArray<OnPreDrawListener> listeners = mOnPreDrawListeners;
        if (listeners != null && listeners.size() > 0) {
            CopyOnWriteArray.Access<OnPreDrawListener> access = listeners.start();
            try {
                int count = access.size();
                for (int i = 0; i < count; i++) {
                    //取反
                    cancelDraw |= !(access.get(i).onPreDraw());
                }
            } finally {
                listeners.end();
            }
        }
        return cancelDraw;
    }
```

```java
private void performTraversals(){
    
    //根据onPreDraw的返回值来决定执行绘制还是跳过绘制，再一次Traversal
     boolean cancelDraw = mAttachInfo.mTreeObserver.dispatchOnPreDraw() || !isViewVisible;
        if (!cancelDraw) {
            if (mPendingTransitions != null && mPendingTransitions.size() > 0) {
                for (int i = 0; i < mPendingTransitions.size(); ++i) {
                    mPendingTransitions.get(i).startChangingAnimations();
                }
                mPendingTransitions.clear();
            }
		   //执行绘制流程
            performDraw();
        } else {
            if (isViewVisible) {
                //重新触发Traversals
                scheduleTraversals();
            } else if (mPendingTransitions != null && mPendingTransitions.size() > 0) {
                for (int i = 0; i < mPendingTransitions.size(); ++i) {
                    mPendingTransitions.get(i).endChangingAnimations();
                }
                mPendingTransitions.clear();
            }
        }
}
```

### OnDrawListener

调用时机：发在view tree 测量、布局之后，绘制之前。

和OnPreDrawListener的区别——

1. OnPreDrawListener可以通过onPreDraw返回值使得在即将绘制前重新再测量、布局再绘制。
2. OnDrawListener在发生回调时不可以对其所在的ArrayList进行增量操作，而OnPreDrawListener是用CopyOnWriteArrayList保存，所以可以迭代回调时移除该listener。

```java
/**
 * Interface definition for a callback to be invoked when the view tree is about to be drawn.
 */
public interface OnDrawListener {
    /**
     * <p>Callback method to be invoked when the view tree is about to be drawn. At this point,
     * views cannot be modified in any way.</p>
     * 
     * <p>Unlike with {@link OnPreDrawListener}, this method cannot be used to cancel the
     * current drawing pass.</p>
     * 
     * <p>An {@link OnDrawListener} listener <strong>cannot be added or removed</strong>
     * from this method.</p>
     *
     * @see android.view.View#onMeasure
     * @see android.view.View#onLayout
     * @see android.view.View#onDraw
     */
    public void onDraw();
}
```

**源码**

```java
private void performTraversals(){
    //...
      boolean canUseAsync = draw(fullRedrawNeeded);
}

   private boolean draw(boolean fullRedrawNeeded) {
        Surface surface = mSurface;
        if (!surface.isValid()) {
            return false;
        }

        if (mAttachInfo.mViewScrollChanged) {
            mAttachInfo.mViewScrollChanged = false;
            //dispatchOnScrollChanged
            mAttachInfo.mTreeObserver.dispatchOnScrollChanged();
        }
		
       //调用OnDrawListener
        mAttachInfo.mTreeObserver.dispatchOnDraw();

        if (!dirty.isEmpty() || mIsAnimating || accessibilityFocusDirty) {
            if (mAttachInfo.mThreadedRenderer != null && mAttachInfo.mThreadedRenderer.isEnabled()) {
              	//硬件加速
                mAttachInfo.mThreadedRenderer.draw(mView, mAttachInfo, this);
            } else {
                //软件绘制
                if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset,
                        scalingRequired, dirty, surfaceInsets)) {
                    return false;
                }
            }
        }
    }
```



