# cs61b实验记录（五）HW3、Lab10

## HW3

### equals

```java
@Override
    public boolean equals(Object o) {
        if (o==this)return true;
        if (o==null)return false;
        if (o instanceof SimpleOomage){
            SimpleOomage anothero=(SimpleOomage)o;
            return this.red==anothero.red&&this.blue==anothero.blue&&this.green==anothero.green;
        }return false;
    }
```

### testHashCodePerfect

之前的hashcode方法：`return red + green + blue;`

这样会导致很多不必要的碰撞，two `SimpleOomage`s may only have the same hashCode only if they have the **exact same** red, green, and blue values

**fill in the `testHashCodePerfect` of `TestSimpleOomage` with code that tests to see if the `hashCode` function is perfect.**

直接判断两个颜色不同的对象的hashCode是否相同

```java
@Test
    public void testHashCodePerfect() {
        SimpleOomage ooA = new SimpleOomage(20, 10, 55);
        SimpleOomage ooA2 = new SimpleOomage(10, 20, 55);
        SimpleOomage ooA3 = new SimpleOomage(20, 10, 55);
        assertNotEquals(ooA.hashCode(),ooA2.hashCode());
        assertEquals(ooA.hashCode(),ooA3.hashCode());
    }
```

### A Perfect hashCode

```java
@Override
    public int hashCode() {
        if (!USE_PERFECT_HASH) {
            return red + green + blue;
        } else {
            return 61*(61*(red*61+green)+blue);
        }
    }
```

### haveNiceHashCodeSpread

判断hashCode是否均匀散列，减少不必要的碰撞

```java
public static boolean haveNiceHashCodeSpread(List<Oomage> oomages, int M) {

        int[] buckets=new int [M];
        for (Oomage o:oomages){
            int bucketNum = (o.hashCode() & 0x7FFFFFFF) % M;
            buckets[bucketNum]++;
            if (buckets[bucketNum]>=oomages.size()/2.5){
                return false;
            }
        }
        for (int i=0;i<buckets.length;i++){
            if (buckets[i]<=oomages.size()/50.0)
                return false;
        }
        return true;
    }
```

### Evaluating the perfect hashCode Visually

因为red，blue，green都是5的倍数，通过之前的hashCode方法，不管怎么算，得到的hashCode永远是5的倍数。而由于bucketNum的计算方法是将hashCode对bucket的数量求余，所以如果bucket的数量不是5的倍数，那么就可以均匀散列，但是如果bucket的数量是5的倍数，比如10，那么结果就会集中在5和0的位置。

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210416100635682.png" alt="image-20210416100635682" style="zoom:50%;" />

如果将bucket的数量改为9，就可以均匀散列

<img src="https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210416100810361.png" alt="image-20210416100810361" style="zoom:50%;" />

所以我们在计算hashCode前，需要将red、blue、green的值除以5，再计算。

```java
@Override
    public int hashCode() {
        if (!USE_PERFECT_HASH) {
            return red + green + blue;
        } else {
            return 61*(61*(61*red/5+green/5)+blue/5);
        }
    }
```



### Evaluating the ComplexOomage hashCode

对于ComplexOomage，它有一个Integer的List，它的hashCode计算方法是将这个List中的每个数乘以256后和下一个数相加，再乘以256 。

```java
 @Override
    public int hashCode() {
        int total = 0;
        for (int x : params) {
            total = total * 256;
            total = total + x;
        }
        return total;
    }
```

256是1后面跟8个0，而int的表示范围是1后面跟32个0，所以最多乘四次就会溢出。所以hashCode的结果取决于ComplexOomage的List中的最后四个数。

我们只需要将每一个ComplexOomage对象的List中的最后四个数设置为相同的就可以实现testWithDeadlyParams

```java
@Test
    public void testWithDeadlyParams() {
        List<Oomage> deadlyList = new ArrayList<>();
        int deadlyListSize=10000;
        for (int i=0;i<deadlyListSize;i++){
            int N = StdRandom.uniform(1, 10);
            ArrayList<Integer> params = new ArrayList<>(N);
            for (int j = 0; j < N - 4; j += 1) {
                params.add(StdRandom.uniform(0, 255));
            }
            for (int j = N - 4; j < N; j++) {
                params.add(1);
            }
            deadlyList.add(new ComplexOomage(params));
        }
        assertTrue(OomageTestUtility.haveNiceHashCodeSpread(deadlyList, 10));
    }
```

## Lab 10

```java
private static int leftIndex(int i) {
        return i * 2;
    }

    /**
     * Returns the index of the node to the right of the node at i.
     */
    private static int rightIndex(int i) {
        return i * 2 + 1;
    }

    /**
     * Returns the index of the node that is the parent of the node at i.
     */
    private static int parentIndex(int i) {
        return i / 2;
    }
```

```java
/**
     * Bubbles up the node currently at the given index.
     */
    private void swim(int index) {
        // Throws an exception if index is invalid. DON'T CHANGE THIS LINE.
        validateSinkSwimArg(index);
        if (index > 1 && contents[index].myPriority < contents[index / 2].myPriority) {
            swap(index, index / 2);
            swim(index / 2);
        }
    }

    /**
     * Bubbles down the node currently at the given index.
     */
    private void sink(int index) {
        // Throws an exception if index is invalid. DON'T CHANGE THIS LINE.
        validateSinkSwimArg(index);
        if (index*2<=size){
            int j=index*2;
            if (j+1<=size&&contents[j+1].myPriority<contents[j].myPriority) j++;
            if (contents[j].myPriority>=contents[index].myPriority) return;
            sink(j);
        }
    }
```

```java
@Override
    public void insert(T item, double priority) {
        /* If the array is totally full, resize. */
        if (size + 1 == contents.length) {
            resize(contents.length * 2);
        }
        contents[size+1]=new Node(item, priority);
        size++;
        if (size>=2)
            swim(size);
    }
public T removeMin() {
        T min=contents[1].myItem;
        swap(1, size);
        contents[size]=null;
        size--;
        sink(1);
        return min;
    }
```

```java
@Override
    public void changePriority(T item, double priority) {
        for (int i=1;i<=size;i++){
            if (contents[i].myItem.equals(item)){
                double temp=contents[i].myPriority;
                contents[i].myPriority=priority;
                if (temp>priority) swim(i);
                else sink(i);
                return;
            }
        }
    }
```

