# Lec1 概述

本课的主题：

- 光栅化通常用在实时（每秒生成30幅画面，即30帧，30FPS）的场景中
- 如何表示光滑的曲线和圆
- 光线追踪计算量很大，很慢，所以通常用在非实时的场景中（比如电影动画）
- 模拟现实的世界的物理规律

计算机视觉和图形学的区别：计算机视觉是让计算机看一个图像，让他分析理解猜测这个图像的模型。而图形学是给定一个模型让计算机生成一个好看的图像。所以二者是一个相反的过程

![image-20230920185855489](C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20230920185855489.png)

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20230921110920302.png" alt="image-20230921110920302" style="zoom:67%;" />

给定一个向量，如果不说明，默认是一个列向量

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20230921110959179.png" alt="image-20230921110959179" style="zoom: 50%;" />

点乘在图形学中最大的作用就是找到两个向量之间的夹角

点乘还可以用来判断两个向量方向是否相同，如果方向相同则大于0，否则小于0。如果两个向量（单位向量）完全重合就等于1

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20230921111952636.png" alt="image-20230921111952636" style="zoom: 67%;" />

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20230921112800224.png" alt="image-20230921112800224" style="zoom: 67%;" />

如果x叉乘y得到z，就说这是右手坐标系，因为这是由右手螺旋定理得到的

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20230921113044230.png" alt="image-20230921113044230" style="zoom:67%;" />

在图形学中，叉乘用来判断两个向量的左右，和一个点是否在图形的内

如果两个向量相乘，得到的向量在z轴的方向，就说明b在a的左边

下图中的三角形，沿着逆时针方向，每个向量与（从起点到P点构成的向量）相乘，如果p点在所有向量的同一个方向，就说明该点在三角形的内部

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20230921124813536.png" alt="image-20230921124813536" style="zoom:67%;" />

# 变换

![image-20230925145326169](C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20230925145326169.png)

用矩阵表示变换

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20230925145706252.png" alt="image-20230925145706252" style="zoom: 67%;" />

非均匀的缩放：

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20230925150122559.png" alt="image-20230925150122559" style="zoom: 67%;" />

反转变换：

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20230925150648187.png" alt="image-20230925150648187" style="zoom:67%;" />

水平方向的变换与高有关，对于a=1，当高为0时变换是0，当高为1时变换是1，所以水平方向的变换等于a乘y

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20230925150733703.png" alt="image-20230925150733703" style="zoom:67%;" />

旋转变换：

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20230925161700096.png" alt="image-20230925161700096" style="zoom:50%;" />

线性变换：**可以用矩阵相乘表示的变换**

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20230925162213217.png" alt="image-20230925162213217" style="zoom:67%;" />

但是对于平移变换，我们无法仅仅通过矩阵乘法来表示

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20230925162258751.png" alt="image-20230925162258751" style="zoom:50%;" />

平移变化必须要加上一个向量才能表示，所以平移变换并不是线性变换

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20230925162430780.png" alt="image-20230925162430780" style="zoom: 67%;" />

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20230925171314179.png" alt="image-20230925171314179" style="zoom:67%;" />

**线性变换加平移变换可以称之为仿射变化**，神经网络中的全连接层也可以叫做仿射层

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20230925201213473.png" alt="image-20230925201213473" style="zoom:67%;" />

在齐次坐标系下，两个点相加就相当于中点（因为只有W等于1才是一个点，否则就需要将xyw都除以w）

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20230925171555374.png" alt="image-20230925171555374" style="zoom:67%;" />

使用其次坐标的平移操作对应于变换矩阵的最后一列

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20230925172045920.png" alt="image-20230925172045920" style="zoom:67%;" />

矩阵的逆表示逆变换

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20230925172207052.png" alt="image-20230925172207052" style="zoom:67%;" />

我们可以用上面的这些基础的变换组合起来构成一个复杂的变换

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20230925172253350.png" alt="image-20230925172253350" style="zoom:67%;" />

所以上面的变化用矩阵表示是：变换矩阵之间也可以合并，合并后就是在中间矩阵的最后一列的X对应的平移上多了1

![image-20230925172400390](C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20230925172400390.png)

之前的旋转是绕着0点旋转，如果我们想绕着某一个特定的点旋转，可以让图形先向原点移动一个距离，旋转完后再移动回去

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20230925173019653.png" alt="image-20230925173019653" style="zoom:67%;" />

三维的变换：

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20230925173555660.png" alt="image-20230925173555660" style="zoom:67%;" />

在三维空间中变换矩阵的格式也是一样的，最后一列还是对应平移变换，最后一行还是全0一个1

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20230925173844104.png" alt="image-20230925173844104" style="zoom:67%;" />

