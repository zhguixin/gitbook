#### Android的进程间通信的异步方式

在通过Binder进行跨进程调用时，有同步调用和异步调用之分：

- 同步调用，服务端在未将处理结果返回调用的客户端时，客户端会一直阻塞；
- 异步调用，客户端跨进程向服务端发起调用后，就继续执行，不会阻塞。

#### 异步调用实现

实现方式，在AIDL文件中通过关键字`oneway`定义异步远程调用：

```java
// 异步接口
oneway interface IAsynchronousInterface {
  void method1();
  void method2();
}

// 异步方法
interface IAsynchronousInterface {
  oneway void method1();
  void method2();
}
```

比如，下面这个例子通过跨进程异步调用，得到服务端的线程名，先定义AIDL接口如下：

```java
interface IAsynchronous1 {
  oneway void getThreadNameSlow(IAsynchronousCallback callback);
}

// 回调接口
interface IAsynchronousCallback {
	void handleResult(String name);
}
```

在服务端进程，实现该接口：

```java
IAsynchronous1.Stub mIAsynchronous1 = new IAsynchronous1.Stub() {
  @Override
  public void getThreadNameSlow(IAsynchronousCallback callback)
    throws RemoteException {
    	// Simulate a slow call
    	String threadName = Thread.currentThread().getName();
		SystemClock.sleep(10000);
		callback.handleResult(threadName);
	}
}
```

在客户端，通过回调的方式得到服务端的处理结果：

```java
private IAsynchronousCallback.Stub mCallback = new IAsynchronousCallback.Stub() {
  @Override
  public void handleResult(String remoteThreadName) throws RemoteException {
    // Handle the callback
    Log.d(TAG, "remoteThreadName = " + remoteThreadName);
    Log.d(TAG, "currentThreadName = " + Thread.currentThread().getName());
  }
}
```

打印的结果是：

> remoteThreadName = Binder_1
>
> currentThreadName = Binder_1