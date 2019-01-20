---

title: 编写python爬虫拿到电影天堂最新电影
date: 2017-10-24
tags: Python 爬虫
categories: 

---

Python这种短小精悍的语言，拿来编写爬虫真是再容易不过了。今天尝试爬取[电影天堂](http://www.dytt8.net/html/gndy/dyzz/index.html)上的最新电影。抛砖引玉。。

#### 页面分析

电影天堂上的页面没有过多的JavaScript等动态加载的页面，几乎是纯纯的html页面，分析起来就很方便了。

我们看到在最新电影列表中每一部电影的呈现方式如下：

![](./images/dytt.png)

分析一下他的页面布局（格式处理了，方便分析）：

```html
<table width="100%" border="0" cellspacing="0" cellpadding="0" class="tbspan" style="margin-top:6px">
<tr> 
	<td height="1" colspan="2" background="/templets/img/dot_hor.gif"></td>
</tr>
<tr> 
	<td width="5%" height="26" align="center">
		<img src="/templets/img/item.gif" width="18" height="17">
	</td>
	<td height="26">
		<b>
			<a href="/html/gndy/dyzz/20171021/55350.html" class="ulink">2017年刘德华舒淇动作《侠盗联盟》BD国粤双语中字</a>
		</b>
	</td>
</tr>
<tr> 
	<td height="20" style="padding-left:3px">&nbsp;</td>
	<td style="padding-left:3px">
		<font color="#8F8C89">日期：2017-10-21 23:26:13 点击：0 </font>
	</td>
</tr>
<tr> 
	<td colspan="2" style="padding-left:3px">侠盗联盟][BD-mkv.720p.国粤双语中字][2017年刘德华舒淇动作] ◎译 名 侠盗联盟 ◎片 名 The Adventurers ◎年 代 2017 ◎产 地 中国/中国香港/捷克 ◎类 别 动作/冒险 ◎语 言 国语/粤语 ◎字 幕 中文 ◎IMDb评分 6.9/10 from 164 users ◎豆瓣评分 5.6/10 from 19,372</td>
</tr>
</table>
```

从html中可以看到，每一部电影及详细信息都在`table`标签中，包括电影上面那条虚线（/templets/img/dot_hor.gif），总共分成了四行。其中`<tr>标签标示行、`<td>标签标示列。

知道了整个结构，用python爬取结果就容易很多了。

#### 编写python爬虫

python爬虫框架有很多，这里采用`request` + `BeautifulSoup`的方式，直接上代码：

```python

```

#### 编写java爬虫

python下有`BeautifulSoup`，java中有一个`jsoup`，详细描述见：[jsoup: Java HTML Parser](https://jsoup.org/)。

