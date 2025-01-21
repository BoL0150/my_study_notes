# cs61b实验记录（七）Lab11

## Breadth First Search

BFS and DFS are very similar. BFS uses a queue (First In First Out, a.k.a. FIFO) to store the fringe, whereas DFS uses a stack (First In Last Out, a.k.a. FILO). Naturally, programmers often use recursion for DFS, since we can take advantage of and use the implicit recursive call stack as our fringe. For BFS, there is no implicit data structures that we can use. We must instead use an explicit data structure, i.e. some sort of instance of a queue.

```java
public class MazeBreadthFirstPaths extends MazeExplorer {
    /* Inherits public fields:
    public int[] distTo;
    public int[] edgeTo;
    public boolean[] marked;
    */
    private int s;
    private int t;

    public MazeBreadthFirstPaths(Maze m, int sourceX, int sourceY, int targetX, int targetY) {
        super(m);
        maze = m;
        s = maze.xyTo1D(sourceX, sourceY);
        t = maze.xyTo1D(targetX, targetY);
        distTo[s] = 0;
        edgeTo[s] = s;
    }

    /** Conducts a breadth first search of the maze starting at the source. */
    private void bfs() {
        // Your code here. Don't forget to update distTo, edgeTo, and marked, as well as call announce()
        Queue<Integer> queue = new LinkedList<>();
        queue.add(s);
        marked[s]=true;
        announce();
        while (!queue.isEmpty() && queue.peek() != t) {
            int v = queue.poll();
            for (int w : maze.adj(v)) {
                if (marked[w] == false) {
                    marked[w] = true;
                    queue.offer(w);
                    edgeTo[w] = v;
                    distTo[w] = distTo[v] + 1;
                    announce();
                }
            }
        }
    }


    @Override
    public void solve() {
        bfs();
    }
}
```

## Depth First Search & Cycle Check

a weighted-quick union object (without path compression) can be used to detect cycles in `O(E * logV)` time. For today’s exercise, we will use DFS to detect cycles in this maze (an undirected graph) in `O(V + E)`. The idea is relatively simple: For every visited vertex `v`, if there is an adjacent `u` such that `u` is already visited and `u` is not parent of `v`, then there is a cycle in graph.

> If it identifies any cycles, it should connect the vertices of the cycle using purple lines (by setting values in the `edgeTo` array and calling `announce()`) and terminating immediately. All visited vertices should marked, but there should be no edges connecting the part of the graph that doesn’t contain a cycle. Instead, the only edges that should be drawn are the ones connecting the cycle.

```java
public class MazeCycles extends MazeExplorer {
    /* Inherits public fields:
    public int[] distTo;
    public int[] edgeTo;
    public boolean[] marked;
    */
    private class Node{
        int v;
        int parent;
        public Node(int v,int parent){
            this.v=v;
            this.parent=parent;
        }
    }
    public MazeCycles(Maze m) {
        super(m);
    }

    @Override
    public void solve() {
        DFS(new Node(0, 0));
        while (!stack.isEmpty()) {
            int top=stack.pop();
            if (stack.isEmpty())
                break;
            edgeTo[top] =  stack.peek();
            announce();
        }
    }

    private boolean isCycle=false;
    private Stack<Integer>stack=new Stack<>();
    private boolean overlap;
    private int overlapPoint;
    private void DFS(Node node){
        marked[node.v]=true;
        announce();
        for (int w : maze.adj(node.v)) {
            if (marked[w]&&w!=node.parent){
                isCycle=true;
                stack.push(w);
                overlap=false;
                overlapPoint = w;
                return;
            }
            if (!marked[w]){
                marked[w]=true;
                DFS(new Node(w,node.v));
            }
            if (isCycle){
                if (overlap==false)
                    stack.push(w);
                if (overlapPoint==w){
                    overlap=true;
                }
                return;
            }
        }
    }
    // Helper methods go here
}
```

以上代码是在匆忙之中写的，非常丑，仅供参考

## A*

对于A* search algorithm来说，不仅要避免将自己父节点加入队列，同样也需要避免将所有已经以它为中心探测过的点（即已经出队的点）加入队列，因为这样是没有任何意义的。不仅A*如此，所有的图算法都是如此，如prim，Dijkstra，已经访问过的点不能重复添加进优先队列！

```java
public class MazeAStarPath extends MazeExplorer {
    private int s;
    private int t;
    private boolean targetFound = false;
    private Maze maze;

    public MazeAStarPath(Maze m, int sourceX, int sourceY, int targetX, int targetY) {
        super(m);
        maze = m;
        s = maze.xyTo1D(sourceX, sourceY);
        t = maze.xyTo1D(targetX, targetY);
        distTo[s] = 0;
        edgeTo[s] = s;
    }

    /** Estimate of the distance from v to the target. */
    private int h(int v) {
        int size=maze.N();
        int v_X = v % size + 1;
        int v_Y = v / size + 1;
        int target_X = t % size + 1;
        int target_Y = t / size + 1;
        return Math.abs(target_X - v_X) + Math.abs(target_Y - v_Y);
    }

    private class Node implements Comparable<Node>{
        public int vertice;
        public Node parent;
        public int moves;
        public int priority;
        Node(int v,int moves,Node parent){
            this.vertice = v;
            this.moves=moves;
            this.parent = parent;
            this.priority = h(v) + moves;
        }

        @Override
        public int compareTo(Node o) {
            return this.priority - o.priority;
        }
    }
    //标记图中哪些点已经访问过（指曾经以该点为中心向四周探索的点，即从优先队列中出队的点）
    private boolean book[]=new boolean[marked.length];
    /** Performs an A star search from vertex s. */
    private void astar(int s) {
        marked[s] = true;
        announce();
        PriorityQueue<Node>pq=new PriorityQueue<>();
        pq.offer(new Node(s, 0, null));
        while (!pq.isEmpty() && h(pq.peek().vertice) != 0) {
            Node v = pq.poll();
            book[v.vertice]=true;
            for (int w : maze.adj(v.vertice)) {
                //曾经加入过优先队列的点没有访问的意义
                if ((v.parent == null || w != v.parent.vertice) && book[w] == false) {
                    pq.offer(new Node(w, v.moves + 1, v));
                    marked[w] = true;
                    edgeTo[w] = v.vertice;
                    distTo[w] = distTo[v.vertice] + 1;
                    announce();
                }
            }
        }
    }

    @Override
    public void solve() {
        astar(s);
    }

}
```



总结：这个几个图的实验真的非常好，让人形象深刻地理解了图算法的本质与区别，以及它们的使用场景。

`DFS`：

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210418010751038.png" alt="image-20210418010751038" style="zoom: 67%;" />

`BFS`：

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210418010832107.png" alt="image-20210418010832107" style="zoom:67%;" />

`A*`

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210418010901188.png" alt="image-20210418010901188" style="zoom:67%;" />

AG评分：

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210418011025846.png" alt="image-20210418011025846" style="zoom:50%;" />