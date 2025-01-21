# CS61B学习记录（一）

[TOC]

# java基础语法部分：

## Lecture4

**naked linked lists like this are hard to use**，只有一个节点，使用者必须要对引用非常熟悉，并且要递归地思考

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210306193939617.png" alt="image-20210306193939617" style="zoom: 67%;" />

Improvement：

使用SLList将节点包装起来，使用者可以更形象的使用链表

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210306194401256.png" alt="image-20210306194401256" style="zoom:67%;" />



本质上实现链表只需要一个节点对象就行了，所有操作直接在这个节点对象上进行。但是为了用户更好的操作，应该使用一个包装类将节点包装在其中，这个包装类提供了对链表进行操作的方法，所有对链表的操作都是通过这个包装类进行。

注意！人们往往会有误解，把包装类对象想象成一条完整的链表，认为这个对象中包含了链表的所有节点。**实际上包装类对象只是一个对象而已，只包含了链表的第一个节点或者最后一个节点，以及一些操作链表的方法。包装类对象本质上就是一个操作链表的接口，而不是链表的主体**。

![image-20210306200308027](https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210306200308027.png)

size( ):

IntNode的size( )方法：

```java
public class IntNode{
        int item;
        IntNode next;
        IntNode(int x,IntNode next){
            this.item=x;
            this.next=next;
        }
        int size(){
            if (next==null)return 1;
            return next.size()+1;
        }
    }
```

但是，对于SLList却无法这样写，我们无法写出类似下面的语句：

```
int size(){
	.....
	return next.size()+1;
}
```

因为，SLLit本身不是一个递归结构，我们需要这样

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210306205541679.png" alt="image-20210306205541679" style="zoom:50%;" />

我们还需要提升size( )的速度

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210306210135004.png" alt="image-20210306210135004" style="zoom:50%;" />

对于一条链表，如果我需要随着它的扩大，实时得到它的size，那么这个速度将会很慢，因为我们每一次调用size()都需要从后往前遍历，这中间有很多重复的操作。**为了提升速度，我们应该实时保存每一个节点（链表）的大小**。我们为SLList创建一个int字段，每次链表大小改变时，我们都改变这个字段的值。

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210306210617388.png" alt="image-20210306210617388" style="zoom: 67%;" />

在这里也能体现出SLList比nakedLinkedList的好处：

使用包装类对象，**我们可以储存类似size的变量**，每次链表大小发生变化时，我们只需要改变SLList对象的size字段即可。但是对于nakedLinkedList，我们需要为每一个节点对象都设置一个size字段，发生改变时，我们需要改变所有被影响的节点对象的字段，非常低效。

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210306212144499.png" alt="image-20210306212144499" style="zoom: 67%;" />

我们可以构造一个为空的SLLIst：

```java
public SLList(){
        first=null;
        size=0;
    }
```

但是这也会带来问题：

```java
public void addLast(int x){
    IntNode p=first;
    while (p.next!=null){//此时p等于null，出现nullPointerException
        p=p.next;
    }
    p.next=new IntNode(x,null);
    size++;
}
```

我们可以对特殊情况进行修复：

```java
if (p==null){
    first=new IntNode(x,null);
    return;
}
```

然而这样代码变得丑陋

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210306215537749.png" alt="image-20210306215537749" style="zoom:50%;" />

最重要的是，不要经常对特殊情况进行修复，因为对于更大的数据结构会拥有更多的特殊案例。只有简单的代码才是好的代码，**You want to restrict the amount of complexity in your life!我们应该找到一个适用于所有情况的相同的方法**。

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210306215651065.png" alt="image-20210306215651065" style="zoom:50%;" />

所以，为了让所有情况都相同（即链表不管有没有元素都是一样的），即使链表为空，SLList中也要有一个节点存在，这样我们就没有必要检查该节点是否为null（因为它永远都不为null）

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210306221201465.png" alt="image-20210306221201465" style="zoom:50%;" />

```java
public class SLList {
    private static class IntNode{
        int item;
        IntNode next;
        IntNode(int x,IntNode next){
            this.item=x;
            this.next=next;
        }
    }
    private int size;
    private IntNode sentinel;
    public SLList(int x){
        sentinel=new IntNode(0,null);
        sentinel.next=new IntNode(x,null);
        size=1;
    }
    public SLList(){
        sentinel=new IntNode(0,null);
        size=0;
    }
    public void addFirst(int x){
        sentinel.next=new IntNode(x,sentinel.next);
        size++;
    }
    public int getFirst(){
        return sentinel.next.item;
    }
    public void addLast(int x){
        IntNode p=sentinel;
        while (p.next!=null){
            p=p.next;
        }
        p.next=new IntNode(x,null);
        size++;
    }
    //cached edition
    public int size(){
        return size;
    }
/**recursive edition
    public int size(){
        return size(first);
    }

    private int size(IntNode x){
        if (x.next==null)return 1;
        return 1+size(x.next);
    }
 */

    /*iterative edition
    public int size(){
        IntNode p=first;
        int cnt=0;
        while (p!=null){
            p=p.next;
            cnt++;
        }
        return cnt;
    }
     */
}

```

