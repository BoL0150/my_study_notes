# CMU 15-445实验记录（二）：Project 1

在project 1中，已经为我们提供了磁盘管理器以及page layouts ，我们要构建自己的buffer pool管理器以及替换 策略，根据需要调⽤磁盘管理器将这些数据写出到磁盘。

## LRU REPLACEMENT POLICY

这一部分负责跟踪缓冲池中的page使用情况，我们需要在 `src/include/buffer/lru_replacer.h` 中实现一个名为 `LRUReplacer` 的新子类，并在 `src/buffer/lru_replacer.cpp` 中实现其对应的实现文件。 `LRUReplacer` 继承了包含函数规范的`Replacer` 抽象类（`src/include/buffer/replacer.h`）

LRUReplacer 的大小与缓冲池相同，因为它包含 BufferPoolManager 中所有frame的占位符。然而，并不是所有的frame都被认为是在 LRUReplacer 中的。 LRUReplacer 被初始化为没有frame。只有新的没有被固定的（unpinned）那些frame才会被认为是在 LRUReplacer 中的。

Locks和Latches的区别：

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211020200401635.png" alt="image-20211020200401635" style="zoom:50%;" />

```cpp
/**
 * LRUReplacer implements the lru replacement policy, which approximates the Least Recently Used policy.
 */
class LRUReplacer : public Replacer {
 public:
  /**
   * Create a new LRUReplacer.
   * @param num_pages the maximum number of pages the LRUReplacer will be required to store
   */
  explicit LRUReplacer(size_t num_pages);

  /**
   * Destroys the LRUReplacer.
   */
  ~LRUReplacer() override;

  bool Victim(frame_id_t *frame_id) override;

  void Pin(frame_id_t frame_id) override;

  void Unpin(frame_id_t frame_id) override;

  size_t Size() override;

 private:
  std::mutex latch;
  int capacity;
  std::list<frame_id_t> lru_list;
  std::unordered_map<frame_id_t, std::list<frame_id_t>::iterator> lru_map;
};
```

`bool Victim(frame_id_t *frame_id)`：将被Replacer追踪的最近最少使用的page从LRU数据结构中移除，将该page所在的frame_id放入参数中，并返回true；如果Replacer追踪的page数为空，那么就返回false，参数frame_id为nullptr

- 当BufferPool满了（free_list为空）时，需要调用此函数

`Pin(frame_id_t frame_id)`：给定frame_id，将被pin的frame从LRU数据结构中移除。

- 如果BufferPool中的某一页被某个进程引用了，那么该page就应该被固定（pinned）在该frame上，不能从缓冲区中替换出去。此时BufferPoolManager需要调用LRU的Pin函数。

`Unpin(frame_id_t frame_id)`：给定frame_id，将该frame添加到LRU数据结构。

- 当BufferPool中某一页的pin_count变为0时，即引用该页的线程变为0，此时BufferPoolManager需要调用LRU的Unpin函数。

```cpp
#include "../include/buffer/lru_replacer.h"

namespace bustub {

LRUReplacer::LRUReplacer(size_t num_pages) : capacity(num_pages) {}

LRUReplacer::~LRUReplacer() = default;
// 将被Replacer追踪的最近最少使用的page移除，将该page的frame_id放入参数中，并返回true
// 如果Replacer追踪的page数为空，那么就返回false，参数frame_id为nullptr
bool LRUReplacer::Victim(frame_id_t *frame_id) {
  latch.lock();
  if (lru_map.empty()) {
    latch.unlock();
    return false;
  }
  frame_id_t lru_frame = lru_list.back();
  // 将lru frame从map中移除
  lru_map.erase(lru_frame);
  // 将lru frame从list中移除
  lru_list.pop_back();
  *frame_id = lru_frame;
  latch.unlock();
  return true;
}
// 如果BufferPool中的某一页被某个进程引用了，那么该page就应该被固定（pinned）
// 在该frame上，不能从缓冲区中替换出去，此时调用Pin函数，将被pin的frame从我们的数据结构中移除
void LRUReplacer::Pin(frame_id_t frame_id) {
  latch.lock();
  if (lru_map.count(frame_id) != 0) {
    // 指定迭代器从list中删除
    lru_list.erase(lru_map[frame_id]);
    // 指定key从map中删除（也可以指定迭代器）
    lru_map.erase(frame_id);
  }
  latch.unlock();
}

// 当某一页的pin_count变为0时，即引用该页的线程变为0，可以从缓冲区替换出去,需要将该页
// 取消固定（Unpin）。调用Unpin函数，将该frame加入我们的数据结构
void LRUReplacer::Unpin(frame_id_t frame_id) {
  latch.lock();
  // 如果不在追踪的链表中，添加到头部
  if (lru_map.count(frame_id) == 0) {
    lru_list.push_front(frame_id);
    lru_map[frame_id] = lru_list.begin();
  }
  latch.unlock();
}

size_t LRUReplacer::Size() { return lru_list.size(); }

}  // namespace bustub
```



