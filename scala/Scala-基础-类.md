## 一. 类和对象
是对象的抽象，而对象是类的具体实例
new创建对象

```scala
class Point(xc: Int, yc: Int) {
   var x: Int = xc
   var y: Int = yc

   def move(dx: Int, dy: Int) {
      x = x + dx
      y = y + dy
      println ("x 的坐标点: " + x);
      println ("y 的坐标点: " + y);
   }
}

object Test {
   def main(args: Array[String]) {
      val pt = new Point(10, 20);

      // 移到一个新的位置
      pt.move(10, 10);
   }
}
```
Scala 继承
与java相似，但是要注意几点
1. 重写一个非抽象方法必须使用override修饰符
2. 只有主构造函数才可以往基类的构造函数里写参数
3. 在子类中重写超类的抽象方法时，你不需要使用override关键字

```scala
class Point(val xc: Int, val yc: Int) {
   var x: Int = xc
   var y: Int = yc
   def move(dx: Int, dy: Int) {
      x = x + dx
      y = y + dy
      println ("x 的坐标点 : " + x);
      println ("y 的坐标点 : " + y);
   }
}

class Location(override val xc: Int, override val yc: Int,
   val zc :Int) extends Point(xc, yc){
   var z: Int = zc

   def move(dx: Int, dy: Int, dz: Int) {
      x = x + dx
      y = y + dy
      z = z + dz
      println ("x 的坐标点 : " + x);
      println ("y 的坐标点 : " + y);
      println ("z 的坐标点 : " + z);
   }
}

object Test {
   def main(args: Array[String]) {
      val loc = new Location(10, 20, 15);

      // 移到一个新的位置
      loc.move(10, 10, 5);
   }
}
```
重写非抽象方法，需带override修饰符

```scala
class Person {
  var name = ""
  override def toString = getClass.getName + "[name=" + name + "]"
}

class Employee extends Person {
  var salary = 0.0
  override def toString = super.toString + "[salary=" + salary + "]"
}

object Test extends App {
  val fred = new Employee
  fred.name = "Fred"
  fred.salary = 50000
  println(fred)
}
```
Scala 使用 extends 关键字来继承一个类。实例中 Location 类继承了 Point 类。Point 称为父类(基类)，Location 称为子类。
override val xc 为重写了父类的字段。
继承会继承父类的所有属性和方法，Scala 只允许继承一个父类。
**单例对象**
没有static，但是能使用object
Scala 中使用单例模式时，除了定义的类之外，还要定义一个同名的 object 对象，它和类的区别是，object对象不能带参数。
当单例对象与某个类共享同一个名称时，他被称作是这个类的伴生对象：companion object
必须在同一个源文件里定义类和它的伴生对象。类被称为是这个单例对象的伴生类：companion class。
类和它的伴生对象可以互相访问其私有成员。
不与伴生类共享名称的单例对象被称为孤立对象：standalone object。

```scala
bject AbstractTypeTest1 extends Application {
  def newIntSeqBuf(elem1: Int, elem2: Int): IntSeqBuffer =
    new IntSeqBuffer {
         type T = List[U]
         val element = List(elem1, elem2)
       }
  val buf = newIntSeqBuf(7, 8)
  println("length = " + buf.length)
  println("content = " + buf.element)
}
```

伴生对象
通过伴生对象访问类的实例，提供了单例的机会

```scala
class Worker private{
  def work() = println("I am the only worker!")
}

object Worker{
  val worker = new Worker
  def GetWorkInstance() : Worker = {
    worker.work()
    worker
  }
}

object Job{
  def main(args: Array[String]) { 
		for (i <- 1 to 5) {
		  Worker.GetWorkInstance();
		}
	}
}
```

## 二. Trait特征
Trait相当于java的接口，比接口还功能强大；
它还可以定义属性和方法的实现
一般情况下Scala的类只能够继承单一父类，但是如果是 Trait(特征) 的话就可以继承多个，从结果来看就是实现了多重继承

```scala
trait Equal {
  def isEqual(x: Any): Boolean
  def isNotEqual(x: Any): Boolean = !isEqual(x)
}
```
上述由两个方法组成：`isEqual` 和 `isNotEqual`
isEqual 方法没有定义方法的实现，isNotEqual定义了方法的实现。
子类继承特征可以实现未被实现的方法。所以其实 Scala Trait(特征)更像 Java 的抽象类。
特征完整例子：

