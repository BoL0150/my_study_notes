# CMU 15-445实验记录（三）：Project 2 B+Tree的插入与删除

## B+Tree的删除的五种情况：

- 叶结点被删除后没有underflow，直接删除对应的key和recordPtr即可

  <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211101205627356.png" alt="image-20211101205627356" style="zoom: 50%;" />

- 叶结点被删除后有underflow，从sibling节点借一个key和recordPtr，从相邻节点借了一个key过来后，两个节点的key的范围都发生了变化，为了正确地反映指针指向的key的范围，必须更新中间节点，也就是父节点的key

  <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211101205903093.png" alt="image-20211101205903093" style="zoom:50%;" />

  <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211101210301082.png" alt="image-20211101210301082" style="zoom:50%;" />

- 叶结点被删除后有underflow，但是sibling的key的数量太少，如果被借走key自己也会underflow。此时需要与一个sibling合并，合并后两个节点的key的范围都发生了变化，为了正确地反映指针指向的key的范围，必须更新中间节点，也就是父节点的key。**调用`Delete (5, rightTree(5))`删除父节点的指针和key**，删除后父节点没有发生underflow。

  <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211101210942387.png" alt="image-20211101210942387" style="zoom:50%;" />

  <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211101211326646.png" alt="image-20211101211326646" style="zoom:50%;" />

  

  <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211101212042658.png" alt="image-20211101212042658" style="zoom:50%;" />

  

  <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211101212601456.png" alt="image-20211101212601456" style="zoom:50%;" />

- 叶结点被删除后有underflow，与一个sibling合并，再调用`Delete (5, rightTree(5))`删除父节点的指针和key，删除后父节点也发生了underflow，父节点再从它的sibling借一个key和recordPtr。

  <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211101212747565.png" alt="image-20211101212747565" style="zoom:50%;" />

  <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211101212924899.png" alt="image-20211101212924899" style="zoom:50%;" />

  

  <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211101213639050.png" alt="image-20211101213639050" style="zoom:50%;" />

  <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211101214926175.png" alt="image-20211101214926175" style="zoom:50%;" />

  <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211101214945937.png" alt="image-20211101214945937" style="zoom:50%;" />

- 叶结点被删除后有underflow，与一个sibling合并后父节点也发生了underflow。父节点underflow后无法从sibling借key，而是与sibling进行合并。父节点merge后需要删除父节点的父节点的一个key，如果没有underflow，则B+Tree的删除到此结束；如果又发生了underflow，如果不是根节点，则还需要对父节点的父节点进行处理；如果是根节点，则删除根节点后结束。

  <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211101222333603.png" alt="image-20211101222333603" style="zoom:50%;" />

  <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211101222452858.png" alt="image-20211101222452858" style="zoom:50%;" />

  <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211101223118895.png" alt="image-20211101223118895" style="zoom:50%;" />

  <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211101223218529.png" alt="image-20211101223218529" style="zoom:50%;" />

  **调用`Delete (13, rightTree(13))`删除父节点的指针和key**

  <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211101223442204.png" alt="image-20211101223442204" style="zoom:50%;" />

  删除后根节点变成了空，删除根节点

  <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211101224240866.png" alt="image-20211101224240866" style="zoom:50%;" />

  <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211101224326124.png" alt="image-20211101224326124" style="zoom:50%;" />

## B+Tree处理叶结点和内部节点的对比

- 从sibling转移

  - 叶结点：

    将左边的（或右边的）sibling的最后一个（或第一个）key和ptr转移到L的第一个位置（或最后一个位置），转移后两个节点的key的范围都发生了变化，必须更新中间节点，也就是将父节点的key更新为新的右子节点的第一个key的值。

    <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211101235115709.png" alt="image-20211101235115709" style="zoom:50%;" />

    <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211101235132307.png" alt="image-20211101235132307" style="zoom:50%;" />

  - 内部节点：

    将左边的（或右边的）sibling的最后一个（或第一个）的ptr转移到N的第一个指针（或最后一个指针），为了保证这个指针指向的范围不变，将指针原位置的key放入父节点中，将父节点原来的key放入N中。

    <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211101235812975.png" alt="image-20211101235812975" style="zoom:50%;" />

    <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211101235838569.png" alt="image-20211101235838569" style="zoom:50%;" />

- 与sibling进行合并：

  - 叶结点：

    如果左sibling存在，就将L的key和recordPtr以及最后一个link ptr合并到左sibling的后面；如果右sibling存在，就将右sibling合并到L的后面。合并后由于父节点少了一个子节点，需要删除父节点中的右子树和对应的key。

    <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211102110544465.png" alt="image-20211102110544465" style="zoom:50%;" />

    <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211102110619459.png" alt="image-20211102110619459" style="zoom:50%;" />

  - 内部节点：

    如果左sibling存在，将N的所有ptr合并到左sibling的后面，为了保证这些指针指向的范围不变，将父节点key和原N的key也放入左sibling中。合并后由于父节点少了一个子节点，需要删除父节点中的右子树和对应的key。

    如果是右sibling存在，则将右sibling所有的ptr合并到N的后面。

    <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211102112826226.png" alt="image-20211102112826226" style="zoom:50%;" />

    <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211102112843010.png" alt="image-20211102112843010" style="zoom:50%;" />

**B+Tree的合并和转移本质上是移动ptr，为了保证这些ptr指向的范围不变，需要相应地修改B+Tree中的key**。

叶结点和内部节点数量的关注点是不同的，叶结点限制的是key的数量；而内部节点限制的是指针的数量。而在具体实现中，内部节点第一个键是不使用的，所以内部节点的kv对的数量就等于指针的数量，叶结点的kv对的数量等于key的数量。

内部节点的下限：**至少有一半的ptr被使用**，即内部节点至少包含最大指针数除二再向上取整的ptr。上限为所有指针都被使用

- <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211102113848010.png" alt="image-20211102113848010" style="zoom:50%;" />

叶结点的下限：至少一半的**key**被使用，即叶结点至少包含最大值数除二再向上取整的**key**。上限为所有**key**都被使用

- <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211103134418240.png" alt="image-20211103134418240" style="zoom:50%;" />

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211103135107121.png" alt="image-20211103135107121" style="zoom:50%;" />

# 具体实现

> 对所有节点来说，不管是叶结点还是内部节点，可以存储的最大**有效**kv对的数量是max_size -1。而max_size只把有效的kv对计算在内，对内部节点来说，第一个key为空，不算有效的kv对，所以内部节点的max_size就等于节点实际最多可以存储的kv对数量；而叶结点的max_size等于节点实际最多可以存储的kv对数量加一。
>
> 除了max_size之外，其他的与size相关的方法返回的是实际的kv对数量。

page对象：

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20220302123952647.png" alt="image-20220302123952647" style="zoom:50%;" />

`BPlusTreeLeafPage`和`BPlusTreeInternalPage`对象继承自`b_plus_tree_page`对象，公共属性为

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20220302124354442.png" alt="image-20220302124354442" style="zoom:50%;" />

