# CMU 15-445实验记录（四）project3

环境问题：

- 群里的测试始终无法编译，最后把后缀改成.cpp才终于可以编译
- make时出现找不到某一个文件，最后把CMakeList中的内容修改了才可以make

## TASK #1 - SYSTEM  CATALOG

catalog是数据库的目录，来记录数据库的元数据，用来回答当前有哪些表存在，这些表对应的名字、id、和索引是什么。

内部维护了DBMS中table id到table metadata，table name到table id的映射，还维护了index id到index metadata，表名到索引名和索引id之间的映射。

table metadata中包含了表的schema，表名，表id，和tableHeap对象

- tableHeap对象就是一个由page组成的双向链表，包含了一个表中的所有的数据，表明数据的存储方式是heap file，也就是说和mysql不同, mysql是聚簇索引的方式, 所有数据存储在B+树叶子节点上. **而在这里, b+树以及hash_table都是只作为索引来使用，叶子节点存储的是record id，由record id在table heap中查找对应的tuple**

- schema对象中包含了一个Column对象的vector

  - Column对象是某一列的schema，不包括此列Column的数据，只有列名和列的数据的类型以及此列数据在tuple中的偏移量

  所以由这个Column对象的vector就可以组成整张表的Schema

所以在catalog中，tables_拥有tableMetadata，tableMetadata拥有tableHeap

createIndex方法要将表中的所有tuple插入索引中，使用tableHeap的迭代器遍历一次table，每次调用tuple的KeyFromTuple方法获取Key，调用tuple的GetRid()获取对应RID

## TASK #2 - EXECUTORS

以下实际上实现的是需求驱动的流式模型（火山模型）执行器

- 需求驱动即从算子树的顶端将数据往上拉，也就是像此lab一样，由父算子调用子算子的next方法，子算子的next方法中再调用下一级算子的next方法，获取tuple
- 除此之外还有生产者驱动的方式，将数据从算子树的底层往上推

`value`就是最小单位，值，就是对C++的内置类型进行了包装，

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20230226194134548.png" alt="image-20230226194134548" style="zoom:50%;" />

`tuple`相当于一行，存多个值，传入`vector<Value>`和对应的schema可以将一串value对象转化为tuple

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20230226194523761.png" alt="image-20230226194523761" style="zoom:50%;" />

`TableHeap`是一张表，提供了迭代器去访问，迭代器按行（tuple）遍历

`column`是列的名字（抬头），包含了这一列数据的信息：列名，列数据的类型，在tuple中offset，创建这一列的expression

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20230226195655885.png" alt="image-20230226195655885" style="zoom:50%;" />

`schema`则存了一串column，表示这张表有哪些列

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20230226200042162.png" alt="image-20230226200042162" style="zoom: 50%;" />

### executor_engine

`executor_engine.h`是执行器引擎，它的Execute方法是启动执行一个执行计划的入口，传入一个表示查询计划的plan节点（`AbstractPlanNode`），将结果返回result_set

因为目前的bustub不支持sql，所以我们的实现直接在手写的查询计划上执行。即每个executor都有对应的一个query plan node，plan node告诉executor如何执行

![image-20230222174508631](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230222174508631.png)

**首先调用ExecutorFactory来根据传入的查询计划（`AbstractPlanNode`）创建一个Executor算子（`AbstractExecutor`）**，它会根据plan的不同type来创建不同的Executor, 对于简单的plan如SeqScan, 他会直接创建

```cpp
    case PlanType::SeqScan: {
      return std::make_unique<SeqScanExecutor>(exec_ctx, dynamic_cast<const SeqScanPlanNode *>(plan));
    }
```

对于存在子查询的非底层的plan node，如Limit或Insert等, 则会先根据child_plan生成child_executor，再将child_executor生成该plan node的executor

```cpp
    case PlanType::Limit: {
      auto limit_plan = dynamic_cast<const LimitPlanNode *>(plan);
      auto child_executor = ExecutorFactory::CreateExecutor(exec_ctx, limit_plan->GetChildPlan());
      return std::make_unique<LimitExecutor>(exec_ctx, limit_plan, std::move(child_executor));
    }
```

有的plan node有两个子节点，如NestedLoopJoin

