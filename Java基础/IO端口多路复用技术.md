IO端口的操作，借助于read和write。对于单线程来说，没有数据可读时，会造成阻塞，Linux系统在内核空间返回给用户空间的是文件描述符fd。为了能够同时监听多个文件描述符，就可以借助于：select或者poll。

#### 传统的IO端口复用机制

select使用数组保存文件描述符，一个进程可以最大保存1024个文件描述符；poll使用链表保存文件描述符，没有了这个限制。

但是由于，select或者poll是轮询每个文件描述符来告知用户空间数据就绪，随着描述符数量的增加，同时伴随着句柄数据在内核空间与用户空间的拷贝，效率会变得比较低下。

#### epoll机制

为了解决这个问题，引入了epoll这个IO端口多路复用机制。

epoll的操作分为三大部分：

* epollFd = epoll_create()，创建一个epoll对象
* epoll_ctrl(epollFd, EPOLL_CTL_ADD, socket, EPOLLIN)，将要监听的事件(socket)加入epoll对列
* epoll_wait(epollFd, result, result.length, tiemout)，监听事件，等待事件响应

epoll内部用户保存事件的数据结构使用的是红黑树，查找、删除的速度很快。

#### 后记

在JDK1.4之后，Java引入了一个新的包——NIO包，这个包提高了IO端口的操作效率。NIO三个主要的概念就是：Buffer、Channel、Selector。

Selector监听的Channel有数据准备好后，调用select()方法。在select()方法中会根据操作系统，在底层选择poll机制或者是epoll机制。



更新：ioctl函数

ioctl是设备驱动程序对设备IO通道进行管理的函数。

```cpp
// fd是使用open打开的设备描述符
// cmd表示用户对设备的控制命令
// 省略的参数作为补充参数
int ioctl(int fd, int cmd, ...);
```

用户通过命令码来告知驱动程序该干什么，对命令码的解析和对应命令码的操作都需要驱动程序来实现。

比如Android的Binder驱动中命令码的种类：

```cpp
int ioctl(int fd, ioctl命令, 数据类型);
```

| ioctl命令              | 对应的Binder驱动的额操作           | 数据类型                 |
| ---------------------- | ---------------------------------- | ------------------------ |
| BINDER_WRITE_READ      | 在进程间接收发送IPC数据            | struct bidner_write_read |
| BINDER_SET_MAX_THREADS | 设定注册到Binder驱动中线程最大数目 | size_t                   |
| ...                    | ...                                | ...                      |

