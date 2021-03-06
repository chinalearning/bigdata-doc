CAP原理和数据存储方案选择  
====================

Date: 2014-03-11  
Title: CAP原理和数据存储方案选择  
Published: true  
Type: post  
Excerpt:   

在理论计算机科学中，CAP定理，又被称作布鲁尔定理（Brewer's theorem），它指出对于一个分布式计算系统来说，不可能同时满足以下三点：
一致性（Consistency)：所有节点在同一时间具有相同的数据；
可用性（Availability）：保证每个请求不管成功或者失败都有响应；
分隔容忍（Partition tolerance）：系统中任意信息的丢失或失败不会影响系统的继续运作；# 一致性模型（Consistency）数据的一致性，一般来说有三种类型：弱一致性、最终一致性和强一致性。<!--当然，如果细分的话，还有很多一致性模型，如：顺序一致性、FIFO一致性、会话一致性，单读一致性，单写一致性等等。-->* 弱一致性（Weak）：当你写入一个新值后，读操作在数据副本上可能读出来，也可能读不出来。比如：某些Cache系统，VOIP系统，或是百度搜索引擎；  * 最终一致性（Eventually）：当你写入一个新值后，有可能读不出来，但在某个时间窗口之后保证最终能读出来。比如：DNS、电子邮件系统、Amazon S3、Google搜索引擎。* 强一致性（Strong）：新的数据一旦写入，在任意副本任意时刻都能读到新值。比如：文件系统、关系型数据库、Azure Table都是强一致性的。从这三种一致型的模型我们可以看到，弱一致性和最终一致性一般来说是异步冗余的，而强一致性一般来说是同步冗余的。需要注意的是，异步的通常意味着更好的性能，但也意味着更复杂的状态控制。同步意味着简单，但也意味着性能下降。# Master-Slave架构首先是Master-Slave结构，对于这种架构，Slave一般是Master的备份。在这样的系统中，一般是如下设计的：

* 读写请求都由Master负责。
* 写请求写到Master上后，由Master同步到Slave上。

从Master同步到Slave上，你可以使用异步，也可以使用同步，可以使用Master来push，也可以使用Slave来pull。 通常来说是Slave来周期性的pull，所以，是最终一致性。这个设计的问题是，如果Master在pull周期内垮掉了，那么会导致这个时间片内的数据丢失。如果你不想让数据丢掉，Slave只能成为Read-Only的方式等Master恢复。

当然，如果你可以容忍数据丢掉的话，你可以马上让Slave代替Master工作（对于只负责计算的结点来说，没有数据一致性和数据丢失的问题，Master-Slave的方式就可以解决单点问题了） 当然，Master Slave也可以是强一致性的， 比如：当我们写Master的时候，Master负责先写自己，等成功后，再写Slave，两者都成功后返回成功，整个过程是同步的，如果写Slave失败了，那么两种方法，一种是标记Slave不可用报错并继续服务（等Slave恢复后同步Master的数据，可以有多个Slave，这样少一个，还有备份，就像前面说的写三份那样），另一种是回滚自己并返回写失败。（注：一般不先写Slave，因为如果写Master自己失败后，还要回滚Slave，此时如果回滚Slave失败，就得手工订正数据了）你可以看到，如果Master-Slave需要做成强一致性有多复杂。
# Master-Master架构
Master-Master，又叫Multi-master，是指一个系统存在两个或多个Master，每个Master都提供read-write服务。这个模型是Master-Slave的加强版，数据间同步一般是通过Master间的异步完成，所以是最终一致性。 Master-Master的好处是，一台Master挂了，别的Master可以正常做读写服务，他和Master-Slave一样，当数据没有被复制到别的Master上时，数据会丢失。很多数据库都支持Master-Master的Replication的机制。

另外，如果多个Master对同一个数据进行修改的时候，这个模型的恶梦就出现了——对数据间的冲突合并，这并不是一件容易的事情。看看Dynamo的Vector Clock的设计（记录数据的版本号和修改者）就知道这个事并不那么简单，而且Dynamo对数据冲突这个事是交给用户自己搞的。就像我们的SVN源码冲突一样，对于同一行代码的冲突，只能交给开发者自己来处理。（在本文后后面会讨论一下Dynamo的Vector Clock）

