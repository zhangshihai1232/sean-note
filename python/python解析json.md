## 一. json
使用json模块
- json.dumps(): 对数据进行编码
- json.loads(): 对数据进行解码

python转json

```bash
Python		JSON
dict		object
list		tuple	array
str			string
int			float, int- & float-derived Enums	number
True		true
False		false
None		null
```

json转python

```bash
JSON			Python
object			dict
array			list
string			str
number (int)	int
number (real)	float
true			True
false			False
null			None
```

**基本类型**
使用json.dumps和json.loads方法序列化和反序列化
dumps参数
- separators: 指定生成的json字符串所用的分隔符，两个分别用于代替“,”、“:”。
- indent:     用于格式化生成的json字符串，接受整数参数的缩进量。
- sort_key:	  true/false，指定dict在序列化时是否按照key排序。

```python
import json
array = ['d', 'b', 'c', 'a', {'b':100, 'a':'letter a'}]
encodestr = json.dumps(array)
org_obj = json.loads(encodestr)
```

**自定义类型-函数**
dumps函数的default参数用于指定一个函数，用于把自定义类型的对象转换成可序列化的基本类型；
loads函数接受参数objecthook用于指定函数，反序列化后的基本类型对象转换成自定义类型的对象

```python
import json
class boy:
    def __init__(self,name,age):
        super().__init__()
        self.name=name
        self.age=age

boy1 = boy('Will', 20)

#default method for decode
def boydefault(obj):
    if isinstance(obj, boy):
        return {'name': obj.name, 'age': obj.age}
    return obj;

def boyhook(dic):
    print('test')
    if dic['name']:
        return boy(dic['name'], dic['age'])
    return dic

boy_encode_str = json.dumps(boy1, default=boydefault)
new_boy = json.loads(boy_encode_str, object_hook=boyhook)
print(boy_encode_str)
print(new_boy)
```

**自定义类型-类**
JSONEncoder和JSONDecoder两个类实现
JSONEncoder主要方法
- default：目的和dumps的default参数一样
- encode：实现序列化的逻辑部分

一个是重载Decoder的构造函数设置object_hook
另一个是重载Decoder的decode方法添加由dict转换为自定义类的逻辑

```python
class BoyEncoder(json.JSONEncoder):
    def default(self, o):
        if isinstance(o, boy):
            return {'name': o.name, 'age': o.age}
        return json.JSONEncoder.default(o);

#override decode method
class BoyDecoder(json.JSONDecoder):
    def decode(self, s):
        dic = super().decode(s);
        return boy(dic['name'], dic['age']);

#override __init__ method
class BoyDecoder2(json.JSONDecoder):
    def __init__(self):
        json.JSONDecoder.__init__(self)
        self.object_hook = boyhook

boy_encode_str = json.dumps(boy1, cls=BoyEncoder)
new_boy = json.loads(boy_encode_str, cls=BoyDecoder)
new_boy2 = json.loads(boy_encode_str, cls=BoyDecoder2)
print(boy_encode_str)
print(new_boy)
print(new_boy2)
```

**例子**
python数据结构转为JSON

```python
import json

# Python 字典类型转换为 JSON 对象
data = {
    'no' : 1,
    'name' : 'Runoob',
    'url' : 'http://www.runoob.com'
}

json_str = json.dumps(data)
print ("Python 原始数据：", repr(data))
print ("JSON 对象：", json_str)
```
JSON编码的字符串转换回一个Python数据结构

```python
import json
...
# 将 JSON 对象转换为 Python 字典
data2 = json.loads(json_str)
print ("data2['name']: ", data2['name'])
print ("data2['url']: ", data2['url'])
```
如果处理的是文件，需要用`json.dump()`和`json.load()`来编解码

```python
# 写入 JSON 数据
with open('data.json', 'w') as f:
    json.dump(data, f)

# 读取数据
with open('data.json', 'r') as f:
    data = json.load(f)
```

