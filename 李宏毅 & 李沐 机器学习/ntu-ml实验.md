![image-20230727221203492](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230727221203492.png)

Conda是开源的包和环境管理系统，比如Anaconda 

它的作用是：在使用Pip3包管理器安装PyTorch包时，会安装一大堆PyTorch所依赖的Python3包，这些包可能会和本地其它Python3开发项目的包发生冲突。为了避免可能出现的冲突，我们建议使用虚拟环境管理器（如conda、virtualenv、venv等）——虚拟环境管理器可以为电脑中你指定的Python项目创建一个独属它自己的“虚拟环境”。这样一来，不同项目的Python包就可以安装在不同的环境中，相互隔离，互不干扰。

每个环境在anaconda3下都有一个同名目录，里面存放在这个环境中安装的东西

可以使用 `conda info --envs`来查看所有环境的具体信息

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230828000235735.png" alt="image-20230828000235735" style="zoom:67%;" />



![image-20230728005025713](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230728005025713.png)

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20230728005047664.png" alt="image-20230728005047664" style="zoom:67%;" />

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20230728101905306.png" alt="image-20230728101905306" style="zoom:67%;" />

CUDA：一个并行计算平台和应用程序编程接口，允许软件使用 NVIDIA GPU。CUDA分为运行时API和驱动API，当我们说安装cuda时通常指的是cuda运行时

虚拟机和容器的区别：

**Hypervisor会将宿主机的硬件虚拟化，然后在虚拟出的硬件上运行虚拟机的操作系统**；而容器技术严格来说并不是虚拟化，没有客户机操作系统，是共享内核的。容器可以视为软件供应链的集装箱，能够把应用需要的运行环境、缓存环境、数据库环境等等封装起来。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20230728105546571.png" alt="image-20230728105546571" style="zoom:67%;" />

所以，本质上来说**vm是将硬件层及以上都虚拟化，而容器只虚拟化OS以上的软件层**（不包括OS）。所以vm的隔离性更好，容器的性能更好

## conda命令

进入conda虚拟环境：

```
conda create -n test_env python=3.8 numpy pandas # 指定新创建的环境的python版本和继承的包
conda activate test_env
conda deactivate
```

```
conda create --name my_clone --clone base
conda env list
conda list
conda env remove --name env_name
```

.

由于我们的pytorch安装在conda中，只有在conda中才能写代码和运行代码，退出conda后，之前虚拟环境中安装的包将**不再可被pip、python直接访问**；同样，实际环境中安装的软件包也不会对虚拟环境产生影响。

在vscode中使用ctrl shift + p选择python解释器，只有选择conda才能运行pytorch代码

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20230730175530167.png" alt="image-20230730175530167" style="zoom:67%;" />

pip install安装有时会很卡，解决方法是：使用国内的镜像源安装。在原来安装时在命令里加一个参数 -i，然后在i后面加国内镜像地址。

```
选择国内的镜像源列表如下：

清华源： https://pypi.tuna.tsinghua.edu.cn/simple/

阿里云： http://mirrors.aliyun.com/pypi/simple/

中国科技大学： https://pypi.mirrors.ustc.edu.cn/simple/
```

有时候pip装了一个包后依赖依然有问题，就可以使用conda再装一次（通常建议先在虚拟环境中使用conda）

conda安装命令：

```
conda install -c anaconda tqdm
```

有时候在终端安装了一个包后依赖依然有问题，就可以在notebook的代码块中再装一次

```
%pip install -U d2l -i https://pypi.tuna.tsinghua.edu.cn/simple/
```

Ubuntu 22.04安装python3.8

```
ubuntu 22.04默认是python 3.10,由于开发之前的代码代码导入包还是包里之前的版本，所以有import module not exist不匹配的问题，所以需要安装Python 3.8,

sudo apt update && sudo apt upgrade

sudo apt install software-properties-common -y

sudo add-apt-repository ppa:deadsnakes/ppa -y

sudo apt install python3.8 -y

python3.8 --version
```

Ubuntu修改默认Python版本（用户级

```
先查看系统中有那些Python版本：
$ ls /usr/bin/python*
再查看系统默认的Python版本：
$ python --version
为某个特定用户修改Python版本，只需要在其home目录下创建一个alias。
打开该用户的~/.bashrc文件：
vim ~/.bashrc
添加新的别名来修改默认Python版本：
alias python='/usr/bin/python3.5'
重新登录或者重新加载.bashrc文件，使操作生效：
bash
```

ubuntu深度学习环境配置

https://blog.csdn.net/weixin_41973774/article/details/117223425

之前配好的环境后来出问题了，最简单的方法是重新配一遍，不要想着把环境修好，这是非常困难的

# python

## python内置数据结构
python内置的数据结构有：列表、字典、集合、元组

- 列表相当于数据类型不唯一的数组
- 字典相当于数据类型不唯一的~~hashmap~~，而是相当于有序的map（key和value的类型都不唯一）
- 集合相当于数据类型不唯一的hashset，集合是无序的
- 以上三个数据结构都可变，而元组相当于不可变的列表

这些数据结构都可以用索引和切片

## values()

用于获取字典中所有数据的视图，可以通过迭代来访问字典中的所有值

## range

`range()` 是一个内置函数，用于生成一系列数字的序列。它常用于循环、迭代和生成整数序列。`range()` 函数有以下三种常见的语法形式：

1. **只提供结束值：**
   ```
   range(stop)
   ```
   这将生成一个从 0 开始，到 `stop - 1` 结束的整数序列。例如：`range(5)` 将生成序列 `[0, 1, 2, 3, 4]`。