```cpp
    case PlanType::NestedLoopJoin: {
      auto nested_loop_join_plan = dynamic_cast<const NestedLoopJoinPlanNode *>(plan);
      auto left = ExecutorFactory::CreateExecutor(exec_ctx, nested_loop_join_plan->GetLeftPlan());
      auto right = ExecutorFactory::CreateExecutor(exec_ctx, nested_loop_join_plan->GetRightPlan());
      return std::make_unique<NestedLoopJoinExecutor>(exec_ctx, nested_loop_join_plan, std::move(left), std::move(right));
    }
```

**所以plan和executor都有child结构，plan node对象和executor是一对一的关系**，根据plan node的结构从子节点到父节点（也就是plan tree）生成对应的executor对象：

- ExecutorFactory先将传入的plan node拆分，根据最底层的plan node生成最底层的executor算子，再用这个executor和高层的plan node生成父executor算子，以此类推，生成最高层的executor算子返回。**executor内部嵌套了子executor算子，形成了一个火山结构**（火山模型将关系代数中每一种操作抽象为一个 Operator，将整个 SQL 构建成一个 Operator 树，从根节点到叶子结点自上而下地递归调用 next() 函数）

创建完executor后，Execute方法中还有一个while循环，不停从刚才返回的executor（**也就是火山模型中最高的executor算子**）中调用next方法，获取下一行数据并添加到result_set。

- 在每个非底层的executor算子的next方法中还会循环调用子executor的next方法，获取了子executor返回的tuple后对其进行处理后再返回给父executor。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20230222174943209.png" alt="image-20230222174943209" style="zoom:50%;" />

所以Execute方法实际上只是调用了最高级的executor算子的next方法，由executor算子**递归调用**更低级的executor算子的next方法

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20230222175429821.png" alt="image-20230222175429821" style="zoom:50%;" />

最高级的算子返回tuple前要根据plan node中的output_schema将tuple进行projection，筛选出想要的column后形成新的tuple才能返回

### AbstractPlanNode

这是所有PlanNode的父类。对应的有一个枚举类PlanType，表示所有可能的PlanNode类型。

```c
enum class PlanType { SeqScan, IndexScan, Insert, Update, Delete, Aggregation, Limit, NestedLoopJoin, NestedIndexJoin };
```

AbstractPlanNode只有两个成员变量，

- **一个是output_shcema**,在Next返回tuple（如果需要返回tuple）时可以根据output_schema选择输出tuple的哪几个column（projection）。
- **另一个是vector children_**, 里面有所有children的常量指针

### IndexMetadata

属性：

- `vector<uint32_t> key_attrs_`，表示tuple的schema中哪些列需要被提取出来作为索引中的key，一个索引中的key可以由tuple中的多个字段构成（即联合索引）

  - 联合索引是指对tuple中的多个字段建立索引，假设，我们对(a,b)字段建立索引，会**按照a来进行排序，在a相等的情况下才会按照b来排序**。所以**a是全局有序的，而b只是全局无序，局部有序的状态**。

    - 直接执行`b = 2`这种查询条件没有办法利用索引，
    - 局部来看，当a的值确定时，b是有序的，所以执行`a = 1 and b = 2`是能用到索引的
    - 执行`a > 1 and b = 2`时，a的值此时是一个范围，不是固定的，在这个范围内b值不是有序的，因此b字段用不上索引。

    以上就是最左匹配原则，**当联合索引左边的字段是范围查询（包括缺失的情况）时就无法使用索引**。

    **建立联合索引时要将区分度高的字段放在前面，区分度低的字段放在后面**。

- `Schema *key_schema_`：用来索引的key的schema

传入tuple的schema和`key_attrs`创建一个index，调用CopySchema按照`key_attrs_`，将tuple schema中的特定列选出来作为index的`key_schema`

```cpp
  IndexMetadata(std::string index_name, std::string table_name, const Schema *tuple_schema,
                std::vector<uint32_t> key_attrs)
      : name_(std::move(index_name)), table_name_(std::move(table_name)), key_attrs_(std::move(key_attrs)) {
    key_schema_ = Schema::CopySchema(tuple_schema, key_attrs_);
  }
  static Schema *CopySchema(const Schema *from, const std::vector<uint32_t> &attrs) {
    std::vector<Column> cols;
    cols.reserve(attrs.size());
    for (const auto i : attrs) {
      cols.emplace_back(from->columns_[i]);
    }
    return new Schema{cols};
  }
```

### AbstractValueExpression

