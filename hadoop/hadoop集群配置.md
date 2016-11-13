---
title: hadoop集群配置
categories: hadoop
toc: true
---

## 完全分布式配置
## 一. 免密登录

```bash
127.0.0.1 localhost
192.168.31.252 master1
192.168.31.250 master2
192.168.31.217 master3
192.168.31.218 master4
```

### 1.1 服务器配置
1. 修改`/etc/ssh/sshd_config`文件中，找到以下内容，并去掉注释符

```bash
=========================
　　RSAAuthentication yes
　　PubkeyAuthentication yes
　　AuthorizedKeysFile  .ssh/authorized_keys
=========================
```
2. 配置authorized_keys文件,修改权限

```bash
touch ~/.ssh/authorized_keys
sudo chmod 600 ~/.ssh/authorized_keys
sudo chmod 700 ~/.ssh
```
如果没有，需要建立`~/.ssh/authorized_keys`文件
把客户机的`id_rsa.pub`文件拷贝到`authorized_keys`中

### 1.2 客户机配置
1.生成公钥

```bash
ssh-keygen 
```
2.执行

```bash
cat ~/.ssh/id_rsa.pub | ssh zsh@master1 "cat - >> ~/.ssh/authorized_keys"
cat ~/.ssh/id_rsa.pub | ssh zsh@master2 "cat - >> ~/.ssh/authorized_keys"
cat ~/.ssh/id_rsa.pub | ssh zsh@master3 "cat - >> ~/.ssh/authorized_keys"
cat ~/.ssh/id_rsa.pub | ssh zsh@master4 "cat - >> ~/.ssh/authorized_keys"
```

## 二. 配置PATH变量

```bash
vim ~/.bashrc
export PATH=$PATH:/usr/local/hadoop/bin:/usr/local/hadoop/sbin
source ~/.bashrc
```


## 三. 配置hadoop信息
需要配置五个文件
slaves
[core-default.xml](http://hadoop.apache.org/docs/r2.6.0/hadoop-project-dist/hadoop-common/core-default.xml)
[hdfs-default.xml](http://hadoop.apache.org/docs/r2.6.0/hadoop-project-dist/hadoop-hdfs/hdfs-default.xml)
[mapred-default.xml](http://hadoop.apache.org/docs/r2.6.0/hadoop-mapreduce-client/hadoop-mapreduce-client-core/mapred-default.xml)
[yarn-default.xml](http://hadoop.apache.org/docs/r2.6.0/hadoop-yarn/hadoop-yarn-common/yarn-default.xml)
### 3.1 slaves
slaves文件，记录dataNode的主机名

```xml
master2
master3
master4
```
### 3.2 core-site.xml 

```xml
<configuration>
    <property>
        <name>fs.default.name</name>
        <value>hdfs://master1:9000</value>
    </property>
    <property>
        <name>hadoop.tmp.dir</name>
        <value>file:/home/zsh/hadoop/tmp</value>
        <description>Abase for other temporary directories.</description>
    </property>
</configuration>
```
hadoop.tmp.dir是hadoop文件系统依赖的基础配置,默认在/tmp/{$user}路径下，这样重启会被清空；

### 3.3 hdfs-site.xml

```xml
<configuration>
        <property>
                <name>dfs.namenode.secondary.http-address</name>
                <value>master1:50090</value>
        </property>
        <property>
                <name>dfs.replication</name>
                <value>1</value>
        </property>
        <property>
                <name>dfs.namenode.name.dir</name>
                <value>file:/home/zsh/hadoop/tmp/dfs/name</value>
        </property>
        <property>
                <name>dfs.datanode.data.dir</name>
                <value>file:/home/zsh/hadoop/tmp/dfs/data</value>
        </property>
</configuration>
```
### 3.4 mapred-site.xml 

```xml
<configuration>
        <property>
                <name>mapreduce.framework.name</name>
                <value>yarn</value>
        </property>
        <property>
                <name>mapreduce.jobhistory.address</name>
                <value>master1:10020</value>
        </property>
        <property>
                <name>mapreduce.jobhistory.webapp.address</name>
                <value>master1:19888</value>
        </property>
</configuration>
```

### 3.5 yarn-site.xml

```xml
<configuration>
        <property>
                <name>yarn.resourcemanager.hostname</name>
                <value>master1</value>
        </property>
        <property>
                <name>yarn.nodemanager.aux-services</name>
                <value>mapreduce_shuffle</value>
        </property>
</configuration>
```

### 3.5 将master1上的hadoop目录分发到各个节点
master1上压缩并复制到各个节点上

```bash
cd ~/work
tar -zcf hadoop.master1.tar.gz ./hadoop-2.6.4   # 先压缩再复制
scp hadoop.master1.tar.gz master2:/home/zsh/work
scp hadoop.master1.tar.gz master3:/home/zsh/work
scp hadoop.master1.tar.gz master4:/home/zsh/work
```
各个节点分别解压

```bash
cd ~/work
tar vxzf hadoop.master1.tar.gz
```

## 四. 启动集群
centos6关闭防火墙

```bash
sudo service iptables stop   # 关闭防火墙服务
sudo chkconfig iptables off  # 禁止防火墙开机自启，就不用手动关闭了
```
首次启动在master1上格式化hdfs

```bash
hdfs namenode -format
```
在master1节点上运行

```bash
start-dfs.sh
start-yarn.sh
mr-jobhistory-daemon.sh start historyserver
```
通过jps查看各个节点所启动的进程
在master1节点上可以看到如下进程

```bash
NameNode
ResourceManager
SecondrryNameNode
JobHistoryServer
```
在其他节点上看到如下进程

```bash
DataNode
NodeManager  
```
这些进程缺少一个说明出错了，另外在master节点上云行`hdfs dfsadmin -report`查看datanode是否正常启动，如果live datanodes不为0，说明成功；
在web网页`http://master:50070/`查看集群状态；
关闭集群
先执行`./sbin/stop-dfs.sh`
再执行`./sbin/stop-yarn.sh`
执行：`mr-jobhistory-daemon.sh stop historyserver`