2. **提供开始和结束值：**
   
   ```
   range(start, stop)
   ```
   这将生成一个从 `start` 开始，到 `stop - 1` 结束的整数序列。例如：`range(2, 7)` 将生成序列 `[2, 3, 4, 5, 6]`。
   
3. **提供开始、结束和步长值：**
   ```
   range(start, stop, step)
   ```
   这将生成一个从 `start` 开始，以步长 `step` 递增，直到不超过 `stop` 为止的整数序列。例如：`range(1, 10, 2)` 将生成序列 `[1, 3, 5, 7, 9]`。

需要注意的是，**`range()` 函数生成的序列并不是一个实际的列表，而是一个类似于序列的对象**，称为 **range object**。这是一种节省内存的方式，特别适用于大范围的数字序列。**如果需要将其转换为列表，可以使用 `list()` 函数来实现**，例如：`list(range(5))` 将返回 `[0, 1, 2, 3, 4]`。

以下是一些示例：

```python
# 生成从0到4的序列
for num in range(5):
    print(num)

# 生成从2到6的序列
for num in range(2, 7):
    print(num)

# 生成从1到9，步长为2的序列
for num in range(1, 10, 2):
    print(num)
```

使用 `range()` 函数可以方便地生成指定范围内的整数序列，用于循环、迭代和其他需要顺序访问数字的情况。

## enumerate

如果我们想在循环体中访问每个元素的下标，那么就使用enumerate函数

```py
animals = ['cat', 'dog', 'monkey']
for idx, animal in enumerate(animals):
    print('#%d: %s' % (idx + 1, animal))
# Prints "#1: cat", "#2: dog", "#3: monkey", each on its own line
```

## 列表推导式

如果我们想把一个列表中的元素做一些操作转换成另一个列表的元素，如：

```py
nums = [0, 1, 2, 3, 4]
squares = []
for x in nums:
    squares.append(x ** 2)
print(squares)   # Prints [0, 1, 4, 9, 16]
```

使用列表推导式可以使这种操作变得更简单：

```python
nums = [0, 1, 2, 3, 4]
squares = [x ** 2 for x in nums]
print(squares)   # Prints [0, 1, 4, 9, 16]
```

列表推导式也可以包含条件：

```python
nums = [0, 1, 2, 3, 4]
even_squares = [x ** 2 for x in nums if x % 2 == 0]
print(even_squares)  # Prints "[0, 4, 16]"
```

## 字典

`print('cat' in d)`是检查cat是否在字典中，返回的是一个bool值

```python
d = {'cat': 'cute', 'dog': 'furry'}  # Create a new dictionary with some data
print(d['cat'])       # Get an entry from a dictionary; prints "cute"
print('cat' in d)     # Check if a dictionary has a given key; prints "True"
d['fish'] = 'wet'     # Set an entry in a dictionary
print(d['fish'])      # Prints "wet"
# print(d['monkey'])  # KeyError: 'monkey' not a key of d
print(d.get('monkey', 'N/A'))  # Get an element with a default; prints "N/A"
print(d.get('fish', 'N/A'))    # Get an element with a default; prints "wet"
del d['fish']         # Remove an element from a dictionary
print(d.get('fish', 'N/A')) # "fish" is no longer a key; prints "N/A"
```

遍历字典：正常情况下我们只能遍历字典的Key

```python
d = {'person': 2, 'cat': 4, 'spider': 8}
for animal in d:
    legs = d[animal]
    print('A %s has %d legs' % (animal, legs))
```

如果我们想同时访问key和value，那么需要使用item方法：

```python
d = {'person': 2, 'cat': 4, 'spider': 8}
for animal, legs in d.items():
    print('A %s has %d legs' % (animal, legs))
# Prints "A person has 2 legs", "A cat has 4 legs", "A spider has 8 legs"
```

字典的推导式和列表一样，但是创建的是字典：

```python
nums = [0, 1, 2, 3, 4]
even_num_to_square = {x: x ** 2 for x in nums if x % 2 == 0}
print(even_num_to_square)  # Prints "{0: 0, 2: 4, 4: 16}"
```



## 切片和数组索引语法

Python的切片语法用于对序列（如字符串、列表、元组、数组等）进行切片操作，从而获取序列中的一部分元素。切片操作使用方括号`[]`和冒号`:`组合而成，具体的语法为：

```
sequence[start:stop:step]
```

- `start`：表示切片的起始索引（包含在切片内），默认为0。如果未指定，则从序列的开头开始切片。
- `stop`：表示切片的终止索引（不包含在切片内）。切片将一直持续到此索引之前的元素，**不包括该索引对应的元素**。如果未指定，则切片会一直延续到序列的末尾。
- `step`：表示切片的步长（即每隔多少个元素取一个元素）。默认为1，表示按顺序逐个取元素。如果指定为负数，则表示逆序取元素。

所以**切片操作是一个前闭后开的区间**，切片操作不会修改原始序列，而是返回一个新的切片对象。

在Python的切片语法中，负数用于表示索引从序列的末尾开始的位置。

假设有一个列表`my_list`，其元素依次为 `[10, 20, 30, 40, 50]`。

- `my_list[-1]` 将返回 `50`，即最后一个元素。
- `my_list[-2]` 将返回 `40`，即倒数第二个元素。
- `my_list[-3]` 将返回 `30`，即倒数第三个元素。

