## 一. 字符串
字符串赋予常量，两种形式

```scala
var greeting = "Hello World!";			//自动推断
var greeting:String = "Hello World!";
```
实际为java的String，本身没有String类，不可变
如果需要修改，创建StringBuilder

```scala
	val buf = new StringBuilder;
	buf += 'a'
	buf ++= "bcdef"
	println( "buf is : " + buf.toString );
```
字符串长度palindrome.length()``
拼接:`string1.concat(string2);`或者+号
创建格式化字符串

```scala
var fs = printf("浮点型变量为 " +
           "%f, 整型变量为 %d, 字符串为 " +
           " %s", floatVar, intVar, stringVar)
```
java.lang.String中字符串常用方法，在scala中可以使用

## 二. 数组
从0开始
声明

```scala
var z:Array[String] = new Array[String](3)
var z = new Array[String](3)
var z = Array("Runoob", "Baidu", "Google")
```
访问

```scala
z(0) = "Runoob"; z(1) = "Baidu"; z(4/2) = "Google"
```
处理

```scala
object Test {
   def main(args: Array[String]) {
      var myList = Array(1.9, 2.9, 3.4, 3.5)
      
      // 输出所有数组元素
      for ( x <- myList ) {
         println( x )
      }

      // 计算数组所有元素的总会
      var total = 0.0;
      for ( i <- 0 to (myList.length - 1)) {
         total += myList(i);
      }
      println("总和为 " + total);

      // 查找数组中的最大元素
      var max = myList(0);
      for ( i <- 1 to (myList.length - 1) ) {
         if (myList(i) > max) max = myList(i);
      }
      println("最大值为 " + max);
    
   }
}
```
多维数组
二维

```scala
var myMatrix = ofDim[Int](3,3)
```
二维数组处理完整例子

```scala
object Test {
   def main(args: Array[String]) {
      var myMatrix = ofDim[Int](3,3)
      
      // 创建矩阵
      for (i <- 0 to 2) {
         for ( j <- 0 to 2) {
            myMatrix(i)(j) = j;
         }
      }
      
      // 打印二维阵列
      for (i <- 0 to 2) {
         for ( j <- 0 to 2) {
            print(" " + myMatrix(i)(j));
         }
         println();
      }
    
   }
}
```
合并数组concat()方法，接收多个数组参数

```scala
object Test {
   def main(args: Array[String]) {
      var myList1 = Array(1.9, 2.9, 3.4, 3.5)
      var myList2 = Array(8.9, 7.9, 0.4, 1.5)

      var myList3 =  concat( myList1, myList2)
      
      // 输出所有数组元素
      for ( x <- myList3 ) {
         println( x )
      }
   }
}
```
区间数组

```scala
import Array._
object Test {
   def main(args: Array[String]) {
      var myList1 = range(10, 20, 2)
      var myList2 = range(10, 20)

      // 输出所有数组元素
      for ( x <- myList1 ) {
         print( " " + x )
      }
      println()
      for ( x <- myList2 ) {
         print( " " + x )
      }
   }
}
```
数组方法,使用数组前引入 import Array._ 包

