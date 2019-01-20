python中那些有意思的第三库

- itchat

  官网地址：[itchat](http://itchat.readthedocs.io/zh/latest/)

  热启动，`auto_login` 方法传入值为真的 `hotReload` 。

  该方法会生成一个静态文件 `itchat.pkl` ，用于存储登陆的状态。

- chatterbot

  聊天机器人，[ChatterBot](http://chatterbot.readthedocs.io/en/stable/index.html)

  ChatterBot支持接入Django，数据库配置也可以选择非关系型数据库MogonDB。

  Django使用MongoDB数据库，[官网教程](https://django-mongodb-engine.readthedocs.io/en/latest/index.html) 。

  > MongoDB 是一个基于分布式文件存储的数据库。由 C++ 语言编写。旨在为 WEB 应用提供可扩展的高性能数据存储解决方案。MongoDB没有了表的概念，MongoDB 将数据存储为一个文档，数据结构由键值(key=>value)对组成。MongoDB 文档类似于 JSON 对象

  官方提供了一个Demo：[django_app](https://github.com/gunthercox/ChatterBot/tree/master/examples/django_app)




#### 机器学习

- NLTK(Natural Language Toolkit)，自然语言处理工具库，可以用来分析实现垃圾短信智能识别。

  [官网地址](http://www.nltk.org/) 有详细教程。

