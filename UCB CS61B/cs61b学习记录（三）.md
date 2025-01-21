# cs61b学习记录（三）

## Lecture17、B-Trees (2-3, 2-3-4 Trees)

对于二叉搜索树，如果我们随机地插入，可以保证Θ(log N)的时间复杂度，但是如果我们不是随机地插入，二叉搜索树就有可能失去平衡

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210330164439243.png" alt="image-20210330164439243" style="zoom:50%;" />

BST失去平衡的原因是：当我们插入叶的时候，树的高度会增高，所以这个时候我们不能产生新的叶结点，避免高度增加，解决办法只能是**扩充被插入的结点**。

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210330164920960.png" alt="image-20210330164920960" style="zoom:50%;" />

被扩展的树永远是平衡的，因为它的高度不会改变。

然而，当一个节点被扩展地太多时，该节点就退化成了链表

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210330165150713.png" alt="image-20210330165150713" style="zoom:50%;" />

所以我们的解决方案是：为每一个结点设置一个“容量”，当该结点的容量达到上限时，就把其中的一个结点传给父结点

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210330165430361.png" alt="image-20210330165430361" style="zoom:50%;" />

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210330165529217.png" alt="image-20210330165529217" style="zoom:50%;" />

继续添加结点后B-tree的变化

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210330165717327.png" alt="image-20210330165717327" style="zoom:50%;" />

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210330165910029.png" alt="image-20210330165910029" style="zoom:50%;" />

当根结点太满的时候，就将所有结点的高度增加一层

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210330171953063.png" alt="image-20210330171953063" style="zoom:50%;" />

- 如果我们分开根结点，则所有的结点向下推一层，树的高度加一
- 如果我们分开叶结点，则高度不变

这种数据结构的名字叫做B-tree

- 对结点容量为3的B-tree，也叫做2-3-4 tree或2-4tree
- 对结点容量为2的B-tree，也叫做2-3tree

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210330172720678.png" alt="image-20210330172720678" style="zoom:50%;" />

runtime analysis

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210330181207830.png" alt="image-20210330181207830" style="zoom:50%;" />

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210330181540958.png" alt="image-20210330181540958" style="zoom:50%;" />

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210330181942765.png" alt="image-20210330181942765" style="zoom:50%;" />

## Lecture18、Red Black Trees

对于同样元素的BST，会有很多不同的状态

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210401201602964.png" alt="image-20210401201602964" style="zoom:50%;" />

我们可以使用**旋转**操作，让BST保留所有元素的同时，在这些不同的状态之间转换。同样，**旋转操作也可以使一颗非常不平衡的树变成平衡状态**。

The formal definition of rotation is:

- `rotateLeft(G): Let x be the right child of G. Make G the new left child of x.`

  向左旋转也可以看作是让G和它的右结点合并，然后G下降为左节点

  <img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210401204445470.png" alt="image-20210401204445470" style="zoom:50%;" />

- `rotateRight(G): Let x be the left child of G. Make G the new right child of x.`

  向右旋转也可以看作是让G和它的左结点合并，然后G下降为右节点

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210401204634442.png" alt="image-20210401204634442" style="zoom:50%;" />

通过旋转让BST保持平衡：

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210401205148328.png" alt="image-20210401205148328" style="zoom:50%;" />

使用旋转，我们可以增高一棵树，也可以缩短一棵树，也可以让树的高度不变。

所以，从理论上来讲，我们可以随意插入BST，让它变得不平衡，最后使用大量的旋转操作让这颗BST变成平衡状态，但是这样会很丑陋，并且低效。我们需要做的是，在树的**建造过程**中，使用旋转操作**维持**树的平衡。

现在我们已经认识了两种树：

- BST：容易实现，但是难以保持平衡
- B-tree：可以保持平衡，但是难以实现

我们可以结合两者的优点，**建造一种结构上类似B-tree的BST**。所以我们需要做的是：如何将一颗2-3tree转换成一颗BST？

- 对于只有2-nodes的B-tree，不需要转换，因为它和BST完全一样

  <img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210401212152330.png" alt="image-20210401212152330" style="zoom:50%;" />

