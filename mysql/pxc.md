## 一. MySQL高可用方案
- 基于主从复制；
- 基于Galera协议；
- 基于NDB引擎；
- 基于中间件/proxy；
- 基于共享存储；
- 基于主机高可用；
最常用的是主从复制和Galera

## 二. 主从复制方案
### 2.1 双节点主从 + keepalived/heartbeat
中小型规模，比较方便；
两节点一主一从，或者双主，放置在同一个VLAN中；
master故障后，利用`keepalived/heartbeat`的高可用，快速切换到slave节点
如下图：
![](http://imysql.com/wp-content/uploads/2015/09/mysqlha-2node-keepalived-wm.png)



## 三