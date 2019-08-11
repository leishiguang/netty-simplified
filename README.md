

[toc]

# Netty原理 & 手写简化版Netty

## 背景&目的

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

**本文分为两个部分：**

* 手写简化版 Netty
* Netty 原理

## 手写简化版Netty

netty 源码稍微有些复杂，考虑了许多可扩展、安全、性能等，netty 的核心就隐藏在这百花深处。那么，何不拨开这些，自己尝试尝试，触及 netty 的核心原理呢~

陆续完善中，详见 github：[https://github.com/leishiguang/netty-simplified.git](https://github.com/leishiguang/netty-simplified.git)


## Netty 原理

### Reactor 线程模型

![48ea5d631c7d5e3d591a3ff61c8d7c4b.png](https://img-blog.csdnimg.cn/20190811100216523.png)

大致的步骤是这样：

1. 客户端的请求进来，首先到达 reactor 线程，由它专门负责轮询；
2. reactor 线程不对请求进行处理，而是轮询事件，分发到对应的 worker 线程；
3. worker 线程依据配置，继续分发，或者对请求进行处理；

#### 简化为两个线程组

**抽象线程对象**

- 每个线程有自己的选择器：来源于构造器，与线程对象一起完成初始化。
- 有自己的任务队列：保存所需要处理的工作。
- 通道注册的方法：提交注册任务到自己的任务队列，会把通道注册在自己的选择器上。
- 有启动的方法与状态：线程对象初次创建的时候，没有 Thread 正在工作，需要手动激活。
- run方法：循环做两件事情，执行任务队列中的任务、从自己的 selector 中获取事件并执行 handler 方法。
- 抽象的 handler 方法，交给实现类去处理，收到事件之后的不同处理逻辑。

其中：

1. 任务队列为 LinkedBlockingQueue<Runnable>，存储可执行的工作任务，直接调用对应的 run 方法执行，如 FutureTask.run()。
2. 注册方法会阻塞，直到注册任务处理完。假如工作队列太多任务堆积，会影响系统性能。
3. 抽象线程对象继承于 Thread，由其它类调用 Thread 的 start 方法进行启动。
4. 假如通道已经关闭，或者该通道发生异常，则自动取消这个该通道事件的订阅。

**两个线程区别**

MainReactorThread 与 SubReactorThread 线程，分别用于处理 ACCEPT 事件与其它 I/O 事件。

MainReactorThread：

- 在 main 方法中被调用 start 方法；
- 只注册了 ServerSocketChanel 的 ACCETP 事件；
- 知道全部 SubReactorThread 对象；
- 具有一个从 SubReactorThread 组中挑选出某一个 Thread 的方法；
- 具有启动 SubReactorThread 的方法；
- 将 channnel 视为 ServerSocketChannel；
- 调用 ServerSocketChannel.accept() 方法取得一个客户端的 SocketChannel；
- 把 SocketCahnnel 注册到 I/O 线程，关注 READ 事件。

SubReactorThread：

- 在 MainReactorThread 方法中被调用 start 方法；
- 只注册了 SocketChannel 的 READ 事件；
- 知道更多的 workThread，用于业务操作；
- 具有挑选 workThread 的方法；
- 从 channel 中读取数据；
- 把业务操作提交到 workThread；
- 写入响应数据；

#### Selector 事件注册

正因为现在的 channel 是异步，每次调用 channel 的 write/read 方法，都可能没有数据。
于是，就有了 Selector 检查一个或多个 NIO 通道，并确定好哪些通道已经准备好进行读取或写入。
借此实现了一个线程管理多个 channel~

**事件驱动机制**

更底层的是操作系统的多路复用机制，却也可以按这种方式理解：

1. selector 是 channel 的观察者，channel 是消息发布者。
2. selector 从 channel 订阅一些主题，当 channel 有新消息的时候，会通知 selector。
3. selector 会将“通知”进行集中，并提供给开发者。

如此，当开发者调用 selector.select() 方法的时候，就可以集中处理当前已经就绪的 channel。

#### 事件响应与处理

对于不同事件的处理，安排给了不同的线程组。
每个线程对于不同的事件，也都有不一样的处理过程。

- 对于 ACCEPT 事件，需要确保响应速度，由 ACCEPT 线程组进行执行；
- 对于 IO 事件，高耗时的 IO 操作，与 ACCEPT 隔离开来，交给 IO 线程组执行；

注意：

如果要自己实现 selector，就需要考虑线程安全的问题。虽然 selector 只属于一个线程，但 selector 中的事件集合，是其它线程提供的。


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

netty 中的 Channel 是一个抽象的概念，可以理解为对 JDK Channel 的增强和拓展。增加了许多的属性和方法，具体的信息需要查看代码注释，比如这么几个常见的属性和方法：

- pipeline DefaultChannelPipeline：通道内事件处理链路
- eventLoop EventLoop：半丁的 Event Loop 用于执行操作
- unsafe Unsafe：提供 I/O 相关操作的封装
- config() ChannelConfig：返回通道配置信息
- read() Channel：开始读取数据，触发读取链路调用
- write(Object msg) ChannelFuture：写数据，触发写入链路调用
- bind(SocketAddress SocketAddress) ChannelFuture：绑定

### Pipeline 职责链

该模式为请求创建了一个处理对象的处理链，让发起请求和具体处理请求的过程进行解耦。职责链上的处理者负责处理请求，客户端只需要将请求发送到职责链上即可，无需关心请求处理细节和请求的传递。

在 netty 中，pipeline 管道保存了通道所有处理器的信息。每次创建新 channel 时自动创建一个专有的 pipeline。入站事件和出站操作会调用 pipeline 上的处理器。

**入站事件：** 通常指 I/O 线程生成了入栈数据。

比如 EventLoop 收到 selector 的 OP_READ 事件，入栈处理器调用 socketChannel.read(ByteBuffer) 接收到数据后，这将导致通道的 ChannelPipeline 中包含的下一个链，其中的 channelRead 方法被调用。

**出站事件：** 经常是指 I/O 线程执行实际的输出操作。

比如 bind 方法用意是请求 server socket 绑定到给定的 SocketAddress，这将导致通道的 ChannelPipeline 中包含的下一个出站处理器中的 bine 方法被调用。

**维护 Pipeline 的 handler**

ChannelPipeline 是线程安全的，ChannelHandler 可以在任何时候添加或删除。
例如，你可以在即将交换敏感信息时插入加密处理程序，并在交换后删除它。
一般的操作，初始化的时候加进去，较少删除~

### ByteBuf 复用机制

ByteBuffer 可以理解为一个缓冲区，本质上是一个可以写入数据的内存块（类似数组），然后可以再次读取。相比直接对数组进行操作，Buffer API 更加容易操作和管理。

使用 Buffer 进行数据写入和读取，一般需要进行如下四个步骤：

1. 将数据写入缓冲区；
2. 调用 buffer.flip()，转换为读取模式；
3. 缓冲区读取数据；
4. 调用 buffer.clear() 或 buffer.compact() 清除缓冲区。

而 netty 在原有 ByteBuf 的基础上做了一些改进与增强，除了对象复用（减少内存的拷贝，提高系统性能）以外，也提供了新的 API，这部分多多练手，实际使用着就知道咯~

## 总结

是吧，[netty 很美妙吧](https://netty.io/)，学习了 netty 的线程模型之后，跟着代码走一走。再返回来继续研究 selector 线程模型，又去跟踪跟踪代码...

如此往复几次，对于 netty 的理解就会愈加深刻。
netty 官方也提供了许多的 example 示例，都很好理解~
也怪不得 netty 能够这么火热~

本篇笔记主要记录了 reactor 、channel 以及 pipeline，还有 netty 的 ByteBuf 复用机制没有深入探讨。留下点儿小小的遗憾，假想着以后还有机会的话……:D

最后呢，附一张 netty 的启动时序图

![d6200d3c8f90698f5fa401d609975b75.png](https://img-blog.csdnimg.cn/2019081110024461.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3Vlc3RjX3Rpbmc=,size_16,color_FFFFFF,t_70)

祝愿大家生活愉快

