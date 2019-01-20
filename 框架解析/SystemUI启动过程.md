### SystemUI的启动流程
SystemUI是核心系统应用，需要开机启动，启动SystemUI进程，是通过启动SystemUIService来实现的。而SystemUIService何时由谁启动呢？
SystemUIService存在于：`frameworks\base\services\java\com\android\server\SystemServer.java`
答案就是在SystemServer启动后，紧接着启动ActivityManagerService服务，当ActivityManagerService调用 systemReady()后，会去启动SystemUIService。

```java
private void startOtherServices() {
    // ...
    mActivityManagerService.systemReady(new Runnable() {
        // ...
        startSystemUi(context);
    });
}

static final void startSystemUi(Context context) {
    Intent intent = new Intent();
    intent.setComponent(new ComponentName("com.android.systemui",
                                          "com.android.systemui.SystemUIService"));
    intent.addFlags(Intent.FLAG_DEBUG_TRIAGED_MISSING);
    //Slog.d(TAG, "Starting service: " + intent);
    context.startServiceAsUser(intent, UserHandle.SYSTEM);
}
```

> ActivityManagerService的systemReady函数接收一个名为goingCallback的Runnable实例作为参数。顾名思义，当AMS完成对systemReady的处理将会回调goingCallback的run方法，此时启动SystemUI。

SystemServer启动SystemUIService后，会走到SystemUIService的onCreate函数。

```java
public class SystemUIService extends Service {
    @Override
    public void onCreate() {
        super.onCreate();
        ((SystemUIApplication) getApplication()).startServicesIfNeeded();
    }
```
SystemUIService就是一个普通的Service，在onCreate里面，会调用SystemUIApplication的startServicesIfNeeded()方法。
```java
/**
 * Application class for SystemUI.
 */
public class SystemUIApplication extends Application {
    // SystemUIApplication的 TAG 居然叫 SystemUIService
    private static final String TAG = "SystemUIService";
	// 由SystemUI管理的各个组件的 Class对象
    private final Class<?>[] SERVICES = new Class[] {
            com.android.systemui.tuner.TunerService.class,
            com.android.systemui.keyguard.KeyguardViewMediator.class,
            com.android.systemui.recents.Recents.class,
            com.android.systemui.volume.VolumeUI.class,
            com.android.systemui.statusbar.SystemBars.class,
            com.android.systemui.usb.StorageNotification.class,
            com.android.systemui.power.PowerUI.class,
            com.android.systemui.media.RingtonePlayer.class,
    };
   
  	@Override  
  	public void onCreate() {  
      super.onCreate();  
      setTheme(R.style.systemui_theme);  
  
      // 注册开机完成的广播
      IntentFilter filter = new IntentFilter(Intent.ACTION_BOOT_COMPLETED);  
      filter.setPriority(IntentFilter.SYSTEM_HIGH_PRIORITY);  
      registerReceiver(new BroadcastReceiver() {  
          @Override  
          public void onReceive(Context context, Intent intent) {  
              if (mBootCompleted) return;  
              unregisterReceiver(this);  
              mBootCompleted = true;  
              if (mServicesStarted) {  
                  final int N = mServices.length;  
                  for (int i = 0; i < N; i++) {  
                      mServices[i].onBootCompleted();  
                  }  
              }  
          }  
      }, filter);  
  }
  
   // 启动各个组件，稍后分析
   public void startServicesIfNeeded() {
        startServicesIfNeeded(SERVICES);
   }
```
SystemUIApplication是一个Application实现，在Application的onCreate()函数中注册开机完成广播，当收到开机完成的广播后，如果各个组件已经启动（即调用各个组件的start函数），再次调用各个组件的`onBootCompleted` 方法。

SystemUIApplication的`startServicesIfNeeded`函数定义如下：
```java
public void startServicesIfNeeded() {  
    // 各个组件已经完成了启动，直接返回
    if (mServicesStarted) {  
        return;
    }  
  
    // 如果还没有收到开机完成的广播，主动查询一下系统属性，看看是否已经开机完成
    if (!mBootCompleted) {  
        if ("1".equals(SystemProperties.get("sys.boot_completed"))) {  
            mBootCompleted = true;  
            if (DEBUG) Log.v(TAG, "BOOT_COMPLETED was already sent");  
        }  
    }  
  
    Log.v(TAG, "Starting SystemUI services.");  
    final int N = SERVICES.length;
    // 循环遍历各个组件，完成各个组件的启动
    for (int i=0; i<N; i++) {  
        Class<?> cl = SERVICES[i];  
        if (DEBUG) Log.d(TAG, "loading: " + cl);  
        try {
            // 这个工厂方法createInstance的返回值一直未null
            Object newService = SystemUIFactory.getInstance().createInstance(cl);
            // 等价于：mServices[i] = (SystemUI)cl.newInstance()；
            mServices[i] = (SystemUI) ((newService == null) ? cl.newInstance() : newService);
        } catch (IllegalAccessException ex) {  
            throw new RuntimeException(ex);  
        } catch (InstantiationException ex) {  
            throw new RuntimeException(ex);  
        }  
        mServices[i].mContext = this;  
        mServices[i].mComponents = mComponents;  
        if (DEBUG) Log.d(TAG, "running: " + mServices[i]);
        // 调用各个组件的start方法，完成组件的启动
        mServices[i].start();  
  
        // 这个地方调用到的场景是：没有收到开机完成的广播，但是查询到系统属性表明已经开机完成
        // 以后再次收到开机完成的广播的后，直接返回了
        if (mBootCompleted) {  
            mServices[i].onBootCompleted();  
        }
    }
    mServicesStarted = true;  
}  
```


