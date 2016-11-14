## 一. Channel
通道与流类似，但是有些区别
- 既可以从通道中读取数据，又可以写数据到通道。但流的读写通常是单向的
- 通道可以异步地读写
- 通道中的数据总是要先读到一个Buffer，或者总是要从一个Buffer中写入

常用的通道
- FileChannel：从文件中读写数据。
- DatagramChannel：能通过UDP读写网络中的数据。
- SocketChannel：能通过TCP读写网络中的数据。
- ServerSocketChannel：可以监听新进来的TCP连接，像Web服务器那样。对每一个新进来的连接都会创建一个SocketChannel

例子

```java
RandomAccessFile aFile = new RandomAccessFile("data/nio-data.txt", "rw");  
FileChannel inChannel = aFile.getChannel();  
ByteBuffer buf = ByteBuffer.allocate(48);  
int bytesRead = inChannel.read(buf);  
while (bytesRead != -1) {  
	System.out.println("Read " + bytesRead);  
	buf.flip();  
	while(buf.hasRemaining()){  
		System.out.print((char) buf.get());  
	}  
	buf.clear();  
	bytesRead = inChannel.read(buf);  
}  
aFile.close();  
```
buf.flip() 的调用，首先读取数据到Buffer，然后反转Buffer,接着再从Buffer中读取数据。

## 二. Buffer
缓冲区本质上是一块可以写入数据，然后可以从中读取数据的内存。这块内存被包装成NIO Buffer对象，并提供了一组方法，用来方便的访问该块内存。 
使用Buffer读写数据一般遵循以下四个步骤： 
- 写入数据到Buffer
- 调用flip()方法
- 从Buffer中读取数据
- 调用clear()方法或者compact()方法

向buffer写入数据时，buffer会记录下写了多少数据。一旦要读取数据，需要通过flip()方法将Buffer从写模式切换到读模式。在读模式下，可以读取之前写入到buffer的所有数据。 
读完了所有数据，需要清空缓冲区，有两种方式：
- clear()方法会清空整个缓冲区
- compact()方法只会清除已经读过的数据，未被读取的数据移动到缓冲区起始处，新写入数据放在未读取数据后面

使用buffer例子

```java
RandomAccessFile aFile = new RandomAccessFile("data/nio-data.txt", "rw");  
FileChannel inChannel = aFile.getChannel();  
ByteBuffer buf = ByteBuffer.allocate(48);  
int bytesRead = inChannel.read(buf); //read into buffer.  
while (bytesRead != -1) {  
  buf.flip();  //make buffer ready for read  
  while(buf.hasRemaining()){  
      System.out.print((char) buf.get()); // read 1 byte at a time  
  }  
  buf.clear(); //make buffer ready for writing  
  bytesRead = inChannel.read(buf);  
}  
aFile.close();  
```

**buffer内部结构**
capacity

```bash
固定的大小值
```
position

```bash
当你写数据到Buffer中时，position表示当前的位置；当读取数据时，也是从某个特定位置读。当将Buffer从写模式切换到读模式，position会被重置为0。当从Buffer的position处读取数据时，position向前移动到下一个可读的位置。 
```
limit

