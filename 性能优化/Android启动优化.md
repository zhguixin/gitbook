APP的启动过程分为：冷启动、热启动、温启动

#####APP启动优化

我们在启动APP时，如果在`Application`的`OnCreate()`、`MainActivity`的`OnCreate()`中做得处理耗时较长，会导致一段时间的白屏或者黑屏。冷启动的话这段时间持续的会比较长。

从Launcher启动一个APP时，这段空白时间由一个叫`StartingWindow`的窗口填充，它是个临时窗口，对应的`WindowType`是`TYPE_APPLICATION_STARTING`。这个空白窗口是为了告诉用户，系统已经接受到操作，正在响应。应用程序初始化完成加载完第一帧之后，这个窗口就会被移除。

为什么有的时候是白屏，有的时候是黑屏呢？其实这个和你设置的主题有关：`windowBackground`比`Activity`的`setContentView()`要先加载，这一段时间如果Theme是Light，屏幕就是白色的；如果是Dark，就会显示黑屏。

我们可以在主题中设置`windowBackground`，比如加载一个带有logo的drawable。

```xml
<style name="AppTheme" parent="@style/Theme.ZTE.Light">
  <!-- All customizations that are NOT specific to a particular API-level can go here. -->
  <item name="android:windowBackground">@drawable/splash</item>
</style>
```

其中splash为一个layer-list定义的drawable：

```xml
<layer-list xmlns:android="http://schemas.android.com/apk/res/android"
            android:opacity="opaque">
  <item android:drawable="@android:color/white"/>
  <item>
    <bitmap 
            android:src="@drawable/logo"
            android:gravity="center"/>
  </item>
</layer-list>
```
当然，这种方式比较讨巧，就是相当于加入了一个SplashActivity。真正的优化还是要减少OnCreate中的耗时操作。辅助与TraceView进行函数调用时长分析。

#### 启动时长统计

通过adb命令：

`adb shell am start -S -W 包名/启动类的全限定名`

输出结果如下：

```
Stopping: site.zhguixin.dytt
Starting: Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] cmp=site.zhguixin.dytt/.ui.MainActivity }
Status: ok
Activity: site.zhguixin.dytt/.ui.MainActivity
ThisTime: 2423
TotalTime: 2423
WaitTime: 2487
Complete
```

其中，ThisTime表示启动的Activity所消耗时长，TotalTime表示启动几个Activity时总消耗时长（这里只启动了一个Activity，所以ThisTime和TotalTime相等），WaitTime表示应用进程的创建过程 + TotalTime。

我们先看看这几个时间是怎么计算出来的，Activity启动起来的标准是什么？

在**ActivityRecord** 类中的`reportLaunchTimeLocked` 方法中：

```java
private void reportLaunchTimeLocked(final long curTime) {
    final ActivityStack stack = task.stack;
    if (stack == null) {
        return;
    }
    final long thisTime = curTime - displayStartTime;
    final long totalTime = stack.mLaunchStartTime != 0
        ? (curTime - stack.mLaunchStartTime) : thisTime;
}
```

