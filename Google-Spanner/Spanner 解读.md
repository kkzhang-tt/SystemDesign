## 前言

最近重读了Spanner的论文，之前所困惑的Commit Wait等概念终于有了新的理解，也发现了文章中的更多细节。这里整理出十个重点问题，旨在理清诸多设计细节。

本文的定位不是科普性质，需要对Spanner的论文有深入研读才能理解，在此之前最好先读一下paper，以及知乎上的许多解读。

## 1. Replication & Sharding

![img](https://pic4.zhimg.com/80/v2-419454927f4e62c1dbe7d3b6359ff113_1440w.jpg)

![img](https://pic3.zhimg.com/80/v2-10d5300b36a81f9ffda730064c1cd60e_1440w.jpg)

首先是基本的Sharding和Replication问题。

逻辑上最小的分片单元是Directory（实际更小的是Fragment不过不影响讨论暂且不提），Directory就是按照Range进行Sharding之后得到的连续区间，出于负载均衡和迁移的考虑，Directory通常不会太大，几十MB的数据在几秒内可以完成迁移。

每个Directory会归属到唯一的Paxos Group中，不过会时常在Paxos Group之间迁移，比如某个Paxos Group的负载太高，可以把其中一些Directory迁移到其他的Paxos Group；或者出于局部性的考虑，把两个Directory放到一个Paxos Group，那么对这两个Directory的操作就不需要分布式事务也不需要额外的网络开销了。至于Directory的迁移、分裂，也是老生常谈的问题，暂且不表。

Paxos Group，顾名思义就是一个Paxos复制组。Spanner实现的是Multi-Paxos，会有一个long-live的leader，leader同时也会维护一个`lock table`和`transaction manager`，来协调单机事务和分布式事务。Spanner对Paxos的实现提及不多，只是说会pipeline复制、乱序commit、顺序apply，不过想必这里的实现也是相当复杂的。

再往上层，则是Paxos Group和机器甚至数据中心的映射关系。一个机器上会有几百个Paxos Group，每个Paxos Group的多个副本也会分布在不同的Region以实现容灾和地域局部性的效果。

虽然分布式系统的Sharding&Replication看起来都大同小异，但是深究一下会发现Spanner这样的设计还是有一些亮点的：

1. failover：未实现Paxos Group的成员变更，而是通过Directory在Paxos Group之间的迁移来达到故障failover、负载均衡等目的；众所周知，实现复制协议的成员变更不是一个简单的事情，需要保证变更的原子性、非阻塞，而借助原生支持的Transaction，实现Directory的迁移相对简单
2. 网络连接：朴素的实现中，每个paxos group中leader都会建立到follower的连接，如果group增加，网络连接的数量也会非常可观；解法是像multi-raft那样，合并网络连接；不过在spanner这样几百个paxos group的场景下，网络连接不会成为问题
3. 磁盘随机IO：每个paxos group都会写log，如果用单独的文件，会带来大量的随机IO
4. 恢复速度&可靠性：发生节点故障时，数据恢复以paxos 为单位，太少的paxos group会限制恢复速度，降低可靠性
5. 分布式事务：paxos group越多， 通常多行记录的修改在不同的paxos group里的概率越高，因此较少的paxos group能够减少分布式事务；spanner的paxos group足够大，能够减少跨越paxos group的分布式事务
6. 局部索引：局部索引在更新时不需要分布式事务，通常认为具有较好的性能；但同时也会造成scatter query，把一个查询发到所有shard去执行；在paxos group较少的情况下，这个开销能够接受
7. 负载均衡：如果一个节点上的paxos group太少，容易因为某个paxos group过热而导致节点的负载不均，在几百个paxos group的情况下能够比较好地平衡这一点

## 2. 2PC如何设计

![img](https://pic4.zhimg.com/80/v2-4c80410b4a7d567712a486e38321f93f_1440w.jpg)



接下来是分布式事务的2PC问题。

Spanner面对的是跨区域复制的场景，首先面临的一个挑战就是广域网的延迟问题。所以Spanner的2PC做了一个比较特别的设计：

1. Client选择事务的某一个Paxos Group作为Coordinator，其他的作为Participant
2. 发送Prepare请求给所有的Participant和Coordinator，对于Coordinator要告知所有Participant信息，而Participant的消息中要告知Coordinator信息
3. Participant进行资源申请，并记下Prepare Log
4. Coordinator不写Prepare Log，而是等待所有Participant的Prepared消息
5. Coordinator收到所有Participant的Prepared消息之后，申请自己的资源，连带自己的Prepare以及Commit一起持久化，然后告知Client成功Commit
6. 异步通知所有Participant，释放资源

由于最初的Prepare请求都是由Client广播，而非Client发送给Coordinator再进行广播，可以减少广域网下消息延迟；而Coordinator兼具Participant的角色，会把Prepare和Commit记录一起持久化。

2PC的正常流程非常简单，接下来分析一下异常流程下的状态转换，此部分论文没有详细描述，纯属笔者推断。

## Participant

![img](https://pic1.zhimg.com/80/v2-9cd4022ea38d1ca73d4ccc00bdb04008_1440w.jpg)



Participant的状态完全由Client和Coordinator来驱动，无权自行修改。正常流程会依次执行：

1. 收到来自Client的Prepare消息
2. 发送Prepared消息给Coordinator
3. 收到来自Coordinator的Commit消息
4. 清理事务状态

注意在Prepared状态下，需要轮询Coordinator，这是为了处理Coordinator重启，无人驱动状态变更的场景。

这里考虑了事务状态的清理，最终到达Purged状态，也是一次Paxos Write。在此之后，如果因为消息的延迟和重复，再次收到这条事务的任何消息，会再次从Absent状态处理。

## Coordinator

![img](https://pic3.zhimg.com/80/v2-484a02b587400dec0865878c97c185de_1440w.jpg)



Coordinator进入Committing状态有两种可能，收到Client的Prepare，或者Participant的Prepared；由于在Commit之前没有进行状态持久化，如果发生重启会再次回到Absent状态，这里需要保证幂等性。

Coordinator在清理事务状态之前，需要等待所有Participant确认这个状态，并且是Participant持久化了Commit/Abort消息之后，否则会出现无法判断事务状态的情况。

## 正确性

正确性包括两方面，safety和liveness。safety指的是状态不会出错，在事务的语境下则是答复给Client的Commit或者Abort不会回滚；liveness则是面对各种异常，事务最终能够Commit或者Abort。

安全性首先来自于异常，一方面是节点故障，虽然用Paxos做了复制，但每次failover之后，只能回到上一次持久化的状态；另一方面是网络异常，也就是分布式系统中典型的消息重复、延迟、丢失、乱序，这个对状态机的设计是一个挑战。

上面的2PC过程中持久化状态，是为了应对节点故障：事务推进到一个状态并持久化之后，不会发生状态的回退。基于这一点，再保证liveness：处于Prepared状态下的Participant，会轮询Coordinator，确保Participant的liveness；处于Committed状态下的Coordinator，会轮询Participant，确保自己的liveness。

## 1PC

在事务仅有一个Paxos Group参与的情况下，可以优化成1PC，获得性能方面的提升。

Coordinator不需要做特殊处理，1PC的区别在于，Participant只有它自己，因此： 1. Coordinator不需要在Committing状态等待，而是可以直接写Prepare&Commit，然后回复给客户端 2. 从Committed状态进入Completed状态也无须等待

## 3. Read-Write Transaction

## Transaction Isolation

Spanner实现了两种事务，一种是Read-Write，另一种是Read-Only。Read-Write用经典的两阶段的读写锁来实现，所有加锁请求先于解锁请求；Read-Only则使用Snapshot Isolation的技术，在开始读之前获得时间戳，读可见性由事务的读时间戳和数据的写时间戳判定。

但Spanner实现的隔离级别和传统的理解却有一些差异：如果说RW事务是`Serializable`的，它又没实现Gap Lock之类的技术来解决Phantom；如果说RO事务是Snapshot Isolation的，又不会出现write skew。至于具体是什么隔离级别暂且不讨论，不做文字之争。

## S2PL

S2PL说来也简单，无非是读之前加读锁，写之前加写锁，在释放锁之后不可以再加锁，写锁需要等到事务Commit之后再释放。在Spanner的语境下，还需要考虑这几点：

1. 如何与2PC结合
2. 锁要持久化吗
3. 死锁如何处理

与2PC的结合顺理成章，Prepare阶段申请锁，写Prepare Log；Commit之后释放锁，写Commit Log，并且Apply数据变更。

![img](https://pic2.zhimg.com/80/v2-6134c3ce3925348ae9862d138d03cb35_1440w.jpg)



至于持久化，写锁自然是持久化的，因为写了Prepare Log，在Crash Recover的时候会把锁恢复出来；而读锁，是不做持久化的，也就是说不会在Prepare Log中记录。这是通过Commit之前的Check实现的：

1. 读阶段：在Participant Leader上获得读锁，然后读数据；这个读锁仅存在Participant Leader的内存中，不做复制不做持久化
2. Prepare阶段：在Participant Leader获得写锁，写Prepare Log
3. Commit Check：当所有Participant 都完成Prepare之后，检查所有读节点的读锁是否仍然存在，如果存在则释放，如果不存在则Abort当前事务
4. Commit阶段：写Commit Log，Apply 数据改动，释放写锁

这个Check保证了S2PL要求的所有加锁请求先于解锁请求，注意并不要求读锁也持有到Commit之后，因此在Prepare完成Commit之前就可以释放读锁了。

死锁处理使用伤害等待（`wound-wait`）的方式，新的事务会等待旧事务，而旧事务杀死新事务，事务被杀死后等待一段时间后用旧的时间戳重启，保证不会饿死也不会出现死锁。但在2PC的场景下，显然Prepare的事务是不能被杀死的，因为只有Coordinator才有权利把Prepared的事务Abort掉；因此按我的理解，这里的伤害等待指的是写锁遇到读锁的冲突，如果持有写锁的事务时间比较旧，就释放掉这个读锁，当持有读锁的事务在Commit Check的时候就能够发现；如果持有写锁的事务时间戳比较新，则等待这个读锁释放。其他任何情况，例如写锁和写锁的冲突，读锁和写锁的冲突，都是不能杀死写锁的，而是只能选择Abort。

按经典的理论，`wound-wait`的abort rate会比较低，考虑到2PC的特殊性，实施wound-wait的代价又会比较大，因此在很多时候还得使用`wait-die`的策略。

## 4. TrueTime & External Consistency

以上讲的都是单版本的场景，实际上Spanner实现的是多版本的并发控制，而多版本的重点在于版本管理。

在S2PL中，可以认为所有读写操作都是在加完所有锁、释放锁之前的时间点完成的，而最后的调度顺序等价于按这个时间点的串行调度，也意味着这个时间点成了定义顺序的关键。Spanner保证所谓的外部一致性（External Consistency）：

> 如果一个事务 ![[公式]](https://www.zhihu.com/equation?tex=T_%7Bi%7D) 的Commit先于另一个事务 ![[公式]](https://www.zhihu.com/equation?tex=T_%7Bj%7D) 的开始，那么 ![[公式]](https://www.zhihu.com/equation?tex=T_%7Bi%7D) 的CommitTS小于 ![[公式]](https://www.zhihu.com/equation?tex=T_%7Bj%7D)

也就是在`Serializable`的基础上，保证调度顺序和CommitTS的一致，达到了外部一致性。而外部一致有什么用？简单来说，写进去的数据能够立即被读到，在被修改之前，读到的数据都是一样的。也就是Linearizability + Serializable的效果。那么如何实现？

Spanner用了TrueTime，它提供这样一个API：`TT.now() = [earliest, latest]`，返回最早的物理时间和最晚的物理时间。基于这个API，结合Commit Wait规则实现外部一致性就十分简单，这里假定事务T1的commit先于T2的start，然后证明其CommitTS满足外部一致性，记 ![[公式]](https://www.zhihu.com/equation?tex=t_%7Babs%7D%28e%29) 为事件 ![[公式]](https://www.zhihu.com/equation?tex=e+) 的物理时间， ![[公式]](https://www.zhihu.com/equation?tex=s_%7B1%7D) 为事务T1的commitTS：

1. ![[公式]](https://www.zhihu.com/equation?tex=s_%7B1%7D+%3C+t_%7Babs%7D%28e%5E%7Bcommit%7D_%7B1%7D%29)*)：commit wait规则，先获取*![[公式]](https://www.zhihu.com/equation?tex=s%7B1%7D+%3D+TT.now%28%29.latest+) 作为时间戳，等待 ![[公式]](https://www.zhihu.com/equation?tex=s%7B1%7D+%3D+TT.now%28%29.latest) 之后才commit
2. ![[公式]](https://www.zhihu.com/equation?tex=s%7B1%7D+%3D+TT.now%28%29.latest)*：基本假定，事件*![[公式]](https://www.zhihu.com/equation?tex=e%7B1%7D) 的commit先于 ![[公式]](https://www.zhihu.com/equation?tex=e_%7B2%7D) 的start
3. ![[公式]](https://www.zhihu.com/equation?tex=t_%7Babs%7D%28e%5E%7Bstart%7D%7B2%7D%29+%5Cleq+t%7Babs%7D%28e%5E%7Bserver%7D_%7B2%7D%29) ：基本的因果关系，事务先开始，才发送请求到服务器
4. ![[公式]](https://www.zhihu.com/equation?tex=t_%7Babs%7D%28e%5E%7Bserver%7D%7B2%7D%29+%5Cleq+s%7B2%7D) ：start规则，事务 ![[公式]](https://www.zhihu.com/equation?tex=s_%7B2%7D) 的时间大于等于 ![[公式]](https://www.zhihu.com/equation?tex=TT.now%28%29.latest) ，因此比物理时间要大
5. ![[公式]](https://www.zhihu.com/equation?tex=s_%7B1%7D+%3C+s_%7B2%7D) ：最后根据传递性，得出结论

## Global Snapshot

接下来的问题是，外部一致性和全局一致性快照是什么关系？例如对于一个银行来说，各个账户之间转来转去，要满足的一个性质就是总的资金不变，上面的外部一致性能保证这一点吗？

![img](https://pic1.zhimg.com/80/v2-1788bc4ba2aa20eeccaacc2150289370_1440w.jpg)



以上图为例，`Serializable`的结果等价于各个事务串行执行，最初ABC的值分别是7、6、5；T1执行了`A++, B--`，变成8、5、5，T2执行了`B++, C--`，变成8、6、4；最后T3进行一个summary操作。

如果是串行执行那没有问题，但可串行化调度不等价于串行执行，因此这里蕴含了一个假设，T2能读到T1更改的数据，而不是更早的版本，T3能读到T1和T2分别修改的版本。外部一致性很好地保证了这一点：

1. 事务1的commit先于事务2的start，则事务T1的commit小于事务T2的commit
2. 事务T1的commit小于事务T2的commit，因此T2能读到T1写的数据

这里可能产生一点疑惑，对于只读事务，它的CommitTS是什么？对只读事务来说，称之为 ![[公式]](https://www.zhihu.com/equation?tex=S_%7Bread%7D) 可能更加合适，只要这个 ![[公式]](https://www.zhihu.com/equation?tex=S_%7Bread%7D) 满足外部一致性的要求，它就等价于CommitTS，也就是如果有其他事务的commit先于只读的事务的start，这个 ![[公式]](https://www.zhihu.com/equation?tex=S_%7Bread%7D) 就要大于先前事务的commitTS。

## 5. Read-Only Transaction

Spanner里的另一种事务就是只读事务，用Snapshot Isolation的方式来执行。

这个事务的关键在于如何选择一个 ![[公式]](https://www.zhihu.com/equation?tex=S_%7Bread%7D)，满足外部一致性的要求。最简单的做法，用 ![[公式]](https://www.zhihu.com/equation?tex=TT.now%28%29.latest) ![[公式]](https://www.zhihu.com/equation?tex=TT.now%28%29.latest) ，能满足要求，但是可能会发生阻塞。阻塞有两种来源，一个是并发事务的读写冲突，另一个是复制协议的复制延迟。

读写冲突造成的阻塞：如果只读事务遇到一个正在Prepare但是没有Commit的事务，是否能读它的数据呢，如果读了，那有可能发生脏读，如果不读选择忽略，就会违背可重复读的原则（后续事务提交后读事务就可以读到数据了）。因此，这时候只能选择等待。

复制协议造成的阻塞：在Multi-Paxos中用Lease机制保证读Leader总是最新的，但对于Follower就未必；在Leader已经完成了2PC commit的paxos write之后，follower没有收到这个2PC commit，此时也不能读到最新的数据。

因此，只读事务首先需要选择一个合适的时间戳，然后在相应的节点确认不会发生读写冲突、不会有复制协议的落后的情况下，可以处理这个读请求了。

## 6. ![[公式]](https://www.zhihu.com/equation?tex=S_%7Bread%7D)

先看只读事务时间戳的选择。这个时间戳决定了版本的可见性：如果一个版本的 ![[公式]](https://www.zhihu.com/equation?tex=CommitTS+%5Cleq+S_%7Bread%7D) ，这个版本对只读事务就是可见的。因此这个时间戳直接决定了能否保证外部一致性。举个反例，如果 ![[公式]](https://www.zhihu.com/equation?tex=S_%7Bread%7D+) 非常旧， 就会读不到最新的数据，进而违背了外不一致。

最差情况下，我们用 ![[公式]](https://www.zhihu.com/equation?tex=TT.now%28%29.latest) 作为 ![[公式]](https://www.zhihu.com/equation?tex=S_%7Bread%7D) ，根据前面的 ![[公式]](https://www.zhihu.com/equation?tex=Start) 规则可保证外部一致性，不再赘述。

其次，还存在几个时间可供选择：

1. paxos write时间：每次paxos write时，leader都会生成一个时间戳，记录在paxos entry中；这个时间戳保证单调递增，即便发生failover
2. prepareTS：每次Participant都会提供一个prepareTS，这里等价于写prepare log时的paxos write时间
3. commitTS：事务的CommitTS，由Coordinator选择，大于等于所有Participant的PrepareTS，大于等于 ![[公式]](https://www.zhihu.com/equation?tex=S_%7Bread%7D) ，大于等于先前使用过的paxos write

![img](https://pic4.zhimg.com/80/v2-0a03dc1423badd9f7ab5718d1ebac75f_1440w.jpg)

如上图，这几个时间存在这样一个关系：

1. prepare时，每个Participant提供的prepare时间不一样，等于各自的paxos write时间
2. coordinator commit时，选择的CommitTS大于等于所有的PrepareTS，也等于自己的paxos write时间
3. participant commit时，这时候的paxos write时间会大于CommitTS，因为 ![[公式]](https://www.zhihu.com/equation?tex=TT.after%28CommitTS%29) 之后，coordinator才发送消息给Client和Participant

因此，对于不存在Prepared事务的Leader：

1. 选择Last Commit的paxos write时间即可，记为 ![[公式]](https://www.zhihu.com/equation?tex=LastTS)
2. 这个时间能保证在Leader和Follower上都能读到最新的数据，类似于Raft中读Follower时要从Leader获取CommitIndex再去Follower读

![img](https://pic4.zhimg.com/80/v2-960b0c34d973a5870b0c8194ef704deb_1440w.jpg)



对于Leader存在Prepared事务，能否继续用这个呢？很遗憾不能，例如上图， ![[公式]](https://www.zhihu.com/equation?tex=LastTS+) 就是 ![[公式]](https://www.zhihu.com/equation?tex=Prepare%28txn2%29) 的paxos write时间，问题在于这个事务的CommitTS未知，只能知道 ![[公式]](https://www.zhihu.com/equation?tex=LastTS+%5Cleq+CommitTS) ，而只读事务和这个Prepared事务的先后关系不确定，而这个先后关系后影响到版本的可见性，也就不确定这个版本是否可见。这时候用 ![[公式]](https://www.zhihu.com/equation?tex=LastTS) 作为 ![[公式]](https://www.zhihu.com/equation?tex=S_%7Bread%7D+) 其实就假定了只读事务的start先于Txn2的Commit，而这个假定是不成立的。因此，这种情况下只能退化到最朴素的情况，用 ![[公式]](https://www.zhihu.com/equation?tex=TT.now%28%29.latest) 。

这么一来这个 ![[公式]](https://www.zhihu.com/equation?tex=S_%7Bread%7D) 就很鸡肋了，只要一个Paxos Group有事务未Commit，就得用 ![[公式]](https://www.zhihu.com/equation?tex=TT.now%28%29.latest) ，带来阻塞。不过这里显然可以做另一个优化：

1. 没有读写冲突的事务，不应该被阻塞
2. 因此在lock table中额外维护key range的最大的CommitTS
3. 对于只读事务来说，根据其查询范围在lock table中找到最大的CommitTS，如果key range内没有任何锁，就用最大的CommitTS作为 ![[公式]](https://www.zhihu.com/equation?tex=S_%7Bread%7D+)
4. 这里的原理是，如果某一行数据在被修改，我们不能确定这个版本的可见性；但如果这个数据没有在被修改，那么这个版本一定是可见的

对应到上图的例子，如果只读事务读的是txn1的数据，且和txn2没有冲突，就应该用txn1的commitTS，而不是被txn2阻塞。

上面是单个Paxos Group的场景，推广到多个Paxos Group也很简单，不再赘述。看起来很完美。然而，会不会隐约觉得有些不对劲？交互式的事务，如何预先知道一个事务要查询的所有key？不知道要查询什么，就无从计算 ![[公式]](https://www.zhihu.com/equation?tex=S_%7Bread%7D) 。其实这里隐含了一个条件，Spanner的查询语言允许预先声明要查询的key range，称之为`scope表达式`。而交互式的SQL是不具备这样的能力的！不过不用遗憾，对于单行的点读还是很有效的优化。

## 7. ![[公式]](https://www.zhihu.com/equation?tex=T%5E%7Bpaxos%7D_%7Bsafe%7D)

拿到 ![[公式]](https://www.zhihu.com/equation?tex=S_%7Bread%7D) 之后，就可以把这个读请求发到Leader或者Follower了，很棒，在Spanner里可以读Follower。

但是还要等一下，如果是Follower，万一这个Follower还没有复制到相应的数据怎么办呢？

很简单，paxos follower维护一个 ![[公式]](https://www.zhihu.com/equation?tex=T%5E%7Bpaxos%7D_%7Bsafe%7D) *，Apply过的最大的事务时间。如果$S*{read}$大于 ![[公式]](https://www.zhihu.com/equation?tex=T%5E%7Bpaxos%7D_%7Bsafe%7D) ，就需要阻塞，因为数据还没有复制过来。

然而，如果用 ![[公式]](https://www.zhihu.com/equation?tex=TT.now%28%29.latest) 作为 ![[公式]](https://www.zhihu.com/equation?tex=S_%7Bread%7D) ，并且paxos group一直没有write怎么办？

Leader会一个未来的 ![[公式]](https://www.zhihu.com/equation?tex=MinNextTS) 发给Follower，Follower可以把当前的 ![[公式]](https://www.zhihu.com/equation?tex=T%5E%7Bpaxos%7D_%7Bsafe%7D) *移到* ![[公式]](https://www.zhihu.com/equation?tex=MinNextTS-1) *，然后就可以使得* ![[公式]](https://www.zhihu.com/equation?tex=S%7Bread%7D+%3C+T%5E%7Bpaxos%7D_%7Bsafe%7D) *了。不过Leader怎么保证未来不会使用这个时间呢，只要每次更新* ![[公式]](https://www.zhihu.com/equation?tex=MinNextTS) *之后，同时把自己的* ![[公式]](https://www.zhihu.com/equation?tex=s_%7Bmax%7D) 也更新，未来的paxos write使用大于 ![[公式]](https://www.zhihu.com/equation?tex=s_%7Bmax%7D) 即可。

例如，Leader 的Lease是10秒，每8秒把 ![[公式]](https://www.zhihu.com/equation?tex=MinNextTS) 提前到未来的8秒之后，8秒内的读请求就不会阻塞了。

## 8. ![[公式]](https://www.zhihu.com/equation?tex=T%5E%7BTM%7D_%7Bsafe%7D)

除了复制协议造成的阻塞之外，还有冲突事务造成的阻塞。

最简单的做法，Transaction Manager维护Prepared事务的最小PrepareTS，记为 ![[公式]](https://www.zhihu.com/equation?tex=T%5E%7BTM%7D_%7Bsafe%7D+%3D+min%28s%5E%7Bprepare%7D_%7Bg%7D%29+-+1) ，满足 ![[公式]](https://www.zhihu.com/equation?tex=S_%7Bread%7D+%3C+T%5E%7BTM%7D_%7Bsafe%7D) 即可处理读请求。

另外，还得保证事务提交时，用的CommitTS大于所有Participant的PrepareTS。如此一来，即保证 ![[公式]](https://www.zhihu.com/equation?tex=S_%7Bread%7D+%3C+s_%7Bprepare%7D+%3C%3D+commitTS) ，可以忽略这些未提交的数据。

不过这样显然还是会出现false conflict的情况，事务读的数据和正在Prepared的事务没有关系，不应该被阻塞。

因此， ![[公式]](https://www.zhihu.com/equation?tex=T%5E%7BTM%7D_%7Bsafe%7D) *也可以更加细化: 1. 在Lock Table中维护key range的最小prepareTS，而非所有事务的prepareTS 2. 只读事务在查询区间内，计算最小的prepareTS，只要* ![[公式]](https://www.zhihu.com/equation?tex=S%7Bread%7D+%3C+minPrepareTS) 即可读。

## 9. CommitTS

![img](https://pic4.zhimg.com/80/v2-18be0bcd972ace91ea7632ca08297a6b_1440w.jpg)



CommitTS已经提了很多，但没有明确定义，这里再确认一下。CommitTS指的是事务的数据版本号，用来进行可见性判断，并且要保证外部一致性。具体来说，在所有Participant返回Prepared之后，Coordinator获得CommitTS，同时写Commit Log以及等待 ![[公式]](https://www.zhihu.com/equation?tex=TT.after%28CommitTS%29) ，Paxos write的时间在几毫秒，而TrueTime的误差也在毫秒级，所以两个时间通常是交叠的，不会带来时间上的浪费。

CommitTS 有几点要求：

1. 大于等于 ![[公式]](https://www.zhihu.com/equation?tex=TT.now%28%29.latest) ，这是保证外部一致性的要求
2. 大于等于所有Participant返回的PrepareTS，这是保证 ![[公式]](https://www.zhihu.com/equation?tex=T%5E%7BTM%7D_%7Bsafe%7D)*的安全性*
3. *大于之前用过的paxos write时间，保证paxos write的时间单调递增，用来保证*![[公式]](https://www.zhihu.com/equation?tex=T%5E%7Bpaxos%7D_%7Bsafe%7D) 的安全

## 10. Blind Write

在MVCC中，对于Write-Only的事务，更新同一行数据的并发事务是否一定被认为冲突呢？可以写到不同的版本吗？

Spanner实现了称之为`Blind Write`的优化，在一定情况下使得写锁可以共享，允许对冲突数据的并发写入，最终用CommitTS来调度写的顺序。

关于这个的资料比较少，暂且猜测一下，不对正确性负责：

![img](https://pic4.zhimg.com/80/v2-7f3c5ed4b013e34b94c4aea322330cef_1440w.jpg)

## 总结

Spanner之所以复杂，是因为融合了Paxos、2PC、Transaction Isolation，以及External Consistency，其中每一点都可以大做文章，而Google大方地写到一篇文章中，气势磅礴毋庸置疑，但也给读者带来了巨大的压力。选择一个合适的切入点，合适的维度去解构，才能够理清其中的诸多细节。

受限于本人的知识水平，未尽之处还请指教。



> 原文链接：https://zhuanlan.zhihu.com/p/47870235