`ComparisonExpression`：表示两个expression进行比较，表示谓词和having。predicate和having的区别就在于：predicate的子表达式是`ColumnValueExpression`，而having的子表达式是`AggregateValueExpression`

-   `std::vector<const AbstractExpression *> children_`：存储进行比较的左右两个expression，进行比较的子expression通常是`ColumnValueExpression`（表示table.member，即**某个表中**的Column）和`ConstantValueExpression`（表示一个常数）

-  `ComparisonType comp_type_`：比较的类型

- Evaluate函数：适用于predicate对一条tuple的两个value进行比较，即predicate后进行比较的两个col表达式在同一个表中的情况。

  如 `select * from test_1 where test_1.A = test_1.B`，`where test_1.A = test_1.B`语句是`ComparisonExpression`，两个子表达式是 `test_1.A`和 `test_1.B`，类型是`ColumnValueExpression`，传入的tuple就是从子算子中获取的test_1的tuple，schema是test_1的shema。通过`ColumnValueExpression`的`Evaluate`方法获取`test_1.A`和`test_1.B`在该tuple中对应的Value，再将这两个Value进行比较。

  ```cpp
    Value Evaluate(const Tuple *tuple, const Schema *schema) const override {
      Value lhs = GetChildAt(0)->Evaluate(tuple, schema);
      Value rhs = GetChildAt(1)->Evaluate(tuple, schema);
      return ValueFactory::GetBooleanValue(PerformComparison(lhs, rhs));
    }
  ```

- `EvaluateJoin`函数：判断来自不同表的两个tuple能否join，即predicate后进行比较的两个col表达式不在同一个表中的情

  如 `select * from test_1,test_2 where test_1.id = test_2.id`，`where test_1.id = test_2.id`语句就是`ComparisonExpression`，两个子表达式分别是 `test_1.id`和 `test_2.id`，类型是`ColumnValueExpression`。传入的left_tuple是从join左边的子算子中获取的test_1的tuple，left_schema是test_1的schema，right_tuple是从join右边的子算子中获取的test_2的tuple，right_schema是test_2的schema。通过`ColumnValueExpression`的`EvaluateJoin`方法获取`test_1.id`和`test_2.id`在各自的tuple中对应的value，再将这两个Value进行比较

  ```cpp
    Value EvaluateJoin(const Tuple *left_tuple, const Schema *left_schema, const Tuple *right_tuple,
                       const Schema *right_schema) const override {
      Value lhs = GetChildAt(0)->EvaluateJoin(left_tuple, left_schema, right_tuple, right_schema);
      Value rhs = GetChildAt(1)->EvaluateJoin(left_tuple, left_schema, right_tuple, right_schema);
      return ValueFactory::GetBooleanValue(PerformComparison(lhs, rhs));
    }
  ```

- `EvaluateAggregate`函数：判断AggregationHashTable的某一组的聚集结果是否满足having表达式

  如 `select count(A),B from test_1 group by B having count(A) > B`，其中`having count(A) > B`语句就是`ComparisonExpression`，两个子表达式分别是count(A) 和B，类型是`AggregateValueExpression`，传入的group_bys的内容是某tuple中所有被分组字段的value，aggregates的内容是某tuple中所有被聚集字段聚集的结果。调用`AggregateValueExpression`的`EvaluateAggregate`方法，从中获得`count(A)`和`B`的value

  ```cpp
    Value EvaluateAggregate(const std::vector<Value> &group_bys, const std::vector<Value> &aggregates) const override {
      Value lhs = GetChildAt(0)->EvaluateAggregate(group_bys, aggregates);
      Value rhs = GetChildAt(1)->EvaluateAggregate(group_bys, aggregates);
      return ValueFactory::GetBooleanValue(PerformComparison(lhs, rhs));
    }
  ```

`ColumnValueExpression`：**表示table.member**，即某个表中的Column表达式

- `tuple_idx_`表示该tuple在join的左边还是右边
- `col_idx_`表示member在table中的相对位置

- Evaluate函数：传入tuple和该tuple对应的schema，获取与该expression相对应的col在此tuple中的value

  `select table.A from table`，其中table.A就是ColumnValueExpression，evaluate函数获取table.A在此tuple中的value 

  ```cpp
  Value Evaluate(const Tuple *tuple, const Schema *schema) const override { 
    	return tuple->GetValue(schema, col_idx_); 
  } 
  ```

