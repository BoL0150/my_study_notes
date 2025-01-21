# cs61b实验记录（二）project1

# project1A

## LinkedListDeque

### circular sentinel

错误写法：

```java
public LinkedListDeque(){
        sentinel=new Node(sentinel,null,sentinel.next);
        size=0;
    }
```

在实例化sentinel时，sentinel本身还只是null，用sentinel和sent.next去实例化sentinel当然是行不通的。

正确写法：

```java
public LinkedListDeque(){
        sentinel=new Node(null,null,null);
        sentinel.next=sentinel;
        sentinel.prev=sentinel;
        size=0;
    }
```

在对sentinel实例化完成后再对它的next和prev字段赋值。

完整实现：

```java
public class LinkedListDeque<T> {

    private class Node{
        Node prev;
        T item;
        Node next;
        Node(Node prev,T item,Node next){
            this.item=item;
            this.prev=prev;
            this.next=next;
        }
    }
    private int size;
    private Node sentinel;
    public LinkedListDeque(){
        sentinel=new Node(null,null,null);
        sentinel.next=sentinel;
        sentinel.prev=sentinel;
        size=0;
    }
    public void addFirst(T item) {
        sentinel.next=new Node(sentinel,item,sentinel.next);
        sentinel.next.next.prev=sentinel.next;
        size++;
    }
    public void addLast(T item) {
        sentinel.prev=new Node(sentinel.prev,item,sentinel);
        sentinel.prev.prev.next=sentinel.prev;
        size++;
    }

    public boolean isEmpty() {
        return size==0;
    }

    public int size() {
        return size;
    }

    public void printDeque() {
        Node pos=sentinel.next;
        for (int i=0;i<size-1;i++){
            System.out.print(pos.item+" ");
            pos=pos.next;
        }
        System.out.print(pos.item);
    }

    public T removeFirst() {
        if (sentinel.next==sentinel)
            return null;
        Node temp=sentinel.next;
        sentinel.next.next.prev=sentinel;
        sentinel.next=sentinel.next.next;
        size--;
        return temp.item;
    }

    public T removeLast() {
        if (sentinel.prev==sentinel)
            return null;
        Node temp=sentinel.prev;
        sentinel.prev.prev.next=sentinel;
        sentinel.prev=sentinel.prev.prev;
        size--;
        return temp.item;
    }

    public T get(int index) {
        Node pos=sentinel.next;
        if (index>=size)
            return null;
        for (int i=0;i<index;i++){
            pos=pos.next;
        }
        return pos.item;
    }
    public T getRecursive(int index){
        if (index>=size)
            return null;
        return getRecursive(0,index,sentinel.next);
    }
    private T getRecursive(int pos,int index,Node x){
        if (pos==index)
            return x.item;
        return getRecursive(pos+1,index,x.next);
    }
}

```



## ArrayDeque

应该一步一步减小数据结构实现的复杂度，对它的实现进行分层。

**首先不应该考虑resize，先把最基本的数据结构实现出来**

```java
public class ArrayDeque<T> {

        private int nextFirst;
        private int nextLast;
        private T[]items;
        private int size;
        public ArrayDeque(){
            items=(T[])new Object[8];
            nextFirst=0;
            nextLast=1;
            size=0;
        }
        public void addFirst(T item) {
            items[nextFirst]=item;
            size++;
            nextFirst=nextFirst==0?items.length-1:nextFirst-1;
        }

        public void addLast(T item) {
            items[nextLast]=item;
            size++;
            nextLast=nextLast==items.length-1?0:nextLast+1;
        }

        public boolean isEmpty() {
            return size==0;
        }

        public int size() {
            return size;
        }

        public void printDeque() {
            for (int i=nextFirst+1;i!=nextLast;i=(i+1)%items.length)
                System.out.print(items[i]);
        }

        public T removeFirst() {
            if (nextFirst==0)return null;
            nextFirst++;
            T temp=items[nextFirst];
            items[nextFirst]=null;
            size--;
            return temp;
        }

        public T removeLast() {
            if (nextLast==1)return null;
            nextLast--;
            T temp=items[nextLast];
            items[nextLast]=null;
            size--;
            return temp;
        }

        public T get(int index) {
            if (index>=size)
                return null;
            return items[(nextFirst+1+index)%items.length];
        }
    }
```

