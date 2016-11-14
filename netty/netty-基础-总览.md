## 一. Netty-异步和数据驱动
网络客户端/服务器框架
下面展示了 Netty 技术和方法的特点

**设计**
- 针对多种传输类型的统一接口 阻塞和非阻塞
- 简单但更强大的线程模型
- 真正的无连接的数据报套接字支持
- 链接逻辑支持复用
**易用性**
- 大量的 Javadoc 和 代码实例
- 除了在 JDK 1.6 + 额外的限制。（一些特征是只支持在Java 1.7 +。可选的功能可能有额外的限制。）
**性能**
- 比核心 Java API 更好的吞吐量，较低的延时
- 资源消耗更少，这个得益于共享池和重用
- 减少内存拷贝
**健壮性**
- 消除由于慢，快，或重载连接产生的 OutOfMemoryError
- 消除经常发现在 NIO 在高速网络中的应用中的不公平的读/写比
**安全**
- 完整的 SSL / TLS 和 StartTLS 的支持
- 运行在受限的环境例如 Applet 或 OSGI
**社区**
- 发布的更早和更频繁
- 社区驱动

## 二. 构成部分
**Future**
能够执行IO操作的连接
**Callback**
一旦回调被触发，马上会被调用，比如channelActive()，channel创建的时候会调用

```java
public class ConnectHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelActive(ChannelHandlerContext ctx) throws Exception {   //1
        System.out.println(
                "Client " + ctx.channel().remoteAddress() + " connected");
    }
}
```
**Future**
JDK的future需要手动检查是否完成阻塞
netty提供ChannelFuture可以不手动检查就能完成操作
通过注册ChannelFutureListener，回调方法operationComplete会在执行完成时调用，可以针对错误或成功执行不同的操作
outbound I/O 操作都会返回一个 ChannelFuture，这样就不会阻塞

```java
Channel channel = ...;
//不会阻塞
ChannelFuture future = channel.connect(
    new InetSocketAddress("192.168.0.1", 25)); 
```
注册一个ChannelFutureListener到这个ChannelFuture上

```java
Channel channel = ...;
//不会阻塞
ChannelFuture future = channel.connect(            //1
        new InetSocketAddress("192.168.0.1", 25));
future.addListener(new ChannelFutureListener() {  //2
@Override
public void operationComplete(ChannelFuture future) {
    if (future.isSuccess()) {                    //3
        ByteBuf buffer = Unpooled.copiedBuffer(
                "Hello", Charset.defaultCharset()); //4
        ChannelFuture wf = future.channel().writeAndFlush(buffer);                //5
        // ...
    } else {
        Throwable cause = future.cause();        //6
        cause.printStackTrace();
    }
}
});
```
过程
1. 异步连接到远程对等。调用立即返回并提供 ChannelFuture。
2. 操作完成后通知注册一个 ChannelFutureListener 。
3. 当 operationComplete() 调用时检查操作的状态。
4. 如果成功就创建一个 ByteBuf 来保存数据。
5. 异步发送数据到远程。再次返回ChannelFuture。
6. 如果有一个错误则抛出 Throwable,描述错误原因。

**Event 和 Handler**
Netty使用不同的事件通知更改的状态或操作的状态；
我们可以根据发生的事件触发适当的行为，包括：日志，数据转换，流控制，应用程序逻辑
netty是网络框架，事件和入站出站的数据流有关
传入数据状态变化：活动或非活动连接，数据的读取，用户事件，错误
出站事件：打开或关闭一个连接到远程，写或冲刷数据到 socket

