---
title: hive导出数据到csv
categories: hive
toc: true
---

hive导出数据

```bash
hive - e"select * from table">aa.csv
```
乱码原因：
1. excel打开csv时格式默认为gbk，但是从hive中导出来的是utf8的
2. csv文件的列分隔符是逗号或者\t，而hive中默认使用\001

解决方式`concat_ws`函数组成列

```bash
hive -e " select concat_ws(',',cat1,cat2,dd_name) as onecl from dd_prod">testaa.csv
```
利用iconv转码

```bash
iconv -f UTF-8 -c  -t GBK testaa.csv > testbb.csv
```
