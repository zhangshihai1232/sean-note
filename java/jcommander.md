## JCommander
## 一. 使用例子
非常小的java框架，用于解析命令行参数
注解描述符例子

```java
import com.beust.jcommander.Parameter;
 
public class JCommanderExample {
  @Parameter
  private List<String> parameters = new ArrayList<>();
 
  @Parameter(names = { "-log", "-verbose" }, description = "Level of verbosity")
  private Integer verbose = 1;
 
  @Parameter(names = "-groups", description = "Comma-separated list of group names to be run")
  private String groups;
 
  @Parameter(names = "-debug", description = "Debug mode")
  private boolean debug = false;
}
```
使用JCommander解析 

```java
JCommanderExample jct = new JCommanderExample();
String[] argv = { "-log", "2", "-groups", "unit" };
new JCommander(jct, argv);
 
Assert.assertEquals(jct.verbose.intValue(), 2);
```
实际观察到的结果入下

```java
class Main {
    @Parameter(names={"--length", "-l"})
    int length;
    @Parameter(names={"--pattern", "-p"})
    int pattern;
 
    public static void main(String ... args) {
        Main main = new Main();
        new JCommander(main, args);
        main.run();
    }
 
    public void run() {
        System.out.printf("%d %d", length, pattern);
    }
}
```

## 二.支持的类型
**Boolean**
可以设置默认值，不传这个参数使用默认值

```java
@Parameter(names = "-debug", description = "Debug mode")
private boolean debug = false;
```
**String, Integer, Long**
这三种类型，JCommander会将它们转化为相应的类型

**Lists**
list类型，可以使用很多次标记传入

```java
@Parameter(names = "-host", description = "The host")
private List<String> hosts = new ArrayList<>();
```
使用

```java
java Main -host host1 -verbose -host host2
```

**Password**
密码之类不做记录的参数，设置为password类型，JCommander会让你在终端中填入

```java
public class ArgsPassword {
  @Parameter(names = "-password", description = "Connection password", password = true)
  private String password;
}
```
使用例子

```java
Value for -password (Connection password):
```
**显示输入**
设置`echoInput`为true

```java
public class ArgsPassword {
  @Parameter(names = "-password", description = "Connection password", password = true, echoInput = true)
  private String password;
}
```
## 三. 定制类型
**注解**
需要文件、主机名、等参数时需要定制
需要实现`IStringConverter`接口写一个类型转换器

```java
public interface IStringConverter<T> {
  T convert(String value);
}
```
string转文件

```java
public class FileConverter implements IStringConverter<File> {
  @Override
  public File convert(String value) {
    return new File(value);
  }
}
```
然后定义转换的类型即可

```java
@Parameter(names = "-file", converter = FileConverter.class)
File file;
```
**工厂方法**
实现接口

```java
public interface IStringConverterFactory {
  <T> Class<? extends IStringConverter<T>> getConverter(Class<T> forType);
}
```
比如转换host和port:`java App -target example.com:8080`
定义holder类

```java
public class HostPort {
  private String host;
  private Integer port;
}
```
定义`string converter`在类中创建实例

```java
class HostPortConverter implements IStringConverter<HostPort> {
  @Override
  public HostPort convert(String value) {
    HostPort result = new HostPort();
    String[] s = value.split(":");
    result.host = s[0];
    result.port = Integer.parseInt(s[1]);
 
    return result;
  }
}
```
工厂如下

```java
public class Factory implements IStringConverterFactory {
  public Class<? extends IStringConverter<?>> getConverter(Class forType) {
    if (forType.equals(HostPort.class)) return HostPortConverter.class;
    else return null;
  }
```
可以直接使用，不增加额外的参数

```java
public class ArgsConverterFactory {
  @Parameter(names = "-hostport")
  private HostPort hostPort;
}
```
只需要在JCommander对象中添加工厂即可

```java
ArgsConverterFactory a = new ArgsConverterFactory();
JCommander jc = new JCommander(a);
jc.addConverterFactory(new Factory());
jc.parse("-hostport", "example.com:8080");
 
Assert.assertEquals(a.hostPort.host, "example.com");
Assert.assertEquals(a.hostPort.port.intValue(), 8080);
```

## 四. 参数校验
两种方式：
- 全局
- 局部
**局部校验**
实现接口IParameterValidator可以做早期校验

```java
public interface IParameterValidator {
  void validate(String name, String value) throws ParameterException;
}
```
验证数为正数的例子

```java
public class PositiveInteger implements IParameterValidator {
 public void validate(String name, String value)
      throws ParameterException {
    int n = Integer.parseInt(value);
    if (n < 0) {
      throw new ParameterException("Parameter " + name + " should be positive (found " + value +")");
    }
  }
}
```
在注解中，在validateWith中指定实现接口的类名

