web前端开发学习笔记

2017/12/2

#### jade——html模板学习

######准备工作

命令行中安装：`npm install -g jade`，首先先要安装`npm`。

sublime中编写jade文件，首先安装支持jade语法高亮的插件。

定义好sublime的缩进格式，在Preference — > Setting 中配置：

```jade
"tab_size": 2, // 制表符占两位
"translate_tabs_to_spaces": true,// 制表符显示为空格
"draw_white_space": "all"// 显示空格
```

为了实时查看渲染的html文件，在命令行中：`jade -P -w index.jade`，其中`-P`表示渲染成排好版的html文件。sublime中，开启双栏显示。

###### 基本语法

```jade
doctype html
html
  head
    title jade study
  body
    // 这个注释会被渲染
    //- 这个注释不会被渲染出来
    h1#title.title zhguixin
    h2(id='title' class='title')
    a(href='www.zhguixin.site' title="this is link") link
```

首先主要一点的是要做好缩进，否则会渲染失败。

其中，渲染后的HTML文件如下：

```html
<!DOCTYPE html>
<html>
  <head>
    <title>jade study</title>
  </head>
  <body>
    <!-- 这个注释会被渲染-->
    <h1 id="title" class="title">zhguixin</h1>
    <h2 id="title" class="title"></h2><a href="www.zhguixin.site" title="this is link">link</a>
  </body>
</html>
```

通过向模板注入数据。

