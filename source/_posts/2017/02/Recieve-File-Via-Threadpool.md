---
title: 服务器基于ThreadPool接收文件
date: 2017-02-09 23:44:40
categories: [java,ThreadPool]
tags: [Java, 线程池, IO, Socket, ThreadPool]
---

## 背景

上篇文章([一文带你进入Java之ThreadPool](http://hulong.me/2017/01/19/Java-ThreadPool.html))基本上介绍了Java中的线程池的类型，以及如何按照业务不同自定义线程池。那么问题来了，池建好了，如何让它运行起来呢？本文主要围绕这一主题——让线程池跑起来，进行测试！ 
<!--more-->
## 测试内容

**思路**：模拟多个客户端向服务器发送文件，服务器端一旦监听到请求，立即抛给线程池进行处理，工作线程接受客户端数据，将文件落地。

*tips:* 只是一个简单的c/s模式，主要测试线程池性能。

### 步骤

1. 开启服务器进行监听
2. 用多个线程模拟多个客户端，向服务器发送消息
3. 服务器监听到请求，立即交给线程池处理
4. 线程池分配线程接受任务

### 代码实现

先说明各个文件作用

1. `TestServerMain.java` 用于开启服务器，并开始监听请求
2. `TestClientMain.java` 用于开启客户端，并开始发送文件
3. `Client.java` 发送文件的实现类
4. `Server.java` 服务器初始化和监听实现类
5. `ServerFileHandler.java` 服务器端存储文件实现类

#### `TestServerMain.java`

开启服务器

```java
package com.norman.threadpool;

import java.io.IOException;

/**
 * 测试方法
 *
 * @author norman
 * @version 1.0
 */
public class TestServerMain {

    /**
     * 测试主方法.
     */
    public static void main(String[] args) throws InterruptedException {
        // 运行服务器
        new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    Server.start();
                } catch (IOException ex) {
                    ex.printStackTrace();
                }
            }
        }).start();
    }
}
```

#### `TestClientMain.java`

开启客户端

```java
package com.norman.threadpool;

import java.io.File;

/**
 * 开启客户端
 *
 * @author norman
 * @version 1.0
 */
public final class TestClientMain {

    /**
     * 模拟客户端数量.
     */
    private static final int N_THREADS = 10;

    /**
     * 开启N_THREADS个客户端.
     *
     * @throws InterruptedException 中断异常.
     */
    public static void main(String[] args) throws InterruptedException {

        //文件路径
        final String pathToFile = "/home/norman/wps-office_8.1.0.3724-b1p2_i386.deb";

        new Thread(new Runnable() {
            @Override
            public void run() {
                for (int i = 0; i < N_THREADS; i++) {
                    new Thread(new Runnable() {
                        @Override
                        public void run() {
                            Client.send(new File(pathToFile));
                        }
                    }).start();
                }
            }
        }).start();
    }
}
```

#### `Client.java`

客户端发送文件

```java
package com.norman.threadpool;

import java.io.BufferedReader;
import java.io.File;
import java.io.FileInputStream;
import java.io.IOException;
import java.io.InputStreamReader;
import java.io.OutputStream;
import java.net.Socket;

/**
 * 客户端实体类.
 *
 * @author norman
 * @version 1.0
 */
public class Client {
    /**
     * 发送文件.
     *
     * @param file 要进行传输的文件
     */
    public static void send(File file) {
        String  defaultServerIp = "192.168.199.101";
        int  defaultServerPort = 12345;
        send(defaultServerIp, defaultServerPort, file);
        return;
    }

    /**
     * 开始发送文件.
     *
     * @param ip   ip address
     * @param port port
     * @param file 文件对象
     */
    public static void send(String ip, int port, File file) {
        if (!file.exists()) {
            System.out.println("文件不存在");
            return;
        }

        System.out.println("文件>>>>"
                + file.getName()
                + ",正在上传......");
        Socket socket = null;
        BufferedReader in = null;
        OutputStream out = null;
        try {
            socket = new Socket(ip, port);
            in = new BufferedReader(new InputStreamReader(socket.getInputStream()));
            out = socket.getOutputStream();
            FileInputStream fis = new FileInputStream(file);

            byte[] buf = new byte[1024];
            int len;
            while ((len = fis.read(buf)) != -1) {
                out.write(buf, 0, len);
                out.flush();
            }
            fis.close();
            socket.shutdownOutput();

            //接受服务器传回的消息
            String temp;
            long reply = 0;
            while ((temp = in.readLine()) != null) {
                try {
                    reply = Long.parseLong(temp);
                } catch (Exception ex) {
                    System.out.println("数字解析出错");
                }
                System.out.println("客户端收到服务器响应: "
                            + temp
                            + " ,文件上传成功率: "
                            + reply / file.length() / 1.0 * 100
                            + " % ");
            }
        } catch (IOException ex) {
            ex.printStackTrace();
        }
    }
}
```

#### `Server.java`

服务器接受请求，具体处理交给ServerFileHandler去处理

```java
package com.norman.threadpool;

import java.io.IOException;
import java.net.ServerSocket;
import java.net.Socket;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

/**
 * BIO服务端源码__伪异步I/O
 *
 * @author norman
 * @version 1.0
 */

public final class Server {
    //默认监听的端口号
    private static int DEFAULT_PORT = 12345;
    //单例的ServerSocket
    private static ServerSocket serverSocket;
    //线程池 懒汉式的单例
    private static ExecutorService executorService = Executors.newFixedThreadPool(32);
    private static int count = 0;

    //根据传入参数设置监听端口，如果没有参数调用以下方法并使用默认值
    public static void start() throws IOException {
        //使用默认值
        start(DEFAULT_PORT);
    }

    //这个方法不会被大量并发访问，不太需要考虑效率，直接进行方法同步就行了
    private static synchronized  void start(int port) throws IOException {
        if (serverSocket != null) {
            return;
        }
        try {
            //通过构造函数创建ServerSocket
            //如果端口合法且空闲，服务端就监听成功
            serverSocket = new ServerSocket(port);
            System.out.println("服务器已启动，端口号：" + port);
            Socket socket;
            //通过无线循环监听客户端连接
            //如果没有客户端接入，将阻塞在accept操作上。
            while (true) {
                socket = serverSocket.accept();
                //当有新的客户端接入时，会执行下面的代码
                //然后创建一个新的线程处理这条Socket链路
                System.out.println("服务器已接收到第 " + ++count + " 个任务......");
                executorService.execute(new ServerFileHandler(socket, count));
            }
        } finally {
            //一些必要的清理工作
            if (serverSocket != null) {
                System.out.println("服务器已关闭。");
                serverSocket.close();
                serverSocket = null;
            }
        }
    }
}
```

#### `ServerFileHandler.java`

服务器端处理客户端发来的数据

```java
package com.norman.threadpool;

import java.io.File;
import java.io.FileOutputStream;
import java.io.IOException;
import java.io.InputStream;
import java.io.PrintWriter;
import java.net.Socket;

public class ServerFileHandler implements Runnable {
    private final int count;
    private Socket socket;

    public ServerFileHandler(Socket socket, int count) {
        this.count = count;
        this.socket = socket;
    }

    @Override
    public void run() {
        InputStream in = null;
        PrintWriter out = null;
        try {
            in = socket.getInputStream();
            out = new PrintWriter(socket.getOutputStream(), true);

            File dstFile = new File("/home/norman/temp/No." + count + ".deb");
            FileOutputStream fos = new FileOutputStream(dstFile);

            byte[] buf = new byte[4096];
            int len = 0;
            long size = 0L;

            long start = System.currentTimeMillis();
            while ((len = in.read(buf)) != -1) {
                fos.write(buf, 0, len);
                fos.flush();
                size += len;
            }
            out.println(size);
            out.flush();
            long time = System.currentTimeMillis() - start;
            double vv = size / 1024.0 / 1024 / (time / 1000.0);
            System.out.println("服务器已接受第 "
                    + count
                    + " 个任务对应的文件,\t"
                    + "耗时 "
                    + time / 1000.0
                    + "s,\t 上传速率 "
                    + vv
                    + " MB/s");
            fos.close();
            in.close();
            out.close();
        } catch (IOException ex) {
            ex.printStackTrace();
        }
    }
}
```


客户端发送文件后截图：

![客户端](http://ojxisatrc.bkt.clouddn.com/image/threadpool/client.png)

服务器端收到文件后截图：
![服务器端](http://ojxisatrc.bkt.clouddn.com/image/threadpool/server.png)

