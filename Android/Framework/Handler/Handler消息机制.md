参考：

["一篇就够"系列: Handler消息机制完全解析](https://juejin.cn/post/6924084444609544199#heading-5)

[Android消息机制](https://ljd1996.github.io/2020/01/06/Android%E6%B6%88%E6%81%AF%E6%9C%BA%E5%88%B6/)

# Android消息机制

**Handler机制是Android中基于单线消息队列模式的一套线程消息机制。**

## ThreadLocal

### 概述

`ThreadLocal`的存在是为了解决——**如何将数据与线程绑定起来，从而该数据只能在绑定的线程里访问，而其它线程无法访问**

<img src="pic\image-20210402144749543.png" alt="image-20210402144749543" style="zoom:67%;" />

ThreadLocal会从各自的线程，取出自己维护的ThreadLocalMap，其key为ThreadLocal，value为ThreadLocal对应的泛型对象，这样每个ThreadLocal就可以把自己作为key把不同的value存储在不同的ThreadLocalMap，当获取数据的时候，同个ThreadLocal就可以从不同线程的ThreadLocalMap中得到不同的数据。因此当我们以线程作为作用域，并且不同线程需要具有不同数据副本的时候，我们就可以考虑使用ThreadLocal。而Looper正好适用于这种场景

## Handler

### 概述

Handler机制指的是Android单线程消息队列机制。**Handler对象本身在机制中担任消息的发送和接收者**。通常Hander用来切线程。在工作线程利用Handler发送Runnable，之后UI线程会将Runnable轮询出来执行。

<img src="pic\image-20210406101914254.png" alt="image-20210406101914254"  />

### 构造函数

```java
//————默认————
public Handler() {
        this(null, false);
    }

//————CallBack——————
public Handler(Callback callback) {
        this(callback, false);
    }
//———Looper—————
public Handler(Looper looper) {
        this(looper, null, false);
    }
public Handler(Looper looper,Callback callback) {
        this(looper, callback, false);
    }
//————Async（requires API level 28）————
public static Handler createAsync(Looper looper) {
        if (looper == null) throw new NullPointerException("looper must not be null");
        return new Handler(looper, null, true);
    }

public static Handler createAsync(Looper looper, @NonNull Callback callback) {
        if (looper == null) throw new NullPointerException("looper must not be null");
        if (callback == null) throw new NullPointerException("callback must not be null");
        return new Handler(looper, callback, true);
    }
```

### 使用

#### 发送消息

所有发送消息的方法分为以下2类：

1. post Runnable

   <img src="pic\image-20210429102845320.png" alt="image-20210429102845320" style="zoom:80%;" />

   Runnable：被封装进`Message.callback`

   Object：为`Message.obj`(`token`)，标识哪一个Message。用于移除指定的Message。

   long：延迟或者指定时间被Looper取出

2. send Message

   <img src="pic\image-20210429103322663.png" alt="image-20210429103322663" style="zoom:80%;" />

   int：`Message`的`what`，唯一标识消息

   long：延迟或者指定时间被`Looper`取出

   Object（`Message.obj`）：当发送不为空的消息时，`Object`既是`Data`又是`Token`

注意：

所有的send方法最终都会走enqueueMessage，回将Handler绑定到Message.target

```java
private boolean enqueueMessage(@NonNull MessageQueue queue, @NonNull Message msg,
        long uptimeMillis) {
    msg.target = this;//设置target=Handler
    msg.workSourceUid = ThreadLocalWorkSource.getUid();

    if (mAsynchronous) {
        msg.setAsynchronous(true);//设置是否异步数据
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```

#### 接收消息

```java
public void dispatchMessage(@NonNull Message msg) {
        if (msg.callback != null) {//1.优先返回给每一个Message的CallBack
            handleCallback(msg);
        } else {
            if (mCallback != null) {//2.返回给Handler的CallBack
                if (mCallback.handleMessage(msg)) {
                    return;
                }
            }
            handleMessage(msg);//3.最后返回给Handler的handleMessage方法
        }
    }
```

#### 移除消息

<img src="pic\image-20210429110002008.png" alt="image-20210429110002008" style="zoom:80%;" />

移除消息API分为3大类：

1. 移除指定、所有的`CallBack`。
2. 移除指定、所有`Message`。
3. 移除指定、所有`Runnable`和`Message`。

其中Object表示token，用于**移除指定的CalBack或者Message**。

```java
void removeMessages(Handler h, int what, Object object) {
    synchronized (this) {
        Message p = mMessages;
        //若设置token，则Message.obj和token相等才会移除
        while (p != null && p.target == h && p.what == what
               && (object == null || p.obj == object)) {
            Message n = p.next;
            mMessages = n;
            p.recycleUnchecked();
            p = n;
        }
    }
}  

void removeMessages(Handler h, Runnable r, Object object) {
    synchronized (this) {
        Message p = mMessages;
        //若设置token，则Message.obj和token相等才会移除(注意这里匹配的callback是Message中的，而并不匹配Handler的CallBack)
        while (p != null && p.target == h && p.callback == r
                   && (object == null || p.obj == object)) {
                Message n = p.next;
                mMessages = n;
                p.recycleUnchecked();
                p = n;
       }
    }
}  
```

------

## Looper

### 概述

Looper为线程提供了消息循环。

当线程创建Looper，并且启动循环，则Looper会不停的从MessageQueue中取出Message，并调用Message.target.dispatchMessage——及调用Hander去处理

### 使用

#### **初始化Looper**

```java
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

//—————————hide———————————
 private static void prepare(boolean quitAllowed) {
        if (sThreadLocal.get() != null) {
            throw new RuntimeException("Only one Looper may be created per thread");
        }
     //利用ThreadLocal将Looper设置到当前线程的变量中
        sThreadLocal.set(new Looper(quitAllowed));
    }

private Looper(boolean quitAllowed) {
        mQueue = new MessageQueue(quitAllowed);
        mThread = Thread.currentThread();
    }
```

`Looper.prepare()`会初始化一个Looper（顺便初始化MessageQueue），并将这个Looper添加到当前线程的私有变量中，其中key为Looper的全局静态变量`sThreadLocal`

#### **获取Looper**

```java
//获取UI线程的Looper
public static Looper getMainLooper() {
        synchronized (Looper.class) {
            return sMainLooper;
        }
    }
//获取当前线程的Looper
public static @Nullable Looper myLooper() {
        return sThreadLocal.get();
    }
```

#### **退出**

```java
//立即退出，在MessageQueue中剩余的消息无法得到执行，Handler无法发送消息
public void quit() {
        mQueue.quit(false);
    }
//安全退出，在MessageQueue中剩余的消息会执行（除了延迟的），Handler无法发送消息
public void quitSafely() {
        mQueue.quit(true);
    }
```

#### **轮询**

```java

//循环取消息，分发给Handler
public static void loop() {
    final Looper me = myLooper();
    final MessageQueue queue = me.mQueue;
      for (;;) {
           Message msg = queue.next();
           msg.target.dispatchMessage(msg);
      }
}
```

## MessageQueue

### 概述

MessageQueue是Handler机制的核心。

MessageQueue是消息队列（单链表结构）。它可以对链表进行插入、取出（取出顺便删除）。

### 使用

```java
//1. 插入消息：Handler发送消息的所有接口最终都是调用该方法
boolean enqueueMessage(Message msg, long when) 
 
//2. 移除消息（Handler调用）
void removeCallbacksAndMessages(Handler h, Object object) //移除runnable和message
void removeMessages(Handler h, int what, Object object)//移除message
void removeMessages(Handler h, Runnable r, Object object)//移除runnable
    
//3. 轮询消息（Looper调用）
Message next()

//4. 退出（由Looper调用）
void quit(boolean safe)
   
//5.插入一个同步屏障    
public int postSyncBarrier()

//6.添加一个IdleHandler
public void addIdleHandler(@NonNull IdleHandler handler) 
```

## Message

### 概述

任意数据的载体，用于发送给Handler去处理。

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
public long when;

// 处理这个Message的Handler
Handler target;

// 当我们使用Handler的post方法时候就是把runnable对象封装成Message
Runnable callback;

// MessageQueue是一个链表，next表示下一个
Message next;

//设置
 public void setAsynchronous(boolean async) {
        if (async) {
            flags |= FLAG_ASYNCHRONOUS;
        } else {
            flags &= ~FLAG_ASYNCHRONOUS;
        }
    }
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

## 异步消息与同步屏障

Message实际上分为三种类型：**同步消息**、**异步消息**、**同步屏障**。异步消息和同步消息的唯一区别在于用`FLAG_ASYNCHRONOUS`来区分：

```java
public void setAsynchronous(boolean async) {
    if (async) {
        flags |= FLAG_ASYNCHRONOUS;
    } else {
        flags &= ~FLAG_ASYNCHRONOUS;
    }
}
```

同步屏障也是一种特殊的消息，它的target为空，而无论是同步消息还是异步消息它们的target都指向Handler。来看一下如何发送同步屏障：

```java
//MessageQueue.java
@TestApi
public int postSyncBarrier() {
    return postSyncBarrier(SystemClock.uptimeMillis());
}

private int postSyncBarrier(long when) {
    synchronized (this) {
        final int token = mNextBarrierToken++;
        //1.构建一个Message，注意并没有指定target
        final Message msg = Message.obtain();
        msg.markInUse();
        msg.when = when;
        msg.arg1 = token;

        Message prev = null;
        Message p = mMessages;
        //2.后边逻辑就是将这个特殊的msg（同步屏障）插入到正确的位置
        if (when != 0) {
            while (p != null && p.when <= when) {
                prev = p;
                p = p.next;
            }
        }
        if (prev != null) { 
            msg.next = p;
            prev.next = msg;
        } else {
            msg.next = p;
            mMessages = msg;
        }
        return token;
    }
}
```

插入同步屏障和插入普通消息类似，但是这里有几个注意点：

1. 同步屏障没有设置target
2. 插入同步屏障没有唤醒队列
3. postSyncBarrier被标记为TestApi，Google不允许我们来调用。

## 插入消息队列

```java
boolean enqueueMessage(Message msg, long when) {
    if (msg.target == null) {
        throw new IllegalArgumentException("Message must have a target.");
    }
    if (msg.isInUse()) {
        throw new IllegalStateException(msg + " This message is already in use.");
    }

    synchronized (this) {
        //1.退出状态时不允许再插入
        if (mQuitting) {
            IllegalStateException e = new IllegalStateException(
                    msg.target + " sending message to a Handler on a dead thread");
            Log.w(TAG, e.getMessage(), e);
            msg.recycle();
            return false;
        }

        //2.标记为使用
        msg.markInUse();
        msg.when = when;
        Message p = mMessages;
        boolean needWake;
        if (p == null || when == 0 || when < p.when) {//空队列
            //3.插入至表头，需要唤醒队列
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;
        } else {//插入到中间位置。
            //4.一般不需要唤醒，除非有一个同步屏障在表头并且这个被插入的消息是队列中最早要被执行的异步消息
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            msg.next = p; 
            prev.next = msg;
        }

        // We can assume mPtr != 0 because mQuitting is false.
        if (needWake) {
            nativeWake(mPtr);
        }
    }
    return true;
}
```

插入消息队列就是根据msg的when将msg插入到合适的位置，有几点需要注意：

1. 只有MessageQueue处于退出的状态时才会插入失败
2. 插入消息会根据当前Queue是否阻塞、同步屏障及异步消息来决定是否唤醒

## 轮询取出消息

```java
Message next() {
    final long ptr = mPtr;
    if (ptr == 0) {
        return null;
    }

    int pendingIdleHandlerCount = -1; // -1 only during first iteration
    int nextPollTimeoutMillis = 0;
    for (;;) {
        if (nextPollTimeoutMillis != 0) {
            Binder.flushPendingCommands();
        }

        //1.⭐使cpu休眠，nextPollTimeoutMillis表示到下次自动唤醒的时差，当值为-1时表示永久休眠。
        nativePollOnce(ptr, nextPollTimeoutMillis);

        synchronized (this) {
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            //2.判断该消息是否为同步屏障
            if (msg != null && msg.target == null) {
                //若该消息是同步屏障，则开始循环找出链表后面最近的异步消息
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) {
                if (now < msg.when) {
                    //3.当该消息还未到需要执行的时刻，则计算CPU需要休眠的时差使CPU休眠
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    //4.当该消息到了执行时刻
                    mBlocked = false;
                    //后边就是将该消息移除返回
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                    msg.markInUse();
                    return msg;
                }
            } else {
                //5.⭐执行到这里表示队列为空，则cpu永久休眠
                nextPollTimeoutMillis = -1;
            }

            //6.当消息队列需要销毁时会执行这里。
            if (mQuitting) {
                dispose();
                return null;
            }

            // If first time idle, then get the number of idlers to run.
            // Idle handles only run if the queue is empty or if the first message
            // in the queue (possibly a barrier) is due to be handled in the future.
            if (pendingIdleHandlerCount < 0
                    && (mMessages == null || now < mMessages.when)) {
                pendingIdleHandlerCount = mIdleHandlers.size();
            }
            if (pendingIdleHandlerCount <= 0) {
                // No idle handlers to run.  Loop and wait some more.
                mBlocked = true;
                continue;
            }

            if (mPendingIdleHandlers == null) {
                mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
            }
            mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
        }

        // Run the idle handlers.
        // We only ever reach this code block during the first iteration.
        for (int i = 0; i < pendingIdleHandlerCount; i++) {
            final IdleHandler idler = mPendingIdleHandlers[i];
            mPendingIdleHandlers[i] = null; // release the reference to the handler

            boolean keep = false;
            try {
                keep = idler.queueIdle();
            } catch (Throwable t) {
                Log.wtf(TAG, "IdleHandler threw exception", t);
            }

            if (!keep) {
                synchronized (this) {
                    mIdleHandlers.remove(idler);
                }
            }
        }

        // Reset the idle handler count to 0 so we do not run them again.
        pendingIdleHandlerCount = 0;

        // While calling an idle handler, a new message could have been delivered
        // so go back and look again for a pending message without waiting.
        nextPollTimeoutMillis = 0;
    }
}
```

