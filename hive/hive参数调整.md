调整map和reduce的内存大小
```bash
set mapreduce.map.memory.mb=2000;
set mapred.child.map.java.opts=-Xmx1800M;
set mapreduce.map.java.opts=-Xmx1800M;
set mapreduce.reduce.memory.mb=3500;
set mapreduce.reduce.java.opts=-Xmx3500M;
```