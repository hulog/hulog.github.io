---
title: Netty入门第三章——粘包和拆包
date: 2017-11-01 20:13:23
description:
categories: netty
tags: netty
---

# TCP粘包产生原因
1. 应用程序write写入的字节数大小大于套接字发送缓冲区的大小。
2. 进行MSS大小的TCP分段。
3. 以太网帧的payload大于MTU进行IP分片。

<!--more-->
# 用于解决TCP粘包问题的编码器

  |序号|名字|作用|
  |:--|:--|:--|
  |1|LineBasedFrameDecoder|基于** *行* **的解码器|
  |2|StringDecoder|基于** *字符串* **的解码器|
  |3|DelimiterBasedFrameDecoder|基于** *分隔符* **作为码流结束标示的消息解码器|
  |4|FixedLengthFrameDecoder|基于** *固定长度* **解码器|
1. **LineBasedFrameDecoder** -- 基于** *行* **的解码器。遍历ByteBuf中的可读字节，判断是否有`"\n"`或`"\r\n"`，如有则以此结束位置。可配置单行最大长度，如达到最大长度，仍没有发现换行符，则抛出异常。(见代码片段1)
```java
// 代码片段1
public class TimeServer {
    private class ChildChanneHandler extends ChannelInitiallizer<SocketChannel> {
        @override
        protected void initChannel(SocketChannel arg0) throws Execption {
            arg0.pipeline().addLast ( new LineBasedFrameDecoder(1024));
            arg0.pipeline().addLast ( new StringDecoder());
            arg0.pipeline().addLast ( new TimeClientHandler());
        }
    }
}
```
2. **StringDecoder** -- 基于** *字符串* **的解码器。将接收到的byte[]转换成字符串。LineBasedFrameDecoder+StringDecoder的组合即为按行切换的文本解码器。
3. **DelimiterBasedFrameDecoder** -- 基于** *分隔符* **作为码流结束标示的消息解码器。(见代码片段2)
```java
// 代码片段2
public class EchoServer {
    private class ChildChanneHandler extends ChannelInitiallizer<SocketChannel> {
        @override
        protected void initChannel(SocketChannel arg0) throws Execption {
            ByteBuf delimiter = Unpooled.copiedBuffer("$_".getBytes()))
            arg0.pipeline().addLast ( new DelimiterBasedFrameDecoder(1024,delimiter));
            // arg0.pipeline().addLast ( new LineBasedFrameDecoder(1024));
            arg0.pipeline().addLast ( new StringDecoder());
            arg0.pipeline().addLast ( new TimeClientHandler());
        }
    }
}
```
4. **FixedLengthFrameDecoder** -- ** *固定长度* **解码器.(见代码片段3)
 ```java
// 代码片段3
public class EchoServer {
    private class ChildChanneHandler extends ChannelInitiallizer<SocketChannel> {
        @override
        protected void initChannel(SocketChannel arg0) throws Execption {
            ByteBuf delimiter = Unpooled.copiedBuffer("$_".getBytes()))
            // arg0.pipeline().addLast ( new DelimiterBasedFrameDecoder(1024,delimiter));
            // arg0.pipeline().addLast ( new LineBasedFrameDecoder(1024));
            arg0.pipeline().addLast ( new FixedLengthFrameDecoder(20));
            arg0.pipeline().addLast ( new StringDecoder());
            arg0.pipeline().addLast ( new TimeClientHandler());
        }
    }
}
```
