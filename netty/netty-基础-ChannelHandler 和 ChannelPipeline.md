## ChannelHandler 和 ChannelPipeline
Channel
ChannelHandler
ChannePipeline
ChannelHandlerContext

## 一. ChannelHandler
**Channel 生命周期**

```bash
状态							描述
channelUnregistered			channel创建但未注册到一个 EventLoop.
channelRegistered			channel 注册到一个 EventLoop.
channelActive				channel 的活动的(连接到了它的 remote peer（远程对等方）)，现在可以接收和发送数据了
channelInactive				channel 没有连接到 remote peer（远程对等方）
```
Channel 状态发生变化，对应的事件会生成，这样与ChannelPipeline中的ChannelHandler交互就能及时响应；
状态模型如下
![](http://waylau.com/essential-netty-in-action/images/Figure%206.1%20Channel%20State%20Model.jpg)

声明周期方法，每个方法都带有ChannelHandlerContext参数

```bash
类型				描述
handlerAdded	当 ChannelHandler 添加到 ChannelPipeline 调用
handlerRemoved	当 ChannelHandler 从 ChannelPipeline 移除时调用
exceptionCaught	当 ChannelPipeline 执行发生错误时调用
```

ChannelHandler子接口
- ChannelInboundHandler 处理进站数据，并且所有状态都更改
- ChannelOutboundHandler 处理出站数据，允许拦截各种操作

另外提供三个适配器：ChannelHandlerAdapter、ChannelInboundHandlerAdapter、ChannelOutboundHandlerAdapter方便使用

**ChannelInboundHandler**
ChannelInboundHandler的生命周期方法

```bash
channelRegistered			channel被注册到EventLoop并且可以处理io
channelUnregistered			channel从EventLoop卸载，并且不能处理io
channelActive				channel变为active模式，通道connected/boundb准备好了
channelInactive				channel不活跃，不再连接远程的	
channelReadComplete			channel上的读操作完成了
channelRead					数据从Channel中读出了
channelWritabilityChanged	Channel的读写性改变时调用，
userEventTriggered(...) 	用户调用Channel.fireUserEventTriggered(...)，从ChannelPipeline传递特定的消息
```
ChannelInboundHandler覆盖了channelRead方法，处理数据
Netty 在 ByteBuf 上使用了资源池，执行释放资源时可以减少内存的消耗；

```java
@ChannelHandler.Sharable
public class DiscardHandler extends ChannelInboundHandlerAdapter {        //1
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        ReferenceCountUtil.release(msg); //2
    }
}
```
1. 扩展 ChannelInboundHandlerAdapter
2. ReferenceCountUtil.release() 来丢弃收到的信息

netty使用一个WARN级别的日志记录未释放的资源，手工释放比较麻烦因此，使用SimpleChannelInboundHandler可以简化问题；
也就是说，不发出就会在channelHandler中自动释放了

```java
@ChannelHandler.Sharable
public class SimpleDiscardHandler extends SimpleChannelInboundHandler<Object> {  //1
    @Override
    public void channelRead0(ChannelHandlerContext ctx, Object msg) {
        // No need to do anything special //2
    }
}
```
1. 扩展 SimpleChannelInboundHandler
2. 不需做特别的释放资源的动作

**ChannelOutboundHandler**
提供了出站的方法，这些方法会被Channel, ChannelPipeline, 和 ChannelHandlerContext调用
可以延迟操作

```java
bind    
connect 
disconnect
close   
deregister
read    
flush   
write
```
ChannelPromise作为参数，一旦请求结束要通过 ChannelPipeline 转发的时候，必须通知此参数。
ChannelPromise 是 特殊的 ChannelFuture，任何时候调用通道方法，例如Channel.write(...) ，都会创建新的ChannelPromise，并通过ChannelPipeline传递；

**资源管理**
可以这样处理数据

```java
ChannelInboundHandler.channelRead(...) 
ChannelOutboundHandler.write(...)
```
Netty使用引用计数处理池化的ByteBuf，ByteBuf 完全处理后，要确保引用计数器被调整。
当JVM仍在GC这个消息，或者不小心释放，很可能会耗尽资源；
ResourceLeakDetector从已分配的缓冲区 1% 作为样品来检查是否存在在应用程序泄漏
泄露等级

```bash
Disables
SIMPLE
ADVANCED
PARANOID
```
在ChannelInboundHandler.channelRead(...) 和 ChannelOutboundHandler.write(...)中避免泄露
当处理channelRead并在消费消息时释放，如下

```java
@ChannelHandler.Sharable
public class DiscardInboundHandler extends ChannelInboundHandlerAdapter {  //1
    @Override
    public void channelRead(ChannelHandlerContext ctx,Object msg) {
        ReferenceCountUtil.release(msg); //2
    }
}
```
1. 继承 ChannelInboundHandlerAdapter
2. 使用 ReferenceCountUtil.release(...) 来释放资源

另外SimpleChannelInboundHandler能够自动释放，channelRead0() 提供的消息
处理写操作，并丢弃消息时，需要释放它

```java
@Override
public void write(ChannelHandlerContext ctx,Object msg, ChannelPromise promise) {
    ReferenceCountUtil.release(msg);  //2
    promise.setSuccess();    //3
}
```
1. 继承 ChannelOutboundHandlerAdapter
2. 使用 ReferenceCountUtil.release(...) 来释放资源
3. 通知 ChannelPromise 数据已经被处理

这里重要的是释放资源并通知ChannelPromise，否则可能会引发ChannelFutureListener不会被处理消息通知的状况；
如果消息是被 消耗/丢弃 并不会被传入下个 ChannelPipeline 的 ChannelOutboundHandler ，调用 ReferenceCountUtil.release(message) 。
一旦消息经过实际的传输，在消息被写或者 Channel 关闭时，它将会自动释放。

## 二. ChannelPipeline
一个事件将由 ChannelInboundHandler 或 ChannelOutboundHandler 处理
通过ChannelHandlerContext转发

![](http://waylau.com/essential-netty-in-action/images/Figure%206.2%20ChannelPipeline%20and%20ChannelHandlers.jpg)
ChannelHandler可以实时修改 ChannelPipeline 的布局，通过添加、移除、替换其他 ChannelHandler，或者移除自身；
ChannelHandler修改ChannelPipeline的方法

```bash
addFirst addBefore addAfter 	addLast	添加 ChannelHandler 到 ChannelPipeline.
Remove							从 ChannelPipeline 移除 ChannelHandler.
Replace							在 ChannelPipeline 替换另外一个 ChannelHandler
```
操作如下

```java
ChannelPipeline pipeline = null; // get reference to pipeline;
FirstHandler firstHandler = new FirstHandler(); //1
pipeline.addLast("handler1", firstHandler); //2
pipeline.addFirst("handler2", new SecondHandler()); //3
pipeline.addLast("handler3", new ThirdHandler()); //4

pipeline.remove("handler3"); //5
pipeline.remove(firstHandler); //6 

pipeline.replace("handler2", "handler4", new ForthHandler()); //6
```
- 创建一个 FirstHandler 实例
- 添加该实例作为 "handler1" 到 ChannelPipeline
- 添加 SecondHandler 实例作为 "handler2" 到 ChannelPipeline 的第一个槽，这意味着它将替换之前已经存在的 "handler1"
- 添加 ThirdHandler 实例作为"handler3" 到 ChannelPipeline 的最后一个槽
- 通过名称移除 "handler3"
- 通过引用移除 FirstHandler (因为只有一个，所以可以不用关联名字 "handler1"）.
- 将作为"handler2"的 SecondHandler 实例替换为作为 "handler4"的 FourthHandler

**执行ChannelPipeline和阻塞**
ChannelHandler添加到ChannelPipeline，处理事件传递到EventLoop，一定不要阻塞线程；
ChannelPipeline的add方法，传入一个EventExecutorGroup，默认的实现DefaultEventExecutorGroup；
其他操作

```bash
get(...)
context(...)
names() iterator()
```
**发送事件**
入站操作

```bash
fireChannelRegistered 		channelRegistered
fireChannelUnregistered 	channelUnregistered
fireChannelActive 			channelActive
fireChannelInactive 		channelInactive
fireExceptionCaught 		exceptionCaught(ChannelHandlerContext, Throwable) 
fireUserEventTriggered 		userEventTriggered(ChannelHandlerContext, Object)
fireChannelRead 			channelRead(ChannelHandlerContext, Object msg) 
fireChannelReadComplete  	channelReadComplete(ChannelHandlerContext) 
```
出站操作

```bash
bind			下一个的bind(ChannelHandlerContext, SocketAddress, ChannelPromise)	
connect 		下一个调用connect(ChannelHandlerContext, SocketAddress, ChannelPromise)
disconnect 		下一个disconnect
close 			下一个close
deregister 		下一个deregister
flush 			下一个flush 
write 			下一个write
writeAndFlush 	write() then flush()，分解成flush和write
read 			下一个read
```
总结：
- 一个 ChannelPipeline 是用来保存关联到一个 Channel 的ChannelHandler
- 可以修改 ChannelPipeline 通过动态添加和删除 ChannelHandler
- ChannelPipeline 有着丰富的API调用动作来回应入站和出站事件

## 三. ChannelHandlerContext
ChannelHandlerContext代表ChannelHandler和ChannelPipeline之间的关联;
在ChannelHandler添加到ChannelPipeline时创建一个ChannelHandlerContext的实例，管理同一个ChannelPipeline关联的ChannelHandler之间的交互；
ChannelHandlerContext有许多方法，通过Channel 或ChannelPipeline调用的方法，会在pipeline中传播；
而在ChannelHandlerContext上调用，就会传递到下一个ChannelHandler ；
ChannelHandlerContext的API

```bash
bind					Request to bind to the given SocketAddress and return a ChannelFuture.
channel					Return the Channel which is bound to this instance.
close					Request to close the Channel and return a ChannelFuture.
connect					Request to connect to the given SocketAddress and return a ChannelFuture.
deregister				Request to deregister from the previously assigned EventExecutor and return a ChannelFuture.
disconnect				Request to disconnect from the remote peer and return a ChannelFuture.
executor				Return the EventExecutor that dispatches events.
fireChannelActive		A Channel is active (connected).
fireChannelInactive		A Channel is inactive (closed).
fireChannelRead			A Channel received a message.
fireChannelReadComplete	Triggers a channelWritabilityChanged event to the next ChannelInboundHandler
handler 
isRemoved 
name 
pipeline 
read 
write 
```
- ChannelHandlerContext 与 ChannelHandler 的关联从不改变，所以缓存它的引用是安全的。
- 正如我们前面指出的,ChannelHandlerContext 所包含的事件流比其他类中同样的方法都要短，利用这一点可以尽可能高地提高性能

**使用 ChannelHandler**
ChannelPipeline, Channel, ChannelHandler 和 ChannelHandlerContext 的关系如图
![](
http://waylau.com/essential-netty-in-action/images/Figure%206.3%20Channel,%20ChannelPipeline,%20ChannelHandler%20and%20ChannelHandlerContext.jpg)

- Channel 绑定到 ChannelPipeline
- ChannelPipeline 绑定到 包含 ChannelHandler 的 Channel
- ChannelHandler
- 当添加 ChannelHandler 到 ChannelPipeline 时，ChannelHandlerContext 被创建

通过ChannelHandlerContext访问Channel 

```java
ChannelHandlerContext ctx = context;
Channel channel = ctx.channel();  //1
channel.write(Unpooled.copiedBuffer("Netty in Action",
        CharsetUtil.UTF_8));  //2
```
- 得到与 ChannelHandlerContext 关联的 Channel 的引用
- 通过 Channel 写缓存

通过ChannelHandlerContext访问ChannelPipeline 

```java
ChannelHandlerContext ctx = context;
ChannelPipeline pipeline = ctx.pipeline(); //1
pipeline.write(Unpooled.copiedBuffer("Netty in Action", CharsetUtil.UTF_8));  //2
```
- 得到与 ChannelHandlerContext 关联的 ChannelPipeline 的引用
- 通过 ChannelPipeline 写缓冲区

Channel或者ChannelPipeline的write方法，会在整个通道内传播，但是ChannelHandler级别上，只能调用ChannelHandlerContext的方法传递到下一个ChannelHandler上

![](http://waylau.com/essential-netty-in-action/images/Figure%206.4%20Event%20propagation%20via%20the%20Channel%20or%20the%20ChannelPipeline.jpg)
- ChannelHandlerContext 方法调用
- 事件发送到了下一个 ChannelHandler
- 经过最后一个ChannelHandler后，事件从 ChannelPipeline 移除

使用例子ChannelPipeline事件

```java
ChannelHandlerContext ctx = context;
ctx.write(Unpooled.copiedBuffer("Netty in Action",              CharsetUtil.UTF_8));
```
- 获得 ChannelHandlerContext 的引用
- write() 将会把缓冲区发送到下一个 ChannelHandler

如下所示,消息将会从下一个ChannelHandler开始流过 ChannelPipeline ,绕过所有在它之前的ChannelHandler。

![](http://waylau.com/essential-netty-in-action/images/Figure%206.5%20Event%20flow%20for%20operations%20triggered%20via%20the%20ChannelHandlerContext.jpg)

1. ChannelHandlerContext 方法调用
2. 事件发送到了下一个 ChannelHandler
3. 经过最后一个ChannelHandler后，事件从 ChannelPipeline 移除

**高级用法**
调用ChannelHandlerContext的 pipeline()方法,可以得到一个封闭的ChannelPipeline引用，可以在运行时操作pipeline的ChannelHandler，可以实现复杂的需求，比如添加一个 ChannelHandler 到 pipeline 来支持动态协议改变；
又比如保持一个ChannelHandlerContext引用，供以后使用，可能发生在任何ChannelHandler的方法中，甚至不同的线程；

```java
public class WriteHandler extends ChannelHandlerAdapter {
    private ChannelHandlerContext ctx;
    @Override
    public void handlerAdded(ChannelHandlerContext ctx) {
        this.ctx = ctx;        //1
    }
    public void send(String msg) {
        ctx.writeAndFlush(msg);  //2
    }
}
```
- 存储 ChannelHandlerContext 的引用供以后使用
- 使用之前存储的 ChannelHandlerContext 来发送消息

ChannelHandler可以属于多个ChannelPipeline，可以绑定多个ChannelHandlerContext实例，

```java
@ChannelHandler.Sharable            //1
public class SharableHandler extends ChannelInboundHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        System.out.println("channel read message " + msg);
        ctx.fireChannelRead(msg);  //2
    }
}
```
1. 添加 @Sharable 注解
2. 日志方法调用， 并专递到下一个 ChannelHandler

为什么共享 ChannelHandler
多个 ChannelPipelines 上安装一个 ChannelHandler 以此来实现跨多个渠道收集统计数据的目的。

## 四. Decoder解码器
- Decoder(解码器)
- Encoder(编码器)
- Codec(编解码器)

Netty 提供了丰富的解码器抽象基类,大致分两种
- 解码字节到消息（ByteToMessageDecoder 和 ReplayingDecoder）
- 解码消息到消息（MessageToMessageDecoder）

decoder是一种ChannelInboundHandler 
encoder是一种ChannelInboundHandler

**ByteToMessageDecoder**
用于将字节转为消息，或其他字节序列；
接收的可能不是完整信息，这个类会缓存入站的数据，知道准备好了用于处理；

```bash
Decode	
decodeLast	
```
假设我们接收一个包含简单整数的字节流,每个都单独处理。在本例中,我们将从入站 ByteBuf 读取每个整数并将其传递给 pipeline 中的下一个ChannelInboundHandler。“解码”字节流成整数我们将扩展ByteToMessageDecoder，实现类为“ToIntegerDecoder”,如图

![](http://waylau.com/essential-netty-in-action/images/Figure%207.1%20ToIntegerDecoder.jpg)
当不能再添加数据到lsit时，内容就会被传到ChannelInboundHandler；

ToIntegerDecoder代码如下：
每次从ByteBuf读取4字节，添加到List，满了之后发送到下一个ChannelInboundHandler

```java
public class ToIntegerDecoder extends ByteToMessageDecoder {  //1
    @Override
    public void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out)
            throws Exception {
        if (in.readableBytes() >= 4) {  //2
            out.add(in.readInt());  //3
        }
    }
}
```
- 实现继承了 ByteToMessageDecode 用于将字节解码为消息
- 检查可读的字节是否至少有4个 ( int 是4个字节长度)
- 从入站 ByteBuf 读取 int ， 添加到解码消息的 List 中

引用计数问题，在编码和解码时它自动调用`ReferenceCountUtil.release(message)`
如果还需要使用这个引用，不马上释放可以调用`ReferenceCountUtil.retain(message)`增加引用计数,防止消息被释放

**ReplayingDecoder**
byte-to-message解码的一种特殊的抽象基类,读取缓冲区之前需要检查缓冲区是否满足又有足够字节，ReplayingDecoder不必手动检查；
ByteBuf中有足够的字节会正常读取，没有足够字节则会停止解码；
 ReplayingDecoder 继承自 ByteToMessageDecoder
- 不是所有的标准 ByteBuf 操作都被支持，如果调用一个不支持的操作会抛出 UnreplayableOperationException
- ReplayingDecoder 略慢于 ByteToMessageDecoder

不引入过多的复杂性 使用 ByteToMessageDecoder 。否则,使用ReplayingDecoder

```java
public class ToIntegerDecoder2 extends ReplayingDecoder<Void> {   //1
    @Override
    public void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out)
            throws Exception {
        out.add(in.readInt());  //2
    }
}
```
- 实现继承自 ReplayingDecoder 用于将字节解码为消息
- 从入站 ByteBuf 读取整型，并添加到解码消息的 List 中

更多比如：
io.netty.handler.codec.LineBasedFrameDecoder 通过结束控制符("\n" 或 "\r\n").解析入站数据。
io.netty.handler.codec.http.HttpObjectDecoder 用于 HTTP 数据解码

**MessageToMessageDecoder**
一种消息转为另一种消息，比如POJO到POJO

```bash
decode
decodeLast
```
将 Integer 转为 String，我们提供了 IntegerToStringDecoder继承自MessageToMessageDecoder

```java
public class IntegerToStringDecoder extends MessageToMessageDecoder<Integer>
```
decode() 方法的签名是

```java
protected void decode( ChannelHandlerContext ctx,Integer msg, List<Object> out ) throws Exception
```
入站消息是按照在类定义中声明的参数类型而不是 ByteBuf来解析的
解码消息被添加到List，并传递到下个ChannelInboundHandler

![](http://waylau.com/essential-netty-in-action/images/Figure%207.2%20IntegerToStringDecoder.jpg)

```java
public class IntegerToStringDecoder extends
        MessageToMessageDecoder<Integer> { //1

    @Override
    public void decode(ChannelHandlerContext ctx, Integer msg, List<Object> out)
            throws Exception {
        out.add(String.valueOf(msg)); //2
    }
}
```
- 实现继承自 MessageToMessageDecoder
- 通过 String.valueOf() 转换 Integer 消息字符串

decode()方法的消息参数的类型是由给这个类指定的泛型的类型(这里是Integer)确定的。

HttpObjectAggregator继承自MessageToMessageDecoder

**在解码时处理太大的帧**
解码器不能缓存太多数据，会引发TooLongFrameException，可以在解码器中设置阈值，如果超出会导致TooLongFrameException，会被ChannelHandler.exceptionCaught() 捕获；然后由译码器的用户决定如何处理它；
ByteToMessageDecoder可以利用TooLongFrameException通知其他ChannelPipeline中的ChannelHandler；

```java
public class SafeByteToMessageDecoder extends ByteToMessageDecoder {  //1
    private static final int MAX_FRAME_SIZE = 1024;

    @Override
    public void decode(ChannelHandlerContext ctx, ByteBuf in,
                       List<Object> out) throws Exception {
        int readable = in.readableBytes();
        if (readable > MAX_FRAME_SIZE) { //2
            in.skipBytes(readable);        //3
            throw new TooLongFrameException("Frame too big!");
        }
        // do something
    }
}
```
- 实现继承 ByteToMessageDecoder 来将字节解码为消息
- 检测缓冲区数据是否大于 MAX_FRAME_SIZE
- 忽略所有可读的字节，并抛出 TooLongFrameException 来通知 ChannelPipeline 中的 ChannelHandler 这个帧数据超长

这种保护是很重要的，尤其是当你解码一个有可变帧大小的协议的时候;


## 五. Encoder(编码器)
encoder把一种格式转化为另一种格式，实现了ChanneOutboundHandler；
- 编码从消息到字节
- 编码从消息到消息

**MessageToByteEncoder**
将消息转化为字节

```java
encode
```
这里只有一个消息，但是decoder却是两个，decoder 经常需要在 Channel 关闭时产生一个“最后的消息”。
出于这个原因，提供了decodeLast()，而 encoder 没有这个需求；

ShortToByteEncoder
![](http://waylau.com/essential-netty-in-action/images/Figure%207.3%20ShortToByteEncoder.jpg)
encoder 收到了 Short 消息，编码他们，并把他们写入 ByteBuf。
ByteBuf 接着前进到下一个 pipeline 的ChannelOutboundHandler。
每个 Short 将占用 ByteBuf 的两个字节;

```java
public class ShortToByteEncoder extends
        MessageToByteEncoder<Short> {  //1
    @Override
    public void encode(ChannelHandlerContext ctx, Short msg, ByteBuf out)
            throws Exception {
        out.writeShort(msg);  //2
    }
}
```
- 实现继承自 MessageToByteEncoder
- 写 Short 到 ByteBuf

提供很多例子：WebSocket08FrameEncoder 

**MessageToMessageEncoder**
从一个消息格式解码成另一个格式

```java
encode
```
IntegerToStringEncoder
encoder 从出站字节流提取 Integer,以 String 形式传递给ChannelPipeline 中的下一个 ChannelOutboundHandler;

```java
public class IntegerToStringEncoder extends
        MessageToMessageEncoder<Integer> { //1

    @Override
    public void encode(ChannelHandlerContext ctx, Integer msg, List<Object> out)
            throws Exception {
        out.add(String.valueOf(msg));  //2
    }
}
```
- 实现继承自 MessageToMessageEncoder
- 转 Integer 为 String，并添加到 MessageBuf

更复杂的ProtobufEncoder

## 六. 抽象Codec(编解码器)类
把入站和出站的数据和信息转换都放在同一个类中
这些类同时实现了ChannelInboundHandler和ChannelOutboundHandler

**ByteToMessageCodec**
结合了ByteToMessageDecoder和MessageToByteEncoder

```bash
decode
decodeLast
encode
```

**MessageToMessageCodec**

```bash
decode
decodeLast
encode
```
一个参数化的类
`public abstract class MessageToMessageCodec<INBOUND,OUTBOUND>`
完整的签名都是这样的

```java
protected abstract void encode(ChannelHandlerContext ctx,
OUTBOUND msg, List<Object> out)
protected abstract void decode(ChannelHandlerContext ctx,
INBOUND msg, List<Object> out)
```
WebSocketConvertHandler 继承了INBOUND WebSocketFrame和OUTBOUND WebSocketFrame

```java
public class WebSocketConvertHandler extends MessageToMessageCodec<WebSocketFrame, WebSocketConvertHandler.WebSocketFrame> {  //1

    public static final WebSocketConvertHandler INSTANCE = new WebSocketConvertHandler();

    @Override
    protected void encode(ChannelHandlerContext ctx, WebSocketFrame msg, List<Object> out) throws Exception {   
        ByteBuf payload = msg.getData().duplicate().retain();
        switch (msg.getType()) {   //2
            case BINARY:
                out.add(new BinaryWebSocketFrame(payload));
                break;
            case TEXT:
                out.add(new TextWebSocketFrame(payload));
                break;
            case CLOSE:
                out.add(new CloseWebSocketFrame(true, 0, payload));
                break;
            case CONTINUATION:
                out.add(new ContinuationWebSocketFrame(payload));
                break;
            case PONG:
                out.add(new PongWebSocketFrame(payload));
                break;
            case PING:
                out.add(new PingWebSocketFrame(payload));
                break;
            default:
                throw new IllegalStateException("Unsupported websocket msg " + msg);
        }
    }

    @Override
    protected void decode(ChannelHandlerContext ctx, io.netty.handler.codec.http.websocketx.WebSocketFrame msg, List<Object> out) throws Exception {
        if (msg instanceof BinaryWebSocketFrame) {  //3
            out.add(new WebSocketFrame(WebSocketFrame.FrameType.BINARY, msg.content().copy()));
        } else if (msg instanceof CloseWebSocketFrame) {
            out.add(new WebSocketFrame(WebSocketFrame.FrameType.CLOSE, msg.content().copy()));
        } else if (msg instanceof PingWebSocketFrame) {
            out.add(new WebSocketFrame(WebSocketFrame.FrameType.PING, msg.content().copy()));
        } else if (msg instanceof PongWebSocketFrame) {
            out.add(new WebSocketFrame(WebSocketFrame.FrameType.PONG, msg.content().copy()));
        } else if (msg instanceof TextWebSocketFrame) {
            out.add(new WebSocketFrame(WebSocketFrame.FrameType.TEXT, msg.content().copy()));
        } else if (msg instanceof ContinuationWebSocketFrame) {
            out.add(new WebSocketFrame(WebSocketFrame.FrameType.CONTINUATION, msg.content().copy()));
        } else {
            throw new IllegalStateException("Unsupported websocket msg " + msg);
        }
    }

    public static final class WebSocketFrame {  //4
        public enum FrameType {        //5
            BINARY,
            CLOSE,
            PING,
            PONG,
            TEXT,
            CONTINUATION
        }

        private final FrameType type;
        private final ByteBuf data;
        public WebSocketFrame(FrameType type, ByteBuf data) {
            this.type = type;
            this.data = data;
        }

        public FrameType getType() {
            return type;
        }

        public ByteBuf getData() {
            return data;
        }
    }
}
```
- 编码 WebSocketFrame 消息转为 WebSocketFrame 消息
- 检测 WebSocketFrame 的 FrameType 类型，并且创建一个新的响应的 FrameType 类型的 WebSocketFrame
- 通过 instanceof 来检测正确的 FrameType
- 自定义消息类型 WebSocketFrame
- 枚举类明确了 WebSocketFrame 的类型
- CombinedChannelDuplexHandl

**CombinedChannelDuplexHandler**
解码器和编码器在一起可能会牺牲可重用性
`public class CombinedChannelDuplexHandler<I extends ChannelInboundHandler,O extends ChannelOutboundHandler>`
这个类是扩展 ChannelInboundHandler 和 ChannelOutboundHandler 参数化的类型,这样无需扩展了
ByteToCharDecoder

```java
public class ByteToCharDecoder extends
        ByteToMessageDecoder { //1

    @Override
    public void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out)
            throws Exception {
        if (in.readableBytes() >= 2) {  //2
            out.add(in.readChar());
        }
    }
}
```
1. 继承 ByteToMessageDecoder
2. 写 char 到 MessageBuf

CharToByteEncoder

```java
public class CharToByteEncoder extends
        MessageToByteEncoder<Character> { //1

    @Override
    public void encode(ChannelHandlerContext ctx, Character msg, ByteBuf out)
            throws Exception {
        out.writeChar(msg);   //2
    }
}
```
1. 继承 MessageToByteEncoder
2. 写 char 到 ByteBuf

CombinedByteCharCodec

```java
public class CombinedByteCharCodec extends CombinedChannelDuplexHandler<ByteToCharDecoder, CharToByteEncoder> {
    public CombinedByteCharCodec() {
        super(new ByteToCharDecoder(), new CharToByteEncoder());
    }
}
```
1. CombinedByteCharCodec 的参数是解码器和编码器的实现用于处理进站字节和出站消息
2. 传递 ByteToCharDecoder 和 CharToByteEncoder 实例到 super 构造函数来委托调用使他们结合起来