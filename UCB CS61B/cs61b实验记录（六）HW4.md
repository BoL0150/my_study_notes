# cs61b实验记录（六）HW4、lab11  A* search algorithm

Specifically, given an object of type `WorldState`, our solver will take that `WorldState` and **find a sequence of valid transitions between world states such that the puzzle is solved**.

all puzzles will be implementations of the `WorldState` interface, given below

```java
public interface WorldState {
    /** Provides an estimate of the number of moves to reach the goal.
     * Must be less than or equal to the correct distance. */
    int estimatedDistanceToGoal();

    /** Provides an iterable of all the neighbors of this WorldState. */
    Iterable<WorldState> neighbors();

    /** Estimates the distance to the goal. Must be less than or equal
     *  to the actual (and unknown) distance. */
    default public boolean isGoal() {
      return estimatedDistanceToGoal() == 0;
    }
}
```

every `WorldState` object must be able to 

- return all of its neighbors
- it must have a method to return the estimated distance to the goal and this estimate must be **less than or equal** to the true distance

Suppose that the current `WorldState` is the starting word “horse”, and our goal is to get to the word “nurse”. In this case,

-  `neighbors()` would return `["house", "worse", "hose", "horses"]`，neighbor就是所有只需要一次操作就可以到达的状态。
-  `estimatedDistanceToGoal` might return 2, since changing ‘h’ into ‘n’, and ‘o’ into ‘u’ would result in reaching the goal.只需要检查开始单词和目标单词有多少个字符不一致就行了，这个结果一定小于或等于转换成目标字符所需要的操作次数。

## A* search algorithm 

我们需要定义在puzzle中的搜索单位，`SearchNode`类，其中的属性为：

- `WorldState`
- 从起始状态到当前状态所需要的操作次数
- 指向上一次的状态的引用

`Best-First Search`算法的过程如下：

- 我们需要维护一个队列，该队列的内容就是`SearchNode`对象，从该队列中获取最佳的移动状态
- 将队列中的最佳移动状态移除
- 如果该状态是目标状态，那么结束
- 如果该状态不是目标状态，那么以该状态为中心，获取它的neighbor，创建`SearchNode`对象，将它们加入队列中
- 再从队列中获取最佳的状态，如此往复

而队列的优先级比较机制如下所示：每个`SearchNode`对象的优先级等于 **从起始状态到当前状态的操作次数与当前状态到目标状态的估计操作次数之和**。

The A* algorithm can also be thought of as “Given a state, pick a neighbor state such that (distance so far + estimated distance to goal) is minimized. Repeat until the goal is seen”

