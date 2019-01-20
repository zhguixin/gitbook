---
title: Hexo博客插入图片
date: 2016-7-22
tags: 技巧
categories: 
grammar_cjkRuby: true
---
早期大部分的方案是把图片放在 source/img 下，然后在 markdown 里写 ![img](/source/img/img.png) 。显然这样在本地的编辑器里完全不能正确识别图片的位置。

1.2 asset-image

在 hexo 2.x 时出现的插件，后来被吸纳进 hexo 3 core ，用法的介绍见 资源文件夹 | Hexo 。比较尴尬的是，这种方法直接放弃了 markdown 原来的语法，使用类似 的语法，。markdown 本来有插入图片的语法不好好支持，专门用一个新的语法来插入本地图片，让我这种强迫症不太能接受。

2 解决方案

CodeFalling/hexo-asset-image

2.1 使用

首先确认 _config.yml 中有 post_asset_folder:true 。

在 hexo 目录，执行

npm install https://github.com/CodeFalling/hexo-asset-image --save
假设在

MacGesture2-Publish
├── apppicker.jpg
├── logo.jpg
└── rules.jpg
MacGesture2-Publish.md
这样的目录结构（目录名和文章名一致），只要使用 ![logo](MacGesture2-Publish/logo.jpg) 就可以插入图片。

生成的结构为

public/2015/10/18/MacGesture2-Publish
├── apppicker.jpg
├── index.html
├── logo.jpg
└── rules.jpg
同时，生成的 html 是

<img src="/2015/10/18/MacGesture2-Publish/logo.jpg" alt="logo">
而不是愚蠢的

<img src="MacGesture2-Publish/logo.jpg" alt="logo">
值得一提的是，这个插件对于 CodeFalling/hexo-renderer-org 同样有效。

#### 使用图床
可以使用图床的形式在博客中插入图片，比如：![hello](http://i2.piimg.com/567571/5bdd045858615fcf.jpg)
可以直接搜索图床，常用的有：七牛图床、外链吧等
#### Hexo + Next插入背景图片
博客添加一张背景图片：
- 首先找到一个背景图片放到**hexo（hexo工程文件）-> themes -> next -> source -> images** 的路径下；

- 然后找到**hexo（hexo工程文件）-> themes -> next -> source -> css -> _schemes ->Pisces**路径下的index.styl文件，在文件的最上方加上一代码：
```javascript
body { background:url(/images/backGround.jpg（这是你之前加的背景图片的名字）);} 
```


![this is example][1]


  [1]: http://i2.piimg.com/567571/5bdd045858615fcf.jpg