- EvaluateJoin函数：传入参加join的两个tuple和schema，获取本表达式所在的tuple的相应Value

   `select * from test_1,test_2 where test_1.id = test_2.id`，其中table1.id和table2.id就是ColumnValueExpression，传入左右两边的tuple，获取`table1.id`和`table2.id`在各自tuple中的value

  ```cpp
    Value EvaluateJoin(const Tuple *left_tuple, const Schema *left_schema, const Tuple *right_tuple,const Schema *right_schema) const override {
      return tuple_idx_ == 0 ? left_tuple->GetValue(left_schema, col_idx_)
                             : right_tuple->GetValue(right_schema, col_idx_);
    }
  ```

`AggregateValueExpression`：可以用来表示两种表达式，一种是aggregate，一种是group by

- `is_group_by_term`:标记此表达式是否是group by表达式

- `term_idx`:如果是aggregate表达式，则表示此条表达式在所有aggregate表达式中的index；如果是group by表达式，则表示此条表达式在所有group by的表达式（col）中的index。aggregate表达式和group by表达式分开定位

- `EvaluateAggregate`函数： `select count(A),B from test_1 group by B having count(A) > B`，`count(A)` 和`B`的类型就是`AggregateValueExpression`，传入的group_bys的内容是某tuple中所有被分组字段的value，aggregates的内容是某tuple中所有被聚集字段聚集的结果，此函数用来从这些分组和聚集的结果中获取本表达式对应的value

  ```cpp
    Value EvaluateAggregate(const std::vector<Value> &group_bys, const std::vector<Value> &aggregates) const override {
      return is_group_by_term_ ? group_bys[term_idx_] : aggregates[term_idx_];
    }
  ```

  



### SEQUENTIAL SCAN（全表扫描）

SeqScanPlanNode的成员变量：

- `predicate_`：即where语句后的谓词。并不是所有Plan node都需要predicate，只有select语句（包括后面的IndexScan）才需要predicate，所以predicate并不作为基类AbstractPlanNode中的成员变量
- `table_oid_`：所遍历的表的id，只有最底层的算子才需要`table_oid_`来指定对哪个表进行操作

next方法并不是简单地遍历整张表，然后把表中的每一个tuple直接返回。这个seq_scan算子实际上是select算子，也就是要根据传入的查询计划中的predicate来对表中的tuple进行筛选，满足predicate的tuple才能被返回。

并且不能直接返回原表的tuple，**实际中执行时并不像火山模型一样select算子和projection算子分离的，而是select完后先projection再返回给父算子**。

- `AbstractPlanNode`中有output_schema，指定了返回的tuple的schema，要根据这个schema从原表的tuple中选出对应的列的value组成新的tuple，再返回这个tuple。

  首先遍历output_schema中的每一个column，根据这个column的名字获取它在**原来的**tuple的schema中的位置，再调用原tuple的GetValue()方法获取指定位置的Value，再将value放入vector中，最后所有value组成的vector就是经过projection的tuple。**注意区分原tuple的schema和`output_schema`！**

  - GetValue方法需要传入该tuple的schema和想要获取的value在该tuple中的index，schema用来解析该tuple
  - **构造tuple对象时要传入一个value数组（vector），以及该tuple的schema**

```cpp
Tuple SeqScanExecutor::generateTuple(Tuple *origin_tuple){
  std::vector<Value>res;
  const Schema *schema_of_output_tuple = GetOutputSchema();
  const Schema *schema_of_origin_tuple = &table_metadata->schema_;
  for(const Column &output_col : schema_of_output_tuple->GetColumns()){
    std::string name_of_output_col = output_col.GetName();
    uint32_t index_of_output_col_in_origin_tuple = schema_of_origin_tuple->GetColIdx(name_of_output_col);
    Value val = origin_tuple->GetValue(schema_of_origin_tuple,index_of_output_col_in_origin_tuple);
    res.push_back(val);
    // equivalent to the below code
    // Value val = col.GetExpr()->Evaluate(origin_tuple,&table_metadata->schema_);
    // res.push_back(val);
  } 
  return {res,schema_of_output_tuple};
```



### INDEX SCANS（索引扫描）

`IndexScanPlanNode`的成员变量：

- `predicate_`：即where语句后的谓词。
- `index_oid_`：所遍历的索引的id

