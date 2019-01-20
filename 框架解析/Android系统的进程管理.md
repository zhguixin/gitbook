在AMS中，每一个运行中的进程都对应着一个`ProcessRecord`。

```java
    // all activities running in the process
    final ArrayList<ActivityRecord> activities = new ArrayList<>();
    // all ServiceRecord running in this process
    final ArraySet<ServiceRecord> services = new ArraySet<>();
    // services that are currently executing code (need to remain foreground).
    final ArraySet<ServiceRecord> executingServices = new ArraySet<>();
    // All ConnectionRecord this process holds
    final ArraySet<ConnectionRecord> connections = new ArraySet<>();
    // all IIntentReceivers that are registered from this process.
    final ArraySet<ReceiverList> receivers = new ArraySet<>();
    // class (String) -> ContentProviderRecord
    final ArrayMap<String, ContentProviderRecord> pubProviders = new ArrayMap<>();
    // All ContentProviderRecord process is using
    final ArrayList<ContentProviderConnection> conProviders = new ArrayList<>();
```



进程的启动，停止需要调用`Process` 中的方法。