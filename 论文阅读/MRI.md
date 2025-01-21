# MRI

脑成像（brain imaging）分别包括结构性脑成像和功能性脑成像

<img src="C:\Users\李博\Desktop\mynotes\MRI.assets\image-20231009192228535.png" alt="image-20231009192228535" style="zoom: 50%;" />

- 结构成像（T1-weighted）用来研究大脑的结构，相当于对大脑某些部分的解剖，还有对疾病和伤害的诊断。结构性脑成像的最主要的方式就是MRI
- 功能成像（T2*-weighted)用来研究大脑的功能，就是对大脑或特定区域的任务相关活动的测量，或者对大脑跨区域的功能关系（brain connectivity，大脑连接）的测量，或者对大脑和身体之间关系（physiological connectivity，生理连接）的衡量

![image-20231009191741292](C:\Users\李博\Desktop\mynotes\MRI.assets\image-20231009191741292.png)

即使MRI是一种单一模式，它也有专注于底层组织不同方面的MRI图像。它们以不同的方式表示同一个大脑的相同内容

<img src="C:\Users\李博\Desktop\mynotes\MRI.assets\image-20231009192554504.png" alt="image-20231009192554504" style="zoom:67%;" />

功能性脑成像可以用来研究认知和情感过程，功能性脑成像最主要的方式是fMRI。

fMRI和MRI的区别是，fMRI是随着时间的变化获取一系列的图像并且研究事物如何随着时间变化（即**fMRI是由一系列单独的MRI图像组成**）

- 在fMRI的实验过程中，会采集一系列的大脑图像，然后根据每个图像之间测量出的信号的变化来推断大脑关于某些任务的刺激 

<img src="C:\Users\李博\Desktop\mynotes\MRI.assets\image-20231009193721718.png" alt="image-20231009193721718" style="zoom:67%;" />

每张图片（脑容量volume）由十万个大小相同的voxels（立方体）组成，每个voxel是我们在fMRI中使用的基本单位。每个voxel对应一个空间位置，并且有一个数字与其关联，代表它的强度。在fMRI的过程中，大约每两秒左右就要测量整个大脑，获取一个这样的脑容量（volume），一共要获取上百个。

<img src="C:\Users\李博\Desktop\mynotes\MRI.assets\image-20231009203819012.png" alt="image-20231009203819012" style="zoom:50%;" />

如果我们在不同的时间点的脑容量上选取相同位置的voxel，构成一个时间序列，可以研究正在发生的事情随着时间变化对这个voxel的强度的影响。

<img src="C:\Users\李博\Desktop\mynotes\MRI.assets\image-20231009203837685.png" alt="image-20231009203837685" style="zoom:50%;" />

<img src="C:\Users\李博\Desktop\mynotes\MRI.assets\image-20231009203705442.png" alt="image-20231009203705442" style="zoom: 50%;" />

fMRI的数据量是非常惊人的

<img src="C:\Users\李博\Desktop\mynotes\MRI.assets\image-20231009203943421.png" alt="image-20231009203943421" style="zoom:67%;" />

fMRI数据处理的全过程：

<img src="C:\Users\李博\Desktop\mynotes\MRI.assets\image-20231009204056829.png" alt="image-20231009204056829" style="zoom:67%;" />

fMRI数据分析有三个主要目标：

1. 定位：确定大脑的哪些特定区域在特定任务期间处于活跃状态，或者与特定的心理事件相关。并且存在多种定位，这些共同构成了我们所说的大脑映射方法（brain Mapping）

2. 连接：确定大脑的一个区域在功能上如何与另一个区域相关。通常意味着随着时间变化的测量值之间的相关性

   连接有多个种类：

   <img src="C:\Users\李博\Desktop\mynotes\MRI.assets\image-20231009205336704.png" alt="image-20231009205336704" style="zoom:67%;" />

