# Princeton Algorithms学习记录：Graph

## topological sort

解决优先级限制下的调度问题。给定一组需要完成的任务，以及一组关于任务完成的先后次序的优先级限制。在满足限制条件的前提下应该如何安排并完成所有任务?
对于任意一个这样的问题，我们都可以马上画出一张有向图， 其中顶点对应任务，有向边对应优先级顺序。

**拓扑排序**：给定一副有向图，将所有的顶点排序，使得所有的边都从前面的顶点指向后面的顶点

有向无环图（DAG）是一副不含有环的有向图，**只有有向无环图才能进行拓扑排序，有环图也可以进行拓扑排序的算法过程（即找到一个图的reverse post-order），只不过不是拓扑排序**。

**查找一个有向图是否有环**：使用DFS，标记所有经过的点，退出该点时取消标记。当某一个点被重复标记时，就说明图中有环。（使用由顶点id索引的数组，经过一个点就标为true）

优先级限制下的调度问题等价于计算有向无环图中的所有顶点的拓扑排序

**拓扑排序**过程：

1. 创建标记数组和栈
2. 从图中任意没有标记过的点开始DFS
3. DFS中将经过的点都标记，后退时将点加入栈，不取消标记
4. DFS结束，回到第二步，直到图中所有的点都走过了

```java
public class DepthFirstOrder {
    private boolean[] marked;
    private Stack<Integer> reversePost;

    public DepthFirstOrder(Digraph G) {
        marked = new boolean[G.V()];
        reversePost = new Stack<>();
        //拓扑排序可能需要多次dfs，以所有没有标记的点为起点进行dfs
        for (int v = 0; v < G.V(); v++)
            if (!marked[v]) dfs(G, v);
    }

    public void dfs(Digraph G, int v) {
        marked[v] = true;
        for (int w : G.adj(v))
            dfs(G, w);
        //退出某一点的时候，将该点加入栈中
        reversePost.push(v);
        //拓扑排序每个点只能经过一次，所以不能取消标记
    }

    public Iterable<Integer> reversePost() {
        return reversePost;
    }
}
```

## Strongly-connected components

在一幅无向图中， 如果有一条路径连接顶点v和W，则它们就是**连通**的，既可以由这条路径从w到达v,也可以从v到达w。相反，在一幅有向图中，如果从顶点v有一条有向路径到达w,则顶点w是从顶点v可达的，但顶点v不一定是从顶点w可达的。

在有**向图中，如果两个顶点v和w是互相可达的，则称它们为强连通的**。也就是说，既存在一条从v到w的有向路径，也存在一条从w到v的有向路径。如果一幅有向图中的任意两个顶点都是强连通的，则称这幅有向图也是强连通的。

强连通性将所有顶点分为了一些平等的部分，每个部分都是由相互均为强连通的顶点的最大子集组成的。我
们将这些子集称为**强连通分量**，一个含有V个顶点的有向图含有1 ~ V个强连通分量。一个强连通图只含有一个强连通分量，而一个有向无环图中则含有V个强连通分量。需要注意的是强连通分量的定义是基于顶点的，而非边。

计算无向图的连通分量：

1. 对每个没有标记的点进行dfs
2. **同一个dfs中，经过的点就是同一个连通分量**
3. 判断任意两点是否属于同一个连通分量（是否连通），只需要判断数组的值是否相同

```java
public class connectedComponent {
    private boolean[]marked;
    private int[]id;
    private int count;
    public connectedComponent(Graph G){
        marked=new boolean[G.V()];
        id=new int[G.V()];
        for (int s=0;s<G.V();s++){
            if (marked[s]!=true){
                dfs(G,s);
                count++;
            }
        }
    }
    private void dfs(Graph G,int v){
        marked[v]=true;
        id[v]=count;
        for (int w:G.adj(v))
            if (marked[w]!=true)
                dfs(G,w);
        
    }
    public boolean connected(int v,int w){
        return id[v]==id[w];
    }
    public int id(int v){
        return id[v];
    }
    public int count(){
        return count;
    }
}

```

计算有向图的强连通分量：

### Kosaraju-Sharir algorithm

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210523110946005.png" alt="image-20210523110946005" style="zoom:67%;" />

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210523111018264.png" alt="image-20210523111018264" style="zoom: 67%;" />

 

```java
public class KosarajuSharirSCC {
    private boolean[]marked;
    private int[]id;
    private int count;
    public KosarajuSharirSCC(Digraph G){
        marked=new boolean[G.V()];
        id=new int[G.V()];
        //先对G的反向图求拓扑排序的点的顺序
        DepthFirstOrder dfs = new DepthFirstOrder(G.reverse());
        //再按照reversePost的顺序进行dfs，在同一个dfs中的点就是同一个强连通分量
        for (int v : dfs.reversePost()) {
            if (!marked[v]) {
                dfs(G, v);
                count++;
            }
        }
    }
    private void dfs(Digraph G,int v){
        marked[v]=true;
        id[v]=count;
        for (int w:G.adj(v))
            if (!marked[w])
                dfs(G,w);
    }
    //判断任意两点是否是强连通
    public boolean connected(int v,int w){
        return id[v]==id[w];
    }
    public int id(int v){
        return id[v];
    }
    //一共有多少强连通分量
    public int count(){
        return count;
    }
}

```

