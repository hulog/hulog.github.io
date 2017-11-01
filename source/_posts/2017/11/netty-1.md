---
title: Nette入门第一章——IO演进
date: 2017-11-01 20:01:46
description:
categories: netty
tags: netty
---

# 1.  IO 基础

## 1.1. linux网络IO模型

1. 阻塞IO模型
2. 非阻塞IO模型
3. IO多路复用模型（NIO）
4. 信号驱动IO模型
5. 异步IO模型
<!--more-->
## 1.2. IO多路复用模型

目前支持IO多路复用的系统调用有`select`,`pselect`,`poll`,`epoll`。`epoll`相对于`select`特点如下：

1. 支持一个进程打开的`socket`描述符(FD)不受限制(仅受限与操作系统的最大文件句柄数)。**1**GB内存大约是**10W**个句柄，通过`cat /proc/sys/fs/file-max` 查看。
2. IO效率不会随着FD数目的增加而线性下降。`select/poll`每次是线性扫描全部集合，因此会线性下降。`epoll`只处理活跃的socket，但如果所有的socket都活跃， `epoll`并不比`select/poll`高很多。
3. 使用mmap加速内核与用户空间的消息传递。epoll是通过和用户空间mmap同一块内存实现。
4. `epoll`的API更加简单。

# 2. JAVA IO演进之路

JDK1.4之前--BIO
JDK1.4--NIO以JSR-51的身份出现。但是所有文件操作都是同步阻塞调用，不支持文件异步读写操作。
JDK1.7--NIO2.0由JSR-203演进而来。具体改进地方如下:
1. 提供批量获取文件属性的API。
2. 提供AIO功能，支持基于文件的异步IO和套接字的异步操作。
