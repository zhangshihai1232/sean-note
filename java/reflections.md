## 一. Reflections简介
jdk的反射api很难用
比如:要取出一个类的所有返回string，不带参数的，且以to开头的public方法
代码如下

```java
ArrayList<Method> results = new ArrayList<Method>();    
for (Method m : String.class.getDeclaredMethods()) {               
      if (Modifier.isPublic(m.getModifiers()) &&          
            m.getReturnType().equals(String.class) &&   
            m.getParameterCount() == 0 &&               
            m.getName().startsWith("to")) {             
            results.add(m);                                 
     }                                                   
}    
```
Reflections库可以简化这个过程，同样的查询如下

```java
Set<Method> results = getMethods(String.class,
      withModifier(Modifier.PUBLIC),
      withReturnType(String.class),    
      withParametersCount(0),
      withPrefix("to"));  
```
Reflections浏览classpath，索引metadata，能够运行时查询如下类型的元数据
- 子类型
- 注解的类型，与参数匹配的类型
- 匹配正则的
- 特定签名的方法

典型应用如下

```java
Reflections reflections = new Reflections("my.project");
Set<Class<? extends SomeType>> subTypes = reflections.getSubTypesOf(SomeType.class);
Set<Class<?>> annotated = reflections.getTypesAnnotatedWith(SomeAnnotation.class);
```
## 二. 用法
使用默认的scanners，浏览url包含my.package的路径，包括以my.package开头的

```java
Reflections reflections = new Reflections("my.package");
```
使用ConfigurationBuilder

```java
new Reflections(new ConfigurationBuilder()
     .setUrls(ClasspathHelper.forPackage("my.project.prefix"))
     .setScanners(new SubTypesScanner(), 
                  new TypeAnnotationsScanner().filterResultsBy(optionalFilter), ...),
     .filterInputsBy(new FilterBuilder().includePackage("my.project.prefix"))
     ...);
```
然后方便的使用查询方法，这要根据具体scanners配置
**SubTypesScanner**

```java
Set<Class<? extends Module>> modules = 
    reflections.getSubTypesOf(com.google.inject.Module.class);
```
**TypeAnnotationsScanner**

```java
Set<Class<?>> singletons = 
    reflections.getTypesAnnotatedWith(javax.inject.Singleton.class);
```
**ResourcesScanner**

```java
Set<String> properties = 
    reflections.getResources(Pattern.compile(".*\\.properties"));
```
**MethodAnnotationsScanner**

```java
Set<Method> resources =
    reflections.getMethodsAnnotatedWith(javax.ws.rs.Path.class);
Set<Constructor> injectables = 
    reflections.getConstructorsAnnotatedWith(javax.inject.Inject.class);
```
**FieldAnnotationsScanner**

```java
Set<Field> ids = 
    reflections.getFieldsAnnotatedWith(javax.persistence.Id.class);
```
**MethodParameterScanner**

```java
Set<Method> someMethods =
    reflections.getMethodsMatchParams(long.class, int.class);
Set<Method> voidMethods =
    reflections.getMethodsReturn(void.class);
Set<Method> pathParamMethods =
    reflections.getMethodsWithAnyParamAnnotated(PathParam.class);
```
**MethodParameterNamesScanner**

```java
List<String> parameterNames = 
    reflections.getMethodParamNames(Method.class)
```
**MemberUsageScanner**

```java
Set<Member> usages = 
    reflections.getMethodUsages(Method.class)
```
如果没有配置scanner，默认使用`SubTypesScanner`和`TypeAnnotationsScanner`
也可以配置Classloader，用来解析某些实时类
保证能够解析到url
git上的例子：https://github.com/ronmamo/reflections/tree/master/src/test/java/org/reflections

## 三. ReflectionUtils
ReflectionsUtils包含了一些方便的方法，形式类似`*getAllXXX(type, withYYY)`
比如

```java
import static org.reflections.ReflectionUtils.*;

Set<Method> getters = getAllMethods(someClass,
  withModifier(Modifier.PUBLIC), withPrefix("get"), withParametersCount(0));

//or
Set<Method> listMethodsFromCollectionToBoolean = 
  getAllMethods(List.class,
    withParametersAssignableTo(Collection.class), withReturnType(boolean.class));

Set<Fields> fields = getAllFields(SomeClass.class, withAnnotation(annotation), withTypeAssignableTo(type));
```

## 四. ClasspathHelper
获取包、class、classloader的方法
使用maven可以很方便的集成到项目中，可以把浏览的元数据存储到xml/json文件中，下一次不必浏览，直接使用
在maven中，使用`Reflections Maven plugin`插件
其他用法
- 并行查找url
- 序列化查找为xml/json
- 直接利用存储的元数据，快速load,不必再次scan
- 存储模型实体为.java文件，可以通过静态方式引用`types/fields/methods/annotation`
- 初始化srping的包浏览

## 五. 注解的例子
获取包中，带TaskOption注解的类，然后获取注解的task()

```java
    Map<Class<? extends Task>, Class<?>> optionMap = new HashMap<>();
    for (Class<?> clazz : reflections.getTypesAnnotatedWith(TaskOption.class)) {
        TaskOption taskOption = clazz.getAnnotation(TaskOption.class);
        if (taskOption == null) continue; // this shouldn't happen
        optionMap.put(taskOption.task(), clazz);
    }
```
