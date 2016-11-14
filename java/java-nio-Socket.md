## 一. Socket 通道
创建方式
1. 打开一个SocketChannel并连接到互联网上的某台服务器
2. 一个新连接到达ServerSocketChannel时，会创建一个SocketChannel

打开 SocketChannel 

```java
SocketChannel socketChannel = SocketChannel.open();
socketChannel.connect(new InetSocketAddress("http://jenkov.com", 80));
```
关闭 SocketChannel 

```java
socketChannel.close();  
```
从 SocketChannel 读取数据 

```java
ByteBuffer buf = ByteBuffer.allocate(48);
int bytesRead = socketChannel.read(buf);
```
如果返回的是-1，表示已经读到了流的末尾
写入 SocketChannel 

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
注意SocketChannel.write()方法的调用是在一个while循环中的。Write()方法无法保证能写多少字节到SocketChannel。所以，我们重复调用write()直到Buffer没有要写的字节为止。
**非阻塞模式**
设置之后，就可以在异步模式下调用connect(), read() 和write()
connect() 
调用connect，可能连接建立之前就返回，调用finishConnect确定连接是否建立

```java
socketChannel.configureBlocking(false);
socketChannel.connect(new InetSocketAddress("http://jenkov.com", 80));
while(! socketChannel.finishConnect() ){
    //wait, or do something else...
}
```
write()
非阻塞模式下，write()方法在尚未写出任何内容时可能就返回了。所以需要在循环中调用write()
read()
非阻塞模式下,read()方法在尚未读取到任何数据时可能就返回了。所以需要关注它的int返回值，它会告诉你读取了多少字节。
非阻塞模式与选择器 
通过将一或多个SocketChannel注册到Selector，可以询问选择器哪个通道已经准备好了读取，写入等。

## 二. ServerSocket 通道
ServerSocketChannel监听tcp连接

```java
ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
serverSocketChannel.socket().bind(new InetSocketAddress(9999));
while(true){
    SocketChannel socketChannel =
            serverSocketChannel.accept();
    //do something with socketChannel...
}
```
打开 ServerSocketChannel 

```java
ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
```
关闭 ServerSocketChannel 

```java
serverSocketChannel.close();
```
监听新进来的连接(accept方法是阻塞的)
**阻塞模式**
通过 ServerSocketChannel.accept() 方法监听新进来的连接。当 accept()方法返回的时候，它返回一个包含新进来的连接的 SocketChannel。因此，accept()方法会一直阻塞到有新连接到达

```java
while(true){
    SocketChannel socketChannel =
            serverSocketChannel.accept();
    //do something with socketChannel...
}
```

**非阻塞模式**
accept() 方法会立刻返回，如果还没有新进来的连接，返回的将是null。 因此，需要检查返回的SocketChannel是否是null；

```java
ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
serverSocketChannel.socket().bind(new InetSocketAddress(9999));
serverSocketChannel.configureBlocking(false);
while(true){
    SocketChannel socketChannel =
            serverSocketChannel.accept();
    if(socketChannel != null){
        //do something with socketChannel...
    }
}
```

## 三. Datagram 通道
它发送和接收的是数据包
打开 DatagramChannel 

```java
DatagramChannel channel = DatagramChannel.open();
channel.socket().bind(new InetSocketAddress(9999));
```
这个例子打开的 DatagramChannel可以在UDP端口9999上接收数据包
接收数据
receive()方法会将接收到的数据包内容复制到指定的Buffer. 如果Buffer容不下收到的数据，多出的数据将被丢弃。
 
```java
ByteBuffer buf = ByteBuffer.allocate(48);
buf.clear();
channel.receive(buf);
```
发送数据 
通过send()方法从DatagramChannel发送数据

```java
String newData = "New String to write to file..." + System.currentTimeMillis();
ByteBuffer buf = ByteBuffer.allocate(48);
buf.clear();
buf.put(newData.getBytes());
buf.flip();
int bytesSent = channel.send(buf, new InetSocketAddress("jenkov.com", 80));
```
因为服务端并没有监控这个端口，所以什么也不会发生。也不会通知你发出的数据包是否已收到，因为UDP在数据传送方面没有任何保证
连接到特定的地址 
并不是真正的连接，而是锁住DatagramChannel ，让其只能从特定地址收发数据

```java
channel.connect(new InetSocketAddress("jenkov.com", 80));
```
当连接后，也可以使用read()和write()方法，就像在用传统的通道一样。只是在数据传送方面没有任何保证

```java
int bytesRead = channel.read(buf);  
int bytesWritten = channel.write(but); 
```

## 四. Pipe
单向数据连接,Pipe有一个source通道和一个sink通道；
数据会被写到sink通道，从source通道读取；
![](http://dl2.iteye.com/upload/attachment/0096/5625/6094cf4e-cc1f-3f90-a185-5854daf930ec.bmp)

创建管道

```java
Pipe pipe = Pipe.open();
```
写数据,需要访问sink通道

```java
Pipe.SinkChannel sinkChannel = pipe.sink();
```
过调用SinkChannel的write()方法，将数据写入SinkChannel

```java
String newData = "New String to write to file..." + System.currentTimeMillis();
ByteBuffer buf = ByteBuffer.allocate(48);
buf.clear();
buf.put(newData.getBytes());
buf.flip();
while(buf.hasRemaining()) {
    <sinkChannel.write(buf);
}
```
从管道读取数据 
需要访问source通道

```java
Pipe.SourceChannel sourceChannel = pipe.source();
```
read()方法读取

```java
ByteBuffer buf = ByteBuffer.allocate(48);
int bytesRead = inChannel.read(buf);
```