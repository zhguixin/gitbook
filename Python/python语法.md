python语法

[TOC]

### python中支持while..else

```python
count = 0
while count < 5:
   print count, " is  less than 5"
   count += 1 # python不支持 count++
else:
   print count, " is not less than 5"
```

python中的`for..in`循环，支持迭代字符串！！

```python
for letter in 'Python':
   print '当前字母 :', letter
```



### 保持语句简洁的`with..as`

```python
#!/usr/bin/python
# -*- coding: UTF-8 -*-

'''
这个语法是用来代替传统的try...finally语法的。 

基本思想是with所求值的对象必须有一个__enter__()方法，一个__exit__()方法。
紧跟with后面的语句被求值后，返回对象的__enter__()方法被调用，这个方法的返回值将被赋值
给as后面的变量。当with后面的代码块全部被执行完之后，将调用前面返回对象的__exit__()方法。
'''
file = open("./for_demo1")  
try:  
    data = file.read()  
finally:  
    file.close()  

# 使用with..as..
with open("/tmp/foo.txt") as file:  
    data = file.read()

```

### 列表推导式

Python中，所有语句放到一个列表里，生成新的列表，也就是列表推导式：

```python
'''
[列表中要显示的值 循环 判断表达式]
'''
list = ["1","2","3"]

img_list = [m for m in list if m == "1"]

print img_list

### 把json文件读到列表中
items = [json.loads(line) for line in open('video.json', 'rb')]
```



### 常用函数

```python
range(10) 输出：[0,9] # 迭代输出
```

### 格式化输出字符串

Python2.6 开始，新增了一种格式化字符串的函数 `str.format()` 。

### 函数装饰器

装饰器可以动态改变函数的行为，有点切面编程的思想(Aspect-Oriented Programming)。比如一个已经定义好的函数，想要在前面打印log的输出。其他用到切面编程的场景有：性能测试、事务处理、缓存、权限校验等。

```python
import functools

def log(func):
    @functools.wraps(func) # 这句话主要是为了防止，func 函数的 __name__ 值变化。
    def wrapper(*args, **kw):
        print 'start %s():' % func.__name__
        return func(*args, **kw)
    return wrapper
```

python中的函数可以作为变量被赋值使用（符合 python语法中一切皆对象的理念），我们定义的`log` 函数接收一个函数变量。

```python
# 被装饰的函数
def query():
    print('query func')
   
# query 被重新赋值了
query = log(query)
# 再次调用的query 已经被 log 【装饰】了
query()
```

借助于**@**语法，可以变得更加简洁，这是python提供的语法糖：

```python
@log
def query():
    # do something

# 省去了 query = log(query) 这一赋值语句
query()
```

输出：

> start query():
>
> ...

log也可以接收参数，需要再嵌套一层：

```python
def log(text):
    def decorator(func):
        @functools.wraps(func) # 这句话主要是为了防止，func 函数的 __name__ 值变化。
        def wrapper(*args, **kw):
            print '%s, %s():' % (text, func.__name__)
            return func(*args, **kw)
        return wrapper
    return decorator

# 使用方式
@log('query my blog table')
def query():
    # do something
    
# 调用 query 函数
query()
```

输出如下：

> query my blog table, query():
>
> ...



### requests库的使用

通过调用requests的get方法访问网址：

```python
url = 'https://www.jianshu.com/c/V2CqjW'
num = 1
# 添加头部信息，增加代理
headers = {'user-agent': 'Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/63.0.3239.84 Safari/537.36'}
# 构建查询参数
queryParams = {'order_by':'added_at','page':num}
# 访问网址
res = requests.get(url, params=queryParams, headers=headers, timeout=200)
```

该响应对象`res` 包含服务器返回的所有信息，也包含你原来创建的 `Request` 对象。比如：

```python
# 得到访问服务器返回给我们的响应头部信息
res.headers
# 得到发送到服务器的请求的头部
r.request.headers
# 由查询参数拼接好的url
res.url
# 收到的响应的编码格式
res.encoding
# 响应状态码
if res.status_code == 200:
    print('success')
```

为了重用连接(向同一主机发送多个请求，底层的 TCP 连接将会被重用)，并且在同一个网站请求内保持cookie，我们不直接用requests的get方法进行请求，而是通过**Session**进行请求：

```python
s = requests.Session()
s.get('https://www.jianshu.com/c/V2CqjW')
```



