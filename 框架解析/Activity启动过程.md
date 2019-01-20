Activity启动过程分析

通过adb命令来启动一个Activity，先从AMS中的`startActivityAndWait` 函数开始分析之旅。

1、`startActivityAndWait` 方法：

```java
public final WaitResult startActivityAndWait(IApplicationThread caller,// 与应用进程的通信接口
                                             String callingPackage,
                                             Intent intent, // 启动调用的Intent
                                             String resolvedType, IBinder resultTo, 
                                             String resultWho, int requestCode,
                                             int startFlags,
                                             ProfilerInfo profilerInfo, Bundle bOptions, 
                                             int userId) {
     mActivityStarter.startActivityMayWait(/***/)
}
```

又调用到了**AcitvityStarter**的`startActivityMayWait` 方法。

2、`startActivityMayWait` 方法

```java
final int startActivityMayWait(/**/) {
    // 匹配解析Intent
    ActivityInfo aInfo = mSupervisor.resolveActivity(intent, rInfo, startFlags, profilerInfo);
    
    // 启动Activity
    int res = startActivityLocked(/**/);
    
    // 根据res值做相应的处理
    // 等待res结果值为：START_TASK_TO_FRONT
}
```

3、重头戏`startActivityLocked` 方法

```java
final int startActivityLocked(/***/) {
    // 待启动Activity的pid、uid
    
    // 描述启动目标Activity的那个Activity
    ActivityRecord sourceRecord = null;
    // 描述接收启动结果的Activity，该Activity的onActivityResul将被调用
    ActivityRecord resultRecord = null;
    
    // 获取Intent设置的Activity启动方式
    final int launchFlags = intent.getFlags();
    
    // 创建一个ActivityRecord对象，并放入函数参数outActivity这个数组中
    ActivityRecord r = new ActivityRecord(/**/);
    if (outActivity != null) {
        outActivity[0] = r;
    }
    
    // 启动处于Pending状态的Activity
    doPendingActivityLaunchesLocked(false);
    
    // 为新创建的ActivityRecord找到一个合适的Task
    startActivityUnchecked(/**/);
}
```

4、`startActivityUnchecked` 方法

```java
// 根据启动模式，决定是否新建Task，以及对应的处理
private int startActivityUnchecked(/**/) {
    
    /**
       调用ActivityStackSupervisor的resumeFocusedStackTopActivityLocked
       -->> ActivityStack的resumeTopActivityUncheckedLocked -->> resumeTopActivityInnerLocked
    */
}
```

调用链比较长，最终会调用到ActivityStackSupervisor的`startSpecificActivityLocked` 方法。该方法内部又会调用ActivityManagerService的`startProcessLocked` 来向Zygote请求启动一个进程，然后调用ActivityThread的`main` 方法。

```java
private final void startProcessLocked(/**/) {
    //...
    Process.ProcessStartResult startResult = Process.start(/**/);
    
    mPidsSelfLocked.put(startResult.pid, app);
    // isActivityProcess有函数参数entryPoint是否为null决定，一般为null。随后赋值为
    // android.app.ActivityThread
    // 向Zygote请求创建进程，10s内没有收到创建的应用进程回应，则认为进程创建失败
    if (isActivityProcess) {
        Message msg = mHandler.obtainMessage(PROC_START_TIMEOUT_MSG);
        msg.obj = app;
        mHandler.sendMessageDelayed(msg, startResult.usingWrapper
                                    ? PROC_START_TIMEOUT_WITH_WRAPPER : PROC_START_TIMEOUT);
    }    
}
```

在ActivityThread的main方法中调用attach方法，再次通过Binder调用AMS的`attachApplicationLocked` 方法来移除那个延迟消息。

至此，Activity前半段的启动，以`startProcessLocked` 结束。

5、ActivityThread登场

ActivityThread是由Zygote调用main函数后，流转进来的。在RuntimeInit.java中反射调用执行ActivityThread的`main` 函数。

```java
public static void main(String[] args) {
    // ...
    Looper.prepareMainLooper();
    ActivityThread thread = new ActivityThread();
    thread.attach(false);// 传入的值是false
    Looper.loop();
}
```

在`attach` 方法中，调用AMS的`attachApplication` 方法，并将`IApplicationThread` 传入进去。

```java
public final void attachApplication(IApplicationThread thread) {
    int callingPid = Binder.getCallingPid();
    attachApplicationLocked(thread, callingPid);
}

private final boolean attachApplicationLocked(IApplicationThread thread, int pid) {
    // ...
    // 又回调到应用进程中的bindApplication方法
    thread.bindApplication(/**/);
    
    // 会调用ActivityStackSupervisor的realStartActivityLocked方法启动Activity，稍后分析
    mStackSupervisor.attachApplicationLocked(app);
    
    // 启动服务
    mServices.attachApplicationLocked(app, processName);
    
    // 是否有广播要处理
    sendPendingBroadcastsLocked(app);
    
    // 如果有上述任一组件启动，didSomething的值为true；
    // 这里的启动只是向应用进程发出指令，是否启动成功不确定，因此没有必要调整OomAdj的值
    if (!didSomething) {
        updateOomAdjLocked();
    }

}
```

在ActivityThread中有个内部类：ApplicationThread，收到AMS调用后，通过handler发送消息到UI线程，由ActivityThread的`handleBindApplication` 方法进行处理，设置进程名、创建Application对象、LoadedApk对象，如果设置了ContentProvider还需要为该进程设置ContentProvider。

6、`realStartActivityLocked` 方法

该方法在ActivityStackSupervisor类中：

```java
// 由attachApplicationLocked调用过来，后面两个boolean值为true
final boolean realStartActivityLocked(ActivityRecord, ProcessRecord, boolean andResume, boolean checkConfig) {
    
    // 回调到应用进程，启动Activity
    app.thread.scheduleLaunchActivity(/**/);
    
    if (andResume) {
        // stack为ActivityStack的实例，在该方法内部又调用到：completeResumeLocked(r);
        stack.minimalResumeActivityLocked(r);
    }
}
```

7、`completeResumeLocked` 方法：

```java
private void completeResumeLocked(ActivityRecord next) {
    // ...
    mStackSupervisor.scheduleIdleTimeoutLocked(next);
    // ...
}
```

 看到这里，又调用到了ActivityStackSupervisor中，该方法会设置一个10s的超时等待，等待应用进程在Activity的onResume执行完毕后，向AMS调用activityIdle方法。

这个地方是Activity启动过程中，应用进程与AMS最后一次交互。目的就是让AMS处理因为此次启动Activity而导致的其他Activity暂停或finish。