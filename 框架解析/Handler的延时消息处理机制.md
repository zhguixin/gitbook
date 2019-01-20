Handler的延时消息处理机制

Handler在Android中有着举足轻重的地位。今天选取其中一个小的知识点来看一下。

消息队列中延迟消息机制的处理。通过Handler向消息队列发送消息的方法有：

```java
post(Runnable r);
postAtTime(Runnable r, long uptimeMillis);// runnable执行的绝对时间
postDelayed(Runnable r, long delayMillis);// runnable延迟delayMillis执行
postAtFrontOfQueue(Runnable r);

sendMessage(Message msg);
sendMessageDelayed(Message msg, long delayMillis)
sendEmptyMessage(int what);
sendEmptyMessageDelayed(int what, long delayMillis);
sendEmptyMessageAtTime(int what, long uptimeMillis)
```

`postXXX`方法，也是调用到`sendMsgXXX`方法，并将`postXXX`方法的runnable赋值给了msg的callback：

```java
public final boolean post(Runnable r) {
    return  sendMessageDelayed(getPostMessage(r), 0);
}

private static Message getPostMessage(Runnable r) {
    Message m = Message.obtain();
    m.callback = r;
    return m;
}

public final boolean sendMessageDelayed(Message msg, long delayMillis) {
    if (delayMillis < 0) {
        delayMillis = 0;
    }
    // 系统启动后到现在的时间 + 指定的延迟消息时长
    return sendMessageAtTime(msg, SystemClock.uptimeMillis() + delayMillis);
}
```

Handler中提供的向**MessageQueue**提交消息的方法，最终都会调用到`sendMessageAtTime`方法：

```java
public boolean sendMessageAtTime(Message msg, long uptimeMillis) {
    MessageQueue queue = mQueue;
    if (queue == null) {
        RuntimeException e = new RuntimeException(
            this + " sendMessageAtTime() called with no mQueue");
        Log.w("Looper", e.getMessage(), e);
        return false;
    }
    return enqueueMessage(queue, msg, uptimeMillis);
}

private boolean enqueueMessage(MessageQueue queue, Message msg, long uptimeMillis) {
    msg.target = this;
    if (mAsynchronous) {
        msg.setAsynchronous(true);
    }
    return queue.enqueueMessage(msg, uptimeMillis);
}
```

在**MessageQueue**类中，`enqueueMessage()`方法将消息放到消息队列，对应的就是链表插入的过程，when为0的直接插入到了链表末端，便于消息的及时摘取，延迟执行的消息，根据when的值插入到中间某个部分：

```java
boolean enqueueMessage(Message msg, long when) {
    // ...
    
    // 对整个MessageQueue加锁
    synchronized (this) {
        // ...
        
        msg.when = when;
        Message p = mMessages;
        boolean needWake;
        // 消息队列没有消息，或者有消息时间延时已到需要根据 mBloced 决定是否唤醒消息队列
        if (p == null || when == 0 || when < p.when) {
            // New head, wake up the event queue if blocked.
            msg.next = p;
            mMessages = msg;
            needWake = mBlocked;// 消息队列阻塞的话需要唤醒
        } else {
            // Inserted within the middle of the queue.  Usually we don't have to wake
            // up the event queue unless there is a barrier at the head of the queue
            // and the message is the earliest asynchronous message in the queue.
            needWake = mBlocked && p.target == null && msg.isAsynchronous();
            Message prev;
            for (;;) {
                prev = p;
                p = p.next;
                if (p == null || when < p.when) {
                    break;
                }
                if (needWake && p.isAsynchronous()) {
                    needWake = false;
                }
            }
            msg.next = p; // invariant: p == prev.next
            prev.next = msg;
        }
        
        // We can assume mPtr != 0 because mQuitting is false.
        if (needWake) {
            // 唤醒消息队列，native方法，底层借助于epol实现的IO端口复用机制
            nativeWake(mPtr);
        }
    }
    
    return true;
}
```

在**Looper**中的`loop()`方法对消息队列进行循环遍历：

```java
        for (;;) {
            Message msg = queue.next(); // might block
            if (msg == null) {
                // No message indicates that the message queue is quitting.
                return;
            }

            // This must be in a local variable, in case a UI event sets the logger
            final Printer logging = me.mLogging;
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

            if (logging != null) {
                logging.println("<<<<< Finished to " + msg.target + " " + msg.callback);
            }
            // 省略
        }
```

