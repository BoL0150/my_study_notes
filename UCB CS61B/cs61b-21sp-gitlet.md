# cs61b-21sp-gitlet

To compile the source file and all of its dependencies, run this command within your terminal:

```
$ javac *.java
```

**The `*.java` wildcard simply returns all the `.java` files in the current directory**. 

 `Main.java` is inside a package, so we must use it’s fully canonical name which is `capers.Main`. 

 to **run a Java file that is within a package**, we must enter the parent directory (in our case, `lab6`) and use the fully canonical name.

```bash
$ cd ..                 # takes us up a directory to sp21-s***/lab6
$ java capers.Main
```

One last thing about command line execution: how do we pass arguments to the `main` method? To pass arguments, simply add them in the call to `java`:

```bash
$ java capers.Main story "this is a single argument"
```

In the above execution, the `String[] args` variable had these contents:

```
{"story", "this is a single argument"}
```

You can make a File object in Java with the File constructor and passing in the path to the file:

```
File f = new File("dummy.txt");
```

**当我们通过File构造函数创建file对象时，我们并没有实际创造了一个dummy文件，这个file对象是对实际的dummy.txt文件的引用**，如果我们以后想对dummy文件进行操作，只需要对这个file对象进行操作即可。 

为了实际创建一个dummy文件，我们还需要对dummy的引用调用`createNewFile()`函数

```
f.createNewFile();
```

此时，dummy文件才真正被创建在文件系统中 

由于文件和目录本质上是一样的，如果想创建一个dummy目录，只需要对f调用`mkdir()`函数

```
f.mkdir();
```

You can check if the file “dummy.txt” already exists or not with the `exists` method of the File class:

```
f.exists()
```

Actually writing to a file is pretty ugly in Java. To keep things simple, we’ve provided you with a `Utils.java`.

As an example, if you want to write a String to a file, you can do the following:

```
Utils.writeContents(f, "Hello World");
```

Now `dummy.txt` would now have the text “Hello World” in it.

## Serialization 

Serialization is the process of **translating an object to a series of bytes that can then be stored in the file**.

We can then *deserialize* those bytes and get the original object back in a future invocation of the program.

To enable this feature for a given class in Java, **this simply involves implementing the `java.io.Serializable` interface**:

```
import java.io.Serializable;

public class Model implements Serializable {
    ...
}
```

To lower the amount of code you have to write，**we have provided helper function in `Utils.java` that handles reading and writing objects.**

```java
Model m;
File outFile = new File(saveFileName);

// Serializing the Model object
writeObject(outFile, m);

File inFile = new File(saveFileName);

// Deserializing the Model object
m = readObject(inFile, Model.class);
```

## git原理

**工作目录** 下文件的状态不外乎有两种：**已跟踪（tracked）** 或 **未跟踪（untracked）**。

**已跟踪文件** 是指那些 `git` 已经知道的文件。它们要么已经在上一次快照（**提交**）中，要么已经被 **暂存（staged）**

在git中，有三种对象：blob对象，tree对象，commit对象，这三种对象都以文件的形式存放在.git/object目录下，它们的文件名就是它们文件内容的sha-1哈希值，而它们对应的文件中的内容分别是：

- 文件的**内容**存储在 **blob** （二进制大对象）对象中

- **树对象（tree）** 相当于目录。一个 **树对象** 基本上就是一级目录，它引用着这一级目录下的文件和子目录。所以tree对象的文件内容是：该目录下的文件或子目录的文件名，以及它在.git中对应的对象的sha-1哈希值。

  ![image-20230206112931608](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230206112931608.png)

  - git中文件的元数据和数据是分离的，这种设计与文件系统类似。文件的数据存在blob中，而文件的元数据（如文件名）和对应的hash id存在tree对象中，由hash id找到对应的文件。
  - 文件系统的目录的数据块存放文件的dirent（即文件名和文件对应的inode#），与文件本身的数据块也是分离的，由inode#找到。