除了上面的这些公共字段外，`BPlusTreeLeafPage`和`BPlusTreeInternalPage`对象各有一个kv数组。

Page对象是缓冲池的buffer，从磁盘调入内存的页就存放于Page对象的data字段中，不管是B+树的索引页还是堆文件的数据页都存放于Page对象的data字段中：

- 对于B+树的索引页：每一次我们读取或写入一个叶子或内部页，我们需要先通过唯一的`page_id`将该页从缓冲池中fetch，再使用`reinterpret_cast`将该页的**内容**（**即获取的Page对象的`data_`部分！而不是Page对象本身！**）重新解释为`BPlusTreeLeafPage`或`BPlusTreeInternalPage`对象
- 对于堆文件的数据页：Page对象的整个data部分就是数据页的内容，被分为三个部分：header，空闲空间和tuple部分，所以有一个TablePage类来表示内存中的数据页，该类继承自Page类，没有多出的字段，只有多出的方法，用来操作数据页，也就是Page对象的data部分。当遍历一张表的数据时，需要先根据tableHeap中的first_page_id从缓冲池中fetch该表的第一个数据页，此时获取的是Page对象，要操作数据页还需要将Page对象转成TablePage对象，再调用它的方法来操作data部分的不同偏移量。

而**缓冲池中的page对象实际上相当于文件系统的缓冲池中的buffer对象**，每次read或write系统调用就是在buffer->data和page->data之间读取数据，**buffer->data和page->data才是真正存储在磁盘中的页或块的数据**！而buffer对象和page对象只是内存中用来缓存真实的数据的frame！

读或写结束后，再调用bpm的`Unpin`方法结束对该Page对象的操作。**如果改变了LeafPage对象或InternalPage对象的属性，就相当于改变了Page对象的`data_`部分**，所以还需要将Page对象的dirty置为true。

我们的B +树索引只能支持唯一的键。也就是说，当我们尝试将重复的键插入索引中时，它不应执行插入并返回false。**如果插入导致某一页中的kv对等于`max_size`，需要将这个节点进行split**。

`header_page`记录了系统中的所有索引的名字以及对应索引的root_page_id，所以每当我们对某一个索引的根节点进行修改时，或者创建了一个新的根节点时，都需要调用`UpdateRootPageId`方法插入（参数为true）或更新（参数为false）`header_page`。

B+Tree初始化时给的max_size对叶节点和内部节点的含义不同：

- 对叶节点来说，max_size指节点最多可以容纳的kv对数量加一，**一到达leaf_max_size就需要进行分裂**
- 对内部节点来说，max_size是节点最多可容纳的kv对数量，也就是指针的数量，**到达internal_max_size+1才需要进行分裂**

我也实在不明白为什么要这样设计，这个问题在测试时才发现

因此：

- 对叶节点来说，min_size为max_size/2
- 对内部节点来说，min_size为(max_size+1)/2

小于min_size就发生了underflow，需要coalesce或redistribute

我们还需要指定函数调用Unpin的规则：

- 执行了fetchPage或newPage操作，并且没有将该page作为返回值，那么需要在本函数内unpin该page

- 调用了返回值是page对象的函数（或者返回参数是page对象的函数），那么需要在本函数内unpin该page

一个page对象的使用在哪个函数内结束，哪个函数就负责Unpin这个page

## 实现插入算法

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/2282357-20210125113040220-749897502.png" alt="image-20210124184901398" style="zoom: 50%;" />

如果当前为空树则创建一个新的树插入其中，否则就插入到叶结点中。

```cpp
INDEX_TEMPLATE_ARGUMENTS
bool BPLUSTREE_TYPE::Insert(const KeyType &key, const ValueType &value, Transaction *transaction) {
  if (IsEmpty()) {
    StartNewTree(key, value);
    return true;
  }
  return InsertIntoLeaf(key, value, transaction);
```

### 将一对kv插入到一个空树中

将一对kv插入到一个空树中：从缓冲池中请求一个新的page，这个page就是我们的root page，将page id赋给root_page_id_，同时这个page也是叶page，将这个page对象转化为leaf_page对象，再初始化leaf_page对象，再向其中插入kv对。更新rootPageId，结束操作后，将缓冲池中的该页Unpin。

```cpp
INDEX_TEMPLATE_ARGUMENTS
void BPLUSTREE_TYPE::StartNewTree(const KeyType &key, const ValueType &value) {
  Page *root_page = buffer_pool_manager_->NewPage(&root_page_id_);
  if (root_page == nullptr) {
    throw "out of memory";
  }
  LeafPage *leaf_page = reinterpret_cast<LeafPage *>(root_page->GetData());
  leaf_page->Init(root_page_id_, INVALID_PAGE_ID, leaf_max_size_);
  // true为插入，false为更新
  UpdateRootPageId(true);
  leaf_page->Insert(key, value, comparator_);
  buffer_pool_manager_->UnpinPage(root_page_id_, true);
}
```

向指定叶结点中插入kv对：在该节点中找到大于等于插入key的第一个key，将kv数组中在该key之后的所有kv对向后移动一格，将我们的kv对插入，将页的大小加一。

```cpp
INDEX_TEMPLATE_ARGUMENTS
int B_PLUS_TREE_LEAF_PAGE_TYPE::Insert(const KeyType &key, const ValueType &value, const KeyComparator &comparator) {
  int index = KeyIndex(key, comparator);
  assert(index >= 0);
  int end = GetSize();
  for (int i = end; i > index; i--) {
    array[i].first = array[i - 1].first;
    array[i].second = array[i - 1].second;
  }
  array[index].first = key;
  array[index].second = value;
  IncreaseSize(1);
  return GetSize();
}
```

在叶结点中找到大于等于要插入key的第一个key的index：

```cpp
INDEX_TEMPLATE_ARGUMENTS
int B_PLUS_TREE_LEAF_PAGE_TYPE::KeyIndex(const KeyType &key, const KeyComparator &comparator) const {
  assert(GetSize() >= 0);
  int l = 0;
  int r = GetSize() - 1;
  while (l <= r) {
    int mid = (r - l) / 2 + l;
    if (comparator(array[mid].first, key) < 0) {
      l = mid + 1;
    } else {
      r = mid - 1;
    }
  }
  return r + 1;
}
```

### 查找叶结点并插入其中

在B+树中查找插入元素对应的叶结点并插入其中：

1. 查找到插入元素对应的叶结点
2. 向查找到的叶结点中插入
   1. 由于我们的B+Tree只支持唯一的key，所以如果叶结点中已经存在相同的key，则马上返回false。否则就直接插入。
   2. 如果插入后该叶结点内的kv对个数大于最大值则需要进行split，产生两个新节点，将右边节点的第一个key复制后插入到父节点

#### **1、查找到插入元素对应的叶结点**

