# Android课程设计：迷宫游戏

## 项目介绍及结构

本项目是一个迷宫游戏，用户从起点走到终点就代表成功。功能包括：

- 点击地图上的一个相邻位置，如果该位置不是障碍物，则在该位置处进行标记，表示走过此处。
- 点击后退按钮，取消当前所在位置的标记。
- 点击重新加载按钮，可重新加载当前关卡的地图。
- 点击显示结果按钮，可显示从起点到终点的完整最短路径。
- 点击下一关按钮，当到达终点后，可以进入下一关。
- 显示到目前为止一共移动了多少次。



## 页面布局

### MainActivity的布局

使用约束性布局，`ConstraintLayout`可使用扁平视图层次结构（无嵌套视图组）创建复杂的大型布局。它与 `RelativeLayout`相似，其中所有的视图均根据同级视图与父布局之间的关系进行布局，但其灵活性要高于 `RelativeLayout`，并且更易于与 Android Studio 的布局编辑器配合使用。

#### GridView控件

要想实现一个迷宫，首先要有网格`GridView`控件，`GridView` 跟`ListView` 很类似，`Listview` 主要以列表形式显示数据，`GridView` 则是以网格形式显示数据，`GridView`网格视图是按照行，列分布的方式来显示多个组件,通常用于显示图片或是图标等，在使用网格视图时,首先需要要在屏幕上添加`GridView`组件。它和`ListView`一样是`AbsListView`的子类。

`GridView`的继承关系如下：

```css
java.lang.Object
   ↳    android.view.View
       ↳    android.view.ViewGroup
           ↳    android.widget.AdapterView<android.widget.ListAdapter>
               ↳    android.widget.AbsListView
                   ↳    android.widget.GridView
```

下面是GridView中的一些属性：

- android:columnWidth：设置列的宽度
- android:gravity：组件对其方式
- android:horizontalSpacing：水平方向每个单元格的间距
- android:verticalSpacing：垂直方向每个单元格的间距
- android:numColumns：设置列数
- android:stretchMode：设置拉伸模式，可选值如下： none：不拉伸；spacingWidth：拉伸元素间的间隔空隙 columnWidth：仅仅拉伸表格元素自身 spacingWidthUniform：既拉元素间距又拉伸他们之间的间隔空隙

代码如下：

```xml
   <GridView
        android:id="@+id/gv"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:layout_marginBottom="@dimen/grid_marginBottom"
        android:gravity="center"
        android:horizontalSpacing="@dimen/grid_spacing"
        android:verticalSpacing="@dimen/grid_spacing"></GridView>
```

在GridView控件中，我们通过android:horizontalSpacing="1dp"和android:verticalSpacing="1dp"指定了网格之间的水平距离和垂直距离都为1dp。

#### 按钮

四个按钮，分别操作：

- 返回:可后退一步
- 加载:可重新载入迷宫
- 提示:可显示最短路径
- 刷新:可刷新迷宫地图

```xml
<!--加载:可重新载入迷宫-->
    <ImageView
        android:id="@+id/iv_reload"
        android:layout_width="0dp"
        android:layout_height="@dimen/op_height"
        android:scaleType="center"
        android:src="@drawable/reload"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toStartOf="@id/iv_show"
        app:layout_constraintStart_toEndOf="@id/iv_back" />
<!--提示:可显示最短路径-->
    <ImageView
        android:id="@+id/iv_show"
        android:layout_width="0dp"
        android:layout_height="@dimen/op_height"
        android:scaleType="center"
        android:src="@drawable/hide"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toStartOf="@id/iv_next"
        app:layout_constraintStart_toEndOf="@id/iv_reload" />
<!--刷新:可刷新迷宫地图-->
    <ImageView
        android:id="@+id/iv_next"
        android:layout_width="0dp"
        android:layout_height="@dimen/op_height"
        android:scaleType="center"
        android:src="@drawable/selector_next"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toEndOf="@id/iv_show" />
```

- app:layout_constraintTop_toBottomOf 在指定View的下方  
- app:layout_constraintStart_toEndOf 在指定View的右边
- app:layout_constraintLeft_toRightOf 在指定View的右边
- app:layout_constraintBottom_toTopOf 在指定View的上方
- app:layout_constraintEnd_toStartOf 在指定View的左边 
- app:layout_constraintRight_toLeftOf 在指定View的左边

#### 文本

显示已经使用的步数

```xml
<TextView
          android:id="@+id/tv_step_title"
          android:layout_width="wrap_content"
          android:layout_height="@dimen/step_height"
          android:layout_marginStart="@dimen/step_margin"
          android:layout_marginLeft="@dimen/step_margin"
          android:gravity="center|left"
          android:text="已用步数"
          android:textSize="@dimen/step_textSize"
          app:layout_constraintBottom_toTopOf="@id/iv_back"
          app:layout_constraintStart_toStartOf="parent" />
<TextView
          android:id="@+id/tv_step"
          android:layout_width="wrap_content"
          android:layout_height="@dimen/step_height"
          android:layout_marginStart="@dimen/step_margin"
          android:layout_marginLeft="@dimen/step_margin"
          android:gravity="center|left"
          android:text="0"
          android:textSize="@dimen/step_textSize"
          app:layout_constraintBottom_toTopOf="@id/iv_back"
          app:layout_constraintStart_toEndOf="@id/tv_step_title" />
```

![image-20210611105424404](https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210611105424404.png)

### GridView 的 Item的布局

在每个网格内，我们都需要显示两项内容：玩家自己走过的位置或者是提示的正确路线。因此，我们还需要对网格内元素进行相应的布局。

