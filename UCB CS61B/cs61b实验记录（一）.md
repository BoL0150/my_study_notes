# cs61b实验记录（一）Lab2、Lab3

## Lab2

### Debug

- step into

  “step into” shows the **literal next step** of the program

- step over

  “step over” button allows us to **complete a function call without showing the function executing**

- step out

  escape the function

- resume重新开始，重新回到这个位置

  <img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210313171246942.png" alt="image-20210313171246942" style="zoom:67%;" />

- Conditional Breakpoints

  An even faster approach is to make our **breakpoint conditional**. To do this, right (or two-finger) click on the red breakpoint dot. Here, you can set a condition for when you want to stop. In the condition box, enter “newTotal < 0”, stop your program, and click “debug” again. 

  <img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210313171315988.png" alt="image-20210313171315988"  />

### Implementing Destructive vs. Non-destructive Methods

- `dcatenate(IntList A, IntList B)`: returns a list consisting of all elements of A, followed by all elements of B. May modify A. To be completed by you.

  错误：

  ![image-20210313171344357](https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210313171344357.png)

  通过临时对象A调用方法、属性是对原对象操作，但是给A赋值却只是改变了A的指向

  ![image-20210306155556156](https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210306155556156.png)

  最后将B赋值给A只是改变了A这个临时对象的指向，没有改变之前的对象。

  **所以我们需要将B和A的最后一个节点真正连接上，而不是对临时对象进行赋值**

  想要真正连接上，不能只是简单的将B赋值给A（像上面这样）

  ```java
  A=B;
  ```

  而是应该通过调用A这个临时对象的属性，从而访问到真正的对象，再将它们进行连接

  ```
  A.rest=B;
  ```

  因此，最直观的做法是

  <img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210306164105583.png" alt="image-20210306164105583" style="zoom: 50%;" />

  考虑到A一开始为null：

  <img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210306164151613.png" alt="image-20210306164151613" style="zoom: 50%;" />

  还可以采用更简洁的做法，在A的最后一个节点接上B后，再往前一个一个节点更新它们的链接

  <img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210306164539361.png" alt="image-20210306164539361" style="zoom: 50%;" />

  注意！不可这样写

  <img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210306164850163.png" alt="image-20210306164850163" style="zoom: 50%;" />

  返回的并不是A，而是A.rest !

- `catenate(IntList A, IntList B)`: returns a list consisting of all elements of A, followed by all elements of B. May not modify A. To be completed by you.

  obviously:

  <img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210306165050998.png" alt="image-20210306165050998" style="zoom:50%;" />

Autograder结果：

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210313175900622.png" alt="image-20210313175900622" style="zoom:50%;" />

## Lab3

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210313133753175.png" alt="image-20210313133753175" style="zoom:67%;" />

### 单向链表反转

```java
/**
 * Returns the reverse of the given IntList.
 * This method is destructive. If given null
 * as an input, returns null.
 */
public static IntList reverse(IntList A)
```

Your test should test at least the following three situations:

- That the function returns a reversed list.
- That the function is **destructive**, i.e. **when it is done running, the list pointed to by A has been tampered with**. You can use `assertNotEquals`. This is sort of a silly test.
- That the method handles a null input properly.

这道题目我开始理解错了，我以为在调用reverse方法后，不仅要返回一个reversed IntList，而且A所指向的IntList也要是reversed。

但是**实际上要求仅仅是调用reverse方法返回一个reversed List，而A指向的IntList并没有要求是reversed**。

实际上只需要改变两个节点之间的链接指向即可。

迭代解法：

将链表从头到尾遍历一次，用x指向每一个节点，同时还有两个辅助的临时对象，temp1指向x的前一个节点，temp2指向x的后一个节点，temp2用来保存x的下一个节点的引用，temp1保存x的前一个节点的引用。对每一次迭代，先用temp2保存x.rest，再将x.rest指向temp1，然后先将temp1向后移动一个节点（即x所在位置），再将x也移动到下一个节点。移动到最后一个位置，返回最后一个节点的引用。

```java
public static IntList reverse(IntList A){
        IntList temp1=null;
        IntList x=A;
        while (x!=null){
            IntList temp2=x.rest;
            x.rest=temp1;
            temp1=x;
            x=temp2;
        }
        return temp1;
    }
```

递归解法：

此题使用递归其实更简单，我们只需要一直找到链表的最后一个节点，然后原路返回，同时改变每一个节点的指向即可

```java
public static IntList recursiveReverse(IntList A){
        if (A==null||A.rest==null)
            return A;
        IntList reversed=recursiveReverse(A.rest);
        A.rest.rest=A;
        A.rest=null;
        return reversed;
    }
```

Autograder结果：

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210313174627800.png" alt="image-20210313174627800" style="zoom:50%;" />