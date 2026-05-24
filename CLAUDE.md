# Netty 源码学习指南

本项目是 Netty 4.1.x 源码，用于系统性学习 Netty 的设计思想与实现细节。

## 学习目标

通过阅读源码 + 运行示例 + 编写注释/笔记的方式，由浅入深掌握 Netty 核心原理。每个阶段结束后应能用自己的话解释该阶段的核心概念。

---

## 阶段一：快速上手 — 跑通一个例子（1-2天）

**目标：** 建立直觉，理解 Netty 程序的基本骨架。

**入口文件：**
- `example/src/main/java/io/netty/example/echo/` — 最简单的 Echo 服务端/客户端
- `example/src/main/java/io/netty/example/discard/` — 丢弃服务器（比 echo 更简单）
- `example/src/main/java/io/netty/example/telnet/` — 基于文本行的交互式示例

**关键问题：**
1. `ServerBootstrap` 的 `group()`, `channel()`, `childHandler()` 各做了什么？
2. `ChannelInitializer.initChannel()` 何时被调用？
3. 一条消息从网络到达到被你的 Handler 处理，经历了哪些步骤？

**验证方式：** 能独立写出一个简单的 TimeServer（发送当前时间后关闭连接）。

---

## 阶段二：核心模型 — Channel、EventLoop、Pipeline（3-5天）

**目标：** 理解 Netty 的三大核心抽象及其协作关系。

### 2.1 EventLoop 线程模型
- `common/src/main/java/io/netty/util/concurrent/` — `EventExecutor`, `SingleThreadEventExecutor`
- `transport/src/main/java/io/netty/channel/nio/NioEventLoop.java` — NIO 事件循环的核心实现
- `transport/src/main/java/io/netty/channel/nio/NioEventLoopGroup.java`

**关键问题：**
1. `NioEventLoop.run()` 的主循环做了哪三件事？（select → processSelectedKeys → runAllTasks）
2. 一个 EventLoop 绑定多少个线程？一个 Channel 绑定到哪个 EventLoop？
3. 为什么 Netty 说自己是"无锁化"的？

### 2.2 Channel 体系
- `transport/src/main/java/io/netty/channel/Channel.java` — 顶层接口
- `transport/src/main/java/io/netty/channel/AbstractChannel.java` — 模板方法模式
- `transport/src/main/java/io/netty/channel/socket/nio/NioSocketChannel.java`
- `transport/src/main/java/io/netty/channel/socket/nio/NioServerSocketChannel.java`

**关键问题：**
1. `Channel`, `Unsafe`, `Pipeline` 三者的关系是什么？
2. `NioServerSocketChannel` 如何 accept 新连接并注册到 childGroup？

### 2.3 ChannelPipeline 与 ChannelHandler
- `transport/src/main/java/io/netty/channel/ChannelPipeline.java`
- `transport/src/main/java/io/netty/channel/DefaultChannelPipeline.java`
- `transport/src/main/java/io/netty/channel/ChannelHandlerContext.java`
- `transport/src/main/java/io/netty/channel/AbstractChannelHandlerContext.java`

**关键问题：**
1. Pipeline 是什么数据结构？Inbound 和 Outbound 事件分别如何传播？
2. `ctx.fireChannelRead()` vs `ctx.channel().pipeline().fireChannelRead()` 区别？
3. `HeadContext` 和 `TailContext` 各承担什么职责？

**验证方式：** 画出一个包含 3 个 Handler 的 Pipeline，标注一条消息从入站到出站的完整流转路径。

---

## 阶段三：ByteBuf 内存管理（3-5天）

**目标：** 理解 Netty 高性能内存管理的设计。

**入口文件：**
- `buffer/src/main/java/io/netty/buffer/ByteBuf.java` — 核心接口
- `buffer/src/main/java/io/netty/buffer/AbstractByteBuf.java`
- `buffer/src/main/java/io/netty/buffer/PooledByteBufAllocator.java` — 池化分配器
- `buffer/src/main/java/io/netty/buffer/UnpooledByteBufAllocator.java`
- `buffer/src/main/java/io/netty/buffer/CompositeByteBuf.java` — 零拷贝组合

**关键问题：**
1. `readerIndex` / `writerIndex` 双指针设计相比 `java.nio.ByteBuffer` 的 `position/limit/flip` 好在哪里？
2. 池化分配的 Arena → Chunk → Page → Subpage 层级是怎样的？
3. 引用计数（`ReferenceCounted`）何时 retain，何时 release？泄漏了怎么检测？
4. `CompositeByteBuf` 如何实现零拷贝？

**验证方式：** 用 `-Dio.netty.leakDetection.level=PARANOID` 跑测试，理解泄漏检测日志。

---

## 阶段四：编解码器（2-3天）

**目标：** 理解 Netty 如何解决粘包/拆包问题，掌握编解码模式。

**入口文件：**
- `codec/src/main/java/io/netty/handler/codec/ByteToMessageDecoder.java` — 解码器基类
- `codec/src/main/java/io/netty/handler/codec/MessageToByteEncoder.java` — 编码器基类
- `codec/src/main/java/io/netty/handler/codec/LengthFieldBasedFrameDecoder.java` — 最通用的帧解码器
- `codec/src/main/java/io/netty/handler/codec/LineBasedFrameDecoder.java`
- `codec/src/main/java/io/netty/handler/codec/DelimiterBasedFrameDecoder.java`

