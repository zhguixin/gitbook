通过构建好的**OkHttpCall**向网络发起请求，提供了两种方式：

* enqueue，异步的方式
* execute，同步的方式

接下来，分析下异步的方式。

```java
OkHttpClient mOkHttpClient = new OkHttpClient();
Request request = new Request.Builder()
    .url("http://www.baidu.com/")
    .build();
okhttp3.Call call = mOkHttpClient.newCall(request);
call.enqueue(new okhttp3.Callback() {});
```

通过OkHttpClient的`newCall` 方法，构造出`RealCall` 的实例：

```java
public Call newCall(Request request) {
    return new RealCall(this, request, false /* for web socket */);
}
```

调用`RealCall` 的enqueue方法：

```java
public void enqueue(Callback responseCallback) {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    // dispatcher允许用户通过Builder的方式传入，默认使用Dispatcher
    client.dispatcher().enqueue(new AsyncCall(responseCallback));
}
```

调用`Dispatcher` 类的enqueue方法：

```java
synchronized void enqueue(AsyncCall call) {
    // 最大请求数为64，访问同一主机的最大请求数为5
    if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
        runningAsyncCalls.add(call);
        // 创建线程池
        executorService().execute(call);
    } else {
        // 超过了最大请求数，放到readyAsyncCalls这一队列中等待被处理，先进先出
        readyAsyncCalls.add(call);
    }
}
```

OkHttpClient创造的线程池，构造函数如下：

```java
  public synchronized ExecutorService executorService() {
    if (executorService == null) {
      executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
          new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
    }
    return executorService;
  }
```

核心线程数为0，非核心线程空闲存活时间为60秒，使用的阻塞队列为：`SynchronousQueue` ，该阻塞队列不会保留请求，收到请求便会创建线程进行处理，因此一般要将最大线程数设置为：`Integer.MAX_VALUE` 。

在返回到`RealCall` 的enqueue方法中：

```java
public void enqueue(Callback responseCallback) {
    // ...
    client.dispatcher().enqueue(new AsyncCall(responseCallback));
}
```

也就是线程池执行的是`AsyncCall` 的实例。`AsyncCall` 是RealCall的一个内部类：

```java
final class AsyncCall extends NamedRunnable {
    // 构造函数，responseCallback对应于我们传入进去的Callback实例
    AsyncCall(Callback responseCallback) {
        super("OkHttp %s", redactedUrl());
        this.responseCallback = responseCallback;
    }
}
```

其中，NamedRunnable是一个抽象类并继承了Runnable接口，在`run` 方法中会设置线程名，调用抽象方法`execute` 方法。因此，添加到线程池的请求，默认执行的方法由`run` 方法变到`execute` 方法。

在AsyncCall这个内部类的`execute` 方法：

```java
@Override protected void execute() {
    boolean signalledCallback = false;
    try {
        // 遍历OkHttp中添加的拦截器，先处理外部定义的拦截器，再处理OkHttp自身定义的拦截器
        Response response = getResponseWithInterceptorChain();
        if (retryAndFollowUpInterceptor.isCanceled()) {
            signalledCallback = true;
            // 回调到应用程序
            responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
        } else {
            signalledCallback = true;
            // 回调到应用程序
            responseCallback.onResponse(RealCall.this, response);
        }
    } catch (IOException e) {
        if (signalledCallback) {
            // Do not signal the callback twice!
            Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
        } else {
            // 回调到应用程序
            responseCallback.onFailure(RealCall.this, e);
        }
    } finally {
        // 将这个执行完的请求移除，这样后面等待的请求可以继续执行
        client.dispatcher().finished(this);
    }
}
```

OkHttp的精华就在这些拦截器的处理中了，将请求`request` 一步步的交给拦截器处理，最底层的拦截处理完之后，得到响应`response` 再向上一层层的返回到拦截器，最终回调给用户。这就是典型的责任链模式。

```java
  Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    // 先添加用户自定义的拦截器
    interceptors.addAll(client.interceptors());
    // 该拦截器负责失败重试以及重定向
    interceptors.add(retryAndFollowUpInterceptor);
    // 把用户的请求转换为发送到服务器的请求
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    // 缓存管理
    interceptors.add(new CacheInterceptor(client.internalCache()));
    // 负责与服务器建立连接，通过Socket以及自己的IO库--okio来处理
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
      interceptors.addAll(client.networkInterceptors());
    }
    // 负责向向服务器发送请求数据、从服务器读取数据
    interceptors.add(new CallServerInterceptor(forWebSocket));

    Interceptor.Chain chain = new RealInterceptorChain(
        interceptors, null, null, null, 0, originalRequest);
    // 调用RealInterceptorChain的proceed方法
    return chain.proceed(originalRequest);
  }
```

在`RealInterceptorChain` 类的`proceed` 方法中，从**interceptors** 列表的第一个元素(index=0)开始取拦截器，将请求request交给这个拦截器，调用`intercept` 方法：

```java
public Response proceed(Request request) throws IOException {
    return proceed(request, streamAllocation, httpCodec, connection);
}

public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
      RealConnection connection) throws IOException {
    // ...
    // 新建一个RealInterceptorChain实例，作为拦截器intercept方法的入参
    RealInterceptorChain next = new RealInterceptorChain(
        interceptors, streamAllocation, httpCodec, connection, index + 1, request);
    // 取出一个拦截器列表中的第一个拦截器
    Interceptor interceptor = interceptors.get(index);
    Response response = interceptor.intercept(next);
    
    return response;
}
```

在调用拦截器的`intercept` 方法中，又会调用`RealInterceptorChain` 的`proceed` 方法，并根据需要构造`streamAllocation` 、`httpCodec` 和`connection` 。如此一来，就将request一层层传递下去，处理到最后一个拦截器——**CallServerInterceptor** ，获得响应response后，又一层层将响应回传过来。（有点像函数的递归）

这种做法就有点TCP/IP分层的概念，每一层（对应每个拦截器）只需要处理自己关心的那一部分就可。