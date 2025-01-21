# cs61b实验记录（三）project 2 prim迷宫随机生成算法

## HW1

nothing special

具体详见我的GitHub：https://github.com/BoL0150/Berkeley-cs61b/tree/main/hw1

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210319195834911.png" alt="image-20210319195834911" style="zoom:50%;" />

## Lab5

### Random

Random对象是一个“伪随机数”生成器，它可以产生一串无穷的看起来是随机数的数字序列，调用nextInt方法获取序列中的每一个数字。

```java
Random r = new Random(1000);
System.out.println(r.nextInt());
System.out.println(r.nextInt());
System.out.println(r.nextInt());
```

它之所以叫“伪随机数”是因为它产生的序列并不是真正随机的。我们获取不同的序列的方式是向Random的构造器中传入一个数字，这个数字被称为“seed”，如果我们用相同的seed构造Random，那么我们一定会获得完全相同的序列。

java.lang.Math中也有一个名为random()的静态方法可以生成随机数

Math.random() 方法生成[0, 1)范围内的double类型随机数；**Random类中的nextXxxx(n)系列方法生成0（包括）到n（不包括）之间的随机数；**

## project2 phase1

绘制地图：

从宏观上来看，主要分为这么几步：

1. 首先加入房间
2. 然后在房间的周围修建迷宫
3. 将房间和迷宫打通

### 0、准备工作
首先描述一下我的代码的整体架构（将这一步放在最前面只是为了方便阅读代码，在实际的写代码的过程中，并不是一开始就能有一个清晰的架构的）
主要包含以下几个类：

 - `MapGenerator`：生成地图的整体框架，该类中有如下的全局变量
   - `private int WIDTH`：地图的宽
   - `private int HEIGHT`：地图的高
   - `private Random RANDOM`：Random对象
   - `private long SEED`：当前地图的seed
   - `private static final int TIMES=100`：随机生成房间的上限次数
   - `private static TETile[][]world`：生成的世界数组
   - `private static TETile[][]positionOfRoom`：记录房间位置的数组
   - `private static ArrayList<Room>Rooms`：记录生成的房间的数据的List
- `HallWay`：用来生成迷宫的工具类
- `Room`：房间类
  - `public int x`
  - `public int y`
  - `public int WIDTH`
  - `public int HEIGHT`，以上分别是每一个房间的坐标和宽高
- `Position`：位置类
  - `public int x`
  - `public int y`

### 1、生成房间

创建一个Room类，将每一个房间当作一个对象，随机生成房间的坐标以及长和宽，然后判断房间是否重叠，

- 如果重叠则重新生成
- 如果不重叠就在数组中建造房间

同时设置一个参数，控制房间重复生成的上限次数。**将所有生成的房间对象放在一个List中**，因为在修建完迷宫后还需要对每一个房间与它旁边的迷宫进行打通，以便获取房间的位置和参数。

```java
private void fillWithRoom(){
        //nextInt方法返回0到WIDTH-1之间和0到HEIGHT-1之间的数
        for (int i=0;i<TIMES;i++){
            int x=RANDOM.nextInt(WIDTH);
            int y=RANDOM.nextInt(HEIGHT);
            int width=RANDOM.nextInt(WIDTH/10)+2;
            int height=RANDOM.nextInt(HEIGHT/5)+2;
            if (y+height+1>=HEIGHT||x+width+1>=WIDTH)
                continue;
            if (isOverlap(x,y,width,height))
                continue;
            buildRoom(x,y,width,height);
            //将room添加到ArrayList中记录下来，以便在Room外添加完迷宫后对每一个房间打通和迷宫的连接。
            Rooms.add(new Room(x,y,width,height));
        }

    }
//造房间
    private void buildRoom(int x,int y,int width,int height){
        for (int i=x;i<=x+width+1;i++){
            for (int j=y;j<=y+height+1;j++){
                if (i==x||i==x+width+1||j==y||j==y+height+1){
                    positionOfRoom[i][j]=Tileset.WALL;
                    world[i][j]=Tileset.WALL;
                    continue;
                }
                //由于FLOOR和在房间外的迷宫的连通条件相冲突，
                //所以先在房间内部填GRASS
                world[i][j]=Tileset.GRASS;
                positionOfRoom[i][j]=Tileset.GRASS;
            }
        }
    }
```

