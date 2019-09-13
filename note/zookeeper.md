[TOC]



# Zookeeper

顺序一致性
从同一个客户端发起的事务请求，最终将会严格按照其发起顺序被应用到ZooKeeper中。
在ZooKeeper中，有三种角色：

* Leader

* Follower

  Follower 可直接处理并返回客户端的读请求,同时会将写请求转发给 Leader 处理,并且负责在 Leader 处理写请求时对请求进行投票。

* Observer

  Observers 接受客户端的连接,并将写请求转发给 leader 节点;

  zookeeper-server status 可以看当前节点的ZooKeeper是什么角色,ZooKeeper默认只有Leader和Follower两种角色，没有Observer角色。为了使用Observer模式，在任何想变成Observer的节点的配置文件中加入：peerType=observer
  并在所有server的配置文件中，配置成observer模式的server的那行配置追加:observer，例如：
  server.1:localhost:2888:3888:observer

Leader服务器为客户端提供读和写服务。
Follower和Observer都能提供读服务，不能提供写服务。两者唯一的区别在于，Observer机器不参与Leader选举过程，也不参与写操作的『过半写成功』策略，因此Observer可以在不影响写性能的情况下提升集群的读性能。

Session是指客户端会话，客户端启动时，首先会与服务器建立一个TCP连接，通过这个连接，客户端能够通过心跳检测和服务器保持有效的会话，也能够向ZooKeeper服务器发送请求并接受响应，同时还能通过该连接接收来自服务器的Watch事件通知。Session的SessionTimeout值用来设置一个客户端会话的超时时间。只要在SessionTimeout规定的时间内能够重新连接上集群中任意一台服务器，那么之前创建的会话仍然有效。

持久节点
所谓持久节点是指一旦这个ZNode被创建了，除非主动进行ZNode的移除操作，否则这个ZNode将一直保存在ZooKeeper上。

临时节点
临时节点的生命周期跟客户端会话绑定，一旦客户端会话失效，那么这个客户端创建的所有临时节点都会被移除。

对应于每个ZNode，ZooKeeper都会为其维护一个叫作Stat的数据结构，Stat中记录了这个ZNode的三个数据版本，分别是version（当前ZNode的版本）、cversion（当前ZNode子节点的版本）和aversion（当前ZNode的ACL版本）。

get /yarn-leader-election/appcluster-yarn/ActiveBreadCrumb

appcluster-yarnrm1
cZxid = 0x1b00133dc0    //Created ZXID,表示该ZNode被创建时的事务ID
ctime = Tue Jan 03 15:44:42 CST 2017    //Created Time,表示该ZNode被创建的时间
mZxid = 0x1d00000063    //Modified ZXID，表示该ZNode最后一次被更新时的事务ID
mtime = Fri Jan 06 08:44:25 CST 2017    //Modified Time，表示该节点最后一次被更新的时间
pZxid = 0x1b00133dc0    //表示该节点的子节点列表最后一次被修改时的事务ID。注意，只有子节点列表变更了才会变更pZxid，子节点内容变更不会影响pZxid。
cversion = 0    //子节点的版本号
dataVersion = 11    //数据节点的版本号
aclVersion = 0    //ACL版本号
ephemeralOwner = 0x0    //创建该节点的会话的seddionID。如果该节点是持久节点，那么这个属性值为0。
dataLength = 22    //数据内容的长度
numChildren = 0    //子节点的个数

在ZooKeeper中，version属性是用来实现乐观锁机制中的『写入校验』的（保证分布式数据原子性操作）。

能改变ZooKeeper服务器状态的操作称为事务操作。一般包括数据节点创建与删除、数据内容更新和客户端会话创建与失效等操作。对应每一个事务请求，ZooKeeper都会为其分配一个全局唯一的事务ID，用ZXID表示，通常是一个64位的数字。每一个ZXID对应一次更新操作，从这些ZXID中可以间接地识别出ZooKeeper处理这些事务操作请求的全局顺序。

ZooKeeper允许用户在指定节点上注册一些Watcher，并且在一些特定事件触发的时候，ZooKeeper服务端会将事件通知到感兴趣的客户端上去。

ZooKeeper采用ACL（Access Control Lists）策略来进行权限控制。ZooKeeper定义了如下5种权限。

CREATE: 创建子节点的权限。
READ: 获取节点数据和子节点列表的权限。
WRITE：更新节点数据的权限。
DELETE: 删除子节点的权限。
ADMIN: 设置节点ACL的权限。

