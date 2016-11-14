## Spliterator
Iterator只能顺序访问
Spliterator可以将元素分割成多份，分别交于不于的线程去遍历，以提高效率
不是从Spliterator中获取元素，而是tryAdvance() 或 forEachRemaining()对元素操作；
Spliterator可以估计保存的元素数量
http://www.ibm.com/developerworks/cn/java/j-jvmc2/