此时在房间中填GRASS以及使用两个数组的原因稍后进行解释。

```java
//判断房间是否重叠
    private boolean isOverlap(int x,int y,int width,int height){
        for (int i=x;i<=x+width+1;i++)
            for (int j=y;j<=y+height+1;j++)
                if (positionOfRoom[i][j]==Tileset.WALL||positionOfRoom[i][j]==Tileset.GRASS)
                    return true;
        return false;
    }
```

### 2、在房间的周围生成迷宫

我们将这一步分解来看，首先：如何在一张空的图上生成迷宫？

#### 2.1在一张空的图上生成迷宫

我们采用**prim迷宫随机生成算法**，此算法的原理及具体实现如下：

原理：

1. 将图上的点用墙分隔开，如下图所示

   ![image-20210328081625328](https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210328081625328.png)

2. 此时，我们可以将图中空白的点看作路，而我们要做的就是随机地打通**每个点**和它**相邻**的某个点（“相邻”指两点之间隔着一面墙）。我们需要遍历图中所有空白的点，同时打通每一个空白的点和它相邻的某个点。当图上所有空白的点都被我们经过后，图上的所有的点也就都是相互可达的，被我们打通的墙就构成了路，剩下的墙就是迷宫。

具体实现：

1. 一排隔着一排，一列隔着一列生成像棋盘一样的网格图

   ```java
   public static void initializeHallWay(TETile[][]world){
           for (int i=0;i<=world.length-1;i++)
               for (int j=0;j<=world[0].length-1;j++)
                   world[i][j]= Tileset.WALL;
   
           for (int i=1;i<=world.length-2;i+=2)
               for (int j=1;j<=world[0].length-2;j+=2)
                   world[i][j]=Tileset.NOTHING;
       }
   ```

2. 在所有空白的点中（即没有墙的点）随机选择一个点作为起始点

   ```java
   private static Position RandomStart(TETile[][]world,long seed){
           Random RANDOM=new Random(seed);
           while (true){
               int startX=RANDOM.nextInt(world.length-2);
               int startY=RANDOM.nextInt(world[0].length-2);
               startX=startX%2==1?startX:startX+1;
               startY=startY%2==1?startY:startY+1;
               if (world[startX][startY]==Tileset.NOTHING)
                   return new Position(startX,startY);
           }
       }
   ```

   以该点为中心，检查上下左右的四个点，将没有经过的点加入候选列表，如果周围有经过的点，就打通当前的点和周围**某一个**经过的点之间的墙。当前点的周围的点检查完毕后，从候选列表中随机选出一个点为中心点，检查上下左右的四个点。如此反复，直到候选列表为空（即图中的所有点都已经经过了）。

   ```java
   public static void generateHallWay(TETile[][]world,Position start,long seed){
           LinkedList<Position> positionList=new LinkedList<>();
   
           positionList.add(start);
           Random RANDOM=new Random(seed);
           while (!positionList.isEmpty()){
               //随机获取一个点
               int index=RANDOM.nextInt(positionList.size());
               Position curPos=positionList.get(index);
               //当前点的位置
               int curX=curPos.x;
               int curY=curPos.y;
               //将该点置为FLOOR，标记该点已经被走过
               world[curX][curY]=Tileset.FLOOR;
               //将这个点周围的空的点加入候选队列，顺便打通和已经过的点之间的墙
               PositionsAvailableAroundAndConnectPath(curPos,world,positionList,seed);
               positionList.remove(index);
           }
   
       }
   ```

   寻找某一点周围的可以扩展的点，将它们加入候选列表，顺便打通和已经过的点之间的墙

   ```java
   //寻找某一点周围的可以扩展的点，将它们加入候选列表，顺便打通和已经过的点之间的墙
       private static void PositionsAvailableAroundAndConnectPath(Position position, TETile[][]world, LinkedList<Position>positionList,long seed){
   
           int[][]nextDirction={
                   {2,0},
                   {0,-2},
                   {-2,0},
                   {0,2}
           };
           Random RANDOM=new Random(seed);
           //标记每一个方向是否走过
           boolean[]book=new boolean[4];
           //只能连接一次
           boolean isConnected=false;
           while (book[0]==false||book[1]==false||book[2]==false||book[3]==false){
               //随机选取一个方向
               int next=RANDOM.nextInt(4);
               //如果这个方向已经走过了，则重新选择方向
               if (book[next])continue;
               book[next]=true;
               int nextX=position.x+nextDirction[next][0];
               int nextY=position.y+nextDirction[next][1];
               //如果这个点越界了，就选下一个点
               if (nextX<0||nextX>world.length-1||nextY<0||nextY>world[0].length-1)
                   continue;
               //如果这个点什么也没有，符合要求，加入候选队列
               if (Tileset.NOTHING.equals(world[nextX][nextY])){
                   //将该点变为Flower，标记它在候选队列中
                   world[nextX][nextY]=Tileset.FLOWER;
                   positionList.add(new Position(nextX,nextY));
               }
               //如果这个点已经经过，就将它们相连，即打通它们之间的墙
               if (Tileset.FLOOR.equals(world[nextX][nextY])&&isConnected==false){
                   int mx=(position.x+nextX)/2;
                   int my=(position.y+nextY)/2;
                   //将它们之间的墙变为FLOOR
                   world[mx][my]=Tileset.FLOOR;
                   isConnected=true;
               }
           }
       }
   ```

