## 一. 运算符
**算术运算符**
+ - * / %
**关系运算符**
== != > < >= <=
**逻辑运算符**
&& || !
**位运算符**
~ & | ^ << >> >>>
**赋值运算符**
= += -= *= /= %= << >> &= ^= |=
**运算符优先级**
- 指针最优，单目运算优于双目运算。如正负号。
- 先乘除（模），后加减。
- 先算术运算，后移位运算，最后位运算。请特别注意：1 << 3 + 2 & 7 等价于 (1 << (3 + 2))&7
- 逻辑运算最后计算

```bash
运算符类型		运算符								结合方向
表达式运算		() [] . expr++ expr--				左到右
一元运算符	    * & + - ! ~ ++expr --expr
				* / %
				+ -									右到左
				>> <<
				< > <= >=
				== !=
位运算符			&
				^
				|									左到右
				&&
				||
三元运算符		?:									右到左
赋值运算符		= += -= *= /= %= >>= <<= &= ^= |=	右到左
逗号				,									左到右
```

## 二. 条件语句

```java
if(布尔表达式 1){
   ...
}else if(布尔表达式 2){
   ...
}else if(布尔表达式 3){
   ...
}else {
   ...
}
```

## 三. 循环
**while**
i to j(包含j)

```java
for( a <- 1 to 10){
	println( "Value of a: " + a );
}
```
i until j(不包含j)

```java
for( a <- 1 until 10){
	println( "Value of a: " + a );
}
```
**do...while**
**for**
使用;设置多个区间，迭代所有可能

```java
for( a <- 1 to 3; b <- 1 to 3){
	println( "Value of a: " + a );
	println( "Value of b: " + b );
}
```
循环集合

```java
for( var x <- List ){
	statement(s);
}
```
List集合

```java
val numList = List(1,2,3,4,5,6);
for( a <- numList ){
	println( "Value of a: " + a );
}
```
循环过滤，使用多个语句过滤，条件满足的会执行语句

```java
for( var x <- List
	if condition1; if condition2...
){
statement(s);
```
例子

```java
val numList = List(1,2,3,4,5,6,7,8,9,10);
// for 循环
for( a <- numList
	if a != 3; if a < 8 ){
	println( "Value of a: " + a );
}
```
for使用yield
可以将for魂环的返回值作为一个变量存储,for中的yield会将不同的元素存储成list

```java
var retVal = for{ var x <- List
	if condition1; if condition2...
}yield x
```
例子

```java
val numList = List(1,2,3,4,5,6,7,8,9,10);
// for 循环
var retVal = for{ a <- numList 
                if a != 3; if a < 8
              }yield a

// 输出返回值
for( a <- retVal){
 println( "Value of a: " + a );
}
```
**break**
break有些不一样

```java
// 创建 Breaks 对象
val loop = new Breaks;
// 在 breakable 中循环
loop.breakable{
    // 循环
    for(...){
       ....
       // 循环中断
       loop.break;
   }
}
```
嵌套循环中使用

```java
var a = 0;
var b = 0;
val numList1 = List(1,2,3,4,5);
val numList2 = List(11,12,13);
val outer = new Breaks;
val inner = new Breaks;

outer.breakable {
	for( a <- numList1){
		println( "Value of a: " + a );
		inner.breakable {
		   for( b <- numList2){
		      println( "Value of b: " + b );
		      if( b == 12 ){
		         inner.break;
		      }
		   }
		} // 内嵌循环中断
	}
} // 外部循环中断
```

## 四. 函数
**定义**

```java
def addInt( a:Int, b:Int ) : Int = {
	var sum:Int = 0
	sum = a + b
	return sum
}
```
如果没有返回值，可以返回Unit，类似java中的void

```java
object Hello{
   def printMe( ) : Unit = {
      println("Hello, Scala!")
   }
}
```

**调用**
标准格式:`functionName( 参数列表 )`
对象来调用:`[instance.]functionName( 参数列表 )`
例子：

```java
object Test {
   def main(args: Array[String]) {
        println( "Returned Value : " + addInt(5,7) );
   }
   def addInt( a:Int, b:Int ) : Int = {
      var sum:Int = 0
      sum = a + b

      return sum
   }
}
```