遍历一个指定的index（index由plan node指定）来获取tuple的record id，然后再用这些record id在该index对应的表中获取相应的tuple（使用catalog获取对应的table_heap），根据predicate（同样包含在plan node中），不满足predicate的tuple直接跳过，满足的返回。一次返回一条tuple，与seq_scan一样，返回前进行projection

### INSERT

将tuple插入到一个表中并且更新index，有两种类型的插入：

- 第一种是要插入的值就在plan node中，根据plan node中的rawValue构造tuple插入表中，rawValue是一个value的二维数组（`vector<vector<Value>>`），一个`vector<Value>`就是一个tuple

- 第二种是从子算子中（child executor）获取要插入的值，比如InsertPlanNode的子算子可以是`SeqScanPlanNode` or `IndexScanPlanNode`，从而将一个表的内容复制到另一个表

  - 例如：`INSERT INTO empty_table2 SELECT colA, colB FROM test_1 WHERE colA > 500`需要先执行后面的select语句再进行insert。

  在init中要记得调用子算子的init，在next中先调用子算子的next，将子算子返回的tuple插入到表中。

以上在插入时还要同步修改该table对应的所有index（一个table可能有好几个index）

### UPDATE

`UPDATE empty_table2 SET colA = colA+10 WHERE colA < 50`

`UpdatePlanNode`的成员变量：

- `table_oid_`：虽然update算子不是最底层的算子，但是因为update算子获取下层的select算子筛选出来的tuple后，还是需要直接对表进行操作，所以需要在update算子内部维护一个table_id
- `update_attrs_`：是一个从column index到UpdateInfo的数组，column index是要修改的列号，UpdateInfo是一个结构体，包含了对该列进行修改的信息，如修改的类型（是对该列的值进行Add还是重置为某个值）和修改的数值。

update算子修改给定表中的某一tuple，并且更新该表的索引。update的子算子 `SeqScanPlanNode` or `IndexScanPlanNode` 会给update算子提供需要它修改的tuple和rid。

- next方法中先调用child的next方法获取满足谓词的tuple，调用GenerateUpdatedTuple传入原来的tuple，对其中的某一列进行修改，生成更新后的tuple，然后调用table_heap的updateTuple在表中更新tuple，再在该表的所有index中删除原来的tuple，再插入新的tuple，最后返回tuple的rid即可。
  - GenerateUpdatedTuple函数已经实现好了，将传入的tuple根据plan node指定的更新方式（add/set）进行更新

### DELETE

delete算子同样也是从子算子 `SeqScanPlanNode` or `IndexScanPlanNode` 中（调用子算子的next方法）获取目标tuple，再调用table_heap的MarkDelete将该tuple删除

- **这里MarkDelete的意思是使tuple invisable，并不真的删除它**。只有在事务提交的时候才真的删除。这样如果事务还没提交就abort了，回滚时只需要将mark的标志撤销就好。

### NESTED LOOP JOIN（嵌套循环join）

此算子有左右两个child_executor，分别遍历左右两张表，对内表中的每个tuple都要遍历外表中的所有tuple，判断它们是否匹配（由planNode中的predicate指定），将匹配的tuple合并成一个新的tuple返回。

- 最坏的情况（缓冲池不够大）下对于A中每个tuple，都需要将B中所有page都读入一遍，再加上A中所有page需要读入，**这种算法的io效率是很低的。对此的改进是Block Nested loop join**

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20230224203501639.png" alt="image-20230224203501639" style="zoom:50%;" />

init函数需要调用left和right child的init方法，next方法中先调用left_child的next方法，再调用right_child的next方法，一直找到第一个符合predicate的left tuple和right tuple的组合。在找到下一个匹配的tuple对之前，left_child和right_child都有可能会多次调用next方法。

- 判断tuple对是否满足predicate需要使用`predicate_`的`EvaluateJoin`方法，然后将返回的Value调用`GetAs<bool>`

如果左右两个tuple符合条件，需要将两个tuple合并。这里的子算子传入的两个tuple已经是根据自身的output_schema进行projection之后的结果，所以合并不需要projection，只需要把两个tuple的values都取出来放到一个vector中，然后**使用这个vector和Nested Loop Join自身的outputSchema构造对应的tuple**即可

### INDEX NESTED LOOP JOIN（索引join）

