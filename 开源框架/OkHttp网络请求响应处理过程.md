在上一篇文章，以一次异步请求搭建了OKHttp整个网络处理框架后，来看一看OkHttp与网络处理具体相关的逻辑代码。

在不考虑，用户自定义拦截器的情况下，我们只需要关注两个主要的拦截器：`ConnectInterceptor` 和` CallServerInterceptor` 。

先看一下`ConnectInterceptor` 拦截器的处理过程：

```java
  @Override 
  public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Request request = realChain.request();
    // StreamAllocation在RetryAndFollowUpInterceptor被实例化
    StreamAllocation streamAllocation = realChain.streamAllocation();

    boolean doExtensiveHealthChecks = !request.method().equals("GET");
    // 得到HttpCodec对象：HttpCodec是一个接口，有两个实现类：Http1Codec和Http2Codec
    HttpCodec httpCodec = streamAllocation.newStream(client, doExtensiveHealthChecks);
    // 得到RealConnection对象
    RealConnection connection = streamAllocation.connection();
	// 交给下一个拦截器
    return realChain.proceed(request, streamAllocation, httpCodec, connection);
  }
```

下一个拦截器为` CallServerInterceptor` ：

```java
@Override public Response intercept(Chain chain) throws IOException {
    // 得到上一个拦截器传过来的对象
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    HttpCodec httpCodec = realChain.httpStream();
    StreamAllocation streamAllocation = realChain.streamAllocation();
    RealConnection connection = (RealConnection) realChain.connection();
    Request request = realChain.request();
    // 写入请求信息
    httpCodec.writeRequestHeaders(request);
    
    httpCodec.finishRequest();
    
    if (responseBuilder == null) {
        responseBuilder = httpCodec.readResponseHeaders(false);
    }
    
    // 建造者模式慢慢构造response
    Response response = responseBuilder
        .request(request)
        .handshake(streamAllocation.connection().handshake())
        .sentRequestAtMillis(sentRequestMillis)
        .receivedResponseAtMillis(System.currentTimeMillis())
        .build();
    
    int code = response.code();
    // code==101表示http版本需要升级，升级后的body随后发送，这里OkHttp给了用户一个空的响应体
    if (forWebSocket && code == 101) {
      response = response.newBuilder()
          .body(Util.EMPTY_RESPONSE)
          .build();
    } else {
      response = response.newBuilder()
          .body(httpCodec.openResponseBody(response))
          .build();
    }
    // 返回构造好的响应response
    return response;
}
```

以上代码省略了部分判断逻辑，从上可以看出多处实现需要调用`HttpCodec`里的方法 。

`HttpCodec`是一个接口具体的实现类有两个：Http1Codec和Http2Codec。分别对应于Http的两个版本：Http1.x于Http2.0。

在HttpCodec中，与Http2Connection、Http2Stream等来处理网络相关操作，大致就是创建Socket，通过okio来做IO处理，最后得到响应回调给用户。

*有机会再分析了。。。*