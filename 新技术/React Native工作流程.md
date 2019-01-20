在Android平台端

**ReactActivityDelegate**类

ReactActivityDelegate中得到Application的ReactNativeHost，在ReactNativeHost中实例化ReactInstanceManager传入到ReactRootView 中。调用过程：

`onCreate() --> loadApp()-->mRootView.startReactApplication()` 

**ReactRootView**类

在startReactApplication函数中先判断是否初始化了与React通信的环境，没有初始化的话调用：

`mReactInstanceManager.createReactContextInBackground();` 

进行初始化操作，接着调用attachToReactInstanceManager()把ReactRootView与CatalystInstance相关联，达到与JS通信渲染结果的目的，如果是第一次安装会直接调用到runApplication()方法，进而调用到React端的入口函数：index.js