**call by name**
在解析函数参数，两种方式
- 传值调用：先计算参数表达式的值，再应用到函数内部
- 传名调用：将未计算的参数表达式直接应用到函数内部

这样，每次使用传名调用，解释器都会计算一次表达式的值

```java
object Test {
	def main(args: Array[String]) {
		delayed(time());
	}
	def time() = {
		println("获取时间，单位为纳秒")
		System.nanoTime
	}
	def delayed( t: => Long ) = {
		println("在 delayed 方法内")
		println("参数： " + t)
		t
	}
}
```
例子中声明了delayed方法，在变量名和变量类型使用=> 符号来设置传名调用;

**指定参数名**
用的时候，可以指定参数名，不必按照顺序输出

```java
   def main(args: Array[String]) {
        printInt(b=5, a=7);
   }
   def printInt( a:Int, b:Int ) = {
      println("Value of a : " + a );
      println("Value of b : " + b );
   }
```

**可变参数**
可变参数，传入的是数组

```java
	def main(args: Array[String]) {
	    printStrings("Runoob", "Scala", "Python");
	}
	def printStrings( args:String* ) = {
	  var i : Int = 0;
	  for( arg <- args ){
	     println("Arg value[" + i + "] = " + arg );
	     i = i + 1;
	  }
	}
```

**递归函数**
计算阶乘

```java
object Test {
   def main(args: Array[String]) {
      for (i <- 1 to 10)
         println(i + " 的阶乘为: = " + factorial(i) )
   }
   
   def factorial(n: BigInt): BigInt = {  
      if (n <= 1)
         1  
      else    
      n * factorial(n - 1)
   }
}
```

**默认参数值**
不传参按照默认值，传参按照参数传

```java
object Test {
   def main(args: Array[String]) {
        println( "返回值 : " + addInt() );
   }
   def addInt( a:Int=5, b:Int=7 ) : Int = {
      var sum:Int = 0
      sum = a + b

      return sum
   }
}
```

## 五. 高阶函数
在数学和计算机科学中，高阶函数是至少满足下列一个条件的函数:
- 接受一个或多个函数作为输入
- 输出一个函数

例子：
假设有一个函数对给定两个数区间中的所有整数求和

```scala
def sumInts(a: Int, b: Int): Int = 
  if(a > b) 0 else a + sumInts(a + 1, b)
```
如果现在要求连续整数的平方和：

```scala
def square(x: Int): Int = x * x
def sumSquares(a: Int, b: Int): Int = 
  if(a > b) 0 else square(a) + sumSquares(a + 1, b)
```

如果要计算2的幂次的和：

```scala
def powerOfTwo(x: Int): Int = if(x == 0) 1 else 2 * powerOfTwo(x-1)
def sumPowersOfTwo(a: Int, b: Int): Int = 
  if(a > b) 0 else powerOfTwo(a) + sumPowersOfTwo(a+1, b)
```
上面的函数都是从a到b的f(n)的累加形式，我们可以抽取这些函数中共同的部分重新编写函数sum，其中定义的f作为一个参数传入到高阶函数sum中：

```scala
def sum(f: Int => Int, a: Int, b: Int): Int = 
  if(a > b) 0 else f(a) + sum(f, a+1, b)

def id(x: Int): Int = x
def square(x: Int): Int = x * x
def powerOfTwo(x: Int): Int = if(x == 0) 1 else 2 * powerOfTwo(x-1)

def sumInts(a: Int, b: Int): Int = sum(id, a, b)
def sumSquared(a: Int, b: Int): Int = sum(square, a, b)
def sumPowersOfTwo(a: Int, b: Int): Int = sum(powerOfTwo, a, b)
```
有用的高阶函数

```scala

```
http://www.jianshu.com/p/5eb84e76ca6f

## 六. 内嵌函数
**局部函数**

