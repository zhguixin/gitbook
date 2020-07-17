### 前言

Binder是Android中实现跨进程通信的一种机制，众所周知，Android基于Linux内核，而Linux提供了许多的跨进程通信机制：管道、socket、共享内存等。为什么Android要选用Binder呢。这个问题可以参考这个链接：

[为什么Android要采用Binder作为IPC机制](https://www.zhihu.com/question/39440766/answer/89210950)

### 基本概念

进程间的数据是相互隔离的，不能直接访问。要实现跨进程通信，就需要一个桥梁，而这个桥梁就需要在内核空间做支撑，用户空间的进程通过系统调用进入到内核空间，来得到其他进程通过系统调用放入到内核空间缓存区的的数据。

> 系统调用主要通过如下两个函数来实现：
>
> ```java
> copy_from_user() //将数据从用户空间拷贝到内核空间
> copy_to_user() //将数据从内核空间拷贝到用户空间
> ```

![](..\images\Binder.JPG)

Linux系统支持动态可加载内核模块机制，在Android系统中，运行在内核空间，负责各个用户进程通过Binder实现跨进程通信的模块叫作Binder驱动（Binder Driver）。

### Bidner实现跨进程通信的原理

说Android为什么使用Binder实现IPC时，都会提到一点：只需要一次数据拷贝，性能上仅次于共享内存。

只需要一次数据拷贝，这要得益于**内存映射** 机制，Linux系统中通过`mmap()` 方法实现内存映射。

内存映射，就是将两块内存地址建立一个映射关系，在其中一块内存地址进行操作，可以直接反应到另一块内存地址上。

Binder驱动借助于`mmap` 实现了**两个映射**：

![](..\images\binder_mmap.jpg)

如上图，从而确保了只有一次数据拷贝。

### Binder通信过程

#### 整体架构

Binder是基于C/S架构的，有四个重要的组成部分：Client、Server、Binder Driver和Service Manager。

* Binder Driver，Binder驱动负责中转，将Client的请求转发到对应的Server中去执行，并将处理结果或数据交还给Client，此外Binder驱动还负责Binder引用计数管理
* Service Manager，负责将Client请求的Binder的描述符转化为具体的Server地址，确保Binder驱动能够转发到具体的Server。如果一个Server想要提供Binder服务，就要先向Service Manager注册。

![](..\images\binder_fw_1.png)

Binder Driver 工作在内核空间，Client、Server、Service Manager工作在用户空间，用户空间向内核空间的系统调用过程是通过`open` 和`ioctl` 这两个方法实现的，并没有用`read` 和`write` 。主要是由于`ioctl` 系统调用接口更加的灵活方便，并且根据Binder和用户程序之间商定的接口协议，实现更便捷的交互。

```java
// fd表示打开的Binder驱动的文件描述符
// cmd表示此次交互的命令包括读写等操作
// arg表示此次交互所携带的参数
ioctl(fd, cmd, arg);
```

#### 通信细节

Client和Server要想实现进程通信，是需要通过Service Manger的，`Service Manager` 是比较特殊的存在。`Service Manager`进程是在**init** 进程解析`init.rc` 后启动的，启动之后和Binder驱动进行通信，通过`ioctl` 发送**BINDER_SET_CONTEXT_MGR** 命令告诉Binder驱动使得自己成为`Service Manager` ，Binder驱动会为它自动创建Binder实体，而Client、Server进程要想得到这个Binder实体的引用，只需要获得**0号引用**即可，表示这是预先配置好的。

Client和Server一次完整的通信过程：

1. Server进程通过获得Service Manager的Binder实体引用（即0号引用），向Service Manager注册Binder实体和名字，表明可以对外服务；
2. Client进程通过获得Service Manager的Binder实体引用（即0号引用），向Service Manager请求名字为某个特定服务的Binder引用；
3. Service Manager通过服务的名字得到Binder引用，将Binder引用返回给Client进程；
4. Client获得Server的Binder引用后，即可与Server进程通信了

### 从代码层面实现Binder通信

从上面分析可知，Client要想和Server进行通信，需要拿到Server进程中Binder实体的引用，这个引用具体是什么呢？其实就是Binder实体的代理——BinderProxy，它和Binder实体有一模一样的方法，但是这些方法并没有执行具体任务的能力，这些方法只需要把请求参数交给驱动，驱动中也存在一个BinderProxy，通过驱动调用到真正的Binder实体执行具体任务。

从一个实例入手，实现功能：Client向Server端发送两个整型参数，Server端计算两个整型参数的和，然后返回给Client。

1. 编写`IRemoteServer`接口继承`android.os.IInterface` 接口：

   ```java
   import android.os.IInterface;
   
   public interface IRemoteServer extends IInterface {
       public int sum(int a, int b);
   }
   ```

2. 实现可跨进程调用的Server端：RemoteServer，继承Binder类

   ```java
   import android.os.Parcel;
   import android.os.RemoteException;
   
   public class RemoteServer extends Binder implements IInterface {
       
       public RemoteServer() {
           this.attachInterface(this, "Server");
       }
       
       @Override
       public boolean onTransact(int code, Parcel data, Parcel reply, int flags) throws RemoteException {
           // 根据code的值不同，确定调用哪个业务逻辑代码
           switch (code) {
               case CODE：
           		int arg1 = data.readInt();
           		int arg2 = data.readInt();
           		// 调用Server端的具体业务执行代码（sum求和函数），并将处理结果返回
           		repy.writeInt(sum(arg1, arg2));
                   return true;
           }
   
           return super.onTransact(code, data, reply, flags); 
       }
       
       public int sum(int a, int b) {
           return a + b;
       }
       
       // 实现IInterface接口中的唯一方法
       @Override
       public IBinder asBinder() {
           return this;
       }
   }
   ```

3. 实现客户端进程代码，需要获得Server端的Binder引用

   ```java
   import android.os.IBinder;
   import android.os.RemoteException;
   import android.os.Parcel;
   import android.os.IInterface;
   
   public class Client extends Binder{
       private IBinder mRemote;
       private static fianl int CODE = 1;
       // 通过构造函数传入远程Server端Binder引用
       Client(IBinder obj) {
           IInterface iin = queryLocalInterface("Server");
           if (iin == null) {
               mRemote = obj
           }
       }
       
       public int sum（int a, int b) throws android.os.RemoteException {
           Parcel data = Parcel.obtain();
           Parcel reply = Parcel.obtain();
           int result;
           try {
               data.writeInt(a);
               data.writeInt(b);
               // 通过Binder引用：mRemote，触发进程间通信调用
               mRemote.transact(CODE, data, null, IBinder.FLAG_ONEWAY);
               reply.readException();
               result = reply.readInt();
           } finally {
               data.recycle();
           }
           return result;
       } // end sum
       
   }
   ```

以上代码都书写完了，我们在调用Client的代码时，怎么向构造函数传递`IBider` 呢，也就是说如何获取Server端的Binder引用呢。

其实，这就要借助于Service Manager了，先来看看Android Framework层的`WindowManagerGlobal`类中如何获得WMS的Binder引用的：

```java
public static IWindowManager getWindowManagerService() {
    synchronized (WindowManagerGlobal.class) {
        if (sWindowManagerService == null) {
            // 关注这句话
            sWindowManagerService = IWindowManager.Stub.asInterface(
                ServiceManager.getService("window"));
            try {
                sWindowManagerService = getWindowManagerService();
                ValueAnimator.setDurationScale(
                    sWindowManagerService.getCurrentAnimatorScale());
            } catch (RemoteException e) {
                throw e.rethrowFromSystemServer();
            }
        }
        return sWindowManagerService;
    }
}
```

代码中：

```java
sWindowManagerService = IWindowManager.Stub.asInterface(
    ServiceManager.getService("window"));
```

由此可知WMS已经提前向ServiceManager中注册了自己的Binder实体，并且名字为：`window`。因此，我们也要向ServiceManager注册我们Server端的Binder实体，但是ServiceManager是**hide**的，非系统APP是无法调用的，当然你可以借助于反射，不过Android P之后提出了限制API，一定程度上对这种通过反射实现的"黑科技"造成一定程度上的影响。:cry:


>在SystemServer类中的`startOtherServices`方法中注册WMS服务
>  ```java
> wm = WindowManagerService.main(context, inputManager,
>          mFactoryTestMode != FactoryTest.FACTORY_TEST_LOW_LEVEL,
>          !mFirstBoot, mOnlyCore);
> // Context.WINDOW_SERVICE的值为"window"         
> ServiceManager.addService(Context.WINDOW_SERVICE, wm);
>  ```

那就没有办法了吗？那只好乖乖用**Service**了，在服务端的`onBind`返回Server端自己的Binder实体，客户端通过`bindService`方法实现Service绑定。

```java
// 服务端，运行在另一个进程，作为后台服务
Class RemoteService extends Service{
  
  	@Override
    public IBinder onBind(Intent intent) {
        return new RemoteServer(); // 返回Binder实体
    }
}

// 客户端
Intent intent = new Intent();
ComponentName componentName = new ComponentName("com.remote.service",
                                                "com.remote.service.RemoteService");
intent.setComponent(componentName);
bindService(intent, serviceConnection, Context.BIND_AUTO_CREATE));

private ServiceConnection serviceConnection = new ServiceConnection() {

  @Override
  public void onServiceConnected(ComponentName name, IBinder service) {
    // 在服务连接上的时候，会携带Binder引用过来，省去了自己向ServiceManager注册的麻烦
    Log.d(TAG, "onServiceConnected");
    mRemoteService = IIntentReceiver.Stub.asInterface(service);
  }

  @Override
  public void onServiceDisconnected(ComponentName name) {
    Log.i(TAG, "Service Disconnected");
    mRemoteService = null;
  }
};
```



### 源码分析

#### Java层的Binder

在Java层书写一个通过Binder实现的进程间通信，是需要写很多样板代码的，因此Android提供了**AIDL**这种技术来实现进程间通信。通过分析由AIDL生成的代码，有四个关键类：

- **IBinder** : IBinder 是一个接口，代表了一种跨进程通信的能力。只要实现了这个借口，这个对象就能跨进程传输。
- **IInterface** : IInterface 代表的就是 Server 进程对象具备什么样的能力（能提供哪些方法，其实对应的就是 AIDL 文件中定义的接口）
- **Binder** : Java 层的 Binder 类，代表的其实就是 Binder 本地对象。BinderProxy 类是 Binder 类的一个内部类，它代表远程进程的 Binder 对象的本地代理；这两个类都继承自 IBinder, 因而都具有跨进程传输的能力；实际上，在跨越进程的时候，Binder 驱动会自动完成这两个对象的转换。
- **Stub** : AIDL 的时候，编译工具会给我们生成一个名为 Stub 的静态内部类；这个类继承了 Binder, 说明它是一个 Binder 本地对象，它实现了 IInterface 接口，表明它具有 Server 承诺给 Client 的能力；Stub 是一个抽象类，具体的 IInterface 的相关实现需要开发者自己实现





#### C++层的Binder

不仅Java层需要进程间通信，C++侧同样需要进程间通信。

Android借助于Java中的额JNI技术也实现了Java层与C++层（native层）两侧的代码。在Java层的Binder中的不少方法就直接调用了native中的方法。很有必要看一看Native侧的实现。



#### ServiceManager分析

ServiceManager与Binder驱动的通信

ServiceManager提供的业务主要包括：

```java
android/os/IServiceManager.java

public IBinder getService(String name) throws RemoteException;
public IBinder checkService(String name) throws RemoteException;
public void addService(String name, IBinder service, boolean allowIsolated)
```

在ServiceManager中获得IServiceManager的Binder引用：

```java
private static IServiceManager getIServiceManager() {
    if (sServiceManager != null) {
        return sServiceManager;
    }

    // Find the service manager
    sServiceManager = ServiceManagerNative.asInterface(BinderInternal.getContextObject());
    return sServiceManager;
}
```

`BinderInternal.getContextObject()` 的是一个native方法，追踪到native层。

```c++
base\core\jni\android_util_Binder.cpp

// 注册JNI方法
static const JNINativeMethod gBinderInternalMethods[] = {
     /* name, signature, funcPtr */
    { "getContextObject", "()Landroid/os/IBinder;", (void*)android_os_BinderInternal_getContextObject },
	// ...
};

// 实际上调用了ProceState的getContextObject方法
static jobject android_os_BinderInternal_getContextObject(JNIEnv* env, jobject clazz)
{
    sp<IBinder> b = ProcessState::self()->getContextObject(NULL);
    return javaObjectForIBinder(env, b);
}
```

`ProcessState` 位于：`native\libs\binder\ProcessState.cpp`  中，是个单例类。

```c++
sp<ProcessState> ProcessState::self()
{
    Mutex::Autolock _l(gProcessMutex);
    if (gProcess != NULL) {
        return gProcess;
    }
    // 构造函数，传入Binder驱动的设备字符
    gProcess = new ProcessState("/dev/binder");
    return gProcess;
}

ProcessState::ProcessState(const char *driver)
    : mDriverName(String8(driver))
    , mDriverFD(open_driver(driver)) // 打开binder驱动
    , mVMStart(MAP_FAILED)
    // ...
{
    if (mDriverFD >= 0) {
        // mmap the binder, providing a chunk of virtual address space to receive transactions.
        // 通过mmap()这个系统调用，在内核中开辟一个区域，以便存放接收到的IPC数据
        mVMStart = mmap(0, BINDER_VM_SIZE, PROT_READ, MAP_PRIVATE | MAP_NORESERVE, mDriverFD, 0);
        if (mVMStart == MAP_FAILED) {
            // *sigh*
            ALOGE("Using /dev/binder failed: unable to mmap transaction memory.\n");
            close(mDriverFD);
            mDriverFD = -1;
            mDriverName.clear();
        }
    }
}
```

调用`getContextObject` 方法，传入NULL

```c++
sp<IBinder> ProcessState::getContextObject(const sp<IBinder>& /*caller*/)
{
    // 调用该方法，相当于：return new BpBinder(0);
    // 函数getStrongProxyForHandle的参数handle为0，有着特殊的意义，binder驱动根据这个handle来找到对应的Service，handle为0就是特指serivce_manager
    return getStrongProxyForHandle(0);
}
```

至此，我们可以知道，在`ServiceManagerNative` 类中传递给`asInterface` 方法的参数就是**BpBinder** 对象。

调用业务函数`addService` 就是调用到了`ServiceManagerProxy` 类中的`addService`方法。ServiceManagerProxy的成员变量`mRemote` 指向**BpBinder** 对象。

```java
public void addService(String name, IBinder service, boolean allowIsolated)
    throws RemoteException {
    Parcel data = Parcel.obtain();
    Parcel reply = Parcel.obtain();
    data.writeInterfaceToken(IServiceManager.descriptor);
    data.writeString(name);
    data.writeStrongBinder(service);
    data.writeInt(allowIsolated ? 1 : 0);
    // 实际上调用到BpBinder，具体实现在IPCThreadState中
    mRemote.transact(ADD_SERVICE_TRANSACTION, data, reply, 0);
    reply.recycle();
    data.recycle();
}
```

在`native\libs\binder\IPCThreadState.cpp` 中通过ioctl与binder驱动进行通信。关键代码：

```c++
status_t IPCThreadState::transact(int32_t handle,
                                  uint32_t code, const Parcel& data,
                                  Parcel* reply, uint32_t flags) {
    if (err == NO_ERROR) {
        err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, NULL);
    }
    
    if ((flags & TF_ONE_WAY) == 0) {
        if (reply) {
            err = waitForResponse(reply);
        } else {
            Parcel fakeReply;
            err = waitForResponse(&fakeReply);
        }
    } else {
        err = waitForResponse(NULL, NULL);
    }
    return err;
}

// waitForResponse实现方法
status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult) {
    uint32_t cmd;
    int32_t err;
    
    while (1) {
        // talkWithDriver，通过ioctl系统调用与内核空间通信
        if ((err=talkWithDriver()) < NO_ERROR) break;
        
        cmd = (uint32_t)mIn.readInt32();
        switch (cmd) {
            case BR_TRANSACTION_COMPLETE://...
            case BR_DEAD_REPLY: // ...
            // ...
            // 没有相应的命令码，立即得到service端的响应，退出死循环
            default:
                err = executeCommand(cmd);
                if (err != NO_ERROR) goto finish;
                break;
    }
finish:
    if (err != NO_ERROR) {
        if (acquireResult) *acquireResult = err;
        if (reply) reply->setError(err);
        mLastError = err;
    }

    return err;
}

// talkWithDriver方法的实现
status_t IPCThreadState::talkWithDriver(bool doReceive) {
    binder_write_read bwr;
    // 填充bwr数据结构
    // ...
    
    do {
// 利用宏开关，来定义访问内核空间使用的方式，非Android系统，直接定义为非法访问    
#if defined(__ANDROID__)
        // 系统调用：ioctl(文件描述符,ioctl命令,数据类型)
        // BINDER_WRITE_READ 命令表示在进程间发送接收Binder IPC数据
        if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0)
            err = NO_ERROR;
        else
            err = -errno;
#else
        err = INVALID_OPERATION;
#endif
        if (mProcess->mDriverFD <= 0) {
            err = -EBADF;
        }
    } while(err == -EINTR)
    // ...
}
```

其中`waitForResponse` 方法等待响应。

* mOut承担向Binder驱动写入数据
* mIn从Binder驱动读取数据

#### Binder线程

`ProcessState` 调用`startThreadPool` 方法创建一个线程池，在线程池创建一个线程，创建的线程与binder驱动进行通信。

> startThreadPool(); -->> spawnPooledThread(isMain=true); -->> new PoolThread(isMain); -->> IPCThreadState.joinThreadPool(mIsMain) -->> IPCThreadState.talkWithDriver(false);

#### Binder驱动

在Binder驱动中，Binder本地对象的代表是一个叫做`binder_node`的数据结构，Binder代理对象是用`binder_ref`代表的。