切片中的负数索引也可以与正数索引一起使用。例如，使用`my_list[-3:-1]`可以获取倒数第三个到倒数第二个元素的切片，结果为 `[30, 40]`。

下面是一些常见的切片示例：

1. 获取列表的前三个元素：`my_list[:3]`
2. 获取列表的第二个到第四个元素：`my_list[1:4]`
3. 获取字符串的后两个字符：`my_string[-2:]`（从倒数第二个开始，到最后一个）
4. 获取列表的偶数索引元素：`my_list[::2]`
5. 获取列表的逆序：`my_list[::-1]`

而`train_data[:, -1]`的具体解释如下：

- `train_data`: 这是一个数据集，是一个**二维数组**。
- `[:, -1]`: 这是切片语法与数组索引语法的结合。逗号的左边用来指定行，逗号的右边用来指定列。因此，`train_data[:, -1]`的含义是取`train_data`中的所有行（`:`表示所有行）和最后一列（`-1`表示最后一列）。

**在数据集中，通常最后一列是目标变量（标签），而其他列则是特征**。这样，通过`train_data[:, -1]`可以提取出数据集中所有样本的目标变量，并存储在一个单独的数组（或列表）中，如代码中的`y_train`。

```cpp
feat_idx = list(range(raw_x_train.shape[1])
return raw_x_train[:,feat_idx]
```

这句代码使用Python的`range`函数和`list`函数来生成一个包含一系列整数的列表。

1. `raw_x_train.shape[1]`：`raw_x_train`是一个二维数组（或矩阵），`.shape`是NumPy数组的属性，用于获取数组的形状（维度）。对于二维数组，`.shape`返回一个元组，元组中的第一个元素表示行数，第二个元素表示列数。因为我们使用了索引`[1]`，所以`raw_x_train.shape[1]`表示`raw_x_train`数组的列数。

2. `range(raw_x_train.shape[1])`：`range()`是Python的内置函数，它返回一个不可变的**数字序列**。当使用一个整数作为参数时，`range()`会**生成从0开始到指定整数之前（不包含指定整数）的一系列整数**。

3. `list(range(raw_x_train.shape[1]))`：将`range(raw_x_train.shape[1])`**生成的整数序列转换成一个列表**。这样得到的列表**包含了从0到`raw_x_train.shape[1] - 1`的一系列整数**，对应着`raw_x_train`数组的列索引。

并且**列表可以作为索引传递给NumPy数组的切片操作，用于提取数组的特定列**。

在返回语句中，`raw_x_train[:, feat_idx]`使用`feat_idx`列表作为索引，从`raw_x_train`中选择了对应列的子集。因为`feat_idx`是一个列表，可以作为索引传递给NumPy数组的切片操作，用于选择特定的列。这样，函数将根据选择的特征索引提取出相应的特征列，并返回这些特征列组成的新的数据矩阵。

##  在一个矩阵中每一行取一个值

这种语法常用于计算cross entropy：两个list对应的位置的两个数组成一个坐标对，根据这些坐标对从矩阵中选取数组成行向量

```py
# 每个example预测的真实类别在该example的预测结果中的位置
true_category_indices = [0,2]
predict_result = torch.tensor([[0.1, 0.3, 0.6], [0.3, 0.2, 0.5]])
predict_result[[0,1],true_category_indices]
# 更简单的方式
predict_result[torch.arrange(2),true_categotry_indices]
```

##  解包序列

`*layers` 是一个Python中的特殊语法，它用于解包一个序列（通常是元组或列表）中的元素，并将这些元素作为独立的参数传递给一个函数或构造函数。

比如这个函数：layer是一个list，将layer内的元素作为独立的参数传入Sequential中

```py
nn.Sequential(*layer)
```



## main

在C++中，include的文件在预处理时被展开到文件的头部，在python中，import的文件也是被整个展开到文件的头部，在C++中，一个项目只有一个main函数，只有main函数中的代码是可执行的；但是在python中，每个文件都可以有可执行的代码（就相当于main函数），所以为了避免import文件时不小心执行了其他文件的可执行代码，python引入了`__name__` 属性

`__name__` 属性是一个特殊的内置属性，用于确定一个模块是作为主程序运行还是被导入到其他模块中使用。

当一个 Python 脚本文件被执行时，解释器会将特殊的内置变量 `__name__` 设置为 `"__main__"`，以指示这个脚本是主程序，然后开始执行脚本中的代码。当这个脚本被导入为模块到其他脚本中时，`__name__` 的值将是模块的名称，而不是 `"__main__"`。

这个特性在许多情况下很有用，特别是当你**希望某些代码只在脚本作为主程序运行时才执行，而在作为模块被导入时不执行。**只要将所有文件的可执行代码都写入到`__name__` 下面，就可以实现C++的 一个项目中只有一个main函数的功能

举个例子，考虑一个名为 `example_module.py` 的模块：

```python
# example_module.py

def my_function():
    print("Function from example_module")

print("This will always be executed")

if __name__ == "__main__":
    print("This will only be executed if the module is run as the main program")
```

如果你直接运行 `example_module.py`，你会看到输出：

```
This will always be executed
This will only be executed if the module is run as the main program
```

但是，如果你在另一个脚本中导入了这个模块，例如 `main_script.py`：

```python
# main_script.py

import example_module

print("This is the main script")
```

运行 `main_script.py` 将会输出：

```
This will always be executed
This is the main script
```

可以看到，`example_module.py` 中的 `"This will only be executed if the module is run as the main program"` 部分在 `main_script.py` 中并未执行。这是因为在 `example_module.py` 中，使用了 `if __name__ == "__main__":` 条件来检查模块是否是作为主程序运行，从而控制代码的执行。