实际上我们可以发现，由于nextFirst是0，nextLast是1，当遇到数组的拐点时要进行特殊判断，对我们的方法的实现造成了麻烦。我们可以进一步地思考，为什么不能将nextFirst设为数组的最后一个元素，将nextLast设为0呢？我们惊奇地发现，此时减少了很多的特殊判断。

```java
public class ArrayDeque<T> {

        private int nextFirst;
        private int nextLast;
        private T[]items;
        private int size;
        public ArrayDeque(){
            items=(T[])new Object[8];
            nextFirst=items.length-1;
            nextLast=0;
            size=0;
        }
        public void addFirst(T item) {
            items[nextFirst]=item;
            size++;
            nextFirst--;
        }

        public void addLast(T item) {
            items[nextLast]=item;
            size++;
            nextLast++;
        }

        public boolean isEmpty() {
            return size==0;
        }

        public int size() {
            return size;
        }

        public void printDeque() {
            for (int i=nextFirst+1;i!=nextLast;i=(i+1)%items.length)
                System.out.print(items[i]);
        }

        public T removeFirst() {
            if (nextFirst==items.length-1)return null;
            nextFirst++;
            T temp=items[nextFirst];
            items[nextFirst]=null;
            size--;
            return temp;
        }

        public T removeLast() {
            if (nextLast==0)return null;
            nextLast--;
            T temp=items[nextLast];
            items[nextLast]=null;
            size--;
            return temp;
        }

        public T get(int index) {
            if (index>=size)
                return null;
            return items[(nextFirst+1+index)%items.length];
        }
    }
```

由于nextLast和nextFirst不可能越过拐点，我们在add方法中不再需要特殊判断

同时，resize的实现也变得简单了，只需将原数组的首尾分开复制到新的数组即可。

完整实现：

```java
public class ArrayDeque<T> {

        private int nextFirst;
        private int nextLast;
        private T[]items;
        private int size;
        public ArrayDeque(){
            items=(T[])new Object[8];
            nextFirst=items.length-1;
            nextLast=0;
            size=0;
        }
        private void resize(int capacity){
            T[]a=(T[])new Object[capacity];
            System.arraycopy(items,0,a,0,nextLast);
            int length=items.length-1-nextFirst;
            System.arraycopy(items,nextFirst+1,a,capacity-length,length);
            nextFirst=capacity-length-1;
            items=a;
        }
        public void addFirst(T item) {
            if (nextFirst-1==nextLast)
                resize(items.length*2);
            items[nextFirst]=item;
            size++;
            nextFirst--;
        }

        public void addLast(T item) {
            if (nextFirst-1==nextLast)
                resize(items.length*2);
            items[nextLast]=item;
            size++;
            nextLast++;
        }

        public boolean isEmpty() {
            return size==0;
        }

        public int size() {
            return size;
        }

        public void printDeque() {
            for (int i=nextFirst+1;i!=nextLast-1;i=(i+1)%items.length)
                System.out.print(items[i]+" ");
            System.out.print(items[nextLast-1]);
        }

        public T removeFirst() {
            if (nextFirst==items.length-1)return null;
            nextFirst++;
            T temp=items[nextFirst];
            items[nextFirst]=null;
            size--;
            if (items.length>=16&&size<items.length/4)
                resize(items.length/2);
            return temp;
        }

        public T removeLast() {
            if (nextLast==0)return null;
            nextLast--;
            T temp=items[nextLast];
            items[nextLast]=null;
            size--;
            if (items.length>=16&&size<items.length/4)
                resize(items.length/2);
            return temp;
        }

        public T get(int index) {
            if (index>=size)
                return null;
            return items[(nextFirst+1+index)%items.length];
        }
    }

```

