web开发总结

2017/12/14

#### Nodejs简介

Node.js是一个JavaScript的解析引擎，可以运行js代码，又是一个命令行工具，并且提供了访问网络以及访问文件系统API的模块。可以用做后台服务开发。这样一来，省去了Apache或者Nginx的Http服务器。

我们使用Node.js时，不仅仅在实现一个应用，同时还实现了整个HTTP服务器。



#### 使用Node.js开启一个服务器

主要包括几个部分：

* 引入required模块，通过`require` 指令来载入Node.js的模块，类似于python中的`import` 
* 创建服务器，该服务器用来监听客户端发来的请求；
* 接收请求与响应请求，接收请求后，做出相应处理并返回

```javascript
// 使用 require 请求载入Node.js模块
var http = require('http')

// 创建服务器，通过回调函数的 request, response 参数来接收和响应数据
http.createServer(function (request, response) {
	response.writeHead(200,{'Content-Type':'text/plain'})
	response.end('hello world\n')
}).listen(8888);// 监听8888端口

// 终端数据打印
console.log('server running at http://127.0.0.1:8888/')
```

更进一步，我们构建一个服务端程序，在模拟APP访问网络返回json串：

```javascript
var http = require('http');
var url = require('url');
var fs = require('fs');

http.createServer(function (request, response) {
	var pathName = url.parse(request.url).pathname;
	console.log('request for ' + pathName);

	fs.readFile(pathName.substr(1), function(err,data) {
		if (err) {
			console.log(err);
			response.writeHead(404,{'Content-Type':'text/plain'});
		} else {
			response.writeHead(200,{'Content-Type':'text/plain'});
			response.write(data.toString()); 
		}
		response.end();
	});
	// response.writeHead(200,{'Content-Type':'text/plain'})
	// response.write('hello world\n');
	// response.end();
}).listen(8888);

console.log('server running at http://127.0.0.1:8888/')
```
保存为`server.js`，在该目录下存放movies.json文件，命令行开启服务：

```bash
$ node server.js
```
手机访问的网络：`PC的IP地址/movies.json`， 通过OkHttp进行网络加载。

> 通过node 运行js文件，如果只在终端中键入node，进入Node.js提供的交互式解释器(Read Eval Print Loop)界面，类似于在终端中键入 python命令，可以在这个交互式解释器里键入代码，直接查看结果.



#### 使用Node创建web客户端

Node.js除了创建web服务器应用，也可以创建一个web客户端应用，比如通过Node.js实现一个网络爬虫应用。

这里只举一个简单的小例子：

```javascript
var http = require('http');
 
// 用于请求的选项
var options = {
   host: 'localhost',
   port: '8080',
   path: '/index.html'  
};
 
// 处理响应的回调函数
var callback = function(response){
   // 不断更新数据
   var body = '';
   response.on('data', function(data) {
      body += data;
   });
   
   response.on('end', function() {
      // 数据接收完成
      console.log(body);
   });
}
// 向服务端发送请求
var req = http.request(options, callback);
req.end();
```



#### NPM

npm是在安装Node.js时一同安装的包管理工具，通过在终端键入npm命令，可以下载安装第三方包到本地使用或者上传自己的包（或模块）到NPM服务器供别人使用。

NPM，常用命令：

```bash
$ npm install express    # 本地安装 express 模块，只需要通过 require('express')即可使用
$ npm install express -g # 全局安装 express 模块

$ npm list -g	# 查看所有全局安装的模块
$ npm uninstall express # 卸载某个 Node.js 模块
$ npm update express # 更新模块
```

#### 回调函数和事件

Node.js 是单进程单线程应用程序，但是通过事件和回调支持并发，所以性能非常高。

![](F:\gitbook\web开发\Event_Loop.png)

具体来说，就是将各个回调任务，放到任务队列中去（称之为**事件队列**），主线程（单一线程）中运行一个循环（称之为**事件循环**）不断的从任务队列中去取任务。

```javascript
console.log('-----start-----');

setTimeout(function() {console.log('hello');}, 200);
setTimeout(function() {console.log('world');}, 100);

console.log('-----end-------');
```

输出结果为：

> -----start-----
>
> -----end-------
>
> world
>
> Hello

注意并不是setTimeout加入了事件队列，而是setTimeout里的回调函数加入了事件队列，`setTimeout` 抛给**Timer**模块，JS引擎（即主线程）继续运行，在**Timer**监测到定时时间到时，将`callback`加入到任务队列。

考虑如下代码的输出：

```javascript
console.log(1);

setTimeout(function() {console.log(2);}, 400);

setTimeout(function() {console.log(3);}, 300);

// 需要 3000ms
for (var i = 0; i < 10000; i++) {
    console.log(4)
}

setTimeout(function() {console.log(5);}, 100);
console.log('--------end---------');
```

输出结果为：

> 1
>
> 4（10000）
>
> 3
>
> 2
>
> 5

可以看到先输出了循环1万次后的打印4，即使任务已经在任务队列里了（因为定时已到，Timer模块将`callback`放到了任务队列中）因为**主线程阻塞住了，无法去任务队列里面取任务**，后面的语句

```javascript
setTimeout(function() {console.log(5);}, 100);
```

连交给Timer模块都没有得以执行。（在上图中，即压入栈的语句是顺序执行的）

参考文章：

[JavaScript的执行原理](https://blog.csdn.net/gy_u_yg/article/details/72869315)

[Nodejs探秘：深入理解单线程实现高并发原理](https://cloud.tencent.com/developer/article/1169075)