## Lecture5

对于单向链表SLList来说，对尾部进行操作非常地慢

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210309222737858.png" alt="image-20210309222737858" style="zoom:67%;" />

我们需要从前往后遍历，一直到链表的尾部才能添加节点。

解决方法：将尾结点缓存起来

![image-20210309223035856](https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210309223035856.png)

然而对尾部进行删除操作却依旧缓慢。

删除尾结点需要改变它前面一个节点的指向，我们要获取前面的节点依然需要从前往后遍历一次。

所以，低效的根本原因是不能从尾结点向前访问。

我们可以使用双向链表

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210309225514358.png" alt="image-20210309225514358" style="zoom: 50%;" />

但是，双向链表也会有特例。

我们可以再添加一个尾部的Sentinel来避免special case

![image-20210309225626541](https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210309225626541.png)

或者使用更好的方式：Circular Sentinel

![image-20210309225805464](https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210309225805464.png)

此时也不再需要一个变量last来记录尾结点的位置

### 2D Arrays

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210309233454521.png" alt="image-20210309233454521" />

<img src="C:%5CUsers%5Cleeb%5CDesktop%5CGitHubManagement%5Cmynotes%5CCS61B%E5%AD%A6%E4%B9%A0%E7%AC%94%E8%AE%B0.assets%5Cimage-20210309233454521.png" alt="image-20210309233454521" style="zoom:67%;" />

## Lecture6

对于链表来说，进行随机访问非常慢，所以我们使用数组（ArrayList）来进行random access。

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210311130623230.png" alt="image-20210311130623230" style="zoom: 50%;" />

**Invariants is very important**

![image-20210311132137701](https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210311132137701.png)

![image-20210311132710715](https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210311132710715.png)

**resizing array**

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210311152027999.png" alt="image-20210311152027999" style="zoom: 67%;" />

然而一个一个地增加数组的容量效率极低，所以我们需要成倍地扩容

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210311152223380.png" alt="image-20210311152223380" style="zoom:67%;" />

除了在数组容量不够时扩容，我们还需要在数组空间利用率太低的时候缩小数组的容量

![image-20210311152410541](https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210311152410541.png)

当数组的利用率小于25%时，我们需要将数组的大小缩小为二分之一。

我们还可以使用泛型AList，使用泛型时要注意将没有用的空间置为null

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210311155443985.png" alt="image-20210311155443985" style="zoom:67%;" />

java只会销毁失去引用的对象，在AList中，我们调用了removeLast()后只是减小了size，原来的最后一个元素永远不会再被使用了，它应该被销毁。但是它的引用却仍然在数组中（当该元素是引用类型是），此时java并不会回收这个对象，所以为了节约空间，我们需要手动将数组的这个空间置为null，这样就没有引用指向该对象，它就会被java回收。

```java
public class AList<Item> {
    private Item[]items;
    private int size;
    public AList(){
         items=(Item[]) new Object[100];
         size=0;
    }
    public void resize(int capacity){
            Item[]a=(Item[]) new Object[capacity];
            System.arraycopy(items,0,a,0,size);
            items=a;
    }
    public Item removeLast(){
        return items[--size];
    }
    public void addLast(Item i){
        if (size==items.length){
            resize(2*size);
        }
        items[size]=i;
        size++;
    }
    public Item getLast(){
        return items[size-1];

    }
   public Item get(int i){
        return items[i];
    }
    public int size(){
        return size;
    }
}

```



## Lecture7

### Test

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210311173810635.png" alt="image-20210311173810635" style="zoom:67%;" />

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210311173920197.png" alt="image-20210311173920197" style="zoom:67%;" />

### Selection Sort

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210311184952618.png" alt="image-20210311184952618" style="zoom: 67%;" />

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210311203558751.png" alt="image-20210311203558751" style="zoom:67%;" />

## Lecture8

### Interface

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210311214814304.png" alt="image-20210311214814304" style="zoom:67%;" />

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210311225753784.png" alt="image-20210311225753784" style="zoom:67%;" />



<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210311225858654.png" alt="image-20210311225858654" style="zoom:67%;" />

#### Interface vs. Abstract class

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210318210355893.png" alt="image-20210318210355893" style="zoom: 50%;" />

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210318210450644.png" alt="image-20210318210450644" style="zoom:50%;" />

