# project4

首先设计了一个事务管理器，它能够跟踪和管理所有的活跃事务，并提供了提交和回滚事务的接口。然后，我实现了锁管理器来控制并发事务对共享资源的访问，它能够对数据库中的数据进行加锁和解锁操作，并且能够支持两阶段锁协议；当多个事务同时请求相同的时，系统需要正确地处理锁的冲突，以避免死锁和饥饿问题。

## TASK #1 - LOCK MANAGER

### 事务隔离级别

不同的隔离级别会对并发性能和数据一致性产生不同的影响

读未提交：写锁遵循二阶段锁协议，读的时候不加读锁，无法解决任何问题

- 读未提交的排他锁要遵守二阶段锁协议的原因是避免死锁，要拿排他锁的原因是避免出现不一致

读已提交：写锁遵循严格的二阶段锁协议，读的时候加读锁，但是不遵循二阶段锁协议，读完数据后马上可以释放，随后又可以再次获取。可以解决脏读

- 写锁一直到commit之前才会释放，而其他事务读的时候要加读锁，从而其他事务无法读取到已修改未提交的数据

- 如果写锁是不严格的二阶段锁协议，写完之后就释放，则其他的事务就可以读该数据，发生脏读
- 如果读的时候不用加读锁，即便写锁采取了严格的二阶段锁协议，一个事务在修改数据时，其他事务随时可以读此数据，会发生脏读

可重复读：写锁遵循严格的二阶段锁协议，读锁遵循二阶段锁协议，释放后不能再获取，可以解决脏读和不可重复读

- 由于读锁也采用二阶段锁协议，当前事务释放了读锁后就进入了shrinking阶段，不能再获取读锁，也就不能再重复读此数据，从而解决了不可重复读
- 二阶段锁是对已有的数据加锁，无论如何都无法解决幻读（插入新数据）的问题

可序列化：读写锁都遵循严格的二阶段锁协议，并且加索引锁，可以再解决幻读

所以**前三级隔离级别的区别在于共享锁的强度，从不加读锁，到加普通的读锁再到按照s2pl协议加读锁**，而写锁统一都是严格的二阶段锁

### 锁管理器的结构

lock_manager中有一个lock_table，记录了表中的tuple和请求这个tuple的锁的所有事务之间的映射，

请求某个锁的事务用`LockRequest`对象表示，

- 该对象中记录了对应事务的id，
- 请求什么样的锁（独占锁还是共享锁），
- 以及该锁是否被授予

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20230308195214187.png" alt="image-20230308195214187" style="zoom:50%;" />

全局有一个map，维护了tuple的rid到该tuple的锁请求队列（`LockRequestQueue`）之间的映射

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20230312132731000.png" alt="image-20230312132731000" style="zoom:50%;" />

请求某个的tuple的锁的**所有**事务用`LockRequestQueue`对象表示

- 内部用一个List表示锁请求队列，记录了所有的`LockRequest`对象，**包括已经获取了该锁的事务和想要获取该锁被阻塞的事务**。此队列用于在后面检测死锁，遍历map，**同一个锁请求队列中未授予锁的事务被已授予锁的事务阻塞**，比如：事务A在tuple1的锁请求队列中阻塞了事务B，而事务B在tuple2的锁请求队列中阻塞了事务A，这样就构成了死锁。我们就可以依据这个Map和每个tuple的锁请求队列构建出事务的等待图，再使用dfs检测死锁。
- 还有一个条件变量，想要获取该锁被阻塞的事务wait在此条件变量上，
- 还要有一个条件锁，用来保护~~wait的条件~~LockRequestQueue内部的成员变量

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20230308220319874.png" alt="image-20230308220319874" style="zoom:50%;" />

全局还有一个锁来保护整个Map并发访问。这是常见的并发模型，在一个容器内部有许多对象，有一个大锁保护容器，容器内的对象内部还有一个锁来保护对象。对容器内的对象进行操作的流程：先获取容器的大锁，此时才能访问容器。遍历容器，找到此对象，若没有此对象也可以创建新的对象，然后对容器插入。找到此对象后，**先获取对象内部的小锁，再释放容器的大锁**，此时才能对此对象进行读或修改的操作。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20230312133416338.png" alt="image-20230312133416338" style="zoom: 50%;" />

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20230312133458717.png" alt="image-20230312133458717" style="zoom: 50%;" />

如上图所示，对lock_table进行访问就要拿大锁，找到lock_table中的`LockRequestQueue`中后，先拿`LockRequestQueue`对象内部的条件锁，再释放lock_table的大锁，即可操作锁请求队列。

### 独占锁和共享锁的实现

如果一个事务想获取tuple的共享锁，只有当该tuple的`LockRequestQueue`中在该事务~~之前的所有事务都已经获取了共享锁~~  没有事务获取独占锁（共享锁可以不按照请求的顺序来获取，当某个事务释放独占锁的时候，会把所有等待此tuple共享锁的事务唤醒，这些事务不需要按照顺序来获取共享锁），该tuple的共享锁才能被此事务获取。