```scala
//创建指定对象 T 的数组, T 的值可以是 Unit, Double, Float, Long, Int, Char, Short, Byte, Boolean。
def apply( x: T, xs: T* ): Array[T]
//合并数组 
def concat[T]( xss: Array[T]* ): Array[T]
//复制一个数组到另一个数组上。相等于 Java's System.arraycopy(src, srcPos, dest, destPos, length)。 
def copy( src: AnyRef, srcPos: Int, dest: AnyRef, destPos: Int, length: Int ): Unit
//返回长度为 0 的数组
def empty[T]: Array[T]
// 返回指定长度数组，每个数组元素为指定函数的返回值。

def iterate[T]( start: T, len: Int )( f: (T) => T ): Array[T]
//以下实例数组初始值为 0，长度为 3，计算函数为a=>a+1：
scala> Array.iterate(0,3)(a=>a+1)
res1: Array[Int] = Array(0, 1, 2)

//返回数组，长度为第一个参数指定，同时每个元素使用第二个参数进行填充。
def fill[T]( n: Int )(elem: => T): Array[T]

//返回二数组，长度为第一个参数指定，同时每个元素使用第二个参数进行填充。
def fill[T]( n1: Int, n2: Int )( elem: => T ): Array[Array[T]]

//创建指定长度的数组
def ofDim[T]( n1: Int ): Array[T]

//创建二维数组
def ofDim[T]( n1: Int, n2: Int ): Array[Array[T]]

//创建三维数组
def ofDim[T]( n1: Int, n2: Int, n3: Int ): Array[Array[Array[T]]]

//创建指定区间内的数组，step 为每个元素间的步长 
def range( start: Int, end: Int, step: Int ): Array[Int]

//创建指定区间内的数组  
def range( start: Int, end: Int ): Array[Int]

//返回指定长度数组，每个数组元素为指定函数的返回值，默认从 0 开始。  
def tabulate[T]( n: Int )(f: (Int)=> T): Array[T]
//以下实例返回 3 个元素：
scala> Array.tabulate(3)(a => a + 5)
res0: Array[Int] = Array(5, 6, 7)

//返回指定长度的二维数组，每个数组元素为指定函数的返回值，默认从 0 开始。
def tabulate[T]( n1: Int, n2: Int )( f: (Int, Int ) => T): Array[Array[T]]
```

## 三. Collection
两种：可变、不可变
不可变集合的操作会返回新的集合
- List
- Set
- Map
- 元组
- Option
- Iterator

## 四. List
列表不可变
列表例子

```scala
// 字符串列表
val site: List[String] = List("Runoob", "Google", "Baidu")

// 整型列表
val nums: List[Int] = List(1, 2, 3, 4)

// 空列表
val empty: List[Nothing] = List()

// 二维列表
val dim: List[List[Int]] =
   List(
      List(1, 0, 0),
      List(0, 1, 0),
      List(0, 0, 1)
   )
```
构造列表的两个基本单位是`Nil` 和`::`
Nil也是一张空列表，以上例子可以写成如下

```scala
// 字符串列表
val site = "Runoob" :: ("Google" :: ("Baidu" :: Nil))

// 整型列表
val nums = 1 :: (2 :: (3 :: (4 :: Nil)))

// 空列表
val empty = Nil

// 二维列表
val dim = (1 :: (0 :: (0 :: Nil))) ::
          (0 :: (1 :: (0 :: Nil))) ::
          (0 :: (0 :: (1 :: Nil))) :: Nil
```
基本操作
Scala列表有三个基本操作：
- head 返回列表第一个元素
- tail 返回一个列表，包含除了第一元素之外的其他元素
- isEmpty 在列表为空时返回true

任何操作都可以用这三个操作表达

```scala
object Test {
   def main(args: Array[String]) {
      val site = "Runoob" :: ("Google" :: ("Baidu" :: Nil))
      val nums = Nil
      println( "第一网站是 : " + site.head )
      println( "最后一个网站是 : " + site.tail )
      println( "查看列表 site 是否为空 : " + site.isEmpty )
      println( "查看 nums 是否为空 : " + nums.isEmpty )
   }
}
```
连接列表
可以使用 ::: 运算符或 List.:::() 方法或 List.concat() 方法来连接两个或多个列表

```scala
   def main(args: Array[String]) {
      val site1 = "Runoob" :: ("Google" :: ("Baidu" :: Nil))
      val site2 = "Facebook" :: ("Taobao" :: Nil)

      // 使用 ::: 运算符
      var fruit = site1 ::: site2
      println( "site1 ::: site2 : " + fruit )
      
      // 使用 Set.:::() 方法
      fruit = site1.:::(site2)
      println( "site1.:::(site2) : " + fruit )

      // 使用 concat 方法
      fruit = List.concat(site1, site2)
      println( "List.concat(site1, site2) : " + fruit  )
   }
```
List.fill()创建一个指定重复数量的元素列表

