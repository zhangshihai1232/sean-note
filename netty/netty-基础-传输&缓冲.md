## 一. Transport
NIO和OIO的切换，只需修改一行代码，在创建线程池的地方

```java
EventLoopGroup group = new OioEventLoopGroup();
NioEventLoopGroup group = new NioEventLoopGroup();
```
传输的核心在Channel 接口，用于所有的出站操作
![](http://waylau.com/essential-netty-in-action/images/Figure%204.1%20Channel%20interface%20hierarchy.jpg)
每个 Channel 都会分配一个 ChannelPipeline 和ChannelConfig
ChannelConfig:设置并存储Channel的配置,并允许在运行期间更新它们
ChannelPipeline:容纳了使用的 ChannelHandler实例

ChannelHandler处理通道传递的“入站”和“出站”数据以及事件，允许你改变数据状态和传输数据；
ChannelHandler可以做的事情：
- 传输数据时，将数据从一种格式转换到另一种格式
- 异常通知
- Channel 变为 active（活动） 或 inactive（非活动） 时获得通知，Channel 被注册或注销时从 EventLoop 中获得通知
- 通知用户特定事件

**拦截过滤器**
ChannelPipeline实现了Intercepting Filter设计模式
channel的核心方法

```bash
方法名称				描述
eventLoop()			返回分配给Channel的EventLoop
pipeline()			返回分配给Channel的ChannelPipeline
isActive()			返回Channel是否激活，已激活说明与远程连接对等
localAddress()		返回已绑定的本地SocketAddress
remoteAddress()		返回已绑定的远程SocketAddress
write()				写数据到远程客户端，数据通过ChannelPipeline传输过去
flush()				刷新先前的数据
writeAndFlush(...)	一个方便的方法用户调用write(...)而后调用y flush()
```
操作都是在相同的接口上运行,Netty 的高灵活性让你可以以不同的传输实现进行重构；
`写数据到远程已连接客户端可以调用Channel.write()方法`代码如下

```java
Channel channel = ...; // 获取channel的引用
ByteBuf buf = Unpooled.copiedBuffer("your data", CharsetUtil.UTF_8);            //1
ChannelFuture cf = channel.writeAndFlush(buf); //2

cf.addListener(new ChannelFutureListener() {    //3
    @Override
    public void operationComplete(ChannelFuture future) {
        if (future.isSuccess()) {                //4
            System.out.println("Write successful");
        } else {
            System.err.println("Write error");    //5
            future.cause().printStackTrace();
        }
    }
});
```
Channel 是线程安全的,存储对Channel的引用,多线程时使用;

```java
final Channel channel = ...; // 获取channel的引用
final ByteBuf buf = Unpooled.copiedBuffer("your data",
        CharsetUtil.UTF_8).retain();    //1
Runnable writer = new Runnable() {        //2
    @Override
    public void run() {
        channel.writeAndFlush(buf.duplicate());
    }
};
Executor executor = Executors.newCachedThreadPool();//3
//写进一个线程
executor.execute(writer);        //4
//写进另外一个线程
executor.execute(writer);        //5
```

Netty自带了一些传输协议的实现；
传输方式

```bash
方法名称		包							描述
NIO			io.netty.channel.socket.nio	基于java.nio.channels的工具包，使用选择器作为基础的方法。
OIO			io.netty.channel.socket.oio	基于java.net的工具包，使用阻塞流。
Local		io.netty.channel.local		用来在虚拟机之间本地通信。
Embedded	io.netty.channel.embedded	嵌入传输，它允许在没有真正网络的传输中使用 ChannelHandler，测试用
```

**NIO**
完全异步
可以注册一个通道或获得某个通道的改变的状态
- 一个新的 Channel 被接受并已准备好
- Channel 连接完成
- Channel 中有数据并已准备好读取
- Channel 发送数据出去

SelectionKey定义操作

```bash
方法名称		描述
OP_ACCEPT	有新连接时得到通知
OP_CONNECT	连接完成后得到通知
OP_REA		准备好读取数据时得到通知
OP_WRITE	写入更多数据到通道时得到通知，大部分时间
```
![](http://waylau.com/essential-netty-in-action/images/Figure%204.2%20Selecting%20and%20Processing%20State%20Changes.jpg)
1. 新信道注册 WITH 选择器
2. 选择处理的状态变化的通知
3. 以前注册的通道
4. Selector.select（）方法阻塞，直到新的状态变化接收或配置的超时
5. 检查是否有状态变化
6. 处理所有的状态变化
7. 在选择器操作的同一个线程执行其他任务

NIo支持zero-file-copy，支持原生内容的传输；

**OIO**
构建在 java.net 的阻塞实现上，有时需要用阻塞的方式，比如jdbc
如何在nio架构上支持oio呢？
SO_TIMEOUT标志，timeout指定最大毫秒，等待io完成；
如果指定时间之内失败，跑出SocketTimeoutException，捕获继续处理循环；
![](http://waylau.com/essential-netty-in-action/images/Figure%204.3%20OIO-Processing%20logic.jpg)
1. 线程分配给 Socket
2. Socket 连接到远程
3. 读操作（可能会阻塞）
4. 读完成
5. 处理可读的字节
6. 执行提交到 socket 的其他任务
7. 再次尝试读

**Transport 使用情况**

```bash
Transport	TCP		UDP		SCTP*	UDT
NIO			X		X		X		X
OIO			X		X		X		X
```
下面是你可能遇到的用例:
- OIO-在低连接数、需要低延迟时、阻塞时使用
- NIO-在高连接数时使用
- Local-在同一个JVM内通信时使用
- Embedded-测试ChannelHandler时使用

## 二. Buffer
ByteBuf针对Netty的ChannelPipeline语意设计
使用引用计数,判断何时可以释放 ByteBuf 或 ByteBufHolder 和其他相关资源
缓冲api优势
- 可以自定义缓冲类型
- 通过一个内置的复合缓冲类型实现零拷贝
- 扩展性好，比如 StringBuilder
- 不需要调用 flip() 来切换读/写模式
- 读取和写入索引分开
- 方法链
- 引用计数
- Pooling(池)

**ByteBuf**
网络通信基于底层的字节流
ByteBuf有2部分：一个用于读，一个用于写


**堆缓冲区**
存储在JVM 的堆空间，快速分配和释放
ByteBuf.array() 来获取 byte[]数据

```java
ByteBuf heapBuf = ...;
if (heapBuf.hasArray()) {                //1
    byte[] array = heapBuf.array();        //2
    int offset = heapBuf.arrayOffset() + heapBuf.readerIndex();                //3
    int length = heapBuf.readableBytes();//4
    handleArray(array, offset, length); //5
}
```
1. 检查 ByteBuf 是否有支持数组。
2. 如果有的话，得到引用数组。
3. 计算第一字节的偏移量。
4. 获取可读的字节数。
5. 使用数组，偏移量和长度作为调用方法的参数

不能访问非堆缓冲区，需要使用ByteBuf.hasArray()检查是否支持访问数组
用法和jdk的byteBuffer类似

**直接缓冲区**
本地方法，分配内存
- 通过免去中间交换的内存拷贝, 提升IO处理速度; 直接缓冲区的内容可以驻留在垃圾回收扫描的堆区以外
- DirectBuffer 在 -XX:MaxDirectMemorySize=xxM大小限制下, 使用 Heap 之外的内存, GC对此”无能为力”,也就意味着规避了在高负载下频繁的GC过程对应用线程的中断影响

```java
ByteBuf directBuf = ...
if (!directBuf.hasArray()) {            //1
    int length = directBuf.readableBytes();//2
    byte[] array = new byte[length];    //3
    directBuf.getBytes(directBuf.readerIndex(), array);        //4    
    handleArray(array, 0, length);  //5
}
```
1. 检查 ByteBuf 是不是由数组支持。如果不是，这是一个直接缓冲区。
2. 获取可读的字节数
3. 分配一个新的数组来保存字节
4. 字节复制到数组
5. 将数组，偏移量和长度作为参数调用某些处理方法

**复合缓冲区**
复合缓冲区就像一个列表，我们可以动态的添加和删除其中的 ByteBuf
CompositeByteBuf是ByteBuf子类，用于处理复合缓冲区
CompositeByteBuf.hasArray() 总是返回 false，因为它可能既包含堆缓冲区，也包含直接缓冲区

一条消息由 header 和 body 两部分组成，将 header 和 body 组装成一条消息发送出去，可能 body 相同，只是 header 不同，使用CompositeByteBuf 就不用每次都重新分配一个新的缓冲区。

![](http://waylau.com/essential-netty-in-action/images/Figure%205.2%20CompositeByteBuf%20holding%20a%20header%20and%20body.jpg)

jdk的ByteBuffer需要重新创建新的ByteBuffer，把两个复制进来；
使用CompositeByteBuf的版本如下

```java
CompositeByteBuf messageBuf = ...;
ByteBuf headerBuf = ...; // 可以支持或直接
ByteBuf bodyBuf = ...; // 可以支持或直接
messageBuf.addComponents(headerBuf, bodyBuf);
// ....
messageBuf.removeComponent(0); // 移除头    //2

for (int i = 0; i < messageBuf.numComponents(); i++) {                        //3
    System.out.println(messageBuf.component(i).toString());
}
```

1. 追加 ByteBuf 实例的 CompositeByteBuf
2. 删除 索引1的 ByteBuf
3. 遍历所有 ByteBuf 实例。

你可以简单地把 CompositeByteBuf 当作一个可迭代遍历的容器，不能直接访问数组
访问方式

```java
CompositeByteBuf compBuf = ...;
int length = compBuf.readableBytes();    //1
byte[] array = new byte[length];        //2
compBuf.getBytes(compBuf.readerIndex(), array);    //3
handleArray(array, 0, length);    //4
```

1. 得到的可读的字节数
2. 分配一个新的数组,数组长度为可读字节长度
3. 读取字节到数组
4. 使用数组，把偏移量和长度作为参数

## 三. 字节级别的操作
**随机访问使用索引**

```java
ByteBuf buffer = ...;
for (int i = 0; i < buffer.capacity(); i++) {
    byte b = buffer.getByte(i);
    System.out.println((char) b);
}
```
索引访问不会推进readerIndex和writerIndex，可以通过ByteBuf 的 readerIndex(index) 或 writerIndex(index) 来分别推进读索引或写索引；
jdk的ByteBuffer只有一个索引，所以要引入flip()方法切换

![](http://waylau.com/essential-netty-in-action/images/Figure%205.3%20ByteBuf%20internal%20segmentation.jpg)
1. 字节，可以被丢弃，因为它们已经被读
2. 还没有被读的字节是：“readable bytes（可读字节）”
3. 空间可加入多个字节的是：“writeable bytes（写字节）”

可丢弃字节可以被回收，调用discardReadBytes()回收，这个段的初始大小存储在readerIndex为0，当“read”操作被执行时递增（“get”操作不会移动 readerIndex）。
调用 discardReadBytes() 之后，在丢弃字节段的空间已变得可用写。

![](http://waylau.com/essential-netty-in-action/images/Figure%205.4%20ByteBuf%20after%20discarding%20read%20bytes.jpg)
ByteBuf.discardReadBytes() 可以用来清空 ByteBuf 中已读取的数据，从而使 ByteBuf 有多余的空间容纳新的数据，但是discardReadBytes() 可能会涉及内存复制，因为它需要移动 ByteBuf 中可读的字节到开始位置，这样的操作会影响性能，一般在需要马上释放内存的时候使用收益会比较大。

**可读字节**
遍历可读字节

```java
ByteBuf buffer= ...;
while (buffer.isReadable()) {
    System.out.println(buffer.readByte());
}
```
填充缓冲区

```java
//填充随机整数到缓冲区中
ByteBuf buffer = ...;
while (buffer.writableBytes() >= 4) {
    buffer.writeInt(random.nextInt());
}
```

**索引管理**
您可以设置和重新定位ByteBuf readerIndex 和 writerIndex 通过调用 markReaderIndex(), markWriterIndex(), resetReaderIndex() 和 resetWriterIndex();
可以通过调用 readerIndex(int) 或 writerIndex(int) 将指标移动到指定的位置。
在尝试任何无效位置上设置一个索引将导致 IndexOutOfBoundsException 异常。
调用 clear() 可以同时设置 readerIndex 和 writerIndex 为 0,这不会清除内存中的内容；

![](http://waylau.com/essential-netty-in-action/images/Figure%205.5%20Before%20clear%20is%20called.jpg)

调用clear()之后，整个ByteBuf空间都是可用的了
![](http://waylau.com/essential-netty-in-action/images/Figure%205.6%20After%20clear%20is%20called.jpg)
clear() 比 discardReadBytes() 更低成本，因为他只是重置了索引，而没有内存拷贝。

**查询操作**
缓冲器中的指定值的索引,最简单的是使用 indexOf() 方法；
更复杂的搜索执行以 ByteBufProcessor 为参数的方法；
这个接口定义了一个方法，boolean process(byte value)，它用来报告输入值是否是一个正在寻求的值；

```java
//报告输入值是否是一个正在寻求的值
boolean process(byte value)
//Flash sockets使用NULL结尾的内容
forEachByte（ByteBufProcessor.FIND_NUL）
//查找\r
ByteBuf buffer = ...;
int index = buffer.forEachByte(ByteBufProcessor.FIND_CR);
```

**衍生的缓冲区**
展示ByteBuf的视图，由 duplicate(), slice(), slice(int, int),readOnly(), 和 order(ByteOrder) 方法创建；
返回一个新的 ByteBuf 实例包括它自己的 reader, writer 和标记索引；
修改一个地方，影响全部的视图，包括原来的byteBuf

操作某段数据

```java
Charset utf8 = Charset.forName("UTF-8");
ByteBuf buf = Unpooled.copiedBuffer("Netty in Action rocks!", utf8); //1

ByteBuf sliced = buf.slice(0, 14);          //2
System.out.println(sliced.toString(utf8));  //3

buf.setByte(0, (byte) 'J');                 //4
assert buf.getByte(0) == sliced.getByte(0);
```
1. 创建一个 ByteBuf 保存特定字节串。
2. 创建从索引 0 开始，并在 14 结束的 ByteBuf 的新 slice。
3. 打印 Netty in Action
4. 更新索引 0 的字节。
5. 断言成功，因为数据是共享的，并以一个地方所做的修改将在其他地方可见。

copy一个bytebuf

```java
Charset utf8 = Charset.forName("UTF-8");
ByteBuf buf = Unpooled.copiedBuffer("Netty in Action rocks!", utf8);     //1

ByteBuf copy = buf.copy(0, 14);               //2
System.out.println(copy.toString(utf8));      //3

buf.setByte(0, (byte) 'J');                   //4
assert buf.getByte(0) != copy.getByte(0);
```
1. 创建一个 ByteBuf 保存特定字节串。
2. 创建从索引0开始和 14 结束 的 ByteBuf 的段的拷贝。
3. 打印 Netty in Action
4. 更新索引 0 的字节。
5. 断言成功，因为数据不是共享的，并以一个地方所做的修改将不影响其他。

代码几乎是相同的，但所 衍生的 ByteBuf 效果是不同的。因此，使用一个 slice 可以尽可能避免复制内存。

**读/写操作**
读/写操作主要由2类：
- get()/set() 操作从给定的索引开始，保持不变
- read()/write() 操作从给定的索引开始，与字节访问的数量来适用，递增当前的写索引或读索引

常见的get()操作

```java
方法名称									描述
getBoolean(int)							返回当前索引的 Boolean 值
getByte(int)/getUnsignedByte(int)		返回当前索引的(无符号)字节
getMedium(int)/getUnsignedMedium(int)	返回当前索引的 (无符号) 24-bit 中间值
getInt(int)/getUnsignedInt(int)			返回当前索引的(无符号) 整型
getLong(int)/getUnsignedLong(int)		返回当前索引的 (无符号) Long 型
getShort(int)/getUnsignedShort(int)		返回当前索引的 (无符号) Short 型
getBytes(int, ...)						字节
```
常见 set() 操作

```java
方法名称							描述
setBoolean(int, boolean)		在指定的索引位置设置 Boolean 值
setByte(int, int)				在指定的索引位置设置 byte 值
setMedium(int, int)				在指定的索引位置设置 24-bit 中间 值
setInt(int, int)				在指定的索引位置设置 int 值
setLong(int, long)				在指定的索引位置设置 long 值
setShort(int, int)				在指定的索引位置设置 short 值
```
get和set例子

```java
Charset utf8 = Charset.forName("UTF-8");
ByteBuf buf = Unpooled.copiedBuffer("Netty in Action rocks!", utf8);    //1
System.out.println((char)buf.getByte(0));                    //2

int readerIndex = buf.readerIndex();                        //3
int writerIndex = buf.writerIndex();

buf.setByte(0, (byte)'B');                            //4

System.out.println((char)buf.getByte(0));                    //5
assert readerIndex == buf.readerIndex();                    //6
assert writerIndex ==  buf.writerIndex();
```
read操作

```java
方法名称
readBoolean()　	
readByte()/readUnsignedByte()　
readMedium()/readUnsignedMedium()　
readInt()/readUnsignedInt()	　
readLong()/readUnsignedLong()　	
readShort()/readUnsignedShort()　	
readBytes(int,int, ...)
```
write操作

```java
writeBoolean(boolean)  
writeByte(int)  
writeMedium(int) 
writeInt(int)   
writeLong(long) 
writeShort(int) 
writeBytes(int，...）
```
read/write例子

```java
Charset utf8 = Charset.forName("UTF-8");
ByteBuf buf = Unpooled.copiedBuffer("Netty in Action rocks!", utf8);    //1
System.out.println((char)buf.readByte());                    //2

int readerIndex = buf.readerIndex();                        //3
int writerIndex = buf.writerIndex();                        //4

buf.writeByte((byte)'?');                            //5

assert readerIndex == buf.readerIndex();
assert writerIndex != buf.writerIndex();
```
更多操作

```java
isReadable()    
isWritable()    
readableBytes() 
writablesBytes()
capacity() 
maxCapacity()   
hasArray() 
array()
```

## 四. ByteBufHolder

```java
data()	//返回 ByteBuf 保存的数据
copy()	//制作一个 ByteBufHolder 的拷贝，但不共享其数据(所以数据也是拷贝).
```

## 五. ByteBuf 分配
**ByteBufAllocator**
池类，减少内存的分配和释放
提供的操作

```bash
buffer()
heapBuffer()
directBuffer()
compositeBuffer()
ioBuffer()
```

## 六. 引用计数器
ByteBuf和ByteBufHolder，都实现了ReferenceCounted接口；
引用计数大于0不被释放，减少到0释放；

```java
Channel channel = ...;
ByteBufAllocator allocator = channel.alloc(); //1
....
ByteBuf buffer = allocator.directBuffer(); //2
assert buffer.refCnt() == 1; //3
...
```
1. 从 channel 获取 ByteBufAllocator
2. 从 ByteBufAllocator 分配一个 ByteBuf
3. 检查引用计数器是否是 1

释放

```java
ByteBuf buffer = ...;
boolean released = buffer.release(); //1
```
release将会递减对象引用的数目。当这个引用计数达到0时，对象已被释放，并且该方法返回 true。
最后访问的对象负责释放它