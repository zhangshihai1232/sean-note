## 一. 流的操作
三类：Intermediate、Terminal、Short-circuiting
Intermediate

```bash
map (mapToInt, flatMap 等)、 filter、 distinct、 sorted、 peek、 limit、 skip、 parallel、 sequential、 unordered
```
Terminal

```bash
forEach、 forEachOrdered、 toArray、 reduce、 collect、 min、 max、 count、 anyMatch、 allMatch、 noneMatch、 findFirst、 findAny、 iterator
```
Short-circuiting

```bash
anyMatch、 allMatch、 noneMatch、 findFirst、 findAny、 limit
```
## 二. 典型用法
**map/flatMap**
map负责一对一的映射
把input Stream的每个元素，映射成output Stream的另一个元素
转换大小写
```java
List<String> output = wordList.stream().
map(String::toUpperCase).
collect(Collectors.toList());
```
平方数
```java
List<Integer> nums = Arrays.asList(1, 2, 3, 4);
List<Integer> squareNums = nums.stream().
map(n -> n * n).
collect(Collectors.toList());
```
flatMap负责一对多的映射
```java
Stream<List<Integer>> inputStream = Stream.of(
 Arrays.asList(1),
 Arrays.asList(2, 3),
 Arrays.asList(4, 5, 6)
 );
Stream<Integer> outputStream = inputStream.
flatMap((childList) -> childList.stream());
```
**filter**

**forEach**
**findFirst**
**reduce**
**limit/skip**

**sorted**
**min/max/distinct**
**Match**

## 五. 自己生成流
**Stream.generate**
**Stream.iterate**

## 六. 用 Collectors 来进行 reduction 操作
**groupingBy/partitioningBy**
http://www.ibm.com/developerworks/cn/java/j-lo-java8streamapi/