```scala
object Test {
   def main(args: Array[String]) {
      println( factorial(0) )
      println( factorial(1) )
      println( factorial(2) )
      println( factorial(3) )
   }

   def factorial(i: Int): Int = {
      def fact(i: Int, accumulator: Int): Int = {
         if (i <= 1)
            accumulator
         else
            fact(i - 1, i * accumulator)
      }
      fact(i, 1)
   }
}
```
**匿名函数**
Scala 的类型推测系统会推测出参数的类型

```scala
var inc = (x:Int) => x+1
```
等效于

```scala
def add2 = new Function1[Int,Int]{  
	def apply(x:Int):Int = x+1;  
} 
```
inc可以作为一个函数,使用方如下

```scala
var x = inc(7)-1
```
inc 现在可作为一个函数，使用方式如下：

```scala
var x = inc(7)-1
```
多个参数

```scala
var mul = (x: Int, y: Int) => x*y
```
作为函数使用

```scala
println(mul(3, 4))
```
不给匿名函数设置参数

```scala
var userDir = () => { System.getProperty("user.dir") }
```
user可作为函数

```scala
println( userDir )
```
**偏应用函数**
不需要提供所有参数，只需要提供部分，或不提供

```scala
object Test {
   def main(args: Array[String]) {
      val date = new Date
      log(date, "message1" )
      Thread.sleep(1000)
      log(date, "message2" )
      Thread.sleep(1000)
      log(date, "message3" )
   }

   def log(date: Date, message: String)  = {
     println(date + "----" + message)
   }
}
```
可以绑定一个参数，用_代替另外一个参数，修改后的代码如下

```scala
object Test {
   def main(args: Array[String]) {
      val date = new Date
      val logWithDateBound = log(date, _ : String)

      logWithDateBound("message1" )
      Thread.sleep(1000)
      logWithDateBound("message2" )
      Thread.sleep(1000)
      logWithDateBound("message3" )
   }

   def log(date: Date, message: String)  = {
     println(date + "----" + message)
   }
}
```
**函数柯里化**
将原来接受两个参数的函数变成新的接受一个参数的函数的过程
新的函数返回一个以原有第二个参数为参数的函数
定义一个函数

```scala
def add(x:Int,y:Int)=x+y
```
变形

```scala
def add(x:Int)(y:Int) = x + y
```
使用的时候`add(1)(2)`最后的结果都是3
实现过程如下
add(1)(2)实际是调用两个函数，第一次调用使用一个参数x，第二次使用参数y调用这个函数类型的值
先演变成这样一个方法

```scala
def add(x:Int)=(y:Int)=>x+y
```
接收一个x参数，返回一个匿名函数，该匿名函数的定义是：接收一个int型参数y，函数体为x+y，现在我们来对这个方法进行调用

```scala
val result = add(1)
```
返回result，该值是一个匿名函数，继续调用`val sum = result(2)`输出结果3

完整实例

```scala
object Test {
   def main(args: Array[String]) {
      val str1:String = "Hello, "
      val str2:String = "Scala!"
      println( "str1 + str2 = " +  strcat(str1)(str2) )
   }

   def strcat(s1: String)(s2: String) = {
      s1 + s2
   }
}
```

## 六. 闭包
闭包是一个函数，返回值依赖于声明在函数外部的一个或多个变量。
闭包通常来讲可以简单的认为是可以访问一个函数里面局部变量的另外一个函数。

```scala
val multiplier = (i:Int) => i * 10  
```
函数体内有一个变量 i，它作为函数的一个参数

```scala
val multiplier = (i:Int) => i * factor
```
内部有两个变量i 和 factor，i是形式参数，在multiplier被调用时，i被赋予一个新的值。
factor不是形参，是自由变量

```scala
var factor = 3  
val multiplier = (i:Int) => i * factor 
```
factor变量在函数外面
这样顶一个multiplier是一个闭包，它引用函数外定义的变量，过程为自由变量捕获构成一个封闭函数
完整例子

```scala
object Test {  
   def main(args: Array[String]) {  
      println( "muliplier(1) value = " +  multiplier(1) )  
      println( "muliplier(2) value = " +  multiplier(2) )  
   }  
   var factor = 3  
   val multiplier = (i:Int) => i * factor  
}  
```