我们可以在项目工程的layout目录下新建一个名为“item_grid.xml”的xml布局文件，完成对网格内元素的布局。在该xml布局文件中，我们使用相对布局RelativeLayout对网格内的元素进行排列，将一个ImageView控件以水平居中的形式放置在网格内，用来玩家自己走过位置的图标；将一个ImageView控件以水平居中的形式放置在网格内，用来显示正确路线的图标。具体的item_grid.xml源码如下：

```xml
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/item"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="@drawable/shape_maze_normal">

    <ImageView
        android:id="@+id/icon"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_centerInParent="true"
        android:scaleType="center" />

    <ImageView
        android:id="@+id/perfect"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_centerInParent="true"
        android:scaleType="center" />

</RelativeLayout>
```

## MainActivity

### 数据结构

网格中的item：

```java
public class GridItem {
    public int x;
    public int y;
    //是否是障碍物
    public boolean barrier;
    //是否是起点
    public boolean start;
    //是否是终点
    public boolean end;
    //该网格是否是正确路线
    public boolean result;
    //是否已经走过
    public boolean selected;
    public GridItem() {

    }
    public GridItem(int x, int y) {
        this.x = x;
        this.y = y;
    }
}
```

路径上的点：

```java
public class Grid implements Comparable<Grid>{
    public int x;
    public int y;
    //该点在优先队列中的优先级
    public int priority;
    //到起点的距离
    public int distanceToStart;
    //到终点的估计距离
    public int h;
    public Grid parent;

    public Grid(int x, int y) {
        this.x = x;
        this.y = y;
    }
    public  void initGrid(Grid parent, Grid end){
        this.parent = parent;
        if (parent != null) {
            this.distanceToStart = parent.distanceToStart + 1;
        } else {
            this.distanceToStart = 1;
        }
        //到终点的估计距离
        this.h = Math.abs(this.x - end.x) + Math.abs(this.y - end.y);
        //优先级等于到起点的距离和到终点的估计距离
        this.priority = this.distanceToStart + this.h;
    }

    @RequiresApi(api = Build.VERSION_CODES.KITKAT)
    @Override
    public int compareTo(Grid grid) {
        return Integer.compare(this.priority, grid.priority);
    }
}
```

### 初始化线程池

为了提高应用性能，使用了多线程来负责不同的功能。

Java 给多线程编程提供了内置的支持。 一条线程指的是进程中一个单一顺序的控制流，一个进程中可以并发多个线程，每条线程并行执行不同的任务。多线程是多任务的一种特别的形式，但多线程使用了更小的资源开销。多线程能满足程序员编写高效率的程序来达到充分利用 CPU 的目的。

一个线程的生命周期：

线程是一个动态执行的过程，它也有一个从产生到死亡的过程。

下图显示了一个线程完整的生命周期。

- 新建状态:

  使用 new 关键字和 Thread 类或其子类建立一个线程对象后，该线程对象就处于新建状态。它保持这个状态直到程序 start() 这个线程。

- 就绪状态:

  当线程对象调用了start()方法之后，该线程就进入就绪状态。就绪状态的线程处于就绪队列中，要等待JVM里线程调度器的调度。

- 运行状态:

  如果就绪状态的线程获取 CPU 资源，就可以执行 run()，此时线程便处于运行状态。处于运行状态的线程最为复杂，它可以变为阻塞状态、就绪状态和死亡状态。

- 阻塞状态:

  如果一个线程执行了sleep（睡眠）、suspend（挂起）等方法，失去所占用资源之后，该线程就从运行状态进入阻塞状态。在睡眠时间已到或获得设备资源后可以重新进入就绪状态。可以分为三种：

  - 等待阻塞：运行状态中的线程执行 wait() 方法，使线程进入到等待阻塞状态。
  - 同步阻塞：线程在获取 synchronized 同步锁失败(因为同步锁被其他线程占用)。
  - 其他阻塞：通过调用线程的 sleep() 或 join() 发出了 I/O 请求时，线程就会进入到阻塞状态。当sleep() 状态超时，join() 等待线程终止或超时，或者 I/O 处理完毕，线程重新转入就绪状态。

- 死亡状态:

  一个运行状态的线程完成任务或者其他终止条件发生时，该线程就切换到终止状态。

在此应用中，主线程负责点击网格的事件处理，一个线程负责点击后退按钮的事件处理，一个线程负责点击重新加载地图按钮的事件处理，一个线程负责点击展示路径按钮的事件处理，一个线程负责点击进入下一关的按钮的事件处理。

Java语言虽然内置了多线程支持，启动一个新线程非常方便，但是，创建线程需要操作系统资源（线程资源，栈空间等），频繁创建和销毁大量线程需要消耗大量时间。那么我们就可以把很多小任务让一组线程来执行，而不是一个任务对应一个新线程。这种能接收大量小任务并进行分发处理的就是线程池。线程过多会带来额外的开销，其中包括创建销毁线程的开销、调度线程的开销等等，同时也降低了计算机的整体性能。线程池维护多个线程，等待监督管理者分配可并发执行的任务。这种做法，一方面避免了处理任务时创建销毁线程开销的代价，另一方面避免了线程数量膨胀导致的过分调度问题，保证了对内核的充分利用。

简单地说，线程池内部维护了若干个线程，没有任务的时候，这些线程都处于等待状态。如果有新任务，就分配一个空闲线程执行。如果所有线程都处于忙碌状态，新任务要么放入队列等待，要么增加一个新线程进行处理。

使用线程池可以带来一系列好处：