## Minimum Spanning Trees

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210411105722124.png" alt="image-20210411105722124" style="zoom:50%;" />

一个图的生成树：

- 连通的
- 没有回路
- 包含所有的点

最小生成树：总权值最小的生成树。

最短路径和最小生成树的关系：

- A shortest paths tree depends on the start vertex:Because it tells you how to get from a source to EVERYTHING.
- There is no source for a MST.

Nonetheless, **the MST sometimes happens to be an SPT for a specific vertex**.

所以Dijkstra算法对于最小生成树无法使用，只有对于特殊的节点，该点的最短路径才有可能同时是最小生成树。

Dijkstra：https://docs.google.com/presentation/d/1_bw2z1ggUkquPdhl7gwdVBoTaoJmaZdpkV6MoAgxlJc/pub?start=false&loop=false&delayms=3000&slide=id.g771336078_0_655

切分定理：在一幅加权图中，给定任意的切分，它的横切边中的权重最小者必然属于图的最小生成树。

图的一种切分是将图的所有顶点分为两个非空且不重复的两个集合。横切边是一条连接两个属于不同集合的顶点的边。切分定理的这条性质将会把加权图中的所有顶点分为两个集合、检查横跨两个集合的所有边并识别哪条边应属于图的最小生成树。

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210411191615285.png" alt="image-20210411191615285" style="zoom:50%;" />

切分定理是解决最小生成树问题的所有算法的基础。更确切的说，**这些算法都是一种贪心算法的特殊情况:使用切分定理找到最小生成树的一条边，不断重复直到找到最小生成树的所有边**。

- prim：

  我们使用了一个私有方法visit()来为树添加一个顶点、将它标记为“已访问”并将与它关联的**所有未失效**的边（**即这条边的另一个点没有被访问过**，如果另一个点也被访问过的话，就会构成回路）加入优先队列，以保证队列含有所有连接树顶点和非树顶点的边(也可能含有一些已经失效的边)。代码的内循环是算法的具体实现:我们从优先队列中取出权重最小的一条边并将它添加到树中**(如果它还没有失效的话)**（即这条边的两个点中有一个没有被访问过），再把这条边的另一个顶点也添加到树中，然后用新顶点作为参数调用visit()方法来更新横切边的集合。

  两个重点：

  1. 不要将已经另一个端点已经访问的边加入优先队列，因为这样的边已经被访问过了，如果这样的边在优先队列中，它也是失效的边，不能添加到生成树中
  2. 不要将优先队列中失效的边添加到生成树中（即两个端点都被访问过的边）

  <img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210417145658097.png" alt="image-20210417145658097" style="zoom:50%;" />

  总而言之：两端都访问过的边不能加入优先队列，如果已经加入了，不能加入生成树。

  也就是说：

  - 在将边加入优先队列前，需要判断该边的另一个点是否访问过。
  - 在从优先队列中获取最短的边加入生成树前，需要判断该边是否已经失效，即两个边是否都被访问过（**延迟实现**，保留优先队列中失效的边）
  - 将优先队列中失效的边直接删除（**即时实现**）

  

  对于Dijkstra来说：只需要在松弛某一个点之前，判断该点是否访问过即可，即在优先队列中更新起点到某一点的距离之前，判断该点是否访问过，已访问过的点不需要更新距离。

  我们可以将Dijkstra的过程看作是一棵树，图中每一个点都是其中的一个节点，它的相邻的点（neighbor）对应它的子节点。叶结点都在优先队列中，而所有的内部节点都已经处理过了，从优先队列中删除。每一次以一个叶结点为中心向外探索，将该叶结点从优先队列中删除，将与它相邻的没有处理过的节点加入优先队列，变成新的叶结点。

  延迟实现：

  ```java
  public class prim {
  
      private boolean[]marked;
      private Queue<Edge>mst=new LinkedList<>();
      private PriorityQueue<Edge>pq=new PriorityQueue<>();
      public prim(EdgeWeightedGraph G){
          marked=new boolean[G.V()];
          visit(G,0);
          while (!pq.isEmpty()){
              //从优先队列中获取离生成树最近的点
              Edge e=pq.poll();
              int v=e.either(),w=e.other(v);
              if (marked[v]&&marked[w])continue;
              mst.add(e);
              //以没有经过的点为中心将周围的边加入优先队列
              if (!marked[v])visit(G,v);
              if (!marked[w])visit(G,w);
          }
      }
      //对每一个顶点进行访问
      private void visit(EdgeWeightedGraph G,int v){
          //将访问过的点进行标记
          marked[v]=true;
          for (Edge e:G.adj(v))
              //将所有没有标记的端点的边加入队列
              if (!marked[e.other(v)])pq.offer(e);
      }
  }
  ```

