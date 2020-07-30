### smack库分析

#### 通过XMPPTCPConnection连接XMPP服务器

```java
// 配置连接参数
mXMPPConnectionConfig = XMPPTCPConnectionConfiguration.builder()
    .setUsernameAndPassword(username, password)
    .setXmppDomain(IMXmppConfig.getXmppDomain())
    .setHost(IMXmppConfig.getXmppHost())
    .setResource(xmmppResource)
    .setPort(IMXmppConfig.getXmppPort())
    .setSecurityMode(ConnectionConfiguration.SecurityMode.disabled)
    .setDebuggerEnabled(IMSDK.getSDKOptions().enableLog)
    .setConnectTimeout(15000)
    .build();

// 初始化 XMPPTCPConnection
mXMPPConnection = new XMPPTCPConnection(mXMPPConnectionConfig);
mXMPPConnection.setUseStreamManagement(false);
mXMPPConnection.setUseStreamManagementResumption(false);

// 注册接收数据包的监听
mXMPPConnection.addAsyncStanzaListener(mStanzaListener, packetFilter);
// 注册链路连接状况监听
mXMPPConnection.addConnectionListener(mConnectionListener);

// 连接
if (!mXMPPConnection.isConnected()) {
    mXMPPConnection.connect();
}

// 登录
if (!mXMPPConnection.isAuthenticated()) {
    mXMPPConnection.login();
}

// 发送消息
mXMPPConnection.sendStanza(stanza);
```

> 发送消息的方法定义在`AbstractXMPPConnection`：
>
> ```java
> public void sendStanza(Stanza stanza) throws NotConnectedException, InterruptedException
> ```
>
> 最终会调用到抽象方法：
>
> ```java
> protected abstract void sendStanzaInternal(Stanza var1);
> ```
>
> 该方法由子类实现，即`XMPPTCPConnection`。
>
> Stanza类有三个子类：IQ、Presence、Message，分别对应了XMPP对应的三个消息类型。

#### socket连接建立

在`connectUsingConfiguration()`方法中，创建套接字Socket，根据域名地址选择多个IP地址进行连接，直到连接上。

```java
this.socket = socketFactory.createSocket();
this.socket.setTcpNoDelay(true);
this.socket.setKeepAlive(true);
InetAddress inetAddress = (InetAddress)inetAddresses.next();
String inetAddressAndPort = inetAddress + " at port " + port;
LOGGER.finer("Trying to establish TCP connection to " + inetAddressAndPort);

try {
    this.socket.connect(new InetSocketAddress(inetAddress, port), timeout);
} catch (Exception var13) {
    hostAddress.setException(inetAddress, var13);
    if (inetAddresses.hasNext()) {
        continue;
    }
}
```



#### 数据发送读取

初始化`writer`和`reader`用以从socke缓冲区，写入读取数据：

```java
private void initReaderAndWriter() throws IOException {
    InputStream is = this.socket.getInputStream();
    OutputStream os = this.socket.getOutputStream();
    if (this.compressionHandler != null) {
        is = this.compressionHandler.getInputStream(is);
        os = this.compressionHandler.getOutputStream(os);
    }

    this.writer = new OutputStreamWriter(os, "UTF-8");
    this.reader = new BufferedReader(new InputStreamReader(is, "UTF-8"));
    this.initDebugger();
}
```

初始化`PacketWriter`和`PacketReader`，这两个类是`XMPPTCPConnection`的内部类，在这两个类的内部调用`writer`和`reader`写入或者读取数据。

```java
private void initConnection() throws IOException {
    boolean isFirstInitialization = this.packetReader == null || this.packetWriter == null;
    this.compressionHandler = null;
    this.initReaderAndWriter();
    if (isFirstInitialization) {
        // 初始化
        this.packetWriter = new XMPPTCPConnection.PacketWriter();
        this.packetReader = new XMPPTCPConnection.PacketReader();
        // 是否开启调试功能，忽略
        if (this.config.isDebuggerEnabled()) {
            this.addAsyncStanzaListener(this.debugger.getReaderListener(), (StanzaFilter)null);
            if (this.debugger.getWriterListener() != null) {
                this.addPacketSendingListener(this.debugger.getWriterListener(), (StanzaFilter)null);
            }
        }
    }

    // 调用PacketWriter、PacketReader的init()方法，开启死循环
    this.packetWriter.init();
    this.packetReader.init();
}
```

