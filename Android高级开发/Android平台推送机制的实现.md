Android平台推送机制的实现

Android平台想要接收服务器的推送通知，就要保持一种长连接。谷歌有一套自己的推送服务GCM（Google Cloud Messaging），由于众所周知的原因，大陆是没法使用的。第三方平台的SDK，比如：友盟推送、小米推送、百度推送、极光推送等，也是一个很不错的可选方案。

本着知其然知其所以然的思想，来实现、研究一下，Android推送机制。

#### 基本原理

由于要建立TCP长连接，所以就不能使用HttpUrlConnection或者HttpClient了，它们是更上层的，遵循Http协议的。无法建立长连接。我们可以使用Android（Java）提供的对TCP协议封装的网络套接字Socket实现。

一般来说，需要包括下面几个功能：

* 与服务端建立TCP连接
* 发送数据到服务端
* 从服务端接收数据
* 实现心跳包

##### 与服务端建立连接

```java
mSinglePool.execute(new Runnable() {
    @Override
    public void run() {
        doConnectAsync(); // 耗时操作，放在工作线程
    }
});

private void doConnectAsync() {
    try {
        InetSocketAddress socketAddress = new InetSocketAddress(HOST_NAME, PORT);
        mSocket = new Socket();
        mSocket.connect(socketAddress, CONNECT_TIME_OUT);
        mSocket.setKeepAlive(true);// 保持长连接
    } catch (IOException e) {
        e.printStackTrace();
    }
    }
```

##### 发送数据到服务端

上面创建的mSocket，提供了到服务端连接通道，我们可以通过mSocket向服务端发送数据：

```java
private sendToServer(String content) {
    OutputStream outputStream = mSocket.getOutputStream();
    DataOutStream out = new DataOutStream(outputStream);
    short len = (short) content.length;
    // 前两个字节存放要发送数据的长度
    byte b1 = (byte) (0xff & len);
    byte b2 = (byte) (0xff & (len >> 8));
    out.writeByte(b1);
    out.writeByte(b2);
    out.write(content);
    out.flush();
}
```

##### 从服务端读取数据

```java
InputStream inputStream = mSocket.getInputStream()
DataInputStream in = new DataInputStream(inputStream);

byte b1 = in.readByte();
byte b2 = in.readByte();
// 根据这两个字节算出数据长度
```

##### 心跳包的实现

心跳包的作用就是保证了长连接不会断掉，每隔一段时间向服务端发送一组数据，这组数据是要遵循自定义协议的格式。

这个时间间隔不能设计过长，否则NAT超时导致连接断掉；也不能设计过短，否则损耗电量、CPU以及导致网络拥堵。建议设置为4分钟。

参考：[自适应的心跳保活机制](https://juejin.im/entry/5aa6144e51882555731bc3e5)



#### PN-Clinet使用gradle构建

出现的问题：

1. `Notification`使用构建者设计模式创建，`Notification` 的方法：`setLatestEventInfo`已弃用

2. `startService`或者`bindService`要显示声明Intent，在`ServiceManager`中：

   ```java
   Intent intent = new Intent(context, NotificationService.class);
   context.startService(intent);
   ```

3. 读取IMEI号的权限要动态获取。