## python面向对象

类有一个名为 `__init__()` 的特殊方法（**构造方法**），该方法在类实例化时会自动调用

```python
class Complex:
    def __init__(self, realpart, imagpart):
        self.r = realpart
        self.i = imagpart
x = Complex(3.0, -4.5)
print(x.r, x.i)   # 输出结果：3.0 -4.5
```

类的方法与普通的函数只有一个特别的区别——它们必须有一个额外的**第一个参数名称**, 按照惯例它的名称是 self。self代表类的实例，而非类。**self代表当前对象的地址**，所以self实际上相当于java和C++的this指针，指向当前的对象。（self更类似于java中的this，python和java都没有显式的指针，所有的对象默认就是指针）。

我们也可以把self换成其他的名字，比如this，python会把方法的第一个参数认为是this指针，不管它叫什么名字。

python中类内的私有属性（或方法）用两个下划线（`__`）开头，声明该属性（或方法）为私有，不能在类的外部被使用或直接访问。

```py
class people:
    #定义基本属性
    name = ''
    age = 0
    #定义私有属性,私有属性在类外部无法直接进行访问
    __weight = 0
    #定义构造方法
    def __init__(self,n,a,w):
        self.name = n
        self.age = a
        self.__weight = w
    def speak(self):
        print("%s 说: 我 %d 岁。" %(self.name,self.age))
```

子类继承父类时，可以调用父类的`__init__`方法，也可以重写父类的方法

```py
#类定义
class people:
    #定义基本属性
    name = ''
    age = 0
    #定义私有属性,私有属性在类外部无法直接进行访问
    __weight = 0
    #定义构造方法
    def __init__(self,n,a,w):
        self.name = n
        self.age = a
        self.__weight = w
    def speak(self):
        print("%s 说: 我 %d 岁。" %(self.name,self.age))
 
#单继承示例
class student(people):
    grade = ''
    def __init__(self,n,a,w,g):
        #调用父类的构函
        people.__init__(self,n,a,w)
        self.grade = g
    #覆写父类的方法
    def speak(self):
        print("%s 说: 我 %d 岁了，我在读 %d 年级"%(self.name,self.age,self.grade))
 
 
s = student('ken',10,60,3)
s.speak()
```

### 类的专有方法：

- **`__init__` :** 构造函数，在生成对象时调用
- **`__del__` :** 析构函数，释放对象时使用
- **`__repr__` :** 打印，转换
- **`__setitem__` :** 按照索引赋值
- **`__getitem__`:** 按照索引获取值
- **`__len__`:** 获得长度
- **`__cmp__`:** 比较运算
- **`__call__`:** 函数调用
- **`__add__`:** 加运算
- **`__sub__`:** 减运算
- **`__mul__`:** 乘运算
- **`__truediv__`:** 除运算
- **`__mod__`:** 求余运算
- **`__pow__`:** 乘方

### 可调用对象
在python中除了函数之外，调用运算符还可以用在其他对象上。
- 类：调用类时会先调用__new__方法创建一个实例，然后运行__init__方法初始化实例，最后把实例返回给调用方，因为python没有new运算符，所以调用类就相当于调用函数
- 类的实例：如果类定义了__call__方法，那么它的实例可以作为函数调用
比如Loss类的实例，调用它就默认调用Loss实例的forward函数 

如果想判断一个对象能否调用，可以使用内置的callable函数。
### python参数传递
在python中参数传递是引用传递，但是与C++的引用传递有些不同：
- 可变对象会保留修改：如果传递的是可变对象，如list，字典，集合，函数内部的修改在外部是可见的，因为它们引用的是同一个对象
- 不可变对象不会影响外部：如果传递的是不可变对象，如元组、数字、字符串等，函数内部对这些对象的修改实际上是创建了一个新的对象，不会影响到外部
## 文件操作

`with open(xxx, 'w') as f:` 是 Python 中用于打开文件的一种常用语法结构。它通常用于以一种更安全和更简洁的方式处理文件的读写操作。这种语法结构称为 "上下文管理器"（Context Manager）。

具体解释如下：

- `with`: `with` 是 Python 的关键字，**用于创建一个上下文环境，在这个环境中执行特定操作，通常是为了确保资源在使用完毕后被正确释放**。
- `open(xxx, 'w')`: 这是一个内置函数 `open`，用于打开文件。`xxx` 是文件的路径（包括文件名），`'w'` 表示以写入模式打开文件。如果文件不存在，则会创建一个新文件。如果文件已经存在，原有内容将被清空。
- `as f`: 这里的 `as f` 是为打开的文件对象指定一个名称，这里命名为 `f`。通过这个名称，你可以在 `with` 块中使用这个文件对象进行读写操作。

整个语法结构的作用是：**当进入 `with` 块时，文件被打开并分配给变量 `f`。在块内部，你可以使用 `f` 进行文件的读写操作。当退出 `with` 块时，无论是因为代码块执行结束还是因为出现异常，都会自动关闭文件，确保资源得到正确释放，避免内存泄漏和资源泄漏问题。**

**文件在代码块外部会被自动关闭，不需要手动关闭文件**

## permute

返回一个新的张量，其中各个维度的顺序被重新排列

```py
import torch
# 创建一个形状为 (3, 224, 224) 的示例图像张量
image = torch.randn(3, 224, 224)
# 使用 permute 将通道维度移到最后
permuted_image = image.permute(1, 2, 0)
print(permuted_image.shape)  # 输出：torch.Size([224, 224, 3])

```

