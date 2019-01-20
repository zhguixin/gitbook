状态栏通知Notification是看不见的程序组件（Broadcast Receiver、Service和不活跃的Activity）警示用户需要注意事情发生的最好途径。

### 1.创建通知栏消息的步骤

- 通过getSystemService()方法得到NotificationManager对象；
- 创建Notificationt.Builder的实例，通过builer的一些方法设置通知的一些属性，如内容、图标、标题，对相应的Notification的动作进行处理等；
- 通过NotificationManager对象的notify()方法执行Notification的通知；
- 通过NotificationManager对象的cancel()方法取消Notification的通知。

```java
mNotiManger=(NotificationManager)getSystemService(NOTIFICATION_SERVICE);
Notificationt.Builder builder = new Notificationt.Builder(MainActivity.this);

builder.setSmallIcon(R.drawable.ic_launcher);
builder.setContentTitle("examle");
builder.setContentText("Hello There");
builder.setTicker("Hello There");
//builder.setOngoing(true);//使通知不可被删除
mNotiManger.notify(NOTIFICATION_FLAG,builder.build());
```

其中`NOTIFICATION_FLAG`唯一标识了这个通知。以上就简单创建了一个通知消息。

分析`notify` 的源码得知，实际上拿到了NotificationManagerService(NMS)的代理来与NMS进行通信。调用到了：NMS的`enqueueNotificationInternal` 方法。

```java
void enqueueNotificationInternal() {
    // 是否是系统通知
    // 防止恶意应用疯狂弹出通知的DOS攻击
    
    // 构造StatusBarNotification
    final StatusBarNotification n = new StatusBarNotification(
        pkg, opPkg, id, tag, callingUid, callingPid, 0, notification,
        user);
    final NotificationRecord r = new NotificationRecord(getContext(), n);
    mHandler.post(new EnqueueNotificationRunnable(userId, r));
```

其中`EnqueueNotificationRunnable` 实现了Runnable接口：

```java
private class EnqueueNotificationRunnable implements Runnable { 
    @Override
    public void run() {
        // ...
        // n即为StatusBarNotification对象
        int index = indexOfNotificationLocked(n.getKey());
        if(index < 0) {
            mNotificationList.add(r);
        } else {// 更新通知
        }

        mRankingHelper.sort(mNotificationList);
        // 有smallIcon，弹出通知
        // mListeners = new NotificationListeners();
        mListeners.notifyPostedLocked(n, oldSbn);
    }

}
```

mListeners为`NotificationListeners` 的实例对象，继承自`ManagedServices` ：

```java
public class NotificationListeners extends ManagedServices {
    // 这里传入的IBinder是 bind SystemUI中的NotificationListenerService拿到的
    @Override
    protected IInterface asInterface(IBinder binder) {
        return INotificationListener.Stub.asInterface(binder);
    }
    
    public void notifyPostedLocked(StatusBarNotification sbn, StatusBarNotification oldSbn) {
        TrimCache trimCache = new TrimCache(sbn);
        
        final StatusBarNotification sbnToPost =  trimCache.ForListener(info);
        mHandler.post(new Runnable() {
            @Override
            public void run() {
                notifyPosted(info, sbnToPost, update);
            }
        });
    }
    
    private void notifyPosted(final ManagedServiceInfo info,
       final StatusBarNotification sbn, NotificationRankingUpdate rankingUpdate) {
        // SystemUI是INotificationListener真正实现者，这里拿到了代理
        final INotificationListener listener = (INotificationListener)info.service;
        StatusBarNotificationHolder sbnHolder = new StatusBarNotificationHolder(sbn);
        try {
            // 跨进程调用到SystemUI，弹出通知到SystemUI的下拉通知栏
            listener.onNotificationPosted(sbnHolder, rankingUpdate);
        } catch (RemoteException ex) {
            Log.e(TAG, "unable to notify listener (posted): " + listener, ex);
        }
    }
}
```

在`ManagedServices` 中的 `registerServiceLocked` 去bindService：

```java
private void registerServiceLocked(final ComponentName name, userid,/**/) {
    Intent intent = new Intent(mConfig.serviceInterface);
    intent.setComponent(name);
    
    ServiceConnection serviceConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder binder) {
            // 代码省略
            IInterface mService = asInterface(binder);
        }
    }
    
    mContext.bindServiceAsUser(intent,
                serviceConnection,
                BIND_AUTO_CREATE | BIND_FOREGROUND_SERVICE | BIND_ALLOW_WHITELIST_MANAGEMENT,
                new UserHandle(userid)))；
}
```

> 其中ComponentName是SystemUI在start函数调用NotificationManagerService的 `registerListener` 把SystemUI自己注册进去的。

在`NotificationListenerService` 中通过Handler post到UI线程来进行通知条的显示。

```java
protected class NotificationListenerWrapper extends INotificationListener.Stub {
    @Override
    public void onNotificationPosted(IStatusBarNotificationHolder sbnHolder,
                                     NotificationRankingUpdate update) {
        // ...
        mHandler.obtainMessage(MyHandler.MSG_ON_NOTIFICATION_POSTED,
                            args).sendToTarget();
    }
}
```