从根节点开始查找，根据page_id从缓冲池中获取每一个节点对应的page对象，将page对象重新解释为Internal_page，调用Internal_page的lookup方法，查找到插入元素对应的下一级节点的page_id，再从缓冲池中获取下一级page对象，同时Unpin刚才操作的page对象。直到查找到叶结点为止。

参数中的leftMost是用来在实现Iterator时，先调用此函数定位到最左边的叶子节点的。所以如果leftMost为true，查找时的每一次循环直接使用当前节点中kv数组的第一个page_id即可。

```cpp
INDEX_TEMPLATE_ARGUMENTS
Page *BPLUSTREE_TYPE::FindLeafPage(const KeyType &key, bool leftMost) {
  if (IsEmpty()) {
    return nullptr;
  }
  page_id_t next_page_id = root_page_id_;
  InternalPage *internal_page;
  for (internal_page = reinterpret_cast<InternalPage *>(buffer_pool_manager_->FetchPage(next_page_id)->GetData());
       !internal_page->IsLeafPage();
       internal_page = reinterpret_cast<InternalPage *>(buffer_pool_manager_->FetchPage(next_page_id)->GetData())) {
    page_id_t old_page_id = next_page_id;
    if (leftMost) {
      next_page_id = internal_page->ValueAt(0);
    } else {
      next_page_id = internal_page->Lookup(key, comparator_);
    }
    buffer_pool_manager_->UnpinPage(old_page_id, false);
  }
  return reinterpret_cast<Page *>(internal_page);
}
```

InternalPage的Lookup方法：在某个内部节点的kv数组中查找小于等于输入key的最大的key，对应的ptr就指向包含输入key的page

```cpp
INDEX_TEMPLATE_ARGUMENTS
ValueType B_PLUS_TREE_INTERNAL_PAGE_TYPE::Lookup(const KeyType &key, const KeyComparator &comparator) const {
  assert(GetSize() > 1);
  for (int i = 1; i < GetSize(); i++) {
    if (comparator(key, array[i].first) < 0) {
      return array[i - 1].second;
    }
  }
  return array[GetSize() - 1].second;
}
```

#### **2、向该叶结点中插入**

由于我们的B+Tree只支持唯一的key，所以先查找叶结点中的kv数组，如果叶结点中已经存在相同的key，则马上返回false（**在返回前还需要unpin当前的叶结点**）。否则就直接插入。

- 如果插入后该叶结点内的kv对个数**大于**最大值则需要进行`split`，产生两个新节点（实际上是一个新节点，左节点还是使用之前的指针）

- 再调用`InsertIntoParent`，将右边节点的第一个key和指向右节点的指针插入到父节点中左节点对应的kv对的后面（**split后要记得unpin分裂产生的新的叶结点**）。**插入结束后，还是要记住Unpin叶结点，同时dirty位置为true，因为此页的内容已经被修改了**。
  - 如果插入后，父节点也满了，则还需要对父节点递归进行`split`和`InsertIntoParent`。直到自己变成根节点或父节点没有满为止。

```cpp
INDEX_TEMPLATE_ARGUMENTS
bool BPLUSTREE_TYPE::InsertIntoLeaf(const KeyType &key, const ValueType &value, Transaction *transaction) {
  Page *page = FindLeafPage(key, false);
  LeafPage *leaf_page = reinterpret_cast<LeafPage *>(page->GetData());
  ValueType v;
  if (leaf_page->Lookup(key, &v, comparator_)) {
    buffer_pool_manager_->UnpinPage(leaf_page->GetPageId(), false);
    return false;
  }
  leaf_page->Insert(key, value, comparator_);
  // 到达叶节点的max_size就需要进行分裂
  if (leaf_page->GetSize() >= leaf_page->GetMaxSize()) {
    LeafPage *new_leaf_page = Split(leaf_page);
    InsertIntoParent(leaf_page, new_leaf_page->KeyAt(0), new_leaf_page, transaction);
    buffer_pool_manager_->UnpinPage(new_leaf_page->GetPageId(), true);
  }
  buffer_pool_manager_->UnpinPage(leaf_page->GetPageId(), true);
  return true;
}
```

##### split

对叶结点split：

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211105114457821.png" alt="image-20211105114457821" style="zoom: 50%;" />

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211105114616317.png" alt="image-20211105114616317" style="zoom:50%;" />

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211105114650496.png" alt="image-20211105114650496" style="zoom:50%;" />

对内部节点split：

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211105114756223.png" alt="image-20211105114756223" style="zoom:33%;" />

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211105114858459.png" alt="image-20211105114858459" style="zoom:50%;" />

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211105114930711.png" alt="image-20211105114930711" style="zoom:50%;" />



`split`：首先从缓冲池中请求一个新的page对象，将它重新解释为Internal_page或leaf_page，再进行初始化。将输入的page中的一半的kv对移动到新的page对象中，再修改next_page_id的指向，将新的page插入到输入page的后面（如果输入的是内部节点，则不需要修改next_page_id的指向）。**注意，如果是内部节点，新的page中的第一个key就是要送给父节点的key，刚好可以忽略它，这样就将leaf_page和Internal_page的`MoveHalfTo`的操作统一起来了。**

```cpp
INDEX_TEMPLATE_ARGUMENTS
template <typename N>
N *BPLUSTREE_TYPE::Split(N *node) {
  page_id_t new_page_id;
  Page *new_page = buffer_pool_manager_->NewPage(&new_page_id);
  if (new_page == nullptr) {
    throw "out of memory";
  }
  if (node->IsLeafPage()) {
    LeafPage *new_leaf_page = reinterpret_cast<LeafPage *>(new_page->GetData());
    LeafPage *old_leaf_page = reinterpret_cast<LeafPage *>(node);
    new_leaf_page->Init(new_page_id, old_leaf_page->GetParentPageId(), leaf_max_size_);
    old_leaf_page->MoveHalfTo(new_leaf_page);
    new_leaf_page->SetNextPageId(old_leaf_page->GetNextPageId());
    old_leaf_page->SetNextPageId(new_page_id);
  } else {
    InternalPage *new_internal_page = reinterpret_cast<InternalPage *>(new_page->GetData());
    InternalPage *old_internal_page = reinterpret_cast<InternalPage *>(node);
    new_internal_page->Init(new_page_id, old_internal_page->GetParentPageId(), internal_max_size_);
    // 注意！！内部节点分裂后，需要更新分配给新的内部节点的所有子节点的父节点指向
    old_internal_page->MoveHalfTo(new_internal_page, buffer_pool_manager_);
  }
  return reinterpret_cast<N *>(new_page->GetData());
}
```

`leaf_page`的`MoveHalfTo`方法：

`MoveHalfTo`：调用copyNFrom方法从被调用leaf_page对象的array中移动一半的kv对到recipient中，再更新两个leaf_page对象的大小。

