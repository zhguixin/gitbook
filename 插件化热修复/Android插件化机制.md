使用动态代理机制，拦截某个方法，在方法执行前和执行后可以做点手脚。同样可以自己创建对象，用创建的代理对象来替换掉原来的对象。这种方式称为Hook。

良好的Hook点是不容易发生变化的静态变量对象和单例对象。

比如，拦截Activity的启动过程，每次启动Activity之前打印一个log。

正常的Activity启动过程：

```java
// ContextImpl
@Override
public void startActivity(Intent intent, Bundle options) {
    warnIfCallingFromSystemProcess();
    // 在Service等非Activity的Context里面启动Activity需要添加FLAG_ACTIVITY_NEW_TASK
    if ((intent.getFlags()&Intent.FLAG_ACTIVITY_NEW_TASK) == 0) {
        throw new AndroidRuntimeException(
                "Calling startActivity() from outside of an Activity "
                + " context requires the FLAG_ACTIVITY_NEW_TASK flag."
                + " Is this really what you want?");
    }
    mMainThread.getInstrumentation().execStartActivity(
        getOuterContext(), mMainThread.getApplicationThread(), null,
        (Activity)null, intent, -1, options);
}
```

其中`mMainThread` 为ActivityThread对象，调用`getInstrumentation()` 方法得到`mInstrumentation` 成员变量。

我们Hook住ActivityThread，替换它的`mInstrumentation` 为自己定义的**HookInstrumentation** 。



插件机制

要启动的Activity存在于独立的文件或者网络中，需要使用插件类加载机制。Android Framework提供了DexClassLoader来动态加载外部的dex或者apk文件。



运行插件中的Activity

插件中的Activity可以通过ClassLoader动态加载到虚拟机里面，但是运行的话无法正常运行，因为没有在AndroidManifest中注册。

```kotlin
fun startActivityInPlugin() {
  val intent = Intent(this@MainActivity, TargetActivity::class.java)
  startActivity(intent)
}
```

提示TargetActivity未注册。

> 动态加载：
>
> （1）自定义类加载器DexClassLoader，告诉DexClassLoader其中`dex`或`apk`文件地址，它就能把里面的类文件加载到虚拟机中了，dex文件或者apk文件要放到私有目录下；（此种方法是新建了一个dexElements数组）[DexClassLoader源码](http://androidxref.com/7.1.2_r36/xref/libcore/dalvik/src/main/java/dalvik/system/DexClassLoader.java)
>
> （2）参考MultiDex.install方法，他会把secondary dex合并到dexElements数组中

要先启动插件中的Activity，目前采用的方式是"预埋"一个`StubActivity`到AndroidManifest中去。具体过程：

（1）在ActivityManagerNative中，hook掉`startActivity`方法，把Intent信息中的**TargetActivity替换为StubActivity**

（2）从AMS所在的system_server所在进程返回到应用进程后，hook掉`H`这个Handler，设置这个Handler的Callback，拦截`LAUNCH_ACTIVITY`的实现，返回true，这样其他操作继续交给了`H`的handleMessage。

> 这个地方的关键点是，Handler的dispatchMessage操作，会先处理Handler的callback，callback返回值为false才继续处理handleMessage
>
> ```java
> public void dispatchMessage(Message msg) {
>   if (msg.callback != null) {
>     handleCallback(msg);
>   } else {
>     if (mCallback != null) {
>       if (mCallback.handleMessage(msg)) {
>         return;
>       }
>     }
>     handleMessage(msg);
>   }
> }
> ```
>
> 

使用插件中的资源

基本思路是反射AssesManager的`addAssetPath`方法，把插件APK路径传入进去，并通过这个新构造的AssetManager构造一个Resource，后面可以通过这个Resource通过资源ID得到对应资源

```java
AssetManager assetManager = AssetManager.class.newInstance();
Class cls = AssetManager.class;
Method method = cls.getMethod("addAssetPath", String.class);
method.invoke(assetManager, path);
Resources resources = new Resources(assetManager, mContext.getResources().getDisplayMetrics(),
                                    mContext.getResources().getConfiguration());
```