To see an example of this algorithm in action, see [this video](https://youtu.be/YFYBjtdl_w4) or these [slides](https://docs.google.com/presentation/d/1FNt7kEFy7R0k0RQS85DgkDk93GLuzAMMcuUjtF4i7Kg).

## Solver

### Optimizations

对应同一个状态的`SearchNode`可能会被多次加入队列中，所以为了避免重复地加入和探索相同的状态，我们需要将所有**经过**的状态进行标记。注意！“经过的状态”指曾经以该状态为中心，将它的neighbor加入队列，也就是**已经出队**的状态！**而不是已经入队**，但是还没有以其为中心向四周探索过的状态！其原理与`Dijkstra`相同，通往这个状态的路不止一条，我们只是探索了其中的一条，此时还无法确定这条路是否是最短的。而将这个状态标记则意味着通向它的路已经确定了，其他的路无法再通往这个状态。我们只有通过优先队列的下一次出队，才能判断此时哪一条路是最短的。出队后，意味着该点已经是优先队列判断出来的离终点最近的点。此时我们才可以将这个点进行标记，表明我们不需要再对这个点进行探索。

标记的方法：可以对所有经过的状态进行标记（`Dijkstra`采用的方法），在本例中的方法是在`SearchNode`中添加一个`previous`属性，引用上一个节点，只需要避免将自己的父节点添加进队列即可。

**A second optimization**：

 To avoid recomputing the `estimatedDistanceToGoal()` result from scratch each time during various priority queue operations, **compute it at most once per object**; save its value in an instance variable; 

### implementation

`SearchNode`：

```java
public class SearchNode implements Comparable<SearchNode>{
    public WorldState word;
    public int moves;
    public SearchNode previous;
    public int priority;
    public SearchNode(WorldState w,int m,SearchNode p){
        word=w;
        moves=m;
        previous=p;
        priority=moves+w.estimatedDistanceToGoal();
    }
    @Override
    public int compareTo(SearchNode o) {
        return this.priority-o.priority;
    }
}
```



`Solver`：

- a naive way：

  ```java
  public class Solver {
      private int moves=0;
      private MinPQ<SearchNode>MP=new MinPQ<>();
      private ArrayList<WorldState>solutions=new ArrayList<>();
  
      public Solver(WorldState initial){
  
          MP.insert(new SearchNode(initial,0,null));
  
          while (!MP.min().word.isGoal()){
              //每以一个点为中心向四周探索就将步数加一
              moves++;
              SearchNode min=MP.min();
              for (WorldState w:min.word.neighbors()){
                  //如果当前探测的neighbor和自己的父节点不相同，就可以加入队列
                  if (min.previous==null||!w.equals(min.previous.word)){
                      MP.insert(new SearchNode(w, moves,min));
                  }
              }
              solutions.add(min.word);
              MP.delMin();
          }
          solutions.add(MP.min().word);
      }
      public int moves(){
          return moves;
      }
  
      public Iterable<WorldState> solution() {
          return solutions;
      }
  }
  ```

  在上面的做法中，我犯了一个致命的错误，我将每一个以它为中心向外探索的点都标记为moves++，但是这实际上并不是该点的真实移动次数。我们需要求的是到达目标状态的最短路径，某一点的真实移动次数应该是起点到该点的直接距离。我误以为从起点到目标点可以沿着一条路走下去，永不回头，直到目标点。但是与Dijkstra和Prim类似，我们永远是在优先队列中选择最佳的路径，我们在一条路上有可能回过头来选择另一条路。

  比如：从horse->nurse，从horse可以分出[hose 4,worse 3,horses 4]，此时我们应该选择worse，将horse从队列中排除出去。从worse可以分出[worst 5]，显然此时我们需要返回去选择hose或horses。然而如果按照我上面的做法，此时如果选择hose，它的移动次数moves就会变成2次（horse->worse->hose），但是它的实际移动次数应该是1次（horse->hose）。所以不应该每探索一个点就把moves++，所有的SearchNode对象的moves应该基于它的父节点的moves+1（类似于DFS和BFS求路径长度的操作）。

  同样，最后求出来的具体路径，应该是从终点开始，沿着每一个点的父节点向上溯源，直到起始点。

- 正确的做法：

  ```java
  public class Solver {
      private MinPQ<SearchNode>MP=new MinPQ<>();
      private ArrayList<WorldState>solutions=new ArrayList<>();
      
      public Solver(WorldState initial){
          MP.insert(new SearchNode(initial,0,null));
          while (!MP.min().word.isGoal()){
              SearchNode min=MP.delMin();
              for (WorldState w:min.word.neighbors())
                  //如果当前探测的neighbor和自己的父节点不相同，就可以加入队列
                  if (min.previous==null||!w.equals(min.previous.word))
                      MP.insert(new SearchNode(w, min.moves+1,min));
          }
      }
      
      public int moves(){
          return MP.min().moves;
      }
  
      public Iterable<WorldState> solution() {
          Stack<WorldState>stack=new Stack<>();
          SearchNode pos=MP.min();
          while (pos!=null) {
              stack.push(pos.word);
              pos=pos.previous;
          }
          while (!stack.isEmpty())
              solutions.add(stack.pop());
          return solutions;
      }
  }
  ```

  BTW，这个算法实在是慢得离谱，我等了半天都没有结果，让我以为是算法写错了，害得我找了一天的bug，结果最后发现是对的，只不过太慢了.........

## Board

### Goal Distance Estimates

We consider two goal distance estimates:

- *Hamming estimate*: The number of tiles in the wrong position.
- *Manhattan estimate*: The sum of the Manhattan distances (sum of the vertical and horizontal distance) from the tiles to their goal positions.

```
8  1  3        1  2  3     1  2  3  4  5  6  7  8    1  2  3  4  5  6  7  8
4     2        4  5  6     ----------------------    ----------------------
7  6  5        7  8        1  1  0  0  1  1  0  1    1  2  0  0  2  2  0  3

initial          goal         Hamming = 5 + 0          Manhattan = 10 + 0
```

我们可以将A* search algorithm的过程看作是一棵树，每一个状态都是其中的一个节点，它的相邻的状态（neighbor）对应它的子节点。叶结点都在优先队列中，而所有的内部节点都已经处理过了，从优先队列中删除。每一次以一个叶结点为中心向外探索，将该叶结点从优先队列中删除，将与它相邻的没有处理过的节点加入优先队列，变成新的叶结点。以上的过程与Prim，Dijkstra算法基本类似。

`Board`：

```java
public class Board implements WorldState{

    private int N;
    private int[][]start;
    private int[][]goal;
    private static final int BLANK=0;
    public Board(int[][] tiles){
        N=tiles.length;
        goal=new int[N][N];
        start = new int[N][N];
        int cnt=1;
        for (int i=0;i<N;i++){
            for (int j=0;j<N;j++){
                goal[i][j]=cnt;
                cnt++;
            }
        }
        goal[N-1][N-1]=0;
        for (int i=0;i<N;i++){
            for (int j=0;j<N;j++){
                start[i][j]=tiles[i][j];
            }
        }
    }
    public int tileAt(int i, int j){
        if (i<0||j<0||i>=N||j>=N)
            throw new IndexOutOfBoundsException("i and j must between 0 and N-1");
        return start[i][j];
    }
    public int size(){
        return N;
    }
    public int hamming(){
        int hamming=0;
        for (int i=0;i<N;i++){
            for (int j=0;j<N;j++){
                if (goal[i][j] != start[i][j] && goal[i][j] != BLANK) {
                    hamming++;
                }
            }
        }
        return hamming;
    }
    public int manhattan(){
        int manhattan=0;
        for (int i=0;i<N;i++){
            for (int j=0;j<N;j++){
                int number=start[i][j];
                if (number!=BLANK){
                    int goalX = (number - 1) / N;
                    int goalY = (number - 1) % N;
                    manhattan += Math.abs(i - goalX) + Math.abs(j - goalY);
                }
            }
        }
        return  manhattan;
    }
    public boolean equals(Object y){
        if (y == this) return true;
        if (y == null || y.getClass() != this.getClass()) return false;
        Board a=(Board)y;
        if (a.size()!=this.size()) return false;
        for (int i=0;i<N;i++)
            for (int j=0;j<N;j++)
                if (a.tileAt(i,j)!=this.tileAt(i,j))
                    return false;
        return true;
    }

    @Override
    public int estimatedDistanceToGoal() {
        return manhattan();
    }
    @Override
    public Iterable<WorldState> neighbors() {
        Queue<WorldState> neighbors = new Queue<>();
        int hug = size();
        int bug = -1;
        int zug = -1;
        for (int rug = 0; rug < hug; rug++) {
            for (int tug = 0; tug < hug; tug++) {
                if (tileAt(rug, tug) == BLANK) {
                    bug = rug;
                    zug = tug;
                }
            }
        }
        int[][] ili1li1 = new int[hug][hug];
        for (int pug = 0; pug < hug; pug++) {
            for (int yug = 0; yug < hug; yug++) {
                ili1li1[pug][yug] = tileAt(pug, yug);
            }
        }
        for (int l11il = 0; l11il < hug; l11il++) {
            for (int lil1il1 = 0; lil1il1 < hug; lil1il1++) {
                if (Math.abs(-bug + l11il) + Math.abs(lil1il1 - zug) - 1 == 0) {
                    ili1li1[bug][zug] = ili1li1[l11il][lil1il1];
                    ili1li1[l11il][lil1il1] = BLANK;
                    Board neighbor = new Board(ili1li1);
                    neighbors.enqueue(neighbor);
                    ili1li1[l11il][lil1il1] = ili1li1[bug][zug];
                    ili1li1[bug][zug] = BLANK;
                }
            }
        }
        return neighbors;
    }
    /** Returns the string representation of the board. 
      * Uncomment this method. */
    public String toString() {
        StringBuilder s = new StringBuilder();
        int N = size();
        s.append(N + "\n");
        for (int i = 0; i < N; i++) {
            for (int j = 0; j < N; j++) {
                s.append(String.format("%2d ", tileAt(i,j)));
            }
            s.append("\n");
        }
        s.append("\n");
        return s.toString();
    }

    @Override
    public int hashCode() {
        return 0;
    }
}
```

autograder评分：

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210417183936503.png" alt="image-20210417183936503" style="zoom:50%;" />

