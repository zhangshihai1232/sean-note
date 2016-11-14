## 一. Scala 简介
**特性**
**面向对象特性**
- 每个值都是对象
- 对象的类型和行为由类的特质描述
- 类抽象机制扩展，并解决多重继承问题：子类继承、灵活的混入机制

**函数式编程**
- 函数式：函数能当做值，轻量级匿名函数，支持高阶函数，嵌套多层函数，支持柯里化
- 常用的代数类型：case class、内置的模式匹配
- 可以利用模式匹配，编写类似正则的代码处理XML

**静态类型**
具备类型系统，编译时检查
- 泛型类
- 协变和逆变
- 标注
- 类型参数的上下限约束
- 把类别和抽象类型作为对象成员
- 复合类型
- 引用自己时显式指定类型
- 视图
- 多态方法

**扩展性**
- 任何方法可用作前缀或后缀操作符
- 可以根据预期类型自动构造闭包

**并发性**
Actor作为并发模型，Akka作为默认的Actor

**Scala Web 框架**
Lift 框架
Play 框架

## 二. 基础语法
**基本概念**
可认为Scala 程序是对象的集合，通过调用彼此的方法来实现消息传递
对象：对象有属性和行为。例如：一只狗的状属性有：颜色，名字，行为有：叫、跑、吃等。对象是一个类的实例。
类：类是对象的抽象，而对象是类的具体实例。
方法：方法描述的基本的行为，一个类可以包含多个方法。
字段：每个对象都有它唯一的实例变量集合，即字段。对象的属性通过给字段赋值来创建。

**基本语法**
区分大小写
类名：类名第一个字母大写
方法名称：第一个字母名称小写
程序文件名：与对象名称完全匹配
def main(args: Array[String])：入口

**标识符**
两种：字符数字、符号
$可看做字母，$开头的标识符为Scala编译器产生的标识符使用
符号标志符：`+ ++ ::: < ?> :->`

**关键字**

```bash
abstract	case		catch		class
def	do		else		extends
false		final		finally		for
forSome		if			implicit	import
lazy		match		new	null
object		override	package		private
protected	return		sealed		super
this		throw		trait		try
true		type		val			var
while		with		yield	 
-	        :			=			=>
<-			<:			<%			>:
#			@	
```

**包**

方式一：
```java
package com.runoob
class HelloWorld
```

方式二：

```java
package com.runoob {
	class HelloWorld 
}
```

**引用**
自动引入如下包：
`java.lang._ 、 scala._ 和 Predef._`

## 三. 数据类型
scala支持的数据类型

```java
Byte	8位有符号补码整数。数值区间为 -128 到 127
Short	16位有符号补码整数。数值区间为 -32768 到 32767
Int		32位有符号补码整数。数值区间为 -2147483648 到 2147483647
Long	64位有符号补码整数。数值区间为 -9223372036854775808 到 9223372036854775807
Float	32位IEEE754单精度浮点数
Double	64位IEEE754单精度浮点数
Char	16位无符号Unicode字符, 区间值为 U+0000 到 U+FFFF
String	字符序列
Boolean	true或false
Unit	表示无值，和其他语言中void等同。用作不返回任何结果的方法的结果类型。Unit只有一个实例值，写成()。
Null	null 或空引用
Nothing	Nothing类型在Scala的类层级的最低端；它是任何其他类型的子类型。
Any		Any是所有其他类的超类
AnyRef	AnyRef类是Scala里所有引用类(reference class)的基类
```
字面量
整型字面量，加L或l表示Long型

```java
0
035
21 
0xFFFFFFFF 
0777L
```
浮点型字面量,后加f或F为float型，否则为Double型

```java
0.0 
1e30f 
3.14159f 
1.0e100
.1
```
布尔字面量：true和false
符号字面量： '<标识符>
标识符可以是字母或数字的标识
这种字面量映射成预定义类scala.Symbol的实例
符号字面量 'x 是表达式 scala.Symbol("x") 的简写，定义如下

