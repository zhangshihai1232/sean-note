## NIO
提供了与标准IO不同的工作方式：
- Channels and Buffers（通道和缓冲区）：标准的IO基于字节流和字符流进行操作的，而NIO是基于通道（Channel）和缓冲区（Buffer）进行操作，数据总是从通道读取到缓冲区中，或者从缓冲区写入到通道中。
- Asynchronous IO（异步IO）：Java NIO可以让你异步的使用IO，例如：当线程从通道读取数据到缓冲区时，线程还是可以进行其他事情。当数据被写入到缓冲区时，线程可以继续处理它。从缓冲区写入通道也类似。
- Selectors（选择器）：Java NIO引入了选择器的概念，选择器用于监听多个通道的事件（比如：连接打开，数据到达）。因此，单个的线程可以监听多个数据通道。
提供了三个核心组件：Channels、Buffers、Selectors

## 一. Channels和Buffer
Channels和Buffer是成对出现的，数据可以从Channel读到Buffer中，也可以从Buffer写到Channel中；
![](http://dl2.iteye.com/upload/attachment/0096/3970/e20c73df-9ade-3121-be5f-307e6baf328f.png)
主要的Channel：FileChannel、DatagramChannel、SocketChannel、ServerSocketChannel
可见，这些通道涵盖了文件和网络
关键的Buffer：ByteBuffer、CharBuffer、DoubleBuffer、FloatBuffer、IntBuffer、LongBuffer、ShortBuffer
涵盖了IO收发的基本数据类型；
NIO中还包括Mappedyteuffer，表示内存映射文件；

## 二. Selector
Selector允许单线程处理多个 Channel;
打开了多个连接，每个连接的流量比较低，使用Selector就会很方便。
一个Selector处理3个Channel的图
![](http://dl2.iteye.com/upload/attachment/0096/3972/79224e12-3615-3917-9e85-42e7edbd8b40.png)
使用Selector，需要向Selector中注册Channel,然后调用select()方法；
这个方法会一直阻塞到某个注册的通道有事件就绪。
一旦这个方法返回，线程就可以处理这些事件，事件的例子有如新连接进来，数据接收等。 

对于操作系统来说，线程之间上下文切换的开销很大，而且每个线程都要占用系统的一些资源（如内存）。因此，使用的线程越少越好。selector能处理多个通道；

```java
Selector selector = Selector.open();
```
执行的操作如下

```java
    public static Selector open() throws IOException {
        return SelectorProvider.provider().openSelector();
    }
```
SelectorProvider的provider根据配置，调用具体SelectorProvider的create方法，创建一个provider，最后返回

```java
     public static SelectorProvider provider() {
        synchronized (lock) {
            if (provider != null)
                return provider;
            return AccessController.doPrivileged(
                new PrivilegedAction<SelectorProvider>() {
                    public SelectorProvider run() {
                            if (loadProviderFromProperty())
                                return provider;
                            if (loadProviderAsService())
                                return provider;
                            provider = sun.nio.ch.DefaultSelectorProvider.create();
                            return provider;
                        }
                    });
        }
    }
```
创建的这个provider根据平台区分，都继承自SelectorProviderImpl

```java
public class WindowsSelectorProvider extends SelectorProviderImpl {
    public WindowsSelectorProvider() {
    }
    public AbstractSelector openSelector() throws IOException {
        return new WindowsSelectorImpl(this);
    }
}
```

## 三. NIO和IO的主要区别

|io|nio|
|-|-|
|Stream oriented|Buffer oriented|
|Blocking IO|Non blocking IO|
| 	|Selectors|

IO是面向流的

```bash
Java IO面向流意味着每次从流中读一个或多个字节，直至读取所有字节，它们没有被缓存在任何地方。
不能前后移动流中的数据,如果需要前后移动从流中读取的数据，需要先将它缓存到一个缓冲区。
```
NIO是面向缓冲区的

```bash
数据读取到一个它稍后处理的缓冲区，需要时可在缓冲区中前后移动。
还需要检查是否该缓冲区中包含所有您需要处理的数据。而且，需确保当更多的数据读入缓冲区时，不要覆盖缓冲区里尚未处理的数据。 
```
Java IO的各种流是阻塞的

```bash
当一个线程调用read() 或 write()时，该线程被阻塞，直到有一些数据被读取，或数据完全写入。该线程在此期间不能再干任何事情了
```
Java NIO的非阻塞模式

```bash
使一个线程从某通道发送请求读取数据，但是它仅能得到目前可用的数据，如果目前没有数据可用时，就什么都不会获取。而不是保持线程阻塞，所以直至数据变的可以读取之前，该线程可以继续做其他的事情。 
线程通常将非阻塞IO的空闲时间用于在其它通道上执行IO操作，所以一个单独的线程现在可以管理多个输入和输出通道
```
Selectors

```bash
Java NIO的选择器允许一个单独的线程来监视多个输入通道
你可以注册多个通道使用一个选择器，然后使用一个单独的线程来“选择”通道
这些通道里已经有可以处理的输入，或者选择已准备写入的通道
```
影响方面
- 对NIO或IO类的API调用
- 数据处理
- 用来处理数据的线程数

## 五. API调用 
io从流中读取，比如代码

```java
InputStream input = … ; // get the InputStream from the client socket  
BufferedReader reader = new BufferedReader(new InputStreamReader(input));  
  
String nameLine   = reader.readLine();  
String ageLine    = reader.readLine();  
String emailLine  = reader.readLine();  
String phoneLine  = reader.readLine(); 
```
读取过不能回退
![](http://dl2.iteye.com/upload/attachment/0096/5635/d816b6e7-0b89-3cbf-bc24-dc7e1bb971de.png)
nio实现

```java
ByteBuffer buffer = ByteBuffer.allocate(48);  
int bytesRead = inChannel.read(buffer);  
```
数据从通道读入ByteBuffer，方法调用返回时，你不知道你所需的所有数据是否在缓冲区内
如果使用检查的方式，会杂乱不堪

```java
ByteBuffer buffer = ByteBuffer.allocate(48);  
int bytesRead = inChannel.read(buffer);  
while(! bufferFull(bytesRead) ) {  
bytesRead = inChannel.read(buffer);  
}  
```
循环等待的过程
![](http://dl2.iteye.com/upload/attachment/0096/5637/e97ec9e9-62d4-3375-80a6-d4238d6a0664.png)
如果需要管理同时打开的成千上万个连接，这些连接每次只是发送少量的数据，nio更适合
如果你有少量的连接使用非常高的带宽，一次发送大量的数据，也许典型的IO服务器实现可能非常契合