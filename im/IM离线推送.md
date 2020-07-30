### 离线推送接入

APP接入离线推送，PushSDK：

```java
// 1、接入推送SDK
PushSDK push = PushSDK.getInstance(context);
push.enableLog(true); // 设置是否打开log
push.setServerConfig("10.2.9.43",8080); // 服务器ip地址，端口
push.registerApp(clientId);// 设备ID，用来唯一标识设备
push.registerAPPID(BaseLibInit.APPID, clientCode, Environment.DEV);
push.startPush(); // 启动PushService，开启长连接
PushSDK.getInstance(context).reportApp(longitude, latitude);// 上报经纬度

// 2、自建推送拉起厂商推送通道
PushClient.Instance.init(this); // 启动XMIntentService，拉起厂商离线推送
// 该pushListener主要用于获取token，从厂商那里获取，并存储到本地
PushClient.Instance.setPushListener(new PushClient.PushListener() {
    @Override
    public void onToken(String s) {
        // 收到该token上报服务端
        Bus.callData(MainApplication.getInstance(),"app/registerPushToken",s);
    }
});
```

自定义广播接收器，接受离线消息：

```xml
<!-- appID在MTP平台申请 -->
<meta-data
    android:name="appID"
    android:value="1007" />

<receiver android:name=".app.receiver.PushReceiver">
    <intent-filter>
        <action android:name="app.android.pushsdk.receiver.1007"/>
        <action android:name="action.open.notification"/>
    </intent-filter>
</receiver>
```



### 几个重要的类

Service：**PushService**、**XMIntentService**
BroadcastReceiver：**DaemonReceiver**、**PushMsgCenter**

#### PushService

维护长连接，长连接由**LongConnection**类维护，长连接收到消息后，发送广播到**PushMsgCenter**

（在**LongConnection**发送）

```java
private void onPushMsgArrived(PushMessage pushMsg) {
    PushLog.d("PushService", "收到推送：" + pushMsg.mid);
    Intent intent = new Intent();
    intent.setPackage(this.context.getPackageName());
    intent.setAction("app.android.pushsdk.RECEIVE_MESSAGE");
    intent.putExtra("PARAM_PUSH_MSG_ACID", pushMsg.acid);
    intent.putExtra("PARAM_PUSH_MSG_MID", pushMsg.mid);
    intent.putExtra("PARAM_PUSH_MSG_EXPIRED", pushMsg.expired);
    intent.putExtra("PARAM_PUSH_MSG_TITLE", pushMsg.title);
    intent.putExtra("PARAM_PUSH_MSG_BODY", pushMsg.body);
    intent.putExtra("PARAM_PUSH_MSG_EXT", pushMsg.extension);
    this.context.sendBroadcast(intent);
}
```



#### DaemonReceiver

监听开关机广播、网络状态变更广播，收到广播后检测PushService是否存活，不存活则拉起PushService

#### PushMsgCenter

接受离线消息，发送广播到`"app.android.pushsdk.receiver." + appID`（外部APP使用，VBK）

#### XMIntentService
由**BasePushManager**调用registerPush启动该service，该Service用来拉起厂商的离线推送。

> 在PushClient的init()方法中，检查设备初始化不同的BasePushManager
>
> ```java
> public void init(Context context) {
>     if (context == null) {
>         throw new NullPointerException("context == null");
>     } else {
>         this._context = context;
>         // 根据运行设备初始化不同的PushManager
>         this.pushManager = PushManagerFactory.createPushManager(context);
>         this.isSupportDevice = this.pushManager != null;
>         if (this.isSupportDevice) {
>             // 启动XMIntentService，拉起厂商的离线推送
>             this.pushManager.registerPush(context);
>         }
> 
>         this.isInit = true;
>     }
> }
> ```
>
> 