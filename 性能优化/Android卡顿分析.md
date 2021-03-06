界面卡段的分析基本思路：

在开发者选项中，打开：【调试GPU过度绘制】和【GPU呈现模式分析】的开关。直观的观察界面的绘制和掉帧情况。

借助于**Hierarchy View**详细的观察分析，布局层级关系。可以得到onMeasure、onLayout、onDraw的各个时间，总和超过16ms，掉帧就比较严重了。

借助于**systrace view**详细分析掉帧结果。

Systrace的基本原理是在系统的一些关键链路（比如System Service，虚拟机，Binder驱动）插入一些信息（称之为Label），通过Label的开始和结束来确定某个核心过程的执行时间，然后把这些Label信息收集起来得到系统关键路径的运行时间信息，进而得到整个系统的运行性能信息。Android Framework里面一些重要的模块都插入了Label信息（Java层的通过android.os.Trace类完成，native层通过ATrace宏完成），用户App中可以添加自定义的Label，这样就组成了一个完成的性能分析系统。

```java
Trace.beginSection("begin");
// 内部代码
Trace.endSection();
```

> Systrace对系统版本有一个要求，就是需要Android 4.1以上，最好是Android 4.3以上。

借助于**trace view**详细分析线程和函数的执行时间。

TraceView试图收集某个阶段所有函数的运行信息，比较影响性能，Google后续全力支持Systrace。

#### Hierarchy View的使用



#### Systrace的使用

systrace的功能包括跟踪系统的I/O操作、内核工作队列、CPU负载以及Android各个子系统的运行状况等。

借助于Android Device Monitor工具，点击红圈内的图标可以打开systrace工具

![monitor](.\imgs\monitor.JPG)

配置好后，输出的`trace.html`文件用Chrome浏览器打开。

![systrace](.\imgs\systrace.JPG)

在**Frames**那一行，直观的显示了每一帧的渲染，红色代表有卡顿，发生丢帧，渲染时长已经超过了16ms。

在浏览器打开后，常用快捷键有：

* w、s、a、d，实现放大缩小、左右移动；
* f，放大选中区域
* 0（是数字零）恢复到初始状态
* g，显示16ms分界线
* ？，显示帮助信息，方便查看其它快捷键；

 Systrace 是一个总览工具，查看CPU时间片的具体分配，观察每个方法的具体调用花费时长，可以借助于TraceView。

#### TraceView的使用

TraceView可以在代码中加入，需要检测的方法调用前、调用后分别加入：

```java
Debug.startMethodTracing("test");// 会生成：/sdcard/test.trace文件
// 方法调用
Debug.stopMethodTracing(); 
```

另外一个就是在Android SDK的tools目录下的monitor工具中，选中待调试进程，点击：【start Method Profiling】按钮，操作后，再点击一次即可结束。

![](.\imgs\monitor_trace.JPG)

通过TraceView，我们可以找到频繁调用的方法，也可以找到耗时方法，来分析解决潜在的卡顿问题。

![](.\imgs\traceview.jpg)

在打开的TraceView视图中，分为上下两部分：

* 时间线面板以每个线程为一行，右边是该线程在整个过程中方法执行的情况，每一个小立柱的颜色代表一个方法
* 下面是方法详细面板，以表格的形式展示所有线程的方法的各项指标

具体使用参考：[TraceView工具使用](https://www.kancloud.cn/digest/itfootballprefermanc/100911)