**关键问题：**
1. `ByteToMessageDecoder.channelRead()` 中的 cumulation（累积）逻辑是怎样的？
2. `LengthFieldBasedFrameDecoder` 的 4 个参数（lengthFieldOffset, lengthFieldLength, lengthAdjustment, initialBytesToStrip）如何组合？
3. 为什么 Decoder 是 Inbound，Encoder 是 Outbound？

**实战：** 阅读 `codec-http/` 中的 HTTP 编解码实现：
- `codec-http/src/main/java/io/netty/handler/codec/http/HttpRequestDecoder.java`
- `codec-http/src/main/java/io/netty/handler/codec/http/HttpResponseEncoder.java`
- `codec-http/src/main/java/io/netty/handler/codec/http/HttpObjectAggregator.java`

**验证方式：** 自己实现一个自定义协议的编解码器（如：4字节长度头 + body）。

---

## 阶段五：Bootstrap 启动流程（2-3天）

**目标：** 端到端追踪一个服务端从 `bind()` 到就绪接受连接的完整过程。

**入口文件：**
- `transport/src/main/java/io/netty/bootstrap/AbstractBootstrap.java`
- `transport/src/main/java/io/netty/bootstrap/ServerBootstrap.java`
- `transport/src/main/java/io/netty/bootstrap/Bootstrap.java`

**关键调用链：**
```
ServerBootstrap.bind()
  → AbstractBootstrap.doBind()
    → initAndRegister()
      → channelFactory.newChannel()          // 反射创建 NioServerSocketChannel
      → init(channel)                        // 配置 pipeline, 添加 ServerBootstrapAcceptor
      → group().register(channel)            // 注册到 EventLoop
    → doBind0()
      → channel.bind()                       // 通过 Pipeline 传播到 HeadContext → Unsafe.bind()
```

**关键问题：**
1. `ServerBootstrapAcceptor` 是什么？它如何把新连接分配给 childGroup？
2. `register` 操作具体做了什么？（注册到 Selector、触发 channelRegistered 事件）
3. 为什么 `bind` 要通过 Pipeline 传播而不是直接调 JDK？

**验证方式：** 在关键节点打断点，跟踪一次完整的 bind 流程。

---

## 阶段六：高级主题（按兴趣选学）

### 6.1 写操作与 Flush 机制
- `transport/src/main/java/io/netty/channel/ChannelOutboundBuffer.java`
- 理解 write 只是缓存到 `ChannelOutboundBuffer`，flush 才真正写入 socket
- 水位线机制（`WriteBufferWaterMark`）如何做背压

### 6.2 HTTP/2 实现
- `codec-http2/src/main/java/io/netty/handler/codec/http2/`
- 多路复用 stream、flow control、HPACK 头部压缩

### 6.3 SSL/TLS
- `handler/src/main/java/io/netty/handler/ssl/SslHandler.java`
- `handler/src/main/java/io/netty/handler/ssl/SslContext.java`
- OpenSSL vs JDK SSL 引擎的选择

### 6.4 Native Transport
- `transport-classes-epoll/` — Linux epoll
- `transport-classes-kqueue/` — macOS kqueue
- 与 NIO transport 的 API 差异和性能差异

### 6.5 内存泄漏检测
- `buffer/src/main/java/io/netty/buffer/AbstractByteBufAllocator.java`
- `common/src/main/java/io/netty/util/ResourceLeakDetector.java`
- 四个检测级别：DISABLED, SIMPLE, ADVANCED, PARANOID

### 6.6 HashedWheelTimer
- `common/src/main/java/io/netty/util/HashedWheelTimer.java`
- 时间轮算法在超时检测、心跳等场景的应用

---

## 项目模块速查

| 模块 | 说明 |
|------|------|
| `common` | 工具类、并发原语、`Future/Promise`、`Timer`、内部工具 |
| `buffer` | `ByteBuf` 体系、内存分配器、池化管理 |
| `transport` | `Channel`、`EventLoop`、`Pipeline`、`Bootstrap` 核心框架 |
| `codec` | 编解码器基类、通用帧解码器 |
| `codec-http` | HTTP 1.x 编解码、multipart |
| `codec-http2` | HTTP/2 协议实现 |
| `codec-mqtt` | MQTT 协议编解码 |
| `codec-dns` | DNS 协议编解码 |
| `handler` | SSL、流控、日志、IP 过滤等通用 Handler |
| `handler-proxy` | HTTP/SOCKS 代理支持 |
| `resolver-dns` | 异步 DNS 解析器 |
| `transport-classes-epoll` | Linux epoll 原生传输 |
| `transport-classes-kqueue` | macOS kqueue 原生传输 |
| `example` | 各种协议的示例代码 |

## 学习约定

- 在源码中用 `// [LEARN]` 标记自己的学习笔记，方便日后 `grep -r "\[LEARN\]"` 汇总
- 每完成一个阶段，在 `example/` 下写一个自己的练习代码验证理解
- 遇到不理解的设计，先提问"这里为什么这样做"，再看代码找答案
- 优先阅读测试代码理解行为，测试路径与源码路径镜像（`src/main` → `src/test`）

## 构建与运行

```bash
# 编译（跳过测试加速）
mvn clean install -DskipTests -pl example -am

# 运行 Echo 示例
mvn -pl example exec:java -Dexec.mainClass="io.netty.example.echo.EchoServer"
mvn -pl example exec:java -Dexec.mainClass="io.netty.example.echo.EchoClient"
```
