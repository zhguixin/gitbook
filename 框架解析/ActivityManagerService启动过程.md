### 深入理解ActivityManagerService

#### AMS的启动过程

AMS的启动入口，在SystemServer的run方法中，看一下代码：

```java
    /**
     * The main entry point from zygote.
     * base/services/java/com/android/server/SystemServer.java
     */
    public static void main(String[] args) {
        new SystemServer().run();
    }

	public SystemServer() {
        // Check for factory test mode.
        mFactoryTestMode = FactoryTest.getMode();
    }

    private void run() {
      //一些手机本地化、虚拟机的设置
      
      // Initialize native services.
      System.loadLibrary("android_servers");
      
      // 调用ActivityThread的静态systemMain方法
      createSystemContext();
      
      // Create the system service manager.
      mSystemServiceManager = new SystemServiceManager(mSystemContext);
      LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);
      
      // Start services.
      startBootstrapServices();
      startCoreServices();
      startOtherServices();
    }
```

先重点看一下createSystemContext方法：

```java
    private void createSystemContext() {
        // systemMain创建ActivityThread实例
        ActivityThread activityThread = ActivityThread.systemMain();
        // getSystemContext创建ContextImpl的实例、创建LoadedApk实例
        // mSystemContext在反射调用时，传入AMS的构造函数
        mSystemContext = activityThread.getSystemContext();
        mSystemContext.setTheme(DEFAULT_SYSTEM_THEME);
    }
```

ActivityThread是应用进程的UI线程。system_server作为一个特殊的“应用进程”，在systemMain函数中：

```java
public static ActivityThread systemMain() {
    ActivityThread thread = new ActivityThread();
    thread.attach(true); // 注意此处传入的为true，表示由system_server调用
    return thread;    
}
```

保证了系统进程(即system_server进程)和普通进程(直接调用ActivityThread的main函数)一样拥有一个完备的Android运行环境。

紧接着，在`startBootstrapServices`方法中启动了AMS：

```java
// Activity manager runs the show.
mActivityManagerService = mSystemServiceManager.startService(
    ActivityManagerService.Lifecycle.class).getService();
```

通过调用SystemServiceManager的startService方法来启动AMS。具体的就是通过Lifecycle的类名(Lifecyle为AMS的静态内部类)反射调用，执行Lifecycle的构造函数拿到Lifecycle的实例，然后调用Lifecycle的onStart方法启动AMS，最后调用Lifecycle的getService方法返回AMS。Lifecycle的代码如下：

```java
    public static final class Lifecycle extends SystemService {
        private final ActivityManagerService mService;

        public Lifecycle(Context context) {
            super(context);
            mService = new ActivityManagerService(context);
        }

        @Override
        public void onStart() {
            mService.start();
        }

        public ActivityManagerService getService() {
            return mService;
        }
    }
```

AMS在启动过程中，先执行构造函数完成初始化，再执行start方法完成启动。

实例化AMS：

```java
// AMS在UI线程启动
public ActivityManagerService(Context systemContext) {
    
    // mSystemThread代表ActivityThread的实例
    mSystemThread = ActivityThread.currentActivityThread();

    // mHandlerThread是自定义的HandlerThread类，一些需要在工作线程处理的事情
    // 通过mHandler发送消息到工作线程处理
    mHandler = new MainHandler(mHandlerThread.getLooper());
    mUiHandler = new UiHandler();
    
    mProcessStats = new ProcessStatsService(this, new File(systemDir, "procstats"));
    
    mUserController = new UserController(this);
    
    // Activity栈管理
    mStackSupervisor = new ActivityStackSupervisor(this);
    mActivityStarter = new ActivityStarter(this, mStackSupervisor);
    mRecentTasks = new RecentTasks(this, mStackSupervisor);    
}
```

start方法：

```java
private void start() {
    Process.removeAllProcessGroups();
    mProcessCpuThread.start();

    mBatteryStatsService.publish(mContext);
    mAppOpsService.publish(mContext);
    Slog.d("AppOps", "AppOpsService published");
    LocalServices.addService(ActivityManagerInternal.class, new LocalService());
}
```



接着看，根据SystemServer的执行顺序，看看AMS还干了些什么事：(选取了一部分)

```java
   // 初始化电源管理
   mActivityManagerService.initPowerManagement();

   //添加一些服务到ServiceManager
   mActivityManagerService.setSystemProcess();
   
   mActivityManagerService.installSystemProviders();
   
   mActivityManagerService.setWindowManager(wm);

  // AMS先调用systemReady方法，完成一些准备后，回调Runnable的run方法
  mActivityManagerService.systemReady(new Runnable() {
  	mActivityManagerService.startObservingNativeCrashes();
    
    startSystemUi(context);//启动SystemUI，通过发送Intent跳转到SystemUIService
    ....
  }
```

#### AMS内部执行流程

##### 1、先看一下，AMS的setSystemProcess方法：

调用该方法之前已经实例化了AMS，并调用了start方法。

```java
public void setSystemProcess() {
    try {   
        //添加一些服务到ServiceManager
        ServiceManager.addService(Context.ACTIVITY_SERVICE, this, true);
        ServiceManager.addService(ProcessStats.SERVICE_NAME, mProcessStats);

        // 将资源APK的ApplicationInfo，绑定到system_server进程
        ApplicationInfo info = mContext.getPackageManager().getApplicationInfo(
            "android", STOCK_PM_FLAGS | MATCH_SYSTEM_ONLY);
        mSystemThread.installSystemApplicationInfo(info, getClass().getClassLoader());

        // 保证AMS实现进程管理，该ProcessRecord代表system_server
        // system_server进程也由AMS管理
        ProcessRecord app = newProcessRecordLocked(info, info.processName, false, 0);

    } catch (PackageManager.NameNotFoundException e) {
        throw new RuntimeException(
            "Unable to find android system package", e);
    }
}
```

##### 2、AMS的installSystemProviders方法

加载SettingsProvider.APK，加入到system_server进程。主要是启动SettingProvider，方便读取配置信息。

##### 3、AMS的systemReady方法

完成系统就绪的必要工作，然后会启动Launcher、在回调中又会启动SystemUI。在该方法中，先判断AMS中的成员变量mSystemReady的值，如果为true，执行callback的run方法后，就直接返回了。

```java
public void systemReady(final Runnable goingCallback) {
    before goingCallback;
    if (goingCallback != null) goingCallback.run();
    after goingCallback;
}
```





#### AMS对Activity的管理

AMS从名字上就能看出来是对Activity的管理，AMS提供了几个重要的类(或数据结构)来管理Activity。

- ActivityRecord，记录一个Activity，保存了Activity的状态
- TaskRecord，用来表示Task
- ActivityStack，用来管理一系列Task
- ActivityStackSupervisor，该类是在4.4版本之后引入的，用来组织管理ActivityStack

![](F:\gitbook\images\ActivityTask.png)



#### AMS启动Activity的流程分析

直接从AMS的startActivity开始分析，在该方法内部拿到IApplicationThread、TaskRecord的实例后又调用到ActivityStarter中的startActivityMayWait函数。

- Intent的解析（Intent可能是隐式的：关于[Intents and Intent Filters](https://developer.android.com/guide/components/intents-filters.html)）
- Activity的匹配（符合Intent的Activity可能会有多个）
- 应用进程的创建
- Task，Stack的获取或者创建
- Activity窗口的创建
- Activity生命周期的调度（onCreate，onResume等）