3. 此时迷宫就已形成了

   ![image-20210328084951433](https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210328084951433.png)

生成迷宫时要注意**随机**从候选列表中选取点，否则生成的迷宫会朝着同一个方向

#### 2.2在房间的周围生成迷宫

我们知道了如何在空的图上生成迷宫，也可以由此推断出如何在房间的周围生成迷宫

1. 先在图中生成类似棋盘的网格图

2. 再在网格图中生成房间

3. 由于图中已经有了很多墙，所以我们难以判断房间是否重叠。因此我另外造了一个数组，专门标记房间的位置，来判断房间是否重叠

4. 由于生成迷宫时，打通两点之间的墙的条件是两点都已经过，即两点都变成了FLOOR。所以，为了避免此条件与房间内的FLOOR相冲突，所以我们在初始化房间时，将房间内的内容暂时置为GRASS。

   ```java
   private void fillWithRoom(){
           //nextInt方法返回0到WIDTH-1之间和0到HEIGHT-1之间的数
           for (int i=0;i<TIMES;i++){
               int x=RANDOM.nextInt(WIDTH);
               int y=RANDOM.nextInt(HEIGHT);
               int width=RANDOM.nextInt(WIDTH/10)+2;
               int height=RANDOM.nextInt(HEIGHT/5)+2;
               if (y+height+1>=HEIGHT||x+width+1>=WIDTH)
                   continue;
               if (isOverlap(x,y,width,height))
                   continue;
               buildRoom(x,y,width,height);
               //将room添加到ArrayList中记录下来，以便在Room外添加完迷宫后对每一个房间打通和迷宫的连接。
               Rooms.add(new Room(x,y,width,height));
           }
   
       }
       //造房间
       private void buildRoom(int x,int y,int width,int height){
           for (int i=x;i<=x+width+1;i++){
               for (int j=y;j<=y+height+1;j++){
                   if (i==x||i==x+width+1||j==y||j==y+height+1){
                       positionOfRoom[i][j]=Tileset.WALL;
                       world[i][j]=Tileset.WALL;
                       continue;
                   }
                   //由于FLOOR和在房间外的迷宫的连通条件相冲突，
                   //所以先在房间内部填GRASS
                   world[i][j]=Tileset.GRASS;
                   positionOfRoom[i][j]=Tileset.GRASS;
               }
           }
       }
       //判断房间是否重叠
       private boolean isOverlap(int x,int y,int width,int height){
           for (int i=x;i<=x+width+1;i++)
               for (int j=y;j<=y+height+1;j++)
                   if (positionOfRoom[i][j]==Tileset.WALL||positionOfRoom[i][j]==Tileset.GRASS)
                       return true;
           return false;
       }
   ```

### 3、将房间和迷宫打通

这一步没什么好说的，对每一个房间随机选取一条边上的一点向外打通，如果不符合条件（如碰到NOTHING等）就重新选取一点。

