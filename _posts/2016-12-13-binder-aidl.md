---
layout: post
title:  "从AIDL看Android跨进程通信"
desc: "android跨进程通信"
keywords: "binder"
date: 2016-12-13
categories: [Android]
tags: [AOSP,Frameworks,Binder]
icon: icon-java
---
　　AIDL是Android实现IPC的一种重要的方式，理解它的原理对理解Android进程间通信有很大的帮助。AIDL的定义，已经有很多介绍的文章了，这里就不做详解了。我们直接从实例入手来分析AIDL实现原理。

　　**AIDL的使用**

　　首先需要定义AIDL接口IMyService.aidl：
　　

```java
// IMyService.aidl
package com.chuck.aidldemo;

// Declare any non-default types here with import statements

interface IMyService {
    /**
     * Demonstrates some basic types that you can use as parameters
     * and return values in AIDL.
     */
     String getValue();
}
```
　　定义了getValue（）方法，返回String。我们知道定义好aidl文件后，IDE在编译项目时会自动帮我们生成一个文件IMyService.java文件，稍后会详细介绍。

　　接下来需要定义一个service，这里定义MyService.java
　　

```java
public class MyService extends Service {
    @Override
    public void onCreate() {
        super.onCreate();
        Log.e("myService","onCreate"    );
    }

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return ims;
    }

    IMyService.Stub ims=new IMyService.Stub() {
        @Override
        public String getValue() throws RemoteException {
            return   "hello AIDL";
        }
    };

    @Override
    public void onDestroy() {
        super.onDestroy();
        Log.e("myService","onDestroy");
    }
}
```
　　重写onBind方法，返回IMyService.Stub对象ims，IMyService.Stub是定义在IMyService.java中，其实现了IBinder，所以这里可以在OnBinder中作为返回对象。同时在AIDL中定义getValue方法的真正实现，就是在这里。我们仅仅是返回一个"hello AIDL"字符串。

　　为了实现跨进程，我们还需要在AndroidMenifast.xml文件中设置process=":Remote",这样service就和client不在同一个进程中了：
　　

```xml
<service
            android:name=".MyService"
            android:process=":Remote" />
```

　　最后在定义client，为了方便我们就直接在MainActivity.java实现了：
　　

```java
public class MainActivity extends AppCompatActivity {
    private IMyService ims;
    private String text;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    }

    public void onClick(View view) {
      if (view.getId() == R.id.btn_bind) {
            Intent intent = new Intent(this, MyService.class);
            bindService(intent, sc, BIND_AUTO_CREATE);
        }else if (view.getId()==R.id.btn_unbind){
            unbindService(sc);
        }
    }

    private ServiceConnection sc = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            ims = IMyService.Stub.asInterface(service);
            try {
                text = ims.getValue();
                Log.e("text",text);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {

        }
    };
}
```
　　这和平时bindService使用是一样的，需要一个ServiceConnection实例，将该实例作为bindService参数传入。这样client就可以启动并绑定MyService了。

　　一般我们使用AIDL的基本步骤就是这些，现在我们需要着重分析自动生成的IMyService.java了，如果不知道文件位置，直接在IDE搜索一下就可以了。先把代码贴出来：
　　