ZAB协议并不像Paxos算法和Raft协议一样，是通用的分布式一致性算法，它是一种特别为ZooKeeper设计的崩溃可恢复的原子广播算法。

ZooKeeper使用了一个单一的主进程（Leader）来接收并处理客户端的所有事务请求，并采用ZAB的原子广播协议，将服务器数据的状态变更以事务Proposal的形式广播到所有的副本进程上去（Follower）。

ZAB协议的核心是定义了对应那些会改变ZooKeeper服务器数据状态的事务请求的处理方式

所有事务请求必须由一个全局唯一的服务器来协调处理，这样的服务器被称为Leader服务器，而剩下的其他服务器则成为Follower服务器。Leader服务器负责将一个客户端事务请求转换成一个事务Proposal（提案）并将该Proposal分发给集群中所有的Follower服务器。之后Leader服务器需要等待所有Follower服务器的反馈，一旦超过半数的Follower服务器进行了正确的反馈后，Leader就会再次向所有的Follower服务器分发Commit消息，要求对刚才的Proposal进行提交。

ZAB协议包括两种基本的模式，分别是崩溃恢复和消息广播。
在整个ZooKeeper集群启动过程中，或是当Leader服务器出现网络中断、崩溃退出与重启等异常情况时，ZAB协议就会进入恢复模式并选举产生新的Leader服务器。当选举产生了新的Leader服务器，同时集群中有过半的机器与该Leader服务器完成了状态同步之后，ZAB协议就会退出恢复模式。其中，状态同步是指数据同步，用来保证集群中存在过半的机器能够和Leader服务器的数据状态保持一致。然后整个集群就可以进入消息广播模式了。

ZooKeeper会为每一个事务生成一个唯一且递增长度为64位的ZXID,ZXID由两部分组成：低32位表示计数器(counter)和高32位的纪元号(epoch)。epoch为当前leader在成为leader的时候生成的，且保证会比前一个leader的epoch大
每一个follower节点都会有一个先进先出（FIFO)的队列用来存放收到的事务请求，保证执行事务的顺序
1. leader从客户端收到一个写请求
2. leader生成一个新的事务并为这个事务生成一个唯一的ZXID，
3. leader将这个事务发送给所有的follows节点
4. follower节点将收到的事务请求加入到历史队列(history queue)中,并发送ack给ack给leader
5. 当leader收到大多数follower（超过法定数量）的ack消息，leader会发送commit请求
6. 当follower收到commit请求时，会判断该事务的ZXID是不是比历史队列中的任何事务的ZXID都小，如果是则提交，如果不是则等待比它更小的事务的commit
    恢复过程的步骤大致可分为
7. 当leader崩溃后，集群进入选举阶段，开始选举出潜在的新leader(一般为集群中拥有最大ZXID的节点)
8. 进入发现阶段，follower与潜在的新leader进行沟通，如果发现超过法定人数的follower同意，则潜在的新leader将epoch加1，进入新的纪元。新的leader产生
9. 集群间进行数据同步，保证集群中各个节点的事务一致
10. 集群恢复到广播模式，开始接受客户端的写请求
    当 leader在commit之后但在发出commit消息之前宕机，即只有老leader自己commit了，而其它follower都没有收到commit消息 新的leader也必须保证这个proposal被提交.(新的leader会重新发送该proprosal的commit消息)
    当 leader产生某个proprosal之后但在发出消息之前宕机，即只有老leader自己有这个proproal，当老的leader重启后(此时左右follower),新的leader必须保证老的leader必须丢弃这个proprosal.(新的leader会通知上线后的老leader截断其epoch对应的最后一个commit的位置)

客户端想服务端注册自己需要关注的节点，一旦该节点的数据发生变更，那么服务端就会向相应的客户端发送Watcher事件通知，客户端接收到这个消息通知后，需要主动到服务端获取最新的数据。

Master选举可以说是ZooKeeper最典型的应用场景了。
如果同时有多个客户端请求创建同一个临时节点，那么最终一定只有一个客户端请求能够创建成功。利用这个特性，就能很容易地在分布式环境中进行Master选举了。同时，其他没有成功创建该节点的客户端，都会在该节点上注册一个子节点变更的Watcher，用于监控当前Master机器是否存活，一旦发现当前的Master挂了，那么其他客户端将会重新进行Master选举。