调用MessageQueue的next方法不断去消息，得到Msg后调用`msg.target.dispatchMessage(msg);`处理，其中**msg.target对应的就是发送消息的Handler**。

其中MessageQueue的next方法是会发生阻塞的，源码如下：

```java
Message next() {
    // 消息循环退出，直接返回
    final long ptr = mPtr;
    if (ptr == 0) {
        return null;
    }

    int pendingIdleHandlerCount = -1; // -1 only during first iteration
    int nextPollTimeoutMillis = 0;
    for (;;) {
        if (nextPollTimeoutMillis != 0) {
            Binder.flushPendingCommands();
        }

        // 此处会阻塞，阻塞时间由nextPollTimeoutMillis确定
        // 调用了native方法，阻塞等待是由epoll这种IO端口复用机制支持实现的
        nativePollOnce(ptr, nextPollTimeoutMillis);

        synchronized (this) {
            // Try to retrieve the next message.  Return if found.
            // 获取系统启动后到现在的时间
            final long now = SystemClock.uptimeMillis();
            Message prevMsg = null;
            Message msg = mMessages;
            if (msg != null && msg.target == null) {
                // Stalled by a barrier.  Find the next asynchronous message in the queue.
                do {
                    prevMsg = msg;
                    msg = msg.next;
                } while (msg != null && !msg.isAsynchronous());
            }
            if (msg != null) {
                if (now < msg.when) {
                    // Next message is not ready.  Set a timeout to wake up when it is ready.
                    // 如果时间未到，设置下一轮需要等待的时间
                    nextPollTimeoutMillis = (int) Math.min(msg.when - now, Integer.MAX_VALUE);
                } else {
                    // Got a message.
                    mBlocked = false;
                    if (prevMsg != null) {
                        prevMsg.next = msg.next;
                    } else {
                        mMessages = msg.next;
                    }
                    msg.next = null;
                    if (DEBUG) Log.v(TAG, "Returning message: " + msg);
                    msg.markInUse();
                    return msg;
                }
            } else {
                // No more messages.
                nextPollTimeoutMillis = -1;
            }

            // Process the quit message now that all pending messages have been handled.
            if (mQuitting) {
                dispose();
                return null;
            }

            // If first time idle, then get the number of idlers to run.
            // Idle handles only run if the queue is empty or if the first message
            // in the queue (possibly a barrier) is due to be handled in the future.
            if (pendingIdleHandlerCount < 0
                && (mMessages == null || now < mMessages.when)) {
                pendingIdleHandlerCount = mIdleHandlers.size();
            }
            if (pendingIdleHandlerCount <= 0) {
                // No idle handlers to run.  Loop and wait some more.
                // 没有idle handlers处理，继续循环阻塞等待
                mBlocked = true;
                continue;
            }

            if (mPendingIdleHandlers == null) {
                mPendingIdleHandlers = new IdleHandler[Math.max(pendingIdleHandlerCount, 4)];
            }
            mPendingIdleHandlers = mIdleHandlers.toArray(mPendingIdleHandlers);
        }

        // Run the idle handlers.
        // We only ever reach this code block during the first iteration.
        for (int i = 0; i < pendingIdleHandlerCount; i++) {
            final IdleHandler idler = mPendingIdleHandlers[i];
            mPendingIdleHandlers[i] = null; // release the reference to the handler

            boolean keep = false;
            try {
                keep = idler.queueIdle();
            } catch (Throwable t) {
                Log.wtf(TAG, "IdleHandler threw exception", t);
            }

            if (!keep) {
                synchronized (this) {
                    mIdleHandlers.remove(idler);
                }
            }
        }

        // Reset the idle handler count to 0 so we do not run them again.
        pendingIdleHandlerCount = 0;

        // While calling an idle handler, a new message could have been delivered
        // so go back and look again for a pending message without waiting.
        // 处理Idle Handlers期间有可能会有新消息投递到消息队列，阻塞等待时间置为0
        nextPollTimeoutMillis = 0;
    }
}
```

`next()`方法的主要过程是：

进入一个死循环（等待延迟消息到时），为整个MessageQueue加锁，防止并发。之后有个处理过程是关于“同步分割栏”的处理（[点击了解](https://my.oschina.net/youranhongcha/blog/492591) ），之后如果消息到时直接取出返回，否则循环阻塞等待。如果Handler退出，已清空消息队列，也直接返回退出。最后是对`pendingIdleHandler`的处理。