3. 预测：使用一个人的大脑活动来预测他们的感知行为和健康状况。比如我们可以训练一个分类器模型，让输入的大脑活动与分类器参数内积，得到对该大脑的预测结果

在fMRI实验中，我们需要平衡空间分辨率（spatial resolution）和时间分辨率（temporal resolution），

- 在fMRI中时间分辨率（TR）是指图片采样的频率
  - 比如结构图像（T1图像）的空间分辨率高，时间分辨率低（实际上它根本没有时间分辨率，因为它只有一张图片）
- 空间分辨率是指分辨一张图片中不同位置的变化的能力
  - 而功能图像（T2*图像）的空间分辨率低，所以它们比实际对应的物体要模糊得多，但是时间分辨率高，可以用来将信号的变化与实验操作联系起来

![image-20231011154854944](C:\Users\李博\Desktop\mynotes\MRI.assets\image-20231011154854944.png)

大脑切片：我们测量三维的大脑体积，但是我们经常研究大脑切片  

![image-20231011155656786](C:\Users\李博\Desktop\mynotes\MRI.assets\image-20231011155656786.png)

MRI通常按照axial方向获取大脑的切片，一次获取一个，可以沿某一个方向顺序获取，也可以交错获取。最后获取多个切片合在一起拼成三维的脑容量（brain volume），这就是我们最终要分析的东西。

视场（field of view）：指大脑在图像内部的范围

每个切片也有厚度，然后将每个切片分割成voxel。所以如果把xy轴长度为192mm的切片切成64x64个voxel，那么每个voxel的厚度就是3 x 3 x3mm。这将是我们在一个核磁共振中感兴趣的测量单位的大小

<img src="C:\Users\李博\Desktop\mynotes\MRI.assets\image-20231011160536319.png" alt="image-20231011160536319" style="zoom:67%;" />

![image-20231011161323611](C:\Users\李博\Desktop\mynotes\MRI.assets\image-20231011161323611.png)

![image-20231011161359962](C:\Users\李博\Desktop\mynotes\MRI.assets\image-20231011161359962.png)

每个volume随着时间推移获取，所以每个voxel也都有一个与之相关的时间序列



在fMRI中，我们可以选择对volume哪些部分（即voxel）进行测试：

![image-20231011162741163](C:\Users\李博\Desktop\mynotes\MRI.assets\image-20231011162741163.png)

核磁共振（Magnetic Resonance）的物理原理：

<img src="C:\Users\李博\Desktop\mynotes\MRI.assets\image-20231011180753001.png" alt="image-20231011180753001" style="zoom:67%;" />

大多数的MRI和fMRI都涉及到将一系列二维的切片构造成三维的volume

<img src="C:\Users\李博\Desktop\mynotes\MRI.assets\image-20231011183116322.png" alt="image-20231011183116322" style="zoom:67%;" />

在K-space中获取数据，然后通过傅里叶变换得到图像

<img src="C:\Users\李博\Desktop\mynotes\MRI.assets\image-20231011192824556.png" alt="image-20231011192824556" style="zoom:67%;" />

图像空间中的voxel和K-space中的点不是一一对应的关系，K-space中每个点会对整个图像空间的每个voxel都产生影响

K-space中的一个点通过傅里叶变换后会得到一个正弦函数（二维的），该函数的移动方向是K-space中的点到原点的方向，频率是K-space中该点到原点的距离。所以K-space中的点的位置告诉我们该点在重建图像中的相对贡献，将所有的点的波线性组合就得到了真实的图像

<img src="C:\Users\李博\Desktop\mynotes\MRI.assets\image-20231011193413339.png" alt="image-20231011193413339" style="zoom: 50%;" />

Kspace中高频的点用于生成图像的边界，低频的点用于对比度

<img src="C:\Users\李博\Desktop\mynotes\MRI.assets\image-20231011193850468.png" alt="image-20231011193850468" style="zoom:67%;" />

