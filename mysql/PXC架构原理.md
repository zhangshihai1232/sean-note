## 一. PXC
`Percona XtraDB Cluster`
基于验证复制，强一致性mysql集群，特点如下：
1. 高可用性，节点不可用不影响集群正常运行
2. 强一致性，可以将读扩展到多个节点上   
3. 节点的增加数据同步自动化（IST,SST）
4. 可实现多点读写，但写压力仍要同步到所有节点

缺点：
1. 由于ddl需全局验证通过，则集群性能由集群中最差性能节点决定。
2. 为保证一致性，galera 总是优先保证数据一致性，在多点并发写时，锁冲突问题严重
3. 新节点加入或延后较大的节点重新加入需全量拷贝数据（sst），作为donor的节点在同步过程中无法提供读写
4. 数据冗余度为节点数

名词：
```bash
WS：write set 写数据集
IST:Incremental State Transfer 增量同步
SST：State Snapshot Transfer 全量同步 
UUID：节点状态改变及顺序的唯一标识。
GTID:Global Transaction ID ，由UUID和偏移量组成。wsrep api 中定义的集群内全局事务id。
```
