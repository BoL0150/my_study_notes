# CMU 15-445实验记录（六）：Project 4 logging recovery

在bustub中，有很多XXX_manager类，用法是：将这些类实例化，整个bustub中这些manager类各有且只有一个实例，以后由这些实例化的manager对象来管理XXX，所有对XXX的操作都要调用manager的方法。如`disk_manager`_，所有对磁盘进行的操作都要调用`disk_manager_`的方法，所有对缓冲池进行的操作都要调用buffer_pool_manager对象中的方法，所有对事务进行的操作都要调用`txn_manager`对象中的方法（实际上这有点违反常理，按理来说对事务进行操作应该调用txn对象中的方法，但是`txn_manager`对象中的方法是用来管理全局的所有事务的，所以把事务的commit、abort、begin等方法放在`txn_manager`中是合理的）

page对象：

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20220302123952647.png" alt="image-20220302123952647" style="zoom:50%;" />

`BPlusTreeLeafPage`和`BPlusTreeInternalPage`对象继承自`b_plus_tree_page`对象，公共属性为

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20220302124354442.png" alt="image-20220302124354442" style="zoom:50%;" />

除了上面的这些公共字段外，`BPlusTreeLeafPage`和`BPlusTreeInternalPage`对象各有一个kv数组。

Page对象是缓冲池的buffer，从磁盘调入内存的页就存放于Page对象的data字段中，不管是B+树的索引页还是堆文件的数据页都存放于Page对象的data字段中：

- 对于B+树的索引页：每一次我们读取或写入一个叶子或内部页，我们需要先通过唯一的`page_id`将该页从缓冲池中fetch，再使用`reinterpret_cast`将该页的**内容**（**即获取的Page对象的`data_`部分！而不是Page对象本身！**）重新解释为`BPlusTreeLeafPage`或`BPlusTreeInternalPage`对象
- 对于堆文件的数据页：Page对象的整个data部分就是数据页的内容，被分为三个部分：header，空闲空间和tuple部分，所以有一个TablePage类来表示内存中的数据页，该类继承自Page类，没有多出的字段，只有多出的方法，用来操作数据页，也就是Page对象的data部分。当遍历一张表的数据时，需要先根据tableHeap中的first_page_id从缓冲池中fetch该表的第一个数据页，此时获取的是Page对象，要操作数据页还需要将Page对象转成TablePage对象，再调用它的方法来操作data部分的不同偏移量。

索引页和数据页的元数据和实际的数据都存在Page对象的data部分中，而二者的区别是：

- 索引页需要将data部分重新解释成类，通过类的字段来访问索引页的元数据；
- 而数据页是根据元数据在data中的偏移量来访问元数据的，

造成此差异的原因是：当插入tuple或删除tuple时，数据页的元数据数量是会变化的。

## 创建log的时机

1. 当begin、commit、abort事务时需要创建log
2. 事务通过tablePage的方法对数据库进行insert，delete，update的修改时需要创建log

在txn_manager中的begin，commit，abort

- begin函数只会创建一个新的事务，不会对DBMS的数据产生任何影响，
- commit函数遍历writeSet，调用tableHeap中的ApplyDelete函数将之前的删除操作（MarkDelete）真正执行，再释放该事务持有的所有锁
- abort函数也要遍历writeSet，对DELETE操作调用tableHeap的RollbackDelete函数，对INSERT操作调用tableHeap的ApplyDelete函数，对UPDATE操作调用tableHeap的UpdateTuple函数，再释放该事务持有的所有锁
  - 用户对一张表对应的堆文件进行操作时调用的是tableHeap中的函数，TableHeap中的函数只是先从缓冲池中获取具体所操作的页，再调用该页的方法进行修改，所以主要的修改代码都在tablePage的函数中。**tablePage中的方法对page修改完后，也需要打上各自类型的log**

最后这三个函数都会打上各自类型的log。

## 创建log的具体流程

打log的时候需要进行一些LSN的更新：

为了实现日志恢复，系统的不同部分都需要记录某些相关的LSN信息：

- 在Page中记录了pageLSN，代表着对这个页进行最近一次修改操作对应的日志的LSN（实际上没有pageLsn是不影响结果的正确性的，它仅仅是用来避免不必要的redo，因为redo时如果page的lsn大于当前的record的lsn就说明这条 record的操作已经落盘了，不需要重做）
- 在事务中记录了prevLSN，代表这个事务上一次的操作对应的日志的LSN（此LSN实际上在日志恢复时不起作用，因为内存中已经不存在这个事务了，它的作用是以便在创建log_record时知道上一次的日志的LSN）
- 在log record对象中记录了当前record的LSN和在该事务中上一条日志的LSN
- 在内存中维护了presidentLSN，代表目前刷到磁盘中的日志的LSN

