基本使用流程，见参考文档。

LeakCanary在应用中配置好后（Debug版本），寻找能够触发内存泄漏的场景进行手动测试。如果有内存会在通知栏中弹出一个通知。

具体原理：

LeakCanary默认监测Activity泄漏。在Application的OnCreate中配置：

```java
    @Override
    public void onCreate() {
        super.onCreate();
        refWatcher = LeakCanary.install(this);
    }
```

install中会为Application注册ActivityLifecycleCallbacks，监听Activity的onDestroy方法：

```java
@Override public void onActivityDestroyed(Activity activity) {
    ActivityRefWatcher.this.onActivityDestroyed(activity);
}

void onActivityDestroyed(Activity activity) {
    refWatcher.watch(activity);
}
```

watch方法主要功能：

* 创建一个弱引用(KeyedWeakReference)到要被监控的对象
* 在后台检查引用是否被清除，如果没有清除，主动触发一次GC
* 如果引用还是没有被清除，把内存信息dump出来
* 由`ServiceHeapDumpListener` 后台解析这个内存信息
* 分析完成后传递到`DisplayLeakService`， 并以通知的形式展示出来

源码比较简单，可结合源发分析。

#### LeakCanary是怎么检查引用是否被清除的？

由于在watch会创建一个到被监控Activity的弱引用：

```java
queue = new ReferenceQueue<>();   

final KeyedWeakReference reference =
        new KeyedWeakReference(watchedReference, key, referenceName, queue);
```

被gc回收的话，会将该引用加入引用队列`ReferenceQueue` 。

传入的key也是唯一的：

```java
retainedKeys = new CopyOnWriteArraySet<>();

String key = UUID.randomUUID().toString();
retainedKeys.add(key);
```

检查引用是被清除的关键点：

```java
void ensureGone( ) {
    // ...
    removeWeaklyReachableReferences();
    if (gone(reference) ) {}
    // ...
}

// 如果引用已经加入引用队列，移除引用对应的key
private void removeWeaklyReachableReferences() {
    // WeakReferences are enqueued as soon as the object to which they point to becomes weakly
    // reachable. This is before finalization or garbage collection has actually happened.
    KeyedWeakReference ref;
    while ((ref = (KeyedWeakReference) queue.poll()) != null) {
        retainedKeys.remove(ref.key);
    }
}

// 如果引用key在 retainedKeys 数组中，引用没有被清除，说明发生内存泄漏
private boolean gone(KeyedWeakReference reference) {
    return !retainedKeys.contains(reference.key);
}
```



参考文章：[LeakCanary使用说明](https://www.liaohuqiu.net/cn/posts/leak-canary-read-me/)