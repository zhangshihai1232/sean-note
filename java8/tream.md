## 为何需要Stream
Stream是对Collection的增强，集合对象的聚合/bulk操作，并提供串行和并行两种方式进行聚集；
并行处理时，不需要编写多线程代码；
**聚合操作例子**
- 客户每月平均消费金额
- 最昂贵的在售商品
- 本周完成的有效订单（排除了无效的）
- 取十个数据样本作为首页推荐

RDBMS提供这些，但是脱离了之后，或者数据量比较大，Iterator非常低效；
**java7例子**
发现type为grocery 的所有交易，然后返回以交易值降序排序好的交易ID集合
```java
List<Transaction> groceryTransactions = new Arraylist<>();
for (Transaction t : transactions) {
    if (t.getType() == Transaction.GROCERY) {
        groceryTransactions.add(t);
    }
}
Collections.sort(groceryTransactions, new Comparator() {
    public int compare(Transaction t1, Transaction t2) {
        return t2.getValue().compareTo(t1.getValue());
    }
});
List<Integer> transactionIds = new ArrayList<>();
for (Transaction t : groceryTransactions) {
    transactionsIds.add(t.getId());
}
```
**java8 stream方式**
```java
List<Integer> transactionsIds = 
transactions.parallelStream()
			.filter(t -> t.getType() == Transaction.GROCERY)
			.sorted(comparing(Transaction::getValue)
			.reversed())
			.map(Transaction::getId)
			.collect(toList());
```

## stream
不保存数据，是一种计算方式，高级版本的Iterator，数据只能遍历一次
串行和迭代器类似
并行操作，数据分段，每一段在不同的线程中执行，统一输出；
依赖于java7的Fork/Join框架，拆分任务和加速处理；
并行API演变
```bash
1.0-1.4 	中的 java.lang.Thread
5.0			中的 java.util.concurrent
6.0 		中的 Phasers 等
7.0 		中的 Fork/Join 框架
8.0 		中的 Lambda
```
Stream 的另外一大特点是，数据源本身可以是无限的


http://www.ibm.com/developerworks/cn/java/j-lo-java8streamapi/