- 一个快照就是一个 **提交（commit）**。一个 **提交** 对象包括一个**指向主要 树对象（根目录）的指针**和一些像 **提交者**、**提交信息** 和 **提交时间** 这样的元数据。在大多数情况下，一个 **提交** 还会有一个或多个父 **提交**——之前的快照。

  ![image-20230206113346971](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230206113346971.png)

如果我们修改了1.txt，加一个感叹号——也就是把文件的内容由 `HELLO WORLD` 变为 `HELLO WORLD!`，即便如此小的改动，我们仍然需要将新的1.txt单独完整存储为一个blob。该blob对象的文件名也会被改成新的hash值，所以引用1.txt的tree对象的内容也会改变，也需要重新存储一个tree对象，该tree对象的hash值也会改变。从而所有更高级的tree对象以及commit对象都会被改变。这个commit之后的所有commit的id都会随之改变。

所以，**git通过这种类似默克尔树的方法，保证了历史记录不可篡改**。

**只要对象的内容改变了，就需要重新存储一个新的对象**

![image-20230206114114651](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230206114114651.png)

由于这次 **提交** 不是第一次 **提交**，所以它有一个父节点——**commit A1337**

`git` 是怎么知道我们当前所在的分支呢？答案是它维护了一个名为 `HEAD` 的特殊指针。通常情况下，`HEAD` 会指向一个分支指针，这个分支指针指向该分支的一个 **commit**

要将活动分支切换到另一个分支，只需要将HEAD指针指向另一个分支指针即可

<img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20230206114855021.png" alt="image-20230206114855021" style="zoom:50%;" />

如果在test分支上进行新的commit，只需要将test指针移动到新的commit上，HEAD指针仍然指向test

如果我们使用 `git checkout master` 回到 master 分支，我们就让 `HEAD` 的再次指向 `master` 了

### 具体实现

分支的实现：**HEAD存储分支指针文件路径，分支指针文件存储commit id**

- 分支指针在git仓库中以引用（refs）的形式存在。**分支** 是commit对象的命名引用，`git` 内部将 **分支** 称为 **heads**，所以分支指针是`.git/refs/heads/`下的一个文件，**文件名就是分支名，文件的内容是分支指针所指向的commit对象的hash id**。更新commit时，只需要将分支指针文件的内容改为新的commit的hash id即可。

- 而HEAD指针是.git/HEAD文件，**该文件的内容是HEAD所指向分支指针的文件路径**。切换分支时，只需要将HEAD文件的内容改为新的分支指针文件的路径即可。

  `$ echo "ref: refs/heads/master" > .git/HEAD`

当我们使用git add的时候就会将文件创建为blob对象，`git` 是为 *整个* 暂存的文件创建 **blob**。即使文件中只有修改或添加了一个字符，该文件也会有一个新的 **blob**，这个 **blob** 有着新的哈希值。

blob存在`.git/objects`目录下，`git` 实际上是使用 SHA-1 哈希值的前两个字符作为目录的名字，剩余字符用作 **blob** 所在文件的文件名。这种做法的原因是将文件分成256份，方便在大量文件中查找。

如果**blob** 的哈希值为 `54f6...36`， 就会在`.git\objects` 下创建一个名为 `54` 的目录，目录内有一个名为 `f6..36` 的文件。

暂存区（或索引）是.git/index文件，通过将**blob的hash id和文件名**写入index文件就相当于将该文件add入了暂存区

![image-20230206131453380](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230206131453380.png)

- 可能的实现方式：工作目录就是所有文件的根目录，而commit就是工作目录的一个快照，所有的commit一定指向代表工作目录的tree对象，~~而暂存区只是帮下一个commit对象持有工作目录的tree对象。所以将第一个文件加入暂存区的时候就应该创建工作目录的tree对象，随后所有add的文件，都由这个tree对象引用，index文件中只需要存放该tree对象的hash id和名字即可。commit是只需要创建一个commit对象引用暂存区中的tree对象，再清空index 文件的内容即可~~。

  此法不可行！不能在暂存区里用tree存储所有的add的文件blob，因为Tree的内容会随着add而变化，hashid也会随之变化！**所以暂存区只能存放所有文件的blob，只有在commit之后才能把暂存区中的blob挂到Tree上**！