```java
/*
 * This file is auto-generated.  DO NOT MODIFY.
 * Original file: /Users/chenkai/Android/codeExcise/aidl/AidlDemo/app/src/main/aidl/com/chuck/aidldemo/IMyService.aidl
 */
package com.chuck.aidldemo;
// Declare any non-default types here with import statements

public interface IMyService extends android.os.IInterface {
    /**
     * Local-side IPC implementation stub class.
     */
    public static abstract class Stub extends android.os.Binder implements com.chuck.aidldemo.IMyService {
        private static final java.lang.String DESCRIPTOR = "com.chuck.aidldemo.IMyService";

        /**
         * Construct the stub at attach it to the interface.
         */
        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }

        /**
         * Cast an IBinder object into an com.chuck.aidldemo.IMyService interface,
         * generating a proxy if needed.
         */
        public static com.chuck.aidldemo.IMyService asInterface(android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (((iin != null) && (iin instanceof com.chuck.aidldemo.IMyService))) {
                return ((com.chuck.aidldemo.IMyService) iin);
            }
            return new com.chuck.aidldemo.IMyService.Stub.Proxy(obj);
        }

        @Override
        public android.os.IBinder asBinder() {
            return this;
        }

        @Override
        public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
            switch (code) {
                case INTERFACE_TRANSACTION: {
                    reply.writeString(DESCRIPTOR);
                    return true;
                }
                case TRANSACTION_getValue: {
                    data.enforceInterface(DESCRIPTOR);
                    java.lang.String _result = this.getValue();
                    reply.writeNoException();
                    reply.writeString(_result);
                    return true;
                }
            }
            return super.onTransact(code, data, reply, flags);
        }

        private static class Proxy implements com.chuck.aidldemo.IMyService {
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

            /**
             * Demonstrates some basic types that you can use as parameters
             * and return values in AIDL.
             */
            @Override
            public java.lang.String getValue() throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                java.lang.String _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    mRemote.transact(Stub.TRANSACTION_getValue, _data, _reply, 0);
                    _reply.readException();
                    _result = _reply.readString();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }
        }

        static final int TRANSACTION_getValue = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
    }

    /**
     * Demonstrates some basic types that you can use as parameters
     * and return values in AIDL.
     */
    public java.lang.String getValue() throws android.os.RemoteException;
}

```
　　代码结构很清晰，先来看一下类图：

　　<img src="http://img.blog.csdn.net/20160723151117802" width="400" height="300">


　　1.IInterface，我们定义的Binder接口都需要继承自它。

　　2.我们定义的IMyService接口继承自接口IInterface。

　　3.IBinder 远程对象的接口。

　　4.Binder实现了IBinder，Android通过Binder进行IPC

　　5.BinderProxy是在定义在Binder.java中的，也是实现了IBinder接口，这里只要知道它是Binder的代理就可以了。我们Client中拿到的就是这个类的对象，之后会有讲。

　　6.IMyService.Stub继承自Binder并且实现了IMyService。也就是说Stub其实是个Binder。还记得吗？在前文MyService中我们生成了一个Stub的实例ims，Stub的本意是存根，这里的用意就是Service的存根，通过它，我们在Service端就拥有了一个Binder。这样我们就可以通过底层的Binder驱动进行跨进程通信了。

　　7.IMyService.Proxy，顾名思义它是一个代理，主要是实现了IMyService接口，客户端通过它发起远程请求。

　　简单的介绍了涉及到的类，下面来分析一下请求的流程。

　　我们知道，在client bindService（这是一个比较复杂的过程，涉及到AMS，本身也是一个跨进程的通信）之后，如果远程服务，这里是MyService，处理完请求之后，通过一系列复杂的操作，client中的onServiceConnected方法将会回调，为了方便我把前边相关的部分代码放在这：
　　

```java
private ServiceConnection sc = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            ims = IMyService.Stub.asInterface(service);
            try {
                text = ims.getValue();
                Log.e("text",text);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {

        }
    };
```
　　第3行当回调onServiceConnected时，会传过来一个IBinder对象service，client就是需要通过它来发起远程请求的，那么service的具体实现到底是Binder还是BinderProxy呢，如果对Binder的机制比较清楚的都会知道其实是BinderProxy的，为了验证这个我们可以调试一下：


<img src="http://img.blog.csdn.net/20160723161623506" width="400">

　　我们得到BinderProxy后会将其传给IMyService.Stub.asInterface(service)，asInterface接受一个IBinder对象作为参数，这里就是刚才说的BinderProxy对象。具体看看asInterface做了什么：
　　