# 两段式提交（及三段式提交）

这个协议的缩写又叫2PC，中文叫两阶段提交。在分布式系统中，每个节点虽然可以知晓自己的操作时成功或者失败，却无法知道其他节点的操作的成功或失败。当一个事务跨越多个节点时，为了保持事务的ACID特性，需要引入一个作为协调者的组件来统一掌控所有节点(称作参与者)的操作结果并最终指示这些节点是否要把操作结果进行真正的提交(比如将更新后的数据写入磁盘等等)。 两阶段提交的算法如下：

第一阶段：

* 协调者会问所有的参与者结点，是否可以执行提交操作。
* 各个参与者开始事务执行的准备工作：如：为资源上锁，预留资源，写undo/redo log……
* 参与者响应协调者，如果事务的准备工作成功，则回应“可以提交”，否则回应“拒绝提交”。

第二阶段

* 如果所有的参与者都回应“可以提交”，那么，协调者向所有的参与者发送“正式提交”的命令。参与者完成正式提交，并释放所有资源，然后回应“完成”，协调者收集各结点的“完成”回应后结束这个Global Transaction。
* 如果有一个参与者回应“拒绝提交”，那么，协调者向所有的参与者发送“回滚操作”，并释放所有资源，然后回应“回滚完成”，协调者收集各结点的“回滚”回应后，取消这个Global Transaction。

![image](image/cap01.png)

我们可以看到，2PC说白了就是第一阶段做Vote，第二阶段做决定的一个算法，也可以看到2PC这个事是强一致性的算法。在前面我们讨论过Master-Slave的强一致性策略，和2PC有点相似，只不过2PC更为保守一些——先尝试再提交。 2PC用的是比较多的，在一些系统设计中，会串联一系列的调用，比如：A -> B -> C -> D，每一步都会分配一些资源或改写一些数据。比如我们B2C网上购物的下单操作在后台会有一系列的流程需要做。如果我们一步一步地做，就会出现这样的问题，如果某一步做不下去了，那么前面每一次所分配的资源需要做反向操作把他们都回收掉，所以，操作起来比较复杂。现在很多处理流程（Workflow）都会借鉴2PC这个算法，使用 try -> confirm的流程来确保整个流程的能够成功完成。 举个通俗的例子，西方教堂结婚的时候，都有这样的桥段：

* 牧师分别问新郎和新娘：你是否愿意……不管生老病死……（询问阶段）
* 当新郎和新娘都回答愿意后（锁定一生的资源），牧师就会说：我宣布你们……（事务提交）这是多么经典的一个两阶段提交的事务处理。 另外，我们也可以看到其中的一些问题， A）其中一个是同步阻塞操作，这个事情必然会非常大地影响性能。 B）另一个主要的问题是在TimeOut上，比如：
* 如果第一阶段中，参与者没有收到询问请求，或是参与者的回应没有到达协调者。那么，需要协调者做超时处理，一旦超时，可以当作失败，也可以重试。* 如果第二阶段中，正式提交发出后，如果有的参与者没有收到，或是参与者提交/回滚后的确认信息没有返回，一旦参与者的回应超时，要么重试，要么把那个参与者标记为问题结点剔除整个集群，这样可以保证服务结点都是数据一致性的。* 糟糕的情况是，第二阶段中，如果参与者收不到协调者的commit/fallback指令，参与者将处于“状态未知”阶段，参与者完全不知道要怎么办，比如：如果所有的参与者完成第一阶段的回复后（可能全部yes，可能全部no，可能部分yes部分no），如果协调者在这个时候挂掉了。那么所有的结点完全不知道怎么办（问别的参与者都不行）。为了一致性，要么死等协调者，要么重发第一阶段的yes/no命令。
两段提交最大的问题就是第三项，如果第一阶段完成后，参与者在第二阶没有收到决策，那么数据结点会进入“不知所措”的状态，这个状态会block住整个事务。也就是说，协调者Coordinator对于事务的完成非常重要，Coordinator的可用性是个关键。 因些，我们引入三段提交，三段提交在Wikipedia上的描述如下，他把二段提交的第一个段break成了两段：询问，然后再锁资源。最后真正提交。三段提交的示意图如下：![image](image/cap02.png)
三段提交的核心理念是：** 在询问的时候并不锁定资源，除非所有人都同意了，才开始锁资源。**