- kruskul

  该算法的关键是：

  1. 将所有的边按权重从小到大排序
  2. 选出最小的边
  3. 如果该边在生成树中构成回路，就丢弃该边，重新选择最小的边
  4. 如果不构成回路，直接加入最小生成树
  5. 重复第二步

  ```java
  class UF{
      private int[]rank;
      private int[]parent;
      public UF(int n){
          parent=new int[n];
          rank=new int[n];
          for (int i=0;i<n;i++){
              parent[i]=i;
              rank[i]=1;
          }
      }
      private int find(int p){
          if (parent[p]!=p)
              parent[p]=find(parent[p]);
          return parent[p];
      }
      public boolean connected(int p,int q){
          return find(p)==find(q);
      }
      public void union(int p,int q){
          int pRoot=find(p);
          int qRoot=find(q);
          if (rank[pRoot]<rank[qRoot])parent[pRoot]=qRoot;
          else if (rank[pRoot]>rank[qRoot])parent[qRoot]=pRoot;
          else {
              parent[qRoot]=pRoot;
              rank[qRoot]++;
          }
      }
  }
  public class Kruskul {
      private Queue<Edge>mst=new LinkedList<>();
      private PriorityQueue<Edge>pq=new PriorityQueue<>();
      public Kruskul(EdgeWeightedGraph G){
          for (Edge e:G.Edges())pq.offer(e);
          UF uf=new UF(G.V());
          while (!pq.isEmpty()&&mst.size()<G.V()-1){
              Edge e=pq.poll();
              int v=e.either(),w=e.other(v);
              if (uf.connected(v,w))continue;
              uf.union(v,w);
            mst.offer(e);
          }
      }
      public Iterable<Edge>edges(){
          return mst;
      }
  }
  ```

  

## SPT

最短路径一定不包括环（负权环和正权环）

Dijkstra：

```java
public class DijkstraSP {
    private int[] edgeTo;
    private double[] distTo;
    //内部存放的值是Int（节点的id），优先级是double（离起点的距离）
    private ArrayHeapMinPQ<Integer> pq;
    private boolean[] marked;

    public DijkstraSP(EdgeWeightedDigraph G, int s) {
        edgeTo = new int[G.V()];
        distTo = new double[G.V()];
        pq = new ArrayHeapMinPQ<>();
        marked = new boolean[G.V()];
        
        for (int v = 0; v < G.V(); v++) distTo[v] = Double.POSITIVE_INFINITY;
        distTo[s] = 0.0;
        edgeTo[s] = -1;
        
        pq.add(s, 0.0);
        while (!pq.isEmpty()) {
            relax(G, pq.removeSmallest());
        }
    }
    //对某个点的周围所有没有标记的边进行松弛
    private void relax(EdgeWeightedDigraph G, int v) {
        marked[v] = true;
        for (DirectedEdge e : G.adj(v)) {
            int w = e.to();
            if (!marked[w] && distTo[w] > distTo[v] + e.weight()) {
                distTo[w] = distTo[v] + e.weight();
                edgeTo[w] = v;
                if (pq.contains(w)) pq.changePriority(w, distTo[w]);
                else pq.add(w, distTo[w]);
            }
        }
    }

    public Iterable<Integer> pathTo(int v) {
        ArrayList<Integer> path = new ArrayList<>();
        int pos = v;
        while (pos != -1) {
            path.add(pos);
            pos = edgeTo[pos];
        }
        Collections.reverse(path);
        return path;
    }
}
```

## 无环加权有向图的最短路径树

Dijkstra适用于任何**正权**图，但是不能用于负权图。而对于**无环加权（负权也可以）有向图**，可以使用AcyclicSP（无环最短路径）。

- 能够在线性时间内解决单点最短路径问题;
- 能够处理负权重的边;
- 能够解决相关的问题，例如找出最长的路径。

只要将顶点的放松和拓扑排序结合起来，马上就能够得到一种解决无环加权有向图中的最短路径问题的算法。

1. 首先，将distTo[s]初始化为0，其他distTo[]元素初始化为无穷大
2. 一个一个地按照拓扑顺序放松所有顶点。

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210523214127060.png" alt="image-20210523214127060" style="zoom:67%;" />

## 负权边的最短路径

Dijkstra的正确性来源于它的访问顺序是从距离源点最近的点到最远的点，所有已经经过的点的距离已经是最优的了，才能在这个点的基础上对周围的边进行松弛。如果有负权边，那么无法保证已经过的点的距离是最优的，还有可能更短，所以无法对负权边使用Dijkstra。

解决负权边的方法就是：将访问顺序从距离源点最近的点到最远的点的贪心算法，变为**每一次都对所有的边进行松弛**。每松弛一轮，从源点出发可以到达的点的距离就增加一条边。所以最多需要松弛n-1轮（n是图中的顶点数），n个顶点的图中，任意两点的距离最多n-1条边。

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210523234148115.png" alt="image-20210523234148115" style="zoom:50%;" />