## BUFFER POOL MANAGER

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20220128123316562.png" alt="image-20220128123316562" style="zoom:50%;" />

接下来，我们需要在我们的系统中实现缓冲池管理器（BufferPoolManager）。 BufferPoolManager 负责从 DiskManager 中获取数据库page并将它们存储在内存中。 BufferPoolManager 也可以在明确指示这样做时或当它需要驱逐一个页面为新页面腾出空间时将脏页写出到磁盘

已经为我们提供了磁盘管理器（`DiskManager`），负责将数据从磁盘中读取和写入。

```cpp
  // 驱逐缓冲池中的某一个page
  bool evict(frame_id_t *frame_id);
  // 在缓冲池中找到空闲的frame，如果没有空闲的的frame则
  // 使用替换策略驱逐一个page
  bool find_free_frame(frame_id_t *frame_id);
  bool find_frame(page_id_t page_id, frame_id_t *frame_id);
  /**给定页号，从缓冲池中获取该页。
   * Fetch the requested page from the buffer pool.
   * @param page_id id of page to be fetched
   * @return the requested page
   */
  Page *FetchPageImpl(page_id_t page_id);

  /**
   * Unpin the target page from the buffer pool.
   * @param page_id id of page to be unpinned
   * @param is_dirty true if the page should be marked as dirty, false otherwise
   * @return false if the page pin count is <= 0 before this call, true otherwise
   */
  bool UnpinPageImpl(page_id_t page_id, bool is_dirty);

  /**
   * Flushes the target page to disk.
   * @param page_id id of page to be flushed, cannot be INVALID_PAGE_ID
   * @return false if the page could not be found in the page table, true otherwise
   */
  bool FlushPageImpl(page_id_t page_id);

  /**
   * Creates a new page in the buffer pool.
   * @param[out] page_id id of created page
   * @return nullptr if no new pages could be created, otherwise pointer to new page
   */
  Page *NewPageImpl(page_id_t *page_id);

  /**
   * Deletes a page from the buffer pool.
   * @param page_id id of page to be deleted
   * @return false if the page exists but could not be deleted, true if the page didn't exist or deletion succeeded
   */
  bool DeletePageImpl(page_id_t page_id);

  /**
   * Flushes all the pages in the buffer pool to disk.
   */
  void FlushAllPagesImpl();

  /** Number of pages in the buffer pool. */
  size_t pool_size_;
  /** Array of buffer pool pages.
   *  存放缓冲区中页的数组，即存放了page的帧*/
  Page *pages_;
  /** Pointer to the disk manager. */
  DiskManager *disk_manager_ __attribute__((__unused__));
  /** Pointer to the log manager. */
  LogManager *log_manager_ __attribute__((__unused__));
  /** 页表，用于将页号转换为该页存放在缓冲区中的帧号*/
  std::unordered_map<page_id_t, frame_id_t> page_table_;
  /** Replacer to find unpinned pages for replacement. */
  Replacer *replacer_;
  /** 双向链表，存放内存中空的帧的帧号*/
  std::list<frame_id_t> free_list_;
  /** This latch protects shared data structures. We recommend updating this comment to describe what it protects. */
  std::mutex latch_;
```

frame中的page对象需要改变metadata的几种情况：

- 清空page对象时需要重置metadata（dirty_flag变为false，引用计数pin_count变为0，page_id变为`INVALID_ID`）
- 向page对象中写入新的物理page时需要更新metadata（dirty_flag变为false，page_id变为物理page实际id）
- page对象即将被某个进程引用时（pin_count加一）
- page对象被某个进程取消引用时（pin_count减一，dirty_flag可能也要改变）

函数：