要让图像的空间分辨率更高，会导致图像的生成速度变慢，从而影响功能成像的时间分辨率

<img src="C:\Users\李博\Desktop\mynotes\MRI.assets\image-20231011192414442.png" alt="image-20231011192414442" style="zoom:50%;" />

fMRI的信号会包含一些噪声，这些噪声来源于硬件和参与者自己

![image-20231011213932500](C:\Users\李博\Desktop\mynotes\MRI.assets\image-20231011213932500.png)

要消除这些噪声可以从两方面入手：

1. 采集的时候消除噪声：

   ![image-20231011214422225](C:\Users\李博\Desktop\mynotes\MRI.assets\image-20231011214422225.png)

2. 获取数据之后对数据进行一些预处理：

   <img src="C:\Users\李博\Desktop\mynotes\MRI.assets\image-20231011214442159.png" alt="image-20231011214442159" style="zoom:67%;" />

预处理首先是要查看我们的数据，需要避免出现以下的数据：

![image-20231011215402514](C:\Users\李博\Desktop\mynotes\MRI.assets\image-20231011215402514.png)

空间对齐：

- 即使功能图像和结构图像是在同一个扫描session期间获取的，功能图像通常会失真，因此他们不会精确地匹配到解剖结构（结构图像）
- 结构图像和模版（参考）图像之间也会有错位，

我们需要用所有的解剖图创建一个平均的解剖图，用它作为底层来显示我们的功能结果

切片时间的纠正：

我们每次会对大脑的多个切片进行采样，从而构建一个大脑容量（brain volume），但是每个切片通常是在不同时间采集的，因为我们是按照顺序获取它们的，而不是同时获取的。切片时间纠正（slice time correction）会移动每个voxel的时间序列从而让它们看起来是同时采集的

大脑运动（head motion）：

- 当我们在分析单个voxel对应的时间序列时，我们总是假设它在每个时间点都描述了大脑的同一个区域，但是由于大脑的移动，同一个voxel在不同时间完全可能处于不同的大脑区域

运动校正（motion correction）：

- 目标图片（target image）：指fMRI图片序列中的第一个（或者平均）的图片。

- 运动校正需要把所有输入的图像与target图像对齐，以确保voxel彼此对齐。为了对齐，其中的一张图片需要进行变换，需要使用刚体变换（rigid body transformation）

  <img src="C:\Users\李博\Desktop\mynotes\MRI.assets\image-20231012211008805.png" alt="image-20231012211008805" style="zoom:50%;" />

- 我们的目标是找到一组参数来最小化衡量图片和target之间相似度的cost函数

标准化（normalization）：

- 每个人的大脑都是不同的，标准化就是将每个大脑进行拉伸，挤压和扭曲，这样就与一些标准大脑相同

<img src="C:\Users\李博\Desktop\mynotes\MRI.assets\image-20231012212823501.png" alt="image-20231012212823501" style="zoom:67%;" />

在归一化之后我们要做的下一件事是空间滤波（spatial filtering） ，在fMRI中，在统计分析之前对采集的数据进行空间平滑（spatial smooth）是很常见的

<img src="C:\Users\李博\Desktop\mynotes\MRI.assets\image-20231012213347868.png" alt="image-20231012213347868" style="zoom:50%;" />

## 数据分析

<img src="C:\Users\李博\Desktop\mynotes\MRI.assets\image-20231012214911807.png" alt="image-20231012214911807" style="zoom:50%;" />

之前提到过fMRI的三大目标，最常见的用途是定位为了响应某个特定的任务被激活的大脑的功能区域。GLM模型可以做到这一点

GLM通常分为两个阶段，首先在为每个人拟合一个内部模型或者个体模型，然后进行分组分析，拟  合一个群体分析模型

<img src="C:\Users\李博\Desktop\mynotes\MRI.assets\image-20231012223149451.png" alt="image-20231012223149451" style="zoom: 50%;" />