```scala
   def main(args: Array[String]) {
      val site = List.fill(3)("Runoob") // 重复 Runoob 3次
      println( "site : " + site  )
      val num = List.fill(10)(2)         // 重复元素 2, 10 次
      println( "num : " + num  )
   }
```
List.tabulate()
通过给定函数创建列表，第一个参数指定数量，第二个指定函数

```scala
   def main(args: Array[String]) {
      // 通过给定的函数创建 5 个元素
      val squares = List.tabulate(6)(n => n * n)
      println( "一维 : " + squares  )

      // 创建二维列表
      val mul = List.tabulate( 4,5 )( _ * _ )      
      println( "多维 : " + mul  )
   }
```

List.reverse

```scala
   def main(args: Array[String]) {
      val site = "Runoob" :: ("Google" :: ("Baidu" :: Nil))
      println( "site 反转前 : " + site )

      println( "site 反转前 : " + site.reverse )
   }
```

常用方法:非常多，30多个
参考：http://www.runoob.com/scala/scala-lists.html

## 五. Set
默认使用不可变集合，可变需要引用`scala.collection.mutable.Set`包
默认使用`scala.collection.immutable.Set`

```scala
val set = Set(1,2,3)
println(set.getClass.getName)
println(set.exists(_ % 2 == 0)) //true
println(set.drop(1)) 
```
可变集合`scala.collection.mutable.Set`

```scala
import scala.collection.mutable.Set // 可以在任何地方引入 可变集合
val mutableSet = Set(1,2,3)
println(mutableSet.getClass.getName) // scala.collection.mutable.HashSet
mutableSet.add(4)
mutableSet.remove(1)
mutableSet += 5
mutableSet -= 2
println(mutableSet) // Set(5, 3, 4)
val another = mutableSet.toSet
println(another.getClass.getName) // scala.collection.immutable.Set
```
都可以添加和删除元素，但是不可变Set进行操作，会产生一个新的set
可变集合操作与ListBuffer类似

Scala集合有三个基本操作：
- head 返回集合第一个元素
- tail 返回一个集合，包含除了第一元素之外的其他元素
- isEmpty 在集合为空时返回true

集合中任何操作都可以使用这三个基本操作实现

```scala
  def main(args: Array[String]) {
      val site = Set("Runoob", "Google", "Baidu")
      val nums: Set[Int] = Set()
      println( "第一网站是 : " + site.head )
      println( "最后一个网站是 : " + site.tail )
      println( "查看列表 site 是否为空 : " + site.isEmpty )
      println( "查看 nums 是否为空 : " + nums.isEmpty )
   }
```
连接集合
可以使用 ++ 运算符或 Set.++() 方法来连接两个集合
如果元素有重复的就会移除重复的元素

```scala
object Test {
   def main(args: Array[String]) {
      val site1 = Set("Runoob", "Google", "Baidu")
      val site2 = Set("Faceboook", "Taobao")

      // ++ 作为运算符使用
      var site = site1 ++ site2
      println( "site1 ++ site2 : " + site )

      //  ++ 作为方法使用
      site = site1.++(site2)
      println( "site1.++(site2) : " + site )
   }
}
```
查找最大与最小min或max方法

```scala
   def main(args: Array[String]) {
      val num = Set(5,6,9,20,30,45)
      // 查找集合中最大与最小元素
      println( "Set(5,6,9,20,30,45) 集合中的最小元素是 : " + num.min )
      println( "Set(5,6,9,20,30,45) 集合中的最大元素是 : " + num.max )
   }
```
交集
使用 Set.& 方法或 Set.intersect 方法来查看两个集合的交集元素

