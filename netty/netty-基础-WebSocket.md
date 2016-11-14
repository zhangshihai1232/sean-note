## 一. WebSockets
WebSockets双向，实现实时web
![](http://waylau.com/essential-netty-in-action/images/Figure%2011.1%20Application%20logic.jpg)

1. 客户端/用户连接到服务器，并且是聊天的一部分
2. 聊天消息通过 WebSocket 进行交换
3. 消息双向发送
4. 服务器处理所有的客户端/用户

客户端发送一个消息,消息被广播到所有其他客户端，聊天室的工作方式

**添加 WebSocket 支持**
升级握手机制将标准的 HTTP 或HTTPS 协议转为 WebSocket
WebSocket 的应用程序将始终以 HTTP/S 开始，然后升级
例子中：url以/ws结尾才升级为WebSocket，连接成功后传输都使用WebSocket 

![](http://waylau.com/essential-netty-in-action/images/Figure%2011.2%20Server%20logic.jpg)

1. 客户端/用户连接到服务器并加入聊天
2. HTTP 请求页面或 WebSocket 升级握手
3. 服务器处理所有客户端/用户
4. 响应 URI “/”的请求，转到 index.html
5. 如果访问的是 URI“/ws” ，处理 WebSocket 升级握手
6. 升级握手完成后 ，通过 WebSocket 发送聊天消息

**第一部分：处理 HTTP 请求**
处理FullHttpRequest消息的ChannelInboundHandler的实现类,实现忽略符合 "/ws" 格式的 URI 请求
```java
public class HttpRequestHandler extends SimpleChannelInboundHandler<FullHttpRequest> {    //1
    private final String wsUri;
    private static final File INDEX;

    static {
        URL location = HttpRequestHandler.class.getProtectionDomain().getCodeSource().getLocation();
        try {
            String path = location.toURI() + "index.html";
            path = !path.contains("file:") ? path : path.substring(5);
            INDEX = new File(path);
        } catch (URISyntaxException e) {
            throw new IllegalStateException("Unable to locate index.html", e);
        }
    }

    public HttpRequestHandler(String wsUri) {
        this.wsUri = wsUri;
    }

    @Override
    public void channelRead0(ChannelHandlerContext ctx, FullHttpRequest request) throws Exception {
        if (wsUri.equalsIgnoreCase(request.getUri())) {
            ctx.fireChannelRead(request.retain());                    //2
        } else {
            if (HttpHeaders.is100ContinueExpected(request)) {
                send100Continue(ctx);                                //3
            }

            RandomAccessFile file = new RandomAccessFile(INDEX, "r");//4

            HttpResponse response = new DefaultHttpResponse(request.getProtocolVersion(), HttpResponseStatus.OK);
            response.headers().set(HttpHeaders.Names.CONTENT_TYPE, "text/html; charset=UTF-8");

            boolean keepAlive = HttpHeaders.isKeepAlive(request);

            if (keepAlive) {                                        //5
                response.headers().set(HttpHeaders.Names.CONTENT_LENGTH, file.length());
                response.headers().set(HttpHeaders.Names.CONNECTION, HttpHeaders.Values.KEEP_ALIVE);
            }
            ctx.write(response);                    //6

            if (ctx.pipeline().get(SslHandler.class) == null) {        //7
                ctx.write(new DefaultFileRegion(file.getChannel(), 0, file.length()));
            } else {
                ctx.write(new ChunkedNioFile(file.getChannel()));
            }
            ChannelFuture future = ctx.writeAndFlush(LastHttpContent.EMPTY_LAST_CONTENT);            //8
            if (!keepAlive) {
                future.addListener(ChannelFutureListener.CLOSE);        //9
            }
        }
    }

    private static void send100Continue(ChannelHandlerContext ctx) {
        FullHttpResponse response = new DefaultFullHttpResponse(HttpVersion.HTTP_1_1, HttpResponseStatus.CONTINUE);
        ctx.writeAndFlush(response);
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause)
            throws Exception {
        cause.printStackTrace();
        ctx.close();
    }
}
```
1. 扩展 SimpleChannelInboundHandler 用于处理 FullHttpRequest信息
2. 如果请求是一次升级了的 WebSocket 请求，则递增引用计数器（retain）并且将它传递给在 ChannelPipeline 中的下个 ChannelInboundHandler
3. 处理符合 HTTP 1.1的 "100 Continue" 请求
4. 读取 index.html
5. 判断 keepalive 是否在请求头里面
6. 写 HttpResponse 到客户端
7. 写 index.html 到客户端，根据 ChannelPipeline 中是否有 SslHandler 来决定使用 DefaultFileRegion 还是 ChunkedNioFile
8. 写并刷新 LastHttpContent 到客户端，标记响应完成
9. 如果 请求头中不包含 keepalive，当写完成时，关闭 Channel

HttpRequestHandler 做了下面几件事，

- 如果该 HTTP 请求被发送到URI “/ws”，则调用 FullHttpRequest 上的 retain()，并通过调用 fireChannelRead(msg) 转发到下一个 ChannelInboundHandler。retain() 的调用是必要的，因为 channelRead() 完成后，它会调用 FullHttpRequest 上的 release() 来释放其资源。 （请参考我们先前在第6章中关于 SimpleChannelInboundHandler 的讨论）
- 如果客户端发送的 HTTP 1.1 头是“Expect: 100-continue” ，则发送“100 Continue”的响应。
- 在 头被设置后，写一个 HttpResponse 返回给客户端。注意，这不是 FullHttpResponse，这只是响应的第一部分。另外，这里我们也不使用 writeAndFlush()， 这个是在留在最后完成。
- 如果传输过程既没有要求加密也没有要求压缩，那么把 index.html 的内容存储在一个 DefaultFileRegion 里就可以达到最好的效率。这将利用零拷贝来执行传输。出于这个原因，我们要检查 ChannelPipeline 中是否有一个 SslHandler。如果是的话，我们就使用 ChunkedNioFile。
- 写 LastHttpContent 来标记响应的结束，并终止它
- 如果不要求 keepalive ，添加 ChannelFutureListener 到 ChannelFuture 对象的最后写入，并关闭连接。注意，这里我们调用 writeAndFlush() 来刷新所有以前写的信息。

WebSockets通过帧收发数据，帧代表消息的一部分，一个完整的消息可以利用多个帧

**第二部分：处理WebSocket frame**
六种不同的 frame，Netty 给每个提供了一个pojo实现
```bash
BinaryWebSocketFrame		contains binary data
TextWebSocketFrame			contains text data
ContinuationWebSocketFrame	contains text or binary data that belongs to a previous BinaryWebSocketFrame or TextWebSocketFrame
CloseWebSocketFrame			represents a CLOSE request and contains close status code and a phrase
PingWebSocketFrame			requests the transmission of a PongWebSocketFrame
PongWebSocketFrame			sent as a response to a PingWebSocketFrame
```
这里只使用了四种帧类型
- CloseWebSocketFrame
- PingWebSocketFrame
- PongWebSocketFrame
- TextWebSocketFrame
只需要处理TextWebSocketFrame，其他的由WebSocketServerProtocolHandler自动处理
```java
public class TextWebSocketFrameHandler extends SimpleChannelInboundHandler<TextWebSocketFrame> { //1
    private final ChannelGroup group;

    public TextWebSocketFrameHandler(ChannelGroup group) {
        this.group = group;
    }

    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {    //2
        if (evt == WebSocketServerProtocolHandler.ServerHandshakeStateEvent.HANDSHAKE_COMPLETE) {
            ctx.pipeline().remove(HttpRequestHandler.class);    //3
            group.writeAndFlush(new TextWebSocketFrame("Client " + ctx.channel() + " joined"));//4
            group.add(ctx.channel());    //5
        } else {
            super.userEventTriggered(ctx, evt);
        }
    }

    @Override
    public void channelRead0(ChannelHandlerContext ctx, TextWebSocketFrame msg) throws Exception {
        group.writeAndFlush(msg.retain());    //6
    }
}
```
1. 扩展 SimpleChannelInboundHandler 用于处理 TextWebSocketFrame 信息
2. 覆写userEventTriggered() 方法来处理自定义事件
3. 如果接收的事件表明握手成功,就从 ChannelPipeline 中删除HttpRequestHandler ，因为接下来不会接受 HTTP 消息了
4. 写一条消息给所有的已连接 WebSocket 客户端，通知它们建立了一个新的 Channel 连接
5. 添加新连接的 WebSocket Channel 到 ChannelGroup 中，这样它就能收到所有的信息
6. 保留收到的消息，并通过 writeAndFlush() 传递给所有连接的客户端。

- 当WebSocket 与新客户端已成功握手完成，通过写入信息到 ChannelGroup 中的 Channel 来通知所有连接的客户端，然后添加新 Channel 到 ChannelGroup
- 如果接收到 TextWebSocketFrame，调用 retain() ，并将其写、刷新到 ChannelGroup，使所有连接的 WebSocket Channel 都能接收到它。和以前一样，retain() 是必需的，因为当 channelRead0（）返回时，TextWebSocketFrame 的引用计数将递减。由于所有操作都是异步的，writeAndFlush() 可能会在以后完成，我们不希望它访问无效的引用。

由于 Netty 在其内部处理了其余大部分功能，唯一剩下的需要我们去做的就是为每一个新创建的 Channel 初始化 ChannelPipeline 。要完成这个，我们需要一个ChannelInitializer

**初始化 ChannelPipeline**
我们需要安装我们上面实现的两个 ChannelHandler 到 ChannelPipeline
为此，我们需要继承 ChannelInitializer 并且实现 initChannel()
```java
public class ChatServerInitializer extends ChannelInitializer<Channel> {    //1
    private final ChannelGroup group;

    public ChatServerInitializer(ChannelGroup group) {
        this.group = group;
    }

    @Override
    protected void initChannel(Channel ch) throws Exception {            //2
        ChannelPipeline pipeline = ch.pipeline();
        pipeline.addLast(new HttpServerCodec());
        pipeline.addLast(new HttpObjectAggregator(64 * 1024));
        pipeline.addLast(new ChunkedWriteHandler());
        pipeline.addLast(new HttpRequestHandler("/ws"));
        pipeline.addLast(new WebSocketServerProtocolHandler("/ws"));
        pipeline.addLast(new TextWebSocketFrameHandler(group));
    }
}
```
1. 扩展 ChannelInitializer
2. 添加 ChannelHandler　到 ChannelPipeline

initChannel方法用于设置所有新注册channel的ChannelPipeline，安装所有需要的ChannelHandler
```bash
HttpServerCodec					HttpRequest、HttpContent、LastHttpContent的编码和解码
ChunkedWriteHandler				写文件内容
HttpObjectAggregator			把HttpMessage和HttpContents聚合为FullHttpResponse，后续收到的都是完整的requests
HttpRequestHandler				处理http请求
WebSocketServerProtocolHandler	处理升级握手/PingWebSocketFrames/PongWebSocketFrames/CloseWebSocketFrames
TextWebSocketFrameHandler		处理TextWebSocketFrames和握手完成事件
```
WebSocketServerProtocolHandler处理所有的WebSocket 帧类型和升级握手
如果握手成功，所有的ChannelHandler添加进管道，不需要的被移除；下图为ChannelPipeline刚经过ChatServerInitializer初始化

升级之前的状态
![](http://waylau.com/essential-netty-in-action/images/Figure%2011.3%20ChannelPipeline%20before%20WebSockets%20Upgrade.jpg)
升级成功后，WebSocketServerProtocolHandler 替换HttpRequestDecoder为WebSocketFrameDecoder
HttpResponseEncoder为WebSocketFrameEncoder；
为了提升性能，不需要的ChannelHandler会被移除，其中包括HttpObjectAggregator和HttpRequestHandler


ChannelPipeline 经过调整如下，netty目前支持4个版本的WebSocket协议，选择正确的WebSocketFrameDecoder和WebSocketFrameEncoder自动执行
这又取决于浏览器的支持
![](http://waylau.com/essential-netty-in-action/images/Figure%2011.4%20ChannelPipeline%20after%20WebSockets%20Upgrade.jpg)
**引导**
```java
public class ChatServer {

    private final ChannelGroup channelGroup = new DefaultChannelGroup(ImmediateEventExecutor.INSTANCE);//1
    private final EventLoopGroup group = new NioEventLoopGroup();
    private Channel channel;

    public ChannelFuture start(InetSocketAddress address) {
        ServerBootstrap bootstrap  = new ServerBootstrap(); //2
        bootstrap.group(group)
                .channel(NioServerSocketChannel.class)
                .childHandler(createInitializer(channelGroup));
        ChannelFuture future = bootstrap.bind(address);
        future.syncUninterruptibly();
        channel = future.channel();
        return future;
    }

    protected ChannelInitializer<Channel> createInitializer(ChannelGroup group) {        //3
       return new ChatServerInitializer(group);
    }

    public void destroy() {        //4
        if (channel != null) {
            channel.close();
        }
        channelGroup.close();
        group.shutdownGracefully();
    }

    public static void main(String[] args) throws Exception{
        if (args.length != 1) {
            System.err.println("Please give port as argument");
            System.exit(1);
        }
        int port = Integer.parseInt(args[0]);

        final ChatServer endpoint = new ChatServer();
        ChannelFuture future = endpoint.start(new InetSocketAddress(port));

        Runtime.getRuntime().addShutdownHook(new Thread() {
            @Override
            public void run() {
                endpoint.destroy();
            }
        });
        future.channel().closeFuture().syncUninterruptibly();
    }
}
```
1. 创建 DefaultChannelGroup 用来 保存所有连接的的 WebSocket channel
2. 引导 服务器
3. 创建 ChannelInitializer
4. 处理服务器关闭，包括释放所有资源

**测试程序**
启动服务：`mvn -PChatServer clean package exec:exec`
修改属性：`mvn -PChatServer -Dport=1111 clean package exec:exec`
通过`ttp://localhost:9999`访问地址
![](http://waylau.com/essential-netty-in-action/images/Figure%2011.5%20WebSockets%20ChatServer%20demonstration.jpg)
图中显示了两个已经连接了的客户端。第一个客户端是通过上面的图形界面连接的，第二个是通过Chrome浏览器底部的命令行连接的。 你可以注意到，这两个客户端都在发送消息，每条消息都会显示在两个客户端上。

**如何加密**
添加SslHandler 
```java
public class SecureChatServerIntializer extends ChatServerInitializer {    //1
    private final SslContext context;

    public SecureChatServerIntializer(ChannelGroup group, SslContext context) {
        super(group);
        this.context = context;
    }

    @Override
    protected void initChannel(Channel ch) throws Exception {
        super.initChannel(ch);
        SSLEngine engine = context.newEngine(ch.alloc());
        engine.setUseClientMode(false);
        ch.pipeline().addFirst(new SslHandler(engine)); //2
    }
}
```
1. 扩展 ChatServerInitializer 来实现加密
2. 向 ChannelPipeline 中添加SslHandler
最后修改 ChatServer，使用 SecureChatServerInitializer 并传入 SSLContext
```java
public class SecureChatServer extends ChatServer {//1

    private final SslContext context;

    public SecureChatServer(SslContext context) {
        this.context = context;
    }

    @Override
    protected ChannelInitializer<Channel> createInitializer(ChannelGroup group) {
        return new SecureChatServerIntializer(group, context);    //2
    }

    public static void main(String[] args) throws Exception{
        if (args.length != 1) {
            System.err.println("Please give port as argument");
            System.exit(1);
        }
        int port = Integer.parseInt(args[0]);
        SelfSignedCertificate cert = new SelfSignedCertificate();
        SslContext context = SslContext.newServerContext(cert.certificate(), cert.privateKey());
        final SecureChatServer endpoint = new SecureChatServer(context);
        ChannelFuture future = endpoint.start(new InetSocketAddress(port));

        Runtime.getRuntime().addShutdownHook(new Thread() {
            @Override
            public void run() {
                endpoint.destroy();
            }
        });
        future.channel().closeFuture().syncUninterruptibly();
    }
}
```
1. 扩展 ChatServer
2. 返回先前创建的 SecureChatServerInitializer 来启用加密
现在你可以通过 HTTPS 地址: https://localhost:9999 来访问SecureChatServer 了

## 二. SPDY
**背景**
主要使加载速度加快
- 每个头都是压缩的，消息体的压缩是可选的,因为它可能对代理服务器有问题
- 所有的加密都使用 TLS 每个连接多个转移是可能的 数据集可以单独设置优先级,使关键内容先被转移
对比
```bash
浏览器			HTTP 1.1			SPDY
加密				Not by default		Yes
Header压缩		No					Yes
全双工			No					Yes
Server push		No					Yes
优先级			No					Yes
```
只会提供一些静态内容回客户机
内容将取决于所使用协议是 HTTPS 或 SPDY
如果 服务器提供 SPDY 是可以被客户端浏览器所支持，则自动切换到 SPDY
![](http://waylau.com/essential-netty-in-action/images/Figure%2012.1%20Application%20logic.jpg)

**实现**
SPDY 使用 TLS 的扩展称为 Next Protocol Negotiation (NPN)
java中，两种不同的npn协议
- 使用 ssl_npn,NPN 的开源 SSL 提供者
- 使用通过 Jetty 的 NPN 扩展库
这里使用Jetty库

jetty库提供了一个接口ServerProvider

