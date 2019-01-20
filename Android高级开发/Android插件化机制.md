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