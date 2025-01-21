

生成式模型的核心思想是通过学习一个给定数据集的概率分布，然后使用这种学习到的分布来生成新的数据实例

# Guided Diffusion

**原始的DDPM没有可引导的生成过程**，OpenAI的Guided Diffusion就提出了一种简单有效的类别引导的扩散模型生成方式。

Guided Diffusion的核心思路是在逆向过程的每一步，用一个分类网络对生成的图片进行分类，再基于分类分数和目标类别之间的交叉熵损失计算梯度，**用梯度引导下一步的生成采样**。

在DDPM中，无条件的逆向过程由Ptheta表示，加入类别条件y后，逆向过程可以表示为：

![image-20231103194234173](C:\Users\李博\Desktop\mynotes\ai\stable diffusion.assets\image-20231103194234173.png)

![image-20231103194611640](C:\Users\李博\Desktop\mynotes\ai\stable diffusion.assets\image-20231103194611640.png)

基于这样的改进，不需要重新训练扩散模型，只需要额外训练一个分类器，就能够有效地在添加类别引导。当然，这样的结构也存在一点小问题，就是会引入比较多的额外计算时间（每一步都要过分类模型并求梯度）。

## CLIP Guided

在Semantic Guidence Diffusion （SGD）中，作者就将类别引导改成了基于参考图引导以及基于文本引导两种形式，通过设计对应的梯度项，实现对应的引导效果

CLIP 模型包含一个图像编码网络 <img src="C:\Users\李博\Desktop\mynotes\ai\stable diffusion.assets\image-20231103195707402.png" alt="image-20231103195707402" style="zoom:67%;" /> 和文本编码网络 <img src="C:\Users\李博\Desktop\mynotes\ai\stable diffusion.assets\image-20231103195720715.png" alt="image-20231103195720715" style="zoom:67%;" />，两个编码网络能够各自将文本和图片编码为1*512 大小的向量，然后我们可以通过余弦距离来度量两者之间的相似度

由于CLIP的图像编码网络是没有见过加噪图像的，这会使得度量效果不理想，从而不能提供有效的梯度。因此，作者将Ei 在加噪图像上进行了一定的finetune，得到了适应加噪图像的Ei‘ 

通过CLIP计算扩散模型的每一步中去噪得到的图像与文本之间的相似度（梯度），来引导下一步的采样生成

## Classifier-Free Diffusion Guidence - 无分类器的扩散引导

上述的各种引导函数，基本都是额外的网络前向 + 梯度计算的形式，这种形式虽然有着训练成本低，见效快的优点（因为不需要重新训练扩散模型）。也存在着一些问题：

1. 推理时额外的计算量比较多（要过两个模型）；
2. 引导函数和扩散模型分别进行训练，不利于进一步扩增模型规模，**不能够通过联合训练获得更好的效果**。

**Classifier-Free Diffusion Guidence：原本DDPM的噪声预测模型的输入只有图片，现在给噪声预测模型加入了额外的条件输入**<img src="C:\Users\李博\Desktop\mynotes\ai\stable diffusion.assets\image-20231103200538253.png" alt="image-20231103200538253" style="zoom:67%;" />

训练扩散模型时，结合有条件和无条件两张训练方式，无条件时将y设置为NULL，从而得到一个同时支持有条件和无条件噪声估计的模型

**在此之前要做到多模态或者说文生图的引导生成，通常大家用的是clip模型，来对生成的图像和文本的距离做一个损失差值，用这个差值来引导多模态生成。但有了classifier-free这篇论文之后，将引导的过程融入到了扩散模型中，文生图或者图生图都可以用一个模型，以cross-attention的方式条件于该信息来引导生成，并且生成效果更好更精确**

缺点就是对不同的类型的条件，都需要重新训练整个扩散模型

# ldm中的attention部分

stable diffusion的cross attention部分：

利用cross attention 将 **latent space**的特征与另一模态序列的特征融合

- 在call unet的时候其实是call的UNet2DConditionModel.forward()

- UNet2DConditionModel里包含了（按顺序，以下提到的block各自都是一个class）：

- - 四个downblock：三个带attention的CrossAttnDownBlock2D，和一个DownBlock2D；
  - 一个中间的UNetMidBlock2DCrossAttn
  - 四个upblock：一个UpBlock2D, 和三个带attention的CrossAttnUpBlock2D

