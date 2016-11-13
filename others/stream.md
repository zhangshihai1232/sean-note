===========
title: stream
categories: other 
===========

## java1
public static void main

## 一. 为何需要Stream
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

## 二. stream
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
**流的构成**
使用流的步骤：获取一个数据源（source）→ 数据转换→执行操作获取想要的结果
每次转换，原有的Stream对象不改变，返回一个新的Stream对象
**Stream Source获取方式**
从 Collection 和数组
- Collection.stream()
- Collection.parallelStream()
- Arrays.stream(T array) or Stream.of()
从 BufferedReader
- java.io.BufferedReader.lines()
静态工厂
java.util.stream.IntStream.range()
java.nio.file.Files.walk()
自己构建
java.util.Spliterator
其它
- Random.ints()
- BitSet.stream()
- Pattern.splitAsStream(java.lang.CharSequence)
- JarFile.stream()

**流的操作类型**
第一种：Intermediate

```bash
目的主要是打开流,做出某种程度的数据映射/过滤，返回新的流交给下一个操作使用，lazy的
```
第二种：Terminal

```bash
一个流只能有一个Terminal操作，之后流就被使用光了，这是最后一个操作，Terminal才会遍历，产生结果
```
由于转换操作实lazy的，多个转换在Terminal操作时才融合起来，一次循环完成；
Stream有个操作函数集合，每次转换就是把操作放到集合，Terminal的时候才执行；
第三种：short-circuiting
对于intermediate操作，如果接受的是无限大的Stream，但返回一个有限的新的Stream
terminal操作，如果接受无限大的stream,能在有限的时间计算出结果
操作无限大Stream时，希望在有限的时间内完成，需要有一个`short-circuiting`操作
例子：

```java
int sum = widgets
		.stream()
		.filter(w -> w.getColor() == RED)
 		.mapToInt(w -> w.getWeight())
 		.sum();
```
先获得widgets的source,filter 和 mapToInt 为 intermediate 操作,最后一个sum为terminal操作；


## 三. 流的构造与转换
流的操作就是简单的filter-map-reduce过程

```java
// 1. Individual values
Stream stream = Stream.of("a", "b", "c");
// 2. Arrays
String [] strArray = new String[] {"a", "b", "c"};
stream = Stream.of(strArray);
stream = Arrays.stream(strArray);
// 3. Collections
List<String> list = Arrays.asList(strArray);
stream = list.stream();
```
基本数据类型，有三种对应的包装类型stream：IntStream、LongStream、DoubleStream；
使用 Stream<Integer>、Stream<Long> >、Stream<Double>在boxing和unboxing时很耗时；
数值流的构造

```java
IntStream.of(new int[]{1, 2, 3}).forEach(System.out::println);
IntStream.range(1, 3).forEach(System.out::println);
IntStream.rangeClosed(1, 3).forEach(System.out::println);
```
流转换为其它数据结构

```java
// 1. Array
String[] strArray1 = stream.toArray(String[]::new);
// 2. Collection
List<String> list1 = stream.collect(Collectors.toList());
List<String> list2 = stream.collect(Collectors.toCollection(ArrayList::new));
Set set1 = stream.collect(Collectors.toSet());
Stack stack1 = stream.collect(Collectors.toCollection(Stack::new));
// 3. String
String str = stream.collect(Collectors.joining()).toString();
```

## 四. 基本构造过程
Stream.of

```java
public static<T> Stream<T> of(T... values) {
    return Arrays.stream(values);
}
```
Arrays的stream方法

```java
public static <T> Stream<T> stream(T[] array) {
    return stream(array, 0, array.length);
}
```
带参数的stream方法startInclusive，endExclusive是index的起始和结束
利用StreamSupport的stream方法时，首先调用了一个spliterator的构造函数，false代表不进行并行模式

```java
public static <T> Stream<T> stream(T[] array, int startInclusive, int endExclusive) {
    return StreamSupport.stream(spliterator(array, startInclusive, endExclusive), false);
}
```
StreamSupport的stream方法，返回一个ReferencePipeline的内部类Head

