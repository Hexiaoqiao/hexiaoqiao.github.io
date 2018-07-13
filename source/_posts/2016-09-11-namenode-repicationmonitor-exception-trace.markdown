---
layout: post
author: hexiaoqiao
nick: void
title: "NameNode RepicationMonitor异常追查"
date: 2016-09-13 10:45:00 +0800
comments: true
categories: [hdfs,namenode,replicationmonitor]
summary: "年初将集群版本从2.4.1升级到2.7.1之后，出现了一个诡异的问题，虽没有影响到线上正常的读写服务但是潜在的问题还是比较严重，经过一段时间的追查彻底解决，这里简单整理追查过程。"
---

集群版本从2.4.1升级到2.7.1之后，出现了一个诡异的问题，虽然没有影响到线上正常读写服务，但是潜在的问题还是比较严重，经过追查彻底解决，这里简单整理追查过程。

## 一、问题描述 

异常初次出现时收集到的集群异常表现信息有两条：  

1、两个关键数据结构持续堆积，监控显示UnderReplicatedBlocks和PendingDeletionBlocks表现明显。
<div class=“pic” align="center" padding=“0”>
<img src="/images/monitor/underreplicatedblocks.png" alt="NameNode UnderReplicatedBlocks数据结构变化趋势" align="center"><br />
<label class=“pic_title” align="center">图1 NameNode UnderReplicatedBlocks数据结构变化趋势</label>
</div>  
<div class=“pic” align="center" padding=“0”>
<img src="/images/monitor/pendingblocks.png" alt="NameNode PendingBlocks数据结构变化趋势" align="center"><br />
<label class=“pic_title” align="center">图2 NameNode PendingBlocks数据结构变化趋势</label>
</div>  

*说明：没有找到异常同一时间段的监控图，可将上图时间点简单匹配，基本不影响后续的分析。*  

2、从NameNode的jstack获得信息ReplicationMonitor线程在长期执行chooseRandom函数；  
{% codeblock namenode.jstack lang:xml %}
"org.apache.hadoop.hdfs.server.blockmanagement.BlockManager$ReplicationMonitor@254e0df1" daemon prio=10 tid=0x00007f59b4364800 nid=0xa7d9 runnable [0x00007f2baf40b000]
   java.lang.Thread.State: RUNNABLE
        at java.util.AbstractCollection.toArray(AbstractCollection.java:195)
        at java.lang.String.split(String.java:2311)
        at org.apache.hadoop.net.NetworkTopology$InnerNode.getLoc(NetworkTopology.java:282)
        at org.apache.hadoop.net.NetworkTopology$InnerNode.getLoc(NetworkTopology.java:292)
        at org.apache.hadoop.net.NetworkTopology$InnerNode.access$000(NetworkTopology.java:82)
        at org.apache.hadoop.net.NetworkTopology.getNode(NetworkTopology.java:539)
        at org.apache.hadoop.net.NetworkTopology.countNumOfAvailableNodes(NetworkTopology.java:775)
        at org.apache.hadoop.hdfs.server.blockmanagement.BlockPlacementPolicyDefault.chooseRandom(BlockPlacementPolicyDefault.java:707)
        at org.apache.hadoop.hdfs.server.blockmanagement.BlockPlacementPolicyDefault.chooseTarget(BlockPlacementPolicyDefault.java:383)
        at org.apache.hadoop.hdfs.server.blockmanagement.BlockPlacementPolicyDefault.chooseTarget(BlockPlacementPolicyDefault.java:432)
        at org.apache.hadoop.hdfs.server.blockmanagement.BlockPlacementPolicyDefault.chooseTarget(BlockPlacementPolicyDefault.java:225)
        at org.apache.hadoop.hdfs.server.blockmanagement.BlockPlacementPolicyDefault.chooseTarget(BlockPlacementPolicyDefault.java:120)
        at org.apache.hadoop.hdfs.server.blockmanagement.BlockManager$ReplicationWork.chooseTargets(BlockManager.java:3783)
        at org.apache.hadoop.hdfs.server.blockmanagement.BlockManager$ReplicationWork.access$200(BlockManager.java:3748)
        at org.apache.hadoop.hdfs.server.blockmanagement.BlockManager.computeReplicationWorkForBlocks(BlockManager.java:1408)
        at org.apache.hadoop.hdfs.server.blockmanagement.BlockManager.computeReplicationWork(BlockManager.java:1314)
        at org.apache.hadoop.hdfs.server.blockmanagement.BlockManager.computeDatanodeWork(BlockManager.java:3719)
        at org.apache.hadoop.hdfs.server.blockmanagement.BlockManager$ReplicationMonitor.run(BlockManager.java:3671)
        at java.lang.Thread.run(Thread.java:745)
{% endcodeblock %}

