---

title: 深入理解Android四大组件之广播
date: 2017-10-26
tags: Android
categories: 

---

广播在Android系统中的使用非常之频繁，广播很大程度上实现了Android各个组件间的解耦，使得各个组件甚至进程间的通信变得尤为简单。

#### 从一个栗子开始

一个简单的广播使用：

```java
public class MainActivity extends Activity {
  
  // 继承BroadcastReceiver，重写 onReceive方法，接收广播
  private BroadcastReceiver mReceiver = new BroadcastReceiver() {
    @Override
    public void onReceive(Context context, Intent intent) {
      String action = intent.getAction();
      Log.d(TAG, "action = " + action);
      if (Intent.ACTION_SCREEN_OFF.equals(action)) {
        Log.d(TAG, "onReceive: 屏幕熄灭");
      } else if (Intent.ACTION_SCREEN_ON.equals(action)) {
        Log.d(TAG, "onReceive: 屏幕点亮");
    }
  };
  
    @Override
    protected void onCreate(Bundle savedInstanceState) {
      super.onCreate(savedInstanceState);
      // 通过 IntentFilter 过滤可以收到的广播
      IntentFilter filter = new IntentFilter();
      filter.addAction(Intent.ACTION_SCREEN_OFF);
      filter.addAction(Intent.ACTION_SCREEN_ON);
      // 注册广播接收
      registerReceiver(mReceiver, filter));
    }
  
    @Override
    protected void onDestroy() {
        super.onDestroy();
        Log.d(TAG, "onDestroy: ");
      	// 解注册广播防止内存泄漏
        unregisterReceiver(mReceiver);
    }
}
```

该示例接收系统的亮灭屏广播。使用动态注册广播的方式来注册广播。由于注册实在`onCreate`中，如果Activity没有启动或者Activity被销毁了，就不可能再收到广播。如果想要实现，程序未启动也能监听到广播，可以在**AndroidManifest**中静态注册广播：

```java
<application
	android:allowBackup="true"
	android:icon="@drawable/ic_launcher"
	android:label="@string/app_name"
	android:theme="@style/AppTheme" >

   	<!-- ActivityReceiver类要继承BroadcastReceiver -->
	<receiver android:name=".ActivityReceiver" >
		<intent-filter>
			<action android:name="android.intent.action.BOOT_COMPLETED" />
		</intent-filter>
	</receiver>
</application>
```

*在AndroidO对对静态广播在很大程度上进行了限制，因此鼓励使用动态注册广播的形式*

#### 广播的几种方式

- 普通广播，使用最多一种广播方式

- 有序广播，按顺序一条一条的处理

- 粘性广播，即Sticky广播。通过：`sendStickyBroadcast(Intent intent) `形式发送，该API已经不建议使用了。

  > 粘性消息在发送后就一直存在于系统的消息容器里面，等待对应的处理器去处理，如果暂时没有处理器处理这个消息则一直在消息容器里面处于等待状态，粘性广播的Receiver如果被销毁，那么下次重建时会自动接收到消息数据。
  >
  > 粘性广播处理了这样一种情况，假设要广播发送器每5秒发送一个广播，但广播接收器一旦注册就能收到已经发送的广播，即使错过5秒一循环这个周期。

#### 广播的实现原理

#### 广播注册过程

通过Context的`registerReceiver`最终调用到了，ContextImpl的`registerReceiverInternal`方法：

```java
private Intent registerReceiverInternal(BroadcastReceiver receiver, int userId,
                                        IntentFilter filter, String broadcastPermission,
                                        Handler scheduler, Context context) {
	IIntentReceiver rd = null;
  	// 得到IIntentReceiver的实例
    if (receiver != null) {
      if (mPackageInfo != null && context != null) {
        if (scheduler == null) {
          scheduler = mMainThread.getHandler();
        }
        rd = mPackageInfo.getReceiverDispatcher(
          receiver, context, scheduler,
          mMainThread.getInstrumentation(), true);
      } else {
        if (scheduler == null) {
          scheduler = mMainThread.getHandler();
        }
        // rd为InnerReceiver，（InnerReceiver extends IIntentReceiver.Stub）
        rd = new LoadedApk.ReceiverDispatcher(
          receiver, context, scheduler, null, true).getIIntentReceiver();
      }
    }

  	// 跨进程调用ActivityManagerService的registerReceiver方法
    try {
      final Intent intent = ActivityManagerNative.getDefault().registerReceiver(
        mMainThread.getApplicationThread(), mBasePackageName,
        rd, filter, broadcastPermission, userId);
      if (intent != null) {
        intent.setExtrasClassLoader(getClassLoader());
        intent.prepareToEnterProcess();
      }
      return intent;
    } catch (RemoteException e) {
      throw e.rethrowFromSystemServer();
    }
}
```

其中，`IIntentReceiver`在源码未编译的情况下是一个AIDL文件，定义如下：

```java
oneway interface IIntentReceiver {
    void performReceive(in Intent intent, int resultCode, String data,
            in Bundle extras, boolean ordered, boolean sticky, int sendingUser);
}
```

广播能够实现跨进程通信的基础就是这个`IIntentReceiver`接口，`BroadcastReceiver`的`onReceive`作为客户端通过这个接口来接收广播。这个接口的服务端为LoadedApk.ReceiverDispatcher的`InnerReceiver`类，该类继承`IIntentReceiver.Stub`了，并实现了`performReceive`方法。

*注意：`BroadcastReceiver`的`onReceive`方法运行在UI线程，不能做耗时处理。超过10s就会发生ANR*

> never perform long-running operations in it (there is a timeout of 10 seconds that the system allows before considering the receiver to be blocked and a candidate to be killed). You cannot launch a popup dialog in your implementation of onReceive().

我们可以通过registerReceiver(BroadcastReceiver,  IntentFilter, String, android.os.Handler)来将onReceive切换到工作线程。

此外，在BroadcastReceiver的goAsync方法可以在工作线程去执行：

```java
  @Override
  public void onReceive(Context context, Intent intent) {
    Log.d(TAG, "onReceive: ");
    final PendingResult result = goAsync();
    mPool.execute(new Runnable() {
      @Override
      public void run() {
        Log.d(TAG, "run: name" + Thread.currentThread().getName());
        handleIntent(context, intent);// 处理耗时操作
        result.finish();
        Log.d(TAG, "run: after finsh");// 实际运行是可以运行到这句话的
      }
    });
  }
```

但是它也并没有违背10秒处理不完就会导致ANR。因为界面上的IO操作不能超过5秒，为了保证这个5秒，可以通过goAsync在工作线程完成IO操作。完成后要调用`PendingResult.finish()`来结束整个广播流程。

> This does not change the expectation of being relatively responsive to the broadcast (finishing it within 10s), but does allow the implementation to move work related to it over to another thread to avoid glitching the main UI thread due to disk IO.



接着看广播的注册过程，随后进入到了ActivityManagerService中，对于广播处理的整体流程如下图所示：

![](..\images\broadcast.png)

广播的处理主要在在AMS中流转，以及用户通过`sendBroadcast(new Intent(INSTALL_ACTION));`也是在AMS中做具体处理。