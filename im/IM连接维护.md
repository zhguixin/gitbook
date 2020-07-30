1. TCP的连接和发送超时是多少？； 重试机制如何？
   App现状：
   TCP目前Connect Timeout为15s，Send Timeout 默认15s，若连续三次tcp发送失败，调整为5s；
   TCP当次发送失败后走Http补偿，http send超时目前Android为10s，iOS和tcp超时逻辑一致；

TCP Reconnect机制 :

ReconnectManager 里运行一个子线程（Reconnect Thread）统一管理

1） 立即重连，即使Reconnect Thread未运行完，也立即中断；
a) 监听网络变化，从无网->有网时会立即connect；（中断运行中的Reconnect Thread，重启thread立即connect）；
b）发送消息时判断如果tcp not connected，则直接connect；
C) App端点亮屏幕（iOS为屏幕解锁）/切换到前台 时，若Tcp not connect则立即重连

2）App应用层认为Tcp connect时，以下场景会触发 reconnect; 并判断Reconnect Thread运行情况，若未运行完，则不中断；
a) App端Tcp Idle 大于45秒（Tcp没有上下行通信），会主动Ping Server，收取Pong的timeout为10s；Ping 失败即Tcp Reconnec
b) 若tcp not connected，则Reconnect TCP
c) App端点亮屏幕（iOS为屏幕解锁）/切换到前台 时, 若和上一次此场景Ping的间隔大于15s，会立即Ping，Ping 失败即Tcp Reconnect

3）Reconnect Thread内部如果Reconnect fail，则按照步长时间（5～15s）一直尝试重连，直到连接成功/线程场景 1）中断时，线程才结束运行；

\2. HTTP的连接和服务超时是多少？（HTTP可能并没有单独设置连接超时，而是设置了整体超时时间）；重试机制如何？
App现状：
HTTP 总体超时时间为10s （包括连接和读取数据的时间）；
重试机制：部分核心服务（拉会话/消息）若确实是网络超时会自动重试一次

\3. 哪些情况会触发Sync机制？ 之前整理过部分的
1）TCP重连不管成功/失败都会主动sync会话&消息
2）App端点亮屏幕（iOS为屏幕解锁）/切换到前台 时会主动sync会话&消息
3）App端收到离线push时会主动sync会话&消息
4）App端http补偿发送消息后会主动sync会话&消息；
5）列表页展示时会触发sync会话&消息；
6) 进入具体的聊天页时会主动sync当前会话消息；
注：
上述场景1-5）若某一个场景的sync还未结束通信，则下一个场景的sync不触发；若2次触发间隔大于30s，则立即执行，无论前一次的执行情况如何；
同时聊天页也会监听场景1-5）sync时的通知，触发聊天页主动sync单独某一具体会话的消息；



APP登出操作

调用接口：logout

```java
public void logout(IMResultCallBack resultCallBack) {

    logOutIMClient();
    // 断开TCP连接
    IMConnectManager.instance().disconnect();

    IMConversationManager.instance().reset();
    IMChatManager.instance().reset();
    IMConversationSyncManager.instance().reset();
    IMGroupManager.instance().reset();
    IMUserManager.instance().reset();
    IMSendMessageManager.instance().reset();
    // 调用unbindService,触发IMService的onDestroy方法，导致主线程调用了xmppconnection.disconnect()方法
    IMConnectManager.instance().reset();
    setIsIMUser(false);
    inst.loginInfo = null;

    // 关闭数据库
    CTChatDbStore.instance().close();

    isCompletedInited = false;
}
```

IMService的`onDestroy()`方法：

```java
@Override
public void onDestroy() {
    logger.i("IMService onDestroy");
    // todo 在onCreate中使用startForeground
    // 在这个地方是否执行 stopForeground呐
    EventBus.getDefault().unregister(this);
    // 在主线程运行了该方法
    imxmppManager.reset();
    super.onDestroy();
}
```