- 一个CrossAttnDownBlock2D包含了2-3个component：

- - 一个resnet
  - 一个spatial transformer（where cross attention is applied on）
  - 一个down sampler（optional， depends on是不是最后一个downblock）

- 由于cross attention是包含在spatial transformer里，继续去看spatial transformer的源码，存在attention.py里。spatial transformer其实就是在basic transformer class上做了改动

- 一个BasicTransformerBlock 包含两个CrossAttention layer（attn1，attn2）

- - 第一个layer只是self attention，不与condition embedding做交互
  - 第二个才是cross embedding，当condition embedding为None时也是self-attention

Spatial Transformer 同样接受两个输入：**经过上一个网络模块（一般为 ResnetBlock）处理和变换后的 latent 向量（对应的是是图片 token），及对应的 context embedding（文本 prompt 经过 CLIP 编码后的输出）， cross attention 之后，得到变换后的 latent 向量（通过注意力机制，将 token 对应的语义信息注入到模型认为应该影响的图片 patch 中）**。 Spatial Transformer 输出的 shape 和输入一致，但在对应的位置上融合了语义信息。

ldm的cross-attention过程：

- query是图像，key和value是prompt的token。图像的每个部分（如像素）对prompt的每个token都有一个注意力分数，也就是说，会**对每个token生成一个attention map**（即attention score经过softmax之后得到的东西），如果一句prompt中有十个token，就有十个attention map。再将这十个attention map与分别对对应的token的value进行加权求和，得到一个map，形状与输入的图像一样

  ![image-20231101193524521](C:\Users\李博\Desktop\mynotes\ai\stable diffusion.assets\image-20231101193524521.png)

通过显示token对应的attention map，我们可以发现：在attention  map中，token所描述的区域非常显眼

![image-20231101194339030](C:\Users\李博\Desktop\mynotes\ai\stable diffusion.assets\image-20231101194339030.png)

Prompt-to-Prompt同样是通过控制cross attention map来控制图像生成/修改图像

**Prompt2Prompt**方法，是通过**编辑提示**的方式在预训练的扩散模型中进行图像编辑，包括**局部编辑**（替换一个单词）、**全局编辑**（添加一个描述），甚至可以**精细地控制单词在图像中的反映程度**，而无需进行任何模型训练、微调，也不需要额外的数据。

# Cross-attention 控制

如果想要通过直接编辑prompt来对图像进行编辑，那么最直观的做法是把编辑后的prompt输入模型中重新生成一张图片。但是这么做只会生成一系列外观和结构完全不同的图片，无法做到局部的编辑

之前的模型往往需要用户给定一个 mask 指示需要更改的区域。然而，绘制 mask 一方面多少带来了一些不便，另一方面 mask 掉的区域将被完全无视，丢弃了可能是很重要的信息，导致应用场景受限，例如用户无法更改一个物体的纹理。因此，**Prompt-to-Prompt 致力于不需要 mask 的图像编辑方法**。

方法是使用cross-attention控制：生成图像的空间布局和几何形状取决于cross-attention map

由于注意力反映了图像的整体结构，所以将原来的prompt生成的cross-attention map注入到修改后的prompt和噪声进行的生成中。这样生成的图像不仅可以根据修改后的prompt进行编辑，还可以保留原图像的结构

具体方法：将原prompt和修改后的prompt生成图像的过程同步进行，并且二者的随机数种子相同。

先将两个prompt执行一次去噪过程，使用这个过程中的原prompt的M和修改后prompt的M*进行一些操作（Edit）后得到一个<img src="C:\Users\李博\Desktop\mynotes\ai\stable diffusion.assets\image-20231101195829311.png" alt="image-20231101195829311" style="zoom: 50%;" />，再使用这个map替换掉修改后prompt的attention map（value不变），再执行一次去噪过程，得到真正的需要的去噪结果。

![image-20231101195645161](C:\Users\李博\Desktop\mynotes\ai\stable diffusion.assets\image-20231101195645161.png)

细节：是全局编辑还是局部编辑还是取决场景，只修改一个词有可能是全局编辑（比如从夜晚变成白天，从印象派变成写实派，类似风格迁移）；添加很多个词也有可能是局部编辑（比如一个黑色的猫变成一个黑色的正在睡觉的猫）

