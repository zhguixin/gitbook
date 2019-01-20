#### 1、环境搭建

需要先安装的Node.js，安装完成之后，通过命令行：

```bash
npm install -g react-native-cli
```

配置安装React-Native（后续简称RN）的环境。有了这个RN的环境，在命令行中初始化创建一个RN工程：

```bash
react-native init <project-name>
```

在执行这条命令后，就初始化了一个RN工程，主要的目录有：

* android文件夹，存放原生Android的代码

* ios，存放原生IOS相关代码

* node_moudle，RN项目的依赖，后续添加的依赖也会放到该目录下，该文件是要放到.gitignore中的

* App.js，入口界面的具体实现

* app.json，应用名称的配置

* index.js，应用的入口界面，具体实现在App.js文件里

* package.json，项目的相关依赖，类似于Android中的build.gradle

  > package.json文件中如果要添加相关依赖，不需要手动添加，在命令行中运行：
  >
  > npm install <name> --save，即可会自动下载到node_module文件夹内，更新package.json文件；
  >
  > react-native link，依赖含有native代码，添加到native代码中的 getPackages()中。

创建完成工程后，直接在项目根目录运行：（在此只演示运行Android项目）

```bash
react-native run-android
```

#### 2、React Android分析

创建好工程后，去Android文件下，发现自动创建了一个`MainApplication` 和 `MainActivity` 。

这两个类对于Android开发者已经很熟悉了，但是在这两个类中涉及到几个React-Native的类，整体概括一下：

* ReactActivity，将Activity的生命周期转交给ReactActivityDelegate
* ReactActivityDelegate，得到在MainApplication中ReactNativeHost，进而得到ReactInstanceManager，然后将生命周期相关的调用又转移到了ReactInstanceManager中


* ReactNativeHost，抽象类实例化时，覆写`getPackages` 方法，返回向JS侧注册的原生模块。在该类中存放着ReactInstanceManager类的实例
* ReactInstanceManager，通过建造者模式创建，在该类中又会构建CatalystInstanceImpl
* CatalystInstanceImpl，该类中存在大量native方法，与JNI层进行交互通信



原生Android层与JS层之间的相互通信，通过的是一个由C++语言实现的Bridge层。

首先看看，Java层（也就是Android层）调用JS层（JavaScript层）的组件。在**CatalystInstanceImpl**类中的

`getJsModule()` 方法中，通过分析可知，这种调用的过程最终由名为：JavaScriptModuleInvocationHandler的动态代理类统一拦截处理：

```java
    @Override
    public @Nullable Object invoke(Object proxy, Method method, @Nullable Object[] args)
        throws Throwable {
      NativeArray jsArgs = args != null
        ? Arguments.fromJavaArgs(args)
        : new WritableNativeArray();
      mCatalystInstance.callFunction(getJSModuleName(), method.getName(), jsArgs);
      return null;
    }
```

拦截得到，JS的组件名、方法名和参数列表，直接调用到了**CatalystInstanceImpl** 类的`callFunction` 方法。

其中的参数列表根据`args` 是否为空，构造了不同的`jsArgs` ：

args不为空，jsArgs = Arguments.fromJavaArgs(args)；反之，args为空，直接实例化了WritableNativeArray赋值为jsArgs。

最终，将参数列表中的值都push到了Bridge层。这些值都只在C++侧保存一份，从一定程度上降低了内存占用。