理论上来说，如果第一阶段所有的结点返回成功，那么有理由相信成功提交的概率很大。这样一来，可以降低参与者Cohorts的状态未知的概率。也就是说，一旦参与者收到了PreCommit，意味他知道大家其实都同意修改了。这一点很重要。下面我们来看一下3PC的状态迁移图：（注意图中的虚线，那些F,T是Failuer或Timeout，其中的：状态含义是 q – Query，a – Abort，w – Wait，p – PreCommit，c – Commit）

![image](image/cap03.png)

# PAXOS算法

Paxos 算法解决的问题是在一个可能发生上述异常的分布式系统中如何就某个值达成一致，保证不论发生以上任何异常，都不会破坏决议的一致性。一个典型的场景是，在一个分布式数据库系统中，如果各节点的初始状态一致，每个节点都执行相同的操作序列，那么他们最后能得到一个一致的状态。为保证每个节点执行相同的命令序列，需要在每一条指令上执行一个“一致性算法”以保证每个节点看到的指令一致。一个通用的一致性算法可以应用在许多场景中，是分布式计算中的重要问题。从二十世纪八十年代起对于一致性算法的研究就没有停止过。

Paxos算法是莱斯利·兰伯特（Leslie Lamport，就是 LaTeX 中的”La”，现就职于微软研究院）于1990年提出的一种基于消息传递的一致性算法。由于算法难以理解，起初并没有引起人们的重视，使Lamport在八年后1998年重新发表到ACM Transactions on Computer Systems上（The Part-Time Parliament）。即便如此Paxos算法还是没有得到重视，2001年Lamport 觉得同行无法接受他的幽默感，于是用容易接受的方法重新表述了一遍（Paxos Made Simple）。可见Lamport对Paxos算法情有独钟。近几年Paxos算法的普遍使用也证明它在分布式一致性算法中的重要地位。2006年Google的三篇论文（HDFS、Map-Reduce、Bigtable）初现“云”的端倪，其中的Chubby Lock服务使用Paxos作为Chubby Cell中的一致性算法，Paxos算法开始引起了大家广泛的注意。<!-- Lamport 本人在 他的blog 中描写了他用9年时间发表这个算法的前前后后。-->

简单说来，Paxos的目的是让整个集群的结点对某个值的变更达成一致。Paxos算法基本上来说是个民主选举的算法——大多数的决定会成个整个集群的统一决定。任何一个点都可以提出要修改某个数据的提案，是否通过这个提案取决于这个集群中是否有超过半数的结点同意（所以Paxos算法需要集群中的结点是单数）。这个算法有两个阶段（假设这个有三个结点：A，B，C）：

** 第一阶段：Prepare阶段 **

A把申请修改的请求Prepare Request发给所有的结点A，B，C。注意，Paxos算法会有一个Sequence Number（你可以认为是一个提案号，这个数不断递增，而且是唯一的，也就是说A和B不可能有相同的提案号），这个提案号会和修改请求一同发出，任何结点在“Prepare阶段”时都会拒绝其值小于当前提案号的请求。所以，结点A在向所有结点申请修改请求的时候，需要带一个提案号，越新的提案，这个提案号就越是是最大的。如果接收结点收到的提案号n大于其它结点发过来的提案号，这个结点会回应Yes（本结点上最新的被批准提案号），并保证不接收其它<n的提案。这样一来，结点上在Prepare阶段里总是会对最新的提案做承诺。

优化：在上述 prepare 过程中，如果任何一个结点发现存在一个更高编号的提案，则需要通知 提案人，提醒其中断这次提案。

** 第二阶段：Accept阶段 **

