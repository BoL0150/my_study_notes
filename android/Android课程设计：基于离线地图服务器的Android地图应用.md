# Android开发课程设计：基于离线地图服务器的Android地图应用

此项目的灵感来源于伯克利cs61b的Project3：

cs61b的官网地址：[Project 3: Bear Maps](https://sp18.datastructur.es/materials/proj/proj3/proj3)

我的实验记录：[cs61b实验记录（八）project 3](https://blog.csdn.net/qq_45698833/article/details/116036624?spm=1001.2014.3001.5501)

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/61BC8C29FAE46725FB7A8FA2DE0F4A13.png" alt="img" style="zoom: 25%;" />

此应用实现了一个Android地图应用，以及一个提供必要服务的离线地图服务器。

- Android端：

  使用Mapbox提供的API完成地图的展示和用户界面的控制。

- 服务器端：

  使用Spark框架搭建服务器，自己实现算法，提供Android端的兴趣点（POI）的查询、地点搜索的自动补全、导航以及最短路径查找的服务。

具体流程：Android客户端由MapBox提供的API完成地图的展示和用户界面的控制，当用户需要搜索、导航、查询路径等操作时，由Android客户端发送URL给服务器，请求相应的服务，Android客户端将请求的结果以各种方式展示到用户界面上。

## 服务器端

服务器端使用一个名为`spark`的java框架，这里的 `spark` 并非是大数据相关的 `apache-spark`，而是一个创建Web应用程序的微框架。Spark 框架是为快速开发而构建的简单，轻量级的 Java Web 框架。 它的灵感来自流行的 Ruby 微框架 Sinatra 。Spark 广泛使用 Java 8 的 lambda 表达式，这使 Spark 应用不再那么冗长。 与其他 Java Web 框架相比，Spark 不使用大量的 XML 文件或注释。

spark是一个非常轻量级的web框架，若你不需要那么多的功能，只是提供简单的web接口，那么spark是个非常适合的选择。与我们所熟知的占内存较大Spring的web的框架相比，使用spark框架的项目启动速度飞升。

### 1、创建一个Maven工程

#### Maven介绍

我们在做一个java项目时需要如下几个步骤：

1. 我们需要确定引入哪些依赖包，例如，如果我们需要用到library-18sp，我们就必须把library-18sp的jar包放入classpath。这就是依赖包的管理。
2. 我们要确定项目的目录结构。例如，`src`目录存放Java源码，`resources`目录存放配置文件，`bin`目录存放编译生成的`.class`文件。
3. 我们还需要配置环境，例如JDK的版本，编译打包的流程，当前代码的版本号。

这些工作难度不大，但是非常琐碎且耗时。如果每一个项目都自己搞一套配置，肯定会一团糟。我们需要的是一个标准化的Java项目管理和构建工具。

Maven就是是专门为Java项目打造的管理和构建工具，它的主要功能有：

- 提供了一套标准化的项目结构；
- 提供了一套标准化的构建流程；
- 提供了一套依赖管理机制。

#### 配置

1. 首先我们可以在Intellij idea中直接创建一个Maven项目，[具体流程](http://sparkjava.com/tutorials/maven-setup)

2. 在新创建的Maven项目中找到项目描述文件`pom.xml`，这是Maven中最重要的文件。

   <img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210519123332849.png" alt="image-20210519123332849" style="zoom: 50%;" />

   在`pom.xml`文件中，`groupId`类似于Java的包名，通常是公司或组织名称，`artifactId`类似于Java的类名，通常是项目名称，再加上`version`，一个Maven工程就是由`groupId`，`artifactId`和`version`作为唯一标识。

3. 在其中 `<dependencies>` 添加如下代码，spark的依赖就配置完成了

   ```xml
   <dependency>
       <groupId>com.sparkjava</groupId>
       <artifactId>spark-core</artifactId>
       <version>2.7.2</version>
   </dependency>
   ```

4. 由于服务器端需要给客户端提供如最短路径、POI等信息，这些信息以`JSON`的格式传递给客户端，客户端才能对这些信息加以使用。

   - `JSON`是一种轻量级的资料交换语言，该语言以易于让人阅读的文字为基础，用来传输由属性值或者序列性的值组成的数据对象。

   而在服务器端，我们如果想要将java对象转为`JSON`需要借助`Gson`来实现。

   - **Gson**（又称Google Gson）是Google公司发布的一个开放源代码的java库，主要用途为序列化Java对象为JSON字符串，或反序列化JSON字符串成Java对象。

   所以我们还需要在`pom.xml`中添加Gson的依赖：

   ```xml
   <dependency>
       <groupId>com.google.code.gson</groupId>
       <artifactId>gson</artifactId>
       <version>2.8.2</version>
   </dependency>
   ```

5. 为了对实现的算法进行测试，我们还需要添加JUnit的依赖：

   ```xml
   <dependency>
       <groupId>junit</groupId>
       <artifactId>junit</artifactId>
       <version>4.12</version>
   </dependency>
   ```

### 2、路网数据的处理

想要提供这些服务，服务器必须要获取路网数据，然后对数据进行解析，再实现功能。

我们选择OSM（**OpenStreetMap**）的xml格式的数据（详见我的另一篇博客：[OSM数据的获取方法](https://blog.csdn.net/qq_45698833/article/details/117026243?spm=1001.2014.3001.5501)），**OpenStreetMap**是一个建构自由内容之网上地图协作计划，目标是创造一个内容自由且能让所有人编辑的世界地图，并且让一般的移动设备有方便的导航方案。OSM是一个开放的平台，由志愿者提供开源的地理数据。我们可以自由地，创造性地去使用这些数据。

OSM的数据是XML格式的，主要组成为：

- `tag`：每一个tag由两个部分组成，`key`和 `value`，tag用来描述地图中元素的特性（比如node、way）。

- `node`：OSM数据模型中的核心元素，是空间中的一个单一点，由**latitude**, **longitude** 和**node id**三个属性定义，分别表示地图上某一点的纬度、经度和唯一的id。

  - 如果一个node的tag有key="name"，那么这个node就是一个地点，可以对它进行搜索

  ![node](https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/node_xml.jpg)

- `way`：路，由一连串的点定义，使用 `nd`和属性 `ref`引用点的id。`way`中的 `tag`表明了这条路的类型，如果它有一个"highway"的key，我们才认为这个way是有效的，否则我们不会将这个way加入图中。

  ![way](https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/way_xml.jpg)

OSM文件中先出现所有的点，然后再出现way，通过对之前已经出现的点的引用，将它们连接成路。

1. 首先我们选择无向有权图 `GraphDB`类来存储路网数据中的交点（vertex）和路（edge）的信息。图中的每一个节点代表地图的一个交点，每一条边代表地图的一条路。

   ```java
   /**
    * 用来存储路网数据中的交点（vertex）和路（edge）的信息的图
    * 使用GraphBuildingHandler类将XML文件转化成一个graph
    */
   //解析xml文件，创建一个Graph
   public class GraphDB {
   
       //每个序号对应的点，在调用了clean方法后，就剩下图中所有相连的点，该Map在搜索最短路径时使用
       public Map<Long, point> nodes = new HashMap<>();
       //邻接表，每个点相邻的点，在搜索最短路径时使用
       private Map<Long, ArrayList<Long>> adjNode = new HashMap<>();
       //一个名字可能对应多个点，存放的是cleanString
       private Map<String, ArrayList<Long>> names = new HashMap<>();
       //邻接表，每个点相邻的边，在由最短路径获得对应的导航信息时使用
       private Map<Long, ArrayList<Edge>> adjEdge = new HashMap<>();
       //地图中所有有名字的点，不管是否相连。在搜索location时使用
       public Map<Long, point> locations = new HashMap<>();
       //Trie，字符串匹配时使用
       private Trie<Long> trie = new Trie<>();
       //KDTree，寻找地图上距离最近的点时使用
       private KDTree kdTree;
       public static class Edge {
           //边的端点
           private Long v;
           private Long w;
           //边的权重
           private double weight;
           //边的名字
           private String name;
   
           public Edge(Long v, Long w, double weight, String name) {
               this.v = v;
               this.w = w;
               this.weight = weight;
               this.name = name;
           }
           //该边的一个端点
           public Long either() {
               return v;
           }
           //该边的另一个端点
           public Long other(Long vertex) {
               return vertex.equals(v) ? w : v;
           }
   
           public double getWeight() {
               return weight;
           }
   
           public String getName() {
               return name;
           }
       }
       /**
        * 去掉字符串中除了汉字、字母、数字之外的内容，以便自动补全
        * @param s Input string.
        * @return Cleaned string.
        */
       static String cleanString(String s) {
           //去掉除了汉字、字母、数字之外的内容
           return s.replaceAll("[^A-Za-z0-9\u4e00-\u9fa5]", "").toLowerCase();
       }
   
       /**
        * 将没有连接的、孤立的点从Graph中移除出去，这些点在寻路时、导航时都没有用，但是在寻找兴趣点时有用
        * 所以我们需要将它们从nodes和adjNode中清除，但是需要在locations和names中保存这些点
        */
       private void clean() {
           Iterator<Map.Entry<Long, ArrayList<Long>>> it = adjNode.entrySet().iterator();
           while (it.hasNext()) {
               Map.Entry<Long, ArrayList<Long>> entry = it.next();
               if (entry.getValue().isEmpty()) {
                   //只清理nodes和adjNode
                   nodes.remove(entry.getKey());
                   it.remove();
               }
           }
       }
       /**
        * 返回可迭代的图中所有的顶点id
        */
       Iterable<Long> vertices() {
           //YOUR CODE HERE, this currently returns only an empty list.
           return nodes.keySet();
       }
   
       /**
        * 返回所有与顶点v相邻的顶点id
        */
       Iterable<Long> adjacent(Long v) {
           validateVertex(nodes.get(v));
           return adjNode.get(v);
       }
   
       /**
        * @param v
        * @return 顶点v的邻边
        */
       Iterable<Edge> neighbors(Long v) {
           return adjEdge.get(v);
       }
   
       /**
        * 返回顶点v和顶点w的大圆距离（great-circle distance）
        */
       double distance(Long v, Long w) {
           return distance(lon(v), lat(v), lon(w), lat(w));
       }
   
       static double distance(double lonV, double latV, double lonW, double latW) {
           double phi1 = Math.toRadians(latV);
           double phi2 = Math.toRadians(latW);
           double dphi = Math.toRadians(latW - latV);
           double dlambda = Math.toRadians(lonW - lonV);
   
           double a = Math.sin(dphi / 2.0) * Math.sin(dphi / 2.0);
           a += Math.cos(phi1) * Math.cos(phi2) * Math.sin(dlambda / 2.0) * Math.sin(dlambda / 2.0);
           double c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
           return 3963 * c;
       }
   
       /**
        * 返回两点间的initial bearing
        */
       double bearing(Long v, Long w) {
           return bearing(lon(v), lat(v), lon(w), lat(w));
       }
   
       static double bearing(double lonV, double latV, double lonW, double latW) {
           double phi1 = Math.toRadians(latV);
           double phi2 = Math.toRadians(latW);
           double lambda1 = Math.toRadians(lonV);
           double lambda2 = Math.toRadians(lonW);
   
           double y = Math.sin(lambda2 - lambda1) * Math.cos(phi2);
           double x = Math.cos(phi1) * Math.sin(phi2);
           x -= Math.sin(phi1) * Math.cos(phi2) * Math.cos(lambda2 - lambda1);
           return Math.toDegrees(Math.atan2(y, x));
       }
   
       /**
        * Gets the longitude of a vertex.
        */
       double lon(Long v) {
           validateVertex(nodes.get(v));
           return nodes.get(v).lon;
       }
   
       /**
        * Gets the latitude of a vertex.
        */
       double lat(Long v) {
           validateVertex(nodes.get(v));
           return nodes.get(v).lat;
       }
   
       String getName(Long v) {
           if (nodes.get(v).name == null) {
               throw new IllegalArgumentException();
           }
           return nodes.get(v).name;
       }
   
       void addName(Long id, double lon, double lat, String locationName) {
           //将名字统一转换成小写
           String cleanedName = cleanString(locationName);
           if (!names.containsKey(cleanedName)) {
               names.put(cleanedName, new ArrayList<>());
           }
           //names中存放的是cleanString和id列表，方便我们根据cleanString获取对应的所有点的id
           names.get(cleanedName).add(id);
           //Node对象中的name属性存放的是真实的locationName，通过names获取id后，再用nodes获取id的真正名字
           nodes.get(id).name = locationName;
           locations.get(id).name = locationName;
           //trie里存放的是cleanString，方便检索
           trie.put(cleanedName, id);
       }
   
       //获取对应名字的点
       ArrayList<Long> getLocations(String name) {
   
           return names.get(cleanString(name));
       }
   
       void addNode(Long id, double lon, double lat) {
           point n = new point(id, lon, lat);
           nodes.put(id, n);
           //初始化邻接表
           adjNode.put(id, new ArrayList<>());
           adjEdge.put(id, new ArrayList<>());
           locations.put(id, n);
       }
       //添加way
       void addWay(ArrayList<Long> ways, String wayName) {
           for (int i = 1; i < ways.size(); i++) {
               //每相邻两点为一个edge
               addEdge(ways.get(i - 1), ways.get(i), wayName);
           }
       }
   
       void addEdge(Long v, Long w, String wayName) {
           validateVertex(nodes.get(v));
           validateVertex(nodes.get(w));
           //xml文件中先出现所有的点，然后再出现way，通过对之前已经出现的点的引用，将它们连接成路
           //在addNode中初始化了邻接表，此时可以向邻接表中添加相邻的点和相邻的边
           adjNode.get(v).add(w);
           adjNode.get(w).add(v);
           adjEdge.get(v).add(new Edge(v, w, distance(v, w), wayName));
           adjEdge.get(w).add(new Edge(v, w, distance(v, w), wayName));
       }
   
       void validateVertex(point v) {
   
           if (!nodes.containsKey(v.id)) {
               throw new IllegalArgumentException("Vertex " + v + "is not in the graph");
           }
       }
   
   }
   
   ```

2. 解析路网数据：

   我们选择SAX parser来解析XML文件，SAX是Simple API for XML的缩写，它是一种基于流的解析方式，边读取XML边解析，并以事件回调的方式让调用者获取数据。因为是一边读一边解析，所以无论XML有多大，占用的内存都很小。

   **在每一个元素的开头和结尾，它会分别调用 `startElement` 和`endElement` 回调方法**。所以我们需要重写 `startElement` 和`endElement` 方法，SAX parser解析完XML文件时，就构建了一个图。

   解析的流程如下：

   1. 创建事件处理程序（编写ContentHandler的实现类，一般继承自DefaultHandler类，采用adapter模式），重写`startElement` 和`endElement` 方法
2. 创建SAX解析器
   3. 创建事件处理对象
   4. 将XML文件和事件处理对象注册到解析器，parser从上到下遍历xml文件中的每一个element，回调事件处理方法
   
   具体代码如下：
   
   1. 首先创建事件处理程序（编写ContentHandler的实现类，一般继承自DefaultHandler类，采用adapter模式），重写`startElement` 和`endElement` 方法
   
      `GraphBuildingHandler`类：
   
      ```java
      //解析XML文件的第一步：创建事件处理程序（编写ContentHandler的实现类，一般继承自DefaultHandler类，采用adapter模式）
      public class GraphBuildingHandler extends DefaultHandler {
          //允许的highway的类型
          private static final Set<String> ALLOWED_HIGHWAY_TYPES = new HashSet<>(Arrays.asList
                  ("motorway", "trunk", "primary", "secondary", "tertiary", "unclassified",
                          "residential", "living_street", "motorway_link", "trunk_link", "primary_link",
                          "secondary_link", "tertiary_link"));
          //记录当前的节点的父节点是什么
          private String activeState = "";
          private final GraphDB g;
          //暂存way中的node
          private static ArrayList<Long>ways;
          //标志一条路是否是符合要求的路
          private boolean validWay;
          /**
           * 创建一个新的GraphBuildingHandler对象
           * @param g 用XML数据填充的图
           */
          public GraphBuildingHandler(GraphDB g) {
              this.g = g;
          }
      
          private Long id;
          private double lon;
          private double lat;
          //路的名字默认是未知的
          private String wayName = Router.NavigationDirection.UNKNOWN_ROAD;
      
          /**
           * 在每一个元素起始的时候调用该方法
           * @param uri The Namespace URI, or the empty string if the element has no Namespace URI or
           *            if Namespace processing is not being performed.
           * @param localName The local name (without prefix), or the empty string if Namespace
           *                  processing is not being performed.
           * @param qName 我们正在查看的元素的名字。
           * @param attributes 附加到元素的属性。 如果没有属性，则应为空的Attributes对象。
           * @throws SAXException Any SAX exception, possibly wrapping another exception.
           * @see Attributes
           */
      
          //parser解析器调用了这个事件处理程序中的该方法，参数是解析器传进来的，对应正在解析的element的属性
          @Override
          public void startElement(String uri, String localName, String qName, Attributes attributes)
                  throws SAXException {
              //有些元素内部会有子元素，这些子元素有时会决定我们对该元素的操作，所以我们需要保持对父元素的追踪，
              //activeState就是用来保持对父元素的追踪，在某一个元素的内部，我们可以通过activeState来知道它的父元素是什么
              if (qName.equals("node")) {
                  //遇到了node
                  activeState = "node";
      
                  //获取node的信息
                  id = Long.parseLong(attributes.getValue("id"));
                  lon = Double.parseDouble(attributes.getValue("lon"));
                  lat = Double.parseDouble(attributes.getValue("lat"));
                  //将当前的node添加进Graph
                  g.addNode(id, lon, lat);
      
              } else if (qName.equals("way")) {
                  //遇到way，我们需要继续向下遍历，记录过程中的所有node，遇到tag时才能判断是否是合法的路，
                  //如果是合法的路，将node两两连接成边，加入Graph中
                  activeState = "way";
                  //创建一个List，暂存这个way标签中的node，等到确认这个way是合法的，再将暂存的node加入Graph
                  ways = new ArrayList<>();
                  validWay = false;
              } else if (activeState.equals("way") && qName.equals("nd")) {
                  //在way中遇到node，并不是所有的way都是有效的，因此不应该在这里直接向Graph添加node
                  //因为之后可能会遇到使此way无效的tag，必须删除之前添加的node。而是应该将node暂时存起来，
                  //等到确认此way是有效的，再将node加入Graph
                  ways.add(Long.parseLong(attributes.getValue("ref")));
      
              } else if (activeState.equals("way") && qName.equals("tag")) {
                  //在way中遇到了tag
                  //获取key和value
                  String k = attributes.getValue("k");
                  String v = attributes.getValue("v");
                  if (k.equals("highway")) {
                      /*  判断当前的way是否有效 */
                      if (ALLOWED_HIGHWAY_TYPES.contains(attributes.getValue("v"))) {
                          validWay = true;
                      }
                      //获取way的名字
                  } else if (k.equals("name")) {
                      if (v.equals(""))
                          wayName = Router.NavigationDirection.UNKNOWN_ROAD;
                      else wayName = v;
                  }
                  //在node中，发现了k="name"的tag
              } else if (activeState.equals("node") && qName.equals("tag") && attributes.getValue("k")
                      .equals("name")) {
                  //添加地点
                  g.addName(id, lon, lat, attributes.getValue("v"));
              }
          }
      
          /**
           * 在一个元素结束的时候调用该方法
           * @param uri The Namespace URI, or the empty string if the element has no Namespace URI or
           *            if Namespace processing is not being performed.
           * @param localName The local name (without prefix), or the empty string if Namespace
           *                  processing is not being performed.
           * @param qName The qualified name (with prefix), or the empty string if qualified names are
           *              not available.
           * @throws SAXException  Any SAX exception, possibly wrapping another exception.
           */
          @Override
          public void endElement(String uri, String localName, String qName) throws SAXException {
              if (qName.equals("way")) {
               //如果当前的way元素是合法的话，将way中的所有node连接起来
                  if (validWay) {
                   g.addWay(ways, wayName);
                      wayName = Router.NavigationDirection.UNKNOWN_ROAD;
                  }
              }
          }
      }
      ```
   
   2. 第二步：在 `GraphDB`类的构造函数中调用 `GraphBuildingHandler`事件处理程序来解析XML文件
   
      ```java
      //构造函数解析xml文件，将xml文件以图的形式表示出来
          public GraphDB(String dbPath) {
              try {
                  //读取xml文件
                  File inputFile = new File(dbPath);
                  FileInputStream inputStream = new FileInputStream(inputFile);
                  // GZIPInputStream stream = new GZIPInputStream(inputStream);
                  //第二步：创建SAX解析器
                  SAXParserFactory factory = SAXParserFactory.newInstance();
                  SAXParser saxParser = factory.newSAXParser();
                  //第三步：创建事件处理程序对象
                  GraphBuildingHandler gbh = new GraphBuildingHandler(this);
                  //第四步：将XML文件和事件处理程序分配到解析器，parser从上到下遍历xml文件中的每一个element，
                  // 回调GraphBuildingHandler中的事件处理方法
                  saxParser.parse(inputStream, gbh);
                  System.out.println(this.nodes.size());
              } catch (ParserConfigurationException | SAXException | IOException e) {
                  e.printStackTrace();
              }
              //XML文件中孤立的点不能导航，需要将它们从nodes和adjNode中清除出去
              clean();
          }
      ```

### 3、算法的实现

服务器端具体需要实现如下几个算法：

- 寻找距离平面中任意一点最近的点

- 最短路径的计算
- 导航
- 位置搜索的自动补全

#### 最短路径的计算

我们学习过BFS，知道BFS可以在无权边的图中寻找最短路径。而实际的地图是一种有权图，地图中的每个条路的长度都不一样，对于有权图我们可以使用Dijkstra。

Dijkstra本质上是一种贪心算法，利用优先队列，存储图中的点和离源点的距离，从源点开始，优先访问距离源点最近的点，然后对周围**没有访问过**的点进行松弛，更新被松弛的点和源点的距离。

我们可以将Dijkstra的过程看作是一棵树，图中每一个点都是其中的一个节点，它的相邻的点（neighbor）对应它的子节点。叶结点都在优先队列中，而所有的内部节点都已经处理过了，从优先队列中删除。每一次以一个叶结点为中心向外探索，将该叶结点从优先队列中删除，将与它相邻的没有处理过的节点加入优先队列，变成新的叶结点。

假设 `d`是起点到终点的距离，我们发现Dijkstra算法需要访问所有到起点的距离小于等于 `d`的点，但是在已经知道终点的方向的情况下，我们可以将方向作为探索的启发，朝着终点的方向搜索，从而避免很多不必要的探索，这就是A*搜索算法。

A*搜索算法只需要在Dijkstra的基础上修改一行代码就行了。Dijkstra的优先队列的优先级比较仅仅是当前点到起点的距离，而A *搜索算法的优先队列的优先级则是由**当前点到起点的距离**和**当前点到终点的距离**共同决定。

综上，`Router`类中A*算法的过程如下：

- 首先定义在 `shortestPath`中使用的搜索单位，`SearchNode`类，使用`SearchNode`的目的是**方便从终点回溯**，找到路径。其中属性为：
  - 当前节点的id
  - 指向当前点的父节点，以便在找到终点后可以获取从终点到起点的路径
  - 当前点到起点的距离
  - 当前点在优先队列中的优先级
- `shortestPath`算法的过程如下：
  - 我们需要维护一个优先队列，该队列的内容就是 `SearchNode`对象，可以从该队列中获取最佳的移动节点
  - 将优先队列中的最佳的节点移除，标记被移除的节点
  - 如果该节点是终点，结束查找
  - 如果该节点不是终点，那么以该节点为中心，探索它的相邻节点，将**没有标记**的相邻节点创建 `SearchNode`对象，将它们加入队列
  - 再从队列中获取最佳的节点，如此往复
- 优先队列的优先级比较机制如下：由**当前点到起点的距离**和**当前点到终点的距离**共同决定。

对于我们这个服务器端的`shortestPath`算法的实现，需求是对地图上的**任意两点**求出它们的最短距离。而我们使用的OSM数据只有有限点，并不是所有点都是可达的。所以我们需要先获取距给定经纬度最近的点，再由**获取到的点**计算最短路径。这就涉及到了平面内的最近邻搜索，我们稍后再谈这个问题。

A*搜索算法的具体实现如下：

```java
public class Router {

    //起点和终点
    private static point start;
    private static point destination;
    private static GraphDB graph;
    private static class SearchNode implements Comparable<SearchNode> {
        public Long id;
        //记录当前点的父节点，以便在找到终点后可以获取从终点到起点的路径
        public SearchNode parent;
        //当前点到起点的距离
        public double distanceToStart;
        //当前点在优先队列中的优先级
        public double priorit;

        public SearchNode(Long id, SearchNode parent,double distanceToStart) {
            this.id = id;
            this.parent = parent;
            this.distanceToStart = distanceToStart;
            //优先级由当前点到起点的距离和当前点到终点的估计距离共同决定
            //在Dijkstra中，优先级仅仅由当前点到起点的距离决定，这样会造成很多不必要的探索
            this.priorit = distanceToStart + distanceToDest(id);
        }
        @Override
        public int compareTo(SearchNode o) {
            if (this.priorit < o.priorit) {
                return -1;
            }
            if (this.priorit > o.priorit) {
                return 1;
            }
            return 0;
        }
    }
    //辅助方法，某一点和终点的距离
    private static double distanceToDest(Long id) {
        point v = graph.nodes.get(id);
        return GraphDB.distance(v.lon, v.lat, destination.lon, destination.lat);
    }

    /**
     * 返回一个List<Long>，代表离起点最近的点和离终点最近的点之间的最短路径上的点
     * @param g 使用的地图
     * @param stlon 起点的经度
     * @param stlat 起点的纬度
     * @param destlon 终点的经度
     * @param destlat 终点的纬度
     * @return 返回一个点的列表代表最短路径上顺序经过的所有点
     */
    public static List<Long> shortestPath(GraphDB g, double stlon, double stlat,
                                          double destlon, double destlat) {
        graph = g;
        //由于使用的是OSM的XML路网数据，该数据只有有限的点，地图上并不是所有点都可以到达的，
        //所以需要找出距给定起点和终点最近的可到达的点，再找出这两点之间的最短距离
        start = graph.nodes.get(g.closest(stlon, stlat));
        destination = graph.nodes.get(g.closest(destlon, destlat));

        //标记从优先队列中被移除的节点
        Map<Long, Boolean> marked = new HashMap<>();
        //优先队列
        PriorityQueue<SearchNode> pq = new PriorityQueue<>();
        pq.offer(new SearchNode(start.id, null, 0));
        while (!pq.isEmpty() && !isGoal(pq.peek())) {
            SearchNode v = pq.poll();
            //标记为经过
            marked.put(v.id, true);
            for (Long w : g.adjacent(v.id)) {
                if (!marked.containsKey(w) || marked.get(w) == false) {
                    pq.offer(new SearchNode(w, v, v.distanceToStart + distance(g, w, v.id)));
                }
            }
        }
        SearchNode pos = pq.peek();
        ArrayList<Long> path = new ArrayList<>();
        while (pos != null) {
            path.add(pos.id);
            pos = pos.parent;
        }
        Collections.reverse(path);
        return path; 
    }
    
    private static double distance(GraphDB graph,Long id1, Long id2) {
        point v1 = graph.nodes.get(id1);
        point v2 = graph.nodes.get(id2);
        return GraphDB.distance(v1.lon, v1.lat, v2.lon, v2.lat);
    }
    private static boolean isGoal(SearchNode v) {
        return distanceToDest(v.id) == 0;
    }
}
```

##### 优先队列的优化

在以上的做法中，只要是没有标记的点，全部都可以加入优先队列中，一个点可能多次被存入优先队列，也就是说队列中可能存在到同一个点的多个不同的路线，而我们的需求实际上是让优先队列只存储到某一点的最优路径。所以，没有标记的点在加入优先队列前，需要先判断到该点的距离是否小于优先队列中的距离，只有比优先队列中存储的距离更优，才有加入的必要。其次，不应该直接加入队列，而是将队列中旧的值替换为新的更优的值。

为了实现这个需求，我们需要实现索引优先队列，该队列应该做到可以根据id获取其中的节点，并且可以从外部修改节点的优先级。该队列内部有一个数组，用来构造堆；还有一个Map，用来建立id到数组下标的索引，从而可以根据id直接获取节点。

```java
public class ArrayHeapMinPQForSearchNode {
    //用来构造堆的数组
    private List<Router.SearchNode> pq = new ArrayList<>();
    //建立从id到数组下标的索引，从而能够由id直接获得对应的点，以及该点的优先级
    private Map<Long, Integer> idToIndex = new HashMap<>();

    public ArrayHeapMinPQForSearchNode() {
        pq.add(null);
    }

    /* Inserts an item with the given priority value. */
    void add(Router.SearchNode item) {
        pq.add(item);
        idToIndex.put(item.id, pq.size() - 1);
        swim(pq.size() - 1);
    }

    public boolean isEmpty() {
        return size() == 0;
    }

    private void exch(int i, int j) {
        Router.SearchNode t = pq.get(i);
        idToIndex.put(pq.get(i).id, j);
        idToIndex.put(pq.get(j).id, i);
        pq.set(i, pq.get(j));
        pq.set(j, t);
    }

    private boolean less(int i, int j) {
        return pq.get(i).compareTo(pq.get(j)) < 0;
    }

    private void swim(int k) {
        if (k / 2 >= 1 && less(k, k / 2)) {
            exch(k / 2, k);
            swim(k / 2);
        }
    }

    private void sink(int index) {
        int j = 2 * index;
        if (j > pq.size() - 1) return;
        if (j + 1 <= pq.size() - 1 && less(j + 1, j)) j++;
        if (less(index, j)) return;
        exch(index, j);
        sink(j);
    }

    /* Returns true if the PQ contains the given item. */
    boolean contains(Long id) {
        return idToIndex.containsKey(id);
    }

    //由id获取对应SearchNode的Priority
    Double getPriorityOfID(Long id) {
        return pq.get(idToIndex.get(id)).priority;
    }

    /* Returns the minimum item. */
    Router.SearchNode getSmallest() {
        return pq.get(1);
    }

    /* Removes and returns the minimum item. */
    Router.SearchNode removeSmallest() {
        Router.SearchNode t = pq.get(1);
        exch(1, pq.size() - 1);
        idToIndex.remove(pq.get(pq.size() - 1).id);
        pq.remove(pq.size() - 1);
        sink(1);
        return t;
    }
    /* Changes the priority of the given item. Behavior undefined if the item doesn't exist. */

    /**
     * 改变对应id的searchNode的priority
     *
     * @param id
     * @param newNode
     */
    void changeSearchNode(Long id, Router.SearchNode newNode) {
        int index = idToIndex.get(id);
        double curPriority = pq.get(index).priority;
        //一个新的点parent、distanceToStart、priority都要改变，所以干脆直接新构造一个SearchNode
        pq.set(index, newNode);
        if (newNode.priority > curPriority) sink(index);
        else swim(index);
    }

    /* Returns the number of items in the PQ. */
    int size() {
        return pq.size() - 1;
    }
}
```

`ShortestPath`的改进则是：

向优先队列加入新的点之前，先进行判断

- 如果队列中没有该点，则直接加入
- 如果队列有该点，将新的点的优先级与队列中的优先级进行比较，如果新的点更优，就将队列中的点替换为新的点。

```java
		ArrayHeapMinPQForSearchNode pq = new ArrayHeapMinPQForSearchNode();
        pq.add(new SearchNode(start.id, null, 0));
        while (!pq.isEmpty() && !isGoal(pq.getSmallest())) {
            SearchNode v = pq.removeSmallest();
            //标记为经过
            marked.put(v.id, true);
            for (Long w : g.adjacent(v.id)) {
                if (marked.containsKey(w) && marked.get(w)) continue;

                SearchNode newNode = new SearchNode(w, v, v.distanceToStart + distance(g, w, v.id));
                if (pq.contains(w) && pq.getPriorityOfID(w) > newNode.priority) {
                    //不仅仅是改变优先级，SearchNode的parent、distanceToStart都需要修改
                    pq.changeSearchNode(w, newNode);
                }
                if (!pq.contains(w)) pq.add(newNode);
            }
        }
```

#### 平面中的最近邻搜索

在最短路径的计算中我们需要求出距给定经纬度最近的点，这就是平面中的范围搜索的问题。

范围搜索通常有两种方法：一种是空间哈希（spatial hashing），一种是KDtree（k-dimensional树的简称）。

在这里我们使用KDtree来进行范围搜索。

k-d树（ k-维树的缩写）是在*k*维欧几里得空间中组织点的数据结构。*k*-d树可以使用在多种应用场合，如多维键值搜索（例：范围搜寻及最近邻搜索）。*k*-d树是空间二分树（Binary space partitioning）的一种特殊情况。

*k*-d树是每个叶子节点都为k维点的二叉树。所有非叶子节点可以视作用一个超平面把空间分割成两个半空间。节点左边的子树代表在超平面左边的点，节点右边的子树代表在超平面右边的点。选择超平面的方法如下：每个节点都与k维中垂直于超平面的那一维有关。因此，如果选择按照x轴划分，所有x值小于指定值的节点都会出现在左子树，所有x值大于指定值的节点都会出现在右子树。这样，超平面可以用该x值来确定，其法线为x轴的单位向量。

具体切割的过程可以参考这篇文章：[详解KDTree](https://blog.csdn.net/silangquan/article/details/41483689)

由于我们的应用场景是一个平面，所以我们实现的KDtree是2维的一个特例。

类似BST，KDtree的每个节点的属性分为左节点和右节点，左边是小于当前节点的点，右边是大于当前节点的点。对于2维及以上的KDtree，树的每一层的比较规则都不同。

对于二维的KDtree，每个点拥有两个子空间

- 根节点将整个空间分为左边和右边（用x划分）
- 所有深度为1的节点将子空间分为上面和下面（用y划分）
- 所有深度为2的节点将子空间分为左边和右边（用x划分）

以此类推。在插入时，我们规定相等的点属于右节点。

最近邻搜索算法的过程：

1. 首先将所有点插入KDtree：

   与二叉树的插入基本类似，把插入的点和树中每一层的点进行比较，如果比它小就进入它的左子树继续比较，如果比它大就进入它的右子树比较。如果相等就进行替换，如果为null就创建一个新的叶结点。只不过KDtree中节点的大小比较是由x和y两个值轮流决定的。

   [KDtree Insertion Demo](https://docs.google.com/presentation/d/1WW56RnFa3g6UJEquuIBymMcu9k2nqLrOE1ZlnTYFebg/edit#slide=id.g54b6045b73_0_38)

2. 对给定的点，在KDtree中查找离它最近的点

   goodside是离终点近的那一边，badside是离终点远的那一边

   对于每一个点，首先朝着它的good side搜索，并且动态更新距目标点的最优距离以及记录最近的节点。一直探索到某一点的good side为空时，开始回溯，根据每一点的bad side范围中距离目标点最近的距离与当前的最优距离的对比情况决定是否对bad side进行探索。如果小于最优距离，则需要探索。如果大于最优距离，则不需要探索，继续回溯。如此直到回到根节点，探索完毕。

   [K-d tree nearest demo](https://docs.google.com/presentation/d/1DNunK22t-4OU_9c-OBgKkMAdly9aZQkWuv_tBkDg1G4/edit?usp=sharing)

KDTree中使用的`Point`类：

```java
public class point {

    public final Long id;
    public final double lon;
    public final double lat;
    public String name = null;
    public point(Long id, double lon, double lat) {
        this.id = id;
        this.lon = lon;
        this.lat = lat;
    }

    public point(double lon, double lat) {
        this.id = null;
        this.lon = lon;
        this.lat = lat;
    }
    public double getX() {
        return lon;
    }

    public double getY() {
        return lat;
    }

    public double distanceTo(point p) {
        return GraphDB.distance(this.lon, this.lat, p.lon, p.lat);
    }
    @Override
    public boolean equals(Object obj) {
        if (obj == null) return false;
        if (obj.getClass() != this.getClass()) return false;
        point p = (point) obj;
        return this.getX() == p.getX() && this.getY() == p.getY();
    }
}

```

`KDtree`：

```java
public class KDTree implements PointSet{
    private static final boolean diviedByX = false;
    private static final boolean diviedByY = true;

    private class Node implements Comparable<Node> {
        private Node left;
        private Node right;
        private point p;
        private boolean orientation;
        public double bestDistanceToGoal = Double.MAX_VALUE;

        public Node(point p, boolean orientation) {
            this.p = p;
            this.orientation = orientation;
        }

        @Override
        public int compareTo(Node o) {
            //不同的点有不同的划分规则
            if (this.orientation == diviedByX) {
                return Double.compare(this.p.getX(), o.p.getX());
            } else {
                return Double.compare(this.p.getY(), o.p.getY());
            }
        }
    }

    private Node root;

    public KDTree(List<point> points) {
        for (point p : points) {
            insert(p);
        }
    }
    public KDTree(){

    }

    public void insert(point p) {
        //根节点的空间由x划分
        root = insert(root, p, diviedByX);
    }

    private Node insert(Node x, point p, boolean orientation) {
        if (x == null) return new Node(p, orientation);
        int cmp = x.compareTo(new Node(p, orientation));
        if (cmp < 0) x.left = insert(x.left, p, !orientation);
        else x.right = insert(x.right, p, !orientation);
        if (x.p.equals(p)) x.p = p;
        return x;
    }

    public point nearest(double x, double y) {
        return nearest(root, root, new point(null, x, y), diviedByX).p;
    }

    private Node nearest(Node x, Node best, point goal, boolean orientation) {
        if (x == null) return best;
        if (Double.compare(x.p.distanceTo(goal), best.bestDistanceToGoal) < 0) {
            best = x;
            best.bestDistanceToGoal = x.p.distanceTo(goal);
        }
        int cmp = x.compareTo(new Node(goal, orientation));
        //离终点近的一边是goodSide，远的一边是badSide，始终朝着终点的方向搜索
        Node goodSide = cmp < 0 ? x.left : x.right;
        Node badSide = cmp < 0 ? x.right : x.left;
        best = nearest(goodSide, best, goal, !orientation);
        //goodSide探索完了后，判断badSide是否值得探索
        if (isWorthLook(x, goal, best.bestDistanceToGoal, orientation))
            best = nearest(badSide, best, goal, !orientation);
        return best;
    }

    private boolean isWorthLook(Node curNode, point goal, double bestDistance, boolean orientation) {
        //如果badSide中距离终点最近的点比最优距离小，那么badSide就值得探索
        //如果badSide中距离终点最近的点都比最优距离大，那么badSide中一定没有比最优距离更近的点，不值得探索
        if (orientation == diviedByX)
            return goal.getX() - curNode.p.getX() < bestDistance;
        else
            return goal.getY() - curNode.p.getY() < bestDistance;
    }

}
```

`GraphDB`中初始化完Graph后就将点插入KDTree中

```java
/**
     * 将没有连接的、孤立的点从Graph中移除出去，这些点在寻路时、导航时都没有用，但是在寻找兴趣点时有用
     * 所以我们需要将它们从nodes和adjNode中清除，但是需要在locations和names中保存这些点
     */
    private void clean() {
        Iterator<Map.Entry<Long, ArrayList<Long>>> it = adjNode.entrySet().iterator();
        while (it.hasNext()) {
            Map.Entry<Long, ArrayList<Long>> entry = it.next();
            if (entry.getValue().isEmpty()) {
                //只清理nodes和adjNode
                nodes.remove(entry.getKey());
                it.remove();
            }
        }
        //将清理后的nodes插入到KDTree
        insertToKDtree();
    }

    private void insertToKDtree(){
        kdTree = new KDTree();
        for (Map.Entry<Long, point> entry : nodes.entrySet()) {
            kdTree.insert(entry.getValue());
        }
    }
	long closest(double lon, double lat) {
        return kdTree.nearest(lon, lat).id;
    }
```

#### 导航

在获得最短路径后，我们还需要根据最短路径生成相应的导航信息。

定义了一个 `NavigationDirection`类，表示导航信息。每一条路由一个 `NavigationDirection`对象定义。包含三个属性：

- `direction` 是一个代表方向的常数
- `way` 道路的名字 ，
- `distance` 在这条路上需要行驶的距离 

导航信息的格式如下：

`String.format("在 %s %s 并且继续行驶 %.3f miles.", way, DIRECTIONS[direction], distance)`

DIRECTIONS是一个map，对应着八种方向：

- “出发”
- “直行”
- “向左微转”
- “向右微转”
- “左转”
- “右转”
- “向左急转”
- “向右急转”

```java
/**
     * 表示导航方向的类, 包含三个属性:
     * 行驶的方向，道路的名字 ，在这条路上需要行驶的距离
     */
    public static class NavigationDirection {

        /** 代表方向的常数 */
        public static final int START = 0;
        public static final int STRAIGHT = 1;
        public static final int SLIGHT_LEFT = 2;
        public static final int SLIGHT_RIGHT = 3;
        public static final int RIGHT = 4;
        public static final int LEFT = 5;
        public static final int SHARP_LEFT = 6;
        public static final int SHARP_RIGHT = 7;

        /** 支持的方向的总数 */
        public static final int NUM_DIRECTIONS = 8;

        /** 从int到字符串方向的映射*/
        public static final String[] DIRECTIONS = new String[NUM_DIRECTIONS];

        /** 未知的路的名字 */
        public static final String UNKNOWN_ROAD = "不知名的路";
        
        /** 静态初始化 */
        static {
            DIRECTIONS[START] = "出发";
            DIRECTIONS[STRAIGHT] = "直行";
            DIRECTIONS[SLIGHT_LEFT] = "向左微转";
            DIRECTIONS[SLIGHT_RIGHT] = "向右微转";
            DIRECTIONS[LEFT] = "左转";
            DIRECTIONS[RIGHT] = "右转";
            DIRECTIONS[SHARP_LEFT] = "向左急转";
            DIRECTIONS[SHARP_RIGHT] = "向右急转";
        }

        /** 一个NavigationDirection代表的方向*/
        int direction;
        /** 道路的名字 */
        String way;
        /** 这条路的长度 */
        double distance;

        /**
         * Create a default, anonymous NavigationDirection.
         */
        public NavigationDirection() {
            this.direction = STRAIGHT;
            this.way = UNKNOWN_ROAD;
            this.distance = 0.0;
        }

        /**
         *
         * @param direction
         * @param way
         * @param distance
         */
        public NavigationDirection(int direction, String way, double distance) {
            this.direction = direction;
            this.way = way;
            this.distance = distance;
        }
        //重载了toString
        public String toString() {
            return String.format("在 %s %s 并且继续行驶 %.3f miles.",
                    way, DIRECTIONS[direction], distance);
        }
    }
```

根据最短路径的点生成导航信息的过程：

- 为了方便判断道路的名字，我们首先需要将最短路径提供的点集转化成连接这些点的边

- 遍历这些边，如果当前边的名字与前一条边相同（**在校园地图中用ID判断**），就说明还在同一条道路上。同时累积当前道路上经过的边的长度，用来计算当前道路的长度。

- 如果当前的边的名字与前一条边不同（**在校园地图中用ID判断**），就说明进入了一条新的道路。由于一个 `NavigationDirection`对象代表一条道路，所以此时需要用前一条边的名字、累积的道路长度、和上一条道路的起始方向构造一个 `NavigationDirection`对象，代表上一条道路的导航信息。

- 由于进入了一条新的道路，累积的道路长度清零。

- 计算拐弯的角度，也就是上一条道路和新的道路的夹角。由拐点和下一个点的连线与水平线的夹角减去拐点和上一个点的连线与水平线的夹角得到。

  而拐弯的方向取决于拐弯的角度具体是多少：

  - Between -15 and 15 degrees the direction should be “直行”.
  - Beyond -15 and 15 degrees but between -30 and 30 degrees the direction should be “向左/右微转”.
  - Beyond -30 and 30 degrees but between -100 and 100 degrees the direction should be “左/右转”.
  - Beyond -100 and 100 degrees the direction should be “向左/右急转”.

- 计算结束后，继续遍历新的道路上的边，当这条道路结束时，再创建一个`NavigationDirection`对象。如此反复，直到到达终点。

- 由于`NavigationDirection`重载了 `toString()`方法，所以最后会以“在 xx路上 xx 并且继续行驶 xx miles”的格式展示导航信息。

```java
/**
     * 根据一串点创建导航
     * @param g
     * @param route 需要被转换成导航的路，每一个元素对应路上的一个节点
     * @return 返回一串NavigationDirection对象，每个对象对应一条路
     */
    public static List<NavigationDirection> routeDirections(GraphDB g, List<Long> route) {

        double distance = 0;
        int relativeDirection = NavigationDirection.START;
        ArrayList<NavigationDirection> navigationList = new ArrayList<>();
        //将输入的点转化为连接点的边，如果边的名字相同，则说明在同一条路上，否则就不在同一条路上
        ArrayList<GraphDB.Edge> ways = getWays(g, route);
        if (ways.size() == 1) {
            navigationList.add(new NavigationDirection(NavigationDirection.START, ways.get(0).getName(), ways.get(0).getWeight()));
            return navigationList;
        }
        for (int i = 1; i < ways.size(); i++) {
            GraphDB.Edge preEdge = ways.get(i - 1);
            GraphDB.Edge nextEdge = ways.get(i);

            Long prevVertex = route.get(i - 1);
            Long curVertex = route.get(i);
            Long nextVertex = route.get(i + 1);

            String preWayName = preEdge.getName();
            String nextWayName = nextEdge.getName();

            distance += preEdge.getWeight();
            //如果前后两条路的名字不一样，则说明切换了路线，更新NavigationList，清零distance
            if (!preWayName.equals(nextWayName)) {
                double preBearing = g.bearing(prevVertex, curVertex);
                double nextBearing = g.bearing(curVertex, nextVertex);
                navigationList.add(new NavigationDirection(relativeDirection, preWayName, distance));

                relativeDirection = relativeDirection(preBearing, nextBearing);
                distance = 0;
            }
            if (i == ways.size() - 1) {
                distance += nextEdge.getWeight();
                navigationList.add(new NavigationDirection(relativeDirection, nextWayName, distance));
            }
        }
        return navigationList;
    }

    private static ArrayList<GraphDB.Edge> getWays(GraphDB g, List<Long> route) {
        ArrayList<GraphDB.Edge> ways = new ArrayList<>();
        //通过每两个点来确定两点之间的一条边
        for (int i = 1; i < route.size(); i++) {
            Long curVertex = route.get(i - 1);
            Long nextVertex = route.get(i);
            for (GraphDB.Edge e : g.neighbors(curVertex)) {
                //上一个点和下一个点所在的相同的边就是这两点之间的边
                if (e.other(curVertex).equals(nextVertex)) {
                    ways.add(e);
                }
            }
        }
        return ways;
    }
    private static int relativeDirection(double prevBearing, double curBearing) {
        double relativeBearing = curBearing - prevBearing;
        double absBearing = Math.abs(relativeBearing);
        if (absBearing > 180) {
            absBearing = 360 - absBearing;
            relativeBearing *= -1;
        }
        if (absBearing <= 15) {
            return NavigationDirection.STRAIGHT;
        }
        if (absBearing <= 30) {
            return relativeBearing < 0 ? NavigationDirection.SLIGHT_LEFT : NavigationDirection.SLIGHT_RIGHT;
        }
        if (absBearing <= 100) {
            return relativeBearing < 0 ? NavigationDirection.LEFT : NavigationDirection.RIGHT;
        }
        else {
            return relativeBearing < 0 ? NavigationDirection.SHARP_LEFT : NavigationDirection.SHARP_RIGHT;
        }
    }
```

#### 位置搜索自动补全

服务器端还需要实现给定一个字符串，可以返回所有以该字符串为前缀的位置。这就是自动补全，原理与Google、百度等搜索引擎输入时可以自动补全类似。

当我们存储一些可以被切分成“字符”的键时（比如string），由于很多键的前缀相同，如果像map或BST一样，每个键单独存储，会存储很多重复的字符，造成空间的浪费和查找缓慢。所以，如果这些键可以共享前缀，我们就针对string这个数据类型的BST结构做出了改进，创造出了另一个**针对string**的高效的数据结构：**Tire**

**trie**，又称**前缀树**或**字典树**，是一种有序树，用于保存关联数组，其中的键通常是字符串。与BST不同，键不是直接保存在节点中，而是由节点在树中的位置决定。一个节点的所有子孙都有相同的前缀，也就是这个节点对应的字符串，而根节点对应空字符串。一般情况下，不是所有的节点都有对应的值，只有叶子节点和部分内部节点所对应的键才有相关的值。

Trie把字符串的每一个Character作为一个键存在树的节点中，每个string对象的字符单独存储。有相同前缀的字符串在Trie中分享相同的节点，重复的前缀只存储一次。

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210520155618608.png" alt="image-20210520155618608" style="zoom:50%;" />

Trie的查找操作：

单词查找树中的每个结点都包含了下一个可能出现的所有字符的链接。从根结点开始,首先经过的是键的首字母所对应的链接;在下一个结点中沿着第二个字符所对应的链接继续前进;在第二个结点中沿着第三个字符所对应的链接向前，如此这般直到到达键的最后一个字母所指向的结点或是遇到了一条空链接。这时可能会出现以下3种情况：

- 键的尾字符所对应的节点中的值非空，这是一次命中的查找，键所对应的值就是键的尾字符所对应的结点中保存的值。
- 键的尾字符所对应的结点中的值为空。这是一次未命中的查找，符号表中不存在被查找的键。
- 查找结束于一条空链接。这也是一次未命中的查找。

对于具体实现来说，查找实际分为两种情况：

- 没有遇到空链接，将对应的节点的值直接返回，如果是null就说明查找失败，否则查找成功
- 遇到空链接，查找失败，直接返回

插入操作：

和二叉查找树一样，在插入之前要查找到插入的位置，沿着Trie向下找。在查找的过程中

- 如果遇到空链接，创建一个新节点，在该链接处插入。继续将要查找的字符串剩下的字符转换成节点插入到Trie的后面。

直到要查找的字符串到达末尾，将要插入的值直接赋给当前节点的value属性。

```java
public class Trie<Value> {
    private Node root;

    private static class Node {
        private Object val;
        private Map<Character, Node> next = new HashMap<>();
    }

    public Value get(String key) {
        Node x = get(key, root, 0);
        if (x==null) return null;
        return (Value) x.val;
    }

    private Node get(String key, Node x,int index) {
        if (x == null) return null;
        //index是字符串中下一个要比较的下标，也就是说，当前要比较的下标就是字符串的末尾
        //此时字符串除了最后一个字符，其他的都已经比较完了
        //现在，这个字符存在两种状态：是或不是
        //如果是，就返回该字符对应的值，如果不是，就返回null，所以不需要进行判断
        if (index == key.length()) {
            return x;
        }
        return get(key, x.next.get(key.charAt(index)), index + 1);
    }

    public void put(String key, Value value) {
        root = put(key, value, root, 0);
    }
    //put先查找，如果沿途中的字符与key中的字符一一对应，那么把value赋值给最后一个节点
    //如果在查找时遇到了空节点，那么将key中剩下的字符继续接下去，然后再将value赋值给最后一个节点
    private Node put(String key, Value value, Node x, int index) {
        if (x==null) x = new Node();
        if (index == key.length()) {
            x.val = value;
            return x;
        }
        //由于Trie是从根节点开始向下遍历，而根节点是不存放元素的，
        // 所以index指向的实际是Trie下一个的节点
        char c=key.charAt(index);
        //插入到x的子节点
        x.next.put(c, put(key, value, x.next.get(c), index + 1));
        return x;
    }

}

```

`Trie`的前缀匹配：

我们使用一个私有递归方法 `collect()`来完成这个任务。我们将所有匹配得到的结果保存在一个队列中，同时维护一个字符串用来保存从根节点出发的路径上的一系列字符。每当我们在`collect()`调用中访问一个结点时，方法的第一个参数就是该结点，第二个参数则是和该结点相关联的字符串(从根结点到该结点的路径上的所有字符)。

具体过程如下：

1.  首先在 `keysWithPrefix()` 方法中查找到前缀所对应的节点，将该节点和要匹配的前缀传入 `collect`方法中。
2. 在访问一个节点时
   - 如果这个节点为null，就结束对这条路的访问，直接return
   - 如果这个节点的值不为null，我们就将和它关联的字符串加入队列中
   - 如果这个节点的值为null，就不加入队列
3. 继续访问该节点链接数组所指向的所有可能的字符节点。

```java
public Iterable<String> keys() {
        return keysWithPrefix("");
    }
    public Iterable<String> keysWithPrefix(String s) {
        Queue<String> queue = new LinkedList<>();
        collect(get(s, root, 0), s, queue);
        return queue;
    }
    private void collect(Node x, String pre, Queue<String> queue) {
        if (x == null) return;
        if (x.val != null) queue.offer(pre);
        for (Map.Entry<Character, Node> entry : x.next.entrySet()) {
            collect(entry.getValue(), pre + entry.getKey(), queue);
        }
    }
```

### 4、服务器的启动

1. 首先解析下载的OSM路网数据，初始化地图

   ```java
   /** HTTP failed response. */
       private static final int HALT_RESPONSE = 403;
   
       //路网数据的位置
       private static final String Wuhan_PATH = "Wuhan_City_Group.osm";
       private static final String OSM_DB_PATH = Wuhan_PATH;
       /**
        * 所有发送给服务器的寻路请求都必须有如下的四个参数：
        * start_lat : 起始点的纬度,<br> start_lon : 起始点的经度,<br>
        * end_lat : 终点的纬度, <br>end_lon : 终点的经度.
        **/
       private static final String[] REQUIRED_ROUTE_REQUEST_PARAMS = {"start_lat", "start_lon",
           "end_lat", "end_lon"};
       //解析OSM的路网数据后生成的地图
       private static GraphDB graph;
       //最短路径的结果，List中是路径上每个点的id
       private static List<Long> route = new LinkedList<>();
   
       //解析路网数据，创建地图
       public static void initialize() {
           //从xml文件创建一个图对象
           graph = new GraphDB(OSM_DB_PATH);
       }
   
       public static void main(String[] args) {
           //解析路网数据，初始化地图
           initialize();
       }
   ```

2. 此时在main方法中仅仅只有对`Graph`对象的初始化，还需要接收Android客户端的API调用，进行API调用处理，返回客户端需要的信息。

   - Spark 应用的请求的处理都是由Route来完成， Route将 URL 模式映射到 Java 处理程序。Route使包含三个部分：

     - verb (HTTP请求方式)(get, post, put, delete, head, trace, connect, options)
     - path (请求资源路径)(/hello, /users/:name)
     - callback (回调方法)(request, response) -> { }

     Route按照它们定义的顺序进行匹配，第一个匹配上的Route将会被调用

   - Route的回调方法使用了Java8中`Lambda`表达式。Lambda表达式是一种函数式编程的语法，**函数式编程（Functional Programming）**是把函数作为基本运算单元，函数可以作为变量，可以接收函数，还可以返回函数。

     Lambda表达式的写法如下，它只需要写出方法定义：

     ```java
     (s1, s2) -> {
         return s1.compareTo(s2);
     }
     ```

     其中，参数是`(s1, s2)`，参数类型可以省略，因为编译器可以自动推断出`String`类型。`-> { ... }`表示方法体，所有代码写在内部即可，返回值的类型也是由编译器自动推断。Lambda表达式没有`class`定义，因此写法非常简洁。

   具体实现：

   - 处理**寻路请求**的Route：

     `get()`方法映射 HTTP GET 请求的路由，req是服务器接收的请求，res是服务器的响应，`->{...}`内是对GET请求的处理。将返回的信息以键值对的形式存放，然后再用Gson将Java对象转化为JSON格式返回给客户端。

     返回的信息为：

     - 寻路是否成功
     - 导航是否成功
     - 导航信息
     - 最短路径上的所有的点的经纬度

     ```java
     get("/route", (req, res) -> {
                 //将请求参数转化成对应的Map形式
                 HashMap<String, Double> params =
                         getRequestParams(req, REQUIRED_ROUTE_REQUEST_PARAMS);
     //            System.out.println(params);
                 //获取最短路径
                 route = Router.shortestPath(graph, params.get("start_lon"), params.get("start_lat"),
                         params.get("end_lon"), params.get("end_lat"));
                 //获取该路径的导航信息
                 String directions = getDirectionsText();
                 //以Map键值对的形式表达返回的信息
                 Map<String, Object> routeParams = new HashMap<>();
                 //寻路是否成功、导航是否成功
                 routeParams.put("routing_success", !route.isEmpty());
                 routeParams.put("directions_success", directions.length() > 0);
                 //返回导航的结果
                 routeParams.put("directions", directions);
                 ///返回路径上的所有点的经纬度
                 routeParams.put("route", getPositions(route));
                 //使用Gson库将Java对象转化成Json格式，返回客户端需要的信息
                 Gson gson = new Gson();
                 return gson.toJson(routeParams);
             });
     //根据点的id获取经纬度
         private static List<HashMap<String, Double>> getPositions(List<Long> id) {
             List<HashMap<String, Double>> positionList = new LinkedList<>();
             for (Long i : id) {
                 HashMap<String, Double> position = new HashMap<>();
                 Double lon = graph.nodes.get(i).lon;
                 Double lat = graph.nodes.get(i).lat;
                 position.put("lat", lat);
                 position.put("lon", lon);
                 positionList.add(position);
             }
             return positionList;
         }
     /**
          * 将最短路径转化可以返回给客户端的字符串的导航信息，
          */
         private static String getDirectionsText() {
             //根据最短路径获取对应的导航信息，接收一个装着NavigationDirection对象的List
             List<Router.NavigationDirection> directions = Router.routeDirections(graph, route);
             if (directions == null || directions.isEmpty()) {
               return "";
             }
             //将导航信息整合成一个字符串，以便在客户端直接展示
             StringBuilder sb = new StringBuilder();
             int step = 1;
             //NavigationDirection重载了toString，实际显示的格式是:
             //"%s on %s and continue for %.3f miles."
             for (Router.NavigationDirection d: directions) {
                 sb.append(String.format("%d. %s <br>", step, d));
                 step += 1;
             }
             return sb.toString();
         }
     ```

     其中 `Router.shortestPath()`是我们实现的最短路径算法，`getDirectionsText()`是实现的导航功能。

   - 处理兴趣点搜索以及自动补全请求的Route：

     ```java
     get("/search", (req, res) -> {
                 //返回所有的请求参数的键
                 Set<String> reqParams = req.queryParams();
                 //查询请求参数对应键的值
                 String term = req.queryParams("term");
                 Gson gson = new Gson();
                 //搜索某一确定点的位置
                 if (reqParams.contains("full")) {
                     List<Map<String, Object>> data = getLocations(term);
                     return gson.toJson(data);
                 } else {
                     //返回所有以该字符串为前缀的名字
                     List<String> matches = getLocationsByPrefix(term);
                     return gson.toJson(matches);
                 }
             });
     //获取图中所有前缀与查询字符串匹配的OSM位置集合
         public static List<String> getLocationsByPrefix(String prefix) {
             return graph.keysWithPrefixOf(prefix);
         }
     /**获取图中所有和查询的名字相同的地点，返回每一个点的信息
          * @param locationName 一个用来查询的完整的名字
          * @return 与查询的名字相同的所有位置的List，所有位置的信息都以键值对的形式存在List中，以便转化为JSON
          * response as specified: <br>
          * "lat" : Number, The latitude of the node. <br>
          * "lon" : Number, The longitude of the node. <br>
          * "name" : String, The actual name of the node. <br>
          * "id" : Number, The id of the node. <br>
          */
         public static List<Map<String, Object>> getLocations(String locationName) {
             ArrayList<Long> nodes = graph.getLocations(locationName);
             if (nodes == null) return null;
             //将对象转换成键值对的形式，以便转化为JSON
             //同一个名字对应的点不唯一，所以使用List来存储所有与该名字对应的点
             List<Map<String, Object>> result = new LinkedList<>();
             for (Long i : nodes) {
                 Map<String, Object> nodeInfo = new HashMap<>();
                 point node = graph.locations.get(i);
                 nodeInfo.put("lat", node.lat);
                 nodeInfo.put("lon", node.lon);
                 nodeInfo.put("name", node.name);
                 nodeInfo.put("id", node.id);
                 result.add(nodeInfo);
             }
             return result;
         }
     
     ```

至此，服务器端就可以启动了，默认情况下，Spark 应用在嵌入式 Jetty 服务器中运行。



## Android客户端

客户端需要实现的需求如下：

- 显示用户当前的位置
- 用户在地图上任意一个点上双击，在地图上标注出这个点
- 用户在图上选择了两个点后，获取这两个点位置的经纬度，然后发送给服务器，服务器端会返回一串坐标，将这些坐标在地图上连起来
- 实现一个自动补全搜索框，每当输入的内容发生变化时，将内容发送给服务器，服务器会提供所有以该内容为前缀的字符串，然后将这些字符串展示到搜索框的下拉列表中。当点击搜索框旁边的搜索按钮时，将搜索框中的内容发送给服务器，服务器会返回搜索到的地点的经纬度，将这些点标注在地图上。
- 实现一个按钮，可以清除地图上的标记（点、路线）

我们使用Mapbox地图SDK来完成Android端的操作和控制。

Mapbox是移动和网络应用程序的位置数据平台，提供订制在线地图的大型供应商。提供构建基块，将地图，搜索和导航等位置功能添加到创建的任何应用中。

### 1、配置Mapbox所需的依赖

将下列代码添加到Module中的`build.gradle`：

```java
repositories {
  mavenCentral()
}
 
dependencies {
  compile fileTree(dir: 'libs', include: ['*.jar'])
  testCompile 'junit:junit:4.12'
  compile 'com.android.support:appcompat-v7:26.1.0'
  // add the Mapbox SDK dependency below
  compile ('com.mapbox.mapboxsdk:mapbox-android-sdk:@aar'){
    transitive=true
  }
}
```

在 `AndroidManifest`文件中添加应用许可：

```xml
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
```

从Mapbox官网获取Mapbox的access token，配置到项目中的 `gradle.properties`文件中：

```xml
MAPBOX_DOWNLOADS_TOKEN=your access token here
```

### 2、布局文件

首先添加嵌入式的地图界面：

`MapView`类（class）是Mapbox地图库中十分关键的组成部分。它与其他的`View`类似，可以通过XML layout文件进行静态修改或者在运行过程中用代码动态修改。MapView提供了可嵌入的地图界面。 可以使用此类显示地图信息并从应用程序中操纵地图内容。 可以将地图以给定坐标为中心，指定要显示的区域的大小，并设置地图功能的样式以适合我们的应用程序的用例。

```xml
<!--    嵌入式的地图界面，可以使用这个类显示地图信息并从应用中
        操纵地图内容。-->
    <com.mapbox.mapboxsdk.maps.MapView
        android:id="@+id/mapView"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:layout_constraintTop_toTopOf="parent"
        >
    </com.mapbox.mapboxsdk.maps.MapView>
```

还需要查询文本框 `SearchView`：

```java
<!--    查询文本框，用户输入搜索查询，将请求提交给搜素提供者。显示查询的结果，并允许用户选择一个结果-->
    <SearchView
        android:id="@+id/searchView"
        android:layout_alignParentTop="true"
        android:layout_width="fill_parent"
        android:background="@color/white"
        android:layout_height="60dp"
        app:layout_constraintTop_toTopOf="parent"
        android:queryHint="输入关键字查询"
            tools:ignore="MissingConstraints">
    </SearchView>
```

查询文本框的下拉列表 `ListView`：

```xml
<!--    下拉列表部件，显示垂直可滚动的视图集合，其中每个视图都位于列表中上一个视图的正下方-->
    <ListView
        app:layout_constraintTop_toBottomOf="@+id/searchView"
        android:id="@+id/list"
        android:layout_below="@id/searchView"
        android:layout_width="fill_parent"
        android:background="#F8F8FF"
        android:layout_height="wrap_content"
            tools:ignore="MissingConstraints" />
```

四个按钮，垂直排列，分别实现：路径查找、清除地图上的路径、定位、导航。

```xml
<!--    四个按钮，垂直排列-->
    <LinearLayout
        app:layout_constraintTop_toBottomOf="@+id/searchView"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:orientation="vertical"
        app:layout_constraintRight_toRightOf="parent"
        android:padding="5dp">

<!--路径查找按钮，绑定的回调方法是queryRoad-->
        <androidx.appcompat.widget.AppCompatImageButton
            android:layout_width="40dp"
            android:layout_height="40dp"
            android:layout_marginTop="5dp"
            android:background="@color/white"
            android:onClick="queryRoad"
            android:padding="1dp"
            android:src="@mipmap/lujing" />
<!--清除路径按钮，绑定的回调方法是clear-->
        <androidx.appcompat.widget.AppCompatImageButton
            android:layout_width="40dp"
            android:layout_height="40dp"
            android:layout_marginTop="5dp"
            android:background="@color/white"
            android:onClick="clear"
            android:padding="1dp"
            android:src="@mipmap/qingchu" />
<!--定位按钮，绑定的回调方法是startLocate-->
        <androidx.appcompat.widget.AppCompatImageButton
            android:layout_width="40dp"
            android:layout_height="40dp"
            android:layout_marginTop="5dp"
            android:background="@color/white"
            android:onClick="startLocate"
            android:padding="1dp"
            android:src="@mipmap/dingwei" />
<!--        导航按钮，绑定的回调方法是showInfo-->
        <androidx.appcompat.widget.AppCompatImageButton
            android:layout_width="40dp"
            android:layout_height="40dp"
            android:layout_marginTop="5dp"
            android:id="@+id/roadInfo"
            android:background="@color/white"
            android:onClick="showInfo"
            android:padding="1dp"
            android:src="@mipmap/luxian" />
    </LinearLayout>
```

绑定的回调方法分别是：`queryRoad`、`clear`、`startLocate`、`showInfo`

### 3、Activity

#### 加载地图：

添加完`MapView`并为其赋值之后，需要调用 `MapView.getMapAsync`来创建一个 `MapboxMap`对象。`MapboxMap` 内置的多种方法能够实现改变地图样式或者相机位置\添加标注等功能

```java
@Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        Mapbox.getInstance(this, getString(R.string.mapbox_access_token));
        setContentView(R.layout.activity_main);
        mapView = (MapView) findViewById(R.id.mapView);
        mapView.onCreate(savedInstanceState);
        points=new ArrayList<>();
        //设置一个回调对象，当地图准备好时就会被触发
        mapView.getMapAsync(new OnMapReadyCallback() {
            //当地图准备好时，onMapReady方法就会被调用
            @Override
            public void onMapReady(@NonNull MapboxMap mapboxMap) {
                map=mapboxMap;
                //加载地图的样式，第一个参数指定地图的样式，第二个参数设置了一个回调对象，
                // 当样式加载完毕后就会调用重写的onStyleLoaded方法
                mapboxMap.setStyle(Style.MAPBOX_STREETS, new Style.OnStyleLoaded() {
                    @Override
                    public void onStyleLoaded(@NonNull Style style) {
                        //此时地图和样式都已经加载好了
                        //添加回调函数，当用户点击地图时会被调用
                      map.addOnMapClickListener(MainActivity.this);
                      //CameraPosition用户视点的位置、角度、缩放和倾斜
                      //target设置Camera指向的具体位置，zoom设置地图的缩放级别
                      CameraPosition cameraPosition=new CameraPosition.Builder().target(new LatLng(30.5008,114.2222)).zoom(15).build();
                      map.setCameraPosition(cameraPosition);
                      mapStyle=style;
                    }
                });

            }
        });

    }
```

请求权限，为查询文本框和下拉列表注册监听器

```java
//请求权限
        askPermission();
        //获取下拉列表部件
        listView=findViewById(R.id.list);
        //为列表注册监听器，当列表中的item被点击时调用回调函数
        listView.setOnItemClickListener(this);
        //获取查询文本框部件
        SearchView searchView = findViewById(R.id.searchView);
        searchView.onActionViewExpanded();
        //为查询文本框注册监听器
        searchView.setOnQueryTextListener(this);
        showImageButton=findViewById(R.id.roadInfo);
        showImageButton.setEnabled(false);
```

动态申请权限，使用PermissionTools，一个用于Android权限申请的工具库

```java
/**
     * 动态申请权限
     */
    private void askPermission(){
        //PermissionTools一个用于Android权限申请的工具库
        PermissionTools permissionTools;

        permissionTools =  new PermissionTools.Builder(this)
                .setOnPermissionCallbacks(new PermissionCallbacks() {
                    @Override
                    public void onPermissionsGranted(int requestCode, List<String> perms) {
                        Toast.makeText(MainActivity.this,"权限申请通过",Toast.LENGTH_SHORT).show();
                    }

                    @Override
                    public void onPermissionsDenied(int requestCode, List<String> perms) {
                        Toast.makeText(MainActivity.this,"权限申请被拒绝",Toast.LENGTH_SHORT).show();
                    }
                })
                .setRequestCode(111)
                .build();
        permissionTools.requestPermissions(Manifest.permission.ACCESS_FINE_LOCATION);
        permissionTools.requestPermissions(Manifest.permission.WRITE_EXTERNAL_STORAGE);
    }
```

#### 功能的实现

在地图上标注出用户点击的位置：

当地图被点击时，调用以下回调函数，point是被点击的位置：

```java
//当地图被点击时，调用的回调函数，point是被点击的位置
    @Override
    public boolean onMapClick(@NonNull LatLng point) {
        //在地图上的该点处添加标记和标题
        map.addMarker(new MarkerOptions()
                .position(new LatLng(point.getLatitude(), point.getLongitude()))
                .title("经度："+point.getLongitude()+"\n"+"纬度："+point.getLatitude()));
        //将该点保存起来，以便添加路线
        points.add(point);
        return false;
    }
```

搜索框的自动补全：

查询文本框的回调函数，当文本框的中的内容发生变化时，向服务器发送请求。使用Gson将服务器端返回的JSON格式的路径信息反序列化为`List<String>`类的对象，随后更新下拉列表。

```java
//查询文本框的内容变化的回调函数
    @Override
    public boolean onQueryTextChange(String s) {
        list = new ArrayList<String>();
        //如果查询的字符串不为空，就向服务器发出请求
        if(!TextUtils.isEmpty(s)){
            OkHttpClient okHttpClient = new OkHttpClient();
            String url = BASE_URL +
                    "/search?term=" + s;
            final Request request = new Request.Builder()
                    .url(url)
                    .get()//默认就是GET请求，可以不写
                    .build();
            Call call = okHttpClient.newCall(request);
            //发出请求
            call.enqueue(new Callback() {
                @Override
                public void onFailure(Call call, IOException e) {
                    Log.d(TAG, "onFailure: "+e.getMessage());
                }

                @Override
                public void onResponse(Call call, Response response) throws IOException {
                    String result=response.body().string();
                    runOnUiThread(new Runnable() {
                        @Override
                        public void run() {
                            Gson gson =new GsonBuilder().serializeNulls().create();
                            //将服务器端返回的JSON格式的路径信息反序列化为List<String>类的对象
                            List<String> json = gson.fromJson(result,new TypeToken<List<String>>() {}.getType());
                            //获取所有的字符串
                            list= (ArrayList<String>) json;
                            //更新下拉列表
                            updateList();
                        }
                    });
                }
            });
        }else {
            updateList();
        }

        return false;
    }
```

更新下拉列表，给下拉列表部件绑定数组适配器 `ArrayAdapter`

```java
	private void updateList(){
        //ArrayAdapter数组适配器用于绑定格式单一的数据，数据源可以是集合或者数组
        ArrayAdapter listAdapter = new ArrayAdapter(this,android.R.layout.simple_list_item_1,list);//创建适配器与数据源showlist绑定
        //给下拉列表部件绑定数组适配器
        listView.setAdapter(listAdapter);
    }
```

点击下拉列表中的item，将这个点显示在地图上：

点击下拉列表中的item，向服务器发出请求，获取该名字的点的位置信息，随后将点绘制在地图上

```java
//点击下拉列表中的item的回调函数
    @Override
    public void onItemClick(AdapterView<?> adapterView, View view, int i, long l) {
       String key=list.get(i);
        OkHttpClient okHttpClient = new OkHttpClient();
        String url = BASE_URL +
                "/search?full=true&term=" + key;
        final Request request = new Request.Builder()
                .url(url)
                .get()//默认就是GET请求，可以不写
                .build();
        Call call = okHttpClient.newCall(request);
        //向服务器端发送请求
        call.enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {
                Log.d(TAG, "onFailure: "+e.getMessage());
            }

            @Override
            public void onResponse(Call call, Response response) throws IOException {
                String result=response.body().string();
                runOnUiThread(new Runnable() {
                    @Override
                    public void run() {
                        Gson gson =new GsonBuilder().serializeNulls().create();
                        //将查询的JSON格式的结果变成List<InPoint>
                        List<InPoint> results = gson.fromJson(result,new TypeToken<List<InPoint>>() {}.getType());
                        //在地图上显示点的位置
                        showResult(results);
                    }
                });
            }
        });
        Log.e(TAG,key);

        //查询结束后，清空下拉列表
       onQueryTextChange("");
    }
    //将点绘制在地图上
    private void showResult(List<InPoint> results) {
        clear(null);
        if(results.size()==0){
            return;
        }
        //
        double latMin=results.get(0).getLat();
        double latMax=results.get(0).getLat();
        double lonMin=results.get(0).getLon();
        double lonMax=results.get(0).getLon();
        for (int i = 0; i <results.size() ; i++) {
            //将所有的点都绘制在地图上
            InPoint point=results.get(i);
            map.addMarker(new MarkerOptions()
                    .position(new LatLng(point.getLat(), point.getLon()))
                    .title("名称："+point.getName()+"\n"+"经度："+point.getLon()+"\n"+"纬度："+point.getLat()));


            if(latMax<point.getLat()){
                latMax=point.getLat();
            } if(lonMax<point.getLon()){
                lonMax=point.getLon();
            } if(latMin>point.getLat()){
                latMin=point.getLat();
            } if(lonMin>point.getLon()){
                lonMin=point.getLon();
            }

        }
        //将用户视角设置在第一个点的位置
        LatLngBounds latLngBounds=LatLngBounds.from(latMin,lonMin,latMax,lonMax);
        CameraPosition cameraPosition=   map.getCameraForLatLngBounds(latLngBounds);
        map.setCameraPosition(cameraPosition);
    }
```

四个按钮的回调函数：

- 寻路：向服务器发送寻路的请求

  ```java
  /**
       * 路径查询，查询按钮的回调函数
       * @param view
       */
      public void queryRoad(View view) {
          if(points.size()<2){
              Toast.makeText(MainActivity.this,"请线选择起点和终点",Toast.LENGTH_SHORT).show();
              return;
          }
          int size=points.size();
          //可以用于发送HTTP请求和读取其响应消息
          OkHttpClient okHttpClient = new OkHttpClient();
          //构造URL
          String url = BASE_URL +
                  "/route?start_lon=" + points.get(size - 2).getLongitude() +
                  "&start_lat=" + points.get(size - 2).getLatitude() +
                  "&end_lon=" + points.get(size - 1).getLongitude() +
                  "&end_lat=" + points.get(size - 1).getLatitude();
          final Request request = new Request.Builder()
                  .url(url)
                  .get()//默认就是GET请求，可以不写
                  .build();
          //准备请求
          Call call = okHttpClient.newCall(request);
          //发出请求
          call.enqueue(new Callback() {
              @Override
              public void onFailure(Call call, IOException e) {
                  Log.d(TAG, "onFailure: "+e.getMessage());
              }
              //处理服务器端返回的结果
              @Override
              public void onResponse(Call call, Response response) throws IOException {
                  String result=response.body().string();
                  runOnUiThread(new Runnable() {
                      @Override
                      public void run() {
                          //展示结果
                          showRoad(result);
                      }
                  });
              }
          });
      }
  ```

  将服务器端返回的最短路径展示到地图上

  ```java
  //将服务器端返回的最短路径展示到地图上
      private void showRoad(String jsonText){
          try {
              Gson gson =new GsonBuilder().serializeNulls().create();
              //将服务器端返回的JSON格式的路径信息反序列化为RouteResult类的对象
              RouteResult json = gson.fromJson(jsonText,RouteResult.class);
              if(json.getRouting_success()){
                  Toast.makeText(MainActivity.this,"路径规划成功",Toast.LENGTH_SHORT).show();
              }
              //获取路径上所有点的坐标
              List<Point> routeCoordinates=new ArrayList<>();
              for (int i = 0; i <json.getRoute().size() ; i++) {
                  TempPoint tempPoint=json.getRoute().get(i);
                  //将坐标构造为Point对象，以便在地图上操作
                  routeCoordinates.add(Point.fromLngLat(tempPoint.getLon(), tempPoint.getLat()));
              }
              //在绘制路线之前，先将原来的路线清除
              clearLine();
              //将源添加到地图，将所有的坐标连成线绘制在图层上
              mapStyle.addSource(new GeoJsonSource(ID_SOURCE,
                      FeatureCollection.fromFeatures(new Feature[] {Feature.fromGeometry(
                              LineString.fromLngLats(routeCoordinates)
                      )})));
  
              //添加线图层
              mapStyle.addLayer(new LineLayer(ID_LAYER, ID_SOURCE).withProperties(
                      PropertyFactory.lineDasharray(new Float[] {0.01f, 2f}),
                      PropertyFactory.lineCap(Property.LINE_CAP_ROUND),
                      PropertyFactory.lineJoin(Property.LINE_JOIN_ROUND),
                      PropertyFactory.lineWidth(5f),
                      PropertyFactory.lineColor(Color.parseColor("#e55e5e"))
              ));
              //获取导航信息
              roadInfo=json.getDirections().replaceAll("<br>","\n");
              showImageButton.setEnabled(true);
          }catch (Exception e){
              Log.d(TAG, "failure: "+e.getMessage());
  
          }
      }
  ```

- 清除路线：

  ```java
  //清除地图上的标记和路线
      public void clear(View view) {
          //清除点的记录
          points.clear();
          map.clear();
          clearLine();
          //设置按钮的启动状态
          showImageButton.setEnabled(false);
      }
      //清除线
      private void clearLine(){
          if(mapStyle.getLayer(ID_LAYER)!=null){
              mapStyle.removeLayer(ID_LAYER);
          }
          if(mapStyle.getSource(ID_SOURCE)!=null){
              mapStyle.removeSource(ID_SOURCE);
          }
      }
  ```

- 显示导航信息：

  ```java
  //显示导航信息，导航按钮的回调函数
      public void showInfo(View view) {
             new AlertDialog.Builder(MainActivity.this).setMessage(roadInfo).create().show();
      }
  ```

- 定位：

  ```java
  //定位当前位置，定位按钮的回调函数
      public void startLocate(View view) {
          enableLocationComponent(mapStyle);
      }
      //设置定位
      @SuppressWarnings( {"MissingPermission"})
      private void enableLocationComponent(@NonNull Style loadedMapStyle) {
          //判断是否有权限
          if (PermissionsManager.areLocationPermissionsGranted(this)) {
              LocationComponentOptions customLocationComponentOptions = LocationComponentOptions.builder(this)
                      .pulseEnabled(true)
                      .build();
  
              LocationComponent locationComponent = map.getLocationComponent();
              locationComponent.activateLocationComponent(
                      LocationComponentActivationOptions.builder(this, loadedMapStyle)
                              .locationComponentOptions(customLocationComponentOptions)
                              .build());
              locationComponent.setLocationComponentEnabled(true);
              locationComponent.setCameraMode(CameraMode.TRACKING);
              locationComponent.setRenderMode(RenderMode.NORMAL);
          } else {
              askPermission();
          }
      }
  ```

  