这个和nested loop join的区别在于，上一个join是遍历两个table求join，而INDEX NESTED LOOP JOIN则对内表建立了索引，遍历外表的每一个tuple，调用tuple的KeyFromTuple方法，传入index中key的schema，获取该tuple的index probe key，**再使用内表的索引搜索外表的index probe key**。搜索到匹配的tuple的rid，再用rid去table_heap中获取对应的tuple，再把两个tuple合并后返回。

这显然会快于上一种join（**前提是对内表参与join的column已经建了索引，此时通过索引查比遍历全表明显快得多**）

- 获取tuple的index probe key：

  ```cpp
  Tuple Tuple::KeyFromTuple(const Schema &schema, const Schema &key_schema, const std::vector<uint32_t> &key_attrs) {
    std::vector<Value> values;
    values.reserve(key_attrs.size());
    for (auto idx : key_attrs) {
      values.emplace_back(this->GetValue(&schema, idx));
    }
    return Tuple(values, &key_schema);
  }
  ```

  先用tuple的schema解析tuple，再根据`key_attrs`内部的索引，将tuple中的key value获取出来，再将这些key value加上key_schema构成一个新的tuple返回，这个tuple就是index probe key

### AGGREGATION

#### AggregationPlanNode

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20230227233347661.png" alt="image-20230227233347661" style="zoom:50%;" />

```cpp
TEST_F(GradingExecutorTest, DISABLED_SimpleAggregationTest) {
  // SELECT COUNT(colA), SUM(colA), min(colA), max(colA) from test_1;
  std::unique_ptr<AbstractPlanNode> scan_plan;
  const Schema *scan_schema;
  {
    // 创建output_schema的流程：
    //      1.根据期望输出的列，创建对应的ColumnExpression对象
    //      2.根据ColumnExpression对象创建相应的Column对象
    //      3.使用vector<Column>创建Schema对象
    auto table_info = GetExecutorContext()->GetCatalog()->GetTable("test_1");
    auto &schema = table_info->schema_;
    // 创建colA的ColumnValueExpression对象，此对象的主要作用就是用来创建Column对象
    auto colA = MakeColumnValueExpression(schema, 0, "colA");
    // 先使用ColumnValueExpression对象创建对应的Column对象，将Column对象构成Vector，再用vector<Column>创建对应的Schema
    scan_schema = MakeOutputSchema({{"colA", colA}});
    // 使用 期望输出的schema，谓词predicate，和该查询语句操作的表id 创建一个查询计划plan node 
    scan_plan = std::make_unique<SeqScanPlanNode>(scan_schema, nullptr, table_info->oid_);
  }

  std::unique_ptr<AbstractPlanNode> agg_plan;
  const Schema *agg_schema;
  {
    // 每个子查询都相当于输出了一个表，父查询对这个表的哪些列进行操作，都要对这些列重新生成Expression，还有Vector<Column>和schema
    // 上下两种写法的唯一区别在于：
    //    * 上面生成的colA的col_idx是colA相对于原表的schema的位置，即select test_1.colA form test_1
    //    * 下面的colA的col_idx是colA相对于select的输出表的schema的位置,select temp.colA from temp，temp是上面的select输出的表

    // auto colA = MakeColumnValueExpression(schema, 0, "colA"); 
    const AbstractExpression *colA = MakeColumnValueExpression(*scan_schema, 0, "colA");// 对select输出的colA生成expression
    // AggregateValueExpression可以用来表示两种表达式，一种是aggregate，一种是group by
    // 所以AggregateValueExpression的主要成员是：
    //    * is_group_by_term:标记此表达式是否是group by表达式
    //    * term_idx:如果是aggregate表达式，则表示此条表达式在所有aggregate表达式中的index
    //               如果是group by表达式，则表示此条表达式在所有group by的表达式（col）中的index
    //               aggregate表达式和group by表达式分开定位
    const AbstractExpression *countA = MakeAggregateValueExpression(false, 0);
    const AbstractExpression *sumA = MakeAggregateValueExpression(false, 1);
    const AbstractExpression *minA = MakeAggregateValueExpression(false, 2);
    const AbstractExpression *maxA = MakeAggregateValueExpression(false, 3);

    // 用期望输出的ColumnEXpression创建aggregate的输出schema
    agg_schema = MakeOutputSchema({{"countA", countA}, {"sumA", sumA}, {"minA", minA}, {"maxA", maxA}});
    agg_plan = std::make_unique<AggregationPlanNode>(
        agg_schema, scan_plan.get(), nullptr, std::vector<const AbstractExpression *>{},
        std::vector<const AbstractExpression *>{colA, colA, colA, colA},
        std::vector<AggregationType>{AggregationType::CountAggregate, AggregationType::SumAggregate,
                                     AggregationType::MinAggregate, AggregationType::MaxAggregate});
  }
  std::vector<Tuple> result_set;
  GetExecutionEngine()->Execute(agg_plan.get(), &result_set, GetTxn(), GetExecutorContext());
```