```java
//在房间上开口，打通房间和迷宫的连接
    private void connectRoomWithHallWay(){
        for (int i=0;i<Rooms.size();i++){
            Room curRoom=Rooms.get(i);
            removeWall(curRoom);
        }
    }
    private void removeWall(Room curRoom){
        for (int i=0;i<100;i++){
            int index=RANDOM.nextInt(4);
            int mx,my;
            switch (index){
                //在左边的墙壁上挖洞
                case 0:
                    mx=curRoom.x;
                    my=RANDOM.nextInt(curRoom.HEIGHT)+curRoom.y+1;
                    if (!canBeRemoved(mx,my,index))continue;
                    world[mx][my]=Tileset.FLOOR;
                    if (Tileset.FLOOR.equals(world[mx-1][my]))
                        return;
                    world[mx-1][my]=Tileset.FLOOR;
                    if (Tileset.GRASS.equals(world[mx-2][my]))
                        continue;
                    return;
                //在右边的墙壁上挖洞
                case 1:
                    mx=curRoom.x+curRoom.WIDTH+1;
                    my=RANDOM.nextInt(curRoom.HEIGHT)+curRoom.y+1;
                    if (!canBeRemoved(mx,my,index))continue;
                    world[mx][my]=Tileset.FLOOR;
                    if (Tileset.FLOOR.equals(world[mx+1][my]))
                        return;
                    world[mx+1][my]=Tileset.FLOOR;
                    if (Tileset.GRASS.equals(world[mx+2][my]))
                        continue;
                    return;
                //在下面的墙壁上挖洞
                case 2:
                    mx=RANDOM.nextInt(curRoom.WIDTH)+curRoom.x+1;
                    my=curRoom.y;
                    if (!canBeRemoved(mx,my,index))continue;
                    world[mx][my]=Tileset.FLOOR;
                    if (Tileset.FLOOR.equals(world[mx][my-1]))
                        return;
                    world[mx][my-1]=Tileset.FLOOR;
                    if (Tileset.GRASS.equals(world[mx][my-2]))
                        continue;
                    return;
                //在上面的墙壁上挖洞
                case 3:
                    mx=RANDOM.nextInt(curRoom.WIDTH)+curRoom.x+1;
                    my=curRoom.y+curRoom.HEIGHT+1;
                    if (!canBeRemoved(mx,my,index))continue;
                    world[mx][my]=Tileset.FLOOR;
                    if (Tileset.FLOOR.equals(world[mx][my+1]))
                        return;
                    world[mx][my+1]=Tileset.FLOOR;
                    if (Tileset.GRASS.equals(world[mx][my+2]))
                        continue;
                    return;

            }

        }
    }
    private boolean canBeRemoved(int x,int y,int direcrion){
        if (Tileset.FLOOR.equals(world[x][y]))
            return false;
        switch (direcrion){
            //向左挖：
            case 0:
                if (x<=1)
                    return false;
                if (Tileset.NOTHING.equals(world[x-1][y])||(Tileset.WALL.equals(world[x-2][y])&&Tileset.WALL.equals(world[x-1][y])))
                    return false;
                return true;
            //向右挖：
            case 1:
                if (x>=WIDTH-2)
                    return false;
                if (Tileset.NOTHING.equals(world[x+1][y])||(Tileset.WALL.equals(world[x+2][y])&&Tileset.WALL.equals(world[x+1][y])))
                    return false;
                return true;
            //向下挖：
            case 2:
                if (y<=1)
                    return false;
                if (Tileset.NOTHING.equals(world[x][y-1])||(Tileset.WALL.equals(world[x][y-2])&&Tileset.WALL.equals(world[x][y-1])))
                    return false;
                return true;
            //向上挖：
            case 3:
                if (y>=HEIGHT-2)
                    return false;
                if (Tileset.NOTHING.equals(world[x][y+1])||(Tileset.WALL.equals(world[x][y+2])&&Tileset.WALL.equals(world[x][y+1])))
                    return false;
                return true;
        }
        return false;
    }
```

![打通后的样子](https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210328090734871.png)

然而，此时的图中还有很多的死胡同（即三面都是墙的FLOOR），以及房间中依然是GRASS，所以我们需要填补所有的死胡同，将GRASS替换为FLOOR。

#### 3.1去除所有的死胡同以及GRASS

去除死胡同我们需要对每一个点，查看周围的四个点是否是WALL，然后改变这个点，再进入下一个点。这会让人想起DFS，但是原本的DFS是沿着一条路线，从一头走到另一头，对路上的每一个点都只是**依次**查看周围的点，一旦找到可以通过的点，就立马进入，**无法确定这一点周围是否有3个WALL**。只有当走到头时，扫描了周围的四个点，发现都无法通过，才会往后退。也就是说，只有后退的时候，我们才能知道某一点周围所有点的情况。而填补所有的死胡同需要我们从所有的死胡同的终点出发，向中间汇聚，一边移动一边填补。