- 对于有3-nodes的B-tree，我们应该怎么办？

  - 我们的一个做法是：创造一个“胶水”结点，这个结点没有表示任何信息，仅仅表示它的两个子结点实际上是同一个结点

    <img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210401212542706.png" alt="image-20210401212542706" style="zoom:50%;" />

    然而，这是一个非常不优雅的做法，因为这会占用更多的空间，并且代码会变得丑陋。所以，与其使用一个“胶水结点”，我们可以使用一个**“胶水”链接**。我们使用一条链接作为胶水，来连接被认为是同一个结点的两个不同的结点。

  - We choose arbitrarily to make the left element a child of the right one. This results in a **left-leaning** tree. **We show that a link is a glue link by making it red**. Normal links are black. 

    <img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210401212928821.png" alt="image-20210401212928821" style="zoom:50%;" />

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210401221656718.png" alt="image-20210401221656718" style="zoom:50%;" />

我们将树中的链接分为两种类型：

- 红链接将两个2-结点连接起来构成一个3-结点
- 黑链接则是2-3树中的普通链接。

我们将3-结点表示为由一条左斜的红色链接相连的两个2-结点。

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210401224011377.png" alt="image-20210401224011377" style="zoom:50%;" />

这种表示法的一个优点是，我们无需修改就可以直接使用标准二叉查找树的get()方法。对于任意的2-3树，只要对结点进行转换，我们都可以立即派生出一棵对应的二叉查找树。我们将用这种方式表示2-3 树的二叉查找树称为**红黑二叉查找树**。

红黑树的定义是含有红黑链接并满足下列条件的二叉查找树：

- 红链接均为左链接；
- 没有任何一个结点同时和两条红链接相连；
- 该树是完美黑色平衡的，即任意空链接到根结点的路径上的黑链接数量相同。

满足这样定义的红黑树和相应的2-3树是一一对应的。

如果我们将一棵红黑树中的红链接画平，那么所有的空链接到根结点的距离都将是相同的。如果我们将由红链接相连的结点合并，得到的就是一棵2-3树。**无论我们选择用何种方式去定义它们，红黑树都既是二叉查找树，也是2-3树**。因此，**我们能够将两个算法的优点结合起来:二叉查找树中简洁高效的查找方法和2-3树中高效的平衡插入算法**。

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210401224106779.png" alt="image-20210401224106779" style="zoom:50%;" />

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210401224133305.png" alt="image-20210401224133305" style="zoom:50%;" />

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210401224358693.png" alt="image-20210401224358693" style="zoom:50%;" />

 所以如何对红黑树进行插入操作？对红黑树的插入操作应该与BST一致，但是在插入的过程中，还需要进行旋转操作，使红黑树可以和相应的2-3树一一对应。

红黑树的规则如下：

- 插入的颜色：

  在2-3树中，我们插入的永远是叶结点，**插入的链接的颜色永远是红色**！

  <img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210401231714623.png" alt="image-20210401231714623" style="zoom:50%;" />

  

- 当出现红色右链接时，将它旋转为红色左链接。（违反了“红链接均为左链接”的规定）

  <img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210401233604840.png" alt="image-20210401233604840" style="zoom:50%;" />

- 我们**暂时允许**临时的4-结点存在，4-结点表达为有左右两个红链接的BST结点。（注意！这个也是违反规则的，红黑树中不允许4-结点的存在！**我们只是暂时地允许这种表达形式**！）

  <img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210401234354427.png" alt="image-20210401234354427" style="zoom:50%;" />

- 当出现连续的两条红色左链接时，违反了“没有任何一个结点同时和两条红链接相连”的规则，我们**暂时**将它变为一个有左右两条红链接的结点，即一个4-结点。

  <img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210401234527959.png" alt="image-20210401234527959" style="zoom:50%;" />

- 解决有左右两个红链接的BST结点。将子结点的颜色由红变黑，同时将父结点的颜色由黑变红。（当没有父结点时，直接将子结点的颜色由红变黑）

  <img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210401234940825.png" alt="image-20210401234940825" style="zoom:50%;" />

- 上面的flip操作可能会导致额外的违规情况需要处理：

  <img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210401235816540.png" alt="image-20210401235816540" style="zoom:50%;" />

  <img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210401235857621.png" alt="image-20210401235857621" style="zoom:50%;" />

  