主要在三个地方更新这些LSN：

- 在txn_manager中，begin，commit，abort都要创建log，此时要创建log record对象，并用事务中记录的prevLSN初始化log_record的prevLSN；再将log_record加入log_buffer；再更新txn的prevLSN；由于这三个操作并没有真正修改page，所以不需要更新pageLSN
- 在tablePage中，对page的修改操作也需要创建log，除了进行上面的操作外，还需要更新pageLSN
- 当刷盘时，需要更新persistentLSN

对于commit和abort操作，它们表示事务已经结束了，所以根据持久性的要求，打完commit和abort的log之后，还需要将log_buffer中的所有log落盘。并且为了实现组提交，此时是非强制落盘，要等到其他条件触发log刷盘，比如log_timeout、log_buffer满或者磁盘刷脏，跟着它们一起把log刷盘（此时是在这里阻塞，并没有成功commit，所以不会破坏原子性）

```cpp
  if(enable_logging){
    LogRecord commit_log(txn->GetTransactionId(),txn->GetPrevLSN(),LogRecordType::COMMIT);
    txn->SetPrevLSN(log_manager_->AppendLogRecord(&commit_log));
    // when committing ,we need to group commit
    log_manager_->flush(false);
  }
```

等待落盘就wait在appendLog的条件变量上就行了，当刷盘完成后会把appendLog的条件变量wakeup

```cpp
  void flush(bool force){
    std::unique_lock<std::mutex> ul(latch_);
    // when bpm evict dirty page
    if(force){
      flush_log_to_evict_dirty_page_ = true;
      flush_cv_.notify_all();
      append_cv_.wait(ul);
    // when txn commit or abort,we need to block the current txn
    // until the log manager flush the log to disk
    }else{
      append_cv_.wait(ul);
    }
```

## 添加log和log刷盘

在log_manager中，添加log和log刷盘各有一个条件变量，用于同步这两个操作。

- log_buffer如果满了，添加log的事务就wait在append_cv上，将flush_cv上的事务唤醒；
- log刷盘结束后，就把append_cv上的事务唤醒，刷盘的事务就wait在flush_cv上。

txn_manager中的begin、commit、abort方法和TablePage中修改Page的方法创建完log之后，需要调用log_manager的appendLog方法将log对象序列化到内存中的log_buffer中：

- 先检查log_buffer的容量够不够，如果log_buffer满了，添加log的事务就wait在append_cv上，将flush_cv上的事务唤醒；如果log_buffer的容量足够，就根据log对象的类型，把它的相应字段序列化到log_buffer中。

由log_manager的构造函数创建一个后台线程，运行log的刷盘操作：

```cpp
    enable_logging = true;
    // 创建并运行flush_thread_线程，该线程运行LogManager类的RunFlushThread方法，并且该方法运行在当前所在的对象上（this），也就是LogManger对象上
    flush_thread_ = new std::thread(&LogManager::RunFlushThread, this);
```

log的刷盘函数：

- 首先检查刷盘的条件是否满足（**log_buffer满或者缓冲池准备刷脏**），如果不满足就wait在flush_cv上，**最多wait log_timeout的时长**。

  ```cpp
  std::unique_lock ul(latch_);
  flush_cv_.wait_for(ul,log_timeout,is_log_buffer_full_||flush_log_to_evict_dirty_page_);
  ```

  如果刷盘的条件满足了，该线程会被唤醒，就通过disk_manager将log_buffer的内容写入磁盘中的log文件。然后再更新persistentLSN，将刷盘条件更新，将log_buffer_offset置为0，并且把append_cv上的等待添加log的事务唤醒。

### 条件变量的wait_for使用方法

~~在一个while循环中，首先检查刷盘的条件是否满足（**log_buffer满或者缓冲池准备刷脏**），如果不满足就wait在flush_cv上，**并且wait一段时间（log_timeout）**~~。

```cpp
std::unique_lock ul(latch_);
while(!is_log_buffer_full_ && !flush_log_to_evict_dirty_page_){
	flush_cv_.wait_for(ul,log_timeout);
}
```

注意！以上方法不行！因为即使log_timeout时间到了，事务被唤醒了，但是while循环中的条件依然不变，还是会继续进入sleep！

所以正确写法是使用wait_for方法：

```cpp
bool wait_for( std::unique_lock<std::mutex>& lock,
               const std::chrono::duration<Rep, Period>& rel_time,
               Predicate stop_waiting);
```

