---
layout: post
title: chrome抓包websocket frame为空
description: chrome抓包websocket frame为空的一个弱智问题
date: 2017.12.30 20:00 +08:00
tags: 
- websocket 
---



尝试抓websocket的包，遇上一个奇葩的问题，分享下。
测试网站如下`http://websocket.org/echo.html`
chrome抓websocket的包很简单

* 打开`http://websocket.org/echo.html`
* connect ，send，可以在前端看到输出
* 打开chrome开发者工具，可以看到已经有websocket的连接

然而...啥都没有...
![啥都没有](http://upload-images.jianshu.io/upload_images/5574483-3495187529c9bac9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

很迷，看了许久，搜了很多地方，换了电脑，升级了最新版本，依旧啥都没有...
终于，在StackOverflow 上找到了靠谱的回答...

> To clarify the bottom "frame details panel" completely hides the "frames list" panel, unless you hover your mouse under the column sorting tab and drag down.

呵呵呵... 拉下来就好了...

![呵呵呵](http://upload-images.jianshu.io/upload_images/5574483-d3bd935302ce4bd6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)