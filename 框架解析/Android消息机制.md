Android系统中，一个核心是消息的处理，他是通过Handler实现的。

#### 同步屏障机制（sync barrier）

同步屏障可以通过如下函数设置：

```java
mHandler.getLooper().getQueue().postSyncBarrier();
```

该方法发送了一个没有target的Message到MessageQueue中，在MessageQueue的next方法中，发现没有target的Message时，则跳过同步消息，优先执行异步消息。

```java
if (msg != null && msg.target == null) {
    // Stalled by a barrier.  Find the next asynchronous message in the queue.
    do {
        prevMsg = msg;
        msg = msg.next;
    } while (msg != null && !msg.isAsynchronous());
}
```

同步屏障，顾名思义但将同步消息先设置个屏障拦截住，确保了异步消息先执行，从而实现一个消息的简单优先级，把消息设置为异步的可以调用方法：

```java
Message msg = Message.obtain(mHandler, this);
msg.setAsynchronous(true);
mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
```

在ViewRootImpl.scheduleTraversals方法就使用了同步屏障，确保UI绘制优先执行。

> 值得注意的是，postSyncBarrier()要和removeSyncBarrier()一定要成对使用：
>
> **mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();**
>
> **mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);**



#### 通过Handler打印Log监控ANR

这是看到一个小哥自己仿照LeakCanary做的BlockCanary，核心点在**Looper**类中的`loop`方法：

```java
public static void loop() {
    final Looper me = myLooper();
    final MessageQueue queue = me.mQueue;
    
    for (;;) {
        Message msg = queue.next(); // might block
        if (msg == null) {
            // No message indicates that the message queue is quitting.
            return;
        }

        // This must be in a local variable, in case a UI event sets the logger
        // 传入Printer
        final Printer logging = me.mLogging;
        // 消息处理前，打印一句log
        if (logging != null) {
            logging.println(">>>>> Dispatching to " + msg.target + " " +
                            msg.callback + ": " + msg.what);
        }

        final long traceTag = me.mTraceTag;
        if (traceTag != 0) {
            Trace.traceBegin(traceTag, msg.target.getTraceName(msg));
        }
        try {
            msg.target.dispatchMessage(msg);
        } finally {
            if (traceTag != 0) {
                Trace.traceEnd(traceTag);
            }
        }
        
        // 消息处理后，再打印一句log
        if (logging != null) {
            logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
        }
        msg.recycleUnchecked();
    }
```

从上述代码中可以看出，两句log打印的时间差，就是主线程处理该消息的时长，从而确定主线程运行时长。

为了能打印上述两个log需要为Message设置**Printer**：

```java
Looper.getMainLooper().setMessageLogging(mainLooperPrinter);

// Printer是一个接口其中只有一个方法：println，覆写这个方法
class MainLooperPrinter implements Printer {
    
    @Override
    public void println(String x) {
        if (!mStartedPrinting) {
            mStartTimeMillis = System.currentTimeMillis();
            mStartThreadTimeMillis = SystemClock.currentThreadTimeMillis();
            mStartedPrinting = true;
        } else {
            final long endTime = System.currentTimeMillis();
            mStartedPrinting = false;
            if (isBlock(endTime)) {
                notifyBlockEvent(endTime);
            }
        }
    }
    
    private boolean isBlock(long endTime) {
    	return endTime - mStartTimeMillis > mBlockThresholdMillis;
    }
}

```

参考：http://blog.zhaiyifan.cn/2016/01/16/BlockCanaryTransparentPerformanceMonitor/



#### 消息队列

消息队列MessageQueue存在于Looper中，Looper通过一个`loop()`这个死的循环不断的从MessageQueue中获取信息（即调用MessageQueue的`next()`方法，next()方法也是一个死循环）。应用侧通过Handler的实例调用postMessage、sendMessage方法向MessageQueue传递消息（即调用MessageQueue的enqueueMessage()方法）。

MessageQueue的初始化操作是会调用native方法，在JNI侧初始化NativeMessageQueue。在Java侧的MessageQueue类中有个成员变量mPtr指向NativeMessageQueue。在native侧也有个Looper类，用来处理native层的消息。但是消息队列是同一个，消息处理是优先处理native层的消息。

```c++
#include <utils/Looper.h>
#include <utils/Log.h>
```

对应的实现文件在：

> system/core/libutils/Looper.cpp
> system/core/include/utils/Looper.h

**消息队列的唤醒** 

如果java层的消息队列没有消息处理时，会进入空闲状态以降低功耗。如果有消息插入消息队列（底层是按消息的触发时间维持的单链表结构），则会唤醒消息队列。具体的唤醒过程是在native层。通过IO端口的多路复用机制，采用了epoll的机制。