```cpp
INDEX_TEMPLATE_ARGUMENTS
void B_PLUS_TREE_LEAF_PAGE_TYPE::MoveHalfTo(BPlusTreeLeafPage *recipient) {
  int move_nums = GetSize() / 2;
  MappingType *start = array + GetSize() - move_nums;
  recipient->CopyNFrom(start, move_nums);
  this->IncreaseSize(-1 * move_nums);
  recipient->IncreaseSize(move_nums);
}

```

`CopyNFrom`：将从start开始的size个对象复制到this对象的array中

```cpp
INDEX_TEMPLATE_ARGUMENTS
void B_PLUS_TREE_LEAF_PAGE_TYPE::CopyNFrom(MappingType *items, int size) {
  for (int cnt = 0; cnt < size; cnt++) {
    this->array[cnt] = *(items + cnt);
  }
}
```

`internal_page`的`MoveHalfTo`方法：

```cpp
INDEX_TEMPLATE_ARGUMENTS
void B_PLUS_TREE_INTERNAL_PAGE_TYPE::MoveHalfTo(BPlusTreeInternalPage *recipient,
                                                BufferPoolManager *buffer_pool_manager) {
  int move_nums = GetSize() / 2;
  MappingType *start = array + GetSize() - move_nums;
  recipient->CopyNFrom(start, move_nums, buffer_pool_manager);
  this->IncreaseSize(-1 * move_nums);
  recipient->IncreaseSize(move_nums);
}
```

`CopyNFrom`：与叶节点的相同，需要注意的一点是：**内部节点分裂后，需要更新分配给新的内部节点的所有子节点的父节点指针**

```cpp
INDEX_TEMPLATE_ARGUMENTS
void B_PLUS_TREE_INTERNAL_PAGE_TYPE::CopyNFrom(MappingType *items, int size, BufferPoolManager *buffer_pool_manager) {
  for (int cnt = 0; cnt < size; cnt++) {
    this->array[cnt] = *(items + cnt);
    // 注意！！内部节点分裂后，需要更新分配给新的内部节点的所有子节点的父节点指针
    auto child_page = reinterpret_cast<BPlusTreePage *>(buffer_pool_manager->FetchPage(array[cnt].second)->GetData());
    child_page->SetParentPageId(this->GetPageId());
    buffer_pool_manager->UnpinPage(array[cnt].second, true);
  }
}
```

##### InsertIntoParent

`InsertIntoParent`：

- 如果old_node就是根节点，那么需要向缓冲池请求一个新的页来创建新的根节点，初始化根节点，更新root_page_id的值（**以及`header_page`**），调用`PopulateNewRoot`方法使用`old_node`，key和`new_node`填充新创建的根节点，再修改`old_node`和`new_node`的父指针。最后还需要调用Unpin结束对刚才从缓冲池中获取的新的根页的操作。

- 找到old_node的父节点，调用`InsertNodeAfter`方法将new_ndoe和第一个key插入到父节点中old_ndoe的后面，再修改`new_node`的父指针

  - 如果插入后的大小超过maxsize，还需要对父节点进行`split`和`InsertIntoParent`（**split后要记得unpin分裂产生的新的叶结点**）

  对父节点进行unpin，结束操作。

```cpp
INDEX_TEMPLATE_ARGUMENTS
void BPLUSTREE_TYPE::InsertIntoParent(BPlusTreePage *old_node, const KeyType &key, BPlusTreePage *new_node,
                                      Transaction *transaction) {
  if (old_node->IsRootPage()) {
    // 创建新的根节点要更新root_page_id_和header_page
    Page *new_page = buffer_pool_manager_->NewPage(&root_page_id_);
    UpdateRootPageId(false);
    InternalPage *new_root_page = reinterpret_cast<InternalPage *>(new_page->GetData());
    new_root_page->Init(root_page_id_, INVALID_PAGE_ID, internal_max_size_);
    new_root_page->PopulateNewRoot(old_node->GetPageId(), key, new_node->GetPageId());
    old_node->SetParentPageId(root_page_id_);
    new_node->SetParentPageId(root_page_id_);
    buffer_pool_manager_->UnpinPage(root_page_id_, true);
  } else {
    page_id_t parent_page_id = old_node->GetParentPageId();
    InternalPage *parent_page =
        reinterpret_cast<InternalPage *>(buffer_pool_manager_->FetchPage(parent_page_id)->GetData());
    parent_page->InsertNodeAfter(old_node->GetPageId(), key, new_node->GetPageId());
    new_node->SetParentPageId(parent_page_id);
    // 对内部节点来说，到达max_size+1才需要进行分裂
    if (parent_page->GetSize() > parent_page->GetMaxSize()) {
      InternalPage *uncle_page = Split(parent_page);
      InsertIntoParent(parent_page, uncle_page->KeyAt(0), uncle_page);
      buffer_pool_manager_->UnpinPage(uncle_page->GetPageId(), true);
    }
    buffer_pool_manager_->UnpinPage(parent_page_id, true);
  }
}
```

`PopulateNewRoot`：在当前page中填充两个value（page_id）和一个key，变成一个新的root_page，再设置大小为2 。此方法只能在`InsertIntoParent`中调用

```cpp
INDEX_TEMPLATE_ARGUMENTS
void B_PLUS_TREE_INTERNAL_PAGE_TYPE::PopulateNewRoot(const ValueType &old_value, const KeyType &new_key,
                                                     const ValueType &new_value) {
  array[0].second = old_value;
  array[1].first = new_key;
  array[1].second = new_value;
  SetSize(2);
}
```

`InsertNodeAfter`：在当前page中找到old_value的位置，然后将new_key和new_value插入其中

```cpp
INDEX_TEMPLATE_ARGUMENTS
int B_PLUS_TREE_INTERNAL_PAGE_TYPE::InsertNodeAfter(const ValueType &old_value, const KeyType &new_key,
                                                    const ValueType &new_value) {
  int index = ValueIndex(old_value);
  for (int i = GetSize(); i > index + 1; i--) {
    array[i] = array[i - 1];
  }
  array[index + 1] = MappingType(new_key, new_value);
  IncreaseSize(1);
  return GetSize();
}
```

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211107225937538.png" alt="image-20211107225937538" style="zoom:50%;" />

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211108084900459.png" alt="image-20211108084900459" style="zoom:50%;" />



## 实现删除算法

先找到包含目标key的叶节点（如果是空树则直接返回），再调用`RemoveAndDeleteRecord`在叶节点上直接删除对应的key值（如果该叶中不存在该key，则直接返回）。删除后如果叶节点中key的个数小于最小值，则调用`CoalesceAndRedistribute`函数处理underflow。

```cpp
INDEX_TEMPLATE_ARGUMENTS
void BPLUSTREE_TYPE::Remove(const KeyType &key, Transaction *transaction) {
  if (IsEmpty()) {
    return;
  }
  Page *page = FindLeafPage(key, false);  // unpin
  LeafPage *leaf_page = reinterpret_cast<LeafPage *>(page->GetData());
  leaf_page->RemoveAndDeleteRecord(key, comparator_);
  assert(leaf_page != nullptr);
  if (leaf_page->GetSize() < leaf_page->GetMinSize()) {
    CoalesceOrRedistribute(leaf_page, transaction);
  }
  buffer_pool_manager_->UnpinPage(leaf_page->GetPageId(), true);
}
```

