### 1 什么是zookeeper？

ZooKeeper 是一个开源的分布式应用程序协调服务器，其为分布式系统提供一致性服务。
其一致性是通过基于 Paxos 算法的 ZAB 协议完成的。
其主要功能包括：配置维护、分布式同步、集群管理、分布式事务等。
### 2 zk如何保证数据一致性？

ZAB协议（ZooKeeper Automic Broadcast）原子广播协议，支持崩溃恢复。
先说一下，类似raft协议。当然ZAB是在raft之前出现的。 
定义了三个角色，leader，follower，observer。
leader负责写请求，并将写请求广播出去，超过半数follower节点响应，就提交该请求，这些请求都通过
一个队列来维护，这样就能保证其他角色的接收数据的**顺序性**。
在不要求强一致性的情况下，follower能够处理读请求，如果是写请求则转给leader处理。

### zk是如何进行leader选举的？
启动初始化选择，一开始启动都是follower，经过一个随机等待时间，某个follower率先
发起leader选择投票，得到超过半数响应，则成为leader。

另一种情况，leader挂掉，需要重新选择，集群其他follower处于looking状态，某个follower会
发起leader选举，如果该提案得到另外一台通过，则该follower成为leader。

以3台机器集群为例，分两种情况，一种是follower挂了一个，leader的请求仍然满足超过
半数的响应，挂掉的follower重新同步即可。

### 如何保证数据顺序性？
ZAB定义了一个全局单调递增的事务ID ZXID ，它是一个64位long型，其中高32位表示 epoch 年代，低32位表示事务id。
epoch 是会根据 Leader 的变化而变化的，当一个 Leader 挂了，新的 Leader 上位的时候，年代（epoch）就变了。而低32位可以简单理解为递增的事务id。
新的任期开始低32位为0。

如果只是follower挂了，只要集群中超过半数的机器还正常运行，zk就能正常运行。

如果leader挂了，需要暂停服务，重新选举。

### 为什么使用奇数台服务器构成 ZooKeeper 集群？
剩下的个数必须大于宕掉的个数的话整个zookeeper才依然可用，比如3台和4台，都是最多只允许1个节点挂掉，所以那就3台。

### 有哪些重要的概念？
- Session 客户端与服务端的一个TCP长链接。
通过这个连接，客户端能够通过心跳检测与服务器保持有效的会话，也能够向Zookeeper服务器发送请求并接受响应，同时还能够通过该连接接收来自服务器的Watch事件通知。
- Znode zookeeper 中的最小数据单元，每个 znode 上都可以保存数据，同时还可以挂载子节点，形成一个树形化命名空间。
- ACL  Access Control Lists ，它是一种权限控制。create，read，write，delete，admin，CREATE和DELETE这两种权限都是针对子节点的权限控制。
- watcher 监听机制，客户端可以向服务端指定某些特定的事件，当这些事件发生时，服务端就会通知客户端。
### 如何实现分布式锁？
-分布式锁使用到了zk的三个属性：有序节点，临时节点，监听机制；
- 客户端连接zookeeper，并在/lock下创建临时的且有序的子节点，第一个客户端对应的子节点为/lock/lock-0000000000，第二个为/lock/lock-0000000001，以此类推；
- 客户端获取/lock下的子节点列表，判断自己创建的子节点是否为当前子节点列表中序号最小的子节点，
如果是则认为获得锁，否则监听刚好在自己之前一位的子节点删除消息，获得子节点变更通知后重复此步骤直至获得锁；
- 执行业务代码；
- 完成业务流程后，删除对应的子节点释放锁；
- org.apache.curator 这个开源jar实现了zk的分布式锁；
```cassandraql
<dependency>

<groupId>org.apache.curator</groupId>

<artifactId>curator-recipes</artifactId>

<version>4.0.0</version>

</dependency>
```
优点：可重入，自动释放，无单点问题。  
缺点：羊群效应，一个需要避免的问题是当一个特定的znode 改变的时候ZooKeper 触发了所有watches 的事件。
还有就是性能上不如使用缓存实现分布式锁。
### 参考文献
- [万字带你入门Zookeeper](https://juejin.im/post/5e184673f265da3df716d449)
- [10分钟看懂！基于Zookeeper的分布式锁](https://blog.csdn.net/qiangcuo6087/article/details/79067136)
- [分布式锁实现汇总](https://juejin.im/post/5a20cd8bf265da43163cdd9a)