逆向旋转的变换矩阵等于正向的变换矩阵的转置，也等于正向矩阵的逆

![image-20230925200547551](C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20230925200547551.png)



## 三维变换

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20230925201808679.png" alt="image-20230925201808679" style="zoom:67%;" />

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20230925202540732.png" alt="image-20230925202540732" style="zoom:67%;" />

任何三维的旋转都可以被分解为分别绕xyz三个轴的旋转

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20230925202657558.png" alt="image-20230925202657558" style="zoom:67%;" />

旋转公式：

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20230925203446206.png" alt="image-20230925203446206" style="zoom:67%;" />

# 视图变换

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20230925200902382.png" alt="image-20230925200902382" style="zoom:67%;" />

我们要做的是把三维中的一个物体变成二维的一张图片，这个过程与我们在现实生活中拍照类似，大致分为三步：

1. 找到一个好的位置并且把人们排好（模型（model）变换）
2. 找到一个好的角度放摄像机（视图（view）变换）
3. 拍照（投影（projection）变换，真正把三维变成二维）

所以视图变化也叫相机变换，就是固定相机的位置，相机的位置由三个东西组成：

1. 相机的位置
2. 相机看的角度（俯视和仰视的角度，即Look-at或gaze direction）
3. 相机以镜头为轴旋转的角度（up direction）

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20230925205212493.png" alt="image-20230925205212493" style="zoom:67%;" />

只要我们在变化相机的时候，所有的物体也跟着变换，那么照片就不变。所以我们可以总是把相机移动到原点，并且所有的物体也跟着相机移动

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20230925211544611.png" alt="image-20230925211544611" style="zoom:67%;" />

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20230925212124536.png" alt="image-20230925212124536" style="zoom: 67%;" />

要想让相机的位置和方向满足上面的要求，需要**先把相机平移到原点（计算出Tview平移矩阵），再把相机旋转到指定的角度（计算出Rview旋转矩阵）**

平移矩阵很好求，但是算出从相机原来的角度到坐标轴角度的旋转矩阵比较困难，我们**可以算出从坐标轴角度到相机的角度的变换矩阵，再求逆即可**

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20230925212529054.png" alt="image-20230925212529054" style="zoom:67%;" />

# 投影变换

投影分为正交投影和透视投影，正交投影通常用于工程制图，不遵循近大远小；透视投影则相反

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20230925213557825.png" alt="image-20230925213557825" style="zoom:67%;" />

透视投影相当于相机距离物体很近，把远处和近处的物体全部都投影到一个平面上；正交投影相当于相机距离物体无限远

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20230925213939953.png" alt="image-20230925213939953" style="zoom: 67%;" />

## 正交投影

正交投影相当于把Z轴扔掉。注意，**Z轴定义的是物体距离我们的远近，我们是沿着-Z方向看的，所以数值越大离我们越近**

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20230925214449347.png" alt="image-20230925214449347" style="zoom:67%;" />

另外一种更标准的做法是：先把物体的中心移动到原点，再把任何一种形状变成一个标准的立方体，这个立方体的范围通常是[-1,1]

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20230925215131028.png" alt="image-20230925215131028" style="zoom:67%;" />

具体的变换矩阵如下：右边是平移的矩阵，左边是缩放的矩阵，把物体的长宽高变成2 

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20230925215340839.png" alt="image-20230925215340839" style="zoom:67%;" />

这个变化做完之后长方体被放缩成正方体了，会导致物体的形状变形，所以在之后我们还会再做一次变换

## 透视投影

![image-20230925222209022](C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20230925222209022.png)

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20230925222232711.png" alt="image-20230925222232711" style="zoom:67%;" />

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20230925222305541.png" alt="image-20230925222305541" style="zoom:67%;" />

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20230925222334772.png" alt="image-20230925222334772" style="zoom:67%;" />

# 光栅化

上面所说的观测变换后（视图变化，投影变换），在场景中的所有的物体都会变成-1到1的立方体，下一步就要把它画在屏幕上，这一步叫光栅化

屏幕是由像素组成的数组，这个数组的大小就是分辨率，光栅（raster）就是屏幕的意思

将观测变化得到的立方体画在屏幕上的过程：

像素数组的形状如下：

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20231006154433176.png" alt="image-20231006154433176" style="zoom: 67%;" />

我们先不管Z轴，把XY轴映射到屏幕上

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20231006154736083.png" alt="image-20231006154736083" style="zoom:67%;" />

变换矩阵中对XY轴乘以height / 2，对Z轴乘以1，又因为立方体是以原点为中心，所以还需要移动到width  / 2和height / 2为中心处

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20231006154807172.png" alt="image-20231006154807172" style="zoom:67%;" />