- 降低资源消耗：通过池化技术重复利用已创建的线程，降低线程创建和销毁造成的损耗。
- 提高响应速度：任务到达时，无需等待线程创建即可立即执行。
- 提高线程的可管理性：线程是稀缺资源，如果无限制创建，不仅会消耗系统资源，还会因为线程的不合理分布导致资源调度失衡，降低系统的稳定性。使用线程池可以进行统一的分配、调优和监控。
- 提供更多更强大的功能：线程池具备可拓展性，允许开发人员向其中增加更多的功能。比如延时定时线程池ScheduledThreadPoolExecutor，就允许任务延期执行或定期执行。

为了保证全局只有一个线程池存在，我们还使用了单例模式，单例模式（Singleton Pattern）是 Java 中最简单的设计模式之一。这种类型的设计模式属于创建型模式，它提供了一种创建对象的最佳方式。

这种模式涉及到一个单一的类，该类负责创建自己的对象，同时确保只有单个对象被创建。这个类提供了一种访问其唯一的对象的方式，可以直接访问，不需要实例化该类的对象。

- 单例类只能有一个实例。
- 单例类必须自己创建自己的唯一实例。
- 单例类必须给所有其他对象提供这一实例。

优点：

- 在内存里只有一个实例，减少了内存的开销，尤其是频繁的创建和销毁实例（比如管理学院首页页面缓存）。
- 避免对资源的多重占用（比如写文件操作）。

缺点：没有接口，不能继承，与单一职责原则冲突，一个类应该只关心内部逻辑，而不关心外面怎么样来实例化。

MazeManager类使用单例模式管理线程池：

1. 私有化构造器

   ```java
   private MazeManager() {
   
       }
   ```

2. 内部提供一个静态的当前类的实例，整个类的所有对象共享该实例

   ```java
   public static MazeManager mInstance = new MazeManager();
   ```

3. 提供公共的静态方法，返回MazeManager对象

   ```java
   public static MazeManager get() {
           return mInstance;
       }
   ```

4. 提供指定属性的线程池

   ```java
   //提供指定属性的线程池
       private void init() {
           mExecutor = new ThreadPoolExecutor(0, 10, 60, TimeUnit.SECONDS, new LinkedBlockingDeque<Runnable>(10));
       }
   
       /**
        * 执行任务，执行task中重写的Run方法
        *
        * @param task
        */
       public void execute(Runnable task) {
           //以当前的MazeManager类作为锁
           synchronized (MazeManager.class) {
               if (mExecutor == null) {
                   init();
               }
           }
           //对每个任务，都需要从线程池中取出一个线程来执行
           mExecutor.execute(task);
       }
   ```

### Adapter的定义

GridView主要通过使用自定义BaseAdapter 来适配数据，进而显示到GridView中。

adapter是view和数据的桥梁。在一个ListView或者GridView中，不可能手动给每一个格子都新建一个view，所以这时候就需要Adapter的帮忙，它会帮我们自动绘制view并且填充数据。

GridView需要设置数据适配，就是添加需要显示的内容，所谓适配就是数据与视图之间的桥梁；而ListView有几种适配器：

- BaseAdapter：顾名思义，最基础的适配器，有四个抽象方法，可以方便的根据需求做出更改
- ArrayAdapter：继承BaseAdapter，实现了四个抽象方法，多用于只有文本的列表
- SimpleAdapter：继承BaseAdapter，实现了四个抽象方法，可以简单的实现列表样式（无法轮播）

对于Android程序员来说，BaseAdapter肯定不会陌生，灵活而优雅是BaseAdapter最大的特点。开发者可以通过构造BaseAdapter并搭载到ListView或者GridView这类多控件布局上面，实现软件所需要的布局效果。同时，BaseAdapter也是适配器里面最基础的一个类，其他的例如SimpleAdapter、ArrayAdapter都是直接或者间接继承BaseAdapter，所以说学好BaseAdapter基本就熟练掌握了适配器的使用了。

BaseAdapter有四个主要的抽象方法：

- getCount : 要绑定的item的数目，比如格子的数量
- getItem(int position) : 根据一个索引（位置）获得该位置的对象
- getItemId(int position)：返回该item的行id
- getView(int position, View convertView, ViewGroup parent)：是必须要实现的方法，该方法控制GridView中数据项的显示，方法中的convertView视图是被复用的视图，在实现时对其进行判断，如果为null，则新建视图，否则直接复用视图。

adapter先从getCount里确定数量，然后循环执行getView方法将条目一个一个绘制出来，所以必须重写的是getCount和getView方法。而getItem和getItemId是调用某些函数才会触发的方法，如果不需要使用可以暂时不修改。

MyBaseAdapter的构造函数要传入上下文和需要显示的数据的集合：

```java
    public MazeAdapter(Context context, List<GridItem> list, int rows) {
        this.mContext = context;
        this.mGridsList.addAll(list);
        mItemWidth = (ScreenUtil.getScreenWidth(mContext) - (rows - 1) * ScaleUtil.dip2px(mContext, 1)) / rows;
        mItemHeight = mItemWidth;
    }
```

getCount()返回我们的数据源的长度，要显示几个条目，我们就传入多长的数据源就可以了。

```java
@Override
    public int getCount() {
        return mGridsList.size();
    }
```

getItem(int position)根据一个索引（位置）获得该位置的对象

```java
@Override
    public Object getItem(int position) {
        return mGridsList.get(position);
    }
```

getItemId(int position)返回该item的行id

```java
    @Override
    public long getItemId(int position) {
        return position;
    }
```

getView(int position, View convertView, ViewGroup parent)：返回的是当前Item的视图。三个参数：当前Item的下标，当前视图（可复用），当前视图的父视图（可调整当前视图的宽高）

