
# Netty原理 & 手写简化版Netty

[toc]

## 本文目的&背景

本文提出netty有这么4个核心概念：

1. Reactor线程模型：一种高性能的多线程程序设计思路
2. nettyChannel：增强版的通道概念
3. ChannelPipeline 职责链：事件处理机制
4. 内存管理：增强的 ByteBuf 缓冲区

这么 4 个概念结合在一起，让 netty 这么美妙：

1. 支持Socket等多种传输方式；
2. 提供了多种协议的编码实现；
3. **有多美妙，参看官网：[https://netty.io/](https://netty.io/)**

那么，试着手写一个简化版本的netty，探探这里面究竟是什么？

## Netty 原理

### Reactor 线程模型

![48ea5d631c7d5e3d591a3ff61c8d7c4b.png](en-resource://database/35963:1)

大致的步骤是这样：

1. 客户端的请求进来，首先到达 reactor 线程，由它专门负责轮询；
2. reactor 线程不对请求进行处理，而是轮询事件，分发到对应的 worker 线程；
3. worker 线程依据配置，继续分发，或者对请求进行处理；

要理解 reactor 线程的轮询，我从 netty 的启动（这儿是 netty 最基础的实现，即 echo ）开始说起：

#### 初始化和启动

**EventLoopGroup 初始化过程**

**EventLoop 的启动**

**Bind 绑定端口的过程**


### Channel 通道

与 BIO 的主要区别是：

- 不需要使用 inputStream/outputStream
- Channel 封装了更多 UDP/TCP 网络和文件 IO
- 即可读取，也可写入，使用 ByteBuff 实现了 write、read 的非阻塞。

在网络应用中有 ServerSocketChannel 与 SocketChannel 两类。

SocketChannel：

1. 既可以是服务端，也可以是客户端；
2. 表示了一个网络连接；
3. 需要循环调用 write、read，以确定读到数据、写到数据。

ServerSocketChannel：

1. 主要用于监听服务器端口，代表端口绑定、监听；
2. 一旦监听到了，可以用 accept 方法获取到 socketChannel
3. accept 可以选择异步方式。

#### NettyChannel 通道

### Pipeline 责任链


#### ChannelPipeline

**事件驱动机制**

**入站事件、出站事件**

**ChannelHandler**

#### handler 的执行分析


## 手写简化版Netty



代码地址：https://github.com/leishiguang/netty-simplified.git