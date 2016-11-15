## 字符串格式转换
format(String format, Object... args) 
format(Locale locale, String format, Object... args)

**字符串转换**

```bash
%s	字符串
%c	字符
%b	布尔
%d	十进制
%x	十六进制
%o	八进制
%f	浮点
%a	十六进制浮点
%e	指数
%g	通用浮点(f和e中较短的)
%h	散列码
%%	百分比
%n	换行
%tx	日期时间(x代表不同的日期时间转换符)
```
字符串转换例子

```java
public static void main(String[] args) {  
    String str=null;  
    str=String.format("Hi,%s", "王力");  
    System.out.println(str);  
    str=String.format("Hi,%s:%s.%s", "王南","王力","王张");            
    System.out.println(str);                           
    System.out.printf("字母a的大写是：%c %n", 'A');  
    System.out.printf("3>7的结果是：%b %n", 3>7);  
    System.out.printf("100的一半是：%d %n", 100/2);  
    System.out.printf("100的16进制数是：%x %n", 100);  
    System.out.printf("100的8进制数是：%o %n", 100);  
    System.out.printf("50元的书打8.5折扣是：%f 元%n", 50*0.85);  
    System.out.printf("上面价格的16进制数是：%a %n", 50*0.85);  
    System.out.printf("上面价格的指数表示：%e %n", 50*0.85);  
    System.out.printf("上面价格的指数和浮点数结果的长度较短的是：%g %n", 50*0.85);  
    System.out.printf("上面的折扣是%d%% %n", 85);  
    System.out.printf("字母A的散列码是：%h %n", 'A');  
} 
```

**和转换符一起使用的标志**

```bash
+		为正数或者负数添加符号			("%+d",15)
-		左对齐						("%-5d",15)
0		数字前面补0					("%04d", 99)
空格		整数之前添加指定数量的空格		("% 4d", 99)
,		以“,”对数字分组				("%,f", 9999.99)
(		使用括号包含负数				("%(f", -99.99)
#		浮点数则包含小数点、16进制或8进制则添加0x或0	  ("%#x", 99)
<		格式化前一个转换符所描述的参数	("%f和%<3.2f", 99.45)
$		被格式化的参数索引			("%1$d,%2$s", 99,"abc")
```

标志例子

```java
public static void main(String[] args) {  
    String str=null;  
    //$使用  
    str=String.format("格式参数$的使用：%1$d,%2$s", 99,"abc");             
    System.out.println(str);                       
    //+使用  
    System.out.printf("显示正负数的符号：%+d与%d%n", 99,-99);  
    //补O使用  
    System.out.printf("最牛的编号是：%03d%n", 7);  
    //空格使用  
    System.out.printf("Tab键的效果是：% 8d%n", 7);  
    //.使用  
    System.out.printf("整数分组的效果是：%,d%n", 9989997);  
    //空格和小数点后面个数  
    System.out.printf("一本书的价格是：% 50.5f元%n", 49.8);  
}  
```

## 日期和事件字符串格式化
主要使用字符串格式中的%tx

```bash
c	全部日期时间		星期六 十月 27 14:21:20 CST 2007
F	y-m-d			2007-10-27
D	y/m/d			10/27/07
r	HH:MM:SS PM		02:25:51 下午
T	HH:MM:SS		04:25:51	
R	HH:MM			04:25
```
测试例子

```java
public static void main(String[] args) {  
    Date date=new Date();                                  
    //c的使用  
    System.out.printf("全部日期和时间信息：%tc%n",date);          
    //f的使用  
    System.out.printf("年-月-日格式：%tF%n",date);  
    //d的使用  
    System.out.printf("月/日/年格式：%tD%n",date);  
    //r的使用  
    System.out.printf("HH:MM:SS PM格式（12时制）：%tr%n",date);  
    //t的使用  
    System.out.printf("HH:MM:SS格式（24时制）：%tT%n",date);  
    //R的使用  
    System.out.printf("HH:MM格式（24时制）：%tR",date);  
}  
```
日期转换符,可以通过指定的转换符生成新字符串