在初始显示的时候，每次显示一个item都调用一次getview方法但是每次调用的时候covertview为空（因为还没有旧的view），当显示完了之后。如果屏幕移动了之后，并且导致有些Item（也可以说是view）跑到屏幕外面，此时如果还有新的item需要产生，则这些item显示时调用的getview方法中的convertview参数就不是null，而是那些移出屏幕的view（旧view），我们所要做的就是将需要显示的item填充到这些回收的view（旧view）中去，最后注意convertview为null的不仅仅是初始显示的那些item，还有一些是已经开始移入屏幕但是还没有view被回收的那些item。

view的setTag和getTag方法其实很简单，在实际编写代码的时候一个view不仅仅是为了显示一些字符串、图片，有时我们还需要他们携带一些其他的数据以便我们对该view的识别或者其他操作。于是android 的设计者们就创造了setTag(Object)方法来存放一些数据和view绑定，我们可以理解为这个是view 的标签也可以理解为view 作为一个容器存放了一些数据。而这些数据我们也可以通过getTag() 方法来取出来。

我们通过convertview的setTag方法和getTag方法来将我们要显示的数据来绑定在convertview上。如果convertview 是第一次展示我们就创建新的Holder对象与之绑定，并在最后通过return convertview 返回，去显示；如果convertview 是回收来的那么我们就不必创建新的holder对象，只需要把原来的绑定的holder取出加上新的数据就行了。

```java
@Override
    public View getView(int position, View convertView, ViewGroup parent) {
        ViewHolder viewHolder;
        if (convertView == null) {
            //LayoutInflater是一个用于将xml布局文件加载为View或者ViewGroup对象的工具，我们可以称之为布局加载器。
            //用LayoutInflater的inflate方法就可以将我们的item布局绘制出来
            convertView = LayoutInflater.from(mContext).inflate(R.layout.item_grid, null);
            AbsListView.LayoutParams params = new AbsListView.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, mItemHeight);
            convertView.setLayoutParams(params);
            viewHolder = new ViewHolder();
            viewHolder.item = convertView.findViewById(R.id.item);
            viewHolder.icon = convertView.findViewById(R.id.icon);
            viewHolder.perfect = convertView.findViewById(R.id.perfect);
            convertView.setTag(viewHolder);
        } else {
            viewHolder = (ViewHolder) convertView.getTag();
        }
        GridItem grid = mGridsList.get(position);
        viewHolder.icon.setVisibility(View.GONE);
        viewHolder.perfect.setVisibility(View.GONE);
        viewHolder.item.setBackgroundResource(R.drawable.shape_maze_normal);
        if (grid.barrier) {
            viewHolder.item.setBackgroundResource(R.drawable.shape_maze_disable);
        } else if (grid.start) {
            viewHolder.icon.setVisibility(View.VISIBLE);
            viewHolder.icon.setImageResource(R.drawable.start);
        } else if (grid.end) {
            viewHolder.icon.setVisibility(View.VISIBLE);
            viewHolder.icon.setImageResource(R.drawable.end);
        } else if (grid.result) {
            viewHolder.perfect.setVisibility(View.VISIBLE);
            viewHolder.perfect.setImageResource(R.drawable.path);
        } else if (grid.selected) {
            viewHolder.icon.setVisibility(View.VISIBLE);
            viewHolder.icon.setImageResource(R.drawable.foot);
        } else {
            viewHolder.item.setBackgroundResource(R.drawable.shape_maze_normal);
        }
        return convertView;
    }

```

使用一个ViewHolder来缓存Item布局里面的控件，以后编写BaseAdapter照着这个模板写就对了，另外这个修饰ViewHolder的 static，关于是否定义成静态，跟里面的对象数目是没有关系的，加静态是为了在多个地方使用这个 Holder的时候，类只需加载一次

```java
static class ViewHolder {
    public RelativeLayout item;
    public ImageView icon;
    public ImageView perfect;
}
```

### 初始化地图

准备数据源

```java
private List<GridItem> mList = new ArrayList<>();
```

为数据源设置适配器

```java
private MazeAdapter mAdapter;
```

自定义BaseAdapter适配器来适配数据，用于将静态数据映射到xml文件中定义好的视图当中；MList是数据源

```java
mAdapter = new MazeAdapter(getApplicationContext(), mList, MAZE.length);
```

将适配过后的数据显示在GridView 上

```java
mGridView.setAdapter(mAdapter);
```

使用二维数组来表示地图，MAZE数组表示地图中的数据，1代表障碍物，0代表非障碍物（包括起点和终点）

```java
public static int[][] MAZE = new int[Config.ROWS][Config.COLUMNS];
```

我们使用java内置的Random类生成随机数，从而生成随机的地图。调用这个Math.Random()函数能够返回带正号的double值，该值大于等于0.0且小于1.0，即取值范围是[0.0,1.0)的左闭右开区间，返回值是一个伪随机选择的数，在该范围内（近似）均匀分布。

下面Random()的两种构造方法：

- Random()：创建一个新的随机数生成器，默认当前系统时间的毫秒数作为种子数:Random r1 = new Random();
- Random(long seed)：使用单个 long 种子创建一个新的随机数生成器。在构造Random对象的时候指定种子（这里指定种子有何作用，请接着往下看），如：Random r1 = new Random(20);

```java
private void init() {
        //随机生成起点和终点
        //Random类中的nextInt(n)方法生成0（包括）到n（不包括）之间的随机整数
        mStartGrid = new Grid(new Random().nextInt(Config.ROWS), 0);
        mEndGrid = new Grid(new Random().nextInt(Config.ROWS), Config.COLUMNS - 1);
        //重置所有的按钮的状态和数据源
        reset();
        //线程一：执行mInitMaze任务，初始化地图
        MazeManager.get().execute(mInitMaze);
    }
```

重置所有按钮的状态：