## collections.Counter

collections.Counter用来计算可迭代对象（如列表、元组、字符串等）中各元素出现次数，并且以字典的形式返回这些计数信息

# numpy

我们可以使用python list初始化numpy的数组

```py
a = np.array([1, 2, 3])   # Create a rank 1 array
print(type(a))            # Prints "<class 'numpy.ndarray'>"
print(a.shape)            # Prints "(3,)"

b = np.array([[1,2,3],[4,5,6]])    # Create a rank 2 array
print(b.shape)                     # Prints "(2, 3)"
print(b[0, 0], b[0, 1], b[1, 0])   # Prints "1 2 4"

a = np.zeros((2,2))   # Create an array of all zeros
print(a)              # Prints "[[ 0.  0.]
                      #          [ 0.  0.]]"

b = np.ones((1,2))    # Create an array of all ones
print(b)              # Prints "[[ 1.  1.]]"

c = np.full((2,2), 7)  # Create a constant array
print(c)               # Prints "[[ 7.  7.]
                       #          [ 7.  7.]]"

d = np.eye(2)         # Create a 2x2 identity matrix
print(d)              # Prints "[[ 1.  0.]
                      #          [ 0.  1.]]"

e = np.random.random((2,2))  # Create an array filled with random values
print(e)                     # Might print "[[ 0.91940167  0.08143941]
                             #               [ 0.68744134  0.87236687]]"
```

一个数组的切片是原始数据的一个视图，所以修改切片也会导致原始数据的修改

我们可以同时使用整数索引和切片索引，但是**整数索引会导致数组的维度降低**，而切片索引不会改变原数组的维度

```python
# Create the following rank 2 array with shape (3, 4)
# [[ 1  2  3  4]
#  [ 5  6  7  8]
#  [ 9 10 11 12]]
a = np.array([[1,2,3,4], [5,6,7,8], [9,10,11,12]])

# Two ways of accessing the data in the middle row of the array.
# Mixing integer indexing with slices yields an array of lower rank,
# while using only slices yields an array of the same rank as the
# original array:
row_r1 = a[1, :]    # Rank 1 view of the second row of a
row_r2 = a[1:2, :]  # Rank 2 view of the second row of a
print(row_r1, row_r1.shape)  # Prints "[5 6 7 8] (4,)"
print(row_r2, row_r2.shape)  # Prints "[[5 6 7 8]] (1, 4)"

# We can make the same distinction when accessing columns of an array:
col_r1 = a[:, 1]
col_r2 = a[:, 1:2]
print(col_r1, col_r1.shape)  # Prints "[ 2  6 10] (3,)"
print(col_r2, col_r2.shape)  # Prints "[[ 2]
                             #          [ 6]
                             #          [10]] (3, 1)"
```

整数数组索引：

```py
a = np.array([[1,2], [3, 4], [5, 6]])

# An example of integer array indexing.
# The returned array will have shape (3,) and
print(a[[0, 1, 2], [0, 1, 0]])  # Prints "[1 4 5]"

# The above example of integer array indexing is equivalent to this:
print(np.array([a[0, 0], a[1, 1], a[2, 0]]))  # Prints "[1 4 5]"

# When using integer array indexing, you can reuse the same
# element from the source array:
print(a[[0, 0], [1, 1]])  # Prints "[2 2]"

# Equivalent to the previous integer array indexing example
print(np.array([a[0, 1], a[0, 1]]))  # Prints "[2 2]"
```

**使用整数数组索引从原数组的每一行选取一个数**，并且还可以对其进行修改

```python
# Create a new array from which we will select elements
a = np.array([[1,2,3], [4,5,6], [7,8,9], [10, 11, 12]])

print(a)  # prints "array([[ 1,  2,  3],
          #                [ 4,  5,  6],
          #                [ 7,  8,  9],
          #                [10, 11, 12]])"

# Create an array of indices
b = np.array([0, 2, 0, 1])

# Select one element from each row of a using the indices in b
print(a[np.arange(4), b])  # Prints "[ 1  6  7 11]"

# Mutate one element from each row of a using the indices in b
a[np.arange(4), b] += 10

print(a)  # prints "array([[11,  2,  3],
          #                [ 4,  5, 16],
          #                [17,  8,  9],
          #                [10, 21, 12]])
```

这种语法可以用于softmax

```python
# 每个example预测的真实类别在该example的预测结果中的位置
true_category_indices = [0,2]
predict_result = torch.tensor([[0.1, 0.3, 0.6], [0.3, 0.2, 0.5]])
predict_result[[0,1],true_category_indices]
# 更简单的方式
predict_result[torch.arrange(2),true_categotry_indices]
```

布尔值数组索引：通常用于选取一个数组中满足某些条件的元素

```python
a = np.array([[1,2], [3, 4], [5, 6]])

bool_idx = (a > 2)   # Find the elements of a that are bigger than 2;
                     # this returns a numpy array of Booleans of the same
                     # shape as a, where each slot of bool_idx tells
                     # whether that element of a is > 2.

print(bool_idx)      # Prints "[[False False]
                     #          [ True  True]
                     #          [ True  True]]"

# We use boolean array indexing to construct a rank 1 array
# consisting of the elements of a corresponding to the True values
# of bool_idx
print(a[bool_idx])  # Prints "[3 4 5 6]"

# We can do all of the above in a single concise statement:
print(a[a > 2])     # Prints "[3 4 5 6]"
```

