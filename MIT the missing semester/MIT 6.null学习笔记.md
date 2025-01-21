# MIT 6.null学习笔记

## 1、shell

命令行界面（Command-line Interface）是在图形用户界面得到普及之前使用最为广泛的用户界面，它通常不支持鼠标，**用户通过键盘输入指令，计算机接收到指令后，予以执行**。

**终端（Terminal）**：早期的计算机非常昂贵，非常庞大。它们通常放置在单独的房间内，而操作计算机的人坐在另外的房间里，通过某些设备和计算机进行交互，这些设备就叫做**终端（Terminal）**，也叫做终端机，早期的终端是一种可以连接到计算机上的带输入输出功能的外设，供普通的用户使用

**控制台（Console）**：早期和计算机是一体的，算是计算机的一个组成部分，供系统管理员使用，有着比普通终端更大的权限

在远古的 Unix 大型机时代，**console 是指物理连接在主机上的输入输出设备，而 terminal 是指与 console 进行远程通信的串行设备**。而随着计算机的发展，控制台和终端的概念已经逐渐模糊，Console 与 Terminal 基本可以看作是同义词。在linux上的terminal实际上是一个terminal emulator，不是远古意义上的终端。

而当我们谈到命令行时，我们实际上是指 **shell**。 shell是人与电脑的接口，实际上是个**命令解释程序**，它从键盘读取命令然后交由操作系统来执行。linux下的bash（即Bourne-Again SHell的缩写），windows下的cmd，powershell都属于shell。

**终端与 Shell 的区别**：终端是从用户（通过键盘和鼠标）这里接收输入，然后扔给 Shell，然后把 Shell 返回的结果展示给用户（通过显示器）。Shell 是从终端那里拿到用户输入的命令，解析后交给操作系统内核去执行，然后把执行结果返回给终端。**我们需要通过terminal来操作shell。**

### 使用 shell

可以输入 命令 ，命令最终会被 shell 解析

```bash
# echo命令打印后面的参数
missing:~$ echo hello
hello
```

**shell 基于空格分割命令并进行解析，然后执行第一个单词代表的程序，并将后续的单词作为程序可以访问的参数。**

shell中名字的空格用`\`后加空格表示

```bash
$ mkdir my photo # 错误，只会创建两个目录
# 正确写法：
$ mkdir my\ photo
$ mkdir "my photo"
```

当你在 shell 中执行命令时，您实际上是在执行一段 shell 可以解释执行的简短代码。如果你要求 shell 执行某个指令，但是该指令并不是 shell 所了解的编程关键字，那么它会去咨询 *环境变量* `$PATH`，它会在每个目录中查找名称与你尝试运行的指令匹配的程序或文件，然后运行它

```bash
# 列出当 shell 接到某条指令时，进行程序搜索的路径
missing:~$ echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
# 确定某个程序名代表的是哪个具体的程序，可以使用 which 程序
missing:~$ which echo
/bin/echo
# 我们也可以绕过 $PATH，通过直接指定需要执行的程序的路径来执行该程序
missing:~$ /bin/echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

**选项分为长选项和短选项**：

- **短选项：比如-h，-l，-s等。(-  后面接单个字母)**
  - 短选项都是使用‘-’引导，当有多个短选项时，各选项之间使用空格隔开。
  - 有些命令的短选项可以组合，比如-l –h 可以组合为–lh
  - 有些命令的短选项可以不带-，这通常叫作BSD风格的选项，比如ps aux
  - 有些短选项需要带选项本身的参数，比如-L 512M

- **长选项：比如--help，--list等。(-- 后面接单词)**
  - 长选面都是完整的单词
  - 长选项通常不能组合
  - 如果需要参数，长选项的参数通常需要‘=’，比如--size=1G

```bash
missing:~$ ls -l /home
drwxr-xr-x 1 missing  users  4096 Jun 15  2019 missing
```

第一个字符 `d` 表示 `missing` 是一个目录。然后接下来的九个字符，每三个字符构成一组。 它们分别代表了文件所有者（`missing`），用户组（`users`） 以及其他所有人具有的权限。中 `-` 表示该用户不具备相应的权限。

以上三种权限对文件和目录的含义不同：