这些事件都可以分配给用户，处理逻辑；用事件处理器处理；
![](http://waylau.com/essential-netty-in-action/images/Figure%201.3%20Event%20Flow.jpg)
Netty提供了一组丰富的预定义的处理程序，各种协议的编解码器，HTTP,SSL/TLS等

**FUTURE, CALLBACK 和 HANDLER**
异步模型构建在future和callback概念上
拦截操作、转换入站、出站只需提供回调或利用future返回
目标：你的业务逻辑从网络基础设施应用程序中分离

**SELECTOR, EVENT 和 EVENT LOOP**
Netty 通过触发事件从应用程序中抽象出 Selector，避免了手写调度代码；
EventLoop 分配给每个 Channel 来处理所有的事件，包括

## 三.  echo 服务器
包括：一个handler处理逻辑，一个Bootstrapping启动代码
实现ChannelInboundHandler用来定义处理入站事件的方法，应用比较简单，继承ChannelInboundHandlerAdapter就可以，它是一个默认实现，只需要实现三个方法；
- channelRead() 每个信息入站都会调用
- channelReadComplete() 通知处理器最后的 channelread() 是当前批处理中的最后一条消息时调用
- exceptionCaught()读操作时捕获到异常时调用

EchoServerHandler代码

```java
//实例可以在channel中共享
@Sharable
public class EchoServerHandler extends
        ChannelInboundHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx,
        Object msg) {
        ByteBuf in = (ByteBuf) msg;
        System.out.println("Server received: " + in.toString(CharsetUtil.UTF_8));
        //将消息返回给发送者
		ctx.write(in);
    }

    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
		//冲刷所有待审消息到远程节点。关闭通道后，操作完成
        ctx.writeAndFlush(Unpooled.EMPTY_BUFFER)
        .addListener(ChannelFutureListener.CLOSE);
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx,
        Throwable cause) {
        cause.printStackTrace(); 
        ctx.close();
    }
}
```
关注点分离原则，覆盖在适当的点的钩子hook；
覆盖 channelRead因为我们需要处理所有接收到的数据
覆盖 exceptionCaught 使我们能够应对任何 Throwable 的子类型
每个 Channel 都有一个关联的 ChannelPipeline，它代表了 ChannelHandler 实例的链。
适配器处理的实现只是将一个处理方法调用转发到链中的下一个处理器。
如果一个 Netty 应用程序不覆盖exceptionCaught ，那么这些错误将最终到达 ChannelPipeline，并且结束警告将被记录。
你应该提供至少一个 实现 exceptionCaught 的 ChannelHandler。

**引导服务器**
引导服务器
- 监听和接收进来的连接请求
- 配置 Channel 来通知一个关于入站消息的 EchoServerHandler 实例

EchoServer代码

```java
public class EchoServer {
    private final int port;
    public EchoServer(int port) {
        this.port = port;
    }
    public static void main(String[] args) throws Exception {
    if (args.length != 1) {
        System.err.println(
                "Usage: " + EchoServer.class.getSimpleName() +
                " <port>");
        return;
    }
   	 	int port = Integer.parseInt(args[0]);        //1
    	new EchoServer(port).start();                //2
    }

    public void start() throws Exception {
        NioEventLoopGroup group = new NioEventLoopGroup(); //3
        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(group)                                //4
             .channel(NioServerSocketChannel.class)        //5
             .localAddress(new InetSocketAddress(port))    //6
             .childHandler(new ChannelInitializer<SocketChannel>() { //7
                 @Override
                 public void initChannel(SocketChannel ch) 
                     throws Exception {
                     ch.pipeline().addLast(
                             new EchoServerHandler());
                 }
             });

            ChannelFuture f = b.bind().sync();            //8
            System.out.println(EchoServer.class.getName() + " started and listen on " + f.channel().localAddress());
            f.channel().closeFuture().sync();            //9
        } finally {
            group.shutdownGracefully().sync();            //10
        }
    }
}
```
特殊的类，ChannelInitializer 。当一个新的连接被接受，一个新的子 Channel 将被创建， ChannelInitializer 会添加我们EchoServerHandler 的实例到 Channel 的 ChannelPipeline。正如我们如前所述，这个处理器将被通知如果有入站信息。

## 四.  echo 客户端
ChannelInboundHandler处理数据，SimpleChannelInboundHandler处理任务，覆盖三个方法
- channelActive() 服务器的连接被建立后调用
- channelRead0() 数据后从服务器接收到调用
- exceptionCaught() 捕获一个异常时调用

```java
@Sharable                                //1
public class EchoClientHandler extends
        SimpleChannelInboundHandler<ByteBuf> {

    @Override
    public void channelActive(ChannelHandlerContext ctx) {
        ctx.writeAndFlush(Unpooled.copiedBuffer("Netty rocks!", //2
        CharsetUtil.UTF_8));
    }

    @Override
    public void channelRead0(ChannelHandlerContext ctx,
        ByteBuf in) {
        System.out.println("Client received: " + in.toString(CharsetUtil.UTF_8));    //3
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx,
        Throwable cause) {                    //4
        cause.printStackTrace();
        ctx.close();
    }
}
```
建立连接后该 channelActive() 方法被调用一次
channelRead0()接收到数据时被调用,该字节将按照它们发送的顺序分别被接收,但是接收的次数不一定；
exceptionCaught记录 Throwable 并且关闭通道，在这种情况下终止 连接到服务器。

**SimpleChannelInboundHandler和ChannelInboundHandler**
在于释放ByteBuf和数据写会，具体在哪还不清楚；

**引导客户端**

```java
public class EchoClient {
    private final String host;
    private final int port;

    public EchoClient(String host, int port) {
        this.host = host;
        this.port = port;
    }

    public void start() throws Exception {
        EventLoopGroup group = new NioEventLoopGroup();
        try {
            Bootstrap b = new Bootstrap();                //1
            b.group(group)                                //2
             .channel(NioSocketChannel.class)            //3
             .remoteAddress(new InetSocketAddress(host, port))    //4
             .handler(new ChannelInitializer<SocketChannel>() {    //5
                 @Override
                 public void initChannel(SocketChannel ch) 
                     throws Exception {
                     ch.pipeline().addLast(
                             new EchoClientHandler());
                 }
             });

            ChannelFuture f = b.connect().sync();        //6

            f.channel().closeFuture().sync();            //7
        } finally {
            group.shutdownGracefully().sync();            //8
        }
    }

    public static void main(String[] args) throws Exception {
        if (args.length != 2) {
            System.err.println(
                    "Usage: " + EchoClient.class.getSimpleName() +
                    " <host> <port>");
            return;
        }

        final String host = args[0];
        final int port = Integer.parseInt(args[1]);

        new EchoClient(host, port).start();
    }
}
```

##  五. 基本构建模块
**BOOTSTRAP**
从设置bootstrap开始，该类提供了一个用于应用程序网络层配置的容器；
**CHANNEL**
Netty 中的接口 Channel 定义了与 socket 丰富交互的操作集：bind, close, config, connect, isActive, isOpen, isWritable, read, write 等等；
Netty 提供大量的 Channel 实现来专门使用。这些包括 AbstractChannel，AbstractNioByteChannel，AbstractNioChannel，EmbeddedChannel， LocalServerChannel，NioSocketChannel 等等。
**CHANNELHANDLER**
ChannelHandler提供处理数据的容器，由特定的事件触发； ChannelHandler 可专用于几乎所有的动作，包括将一个对象转为字节（或相反），执行过程中抛出的异常处理。
常用ChannelInboundHandler，类型接收到入站事件（包括接收到的数据）可以处理应用程序逻辑；
业务逻辑经常存活于一个或者多个 ChannelInboundHandler。
需要提供响应时，你也可以从 ChannelInboundHandler 冲刷数据；

**CHANNELPIPELINE**
ChannelPipeline提供容器给ChannelHandler链，提供API管理
每个Channel有ChannelPipeline；
ChannelHandler实现ChannelInitializer，ChannelInitializer子类通过ServerBootstrap注册，调用到initChannel方法时，会将自己定义的ChannelHandler安装到pipeline，完成之后ChannelInitializer子类从ChannelPipeline删除自身；

**EVENTLOOP**
EventLoop处理多个channel
EventLoopGroup包含多个EventLoop，提供迭代方式检索内部的EventLoop

**CHANNELFUTURE**
ChannelFuture的addListener方法注册了一个ChannelFutureListener，操作完成时被通知

## 六. Channel, Event 和 I/O
Netty 使用Threads处理事件，多线程需要关注同步代码，会影响程序的性能；
Netty 的设计保证程序处理事件不会有同步；
不需要在 Channel 之间共享 ChannelHandler 实例；
![](http://waylau.com/essential-netty-in-action/images/Figure%203.1.jpg)
一个 EventLoopGroup 具有一个或多个 EventLoop
当创建一个 Channel，Netty 通过 一个单独的 EventLoop 实例来注册该 Channel,EventLoop就是一个线程
所有 Channel 的 I/O 始终用相同的线程来执行。

## 七. Bootstrapping
服务端使用ServerBootstrap
客户端使用Bootstrap

|分类	|Bootstrap	|ServerBootstrap|
|-|-|-|
|网络功能	|连接到远程主机和端口	|绑定本地端口|
|EventLoopGroup数量	|1	|2|

ServerBootstrap在服务器监听一个端口，轮询客户端的“Bootstrap”或DatagramChannel是否连接服务器；
通常需要调用“Bootstrap”类的connect()方法，但是也可以先调用bind()再调用connect()进行连接，之后使用的Channel包含在bind()返回的ChannelFuture中

具有两个EventLoopGroups的ServerBootstrap如下

![](http://waylau.com/essential-netty-in-action/images/Figure%203.2%20Server%20with%20two%20EventLoopGroups.jpg)
与 ServerChannel 相关 EventLoopGroup 分配一个 EventLoop 是 负责创建 Channels 用于传入的连接请求。一旦连接接受，第二个EventLoopGroup 分配一个 EventLoop 给它的 Channel

## 八.ChannelHandler和ChannelPipeline
ChannelPipeline 是 ChannelHandler 链的容器
ChannelHandler是个通用的容器
![](http://waylau.com/essential-netty-in-action/images/Figure%203.3%20ChannelHandler%20class%20hierarchy.jpg)
若数据是从用户应用程序到远程主机则是-出站(outbound)
相反若数据时从远程主机到用户应用程序则是-入站(inbound)
ChannelHandler操作数据，在引导阶段添加进ChannelPipeline，添加的数据决定处理数据的顺序 	

![](http://waylau.com/essential-netty-in-action/images/Figure%203.4%20ChannelPipeline%20with%20inbound%20and%20outbound%20ChannelHandlers.jpg)

从第一个ChannelInboundHandler传入，一直到末端，此时结束；
出站是从尾部第一个ChannelOutboundHandlers一直到头部；

**Inbound和Outbound Handler**
事件可以通过ChanneHandlerContext被转发到下个处理器中；
netty提供了ChannelOutboundHandlerAdapter和ChannelOutboundHandlerAdapter两个抽象的基类；
可以通过ChannelHandlerContext的方法，传递到下一个处理器中；

当 ChannelHandler 被添加到的 ChannelPipeline 它得到一个 ChannelHandlerContext，它代表一个 ChannelHandler 和 ChannelPipeline 之间的“绑定”。

Netty 发送消息有两种方式。您可以直接写消息给 Channel 或写入 ChannelHandlerContext 对象。主要的区别是， 前一种方法会导致消息从 ChannelPipeline的尾部开始，而 后者导致消息从 ChannelPipeline 下一个处理器开始；

## 九. ChannelHandler
有很多不同类型的 ChannelHandler ,每个 ChannelHandler 做什么取决于其超类;
netty提供了一些默认的adapter类，可以简化开发逻辑；
pipeline 中每个ChannelHandler 负责转发事件到下一个处理器，适配器会自动实现；
适配器可以减少手写ChannelHandlers，比如ChannelHandlerAdapter、ChannelInboundHandlerAdapter、ChannelOutboundHandlerAdapter、ChannelDuplexHandlerAdapter

ChannelHandler有三个子类型
- 编码器
- 解码器
- ChannelInboundHandlerAdapter的子类SimpleChannelInboundHandler

**编码器、解码器**
所有的编码器/解码器适配器类 都实现自 ChannelInboundHandler 或 ChannelOutboundHandler；
对于入站数据，channelRead 方法/事件被覆盖。这种方法在每个消息从入站 Channel 读入时调用。该方法将调用特定解码器的“解码”方法，并将解码后的消息转发到管道中下个的 ChannelInboundHandler。
出站消息是类似的。编码器将消息转为字节，转发到下个的 ChannelOutboundHandler。
例子：ByteToMessageDecoder，MessageToByteEncoder，ProtobufEncoder、ProtobufDecoder

**SimpleChannelHandler**
业务逻辑只需要扩展SimpleChannelInboundHandler，T是要处理的数据类型，覆盖一个或多个方法，获得ChannelHandlerContext 的引用
方法中最重要的是Read0,不要有阻塞操作

```java
channelRead0(ChannelHandlerContext，T)
```
ChannelHandler中禁止阻塞操作，可以指定一个线程池，操作这些io;