```java
@Parameter(names = "-age", validateWith = PositiveInteger.class)
private Integer age;
```
如果校验不成功会抛异常
**全局校验**
JCommander解析后，需要额外的校验，比如校验两个互斥参数；
java实现非常局限，JCommander不提供解决方案

## 五. Main参数
目前参数都通过names属性定义，可以定义一个没有这个属性的参数，这样的参数需要List类型，它会包含所有没有包含的参数，比如

```java
@Parameter(description = "Files")
private List<String> files = new ArrayList<>();
 
@Parameter(names = "-debug", description = "Debugging level")
private Integer debug = 1;
```
传入参数

```java
java Main -debug file1 file2
```
由于file1不能解析为Integer，因此files会存进file1和file2
## 六. private参数
参数可以设置为私有,这样提供get方法

```java
public class ArgsPrivate {
  @Parameter(names = "-verbose")
  private Integer verbose = 1;
 
  public Integer getVerbose() {
    return verbose;
  }
}
```
使用

```java
ArgsPrivate args = new ArgsPrivate();
new JCommander(args, "-verbose", "3");
Assert.assertEquals(args.getVerbose().intValue(), 3);
```
## 七. 参数分隔符
默认的分隔符为空格
可设置分隔符，类似这样

```java
java Main -log:3
java Main -level=42
```
在参数里配置分隔符

```java
@Parameters(separators = "=")
public class SeparatorEqual {
  @Parameter(names = "-level")
  private Integer level = 2;
}
```
## 八. 多种描述
可以把描述参数分配到多个类中，比如

```java
public class ArgsMaster {
  @Parameter(names = "-master")
  private String master;
}
public class ArgsSlave {
  @Parameter(names = "-slave")
  private String slave;
}
```
然后把这两个对象传入JCommander中

```java
ArgsMaster m = new ArgsMaster();
ArgsSlave s = new ArgsSlave();
String[] argv = { "-master", "master", "-slave", "slave" };
new JCommander(new Object[] { m , s }, argv);
 
Assert.assertEquals(m.master, "master");
Assert.assertEquals(s.slave, "slave");
```
## 九. @语法
@语法可以把参数放到文件中，然后通过参数把文件传递过去
`/tmp/parameters`文件如下

```bash
-verbose
file1
file2
file3
```
传递参数

```bash
java Main @/tmp/parameters
```

## 十. 多值参数
**固定**
如果参数为多个值

```java
java Main -pairs slave master foo.xml
```
这时，参数需要存入list,并且制定arity

```java
@Parameter(names = "-pairs", arity = 2, description = "Pairs")
private List<String> pairs;
```
只有list支持多值
**可变**
参数的数量不确定

```java
program -foo a1 a2 a3 -bar
program -foo a1 -bar
```
这样的参数，类型为list，并且设置variableArity为true

```java
@Parameter(names = "-foo", variableArity = true)
public List<String> foo = new ArrayList<>();
```

## 十一. 多个参数名

```java
@Parameter(names = { "-d", "--outputDirectory" }, description = "Directory")
private String outputDirectory;
```
使用定义中的参数名都可以

```bash
java Main -d /tmp
java Main --outputDirectory /tmp
```
## 十二. 其他参数配置
参数的查找方式

```java
JCommander.setCaseSensitiveOptions(boolean)	//是否大小写敏感
JCommander.setAllowAbbreviatedOptions(boolean)//参数是否可以简写
```
简写参数时，比如`-param`可用参数`-par`替代，如果出错会抛异常

## 十三. 是否必填
required=true，为必填，如果空，抛异常

```java
@Parameter(names = "-host", required = true)
private String host;
```

## 十四. 默认值
定义时，可以设置默认值
也可以使用IDefaultProvider，把配置放在xml或.properties中

```java
public interface IDefaultProvider {
  String getDefaultValueFor(String optionName);
}
```
下面是一个默认的IDefaultProvider,将所有的`-debug`设置为42

```java
private static final IDefaultProvider DEFAULT_PROVIDER = new IDefaultProvider() {
  @Override
  public String getDefaultValueFor(String optionName) {
    return "-debug".equals(optionName) ? "false" : "42";
  }
};
 
// ...
 
JCommander jc = new JCommander(new Args());
jc.setDefaultProvider(DEFAULT_PROVIDER);
```

## 十五. help参数
显示用法和帮助信息

```java
@Parameter(names = "--help", help = true)
private boolean help;
```

## 十六. 更复杂的语法
复杂的工具比如git，svn包含一系列的命令，具有自己的语法
比如

