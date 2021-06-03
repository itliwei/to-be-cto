## Netty

[TOC]

上一文我们讲了Java中的几个IO模型，其中讲到NIO的优越性。没错，NIO现在Java程序中网络通信中使用极为频繁的。比如大名鼎鼎的Tomcat、Dubbo、RabbitMQ等都使用到了NIO，而且使用的都是Netty一种NIO的开发框架。

### Netty是什么

> Netty is a NIO client server framework which enables quick and easy development of network applications such as protocol servers and clients. It greatly simplifies and streamlines network programming such as TCP and UDP socket server.

Netty是一个NIO客户端服务器框架，可以**快速简单**地开发网络应用程序，例如协议服务器和客户端。它极大地简化和简化了诸如TCP和UDP套接字服务器之类的网络编程。

Netty经过作者精心设计，结合了许多协议（例如FTP，SMTP，HTTP以及各种基于二进制和文本协议）的经验，成功地找到了一种在损失性能和可维护性的前提下，即可轻松实现开发，性能，稳定性和灵活性的方法。

那么Netty究竟有哪些设计呢？

### Netty架构设计

下图为Netty的架构图：

![image-20210422214453513](/Users/vince/Library/Application Support/typora-user-images/image-20210422214453513.png)

从图上可以看出Netty主要可以分为三块，最底层的一块是Netty设计的核心，包括了：

​	事件模型可以可将关注点明确分离；

​	各种传输类型的统一API，支持阻塞和非阻塞；

​	丰富的ByteBuffer支持Zero-copy。

其次有传输层的协议的支持，这一层可以实现在网络中对数据的高效序列化、压缩从而减少网络传输成本。

还有的就是对用户层协议的支持，提供了丰富的协议支持，包括：TCP、UDP、HTTP等。

接下来我们从这张架构图上分析以下Netty的原理设计。

#### 1.缓冲区结构设计

Netty使用其自己的缓冲区的实现而不是原生NIO的`ByteBuffer` 来表示字节序列。

比如，在通信层之间传输数据时，如果数据太大，我们通常需要对数据进行切片传送，接收后我们则需要将其进行合并。传统上，来自多个数据包的数据通过将它们复制到新的字节缓冲区中进行组合，但是Netty通过`ChannelBuffer`“指向”所需的缓冲区，从而减少了数据的复制。如图所示：

![image-20210422222529194](/Users/vince/Library/Application Support/typora-user-images/image-20210422222529194.png)

除此之外，还提供了一个现成的动态缓冲区类型，其容量可以按需扩展，就像`StringBuffer`。

#### 2.通用I/O模型设计

我们假设一开始客户端数量很少，我们使用BIO模型开发简单便捷，快速实现服务上线。上线后不久，业务成倍增长且服务器需要同时为成千上万的客户提供服务，BIO模型显然吃不消啊。我们需要改造为NIO模型，但是Java的NIO模型与旧的BIO模型不兼容！我们需要重新改造系统，这是不是很坑？

这时候就体现了Netty的价值了。Netty有一个通用异步I/O接口[`Channel`](http://static.netty.io/3.5/api/org/jboss/netty/channel/Channel.html)，该接口抽象了通信所需的所有操作。且实现了多种类型，包括：

- 基于NIO的TCP / IP传输
- 基于BIO的TCP / IP传输
- 基于BIO的UDP/IP传输
- 本地传输

从一种传输方式切换到另一种传输方式通常只需要选择其他[`ChannelFactory`](http://static.netty.io/3.5/api/org/jboss/netty/channel/ChannelFactory.html) 实现。这样就做到了快速、灵活的切换。是不是很妙？

可以看到Netty并非只是一个NIO框架，也同时支持BIO，这也是Netty如此受欢迎的原因。

#### 3.基于拦截器链模式的事件模型

事件驱动，必然减少了系统之间的耦合，具有良好的可扩展性。Netty有一个针对I/O的定义明确的事件模型。在一个 `ChannelPipeline`内部一个 `ChannelEvent` 被一组`ChannelHandler` 处理。因此对于一个事件如何被处理以及`channel`内部处理器间的处理过程，用户都可以把控。事实上，Netty内部的连接处理、协议编解码、超时等机制，都是通过`Handler`完成的。

![image-20210422230148964](/Users/vince/Library/Application Support/typora-user-images/image-20210422230148964.png)

#### 4.其他高级组件

##### 4.1 编解码器

Netty 提供了一组构建在其核心模块之上的编解码实现，这样就省去了自己对数据进行编解码，从而避免了很多问题。同时还提供了性能非常高的Protobuf协议的支持。

##### 4.2 SSL/TLS

NIO 模式下支持 SSL 功能是一个困难的工作，必须借助于SSLEngine，而SSLEngine的维护也非常困难管理所有可能的状态，例如密码套件，密钥协商，证书交换以及认证等。在 Netty 内部，`SslHandler`封装了所有艰难的细节以及使用 SSLEngine 可 能带来的陷阱。你所做的仅是配置并将该 `SslHandler` 加入到你的 `ChannelPipeline` 中。

### Netty线程模型



#### 1.线程模型实现方案：Reactor

Netty的线程模型是采用reactor设计

#### 2.事件驱动实现方案







