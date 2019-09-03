顺序一致性
从同一个客户端发起的事务请求，最终将会严格按照其发起顺序被应用到ZooKeeper中。
在ZooKeeper中，有三种角色：
* Leader
* Follower
* Observer
  zookeeper-server status 可以看当前节点的ZooKeeper是什么角色
  ZooKeeper默认只有Leader和Follower两种角色，没有Observer角色。
  为了使用Observer模式，在任何想变成Observer的节点的配置文件中加入：peerType=observer
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

第一阶段：通过Acceptor.prepare()获取互斥访问权和当前var取值，如果无法获取，返回<error>（锁被别人占用）。
第二阶段：根据当前var的取值f，选择执行：
	•	如果f为null（以前没有proposal设置过var的值），则通过Acceptor.accept(var, V)提交数据；
	•	如果f不为null（接受某个proposal设置过var的值f），则通过Acceptor.release()释放访问权，返回<ok, f>。

但不足的是，Proposal在获取到互斥访问权后，并且还没有释放互斥访问权之前发生了故障，则会导致其他所有Proposal无法获取互斥访问权进入第二阶段运行，此时会导致系统陷入“死锁”的状态。即，方案一无法容忍任意Proposal机器出现故障。

引入抢占式访问权 —— 方案二
对Proposal来说，Proposal向Acceptor申请访问权时会指定编号epoch（越大的epoch越新）。获取到访问权以后，才能向Acceptor提交取值。
对Acceptor来说，采用“喜新厌旧”的原则。即一旦收到更大的新epoch的申请，马上让旧epoch的访问权失效，不再接受他们提交的取值，然后给新的epoch发放访问权限，只接受新的epoch提交的取值。

第一阶段：获取epoch轮次的访问权和当前var的取值，我们可以简单获取当前时间戳为epoch：
	•	通过Acceptor.prepare(epoch)，获取epoch轮次的访问权和当前var的取值；
	•	如果不能获取，返回<error>。
	•	第二阶段：采用“后者认同前者”的原则执行。
	•	如果var取值为null，则肯定旧epoch无法生成确定性取值，则通过Acceptor.accept(var, prepared_epoch, V)提交数据V。
	•	成功后返回<ok, V>；
	•	若accept失败，则说明被更新的epoch抢占或Acceptor故障，返回<error>。
	•	如果var取值存在，则此取值肯定是确定性取值，此时认同它不再更改，直接返回<ok, accepted_value>。
但方案二中只有一个Acceptor，单机模块的Acceptor故障时会导致整个系统宕机，无法提供服务，所以我们仍需要引入多个Acceptor。
Paxos是在方案二的基础上引入多个Acceptor，在paxos方案里面，Acceptor的实现保持不变，仍然采用“喜新厌旧”的原则运行。由于引进多个Acceptor，所以采用“少数服从多数”的思路。一旦某个epoch的取值f被半数以上Acceptor接受，我们就认为这个var的取值被确定为f，不再更改。
第一阶段：选定epoch，获取epoch访问权和对应的var取值，在paxos中需要获取半数以上Acceptor的访问权和对应一组var取值。

第二阶段：采用“后者认同前者”的原则执行。
	•	如果获取的var取值都为null，则旧的epoch无法形成确定性取值，此时努力使<epoch, V>成为确定性取值：
	•	向epoch对应的所有Acceptor提交取值<epoch, V>；
	•	如果收到半数以上成功，则返回<ok, V>；
	•	否则返回<error>（被新的epoch抢占或Acceptor故障）。
	•	如果var的取值存在，认同最大的accepted_epoch对应的取值f，努力使<epoch, f>成为确定性取值。
	•	如果f出现半数以上，则说明f已经是确定性取值，直接返回ok, f<>；
	•	否则，向epoch对应的所有Acceptor提交取值<epoch, f>。

## multi-poxos

当存在一批提案时，用Basic-Paxos一个一个决议当然也可以，但是每个提案都经历两阶段提交，显然效率不高。Basic-Paxos协议的执行流程针对每个提案（每条redo log）都至少存在三次网络交互：1. 产生log ID；2. prepare阶段；3. accept阶段。

Mulit-Paxos基于Basic-Paxos做了优化，在Paxos集群中利用Paxos协议选举出唯一的leader，在leader有效期内所有的议案都只能由leader发起。

这里强化了协议的假设：即leader有效期内不会有其他server提出的议案。因此，对于后续的提案，我们可以简化掉产生log ID阶段和Prepare阶段，而是由唯一的leader产生log ID，然后直接执行Accept，得到多数派确认即表示提案达成一致（每条redo log可对应一个提案）。

----

Paxos和raft都是一旦一个entries（raft协议叫日志，paxos叫提案，叫法而已）得到多数派的赞成，这个entries就会定下来，不丢失，值不更改，最终所有节点都会赞成它。Paxos中称为提案被决定，Raft,ZAB,VR称为日志被提交，**Multi-paxos和Raft都用一个数字来标识leader的合法性**，multi-paxos中叫proposer-id，Raft叫term，意义是一样的，**multi-paxos proposer-id最大的Leader提出的决议才是有效的，raft协议中term最大的leader才是合法的**。