numpy的数组只能放相同类型的元素，与python的list不同。当numpy在创建一个数组时会根据数据来猜测类型，但是我们也可以显式地指定一个数据类型

```python
x = np.array([1, 2])   # Let numpy choose the datatype
print(x.dtype)         # Prints "int64"

x = np.array([1.0, 2.0])   # Let numpy choose the datatype
print(x.dtype)             # Prints "float64"

x = np.array([1, 2], dtype=np.int64)   # Force a particular datatype
print(x.dtype)                         # Prints "int64"
```

numpy可以对数组中的元素做elementwise的运算

```py
print(x + y)
print(np.add(x, y))

print(x - y)
print(np.subtract(x, y))

print(x * y) # 注意，*不是矩阵乘法，而是elementwise相乘，矩阵乘法是np.dot(x,y)或者x.dot(y)
print(np.multiply(x, y))

print(x / y)
print(np.divide(x, y))

print(np.sqrt(x))
```

其他操作：

```python
x = np.array([[1,2],[3,4]])

print(np.sum(x))  # Compute sum of all elements; prints "10"
print(np.sum(x, axis=0))  # Compute sum of each column; prints "[4 6]"
print(np.sum(x, axis=1))  # Compute sum of each row; prints "[3 7]"

print(x.T)  # Prints "[[1 3]
            #          [2 4]]"

```

## 广播

```py
# We will add the vector v to each row of the matrix x,
# storing the result in the matrix y
x = np.array([[1,2,3], [4,5,6], [7,8,9], [10, 11, 12]])
v = np.array([1, 0, 1])
y = x + v  # Add v to each row of x using broadcasting
print(y)  # Prints "[[ 2  2  4]
          #          [ 5  5  7]
          #          [ 8  8 10]
          #          [11 11 13]]"
```

广播的规则：比较两个数组的shape，从shape的**尾部**开始一一比对。

1. 如果两个数组的维度相同，**对应位置上轴的长度相同或其中一个的轴长度为1**,广播兼容，可在轴长度为1的轴上进行广播机制处理。
2. 如果两个数组的维度不同，那么给低维度的数组**前扩展**提升一维，扩展维的轴长度为1,然后在扩展出的维上进行广播机制处理（第一步的操作）。

```python
        # 对axis=1求和，得到的是一个行向量，即(500,)和(5000,)（注意！一维的向量的形状不能写成(500)！因为(500)不是元组，只是500加个括号而已，(500,)才能算元组，才能表示一个向量的形状！
        # 想让(500,)和(5000,)广播相加，只有先把(500,)转换成(500,1)，这样二者才能相加。同时，(5000,)不能转换成(5000,1)，这样的话二者的第一维就不匹配了，无法广播
        dists = np.square(X).sum(axis=1).reshape(-1,1) + np.square(self.X_train).sum(axis=1) - 2 * np.dot(X,self.X_train.T)
```

具体的广播过程是前者先从(500,1)扩展到(500,5000)，这样和(5000,)的最后一个维度就相同了，然后再把(5000,)扩展到(1,5000)，这样二者的维度数量就相同了，然后再把(1,5000)扩展到(500,5000)，最后二者相加，得到的矩阵中每一行是一个测试与所有训练图片的距离

## SciPy

scipy提供了一些对图片的基本操作，比如它可以将图片从磁盘中读到numpy的数组中，进行一些操作后再将numpy数组作为一张图片写回磁盘

```python
# Read an JPEG image into a numpy array
img = imread('assets/cat.jpg')
print(img.dtype, img.shape)  # Prints "uint8 (400, 248, 3)"

# We can tint the image by scaling each of the color channels
# by a different scalar constant. The image has shape (400, 248, 3);
# we multiply it by the array [1, 0.95, 0.9] of shape (3,);
# numpy broadcasting means that this leaves the red channel unchanged,
# and multiplies the green and blue channels by 0.95 and 0.9
# respectively.
# 牛逼的广播操作
img_tinted = img * [1, 0.95, 0.9]

# Resize the tinted image to be 300 by 300 pixels.
img_tinted = imresize(img_tinted, (300, 300))

# Write the tinted image back to disk
imsave('assets/cat_tinted.jpg', img_tinted)
```

## Matplotlib

```py
# Compute the x and y coordinates for points on sine and cosine curves
x = np.arange(0, 3 * np.pi, 0.1)
y_sin = np.sin(x)
y_cos = np.cos(x)

# Plot the points using matplotlib
plt.plot(x, y_sin)
plt.plot(x, y_cos)
plt.xlabel('x axis label')
plt.ylabel('y axis label')
plt.title('Sine and Cosine')
plt.legend(['Sine', 'Cosine'])
plt.show()
```

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230912160944313.png" alt="image-20230912160944313" style="zoom:67%;" />

### Subplots

我们可以在同时打印多张表

```python
# Compute the x and y coordinates for points on sine and cosine curves
x = np.arange(0, 3 * np.pi, 0.1)
y_sin = np.sin(x)
y_cos = np.cos(x)

# Set up a subplot grid that has height 2 and width 1,
# and set the first such subplot as active.
plt.subplot(2, 1, 1)

# Make the first plot
plt.plot(x, y_sin)
plt.title('Sine')

# Set the second subplot as active, and make the second plot.
plt.subplot(2, 1, 2)
plt.plot(x, y_cos)
plt.title('Cosine')

# Show the figure.
plt.show()
```

Matplotlib除了打印表之外还可以使用imshow函数打印图片：

