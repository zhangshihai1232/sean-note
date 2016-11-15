## information_schema
存储元数据，比如库名或表名，列的数据类型，或访问权限；
INFORMATION_SCHEMA有些只读表，这些实际上是视图，没有文件
表说明

```bash
SCHEMATA	数据库信息	
TABLES		表信息
COLUMNS		表中的列信息；show columns from schemaname.tablename的结构就是从这里取
STATISTICS	表索引的信息；show index from schemaname.tablename从这取
USER_PRIVILEGES		用户权限
SCHEMA_PRIVILEGES	方案权限
TABLE_PRIVILEGES	表权限
COLUMN_PRIVILEGES	列权限
CHARACTER_SETS		字符集
COLLATIONS			各字符集的对照信息
COLLATION_CHARACTER_SET_APPLICABILITY	指明了可用于校对的字符集
TABLE_CONSTRAINTS	存在约束的表
KEY_COLUMN_USAGE	具有约束的键列
ROUTINES			存储程序和函数的信息
VIEWS				关于数据库中的视图的信息
TRIGGERS			关于触发程序的信息
```