- `Page *FetchPageImpl(page_id_t page_id)`：给定页号，返回该页。当某线程访问某个page时调用。

  - 如果该页在缓冲池中，直接找到对应的page。

  - 如果该页不在缓冲池中，则先寻找空闲frame

    - 如果空闲frame链表不为空，则选一个空闲frame即可
    - 如果空闲frame链表为空，则
      - 需要使用替换策略，从LRU数据结构中找到最近最少使用的frame，将其从LRU数据结构中移除（调用LRUReplacer中的`Victim`方法）
      - 将page的映射从缓冲池中的页表中移除
      - 如果该frame中的page对象的内容被修改过（dirty），将其写回磁盘
      - 重置该page对象的metadata（dirty_flag变为false，引用计数pin_count变为0，page_id变为`INVALID_ID`）
      - 将该frame加入free_list

    找到空闲frame后，将对应的page从磁盘中读取到缓冲池中的free_frame，更新page的metadata（dirty_flag变为false，page_id变为物理page实际id）

  - 该page即将被进程引用，pin_count加一，调用LRU的Pin函数，将该page从LRU数据结构的跟踪对象中移除

- `bool UnpinPageImpl(page_id_t page_id, bool is_dirty)`：给定页号，某进程结束对该页的操作，取消对该页的引用，并且表明该页是否被修改（is_dirty）

  - 如果该页被修改，需要更新该页的metadata（将dirty_flag变为true）

  - 如果该页的pin_count大于0，则直接--
  - 如果该页的pin_count等于0，需要调用Unpin，加到lru的数据结构中

- `Page *NewPageImpl(page_id_t *page_id)`：向磁盘请求一个新的页，并将此页拉到缓冲池中 ，将新分配的page_id写入返回参数中。返回新分配的page，如果失败则返回nullptr。

  - 在缓冲池中查找到一个空闲frame，将对应的page从磁盘中读取到缓冲池中的free_frame，更新page的metadata（dirty_flag变为false，page_id变为物理page实际id）
  - 该page即将被进程引用，pin_count加一，调用LRU的Pin函数，将该page从LRU数据结构的跟踪对象中移除