```java
    private void reset() {
        mNext.setEnabled(false);
        mStepCount = 0;
        mStep.setText(mStepCount + "");
        mShowResult = false;
        //显示最短路径的按钮变灰，表示可以点击
        mShow.setImageResource(R.drawable.hide);
        //当前所在的格子
        mCurrentGrid = new Grid(mStartGrid.x, mStartGrid.y);
        //清空数据源
        mList.clear();
    }
```

定义Runnable的匿名类，重写run接口，实例化Runnable子类的对象。 此对象可作为实际参数传入线程池中执行，Java中实现多线程有两种方法：继承Thread类、实现Runnable接口，在程序开发中只要是多线程，永远以实现Runnable接口为主，因为实现Runnable接口相比继承Thread类有如下优势：

1、可以避免由于Java的单继承特性而带来的局限；

2、增强程序的健壮性，代码能够被多个线程共享，代码与数据是独立的；

3、适合多个相同程序代码的线程区处理同一资源的情况。

```java
//初始化地图
private Runnable mInitMaze = new Runnable() {
    @Override
    public void run() {
        boolean initMaze = false;
        //生成地图，如果生成的地图不合规（起点到终点没有通路，或距离太短），则重新生成
        while (!initMaze) {
            initMaze = initMaze();
        }
        mHandler.sendEmptyMessage(MSG_INIT_MAZE_COMPLETE);
    }
};
```

初始化地图，MAZE数组表示地图中的数据，1代表障碍物，0代表非障碍物（包括起点和终点），只有当起点到终点存在通路，并且距离大于30时，才是合格的地图，返回true

```java
//初始化地图
private boolean initMaze() {

    int[][] temp = new int[Config.ROWS][Config.COLUMNS];

    Random random = new Random();
    //MAZE数组表示地图中的数据，1代表障碍物，0代表非障碍物（包括起点和终点）
    for (int i = 0; i < Config.ROWS; i++) {
        for (int j = 0; j < Config.COLUMNS; j++) {
            if (i == mStartGrid.x && j == mStartGrid.y) {
                temp[i][j] = 0;
            } else if (i == mEndGrid.x && j == mEndGrid.y) {
                temp[i][j] = 0;
            } else {
                temp[i][j] = random.nextInt(2);
            }
        }
    }

    MAZE = temp;
    //计算起点到终点的路径
    Grid resultGrid = SearchUtil.aStarSearch(mStartGrid, mEndGrid, MAZE);
    //只有当起点到终点存在通路，并且距离大于30时，才是合格的地图，返回true
    return resultGrid != null && calcStep(resultGrid) > 30;

}
```

计算最短路径的距离

```java
//计算最短路径的距离
private int calcStep(Grid grid) {
    int step = 0;
    Grid temp = grid;
    while (temp != null) {
        temp = temp.parent;
        step++;
    }
    return step;
}
```

### 网格的点击事件处理

在实际的应用当中，我们需要对用户的操作进行监听，即需要知道用户点击了哪一个格子。

在网格控件GridView中，常用的事件监听器有两个：OnItemSelectedListener和OnItemClickListener。其中，OnItemSelectedListener用于项目选择事件监听，OnItemClickListener用于项目点击事件监听。

要实现这两个事件监听很简单，继承OnItemSelectedListener和OnItemClickListener接口，并实现其抽象方法即可。其中，需要实现的OnItemClickListener接口的抽象方法如下：

```java
public void onItemClick(AdapterView<?> parent, View view, int position, long id);
```

setEnabled为false，该控件将不再响应点击、触摸以及键盘事件等，处于完全被禁用的状态，并且该控件会被重绘。

```java
// 设置网格中的item点击事件处理，为网格控件设置监听事件
        mGridView.setOnItemClickListener(new AdapterView.OnItemClickListener() {
            //position是被点击的item在网格数据源中的位置
            @Override
            public void onItemClick(AdapterView<?> parent, View view, int position, long id) {
                //如果正在展示最短路径，则不做任何操作，直接返回
                if (mShowResult) {
                    return;
                }
                //如果点击的item是合法的，则改变位置
                if (isValidGrid(mList.get(position))) {
                    //前进
                    //先将点击的item在数据源mList中的位置position转换成在网格中的位置
                    //再在该位置实例化一个Grid对象，表示当前所在的位置
                    Grid grid = new Grid(mList.get(position).x, mList.get(position).y);
                    //设置当前位置的父节点
                    grid.parent = mCurrentGrid;
                    mCurrentGrid = grid;
                    //如果到达终点
                    if (mCurrentGrid.x == mEndGrid.x && mCurrentGrid.y == mEndGrid.y) {
                        Toast.makeText(getApplicationContext(), getApplicationContext().getString(R.string.success), Toast.LENGTH_SHORT).show();
                        //如果到达终点，将进入下一关的按钮设置为可用
                        mNext.setEnabled(true);
                        return;
                    }
                    //将该点标记为已经过
                    mList.get(position).selected = true;
                    //更新适配器的数据源
                    mAdapter.update(mList);
                    //步数增加
                    mStepCount++;
                    mStep.setText(mStepCount + "");

                } else if (mCurrentGrid.parent != null) {
                    //后退
                    if (mCurrentGrid.parent.x == mList.get(position).x && mCurrentGrid.parent.y == mList.get(position).y) {
                        back();
                    }
                }
            }
        });
```

判断点击的位置是否合法，点击障碍物不能移动，点击当前点不能移动，点击已经走过的点也不能移动，只有点击相邻的点才能移动

