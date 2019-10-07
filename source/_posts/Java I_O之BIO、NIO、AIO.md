---
title: Java I/O之BIO、NIO、AIO
date: 2019-9-22 23:20:00
author: xp
categories:
- Java
- IO
tags:
- Java
- IO
---

# Java I/O之BIO、NIO、AIO

## 什么是I/O

I/O就是输入（Input）和输出(Output)，针对不同的操作对象，可以划分为磁盘I/O模型，网络I/O模型，内存映射I/O, Direct I/O、数据库I/O等，只要具有输入输出类型的交互系统都可以认为是I/O系统，也可以说I/O是整个操作系统数据交换与人机交互的通道。一个系统的优化空间，往往都在低效率的I/O环节上。比如数据库查询慢时，我们通常会考虑到建索引，建索引的目的其实也是为了降低I/O次数。正因为如此，Java在I/O上也一直在做持续的优化，从JDK1.4以前的BIO，1.4及其以后的NIO，1.7的AIO。

## BIO（Blocking I/O）同步阻塞I/O

接下来以Socket编程来说明BIO、NIO、AIO的区别
**BIO Server端代码：**

```java
public class Server {
    private final static int PORT = 8765;

    public static void main(String[] args) {
        //try-with-resources
        try(ServerSocket serverSocket = new ServerSocket(PORT)) {
            System.out.println("server start ...");
            Socket socket = serverSocket.accept();
            //在使用传统BufferedReader的readLine方法读取时，在该方法成功返回之前，线程被阻塞，客户端读取服务器数据的线程同样会被阻塞。
            // 所以服务器应该为每个socket，开启一个线程
            //每当客户端连接后启动一个线程为该客户端服务
            //可使用线程池代替
            new Thread(new ServerHandler(socket)).start();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

**BIO ServerHandler代码：**

```java
public class ServerHandler implements Runnable {

    private Socket socket;

    public ServerHandler(Socket socket) {
        this.socket = socket;
    }

    @Override
    public void run() {
        try (BufferedReader in = new BufferedReader(
                new InputStreamReader(this.socket.getInputStream()))) {
            PrintWriter out = new PrintWriter(this.socket.getOutputStream(), true);
            Thread.sleep(3000);
            while (true) {
                String body = in.readLine();
                if (body == null) {
                    break;
                }
                System.out.println("Server：服务器接收到的数据：" + body);
                out.println("这是服务器端响应的数据");
            }
        } catch (IOException | InterruptedException e) {
            e.printStackTrace();
        }

    }
}
```

**BIO Client代码：**

```java
public class Client {

    private final static String ADDRESS = "127.0.0.1";
    private final static int PORT = 8765;