```scala
 def main(args: Array[String]) {
      val num1 = Set(5,6,9,20,30,45)
      val num2 = Set(50,60,9,20,35,55)

      // 交集
      println( "num1.&(num2) : " + num1.&(num2) )
      println( "num1.intersect(num2) : " + num1.intersect(num2) )
   }
```
site其他方法参考如下：
http://www.runoob.com/scala/scala-sets.html

## 六.map
可变：`mutable.Map`
不可变：直接使用

```scala
// 空哈希表，键为字符串，值为整型
var A:Map[Char,Int] = Map()
// Map 键值对演示
val colors = Map("red" -> "#FF0000", "azure" -> "#F0FFFF")
```
添加键值对

```scala
A += ('I' -> 1)
A += ('J' -> 5)
A += ('K' -> 10)
A += ('L' -> 100)
```
三个基本操作：
- keys	返回 Map 所有的键(key)
- values	返回 Map 所有的值(value)
- isEmpty	在 Map 为空时返回true

三个方法基本应用

```scala
   def main(args: Array[String]) {
      val colors = Map("red" -> "#FF0000",
                       "azure" -> "#F0FFFF",
                       "peru" -> "#CD853F")

      val nums: Map[Int, Int] = Map()

      println( "colors 中的键为 : " + colors.keys )
      println( "colors 中的值为 : " + colors.values )
      println( "检测 colors 是否为空 : " + colors.isEmpty )
      println( "检测 nums 是否为空 : " + nums.isEmpty )
   }
```
合并
使用 ++ 运算符或 Map.++() 方法来连接两个 Map,合并时会移除重复的key

```scala
 def main(args: Array[String]) {
      val colors1 = Map("red" -> "#FF0000",
                        "azure" -> "#F0FFFF",
                        "peru" -> "#CD853F")
      val colors2 = Map("blue" -> "#0033FF",
                        "yellow" -> "#FFFF00",
                        "red" -> "#FF0000")

      //  ++ 作为运算符
      var colors = colors1 ++ colors2
      println( "colors1 ++ colors2 : " + colors )

      //  ++ 作为方法
      colors = colors1.++(colors2)
      println( "colors1.++(colors2)) : " + colors )

   }
```
输出 Map 的 keys 和 values

```scala
  def main(args: Array[String]) {
      val sites = Map("runoob" -> "http://www.runoob.com",
                       "baidu" -> "http://www.baidu.com",
                       "taobao" -> "http://www.taobao.com")

      sites.keys.foreach{ i =>  
                           print( "Key = " + i )
                           println(" Value = " + sites(i) )}
   }
```
Map 中是否存在指定的 Key

```scala
   def main(args: Array[String]) {
      val sites = Map("runoob" -> "http://www.runoob.com",
                       "baidu" -> "http://www.baidu.com",
                       "taobao" -> "http://www.taobao.com")

      if( sites.contains( "runoob" )){
           println("runoob 键存在，对应的值为 :"  + sites("runoob"))
      }else{
           println("runoob 键不存在")
      }
      if( sites.contains( "baidu" )){
           println("baidu 键存在，对应的值为 :"  + sites("baidu"))
      }else{
           println("baidu 键不存在")
      }
      if( sites.contains( "google" )){
           println("google 键存在，对应的值为 :"  + sites("google"))
      }else{
           println("google 键不存在")
      }
   }
```
其他方法参考：http://www.runoob.com/scala/scala-maps.html

## 七. 元组
不可变，类列表，可以包含不同类型元素

```scala
val t = (1, 3.14, "Fred")  
val t = new Tuple3(1, 3.14, "Fred")
```
类型取决于实际的类型，比如`(99, "runoob")`是` Tuple2[Int, String]`
`('u', 'r', "the", 1, 4, "me")`为`Tuple6[Char, Char, String, Int, Int, String]`
最大长度22，更大长度可以使用集合或者扩展元组
访问元组可以通过数组索引

```scala
val t = (4,3,2,1)
```
t._1 访问第一个元素， t._2 访问第二个元素