raft协议在leader选举阶段，由于老leader可能也还存活，也会存在不只一个leader的情形，**只是不存在term一样的两个leader**，因为选举算法要求leader得到同一个term的多数派的同意，同时赞同的成员会承诺不接受term更小的任何消息。这样可以根据term大小来区分谁是合法的leader。Multi-paxos的区分leader的合法性策略其实是一样的，谁的proproser-id大谁合法，而proposer-id是唯一的。**因此它们其实在同一个时刻，都只会存在一个合法的leader。**

multi-paxos原本的意思是，即使形成了多数派，仍然需要使用新的proposalID走一遍prepare-accept过程，工程上做了优化之后，对于明确标记了已形成多数派的entry可以不重新投票，但是未确认是否形成多数派的entry，则要求新任leader使用新的proposalID重新投票。

因为multi-paxos并不假设日志间有连续确认的关系，每条日志之间相互独立，没有关系，1号日志尚未确认时，2号日志就可以确认形成多数派备份.

----



## Raft

如果 follower 宕机或者运行缓慢或者丢包，领导人会不断的重试，知道所有的 follower 最终都存储了所有的日志条目。

当 leader 和 follower 日志冲突的时候，leader 将校验 follower 最后一条日志是否和 leader 匹配，如果不匹配，将递减查询，直到匹配，匹配后，删除冲突的日志。这样就实现了主从日志的一致性。

1. raft协议的Leader选举算法，新选举出的Leader已经拥有全部的可以被提交的日志，而multi-paxos择不需要保证这一点，这也意味multi-paxos需要额外的流程从其它节点获取已经被提交的日志。因此raft协议日志可以简单的只从leader流向follower在raft协议中，而multi-paxos则需要额外的流程补全已提交的日志。
2. **Raft协议强调日志的连续性，multi-paxos则允许日志有空洞**。**日志的连续性蕴含了这样一条性质：如果两个不同节点上相同序号的日志，term相同，那么这和这之前的日志必然也相同的。**raft协议利用日志的连续性，leader可以很方便的得知自己的follower拥有的日志的情况，Follower只要告诉Leader自己本地日志文件的最后一个日志的序号和term就可以了；
3. raft协议在leader选举的时候，只需要从一个多数集中选择最后出最后一条日志term最大并且日志数目最多的节点，新leader同样必定拥有所有的已commit的日志。**这是由于任意一条commit的日志，至少被多数派记录，而由于日志的连续性，拥有最后一条commit的日志也就意味着拥有全部的commit日志，因此raft协议中一个多数派必然存在一个节点拥有全部的已提交的日志**。而对于multi-paxos每个日志需要单独被确认是否可以提交，因此当新leader产生后，它只好重新对每个日志进行确认，已确定它们是否可以被提交，甚至于新leader可能缺失可以被提交的日志，需要向其它节点学习到缺失的可以被提交的日志，当然这都可以通过向一个多数派询问完成

## ZAB

leader周期成为epoch,raft为term

raft心跳方向leader->follower,ZAB相反。

- Zk中的读请求，直接由连接的Node处理，不需要和leader汇报，也就是Consul中的stale模式。这种模式可能导致读取到的数据是过时的，但是可以保证一定是半数节点之前确认过的数据
- 为了避免Follower的数据过时，Zk有sync()方法，可以保证读取到最新的数据。可以调用sync()之后，再查询，确保所有的数据一致后再返回结果

1. 角色Zk引入了 Observer的角色来提升性能，既可以大幅提升读取的性能，也可以不影响写的速度和选举的速度，同时一定程度上增加了容错的能力。

Zk集群之间投票消息是单向、网状的，类似于广播，比如A广播A投票给自己，广播出去，然后B接收到A的这个消息之后，会PK A的数据，如果B更适合当leader（数据更新或者myid更大），B会归档A的这个投票，但是不会更新自己的数据，也不会广播任何消息。除非发现A的数据比B当前存储的数据更适合当leader，就更新自己的数据，且广播自己的最新的投票消息。
而Raft集群之间的所有消息都是双向的，发起一个RPC，会有个回复结果。比如A向B发起投票，B要么反馈投票成功，要么反馈投票不成功。

ZK集群中，因为引入了myid的概念，系统倾向让数据最新和myid最大的节点当leader，所以即使有半数节点都投票给同一个Node当leader，这个Node也不一定能成为leader，需要等待200ms，看是不是有更适合的leader产生，

1. Raft的集群模式下：
   Leader创建日志，广播日志，半数节点复制成功后，自己commit日志，运用到状态机中，反馈客户端，并且在下一个心跳包中，通知小弟们commit

Zab的集群模式下：
leader创建Proposal，广播之后，半数节点复制成功后，广播commit。同时自己也commit，commit完之后再运用到内存树，反馈客户端