局部编辑（比如将prompt中的某个词修改为另一个词）：

1. 我们近似了编辑部分的掩码，来限制修改只针对局部区域：
   - 分别计算原始token和新的token对应的attention map的平均值（从step T到step t），将这两个map变成二进制map（类似relu，值大于0.3时变成1，小于0.3时变成0），最后把两个二进制map union，作为权重对原prompt的去噪结果和修改后prompt的去噪结果加权平均，从而限制了对掩码之外区域的修改。

单词交换，例如将a big red bicycle改成a big red car：这里的挑战是在生成新的内容的同时要保证原有的图像结构不变，所以我们用原本的attention map覆盖新prompt的attention map，限制新的token的位置和大致形状与原来的token相同。

但是如果在所有的扩散步骤中都注入cross-attention map可能会过度约束，导致完全忽略了prompt。通过使用跟柔和的attention约束来解决这个问题：（~~这里的M是修改的token对应的M~~）。只在逆向过程的前期注入原本的attention map，后期注入新的attention map，因为图像的结构是在早期的时间确定的，从而保持原本的结构，只修改局部的区域。

![image-20231101203211379](C:\Users\李博\Desktop\mynotes\ai\stable diffusion.assets\image-20231101203211379.png)

<img src="C:\Users\李博\Desktop\mynotes\ai\stable diffusion.assets\image-20231101204722763.png" alt="image-20231101204722763" style="zoom:67%;" />

增加更精细的prompt：保留共同的token对应的attention map，加入新增token对应的attention map

![image-20231101204516790](C:\Users\李博\Desktop\mynotes\ai\stable diffusion.assets\image-20231101204516790.png)

<img src="C:\Users\李博\Desktop\mynotes\ai\stable diffusion.assets\image-20231101204732761.png" alt="image-20231101204732761" style="zoom:67%;" />

还可以通过改变指定单词的attention权重，可以控制它对生成图像的影响程度

<img src="C:\Users\李博\Desktop\mynotes\ai\stable diffusion.assets\image-20231101204848349.png" alt="image-20231101204848349" style="zoom:67%;" />

<img src="C:\Users\李博\Desktop\mynotes\ai\stable diffusion.assets\image-20231101204856472.png" alt="image-20231101204856472" style="zoom:67%;" />

# null-text inversion

真实图像的编辑：相比合成图像，我们并不知道真实图像的 prompt、也不知道喂进扩散模型的产生这张图像的初始的noise是什么（即包含真实图像信息的噪音），而如果要使用prompt-to-prompt方法编辑图像，这两个东西都是需要知道的。比如使用p2p编辑合成图像时，修改后的prompt的noise和修改前的prompt的noise是相同的

![image-20231101231217959](C:\Users\李博\Desktop\mynotes\ai\stable diffusion.assets\image-20231101231217959.png)

对于第一个问题，使用现有的一些 image captioning 模型就行；对于第二个问题，即找到一个喂进扩散模型可以产生某一个指定图像的noise（inversion）（即包含真实图像的噪音），可以使用ddim inversion

## ddim重建

Diffusion是一个把噪声映射到有用信息（比如图片）的过程，但 Diffusion 到噪声的过程是单向的，它并不可逆，不能直接像VAE一样直接把有用信息再映射回到隐空间，即，可以根据一个噪声得到图片，但不能根据一张图片得到“可以得到这张图片的噪声”，但这个噪声又在编辑中非常重要

所谓重建是指的首先用原始图像求逆得到对应的噪音然后再进行生成的过程，我们可以利用如下公式对生成过程进行逆操作：

![image-20231104211901015](C:\Users\李博\Desktop\mynotes\ai\stable diffusion.assets\image-20231104211901015.png)

也就是说，inversion不是一个简单的对图像加噪的过程，**也是需要经过扩散模型的**，根据输入的图像和时间步t（以及这里没有写出来的条件prompt）预测出噪声，将噪声**加到**输入的图像上

这意味着，我们可以由一个原始图像X0得到对应的随机噪音Xt，然后我们再用Xt进行生成就可以重建原始图像X0

![image-20231104214208576](C:\Users\李博\Desktop\mynotes\ai\stable diffusion.assets\image-20231104214208576.png)