`CoalesceAndRedistribute`函数：处理节点的underflow

- 如果是根节点则调用`AdjustRoot`函数，处理根节点的underflow
- 如果不是根节点，则先获取该page的兄弟page。（先获取左边的sibling，如果当前节点的index为0，则获取右边的sibling）
  - 如果这两个page可以合并到同一个page中，则调用`Coalesce`函数进行合并。（在Coalesce函数中，我们统一认为参数neighbor_node为左边的节点，node为右边的节点，所以在调用Coalesce前，如果sibling不是在node的前面，我们需要转换两个指针的指向）
  - 否则只能调用`Redistribute`函数进行借节点

```cpp
INDEX_TEMPLATE_ARGUMENTS
template <typename N>
bool BPLUSTREE_TYPE::CoalesceOrRedistribute(N *node, Transaction *transaction) {
  assert(node != nullptr);
  if (node->IsRootPage()) {
    return AdjustRoot(node);
  }
  N *sibling;
  // 获取的sibling是否是node的下一个节点
  // Unpin
  bool node_prev_sibling = FindSibling(node, &sibling);
  // unpin
  InternalPage *parent_page =
      reinterpret_cast<InternalPage *>(buffer_pool_manager_->FetchPage(node->GetParentPageId())->GetData());

  assert(node != nullptr && sibling != nullptr);
  if (node->GetSize() + sibling->GetSize() <= MaxSize(node)) {
    // 统一认为sibling在左边，node在右边
    if (node_prev_sibling) {
      N *temp = sibling;
      sibling = node;
      node = temp;
    }
    // index是右边的节点在父节点中的index
    int index = parent_page->ValueIndex(node->GetPageId());
    Coalesce(&sibling, &node, &parent_page, index, transaction);
    buffer_pool_manager_->UnpinPage(parent_page->GetPageId(), true);
    buffer_pool_manager_->UnpinPage(sibling->GetPageId(), true);
    return true;
  }
  // index为underflow的节点在父节点中的index
  int index = parent_page->ValueIndex(node->GetPageId());
  Redistribute(sibling, node, index);
  buffer_pool_manager_->UnpinPage(parent_page->GetPageId(), false);
  buffer_pool_manager_->UnpinPage(sibling->GetPageId(), true);
  return false;
}
```

```cpp
INDEX_TEMPLATE_ARGUMENTS
template <typename N>
int BPLUSTREE_TYPE::MaxSize(N *node) {
  return node->IsLeafPage() ? node->GetMaxSize() - 1 : node->GetMaxSize();
}
```

获取sibling节点：

```cpp
// 如果获取的是前一个sibling，则返回false；下一个sibling，则返回true
INDEX_TEMPLATE_ARGUMENTS
template <typename N>
bool BPLUSTREE_TYPE::FindSibling(N *node, N **sibling) {
  // unpin
  InternalPage *parent_page =
      reinterpret_cast<InternalPage *>(buffer_pool_manager_->FetchPage(node->GetParentPageId())->GetData());
  int index = parent_page->ValueIndex(node->GetPageId());
  int sibling_index = index - 1;
  if (index == 0) {
    sibling_index = index + 1;
  }
  page_id_t sibling_page_id = parent_page->ValueAt(sibling_index);
  (*sibling) = reinterpret_cast<N *>(buffer_pool_manager_->FetchPage(sibling_page_id)->GetData());
  buffer_pool_manager_->UnpinPage(parent_page->GetPageId(), false);
  return index == 0;
}
```

`AdjustRoot`：处理根节点的underflow，有两种情况：

- 根节点在underflow之前只有两个子节点，两个子节点合并后，只剩下了一个子节点，**此时根节点为内部节点，且节点内只有一个指针，即根节点的大小为1**。此时需要调用`RemoveAndReturnOnlyChild`函数把根节点删除，并返回该指针指向的唯一的子节点。将子节点更新为新的根节点。
- 整棵B+Tree中的所有值都被删除了，B+Tree为空，此时**根节点为叶结点，且大小为0**，直接将根节点删除即可

否则不需要有page被删除，则直接return flase

```cpp
INDEX_TEMPLATE_ARGUMENTS
bool BPLUSTREE_TYPE::AdjustRoot(BPlusTreePage *old_root_node) {
  // case 2: when you delete the last element in whole b+ tree
  assert(old_root_node != nullptr);
  if (old_root_node->IsLeafPage()) {
    assert(old_root_node->GetSize() == 0);
    assert(old_root_node->GetParentPageId() == INVALID_PAGE_ID);
    buffer_pool_manager_->UnpinPage(old_root_node->GetPageId(), false);
    buffer_pool_manager_->DeletePage(root_page_id_);
    root_page_id_ = INVALID_PAGE_ID;
    // false更新
    UpdateRootPageId(false);
    // 删除了一个page，返回true
    return true;
  }
  // case 1: when you delete the last element in root page, but root page still
  // has one last child
  if (old_root_node->GetSize() == 1) {
    InternalPage *root_page = reinterpret_cast<InternalPage *>(old_root_node);
    page_id_t new_root_page_id = root_page->RemoveAndReturnOnlyChild();
    root_page_id_ = new_root_page_id;
    UpdateRootPageId(false);
    // 将新的根节点的父节点置为INVALID
    InternalPage *new_root_page =
        reinterpret_cast<InternalPage *>(buffer_pool_manager_->FetchPage(new_root_page_id)->GetData());
    new_root_page->SetParentPageId(INVALID_PAGE_ID);
    buffer_pool_manager_->UnpinPage(new_root_page_id, true);
    buffer_pool_manager_->UnpinPage(old_root_node->GetPageId(), false);
    buffer_pool_manager_->DeletePage(old_root_node->GetPageId());
    return true;
  }
  return false;
}
```



### Coalesce的过程

叶节点中的合并：

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211106183635881.png" alt="image-20211106183635881" style="zoom:50%;" />

内部节点中的合并：

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211106183835223.png" alt="image-20211106183835223" style="zoom:50%;" />



`Coalesce`函数：参数为左边的节点，右边的节点，父节点，右边节点在父节点中对应的index。因为在合并时，需要删除右边节点在父节点中对应的指针和key。

调用`MoveAllTo`函数，将右边节点中的所有kv对移动到左边的节点。

- 对叶节点，直接移动即可
- 对内部节点，需要先将右边节点在父节点中对应的key移动到左边节点，再移动右边中的所有的kv对。还需要更新被合并的page的所有子节点的parent_page_id。所以调用内部节点的`MoveAllTo`函数时，还需要传入右边节点在父节点中对应的key，以及bpm