```java
private boolean isValidGrid(GridItem gridItem) {
    if (gridItem.barrier) {
        //障碍
        return false;
    }
    if (gridItem.x == mCurrentGrid.x && gridItem.y == mCurrentGrid.y) {
        //当前节点
        return false;
    }
    if (mCurrentGrid.parent != null && mCurrentGrid.parent.x == gridItem.x && mCurrentGrid.parent.y == gridItem.y) {
        //后退
        return false;
    }

    if (isContain(gridItem)) {
        //已经存在
        return false;
    }

    return isNeighbor(gridItem);
}

private boolean isContain(GridItem gridItem) {
    Grid temp = mCurrentGrid;
    while (temp != null) {
        if (temp.x == gridItem.x && temp.y == gridItem.y) {
            return true;
        }
        temp = temp.parent;
    }
    return false;
}

private boolean isNeighbor(GridItem gridItem) {
    if (gridItem.x == mCurrentGrid.x && gridItem.y == mCurrentGrid.y - 1) {
        //上
        return true;
    }
    if (gridItem.x == mCurrentGrid.x && gridItem.y == mCurrentGrid.y + 1) {
        //下
        return true;
    }
    if (gridItem.y == mCurrentGrid.y && gridItem.x == mCurrentGrid.x - 1) {
        //左
        return true;
    }
    if (gridItem.y == mCurrentGrid.y && gridItem.x == mCurrentGrid.x + 1) {
        //右
        return true;
    }
    return false;
}
```

### 按钮的点击事件处理

对按钮进行监听：

```java
//设置按钮的点击事件处理，为按钮设置监听事件
mBack.setOnClickListener(this);
mReload.setOnClickListener(this);
mShow.setOnClickListener(this);
mNext.setOnClickListener(this);
```

按钮的点击事件处理

```java
//按钮的点击事件处理
@Override
public void onClick(View v) {
    switch (v.getId()) {
        //后退一步
        case R.id.iv_back:
            back();
            break;
        //重新加载地图，不需要重新初始化，只需要重置地图的数据源
        case R.id.iv_reload:
            initData();
            break;
        //展示答案
        case R.id.iv_show:
            //改变按钮状态
            mShowResult = !mShowResult;
            show();
            break;
        //进入下一关
        case R.id.iv_next:
            init();
            break;
        default:
            break;
    }
}
```

#### 后退



```java
private void back() {
    //如果当前位置没有父节点，无法后退，直接返回
    if (mCurrentGrid.parent == null) {
        return;
    }
    //后退至父节点所在位置
    mCurrentGrid = mCurrentGrid.parent;
    //用后退的点重新获取path
    ArrayList<Grid> path = new ArrayList<>();
    Grid temp = mCurrentGrid;
    while (temp != null) {
        path.add(new Grid(temp.x, temp.y));
        temp = temp.parent;
    }
    // 更新数据源中的道路
    for (int i = 0; i < MAZE.length; i++) {
        for (int j = 0; j < MAZE[0].length; j++) {
            mList.get(i * MAZE[0].length + j).result = false;
            mList.get(i * MAZE[0].length + j).selected = false;
            if (SearchUtil.containGrid(path, i, j)) {
                //将走过的路标记为true
                mList.get(i * MAZE[0].length + j).selected = true;
            }
        }
    }
    //更新适配器
    mAdapter.update(mList);
    mStepCount++;
    mStep.setText(mStepCount + "");
}
```

#### 重新加载地图

重新加载地图，不需要重新初始化，只需要重置地图的数据源

```java
public void initData() {
    reset();
    //线程二：初始化数据
    MazeManager.get().execute(mInitListData);

}
private Runnable mInitListData = new Runnable() {
    @Override
    public void run() {
        //将二维数组中的地图数据转换成适配器中的数据源
        for (int i = 0; i < MAZE.length; i++) {
            for (int j = 0; j < MAZE[0].length; j++) {
                GridItem grid = new GridItem();
                grid.x = i;
                grid.y = j;
                //1表示障碍物
                if (MAZE[i][j] == 1) {
                    grid.barrier = true;
                } else if (grid.x == mStartGrid.x && grid.y == mStartGrid.y) {
                    grid.start = true;
                } else if (grid.x == mEndGrid.x && grid.y == mEndGrid.y) {
                    grid.end = true;
                }
                //从左到右，从上到下将数据加入数据源
                mList.add(grid);
            }
        }
        mHandler.sendEmptyMessage(MSG_INIT_LIST_DATA_COMPLETE);
    }
};

public void updateGridView() {
    mGridView.setNumColumns(MAZE[0].length);
    mAdapter.update(mList);
}
```

#### 显示最短路径

```java
private void show() {
    //改变按钮的状态
    if (mShowResult) {
        mShow.setImageResource(R.drawable.show);
    } else {
        mShow.setImageResource(R.drawable.hide);
    }
    //显示最短路径
    MazeManager.get().execute(mShowPath);

}
private Runnable mShowPath = new Runnable() {
    @Override
    public void run() {
        // 搜索迷宫终点
        Grid resultGrid = SearchUtil.aStarSearch(mStartGrid, mEndGrid, MAZE);
        ArrayList<Grid> path = new ArrayList<>();
        while (resultGrid != null) {
            path.add(new Grid(resultGrid.x, resultGrid.y));
            resultGrid = resultGrid.parent;
        }
        // 改变数据源，显示路径或隐藏路径
        for (int i = 0; i < MAZE.length; i++) {
            for (int j = 0; j < MAZE[0].length; j++) {
                if (SearchUtil.containGrid(path, i, j)) {
                    mList.get(i * MAZE[0].length + j).result = mShowResult;
                }
            }
        }
        mHandler.sendEmptyMessage(MSG_MAZE_SEARCH_COMPLETE);
    }
};
```

##### 最短路径算法