```bash
git commit --amend -m "Bug fix"
```
上述commit关键字在JCommander中叫做`commands`,可以为每个命令设置一个arg对象

```java
@Parameters(separators = "=", commandDescription = "Record changes to the repository")
private class CommandCommit {
 
  @Parameter(description = "The list of files to commit")
  private List<String> files;
 
  @Parameter(names = "--amend", description = "Amend")
  private Boolean amend = false;
 
  @Parameter(names = "--author")
  private String author;
}
```

```java
@Parameters(commandDescription = "Add file contents to the index")
public class CommandAdd {
 
  @Parameter(description = "File patterns to add to the index")
  private List<String> patterns;
 
  @Parameter(names = "-i")
  private Boolean interactive = false;
}
```
然后注册进JCommander，在JCommander对象上调用`getParsedCommand()`

```java
CommandMain cm = new CommandMain();
JCommander jc = new JCommander(cm);
 
CommandAdd add = new CommandAdd();
jc.addCommand("add", add);
CommandCommit commit = new CommandCommit();
jc.addCommand("commit", commit);
 
jc.parse("-v", "commit", "--amend", "--author=cbeust", "A.java", "B.java");
 
Assert.assertTrue(cm.verbose);
Assert.assertEquals(jc.getParsedCommand(), "commit");
Assert.assertTrue(commit.amend);
Assert.assertEquals(commit.author, "cbeust");
Assert.assertEquals(commit.files, Arrays.asList("A.java", "B.java"));
```

## 十七. 异常
捕获到任何错误，都会抛ParameterException，都是运行时异常

## 十八. 用法
在JCommander实例上调用，usage()方法，显示所有的使用方法

```bash
Usage: <main class> [options]
  Options:
    -debug          Debug mode (default: false)
    -groups         Comma-separated list of group names to be run
  * -log, -verbose  Level of verbosity (default: 1)
    -long           A long number (default: 0)
```
## 十九. 隐藏参数
不想让参数在help中显示

```java
@Parameter(names = "-debug", description = "Debug mode", hidden = true)
private boolean debug = false;
```

## 二十. 国际化
可以国家化描述
使用`@Parameters`注解，定义名字，然后使用descriptionKey 属性，而不是description

```java
@Parameters(resourceBundle = "MessageBundle")
private class ArgsI18N2 {
  @Parameter(names = "-host", description = "Host", descriptionKey = "host")
  String hostName;
}
```
JCommander会使用默认的locale解析描述

## 二一. 参数`delegates`
如果在同一project中写了很多工具，工具之间可以共享配置，可以继承对象，避免重复代码；
单继承可能会限制灵活性，JCommander支持参数delegates
`@ParameterDelegate`注解

```java
class Delegate {
  @Parameter(names = "-port")
  private int port;
}
 
class MainParams {
  @Parameter(names = "-v")
  private boolean verbose;
 
  @ParametersDelegate
  private Delegate delegate = new Delegate();
}
```
只需要添加MainParams对象到JCommander配置中，就能够使用这个delegate

```java
MainParams p = new MainParams();
new JCommander(p).parse("-v", "-port", "1234");
Assert.assertTrue(p.isVerbose);
Assert.assertEquals(p.delegate.port, 1234);
```

## 二二. 动态参数
允许使用编译时不知道的参数，比如`-Da=b -Dc=d`,必须是Map<String, String>类型，可以在命令行中出现多次

```java
@DynamicParameter(names = "-D", description = "Dynamic parameters go here")
private Map<String, String> params = new HashMap<>();
```
也可以使用除=的分配符
## 二三. scala中使用

```java
import java.io.File
import com.beust.jcommander.{JCommander, Parameter}
import collection.JavaConversions._
 
object Main {
  object Args {
    // Declared as var because JCommander assigns a new collection declared
    // as java.util.List because that's what JCommander will replace it with.
    // It'd be nice if JCommander would just use the provided List so this
    // could be a val and a Scala LinkedList.
    @Parameter(
      names = Array("-f", "--file"),
      description = "File to load. Can be specified multiple times.")
    var file: java.util.List[String] = null
  }
 
  def main(args: Array[String]): Unit = {
    new JCommander(Args, args.toArray: _*)
    for (filename <- Args.file) {
      val f = new File(filename)
      printf("file: %s\n", f.getName)
    }
  }
}
```
## 二四. groovy中使用

```java
import com.beust.jcommander.*
 
class Args {
  @Parameter(names = ["-f", "--file"], description = "File to load. Can be specified multiple times.")
  List<String> file
}
 
new Args().with {
  new JCommander(it, args)
  file.each { println "file: ${new File(it).name}" }
}
```