```bash
在写模式下，Buffer的limit表示你最多能往Buffer里写多少数据。 写模式下，limit等于Buffer的capacity
当切换Buffer到读模式时， limit表示你最多能读到多少数据。因此，当切换Buffer到读模式时，limit会被设置成写模式下的position值。换句话说，你能读到之前写入的所有数据
```
![](http://dl2.iteye.com/upload/attachment/0096/4782/b8a7bad8-ec65-36dc-bb11-4f352e00cd67.png)

Buffer类型,这些Buffer类型代表了不同的数据类型
通过char，short，int，long，float 或 double类型来操作缓冲区中的字节

```java
ByteBuffer
MappedByteBuffer
CharBuffer
DoubleBuffer
FloatBuffer
IntBuffer
LongBuffer
ShortBuffer
```
**buffer操作**
分配

```java
ByteBuffer buf = ByteBuffer.allocate(48);  
CharBuffer buf = CharBuffer.allocate(1024);  
```
写数据，两种方式
- 从Channel写到Buffer
- 通过Buffer的put()方法写到Buffer里

```java
//从Channel写到Buffer的例子 
int bytesRead = inChannel.read(buf);
buf.put(127); 
```
put有很多半分
flip()方法
读取两种方式
- 从Buffer读取数据到Channel
- 使用get()方法从Buffer中读取数据

```java
int bytesWritten = inChannel.write(buf); 
byte aByte = buf.get();
```
get方法也有很多版本
rewind()方法 

```bash
Buffer.rewind()将position设回0，所以你可以重读Buffer中的所有数据。limit保持不变，仍然表示能从Buffer中读取多少个元素（byte、char等
```
clear()与compact()方法,让buffer再次被写入
clear方法，position将被设回0，limit被设置成 capacity的值。
compact()方法将所有未读的数据拷贝到Buffer起始处。然后将position设到最后一个未读元素正后面。limit属性依然像clear()方法一样，设置成capacity。Buffer准备好写数据了，但是不会覆盖未读的数据。

mark()与reset()方法 
调用Buffer.mark()方法，可以标记Buffer中的一个特定position。
之后可以通过调用Buffer.reset()方法恢复到这个position。

```java
buffer.mark();  
//call buffer.get() a couple of times, e.g. during parsing.  
buffer.reset();  //set position back to mark.  
```
equals()
相等条件
- 有相同的类型（byte、char、int等）
- Buffer中剩余的byte、char等的个数相等
- Buffer中所有剩余的byte、char等都相同

equals只比较Buffer中的剩余元素

compareTo()方法，比较剩余元素
如果满足下列条件，则认为一个Buffer“小于”另一个Buffer
- 第一个不相等的元素小于另一个Buffer中对应的元素
- 所有元素都相等，但第一个Buffer比另一个先耗尽(第一个Buffer的元素个数比另一个少)

剩余元素就是position和limit之间的元素

buffer写例子，由于数据以byte形式存储，字符串需要进行编解码

```java
    ByteBuffer buffer = ByteBuffer.allocate(100);
    buffer.clear();
    buffer.put(("什么万一这个是").getBytes(Charset.forName("UTF-8")));
    buffer.flip();
    byte[] bytes = new byte[buffer.limit()-buffer.position()];
    buffer.get(bytes);
    String s = new String(bytes, "UTF-8");
    System.out.println(s);
```
## 三. Scatter、Gather
scatter/gather用于描述从Channel中读取或者写入到Channel的操作；
scatter：从Channel中读取是指在读操作时将读取的数据写入多个buffer中
gather：写入Channel是指在写操作时将多个buffer的数据写入同一个Channel

scatter / gather经常用于需要将传输的数据分开处理的场合，例如传输一个由消息头和消息体组成的消息，你可能会将消息体和消息头分散到不同的buffer中，这样你可以方便的处理消息头和消息体。 

**Scattering Reads**
从一个channel读取到多个buffer中
![](http://dl2.iteye.com/upload/attachment/0096/4789/56355a57-22cc-35d0-bb6c-f951fed0e084.png)

```java
ByteBuffer header = ByteBuffer.allocate(128);  
ByteBuffer body = ByteBuffer.allocate(1024);  
ByteBuffer[] bufferArray = { header, body };  
channel.read(bufferArray);  
```
buffer首先被插入到数组，然后再将数组作为channel.read() 的输入参数;
read()方法按照buffer在数组中的顺序将从channel中读取的数据写入到buffer，当一个buffer被写满后，channel紧接着向另一个buffer中写;
Scattering Reads在移动下一个buffer前，必须填满当前的buffer，这也意味着它不适用于动态消息;
如果存在消息头和消息体，消息头必须完成填充（例如 128byte），Scattering Reads才能正常工作。 

**Gathering Writes**
数据从多个buffer写入到同一个channel,如下图
![](http://dl2.iteye.com/upload/attachment/0096/4791/3eb74d8a-3180-3ca2-9541-4966fb46d44d.png)

```java
ByteBuffer header = ByteBuffer.allocate(128);  
ByteBuffer body   = ByteBuffer.allocate(1024);  
//write data into buffers  
ByteBuffer[] bufferArray = { header, body };  
channel.write(bufferArray);  
```
write()方法会按照buffer在数组中的顺序，将数据写入到channel;
注意只有position和limit之间的数据才会被写入。
Gathering Writes能较好的处理动态消息。 

## 四. 通道之间的数据传输
如果两个通道中有一个是FileChannel，那你可以直接将数据从一个channel传输到另外一个channel；
**transferFrom()**
FileChannel的transferFrom方法，可以将数据从源通道传输到FileChannel中

```java
RandomAccessFile fromFile = new RandomAccessFile("fromFile.txt", "rw");
FileChannel fromChannel = fromFile.getChannel();
RandomAccessFile toFile = new RandomAccessFile("toFile.txt", "rw");
FileChannel toChannel = toFile.getChannel();
long position = 0;
long count = fromChannel.size();
toChannel.transferFrom(position, count, fromChannel); 
```
方法的输入参数position表示从position处开始向目标文件写入数据，count表示最多传输的字节数
需要注意：在SoketChannel的实现中，SocketChannel只会传输此刻准备好的数据（可能不足count字节）。因此，SocketChannel可能不会将请求的所有数据(count个字节)全部传输到FileChannel中

**transferTo()**
transferTo()方法将数据从FileChannel传输到其他的channel中

```java
RandomAccessFile fromFile = new RandomAccessFile("fromFile.txt", "rw");
FileChannel fromChannel = fromFile.getChannel();
RandomAccessFile toFile = new RandomAccessFile("toFile.txt", "rw");
FileChannel toChannel = toFile.getChannel();
long position = 0;
long count = fromChannel.size();
fromChannel.transferTo(position, count, toChannel);
```
除了调用方法的FileChannel对象不一样外，其他的都一样。 
上面所说的关于SocketChannel的问题在transferTo()方法中同样存在。SocketChannel会一直传输数据直到目标buffer被填满。

## 五. Selector
创建

```java
Selector selector = Selector.open();  
```
注册通道,通道必须设置为非阻塞模式
FileChannel不能切换到非阻塞模式，套接字可以

```java
channel.configureBlocking(false);  
SelectionKey key = channel.register(selector,  
    Selectionkey.OP_READ);  
```
第二个参数是一个interest集合，就是监听Selector对什么事件感兴趣
四种类型的事件
- Connect
- Accept
- Read
- Write

服务器上，channel成功连接，另一个服务器就是连接就绪；
一个server socket channel准备好接收新进入的连接称为“接收就绪”；
一个有数据可读的通道可以说是“读就绪”。等待写数据的通道可以说是“写就绪”。 
由四个常量来表示
- SelectionKey.OP_CONNECT
- SelectionKey.OP_ACCEPT
- SelectionKey.OP_READ
- SelectionKey.OP_WRITE

如果对不止一件事情感兴趣，可以使用位或将常量连接起来

```java
int interestSet = SelectionKey.OP_READ | SelectionKey.OP_WRITE;  
```
**SelectionKey**
向selector注册channel时，返回一个SelectionKey对象，包含了一些感兴趣的属性
- interest集合
- ready集合
- Channel
- Selector
- 附加的对象（可选）

**interest集合**
感兴趣的集合，通过SelectionKey读interest集合

```java
int interestSet = selectionKey.interestOps();  
boolean isInterestedInAccept  = (interestSet & SelectionKey.OP_ACCEPT) == SelectionKey.OP_ACCEPT；  
boolean isInterestedInConnect = interestSet & SelectionKey.OP_CONNECT;  
boolean isInterestedInRead    = interestSet & SelectionKey.OP_READ;  
boolean isInterestedInWrite   = interestSet & SelectionKey.OP_WRITE; 
```
用“位与”操作interest 集合和给定的SelectionKey常量，可以确定某个确定的事件是否在interest 集合中
**ready集合**
通道已经准备就绪的操作的集合

```java
int readySet = selectionKey.readyOps(); 
```
可以从selectionKey中检测什么事件或操作已经就绪

```java
selectionKey.isAcceptable();  
selectionKey.isConnectable();  
selectionKey.isReadable();  
selectionKey.isWritable();  
```
**Channel + Selector**
SelectionKey访问Channel和Selector

```java
Channel channel = selectionKey.channel();  
Selector selector = selectionKey.selector();  
```
**附加的对象**
将一个对象或者更多信息附着到SelectionKey上,这样能识别某个给定的通道
比如可以绑定一个buffer到通道上，或是包含聚集数据的某个对象

```java
selectionKey.attach(theObject);  
Object attachedObj = selectionKey.attachment();  
```
可以在向selector注册的时候绑定附加对象

```java
SelectionKey key = channel.register(selector, SelectionKey.OP_READ, theObject);  
```
**通过Selector选择通道**
配置了对那些通道状态感兴趣，selector就会返回那些状态的通道
select()方法
- int select()
- int select(long timeout)
- int selectNow()

select()阻塞，直到至少一个通道就绪了
select(long timeout)和select()一样，最长阻塞timeout时间
selectNow()不阻塞

返回值为int，表示有多少个通道就绪；
自上次调用select方法后，有多少个通道变成就绪态
如果调用select()方法，因为有一个通道变成就绪状态，返回了1，若再次调用select()方法，如果另一个通道就绪了，它会再次返回1。如果对第一个就绪的channel没有做任何操作，现在就有两个就绪的通道，但在每次select()方法调用之间，只有一个通道就绪了。 

**selectedKeys()**
一旦调用了select()方法，并且返回值表明有一个或更多个通道就绪了
可以使用selectedKeys方法，访问set中的就绪通道

```java
Set selectedKeys = selector.selectedKeys();  
```
注册时返回SelectionKey对象，这个对象代表了注册进selector的通道，可以通过SelectionKey的selectedKeySet()方法访问这些对象；
可以遍历这个已选择的键集合来访问就绪的通道

```java
Set selectedKeys = selector.selectedKeys();  
Iterator keyIterator = selectedKeys.iterator();  	
while(keyIterator.hasNext()) {  
    SelectionKey key = keyIterator.next();  
    if(key.isAcceptable()) {  
        // a connection was accepted by a ServerSocketChannel.  
    } else if (key.isConnectable()) {  
        // a connection was established with a remote server.  
    } else if (key.isReadable()) {  
        // a channel is ready for reading  
    } else if (key.isWritable()) {  
        // a channel is ready for writing  
    }  
    keyIterator.remove() 
}  
```
这个循环遍历已选择键集中的每个键，并检测各个键所对应的通道的就绪事件，注意每次迭代末尾的keyIterator.remove()调用。Selector不会自己从已选择键集中移除SelectionKey实例。必须在处理完通道时自己移除。下次该通道变成就绪时，Selector会再次将其放入已选择键集中。 
SelectionKey.channel()返回的方法需要强制类型转换成需要的，如ServerSocketChannel或SocketChannel等。 

**wakeUp**
某个线程调用select()方法后阻塞了，即使没有通道已经就绪。也有办法让其从select()方法返回。
只要让其它线程在第一个线程调用select()方法的那个对象上调用Selector.wakeup()方法即可，阻塞在select()方法上的线程会立马返回。 
如果有其它线程调用了wakeup()方法，但当前没有线程阻塞在select()方法上，下个调用select()方法的线程会立即“醒来（wake up）”。 

**close()**
用完Selector后调用其close()方法会关闭该Selector，且使注册到该Selector上的所有SelectionKey实例无效。通道本身并不会关闭。 
**完整的示例**
打开一个Selector，注册一个通道注册到这个Selector上(通道的初始化过程略去),然后持续监控这个Selector的四种事件（接受，连接，读，写）是否就绪。 

```java
Selector selector = Selector.open();  
channel.configureBlocking(false);  
SelectionKey key = channel.register(selector, SelectionKey.OP_READ);  
while(true) {  
  int readyChannels = selector.select();  
  if(readyChannels == 0) continue;  
  Set selectedKeys = selector.selectedKeys();  
  Iterator keyIterator = selectedKeys.iterator();  
  while(keyIterator.hasNext()) {  
    SelectionKey key = keyIterator.next();  
    if(key.isAcceptable()) {  
        // a connection was accepted by a ServerSocketChannel.  
    } else if (key.isConnectable()) {  
        // a connection was established with a remote server.  
    } else if (key.isReadable()) {  
        // a channel is ready for reading  
    } else if (key.isWritable()) {  
        // a channel is ready for writing  
    }  
    keyIterator.remove(); 
  }  
}  
```
http://www.iteye.com/magazines/132-Java-NIO#583

## 六. FileChannel
只运行在阻塞模式
FileChannel需要利用InputStream、OutputStream或RandomAccessFile打开

```java
RandomAccessFile aFile = new RandomAccessFile("data/nio-data.txt", "rw");  
FileChannel inChannel = aFile.getChannel();  
```
读取数据

```java
ByteBuffer buf = ByteBuffer.allocate(48);  
int bytesRead = inChannel.read(buf);
```
写入数据

```java
String newData = "New String to write to file..." + System.currentTimeMillis();  
ByteBuffer buf = ByteBuffer.allocate(48);  
buf.clear();  
buf.put(newData.getBytes());  
buf.flip();  
while(buf.hasRemaining()) {  
    channel.write(buf);  
}  
```
FileChannel.write()是在while循环中调用的。因为无法保证write()方法一次能向FileChannel写入多少字节，因此需要重复调用write()方法，直到Buffer中已经没有尚未写入通道的字节。 
关闭

```java
channel.close();  
```
position方法
有时可能需要在FileChannel的某个特定位置进行数据的读/写操作。可以通过调用position()方法获取FileChannel的当前位置。 
也可以通过调用position(long pos)方法设置FileChannel的当前位置.

```java
long pos = channel.position();  
channel.position(pos +123);  
```
如果将位置设置在文件结束符之后，然后试图从文件通道中读取数据，读方法将返回-1 —— 文件结束标志。
如果将位置设置在文件结束符之后，然后向通道中写数据，文件将撑大到当前位置并写入数据。这可能导致“文件空洞”，磁盘上物理文件中写入的数据间有空隙。 
size方法
返回该实例所关联文件的大小

```java
long fileSize = channel.size();  
```
FileChannel.truncate()方法截取一个文件,指定长度后面的部分将被删除

```java
channel.truncate(1024);
```
force方法 
将通道里尚未写入磁盘的数据强制写到磁盘上,参数指明是否同时将文件元数据（权限信息等）写到磁盘上。 

```java
channel.force(true);
```