```cpp
#include "buffer/buffer_pool_manager.h"
#include <list>
#include <unordered_map>

namespace bustub {

BufferPoolManager::BufferPoolManager(size_t pool_size, DiskManager *disk_manager, LogManager *log_manager)
    : pool_size_(pool_size), disk_manager_(disk_manager), log_manager_(log_manager) {
  // 给缓冲池中存放page的数组和追踪page使用情况
  // 的LRU数据结构分配空间，两者的大小相同
  pages_ = new Page[pool_size_];
  replacer_ = new LRUReplacer(pool_size);

  // 初始化记录空闲帧的链表
  for (size_t i = 0; i < pool_size_; ++i) {
    free_list_.emplace_back(static_cast<int>(i));
  }
}

BufferPoolManager::~BufferPoolManager() {
  delete[] pages_;
  delete replacer_;
}
  // 驱逐缓冲池中的某一个page
  // 选出LRU的page，将其从LRU数据结构中移除（取消追踪）
  // 拿着该page的frame_id，再将其从缓冲池中的页表中移除
  // 如果该page的内容被修改过（dirty），将其写回磁盘
  // 将该page的引用计数（pin_count）变为0，is_dirty变为true
  // 再将page所在处的frame_id加入free_list。
  // 注意！Page对象只是缓冲池中存储磁盘数据的容器，不指代某一唯一的物理page。在整个系统的生命周期中，
  // BufferPoolManager通过向相同的page对象中的成员变量data字符数组写入来存储磁盘中的数据。同一个page对象也可以包含不同的物理page
bool BufferPoolManager::evict(frame_id_t *frame_id){
    // 选出LRU的page，并将其从LRU数据结构中移除
    if(replacer_->Victim(frame_id)){
        // 获取frame对应的page_id，将它从页表中移除
        Page* page_to_evict = &pages_[*frame_id];
        page_table_.erase(page_to_evict->GetPageId());
        // 如果该页的内容被修改过，需要写回磁盘
        if(page_to_evict->IsDirty()){
            FlushPageImpl(page_to_evict->GetPageId());
        }
        // 重置缓冲池中page对象的属性
        page_to_evict->is_dirty_ = false;
        page_to_evict->pin_count_ = 0;
        free_list_.push_back(static_cast<int>(*frame_id));
        return true;
    }
    return false;
}
// 在缓冲池的空闲链表中查找空闲的frame，如果没有空闲的的frame则
// 在LRU数据结构中使用替换策略驱逐一个page，如果LRU也为空（所有的page都被pin），则返回false
bool BufferPoolManager::find_free_frame(frame_id_t *frame_id){
    // 如果缓冲池的空闲链表非空，直接返回一个空闲frame
    if(!free_list_.empty()){
        *frame_id = free_list_.front();
        free_list_.pop_front();
        return true;
    }
    // 如果缓冲池的空闲链表为空，使用替换策略驱逐一个page
    return evict(frame_id);
}
// 给定页号，获取该页在缓冲池中对应的frame_id
// 如果该页在缓冲池中，直接返回对应的frame_id
// 如果该页不在缓冲池中，则找到一个空闲frame，再从磁盘中读取对应的page数据到frame中
// 将page对象的属性置为新的物理page的属性
// 然后在缓冲池的页表中建立映射，添加到LRU追踪对象中
bool BufferPoolManager::find_frame(page_id_t page_id,frame_id_t *frame_id){
    if(page_table_.count(page_id) != 0){
        *frame_id = page_table_[page_id];
        return true;
    }
    if(find_free_frame(frame_id)){
        Page *page_to_write = &pages_[*frame_id];
        disk_manager_->ReadPage(page_id,page_to_write->GetData());
        page_to_write->page_id_ = page_id;
        page_to_write->is_dirty_ = false;
        page_table_[page_id] = *frame_id;
        replacer_->Unpin(*frame_id);
        return true;
    }
    return false;
}
  // 给定页号，从缓冲池中获取该页。
Page *BufferPoolManager::FetchPageImpl(page_id_t page_id) {
  // 1.     Search the page table for the requested page (P).
  // 1.1    If P exists, pin it and return it immediately.
  // 1.2    If P does not exist, find a replacement page (R) from either the free list or the replacer.
  //        Note that pages are always found from the free list first.
  // 2.     If R is dirty, write it back to the disk.
  // 3.     Delete R from the page table and insert P.
  // 4.     Update P's metadata, read in the page content from disk, and then return a pointer to P.
    latch_.lock();
    // 获取页号对应的帧号，再通过帧号获取page
    // 将该page的pin_count加一，同时将该page从LRU跟踪链表中移除，is_dirty=false
    frame_id_t frame_id = 0;
    if(find_frame(page_id,&frame_id)){
        Page* page = &pages_[frame_id];
        page->pin_count_++;
        replacer_->Pin(frame_id); 
        // page并不是进程获取时变dirty，而是进程结束操作，unpin后才变dirty
        // page->is_dirty_ = true;
        latch_.unlock();
        return page;
    }
    return nullptr;
}

// 给定一个页号，对缓冲池中的该页unpin，即取消对该页的引用，表示进程完成了对该页的操作
// is_dirty参数表示该page是否被修改，
// 如果pin_count大于0，直接--
// 如果pin_count等于0，需要加到lru数据结构中
bool BufferPoolManager::UnpinPageImpl(page_id_t page_id, bool is_dirty) { 
    latch_.lock();
    if(page_table_.count(page_id) == 0){
        latch_.unlock();
        return false;
    }
    Page* unpinned_page = &pages_[page_table_[page_id]];
    if(is_dirty){
        unpinned_page->is_dirty_ = true;
    }
    if(unpinned_page->pin_count_ == 0){
        latch_.unlock();
        return false;
    }
    unpinned_page->pin_count_--;
    if(unpinned_page->pin_count_ == 0){
        replacer_->Unpin(page_table_[page_id]);
    }
    latch_.unlock();
    return true;
}

// 给定一个页号，将缓冲池中的该页的内容写回到磁盘中，不需要刷新缓冲池中的page对象
bool BufferPoolManager::FlushPageImpl(page_id_t page_id) {
    latch_.lock();
    if(page_table_.count(page_id) == 0 || page_id == INVALID_PAGE_ID){
        latch_.unlock();
        return false;
    }
    disk_manager_->WritePage(page_id,pages_[page_table_[page_id]].data_);
    latch_.unlock();
    return true;
}
// 在缓冲池中分配一个来自磁盘的新的物理page（由磁盘来决定分配哪个物理页），
// 将page_id写入返回参数中
Page *BufferPoolManager::NewPageImpl(page_id_t *page_id) {
  // 0.   Make sure you call DiskManager::AllocatePage!
  // 1.   If all the pages in the buffer pool are pinned, return nullptr.
  // 2.   Pick a victim page P from either the free list or the replacer. Always pick from the free list first.
  // 3.   Update P's metadata, zero out memory and add P to the page table.
  // 4.   Set the page ID output parameter. Return a pointer to P.
    latch_.lock();
    page_id_t new_page_id = disk_manager_->AllocatePage();
    frame_id_t frame_id;
    if(find_free_frame(&frame_id)){
        Page *new_page = &pages_[frame_id];
        new_page->page_id_ = new_page_id;
        new_page->pin_count_++;
        replacer_->Pin(frame_id);
        page_table_[new_page_id] = frame_id;
        new_page->is_dirty_ = false;
        *page_id = new_page_id;
        disk_manager_->WritePage(new_page->GetPageId(),new_page->GetData());
        latch_.unlock();
        return new_page;
    }
    return nullptr;
}

// 给定一个页号，将缓冲池中的该页移除
bool BufferPoolManager::DeletePageImpl(page_id_t page_id) {
  // 0.   Make sure you call DiskManager::DeallocatePage!
  // 1.   Search the page table for the requested page (P).
  // 1.   If P does not exist, return true.
  // 2.   If P exists, but has a non-zero pin-count, return false. Someone is using the page.
  // 3.   Otherwise, P can be deleted. Remove P from the page table, reset its metadata and return it to the free list.
  // 不用从lru跟踪对象中移除吗？
    latch_.lock();
    if(page_table_.count(page_id) != 0 && pages_[page_table_[page_id]].pin_count_ == 0){
        Page* page = &pages_[page_table_[page_id]];
        if(page->is_dirty_){
            FlushPageImpl(page_id);
        }
        disk_manager_->DeallocatePage(page_id);
        page_table_.erase(page_id);
        // reset metadata
        page->is_dirty_ = false;
        page->pin_count_ = 0;
        page->page_id_ = INVALID_PAGE_ID;
        free_list_.push_back(page_table_[page_id]);
        latch_.unlock();
        return true;
    }
    latch_.unlock();
    return false;
}
// 将缓冲池中的所有页的信息都写回到磁盘中
void BufferPoolManager::FlushAllPagesImpl() {
    for(const auto &p : page_table_){
        FlushPageImpl(p.first);
    }
}

}  // namespace bustub

```

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211028084219070.png" alt="image-20211028084219070" style="zoom:50%;" />

