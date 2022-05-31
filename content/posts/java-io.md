---
title: "Java IO"
date: 2022-05-26T08:59:56+08:00
description: "Intro to Java IO"
tags: ["java", "io", "nio"]
categories: ["java"]
author: "杨晓峰·geektime"
draft: true
---

Java IO 方式有很多种，基于不同的 IO 抽象模型和交互方式，可以进行简单区分。

+ Blocking IO，主要实现是在 java.io 包，基于流模型实现，提供了我们熟知的如 File、InputStream、OutputStream 等抽象。交互方式是同步、阻塞的方式，也就是说，在读取 InputStream 或者写入 OutputStream 时，在读写操作完成前，线程会一直阻塞，它们之间的调用是可靠的线性顺序。
  
  java.io 包的好处是代码比较简单、直观，缺点是 IO 效率和扩展性存在局限性，容易成为应用性能的瓶颈。很多时候，人们也把 java.net 下面提供的部分网络 API，如 Socket、HttpURLConnection 也归类到同步阻塞 IO 类库，因为网络通信同样是 IO 行为。

+ Non-blocking IO，主要实现在 java.nio 包，于 Java 1.4 引入，通常称 NIO 框架，提供了 Channel、Selector、Buffer 等新的抽象，可以构建多路复用的、同步非阻塞 IO 程序，同时提供了更接近操作系统底层的高性能数据操作方式。
  
  NIO 在 Java7 中有了进一步的改进，也就是 NIO2，引入了异步非阻塞 IO 方式，又称为 AIO（Asynchronous IO）。异步 IO 操作基于事件和回调机制，可以简单理解为，应用操作直接返回，而不是阻塞在那里，当后台处理完成，操作系统会通知线程进行后续工作。

## 一些基本概念

+ **同步**（synchronous）简单来说，同步是一种可靠的有序运行机制，当我们进行同步操作时，后续的任务时等待当前调用返回，才会进行下一步。
+ **异步**（asynchronous）与同步相反，其他任务不需要等待当前调用翻译，通常依靠**事件**、**回调**等机制来实现任务间次序关系。
+ **阻塞**（blocking） 在进行阻塞操作时，当前线程会处于阻塞状态，无法从事其他任务，只有当条件就绪才能继续，如 Socket 新连接建立完毕，或数据读取、写入操作完成。
+ **非阻塞**（non-blocking）非阻塞不管 IO 操作是否结束，直接返回，相应操作在后台继续处理。

## Java IO 简介

Java IO 简单的类图如下：

![java-io](/img/java-io.webp)

### 日常开发常用的一些类的要点

根据类图我们知道很多 IO 工具类都实现了 **Closeable** 接口，因为需要进行资源的释放。例如，打开 **FileInputStream**，就会获取相应的文件描述符（**FileDescriptor**），需要利用 try-with-resources，try-finally 等机制保证 FileInputStream 被明确关闭，进而相应的文件描述符也会失效，否则将导致资源无法被释放。

**InputStream/OutputStream** 用于读取或写入字节，如加载图片、下载文件等；而 **Reader/Writer** 则是用于操作字符，增加了字符编解码等功能，适用于从文件读取或写入文本信息。本质上 PC 的操作都是字节，不管是网络通信还是文件读取，Reader/Writer 相当于构建了应用逻辑和原始数据之间的桥梁。

**BufferedOutputStream** 等带缓冲区的实现，可以避免频繁的磁盘读写，进而提高 IO 处理效率。这种设计利用了缓冲区，将批量数据进行一次操作，需要**注意的是使用中不要忘记 flush**。

## Java NIO 简介

### NIO 的基本组成

+ **Buffer** 高效的数据容器，除了 bool 类型，所有原始数据类型都有相应的 Buffer 实现。
+ **Channel** 类似在 Linux 操作系统上看到的文件描述符，是 NIO 中用来支持批量式 IO 操作的一种抽象。
  
  File 或 Socket，通常也被认为是比较高层次的抽象，而 **Channel 则是操作系统底层的一种抽象**，这也使得 NIO 得以充分利用现代操作系统底层机制，获得特定场景的性能优化，例如 DMA（Direct Memory Access）等。不同层次的抽象是相互关联的，我们可以利用 Socket 获取 Channel，反之亦然。
+ **Selector** NIO实现多路复用的基础，它提供了一种高效机制：可以检测注册在 Selector 上的多个 Channel 中，是否有 Channel 处于就绪状态，进而实现了单线程对多 Channel 的高效管理。**Selector 同样是基于底层操作系统机制**，不同模式、不同版本都存在区别，例如在 Linux 上的实现基于 epoll，而在 Windows 上 AIO 模式则依赖于 iocp.
+ **Charset** 提供 Unicode 字符串定义，NIO 也提供了相应的 encode/decode 等，例如下面的例子：
  
  ``` java
  Charset.defaultCharset().encode("Sample");
  ```