最短路问题是图论理论的一个经典问题。寻找最短路径就是在指定网络中两结点间找一条距离最小的路。最短路不仅仅指一般地理意义上的距离最短,还可以引申到其它的度量,如时间、费用、线路容量等。

最短路径算法的选择与实现是通道路线设计的基础,最短路径算法是计算机科学与地理信息科学等领域的研究热点,很多网络相关问题均可纳入最短路径问题的范畴之中。经典的图论与不断发展完善的计算机数据结构及算法的有效结合使得新的最短路径算法不断涌现。

对最短路问题的研究早在上个世纪60年代以前就卓有成效了,其中对赋权图的有效算法是由荷兰著名计算机专家E.W.Dijkstra在1959年首次提出的,该算法能够解决两指定点间的最短路,也可以求解图G中一特定点到其它各顶点的最短路。后来海斯在Dijkstra算法的基础之上提出了海斯算法。但这两种算法都不能解决含有负权的图的最短路问题。因此由Ford提出了Ford算法,它能有效地解决含有负权的最短路问题。 

算法具体的形式包括：

- 确定起点的最短路径问题 - 也叫单源最短路问题，即已知起始结点，求最短路径的问题。在边权非负时适合使用Dijkstra算法，若边权为负时则适合使用Bellman-ford算法或者SPFA算法。
- 确定终点的最短路径问题 - 与确定起点的问题相反，该问题是已知终结结点，求最短路径的问题。在无向图中该问题与确定起点的问题完全等同，在有向图中该问题等同于把所有路径方向反转的确定起点的问题。
- 确定起点终点的最短路径问题 - 即已知起点和终点，求两结点之间的最短路径。
- 全局最短路径问题 - 也叫多源最短路问题，求图中所有的最短路径。适合使用Floyd-Warshall算法。

我们学习过BFS，知道BFS可以在无权边的图中寻找最短路径。而实际的地图是一种有权图，地图中的每个条路的长度都不一样，对于有权图我们可以使用Dijkstra。

Dijkstra本质上是一种贪心算法，利用优先队列，存储图中的点和离源点的距离，从源点开始，优先访问距离源点最近的点，然后对周围没有访问过的点进行松弛，更新被松弛的点和源点的距离。

我们可以将Dijkstra的过程看作是一棵树，图中每一个点都是其中的一个节点，它的相邻的点（neighbor）对应它的子节点。叶结点都在优先队列中，而所有的内部节点都已经处理过了，从优先队列中删除。每一次以一个叶结点为中心向外探索，将该叶结点从优先队列中删除，将与它相邻的没有处理过的节点加入优先队列，变成新的叶结点。

假设 `d`是起点到终点的距离，我们发现Dijkstra算法需要访问所有到起点的距离小于等于 `d`的点，但是在已经知道终点的方向的情况下，我们可以将方向作为探索的启发，朝着终点的方向搜索，从而避免很多不必要的探索，这就是A*搜索算法。

A\*搜索算法（A* search algorithm）是一种在图形平面上，有多个节点的路径，求出最低通过成本的算法。常用于游戏中的NPC的移动计算，或网络游戏的BOT的移动计算上。

该算法综合了最良优先搜索和Dijkstra算法的优点：在进行启发式搜索提高算法效率的同时，可以保证找到一条最优路径（基于评估函数）。

在此算法中，如果以g(n)表示从起点到任意顶点n的实际距离，h(n)表示任意顶点n到目标顶点的估算距离（根据所采用的评估函数的不同而变化），那么A*算法的估算函数为：f(n)=h(n)+g(n)

这个公式遵循以下特性：

- 如果g(n)为0，即只计算任意顶点n到目标的评估函数h(n)，而不计算起点到顶点n的距离，则算法转化为使用贪心策略的最良优先搜索，速度最快，但可能得不出最优解；
- 如果h(n)不大于顶点n到目标顶点的实际距离，则一定可以求出最优解，而且h(n)越小，需要计算的节点越多，算法效率越低，常见的评估函数有——欧几里得距离、曼哈顿距离、切比雪夫距离；
- 如果h(n)为0，即只需求出起点到任意顶点n的最短路径g(n)，而不计算任何评估函数h(n)，则转化为单源最短路径问题，即Dijkstra算法，此时需要计算最多的顶点；

A*搜索算法只需要在Dijkstra的基础上修改一行代码就行了。Dijkstra的优先队列的优先级比较仅仅是当前点到起点的距离，而A *搜索算法的优先队列的优先级则是由当前点到起点的距离和当前点到终点的距离共同决定。

综上，A*算法的过程如下：

- 首先定义在 shortestPath中使用的搜索单位，Grid类，使用Grid的目的是方便从终点回溯，找到路径。其中属性为：
  - 当前节点在网格中的坐标
  - 指向当前点的父节点，以便在找到终点后可以获取从终点到起点的路径
  - 当前点到起点的距离
  - 当前点在优先队列中的优先级
- shortestPath算法的过程如下：
  - 我们需要维护一个优先队列，该队列的内容就是 Grid对象，可以从该队列中获取最佳的移动节点
  - 将优先队列中的最佳的节点移除，标记被移除的节点
  - 如果该节点是终点，结束查找
  - 如果该节点不是终点，那么以该节点为中心，探索它的相邻节点，将没有标记的相邻节点创建 SearchNode对象，将它们加入队列
  - 再从队列中获取最佳的节点，如此往复
- 优先队列的优先级比较机制如下：由当前点到起点的距离和当前点到终点的距离共同决定。

