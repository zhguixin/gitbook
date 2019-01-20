Python web框架基本原理理解



流行web框架：

遵循WSGI接口协议，该协议由两部分组成：`server`和 `application`。

server负责解析http响应，并传入两个参数：**environ**和**start_response**到application中去。

- environ：一个包含所有HTTP请求信息的`dict`对象；
- start_response：一个发送HTTP响应 Header的函数。

application根据传入的两个参数生成响应头(header)和响应体(body)。最后由`server`，返回此次Http响应的处理结果到客户端。

Django作为一个web框架扮演的角色就是`application`。

一个简单的application实现如下：

```python
def application(environ, start_response):
    # http响应头
    start_response('200 OK', [('Content-Type', 'text/html')])
    # 返回http body部分，即html文档，方便浏览器解析
    return [b'<h1>Hello, web!</h1>']
```

由此可见，server的职责除了解析Http请求还包括生成`environ`字典，以及`start_response`函数的实现。在python中，已经实现了简单的遵循WSGI协议的`server`，结合刚才那个简单的`application`，可以搭建一个最简单的web程序：

```python
# 从wsgiref模块导入:
from wsgiref.simple_server import make_server
# 导入我们自己编写的application函数:
from hello import application

# 创建一个服务器，IP地址为空，端口是8000，处理函数是application:
httpd = make_server('', 8000, application)
print('Serving HTTP on port 8000...')
# 开始监听HTTP请求:
httpd.serve_forever()
```



#### 通过Nginx部署Django项目

在已开发完成的Django目录下，运行命令：

```bash
pip freeze > requirements.txt # 如果没有创建环境，会输出环境变量下的python所有依赖库
```

将依赖的一些第三库以及对应的版本号输入到`requirements.txt`文件中去。

在服务器上，创建虚拟环境，得到一份纯净的python环境（不包含任何第三库）：

```bash
zhguixin@localhost:~/sites/zhguixin$ virtualenv  --no-site-packages --python=python3 env
```

> 提前安装 pip 和 virtualenv， sudo pip install virtualenv

参数`--no-site-packages`，指定已经安装到系统Python环境中的所有第三方包都不会复制到 env 目录下。

使用 --python=python3 来指定克隆 Python3 的环境。如果不指定，会克隆系统默认的环境。比如： ubuntu 系统默认安装了 Python2，如果不特别指定的话 Virtualenv 默认克隆的是 Python2 的环境。

虚拟环境创建成功后，会在当前目录下生成一个**env**的文件夹。

激活虚拟环境：

```bash
source env/bin/activate
```

现在使用的就是虚拟环境下的python了。

> 退出当前的`env`环境，直接在命令行输入`deactivate`命令

详细部署过程：[链接](https://www.zmrenwu.com/post/20/)