python基础笔记

### 并发操作

在Java语言中处理并发操作首先想到的是使用多线程进行处理，并且在juc包下提供了很多处理并发的类。但是在由于GIL（Global Interpreter Lock）全局解释器锁的存在，Python解释器在同一时刻只能运行与一个处理器中，也就无法充分利用多核CPU带来的好处。因此多线程并发编程在Python中并没有多大用武之地，虽然Python中也提供了`threading`线程模块。

> 关于并发操作处理的任务类型通常包含：**计算密集型**和**IO密集型**，在计算密集型的任务中，也是需要占用CPU资源的，如果你又开启多个线程，那么线程的切换同样也会消耗CPU资源，有可能表现出开启多个线程反而效率更低。
>
> IO密集型，比如网络通信等操作，可以开启多个线程来处理而不用担心抢占CPU资源，因为有可能CPU也在等待IO流可用而处于空闲状态。IO密集型的并发操作中，使用Python的多线程是可以接受的。

### 进程

由上小节可知，同一个进程开启多个线程，会出现竞争GIL的情况。因此Python中的并发操作多使用进程进行，创建进程后，每一个进程都有一个Python解释器实例，所以也就不会争夺GIL了。

为了兼容Unix和windows系统，Python中使用多进程要导入`multiprocessing`包下的`Process`。通过初始化`Process`指定target和args来产生一个新的进程：

```python
from multiprocessing import Process
import os

def run(name):
    print('name is: %s, pid is: %s' % (name, os.getpid()))

def multi_process():
    p = Process(target=run, args=('child_process',)) # args参数列表是一个元祖，不要忘了 【，】
    p.start()
    p.join() #父进程等待子进程运行结束
 
if __name__ == '__main__':
    multi_process()
```

### 进程间通信

可以通过队列`Queue` 、管道`Pipe` 来实现进程间通信，一个进程向队列中放入消息（即：`q.put()`） ；另外一个进程从队列中取出数据（即：`q.get()`） 。

通过进程间通信实现一个生产者消费者模型：

```python
from multiprocessing import Process, Queue
import os
import time

def consumer(q):
    print('pid is: %s' % (os.getpid()))
    while True:
        value = q.get(True) # get(True)，表明队列为空时阻塞等待队列，直到队列中有可用的消息
        print('get the value from queue is %s' % value)

def producer(q):
    print('pid is: %s' % (os.getpid()))
    infos = ['a','b','c','d']
    for info in infos:
        q.put(info)
        time.sleep(3) # 每隔3秒向队列中放入消息

def multi_process():
    q = Queue()
    p1 = Process(target=consumer, args=(q,))
    p2 = Process(target=producer, args=(q,))
    p1.start()
    p2.start()
    p2.join() # 等待生产者执行完毕
    p1.terminate() # 消费者退出

if __name__ == '__main__':
    multi_process()
```