如果要commit，就需要先创建一个tree对象，tree对象中记录index文件中的内容。然后再创建一个commit对象，引用刚才的tree对象，这个commit对象中要记录刚才的tree对象的hash id，parent commit id，作者以及comment。

![image-20230206132432801](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230206132432801.png)

- 包含parent commit id的commit对象：

  ![image-20230206132537917](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230206132537917.png)

创建一个新的分支，我需要在 `.git\refs\heads` 下创建一个名为 `test` 的文件，文件的内容应该和 `master` 分支指向的那个 **提交** 的哈希值一致，创建的分支默认指向原来的commit

![image-20230206131836513](https://raw.githubusercontent.com/BoL0150/image2/master/image-20230206131836513.png)

如果在test分支上进行新的commit，只需要将test指针（即`.git\refs\heads\test`）的内容改为新的commit的hash id，HEAD指针仍然指向test。

`git log` 找到 `HEAD`，`HEAD` 告诉它去 `test` 分支，`test` 分支指向着 **提交** `465...5e`，这个提交又链接到它的父 **提交** `80e...8f`。

### 代码实现

相同指的是id相同，即文件名和文件内容都相同

git add：

- 根据add的文件创建blob对象，如果文件不存在，输出`File does not exist.`。否则使用文件的内容和名字生成blob对象的hashId。

  - 如果当前的commit已经有相同的文件了，就不要将这个blob加入到index中，并且，如果index中还有这个blob（不管是addIndex还是removeIndex），就还要将这个blob从index中移除

  否则就可以将blob的id加入index中，再将blob对象写入object文件下

git commit message:

- 创建一个commit对象，通过HEAD获得active branch，再通过active branch获得parent commit id。先把前一个commit的map复制过来，再根据index的文件增删

  由add命令可知index中不会存在与commit中完全一样的文件，所以add有两种情况

  - 一种是add一个commit中不存在的新文件，这种情况可以直接把这个blob添加到commit中

  - 一种将commit的文件的内容进行修改后再add，这种情况要根据路径名把commit中的原文件映射删除后再替换成index中的文件。（父commit的映射依然存在）

  对于rm的情况，可以直接删除commit中相应的映射。然后再清空index

  完成了commit对象的实例化后，将commit对象写入object文件下。再更新branch指针指向新的commit

- 如果`index`中没有内容，输出错误信息`No changes added to the commit.`如果`message`为空，输出错误信息`Please enter a commit message`

git rm fileName：

如果文件既不在index中，也不在commit中，输出错误信息`No reason to remove the file.`（直接使用unix rm命令即可，没必要使用git rm）。除此之外的三种情况：

- 文件刚被`add`进`index`而没有`commit`，直接删除`addstage`中的Blob就可以。（除非文件被commit追踪，否则不要将它从工作目录中删除）
- 文件在commit中，并且在工作目录中，那么就将该文件放入removeIndex中并在工作目录中删除此文件
- 文件在commit中，不在工作目录中，那么就仅需将其放入`removeIndex`即可（`commit`之后手动删除文件，然后执行`rm`，第二次`rm`就对应这种情况）

注意！**rm的文件有可能就不存在在工作目录中**

文档完全没提有可能需要删除本地的blob，实际上这里应该存在bug，对于rm的情况1，是有可能需要删除本地的blob的

- 如果这个文件是第一次被加入到index中，需要删除本地的blob
- 如果不是，比如此文件在commit1时被加入到commit中，然后调用了git rm，在commit2时从commit2中删除，然后又被add到index中，再调用git rm，此时index和commit1引用了同一个blob，此时就不能删除本地的blob

所以严格来说，应该在blob对象上添加引用计数，当引用计数减为0的时候需要删除blob

git log：

- 从当前Commit开始倒着打印所有Commit信息，直到initCommit

- `merge commit`有两个`parents`，打印`merge commit`时只需要选择parentsList中的第一个即可(In regular Git, this is what you get with `git log --first-parent`)，因为**第一个parent是merge进行的时候所在的branch**，第二个parent是被合并的branch。merge commit的打印信息要在第一行下面再加一行，分别是第一个和第二个parent的commitId的前七个字母

  <img src="https://raw.githubusercontent.com/BoL0150/image2/master/image-20230208181937321.png" alt="image-20230208181937321" style="zoom:50%;" />

- 打印的内容有：commitId，commitTime，message。打印的信息前面有===，后面有一行空白

global-log：

打印所有的Commit而不关心顺序

find：

打印所有与输入message相同的Commit的ID，与上面思路类似，先读取所有的Commit，找到符合的，打印出ID即可，如果有多个结果，一行一个。

status：

打印以下信息

```bash
=== Branches ===
*master
other-branch

=== Staged Files ===
wug.txt
wug2.txt

=== Removed Files ===
goodbye.txt

=== Modifications Not Staged For Commit ===
junk.txt (deleted)
wug3.txt (modified)

=== Untracked Files ===
random.stuff
```

首先打印所有分支，读取`Heads`文件夹中的文件名即可，当前分支由`HEAD`文件中的内容确定，在前面加`*`号即可。

第二项读取所有保存在`addstage`中的文件名，打印出来即可。

第三项读取所有保存在`removestage`中的文件名，打印出来即可。

checkout

checkout有三种应用场景：

1. `checkout -- [filename]`将指定的文件回退到最新的commit版本：

   如果当前commit追踪的文件中包含fileName，就将其写入工作目录：如果同名文件存在，则overwrite；如果不存在，则直接写入

   如果当前的Commit中不存在filename的文件，输出`File does not exist in that commit.`

2. `checkout [commitID] -- [filename]`将fileName的文件回退到指定commitID的commit版本：

   如果不存在指定commitID的commit，则输出 `No commit with that id exists.`。如果该commit追踪的文件不包含fileName，则输出`File does not exist in that commit.`，如果包含，则可以将此文件写入到工作目录

3. `checkout [branchname]`从当前的branch切换到指定的branchName的branch：

   如果branchName不存在，输出`No such branch exists.`。如果branchname就是当前分支，输出`No need to checkout the current branch.`。
   把checkout branch（的commit）中的文件复制到工作目录：

   - 对于checkout branch（的commit）有，而当前branch没有的文件
     - 如果该文件不在当前branch的commit中，但是却在工作目录中，那么由于该文件在checkout branch中，所以即将被checkout覆盖，为了避免信息丢失，gitlet就会报错，输出`There is an untracked file in the way; delete it, or add and commit it first.`
     - 如果该文件不在工作目录中，那么直接写入工作目录
   - 对当前branch和checkout branch（的commit）中都有的文件（即名字相同的文件），直接用checkout branch中的文件覆盖
   - 对于当前branch比checkout branch多出来的文件，直接从工作目录中删除

   可以先遍历一遍当前branch commit，找出不在checkout branch 的commit中的文件，从工作目录中删除。

   再遍历一次checkout branch commit，

   - 对于不在当前branch的commit中，但是却在工作目录中的同名文件，输出报错信息；
   - 对于其他的文件，直接写入工作目录（覆盖或创造）

   最后将HEAD指向branchName，并且清空index

真实的checkout不会清空index，并且会把被checkout的文件加入index，并且当会覆盖文件或覆盖index时它也不会进行checkout

branch：

`branch [branchname]`

增加一个Branch，即在`heads`文件夹中添加一个新的名为branchname的文件，内容为当前的commitID。此操作不改变`HEAD`指向，只是单纯增加一个Branch。改变分支依然是通过`checkout`命令来改变。

如果该branchName已经存在，则报错：`A branch with that name already exists.`

rm-branch：

`rm-branch [branchname]`

删除一个Branch，不删除该branch下的commit。此Branch不能为现在的`HEAD`指向的Branch。具体操作为删除`heads`文件夹中的branchname文件即可。

**失败的情况**：如果给定的branchname不存在，输出`A branch with that name does not exist.`如果尝试删除的Branch为当前Branch，输出错误信息`Cannot remove the current branch.`

reset：

`reset [commitID]`将版本回滚到指定的commitID处

- 操作和checkout branch类似，checkout  branch是回退到指定branch的branch file指向的commit，而rest是直接回退到指定commit。

结束后将当前branch指针指向该commit，HEAD指针不变，还是指向当前branch

merge：

`merge [branchname]`将given branchName的branch合并到当前branch

split point是两条branch的最近公共祖先，~~使用leetcode经典方法找到该点~~，不能使用此方法，此方法只能用于两条链表，而git的分支图是一个有向无环图。可以先选一条branch BFS，将沿途的顶点加入set，再从另一条branch BFS，每经过一个点就在set中进行比较，发现相同的顶点就停止，这个点就是两条branch的最近公共祖先。

- 如果split point和given branch是同一个commit，说明当前branch和given branch在同一个分支上，并且领先于given branch，此时merge其实已经完成，直接输出`Given branch is an ancestor of the current branch.`
- 如果split point和当前branch是同一个commit，说明当前branch和given branch在同一个分支上，并且落后于given branch，此时直接将当前branch的branch file同步到指向given branch的commit即可，HEAD不变

然后分别用given branch和当前branch与split point的commit对比（以下修改过和没有修改过都是相对split point commit中的文件而言的）。gitlet的merge不负责产生新的commit？只是将merge的结果更新到工作目录，**merge后，如果工作目录的某个文件被更新，就要把该文件加入stage**，然后再由用户手动调用commit？

- 在given branch中修改过，而当前branch没有修改过的文件，变成在given branch中版本（新的commit引用given branch的版本，工作目录中的原文件也要被given branch的文件覆盖，将新的版本加入stage？）

- 在given branch中没被修改过，而在当前branch被修改过的文件，新的commit引用当前branch的版本，工作目录中的原文件保持不变

- 在两个branch中都改过，并且内容相同的文件（在split point中有，但是在两个branch中都删除了的文件也属于此种情况），merge后工作目录保持不变；

- 在两个branch（的commit）中都没有，但是在工作目录中却存在同名的文件，在工作目录中继续保留，既不提交到新的commit，也不提交到stage

- 在split point中没有，但是只在当前 branch中的文件，保留在工作目录中，新的commit引用该文件

- 在split point中没有，但是只在given branch中的文件，加入工作目录，加入stage，加入新的commit中

- 在split point中存在，在当前branch中未修改，在given branch中不存在的文件，从工作目录中移除，该文件加入removal index，新的commit不引用

- 在split point中存在，在given branch中未修改，在当前branch中不存在的文件，继续不存在于工作目录中

- **两个branch对同一个文件的修改不一致的情况**：

  - 在两个branch中都修改过，并且内容不同；
  - 或者一个branch修改过，一个branch中缺失该文件；
  - 或者split point缺失该文件，而两个branch中该文件的内容不同

  产生冲突要将先输出`Encountered a merge conflict.`再产生冲突的内容写入到发生冲突的文件中，格式如下

  ```text
  <<<<<<< HEAD
  contents of file in current branch
  =======
  contents of file in given branch
  >>>>>>>
  ```

  并且将结果stage。将被删除的文件视为空文件，可能产生如下的结果

  ```
  <<<<<<< HEAD
  contents of file in current branch=======
  contents of file in given branch>>>>>>>
  ```

merge 产生的commit记录当前branch为第一个parent，given branch为第二个parent

**失败的情况**：如果缓存区存在文件，输出`You have uncommitted changes.`如果给定的Branch不存在，输出`A branch with that name does not exist.`如果给定的Branch和当前Branch相同，输出`Cannot merge a branch with itself.`如果工作目录存在仅被merge commit跟踪，且将被覆写的文件，输出`There is an untracked file in the way; delete it, or add and commit it first.`

### Test

+号把右边的在src中的文件内容复制到左边的在临时目录中的文件，

=号是assert，判断左边的临时目录中的文件的内容是否等于右边的src目录中的文件内容

`E NAME` will assert that there exists a file/folder named `NAME` in the temporary directory. It doesn’t check the contents, only that it exists. 

`* NAME`will assert that there does NOT exist a file/folder named `NAME` in the temporary directory

If the `--keep` flag was provided, the temporary directory will remain, otherwise it will be deleted.