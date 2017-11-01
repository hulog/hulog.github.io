---
title: Netty入门第五章——协议栈开发纪要
date: 2017-11-01 20:25:40
description:
categories: netty
tags: netty
---

Netty的HTTP协议栈开发的客户端和服务端具有Netty的天然优势——异步事件驱动。所以以此开发的HTTP协议栈程序也是异步非阻塞的。本章节介绍如何利用Netty提供的基础完成HTTP协议栈的开发。
## 1. Netty+XML 协议栈开发

> HTTP仅仅是承载数据交换的通道，是载体而不是Web容器，没有必要上Tomcat等重量型容器。

## 2. WebSocket开发

>对于HTTP协议，开销较大，服务器只有收到请求才会应答，不适合做低延迟应用。Websocket将网络套接字引入客户端和服务端。浏览器和服务器之间可以通过套接字建立持久的连接。双方都可以互发数据给对方。

<!--more-->
### 2.1. HTTP协议的弊端：
   1. HTTP半双工，两端不能同事传输数据
   2. HTTP消息冗长繁琐，包括消息头、消息体、换行符等，通常，基于文本传输方式，要比其他二进制通信   协议繁琐和冗长。
   3. 针对服务器推送的黑客攻击，如长轮询。

目前，很多网站为了实现**消息推送**，基本上都是采用长轮询(例如每秒1次HTTP Request)，header冗长，占用带宽和服务器的资源。因此HTML5定义了WebSocket协议，更好的节省了服务器资源和带宽实现实时通信。
新的长轮询技术是Comet，使用AJAX，可达到双向通信，但依然需要发送请求，切普遍采用了长连接，也会消耗带宽和资源。

### 2.2. WebSocket特点：
   1. 浏览器与服务器只需做**一个握手操作**(与TCP握手不在同一层次，TCP握手后连接建立，WebSocket握手是指在TCP建立后告诉服务器这个一个WebSocket握手信息)，然后形成快速通道，WebSocket基于TCP的全双工通讯，相比HTTP的半双工，性能得到较大的提升。
   2. 对代理和防火墙透明
   3. 无Header、Cookie和身份认证
   4. 无安全开销
   5. 通过"ping/pong"帧保持链路激活
   6. 服务器主动推送到客户端，无需客户端的长轮询

### 2.3. WebSocket连接的建立和关闭
建立：需要客户端或者浏览器发出握手请求，请求消息为HTTP请求，其中包含了附加的头信息，如`Upgrade: websocket`和`Connection: Upgrade`等。
```
GET ws://echo.websocket.org/?encoding=text HTTP/1.1
Host: echo.websocket.org
Connection: Upgrade
Pragma: no-cache
Cache-Control: no-cache
Upgrade: websocket
Origin: http://websocket.org
Sec-WebSocket-Version: 13
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/54.0.2840.90 Safari/537.36
Accept-Encoding: gzip, deflate, sdch
Accept-Language: zh-CN,zh;q=0.8,de;q=0.6
Cookie: _ga=GA1.2.2145434661.1502418351; _gid=GA1.2.1865172317.1502418351; _gat=1
Sec-WebSocket-Key: VDY/9d/0RyXxpbW6YRd++Q==
Sec-WebSocket-Extensions: permessage-deflate; client_max_window_bits
```
服务器解析头信息并做出应答，包含附加字段如：`Connection: Upgrade`和`Upgrade: websocket`等
```
HTTP/1.1 101 Web Socket Protocol Handshake
Access-Control-Allow-Credentials: true
Access-Control-Allow-Headers: content-type
Access-Control-Allow-Headers: authorization
Access-Control-Allow-Headers: x-websocket-extensions
Access-Control-Allow-Headers: x-websocket-version
Access-Control-Allow-Headers: x-websocket-protocol
Access-Control-Allow-Origin: http://websocket.org
Connection: Upgrade
Date: Fri, 11 Aug 2017 02:41:01 GMT
Sec-WebSocket-Accept: 0xphShwDb6MpA6oZYZsYpiuJNhk=
Server: Kaazing Gateway
Upgrade: websocket
```
连接建立成功。

关闭：
上述连接持续到某一端主动关闭连接。
底层的TCP连接应该首先由服务器关闭，异常情况下，可以由客户端发起TCP Close。

### 3. 私有协议栈开发
> 广义上，通信协议可分为公有和私有协议。私有协议具有更好的灵活性，往往会在公司和组织内部使用，按需定制。绝大多数协议都是基于TCP/IP，所以利用Netty的NIO TCP协议栈可以非常方便进行定制开发。

大型系统往往会被拆分成多个模块，各个模块可能需要实现跨节点通信，在传统的JAVA应用中，通常使用一下4种方式进行跨节点通信：
1. RMI远程服务调用
2. Java的Socket+Java序列化方式
3. 利用开源RPC框架，如Facebook的Thrift、Apache的Avro
4. 利用标准的公有协议，如HTTP+XML、RESTful+JSON、Webservice