ZooKeeper上的一个ZNode可以表示一个锁。例如/exclusive_lock/lock节点就可以被定义为一个锁。获得锁就通过创建ZNode的方式来实现。
* 当前获得锁的客户端机器发生宕机或重启，那么该临时节点就会被删除，释放锁。
* 正常执行完业务逻辑后，客户端就会主动将自己创建的临时节点删除，释放锁。
  ZooKeeper都会通知所有在/exclusive_lock节点上注册了节点变更Watcher监听的客户端。这些客户端在接收到通知后，再次重新发起分布式锁获取，即重复『获取锁』过程。



#　一致性

弱一致性：刚写入记录不能马上读到，但总会读到，因此也叫最终一致性

强一致性：

1. 主从同步
2. Paxos
3. Raft(multi-paxos)
4. ZAB(multi-paxos)

## 主从同步

1. master接受写请求
2. Master复制日志到Slave
3. Master等待直到所有从库返回

一个节点失败Master阻塞。

## Paxos

假如在分布式系统中初始是各个节点的数据是一致的，每个节点都顺序执行系列操作，然后每个节点最终的数据还是一致的。
　　一致性算法：用于保证在分布式系统中每个节点都顺序执行相同的操作序列，在每一个指令上执行一致性算法就能够保证最终各个节点的数据都是一致的。
2PC
两阶段提交主要保证了分布式事务的原子性：即所有结点要么全做要么全不做。所谓的两个阶段是指：第一阶段：准备阶段(投票阶段)和第二阶段：提交阶段（执行阶段）。

准备阶段
1. 协调者节点向所有参与者节点询问是否可以执行提交操作(vote)，并开始等待各参与者节点的响应。
2. 参与者节点执行询问发起为止的所有事务操作，并将Undo信息和Redo信息写入日志。（注意：若成功这里其实每个参与者已经执行了事务操作）
3. 各参与者节点响应协调者节点发起的询问。如果参与者节点的事务操作实际执行成功，则它返回一个“同意”消息；如果参与者节点的事务操作实际执行失败，则它返回一个“中止”消息。

提交阶段

当协调者节点从所有参与者节点获得的相应消息都为“同意”时:
1. 协调者节点向所有参与者节点发出“正式提交(commit)”的请求。
2. 参与者节点正式完成操作，并释放在整个事务期间内占用的资源。
3. 参与者节点向协调者节点发送“完成”消息。
4. 协调者节点受到所有参与者节点反馈的“完成”消息后，完成事务。

如果任一参与者节点在第一阶段返回的响应消息为“中止”，或者协调者节点在第一阶段的询问超时之前无法获取所有参与者节点的响应消息时：
1. 协调者节点向所有参与者节点发出“回滚操作(rollback)”的请求。
2. 参与者节点利用之前写入的Undo信息执行回滚，并释放在整个事务期间内占用的资源。
3. 参与者节点向协调者节点发送“回滚完成”消息。
4. 协调者节点受到所有参与者节点反馈的“回滚完成”消息后，取消事务。

2PC缺点

同步阻塞问题。执行过程中，所有参与节点都是事务阻塞型的。当参与者占有公共资源时，其他第三方节点访问公共资源不得不处于阻塞状态。

单点故障。由于协调者的重要性，一旦协调者发生故障。参与者会一直阻塞下去。尤其在第二阶段，协调者发生故障，那么所有的参与者还都处于锁定事务资源的状态中，而无法继续完成事务操作。（如果是协调者挂掉，可以重新选举一个协调者，但是无法解决因为协调者宕机导致的参与者处于阻塞状态的问题）。

数据不一致。在二阶段提交的阶段二中，当协调者向参与者发送commit请求之后，发生了局部网络异常或者在发送commit请求过程中协调者发生了故障，这回导致只有一部分参与者接受到了commit请求。而在这部分参与者接到commit请求之后就会执行commit操作。但是其他部分未接到commit请求的机器则无法执行事务提交。于是整个分布式系统便出现了数据部一致性的现象。

3PC

1. 引入超时机制。同时在协调者和参与者中都引入超时机制。
2. 在第一阶段和第二阶段中插入一个准备阶段。保证了在最后提交阶段之前各参与节点的状态是一致的。CanCommit、PreCommit、DoCommit三个阶段。