## BPM的并发结构

```cpp
  /** Number of pages in the buffer pool. */
  size_t pool_size_;
  /** Array of buffer pool pages.
   *  存放缓冲区中页的数组，即存放了page的帧*/
  Page *pages_;
  /** Pointer to the disk manager. */
  DiskManager *disk_manager_ __attribute__((__unused__));
  /** Pointer to the log manager. */
  LogManager *log_manager_ __attribute__((__unused__));
  /** 页表，用于将页号转换为该页存放在缓冲区中的帧号*/
  std::unordered_map<page_id_t, frame_id_t> page_table_;
  /** Replacer to find unpinned pages for replacement. */
  Replacer *replacer_;
  /** 双向链表，存放内存中空的帧的帧号*/
  std::list<frame_id_t> free_list_;
  std::mutex latch_;
```

在BPM中有一个锁，此锁用来保护BPM中上面的这些共享数据结构，按照我目前的实现，上层用户访问缓冲池（如fetch，flush，unpin，pin等）都要先获取此锁，对缓冲池的操作结束后再释放此锁，所以目前缓冲池的并发度比较低，几乎只能串行访问缓冲池。

此锁还用来保护page内部的脏位，pin_cnt，page_id等字段，而每个page内部都有读写锁，这个读写锁才是真正用来保护page的数据的，所以在缓冲池中对page进行读写（如修改脏位，pin_cnt等）是不需要加读写锁的，**上层的B+树或堆文件（执行器是更高层的部件，它通过B+树或堆文件进行操作）从缓冲池中获取了想要的page后，才需要获取page内部的读写锁**

BPM和LRUreplacer各用一个大锁，只要对BPM进行任何操作（比如fetchpage或者Unpinpage）就要进行加锁，结束操作后释放锁。对LRUReplacer进行任何操作也要进行加锁，结束操作后释放锁（理论上LRUReplacer在BPM的内部，既然对BPM加了锁就不需要对LRUReplacer再加锁，但是测试时会对LRUReplacer单独测试）

BPM的fetchpage相当于bread，unpinpage相当于brelese。page相当于buffer，内部的data字段才是磁盘中的页（block）的真实数据。

对比一下page结构体和buffer结构体的内容：

- <img src="C:/Users/leeb/AppData/Roaming/Typora/typora-user-images/image-20220226202219496.png" alt="image-20220226202219496" style="zoom:50%;" />
- <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20220226202336709.png" alt="image-20220226202336709" style="zoom:50%;" />