所以我们需要将DFS改造成**前进和后退时都要查看周围所有点的情况**，才能进行下一步。

```java
 private void removeDeadends(){
        for (int i=0;i<WIDTH;i++){
            for (int j=0;j<HEIGHT;j++){
                if ((Tileset.GRASS.equals(world[i][j])||Tileset.FLOOR.equals(world[i][j]))){
                    world[i][j]=Tileset.FLOOR;
                    int cnt=0;
                    if (Tileset.WALL.equals(world[i-1][j]))
                        cnt++;
                    if (Tileset.WALL.equals(world[i+1][j]))
                        cnt++;
                    if (Tileset.WALL.equals(world[i][j-1]))
                        cnt++;
                    if (Tileset.WALL.equals(world[i][j+1]))
                        cnt++;
                    if (cnt>=3)
                        DFS(i,j);
                }
            }
        }
    }
private static int[][]next={
            {1,0},
            {0,-1},
            {-1,0},
            {0,1}
    };
    private boolean[][] isPassed;
    private boolean DFS(int x,int y){
        int cnt=0;
        Queue<Position>accessiblePath=new LinkedList<>();
        //先查找某一点周围所有的点，将可以通行的点加入候选列表
        for (int i=0;i<=3;i++){
            int mx=x+next[i][0];
            int my=y+next[i][1];
            if (mx<0||mx>=WIDTH||my<0||my>=HEIGHT)
                continue;
            if (Tileset.WALL.equals(world[mx][my])){
                cnt++;
                continue;
            }
            if (isPassed[mx][my]==true) {
                continue;
            }
            if (Tileset.GRASS.equals(world[mx][my]))
                world[mx][my]=Tileset.FLOOR;
            if (Tileset.FLOOR.equals(world[mx][my])){
                accessiblePath.offer(new Position(mx,my));
            }
        }
        if (cnt>=3)world[x][y]=Tileset.WALL;
        while (!accessiblePath.isEmpty()){
            Position pos=accessiblePath.peek();
            isPassed[pos.x][pos.y]=true;
            if(DFS(pos.x,pos.y))cnt++;
            if (cnt>=3)
                world[x][y]=Tileset.WALL;
            accessiblePath.poll();
        }
        return cnt>=3;
    }
```

![image-20210328091651206](https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210328091651206.png)

我们还需要移除所有多余的墙，也就是四个角上没有FLOOR的WALL

#### 3.2移除多余的墙

```java
private void destroyWall(){
        int[][]next2={
                {1,1},
                {1,-1},
                {-1,-1},
                {-1,1}
        };
        for (int i=0;i<world.length;i++){
            for (int j=0;j<world[0].length;j++){
                if (Tileset.WALL.equals(world[i][j])){
                    boolean isDestroy=true;
                    //判断某一点对角线上的四个点是否是FLOOR
                    for (int k=0;k<4;k++){
                        int mx=i+next2[k][0];
                        int my=j+next2[k][1];
                        if (mx<0||mx>=WIDTH||my<0||my>=HEIGHT)
                            continue;
                        if (Tileset.FLOOR.equals(world[mx][my]))
                            isDestroy=false;
                    }
                    if (isDestroy==true)
                        world[i][j]=Tileset.NOTHING;
                }
            }
        }
    }
```

最后，再添加上Player和Lock_Door就完成了

```java
private void GenerateDoor(){
        while (true){
            int x=RANDOM.nextInt(WIDTH);
            int y=RANDOM.nextInt(HEIGHT);
            if (Tileset.FLOOR.equals(world[x][y])){
                door=new Position(x,y);
                world[x][y]=Tileset.LOCKED_DOOR;
                return;
        }
    }
    }
    private void GeneratePlayer(){
        while (true){
            int x=RANDOM.nextInt(WIDTH);
            int y=RANDOM.nextInt(HEIGHT);
            if (Tileset.FLOOR.equals(world[x][y])){
                player=new Position(x,y);
                world[x][y]=Tileset.PLAYER;
                return;
            }
        }
    }
```

![image-20210328092137841](https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210328092137841.png)