由于线上环境的日志级别为INFO，而ReplicationMonitor中INFO级别之上的日志非常少，从中几乎不能获取到任何有用信息；

异常出现场景：  
1、坏盘、DataNode Decommision或进程异常退出，但不能稳定复现；  
2、外部环境无任何变化和异常，正常读写服务期偶发。

## 二、追查过程

### 2.1 处理线上问题

ReplicationMonitor线程运行异常，造成数据块的副本不能及时补充，如果异常长期存在，极有可能出现丢数据的情况，在没有其他信息辅助解决的情况下，唯一的办法就是重启NameNode（传说中的“三大招”之一），好在HA架构的支持，不至于影响到正常数据生产。

### 2.2 日志

缺少日志，不能定位问题出现的场景，所以首先需要在关键路径上留下必要的信息，方便追查。由于ReplicationMonitor属于独立线程，合理的日志量输出不至于影响服务性能，经过多次调整基本确定需要收集的日志信息：

1、根据NameNode多次jstack信息，怀疑chooseRandom时不停计算countNumOfAvailableNodes，可能存在死循环，尝试输出两类信息：  
（1）ReplicationMonitor当前处理的整体参数及正在处理的Block；
{% codeblock BlockManager.java lang:java https://github.com/apache/hadoop/blob/branch-2.7.1/hadoop-hdfs-project/hadoop-hdfs/src/main/java/org/apache/hadoop/hdfs/server/blockmanagement/BlockManager.java#L3662 github %}
LOG.info("numlive = " + numlive);
LOG.info("blockToProcess = " + blocksToProcess);
LOG.info("nodeToProcess = " + nodesToProcess);
LOG.info("blocksInvalidateWorkPct = " + this.blocksInvalidateWorkPct);
LOG.info("workFound = " + workFound);
{% endcodeblock %}

（2）chooseRandom逻辑中循环体内（调用了countNumOfAvailableNodes）运行超过1min输出该函数入口的所有参数；问题复现后，日志并没有输出，说明异常并不在chooseRandom逻辑本身；

2、结合NameNode的jstack信息并跟进实现逻辑时发现NetworkTopology.InnerNode#getLoc(String loc)的实现存在性能问题：  
{% codeblock NetworkTopology.java lang:java https://github.com/apache/hadoop/blob/branch-2.7.1/hadoop-common-project/hadoop-common/src/main/java/org/apache/hadoop/net/NetworkTopology.java#L274 github %}
/** Given a node's string representation, return a reference to the node
 * @param loc string location of the form /rack/node
 * @return null if the node is not found or the childnode is there but
 * not an instance of {@link InnerNode}
 */
private Node getLoc(String loc) {
  if (loc == null || loc.length() == 0) return this;           
  String[] path = loc.split(PATH_SEPARATOR_STR, 2);
  Node childnode = null;
  for(int i=0; i<children.size(); i++) {
    if (children.get(i).getName().equals(path[0])) {
      childnode = children.get(i);
    }
  }
  if (childnode == null) return null; // non-existing node
  if (path.length == 1) return childnode;
  if (childnode instanceof InnerNode) {
    return ((InnerNode)childnode).getLoc(path[1]);
  } else {
    return null;
  }
}
{% endcodeblock %}

这段逻辑的用意是通过集群网络拓扑结构中节点的字符串标识（如：/IDC/Rack/hostname）获取该节点的对象。实现方法是从拓扑结构中根节点开始逐层向下搜索，直到找到对应的目标节点，逻辑本身没有问题，但是在line286处应该正常break，实现时出现遗漏，其结果是多出一些不必要的时间开销，对于小集群可能影响不大，但是对于IO比较密集的大集群其实影响还是比较大，线下模拟~5K节点的集群拓扑结构，对于NetworkTopology.InnerNode#getLoc(String loc)本身，break可以提升一半的时间开销。  

3、通过前面两个阶段仍然不能完全解决问题，只能继续追加日志，这里再次怀疑可能BlockManager.computeReplicationWorkForBlocks(List<List<Block>> blocksToReplicate)在调用chooseRandom方法时耗时严重，所以在chooseRandom结束后增加了关键的几条日志：  
{% codeblock BlockManager.java lang:java https://github.com/apache/hadoop/blob/branch-2.7.1/hadoop-hdfs-project/hadoop-hdfs/src/main/java/org/apache/hadoop/hdfs/server/blockmanagement/BlockManager.java#L1322 github %}
LOG.info("ReplicationMonitor: block = " + rw.block);
LOG.info("ReplicationMonitor: priority = " + rw.priority);
LOG.info("ReplicationMonitor: srcNode = " + rw.srcNode);
LOG.info("ReplicationMonitor: storagepolicyid = " + rw.bc.getStoragePolicyID());
if (rw.targets == null || rw.targets.length == 0) {
  LOG.info("ReplicationMonitor: targets is empty");
} else {
  LOG.info("ReplicationMonitor: targets.length = " + rw.targets.length);
  for (int i = 0; i < rw.targets.length; i++) {
    LOG.info("ReplicationMonitor: target = " + rw.targets[i] + ", StorageType = " + rw.targets[i].getStorageType());
  }
}
for (Iterator<Node> iterator = excludedNodes.iterator(); iterator.hasNext();) {
  DatanodeDescriptor node = (DatanodeDescriptor) iterator.next();
  LOG.info("ReplicationMonitor: exclude = " + node);
}
{% endcodeblock %}  

包括当前正在处理的Block，优先级（标识缺块的严重程度），源和目标节点集合；（遗漏了关键的信息：Block的Numbytes，后面后详细解释。）  

通过这一步基本上能够收集到异常现场信息，同时也可确定异常时ReplicationMonitor的运行情况，从后续的日志也能说明这一点：  
{% codeblock namenode.log lang:xml %}
2016-04-19 20:08:52,328 WARN org.apache.hadoop.hdfs.server.blockmanagement.BlockPlacementPolicy: Failed to place enough replicas, still in need of 7 to reach 10 (unavailableStorages=[], storagePolicy=BlockStoragePolicy{HOT:7, storageTypes=[DISK], creationFallbacks=[], replicationFallbacks=[ARCHIVE]}, newBlock=false) For more information, please enable DEBUG log level on org.apache.hadoop.hdfs.server.blockmanagement.BlockPlacementPolicy
2016-04-19 20:08:52,328 WARN org.apache.hadoop.hdfs.server.blockmanagement.BlockPlacementPolicy: Failed to place enough replicas, still in need of 7 to reach 10 (unavailableStorages=[DISK], storagePolicy=BlockStoragePolicy{HOT:7, storageTypes=[DISK], creationFallbacks=[], replicationFallbacks=[ARCHIVE]}, newBlock=false) For more information, please enable DEBUG log level on org.apache.hadoop.hdfs.server.blockmanagement.BlockPlacementPolicy
2016-04-19 20:08:52,328 WARN org.apache.hadoop.hdfs.server.blockmanagement.BlockPlacementPolicy: Failed to place enough replicas, still in need of 7 to reach 10 (unavailableStorages=[DISK, ARCHIVE], storagePolicy=BlockStoragePolicy{HOT:7, storageTypes=[DISK], creationFallbacks=[], replicationFallbacks=[ARCHIVE]}, newBlock=false) All required storage types are unavailable: unavailableStorages=[DISK, ARCHIVE], storagePolicy=BlockStoragePolicy{HOT:7, storageTypes=[DISK], creationFallbacks=[], replicationFallbacks=[ARCHIVE]}
2016-04-19 20:08:52,328 INFO org.apache.hadoop.hdfs.server.blockmanagement.BlockManager: ReplicationMonitor: block = blk_8206926206_7139007477
2016-04-19 20:08:52,328 INFO org.apache.hadoop.hdfs.server.blockmanagement.BlockManager: ReplicationMonitor: priority = 2
2016-04-19 20:08:52,329 INFO org.apache.hadoop.hdfs.server.blockmanagement.BlockManager: ReplicationMonitor: srcNode = 10.16.*.*:*
2016-04-19 20:08:52,329 INFO org.apache.hadoop.hdfs.server.blockmanagement.BlockManager: ReplicationMonitor: storagepolicyid = 0
2016-04-19 20:08:52,329 INFO org.apache.hadoop.hdfs.server.blockmanagement.BlockManager: ReplicationMonitor: targets is empty
2016-04-19 20:08:52,329 INFO org.apache.hadoop.hdfs.server.blockmanagement.BlockManager: ReplicationMonitor: exclude = 10.16.*.*:*
2016-04-19 20:08:52,329 INFO org.apache.hadoop.hdfs.server.blockmanagement.BlockManager: ReplicationMonitor: exclude = 10.16.*.*:*
2016-04-19 20:08:52,329 INFO org.apache.hadoop.hdfs.server.blockmanagement.BlockManager: ReplicationMonitor: exclude = 10.16.*.*:*
2016-04-19 20:08:52,329 INFO org.apache.hadoop.hdfs.server.blockmanagement.BlockManager: ReplicationMonitor: exclude = 10.16.*.*:**
{% endcodeblock %}

日志中容易看到当前发生异常时的场景：  

（1）Block的副本数尝试从3调整的10；（异常时还有其他副本增加的请求）  
（2）ReplicationMonitor尝试进行副本调整时失败，原因是遍历全集群后并没有找到一个合适的节点给该Block提供副本存储，日志显示全集群的节点均被加入到了exclude；  

基本能够确定由于chooseRandom函数遍历了全集群导致处理某（些）Block耗时严重，类似情况累积会恶化这种问题；  

到这里基本可以解释为什么几个关键数据结构（UnderReplicatedBlocks和PendingDeletionBlocks）的量持续增加，根本原因在于ReplicationMonitor在尝试对某个Block进行副本调整时，遍历全集群不能选出合适的节点，导致处理一个Block都会耗时严重，如果多个类似Block累积会滚雪球式使情况恶化，而且更加糟糕的是UnderReplicatedBlocks本质是一个优先级队列，如果正好这些Block的优先级较高，处理失败发生超时后还会回到原来的优先队列里，导致后续正常Block也会被阻塞，即使在超时时间范围内ReplicationMonitor可以正常工作，限于其本身的限流及周期（3s）运行机制，实际上可处理的规模非常小，而UnderReplicatedBlocks及PendingDeletionBlocks的生产者丝毫没有变慢，所以造成了数据源源不断的进入队列，但是消费非常缓慢。线上监控数据看到某次极端情况一度累积到1000K规模的UnderReplicatedBlocks，其实风险已经非常高了。  

虽然从日志能够解释通UnderReplicatedBlocks和PendingDeletionBlocks持续升高了，但是仍然遗留了一个关键问题：为什么在副本调整时全集群遍历都没有选出合适的节点？  

### 2.3 暴力破解

此前已经在社区找到类似问题反馈：
https://issues.apache.org/jira/browse/HDFS-8718
但是很遗憾没看到解决方案；

尝试从各种可能和怀疑中解释前面留下的问题并在线下进行各种场景复现：  
（1）线下模拟了~5000节点集群规模遍历的时间开销，基本能够反映线上的情况；  
（2）构造负载严重不均衡时节点选择的场景，不能复现；  
（3）异构存储实现逻辑可能造成的chooseRandom遍历全集群，尝试构造各种异构存储组合并，不能复现；  
（4）并发进行删除和副本调整，没有复现；（后面详细介绍）  

其实（2）和（3）的验证必要性不是很大，负载问题通过源码简单分析即可，异构存储线上并没有开启。复现结果是：没有结果。  

不得已选择临时解决方案：在BlockManager.computeReplicationWorkForBlocks(List<List<Block>> blocksToReplicate)的第二个阶段，针对需要调整副本的Block集合批量进行chooseTargets时加入时间判断，并设定了阈值，当超时发生时退出本轮目标选择逻辑，可以解决PendingDeletionBlocks长时间不能被处理到的问题，代价是牺牲少量处理UnderReplicatedBlocks的时间；上线后符合预期，PendingDeletionBlocks规模得到了有效控制，但是UnderReplicatedBlocks的问题依然存在。

### 2.4 调整参数

期间，我们从前面新增的日志里同时发现了一个有意思的现象，正常情况下workFound的值相对较高，但是一旦出现异常，开始严重下降。workFound标识的是ReplicationMonitor本轮可以调度出去的Block数，影响该值的三个关键参数如下：  
{% codeblock hdfs-site.xml lang:xml http://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-hdfs/hdfs-default.xml apache %}
<property>
  <name>dfs.namenode.replication.work.multiplier.per.iteration</name>
  <value>5</value>
</property>
<property>
  <name>dfs.namenode.replication.max-streams</name>
  <value>50</value>
  <description>
        The maximum number of outgoing replication streams a given node should have
        at one time considering all but the highest priority replications needed.
  </description>
</property>
<property>
  <name>dfs.namenode.replication.max-streams-hard-limit</name>
  <value>100</value>
  <description>
        The maximum number of outgoing replication streams a given node should have
        at one time.
  </description>
</property>
{% endcodeblock %}  

为控制单DN并发数默认值为<2,2,4>，此外可调度的Block数与集群规模正相关，正常情况其实完全满足运行需求，但是由于存在Block不符预期，所以造成workFound量会下降。  
结合集群实际基础环境，尝试大幅提高并发度，设置为<5,50,100>，提高ReplicationMonitor每一轮的处理效率。参数调整后，情况得到了明显改善。  

*说明：结合实际情况谨慎调整该参数，可能会给集群内的网络带来压力。*  

虽然通过一系列调整能够暂缓和改善线上情况，但是依然没有回答前面留下的疑问，也没有彻底解决问题。只有从头再来梳理流程。  

## 三、ReplicationMonitor工作流程

ReplicationMonitor是NameNode内部线程，负责维护数据块的副本数稳定，包括清理无效块和对不符预期副本数的Block进行增删工作。ReplicationMonitor是周期运行线程，默认每3s执行一次，主要由两个关键函数组成：computeDatanodeWork和processPendingReplications（rescanPostponedMisreplicatedBlocks在NameNode启动/主从切换被调用，不包括在本次异常分析范围内）。  
{% codeblock BlockManager.java lang:java https://github.com/apache/hadoop/blob/branch-2.7.1/hadoop-hdfs-project/hadoop-hdfs/src/main/java/org/apache/hadoop/hdfs/server/blockmanagement/BlockManager.java#L3621 github %}
privateclass ReplicationMonitor implements Runnable {
 
  @Override
  publicvoid run() {
    while (namesystem.isRunning()) {
      try {
        // Process replication work only when active NN is out of safe mode.
        if (namesystem.isPopulatingReplQueues()) {
          computeDatanodeWork();
          processPendingReplications();
          rescanPostponedMisreplicatedBlocks();
        }
        Thread.sleep(replicationRecheckInterval);
      } catch (Throwable t) {
        ......
      }
    }
  }
}
{% endcodeblock %}  

**computeDatanodeWork**  
（1）从UnderReplicatedBlocks中取出给定阈值（默认为集群节点总数的2倍）数量范围内需要进行复制的Block集合；由于UnderReplicatedBlocks是一个优先级队列，所以每次一定是按照优先级从高到低获取；  
（2）遍历选出的Block集合，对于每一个Block，根据当前副本分布及chooseTarget策略，选择合适的DataNode集合作为目标节点，准备副本复制；  
（3）将Block进行副本复制的指令分发到NameNode里对应DatanodeDescriptor数据结构中，待该DataNode下次heartbeat过来后及时下发，同时将该Block从UnderReplicatedBlocks拿出来暂存到pendingReplications；  
（4）DataNode接收到指令后把对应Block复制到目标节点，复制结束后，目标节点向NameNode汇报RECEIVED_BLOCK，此后便可以从pendingReplications中删除对应的Block；这里引入pendingReplications的目的是防止Block在复制过程中出现异常后超时，当在给定时间内（默认为5min）仍没有完成复制，需要将其从pendingReplications转移到timedOutItems集合中；超时检查的工作由PendingReplicationBlocks#PendingReplicationMonitor负责。  
（5）将InvalidateBlocks中待删除的Blocks按照DataNode分组后取出分发到NameNode里对应DatanodeDescriptor数据结构中，同样待该DataNode的heartbeat过来后及时下发删除指令；  

**processPendingReplications**  
computeDatanodeWork步骤4出现超时后，将对应的Block从pendingReplications转移到timedOutItems后并没有其他处理逻辑，但是Block复制的事情还得继续，所以还需要将Block再拿回到UnderReplicatedBlocks后重复前面的工作；从timedOutItems拿回到UnderReplicatedBlocks的工作即由processPendingReplications来负责；  

可以看出computeDatanodeWork，processPendingReplications和PendingReplicationMonitor组成了一个生产者消费者的环，下图可以说明这个过程。
<div class=“pic” align="center" padding=“0”>
<img src="/images/monitor/replicationmonitor.png" alt="ReplicationMonitor相关数据流动图示" align="center"><br />
<label class=“pic_title” align="center">图3 ReplicationMonitor相关数据流动图示</label>
</div>  

ReplicationMonitor涉及到两个关键的数据结构：UnderReplicatedBlocks和InvalidateBlocks；这两个数据结构到底是什么，数据哪里来。

（1）UnderReplicatedBlocks：副本数不足的Block集合；  
* 写数据完成时进行副本检查，副本不足Block；  
* 用户调用setReplication增加副本；  
* DataNode节点异常，其上的所有Block；  

（2）InvalidateBlocks：无效Block集合；  
* 文件删除操作；  
* 用户调用setReplication降低副本；  

可以简单理解副本调整和数据删除本质上是一个异步操作，当NameNode接收到客户端的setReplication或delete请求后，简单处理后即可返回，实际的工作是由ReplicationMonitor周期异步进行处理。  

## 四、根本原因

继续追踪日志时，针对触发问题的Block检索了所有日志，发现另外一个现象，每次chooseTarget目标节点选择失败后，总会紧跟一条删除操作的日志：  
{% codeblock namenode.log lang:xml %}
2016-04-19 09:23:08,453 INFO BlockStateChange: BLOCK* addToInvalidates: blk_8197702254_7129783384 10.16.*.*:* 10.16.*.*:* 10.16.*.*:**
{% endcodeblock %}  

审计日志中也能对照到同一时间点，对应的文件确实有删除操作。这个现象在多次异常复现时稳定发生。可以猜测与删除操作有关联。  

删除操作的流程是先把目录树的节点删除，根据删除结果收集到的Block集合，删除每一个Block。  

其中在逐个Block进行删除过程中，发现其逻辑有疑点，主要在block.setNumBytes(BlockCommand.NO_ACK)，其中NO_ACK=Long.MAX_VALUE，也即先将该Block的numbytes设置为最大值，再后续的操作。  
{% codeblock BlockManager.java lang:java https://github.com/apache/hadoop/blob/branch-2.7.1/hadoop-hdfs-project/hadoop-hdfs/src/main/java/org/apache/hadoop/hdfs/server/blockmanagement/BlockManager.java#L3378 github %}
public void removeBlock(Block block) {
  assert namesystem.hasWriteLock();
  // No need to ACK blocks that are being removed entirely
  // from the namespace, since the removal of the associated
  // file already removes them from the block map below.
  block.setNumBytes(BlockCommand.NO_ACK);
  addToInvalidates(block);
  removeBlockFromMap(block);
  // Remove the block from pendingReplications and neededReplications
  pendingReplications.remove(block);
  neededReplications.remove(block, UnderReplicatedBlocks.LEVEL);
  if (postponedMisreplicatedBlocks.remove(block)) {
    postponedMisreplicatedBlocksCount.decrementAndGet();
  }
}
{% endcodeblock %}

虽然对目录树的操作及removeBlock的操作均会持有全局写锁，但是很自然将Block的NumBytes设置成Long.MAX_VALUE的逻辑与chooseTarget遍历全集群仍不能选出合适节点的事实结合起来。  

接下来自然是验证chooseTarget的处理逻辑：  
{% codeblock BlockManager.java lang:java https://github.com/apache/hadoop/blob/branch-2.7.1/hadoop-hdfs-project/hadoop-hdfs/src/main/java/org/apache/hadoop/hdfs/server/blockmanagement/BlockManager.java#L1390 github %}
......
} finally {
  namesystem.writeUnlock();
}
final Set<Node> excludedNodes = new HashSet<Node>();
for(ReplicationWork rw : work){
  // Exclude all of the containing nodes from being targets.
  // This list includes decommissioning or corrupt nodes.
  excludedNodes.clear();
  for (DatanodeDescriptor dn : rw.containingNodes) {
    excludedNodes.add(dn);
  }
  // choose replication targets: NOT HOLDING THE GLOBAL LOCK
  // It is costly to extract the filename for which chooseTargets is called,
  // so for now we pass in the block collection itself.
  rw.chooseTargets(blockplacement, storagePolicySuite, excludedNodes);
}
namesystem.writeLock();
{% endcodeblock %}

上面的实现可以看出，chooseTargets之前释放了全局锁，chooseTargets后重新申请到全局锁，唯独中间的chooseTargets在锁之外。至此，问题触发条件、场景等基本清楚：  

（1）setReplication将文件的副本调大，此时会有一批属于该文件的Block进入UnderReplicatedBlocks等待ReplicationMonitor处理；  
（2）ReplicationMonitor从UnderReplicatedBlocks中取出部分Block，并在前期根据处理逻辑初始化相关参数，将每个Block打包成ReplicationWork，取出的所有Block完成打包后组成ReplicationWork集合，这个过程持有全局锁；  
（3）当步骤2释放完全局锁后，被删除请求的RPC抢到全局锁，恰好这次删除操作对应文件即是步骤（1）中的文件，此时Block的NumBytes被设置成Long.MAX\_VALUE，并被从BlocksMap,pendingReplications及UnderReplicatedBlocks中删除，但是该Block对象的引用还被步骤（2）中的ReplicationWork集合持有，不会被JVM回收，不同的是ReplicationWork集合中对应Block的NumBytes已经被修改成Long.MAX\_VALUE；  
（4）ReplicationMonitor中computeReplicationWorkForBlocks继续进行chooseTarget时显然已经不可能在集群中选出合适的节点，即使遍历完整个集群，本质上还是由于块大小已经是Long.MAX_VALUE，不可能有节点能满足需求。  

通过单元测试对该场景能够稳定复现。

## 五、解决方式

问题分析完后，解决办法其实比较简单：  

（1）如果ReplicationMonitor遇到了Block的NumBytes=BlockCommand.NO_ACK，直接将该Block从UnderReplicatedBlocks中删除；  
（2）如果chooseTarget时遇到了Block的NumBytes=BlockCommand.NO_ACK，直接返回空，无需再遍历整个集群节点；  

至此彻底解决了线上隐藏将近了一个月的Bug。线上再没有出现该异常。详细Patch见：https://issues.apache.org/jira/browse/HDFS-10453 。 

## 六、经验

回头看追查的整个过程，有几点值得总结的经验：  

1、日志经过多次才调整到位，中间遗漏了关键的信息（block.getNumBytes），如果开始及时收集到这个信息，可以省去很多时间，所以如果能够准确快速收集关键数据，问题已经解决一半；  

2、场景复现时提高并发其实是可以复现的，当时仅利用小工具模拟简单的场景，没有在真实环境进行高并发复现，错过一次可以定位的机会，合理的假设怀疑和严谨的场景复现很重要；  

3、虽然问题在线上存在了超过两周时间，但是并没有实际影响到集群正常服务，得益于中间合理可控的缓解手段。如果不能彻底解决可以尝试通过各种方法缓解或绕过问题值得借鉴，这种方法论随处可见，但是只有亲自趟过坑后才能印象深刻。  

4、回过头再看整个问题，解决问题的思路没有问题，但追查过程其实存在一个严重Bug，不再展开详述。  