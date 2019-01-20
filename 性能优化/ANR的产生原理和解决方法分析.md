### ANR的产生原理和解决方法分析

---
title: ANR的产生原理和解决方法分析
date: 2017-10-9
tags: Java
categories: 

---

#### 1、简述

ANR，即Application Not Response，是由于在UI线程进行耗时操作，导致没有在规定的时间内完成，为提高用户体验，Android框架会弹出一个Dialog来提示用户“应用程序长时间无响应，是等待或是强行关闭”。如下几种常见的操作会导致ANR：

- 输入事件的处理没有在5秒内处理完毕；
- 广播处理`onReceive()`过程没有在10秒内处理完毕；
- 后台服务`Service`没有在20秒内处理完毕；

#### 2、原理分析

关于输入事件产生的ANR，在`InputManagerService`中调用`notifyANR()`，

获取不到目标窗口时，超过5秒钟会发生ANR。

服务长时间未处理完成的话，在**AMS**中的`realStartServiceLocked()`中发送一个延时20秒的Msg，服务处理完毕后调用`removeMsg()`来移除这个消息。

发生ANR后，在AMS中的`appNotResponding()`方法中会将ANR的相关信息进行记录。分析相关ANR的log时，在**/data/anr/traces.txt** 文件中，无需root即可将该log导出。

#### 3、ANR分析

CPU负荷过高，导致时间片没有轮转到事件处理，导致长时间没有反应。

由于事件得不到及时处理，导致的ANR对应于InputDispatcher.cpp中的两段代码：

* no focused window or no focused application，对应的ANR Reason，有**no focuse**关键字，这种情况，一般发生应用窗口启动时，启动时间过长，在这段时间有keyevent事件触发，得不到及时响应。发生在Monkey测试中比较多
* 前面的事件是否及时完成，对应的ANR Reason：Input dispatching time out。这种情况，表示主线程正在执行其他耗时事件，来不及处理后续事件

对于已经发生的ANR，可以通过`/data/anr/traces.txt` 文件进行分析。