如果提案者A收到了超过半数的结点返回的Yes，然后他就会向所有的结点发布Accept Request（同样，需要带上提案号n），如果没有超过半数的话，那就返回失败。当结点们收到了Accept Request后，如果对于接收的结点来说，n是最大的了，那么，它就会修改这个值，如果发现自己有一个更大的提案号，那么，结点就会拒绝修改。

我们可以看以，这似乎就是一个“两段提交”的优化。其实，2PC/3PC都是分布式一致性算法的残次版本，Google Chubby的作者Mike Burrows说过这个世界上只有一种一致性算法，那就是Paxos，其它的算法都是残次品。我们还可以看到：对于同一个值的在不同结点的修改提案就算是在接收方被乱序收到也是没有问题的。

关于一些实例，你可以看一下Wikipedia中文中的“Paxos样例”一节，我在这里就不再多说了。对于Paxos算法中的一些异常示例，大家可以自己推导一下。你会发现基本上来说只要保证有半数以上的结点存活，就没有什么问题。

多说一下，自从Lamport在1998年发表Paxos算法后，对Paxos的各种改进工作就从未停止，其中动作最大的莫过于2005年发表的Fast Paxos。无论何种改进，其重点依然是在消息延迟与性能、吞吐量之间作出各种权衡。为了容易地从概念上区分二者，称前者Classic Paxos，改进后的后者为Fast Paxos。

Amazon的AWS中，所有的云服务都基于一个ALF（Async Lock Framework）的框架实现的，这个ALF用的就是Paxos算法。我在Amazon的时候，看内部的分享视频时，设计者在内部的Principle Talk里说他参考了ZooKeeper的方法，但他用了另一种比ZooKeeper更易读的方式实现了这个算法。

# 超越CAP理论的尝试

## Amazon NWR模型

### NWR 初探


## Google Spanner模型

Spanner是Google的全球级的分布式数据库（Globally-Distributed Database）。Spanner的扩展性达到了令人咋舌的全球级，可以扩展到数百万的机器，数已百计的数据中心，上万亿的行。更给力的是，除了夸张的扩展性之外，他还能同时通过同步复制和多版本来满足外部一致性，可用性也是很好的。冲破CAP的枷锁，在三者之间完美平衡。

Spanner是个可扩展，多版本，全球分布式还支持同步复制的数据库。他是Google的第一个可以全球扩展并且支持外部一致的事务。Spanner能做到这些，离不开一个用GPS和原子钟实现的时间API。这个API能将数据中心之间的时间同步精确到10ms以内。因此有几个给力的功能：无锁读事务，原子schema修改，读历史数据无block。

### F1

和众多互联网公司一样，在早期Google大量使用了Mysql。Mysql是单机的，可以用Master-Slave来容错，分区来扩展。但是需要大量的手工运维工作，有很多的限制。因此Google开发了一个可容错可扩展的RDBMS——F1。和一般的分布式数据库不同，F1对应RDMS应有的功能，毫不妥协。起初F1是基于Mysql的，不过会逐渐迁移到Spannerr。

F1有如下特点：

* 7×24高可用。哪怕某一个数据中心停止运转，仍然可用。
* 可以同时提供强一致性和弱一致。
* 可扩展
* 支持SQL
* 事务提交延迟50-100ms，读延迟5-10ms，高吞吐

众所周知Google BigTable是重要的Nosql产品，提供很好的扩展性，开源世界有HBase与之对应。为什么Google还需要F1，而不是都使用BigTable呢？因为BigTable提供的最终一致性，一些需要事务级别的应用无法使用。同时BigTable还是NoSql，而大量的应用场景需要有关系模型。就像现在大量的互联网企业都使用Mysql而不愿意使用HBase，因此Google才有这个可扩展数据库的F1。而Spanner就是F1的至关重要的底层存储技术。

### Colossus（GFS II）

Colossus也是一个不得不提起的技术。他是第二代GFS，对应开源世界的新HDFS。初代GFS是为批处理设计的。对于大文件很友好，吞吐量很大，但是延迟较高。所以使用他的系统不得不对GFS做各种优化，才能获得良好的性能。那为什么Google没有考虑到这些问题，设计出更完美的GFS ? 因为那个时候是2001年，Hadoop出生是在2007年。如果Hadoop是世界领先水平的话，GFS比世界领先水平还领先了6年。同样的Spanner出生大概是2009年，现在我们看到了论文，估计Spanner在Google已经很完善，同时Google内部已经有更先进的替代技术在酝酿了。笔者预测，最早在2015年才会出现Spanner和F1的山寨开源产品。