仅仅是用两个指针分别指向数组的首尾，就是一种完全的不同的数据结构，就可以借用数组实现双向队列，将头插入从原来的线性时间复杂度，到现在的常数时间复杂度。

---------------

2021年3月22日更新：

**以上ArrayDeque全部都是错的！**

之前的这个做法最后在autoGrader上只有40/50的成绩，其中ArrayDeque只有一半的分数。我以为只是有一些细微的特殊情况没有考虑到，所以就没有去仔细检查与修正。

在评论区@小小张童鞋 的提醒下，我回过头来重新审视这段代码，发现之前的做法还是太naive了，ArrayDeque中大部分都是错误的！修正如下：

在之前我天真的认为只要将nextLast和nextFirst分别置于数组的开始和结尾，它们就不会越过拐点了。显然大错特错！因为即使add操作不会越过拐点，但是执行remove操作时，nextLast和nextFirst依然有可能越过拐点！

所以，既然无论如何nextFirst和nextLast都有可能越过拐点，那么这两个指针指向什么地方都无所谓了。

完整代码：

```java
public class ArrayDeque<T> {

    private int nextFirst;
    private int nextLast;
    private int capacity;
    private T[]items;
    private int size;
    public ArrayDeque(){
        items=(T[])new Object[8];
        this.capacity=items.length;
        nextFirst=capacity-1;
        nextLast=0;
        size=0;
    }
    private void resize(int capacity){
        T[]a=(T[])new Object[capacity];
        //由于nextFirst和nextLast的位置不确定，只能一个一个地复制到新的数组中
        //从nextFirst右边的第一个点开始复制
        //到nextLast左边的第一个点复制结束
        for (int i=1;i<=size;i++)
            a[i]=items[(++nextFirst)%this.capacity];
        this.capacity=capacity;
        //这两个指针指向什么地方已经不重要了
        nextFirst=0;
        nextLast=size+1;
        items=a;
    }
    public void addFirst(T item) {
        //直接当size等于capacity时调整大小，而不是看两个指针的相对位置
        if (size==capacity)
            resize(capacity*2);
        items[nextFirst]=item;
        size++;
        //nextFirst有可能越界
        nextFirst=nextFirst==0?capacity-1:nextFirst-1;
    }

    public void addLast(T item) {
        if (size==capacity)
            resize(capacity*2);
        items[nextLast]=item;
        size++;
        //nextLast有可能越界
        nextLast=(nextLast+1)%capacity;
    }

    public boolean isEmpty() {
        return size==0;
    }

    public int size() {
        return size;
    }

    public void printDeque() {
        //nextFirst有可能指向最后一个位置
        for (int i=(nextFirst+1)%capacity;i!=nextLast-1;i=(i+1)%capacity)
            System.out.print(items[i]+" ");
        System.out.print(items[nextLast-1]);
    }

    public T removeFirst() {
        //当数组的内容为空的时候，才无法进行remove操作，而不是取决于nextFirst的位置。
        if (size==0)return null;
        nextFirst=(nextFirst+1)%capacity;
        T temp=items[nextFirst];
        items[nextFirst]=null;
        size--;
        if (capacity>=16&&size<capacity/4)
            resize(capacity/2);
        return temp;
    }

    public T removeLast() {
        if (size==0)return null;
        nextLast=nextLast==0?capacity-1:nextLast-1;
        T temp=items[nextLast];
        items[nextLast]=null;
        size--;
        if (capacity>=16&&size<capacity/4)
            resize(capacity/2);
        return temp;
    }

    public T get(int index) {
        if (index>=size)
            return null;
        return items[(nextFirst+1+index)%capacity];
    }
}
```

autograder分数：

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210322210555296.png" alt="image-20210322210555296" style="zoom:50%;" />

# project1B

nothing special，不在此赘述

解法详见我的GitHub

https://github.com/BoL0150/Berkeley-cs61b/tree/main/proj1b

























































































