## Lecture19、Hashing

BST和B-tree在查找时可以有Θ(log*N*)的复杂度，然而，它们仍然存在局限性：

1. 需要其中的元素是Comparable的，我们只有通过比较才能确定一个新元素在BST中的位置
2. 它们仍然需要Θ(log*N*)的复杂度，我们可以做的更好！

将String类型的值转化为数组的索引：

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210402112217469.png" alt="image-20210402112217469" style="zoom:50%;" />

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210402112429013.png" alt="image-20210402112429013" style="zoom:50%;" />

如果只有小写字母，我们可以像上面一样使用1~26来表示，当其中还包含大写字母和数字的时候，我们就可以使用 **ASCII**码来表示。

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210402113016922.png" alt="image-20210402113016922" style="zoom:50%;" />

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210402113106518.png" alt="image-20210402113106518" style="zoom:50%;" />

```java
public static int asciiToInt(String s) {
	int intRep = 0;
	for (int i = 0; i < s.length(); i += 1) {       	
        intRep = intRep * 126;
        intRep = intRep + s.charAt(i);
	}
	return intRep;
}
```

对于超出英文字母的字符，我们使用Unicode来表示：

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210402113720873.png" alt="image-20210402113720873" style="zoom:50%;" />

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210402113751249.png" alt="image-20210402113751249" style="zoom:50%;" />

理想情况下，不同的键都能转化为不同的索引值。但是实际我们需要面对两个或者多个键都会散列到相同的索引值的情况，因此，散列查找的第二步就是一个处理碰撞冲突的过程。

## Lecture20、 Heaps and PQs

- Priority Queue is an Abstract Data Type that optimizes for **handling minimum or maximum elements**.
- There can be space/memory benefits to using this specialized data structure.
- Implementations for ADTs that we currently know don't give us efficient runtimes for PQ operations.
  - **A binary search tree among the other structures is the most efficient**

since the *binary search tree* has the best runtime for our PQ operations ， Modifying its structure and the constraints, we can further improve the runtime and efficiency of these operations.

We will define our binary min-heap as being **complete** and obeying **min-heap** property:

- Min-heap: Every node is less than or equal to both of its children
- Complete: **Missing items only at the bottom level** (if any), **all nodes are as far left as possible.**

![img](https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/heap-13.2.1.png)

我们对堆的所有的操作都是基于堆的以上两个特性：

- `add`: 为了不破坏堆的完全二叉树的特性，新的结点必须先**暂时地**添加到堆的最右下方的位置。为了保持堆的有序性，新结点还需要上游到合适的位置。
  - Swimming involves swapping nodes if child < parent
- `getSmallest`: Return the root of the heap (This is guaranteed to be the minimum by our *min-heap* property）
- `removeSmallest`: 为了不破坏堆的完全二叉树的特性，被删除的结点必须是右下角的点。所以我们需要先将根结点和右下角的点交换位置，再删除根结点。 为了保持堆的有序性，交换为根结点的点还需要下沉到合适的位置。
  - Sinking involves swapping nodes if parent > child. Swap with the smallest child to preserve *min-heap* property.

### Tree Representation

the representations previously used for trees all store explicit references to who is below us. **These explicit references take the form of pointers to the actual Tree objects that are our children**。

但是，还可以使用**不直接保存子结点的引用**的形式来表达Tree

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210409011859818.png" alt="image-20210409011859818" style="zoom:50%;" />

由于堆是**完全二叉树**，这一点可以保证树中没有空隙，将完全二叉树按照水平顺序放入数组，我们可以发现子结点和父节点的下标存在对应的数学关系。**因此对于完全二叉树，可以不直接保存子结点的引用，而是通过数学关系来计算出结点的位置**。

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210409105426141.png" alt="image-20210409105426141" style="zoom:50%;" />

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210409115357178.png" alt="image-20210409115357178" style="zoom:50%;" />

对于平衡二叉树，如果使用一个指针指向最小结点所在的位置，getSmallest操作也可以实现O(1)的时间复杂度，BST的表现就和Heap相同。