DDIM inversion可以将图片映射到噪声，但是由于误差累积，从这个噪声得到的新图片会稍微偏离原图片。其实如果不给模型文本prompt，偏离还不会很严重，但是**当文本prompt加强时，偏离就会非常严重**

如下图所示，采用classifier free guidance的ldm模型中，噪声预测是将有条件prompt的预测结果和无条件prompt的预测结果通过权重w结合起来的。所以W越大，文本prompt就越强，重建的结果就越差；W越小，文本prompt就越弱，重建的结果就越好，但是得到的图片和prompt的关联就越弱，也就是图像的可编辑的能力就越差（所以W在stable diffusion中通常比较大，默认是7.5，从而增强条件prompt的影响）。

![image-20231104215454784](C:\Users\李博\Desktop\mynotes\ai\stable diffusion.assets\image-20231104215454784.png)

- 具体的，当引导系数W=1时，DDIM inversion产生轨迹T1提供了原始图像的粗略近似，叫做pivot。如果在W>1的情况下，直接将<img src="C:\Users\李博\Desktop\mynotes\ai\stable diffusion.assets\image-20231104221046989.png" alt="image-20231104221046989" style="zoom:67%;" />作为初始噪声进行DDIM采样，产生的轨迹T2会逐渐偏离原轨迹T1。

因此，作者提出**Pivotal Inversion**来解决classifier-free guidance扩散模型误差累积的问题：使用W=1对输入图片做DDIM inversion得到一系列的噪声向量，这些噪声向量近似于原始图像，所以将它们作为基准轨迹。然后使用一个较大的W（7.5）作为采样过程的引导系数（为了保证生成图片的可编辑性）。**对采样过程的每个时间步执行单独的优化，尽可能使采样的轨迹T2接近标准的T1，从而保留原图片的内容，同时还能可编辑**。

![image-20231104214350235](C:\Users\李博\Desktop\mynotes\ai\stable diffusion.assets\image-20231104214350235.png)

所以优化目标就是：

![image-20231104221749887](C:\Users\李博\Desktop\mynotes\ai\stable diffusion.assets\image-20231104221749887.png)

但是如果要微调去噪模型的话这个花费很大，并且图像编辑是基于prompt-to-prompt的，所以我们也不能修改条件prompt。所以null-text inversion选择的是对null-text embeding进行优化，模型权重和文本条件保持不变：

- 优化的目标还是上面的这个式子（即让去噪的轨道和标准轨道尽可能地接近），但是不是计算这个loss函数关于扩散模型的参数的梯度，优化扩散模型。而是**计算这个loss函数关于空文本的梯度，对空文本进行梯度下降，重复多次得到时间t下最优的空文本，然后使用优化后的空文本传入扩散模型，得到时间t的采样结果，即时间t-1扩散模型的输入。同时将优化后的空文本作为时间t-1扩散模型的空文本输入，继续优化时间t-1的空文本嵌入**。最后使用DDIM inversion得到的噪声<img src="C:\Users\李博\Desktop\mynotes\ai\stable diffusion.assets\image-20231106162015593.png" alt="image-20231106162015593" style="zoom:67%;" />和优化后的空文本嵌入序列<img src="C:\Users\李博\Desktop\mynotes\ai\stable diffusion.assets\image-20231106161920691.png" alt="image-20231106161920691" style="zoom:67%;" />作为扩散模型的输入，对真实图像进行编辑。

  - 我们定义<img src="C:\Users\李博\Desktop\mynotes\ai\stable diffusion.assets\image-20231106161920691.png" alt="image-20231106161920691" style="zoom:67%;" />是从时间t=1到T的每一步的空文本嵌入组成的序列。

  - 原来的ldm只能有一个全局的空文本，即每一个采样步骤都使用相同的空文本embeding；但是作者发现为每一步定义自己的空文本（即每一步都对空文本的embeding独立进行优化）能得到更好的结果。

![image-20231104223433555](C:\Users\李博\Desktop\mynotes\ai\stable diffusion.assets\image-20231104223433555.png)

<img src="C:\Users\李博\Desktop\mynotes\ai\stable diffusion.assets\image-20231105150017232.png" alt="image-20231105150017232" style="zoom:67%;" />