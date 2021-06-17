# Android UI System（2）——Surface创建

参考：

[捋一捋，到底怎么样去理解Window机制？](https://juejin.cn/post/6952341892721803294#heading-6)

[Android-Surface原理解析](https://ljd1996.github.io/2020/11/09/Android-Surface%E5%8E%9F%E7%90%86%E8%A7%A3%E6%9E%90/)

## Window被创建添加

```java
    //ViewRootImpl.java

    public final Surface mSurface = new Surface();
    private final SurfaceControl mSurfaceControl = new SurfaceControl();

    final IWindowSession mWindowSession;
    final W mWindow;
    Choreographer mChoreographer;
    WindowInputEventReceiver mInputEventReceiver;
    
    public ViewRootImpl(Context context, Display display) {
        ...  
        mWindowSession = WindowManagerGlobal.getWindowSession();
        mWindow = new W(this);
        mChoreographer = Choreographer.getInstance();
        ...
    }

    public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
    synchronized (this) {
            if (mView == null) {
                mView = view;
                ......
                requestLayout();
                //1.通过mWindowSession通知WMS添加并显示窗口。
                res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
                            getHostVisibility(), mDisplay.getDisplayId(), mTmpFrame,
                            mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
                            mAttachInfo.mOutsets, mAttachInfo.mDisplayCutout, mInputChannel,
                            mTempInsets);
              ......
              //初始化事件接收器，
              mInputEventReceiver = new WindowInputEventReceiver(mInputChannel,
                            Looper.myLooper());
              
              //2.ViewRootImpl作为Decor的父View,将当前ViewRootImpl实例赋值给DecorView的mParent变量	
              view.assignParent(this);
  }                          
```

### 小结

`mWindowSession`和`mWindow`分别是WMS和App端的aidl接口，两者实现Binder通信。

<img src="pic\image-20210526093142172.png" alt="image-20210526093142172" style="zoom:80%;" />

## Surface被创建

```java
private void performTraversals() {
    //1.surface会在绘制的三大流程之前被创建
    relayoutWindow(params, viewVisibility, insetsPending)
    ...
    performMesaure()
    performLayout()
    performDraw
}

private int relayoutWindow(){
     mWindowSession.relayout(mWindow,mSurfaceControl)
     //2.创建Surface
     mSurface.copyFrom(mSurfaceControl);    
}

```

### 小结

在 Java 层中 ViewRootImpl 实例中持有一个 Surface 对象，该 Surface 对象中的 mNativeObject 属性指向 native 层中创建的 Surface 对象，native 层的 Surface 对应 SurfaceFlinger 中的 Layer 对象，它持有 Layer 中的 BufferQueueProducer 生产者指针，在 Surface 上绘制的内容最终会交由 SurfaceFlinger 来合成渲染送到显示器显示