### NIO 能解决什么问题

我们通过一个例子来分析为什么需要 NIO，为什么需要多路复用。

用**IO 实现**一个简单的 Socket 服务器实现：

``` java
import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.io.PrintWriter;
import java.net.InetAddress;
import java.net.ServerSocket;
import java.net.Socket;

/**
 * @author: zac
 * @date: 2022/5/30
 */
public class SampleServer extends Thread {
    private ServerSocket socket;
    public int getPort() {
        return socket.getLocalPort();
    }
    public void run() {
        try {
            // a.启动 server
            // 端口 0 表示自动绑定一个空闲端口
            socket = new ServerSocket(0);
            while(true) {
                // 调用 accept 方法，阻塞等待 client 连接
                RequestHandler handler = new RequestHandler(socket.accept());
                handler.start();
            }
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            if(socket != null) {
                try {
                    socket.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    public static void main(String[] args) throws IOException{
        SampleServer server = new SampleServer();
        server.start();
        // 利用 socket 模拟一个简单的 client，只进行连接、读取、打印。
        try (Socket client = new Socket(InetAddress.getLocalHost(), server.getPort())) {
            BufferedReader bufferedReader = new BufferedReader(new InputStreamReader(client.getInputStream()));
            bufferedReader.lines().forEach(System.out::println);
        }
    }
}

class RequestHandler extends Thread {
    private Socket socket;
    RequestHandler(Socket socket) {
        this.socket = socket;
    }
    @Override
    public void run() {
        try (PrintWriter out = new PrintWriter(socket.getOutputStream());) {
            out.println("This message send from server!");
            out.flush();
        } catch(Exception e) {
            e.printStackTrace();
        }
    }
}
```

这个方案在扩展性方面，会存在什么问题呢？

我们知道 Java 目前的线程实现是比较重量级的，启动或者销毁一个线程是有明显开销的，每个线程都有单独的线程栈等结构，需要占用非常明显的内存，所以每一个 client 启动一个线程会显得比较浪费。

那么，我们可以引入线程池机制来改造 server 的 run 方法

``` java
public class SampleServer extends Thread {
    ...
    // 端口 0 表示自动绑定一个空闲端口
    socket = new ServerSocket(0);
    executor = Executors.newFixedThreadPool(8);
    while(true) {
        // 调用 accept 方法，阻塞等待 client 连接
        RequestHandler handler = new RequestHandler(socket.accept());
        executor.execute(handler);
    }
    ...
}
```

如果连接数并不多，只有最多几百个连接的普通应用，这种模式往往可以工作的很好，但是如果连接数急剧上升，这种实现方式就无法很好地工作了，因为线程上下文切换开销会在高并发时变得很明显，这是同步阻塞方式的低扩展性劣势。

利用 **NIO 多路复用机制**，我们可以用另一种思路实现：

``` java
/**
 * NIOServer
 */
public class NIOServer extends Thread{
    @Override
    public void run() {
        // 通过 Selector.open() 创建了一个 Selector，作为类似调度员的角色
        try (Selector selector = Selector.open();
        ServerSocketChannel serverSocket = ServerSocketChannel.open();){
            serverSocket.bind(new InetSocketAddress(InetAddress.getLocalHost(), 1234));
            // 设置非阻塞模式，向 selector 注册 channel
            serverSocket.configureBlocking(false);
            // 注册到 Selector，并说明关注点
            serverSocket.register(selector, SelectionKey.OP_ACCEPT);
            while (true) {
                // 阻塞等待就绪的 channel
                selector.select();
                Set<SelectionKey> keys = selector.selectedKeys();
                Iterator<SelectionKey> iter = keys.iterator();
                while (iter.hasNext()) {
                    SelectionKey key = iter.next();
                    try( ServerSocketChannel server = (ServerSocketChannel)key.channel()) {
                        SocketChannel client = server.accept();
                        client.write(Charset.defaultCharset().encode("This message send from client"));
                    }
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

通过比较 IO 和 NIO 的实现，我们可以了解到，IO 同步阻塞模式，需要多线程切换来实现多任务处理。而 NIO 则是利用了单线程轮询事件的机制，通过高效地定位就绪的 Channel，来决定做什么，仅仅 select 阶段是阻塞的，可以有效避免大量客户端连接时，频繁线程切换带来的问题，应用的扩展能力有了非常大的提高。

## 面试扩展

对于 BIO、NIO、AIO 我们可以在很多地方展开：

+ 基础 API 功能与设计。
+ NIO 和 NIO2 的基本组成。
+ 给定场景，分别用不同的模型实现，分析 BIO、NIO 等模式的设计和实现原理。
+ NIO 提供的高性能数据操作方式是基于什么原理，如何使用？
+ 你觉得 NIO 自身实现存在哪些问题？有什么改进的想法吗？