```java
public static <T> Stream<T> stream(Spliterator<T> spliterator, boolean parallel) {
    Objects.requireNonNull(spliterator);
    return new ReferencePipeline.Head<>(spliterator,
                                        StreamOpFlag.fromCharacteristics(spliterator),
                                        parallel);
}
```
head的构造

```java
Head(Spliterator<?> source,
    int sourceFlags, boolean parallel) {
    super(source, sourceFlags, parallel);
}
```
Head的父类似是ReferencePipeline

```java
ReferencePipeline(Supplier<? extends Spliterator<?>> source,
     		int sourceFlags, boolean parallel) {
	super(source, sourceFlags, parallel);
}
```
Head->ReferencePipeline->AbstractPipeline->BaseStream
                       ->Stream
返回的Head是ReferencePipeline，它就是Stream

**spliterator**
构造

```java
public static <T> Spliterator<T> spliterator(T[] array, int startInclusive, int endExclusive) {
    return Spliterators.spliterator(array, startInclusive, endExclusive,
                                    Spliterator.ORDERED | Spliterator.IMMUTABLE);
}
```
继续构造的是ArraySpliterator

```java
public static <T> Spliterator<T> spliterator(Object[] array, int fromIndex, int toIndex,
                                             int additionalCharacteristics) {
    checkFromToBounds(Objects.requireNonNull(array).length, fromIndex, toIndex);
    return new ArraySpliterator<>(array, fromIndex, toIndex, additionalCharacteristics);
}
```
总结：

```bash
集合的stream方法
Stream.of
Arrays.stream方法
```

## 五. IntStream构造过程

```java
IntStream.of(new int[]{1, 2, 3}).forEach(System.out::println);
IntStream.range(1, 3).forEach(System.out::println);
IntStream.rangeClosed(1, 3).forEach(System.out::println);
```
这三种方法，构造的都是IntPipeline就是一个HEAD
foreach方法传入IntConsumer

```java
public void forEach(IntConsumer action) {
    evaluate(ForEachOps.makeInt(action, false));
}
```
Head的forEach方法，会执行Spliterator的forEachRemaining方法
创建的是IntArraySpliterator

```java
public void forEach(IntConsumer action) {
    if (!isParallel()) {
        adapt(sourceStageSpliterator()).forEachRemaining(action);
    }
    else {
        super.forEach(action);
    }
}
```
回顾构造IntArraySpliterator时，把最初的array传入，由IntArraySpliterator内部维持

```java
public IntArraySpliterator(int[] array, int origin, int fence, int additionalCharacteristics) {
    this.array = array;
    this.index = origin;
    this.fence = fence;
    this.characteristics = additionalCharacteristics | Spliterator.SIZED | Spliterator.SUBSIZED;
}
```
index是起始index
fence是传入的截止index
这里是通过数组索引的方式便利了这个array

```java
public void forEachRemaining(IntConsumer action) {
    int[] a; int i, hi; // hoist accesses and checks from loop
    if (action == null)
        throw new NullPointerException();
    if ((a = array).length >= (hi = fence) &&
        (i = index) >= 0 && i < (index = hi)) {
        do { action.accept(a[i]); } while (++i < hi);
    }
}
```

## 六. stream转换为其他数据结构过程
例子

```java
Stream<String> stream = Stream.of("a", "b", "c");
String[] strArray1 = stream.toArray(String[]::new);
// 2. Collection
List<String> list1 = stream.collect(Collectors.toList());
List<String> list2 = stream.collect(Collectors.toCollection(ArrayList::new));
Set set1 = stream.collect(Collectors.toSet());
Stack stack1 = stream.collect(Collectors.toCollection(Stack::new));
// 3. String
String str = stream.collect(Collectors.joining()).toString();
```
`Stream.of`方法构造的是`ReferencePipeline.Head`
toArray方法返回值是一个数组

```java
public final <A> A[] toArray(IntFunction<A[]> generator) {
    @SuppressWarnings("rawtypes")
    IntFunction rawGenerator = (IntFunction) generator;
    return (A[]) Nodes.flatten(evaluateToArrayNode(rawGenerator), rawGenerator)
                          .asArray(rawGenerator);
}
```
IntFunction是专门提供给Int的Function转换函数