```python
import numpy as np
from scipy.misc import imread, imresize
import matplotlib.pyplot as plt

img = imread('assets/cat.jpg')
img_tinted = img * [1, 0.95, 0.9]

# Show the original image，图片矩阵的形状是1*2，即一行有两张图。并且绘制第一张图
plt.subplot(1, 2, 1)
plt.imshow(img)

# Show the tinted image
plt.subplot(1, 2, 2)

# A slight gotcha with imshow is that it might give strange results
# if presented with data that is not uint8. To work around this, we
# explicitly cast the image to uint8 before displaying it.
plt.imshow(np.uint8(img_tinted))
plt.show()
```

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230912161737873.png" alt="image-20230912161737873" style="zoom:67%;" />

# PyTorch

![image-20230720232529813](C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230720232529813.png)

机器学习的前两步分别是写出预测函数和Loss函数，主要计算量集中在第三步的梯度下降中。我们使用pytorch来计算梯度下降。

PyTorch是一个python的机器学习框架，它的主要功能是：

- 可以在GPU上进行N维的张量计算（类似numpy）
- 可以在训练神经网络时进行梯度下降的自动微分

**首先通过深度学习来寻找（也可以叫定义）预测函数（带有未知参数），再定义Loss函数，然后由梯度下降来找出预测函数的最佳的参数。**这三个步骤合起来就是神经网络的训练。

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230720233445481.png" alt="image-20230720233445481" style="zoom:50%;" />

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230720233815608.png" alt="image-20230720233815608" style="zoom: 50%;" />

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230727115303111.png" alt="image-20230727115303111" style="zoom: 50%;" />

每一行代表一笔数据（sample），一笔数据中包含118个feature

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230727115318650.png" alt="image-20230727115318650" style="zoom: 50%;" />

x是输入，y就是label，x的shape是2699 * 117就表示有2699笔数据，每一笔数据有117个feature

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230727115513416.png" alt="image-20230727115513416" style="zoom: 50%;" />

## 数据操作

n维数组，也称为*张量*（tensor）

张量表示一个由数值组成的数组，这个数组可能有多个维度。 具有一个轴的张量对应数学上的*向量*（vector）； 具有两个轴的张量对应数学上的*矩阵*（matrix）； 具有两个轴以上的张量没有特殊的数学名称。

我们可以使用 `arange` 创建一个行向量 `x`，包含从0开始到x之前的x个整数



## 读入训练数据

使用pytorch训练神经网络第一步就是要将训练数据导入：

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230721001023855.png" alt="image-20230721001023855" style="zoom: 50%;" />

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230721001242757.png" alt="image-20230721001242757" style="zoom:50%;" />
### dataset
`TensorDataset` 和 `Dataset` 都是 PyTorch 中用于处理数据集的类，但它们在功能和使用上有一些区别。
- TensorDataset专门将多个张量（tensor）按照相同的索引进行对应，每个索引位置上的数据拼成一个数据样本，不同的张量按索引对应，形成feature-label对，从而创建一个数据集。这个适用于输入是tensor并且每个tensor之间是一一对应的情况。可以直接传入Dataloader

- DataSet是通用的数据集类，可以处理任意类型的数据，而不仅限于张量。在传入Dataloader之前需要重写`__len__`和`__getitem`方法来定义数据集的大小和数据的获取方式。

在实际使用中，你可以根据任务和数据的特点来选择适合的数据集类。如果你的数据和标签是以张量形式对应的，而且只需要进行基本的批次处理，那么使用 TensorDataset 是一个方便的选择。如果你需要更复杂的数据处理和变换，或者处理的数据不限于张量，那么使用继承自 Dataset 的自定义数据集类会更灵活

要导入dataset，首先要为dataset定义一个类，并且重写其中的函数

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230721001614987.png" alt="image-20230721001614987" style="zoom: 67%;" />

Dataloader中才可以调用dataset中的方法来获取指定大小的batch

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230721001706003.png" alt="image-20230721001706003" style="zoom:67%;" />

init负责把y和x变成floatTensor；getitem负责返回第idx笔数据	

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230727120206500.png" alt="image-20230727120206500" style="zoom:67%;" />

### tensor

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230721001854582.png" alt="image-20230721001854582" style="zoom: 50%;" />

维度

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230721001943909.png" alt="image-20230721001943909" style="zoom: 50%;" />

构造tensor

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230721002052063.png" alt="image-20230721002052063" style="zoom:67%;" />

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230721002135837.png" alt="image-20230721002135837" style="zoom:67%;" />

tensor的转置：下图的transpose让0维和1维互换

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230721002233057.png" alt="image-20230721002233057" style="zoom:50%;" />

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230721002414358.png" alt="image-20230721002414358" style="zoom:50%;" />

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230721002436504.png" alt="image-20230721002436504" style="zoom:50%;" />

tensor的拼接：

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230721002517239.png" alt="image-20230721002517239" style="zoom:50%;" />

tensor的数据类型：

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230725122406317.png" alt="image-20230725122406317" style="zoom:67%;" />

numpy的接口与pytorch类似：

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230725122512996.png" alt="image-20230725122512996" style="zoom: 50%;" />

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230725122605003.png" alt="image-20230725122605003" style="zoom: 50%;" />

pytorch中，Tensor默认使用CPU计算，我们可以指定使用哪个设备计算

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230725122751263.png" alt="image-20230725122751263" style="zoom: 50%;" />

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230725122944118.png" alt="image-20230725122944118" style="zoom:50%;" />