#### AggregationHashTable

```cpp
/**
 * A simplified hash table that has all the necessary functionality for aggregations.
 */
class SimpleAggregationHashTable {
 public:
  /**
   * Create a new simplified aggregation hash table.
   * @param agg_exprs the aggregation expressions
   * @param agg_types the types of aggregations
   */
  SimpleAggregationHashTable(const std::vector<const AbstractExpression *> &agg_exprs,
                             const std::vector<AggregationType> &agg_types)
      : agg_exprs_{agg_exprs}, agg_types_{agg_types} {}

  /** @return the initial aggregrate value for this aggregation executor 
   *  维护一个vector<value>记录某一组当前累积的aggregate的结果，每一个Value对应不同类型的aggregate
   *  count和sum aggregate对应的初始值应该是0，而Min的初始值应该是最大的int，Max的初始值应该是最小的int（打擂台）
  */
  AggregateValue GenerateInitialAggregateValue() {
    std::vector<Value> values;
    for (const auto &agg_type : agg_types_) {
      switch (agg_type) {
        case AggregationType::CountAggregate:
          // Count starts at zero.
          values.emplace_back(ValueFactory::GetIntegerValue(0));
          break;
        case AggregationType::SumAggregate:
          // Sum starts at zero.
          values.emplace_back(ValueFactory::GetIntegerValue(0));
          break;
        case AggregationType::MinAggregate:
          // Min starts at INT_MAX.
          values.emplace_back(ValueFactory::GetIntegerValue(BUSTUB_INT32_MAX));
          break;
        case AggregationType::MaxAggregate:
          // Max starts at INT_MIN.
          values.emplace_back(ValueFactory::GetIntegerValue(BUSTUB_INT32_MIN));
          break;
      }
    }
    return {values};
  }

  /** Combines the input into the aggregation result. 
   *  将输入的另一个tuple的aggregateValue（也就是该tuple中所有需要aggregate的Value）聚集到已经Aggregate的结果中
  */
  void CombineAggregateValues(AggregateValue *result, const AggregateValue &input) {
    // 遍历每一个需要aggregate的col，将输入的tuple中的相应列的value聚集到该列已经aggregate的结果中
    // 调用Value的相应方法进行运算，将此次aggregate后结果再存回result中，更新result
    for (uint32_t i = 0; i < agg_exprs_.size(); i++) {
      switch (agg_types_[i]) {
        case AggregationType::CountAggregate:
          // Count increases by one.
          result->aggregates_[i] = result->aggregates_[i].Add(ValueFactory::GetIntegerValue(1));
          break;
        case AggregationType::SumAggregate:
          // Sum increases by addition.
          result->aggregates_[i] = result->aggregates_[i].Add(input.aggregates_[i]);
          break;
        case AggregationType::MinAggregate:
          // Min is just the min.
          result->aggregates_[i] = result->aggregates_[i].Min(input.aggregates_[i]);
          break;
        case AggregationType::MaxAggregate:
          // Max is just the max.
          result->aggregates_[i] = result->aggregates_[i].Max(input.aggregates_[i]);
          break;
      }
    }
  }

  /**
   * Inserts a value into the hash table and then combines it with the current aggregation.
   * @param agg_key AggregateKey是某一个tuple中需要group by的value组成的vector，
   * @param agg_val AggregateValue是某一个tuple中需要被aggregate的value组成的vector
   * 这个hash表使用tuple中需要被group by的value作为key，所以相同的key就是同一个组，不同的key就是不同的组
   * 所以每个组在这个hash表中只会有一个对应的表项，对应的value就是这个组当前累积aggregate的结果
   * 如果一个tuple的aggregateKey在此hash表中不存在，说明出现了一个新的分组，那么就加入此aggregateKey，创建一个新的组
   * 如果存在，那么就把该tuple的aggregateValue和该aggregateKey当前累积的aggregate结果进行聚集
   */
  void InsertCombine(const AggregateKey &agg_key, const AggregateValue &agg_val) {
    if (ht.count(agg_key) == 0) {
      ht.insert({agg_key, GenerateInitialAggregateValue()});
    }
    CombineAggregateValues(&ht[agg_key], agg_val);
  }

  /**
   * An iterator through the simplified aggregation hash table.
   */
  class Iterator {
   public:
    /** Creates an iterator for the aggregate map. */
    explicit Iterator(std::unordered_map<AggregateKey, AggregateValue>::const_iterator iter) : iter_(iter) {}

    /** @return the key of the iterator */
    const AggregateKey &Key() { return iter_->first; }

    /** @return the value of the iterator */
    const AggregateValue &Val() { return iter_->second; }

    /** @return the iterator before it is incremented */
    Iterator &operator++() {
      ++iter_;
      return *this;
    }

    /** @return true if both iterators are identical */
    bool operator==(const Iterator &other) { return this->iter_ == other.iter_; }

    /** @return true if both iterators are different */
    bool operator!=(const Iterator &other) { return this->iter_ != other.iter_; }

   private:
    /** Aggregates map. */
    std::unordered_map<AggregateKey, AggregateValue>::const_iterator iter_;
  };

  /** @return iterator to the start of the hash table */
  Iterator Begin() { return Iterator{ht.cbegin()}; }

  /** @return iterator to the end of the hash table */
  Iterator End() { return Iterator{ht.cend()}; }

 private:
  /** The hash table is just a map from aggregate keys to aggregate values. 
   * 这个hash表使用tuple中需要被group by的value作为key，所以相同的key就是同一个组，不同的key就是不同的组
   * 每个组在这个hash表中只会有一个对应的表项，对应的value就是这个组当前累积aggregate的结果
  */
  std::unordered_map<AggregateKey, AggregateValue> ht{};
  /** The aggregate expressions that we have. 
   *  需要aggregate的col
  */
  const std::vector<const AbstractExpression *> &agg_exprs_;
  /** The types of aggregations that we have. 
   *  需要aggregate的col的聚集类型，与上面的agg_exprs一一对应
  */
  const std::vector<AggregationType> &agg_types_;
};
```

