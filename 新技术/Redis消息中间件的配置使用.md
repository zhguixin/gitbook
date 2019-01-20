### 开启Redis服务

安装完成后，在安装目录的`src`目录下，运行命令：

```bash
$ src/redis-server ./redis.conf
```

> 想要后台启动需要配置redis.conf中的`daemonize on` 改为`daemonize yes`
>
> 将 **bind** 一栏注释掉，默认绑定的是127.0.0.1，允许远程访问

由于Redis默认监听的端口号是**6379**，查看Linux服务器，是否已开启监听该端口号

```bash
netstat -lntp | grep 6379
```

如果端口后没有在监听状态，需要通过`iptables`命令开启端口监听

```bash
iptables -I INPUT -p tcp --dport 6379 -j ACCEPT
```



启动成功后，通过命令查看Redis服务进程

```bash
ps -ef |grep redis
```



客户端通过命令断开连接：

```bash
redis-cli -a <密码> -h 127.0.0.1 -p 6379 shutdown
```

或者没有设置密码

```bash
redis-cli  -h 127.0.0.1 -p 6379 shutdown
```



### 远程客户端连接

#### Python环境使用Celery

Celery是分布式队列的管理工具，通过将任务提交给任务队列来达到异步处理大量任务的功能。Celery提供了连接任务队列、管理任务队列等功能，任务队列可以选用：Redis、Rabbitmq、Zookeeper等。

新建一个异步任务，命名为`tasks.py`

```python
#tasks.py
from celery import Celery

backend = 'redis://localhost:6379/0'
broker='redis://localhost:6379/1'

# 配置好celery的backend和broker
app = Celery('tasks', broker=broker, backend=backend)

# 将普通函数装饰为 celery task
@app.task
def add(x, y):
    return x + y
```

> brokers 指任务队列本身，放置任务的地方
>
> backend 存储队列中的任务运行完后的结果或者状态，结果储存的地方

在终端中运行，等待提交任务

```bash
celery -A tasks worker --loglevel=info
```

> worker 指的是 Celery 中的工作者，类似与生产/消费模型中的消费者，其从队列中取出任务并执行

提交任务，新建一个任务触发的Python文件

```python
# trigger.py
from tasks import add

# 这里需要用 celery 提供的接口 delay 进行异步调用
result = add.delay(4, 4)
while not result.ready():
    time.sleep(1)

print 'task done: {0}'.format(result.get())
```

delay 返回的是一个 AsyncResult 对象，里面存的就是一个异步的结果，当任务完成时`result.ready()` 为 true，然后用 `result.get()` 取结果即可。



#### Java环境使用Jedis

项目中导入 `jedis-2.9.0.jar` 包后，就可以试着连接Redis服务器了。

```java
import redis.clients.jedis.Jedis;
public class RedisTest {
    public static void main(String[] args) {
        Jedis jedis = new Jedis("xxx.xx.xxx.xxx");
        jedis.set("foo", "bar");
        String value = jedis.get("foo");
        System.out.println(value);
    }
}
```