CanCommit阶段
3PC的CanCommit阶段其实和2PC的准备阶段很像。协调者向参与者发送commit请求，参与者如果可以提交就返回Yes响应，否则返回No响应。
PreCommit阶段
假如协调者从所有的参与者获得的反馈都是Yes响应，那么就会执行事务的预执行。

1. 发送预提交请求。协调者向参与者发送PreCommit请求，并进入Prepared阶段。
2. 事务预提交。参与者接收到PreCommit请求后，会执行事务操作，并将undo和redo信息记录到事务日志中。
3. 响应反馈。如果参与者成功的执行了事务操作，则返回ACK响应，同时开始等待最终指令。

假如有任何一个参与者向协调者发送了No响应，或者等待超时之后，协调者都没有接到参与者的响应，那么就执行事务的中断。
1. 发送中断请求。协调者向所有参与者发送abort请求。
2. 中断事务。参与者收到来自协调者的abort请求之后（或超时之后，仍未收到协调者的请求），执行事务的中断。

DoCommit阶段
执行提交
1. 发送提交请求。协调接收到参与者发送的ACK响应，那么他将从预提交状态进入到提交状态。并向所有参与者发送doCommit请求。
2. 事务提交。参与者接收到doCommit请求之后，执行正式的事务提交。并在完成事务提交之后释放所有事务资源。
3. 响应反馈。事务提交完之后，向协调者发送ack响应。
4. 完成事务。协调者接收到所有参与者的ack响应之后，完成事务。
    中断事务
    协调者没有接收到参与者发送的ACK响应（可能是接受者发送的不是ACK响应，也可能响应超时），那么就会执行中断事务。
5. 发送中断请求。协调者向所有参与者发送abort请求。
6. 事务回滚。参与者接收到abort请求之后，利用其在阶段二记录的undo信息来执行事务的回滚操作，并在完成回滚之后释放所有的事务资源。
7. 反馈结果。参与者完成事务回滚之后，向协调者发送ACK消息。
8. 中断事务。协调者接收到参与者反馈的ACK消息之后，执行事务的中断。


在doCommit阶段，如果参与者无法及时接收到来自协调者的doCommit或者rebort请求时，会在等待超时之后，会继续进行事务的提交。当进入第三阶段时，说明参与者在第二阶段已经收到了PreCommit请求，那么协调者产生PreCommit请求的前提条件是他在第二阶段开始之前，收到所有参与者的CanCommit响应都是Yes。所以，一句话概括就是，当进入第三阶段时，由于网络超时等原因，虽然参与者没有收到commit或者abort响应，但是他有理由相信：成功提交的几率很大。）

无论是二阶段提交还是三阶段提交都无法彻底解决分布式的一致性问题。

