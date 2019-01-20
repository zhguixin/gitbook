Android系统是基于Linux内核的，而在Linux系统中，所有的进程都是init进程的子孙进程。在Android系统也不另外，作为Android系统中最重要的进程Zygote进程，也是init进程创建的。

>  init是Linux系统中用户空间的第一个进程(pid=1), Kerner启动后会调用/system/core/init/Init.cpp的main()方法.启动脚本init.rc就在main方法中被解析执行。这都属于Linux内核的操作流程

系统启动脚本**init.rc**：

```shell
# system/core/rootdir/init.rc
service zygote /system/bin/app_process -Xzygote /system/bin --zygote --start-system-server  
    socket zygote stream 666  # 创建socket
    onrestart write /sys/android_power/request_state wake  
    onrestart write /sys/power/state on  
    onrestart restart media  
    onrestart restart netd  
```

> 参考资料：[init脚本语法](http://gityuan.com/2016/02/05/android-init/)

由启动脚本可知，启动了一个叫`zygote` 的服务进程，这个服务进程是由可执行程序：`system/bin/app_process` 创建的，创建的时候传入了参数：` --zygote --start-system-server ` 。

> init进程还会启动servicemanager进程，运行在system_server中的服务都要注册到此进程。

这个可执行程序对应的源代码位于：`frameworks/base/cmds/app_process/app_main.cpp`文件中，入口函数是main。

Zygote进程在启动的过程中，会调用到AndroidRuntime类的成员函数start，然后启动虚拟机。

```java
App_main.main
    AndroidRuntime.start
        AndroidRuntime.startVm // 启动虚拟机
        AndroidRuntime.startReg // 注册JNI函数
        ZygoteInit.main //(首次进入Java世界)
            registerZygoteSocket // 注册Zygote Socket，监听套接字请求，创建应用进程
            preload // 预先加载公共资源，提供应用开启速度
            startSystemServer // 由zygote fork 出system_server进程
            runSelectLoop // 等待请求消息
```

#### system_server进程

startSystemServer() -->> Zygote.forkSystemServer() -->> nativeForkSystemServer()

最终调用到了native方法：`nativeForkSystemServer` 来启动system_server进程。

```c++
// base/core/jni/com_android_internal_os_Zygote.cpp
// JNI函数映射表
static const JNINativeMethod gMethods[] = {
	// ...
    { "nativeForkSystemServer", "(II[II[[IJJ)I",
      (void *) com_android_internal_os_Zygote_nativeForkSystemServer },
    // ...
}
```

在`com_android_internal_os_Zygote_nativeForkSystemServer` 函数中调用了` ForkAndSpecializeCommon` ：

```c++
static pid_t ForkAndSpecializeCommon() {
    // 设置信号处理, 如果system_server进程挂了，zygote进程也kill掉自己
    SetSigChldHandler();
    // fork子进程
    pid_t pid = fork();
}
```

在native侧fork system_server进程后，再回到startSystemServer方法：

```java
private static boolean startSystemServer(String abiList, String socketName) {
    pid = Zygote.forkSystemServer(...);
    
    if (pid == 0) {
       // ...
        handleSystemServerProcess(parsedArgs);
    }
}
```

handleSystemServerProcess函数定义如下：

```java
private static void handleSystemServerProcess(ZygoteConnection.Arguments parsedArgs) {
    // 关闭从socket那里继承下来的socket
    closeServerSocket();
    
    if (parsedArgs.invokeWith != null) {
        WrapperInit.execApplication(parsedArgs.invokeWith, /*...*/);
    } else {
        RuntimeInit.zygoteInit(parsedArgs.targetSdkVersion, parsedArgs.remainingArgs, cl);
    }
}
```

`zygoteInit` 函数定义在`base/core/java/com/android/internal/os/RuntimeInit.java` ：

```java
public static final void zygoteInit(/*..*/) {
    // ...
    commonInit();
    // native层初始化，开启与Binder通信的线程等
    nativeZygoteInit();
    // 反射调用SystemServer.java的main方法
    applicationInit(targetSdkVersion, argv, classLoader);
}

```

重点看一下SystemServer类：

```java
    public static void main(String[] args) {
        new SystemServer().run();
    }

	private void run() {
        // 设置系统时间：1970年
        SystemClock.setCurrentTimeMillis(EARLIEST_SUPPORTED_TIME);
        
        //...
        // 加载libandroid_servers.so库
        System.loadLibrary("android_servers");
        // 创建SystemServiceManager
        mSystemServiceManager = new SystemServiceManager(mSystemContext);
        
        // 启动各种服务
        startBootstrapServices();
        startCoreServices();
        // 在这里启动SystemUI
        // 向SystemServerManager注册添加各种服务
        startOtherServices();
    }
```