```java
package scala
final case class Symbol private (name: String) {
   override def toString: String = "'" + name
}
```
字符字面量

```java
'a' 
'\u0041'
'\n'
'\t'
```
其中 \ 表示转移字符，其后可以跟 u0041 数字或者 \r\n 等固定的转义字符
字符串字面量:

```java
"Hello,\nWorld!"
"菜鸟教程官网：www.runoob.com"
```
多行字符串的表示方法:三引号

```java
	val foo = """菜鸟教程
	www.runoob.com
	www.w3cschool.cc
	www.runnoob.com
	以上三个地址都能访问"""
```
Null 值：scala.Null类型
Scala.Null和scala.Nothing是用统一的方式处理Scala面向对象类型系统的某些"边界情况"的特殊类型
Null类是null引用对象的类型，它是每个引用类（继承自AnyRef的类）的子类。Null不兼容值类型。

**Scala 转义字符**

```bash
\b	\u0008	退格(BS) ，将当前位置移到前一列
\t	\u0009	水平制表(HT) （跳到下一个TAB位置）
\n	\u000c	换行(LF) ，将当前位置移到下一行开头
\f	\u000c	换页(FF)，将当前位置移到下页开头
\r	\u000d	回车(CR) ，将当前位置移到本行开头
\"	\u0022	代表一个双引号(")字符
\'	\u0027	代表一个单引号（'）字符
\\	\u005c	代表一个反斜线字符 '\'
```

## 四. 变量
变量：在程序运行过程中其值可能发生改变的量叫做变量。如：时间，年龄
常量：在程序运行过程中其值不会发生变化的量叫做常量。如：数值 3，字符'A'
变量用var;常量用val;

```java
var myVar : String = "Foo
var myVar : String = "Too"
```
常量

```java
val myVal : String = "Foo"
```
不指定初始值，我这里不好使

```java
var myVar :Int;
val myVal :String;
```
变量类型引用,自动推断

```java
ar myVar = 10;
val myVal = "Hello, Scala!";
```
多个变量声明

```java
val xmax, ymax = 100  // xmax, ymax都声明为100
```
声明元组，可以指定或者不指定类型

```java
val (myVar1: Int, myVar2: String) = Pair(40, "Foo")
val (myVar1, myVar2) = Pair(40, "Foo")
```

## 五. 访问修饰符
类似java:private，protected，public,默认public
private比java严格，嵌套情况外层不能访问内层类的私有成员
**Private**
内部类，或本类可见

```java
class Outer{
    class Inner{
    private def f(){println("f")}
    class InnerMost{
        f() // 正确
        }
    }
    (new Inner).f() //错误,这个访问不在Inner类之内
}
```
java允许外部类访问内部类的私有成员；
**Protected**
Protected比java严格，只允许在子类中使用
在java中同一个包里的其他类也可以进行访问

```java
package p{
class Super{
    protected def f() {println("f")}
    }
	class Sub extends Super{
	    f()
	}
	class Other{
		(new Super).f() //错误
	}
}
```
**Public**
可在任何地方访问

```java
class Outer {
   class Inner {
      def f() { println("f") }
      class InnerMost {
         f() // 正确
      }
   }
   (new Inner).f() // 正确因为 f() 是 public
}
```
**作用域保护**

```java
package bobsrocckets{
    package navigation{
        private[bobsrockets] class Navigator{
         protected[navigation] def useStarChart(){}
         class LegOfJourney{
             private[Navigator] val distance = 100
             }
            private[this] var speed = 200
            }
        }
        package launch{
        import navigation._
        object Vehicle{
        private[launch] val guide = new Navigator
        }
    }
}
```
类Navigator被标记为private[bobsrockets]就是说这个类对包含在bobsrockets包里的所有的类和对象可见。
比如说，从Vehicle对象里对Navigator的访问是被允许的，因为对象Vehicle包含在包launch中，而launch包在bobsrockets中，相反，所有在包bobsrockets之外的代码都不能访问类Navigator