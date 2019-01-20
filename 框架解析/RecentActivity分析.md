## RecentActivity分析

在SystemUI中，点击虚拟按键触发`toggleRecentApps`后，触发到Recents组件的`toggleRecents` 方法，然后根据当前的UI_MODE是否是TV模式在Recents中实例化`RecentsTvImpl` 或者`RecentsImpl` 。

> `RecentsTvImpl` 继承自RecentsImpl，UI上有稍许改动

RecentsImp实现了RecentApp的大部分功能，Recents组件中的方法大都是调用RecentsImpl。

在RecentsImpl的`toggleRecents` 方法：

```java
// 伪代码，需要考虑到是否快速切换、是否是Alt+Tab键切换等
SystemServicesProxy ssp = Recents.getSystemServices();

if(ssp.isRecentsActivityVisible()) {
    // 已经在RecentsApp界面，切换到一个Activity
    EventBus.getDefault().post(new IterateRecentsEvent());
} else {
    // 不在RecentsApp界面，启动RecentsActivity
    ActivityManager.RunningTaskInfo runningTask = ssp.getRunningTask();
    startRecentsActivity()
}
```

其实这个地方，虚拟按键收到ACTION_UP后才会触发`toggleRecents` ，在虚拟按键上如果监听到`ACTION_DOWN`事件就会触发预加载，提前加载RecentsApp所需要的信息。

```java
// 由BaseStatusBar的preloadRecents方法一路调用过来，调用链可自行分析
// systemui/recents/misc/SystemServicesProxy.java
public List<ActivityManager.RecentTaskInfo> getRecentTasks(int numLatestTasks, int userId,
   boolean includeFrontMostExcludedTask, ArraySet<Integer> quietProfileIds) {
        try {
            // ...
            tasks = mAm.getRecentTasksForUser(numTasksToQuery, flags, userId);
            // ...
        } catch (Exception e) {
            Log.e(TAG, "Failed to get recent tasks", e);
        }
}
```

通过Binder调用到ActivityManagerService中的`getRecentTasksForUser`方法。

在AMS中流转的代码相当黑盒处理了。大概就是：

在ActivityManagerService中，createRecentTaskInfoFromTaskRecord()方法把TaskRecord信息转换成RecentTaskInfo的信息。

RecentTask是以TaskReord为单位进行管理的，不是一app或者activity为单位进行管理。

获取到RecentTaskInfo之后就要去获取缩略图信息。除了栈顶正在显示的TaskRecord会去实时的截取屏幕图像，其他的走getLastThumbnail。

缩略图已经保存在**/data/system_ce/0/recent_images**文件夹下。截图的时机是在Activity的onPause之后。

具体可参考资料：[SystemUI讲解之RecentActivity](https://blog.csdn.net/kebelzc24/article/details/53765379)

所以，我们看到RecentsApp中每个Activity只不过是一张截图，并且RecentsApp也就是个Activity。