Paxos如何在分布式存储系统中应用？
我们知道在分布式系统中数据是可变的，用户可以任意的进行增删改查，分布式存储系统为了保证数据的可靠存储，一般会采用多副本的方式进行存储。如果对多个副本执行的操作序列[op1,op2, …,opn]不进行任何控制，那网络延迟、超时等各种故障都会导致各个副本之间的更新操作是不同的，这样很难保证副本间的一致性。所以为了保证分布式存储系统数据的一致性，我们希望在各个副本之间的更新操作序列是相同的、不变的，即每个副本执行[op1,op2, …,opn】顺序是相同的。我们通过Paxos算法依次来确定不可变变量opi
的取值，即第i个操作是什么。每次确定完opi
之后，可以让各个副本来执行opi，依次类推。

系统内部由多个acceptor组成，负责存储和管理var变量；外部有多个proposal机器，可以任意并发的调用系统api，向系统提交不同的var取值；系统对外的api接口为：propose(var, V) => <ok, f> or <error>，其中f为系统内部已经确定下来的var的取值。
系统需要保证var的取值满足一致性，即var的取值初始为null，而一旦var的取值被确定，则不可以被更改，并且可以一直获取到这个值。
除了一致性以外，系统还需要满足容错性，即可以容忍任意proposal机器出现故障，也可以容忍少数（半数以下）acceptor出现故障。

确定不可变变量取值 —— 方案一
先考虑整个系统由单个acceptor组成，通过类似互斥锁的机制，来管理并发的proposal运行。Proposal首先向acceptor申请互斥访问权，然后才能请求acceptor接受自己的取值。而acceptor给proposal发放互斥访问权，谁申请到互斥访问权，就接受谁提交的取值。

- 第一阶段：通过Acceptor.prepare()获取互斥访问权和当前var取值，如果无法获取，返回<error>（锁被别人占用）。
- 第二阶段：根据当前var的取值f，选择执行,如果f为null（以前没有proposal设置过var的值），则通过Acceptor.accept(var, V)提交数据,如果f不为null（接受某个proposal设置过var的值f），则通过Acceptor.release()释放访问权，返回<ok, f>。

但不足的是，Proposal在获取到互斥访问权后，并且还没有释放互斥访问权之前发生了故障，则会导致其他所有Proposal无法获取互斥访问权进入第二阶段运行，此时会导致系统陷入“死锁”的状态。即，方案一无法容忍任意Proposal机器出现故障。

引入抢占式访问权 —— 方案二

1. **对Proposal来说，Proposal向Acceptor申请访问权时会指定编号epoch（越大的epoch越新）。获取到访问权以后，才能向Acceptor提交取值。**
2. **对Acceptor来说，采用“喜新厌旧”的原则。即一旦收到更大的新epoch的申请，马上让旧epoch的访问权失效，不再接受他们提交的取值，然后给新的epoch发放访问权限，只接受新的epoch提交的取值。**

- 第一阶段：获取epoch轮次的访问权和当前var的取值，我们可以简单获取当前时间戳为epoch：通过Acceptor.prepare(epoch)，获取epoch轮次的访问权和当前var的取值；如果不能获取，返回<error>。
- 第二阶段：采用“后者认同前者”的原则执行。如果var取值为null，则肯定旧epoch无法生成确定性取值，则通过Acceptor.accept(var, prepared_epoch, V)提交数据V。成功后返回<ok, V>；若accept失败，则说明被更新的epoch抢占或Acceptor故障，返回<error>。如果var取值存在，则此取值肯定是确定性取值，此时认同它不再更改，直接返回<ok, accepted_value>。
  但方案二中只有一个Acceptor，单机模块的Acceptor故障时会导致整个系统宕机，无法提供服务，所以我们仍需要引入多个Acceptor。
  Paxos是在方案二的基础上引入多个Acceptor，在paxos方案里面，Acceptor的实现保持不变，仍然采用“喜新厌旧”的原则运行。由于引进多个Acceptor，所以采用“少数服从多数”的思路。一旦某个epoch的取值f被半数以上Acceptor接受，我们就认为这个var的取值被确定为f，不再更改。
- 第一阶段：选定epoch，获取epoch访问权和对应的var取值，在paxos中需要获取半数以上Acceptor的访问权和对应一组var取值。
- 第二阶段：采用“后者认同前者”的原则执行。如果获取的var取值都为null，则旧的epoch无法形成确定性取值，此时努力使<epoch, V>成为确定性取值,向epoch对应的所有Acceptor提交取值<epoch, V>；如果收到半数以上成功，则返回<ok, V>；否则返回<error>（被新的epoch抢占或Acceptor故障）。如果var的取值存在，认同最大的accepted_epoch对应的取值f，努力使<epoch, f>成为确定性取值。如果f出现半数以上，则说明f已经是确定性取值，直接返回<ok, f>；否则，向epoch对应的所有Acceptor提交取值<epoch, f>。

## multi-poxos

当存在一批提案时，用Basic-Paxos一个一个决议当然也可以，但是每个提案都经历两阶段提交，显然效率不高。Basic-Paxos协议的执行流程针对每个提案（每条redo log）都至少存在三次网络交互：1. 产生log ID；2. prepare阶段；3. accept阶段。

Mulit-Paxos基于Basic-Paxos做了优化，在Paxos集群中利用Paxos协议选举出唯一的leader，在leader有效期内所有的议案都只能由leader发起。

这里强化了协议的假设：即leader有效期内不会有其他server提出的议案。因此，对于后续的提案，我们可以简化掉产生log ID阶段和Prepare阶段，而是由唯一的leader产生log ID，然后直接执行Accept，得到多数派确认即表示提案达成一致（每条redo log可对应一个提案）。

## Raft

### 角色

1. Leader (领导者 - 日志管理):负责日志的同步管理,处理来自客户端的请求,与 Follower 保持这 heartBeat 的联系;
2. Follower (追随者 - 日志同步):刚启动时所有节点为 Follower 状态,响应 Leader 的日志同步请求,响应 Candidate 的请求,把请求到 Follower 的事务转发给 Leader;
3. Candidate (候选者 - 负责选票):负责选举投票,Raft 刚启动时由一个节点从 Follower 转为 Candidate 发起选举,选举出
   Leader 后从 Candidate 转为 Leader 状态;

### Term(任期)

​     在 Raft 中使用了一个可以理解为周期(第几届、任期)的概念,用 Term 作为一个周期,每个 Term 都是一个连续递增的编号,每一轮选举都是一个 Term 周期,在一个 Term 中只能产生一个 Leader;当某节点收到的请求中 Term 比当前 Term 小时则拒绝该请求。

### 选举(Election)
​    Raft 的选举由定时器来触发,每个节点的选举定时器时间都是不一样的,开始时状态都为Follower 某个节点定时器触发选举后 Term 递增,状态由 Follower 转为 Candidate,向其他节点发起 RequestVote RPC 请求,这时候有三种可能的情况发生:

1. 该 RequestVote 请求接收到 n/2+1(过半数)个节点的投票,从 Candidate 转为 Leader,
   向其他节点发送 heartBeat 以保持 Leader 的正常运转。
2. 在此期间如果收到其他节点发送过来的 AppendEntries RPC 请求,如该节点的 Term 大
   则当前节点转为 Follower,否则保持 Candidate 拒绝该请求。
3. Election timeout 发生则 Term 递增,重新发起选举

在一个 Term 期间每个节点只能投票一次,所以当有多个 Candidate 存在时就会出现每个Candidate 发起的选举都存在接收到的投票数都不过半的问题,这时每个 Candidate 都将 Term递增、重启定时器并重新发起选举,由于每个节点中定时器的时间都是随机的,所以就不会多次存在有多个 Candidate 同时发起投票的问题。

在 Raft 中当接收到客户端的日志(事务请求)后先把该日志追加到本地的 Log 中,然后通过heartbeat 把该 Entry 同步给其他 Follower,Follower 接收到日志后记录日志然后向 Leader 发送ACK,当 Leader 收到大多数(n/2+1)Follower 的 ACK 信息后将该日志设置为已提交并追加到本地磁盘中,通知客户端并在下个 heartbeat 中 Leader 将通知所有的 Follower 将该日志存储在自己的本地磁盘中。

## ZAB

## 事务编号 Zxid (事务请求计数器 + epoch )
在 ZAB ( ZooKeeper Atomic Broadcast , ZooKeeper 原子消息广播协议) 协议的事务编号 Zxid设计中,Zxid 是一个 64 位的数字,其中低 32 位是一个简单的单调递增的计数器,针对客户端每一个事务请求,计数器加 1;而高 32 位则代表 Leader 周期 epoch 的编号,每个当选产生一个新的 Leader 服务器,就会从这个 Leader 服务器上取出其本地日志中最大事务的 ZXID,并从中读取epoch 值,然后加 1,以此作为新的 epoch,并将低 32 位从 0 开始计数。Zxid(Transaction id)类似于 RDBMS 中的事务 ID,用于标识一次更新操作的 Proposal(提议)ID。为了保证顺序性,该 zkid 必须单调递增.

## ZAB 协议 4 阶段
### Leader election (选举阶段 - 选出准 Leader )

Leader election(选举阶段):节点在一开始都处于选举阶段,只要有一个节点得到超半数节点的票数,它就可以当选准 leader。只有到达 广播阶段(broadcast) 准 leader 才会成为真正的 leader。这一阶段的目的是就是为了选出一个准 leader,然后进入下一个阶段.

### Discovery (发现阶段 - 接受提议、生成 epoch 、接受 epoch )

Discovery(发现阶段):在这个阶段,followers 跟准 leader 进行通信,**同步 followers最近接收的事务提议。**这个一阶段的主要目的是发现当前大多数节点接收的最新提议,并且准 leader 生成新的 epoch,让 followers 接受,更新它们的 accepted Epoch
一个 follower 只会连接一个 leader,如果有一个节点 f 认为另一个 follower p 是 leader,f在尝试连接 p 时会被拒绝,f 被拒绝之后,就会进入重新选举阶段。

### Synchronization (同步阶段 - 同步 follower 副本)

Synchronization(同步阶段):同步阶段主要是**利用 leader 前一阶段获得的最新提议历史,同步集群中所有的副本。**只有当 大多数节点都同步完成,准 leader 才会成为真正的 leader。follower 只会接收 zxid 比自己的 lastZxid 大的提议。

协议的 Java 版本实现跟上面的定义有些不同,选举阶段使用的是 Fast Leader Election(FLE),它包含了 选举的发现职责。因为 FLE 会选举拥有最新提议历史的节点作为 leader,这样就省去了发现最新提议的步骤。实际的实现将 发现阶段 和 同步合并为 Recovery Phase(恢复阶段)。所以,ZAB 的实现只有三个阶段:Fast Leader Election;Recovery Phase;Broadcast Phase。

## 投票机制
每个 sever 首先给自己投票,然后用自己的选票和其他 sever 选票对比,权重大的胜出,使用权重较大的更新自身选票箱.

1. 每个 Server 启动以后都询问其它的 Server 它要投票给谁。对于其他 server 的询问,server 每次根据自己的状态都回复自己推荐的 leader 的 id 和上一次处理事务的 zxid(系统启动时每个 server 都会推荐自己).
2. 收到所有 Server 回复以后,就计算出 zxid 最大的哪个 Server,并将这个 Server 相关信息设置成下一次要投票的 Server。
3. 计算这过程中获得票数最多的的 sever 为获胜者,如果获胜者的票数超过半数,则改server 被选为 leader。否则,继续这个过程,直到 leader 被选举出来.
4. leader 就会开始等待 server 连接.
5. Follower 连接 leader,将最大的 zxid 发送给 leader.
6. Leader 根据 follower 的 zxid 确定同步点,至此选举阶段完成。
7. 选举阶段完成 Leader 同步后通知 follower 已经成为 uptodate 状态.
8. Follower 收到 uptodate 消息后,又可以重新接受 client 的请求进行服务了.

```xml
<!--目前有 5 台服务器,每台服务器均没有数据,它们的编号分别是 1,2,3,4,5,按编号依次启动,选举过程如下-->
1. 服务器 1 启动,给自己投票,然后发投票信息,由于其它机器还没有启动所以它收不到反馈信息,服务器 1 的状态一直属于 Looking。
2. 服务器 2 启动,给自己投票,同时与之前启动的服务器 1 交换结果,由于服务器 2 的编号大所以服务器 2 胜出,但此时投票数没有大于半数,所以两个服务器的状态依然是LOOKING。
3. 服务器 3 启动,给自己投票,同时与之前启动的服务器 1,2 交换信息,由于服务器 3 的编号最大所以服务器 3 胜出,此时投票数正好大于半数,所以服务器 3 成为领导者,服务器1,2 成为小弟。
4. 服务器 4 启动,给自己投票,同时与之前启动的服务器 1,2,3 交换信息,尽管服务器 4 的编号大,但之前服务器 3 已经胜出,所以服务器 4 只能成为小弟。
5. 服务器 5 启动,后面的逻辑同服务器 4 成为小弟。
```

## Zookeeper 工作原理(原子广播)

1. Zookeeper 的核心是原子广播,这个机制保证了各个 server 之间的同步。实现这个机制的协议叫做 Zab 协议。Zab 协议有两种模式,它们分别是恢复模式和广播模式。
2. 当服务启动或者在领导者崩溃后,Zab 就进入了恢复模式,当领导者被选举出来,且大多数 server 的完成了和 leader 的状态同步以后,恢复模式就结束了。
3. 状态同步保证了 leader 和 server 具有相同的系统状态
4. 一旦 leader 已经和多数的 follower 进行了状态同步后,他就可以开始广播消息了,即进入广播状态。这时候当一个 server 加入 zookeeper 服务中,它会在恢复模式下启动,发现 leader,并和 leader 进行状态同步。待到同步结束,它也参与消息广播。Zookeeper服务一直维持在 Broadcast 状态,直到 leader 崩溃了或者 leader 失去了大部分的followers 支持。
5. 广播模式需要保证 proposal 被按顺序处理,因此 zk 采用了递增的事务 id 号(zxid)来保证。所有的提议(proposal)都在被提出的时候加上了 zxid。
6. 实现中 zxid 是一个 64 为的数字,它高 32 位是 epoch 用来标识 leader 关系是否改变,每次一个 leader 被选出来,它都会有一个新的 epoch。低 32 位是个递增计数。
7. 当 leader 崩溃或者 leader 失去大多数的 follower,这时候 zk 进入恢复模式,恢复模式需要重新选举出一个新的 leader,让所有的 server 都恢复到一个正确的状态。