```scala
trait Equal {
  def isEqual(x: Any): Boolean
  def isNotEqual(x: Any): Boolean = !isEqual(x)
}

class Point(xc: Int, yc: Int) extends Equal {
  var x: Int = xc
  var y: Int = yc
  def isEqual(obj: Any) =
    obj.isInstanceOf[Point] &&
    obj.asInstanceOf[Point].x == x
}

object Test {
   def main(args: Array[String]) {
      val p1 = new Point(2, 3)
      val p2 = new Point(2, 4)
      val p3 = new Point(3, 3)

      println(p1.isNotEqual(p2))
      println(p1.isNotEqual(p3))
      println(p1.isNotEqual(2))
   }
}
```
特征也可以有构造器，由字段的初始化和其他特征体中的语句构成。这些语句在任何混入该特征的对象在构造时都会被执行。

构造器的执行顺序：
- 调用超类的构造器；
- 特征构造器在超类构造器之后、类构造器之前执行；
- 特质由左到右被构造；
- 每个特征当中，父特质先被构造；
- 如果多个特征共有一个父特质，父特质不会被重复构造
- 所有特征被构造完毕，子类被构造。

构造器的顺序是类的线性化的反向。线性化是描述某个类型的所有超类型的一种技术规格。

## 三. 模式匹配
Scala 提供了强大的模式匹配机制，应用也非常广泛
一个模式匹配包含了一系列备选项，每个都开始于关键字 case
每个备选项都包含了一个模式及一到多个表达式。箭头符号 => 隔开了模式和表达式
整型值模式匹配实例

```scala
object Test {
   def main(args: Array[String]) {
      println(matchTest(3))

   }
   def matchTest(x: Int): String = x match {
      case 1 => "one"
      case 2 => "two"
      case _ => "many"
   }
}
```
不同数据类型的模式匹配

```scala
object Test {
   def main(args: Array[String]) {
      println(matchTest("two"))
      println(matchTest("test"))
      println(matchTest(1))
      println(matchTest(6))

   }
   def matchTest(x: Any): Any = x match {
      case 1 => "one"
      case "two" => 2
      case y: Int => "scala.Int"
      case _ => "many"
   }
}
```
第四个 case 表示默认的全匹配备选项，即没有找到其他匹配时的匹配项，类似 switch 中的 default。
样例类的简单实例

```scala
object Test {
   def main(args: Array[String]) {
   	val alice = new Person("Alice", 25)
	val bob = new Person("Bob", 32)
   	val charlie = new Person("Charlie", 32)
   
    for (person <- List(alice, bob, charlie)) {
    	person match {
            case Person("Alice", 25) => println("Hi Alice!")
            case Person("Bob", 32) => println("Hi Bob!")
            case Person(name, age) =>
               println("Age: " + age + " year, name: " + name + "?")
         }
      }
   }
   // 样例类
   case class Person(name: String, age: Int)
}
```
在声明样例类时，下面的过程自动发生了：
- 构造器的每个参数都成为val，除非显式被声明为var，但是并不推荐这么做；
- 在伴生对象中提供了apply方法，所以可以不使用new关键字就可构建对象；
- 提供unapply方法使模式匹配可以工作；
- 生成toString、equals、hashCode和copy方法，除非显示给出这些方法的定义。

## 四. 正则表达式
Scala 通过 `scala.util.matching` 包种的 Regex 类来支持正则表达式

```scala
object Test {
   def main(args: Array[String]) {
      val pattern = "Scala".r
      val str = "Scala is Scalable and cool"
      println(pattern findFirstIn str)
   }
}
```
返回的结果是`Some(Scala)`
String类的r()方法构造Regex对象，findFirstIn 找到第一个匹配项，如果需要查看所有的匹配项可以使用 findAllIn 方法，可以使用mkString()方法连接正则表达式匹配结果的字符串，并可以使用管道|设置不同的模式

```scala
object Test {
   def main(args: Array[String]) {
      val pattern = new Regex("(S|s)cala")  // 首字母可以是大写 S 或小写 s
      val str = "Scala is scalable and cool"
      println((pattern findAllIn str).mkString(","))   // 使用逗号 , 连接返回结果
   }
}
```
如果你需要将匹配的文本替换为指定的关键词，可以使用 replaceFirstIn( ) 方法来替换第一个匹配项
使用 replaceAllIn( ) 方法替换所有匹配项

