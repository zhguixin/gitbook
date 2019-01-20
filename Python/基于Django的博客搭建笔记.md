基于Django的博客搭建笔记

#### 数据库表创建

在`models.py`中创建一个Blog表，默认使sqlite数据库：

```python
from django.db import models
   
class Blog(models.Model):
    # 如果没有models.AutoField，默认会创建一个id的自增列
    title = models.CharField(max_length=30)
    author = models.CharField(max_length=30)
    email = models.EmailField()
    content = models.TextField()
```

在`admin.py`中修改，控制后台展示界面：

```python

```

#### 增删改查



参考[Django数据库常用增删改查操作](https://www.cnblogs.com/yangmv/p/5327477.html)

#### 数据库操作

数据库迁移，先将旧数据库中的数据导出到一个json文件：

```bash
python manage.py dumpdata [appname] > appname_data.json
```

该json文件的格式形如：

```json
[
  {
    "model": "blog.blog",
    "pk": 1,
    "fields": {
      "title": "Android系统中的消息模型",
      "author": "zhguixin",
      ....
    }
  },
  {
    "model": "myapp.person",
    "pk": 2,
    "fields": {
      "title": "Android事件分发机制",
      "author": "zhguixin",
      ...
    }
  }]
```



导入到新数据库，不需要指定 appname：

```bash
python manage.py loaddata blog_dump.json
```

如果只是想插入批量的插入一些数据，可以在项目目录下创建一个脚本文件`load_data.py` ：

```python
#!/usr/bin/env python
#coding:utf-8
 
import os
os.environ.setdefault("DJANGO_SETTINGS_MODULE", "mysite.settings")

import django
if django.VERSION >= (1, 7):#自动判断版本
    django.setup()
 
 
def main():
    from blog.models import Blog
    f = open('oldblog.txt')
    for line in f:
        title,content = line.split(' ')
        Blog.objects.create(title=title,content=content)
        # 为避免重复可以使用这句话，相应的速度会变慢
        # Blog.objects.get_or_create(title=title,content=content)
    f.close()
 
if __name__ == "__main__":
    main()
    print('Done!')
```

该txt文件按**空格**划分各个表的字段。

#### 基于Ajax的评论提交

使用Ajax做异步请求，有两个关键点：

- 在前端，使用异步请求库(这里选择了JQuery)编写一个向服务器发送请求的函数，并用返回的结果更新页面
- 在服务端，定义一个Django视图来处理该请求，然后返回HTTP响应

参考地址[Django使用Ajax实现页面无刷新评论回复功能](http://www.cnblogs.com/mfc-itblog/p/5188900.html)

#### Django基于Ajax返回JSON格式数据

1、服务端定义个视图函数，接收请求返回JSON格式数据

```python
def api(request):
    response = HttpResponse()
    response['Content-Type'] = "text/javascript; charset=utf-8"
    response.write(serializers.serialize("json", Blog.objects.all(), ensure_ascii=False))
    return response
```

> 注意防止乱码，因为使用的是UTF-8编码的中文

2、浏览器通过jQuery异步请求(演示)

```html
<script>
function update() {
    $.getJSON("/api/", function(data) {
        var html = "<ul>";
        jQuery.each(data, function(){
            html += "<li>" + this.fields.title + "</li>";
            // console.log(this.fields.title);
        });
        html += "</ul>";
        $("#display").html(html);
    });
}

// 每隔10秒请求一次
$(document).ready(function() {
    setInterval("update()", 10000);
})
</script>
```

JSON格式的数据更适合于Android、IOS或者小程序使用，我们可以设计成RestFul API的风格，并且Django有个开源的框架就DjangoRestFramework（DRF），通过DRF这个框架通过简单的配置编写可以写出良好风格的RestFul API。

#### 网站性能测试

在这里使用Apache工具包下的`ab`命令，进行测试。输入命令：

```bash
ab -n 1000 -c 10 http://127.0.0.1:8000/
```

`-n`表示请求总数目；`-c`表示并发请求量，即同一时间请求数

运行后的结果生成运行报告，关注

- Requests per second    每秒请求数

- Time per request    每一个请求平均花费时间

  > 有两个，其中第一个可以理解为用户平均请求等待时间，第二可以理解为服务器平均请求等待时间

#### 加入缓存提高性能

Django中的缓存框架提供了四种数据缓存存储的机制。博客中配置了最简单的一种——全站式缓存。

首先在项目的`settings.py`文件的**MIDDLEWARE**，加入：

```python
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
    'django.middleware.cache.CacheMiddleware', # 加入这一句
]
```

并在该文件中配置缓存位置：

```python
CACHE_BACKEND = "locmem://"
```

> 这是Django缓存框架提供的一种伪URL的设置风格，通过这种URL格式来封装配置参数

更精细的内存缓存策略，比如控制某个页面的缓存，以及它的缓存时间、存储位置等，可以通过Django缓存框架提供的一些装饰器函数(`cache_page`等)来为对应视图函数设置，具体可以参考Django官方文档。