- rel_time是wait的最长时间，时间到了即使predicate不满足也要被唤醒；
- 而当线程被`notify`强制唤醒时，还需要满足`predicate`条件，如果predicate为false，即使被唤醒了依然要重新sleep（predicate必须是lamada表达式）

所以正确写法为：

```cpp
flush_cv_.wait_for(ul,log_timeout,[&]{ return is_log_buffer_full_ || 					flush_log_to_evict_dirty_page_;});
```

注意，此时仍然不完全等同于之前的while循环的形式，因为按照这个写法，第一次执行这里时会直接sleep，而while循环还是需要判断条件。所以还要加上一条if语句，才能等同于while循环的写法

所以下面的两种写法才是等价的：

```cpp
    while(log_buffer_offset_ + log_record->GetSize() > LOG_BUFFER_SIZE) {
        is_log_buffer_full_ = true;
        flush_cv_.notify_all();
        append_cv_.wait(ul);
    }
    // 上下写法等价
    if(log_buffer_offset_ + log_record->GetSize() > LOG_BUFFER_SIZE) {
        is_log_buffer_full_ = true;
        flush_cv_.notify_all();
        append_cv_.wait(ul, [&]{ return log_buffer_offset_ + log_record->GetSize() <= LOG_BUFFER_SIZE;});
    }
```



## 日志恢复的流程

由于我们不支持fuzzy checkpoint，所以日志恢复时不需要分析阶段。

宕机重启后，首先调用redo方法进行重做：

- 将磁盘中的log从头到尾全部读入内存的log_buffer中，再对log_buffer中的每一条log进行反序列化成log_record对象。遍历每一个log_record对象，同时维护active txn表（用于后面的undo操作），维护了crash时每个活跃的txn和它对应的lastLSN。对不同类型的log_record对象：

  - begin，commit，abort操作本身不会对DBMS的数据产生修改，它们对DBMS的修改会产生单独的log（比如COMMIT时调用的ApplyDelete和abort时调用的rollbackDelete、ApplyDelete、updateTuple会各自产生相应的log)，然后才会写入commit和abort的log，并且由于持久性的要求，commit和abort会一直阻塞直到这些log落盘了再返回。所以当我们遇到commit和abort的log时，它们对DBMS产生的修改已经redo完了。

    所以对于begin，commit和abort log，我们不需要对数据库中的数据做任何操作，对于commit和abort，只需要把对应的事务从att表中移除，因为它们表示事务已经结束了。

  - **对于其他的log，也不是每一条日志都需要redo的，只有当日志的LSN大于它所修改的page的LSN，才需要redo**，因为这说明该日志的修改没有落盘。redo直接把该record对应的tablePage的函数重新调用一次即可。

redo结束后再调用undo方法：

- 对att中的每一个事务，从它在表中对应的lastLSN开始，读取相应的log_record，根据log_record的类型做相反的操作，Insert对应ApplyDelete，MarkDelete对应RollBackDelete，Update对应Update。再获取该record的上一条record，以此往复，直到record的prevLSN为INVALID

## checkpoint的实现

txn_manager中有一个读写锁，txn_manager begin一个事务的时候要获取读锁，commit和abort时要释放读锁，打checkpoint时要获取写锁，再将内存中的日志和缓冲池中的脏页全部刷盘；打完checkpoint时要把写锁释放，这样就能实现事务的执行和打checkpoint过程的完全互斥。

checkpoint的好处在于，checkpoint之前的日志所做的数据更新都已经落盘了，并且不需要回滚，因此数据库恢复时完全不需要看checkpoint之前的日志记录，所以磁盘中checkpoint之前的日志都可以直接删掉（内存中的日志每次日志刷盘的时候就可以清空了）

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20230429172726731.png" alt="image-20230429172726731" style="zoom:50%;" />

## 总结

当加锁解锁失败或死锁时，事务要被Abort，Abort除了要释放此事务的获得的所有锁之外，还要撤回此事务之前对数据库（堆文件和索引）进行的所有写操作，为了实现此操作，我们需要在transaction对象内部维护两个write set，在算子执行时，我需要将当前事务对table和index所进行的写操作分别记录这两个集合中，Abort时就可以根据txn内部的这两个set进行回滚。实际上，系统中有两个日志，整个系统全局有一个日志，每个事务内部还有一个日志（就是这两个集合），事务对系统做的修改既记录在事务内部的日志，也记录在全局的日志中，事务内部的日志主要用来对事务自己的操作回滚（ABORT），整个系统的日志主要用来对整个系统redo和undo。