- 文件：r表示可以被读取，w表示可以写入，可以保存文件，x表示可以执行

  在`usr/bin`目录下使用`ls -l`，会发现所有文件都设置了执行位（x），即便对于不是文件所有者的人也是如此。因为我们希望这些命令每个人都可以在计算机上运行

  ![image-20211008153133093](https://raw.githubusercontent.com/BoL0150/image2/master/image-20211008153133093.png)

- 目录：

  - r表示是否允许查看目录中的文件，所以r权限位相当于是否允许使用ls
  - w表示是否允许重命名、创建或删除该目录中的文件
  - x表示是否允许在该目录下搜索（比如cd）

![image-20211008154256327](https://raw.githubusercontent.com/BoL0150/image2/master/image-20211008154256327.png)

rm默认不是递归的，所以我们无法用rm删除目录，需要使用参数-rf指明递归删除目录。

### 在程序间创建连接

最简单的重定向是 `< file` 和 `> file`。这两个命令可以**将程序的输入输出流分别重定向到文件**：

```
missing:~$ echo hello > hello.txt
missing:~$ cat hello.txt
hello
missing:~$ cat < hello.txt
hello
missing:~$ cat < hello.txt > hello2.txt
missing:~$ cat hello2.txt
hello
```

您还可以使用 `>>` 来向一个文件追加内容。

![image-20211008155436378](https://raw.githubusercontent.com/BoL0150/image2/master/image-20211008155436378.png)

使用管道（ *pipes* ），我们能够更好的利用文件重定向。 `|` 操作符允许我们将一个**程序**的输出和另外一个**程序**的输入连接起来。

我们使用`ls -l /`会打印`/`目录下的所有文件，如果我们只想要最后一排输出，可以使用`tail`命令，`tail`命令打印其输入的最后一行。可以使用管道将以上两个命令结合起来：

![image-20211008160159523](https://raw.githubusercontent.com/BoL0150/image2/master/image-20211008160159523.png)



### curl

我们可以使用以下命令获取访问百度响应的http头

```bash
$ curl --head --silent baidu.com
```

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211008171416933.png" alt="image-20211008171416933" style="zoom:50%;" />

grep的-i参数表示忽略大小写（ignore），使用以下命令获取响应头中的content-length参数

```bash
$ curl --head --silent baidu.com | grep -i content-length
```

![image-20211008171745235](https://raw.githubusercontent.com/BoL0150/image2/master/image-20211008171745235.png)

`cut --delimiter=' '`表示将分隔符设置为空格来切割输入，-f2表示获取切割后的第二个字段（filed 2），

```bash
$ curl --head --silent baidu.com | grep -i content-length | cut --delimiter=' ' -f2
```

![image-20211008172031114](https://raw.githubusercontent.com/BoL0150/image2/master/image-20211008172031114.png)



### sys

有一件事情是您必须作为根用户才能做的，那就是向 `sysfs` 文件写入内容。系统被挂载在 `/sys` 下，`sysfs` 文件则暴露了一些内核（kernel）参数。 因此，您不需要借助任何专用的工具，就可以轻松地在运行期间配置系统内核。

![image-20211008173229808](https://raw.githubusercontent.com/BoL0150/image2/master/image-20211008173229808.png)

我们发现，即使加上sudo也无法向向brightness文件中写入。`|`、`>`、和 `<` 是通过 shell 执行的，而不是被各个程序单独执行。 `echo` 等程序并不知道 `|` 的存在，它们只知道从自己的输入输出流中进行读写。

这条命令的含义是：我告诉shell，运行sudo这个程序，参数是echo 500，sudo代表根用户执行echo 500。shell负责打开brightness文件，将输出重定向至此文件。所以在这种情况下，**是shell负责打开brightness文件，而不是sudo**。

所以解决方式是：

```bash
$ echo 3 | sudo tee brightness
```

`tee`指令会从标准输入设备读取数据，将其内容输出到标准输出设备，同时保存成文件。

使用指令"tee"将用户输入的数据同时保存到文件"file1"和"file2"中，输入如下命令：

```bash
$ tee file1 file2                   #在两个文件中复制内容 
```

以上命令执行后，将提示用户输入需要保存到文件的数据，如下所示：

```
My Linux                        #用户输入数据  
My Linux                        #输出数据，进行输出反馈  
```

此时，可以分别打开文件"file1"和"file2"，查看其内容是否均是"My Linux"即可判断指令"tee"是否执行成功。

### chmod

chmod（英文全拼：change mode）命令是控制用户对文件的权限的命令

Linux/Unix 的文件调用权限分为三级 : 文件所有者（Owner）、用户组（Group）、其它用户（Other Users）。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/file-permissions-rwx.jpg" alt="img" style="zoom:67%;" />

- u 表示该文件的拥有者，g 表示与该文件的拥有者属于同一个群体(group)者，o 表示其他以外的人，a 表示这三者皆是。
- \+ 表示增加权限、- 表示取消权限、= 表示唯一设定权限。
- r 表示可读取，w 表示可写入，x 表示可执行，X 表示只有当该文件是个子目录或者该文件已经被设定过为可执行。

### HW

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211008195933855.png" alt="image-20211008195933855" style="zoom: 50%;" />

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211008200050708.png" alt="image-20211008200050708" style="zoom: 50%;" />

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211008195832935.png" alt="image-20211008195832935" style="zoom: 67%;" />

## 2、Shell 工具和脚本

在bash中为变量赋值的语法是`foo=bar`，访问变量中存储的数值，其语法为 `$foo`。 需要注意的是，`foo = bar` （使用空格隔开）是不能正确工作的，因为解释器会调用程序`foo` 并将 `=` 和 `bar`作为参数。 总的来说，在shell脚本中使用空格会起到分割参数的作用，有时候可能会造成混淆，请务必多加检查。

Bash中的字符串通过`'` 和 `"`分隔符来定义，但是它们的含义并不相同。以`'`定义的字符串为原义字符串，其中的变量不会被转义，而 `"`定义的字符串**会将变量值进行替换**

```bash
foo=bar
echo "$foo"
# 打印 bar
echo '$foo'
# 打印 $foo
```

`bash` 也支持函数，它可以接受参数并基于参数进行操作。下面这个函数是一个例子，它会创建一个文件夹并使用`cd`进入该文件夹。

```bash
mcd () {
    mkdir -p "$1"
    cd "$1"
}
```

这里 `$1` 是脚本的第一个参数（类似于main函数中的argv参数，也就是在shell中~~执行该sh文件时~~**调用该函数时的**第一个参数）。与其他脚本语言不同的是，bash使用了很多特殊的变量来表示参数、错误代码和相关变量。

- `$0` - 脚本名
- `$1` 到 `$9` - 脚本的参数。 `$1` 是第一个参数，依此类推。
- `$@` - 所有参数
- `$#` - 参数个数
- `$?` - 前一个命令的返回值
- `$$` - 当前脚本的进程识别码
- `!!` - 完整的上一条命令，包括参数。常见应用：当你因为权限不足执行命令失败时，可以使用 `sudo !!`再尝试一次。
- `$_` - 上一条命令的最后一个参数。如果你正在使用的是交互式shell，你可以通过按下 `Esc` 之后键入 . 来获取这个值。

source是一个Shell内置命令，用以在当前上下文（当前shell环境）中执行某文件中的一组命令，source命令可简写为一个点（.）

使用了`source mcd.sh`后，mcd.sh文件中的命令被执行，我们的shell中已经定义了mcd函数，所以我们现在可以调用该函数

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211008212202422.png" alt="image-20211008212202422" style="zoom:50%;" />

命令通常使用 `STDOUT`来返回输出值，使用`STDERR` 来返回错误及错误码，便于脚本以更加友好的方式报告错误。 返回码或退出状态是脚本/命令之间交流执行状态的方式。返回值0表示正常执行，其他所有非0的返回值都表示有错误发生。

退出码可以搭配`&&` (与操作符) 和 `||` (或操作符)使用，用来进行条件判断，决定是否执行其他程序。它们都属于短路运算符（short-circuiting）。

同一行的多个命令可以用` ; `分隔。程序 `true` 的返回码永远是`0`，`false` 的返回码永远是`1`。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211008213448476.png" alt="image-20211008213448476" style="zoom: 33%;" />

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211008213658868.png" alt="image-20211008213658868" style="zoom: 33%;" />

另一个常见的模式是**以变量的形式获取一个命令的输出**，这可以通过 *命令替换* (*command substitution*)实现。

当您通过 `$( CMD )` 这样的方式来执行`CMD` 这个命令时，它的输出结果会替换掉 `$( CMD )` 。例如，如果执行 `for file in $(ls)` ，shell首先将调用`ls` ，然后遍历得到的这些返回值。

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20211008214825864.png" alt="image-20211008214825864" style="zoom: 33%;" />

还有一个冷门的类似特性是 *进程替换*（*process substitution*）， `<( CMD )` 会执行 `CMD` 并将结果输出到一个临时文件中，并将 `<( CMD )` 替换成临时文件名。这在我们希望返回值通过文件而不是STDIN传递时很有用。例如， `diff <(ls foo) <(ls bar)` 会显示文件夹 `foo` 和 `bar` 中文件的区别。

这段脚本会遍历我们提供的参数，使用`grep` 搜索字符串 `foobar`，如果没有找到，则将其作为注释追加到文件中。

```shell
#!/bin/bash

echo "Starting program at $(date)" # date会被替换成日期和时间

echo "Running program $0 with $# arguments with pid $$"

for file in "$@"; do
    grep foobar "$file" > /dev/null 2> /dev/null
    # 如果模式没有找到，则grep退出状态为 1
    # 我们将标准输出流和标准错误流重定向到Null，因为我们并不关心这些信息
    if [[ $? -ne 0 ]]; then
        echo "File $file does not have any foobar, adding one"
        echo "# foobar" >> "$file"
    fi
done
```

在条件语句中，我们比较 `$?` 是否等于0。 在bash中进行比较时，尽量使用双方括号 `[[ ]]` 而不是单方括号 `[ ]`，这样会降低犯错的几率，

shell的 *通配*（ *globbing*）

- 通配符 - 当你想要利用通配符进行匹配时，你可以分别使用 `?` 和 `*` 来匹配一个或任意个字符。例如，对于文件`foo`, `foo1`, `foo2`, `foo10` 和 `bar`, `rm foo?`这条命令会删除`foo1` 和 `foo2` ，而`rm foo*` 则会删除除了`bar`之外的所有文件。

- 花括号`{}` - 当你有一系列的指令，其中包含一段公共子串时，可以用花括号来自动展开这些命令。这在批量移动或转换文件时非常方便。

  ```bash
  convert image.{png,jpg}
  # 会展开为
  convert image.png image.jpg
  
  cp /path/to/project/{foo,bar,baz}.sh /newpath
  # 会展开为
  cp /path/to/project/foo.sh /path/to/project/bar.sh /path/to/project/baz.sh /newpath
  
  # 也可以结合通配使用
  mv *{.py,.sh} folder
  # 会移动所有 *.py 和 *.sh 文件
  
  mkdir foo bar
  
  # 下面命令会创建foo/a, foo/b, ... foo/h, bar/a, bar/b, ... bar/h这些文件
  touch {foo,bar}/{a..h}
  touch foo/x bar/y
  # 比较文件夹 foo 和 bar 中包含文件的不同
  diff <(ls foo) <(ls bar)
  # 输出
  # < x
  # ---
  # > y
  ```

  

脚本并不一定只有用bash写才能在终端里调用。比如说，这是一段Python脚本，作用是将输入的参数倒序输出：

```bash
#!/usr/local/bin/python
import sys
for arg in reversed(sys.argv[1:]):
    print(arg)
```

内核知道去用python解释器而不是shell命令来运行这段脚本，是因为脚本的开头第一行的shebang。

在 `shebang` 行中使用 `env`命令是一种好的实践，`env` 会在`PATH` 环境变量中的目录中来搜索Python解释器，再用它来解释此文件，这样就提高来您的脚本的可移植性（不同的机器上的Python解释器可能在不同的位置）。例如，使用了`env`的shebang看上去时这样的`#!/usr/bin/env python`。

### 查找文件

程序员们面对的最常见的重复任务就是查找文件或目录。所有的类UNIX系统都包含一个名为 `find`的工具，它是shell上用于查找文件的绝佳工具。`find`命令会递归地搜索符合条件的文件

```bash
# 查找所有名称为src的文件夹
find . -name src -type d
# 查找所有文件夹路径中包含test的python文件
find . -path '*/test/*.py' -type f
# 查找前一天修改的所有文件
find . -mtime -1
# 查找所有大小在500k至10M的tar.gz文件
find . -size +500k -size -10M -name '*.tar.gz'
```

除了列出所寻找的文件之外，`find`还能对所有查找到的文件进行操作

```shell
# 删除全部扩展名为.tmp 的文件
find . -name '*.tmp' -exec rm {} \;
# 查找全部的 PNG 文件并将其转换为 JPG
find . -name '*.png' -exec convert {} {}.jpg \;
```

尽管 `find` 用途广泛，它的语法却比较难以记忆。例如，为了查找满足模式 `PATTERN` 的文件，您需要执行 `find -name '*PATTERN*'` (如果您希望模式匹配时是不区分大小写，可以使用`-iname`选项）

 `fd` 就是一个更简单、更快速、更友好的程序，它可以用来作为`find`的替代品。以模式`PATTERN` 搜索的语法是 `fd PATTERN`。

我们可以有更高效的方法，不要每次都搜索文件而是通过编译索引或建立数据库的方式来实现更加快速地搜索。

`locate` 使用一个由 `updatedb`负责更新的数据库，在大多数系统中 `updatedb` 都会通过 `cron`每日更新。而且，`find` 和类似的工具可以通过别的属性比如文件大小、修改时间或是权限来查找文件，`locate`则只能通过文件名。

### 查找代码

很多时候我们的目标其实是查看文件的内容，很多类UNIX的系统都提供了`grep`命令，它是用于对输入文本进行匹配的通用工具

`grep` 有很多选项，其中经常使用的有

-  `-C` ：获取查找结果的上下文（Context），举例来说， `grep -C 5` 会输出匹配结果前后五行。
- `-v` 将对结果进行反选（Invert），也就是输出不匹配的结果。

- 当需要搜索大量文件的时候，使用 `-R` 会递归地进入子目录并搜索所有的文本文件。

比较常用的替代品是 ripgrep (`rg`) ，因为它速度快，而且用法非常符合直觉。例子如下：

```bash
# 在Python文件中查找所有使用了 requests 库的文件
rg -t py 'import requests'
# 查找所有没有写 shebang 的文件（包含隐藏文件）
rg -u --files-without-match "^#!"
# 查找所有的foo字符串，并打印其之后的5行
rg foo -A 5
# 打印匹配的统计信息（匹配的行和文件的数量）
rg --stats PATTERN
```

### 查找 shell 命令

`history` 命令允许您以程序员的方式来访问shell中输入的历史命令。这个命令会在标准输出中打印shell中的里面命令。如果我们要搜索历史记录，则可以利用管道将输出结果传递给 `grep` 进行模式搜索。 `history | grep find` 会打印包含find子串的命令。 

对于大多数的shell来说，您可以使用 `Ctrl+R` 对命令历史记录进行回溯搜索。敲 `Ctrl+R` 后您可以输入子串来进行匹配，查找历史命令行。

**基于历史的自动补全**。 这一特性最初是由 fish shell 创建的，它可以根据您最近使用过的开头相同的命令，动态地对当前对shell命令进行补全。这一功能在 zsh 中也可以使用，它可以极大的提高用户体验。

### 文件夹导航

我们可以使用`fasd`和`autojump`这两个工具来查找最常用或最近使用的文件和目录。

Fasd 基于 *frecency*对文件和文件排序，也就是说它会同时针对频率（*frequency* ）和时效（ *recency*）进行排序。默认情况下，`fasd`使用命令 `z` 帮助我们快速切换到最常访问的目录。例如， 如果您经常访问`/home/user/files/cool_project` 目录，那么可以直接使用 `z cool` 跳转到该目录。对于 autojump，则使用`j cool`代替即可。

还有一些更复杂的工具可以用来概览目录结构，例如 `tree`, `broot` 或更加完整的文件管理器，例如 `nnn`或 `ranger`。