然后我们还要把图像拆成不同的像素，画在屏幕上

在图形学中通常用三角形来表示图像，因为三角形有一些很好的性质：

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20231006161359585.png" alt="image-20231006161359585" style="zoom:67%;" />

一个像素只能有一个颜色，但是如果一个像素和三角形有交叉的地方，那么如何判断这个像素的颜色是什么？这就涉及到如何判断像素的中心点与三角形的位置关系

先定义一个函数，给定屏幕中的任意一个坐标xy和一个三角形t，此函数能够判断xy是否在三角形t中

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20231006162530957.png" alt="image-20231006162530957" style="zoom:67%;" />

而我们要做的是用此函数对于每个像素中心的xy进行采样，就能知道像素的中心与三角形的位置关系

- 采样是图形学中的核心观点：采样一个函数是给定一些点，求出该函数在这些点的值是多少，也就是将函数离散化

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20231006162811951.png" alt="image-20231006162811951" style="zoom:50%;" />

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20231006162854800.png" alt="image-20231006162854800" style="zoom:67%;" />

inside函数的实现方式就是：Q点分别与三条边（同方向）叉乘，如果在三条边的同方向，就说明此点在三角形的内部

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20231006163123683.png" alt="image-20231006163123683" style="zoom:67%;" />

光栅化加速：

我们不需要遍历整个屏幕的所有像素，只需要遍历三角形周围的像素判断他们的位置即可。这些像素的选取使用包围和（将三角形的顶点坐标中取最小和最大的x和y，就是包围和的四个顶点的坐标）

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20231006163721802.png" alt="image-20231006163721802" style="zoom:67%;" />

由于一个像素只能有一种均匀的颜色，并且像素本身有一定的大小，所以在光栅化图形学中会有一个严重的问题：锯齿

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20231006164549513.png" alt="image-20231006164549513" style="zoom:67%;" />

我们要做的是抗锯齿（反走样）

## 反走样

采样的artifact：

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20231006171545387.png" alt="image-20231006171545387" style="zoom:67%;" />

采样是相对于信号来说的，信号就是函数，采样就是选取一些自变量获取函数的值。而出现采样缺陷（即锯齿，走样）的原因就是**信号变化地太快了，以至于采样的速度跟不上**

反走样的方法之一：将三角形的inside函数或原始信号进行模糊处理，然后再对像素的中心进行采样

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20231006172857971.png" alt="image-20231006172857971" style="zoom:67%;" />

傅里叶级数展开：我们可以用正弦和余弦函数之和来近似任何的函数

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20231006174504139.png" alt="image-20231006174504139" style="zoom:67%;" />

傅里叶变换：我们可以通过傅里叶变化把一个函数变成另一个函数，再通过逆变换变回去

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20231006174516914.png" alt="image-20231006174516914" style="zoom:67%;" />

从下图可以看出，**如果采样的频率跟不上原始信号的频率，那么就很难将原始信号恢复出来**

每个像素的中心点就是一次采样，所以如果分辨率太小，那么像素就太少，同样的区域采样的频率就会更低，难以恢复原始信号（图像）

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20231006174530540.png" alt="image-20231006174530540" style="zoom:67%;" />

走样的标准定义：**同样一个采样方法采样两种不同的频率的函数得到的结果我们无法区分**

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20231006174926562.png" alt="image-20231006174926562" style="zoom:67%;" />

滤波：去掉特定的一系列的频率

卷积本质上是一种加权平均（对信号的每一个区域附近的值做加权平均），滤波就是filter

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20231006181846626.png" alt="image-20231006181846626" style="zoom:67%;" />

对三角形进行卷积，就是模糊化

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20231006195428334.png" alt="image-20231006195428334" style="zoom:67%;" />

具体的卷积方法：使用一个像素大小的核对原始图像进行卷积，实际上就是计算原始图像的覆盖的每个像素的平均值

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20231006200514483.png" alt="image-20231006200514483" style="zoom:67%;" />

我们采用的是近似的计算方法MSAA：

将每个像素点划分成很多个小像素，然后判断每个小像素的中心点是否在三角形内，然后计算出这个大像素被三角形的覆盖率，再用这个覆盖率乘以三角形的颜色就是这个大像素的颜色。所以MSAA并没有提高采样率和分辨率

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20231006201631729.png" alt="image-20231006201631729" style="zoom:67%;" />

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20231006201645899.png" alt="image-20231006201645899" style="zoom:67%;" />

<img src="C:\Users\李博\Desktop\mynotes\图形学\games101.assets\image-20231006201701270.png" alt="image-20231006201701270" style="zoom:67%;" />