```java
public interface IntFunction<R> {
    R apply(int value);
}
```
evaluateToArrayNode

```java
   final Node<E_OUT> evaluateToArrayNode(IntFunction<E_OUT[]> generator) {
        if (linkedOrConsumed)
            throw new IllegalStateException(MSG_STREAM_LINKED);
		//只能消耗一次
        linkedOrConsumed = true;
        if (isParallel() && previousStage != null && opIsStateful()) {
            depth = 0;
            return opEvaluateParallel(previousStage, previousStage.sourceSpliterator(0), generator);
        }
        else {
            return evaluate(sourceSpliterator(0), true, generator);
        }
    }
```
从pipeline阶段获得`source spliterator`
对于非并行的pipline这是一个spliterator
从sourceStage的sourceSpliterator或者sourceSupplier获取到spliterator

```java
   private Spliterator<?> sourceSpliterator(int terminalFlags) {
        Spliterator<?> spliterator = null;
        if (sourceStage.sourceSpliterator != null) {
            spliterator = sourceStage.sourceSpliterator;
            sourceStage.sourceSpliterator = null;
        }
        else if (sourceStage.sourceSupplier != null) {
            spliterator = (Spliterator<?>) sourceStage.sourceSupplier.get();
            sourceStage.sourceSupplier = null;
        }
        else {
            throw new IllegalStateException(MSG_CONSUMED);
        }

        if (isParallel() && sourceStage.sourceAnyStateful) {
            int depth = 1;
            for (@SuppressWarnings("rawtypes") AbstractPipeline u = sourceStage, p = sourceStage.nextStage, e = this;
                 u != e;
                 u = p, p = p.nextStage) {

                int thisOpFlags = p.sourceOrOpFlags;
                if (p.opIsStateful()) {
                    depth = 0;

                    if (StreamOpFlag.SHORT_CIRCUIT.isKnown(thisOpFlags)) {
                        thisOpFlags = thisOpFlags & ~StreamOpFlag.IS_SHORT_CIRCUIT;
                    }

                    spliterator = p.opEvaluateParallelLazy(u, spliterator);
                    thisOpFlags = spliterator.hasCharacteristics(Spliterator.SIZED)
                            ? (thisOpFlags & ~StreamOpFlag.NOT_SIZED) | StreamOpFlag.IS_SIZED
                            : (thisOpFlags & ~StreamOpFlag.IS_SIZED) | StreamOpFlag.NOT_SIZED;
                }
                p.depth = depth++;
                p.combinedFlags = StreamOpFlag.combineOpFlags(thisOpFlags, u.combinedFlags);
            }
        }
        if (terminalFlags != 0)  {
            combinedFlags = StreamOpFlag.combineOpFlags(terminalFlags, combinedFlags);
        }
        return spliterator;
    }
```
之前的evaluate代码如下

```java
final <P_IN> Node<E_OUT> evaluate(Spliterator<P_IN> spliterator,
                                  boolean flatten,
                                  IntFunction<E_OUT[]> generator) {
    if (isParallel()) {
        // @@@ Optimize if op of this pipeline stage is a stateful op
        return evaluateToNode(this, spliterator, flatten, generator);
    }
    else {
        Node.Builder<E_OUT> nb = makeNodeBuilder(
                exactOutputSizeIfKnown(spliterator), generator);
        return wrapAndCopyInto(nb, spliterator).build();
    }
}
```
flatten遍历所有的数组，传入的generator是数组的构造器

```java
public static <T> Node<T> flatten(Node<T> node, IntFunction<T[]> generator) {
    if (node.getChildCount() > 0) {
        long size = node.count();
        if (size >= MAX_ARRAY_SIZE)
            throw new IllegalArgumentException(BAD_SIZE);
        T[] array = generator.apply((int) size);
        new ToArrayTask.OfRef<>(node, array, 0).invoke();
        return node(array);
    } else {
        return node;
    }
}
```