再删除右节点的page，调用`remove`函数移除父节点中对应的KV对。如果父节点又发生了underflow，还需要调用`CoalesceAndRedistribute`递归处理父节点。

```cpp
INDEX_TEMPLATE_ARGUMENTS
template <typename N>
bool BPLUSTREE_TYPE::Coalesce(N **neighbor_node, N **node,
                              BPlusTreeInternalPage<KeyType, page_id_t, KeyComparator> **parent, int index,
                              Transaction *transaction) {
  if ((*node)->IsLeafPage()) {
    LeafPage *leaf_node = reinterpret_cast<LeafPage *>(*node);
    LeafPage *leaf_neighbor_node = reinterpret_cast<LeafPage *>(*neighbor_node);
    leaf_node->MoveAllTo(leaf_neighbor_node);
    leaf_neighbor_node->SetNextPageId(leaf_node->GetNextPageId());
  } else {
     
    InternalPage *internal_node = reinterpret_cast<InternalPage *>(*node);
    InternalPage *internal_neighbor_node = reinterpret_cast<InternalPage *>(*neighbor_node);
    internal_node->MoveAllTo(internal_neighbor_node, (*parent)->KeyAt(index), buffer_pool_manager_);
  }
  buffer_pool_manager_->UnpinPage((*node)->GetPageId(),true);
  buffer_pool_manager_->DeletePage((*node)->GetPageId());
  (*parent)->Remove(index);
  assert((*parent) != nullptr);
  if ((*parent)->GetSize() < (*parent)->GetMinSize()) {
    return CoalesceOrRedistribute((*parent), transaction);
  }
  return true;
}
```

叶节点的`MoveAllTo`方法：

```cpp
INDEX_TEMPLATE_ARGUMENTS
void B_PLUS_TREE_LEAF_PAGE_TYPE::MoveAllTo(BPlusTreeLeafPage *recipient) {
  assert(recipient != nullptr);
  int move_num = this->GetSize();
  int recipient_start_index = recipient->GetSize();
  for (int i = 0; i < move_num; i++) {
    recipient->array[recipient_start_index + i] = this->array[i];
  }
  this->IncreaseSize(-1 * move_num);
  recipient->IncreaseSize(move_num);
}
```

内部节点的`MoveAllTo`方法：

```cpp
INDEX_TEMPLATE_ARGUMENTS
void B_PLUS_TREE_INTERNAL_PAGE_TYPE::MoveAllTo(BPlusTreeInternalPage *recipient, const KeyType &middle_key,
                                               BufferPoolManager *buffer_pool_manager) {
  int move_num = this->GetSize();
  assert(recipient != nullptr);
  int recipient_start_index = recipient->GetSize();
  this->SetKeyAt(0, middle_key);
  for (int i = 0; i < move_num; i++) {
    recipient->array[recipient_start_index + i] = this->array[i];
    BPlusTreePage *child_page =
        reinterpret_cast<BPlusTreePage *>(buffer_pool_manager->FetchPage(array[i].second)->GetData());
    child_page->SetParentPageId(recipient->GetParentPageId());
    buffer_pool_manager->UnpinPage(array[i].second, true);
  }
  this->IncreaseSize(-1 * move_num);
  recipient->IncreaseSize(move_num);
}
```

### Redistribute的过程

叶节点：

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211106190642062.png" alt="image-20211106190642062" style="zoom:50%;" />

内部节点：

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211106190731892.png" alt="image-20211106190731892" style="zoom:50%;" />

`Redistribute`函数：参数为sibling节点，underflow的节点，underflow的节点在父节点中对应的index。

- 如果underflow的节点在左边（index等于0）：
  - 如果是叶节点，则调用叶节点的`MoveFirstToEndOf`函数，此函数将sibling的第一个kv对移动到underflow节点的后面。然后修改**右边**节点在父节点中对应的key
  - 否则调用内部节点的`MoveFirstToEndOf`函数。
- 如果underflow的节点在右边，与上面类似

```cpp
INDEX_TEMPLATE_ARGUMENTS
template <typename N>
void BPLUSTREE_TYPE::Redistribute(N *neighbor_node, N *node, int index) {
  InternalPage *parent_page =
      reinterpret_cast<InternalPage *>(buffer_pool_manager_->FetchPage(node->GetParentPageId())->GetData());
  if (index == 0) {
    // 右边节点在父节点中对应的index
    int index = parent_page->ValueIndex(neighbor_node->GetPageId());
    if (neighbor_node->IsLeafPage()) {
      LeafPage *Leaf_neighbor_node = reinterpret_cast<LeafPage *>(neighbor_node);
      LeafPage *Leaf_node = reinterpret_cast<LeafPage *>(node);
      Leaf_neighbor_node->MoveFirstToEndOf(Leaf_node);
      parent_page->SetKeyAt(index, Leaf_neighbor_node->KeyAt(0));
    } else {
      InternalPage *Internal_neighbor_node = reinterpret_cast<InternalPage *>(neighbor_node);
      InternalPage *Internal_node = reinterpret_cast<InternalPage *>(node);
      Internal_neighbor_node->MoveFirstToEndOf(Internal_node, parent_page->KeyAt(index), buffer_pool_manager_);
      parent_page->SetKeyAt(index, Internal_neighbor_node->KeyAt(0));
    }
  } else {
    // 右边节点在父节点中对应的index
    int index = parent_page->ValueIndex(node->GetPageId());
    if (neighbor_node->IsLeafPage()) {
      LeafPage *Leaf_neighbor_node = reinterpret_cast<LeafPage *>(neighbor_node);
      LeafPage *Leaf_node = reinterpret_cast<LeafPage *>(node);
      Leaf_neighbor_node->MoveLastToFrontOf(Leaf_node);
      parent_page->SetKeyAt(index, Leaf_node->KeyAt(0));
    } else {
      InternalPage *Internal_neighbor_node = reinterpret_cast<InternalPage *>(neighbor_node);
      InternalPage *Internal_node = reinterpret_cast<InternalPage *>(node);
      Internal_neighbor_node->MoveLastToFrontOf(Internal_node, parent_page->KeyAt(index), buffer_pool_manager_);
      parent_page->SetKeyAt(index, Internal_node->KeyAt(0));
    }
  }
  buffer_pool_manager_->UnpinPage(parent_page->GetPageId(), true);
}
```

## 实现迭代器

迭代器的范围为最左边的叶节点，到最右边的叶节点的最后一个kv对的下一个kv对（end）。

`index_iterator.h`：

```cpp
  IndexIterator(B_PLUS_TREE_LEAF_PAGE_TYPE *leftmost_leaf, int index, BufferPoolManager *bpm);
  ~IndexIterator();

  bool isEnd();
  const MappingType &operator*();
  IndexIterator &operator++();
  bool operator==(const IndexIterator &itr) const {
    return itr.cur_leaf_page_ == cur_leaf_page_ && itr.index_ == index_;
  }

  bool operator!=(const IndexIterator &itr) const {
    INDEXITERATOR_TYPE temp_it = *this;
    return !(itr == temp_it);
  }

 private:
  // add your own private member variables here
  B_PLUS_TREE_LEAF_PAGE_TYPE *cur_leaf_page_;
  int index_;
  BufferPoolManager *buffer_pool_manager_;
```