```scala
   def main(args: Array[String]) {
      val t = (4,3,2,1)
      val sum = t._1 + t._2 + t._3 + t._4
      println( "元素之和为: "  + sum )
   }
```
迭代元组

```scala
val t=("test",1,2)
val productIterator: Iterator[Any] = t.productIterator
for (elem <- productIterator) {
    println(elem)
}
```
元组转为字符串`Tuple.toString()`

```scala
val t = new Tuple3(1, "hello", Console)
println("连接后的字符串为: " + t.toString() )
```
元素交换
Tuple2才能用swap

```scala
def main(args: Array[String]) {
	val t = new Tuple2("www.google.com", "www.runoob.com")
	println("交换后的元组: " + t.swap )
}
```

## 八. Option
用来表示一个值是可选的，有或无
Option[T]是类型为T的可选值容器

```scala
// 虽然 Scala 可以不定义变量的类型，不过为了清楚些，还是把他显示的定义上了
val myMap: Map[String, String] = Map("key1" -> "value")
val value1: Option[String] = myMap.get("key1")
val value2: Option[String] = myMap.get("key2")
 
println(value1) // Some("value1")
println(value2) // None
```
返回值是一个`Option[String]`类型
另一个例子

```scala
object Test {
   def main(args: Array[String]) {
      val sites = Map("runoob" -> "www.runoob.com", "google" -> "www.google.com")
      println("sites.get( \"runoob\" ) : " +  sites.get( "runoob" )) // Some(www.runoob.com)
      println("sites.get( \"baidu\" ) : " +  sites.get( "baidu" ))  //  None
   }
}
```
可以通过模式匹配输出值

```scala
object Test {
   def main(args: Array[String]) {
      val sites = Map("runoob" -> "www.runoob.com", "google" -> "www.google.com")
      
      println("show(sites.get( \"runoob\")) : " +  
                                          show(sites.get( "runoob")) )
      println("show(sites.get( \"baidu\")) : " +  
                                          show(sites.get( "baidu")) )
   }
   
   def show(x: Option[String]) = x match {
      case Some(s) => s
      case None => "?"
   }
}
```
getOrElse
获取元组中存在的元素或使用默认的值

```scala
   def main(args: Array[String]) {
      val a:Option[Int] = Some(5)
      val b:Option[Int] = None 
      
      println("a.getOrElse(0): " + a.getOrElse(0) )
      println("b.getOrElse(10): " + b.getOrElse(10) )
   }
```
如果`a.getOrElse(x)`a这个Optional有值则输出值，没有值则输出x
isEmpty方法
检测元组中的元素是否为none

```scala
   def main(args: Array[String]) {
      val a:Option[Int] = Some(5)
      val b:Option[Int] = None 
      println("a.isEmpty: " + a.isEmpty )
      println("b.isEmpty: " + b.isEmpty )
   }
```
Option中的内容为none则为true
Option 常用方法参考：http://www.runoob.com/scala/scala-options.html

## 九. 迭代器
Iterator是一个访问集合的方法
next和hasnext

```scala
object Test {
   def main(args: Array[String]) {
      val it = Iterator("Baidu", "Google", "Runoob", "Taobao")
      
      while (it.hasNext){
         println(it.next())
      }
   }
}
```
查找最大最小值`it.min`和`it.max`

```scala
  def main(args: Array[String]) {
      val ita = Iterator(20,40,2,50,69, 90)
      val itb = Iterator(20,40,2,50,69, 90)
      
      println("最大元素是：" + ita.max )
      println("最小元素是：" + itb.min )

   }
```
获取迭代器的长度`it.size`或`it.length`

```scala
  def main(args: Array[String]) {
      val ita = Iterator(20,40,2,50,69, 90)
      val itb = Iterator(20,40,2,50,69, 90)
      
      println("ita.size 的值: " + ita.size )
      println("itb.length 的值: " + itb.length )

   }
```
Iterator 常用方法常用方法：http://www.runoob.com/scala/scala-iterators.html