## 二. XML解析
- SAX
- DOM
- ElementTree（比较好）
参考
http://www.cnblogs.com/zhangshihai1232/articles/5714098.html
https://docs.python.org/3/library/xml.etree.elementtree.html
http://www.jb51.net/article/63780.htm

## 三. 日期和时间
time模块
calendar模块

获取当前时间戳

```python
import time;  # 引入time模块
ticks = time.time()
print ("当前时间戳为:", ticks)
```

时间元组，一个元组装起来的9组数据

```bash
序号		字段				值
0		4位数年			2008
1		月				1 到 12
2		日				1到31
3		小时				0到23
4		分钟				0到59
5		秒				0到61 (60或61 是闰秒)
6		一周的第几日		0到6 (0是周一)
7		一年的第几日		1到366 (儒略历)
8		夏令时			-1, 0, 1, -1是决定是否为夏令时的旗帜
```
就是struct_time元组

```bash
0	tm_year		2008
1	tm_mon		1 到 12
2	tm_mday		1 到 31
3	tm_hour		0 到 23
4	tm_min		0 到 59
5	tm_sec		0 到 61 (60或61 是闰秒)
6	tm_wday		0到6 (0是周一)
7	tm_yday		1 到 366(儒略历)
8	tm_isdst	-1, 0, 1, -1是决定是否为夏令时的旗帜
```

localtime把time()这个时间戳转化为struct_time

```python
import time
localtime = time.localtime(time.time())
print ("本地时间为 :", localtime)
```

可以使用time模块的strftime方法格式化日期
`time.strftime(format[, t])`

```python
# 格式化成2016-03-20 11:45:39形式
print(time.strftime("%Y-%m-%d %H:%M:%S", time.localtime()))

# 格式化成Sat Mar 28 22:24:24 2016形式
print(time.strftime("%a %b %d %H:%M:%S %Y", time.localtime()))

# 将格式字符串转换为时间戳
a = "Sat Mar 28 22:24:24 2016"
print(time.mktime(time.strptime(a, "%a %b %d %H:%M:%S %Y")))
```

日期格式化符号

```bash
%y 两位数的年份表示（00-99）
%Y 四位数的年份表示（000-9999）
%m 月份（01-12）
%d 月内中的一天（0-31）
%H 24小时制小时数（0-23）
%I 12小时制小时数（01-12）
%M 分钟数（00=59）
%S 秒（00-59）
%a 本地简化星期名称
%A 本地完整星期名称
%b 本地简化的月份名称
%B 本地完整的月份名称
%c 本地相应的日期表示和时间表示
%j 年内的一天（001-366）
%p 本地A.M.或P.M.的等价符
%U 一年中的星期数（00-53）星期天为星期的开始
%w 星期（0-6），星期天为星期的开始
%W 一年中的星期数（00-53）星期一为星期的开始
%x 本地相应的日期表示
%X 本地相应的时间表示
%Z 当前时区的名称
%% %号本身
```

获取某月日历

```bash
import calendar

cal = calendar.month(2016, 1)
print ("以下输出2016年1月份的日历:")
print (cal)
```

**time模块的内置函数**
altzone
asctime([tupletime])
clock()
ctime([secs])
gmtime([secs])
localtime([secs]
mktime(tupletime)
sleep(secs)
strftime(fmt[,tupletime])
strptime(str,fmt='%a %b %d %H:%M:%S %Y')
time( )		#时间戳
tzset()根据环境变量TZ，重新设置
参考：http://www.runoob.com/python3/python3-date-time.html

time两个属性
- timezone时区
- tzname区分夏令时

**Calendar**
calendar(year,w=2,l=1,c=6)
firstweekday( )
isleap(year)
leapdays(y1,y2)
month(year,month,w=2,l=1)
monthcalendar(year,month)
monthrange(year,month)
prcal(year,w=2,l=1,c=6)
prmonth(year,month,w=2,l=1)
setfirstweekday(weekday)
timegm(tupletime)
weekday(year,month,day)

此外还有datetime模块，参考
https://docs.python.org/3/library/datetime.html