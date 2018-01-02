---
layout: post
title: 通过websocket获取火币网比特币实时数据
description: 通过websocket获取火币网比特币实时数据
date: 2017.12.31 20:00 +08:00
tags: 
- websocket 
---

#### websocket

我们知道，http请求只能由客户端发起，服务端应答，这种交互模式在需要实时获取数据的场景是很不合适的，传统的解决方案可能会想斗鱼弹幕一样，通过flash，使用socket进行数据交互。而websocket就是专门为这种场景而设计的，服务器可以主动向客户端推送信息，客户端也可以主动向服务器发送信息。

与http协议相似，websocket同样是基于TCP的一个协议，且默认端口也是80和443。

websocket在建立连接时，会使用HTTP协议

> GET wss://api.huobipro.com/ws HTTP/1.1
> Host: api.huobipro.com
> Connection: Upgrade
> Pragma: no-cache
> Cache-Control: no-cache
> Upgrade: websocket
> Origin: https://www.huobipro.com
> Sec-WebSocket-Version: 13
> Sec-WebSocket-Key: VC6JY+i/mBm6UJEoOzR2Kw==
> Sec-WebSocket-Extensions: permessage-deflate; client_max_window_bits

比较重要的两个参数是`Connection: Upgrade`以及`Upgrade: websocket`，用于告诉服务端需要更改为websocket协议，而后，服务器会返回101相应，告之切换完成，连接建立。而后客户端就可以往服务端发送一定格式的信息了。

##### 数据格式

websocket通过数据帧(frame)来传输数据，并且在帧中通过4bit的Opcode来确定帧的类型

* 文本
* 二进制
* 关闭
* ping
* pong
* …（剩下的暂时用不到）

#### 爬取

了解了以上基本的websocket的知识，足以应付之后的爬取工作了。使用python的一个很大的好处，就是有一堆封装的很好的模块可是直接拿来使用，接下去要用的，是一个叫websocket-client的模块，pip可以直接安装。

##### 