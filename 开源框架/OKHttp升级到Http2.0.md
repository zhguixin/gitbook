OKHttp升级到Http2.0

建立在TLS/SSL握手，OKHttpClient加入SSLSocketFactory

```java
OkHttpClient client = new OkHttpClient();
client.sslSocketFactory(sslSocketFactory, trustCerts);
```

OKhttp WebSocket

```java
// 继承抽象类: WebSocketListener，监听websocket连接状态和消息收发
EchoWebSocketListener listener = new EchoWebSocketListener();
Request request = new Request.Builder()
        .url("ws://echo.websocket.org")
        .build();
OkHttpClient client = new OkHttpClient();
client.newWebSocket(request, listener);
```