`aggregateExecutor`获取子查询的tuple中需要aggregate的value和需要group by的value，分别组成AggregateValue对象和AggregateKey对象，

- tuple中需要group by的value是AggregationHashTable中的key
- tuple中需要aggregate的value是AggregationHashTable中的value

这样就可以实现对不同分组的aggregate

```cpp
  /** @return the tuple as an AggregateKey 
   *  GetGroupBys返回的是被group by的colExpres，这些colExpres是相对于group by(或者说是aggregate）的子查询输出的schema而言的
   *  所以此方法传入一个子查询返回的tuple，获取该tuple中需要被group by的Value
   *  AggregateKey对象就是由某一个tuple中的需要被group by的Value对象组成的vector
  */
  AggregateKey MakeKey(const Tuple *tuple) {
    std::vector<Value> keys;
    for (const auto &expr : plan_->GetGroupBys()) {
      keys.emplace_back(expr->Evaluate(tuple, child_->GetOutputSchema()));
    }
    return {keys};
  }

  /** @return the tuple as an AggregateValue 
   *  GetAggregates方法返回的是被aggregate的colExpres，这些colExpress也是相对于group by(或者说是aggregate)的子查询输出的schema而言的
   *  所以此方法传入一个子查询返回的tuple，获取该tuple中需要被aggregate的Value
   *  AggregateValue对象就是由某一个tuple中的需要被aggregate的Value对象组成的vector
  */
  AggregateValue MakeVal(const Tuple *tuple) {
    std::vector<Value> vals;
    for (const auto &expr : plan_->GetAggregates()) {
      vals.emplace_back(expr->Evaluate(tuple, child_->GetOutputSchema()));
    }
    return {vals};
  }
```

aggregate需要直接遍历整个子查询输出的表，一边遍历一边aggregate，将每一个分组aggregate的结果都作为一个表项存在AggregationHashTable中。需要等整个子查询的表遍历完后，next才能返回（不像之前的算子那样，获取一个子查询的tuple就可以返回）。最后next函数每次返回一个组aggregate的结果，也就是使用HashTable的迭代器，每次返回HashTable的一个表项

