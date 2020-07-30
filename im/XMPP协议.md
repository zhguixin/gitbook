XMPP协议：

XMPP在Client-to-Server通信，和Server-to-Server通信中都使用TLS (Transport Layer Security)协议作为通信通道的加密方法，保证通信的安全。不论什么XMPPserver能够独立于公众XMPP网络（比如在企业内部网络中）。

也就是说在大多数情况下，当两个client进行通讯时，他们的消息都是通过server传递的；采用这种架构可以简化client端，将大多数工作放在server端。

**XMPP中定义了三个角色，XMPPclient，XMPPserver、网关**

主要的网络形式是单client通过TCP／IP连接到单server，然后在之上传输XML，工作原理是：

(1)节点连接到server；(2)server利用本地文件夹系统中的证书对其认证；(3)节点指定目标地址，让server告知目标状态；(4)server查找、连接并进行相互认证；(5)节点之间进行交互

- XMPPClient，XMPP 系统的一个设计标准是必须支持简单的client。其实，XMPP 系统架构对client仅仅有非常少的几个限制。一个XMPP client必须支持的功能有：

  1. 通过 TCP 套接字与XMPP server进行通信；

  2. 解析组织好的 XML 信息包；

  3. 理解消息数据类型。

- XMPPServer，遵循两个主要法则：

  1. 监听client连接，并直接与client应用程序通信；

  2. 与其它 XMPP server通信

  XMPP开源server一般被设计成模块化，由各个不同的代码包构成，这些代码包分别处理Session管理、用户和server之间的通信、server之间的通信、DNS（Domain Name System）转换、存储用户的个人信息和朋友名单、保留用户在下线时收到的信息、用户注冊、用户的身份和权限认证、依据用户的要求过滤信息和系统记录等。

- XMPP网关，MPP 突出的特点是能够和其它即时通信系统交换信息和用户在线状况。因为协议不同，XMPP 和其它系统交换信息必须通过协议的转换来实现。实现这个特殊功能的服务端在XMPP 架构里叫做网关(gateway)。因为网关的存在，XMPP 架构其实兼容全部其它即时通信网络，这无疑大大提高了XMPP 的灵活性和可扩展性



 XMPP 协议具有良好的扩展性。在XMPP 中，即时消息和到场信息都是基于XML 的结构化信息，这些信息以XML 节(XML Stanza)的形式在通信实体间交换。

用户成功登陆到server之后，公布更新自己的在线好友管理、发送即时聊天消息等业务。全部的这些业务都是通过三种主要的XML 节来完毕的：IQ Stanza（IQ 节）, Presence Stanza（Presence 节）, Message Stanza（Message 节）。



#### XMPP地址格式

一个实体在XMPP网络结构中被称为一个接点，它有唯一的标示符jabber identifier(JID)，即实体地址，用来表示一个Jabber用户。

格式是：`node@domain/resource`。

XMPP中定义了  3个顶层XML元素: Message、Presence、IQ。

- Message，用于在两个jabber用户之间发送信息
- Presence，用来表明用户的状态，如：online、away、dnd(请勿打搅)等
- IQ，一种请求／响应机制，从一个实体从发送请求，另外一个实体接受请求，并进行响应

参考文章：[XMPP协议的原理介绍](https://www.cnblogs.com/mfrbuaa/p/3763560.html)



