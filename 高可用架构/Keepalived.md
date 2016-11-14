## 一. 高可用集群
High Availability Cluster：HACluster
通过可靠性(reliability)和可维护性(maintainability)度量
可靠性：平均无故障时间 (MTTF)衡量
可维护性：平均维修时间（MTTR）来度量
于是：HA=MTTF/(MTTF+MTTR)*100%
衡量标准：
99% 一年宕机时间不超过4天
99.9% 一年宕机时间不超过10小时
99.99% 一年宕机时间不超过1小时
99.999% 一年宕机时间不超过6分钟

## 二. 高可用集群层次
![](http://www.linuxidc.com/upload/2013_08/130810130429981.png?_=5844491)
红色部分的Messaging与Membership层
蓝色部分的Cluster Resource Manager（CRM）层
绿色部分的Local Resource Manager（LRM）与Resource Agent（RA）组成
**信息和成员关系层**
Messaging层：节点间传递心跳信息，也称为心跳层；方式：广播，组播，单播
Membership层：主节点（DC）通过Cluster Consensus Menbership Service（CCM或者CCS）服务，产生一个完整的成员关系，这种服务由Messaging层提供信息；将下层信息，转化为成员关系图传递给上层，以通知各个节点的工作状态；将上层对于隔离某一设备予以具体实施。
**集群资源管理层**
`Cluster Resource Manager`(CRM)实现集群服务的层
每个节点都运行一个集群资源管理器，能为实现高可用提供核心组件，包括资源定义，属性等；
每一个节点上CRM都维护有一个CIB(集群信息库 XML文档)和LRM（本地资源管理器）组件。
对于CIB，只有工作在DC（主节点）上的文档是可以修改的，其他CIB都是复制DC上的那个文档而来的。 
对于LRM,是执行CRM传递过来的在本地执行某个资源的执行和停止的具体执行人。
当某个节点发生故障之后，是由DC通过PE（策略引擎）和TE（实施引擎）来决定是否抢夺资源。
**资源代理层**
`Resource Agents`集群资源代理
管理本节点上的属于集群资源的某一资源的启动,停止和状态信息的脚本；
资源代理分为：
- LSB（/etc /init.d/*）
- OCF(比LSB更专业，更加通用)
- Legacy heartbeat（v1版本的资源管理）
![](http://www.linuxidc.com/upload/2013_08/130810130429982.png?_=5844491)
核心组件：
**ccm组件（Cluster Consensus Menbership Service**
监听底层接受的心跳信息，当监听不到心跳信息的时候就重新计算整个集群的票数和收敛状态信息，传递给上层；
上层做出决定采取怎样的措施，ccm还能够生成一个各节点状态的拓扑结构概览图，以本节点做为视角，保证该节点在特殊情况下能够采取对应的动作。
**crm组件（Cluster Resource Manager，集群资源管理器，也就是pacemaker**
实现资源分配，每个节点上的crm维护一个cib（cluster information lib），定义资源特定的属性，资源定义在哪个节点等；
**cib组件（集群信息基库，Cluster Infonation Base**
XML格式配置文件，工作时常驻内存，且需要通知其他节点；只有在DC上的cib才能修改，其他节点的cib都是拷贝过来的；
cib配置方法：1.命令行；2.前台界面
**rm组件（Local Resource Manager，本地资源管理器**
获取本地某个资源状态，实现本地资源管理，如当检测到对方没有心跳信息时，来启动本地的服务进程等。
**pengine组件**
PE（Policy Engine）策略引擎
定义资源转移的一整套方式，只是策略者，不参与资源转移过程，让TE直行自己的策略；
**TE（Transition Engine）**
直行PE的策略，只有在DC上才能运行PE和TE
**stonithd组件**
STONITH(Shoot The Other Node in the Head，”爆头“)
硬件支持，段其他节点的电源；
案例：主服务器繁忙，没时间响应心跳，如果备用服务器一下把资源抢过去，会一直阻塞，直接把主服务器STONITH就好了；

## 三. 高可用集群的分类   
**双机热备（Active/Passive**
使用Pacemaker和DRBD比较方便
![](http://www.linuxidc.com/upload/2013_08/130810130644552.png?_=5844491)
**多节点热备(N+1)**
为了支持多节点同时节省硬件资源，Pacemaker允许多个`active/passive`共享一个backup节点
![](http://www.linuxidc.com/upload/2013_08/130810130644553.png?_=5844491)
**多节点共享存储（N-TO-N）**
能共享存储时，所有的节点都能称为数据备份；
Pacemaker可以运行多拷贝服务，分散每个节点的工作量；
![](http://www.linuxidc.com/upload/2013_08/130810130644551.png?_=5844491)
**共享存储热备 （Split Site）**
Pacemaker1.2可以简化一些节点
![](http://www.linuxidc.com/upload/2013_08/130810130644554.png?_=5844491)

## 四. 高可用集群软件
**信息与关系层**
heartbeat(v1/v2/v3),heartbeat v3分拆为heartbeat pacemaker cluster-glue
corosync
cman
keepalived
ultramokey
keepalive
**CRM**
haresource,crm (heartbeat v1/v2)
pacemaker (heartbeat v3/corosync)
rgmanager (cman)
**常用组合**
heartbeat v2+haresource(或crm) (说明：一般常用于CentOS 5.X)
heartbeat v3+pacemaker (说明：一般常用于CentOS 6.X)
corosync+pacemaker (说明：现在最常用的组合)
cman + rgmanager (说明：红帽集群套件中的组件，还包括gfs2,clvm)
keepalived+lvs (说明：常用于lvs的高可用)
## 五. 共享存储
Web高可用、mysql数据都是共享的一份
- 直接附加存储DAS(Direct attached storage):RAID阵列/SCSI阵列
- 网络附加存储NAS(network attached storage):NFS/FTP/CIFS
- 存储区域网络SAN(storage area network):块级别，模拟scsi/FC光网络/IPSAN
- 
## 六. 集群文件系统与集群LVM（集群逻辑卷管理cLVM）
集群文件系统：gfs2、ocfs2
集群LVM(Logical Volume Manager)逻辑卷管理：cLVM
一般用于双主模型中
![](http://www.linuxidc.com/upload/2013_08/130810130977721.png?_=5844491)

## 七. 主从节点高可用例子
主从双机热备，基本共享存储；

```bash
数据库文件挂在主服务器，用户连接到主服务器进行操作；master故障时，从服务器自动挂载数据库文件，接替master工作，用户未知情况，通过从数据库文件进行操作，主服务器修复重新提供服务；
```
从服务器如何知道主服务器挂了

```bash
检测机制:如心跳检测，每一个节点都会定期向其他节点通知自己的心跳信息，从服务器几个心跳周期内没有检测到，会认为主服务器挂了；心跳常用udp的694端口
如果主服务器由于繁忙心跳延迟，从服务器挂载了资源（共享数据文件），主服务器还没有挂掉，如果有写操作会导致文件系统崩溃；
这时需要隔离手段，比如STONITH
```
心跳线来检测心跳

```bash
服务器上的Heartbeat的连接
可以通过:以太网、独立串口、独立交叉线甚至多种方式连接，只要有一个收到即可；这样可以避免线路单点故障；
这样要求机器距离比较近
```
隔离方法

```bash
节点隔离：STONITH，方式为所有节点都接到一个电源交换机上
资源隔离：fencing 
```
![](http://www.linuxidc.com/upload/2013_08/130810130945371.jpg?_=5844491)

http://www.cnblogs.com/losbyday/p/5844491.html
http://www.linux-ha.org/wiki/Main_Page
http://clusterlabs.org/wiki/Main_Page
http://opencf.org/home.html
http://www.linuxidc.com/Linux/2013-08/89227.htm
http://www.cnblogs.com/losbyday/p/5844491.html
http://www.ibm.com/developerworks/cn/linux/cluster/l-lvsinst/