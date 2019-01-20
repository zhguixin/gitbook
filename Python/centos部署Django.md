centos部署Django

#### 申请远程服务器

在搬瓦工购买服务器，购买成功后，服务器管理页面地址为：https://kiwivm.64clouds.com/main.php

#### 为服务器新建用户

CentOS，新建用户，分配超级用户权限

```bash
useradd -m -s /bin/bash zgx
usermod -a -G root zgx
passwd zgx
sudo zgx
```

让`zgx`拥有超级用户的权限，在`etc/sudoers`中添加：

```bash
## Allow root to run any commands anywhere
root ALL=(ALL) ALL
zgx ALL=(ALL) ALL # 添加这一行
```

vi命令，强制保存退出：`wq!`。

#### 安装Nginx Web服务器

安装Nginx：

```bash
# 根据系统版本的位数下载，我选的是32位的，64位的把 i386 改为 x86_64
wget http://dl.fedoraproject.org/pub/epel/6/i386/epel-release-6-8.noarch.rpmc 
# 安装这个epel库，使得yum能够找到Nginx的源
sudo rpm -ivh epel-release-6-8.noarch.rpm
sudo yum install nginx
```

> yum 卸载软件的命令是`sudo yum remove appname`

启动Nginx：

```bash
sudo service nginx start
```

在浏览器中输入服务器的IP地址，可以看到Nginx已经正常启动了。



Nginx是Web服务器，只负责处理HTTP协议，处理静态页面的内容。对于脚本语言（比如Python）产生的动态内容，它通过WSGI接口交给应用服务器来处理（Django）。WSGI是一种协议，规定了Web服务器与Web应用程序或者框架之间的标准通信接口。

> Django内部也集成了Web服务器来接受Http请求，但是为求稳定性，不用做生成环境

主流选择的WSGI容器有：Gunicorn和uWSGI。通过WSGI容器拉起Django web框架。

#### Nginx配置

部署Django时，通常都是使用一种WSGI应用服务器搭配Nginx作为反向代理。

Nginx的配置文件是以块(block)的形式组织的，每个块以一个`{}`来表示，主要有6种块。

| 块       | 含义                                                         |
| :------- | :----------------------------------------------------------- |
| main     | 全局设置，包含一些Nginx的基本控制功能，位于顶层，包含events和http |
| events   | 事件设置，控制Nginx处理连接的方式                            |
| http     | HTTP设置，它包含server和upstream两个块                       |
| server   | 主机设置                                                     |
| upstream | 负载均衡设置                                                 |
| location | URL模式设置，在server层之下，可以有多个location              |

在服务器的 `/etc/nginx/sites-available/` 目录下新建一个配置文件，假设命名为：nginx_mine.conf，输入：

```nginx
server {
    charset utf-8;
    listen 80;
    server_name site.zhguixin;

    # 请求静态资源的目录
    location /static {
        alias /home/zgx/sites/site.zhguixin/blog/static; 
    }

    location / {
        proxy_set_header Host $host;
        proxy_pass http://unix:/tmp/site.zhguixin.socket;
    }
}
```

先将默认的配置文件备份，然后替换掉默认文件：

```bash
sudo cp ~/nginx_mine.conf /etc/nginx/nginx.conf
```

检查配置文件是否有效：

```bash
sudo /etc/init.d/nginx configtest
```

重新启动Nginx服务：

```bash
sudo /etc/init.d/nginx reload
```

通过**Gunicorn**启动Django应用

```bash
gunicorn --bind unix:/tmp/site.zhguixin.socket blogproject.wsgi:application
```

可以通过正常访问网站了。