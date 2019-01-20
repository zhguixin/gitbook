Android中的Service组件不同于Activity，对用户不可见，也无法与用户直接沟通。

启动Service，通过Context，调用**startService**或者**bindService**来启动。

Service的生命周期函数有：onCreate、onStartCommand、onDestory。

- 无论调用几次startService，Service只保持一个，并且通过调用stopService来停止Service。
- 通过bindService方法启动的Service，调用进程结束后，Service也就销毁了。

在Service每一次的开启关闭过程中，只有onStart可被多次调用（通过多次startService调用），其他onCreate，onBind，onUnbind，onDestory在一个生命周期中只能被调用一次。

如果先通过startService启动了Service，再通过bindService来绑定Service时，直接调用到Service的onBind()。

反过来：先通过bindService启动了Service，再通过startService来绑定Service时，直接调用到Service的onStart()的顺序不同。

注意：*此时无法通过stopSevice来停止Service，必须先调用unBindService方法在调用stopService方法。*

确保开机Service就能够启动，在AndroidManifest中加入：**android:directBootAware="true"**（需要API在24以上）

```xml
<service
   android:name=".RemoteService"
   android:process=":remote"
   android:directBootAware="true"
   android:exported="true" >
  <intent-filter>
    <category android:name="android.intent.category.DEFAULT" />
    <action android:name="com.zgx.site.service" />
  </intent-filter>
</service>
```

可以通过如下方法判断service是否在运行：

```java
public boolean isServiceRunning(Context context, final String className) {
  ActivityManager activityManager = (ActivityManager) 		
    context.getSystemService(Context.ACTIVITY_SERVICE);
  List<ActivityManager.RunningServiceInfo> info = 
    activityManager.getRunningServices(Integer.MAX_VALUE);
  if (info == null || info.size() == 0) {
    return false;
  }

  for (ActivityManager.RunningServiceInfo aInfo : info) {
    if (className.equals(aInfo.service.getClassName())) {
      return true;
    }
  }
  return false;
}
```