```java
public class SearchUtil {

    private static int[][] MAZE;

    /**
     * A*寻路主逻辑
     *
     * @param start 起点
     * @param end   终点
     * @param maze  迷宫
     * @return
     */
    public static Grid aStarSearch(Grid start, Grid end, int[][] maze) {
        MAZE = maze;
        ArrayList<Grid> openList = new ArrayList<>();
        ArrayList<Grid> closeList = new ArrayList<>();
        // 把起点加入openList
        openList.add(start);
        // 主循环，每一轮检查一个当前方格节点
        while (openList.size() > 0) {
            // 在openList中查找F值最小的节点，将其作为当前方格节点
            Grid currentGrid = findMinGrid(openList);
            // 将当前方格节点从openList中移除
            openList.remove(currentGrid);
            // 将当前方格节点放入closeList
            closeList.add(currentGrid);
            // 找到所有邻近节点
            ArrayList<Grid> neighbors = findNeighbors(currentGrid, openList, closeList);
            for (Grid grid : neighbors) {
                if (!openList.contains(grid)) {
                    // 邻近节点不在openList中，标记父节点、G、H、F，并放入openList
                    grid.initGrid(currentGrid, end);
                    openList.add(grid);
                }
            }
            // 如果终点在openList中，直接返回终点格子
            for (Grid grid : openList) {
                if (grid.x == end.x && grid.y == end.y) {
                    return grid;
                }
            }
        }
        // openList用尽，还找不一终点，说明终点不可到达，返回null
        return null;
    }

    private static Grid findMinGrid(ArrayList<Grid> openList) {
        Grid tempGrid = openList.get(0);
        for (Grid grid : openList) {
            if (grid.priority < tempGrid.priority) {
                tempGrid = grid;
            }
        }
        return tempGrid;
    }

    private static ArrayList<Grid> findNeighbors(Grid grid, ArrayList<Grid> openList, ArrayList<Grid> closeList) {
        ArrayList<Grid> gridList = new ArrayList<>();
        if (isValidGrid(grid.x, grid.y - 1, openList, closeList)) {
            gridList.add(new Grid(grid.x, grid.y - 1));
        }
        if (isValidGrid(grid.x, grid.y + 1, openList, closeList)) {
            gridList.add(new Grid(grid.x, grid.y + 1));
        }
        if (isValidGrid(grid.x - 1, grid.y, openList, closeList)) {
            gridList.add(new Grid(grid.x - 1, grid.y));
        }
        if (isValidGrid(grid.x + 1, grid.y, openList, closeList)) {
            gridList.add(new Grid(grid.x + 1, grid.y));
        }
        return gridList;
    }

    private static boolean isValidGrid(int x, int y, ArrayList<Grid> openList, ArrayList<Grid> closeList) {
        // 是否超过边界
        if (x < 0 || x >= MAZE.length || y < 0 || y >= MAZE[0].length) {
            return false;
        }
        // 是否有障碍物
        if (MAZE[x][y] == 1) {
            return false;
        }
        // 是否已经在openList中
        if (containGrid(openList, x, y)) {
            return false;
        }
        // 是否已经在closeList中
        if (containGrid(closeList, x, y)) {
            return false;
        }
        return true;
    }
}
```

#### 进入下一关

随机生成一个新的地图

```java
private void init() {
    //随机生成起点和终点
    //Random类中的nextInt(n)方法生成0（包括）到n（不包括）之间的随机整数
    mStartGrid = new Grid(new Random().nextInt(Config.ROWS), 0);
    mEndGrid = new Grid(new Random().nextInt(Config.ROWS), Config.COLUMNS - 1);
    //重置所有的按钮的状态和数据源
    reset();
    //线程一：执行mInitMaze任务，初始化地图
    MazeManager.get().execute(mInitMaze);
}

private void reset() {
    mNext.setEnabled(false);
    mStepCount = 0;
    mStep.setText(mStepCount + "");
    mShowResult = false;
    //显示最短路径的按钮变灰，表示可以点击
    mShow.setImageResource(R.drawable.hide);
    //当前所在的格子
    mCurrentGrid = new Grid(mStartGrid.x, mStartGrid.y);
    //清空数据源
    mList.clear();
}

//定义Runnable的匿名类，重写run接口，实例化Runnable子类的对象。
// 此对象可作为实际参数传入线程池中执行，MazeManager.get().execute(mInitMaze);
//初始化地图
private Runnable mInitMaze = new Runnable() {
    @Override
    public void run() {
        boolean initMaze = false;
        //生成地图，如果生成的地图不合规（起点到终点没有通路，或距离太短），则重新生成
        while (!initMaze) {
            initMaze = initMaze();
        }
        mHandler.sendEmptyMessage(MSG_INIT_MAZE_COMPLETE);
    }
};
//初始化地图
private boolean initMaze() {

    int[][] temp = new int[Config.ROWS][Config.COLUMNS];

    Random random = new Random();
    //MAZE数组表示地图中的数据，1代表障碍物，0代表非障碍物（包括起点和终点）
    for (int i = 0; i < Config.ROWS; i++) {
        for (int j = 0; j < Config.COLUMNS; j++) {
            if (i == mStartGrid.x && j == mStartGrid.y) {
                temp[i][j] = 0;
            } else if (i == mEndGrid.x && j == mEndGrid.y) {
                temp[i][j] = 0;
            } else {
                temp[i][j] = random.nextInt(2);
            }
        }
    }

    MAZE = temp;
    //计算起点到终点的路径
    Grid resultGrid = SearchUtil.aStarSearch(mStartGrid, mEndGrid, MAZE);
    //只有当起点到终点存在通路，并且距离大于30时，才是合格的地图，返回true
    return resultGrid != null && calcStep(resultGrid) > 30;

}
//计算最短路径的距离
private int calcStep(Grid grid) {
    int step = 0;
    Grid temp = grid;
    while (temp != null) {
        temp = temp.parent;
        step++;
    }
    return step;
}
```