```java
 public static com.chuck.aidldemo.IMyService asInterface(android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (((iin != null) && (iin instanceof com.chuck.aidldemo.IMyService))) {
                return ((com.chuck.aidldemo.IMyService) iin);
            }
            return new com.chuck.aidldemo.IMyService.Stub.Proxy(obj);
        }
```
　　看第5行，因为我们设置了MyService的process属性，也就是说他和client是不在同一个进程的，所以obj.queryLocalInterface(DESCRIPTOR)这个方法返回为空，也就是iin为null，那么将会执行第9行代码，对没错new了一个IMyService.Proxy对象。回到前边onServiceConnected方法中第4行ims就是刚刚生成的IMyService.Proxy对象。第6行ims.getValue()方法执行的就是IMyService.Proxy中的getValue方法：
　　

```java
/**
             * Demonstrates some basic types that you can use as parameters
             * and return values in AIDL.
             */
            @Override
            public java.lang.String getValue() throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                java.lang.String _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    mRemote.transact(Stub.TRANSACTION_getValue, _data, _reply, 0);
                    _reply.readException();
                    _result = _reply.readString();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }
```
　　第7.8行_data,  _reply分别是进行远程调用时的的请求和相应参数。第11行将DESCRIPTOR = "com.chuck.aidldemo.IMyService"作为Token写入 _data.第12行mRemote.transact(Stub.TRANSACTION_getValue, _data, _reply, 0);mRemote是在IMyService.Proxy构造方法中被赋值的，也就是我们在new IMyService.Proxy对象时传进来的BinderProxy对象。所以mRemote.transact(Stub.TRANSACTION_getValue, _data, _reply, 0)的实现应该是在BinderProxy中，这里需要注意一下第一个参数Stub.TRANSACTION_getValue，他是标识需要调用方法的code，在后边会有用到。我们看看他的源码：
　　

```
 public boolean transact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
        Binder.checkParcel(this, code, data, "Unreasonably large binder buffer");
        return transactNative(code, data, reply, flags);
    }
```
　　最终会调用transactNative（）方法，这是一个native方法，真正的实现是用c实现，他是实现在Android源码对应/frameworks/base/core/jni/android_util_Binder.cpp中的static jboolean android_os_BinderProxy_transact函数，将java层的transact转换成c层transact，有兴趣的可以去研究一下。

　　通过底层Binder的一系列操作，最终会回调到远程Stub的onTransact方法，至于怎么调用的，需要去了解Binder机制。这里只要知道会回调onTransact（）方法就好了。不过还是要注意的是，这里的Stub是在远程服务端的，他和client不是在同一个进程中的。他是在远程服务的Binder线程池中。跟进去看onTransact（）方法做了什么处理：
　　

```
 @Override
        public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
            switch (code) {
                case INTERFACE_TRANSACTION: {
                    reply.writeString(DESCRIPTOR);
                    return true;
                }
                case TRANSACTION_getValue: {
                    data.enforceInterface(DESCRIPTOR);
                    java.lang.String _result = this.getValue();
                    reply.writeNoException();
                    reply.writeString(_result);
                    return true;
                }
            }
            return super.onTransact(code, data, reply, flags);
        }
```
　　还记得在transact方法时提到的第一个参数Stub.TRANSACTION_getValue吗，它就是这里onTransact的第一个参数code。所以会执行8-13行代码,第10行java.lang.String _result = this.getValue();很关键这个this是Stub，还记得我们在哪儿有生成它的实例吗？没错就是在MyService中，我们把它作为onBind方法的返回参数。所以这里的this.getValue就是MyService中我们写的那个getValue方法，会返回一个"hello AIDL"的字符串。并把它赋值给_result。在第12行将其写入_reply中。最后在通过Binder的一系列操作，我们将在client中得到这个字符串。

　　总的来说AIDL本身使用比较简单，理解起来也比较简单，但是其内部的Binder机制还是比较复杂的，我们先理解Java层的AIDL对学习Binder也是有帮助的。并且Messager通信是基于AIDL的，理解了AIDL也就理解了Messager。

　　

　　