构造Tensor时要说明需要计算梯度（requires_grad=True）；右下角的2式是将x的每个分量相乘再相加，第三个式子对每个分量求偏导

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230725123737239.png" alt="image-20230725123737239" style="zoom: 50%;" />

## 定义神经网络

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230725124146867.png" alt="image-20230725124146867" style="zoom: 50%;" />

**使用`nn.Linear(in_features, out_features)`定义fully-connected层，也就是一个参数矩阵**

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230725124722937.png" alt="image-20230725124722937" style="zoom: 50%;" />

`nn.Linear(in_features, out_features)`定义出来的就是下图中的参数矩阵

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230725125814310.png" alt="image-20230725125814310" style="zoom:67%;" />

通过weight检查layer的参数，确实符合定义，bias也是一个64维的列向量

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230725125916958.png" alt="image-20230725125916958" style="zoom: 50%;" />

pytorch中也提供激活函数

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230725163609892.png" alt="image-20230725163609892" style="zoom:50%;" />

使用pytorch建立一个完整的神经网络如下：首先创建MyModel类，在构造函数init中定义net字段，**一个完整的net通常由多层网络组成**，首先定义一个10*32的全连接层，再定义一个sigmoid，获取全连接层的输出列向量，再定义一个32 * 1的全连接层，获取sigmoid的输出，最后得到一个数或者列向量。

**再定义MyModel类的forward函数，该函数接受一个feature输入，再将该feature丢到MyModel内的net中即可得到输出**。用户直接调用MyModel的forward函数即可。

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230725163813096.png" alt="image-20230725163813096" style="zoom: 50%;" />

也可以像下面这样一层层地定义网络：

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230725170155688.png" alt="image-20230725170155688" style="zoom:67%;" />

model的输入是input_dim，通过x的shape（得到是一个坐标，一维是x有多少笔数据，二维是一笔数据中有多少个feature）得到input_dim

![image-20230727120642527](C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230727120642527.png)



## 定义Loss函数

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230726110251936.png" alt="image-20230726110251936" style="zoom: 50%;" />

pytorch提供了适用于回归和分类任务的Loss函数，将模型的输出结果和预期结果输入到Loss函数中就可以计算得到loss值

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230726110417968.png" alt="image-20230726110417968" style="zoom:67%;" />

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230727121008234.png" alt="image-20230727121008234" style="zoom:50%;" />

## 优化算法

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230726110839528.png" alt="image-20230726110839528" style="zoom: 50%;" />

`torch.optim`提供了不同的基于梯度下降的优化算法，如果要使用最基本的梯度下降可以调用SGD（随机梯度下降）

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230726111108534.png" alt="image-20230726111108534" style="zoom: 50%;" />

**对每一个batch，先要把上一次迭代算过的梯度归零**，然后backward用来计算梯度，step用来更新参数

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230726111503614.png" alt="image-20230726111503614" style="zoom: 50%;" />

## 训练和测试的完整流程

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230726111745688.png" alt="image-20230726111745688" style="zoom: 50%;" />

神经网络训练的设置

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230726111807995.png" alt="image-20230726111807995" style="zoom:50%;" />

训练的过程：最外层是n个epoch，每个epoch中先把model设置为训练模式，再遍历所有的batch，每个batch先把之前迭代的梯度清零，再把数据移到指定的设备上（cpu或gpu），将数据输入model中得到输出，再定义loss函数，再计算梯度，再更新模型参数

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230726113747879.png" alt="image-20230726113747879" style="zoom: 50%;" />

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230727121116918.png" alt="image-20230727121116918" style="zoom:67%;" />

在进行validate之前，需要像把model调整为evaluation模式，因为对于有些操作比如batch normalization，它的训练阶段和validate阶段的操作是不一样的。然后在循环中将gradient计算禁用，因为validate和test都不需要计算梯度，再将计算平均的Loss（？）

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230726123427317.png" alt="image-20230726123427317" style="zoom:50%;" />

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230726123839633.png" alt="image-20230726123839633" style="zoom: 50%;" />

训练完后要保存模型

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230726123934528.png" alt="image-20230726123934528" style="zoom: 50%;" />

## 常见错误

model和输入的数据在不同的设备上

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230727135705091.png" alt="image-20230727135705091" style="zoom:50%;" />

Tensor的维度不匹配

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230727135853175.png" alt="image-20230727135853175" style="zoom:67%;" />

cudaOOM

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230727140058387.png" alt="image-20230727140058387" style="zoom:67%;" />

tensor的类型不匹配，label应该是long

<img src="C:\Users\李博\Desktop\mynotes\ai\ntu-ml实验.assets\image-20230727140432144.png" alt="image-20230727140432144" style="zoom:67%;" />

# hw1

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20230727115303111.png" alt="image-20230727115303111" style="zoom: 50%;" />

每一行代表一笔数据（sample），一笔数据中包含118个feature。一笔数据中包含一个one-hot 向量，这个向量包含37个州，表明这一笔数据是哪个州的过去五天的数据；然后对于训练数据后面有五天的统计数据，以及这五天分别的positive case；对于测试数据后面也是五天的统计数据，只不过第五天的数据没有positive case，需要我们自己预测。我们需要先根据训练数据训练出**每天**的调查数据和positive case之间的关系，再将测试数据中**第五天**的调查数据带入模型，预测出第五天的positive case

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20230727115318650.png" alt="image-20230727115318650" style="zoom: 50%;" />

x是输入，y就是label，x的shape是2699 * 117就表示有2699笔数据，每一笔数据有117个feature

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20230727115513416.png" alt="image-20230727115513416" style="zoom: 50%;" />



