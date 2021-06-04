# IPC系列（1）——初识Binder

参考：

《Android开发艺术探索》

[Android Binder机制浅析](https://blog.csdn.net/singwhatiwanna/article/details/19756201)

## IPC——跨进程通信

<img src="pic\image-20210522165615391.png" alt="image-20210522165615391" style="zoom: 50%;" />

Android为每个进程分配虚拟空间，虚拟空间分为`用户空间`和`内核控件`。只有内核空间具备和提供跨进程通信的能力。

## Binder通信模型

Binder是Android实现IPC通信的一种方式。

<img src="pic\image-20210522172011055.png" alt="image-20210522172011055" style="zoom:80%;" />

<img src="pic\image-20210522165653558.png" alt="image-20210522165653558" style="zoom:50%;" />

Binder通信模型和网络通信模型非常相似。

Client无法和Service直接通信，必须依赖ServiceManager中注册的Service。这三者又必须依靠Binder驱动在内核空间完成通信。其中ServiceManager和Binder驱动是Android已经提供好的。

## Binder

1. 直观来说，Binder是Android中的一个类，它继承了IBinder接口。服务端通过提供Binder实体，客户端通过Binder引用以一种面向对象的方式去调用服务端。

2. 从IPC角度来说，Binder是Android中的一种跨进程通信方式，Binder还可以理解为一种虚拟的物理设备，它的设备驱动是/dev/binder，该通信方式在linux中没有。

3. 从Android Framework角度来说，Binder是ServiceManager连接各种Manager（ActivityManager、WindowManager，etc）和相应ManagerService的桥梁。

4. 从Android应用层来说，Binder是客户端和服务端进行通信的媒介，当你bindService的时候，服务端会返回一个包含了服务端业务调用的Binder对象，通过这个Binder对象，客户端就可以获取服务端提供的服务或者数据，这里的服务包括普通服务和基于AIDL的服务。

## AIDL

是应用Binder的一种实现方式。AIDL是专门为Binder通信设计出的语言，它更像是一个工具，为我们方便的生成基于Binder通信所必须的Java类，没有这个工具我们也可以自己手写去实现。

生成的Java类其中有4个重要的对象：

1. IBinder——跨进程通信的能力，实现这个接口的对象具备跨进程传输的能力
2. IInterface——Service端定义的其具备哪些功能接口
3. Binder——继承IBinder
4. Stub——继承Binder，实现IInterface。(我们最终会调用这个Stub来操作Binder)

### 编写AIDL

1. 创建`aidl文件夹`，

2. 非基本类型需要传输的对象需声明`aidl文件`（Book.aidl）和`java bean`（Book.java），

3. 创建Server端需要要实现的功能接口`IBookInterface.aidl`，

4. 注意这些文件创建的包名必须相同，否则编译报错。

   <img src="pic\image-20210524101912244.png" alt="image-20210524101912244" style="zoom:67%;" />

下面看一下编译出的java文件`IBookInterface.java`：

```java
package aidl;

public interface IBookInterface extends android.os.IInterface {
    
    public static abstract class Stub extends android.os.Binder implements aidl.IBookInterface {
        //Binder唯一标识，Client和Servce对接的凭证，必须一样
        private static final java.lang.String DESCRIPTOR = "aidl.IBookInterface";
        //方法code唯一标识
        static final int TRANSACTION_getBooks = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0); 

        
        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }

        /*
        1. Client调用方法，获取Binder代理对象引用，转换成IInterface(当client和Servcie处于同一进程则返回Stub本身，否则返回的是Stub.Proxy封装后的Stub)
        */
        public static aidl.IBookInterface asInterface(android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (((iin != null) && (iin instanceof aidl.IBookInterface))) {
                return ((aidl.IBookInterface) iin);
            }
            return new aidl.IBookInterface.Stub.Proxy(obj);
        }

        @Override
        public android.os.IBinder asBinder() {
            return this;
        }

         /*
        3.该方法运行在Server端的线程池中
        (1)客户端发起RPC(远程调用请求)，通过系统封装后，Server调用此方法
        (2)Server根据code知道Client调用的是什么方法
        (3)Servcer从data中取出参数，执行目标方法，之后将结果写入reply中
        */
        @Override
        public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
            java.lang.String descriptor = DESCRIPTOR;
            switch (code) {
                case INTERFACE_TRANSACTION: {
                    reply.writeString(descriptor);
                    return true;
                }
                case TRANSACTION_getBooks: {
                    data.enforceInterface(descriptor);
                    int _arg0;
                    _arg0 = data.readInt();
                    java.util.List<aidl.Book> _result = this.getBooks(_arg0);
                    reply.writeNoException();
                    reply.writeTypedList(_result);
                    return true;
                }
                default: {
                    return super.onTransact(code, data, reply, flags);
                }
            }
        }

        private static class Proxy implements aidl.IBookInterface {
            private android.os.IBinder mRemote;

            Proxy(android.os.IBinder remote) {
                mRemote = remote;
            }

            @Override
            public android.os.IBinder asBinder() {
                return mRemote;
            }

            public java.lang.String getInterfaceDescriptor() {
                return DESCRIPTOR;
            }

              /*
            2.该方法运行在Client端。当客户端调用该方法时，
            (1)创建Parcel类型的_data、_reply。
            (2)将DESCRIPTOR、参数写入_data。
            (3)调用mRemote.transact发起RPC，该线程被挂起，等待服务端调用onTransact
            (4)当RPC返回时，唤醒当前线程，并从_reply取出结果_result并返回。
            */
            @Override
            public java.util.List<aidl.Book> getBooks(int id) throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                java.util.List<aidl.Book> _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    _data.writeInt(id);
                    mRemote.transact(Stub.TRANSACTION_getBooks, _data, _reply, 0);//执行transact后挂起等待返回
                    //4.Client从_reply中获取到结果
                    _reply.readException();
                    _result = _reply.createTypedArrayList(aidl.Book.CREATOR);
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }
        }

 
    }

    public java.util.List<aidl.Book> getBooks(int id) throws android.os.RemoteException;
}
```

注意：Client和Server符合跨进程通信才会会触发`Binder.transact`和`onTransact`方法。否则这里的Stub也仅仅是作为普通的java类。

## 调用流程

<img src="pic\image-20210522175321171.png" alt="image-20210522175321171" style="zoom:50%;" />

1. Client通过获取Server代理的接口（代理接口中定义的方法必须和Service中定义的方法相同）对Server进行调用。
2. Client调用代理接口，传递的参数会被打包成一个Data（Parcel对象），如果该接口有返回值则提供一个空的Reply（Parcel对象）。之后调用remote.transact()
3. 将Data写入到内核中的Binder驱动，Binder驱动唤醒Service。
4. Service校验解包数据，之后调用onTransact()，将处理后的数据写入Reply，返回给Binder驱动
5. Client发送请求之后处于挂起状态，等待Service返回结果后被唤醒



![image-20210522184419452](pic\image-20210522184419452.png)

### 注册服务

![image-20210522184446666](pic\image-20210522184446666.png)

### 获取服务

![image-20210522184524038](pic\image-20210522184524038.png)

### 使用服务

![image-20210522184543284](pic\image-20210522184543284.png)

## Binder架构模型

![image-20210523180426137](pic\image-20210523180426137.png)