#### 状态栏的启动
SystemBars继承于SystemUI，在SystemUIApplication中会启动SystemBars：`mServices[i].start();`
SystemBars.java
```java
@Override  
   public void start() {  
       if (DEBUG) Log.d(TAG, "start");  
       mServiceMonitor = new ServiceMonitor(TAG, DEBUG,  
               mContext, Settings.Secure.BAR_SERVICE_COMPONENT, this);  
       mServiceMonitor.start();  // will call onNoService if no remote service is found  
   }
```
在start函数中会实例化ServiceMonitor以及启动ServieMonitor。
ServiceMonitor.java
```java
public void start() {  
      // listen for setting changes  
      ContentResolver cr = mContext.getContentResolver();  
      cr.registerContentObserver(Settings.Secure.getUriFor(mSettingKey),  
              false /*notifyForDescendents*/, mSettingObserver, UserHandle.USER_ALL);  
  
      // listen for package/component changes  
      IntentFilter filter = new IntentFilter();  
      filter.addAction(Intent.ACTION_PACKAGE_ADDED);  
      filter.addAction(Intent.ACTION_PACKAGE_CHANGED);  
      filter.addAction(Intent.ACTION_PACKAGE_REMOVED);  
      filter.addDataScheme("package");  
      mContext.registerReceiver(mBroadcastReceiver, filter);  
  
      mHandler.sendEmptyMessage(MSG_START_SERVICE);  
  } 
```
这个函数主要做两个事情：
- 1.监听apk的安装卸载，即apk变化事件
- 2.发送MSG_START_SERVICE，启动service

ServiceMonitor.java
```java
private void startService() {  
       mServiceName = getComponentNameFromSetting();  
       if (mDebug) Log.d(mTag, "startService mServiceName=" + mServiceName);  
       if (mServiceName == null) {  
           mBound = false;  
           mCallbacks.onNoService();  
       } else {  
           long delay = mCallbacks.onServiceStartAttempt();  
           mHandler.sendEmptyMessageDelayed(MSG_CONTINUE_START_SERVICE, delay);  
       }  
   } 
```
当ServiceName为NULL时，会触发到mCallbacks的onNoService函数，在不等于NULL的时候，下面的Handler消息也会触发callBacks的onNoService函数。SystemBars的定义：
`public class SystemBars extends SystemUI implements ServiceMonitor.Callbacks`

SystemBars.java
```java
@Override  
   public void onNoService() {  
       if (DEBUG) Log.d(TAG, "onNoService");  
       createStatusBarFromConfig();  // fallback to using an in-process implementation  
   }  
```

createStatusBarFromConfig函数实现：
```java
private void createStatusBarFromConfig() {  
       if (DEBUG) Log.d(TAG, "createStatusBarFromConfig");  
       final String clsName = mContext.getString(R.string.config_statusBarComponent);  
       if (clsName == null || clsName.length() == 0) {  
           throw andLog("No status bar component configured", null);  
       }  
       Class<?> cls = null;  
       try {  
           cls = mContext.getClassLoader().loadClass(clsName);  
       } catch (Throwable t) {  
           throw andLog("Error loading status bar component: " + clsName, t);  
       }  
       try {  
           mStatusBar = (BaseStatusBar) cls.newInstance();  
       } catch (Throwable t) {  
           throw andLog("Error creating status bar component: " + clsName, t);  
       }  
       mStatusBar.mContext = mContext;  
       mStatusBar.mComponents = mComponents;  
       mStatusBar.start();  
       if (DEBUG) Log.d(TAG, "started " + mStatusBar.getClass().getSimpleName());  
   }
```
从代码中可以看出，利用反射，从配置文件中读取StatusBar的全路径名。通过ClassLoader来加载，构造出BaseStatusBar的实例。然后调用BaseStatusBar的start函数。下一篇重点介绍BaseStatusBar以及PhoneStatusBar两个方法。
