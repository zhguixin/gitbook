###Android四大组件之Service

#### 1、概述

Service不同于Activity，对用户不可见，也无法与用户直接沟通。启动Service的方式有两种：

- 通过Context，调用**startService**
- 通过Context，调用**bindService**

Service的生命周期函数有：onCreate、onStartCommand、onDestory。

无论调用几次startService，Service只保持一个，并且通过调用stopService来停止Service；通过bindService方法启动的Service，调用进程结束后，Service也就销毁了。

在Service每一次的开启关闭过程中，只有onStartCommand可被多次调用(通过多次startService调用)，其他onCreate，onBind，onUnbind，onDestory在一个生命周期中只能被调用一次。

如果先通过startService启动了Service，再通过bindService来绑定Service时，直接调用到Service的onBind()。

反过来：先通过bindService启动了Service，再通过startService来绑定Service时，直接调用到Service的onStarCommandt()的顺序不同。注意：*此时无法通过stopSevice来停止Service，必须先调用unBindService方法在调用stopService方法。*

![img](./res/service_lifecircle.png)

Service同样运行与UI线程不能进行耗时的操作，如果需要在Service下进行耗时操作，就要使用IntentService。

Service的典型使用场景：不需要通过UI与用户交互，但是又要长时间在后台运行（Activity退出），比如后台音乐播放、后台下载。

如果只需要在当前界面去做一些耗时操作，界面退出或改变时，工作也要停止，那么这时直接使用Thread（或者AsyncTask, ThreadHandler）会更加合适。

### Android四大组件之BroadCast

#### 1、概述

广播Android中的使用可以最大限度实现模块之间的解耦。发送广播有两种方式：

- 直接调用Context的接口，sendBroadcast
- 直接调用Context的接口， sendOrderBroadcast

分别对应了两种不同的广播类型：

- 无序广播，会异步的发送给所有的Receiver，接收到广播的顺序是不确定的，有可能是同时
- 有序广播，广播会先发送给优先级高(android:priority)的Receiver，而且这个Receiver有权决定是继续发送到下一个Receiver或者是直接终止广播

广播的注册也有两种方式：动态注册和静态注册。

- [ ] `LocalBroadcastManager`类实现了只给本进程发送广播，待研究；

广播的生命周期，即为onReceive方法，当它的onReceive方法执行完成后，它的生命周期就结束了。这时BroadcastReceiver已经不处于active状态，被系统杀掉的概率极高。如果在onReceive中执行耗时操作超过**5秒**就会发送ANR。

### Android四大组件之Activity

#### 1、概述

Activity作为Android开发中最常使用的组件，主要就是通过界面与用户打交道。首先看一下它的生命周期：

![img](./res/Activity_lifecircle.png)

Activity 的**可见生命周期**发生在 onStart调用与 onStop调用之间。在这段时间，用户可以在屏幕上看到 Activity 并与其交互。 onStart的时候，是不可交互的，只是可见，只有到了onResume才是可交互的状态。

调用AlertDialog.Builder形式dialog是不会触发生命周期的，调用activity形式的dialog才会触发生命周期。

Activity的四种启动模式，详细内容参考官网[Task and Back Stack](https://developer.android.com/guide/components/tasks-and-back-stack.html)：

- standard
- singleTop
- singleTask
- singleInstance

注：**项目中遇到过一个问题，快速点击一个按钮启动Activity时，会导致启动两个Activity。这就是采用了默认的Activity的standard启动模式，改为singleTop的模式就OK了！**

#### Android四大组件之ContentProvider

#### 1、概述

Android设计该组件的意义：

> 1. 封装。对数据进行封装，提供统一的接口，使用者完全不必关心这些数据是在DB，XML、Preferences或者网络请求来的。当项目需求要改变数据来源时，使用我们的地方完全不需要修改。
> 2. 提供一种跨进程数据共享的方式。

还可以实现数据更新通知机制：通过ContentResolver接口的notifyChange函数来通知那些注册了监控特定URI的ContentObserver对象，使得它们可以相应地执行一些处理。

在AndroidManifest中注册该组件时，有些属性需要了解：

- android:exported，这个属性用于指示该服务是否能够被其他应用程序组件调用或跟它交互。如果设置为true，则能够被调用或交互，否则不能。设置为false时，只有同一个应用程序的组件或带有相同用户ID的应用程序才能启动或绑定该服务。
- android:multiprocess，该属性的默认值是false，表示ContentProvider是单例的，无论哪个客户端应用的访问都将是同一个ContentProvider对象；如果设为true，系统会为每一个访问该ContentProvider的进程创建一个实例。

调用getContentResolver().query/insert/update/delete时，如果是在同一个进程是和调用者在同一个线程；如果是其他进程调用，则是运行在对应进程的binder线程。**在未返回结果时，都会阻塞调用线程**

ContentResolver虽然是通过Binder进程间通信机制打通了应用程序之间共享数据的通道，但Content Provider组件在不同应用程序之间传输数据是基于匿名共享内存机制来实现的。

**ContentProvider的onCreate方法要比Application的onCreate方法执行的要早。**

关于数据库的操作，如果一次性插入很多条记录，可以重用SQLiteStatement，使用SQLiteDatabase的beginTransaction()方法开启一个事务。

```java
try {
        sqLiteDatabase.beginTransaction();
        SQLiteStatement stat = sqLiteDatabase.compileStatement(insertSQL);

        // 插入10000次
        for (int i = 0; i < 10000; i++) {
            stat.bindLong(1, 123456);
            stat.bindString(2, "test");
            stat.executeInsert();
        }
        sqLiteDatabase.setTransactionSuccessful();
    }
    catch (SQLException e) {
        e.printStackTrace();
    }
    finally {
        // 结束
        sqLiteDatabase.endTransaction();
        sqLiteDatabase.close();
    }
```

*参考代码：[Android面试一天一题之SQLite数据库](http://www.jianshu.com/p/2398aad3bd61)*

