## 一. ORC文件格式
在RCFile的基础上进行了一定的改进，所以与RCFile相比，具有以下一些优势： 
1. ORC中的特定的序列化与反序列化操作可以使ORC file writer根据数据类型进行写出。 
2. 提供了多种RCFile中没有的indexes，这些indexes可以使ORC的reader很快的读到需要的数据，并且跳过无用数据，这使得ORC文件中的数据可以很快的得到访问。 
3. 由于ORC file writer可以根据数据类型进行写出，所以ORC可以支持复杂的数据结构（比如Map等）。 
4. 除了上面三个理论上就具有的优势之外，ORC的具体实现上还有一些其他的优势，比如ORC的stripe默认大小更大，为ORC writer提供了一个memory manager来管理内存使用情况。 
！[](http://img.blog.csdn.net/20160530230000659)

## 二. 数据存储方法
hive表中，记录被横向切分为多个stripes，每个stripe内数据以列为单位存储，所有列的内容保存在一个文件中
stripe默认大小250M,而RCfile最大stripes为4M
Map等复杂字段，会被解析成多个字段，解析如下

```bash
Data type		Chile columns
Array			一个包含所有数组元素的子字段
Map				两个子字段，一个key字段，一个value字段
Struct			每一个属性对应一个子字段
Union			每一个属性对应一个子字段
```
字段类型解析后，由这些字段类型组成一个字段树，只有叶子节点才会保存数据，叶子节点形成一个数据流，如图中的`Data Stream`
为了使reader更高效读取数据，字段的metadata保存在Meta Stream中，字段树中每个叶子节点记录的就是metadata
下图根据表的字段类型，生成字段树
![](http://img.blog.csdn.net/20160530231840815)
Hive-0.13中只支持读取特殊类型字段，不支持只读取特殊类型字段指定部分；
可以使用HDFS每个Block存储一个stripe，stripe一般设置比hdfs block小，如果stripe被拆分到多个block会发生远程访问

## 三. 索引
ORC文件中是稀疏索引
ORC中有两种索引：1.数据统计索引；2.位置指针索引
**数据分析索引**
用这个索引跳过读取不必要的数据
ORC writer生成ORC文件时，会创建这个索引文件，统计信息：数据条数,max, min, sum值；对text和binary类型会记录长度；
复杂数据类型：Array, Map, Struct, Union，子字段类型也会记录这些统计信息
Data Statistics三个级别
1. 文件级别
在ORC文件的末尾会记录文件级别的统计信息，记录整个文件中columns的统计信息，主要用于查询的优化，也可以为一些简单的聚合查询比如max, min, sum输出结果
2. strip级别
会保存每个字段stripe级别的统计信息，`ORC reader`使用这些信息，确定需要读哪些stripe
3. index group级别
逻辑上将一个column的index以一个给定的值，分割为多个组，比如10000为一组，进行统计；
hive查询引擎会将where条件中的约束传递给ORC reader，这些reader根据组级别的统计信息，过滤掉不必要的数据；整个值可以配置
**位置指针索引**
`ORC reader`需要两个位置，才能操作
1. `metadata streams`和`data streams`中每个group开始位置
	每个stripe有多个group,ORC reader需要知道每个group的`metadata streams`和`data streams`开始位置，右边虚线就是这种例子
2. stripes的开始位置 
    一个ORC可能包含多个stripes，一个HDFS block也能包含多个stripes，为了快速定位到stripe的位置，信息保存在`File Footer`中

## 四. 文件压缩
数据流->流式编码器->可选压缩
数据流类型：
- Byte Stream 无编码的字节数据
- Run Length Byte Stream ：保存一系列的字节数据，相同的字节，保存这个重复值以及该值在字节流中出现的位置；
- Integer Stream ：可以对数据量进行字节长度编码以及delta编码，具体根据整形流中的子序列模式确定
- Bit Field Stream ：boolean组成序列，底层`Run Length Byte Stream`实现

**Integer**
对于一个整形字段，会同时使用一个比特流和整形流
比特流用于标识某个值是否为null，整形流用于保存该整形字段非空记录的整数值。 
**String**
检查该字段值中不同的内容数占非空记录总数的百分比不超过0.8，使用字典编码；字段值保存在1个比特流(null值)、1个字节流(字典值)、两个整形流(词条长度、字段值)；
不适用字典编码，说明重复值少，ORC writer会使用一个字节流保存String字段的值，然后用一个整形流来保存每个字段的字节长度。
数据流的底层，用户可自选：ZLIB、Snappy、LZO压缩，编码器会将一个数据流压缩成一个个小单元，默认256KB；
## 五. 内存管理
`ORC writer`写数据时，整个stripe保存在内存中，stripe默认值比较大(250M)，多个`ORC writer`同时写数据，会导致内存不足，需要引入内存管理器；
Map或者Reduce任务中内存管理器会设置一个阈值，限制writer使用的总内存大小，新的writer写数据，会向内存管理器注册大小(stripe的大小)，当内存管理器接收到的总注册大小超过阈值时，
内存管理器会将stripe的实际大小按该writer注册的内存大小与总注册内存大小的比例进行缩小，当有writer关闭时，内存管理器会将其注册的内存从总注册内存中注销。
## 六. 参数

```bash
参数名											默认值				说明
hive.exec.orc.default.stripe.size				256*1024*1024		stripe的默认大小
hive.exec.orc.default.block.size				256*1024*1024		orc文件在文件系统中的默认block大小，从hive-0.14开始
hive.exec.orc.dictionary.key.size.threshold		0.8					String类型字段使用字典编码的阈值
hive.exec.orc.default.row.index.stride			10000				stripe中的分组大小
hive.exec.orc.default.compress					ZLIB				ORC文件的默认压缩方式
hive.exec.orc.skip.corrupt.data					false				遇到错误数据的处理方式，false直接抛出异常，true则跳过该记录
```

## 七.表
表结构如下

```sql
drop table if exists tmp_supplier_offline;
CREATE EXTERNAL TABLE `tmp_supplier_offline` (
	category_id		string,
	product_id		int,
	brand_id		int,
	price			double,
	category_id_2	string
)
LOCATION 'hdfs://qunarcluster/user/qhstats/tmp/tmp_supplier_offline';
```
通过`desc formatted tmp_supplier_offline;`查看表信息如下

```bash
# col_name            	data_type           	comment             
category_id         	string              	                    
product_id          	int                 	                    
brand_id            	int                 	                    
price               	double              	                    
category_id_2       	string              	                    
	 	 
# Detailed Table Information	 	 
Database:           	default             	 
Owner:              	qhstats             	 
CreateTime:         	Wed Nov 02 15:49:20 CST 2016	 
LastAccessTime:     	UNKNOWN             	 
Protect Mode:       	None                	 
Retention:          	0                   	 
Location:           	hdfs://qunarcluster/user/qhstats/tmp/tmp_supplier_offline	 
Table Type:         	EXTERNAL_TABLE      	 
Table Parameters:	 	 
	EXTERNAL            	TRUE                
	transient_lastDdlTime	1478072960          
	 	 
# Storage Information	 	 
SerDe Library:      	org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe	 
InputFormat:        	org.apache.hadoop.mapred.TextInputFormat	 
OutputFormat:       	org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat	 
Compressed:         	No                  	 
Num Buckets:        	-1                  	 
Bucket Columns:     	[]                  	 
Sort Columns:       	[]                  	 
Storage Desc Params:	 	 
	serialization.format	1                   
Time taken: 0.072 seconds, Fetched: 31 row(s)
```
可以根据Location找到hdfs文件
查看文件`rpt_supplier_wrapper_offline_levels_orc`如下

```bash
hive> dfs -ls /user/qhstats/rpt/rpt_supplier_wrapper_offline_levels_orc;
Found 30 items
drwxr-xr-x   - qhstats supergroup          0 2016-10-28 17:36 /user/qhstats/rpt/rpt_supplier_wrapper_offline_levels_orc/dt=20161001
```
## 八.查看dump文件
查看HDFS上ORC文件表格信息
`hive --orcfiledump /user/rpt_supplier_wrapper_offline_levels_orc/000000_0`
内容如下

```bash
File Version: 0.12 with HIVE_13083
Rows: 28
Compression: ZLIB
Compression size: 262144
Type: struct<wrapper_id:string,rate:double>

Stripe Statistics:
  Stripe 1:
    Column 0: count: 28 hasNull: false
    Column 1: count: 28 hasNull: false min: hta2000021a max: wituanctrip sum: 308
    Column 2: count: 28 hasNull: false min: 0.3 max: 0.3 sum: 8.399999999999999

File Statistics:
  Column 0: count: 28 hasNull: false
  Column 1: count: 28 hasNull: false min: hta2000021a max: wituanctrip sum: 308
  Column 2: count: 28 hasNull: false min: 0.3 max: 0.3 sum: 8.399999999999999

Stripes:
    Stripe: offset: 3 data: 133 rows: 28 tail: 46 index: 91
    Stream: column 0 section ROW_INDEX start: 3 length 11
    Stream: column 1 section ROW_INDEX start: 14 length 47
    Stream: column 2 section ROW_INDEX start: 61 length 33
    Stream: column 1 section DATA start: 94 length 112
    Stream: column 1 section LENGTH start: 206 length 7
    Stream: column 2 section DATA start: 213 length 14
    Encoding column 0: DIRECT
    Encoding column 1: DIRECT_V2
    Encoding column 2: DIRECT

File length: 489 bytes
Padding length: 0 bytes
Padding ratio: 0%
```
分为四个部分
**表结构信息**
记录数，压缩方式，压缩大小，以及表结构
**Stripe统计信息**
统计hdfs对应的Stripe信息，包括各个字段的count，min, max, sum信息
Struct只统计count
**File统计信息**
与Stripe信息相同，sum可能不同
**Stripe详细信息**
统计各Stripe的offset，总记录行数等Stripe层次的信息
该Stripe中各字段的Index Data和Row Data，以及每个字段的编码方式
index date统计例子
对于如下表

```bash
字段	类型
category_id		string
product_id		int
brand_id		int
price			double
category_id_2	string
```
![](http://img.blog.csdn.net/20160702220803250)

```bash
起始位置		字段
3……21		STRUCT
22……1141	category_id
1142……3056	product_id
3057……5135	brand_id
5136……7201	price
7202……7938	category_id_2
```
Row Data统计例子

```bash
起始位置				字段				描述
7939……59887			category_id		字段对应词条int流
59888……59898		category_id		词条长度int流
59899……60989		category_id		字典词条数据
60990……3525432		product_id		实际数据int流
3525433……3527085	brand_id		标识IF NULL的byte流
3527086……5708142	brand_id		实际数据int流
5708143……7855016	price			double类型
7855017……7855212	category_id_2	字段对应词条int流
7855213……7855219	category_id_2	词条长度int流
7855220……7855289	category_id_2	字典词条数据
```

每个字段存储方式如下

```bash
字段				类型			存储方式
STRUCT						DIRECT
category_id		String		DICTIONARY_V2
product_id		Int			DIRECT_V2
brand_id		Int			DIRECT_V2
price			Double		DIRECT
category_id_2	String		DICTIONARY_V2
```