Colossus是第二代GFS。Colossus是Google重要的基础设施，因为他可以满足主流应用对FS的要求。Colossus的重要改进有：

* 优雅Master容错处理（不再有2s的停止服务时间）
* Chunk大小只有1MB（对小文件很友好）
* Master可以存储更多的Metadata（当Chunk从64MB变为1MB后，Metadata会扩大64倍，但是Google也解决了）

Colossus可以自动分区Metadata。使用Reed-Solomon算法来复制，可以将原先的3份减小到1.5份，提高写的性能，降低延迟。客户端来复制数据。具体细节笔者也猜不出。

### Spanner与BigTable、MegaStore的对比

Spanner主要致力于跨数据中心的数据复制上，同时也能提供数据库功能。在Google类似的系统有BigTable和Megastore。和这两者相比，Spanner又有什么优势呢。

BigTable在Google得到了广泛的使用，但是他不能提供较为复杂的Schema，还有在跨数据中心环境下的强一致性。Megastore有类RDBMS的数据模型，同时也支持同步复制，但是他的吞吐量太差，不能适应应用要求。Spanner不再是类似BigTable的版本化 key-value存储，而是一个“临时多版本”的数据库。何为“临时多版本”，数据是存储在一个版本化的关系表里面，存储的时间数据会根据其提交的时间打上时间戳，应用可以访问到较老的版本，另外老的版本也会被垃圾回收掉。

Google官方认为 Spanner是下一代BigTable，也是Megastore的继任者。

### Google Spanner的设计

#### 功能

从高层看Spanner是通过Paxos状态机将分区好的数据分布在全球的。数据复制全球化的，用户可以指定数据复制的份数和存储的地点。Spanner可以在集群或者数据发生变化的时候将数据迁移到合适的地点，做负载均衡。用户可以指定将数据分布在多个数据中心，不过更多的数据中心将造成更多的延迟。用户需要在可靠性和延迟之间做权衡，一般来说复制1，2个数据中心足以保证可靠性。

作为一个全球化分布式系统，Spanner提供一些有趣的特性。

* 应用可以细粒度的指定数据分布的位置。精确的指定数据离用户有多远，可以有效的控制读延迟(读延迟取决于最近的拷贝)。指定数据拷贝之间有多远，可以控制写的延迟(写延迟取决于最远的拷贝)。还要数据的复制份数，可以控制数据的可靠性和读性能。(多写几份，可以抵御更大的事故)
* Spanner还有两个一般分布式数据库不具备的特性：读写的外部一致性，基于时间戳的全局的读一致。这两个特性可以让Spanner支持一致的备份，一致的MapReduce，还有原子的Schema修改。

这写特性都得益有Spanner有一个全球时间同步机制，可以在数据提交的时候给出一个时间戳。因为时间是系列化的，所以才有外部一致性。这个很容易理解，如果有两个提交，一个在T1,一个在T2。那有更晚的时间戳那个提交是正确的。

这个全球时间同步机制是用一个具有GPS和原子钟的TrueTime API提供了。这个TrueTime API能够将不同数据中心的时间偏差缩短在10ms内。这个API可以提供一个精确的时间，同时给出误差范围。Google已经有了一个TrueTime API的实现。笔者觉得这个TrueTime API 非常有意义，如果能单独开源这部分的话，很多数据库如MongoDB都可以从中受益。

#### 体系架构

Spanner由于是全球化的，所以有两个其他分布式数据库没有的概念。

* Universe：一个Spanner部署实例称之为一个Universe。目前全世界有3个。一个开发，一个测试，一个线上。因为一个Universe就能覆盖全球，不需要多个。
* Zones：每个Zone相当于一个数据中心，一个Zone内部物理上必须在一起。而一个数据中心可能有多个Zone。可以在运行时添加移除Zone。一个Zone可以理解为一个BigTable部署实例













