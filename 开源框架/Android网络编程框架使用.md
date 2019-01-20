### Android网络编程框架使用

#### 1、OkHttp使用简介

```java
public static void testOkhttp() {
    OkHttpClient mOkHttpClient = new OkHttpClient();
    Request request = new Request.Builder()
        .url("http://www.baidu.com/")
        .build();
    okhttp3.Call call = mOkHttpClient.newCall(request);
    call.enqueue(new okhttp3.Callback() {
        @Override
        public void onFailure(okhttp3.Call call, IOException e) {
            Log.d(TAG, "onFailure: no connection!!!");
        }

        @Override
        public void onResponse(okhttp3.Call call, okhttp3.Response response) throws 
            															IOException {
            Log.d(TAG, "onResponse: " + response.body());// 异步请求，非UI线程
        }
    });
}
```

如上代码构造了一个简单的get请求：

- 首先，构造一个Request对象，表示一个请求。我们可以通过Request.Builder来设置更多的信息，比如：`header`、`method`等；
- 通过OKHttpClient的实例对象方法`newCall`，得到一个Call对象。
- 通过Call对象的`enqueue`(**异步执行**)、或者`execute`(**同步执行**)，来执行刚才传递的请求。

在构造OkHttpClient对象时，我们可以添加一些配置，比如增加拦截器、设置连接超时等：

```java
OkHttpClient mOkHttpClient = new OkHttpClient.Builder()
  		.addInterceptor(new MyRequestInterceptorImpl())
  		.addInterceptor(new HttpLoggingInterceptor())
  		.readTimeout(30L, TimeUnit.SECONDS)
  		.writeTimeout(30L, TimeUnit.SECONDS)
  		.connectTimeout(30L, TimeUnit.SECONDS)
  		.connectionPool(new ConnectionPool(2, 1L, TimeUnit.SECONDS))
  		.retryOnConnectionFailure(true)
  		.build();
```

实现自定义的拦截器，我们可以通过自定义Interceptor来实现很多操作：打印日志、缓存、重试等等，

拦截器的功能就是在**请求发出前和接收到响应后做做手脚**。

要实现自己的拦截器需主要有以下步骤：

- 需要实现`Interceptor`接口，并复写`intercept(Chain chain)`方法，返回response
- 由`chain.request();`拿到原始请求originalRequest ，通过newBuilder方法重新构造一个新的request
- 由`chain.proceed(originalRequest );`拿到原始的响应originalResponse ，通过newBuilder()方法重新构造response，返回该response—— `originalResponse.newBuilder().build()`

```java
class MyRequestInterceptorImpl implements Interceptor {
  @Override
  public Response intercept(Interceptor.Chain chain) throws IOException { // -----1
    // 接口在回调时，会接收一个Chain类型的参数，这个参数保存了Request和Response的相关数据
    Request originalRequest = chain.request(); // -----2
    
    String cacheControl = originalRequest.cacheControl().toString();
    
    Request.Builder requestBuilder = originalRequest.newBuilder()
      	.header("Accept", "application/json")
      	.method(originalRequest.method(), originalRequest.body());
    Request request = requestBuilder.build(); // -----3

    Response originalResponse = chain.proceed(request); // -----4
    
    Response.Builder responseBuilder = originalResponse.newBuilder()
      	// Cache control设置缓存
      	.header("Cache-Control", cacheControl);

    return responseBuilder.build(); // -----5
  }
}
```

该拦截器实现的功能是在请求发出前和接收到响应后，分别打印log。其中：

```java
Response response=chain.proceed(request); 
```

将`Request`和`Response`的拦截串联起来了。

OKHttp底层是通过Socket的方式发起远程连接，通过Okio库进行字节流的读写。并且使用连接池，同一主机共用一个Socket连接。

关于拦截器的详细讲解，可以参考：

