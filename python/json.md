## json基本概念
json有格式校验，必须符合语法
**数据类型**

```bash
Number		在JavaScript中的双精度浮点格式
String		双引号的反斜杠转义的Unicode
Boolean		true 或 false
Array		值的有序序列
Value		它可以是一个字符串，一个数字，真的还是假（true/false），空(null )等
Object		无序集合键值对
Whitespace	可以使用任何一对中的令牌
null		empty
```
**数字**
只能用十进制
有以下三种

```bash
Integer		整数，包括正负
Fraction	小数
Exponent	Exponent like e, e+, e-,E, E+, E-
```
例子

```bash
{"a": 1e3}
```
**字符串**

```bash
"	double quotation
	reverse solidus
/	solidus
b	backspace
f	form feed
n	new line
r	carriage return
t	horizontal tab
u	four hexadecimal digits
```
例子

```bash
{'name': 'Amit'}
```

**数组**

```bash
{
    "books":[
        {
            "language":"Java",
            "edition":"second"
        },
        {
            "language":"C++",
            "lastName":"fifth"
        },
        {
            "language":"C",
            "lastName":"third"
        }
    ]
}
```

**对象**
无序设置的名称/值对,对象名是字符串

```bash
{
 "id": "011A",
 "language": "JAVA",
 "price": 500,
}
```
**空白**
不影响，只是增加可读性
**null**
空类型

```bash
{
 "price": null
}
```
**boolean**
true或者false
**可以取得值**
`String | Number | Object | Array | TRUE | FALSE | NULL`

## json模式
规范，描述，验证
验证库

```bash
Python	Jsonschema
C		WJElement (LGPLv3)
Java	json-schema-validator (LGPLv3)
```
标准格式

```bash
{
    "$schema": "http://json-schema.org/draft-04/schema#",
    "title": "Product",
    "description": "A product from Acme's catalog",
    "type": "object",
    "properties": {
        "id": {
            "description": "The unique identifier for a product",
            "type": "integer"
        },
        "name": {
            "description": "Name of the product",
            "type": "string"
        },
        "price": {
            "type": "number",
            "minimum": 0,
            "exclusiveMinimum": true
        }
    },
    "required": ["id", "name", "price"]
}
```

**与xml比较**
XML不支持数组,json更人性化
json

```bash
{
   "company": Volkswagen,
   "name": "Vento",
   "price": 800000
}
```
xml

```bash
<car>
   <company>Volkswagen</company>
   <name>Vento</name>
   <price>800000</price>
</car>
```
