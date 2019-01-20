---
title: Android的内容提供器
date: 2016-08-20
tags: Android
categories: 
---

### 什么是内容提供器
内容提供器**ContentProvider**用于在不同的应用程序之间实现数据的共享功能，允许一个程序访问另一个程序中的数据，同时还能保证被访问数据的安全性。

Setting里面的一些设置也是通过数据库来保存的，通过数据库来保存就会有Provider了，所以就会有SettingsProvider了。数据库的路径就是：`/data/data/com.android.providers.settings`.
但我们平常获取这里面的数据不是直接通过ContentResolve而是android已经封装了一层，通过Settings这个类来获取，

```java
Settings.System.getInt(mContext.getContentResolver(),
                Settings.System.AIRPLANE_MODE_ON, 0)
```

#### 设计意图

Android设计该ContentProvider组件的意义：

> 1. 封装。对数据进行封装，提供统一的接口，使用者完全不必关心这些数据是在DB，XML、Preferences或者网络请求来的。当项目需求要改变数据来源时，使用我们的地方完全不需要修改。
> 2. 提供一种跨进程数据共享的方式。

还可以实现数据更新通知机制：通过ContentResolver接口的notifyChange函数来通知那些注册了监控特定URI的ContentObserver对象，使得它们可以相应地执行一些处理。

#### 基本使用

在AndroidManifest中注册该组件时，有些属性需要了解：

- android:exported，这个属性用于指示该服务是否能够被其他应用程序组件调用或跟它交互。如果设置为true，则能够被调用或交互，否则不能。设置为false时，只有同一个应用程序的组件或带有相同用户ID的应用程序才能启动或绑定该服务。
- android:multiprocess，该属性的默认值是false，表示ContentProvider是单例的，无论哪个客户端应用的访问都将是同一个ContentProvider对象；如果设为true，系统会为每一个访问该ContentProvider的进程创建一个实例。

调用`getContentResolver().query/insert/update/delete`时，如果是在同一个进程是和调用者在同一个线程；如果是其他进程调用，则是运行在对应进程的binder线程。**在未返回结果时，都会阻塞调用线程**

ContentResolver虽然是通过Binder进程间通信机制打通了应用程序之间共享数据的通道，但Content Provider组件在**不同应用程序之间传输数据是基于匿名共享内存机制**来实现的。

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