[okhttp--interceptors](https://github.com/square/okhttp/wiki/Interceptors)

[okhttp拦截器的实现原理](http://www.cnblogs.com/LuLei1990/p/5534791.html)

#### 2、Retrofit使用简介

```java
    public static void testRetrofit() {
        Retrofit retrofit = new Retrofit.Builder()
                .baseUrl("http://rms.ztems.com/ZteLockScreenAPI/")
                .build();
        IBook books = retrofit.create(IBook.class);
        Call<Book> call = books.getBook(1364);
        call.enqueue(new Callback<Book>() {
            @Override
            public void onResponse(Call<Book> call, Response<Book> response) {
                Log.d(TAG, "onResponse: " + response.body());
            }

            @Override
            public void onFailure(Call<Book> call, Throwable t) {

            }
        });
    }
```

使用Retrofit需要我们去定义一个接口：

```java
public interface IBook {
    // http://rms.ztems.com/ZteLockScreenAPI/praiseRes.json?resid=id
    @GET("praiseRes.json")
    Call<Book> getBook(@Query("resid") int id);
  	
  	// 支持动态URL访问
  	// http://rms.ztems.com/ZteLockScreenAPI/users/{user}/repos 
  	@GET("users/{user}/repos")
  	Call<List<Books>> getBooks(@Path("user") String user);
}
```

然后可以通过调用`retrofit.create(IBook.class);`方法，得到一个接口的实例，最后通过该实例执行我们的操作：

```java
Call<Book> call = books.getBook(1364); // 执行接口中已封装好的某一个请求
call.enqueue(new Callback<Book>() {}); // 类似于使用OkHttp，我们通过enqueue来执行这个请求
```

此外，构造Retrofit对象时，我们可以通过`Retrofit.Builder`设置更多的参数：

```java
  Retrofit retrofit = new Retrofit.Builder()
    .baseUrl("http://rms.ztems.com/ZteLockScreenAPI/")
    // GsonConverterFactory完成对服务器返回的json字符串到对象的转化
    .addConverterFactory(GsonConverterFactory.create()) 
    // 配置自定义的OkHttpClient，可以在OkHttpClient中添加拦截器、超时等
    .callFactory(mOkHttpClient)
    .build();
```



#### 3、Retrofit源码简析

首先，我们知道，Retrofit可以通过定义的接口来拿到接口的实例：

```java
retrofit.create(IBook.class);
```

此处使用了动态代理，看一下，create的源码(抽取关键代码)：

```java
public <T> T create(final Class<T> service) {
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
            @Override 
            public Object invoke(Object proxy, Method method, Object... args) throws Throwable {
              // ...
              // 根据我们的method(也就是我们定义的接口类中的方法）将其包装成ServiceMethod
              ServiceMethod<Object, Object> serviceMethod = 
                	(ServiceMethod<Object, Object>) loadServiceMethod(method);
              // 通过ServiceMethod和方法的参数构造retrofit2.OkHttpCall对象
              OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
              // 将OkHttpCall进行代理包装
              return serviceMethod.callAdapter.adapt(okHttpCall);
       });
  }
```

构建Retrofit时，指定了：

```java
addCallAdapterFactory(RxJava2CallAdapterFactory.create())
```

所以在`serviceMethod.callAdapter.adapt(okHttpCall)`  语句中，调用的是`RxJava2CallAdapterFactory` 中的`adapt` 方法。

如果默认没有添加`CallAdapterFatory` ，使用的是：**ExecutorCallAdapterFactory** 。在构造好的OkHttpCall通过线程池执行request时，会通过Handler切换到UI线程。所以，默认的`onResponse` 和 `onFailure` 在UI线程。

ServiceMethod主要用于将我们接口中的方法转化为一个`Request对象` ，根据我们的接口返回值确定responseConverter，解析我们方法上的注解拿到初步的url，解析我们参数上的注解拿到构建RequestBody所需的各种信息，最终调用toRequest的方法完成Request的构建。

create最后返回的是对OkHttpCall的封装，通过分析源码可知是：ExecutorCallbackCall。

```java
Call<Book> call = books.getBook(1364);
call.enqueue(new Callback<Book>() {});
```

通过解析得到Call对象执行enqueue，执行的就是`ExecutorCallbackCall.enqueue`方法，在这个方法内除了将onResponse和onFailure回调到UI线程，主要的操作还是由delegate完成的，这个delegate实际上就是OkHttpCall对象，其实就是执行了OkHttp中的enqueue方法：`okhttp3.Call.enqueue` 。

Retrofit实际上是为了更方便的使用Okhttp，因为Okhttp的使用就是构建一个Call，而构建Call的大部分过程都是相似的，而Retrofit正是利用了代理机制带我们动态的创建Call，而Call的创建信息就来自于你的注解。并且还可以根据配置Adapter等等对网络请求进行相应的处理和改变，这种插件式的解耦方式也提供了很大的扩展性。

参考：[深入理解Retrofit](http://blog.csdn.net/lmj623565791/article/details/51304204)