```scala
  def main(args: Array[String]) {
      val pattern = "(S|s)cala".r
      val str = "Scala is scalable and cool"
      
      println(pattern replaceFirstIn(str, "Java"))
   }
```
Scala继承了java的语法规则，java大部分使用了perl语法规则
如下：http://www.runoob.com/scala/scala-regular-expressions.html

## 五. 异常处理
Scala 的异常处理和其它语言比如 Java 类似
Scala 的方法可以通过抛出异常的方法的方式来终止相关代码的运行，不必通过返回值
抛出异常`throw`

```scala
throw new IllegalArgumentException
```
捕获异常

```scala
   def main(args: Array[String]) {
      try {
         val f = new FileReader("input.txt")
      } catch {
         case ex: FileNotFoundException =>{
            println("Missing file exception")
         }
         case ex: IOException => {
            println("IO Exception")
         }
      }
   }
```
finally 语句

```scala
   def main(args: Array[String]) {
      try {
         val f = new FileReader("input.txt")
      } catch {
         case ex: FileNotFoundException => {
            println("Missing file exception")
         }
         case ex: IOException => {
            println("IO Exception")
         }
      } finally {
         println("Exiting finally...")
      }
   }
```

## 六. Extractor
从传递给它的对象中提取出构造该对象的参数
Scala 标准库包含了一些预定义的提取器
Scala 提取器是一个带有unapply方法的对象
unapply方法算是apply方法的反向操作：unapply接受一个对象，然后从对象中提取值，提取的值通常是用来构造该对象的值。

```scala
object Test {
   def main(args: Array[String]) {
      println ("Apply 方法 : " + apply("Zara", "gmail.com"));
      println ("Unapply 方法 : " + unapply("Zara@gmail.com"));
      println ("Unapply 方法 : " + unapply("Zara Ali"));
   }
   // 注入方法 (可选)
   def apply(user: String, domain: String) = {
      user +"@"+ domain
   }

   // 提取方法（必选）
   def unapply(str: String): Option[(String, String)] = {
      val parts = str split "@"
      if (parts.length == 2){
         Some(parts(0), parts(1)) 
      }else{
         None
      }
   }
}
```
通过 apply 方法我们无需使用 new 操作就可以创建对象
可以通过语句 Test("Zara", "gmail.com") 来构造一个字符串 "Zara@gmail.com"
unapply方法算是apply方法的反向操作：unapply接受一个对象，然后从对象中提取值，提取的值通常是用来构造该对象的值。
我们使用 Unapply 方法从对象中提取用户名和邮件地址的后缀
实例中 unapply 方法在传入的字符串不是邮箱地址时返回 None

```scala
unapply("Zara@gmail.com") 相等于 Some("Zara", "gmail.com")
unapply("Zara Ali") 相等于 None
```

提取器使用模式匹配
在我们实例化一个类的时，可以带上0个或者多个的参数，编译器在实例化的时会调用 apply 方法。我们可以在类和对象中都定义 apply 方法。
就像我们之前提到过的，unapply 用于提取我们指定查找的值，它与 apply 的操作相反。 当我们在提取器对象中使用 match 语句是，unapply 将自动执行，如下所示：

```scala
object Test {
   def main(args: Array[String]) {
      
      val x = Test(5)
      println(x)

      x match
      {
         case Test(num) => println(x + " 是 " + num + " 的两倍！")
         //unapply 被调用
         case _ => println("无法计算")
      }

   }
   def apply(x: Int) = x*2
   def unapply(z: Int): Option[Int] = if (z%2==0) Some(z/2) else None
}
```

## 七. 文件 I/O
直接使用java的io类

```scala
object Test {
   def main(args: Array[String]) {
      val writer = new PrintWriter(new File("test.txt" ))
      writer.write("菜鸟教程")
      writer.close()
   }
}
```
从屏幕上读取用户输入

```scala
object Test {
   def main(args: Array[String]) {
      print("请输入菜鸟教程官网 : " )
      val line = Console.readLine
      println("谢谢，你输入的是: " + line)
   }
}
```
从文件上读取内容, Scala 的 Source 类及伴生对象来读取文件

```scala
object Test {
   def main(args: Array[String]) {
      println("文件内容为:" )
      Source.fromFile("test.txt" ).foreach{ 
         print 
      }
   }
}
```