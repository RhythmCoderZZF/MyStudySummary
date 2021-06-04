# IPC系列（2）——其他IPC机制

参考：

《Android开发艺术探索》

## Bundle

Android四大组件间进程通信。

## 共享文件

两个程序通过文件、SharedPreferences（本质和文件一样）实现跨进程通信。

## Messenger

Messenger只是对Binder的封装，本质也是利用Binder机制。

### 案例

**Server**

```kotlin
class MessengerService : Service() {
    private var mClientMessenger: Messenger? = null

    private val handler = object : Handler() {
        override fun handleMessage(msg: Message) {
            when (msg.what) {
                //2.接收Client传递的Messenger
                0 -> {
                    mClientMessenger = msg.replyTo
                }
                //3.接收Client传递的Data
                1 -> {
                    val d = msg.data.get("client_data")
                    //4.返回Data给Client
                    mClientMessenger?.send(Message.obtain().apply {
                        data = Bundle().apply {
                            putString("server_data", "来自Server的回复:${d}")
                        }
                    })
                }
            }
        }
    }
    private val mServerMessenger = Messenger(handler)

    override fun onBind(intent: Intent): IBinder {
        //1.暴露Binder
        return mServerMessenger.binder
    }
}
```

**Client**

```kotlin
class MessengerActivity : BaseActivity() {
    private var mServerMessenger: Messenger? = null

    private var mHandler = Handler {
        Toast.makeText(this, "${it.data.get("server_data")}", Toast.LENGTH_SHORT).show()
        true
    }

    private val mClientMessenger = Messenger(mHandler)

    private var conn = object : ServiceConnection {
        override fun onServiceConnected(name: ComponentName?, service: IBinder?) {
            mServerMessenger = Messenger(service)
            val message = Message.obtain().apply {
                what = 0
                replyTo = mClientMessenger
            }
            //2.在绑定Server Binder的时候通过该Binder将ClientMessenger发送给Server
            mServerMessenger?.send(message)
        }

        override fun onServiceDisconnected(name: ComponentName?) {
        }
    }


    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_android_ipc_messenger)

        bind.setOnClickListener {
            //1.绑定服务
            bindService(Intent().apply {
                action = "aidl_messenger_action"
                `package` = "com.example.android_study"
            }, conn, BIND_AUTO_CREATE)
        }

        send.setOnClickListener {
            if (edt.text.isNullOrEmpty()) {
                Toast.makeText(this, "请输入", Toast.LENGTH_SHORT).show()
                return@setOnClickListener
            }
            //3.发送消息给Server
            mServerMessenger?.send(Message.obtain().apply {
                //obj = edt.text.toString() 不要使用obj传递对象，无法序列化
                what = 1
                data = Bundle().apply {
                    putString("client_data", edt.text.toString())
                }
            })
        }
    }

    override fun onDestroy() {
        super.onDestroy()
        unbindService(conn)
    }
}
```

### 工作原理

![image-20210528134647722](pic\image-20210528134647722.png)

1. Messenger维护的Binder会将Message转发给Handler
2. 服务端暴露Binder给客户端，这样客户端可以通过构建一个服务端Messenger去包装这个Binder，从而可以向服务端发送数据。
3. 客户端也可以通过Messenger自己去构建一个本地Binder，通过服务端的Messenger发送给服务端。这样每个端都有对方的Binder去通信了

### 优缺点

优点：基于Binder，安全且效率高；由系统提供了Messenger方便我们开发使用

缺点：由于基于Handle的消息处理，并发处理慢；如果想直接以调用方法的方式去通信，Messenger也不合适。

## AIDL

即编写AIDL实现Binder

## ContentProvider

## Socket

## 不同IPC方式优缺点

<img src="pic\image-20210528145206538.png" alt="image-20210528145206538" style="zoom:80%;" />

