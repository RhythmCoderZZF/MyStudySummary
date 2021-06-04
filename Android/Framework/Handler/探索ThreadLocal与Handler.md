参考：["一篇就够"系列: Handler消息机制完全解析](https://juejin.cn/post/6924084444609544199#heading-5)

# 探索ThreadLocal与Handler

## ThreadLocal

`ThreadLocal`的存在是为了解决——**如何将数据与线程绑定起来，从而该数据只能在绑定的线程里访问，而其它线程无法访问**

<img src="pic\image-20210402144749543.png" alt="image-20210402144749543" style="zoom:67%;" />

ThreadLocal会从各自的线程，取出自己维护的ThreadLocalMap，其key为ThreadLocal，value为ThreadLocal对应的泛型对象，这样每个ThreadLocal就可以把自己作为key把不同的value存储在不同的ThreadLocalMap，当获取数据的时候，同个ThreadLocal就可以从不同线程的ThreadLocalMap中得到不同的数据。因此当我们以线程作为作用域，并且不同线程需要具有不同数据副本的时候，我们就可以考虑使用ThreadLocal。而Looper正好适用于这种场景

## Handler

>Handler机制是Android单线程消息队列机制。**Handler对象本身在机制中担任消息的发送和接收者**。通常Hander用来切线程，发送消息处于Work Thread，而等接收到消息时已经处于UI Thread。

<img src="pic\image-20210406101914254.png" alt="image-20210406101914254"  />

### 构造函数

```java
public Handler() {
        this(null, false);//当前线程Looper
    }
public Handler(Callback callback) {
        this(callback, false);//当前线程Looper
    }
//Lopper
public Handler(Looper looper) {
        this(looper, null, false);//Looper
    }
public Handler(Looper looper,Callback callback) {
        this(looper, callback, false);//Looper
    }
//Async=true
public static Handler createAsync(Looper looper) {
        if (looper == null) throw new NullPointerException("looper must not be null");
        return new Handler(looper, null, true);
    }

public static Handler createAsync(Looper looper, @NonNull Callback callback) {
        if (looper == null) throw new NullPointerException("looper must not be null");
        if (callback == null) throw new NullPointerException("callback must not be null");
        return new Handler(looper, callback, true);
    }

//@hide ————————————————————————————————————————————————————————————————————————————————————————————
public Handler(Callback callback, boolean async) {
        mLooper = Looper.myLooper();//未设置Looper，直接利用全局静态变量sThreadLocal取出当前线程的Looper
        if (mLooper == null) {
            throw new RuntimeException(
                "Can't create handler inside thread " + Thread.currentThread()
                        + " that has not called Looper.prepare()");
        }
        mQueue = mLooper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }

//@hide
public Handler(Looper looper, Callback callback, boolean async) {
        mLooper = looper;//设置Looper就用传入的Looper
        mQueue = looper.mQueue;
        mCallback = callback;
        mAsynchronous = async;
    }


```

Handler 构造函数分为3大类：

1. 是否指定 Looper——不传入Looper，当前线程又没有Looper的话就会崩溃。
2. 是否传入 CallBack
3. 是否指定Async=true

### 发送消息

所有发送消息的方法分为以下2类：

1. post Runnable

   <img src="pic\image-20210429102845320.png" alt="image-20210429102845320" style="zoom:80%;" />

   Runnable：被封装进`Message.callback`

   Object：为`Message.obj`(`token`)，标识哪一个Message。用于移除指定的Message时。

   long：延迟或者指定时间被Looper取出

2. send Message

   <img src="pic\image-20210429103322663.png" alt="image-20210429103322663" style="zoom:80%;" />

   int：`Message`的`what`，唯一标识消息

   long：延迟或者指定时间被`Looper`取出

   Object（`Message.obj`）：当发送不为空的消息时，`Object`既是`Data`又是`Token`

### 接收消息

```java
public void dispatchMessage(@NonNull Message msg) {
        if (msg.callback != null) {//1.返回给每一个Message的CallBack
            handleCallback(msg);
        } else {
            if (mCallback != null) {//2.返回给Handler的CallBack
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);//3.返回给Handler的handleMessage方法
        }
    }
```

### 移除消息

<img src="pic\image-20210429110002008.png" alt="image-20210429110002008" style="zoom:80%;" />

移除消息API分为3大类：

1. 移除指定、所有的`Runnable`。Object（表示token）用于唯一标识
2. 移除指定、所有`Message`。Object（表示Data）用于唯一标识
3. 移除指定、所有`Runnable`和`Message`。Object（表示Data和Token）用于唯一标识

------

## Looper

> Looper是颗“心脏”。Looper所在的线程负责无限循环来将 MessageQueue 中的 Message传递给Handler。因此Looper处于什么线程，Handler得到的消息就处于什么线程。

### 相关API

```java
//1.初始化Looper
public static void prepareMainLooper() {
        prepare(false);
        synchronized (Looper.class) {
            if (sMainLooper != null) {
                throw new IllegalStateException("The main Looper has already been prepared.");
            }
            sMainLooper = myLooper();
        }
    }
 public static void prepare() {
        prepare(true);
    }

//@hide
 private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
        sThreadLocal.set(new Looper(quitAllowed));
    }
//@hide
private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }
//2.获取Looper
public static Looper getMainLooper() {
        synchronized (Looper.class) {
            return sMainLooper;
        }
    }
public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }
//3.退出
public void quit() {
        mQueue.quit(false);
    }
public void quitSafely() {
        mQueue.quit(true);
    }
//4.loop：循环取消息，分发给Handler
public static void loop() {
    final Looper me = myLooper();
    final MessageQueue queue = me.mQueue;
      for (;;) {
           Message msg = queue.next();
           msg.target.dispatchMessage(msg);
      }
}
```

Looper函数涉及4个方面：

1. 初始化——1.初始化Looper、MessageQueue；2.将初始化的Looper加到当前线程的ThreadLocalMap中，其中key是全局静态变量sThreadLocal。
2. 获取当前线程的Looper——通过暴露的全局静态变量sThreadLocal区获取当前线程内ThreadLocalMap保存的Looper
3. 调用MessageQueue来退出
4. loop()——死循环从MessageQueue取Message，分发给Handler。

## MessageQueue

>MessageQueue充当"血液"。它是Message的容器队列，由Looper这颗"心脏"驱动而不断流动。

### 相关API

```java
//1. 压入消息：Handler发送消息的所有接口最终都是调用该方法
boolean enqueueMessage(Message msg, long when) 
 
//2. 移除消息
void removeCallbacksAndMessages(Handler h, Object object) //移除runnable和message
void removeMessages(Handler h, int what, Object object)//移除message
void removeMessages(Handler h, Runnable r, Object object)//移除runnable
    
//3. 取出消息
Message next()

//4. 退出
void quit(boolean safe)
```

## Message

> 任意数据的载体，用于发送给Handler去处理。

### 属性

```java
// 用户自定义，主要用于辨别Message的类型
public int what;
// 用于存储一些整型数据
public int arg1;
public int arg2;
// 可放入一个可序列化对象
public Object obj;
// Bundle数据
Bundle data;

// Message处理的时间。相对于1970.1.1而言的时间
// 对用户不可见
public long when;

// 处理这个Message的Handler
// 对用户不可见
Handler target;

// 当我们使用Handler的post方法时候就是把runnable对象封装成Message
// 对用户不可见
Runnable callback;

// MessageQueue是一个链表，next表示下一个
// 对用户不可见
Message next;
```

### 相关API

```java
//1. 从缓存池中获取一个Message
public static Message obtain() {
        synchronized (sPoolSync) {
            if (sPool != null) {
                Message m = sPool;
                sPool = m.next;
                m.next = null;
                m.flags = 0; // clear in-use flag
                sPoolSize--;
                return m;
            }
        }
        return new Message();
    }
//2. 将Message回收之缓存池
void recycleUnchecked() {
        flags = FLAG_IN_USE;
        what = 0;
        arg1 = 0;
        arg2 = 0;
        obj = null;
        replyTo = null;
        sendingUid = UID_NONE;
        workSourceUid = UID_NONE;
        when = 0;
        target = null;
        callback = null;
        data = null;

        synchronized (sPoolSync) {
            if (sPoolSize < MAX_POOL_SIZE) {
                next = sPool;
                sPool = this;
                sPoolSize++;
            }
        }
    }
```











