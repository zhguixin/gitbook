Android开发调试技巧

##### adb常用命令

```bash
adb shell logcat -c # 清除log缓存区

adb shell logcat | grep '关键词' #查询相关关键词的log

adb shell logcat -s "xxx" #输出当前以xxx为TAG的日志

adb shell input keyevent 26 # keycode为26的按键

adb shell am broadcast -a android.intent.action.AIRPLANE_MODE --ez state true
--ei<int>
--es<string>

adb shell dumpsys notification > notification.txt # 查看通知栏通知的相关信息

adb shell settings list global  # 查询settings_global.xml下key、value值
adb shell settings list system 

在Andorid N（及以上）设备上打开Freeform模式很简单，只需以下两个步骤：
	执行以下命令：adb shell settings put global enable_freeform_support 1
	然后重启手机：adb reboot

adb shell dumpsys meminfo com.android.systemui # 查看某个包名对应进程的内存信息
adb shell dumpsys activity activities | grep systemui

```

查看进程信息：

```bash
ps -e | grep com.zte.mfvkeyguard
u0_a36       11248  1609 1686280  46460 SyS_epoll_wait f2325f94 S com.zte.mfvkeyguard

adb shell am dumpheap 11248  /data/local/tmp/dumpheap/com.zte.mfvkeyguard.A2.hprof
```

并导出堆内存信息。然后可以通过`hprof-conv.exe` 转化后，用MAT分析堆内存信息。

##### 堆栈信息打印

在某个函数中，打印出当前线程的调用栈信息：

```java
@Override
public boolean onTouchEvent(MotionEvent event) {
    Log.d(TAG, "onTouchEvent, ev=" + event.getAction());
    // 调用栈信息打印
    Thread.dumpStack();
    return true;
}
```