##### PacketWriter

该类内部定义了一个阻塞缓冲队列，发送的数据都放到这个自定义的阻塞队列中。

```java
ArrayBlockingQueueWithShutdown<Element> queue = new ArrayBlockingQueueWithShutdown(500, true);
```

该类中的方法`writePackets()`循环从阻塞队列中读取数据，并通过socket发送出去。

##### PacketReader

改了内部定义了一个`XmlPullParser`用以解析从socket缓冲区读取到的XML格式信息。

```java
// PacketParserUtils.java
public static XmlPullParser newXmppParser(Reader reader) throws XmlPullParserException {
    XmlPullParser parser = newXmppParser();
    parser.setInput(reader);// 该reader就是读取socket缓冲区的BufferedReader
    return parser;
}
```

在`parsePackets()`方法中，一直通过`XmlPullParser`循环解析socket缓存区的数据，直到调用`shutdown()`方法。

当解析到XMPP消息协议中三个最重要的节点：`<message>`、`<iq>`、`<presence>`，调用方法：

```java
parseAndProcessStanza(parser);
```

该方法定义于`XMPPTCPConnection`的抽象父类：`AbstractXMPPConnection`中：

```java
protected void parseAndProcessStanza(XmlPullParser parser) throws Exception {
    ParserUtils.assertAtStartTag(parser);
    int parserDepth = parser.getDepth();
    Stanza stanza = null;

    try {
        // 真正解析<message>、<iq>、<presence>
        stanza = PacketParserUtils.parseStanza(parser);
    } catch (Exception var8) {
        // 不能正常解析的，包装为UnparseableStanza回调出去
        CharSequence content = PacketParserUtils.parseContentDepth(parser, parserDepth);
        UnparseableStanza message = new UnparseableStanza(content, var8);
        ParsingExceptionCallback callback = this.getParsingExceptionCallback();
        if (callback != null) {
            callback.handleUnparsableStanza(message);
        }
    }

    ParserUtils.assertAtEndTag(parser);
    if (stanza != null) {
        // (1)如果是<iq>则还需将该消息发送到服务器;(2)如果是<message>、<presence>则notify出去
        processStanza(stanza);
    }

}
```

> <presence></presence>当前在线状态
>
> <message></message>消息的主题内容
>
> <iq></iq>获取联系人、登陆、注册

#### 长连接维护

连接XMPP服务器时，设置了socket的连接方式为：

```java
this.socket.setKeepAlive(true);
```

这句话表示连接建立之后，只要双方均未主动关闭连接，那这个连接就是会一直保持的，TCP连接空闲时需要向对方发送探测包，来保持长连接。这个长连接是**传输层**维护的，对应用层无感知。还需要在应用层做长连接操作。

smack包通过`PingManager`发送心跳包来维持应用层的长连接。

PingManager把心跳包封装到了`<iq>`节点中，子节点为`<ping>`

### disconnect操作

调用方法：AbstractXMPPConnection类的disconnect()

```java
public void disconnect() {
    try {
        this.disconnect(new Presence(Type.unavailable));
    } catch (NotConnectedException var2) {
        LOGGER.log(Level.FINEST, "Connection is already disconnected", var2);
    }
}

public synchronized void disconnect(Presence unavailablePresence) throws NotConnectedException {
    try {
        this.sendStanza(unavailablePresence);
    } catch (InterruptedException var3) {
        LOGGER.log(Level.FINE, "Was interrupted while sending unavailable presence. Continuing to disconnect the connection", var3);
    }

    this.shutdown();
    this.callConnectionClosedListener();
}
```

由代码可知，disconnect操作是发送了一个Type为`unavailable`的**Presence**的节点。

