### 启动模式

Activity的四种LaunchMode：

- standard模式，设置了该启动模式的Activity在被启动时，会创建一个新的实例，即使在栈顶，也会重复创建新的实例。被启动的Activity与主Activity的TaskId保持一致。

- singleTop模式，设置了该启动模式的Activity在被启动时，顾名思义如果位于栈顶则不会重复创建，即`onCreate()`、`onStart()`方法不会被调用，此时`onNetIntent()`方法会被调用。

  现在考虑这样一种情况：如果栈中有该被启动的Activity实例，但是该实例不在栈顶情况又如何呢？答案是，还会重新创建一个新的Activity实例。

- singleTask模式，设置了该启动模式的Activity在被启动时，如果栈中存在这个Activity的实例就会复用这个Activity，不管它是否位于栈顶，复用时，会将它上面的Activity全部出栈，并且会回调该实例的`onNewIntent()`方法。

  我之前的理解的singleTask就是这样的，但其实忽略掉了一个很重要的问题。在AndroidManifest中配置Activity时有个属性值叫**taskAffinity**，该值的作用是指定任务栈(不同的任务栈有不同的TaskId，不指定的话就是默认包名)。启动指定singleTask模式的Activity的过程还存在一个任务栈的匹配，启动时，会在自己需要的任务栈中寻找实例，这个任务栈就是通过taskAffinity属性指定。如果这个任务栈不存在，则会创建这个任务栈，创建新的Activity实例入栈到新创建的Task中去。 不指定的话就是默认包名。

  我们可以将两个不同App中的Activity设置为相同的taskAffinity，这样虽然在不同的应用中，但是Activity会被分配到同一个Task中去

  **taskAffinity属性不对standard和singleTop模式有任何影响，即时你指定了该属性为其他不同的值，这两种启动模式下也不会创建新的task**

- singleInstance模式，该模式具备singleTask模式的所有特性外，与它的区别就是，这种模式下的Activity会单独占用一个Task栈，具有全局唯一性，即整个系统中就这么一个实例，由于栈内复用的特性，后续的请求均不会创建新的Activity实例，直接调用onNewIntent方法，除非这个特殊的任务栈被销毁了。这个模式的典型应用场景时，在手机某个APP(如短信)点击链接跳转到浏览器，浏览器的实例是全局唯一的

除了上述几种启动模式，接着了解下面这几个Intent标签的用法：

- FLAG_ACTIVITY_NEW_TASK，与在xml中配置`singleTask`一致

- FLAG_ACTIVITY_CLEAR_TOP，启动该Activity时，单独使用时连同它都会清除它之上的Activity，然后再重新创建；配合**FLAG_ACTIVITY_NEW_TASK**使用的话效果同`singleTask`

  ```java
  inttent.setFlags(Intent.FLAG_ACTIVITY_CLEAR_TOP | Intent.FLAG_ACTIVITY_NEW_TASK)
  ```

- FLAG_ACTIVITY_SINGLE_TOP，与在xml中配置`singleTop`一致

这种方式启动的Activity指定的启动模式优先级更高。如果你是在Service调用`context.startActivity()`，还要指定`FLAG_ACTIVITY_NEW_TASK`，否则会报错：

```java
// ContextImpl.java
@Override
public void startActivity(Intent intent, Bundle options) {
    warnIfCallingFromSystemProcess();

    // Calling start activity from outside an activity without FLAG_ACTIVITY_NEW_TASK is
    // generally not allowed, except if the caller specifies the task id the activity should
    // be launched in.
    if ((intent.getFlags()&Intent.FLAG_ACTIVITY_NEW_TASK) == 0
        && options != null && ActivityOptions.fromBundle(options).getLaunchTaskId() == -1) {
        throw new AndroidRuntimeException(
            "Calling startActivity() from outside of an Activity "
            + " context requires the FLAG_ACTIVITY_NEW_TASK flag."
            + " Is this really what you want?");
    }
    // ...
}
```

在Activity中调用`startActivity()`会默认加上`FLAG_ACTIVITY_NEW_TASK`属性。因为从Activity启动另外一个Activity已经会存在一个Task任务栈了。

参考资料：[彻底弄懂Activity四大启动模式](http://blog.csdn.net/mynameishuangshuai/article/details/51491074)、[Tasks and Back Stack](https://developer.android.com/guide/components/tasks-and-back-stack.html)

### 生命周期

####正常切换情况下Activity的生命周期

假设正常启动情况下，从Activity A 启动 Activity B：

```bash
A:onPause -> B:onCreate -> B:onStart -> B:onResume -> A:onStop
```

可以看到`onPause()`会先被执行，因此为了确保要启动的Activity能尽快的启动起来，`onPause()`方法不能做太过繁重的任务。并且，Activity A的`onStop()`也是在Activity B启动可见会才会执行，更确保了待启动Activity的尽快呈现。

> onStart()/onStop()表示的是Activity的可见状态；onResume()/onPause()表示的是Activity是否获取了焦点，在Android7.0的分屏状态下，感受更明显。

#### 分屏模式下Activity的生命周期

**场景1：由全屏长按Recent键进入分屏：**

```
onPause->onMultiWindowChanged->onStop->onDestroy->onCreate->onStart->onResume->onPause(焦点切入到另一屏)
```

**场景2：分屏窗口间来回切换焦点：**

```
onPause->onResume来回交替
```

因此，如果是播放类的app, 暂停不能放在onPause里面

**场景3：拖到分屏窗口到1/3或2/3**

生命周期都要销毁创建，进入到onPause或者onResume(取决于是否有焦点)

**场景4：按home键回到Launcher界面**

之前的窗口如果有焦点，则回调onPause先进入"暂停状态"；之前的窗口就没有焦点，则不发生生命周期的变化；

**场景5：退出分屏**

窗口销毁重建进入onResume->onMultiWindowChanged

#### 异常情况下的生命周期

系统配置发生变化时比如语言切换、屏幕旋转，或者资源内存不足被杀死时，Activity在会销毁创建，并会在`onStop()`之前调用`onSaveInstanceState()`方法，重建后会在`onStart()`之后调用`onRestoreInstanceState()`方法。

`onRestoreInstanceState(Bundle savedInstanceState)` 、`onCreate(Bundle savedInstanceState)`两个方法都有一个`savedInstanceState`参数，但是`onCreate()`该有可能为null，取出时需要做非空判断。

如果要想系统配置发生变化时，Activity不销毁重建需要在AndroidManifest中的该Activity标签下配置

```
android:configChanges="orientation|screenSize"
```

> 在Activity中还有一个PhoneWindow给的回调方法：`onWindowFocusChanged()`，在Activity获取焦点时被调用，并传入参数false。如果Activity被意外销毁了，`onWindowFocusChanged()`并不会被再次调用，而是再重建时，重新调用并传入true。

### Activity匹配规则

在AndroidManifest中的Activity标签下，有个IntentFilter可以隐式的匹配Activity。匹配过来信息有：action、category和data

需要匹配该Activity需要保证

- action，存在一个指定项目且相同
- category，要启动该Activity的Intent只要含有category，则要全部匹配相同
- data，同action，可以指定URL协议、主机、资源类型等

