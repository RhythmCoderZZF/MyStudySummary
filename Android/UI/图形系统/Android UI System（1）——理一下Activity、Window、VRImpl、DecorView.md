# Android UI System（1）——理一下Activity、Window、ViewRootImpl、DecorView

参考：

[Android-Window机制原理](https://ljd1996.github.io/2020/08/27/Android-Window%E6%9C%BA%E5%88%B6%E5%8E%9F%E7%90%86/)

[setContentView流程简析](https://juejin.cn/post/6906389320312225806)

[捋一捋，到底怎么样去理解Window机制？](https://juejin.cn/post/6952341892721803294#heading-8)



## 创建Activity、Window、WindowManager

### 创建Activity

Activity初始化位置是在`ActivityThread.performLaunchActivity`中：

```java
 private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {
        if (r.activityInfo.targetActivity != null) {
            component = new ComponentName(r.activityInfo.packageName,
                    r.activityInfo.targetActivity);//targetActivity顾名思义是要跳转的activity的class name
        }
		
        ContextImpl appContext = createBaseContextForActivity(r);//实例化ContextImpl，是Context最基础的实现类
        Activity activity = null;
        java.lang.ClassLoader cl = appContext.getClassLoader();
     
     	//1.反射创建TargetActivity
        activity = mInstrumentation.newActivity(cl, component.getClassName(), r.intent);
        try {
            Application app = r.packageInfo.makeApplication(false, mInstrumentation);
            if (activity != null) {
                
                //2.执行Activity.Attach()
                activity.attach(...);
                int theme = r.activityInfo.getThemeResource();
                if (theme != 0) {
                    activity.setTheme(theme);//设置主题
                }
                //3.执行Activity.create()
                mInstrumentation.callActivityOnCreate(activity, r.state);
                r.activity = activity;
            }
        return activity;
    }
```

### 创建Window、WindowManager

重点看一下attach()做了什么：

```java
final void attach(...) {
    
    //1.初始化PhoneWindow（Window的唯一实现）
    mWindow = new PhoneWindow(this, window, activityConfigCallback);
    
    //2.PhoneWindow初始化WindowManager
    mWindow.setWindowManager(
            (WindowManager)context.getSystemService(Context.WINDOW_SERVICE),
            mToken, mComponent.flattenToString(),
            (info.flags & ActivityInfo.FLAG_HARDWARE_ACCELERATED) != 0);
    
    if (mParent != null) {
        mWindow.setContainer(mParent.getWindow());
    }
    
    //3.将WindowManager添加到Activity
    mWindowManager = mWindow.getWindowManager();
}
```

PhoneWindow初始化了一个WindowManager：

```java
/**
 * Set the window manager for use by this Window to, for example,
 * display panels.  This is <em>not</em> used for displaying the
 * Window itself -- that must be done by the client.
 *
 * Window没有屏幕显示的能力，具体是依靠WindowManager来实现。
 */
public void setWindowManager(WindowManager wm, IBinder appToken, String appName,
        boolean hardwareAccelerated) {
    if (wm == null) {
        wm = (WindowManager)mContext.getSystemService(Context.WINDOW_SERVICE);
    }
    
    //1.实例化一个WindowManagerImpl
    mWindowManager = ((WindowManagerImpl)wm).createLocalWindowManager(this);
}

public WindowManagerImpl createLocalWindowManager(Window parentWindow) {
        return new WindowManagerImpl(mContext, parentWindow);
    }
```

注意：这里的WindowManager并不是WMS，这要归结于`Context.getSystemService`和`SystemManager.getService`的区别。参考[Android系统中ServiceManager.getService和Context.getSystemService区别](https://blog.csdn.net/yezi121363/article/details/79519429)。简单的说，通过`Context.getSystemService`拿到的只是WMS的代理对象。

### 小结

Activity初始化发生在`ActivityThread.performLaunchActivity`中。Activity初始化会创建一个PhoneWindow，而PhoneWindow初始化会创建一个WindowManagerImpl。

## 创建DecorView

`Activity.attach()`之后就会执行`Activity.create()`。其中setContentView是我们不得不调用的方法。

```java
/* AppCompatActivity */
public void setContentView(@LayoutRes int layoutResID) {
    getDelegate().setContentView(layoutResID);//AppCompatDelegateImpl.setContentView
}

/* AppCompatDelegateImpl */
public void setContentView(int resId) {
    
    	//1.初始化DecorView
        ensureSubDecor();
        ViewGroup contentParent = mSubDecor.findViewById(android.R.id.content);
    
    	//2.将我们Activity的ContentView依附到这个id为 android.R.id.content 的布局之上。
        LayoutInflater.from(mContext).inflate(resId, contentParent);
    }
```

`setContentView`就是将我们Activity的ContentView依附到系统提供的DecorView的id为`android.R.id.content`的FrameLayout之上。那么系统创建的DecorView一定是在`ensureSubDecor()`中执行了。

### 创建SubDecor

```java
private void ensureSubDecor() {
    if (!mSubDecorInstalled) {
        //1.调用createSubDecor
        mSubDecor = createSubDecor();
        ...
        mSubDecorInstalled = true;
    }
}

private ViewGroup createSubDecor() {
        TypedArray a = mContext.obtainStyledAttributes(R.styleable.AppCompatTheme);
		...
      	 a.recycle();// 主题样式...
    
		//1.确保PhoneWindow已经创建
        ensureWindow();
    
    	//2.让PhoneWindow去创建DecorView,这行代码执行完毕Decor就已经创建完毕啦！那么很奇怪，后面一大堆是干啥的？？
        mWindow.getDecorView();
		
        final LayoutInflater inflater = LayoutInflater.from(mContext);
        ViewGroup subDecor = null;
	
    	//3.subDecor = R.layout.abc_screen_simple
        subDecor = (ViewGroup) inflater.inflate(R.layout.abc_screen_simple, null);
    	
        //4.contentView = subDecor中R.id.action_bar_activity_content的布局
        final ContentFrameLayout contentView = (ContentFrameLayout) subDecor.findViewById(
                R.id.action_bar_activity_content);
		
        //4.windowContentView = DecorView中R.id.content的布局
        final ViewGroup windowContentView = (ViewGroup) mWindow.findViewById(android.R.id.content);
    	
        if (windowContentView != null) {
            ...
			
            //5.清除DecorView中 android.R.id.content 的布局id
            windowContentView.setId(View.NO_ID);
            
            //6.将contentView的id重新替换为 android.R.id.content
            contentView.setId(android.R.id.content);

        }
    	
    	//7.利用PhoneWindow将subDecor添加到DecorView中
        mWindow.setContentView(subDecor);
        return subDecor;
    }
```

在第2步中通过PhoneWindow去创建DecorView，在第7步中将SubDecor依附到DecorView的中。接下来分析这两步：

```java
/* 
PhoneWindow.java
2.创建DecorView 
*/
public final @NonNull View getDecorView() {
    if (mDecor == null || mForceDecorInstall) {
        installDecor();
    }
    return mDecor;
}

 private void installDecor() {
        mForceDecorInstall = false;
        if (mDecor == null) {
            
            //1.实例化一个DecorView（类型为FrameLayout）
            mDecor = generateDecor(-1);
        } else {
            mDecor.setWindow(this);
        }
        if (mContentParent == null) {
            
            //2.这一步会先去加载一个为R.layout.screen_simple布局（LinearLayout），将其依附到DecorView中。其中该布局中有个id为android.R.id.content的布局（FrameLayout），mContentParent就是这个FrameLayout的实例
            mContentParent = generateLayout(mDecor);
        }    
 }

/* 7.将SubDecor依附到DecorView上 */
public void setContentView(int layoutResID) {
      ...
        //3.将SubDecor依附到ContentParent上（DecorView中R.id.content的布局）
        mLayoutInflater.inflate(layoutResID, mContentParent);
        
    }

```

### **小结**

`setContent`过程主要为一下几步：

1. 利用PhoneWindow去创建一个DecorView（FrameLayout）。加载一个LinearLayout依附到Decor之上，其中有一个布局id为`android.R.id.content`。mContentParent就是这个布局的实例。
2. AppCompatDelegate加载一个subDecor（LinearLayout），先将mContentParent的id注销，将subDecor中一个child的id置为`android.R.id.content`，然后将subDecor依附到mContentParent上，本质就是偷梁换柱。
3. 最后将我们的Activity的布局文件依附到`android.R.id.content`这个布局之上。

最终效果：

<img src="pic\image-20210520150751956.png" alt="image-20210520150751956" style="zoom: 67%;" />

## 创建ViewRootImpl

和Activity的初始化一样，ViewRootImpl初始化位置是在`ActivityThread.handleResumeActivity`中：

```java
public void handleResumeActivity(){
     //1.先调用Activity.resume
      final ActivityClientRecord r = performResumeActivity(token, finalStateRequest, reason)
      final Activity a = r.activity;
    
      //2.获取PhoneWindow中的DecorView
      View decor = r.window.getDecorView();
      ViewManager wm = a.getWindowManager();
      WindowManager.LayoutParams l = r.window.getAttributes();
      a.mDecor = decor;
      l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
    
      //3.重点方法🚀——调用Activity的WindowManagerImpl.ddView，传入Decor
      wm.addView(decor, l);
}

/* WindowManagerImpl */
public void addView(@NonNull View view, @NonNull ViewGroup.LayoutParams params) {
    	//1.通过WindowManagerGlobal（单例）add Decor。
        mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);//mParentWindow=PhoneWindow
    }
```

`WindowManagerGlobal.addView`（注意`WindowManagerGlobal`是个单例）

```java
private final ArrayList<View> mViews = new ArrayList<View>();//Decors
private final ArrayList<ViewRootImpl> mRoots = new ArrayList<ViewRootImpl>();//ViewRootImpls

public void addView(View view, ViewGroup.LayoutParams params,Display display, Window parentWindow) {
    final WindowManager.LayoutParams wparams = (WindowManager.LayoutParams) params;
    ViewRootImpl root;
    View panelParentView = null;
    
    synchronized (mLock) {
	    //1.实例化ViewRootImpl
        root = new ViewRootImpl(view.getContext(), display);
        view.setLayoutParams(wparams);
       
        mViews.add(view);
        mRoots.add(root);
        mParams.add(wparams);

       //2.调用ViewRootImpl.setView，setView会调用requestLayout去启动View绘制的三大流程
       root.setView(view, wparams, panelParentView);
    }
}
```

### 小结

ViewRootImpl的实例化过程发生在`Acrivity.resume`之后，通过`WindowManagerImpl.addView(Decor)`去创建ViewRootImpl。之后便开始View绘制的三大流程。

## 理解Window的概念

查看Window.java类的注释：

```
 * Abstract base class for a top-level window look and behavior policy.  An
 * instance of this class should be used as the top-level view added to the
 * window manager. It provides standard UI policies such as a background, title
 * area, default key processing, etc.

Window是一个掌管行为与外观的顶层窗口，是一个抽象类。
Window的实例应当被视为一个顶层View被添加进Window的管理器。
Window决定着UI样式与默认处理。
```

一句话：Window是一个抽象的顶级窗口（这个窗口实际上显示的东西是Decor），它是`外观`与`行为`的抽象

我们知道，如果要再屏幕上显示一个View，需要调用`WindowManager.addView(View,WindowManager.LayoutParames)`，实际上`外观`对应的就是View；`WindowManager.LayoutParames`对应的是`行为`。



## 启动白屏与快照

App第一次启动（冷启动）到第一个Activity显示会非常耗时，如果Android不提供一种机制会造成用户点击App图片长时间无反应。不过好在Android提供了`StartWindow`去填充这一部分真空期。

当Activity创建前，AMS会读取Activity的注册信息，创建并显示StartWindow，等Activity创建完毕显示出来的时候再移除。StartWindow显示的内容就是在style中定义windowBackground属性，默认为白屏。

当App热启动时，StartWindow就会显示App退出前的快照（也就是App退出前Activity的截图）。比如下拉刷新时立即退出，再进入会发现下拉提示会短暂显示。

## 总结

<img src="pic\image-20210521084255462.png" alt="image-20210521084255462" style="zoom: 50%;" />

<img src="pic\image-20210520164831560.png" alt="image-20210520164831560" style="zoom: 50%;" />

结合这两幅图总结一下Activity、Window、ViewRootImpl、DecorView的概念、作用和关系：

1. DecorView是最终显示到屏幕的根视图，我们在Activity中创建的布局是依附在这个View之上。
2. ViewRootImpl的创建实际为`Activity.resume`之后，是被WindowManagerImpl通过WindowManagerGlobal去创建，紧接着调用setView去绘制Decor。
3. Window（PhoneWindow）掌管Decor的初始化，它决定了Decor的样式与行为。
4. Acrivity是Window的载体。

从上述程序执行从Activity的Create到Resume过程中，总结一下两点：

1. create过程做了两件事——1.初始化PhoneWindow，WindowManagerImpl；2.创建布局（包含我们的布局的Decor）
2. resume之后通过`WindowManagerImpl.addView`去显示到屏幕（创建ViewRootImpl，调用setView）

由此再去看PopupWindow、Dialog，它们也是通过`WindowManagerImpl.addView`去显示View。其中Dialog还通过创建PhoneWindow去创建Decor，简直和Activity中显示View一摸一样。