`index_iterator.cpp`：

```cpp
INDEX_TEMPLATE_ARGUMENTS
INDEXITERATOR_TYPE::IndexIterator(B_PLUS_TREE_LEAF_PAGE_TYPE *leftmost_leaf, int index, BufferPoolManager *bpm)
    : cur_leaf_page_(leftmost_leaf), index_(index), buffer_pool_manager_(bpm) {}

INDEX_TEMPLATE_ARGUMENTS
INDEXITERATOR_TYPE::~IndexIterator(){
  if(cur_leaf_page_ != nullptr){
    buffer_pool_manager_->UnpinPage(cur_leaf_page_->GetPageId(),false);
  }
}

INDEX_TEMPLATE_ARGUMENTS
bool INDEXITERATOR_TYPE::isEnd() {
  assert(cur_leaf_page_ != nullptr);
  return cur_leaf_page_->GetNextPageId() == INVALID_PAGE_ID && index_ == cur_leaf_page_->GetSize();
}

INDEX_TEMPLATE_ARGUMENTS
const MappingType &INDEXITERATOR_TYPE::operator*() { return cur_leaf_page_->GetItem(index_); }

INDEX_TEMPLATE_ARGUMENTS
INDEXITERATOR_TYPE &INDEXITERATOR_TYPE::operator++() {
  index_++;
  assert(cur_leaf_page_ != nullptr);
  if (index_ >= cur_leaf_page_->GetSize() && cur_leaf_page_->GetNextPageId() != INVALID_PAGE_ID) {
    page_id_t next_page_id = cur_leaf_page_->GetNextPageId();
    buffer_pool_manager_->UnpinPage(cur_leaf_page_->GetPageId(), false);
    cur_leaf_page_ =
        reinterpret_cast<B_PLUS_TREE_LEAF_PAGE_TYPE *>(buffer_pool_manager_->FetchPage(next_page_id)->GetData());
    index_ = 0;
  }
  return *this;
}
```

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211111102421122.png" alt="image-20211111102421122" style="zoom: 50%;" />

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211111233513671.png" alt="image-20211111233513671" style="zoom: 50%;" />

如果是用迭代器访问链表这样的数据结构，修改之后不会导致迭代器失效；如果访问的是线性的数据结构，修改后会导致迭代器失效（因为迭代器内部保存的是数据结构中的索引）

# 并发控制

## 读写锁的实现

```cpp
  /**
   * Acquire a read latch.
   */
 void RLock() {
    std::unique_lock<mutex_t> latch(mutex_);
    while (writer_entered_ || reader_count_ == MAX_READERS) {
      reader_.wait(latch);
    }
    reader_count_++;
  }  

void WLock() {
    std::unique_lock<mutex_t> latch(mutex_);
    while (writer_entered_) {
      reader_.wait(latch);
    }
    writer_entered_ = true;// 声明写者已经在排队了，后面的读者就要跟在后面排队，防止读者插队导致写者饿死
    while (reader_count_ > 0) {
      writer_.wait(latch);
    }
  }
  /**
   * Release a write latch.
   */
  void WUnlock() {
    std::lock_guard<mutex_t> guard(mutex_);
    writer_entered_ = false;
    reader_.notify_all(); // 由于获取写锁后，所有的读进程都会被阻塞，而读锁可以多次被获取，所以释放写锁时是将所有的读进程都唤醒，调用notify_all。并且由于写进程也会被阻塞，所以为了唤醒时更方便，将写进程被写进程阻塞wait在reader上。
  }

  /**
   * Release a read latch.
   */
  void RUnlock() {
    std::lock_guard<mutex_t> guard(mutex_);
    reader_count_--;
    if (writer_entered_) {
      if (reader_count_ == 0) {
        writer_.notify_one();// 由于释放读锁只会唤醒一个写者，所以只需要对writer调用notify_one就可以了
      }
    } else {
      if (reader_count_ == MAX_READERS - 1) {
        reader_.notify_one();// 当写者数量达到上限时，释放写者也只会唤醒一个写者，所以只需要对reader调用notify_one
      }
    }
  }

 private:
  mutex_t mutex_;
  cond_t writer_;
  cond_t reader_;
  uint32_t reader_count_{0};
  bool writer_entered_{false};
```

以上是读写锁标准写法，也可以使用简化版的写法，不需要使用两个条件变量，将读者和写者都wait在同一个条件变量上也可以。唯一需要注意的地方是：写者开始等待写锁的时候就要开始标记，防止

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
  // cond_t reader_;
  uint32_t reader_count_{0};
  bool writer_entered_{false};