    public static void main(String[] args) {
        try (Socket socket = new Socket(ADDRESS, PORT)) {
            BufferedReader in = new BufferedReader(new InputStreamReader(socket.getInputStream()));
            PrintWriter out = new PrintWriter(socket.getOutputStream(),true);
            //向服务器发送数据
            out.println("向服务器请求数据...");
            String response = in.readLine();
            System.out.println("Client：服务器返回数据为：" + response);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

BIO主要的问题在于不支持太多的客户端连接，每当有一个新的客户端请求接入时，服务端必须创建一个新的线程来处理这条链路，在需要满足高性能、高并发的场景是没法应用的（大量创建新的线程会严重影响服务器性能）。
![BIO模型图](https://img-blog.csdnimg.cn/20181202164559671.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIxNTczODk5,size_16,color_FFFFFF,t_70)

### 伪异步I/O编程

为了解决一个连接一个线程导致创建线程线程过多，影响服务器性能的问题，可使用线程池来管理这些线程，实现伪异步。只需将server端代码修改，再加一个连接池就OK。
**伪异步 server端代码：**

```java
public class Server {
    private final static int PORT = 8765;

    public static void main(String[] args) {
        //try-with-resources
        try(ServerSocket serverSocket = new ServerSocket(PORT)) {
            System.out.println("server start ...");
            HandlerExecutorPool executorPool = new HandlerExecutorPool(50, 1000);
            while(true){
                Socket socket = serverSocket.accept();
                executorPool.execute(new ServerHandler(socket));
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

**连接池HandlerExecutorPool 代码：**

```java
public class HandlerExecutorPool {

    private ExecutorService executor;

    public HandlerExecutorPool(int maxPoolSize, int queueSize) {
        this.executor = new ThreadPoolExecutor(
                Runtime.getRuntime().availableProcessors(),
                maxPoolSize,
                120L,
                TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(queueSize));
    }

    public void execute(Runnable task) {
        this.executor.execute(task);
    }
}
```

使用线程池可以管理线程（复用），防止线程频繁的创建引起的性能开销，同时也限制了线程的数量，超过最大数量的线程就只能等待，直到线程池中的有空闲的线程可以被复用。所以在大量并发的情况下，读取数据很慢。

![伪异步](https://img-blog.csdnimg.cn/20181202172429246.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIxNTczODk5,size_16,color_FFFFFF,t_70)

关于线程池，可查看我这篇文章：[线程池](https://blog.csdn.net/qq_21573899/article/details/83960456)

## NIO(New I/O) 同步非阻塞I/O

BIO和NIO的区别：其本质就是阻塞和非阻塞的区别。

 -阻塞：应用程序在获取网络数据的时候，如果网络传输数据很慢，那么程序就一直等着，直到传输完毕。
 -非阻塞：应用程序直接可以获取已经准备就绪好的数据，无需等待。

![BIO NIO](https://img-blog.csdnimg.cn/2018120220353379.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIxNTczODk5,size_16,color_FFFFFF,t_70)

### NIO组成

- 缓冲区 Buffer
 &ensp;&ensp;在NIO库中，所有数据都是用缓冲区处理的。在读取数据时，它是直接读到缓冲区中的；在写入数据时，也是写入到缓冲区中。任何时候访问NIO中的数据，都是通过缓冲区进行操作。缓冲区实际上是一个数组，并提供了对数据结构化访问以及维护读写位置等信息。

 具体的缓存区有这些：ByteBuffer、CharBuffer、DoubleBuffer、 FloatBuffer、IntBuffer、 LongBuffer,、ShortBuffer。分别对应byte、char、double、 float、int、 long、 short。他们实现了相同的接口：Buffer。

- 通道 Channel

&ensp;&ensp;我们对数据的读取和写入要通过Channel，它就像水管一样，是一个通道。通道不同于流的地方就是通道是双向的，可以用于读、写和同时读写操作。Channel和IO中的Stream(流)是差不多一个等级的。只不过Stream是单向的，譬如：InputStream, OutputStream。而Channel是双向的，既可以用来进行读操作，又可以用来进行写操作，NIO中的Channel的主要实现有：FileChannel、DatagramChannel、SocketChannel、ServerSocketChannel，分别可以对应文件IO、UDP和TCP（Server和Client）。

- 多路复用器 Selector

&ensp;&ensp;Selector提供选择已经就绪的任务的能力：Selector会不断轮询注册在其上的Channel，如果某个Channel上面发生读或者写事件，这个Channel就处于就绪状态，会被Selector轮询出来，然后通过SelectionKey可以获取就绪Channel的集合，进行后续的I/O操作。
&ensp;&ensp;一个Selector可以同时轮询多个Channel，因为JDK使用了epoll()代替传统的select实现，所以没有最大连接句柄1024/2048的限制。所以，只需要一个线程负责Selector的轮询，就可以接入成千上万的客户端。这是Java NIO库的巨大进步。
&ensp;&ensp;每个管道都会对选择器进行注册不同的事件状态，以便选择器查找。
 &ensp;&ensp;SelectionKey.OP_CONNECT、SelectionKey.OP_ACCEPT、SelectionKey.OP_READ、SelectionKey.OP_WRITE

### 示例

**服务端实现Server：**

```java
public class Server implements Runnable {

    //1 多路复用器（管理所有的通道）
    private Selector seletor;
    //2 建立读缓冲区
    private ByteBuffer readBuf = ByteBuffer.allocate(1024);
   //3 建立写缓冲区
    private ByteBuffer writeBuf = ByteBuffer.allocate(1024);
    public Server(int port) {
        try {
            //1 打开路复用器
            this.seletor = Selector.open();
            //2 打开服务器通道
            ServerSocketChannel ssc = ServerSocketChannel.open();
            //3 设置服务器通道为非阻塞模式
            ssc.configureBlocking(false);
            //4 绑定地址
            ssc.bind(new InetSocketAddress(port));
            //5 把服务器通道注册到多路复用器上，并且监听阻塞事件
            ssc.register(this.seletor, SelectionKey.OP_ACCEPT);
            System.out.println("Server start, port :" + port);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    @Override
    public void run() {
        while (true) {
            try {
                //1 必须要让多路复用器开始监听
                this.seletor.select();
                //2 返回多路复用器已经选择的结果集
                Iterator<SelectionKey> keys = this.seletor.selectedKeys().iterator();
                //3 进行遍历
                while (keys.hasNext()) {
                    //4 获取一个选择的元素
                    SelectionKey key = keys.next();
                    //5 直接从容器中移除就可以了
                    keys.remove();
                    //6 如果是有效的
                    if (key.isValid()) {
                        //7 如果为阻塞状态
                        if (key.isAcceptable()) {
                            this.accept(key);
                        }
                        //8 如果为可读状态
                        if (key.isReadable()) {
                            this.read(key);
                        }
                        //9 写数据
                        if (key.isWritable()) {
                            //this.write(key); //ssc
                        }
                    }

                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    private void write(SelectionKey key) {
        //ServerSocketChannel ssc =  (ServerSocketChannel) key.channel();
        //ssc.register(this.seletor, SelectionKey.OP_WRITE);
    }

    private void read(SelectionKey key) {
        try {
            //1 清空缓冲区旧的数据
            this.readBuf.clear();
            //2 获取之前注册的socket通道对象
            SocketChannel sc = (SocketChannel) key.channel();
            //3 读取数据
            int count = sc.read(this.readBuf);
            //4 如果没有数据
            if (count == -1) {
                key.channel().close();
                key.cancel();
                return;
            }
            //5 有数据则进行读取 读取之前需要进行复位方法(把position 和limit进行复位)
            this.readBuf.flip();
            //6 根据缓冲区的数据长度创建相应大小的byte数组，接收缓冲区的数据
            byte[] bytes = new byte[this.readBuf.remaining()];
            //7 接收缓冲区数据
            this.readBuf.get(bytes);
            //8 打印结果
            String body = new String(bytes).trim();
            System.out.println("Server : " + body);
            // 9..可以写回给客户端数据
        } catch (IOException e) {
            e.printStackTrace();
        }

    }

    private void accept(SelectionKey key) {
        try {
            //1 获取服务通道
            ServerSocketChannel ssc = (ServerSocketChannel) key.channel();
            //2 服务器端执行阻塞方法，等待客户端连接，当有客户端连接成功之后，返回这个客户端连接
            SocketChannel sc = ssc.accept();
            //3 设置非阻塞模式
            sc.configureBlocking(false);
            //此时该客户端和服务端已经建立连接
            //4 将客户端SocketChannel注册到多路复用器上，并设置读取标识，下次论询时直接可以读取。
            sc.register(this.seletor, SelectionKey.OP_READ);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) {
        new Thread(new Server(8765)).start();
    }
}
```

**客户端实现client：**

```java
public class Client {

    //需要一个Selector
    public static void main(String[] args) {
        //创建连接的地址
        InetSocketAddress address = new InetSocketAddress("127.0.0.1", 8765);
        //声明连接通道
        SocketChannel sc = null;
        //建立缓冲区
        ByteBuffer buf = ByteBuffer.allocate(1024);
        try {
            //打开通道
            sc = SocketChannel.open();
            //进行连接（注册到selector上）
            sc.connect(address);
            while (true) {
                //定义一个字节数组，然后使用系统录入功能：
                byte[] bytes = new byte[1024];
                System.in.read(bytes);
                //把数据放到缓冲区中
                buf.put(bytes);
                //对缓冲区进行复位
                buf.flip();
                //写入数据(执行了这个操作，server端的key.isReadable()才是true，才是可读状态)
                sc.write(buf);
                //清空缓冲区数据
                buf.clear();
            }
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            if (sc != null) {
                try {
                    sc.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}
```

运行结果：先启动server，再启动client，client将系统录入的值，发送给server并输出。
NIO的运行流程如下：
![NIO概述](https://img-blog.csdnimg.cn/20181202215847951.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzIxNTczODk5,size_16,color_FFFFFF,t_70)

> 本例仅仅是将client端的数据发送给server端，目的是通过API来介绍NIO。

## AIO（Asynchronous I/O）：异步非阻塞I/O

&ensp;&ensp;在NIO基础之上引入了异步通道的概念，AIO不需要通过多路复用器对注册的通道进行仑村操作即可实现异步读写，从而简化了NIO编程模型，也称之为NI02.0。AIO的异步特性并不是Java实现的伪异步，而是使用了系统底层API的支持，在Unix系统下，采用了epoll IO模型，而windows便是使用了IOCP模型。
AIO和BIO的区别：在于**异步**

- 同步：应用程序会直接参与IO读写操作，并且我们的应用程序会直接阻塞到某一个方法上，直到数据准备就绪；或者采用轮询的策略实时检查数据的就绪状态，如果就绪则获取数据。

- 异步时，则所有的IO读写操作交给操作系统处理，与我们的应用程序没有直接关系，我们程序不需要关心IO读写，当操作系统完成了IO读写操作后，会给应用程序发送信号，此时应用程序直接拿走数据即可。
**Server端代码：**

```java
public class Server {

    //线程池
    private ExecutorService executorService;
    //线程组
    private AsynchronousChannelGroup threadGroup;
    //服务器通道
    public AsynchronousServerSocketChannel assc;

    public Server(int port) {
        try {
            //创建一个缓存池
            executorService = Executors.newCachedThreadPool();
            //创建线程组
            threadGroup = AsynchronousChannelGroup.withCachedThreadPool(executorService, 1);
            //创建服务器通道
            assc = AsynchronousServerSocketChannel.open(threadGroup);
            //进行绑定
            assc.bind(new InetSocketAddress(port));
            System.out.println("server start , port : " + port);
            //进行阻塞
            assc.accept(this, new ServerCompletionHandler());
            //一直阻塞 不让服务器停止
            Thread.sleep(Integer.MAX_VALUE);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) {
        Server server = new Server(8765);
    }
}
```

**ServerCompletionHandler端代码：**

```java
public class ServerCompletionHandler implements CompletionHandler<AsynchronousSocketChannel, Server> {

    @Override
    public void completed(AsynchronousSocketChannel asc, Server attachment) {
        //当有下一个客户端接入的时候 直接调用Server的accept方法，这样反复执行下去，保证多个客户端都可以阻塞
        //如果注释掉该方法，则只能获取到第一个客户端的数据
        attachment.assc.accept(attachment, this);
        read(asc);
    }

    private void read(final AsynchronousSocketChannel asc) {
        //读取数据
        ByteBuffer buf = ByteBuffer.allocate(1024);
        asc.read(buf, buf, new CompletionHandler<Integer, ByteBuffer>() {
            @Override
            public void completed(Integer resultSize, ByteBuffer attachment) {
                //进行读取之后,重置标识位
                attachment.flip();
                //获得读取的字节数
                System.out.println("Server -> " + "收到客户端的数据长度为:" + resultSize);
                //获取读取的数据
                String resultData = new String(attachment.array()).trim();
                System.out.println("Server -> " + "收到客户端的数据信息为:" + resultData);
                String response = "服务器响应, 收到了客户端发来的数据: " + resultData;
                write(asc, response);
            }
            @Override
            public void failed(Throwable exc, ByteBuffer attachment) {
                exc.printStackTrace();
            }
        });
    }

    private void write(AsynchronousSocketChannel asc, String response) {
        try {
            ByteBuffer buf = ByteBuffer.allocate(1024);
            buf.put(response.getBytes());
            buf.flip();
            asc.write(buf).get();
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }
    }

    @Override
    public void failed(Throwable exc, Server attachment) {
        exc.printStackTrace();
    }

}
```

**Client端代码：**

```java
public class Client implements Runnable {

    private AsynchronousSocketChannel asc;

    public Client() throws Exception {
        asc = AsynchronousSocketChannel.open();
    }

    public void connect() {
        asc.connect(new InetSocketAddress("127.0.0.1", 8765));
    }

    public void write(String request) {
        try {
            asc.write(ByteBuffer.wrap(request.getBytes())).get();
            read();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    private void read() {
        ByteBuffer buf = ByteBuffer.allocate(1024);
        try {
            asc.read(buf).get();
            buf.flip();
            byte[] respByte = new byte[buf.remaining()];
            buf.get(respByte);
            System.out.println(new String(respByte, "utf-8").trim());
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        } catch (UnsupportedEncodingException e) {
            e.printStackTrace();
        }
    }

    @Override
    public void run() {
        while (true) {

        }
    }

    public static void main(String[] args) throws Exception {
        Client c1 = new Client();
        c1.connect();
        Client c2 = new Client();
        c2.connect();
        Client c3 = new Client();
        c3.connect();
        new Thread(c1, "c1").start();
        new Thread(c2, "c2").start();
        new Thread(c3, "c3").start();
        Thread.sleep(1000);
        c1.write("c1 aaa");
        c2.write("c2 bbbb");
        c3.write("c3 ccccc");
    }
}
```

## 总结

这儿有一张BIO、NIO、AIO的对比图
![对比图](https://img-blog.csdnimg.cn/20181202232301637.png)

> 图片来自 http://blog.anxpp.com/usr/uploads/2016/05/3849862161.png

一般情况下，我们不需要手动编写NIO、AIO进行通信的代码，他们语法稍显晦涩。如果向进行网络通信，可以考虑更强大、更简单、性能强劲的NIO框架：**Netty**（DubboX、RocketMQ都有使用它）。
