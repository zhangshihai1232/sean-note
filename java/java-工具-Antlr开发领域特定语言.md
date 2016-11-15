## Antlr
基于java，语言识别工具(ANother Tool for Language Recognition)
其他的语言解析工具：JavaCC、SableCC、Lex、YACC；
在 Antlr 中通过解析用户自定义的上下文无关文法，自动生成词法分析器 (Lexer)、语法分析器 (Parser) 和树分析器 (Tree Parser)

## 作用
**编程语言处理**
语言处理工作分为前端和后端两个部分：
前端包括词法分析、语法分析、语义分析、中间代码生成等若干步骤；
后端包括目标代码生成和代码优化等步骤；
Antlr 致力于解决编译前端的所有工作
**文本处理**
使用 Anltr 的词法分析器生成器，可以完成正则表达式的工作，甚至比正则更强大的工作；

## 安装
Antlr和Antlrworks两个工具

## 使用
**文法定义**
创建Antlr文法文件，命名为Expr.g
文件头部是grammar关键字定义文法的名字
```bash
grammar Expr;
```
假设我们的自定义语言只能输入一个算术表达式;
从而整个程序有一个语句构成，语句有表达式或者换行符构成;
```bash
prog: stat 
; 
stat: expr 
  |NEWLINE 
;
```


Antlr 支持多种目标语言，可以把生成的分析器生成为 Java，C#，C，Python，JavaScript 等多种语言；
默认为java，通过options的`{language=?;}`改变目标语言


**运行Antlr**
完成文法定义之后，运行Antlr，为我们生成需要的词法分析器和语法分析器；
命令行如下：
```bash
java  org.antlr.Tool  c:\antlr_intro\src\expr\Expr.g
```
运行之后，会生成三个文件：Expr.tokens、ExprLexer.java和ExprParser.java
Expr.tokens为文法中用到的各种符号做了数字化编码
ExprLexer是Antlr生成的词法分析器
ExprParser是Antlr 生成的语法分析器

**表达式验证**
词法分析器、语法分析器可以验证输入的表达式是否合法，需要调用Antlr的API完成下面的程序
```java
public static void run(String expr) throws Exception { 
	ANTLRStringStream in = new ANTLRStringStream(expr); 
	ExprLexer lexer = new ExprLexer(in); 
	CommonTokenStream tokens = new CommonTokenStream(lexer); 
	ExprParser parser = new ExprParser(tokens); 
	parser.prog(); 	
}
```
将输入的字符串构建ANTLRStringStream流，然后利用ANTLRStringStream流构建词法分析器ExprLexer；
ExprLexer的作用是产生记号，利用ExprLexer构建一个记号流CommonTokenStream，利用记号流tokens构造语法分析器 parser；
最终调用语法分析器的规则 prog，完成对表达式的验证；
输入合法的表达式，分析器没有任何输出，违法时会分析器输出错误原因；

http://www.ibm.com/developerworks/cn/java/j-lo-antlr/
http://www.cnblogs.com/haoxinyue/p/4225006.html
看不太懂...