Interfaces: .

- Primarily for interface inheritance. Limited implementation inheritance.
- Classes can implement multiple interfaces.

Abstract classes:

- **Can do anything an interface can do, and more.**
- Subclasses only extend one abstract class.

In my opinion, you should generally prefer interfaces whenever possible.Why? More powerful programming language constructs introduce complexity.

为什么有了Interface还要有一个功能相近的Abstract Class？

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210318210754909.png" alt="image-20210318210754909" style="zoom:50%;" />

Abstract class用来部分实现Interface中的方法（在2014年之前，那时Interface还没有default method）

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210318211155929.png" alt="image-20210318211155929" style="zoom:50%;" />

### overriding VS overloading

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210311220030292.png" alt="image-20210311220030292" style="zoom:67%;" />

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210311220416318.png" alt="image-20210311220416318" style="zoom:67%;" />

Method Overriding
If a subclass has a method with the exact same signature as in the superclass, we say the subclass overrides the method.

- Even if you don't write @Override, subclass still overrides the method.
- @Override is just an **optional reminder** that you' re overriding.

Why use @Override?

- Main reason: **Protects against typos**. 
  If you say @Override, but the method isn't actually overriding anything, you'll get a compile
  error
- Reminds programmer that method definition came from somewhere higher up in the inheritance hierarchy.



<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210311221649297.png" alt="image-20210311221649297" style="zoom:67%;" />

**if X is a superclass of Y,then memory boxes for x may contain Y**,that's why subclass can be copied to superclass.

### implementation inheritance（default method）

Interface inheritance:

- Subclass inherits signatures, but NOT **implementation**.
  For better or worse, **Java also allows implementation inheritance**.
- **Subclasses can inherit signatures AND implementation**.
  Use the **default** keyword to specify a method that subclasses should inherit from an interface.


we can define a function to print the content of the list in the Interface List61B,Unlike Interface inheritance，which can only allow signature inheritance，**its signatures and implementations will be both inherited by AList and SLList**。

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210311230645290.png" alt="image-20210311230645290" style="zoom:67%;" />

but then there is a question，this method is not efficient for both subclass

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210311230940959.png" alt="image-20210311230940959" style="zoom:67%;" />

and if we dont like the default method， we can override default method

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210311231617873.png" alt="image-20210311231617873" style="zoom:67%;" />

### Dynamic Method Selection

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210311233824027.png" alt="image-20210311233824027" style="zoom:67%;" />

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210311234633061.png" alt="image-20210311234633061" style="zoom:67%;" />

rule:调用方法时，首先进行编译，如果这个方法只有动态类型有，而静态类型没有，则编译不通过。只有两个类型都有相同的方法（即函数签名相同，如果只是函数名相同也不行，这只是重载），才能通过编译。在运行时，从最准确的类型（即动态类型）开始寻找该方法，如果动态类型重写了该方法，就调用动态类型的版本。如果动态类型没有重写方法，就调用静态类型的版本。

## Lecture9

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210312095343269.png" alt="image-20210312095343269" style="zoom:67%;" />

you can use parameter to specify a constructor，if you dont explicitly specify which of the superclass’s constructor to call，it will call the default one with no parameters。

### encapsulation

when building large programs，our enemy is **complexity**。That's the thing that makes a good programmer different from the bad programmer，**the ability to handle complexity**。  

managing complexity is important to large project（e.g project2)

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210312103609149.png" alt="image-20210312103609149" style="zoom: 50%;" />

### Casting

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210312110738002.png" alt="image-20210312110738002" style="zoom: 50%;" />

将父类对象赋值给子类对象时编译无法通过，但是我们可以对父类对象进行强制类型转换，此时父类对象的编译类型就是子类对象的类型，但是注意！父类对象的实际类型（即运行类型）并没有改变，如果该对象的实际类型就是父类，那么运行时会出现ClassCastException。如果父类的实际类型是子类的类型（以及子类的子类），那么可以通过编译。强制类型转换常用于使用多态时，要调用子类对象特有的方法，此时通过父类对象无法调用，我们就可以进行强制类型转换。

为了避免强制类型转换时出现ClassCastException，我们可以使用insistanceof来判断对象的实际类型是什么。

## Lecture10

### Comparable

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210312153822029.png" alt="image-20210312153822029" style="zoom: 50%;" />

### Compartor

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210312165343170.png" alt="image-20210312165343170" style="zoom: 50%;" />

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210312165653025.png" alt="image-20210312165653025" style="zoom:50%;" />

#### Summary

Interfaces provide us with the ability to make **callbacks**:

- **Sometimes a function needs the help of another function that might not have been written yet.**

  - Example: max needs compareTo
  - The helping function is sometimes called a“callback".

