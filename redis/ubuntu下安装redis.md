---
title: ubuntu下安装redis
categories: redis
toc: true
---

## ubuntu redis

### 1、下载安装

```bash
root@21ebdf03a086:/# apt-cache search redis
root@21ebdf03a086:/# apt-get install redis-server
```
a、redis配置文件：/etc/redis/redis.conf
b、redis服务路径：/etc/init.d/redis-server

### 2、启动redis

```bash
root@21ebdf03a086:/# cd /etc/init.d
root@21ebdf03a086:/# redis-server &
```
注：要先进入"/etc/init.d"目录，再执行"redis-server &"命令启动redis

### 3、启动client客户端连接

```bash
root@21ebdf03a086:/# redis-cli
127.0.0.1:6379> set name ljq
OK
127.0.0.1:6379> get name
"ljq"
127.0.0.1:6379>
```
