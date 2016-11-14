## 一. Function
Function的apply用于转换对象
其功能就是将输入类型转换为输出类型

```java
public interface Function<F, T> {
	T apply(@Nullable F input);
}
```
比如一个简单的日期转换

```java
public class DateFormatFunction implements Function<Date, String> {
	@Override
	public String apply(Date input) {
		SimpleDateFormat dateFormat = new SimpleDateFormat("dd/mm/yyyy");
		return dateFormat.format(input);
	}
}
```

## 二. Functions类
接收一个Map集合作为参数，返回一个Function，用于执行Map集合的查找
**forMap**

```java
public static <K, V> Function<K, V> forMap(Map<K, V> map) {
	return new FunctionForMapNoDefault<K, V>(map);
}
//创建FunctionForMapNoDefault，把map传入
```
FunctionForMapNoDefault内部维持一个map,apply方法通过k获取v,不存在会跑异常

```java
public V apply(@Nullable K key) {
	V result = map.get(key);
	checkArgument(result != null || map.containsKey(key), "Key '%s' not present in map", key);
	return result;
}
```
能够把map转为一个函数
例子：

```java
Map<String, State> states = new HashMap<String, State>();
Function<String, State> lookup = Functions.forMap(states);
System.out.println(lookup.apply(key));//key不存在会抛异常

//你也可以给不存在的key指定一个默认值
Function<String, State> lookup = Functions.forMap(states, null);
```
**compose**
`Function<A, C> compose(Function<B, C> g, Function<A, ? extends B> f)`
接收两个Function作为参数，返回两个Function的组合
f的输出会作为g的输入，g输出为最终作为compose的输出 

```java
public static <A, B, C> Function<A, C> compose(Function<B, C> g, Function<A, ? extends B> f) {
	return new FunctionComposition<A, B, C>(g, f);
}
```
apply方法如下

```java
public C apply(@Nullable A a) {
	return g.apply(f.apply(a));
}
```
例子：

```java
//将城市转为字符串的function
public class StateToCityString implements Function<State, String> {
	@Override
	public String apply(State input) {
		return Joiner.on(",").join(input.getMainCities());
	}
}
//通过名字获取state的function
Function<String, State> lookup = Functions.forMap(states);
Function<String, String> stateCitiesFunction = Functions.compose(stateFunction, lookup); //组合Function
```

## 三. Predicate接口
Predicate的apply用于过滤对象

```java
public interface Predicate<T> {
	boolean apply(T input);
}
```
例子：
过滤人口小于500000的城市

```java
public class PopulationPredicate implements Predicate<City> {
	@Override
	public boolean apply(City input) {
		return input.getPopulation() <= 500000;
	}
}
```

## 四. Predicates类
有两个过滤条件

```java
//选择气候为TEMPERATE的城市
public class TemperateClimatePredicate implements Predicate<City> {
	@Override
	public boolean apply(City input) {
		return input.getClimate().equals(Climate.TEMPERATE);
	}
}

//选择雨量小于45.7的城市
public class LowRainfallPredicate implements Predicate<City> {
	@Override
	public boolean apply(City input) {
		return input.getAverageRainfall() < 45.7;
	}
}
使用如下方法实现过滤的组合
Predicates.and(smallPopulationPredicate,lowRainFallPredicate);			//且
Predicates.or(smallPopulationPredicate,temperateClimatePredicate);		//或
Predicate.not(smallPopulationPredicate);								//非
Predicates.compose(smallPopulationPredicate,lookup);					//组合转换再过滤
```
比如and方法

```java
public static <T> Predicate<T> and(Predicate<? super T>... components) {
	return new AndPredicate<T>(defensiveCopy(components));
}
```
defensiveCopy把Predicate添加到list中
AndPredicate内部维持这个list
apply方法如下,逻辑是有一条不合格，那么久返回false

```java
public boolean apply(@Nullable T t) {
	// Avoid using the Iterator to avoid generating garbage (issue 820).
	for (int i = 0; i < components.size(); i++) {
		if (!components.get(i).apply(t)) {
		  return false;
		}
	}
	return true;
}
```

## 五. Supplier
Supplier接口的get方法，用于创建对象

```java
public interface Supplier<T> {
	T get(); //用于创建对象
}
```

## 六. Suppliers类
**memorize方法**

```java
SupplyCity sc = new SupplyCity();
System.out.println(Suppliers.memoize(sc).get());
System.out.println(Suppliers.memoize(sc).get());//返回同一对象, 单例
```
方法如下

```java
public static <T> Supplier<T> memoize(Supplier<T> delegate) {
	return (delegate instanceof MemoizingSupplier)
		? delegate
		: new MemoizingSupplier<T>(Preconditions.checkNotNull(delegate));
}
```
如果传入的是MemoizingSupplier，那么直接返回这个MemoizingSupplier
如果是普通的supplier，会把它传入，并构造一个MemoizingSupplier返回
构造好的MemoizingSupplier，其get方法获取这个对象的引用，因此能够保持单例

```java
public T get() {
	// A 2-field variant of Double Checked Locking.
	if (!initialized) {
		synchronized (this) {
			if (!initialized) {
			T t = delegate.get();
			value = t;
			initialized = true;
			return t;
			}
		}
	}
	return value;
}
```
**memoizeWithExpiration**
memoizeWithExpiration能够设置一个维持时间，类似缓存

```java
public T get() {
  long nanos = expirationNanos;
  long now = Platform.systemNanoTime();
  if (nanos == 0 || now - nanos >= 0) {
    synchronized (this) {
      if (nanos == expirationNanos) { // recheck for lost race
        T t = delegate.get();
        value = t;
        nanos = now + durationNanos;
        expirationNanos = (nanos == 0) ? 1 : nanos;
        return t;
      }
    }
  }
  return value;
}
```
例子：

```java
SupplyCity sc = new SupplyCity(); //超时再新建对象, 类似缓存
Supplier<City> supplier = Suppliers.memoizeWithExpiration(sc, 5, TimeUnit.SECONDS);
City c = supplier.get();
System.out.println(c); 
Thread.sleep(3000);
c = supplier.get();
System.out.println(c); //与之前相等
Thread.sleep(2000);
c = supplier.get();
System.out.println(c); //与之前不等
```