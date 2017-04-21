---
title: 深入详解Java的NIO之Channel
date: 2016-12-13 17:30:58
tags: java
categories: java
keywords: [java,NIO,Channel]
description:
---

# 一、 简介
通道是java.nio在buffer之后的第二个主要创新。通道提供与I/O服务的直接连接。通道是一种在字节缓冲区和通道另一端的实体（通常是文件或套接字）之间高效传输数据的介质。通常通道与操作系统文件描述符具有一对一的关系。通道类提供了支持平台无关性所需的抽象，但仍然具有对现代操作系统的本机I/O建模的能力。通道是网关，通过它可以以最小的开销访问操作系统的本地I/O服务。

<!-- more -->

# 二、 Channel接口
通道与缓冲区不同，通道 API 主要由接口指定。不同的操作系统上通道实现（Channel Implementation）会有根本性的差异，所以通道 API 仅仅描述了可以做什么。因此很自然地，通道实现经常使用操作系统的本地代码。通道接口允许您以一种受控且可移植的方式来访问底层的 I/O服务。

通道类簇图如下：

{% asset_img nio-channel.png %}

`Channel` 作为NIO通道类的顶层类，是一个接口，如下：
```java
package java.nio.channels;

public interface Channel
{
    public boolean isOpen();
    public void close() throws IOException;
}
```
从 Channel 接口引申出的其他接口都是面向字节的子接口，包括 Writable ByteChannel 和ReadableByteChannel。这也正好支持了我们之前所学的：通道只能在字节缓冲区上操作。

# 三、 打开通道
作为我们都知道的，I/O 分为两大类：文件I/O 和 流I/O。因此，这也就对应两种通道类型：`FileChannel`类 和 `SocketChannel`类

`FileChannel` 对象只能通过在一个打开的`RandomAccessFile`，`FileInputStream` 或 `FileOutputStream` 对象上调用 `getChannel()` 方法获得。示例：
```java
RandomAccessFile raf = new RandomAccessFile ("somefile", "r");
FileChannel fc = raf.getChannel();
```

相对于 `FileChannel`，`SocketChannel` 自身有工厂方法来创建新的socket通道，示例：
```java
//How to open SocketChannel
SocketChannel sc = SocketChannel.open();
sc.connect(new InetSocketAddress("somehost", someport));

//How to open ServerSocketChannel
ServerSocketChannel ssc = ServerSocketChannel.open();
ssc.socket().bind (new InetSocketAddress (somelocalport));

//How to open DatagramChannel
DatagramChannel dc = DatagramChannel.open();
```

# 四、 使用通道
通过实现以下接口的相关方法，可以实现读写操作：
```java
public interface ReadableByteChannel extends Channel
{
        public int read (ByteBuffer dst) throws IOException;
}

public interface WritableByteChannel extends Channel
{
        public int write (ByteBuffer src) throws IOException;
}

public interface ByteChannel extends ReadableByteChannel, WritableByteChannel
{
}
```

通道可以是单向或者双向的，一个通道类只实现 `ReadableByteChannel` 接口，那么它就是单向的只读通道；如果一个通道类只实现 `WritableByteChannel` 接口，那么它就是单向的只写通道。如果一个通道类两者都实现了，那么它就是双向的可读写通道。

文件通道和socket通道都实现了这三个接口，都是双向通道。

对于文件通道来说，有一点需要注意：通过 `FileInputStream` 对象的 `getChannel()` 方法获得的 `FileChannel` 对象是只读的，但是因为 `FileChannel` 实现了 `ByteChannel` ，所以从接口层面上是双向的。如果在 `FileChannel` 上调用 `write()` 方法将抛出 `NonWritableChannelException` 异常。因此，请记住，当通道连接到特定的I/O服务时，通道实例的功能将受其连接的服务的特性约束：连接到只读文件的通道实例无法写入，即使该通道实例所属的类可能具有write()方法。示例：
```java
FileInputStream input = new FileInputStream ("readOnlyFile.txt");
FileChannel channel = input.getChannel();

// This will compile but will throw an IOException
// because the underlying file is read-only
channel.write (buffer);
```

`ByteChannel` 的 `read()` 和 `write()` 方法以ByteBuffer对象作为参数。返回传输的字节数，可以小于缓冲区中的字节数，甚至为零。如果执行了部分传输，则可以将缓冲区重新传递到通道，以在其中断的地方继续传输数据。如此重复，直到缓冲区的 `hasRemaining()` 方法返回false。

# 五、工作模式
通道可以工作在阻塞或非阻塞模式。非阻塞模式下，所请求的操作可以立即完成，也可以返回表示什么都没有完成的结果。

只有面向流的信道，例如套接字和管道，可以被置于非阻塞模式。

`FileChannel` 总是阻塞的，因此不能被设置成非阻塞模式

# 六、 关闭通道
想要关闭通道，需要使用 `close()` 方法。一个被关闭的通道不能再被使用。

可以对一个通道多次调用 `close()` 方法，对一个已关闭的通道调用 `close()` 方法时，什么都不做，直接返回。

可以使用 `isOpen()` 放来检查一个通道的打开状态。