- Some languages handle this using explicit function passing.

  ```python
  def print_larger(x,y,compare,stringify):
      if compare(x,y):
          return stringify(x)
      return stringify(y)
  ```

  we can pass a function as parameter.

- In Java, we can't pass function as parameter.we do this by **wrapping up the needed function in an interface** 

  - example:**Arrays. sort** ,the function needs to know how to sort,what the rule is. 

    - In python,we can just use a function type as parameter,when users use this function,they can pass a function(specific rule of sort)to specify how to sort.

    - In java,**we can pass an object and invoke this object's method in the function to specify the rules**.To make this,we need to set the parameter of function as a class.But we don't know what object will be passed in,So we need a general interface with the function we need to fit all the object.**The class of object passed in must inplement the interface and override the function.**In that way,we can use the function in the Array.sort

      <img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210312183917712.png" alt="image-20210312183917712" style="zoom: 50%;" />

      use **interface comparator** as parameter,pass in an object which has overrided the comparator,and invoke the compare method of the object in the function.

  Arrays. sort“calls back" whenever it needs a comparison.

## Lecture11

### exception

```java
public class ArraySet<T> {
    public int size;
    public T[]items;
    public ArraySet(){
        items=(T[])new Object[100];
        size=0;
    }
    public boolean contains(T x){
        for (int i=0;i<size;i++){
            if (x.equals(items[i]))
                return true;
        }
        return false;
    }
    public void add(T x){
        if (contains(x))
            return;
        items[size++]=x;
    }
    public int size(){
        return size;
    }
}
```

当我们add(null)时会出现异常，原因是在add()方法中调用了contains，x.equals(items[i])语句对null调用方法就会出现nullPointerException。

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210312215026595.png" alt="image-20210312215026595" style="zoom: 50%;" />

### Iterator

如何实现for each？拆分为iterator

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210312215909016.png" alt="image-20210312215909016" style="zoom:50%;" />

如何实现Iterator？在数据结构内部实现Iterator接口，实现hasNext和next方法，新增一个Iterator()方法，返回Iterator对象（即我们实现的Iterator类）。

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210312220740501.png" alt="image-20210312220740501" style="zoom:50%;" />

仅仅有Iterator()方法仍然不够，java并不知道该数据结构有Iterator()方法，我们需要将该类实现Iterable接口，告诉java，这个数据结构是可迭代的。

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210312222007923.png" alt="image-20210312222007923" style="zoom:50%;" />

### toString

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210312230354004.png" alt="image-20210312230354004" style="zoom:50%;" />

print(Object x)实际上执行的是print(x.toString)，打印对象的地址。

```java
@Override
    public String toString() {
        String returnString="{";
        for (T item:this){
            returnString+=item.toString();
            returnString+=",";
        }
        returnString+="}";
        return returnString;
    }
```

改变String的执行速度非常慢，由于String对象inmmutable的，每修改一次String对象都要重新开辟空间创建新的对象，然后再进行修改。改用StringBuilding或StringBuffer

```java
@Override
    public String toString() {
        StringBuilder returnString= new StringBuilder("{");
        for (T item:this){
            returnString.append(item.toString());
            returnString.append(",");
        }
        returnString.append("}");
        return returnString.toString();
    }
```

### equals

==比较的是两者的比特位，对于引用来说，==比较的就是两者是否指向同一个对象，而不是内容是否相同

**默认的equals就是==，所以在我们自定义的类型中需要重写equals来比较对象真正的内容。**

```java
@Override
    public boolean equals(Object obj) {
        ArraySet<T>o=(ArraySet<T>)obj;
        if (o.size!=this.size())
            return false;
        for (T item:this){
            if (!o.contains(item))
                return false;
        }
        return true;
    }
```

问题是：当obj是其他类型时或是null时会失败

改进：

```java
@Override
    public boolean equals(Object obj) {
        //当两个比较对象就是同一个对象时，直接返回true
        if (obj==this)return true;
        if (obj==null)return false;
        //当两个比较对象的类型不同时，直接返回false
        if (obj.getClass()!=this.getClass())return false;
        ArraySet<T>o=(ArraySet<T>)obj;
        if (o.size!=this.size())
            return false;
        for (T item:this){
            if (!o.contains(item))
                return false;
        }
        return true;
    }
```

### bonus

ArraySet.of

```java
public static <Glerp> ArraySet<Glerp> of(Glerp... stuff){
        ArraySet<Glerp> returnSet=new ArraySet<>();
        for (Glerp x:stuff){
            returnSet.add(x);
        }
        return returnSet;
    }
```

```java
public static void main(String[] args) {
        ArraySet<String>arraySet1=ArraySet.of("hi","i","wanna","fuck","you");
    }
```

