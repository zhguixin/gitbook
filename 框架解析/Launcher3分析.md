Launcher主要的是一个Activity：Launcher，在IntentFilter中包含一个HOME的category：

```xml
<intent-filter>
    <action android:name="android.intent.action.MAIN" />
    <category android:name="android.intent.category.HOME" /><!--开机检测该category -->
    <category android:name="android.intent.category.DEFAULT" />
    <category android:name="android.intent.category.MONKEY"/>
</intent-filter>
```

在Launcher.java的onCreate方法中：

```java
private LauncherModel mModel;
private IconCache mIconCache;

@Override
protected void onCreate(Bundle savedInstanceState) {
    // ...
	LauncherAppState app = LauncherAppState.getInstance();
    mModel = app.setLauncher(this);
    mIconCache = app.getIconCache();
}
```

* LauncherAppState：单例对象，构造方法中初始化对象、注册应用安装、卸载、更新，配置变化等广播。这些广播用来实时更新桌面图标等，其receiver的实现在LauncherModel类中，LauncherModel也在这里初始化。

* LauncherModel：数据处理类，保存桌面状态，提供读写数据库的API，内部类LoaderTask用来初始化桌面

  数据加载完后，通过Handler回调到UI线程，Launcher实现了回调，进行界面的显示

* IconCache：图标缓存类，应用程序icon和title的缓存，内部类创建了数据库app_icons.db。

参考链接：[Launcher3加载流程](https://blog.csdn.net/yanbober/article/details/50525559)