```



**缓冲池中**的Page对象（即page frame）中有一个读写锁字段，这个Page对象的data部分就是B+树索引中的IntenalPage或LeafPage，可以通过这个读写锁对B+树节点上锁。**我们只能获取page frame的读写锁，不能获取B+Tree节点的读写锁**（比如leafpage或internalpage）

读写锁的实现中，使用了两种锁，std::unique_lock 与std::lock_guard都能实现自动加锁与解锁功能，但是std::unique_lock要比std::lock_guard更灵活

- 二者的主要区别是： unique_lock 虽然一样不可复制（non-copyable），但是它是可以转移的（movable）。所以，unique_lock 不但可以被函数回传，也可以放到 STL 的 container 里。**std::unique_lock增加了灵活性，比如可以对mutex的管理从一个scope通过move语义转到另一个scope，不像lock_guard只能在一个scope中生存**。同时也增加了管理的难度，因此如无必要还是用lock_guard。

**所以我们需要在获取锁的时候使用std::unique_lock，因为std::unique_lock作为条件锁，需要被传入到sleep中（即wait）解锁**。在释放锁（unlock）的时候使用lock_guard。

对B+树的操作中传入了一个transaction指针参数，transaction类中提供了一个deque来保存遍历B+树时上锁的page，还提供了一个set来保存在Remove操作时删除的page（缓冲池的delete page就是将缓冲池中的这个page的内容清空，metadata重置，再将这个page加入freeList中，即这个frame可以被重新使用）

我们必须先释放一个page的锁，再unpin这个page

所有线程都从根到底部获取B+Tree的锁，当我们释放锁的时候，也要以相同的顺序（也就是从根到底部），以避免死锁

当我们在插入或删除时，成员变量**`root_page_id`**也会被更新，我们需要使用`std::mutex`来保护这个共享的变量

Coalesce和Redistribute的时候要获取兄弟节点的锁

我们的加锁方式：

1. 对于插入和删除操作，我们不能一进入子节点就释放父节点的锁，由于B+树的特性，我们对子节点的修改操作也可能会导致父节点被修改，所以我们需要先判断子节点是否是安全的，也就是对子节点的插入和删除是否会导致父节点的修改，如果子节点是安全的，才可以释放父节点的锁

2. 并且我们还需要先获取了子节点的锁，再释放父节点的锁，否则会导致释放父节点的锁和获取子节点的锁之间存在空窗期，使后面的线程乘虚而入，将我们要读的东西取走

3. 以上在修改危险子节点时对父节点加锁的方法，杜绝了在修改子节点导致父节点的修改时，会导致其他的线程获取到了父节点之前的旧的错误的信息，而不是修改后的信息的情况。但是没有考虑到根节点的情况，如果修改了根节点，也会导致root_page_id的变化，比如当插入根节点导致根节点的分裂时，将root_page_id更新，但是另一个线程却还拿着原来的root_page_id，这个id现在已经变成了最左边的叶结点的id了。

   所以对于root_page_id也要采取上面的相同策略，对root_page_id也要维护一个读写锁

   - 当读操作时，对root_page_id加读锁，获取了根节点的读锁后释放root_page_id的读锁
   - 当插入和删除时，对root_page_id加写锁，获取了根节点的写锁后判断根节点是否安全，如果安全才能释放root_page_id的写锁。

   实际上我们可以将root_page_id加入到txn中，在所有的page之前检查

并发错误写法：

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20230303213041570.png" alt="image-20230303213041570" style="zoom: 50%;" />

![image-20230303213144695](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230303213144695.png)

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20230303211534711.png" alt="image-20230303211534711" style="zoom:50%;" />

如果两个线程同时调用此函数

1. 线程1先从缓冲池获取一个新页，然后释放写锁，获取leaf_page，获取读锁之前，
2. 线程2获取写锁，又从缓冲池获取了一个页，改变了root_page_id的值，释放写锁，
3. 线程1再获取读锁，用root_page_id将leaf_page初始化，但是此时root_page_id已经被线程2改变了，不再是leaf_page真正的id了。最后会导致有两个leafPage的id相同，并且缓冲池中此id的Page被unpin了两次，而线程1从缓冲池中第一次获取的Page却没有被unpin，导致缓冲池中的此Page永远无法被替换出去，出现了内存泄漏

~~所以我们并发编程必须要采用类似蟹行协议的方式，释放当前锁之前必须获取下一个锁~~

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20230303214649912.png" alt="image-20230303214649912" style="zoom:50%;" />

~~改成如上方式后还是有问题，此函数内的root_page_id是保持一致了，但是考虑如下情况：两个线程同时进入此函数，线程1执行完此函数后，线程2又会重新执行一遍，导致内存中存在两个root_page，也就是两颗树。~~

~~错误原因在于函数外的IsEmpty()函数用了root_page_id，读了root_page_id之后再修改它，需要对这个过程保持原子操作，**要求读出来的结果和修改的结果保持一致**，所以需要在root_page_id读之前加读锁，在修改完root_page_id之后将读锁释放~~

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20230303223541687.png" alt="image-20230303223541687" style="zoom: 50%;" />

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20230303223603832.png" alt="image-20230303223603832" style="zoom:50%;" />

以上全部都是错的，还没释放写锁就获取同一个锁的读锁，直接造成死锁。前面要读root_page_id，中间修改root_page_id，后面又要读root_page_id，这相当于多个事务操作一个共享变量root_page_id，以上第一次的做法违反了2pl的要求，实际上整个过程都应该加一个写锁，要保持整个过程的状态的一致，原子性。

## 具体实现

每个txn内部有一个deque，插入删除和读都需要调用findleafpage从根到底部查找page，所以findleafpage中，对于每一层的page，

- 如果是读操作，获取当前page的读锁，直接释放txn中的deque中的所有page的锁；
- 如果是写操作，获取当前page的写锁，如果当前的page是safe的情况下，才释放deque中的所有的page。
  - safe的含义：如果是插入操作，该page的size要小于page的最大值；如果是删除操作，该page的size要大于最小值

然后将当前的page加入deque，获取下一个page，一直到叶结点

![image-20230304153707264](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230304153707264.png)

我们将对page解锁和unpin的操作合到一个函数中，遍历txn中的pageSet，如果是读操作就释放读锁，如果是写操作就释放写锁，释放锁之后再进行unpin。最后将txn中的pageSet清空

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20230304153831922.png" alt="image-20230304153831922" style="zoom: 50%;" />



## 乐观锁

用这种方法我们发现，在最悲观的情况下，我们担心最后会对根节点进行修改，每次对索引进行修改首先都要对根节点加写锁，在此期间其他线程完全无法使用索引，然而绝大多数情况下，对叶结点的修改是不会导致根节点的修改的，这就成了高并发情况下的瓶颈。

所以我们可以采用乐观锁，在findLeafPage函数中：

1. 乐观地预计只有叶结点会修改，从根节点到叶结点一路上只加读锁，并且逐步释放，获取了子节点的读锁后就释放父节点的读锁
2. 如果叶结点需要修改才对叶结点加写锁，
   1. 如果叶结点是安全的，照常将父节点的锁释放，然后返回找到的叶结点
   2. 如果不是安全的，就需要像之前没有引入乐观锁一样，先将txn中的pageset清空，重新执行之前的版本的findLeafpage，也就是从上到下加写锁

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20230304154316601.png" alt="image-20230304154316601" style="zoom:50%;" />

使用悲观锁的执行时间：

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20230304154449043.png" alt="image-20230304154449043" style="zoom:50%;" />

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20230304154935604.png" alt="image-20230304154935604" style="zoom: 67%;" />

使用乐观锁的执行时间：

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20230304154513259.png" alt="image-20230304154513259" style="zoom:50%;" />

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20230304155331548.png" alt="image-20230304155331548" style="zoom: 67%;" />

## 重要的编程习惯

定期检查系统中的不变量，比如在CoalesceOrRedistribute函数中，获取了父节点之后可以检查父节点是否上锁了，如果没上锁肯定是有问题的，因为合并或redistribute操作是一定会导致父节点的修改的，所以如果会进行这个操作，那么在之前的findLeafPage函数中就应该已经对父节点上了锁了。

## debug

如果缓冲池太小了，会缓冲池中的page都被pin住，B+树无法从缓冲池fetch到page，出现segfault。

GDB调试技巧：条件断点和backtrace

错误：

1. 在内部节点合并时，要将sibling中所有的kv对都移到当前节点中，此时还需要将这些kv对指向的子节点的父节点指针更新为当前的节点，而不是当前节点的父节点！（不应该是`recipient->GetParentPageId()`！)

   ![image-20230516172954635](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230516172954635.png)

2. 在remove中，进行CoalesceOrRedistribute之后有可能需要将传入的leaf_page删除（将该页从缓冲池和磁盘中都删除），所以对leaf_page的unpin和delete都应该在调用完CoalesceOrRedistribute函数之后进行

   



