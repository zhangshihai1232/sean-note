## hive sql编译过程
## MapReduce实现基本SQL操作的原理
**Join**

```sql
select u.name, o.orderid 
from order o join user u on o.uid = u.uid;
```
map输出的value,不同的表打标记tag;
reduce阶段根据tag判断来源；
![](http://tech.meituan.com/img/hive/join.png)

**Group By**

```sql
select rank, isonline, count(*) 
from city 
group by rank, isonline;
```
GroupBy字段组合为map输出的key值，MapReduce排序，在reduce阶段保存LastKey区分不同的key
![](http://tech.meituan.com/img/hive/groupby.png)
**Distinct**

```java
select dealid, count(distinct uid) num 
from order group by dealid;
```
只有一个distinct字段，只需要将GroupBy字段和Distinct字段组合为map输出key,利用mapreduce排序
同时将GroupBy字段作为reduce的key，在reduce阶段保存LastKey即可完成去重
![](http://tech.meituan.com/img/hive/1distinct.png)

定义多个distinct字段

```sql
select dealid, count(distinct uid), count(distinct date) 
from order group by dealid;
```
方式1：
使用一个distinct字段的方式，无法跟据uid和date分别排序，无法通过LastKey去重；
![](http://tech.meituan.com/img/hive/2distinct-a.png)
方式2：
对所有的distinct字段编号，每行数据生成n行数据，那么相同字段就会分别排序，这时只需要在reduce阶段记录LastKey即可去重。
这种实现方式很好的利用了MapReduce的排序，节省了reduce阶段去重的内存消耗，但是缺点是增加了shuffle的数据量；
在生成reduce value时，除第一个distinct字段所在行需要保留value值，其余distinct数据行value字段均可为空。
![](http://tech.meituan.com/img/hive/2distinct-b.png)

**SQL转化为MapReduce的过程**
6个阶段
- Antlr定义SQL的语法规则，完成SQL词法，语法解析，将SQL转化为抽象语法树AST Tree
- 遍历AST Tree，抽象出查询的基本组成单元QueryBlock
- 遍历QueryBlock，翻译为执行操作树OperatorTree
- 逻辑层优化器进行OperatorTree变换，合并不必要的ReduceSinkOperator，减少shuffle数据量
- 遍历OperatorTree，翻译为MapReduce任务
- 物理层优化器进行MapReduce任务的变换，生成最终的执行计划

## SQL语法解析
Antlr使用：编写一个语法文件，定义词法和语法替换规则即可
完成了：词法分析、语法分析、语义分析、中间代码生成的过程
hive 0.11有四个语法规则文件：SelectClauseParser.g，FromClauseParser.g，IdentifiersParser.g，HiveParser.g

**抽象语法树AST Tree**
在语法分析的同时将输入语句转换成抽象语法树，后续在遍历语法树时完成进一步的处理
hive的SelectStatement包含：select, from, where, groupby, having, orderby等子句
箭头表示对于原语句的改写，改写后加入一些特殊词表示特定语法，比如TOK_QUERY表示一个查询块
```bash
selectStatement
   :
   selectClause
   fromClause
   whereClause?
   groupByClause?
   havingClause?
   orderByClause?
   clusterByClause?
   distributeByClause?
   sortByClause?
   limitClause? -> ^(TOK_QUERY fromClause ^(TOK_INSERT ^(TOK_DESTINATION ^(TOK_DIR TOK_TMP_FILE))
                     selectClause whereClause? groupByClause? havingClause? orderByClause? clusterByClause?
                     distributeByClause? sortByClause? limitClause?))
   ;
```

**样例SQL**
```sql
FROM
( 
  SELECT
    p.datekey datekey,
    p.userid userid,
    c.clienttype
  FROM
    detail.usersequence_client c
    JOIN fact.orderpayment p ON p.orderid = c.orderid
    JOIN default.user du ON du.userid = p.userid
  WHERE p.datekey = 20131118 
) base
INSERT OVERWRITE TABLE `test`.`customer_kpi`
SELECT
  base.datekey,
  base.clienttype,
  count(distinct base.userid) buyer_count
GROUP BY base.datekey, base.clienttype
```

**SQL生成AST Tree**
Antlr对hql的解析如下：HiveLexerX、HiveParser分别是Antlr对语法文件Hive.g编译后自动生成的词法解析和语法解析类；
```java
HiveLexerX lexer = new HiveLexerX(new ANTLRNoCaseStringStream(command));    //词法解析，忽略关键词的大小写
TokenRewriteStream tokens = new TokenRewriteStream(lexer);
if (ctx != null) {
  ctx.setTokenRewriteStream(tokens);
}
HiveParser parser = new HiveParser(tokens);                                 //语法解析
parser.setTreeAdaptor(adaptor);
HiveParser.statement_return r = null;
try {
  r = parser.statement();                                                   //转化为AST Tree
} catch (RecognitionException e) {
  e.printStackTrace();
  throw new ParseException(parser.errors);
}
```

生成的`AST Tree`如图右侧，查询1/2，分别对应右侧第1/2两个部分；
![](http://static.open-open.com/lib/uploadImg/20140521/20140521115153_66.png)
内层子查询也会生成一个TOK_DESTINATION节点,语法改写中特意 增加了的一个节点,原因是Hive中所有查询的数据均会保存在HDFS临时的文件中，无论是中间的子查询还是查询最终的结果，Insert语句最终会将数 据写入表所在的HDFS目录下。
将内存子查询的from字句展开后，得到如下`AST Tree`，每个表生成一个`TOK_TABREF`节点，join条件生成一个`=`节点
其他条件相似
![](http://static.open-open.com/lib/uploadImg/20140521/20140521115153_420.png)


http://www.open-open.com/lib/view/open1400644430159.html