如果一个事务想获取tuple的独占锁，只有当该tuple的`LockRequestQueue`中没有事务获取独占锁，也没有事务获取共享锁才行

所以在`LockRequestQueue`中维护一个bool变量记录当前是否有事务获取此tuple的独占锁，维护一个计数器记录当前有多少事务获取了此tuple的共享锁

所以以上其实就相当于实现一个读写锁，和下面的实现类似

```cpp
  void RLock() {
    std::unique_lock<mutex_t> latch(mutex_);
    while(writer_entered_ || reader_count_ == MAX_READERS){
      writer_.wait(latch);
    }
    reader_count_++;
  }  

void WLock() {
    std::unique_lock<mutex_t> latch(mutex_);
    // 注意，以下写法是错误的，当前面有线程读的时候，写者会被阻塞，而后面的读者却不会被阻塞，导致写者饿死
    // while(writer_entered_ || reader_count_ > 0){
    //   writer_.wait(latch);
    // }
    // 为了避免写者饿死，当前面只有读者时，写者开始排队，此时要标记写者正在排队（writer_entered）
    // 后面的读者看到此标记就要进入睡眠，不能直接插队
    while(writer_entered_ ){
      writer_.wait(latch);
    }
    writer_entered_ = true;
    while(reader_count_ > 0){
      writer_.wait(latch);
    }
  }

  void WUnlock() {
    std::lock_guard<mutex_t> guard(mutex_);
    writer_entered_ = false;
    writer_.notify_all();
  }

  void RUnlock() {
    std::lock_guard<mutex_t> guard(mutex_);
    reader_count_--;
    writer_.notify_all();
  }
    
 private:
  mutex_t mutex_;
  cond_t writer_;
  uint32_t reader_count_{0};
  bool writer_entered_{false};
```

所以事务获取锁不是按照事务发出请求的先后顺序来的，而是随机的（我认为这没有影响，隔离级别的要求只有脏读，不可重复读，幻读，只要保证事务的原子性和隔离性即可，哪个事务先哪个事务后是不影响一致性的）

由于死锁检测的要求，**事务等待被授予rid的锁时，也要放入request_queue中**，否则不知道哪些事务在等待某个rid的锁，也就不知道哪些事务被阻塞，无法构建wait for graph。

事务加共享锁：

- 如果隔离级别是读未提交，不需要加共享锁
- 如果该事务已经进入了shrinking状态，也不能加锁

事务加独占锁：

- 如果该事务已经进入了shrinking状态，也不能加锁

unlock：

解锁时的不同情况：

- Aborted的事务在调用Abort时，会释放之前持有的所有的锁，所以对于Aborted的事务在unlock时不能将状态置为shrinking。对非Aborted的事务，不同隔离级别对unlock时是否置为shrinking不同：
  - 可重复读级别：读写锁都遵循严格的二阶段锁，所以只要unlock就变成shrinking状态
  - 读已提交级别：写锁遵循严格的二阶段锁，读锁不遵循二阶段锁，所以unlock读锁时不需要变成shrinking状态

锁升级：

- 如果对一个数据先读再写，就要对其先加读锁再加写锁，但是由于2pl协议，加了读锁之后就不能再释放锁了，所以要想实现这种操作只能进行锁升级，将读锁直接升级为写锁。实现方式是：将对应的rid的锁请求队列中的共享锁请求删除，再对该rid加独占锁，将独占锁加入到锁请求队列中。

以上的几个函数中解锁的状态变化和加锁的状态检测只体现出了二阶段锁协议，而严格的二阶段锁协议则体现在transaction_manager的commit函数中，在commit之前才释放所有的锁



## **TASK #2 - DEADLOCK DETECTION**

lock manager创建了一个后台线程，定时运行死锁检测算法，检测出等待图中的回路（死锁）后，选择回路中最年轻的事务Abort，将此事务的所有操作回滚，释放此事务的所有的lock（在事务中记录了所获取的所有锁）：

- 死锁检测：每个rid的`LockRequestQueue`中，未授予锁的事务被授予锁的事务阻塞，在等待图中添加一条由未授予锁的事务指向授予锁的事务的边。遍历lock_table中的每个LockRequestQueue，构建出等待图。等待图采用 `std::map<txn_id_t, std::set<txn_id_t>> waits_for_;`构成邻接表。构建完等待图后，

  - 对此图使用dfs，判断图中是否有环，执行dfs时，使用set记录经过的txn_id，当发现重复经过的txn_id就说明此图中存在环，就返回set中最大的txn_id（也就是环中最年轻的事务）。再将此事务的状态置为Aborted，**将此事务从刚才构建的等待图中删除**（包括此顶点涉及到的所有的边），再将此事务唤醒

    - 实际操作中需要对此事务所在的条件变量调用notify_all，也就是会导致所有和此事务请求同一个tuple的事务都被唤醒，所有事务唤醒后会先检查自身的状态，其余事务发现状态不是Aborted会继续睡眠，

    此事务发现自身的状态变成了Aborted，就会将此事务被阻塞的锁请求从对应的锁请求队列中删除，再抛出异常。在executor_engine调用算子（执行算子时调用锁管理器中的方法获取共享锁或独占锁）时会catch抛出的异常，然后**使用TransactionManager中的Abort方法真正abort此事务**

    - Abort方法：回滚此事务之前的操作，包括将此事务所有已授予的锁请求释放（此操作会将它们从对应的锁请求队列中删除，并且唤醒这些请求队列中所有被阻塞的事务，死锁就此被解开）

  - 重复调用dfs检测等待图中是否存在回路，并且重复上述步骤，直到等待图中不存在回路，结束死锁检测，将等待图清空，下次此后台线程需要重新构建等待图

