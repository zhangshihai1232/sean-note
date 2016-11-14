## 一.  SSL/TLS 
Java提供了`javax.net.ssl`的类SslContext 和SslEngine 可以实现加密解密；
netty用SslHandler实现，内部持有一个SslEngine做实际的工作
SslHandler 数据流图
![](http://waylau.com/essential-netty-in-action/images/Figure%208.1%20Data%20flow%20through%20SslHandler%20for%20decryption%20and%20encryption.jpg)
1. 加密的入站数据被 SslHandler 拦截，并被解密
2. 前面加密的数据被 SslHandler 解密
3. 平常数据传过 SslHandler
4. SslHandler 加密数据并它传递出站

SslHandler 使用 ChannelInitializer 添加到 ChannelPipeline

```java
public class SslChannelInitializer extends ChannelInitializer<Channel> {

    private final SslContext context;
    private final boolean startTls;
    public SslChannelInitializer(SslContext context,
    boolean client, boolean startTls) {   //1
        this.context = context;
        this.startTls = startTls;
    }
    @Override
    protected void initChannel(Channel ch) throws Exception {
        SSLEngine engine = context.newEngine(ch.alloc());  //2
        engine.setUseClientMode(client); //3
        ch.pipeline().addFirst("ssl", new SslHandler(engine, startTls));  //4
    }
}
```
- 使用构造函数来传递 SSLContext 用于使用(startTls 是否启用)
- 从 SslContext 获得一个新的 SslEngine 。给每个 SslHandler 实例使用一个新的 SslEngine
- 设置 SslEngine 是 client 或者是 server 模式
- 添加 SslHandler 到 pipeline 作为第一个处理器

SslHandler作为第一个ChannelHandler，在进入前解密，输出前加密；
SslHandler有很多加密方法，例如在握手阶段两端相互验证,商定一个加密方法；
您可以配置 SslHandler 修改其行为或提供 在SSL/TLS 握手完成后发送通知,这样所有数据都将被加密。 SSL/TLS 握手将自动执行。

```bash
setHandshakeTimeout(...) 
setHandshakeTimeoutMillis(...)      设置/获取超时，超时后握手ChannelFuture被通知失败
getHandshakeTimeoutMillis()
setCloseNotifyTimeout(...) 
setCloseNotifyTimeoutMillis(...)    失败后关闭连接
getCloseNotifyTimeoutMillis()
handshakeFuture()                   握手完成返回ChannelFuture
close(...)   
```

## 二. Netty HTTP
**Decoder, Encoder 和 Codec**
netty提供了简单的编码、解码器简化http协议的开发工作

HTTP Request
![](http://waylau.com/essential-netty-in-action/images/Figure%208.2%20HTTP%20request%20component%20parts.jpg)

- HTTP Request 第一部分是包含的头信息
- HttpContent 里面包含的是数据，可以后续有多个 HttpContent 部分
- LastHttpContent 标记是 HTTP request 的结束，同时可能包含头的尾部信息
- 完整的 HTTP request

HTTP response
![](http://waylau.com/essential-netty-in-action/images/Figure%208.3%20HTTP%20response%20component%20parts.jpg)
- HTTP response 第一部分是包含的头信息
- HttpContent 里面包含的是数据，可以后续有多个 HttpContent 部分
- LastHttpContent 标记是 HTTP response 的结束，同时可能包含头的尾部信息
- 完整的 HTTP response

FullHttpRequest和FullHttpResponse是比较特殊的子类型，所有的http消息都实现自HttpObject接口；

http编码器和解码器

```bash
HttpRequestEncoder      编码HttpRequest,HttpContent,LastHttpContent消息到bytes
HttpResponseEncoder     编码HttpResponse,HttpContent,LastHttpContent消息到bytes
HttpRequestDecoder      译码bytes到HttpRequest,HttpContent,LastHttpContent
HttpResponseDecoder     译码bytes到HttpResponse,HttpContent,LastHttpContent
```
只需要添加正确的ChannelHandler到ChannelPipeline中

```java
public class HttpPipelineInitializer extends ChannelInitializer<Channel> {

    private final boolean client;
    public HttpPipelineInitializer(boolean client) {
        this.client = client;
    }

    @Override
    protected void initChannel(Channel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        if (client) {
            pipeline.addLast("decoder", new HttpResponseDecoder());  //1
            pipeline.addLast("encoder", new HttpRequestEncoder());  //2
        } else {
            pipeline.addLast("decoder", new HttpRequestDecoder());  //3
            pipeline.addLast("encoder", new HttpResponseEncoder());  //4
        }
    }
}
```
1. client: 添加 HttpResponseDecoder 用于处理来自 server 响应
2. client: 添加 HttpRequestEncoder 用于发送请求到 server
3. server: 添加 HttpRequestDecoder 用于接收来自 client 的请求
4. server: 添加 HttpResponseEncoder 用来发送响应给 client

**消息聚合**
HTTP请求和响应可以由许多部分组成，需要提供一个聚合器合并消息到FullHttpRequest 和 FullHttpResponse，这样总是能看到完整的消息；
这样的消息会缓冲，知道完整后，才发送到下一个ChannelInboundHandler管道,但是不必担心碎片；

```java
public class HttpAggregatorInitializer extends ChannelInitializer<Channel> {
    private final boolean client;
    public HttpAggregatorInitializer(boolean client) {
        this.client = client;
    }

    @Override
    protected void initChannel(Channel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        if (client) {
            pipeline.addLast("codec", new HttpClientCodec());  //1
        } else {
            pipeline.addLast("codec", new HttpServerCodec());  //2
        }
        pipeline.addLast("aggegator", new HttpObjectAggregator(512 * 1024));  //3
    }
}
```
1. client: 添加 HttpClientCodec
2. server: 添加 HttpServerCodec 作为我们是 server 模式时
3. 添加 HttpObjectAggregator 到 ChannelPipeline, 使用最大消息值是 512kb

**HTTP 压缩**
使用 HTTP 时建议压缩数据以减少传输流量
Netty 支持“gzip”和“deflate”
提供了两个ChannelHandler用于压缩和解压

客户端显示支持加密模式

```java
GET /encrypted-area HTTP/1.1
Host: www.example.com
Accept-Encoding: gzip, deflate
```
服务端需要压缩

```java
public class HttpAggregatorInitializer extends ChannelInitializer<Channel> {

    private final boolean isClient;
    public HttpAggregatorInitializer(boolean isClient) {
        this.isClient = isClient;
    }
    @Override
    protected void initChannel(Channel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        if (isClient) {
            pipeline.addLast("codec", new HttpClientCodec()); //1
            pipeline.addLast("decompressor",new HttpContentDecompressor()); //2
        } else {
            pipeline.addLast("codec", new HttpServerCodec()); //3
            pipeline.addLast("compressor",new HttpContentCompressor()); //4
        }
    }
}
```
1. client: 添加 HttpClientCodec
2. client: 添加 HttpContentDecompressor 用于处理来自服务器的压缩的内容
3. server: HttpServerCodec
4. server: HttpContentCompressor 用于压缩来自 client 支持的 HttpContentCompressor

## 三. HTTPS
启用 HTTPS，只需添加 SslHandler

```java
public class HttpsCodecInitializer extends ChannelInitializer<Channel> {
    private final SslContext context;
    private final boolean client;
    public HttpsCodecInitializer(SslContext context, boolean client) {
        this.context = context;
        this.client = client;
    }

    @Override
    protected void initChannel(Channel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        SSLEngine engine = context.newEngine(ch.alloc());
        pipeline.addFirst("ssl", new SslHandler(engine));  //1

        if (client) {
            pipeline.addLast("codec", new HttpClientCodec());  //2
        } else {
            pipeline.addLast("codec", new HttpServerCodec());  //3
        }
    }
}
```
1. 添加 SslHandler 到 pipeline 来启用 HTTPS
2. client: 添加 HttpClientCodec
3. server: 添加 HttpServerCodec ，如果是 server 模式的话

## 四. WebSocket
允许双向传输，支持文本和二进制，提供了TCP双向的连接
开始于普通 HTTP ，并“升级”为双向 WebSocket；
- Client (HTTP) 与 Server 通讯
- Server (HTTP) 与 Client 通讯
- Client 通过 HTTP(s) 来进行 WebSocket 握手,并等待确认
- 连接协议升级至 WebSocket

需要添加服务端或者客户端的WebSocket ChannelHandler到pipeline
这个Handler会处理WebSocket定义的消息类型，称为帧
http://waylau.com/essential-netty-in-action/iamges/Figure%208.4%20WebSocket%20protocol.jpg

```bash
BinaryWebSocketFrame            数据帧：二进制数据帧
TextWebSocketFrame              数据帧：文本
ContinuationWebSocketFrame      数据帧：文本或在数据，属于上一帧
CloseWebSocketFrame             控制帧：关闭
PingWebSocketFrame              控制帧：请求
PongWebSocketFrame              控制帧：响应应
```
WebSocketServerProtocolHandler在服务器端建立
该类处理协议升级握手以及三个“控制”帧 Close, Ping 和 Pong。
Text 和 Binary 数据帧将被传递到下一个处理程序(由你实现)进行处理。

```java
public class WebSocketServerInitializer extends ChannelInitializer<Channel> {
    @Override
    protected void initChannel(Channel ch) throws Exception {
        ch.pipeline().addLast(
                new HttpServerCodec(),
                new HttpObjectAggregator(65536),  //1
                new WebSocketServerProtocolHandler("/websocket"),  //2
                new TextFrameHandler(),  //3
                new BinaryFrameHandler(),  //4
                new ContinuationFrameHandler());  //5
    }

    public static final class TextFrameHandler extends SimpleChannelInboundHandler<TextWebSocketFrame> {
        @Override
        public void channelRead0(ChannelHandlerContext ctx, TextWebSocketFrame msg) throws Exception {
            // Handle text frame
        }
    }

    public static final class BinaryFrameHandler extends SimpleChannelInboundHandler<BinaryWebSocketFrame> {
        @Override
        public void channelRead0(ChannelHandlerContext ctx, BinaryWebSocketFrame msg) throws Exception {
            // Handle binary frame
        }
    }

    public static final class ContinuationFrameHandler extends SimpleChannelInboundHandler<ContinuationWebSocketFrame> {
        @Override
        public void channelRead0(ChannelHandlerContext ctx, ContinuationWebSocketFrame msg) throws Exception {
            // Handle continuation frame
        }
    }
}
```
1. 添加 HttpObjectAggregator 用于提供在握手时聚合 HttpRequest
2. 添加 WebSocketServerProtocolHandler 用于处理色好给你寄握手如果请求是发送到"/websocket." 端点，当升级完成后，它将会处理Ping, Pong 和 Close 帧
3. TextFrameHandler 将会处理 TextWebSocketFrames
4. BinaryFrameHandler 将会处理 BinaryWebSocketFrames
5. ContinuationFrameHandler 将会处理ContinuationWebSocketFrames

netty in action 11章

## 五. SPDY
Google开发的基于，降低延迟，不是替代http，是对http的增强；
- 压缩报头
- 加密所有
- 多路复用连接
- 提供支持不同的传输优先级

netty in action 12章

## 六. 空闲连接以及超时
为了及时释放资源
常见的方法是发送心跳，暴力的方式是直接断开

```bash
IdleStateHandler    时间过长触发IdleStateEvent,可覆盖userEventTriggered来处理IdleStateEvent
ReadTimeoutHandler  时间内没收到数据，抛ReadTimeoutException并关闭channel,可以覆盖ChannelHandler中的exceptionCaught捕获
WriteTimeoutHandler ChannelHandler中的exceptionCaught捕获
```
利用IdleStateHandler发送心跳例子

```java
public class IdleStateHandlerInitializer extends ChannelInitializer<Channel> {

    @Override
    protected void initChannel(Channel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        pipeline.addLast(new IdleStateHandler(0, 0, 60, TimeUnit.SECONDS));  //1
        pipeline.addLast(new HeartbeatHandler());
    }

    public static final class HeartbeatHandler extends ChannelInboundHandlerAdapter {
        private static final ByteBuf HEARTBEAT_SEQUENCE = Unpooled.unreleasableBuffer(
                Unpooled.copiedBuffer("HEARTBEAT", CharsetUtil.ISO_8859_1));  //2

    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
        if (evt instanceof IdleStateEvent) {
             ctx.writeAndFlush(HEARTBEAT_SEQUENCE.duplicate())
                     .addListener(ChannelFutureListener.CLOSE_ON_FAILURE);  //3
        } else {
            super.userEventTriggered(ctx, evt);  //4
        }
    }
}
```
- IdleStateHandler 将通过 IdleStateEvent 调用 userEventTriggered ，如果连接没有接收或发送数据超过60秒钟
- 心跳发送到远端
- 发送的心跳并添加一个侦听器，如果发送操作失败将关闭连接
- 事件不是一个 IdleStateEvent 的话，就将它传递给下一个处理程序

## 七. 分隔符协议
例如SMTP、POP3、IMAP、Telnet等等

```bash
DelimiterBasedFrameDecoder	接收ByteBuf由一个或多个分隔符拆分，如NUL或换行符
LineBasedFrameDecoder		接收ByteBuf以分割线结束，如"\n"和"\r\n"
```
使用"\r\n"分隔符的处理
![](http://waylau.com/essential-netty-in-action/images/Figure%208.5%20Handling%20delimited%20frames.jpg)
1. 字节流
2. 第一帧
3. 第二帧

LineBasedFrameDecoder例子

```java
public class LineBasedHandlerInitializer extends ChannelInitializer<Channel> {
    @Override
    protected void initChannel(Channel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        pipeline.addLast(new LineBasedFrameDecoder(65 * 1024));   //1
        pipeline.addLast(new FrameHandler());  //2
    }

    public static final class FrameHandler extends SimpleChannelInboundHandler<ByteBuf> {
        @Override
        public void channelRead0(ChannelHandlerContext ctx, ByteBuf msg) throws Exception {  //3
            // Do something with the frame
        }
    }
}
```
1. 添加一个 LineBasedFrameDecoder 用于提取帧并把数据包转发到下一个管道中的处理程序,在这种情况下就是 FrameHandler
2. 添加 FrameHandler 用于接收帧
3. 每次调用都需要传递一个单帧的内容

使用DelimiterBasedFrameDecoder可以方便处理特定分隔符作为数据结构体的情况
1. 传入的数据流是一系列的帧，每个由换行（“\n”）分隔
2. 每帧包括一系列项目，每个由单个空格字符分隔
3. 一帧的内容代表一个“命令”：一个名字后跟一些变量参数

定义类：
- 类 Cmd 存储帧的内容，其中一个 ByteBuf 用于存名字，另外一个存参数
- 类 CmdDecoder 从重写方法 decode() 中检索一行，并从其内容中构建一个 Cmd 的实例
- 类 CmdHandler 从 CmdDecoder 接收解码 Cmd 对象和对它的一些处理

```java
/*
	1. 添加一个 CmdDecoder 到管道；将提取 Cmd 对象和转发到在管道中的下一个处理器
	2. 添加 CmdHandler 将接收和处理 Cmd 对象
	3. 命令也是 POJO
	4. super.decode() 通过结束分隔从 ByteBuf 提取帧
	5. frame 是空时，则返回 null
	6. 找到第一个空字符的索引。首先是它的命令名；接下来是参数的顺序
	7. 从帧先于索引以及它之后的片段中实例化一个新的 Cmd 对象
	8. 处理通过管道的 Cmd 对象
*/
public class CmdHandlerInitializer extends ChannelInitializer<Channel> {

    @Override
    protected void initChannel(Channel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        pipeline.addLast(new CmdDecoder(65 * 1024));//1
        pipeline.addLast(new CmdHandler()); //2
    }

    public static final class Cmd { //3
        private final ByteBuf name;
        private final ByteBuf args;

        public Cmd(ByteBuf name, ByteBuf args) {
            this.name = name;
            this.args = args;
        }

        public ByteBuf name() {
            return name;
        }

        public ByteBuf args() {
            return args;
        }
    }

    public static final class CmdDecoder extends LineBasedFrameDecoder {
        public CmdDecoder(int maxLength) {
            super(maxLength);
        }

        @Override
        protected Object decode(ChannelHandlerContext ctx, ByteBuf buffer) throws Exception {
            ByteBuf frame =  (ByteBuf) super.decode(ctx, buffer); //4
            if (frame == null) {
                return null; //5
            }
            int index = frame.indexOf(frame.readerIndex(), frame.writerIndex(), (byte) ' ');  //6
            return new Cmd(frame.slice(frame.readerIndex(), index), frame.slice(index +1, frame.writerIndex())); //7
        }
    }

    public static final class CmdHandler extends SimpleChannelInboundHandler<Cmd> {
        @Override
        public void channelRead0(ChannelHandlerContext ctx, Cmd msg) throws Exception {
            // Do something with the command  //8
        }
    }
}
```

## 八. 基于长度的协议
帧头中定义了了帧编码的长度，提供两个解码器，用于处理

```bash
FixedLengthFrameDecoder			提取固定长度
LengthFieldBasedFrameDecoder	读取头部长度并提取帧的长度
```

FixedLengthFrameDecoder 的操作是提取固定长度每帧8字节
![](http://waylau.com/essential-netty-in-action/images/Figure%208.5%20Handling%20delimited%20frames.jpg)\
1. 字节流 stream
2. 4个帧，每个帧8个字节

大部分情况，帧大小写在编码头部，使用LengthFieldBasedFrameDecoder，读取头部长度，提取帧长度

![](http://waylau.com/essential-netty-in-action/images/Figure%208.7%20Message%20that%20has%20frame%20size%20encoded%20in%20the%20header.jpg)
LengthFieldBasedFrameDecoder 提供了几个构造函数覆盖各种各样的头长字段配置情况
使用三个构造函数：maxFrameLength，lengthFieldOffset ，lengthFieldLength

```java
public class LineBasedHandlerInitializer extends ChannelInitializer<Channel> {
    @Override
    protected void initChannel(Channel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        pipeline.addLast(new LineBasedFrameDecoder(65 * 1024));  //1
        pipeline.addLast(new FrameHandler()); //2
    }
    public static final class FrameHandler extends SimpleChannelInboundHandler<ByteBuf> {
        @Override
        public void channelRead0(ChannelHandlerContext ctx, ByteBuf msg) throws Exception {
            // Do something with the frame  //3
        }
    }
}
```
1. 添加一个 LengthFieldBasedFrameDecoder ,用于提取基于帧编码长度8个字节的帧。
2. 添加一个 FrameHandler 用来处理每帧
3. 处理帧数据

## 九. 编写大型数据
写大数据时，通知ChannelFuture就返回，但是内存中仍然在接收数据，如果这种连接过多，会产生内存耗尽的风险；
使用zero-copy技术，不占用内存；
`interface FileRegion`支持通过Channel实现zero-copy
通过zero-copy从FileInputStream创建DefaultFileRegion并写入

```java
FileInputStream in = new FileInputStream(file); //1
FileRegion region = new DefaultFileRegion(in.getChannel(), 0, file.length()); //2

channel.writeAndFlush(region).addListener(new ChannelFutureListener() { //3
    @Override
    public void operationComplete(ChannelFuture future) throws Exception {
        if (!future.isSuccess()) {
            Throwable cause = future.cause(); //4
            // Do something
        }
    }
});
```
- 获取 FileInputStream
- 创建一个新的 DefaultFileRegion 用于文件的完整长度
- 发送 DefaultFileRegion 并且注册一个 ChannelFutureListener
- 处理发送失败
这种方式只能做文件传输

需要写入内存处理使用ChunkedWriteHandler，支持异步大数据流不引起高内存消耗

```bash
ChunkedFile			当你使用平台不支持 zero-copy 或者你需要转换数据，从文件中一块一块的获取数据
ChunkedNioFile		与 ChunkedFile 类似，处理使用了NIOFileChannel
ChunkedStream		从 InputStream 中一块一块的转移内容
ChunkedNioStream	从 ReadableByteChannel 中一块一块的转移内容
```
从文件系统赋值到用户内存需要使用ChunkedWriteHandler

```bash
ChunkedFile			当你使用平台不支持 zero-copy 或者你需要转换数据，从文件中一块一块的获取数据
ChunkedNioFile		与 ChunkedFile 类似，处理使用了NIOFileChannel
ChunkedStream		从 InputStream 中一块一块的转移内容
ChunkedNioStream	从 ReadableByteChannel 中一块一块的转移内容
```

WriteStreamHandler 从文件一块一块的写入数据作为ChunkedStream,然后通过SslHandler传播的例子

```java
public class ChunkedWriteHandlerInitializer extends ChannelInitializer<Channel> {
    private final File file;
    private final SslContext sslCtx;

    public ChunkedWriteHandlerInitializer(File file, SslContext sslCtx) {
        this.file = file;
        this.sslCtx = sslCtx;
    }

    @Override
    protected void initChannel(Channel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        pipeline.addLast(new SslHandler(sslCtx.createEngine()); //1
        pipeline.addLast(new ChunkedWriteHandler());//2
        pipeline.addLast(new WriteStreamHandler());//3
    }

    public final class WriteStreamHandler extends ChannelInboundHandlerAdapter {  //4

        @Override
        public void channelActive(ChannelHandlerContext ctx) throws Exception {
            super.channelActive(ctx);
            ctx.writeAndFlush(new ChunkedStream(new FileInputStream(file)));
        }
    }
}
```
- 添加 SslHandler 到 ChannelPipeline.
- 添加 ChunkedWriteHandler 用来处理作为 ChunkedInput 传进的数据
- 当连接建立时，WriteStreamHandler 开始写文件的内容
- 当连接建立时，channelActive() 触发使用 ChunkedInput 来写文件的内容 (插图显示了 FileInputStream;也可以使用任何 InputStream )

需要用户实现ChunkedInput，安装ChunkedWriteHandler

## 十. 序列化数据

**JDK 序列化**
不需要外部依赖
jdk通过ObjectOutputStream和ObjectInputStream通过原始数据类型和POJO进行序列化和反序列化

```bash
CompatibleObjectDecoder		该解码器使用 JDK 序列化，用于与非 Netty 进行互操作。
CompatibleObjectEncoder		该编码器使用 JDK 序列化，用于与非 Netty 进行互操作。
ObjectDecoder				基于 JDK 序列化来使用自定义序列化解码。外部依赖被排除在外时，提供了一个速度提升。否则选择其他序列化实现
ObjectEncoder				基于 JDK 序列化来使用自定义序列化编码。外部依赖被排除在外时，提供了一个速度提升。否则选择其他序列化实现
```
**JBoss Marshalling 序列化**
需要外部依赖可以使用 JBoss Marshalling，速度快3倍
修复bug与` java.io.Serializable `完全兼容

```bash
CompatibleMarshallingDecoder	为了与使用 JDK 序列化的端对端间兼容。
CompatibleMarshallingEncoder	为了与使用 JDK 序列化的端对端间兼容。
MarshallingDecoder				使用自定义序列化用于解码，必须使用
MarshallingEncoder 				使用自定义序列化用于编码
```
使用MarshallingDecoder和MarshallingEncoder例子

```java
public class MarshallingInitializer extends ChannelInitializer<Channel> {
    private final MarshallerProvider marshallerProvider;
    private final UnmarshallerProvider unmarshallerProvider;

    public MarshallingInitializer(UnmarshallerProvider unmarshallerProvider,
                                  MarshallerProvider marshallerProvider) {
        this.marshallerProvider = marshallerProvider;
        this.unmarshallerProvider = unmarshallerProvider;
    }
    @Override
    protected void initChannel(Channel channel) throws Exception {
        ChannelPipeline pipeline = channel.pipeline();
        pipeline.addLast(new MarshallingDecoder(unmarshallerProvider));
        pipeline.addLast(new MarshallingEncoder(marshallerProvider));
        pipeline.addLast(new ObjectHandler());
    }

    public static final class ObjectHandler extends SimpleChannelInboundHandler<Serializable> {
        @Override
        public void channelRead0(ChannelHandlerContext channelHandlerContext, Serializable serializable) throws Exception {
            // Do something
        }
    }
}
```
**ProtoBuf 序列化**
速度快，适合跨语言项目；

```bash
ProtobufDecoder					使用 ProtoBuf 来解码消息
ProtobufEncoder					使用 ProtoBuf 来编码消息
ProtobufVarint32FrameDecoder	在消息的整型长度域中，通过 "Base 128 Varints"将接收到的 ByteBuf 动态的分割
```
用法

```java
public class ProtoBufInitializer extends ChannelInitializer<Channel> {

    private final MessageLite lite;

    public ProtoBufInitializer(MessageLite lite) {
        this.lite = lite;
    }

    @Override
    protected void initChannel(Channel ch) throws Exception {
        ChannelPipeline pipeline = ch.pipeline();
        pipeline.addLast(new ProtobufVarint32FrameDecoder());
        pipeline.addLast(new ProtobufEncoder());
        pipeline.addLast(new ProtobufDecoder(lite));
        pipeline.addLast(new ObjectHandler());
    }

    public static final class ObjectHandler extends SimpleChannelInboundHandler<Object> {
        @Override
        public void channelRead0(ChannelHandlerContext ctx, Object msg) throws Exception {
            // Do something with the object
        }
    }
}
```
1. 添加 ProtobufVarint32FrameDecoder 用来分割帧
2. 添加 ProtobufEncoder 用来处理消息的编码
3. 添加 ProtobufDecoder 用来处理消息的解码
4. 添加 ObjectHandler 用来处理解码了的消息
