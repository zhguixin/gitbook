在Android平台端

#### ReactActivity类

由代理类：ReactActivityDelegate完成主要操作。

#### ReactActivityDelegate类

ReactActivityDelegate中得到Application的ReactNativeHost，在ReactNativeHost中实例化ReactInstanceManager传入到ReactRootView 中。调用过程：

`onCreate() --> loadApp()-->mRootView.startReactApplication()` 

#### ReactRootView类

该类是一个继承自FrameLayout的自定义View，是RN界面的最顶层的View，主要负责进行监听分发事件，渲染元素等工作。整个工程若只有部分页面是RN界面，可以通过该自定义View实现。

```java
mReactRootView = (ReactRootView) findViewById(R.id.react_root_view);
mReactInstanceManager = ReactInstanceManager.builder()
     .setApplication(getApplication())
     .setBundleAssetName("index.android.bundle")
     .setJSMainModuleName("index")
     .addPackage(new MainReactPackage())
     .addPackage(new NativeReactPackage()) // 新添加需要注册的模块
     .setUseDeveloperSupport(BuildConfig.DEBUG)
     .setInitialLifecycleState(LifecycleState.RESUMED)
     .build();
mReactRootView.startReactApplication(mReactInstanceManager, "ReactNativeProject",
                                  getLaunchOptions());
```

主要通过调用`startReactApplication`方法来启动ReactNative的运行时环境。

在startReactApplication函数中先判断是否初始化了与React通信的环境，没有初始化的话调用：

```java
mReactInstanceManager.createReactContextInBackground();
```

来进行初始化操作，创建**ReactContext**；

接着调用`attachToReactInstanceManager()`，关联ReactInstanceManager到当前ReactRootView，并注册监听器来监听View控件树的变化。

#### ReactInstanceManager类

主要被用于管理`CatalystInstance`实例对象。通过`ReactPackage`，它向开发人员提供了配置和使用`CatalystInstance`实例对象的方法，并对该catalyst实例的生命周期进行追踪。`ReactInstanceManager`实例对象的生命周期与`ReactRootView`所在的activity保持一致。

该类的一个方法：`createReactContextInBackground()`（由**ReactRootView**实例的`startReactApplication`方法调用）来创建ReactNative的上下文环境`ReactContext`。

再次调用方法：

```java
  private ReactApplicationContext createReactContext(
      JavaScriptExecutor jsExecutor,
      JSBundleLoader jsBundleLoader)
```

创建**ReactApplicationContext**，在`createReactContext`方法中主要完成了5项工作：

> 1. 处理modules package
> 2. 建立Native模块的注册表
> 3. 创建catalystInstance实例对象
> 4. 创建并初始化`ReactApplicationContext`
> 5. 执行JsBundle

#### CatalystInstance类

顶级异步JSCAPI封装类，提供Java与Js互通的环境，通过ReactBridge将得到（来自Server或者Assets目录）的Js Bundle传送到Js引擎。

#### JavascriptModuleRegisty

Js层模块注册表，负责将所有JavaScriptModule注册到CatalystInstance，通过Java动态代理调用到Js

#### NativeModuleRegisty

Java层模块注册表，即暴露给Js的API集合。在CatalystInstanceImpl构造函数中，初始化ReactBridge时，向JNI层传入Java侧的Moudle。