## TASK #3 - CONCURRENT QUERY EXECUTION

当加锁解锁失败或死锁时，事务要被Abort，Abort除了要释放此事务的获得的所有锁之外，还要撤回此事务之前对数据库（堆文件和索引）进行的所有写操作，为了实现此操作，我们需要在transaction对象内部维护两个write set，在算子执行时，我需要将当前事务对table和index所进行的写操作分别记录这两个集合中，Abort时就可以根据txn内部的这两个set进行回滚。实际上，系统中有两个日志，整个系统全局有一个日志，每个事务内部还有一个日志（就是这两个集合），事务对系统做的修改既记录在事务内部的，也记录在全局的日志中，用事务内部的日志来对事务自己的操作回滚，用整个系统的日志来对整个系统回滚。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20230313105426311.png" alt="image-20230313105426311" style="zoom:50%;" />

我们需要了解事务是如何回滚的：

- 比如update操作，执行时，将原表中的tuple更新为新的tuple，rid不变，再在该表的所有index中删除原tuple的kv对，插入新tuple的kv对。所以我们需要将所修改tuple的id，该tuple的原内容记录在TableWriteRecord中，将原tuple的kv对和新tuple的kv对记录在IndexWriteRecord中，回滚时就能删除新的信息，将原来的信息重新写回。

在第一个事务执行开始之前，创建一个ExecutionEngine对象，以后此数据库所有查询的执行入口都是这个对象的execute方法。而每个事务由多个查询（sql语句）组成，每执行一个查询都要调用一次execute方法，如何让同一个事务的不同查询共享同一个txn的数据？

- 调用execute方法时传入不同的查询计划，相同的txn，就可以实现执行同一个事务的不同查询（SQL语句）。**执行每个查询语句时可以把获取的锁和进行过的写操作记录在同一个txn对象中**。
- 每次调用execute方法执行一条查询时还需要传入`ExecutorContext`对象，这个对象中包含了本次查询所属的txn对象，全局的缓冲池，catalog，以及锁管理器lockManager。执行的算子就可以知道本次执行的上下文，获取tuple时通过全局的锁管理器来获取锁（**所有事务共享同一个锁管理器对象**）。

### 具体实现

由于不同的隔离级别对获取锁的要求不一样，我将获取共享锁LockShared和独占锁LockExclusive的操作封装在两个新函数中，

```cpp
 bool tryExclusiveLock(const RID &rid)
 bool trySharedLock(const RID &rid)
```

不管事务的隔离级别是什么，底层的算子在获取tuple之前都要调用此函数，在这两个函数中再根据事务的隔离级别和具体情况来判断能否获取该锁。

- 由于任何隔离级别的都要加独占锁，所以tryExclusiveLock不需要检查事务的隔离级别，对于一个rid，先检查是否已获得此tuple的独占锁，如果已获得直接返回。否则就检查是否已获得tuple的共享锁，如果是则直接升级为独占锁即可，如果没有则直接加独占锁
  - 锁升级的意义：对于同一个tuple，在火山模型的不同层级的算子中对其进行的操作是不一样的，比如update查询，子算子是seqscan，它只需要负责将相应的tuple读出来然后返回，它不知道父算子会对此tuple进行什么操作，所以它只需要对读出来的tuple加读锁。而此tuple传递给父算子后，父算子需要对其进行修改，所以这个时候就需要获取此tuple的写锁，直接将之前子算子加的读锁升级为写锁即可。
- trySharedLock：如果事务的隔离级别是不可重复读，则不需要加读锁，直接返回。否则就加读锁

对算子的修改：

对于indexScan和seqScan，在获取tuple之前调用`trySharedLock`。如果隔离级别是读已提交，则还要在next函数结束之前将读锁释放

对于更新和删除操作，需要在调用了子算子的next方法后，调用tryExclusiveLock获取该tuple的独占锁，再对其进行修改，~~修改时还要将原内容和修改的内容记录到txn内部的修改记录集合中~~，**tableHeap的方法中会将对表的修改记录在txn内部的修改记录集合中，我们只需要将对索引的修改记录在txn中**。

对于插入操作，由于插入之前此tuple根本没有被创建，所以无法在插入之前对此tuple上锁，而如果在插入之后再上锁，其他事务有可能在插入之后、上锁之前乘虚而入访问此tuple。所以我选择修改tableHeap的InsertTuple方法，在往page中插入了新tuple后，还没有释放此page的锁之前，就对此tuple加上独占锁。

对于join和聚集，二者都是直接调用seqScan，对从seqScan中获得的tuple不做修改，所以不需要在其中加锁。

