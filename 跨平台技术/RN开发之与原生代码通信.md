RN开发之与原生代码通信

#### Native传递信息到React端

在React端进行编码时，会通过props将父组件的信息传递到子组件。

在native端（考虑Android平台）启动的ReactActivity时有一个getLaunchOptions()方法，该方法返回值，就会传递到React端，作为整个父组件的属性。

如果不想要整个页面都通过React Native，则可以通过Native端的自定义View——ReactRootView来限制使用React开发的区域。在ReactRootView中同样提供了，向React父组件传递属性的方法：

```java
Bundle updateProps = mReactRootView.getAppProperties();
updateProps.putStringArrayList("image","http://test/1.jpg");
mReactRootView.setAppProperties(updateProps);
```

在React端就可以获得该属性：

```react
this.props.image
```

#### React端传递信息到Native端

以上简单信息的互相传输有时并不能满足一些开发需求，比如在React端复用在Native端已经开发好的一些模块，就需要在React端去调用这些已用的方法；又比如在Native端移除掉ReactRootView时，给React端一个通知去做一些处理等等。这样又有两种情况：React端调用Native端的方法；Native端调用React端的方法。

#### React端调用Native端的方法

以官方给出的一个示例研究：通过js代码调用Android平台的Toast模块。

- 定义**ReactContextBaseJavaModule**模块：

  1. 在android的目录下定义一个类去继承**ReactContextBaseJavaModule**类,实现`getName()`方法。此方法返回的参数就是你使用js调用的模块名。
  2. 根据自己的需求选择是否去重写`getConstants()`方法，这个方法的作用是给js提供一些常量使用，通过map的键值去存储，map的键是js调用的名称，key是对应的值。
  3. 然后就是定义你对js提供的方法，对js提供的方法必须是没有返回值的，如果需要有返回值的话，则需要增加Callback参数，js通过callback获取函数需要返回的值。需要注意的是必须在方法上面加上**@ReactMethod注解**，才能让js识别调用。

- 定义好模块之后，接下来就需要去注册模块。非常简单，定义一个类去实现，**ReactPackage**接口。这个接口只有两个方法，`createViewManagers()`该方法是注册需要被调用原生控件的，`createNativeModules()`就是注册原生方法需要实现的，他的返回值是个list，创建一个list然后把你之前定义的**ReactContextBaseJavaModule**添加进去就好。

- 上一步定义好的**ReactPackage**模块在application下注册

  ```java
  public class MainApplication extends Application implements ReactApplication {
  
      private final ReactNativeHost mReactNativeHost = new ReactNativeHost(this) {
          @Override
          public boolean getUseDeveloperSupport() {
              return BuildConfig.DEBUG;
          }
  
          @Override
          protected List<ReactPackage> getPackages() {
              return Arrays.<ReactPackage>asList(
                      new MainReactPackage(),
                      new NativeReactPackage() // 新添加需要注册的模块
              );
          }
      };
  ```

  > 整个项目都是RN工程，可在Application中注册。如果只有部分是RN的页面，即通过**ReactRootView**是实现。
  >
  > 注册模块的方法如下：
  >
  > ```java
  > mReactRootView = (ReactRootView) findViewById(R.id.react_root_view);
  > mReactInstanceManager = ReactInstanceManager.builder()
  >         .setApplication(getApplication())
  >         .setBundleAssetName("index.android.bundle")
  >         .setJSMainModuleName("index")
  >         .addPackage(new MainReactPackage())
  >         .addPackage(new NativeReactPackage()) // 新添加需要注册的模块
  >         .setUseDeveloperSupport(BuildConfig.DEBUG)
  >         .setInitialLifecycleState(LifecycleState.RESUMED)
  >         .build();
  > mReactRootView.startReactApplication(mReactInstanceManager, "ReactNativeProject",
  >                                      getLaunchOptions());
  > ```

- 原生端工作已完成。接下来就是js端，只需要加载NativeModules模块，
  从NativeModules中获取之前定义的ReactContextBaseJavaModule对应的getName的值就可以调用对应的方法了

  ```javascript
  import React, {Component} from 'react';
  import {
      AppRegistry,
      StyleSheet,
      Text,
      View,
      NativeModules // 此模块
  } from 'react-native';
  
  ```

#### Native端调用React端的方法

##### 回调方式实现

Native端与React端之间有C++曾作为bridge，React注册JSModule到bridge，Native曾通过JNI调用到bridge，间接调用JSModule。

开发过程中Native调用React时，可由React调用Native时传入的回调函数返回。

##### 调用方式实现

1. 在Java侧定义一个接口类，比如AppRegister类：

```java
public interface AppRegistry extends JavaScriptModule {
  void runApplication(String appKey, WritableMap appParameters);
  void unmountApplicationComponentAtRootTag(int rootNodeTag);
  void startHeadlessTask(int taskId, String taskKey, WritableMap data);
}
```

调用**CatalystInstanceImpl**类的`getJSModule`方法：

```java
public <T extends JavaScriptModule> T getJSModule(Class<T> jsInterface);
// 传参调用如下：
catalystInstance.getJSModule(AppRegistry.class).runApplication(jsAppModuleName, appParams);
```

> 该方法最终会通过动态代理的方式调用**CatalystInstance**类中的`callFunction` 方法，通过JNI调用执行对应js文件：
>
> ```java
> private native void jniCallJSFunction(String module, String method, NativeArray arguments);
> ```

另外，**ReactContext**封装了**CatalystInstanceImpl**类的`getJSModule`方法，方便供外部调用。

2. 在Js侧，定义**AppRegistry.js**文件，并实现对应方法。

#### Native主动发消息到React侧

调用方法如下：

```java
private void sendMessageToReact(String message) {
    mReactContext.getJSModule(DeviceEventManagerModule.RCTDeviceEventEmitter.class)
        .emit("AndroidToReact", message);
}
```

React侧实现：

```java
constructor(props) {
    super(props);
    DeviceEventEmitter.addListener('AndroidToReact', this.handleAndroidMessage.bind(this));
}

handleAndroidMessage(message) {
    alert(message);
}
```

