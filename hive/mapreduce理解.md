## hadoop-mapreduce-运行机制
五个阶段
- 输入分片（input split）
- map阶段、combiner阶段
- shuffle阶段
- reduce阶段

**输入分片**
map之前，会计算input split，每个input split一个map task；
input split存储的不是数据本身，是一个分片长度和一个记录数据的位置的数组；
输入分片和block关系紧密，如果block是64mb，输入分片情况如下

```bash
3mb		->1
65mb	->2
127mb	->2
```
如果不合并小文件，会出现5个不均匀的map task；

**map阶段**
map阶段，在本地操作（单独的数据节点上）
每个输入分片会让一个map任务来处理，默认情况下，以HDFS的一个块的大小（默认为64M）为一个分片，当然我们也可以设置块的大小。

**combiner阶段**
可选，本地reduce；
map计算出中间文件前做一个简单的合并重复key值的操作，减少数据量；
combiner根据需要选择，求总数，最大值，最小值可以使用combiner，平均值不可以；

**shuffle阶段**
将map输出作为reduce的输入就是shuffle阶段；
map产生的结果需要写入磁盘，在内存中开启一个环形缓冲，100mB，设置了写入阈值0.8；
输出守护进程，达到阈值会写到磁盘上，称为spill过程；
另外0.2可以继续写入内存，如果满了会阻塞等待；
写入磁盘时，会进行排序如果定义了combiner函数，combiner在排序前进行；

```bash
在写入磁盘之前，线程首先根据reduce任务的数目将数据划分为相同数目的分区，也就是一个reduce任务对应一个分区的数据。这样做是为了避免有些reduce任务分配到大量数据，而有些reduce任务却分到很少数据，甚至没有分到数据的尴尬局面。其实分区就是对数据进行hash的过程。然后对每个分区中的数据进行排序，如果此时设置了Combiner，将排序后的结果进行Combia操作，这样做的目的是让尽可能少的数据写入到磁盘。
```
每次spill会产生溢出文件，有几次spill就有几个文件，map执行完后会合并这些溢出文件；
然后执行Partitioner，Partitioner决定了Reduce的输入分片，根据key和value的值可以控制，做好这里的负载均衡能提高reduce的效率；
Partitioner会找到对应的map文件，通知reduce会开启几个复制线程开启复制线程，默认5；在reduce中进行排序、合并文件操作；

**reduce阶段**
Reduce会接收到不同map任务传来的数据，并且每个map传来的数据都是有序的。如果文件小，写入内存，如果超过阈值，写入溢出文件；
会逐渐合并溢出文件，排序成一个更大的溢出文件；
最后一次合并的结果并没有写入磁盘，而是直接输入到reduce函数；

![](http://dl.iteye.com/upload/attachment/0066/0130/e1090dee-ee98-30d1-ad55-2f88f774fa73.jpg)
![](http://ww3.sinaimg.cn/mw690/005WTVurjw1eoyphmo5cmj30g00bvta7.jpg)