```java
public static void main(String[] args) {  
    Date date=new Date();                                      
    //b的使用，月份简称  
    String str=String.format(Locale.US,"英文月份简称：%tb",date);       
    System.out.println(str);                                                                              
    System.out.printf("本地月份简称：%tb%n",date);  
    //B的使用，月份全称  
    str=String.format(Locale.US,"英文月份全称：%tB",date);  
    System.out.println(str);  
    System.out.printf("本地月份全称：%tB%n",date);  
    //a的使用，星期简称  
    str=String.format(Locale.US,"英文星期的简称：%ta",date);  
    System.out.println(str);  
    //A的使用，星期全称  
    System.out.printf("本地星期的简称：%tA%n",date);  
    //C的使用，年前两位  
    System.out.printf("年的前两位数字（不足两位前面补0）：%tC%n",date);  
    //y的使用，年后两位  
    System.out.printf("年的后两位数字（不足两位前面补0）：%ty%n",date);  
    //j的使用，一年的天数  
    System.out.printf("一年中的天数（即年的第几天）：%tj%n",date);  
    //m的使用，月份  
    System.out.printf("两位数字的月份（不足两位前面补0）：%tm%n",date);  
    //d的使用，日（二位，不够补零）  
    System.out.printf("两位数字的日（不足两位前面补0）：%td%n",date);  
    //e的使用，日（一位不补零）  
    System.out.printf("月份的日（前面不补0）：%te",date);  
}  
```

## 时间格式转换

```bash
H	2位数字24时制的小时（不足2位前面补0）		15
I	2位数字12时制的小时（不足2位前面补0）		03
k	2位数字24时制的小时（前面不补0）			15
l	2位数字12时制的小时（前面不补0）			3
M	2位数字的分钟（不足2位前面补0）			03
S	2位数字的秒（不足2位前面补0）				09
L	3位数字的毫秒（不足3位前面补0）			015
N	9位数字的毫秒数（不足9位前面补0）			562000000
p	小写字母的上午或下午标记					中：下午
z	相对于GMT的RFC822时区的偏移量				+0800
Z	时区缩写字符串							CST
s	1970-1-1 00:00:00 到现在所经过的秒数		1193468128
Q	1970-1-1 00:00:00 到现在所经过的毫秒数	1193468128984
```

测试代码

```java
public static void main(String[] args) {  
    Date date = new Date();  
    //H的使用  
    System.out.printf("2位数字24时制的小时（不足2位前面补0）:%tH%n", date);  
    //I的使用  
    System.out.printf("2位数字12时制的小时（不足2位前面补0）:%tI%n", date);  
    //k的使用  
    System.out.printf("2位数字24时制的小时（前面不补0）:%tk%n", date);  
    //l的使用  
    System.out.printf("2位数字12时制的小时（前面不补0）:%tl%n", date);  
    //M的使用  
    System.out.printf("2位数字的分钟（不足2位前面补0）:%tM%n", date);  
    //S的使用  
    System.out.printf("2位数字的秒（不足2位前面补0）:%tS%n", date);  
    //L的使用  
    System.out.printf("3位数字的毫秒（不足3位前面补0）:%tL%n", date);  
    //N的使用  
    System.out.printf("9位数字的毫秒数（不足9位前面补0）:%tN%n", date);  
    //p的使用  
    String str = String.format(Locale.US, "小写字母的上午或下午标记(英)：%tp", date);  
    System.out.println(str);   
    System.out.printf("小写字母的上午或下午标记（中）：%tp%n", date);  
    //z的使用  
    System.out.printf("相对于GMT的RFC822时区的偏移量:%tz%n", date);  
    //Z的使用  
    System.out.printf("时区缩写字符串:%tZ%n", date);  
    //s的使用  
    System.out.printf("1970-1-1 00:00:00 到现在所经过的秒数：%ts%n", date);  
    //Q的使用  
    System.out.printf("1970-1-1 00:00:00 到现在所经过的毫秒数：%tQ%n", date);  
}  
```