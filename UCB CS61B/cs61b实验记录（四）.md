# cs61b实验记录（四）HW2，lab9

## HW 2

`Percolation`：

先不解决backwash问题

```java
import edu.princeton.cs.algs4.WeightedQuickUnionUF;

public class Percolation {
    private boolean[][]grid;
    private int N;
    private WeightedQuickUnionUF UF;
    private int numberOfOpenSites;
    public Percolation(int N){
        if (N<=0)
            throw new IllegalArgumentException("N must > 0");
        this.N=N;
        this.numberOfOpenSites=0;
        UF=new WeightedQuickUnionUF(N*N+2);
        grid=new boolean[N][N];
    }
    public void open(int row,int col){
        int next[][]=new int[][]{
                {0,1},
                {1,0},
                {0,-1},
                {-1,0}
        };
        if (row<0||row>=N||col<0||col>=N)
            throw new IllegalArgumentException("row and col must between 0 and N-1");
        if (isOpen(row,col))return;
        grid[row][col]=true;
        numberOfOpenSites++;
        //连接周围的open的点
        for (int i=0;i<=3;i++){
            int mx=row+next[i][0];
            int my=col+next[i][1];
            if (my<0||my>=N)
                continue;
            if (mx==-1){
                UF.union(xyTo1D(row, col),N*N);
                continue;
            }
            else if (mx==N){
                UF.union(xyTo1D(row, col),N*N+1);
                continue;
            }
            if (isOpen(mx,my)&&UF.connected(xyTo1D(row,col),xyTo1D(mx,my))==false)
                UF.union(xyTo1D(row, col),xyTo1D(mx,my));
        }
    }
    private int xyTo1D(int mx,int my){
        return mx*N+my;
    }
    // is the site (row, col) open?
    public boolean isOpen(int row, int col){
        if (row<0||row>=N||col<0||col>=N)
            throw new IllegalArgumentException("row and col must between 0 and N-1");
        return grid[row][col];
    }
    // is the site (row, col) full?
    public boolean isFull(int row, int col){
        if (row<0||row>=N||col<0||col>=N)
            throw new IllegalArgumentException("row and col must between 0 and N-1");
        return UF.connected(xyTo1D(row, col), N * N);
    }
    // number of open sites
    public int numberOfOpenSites(){
        return numberOfOpenSites;
    }
    // does the system percolate?
    public boolean percolates(){
        return UF.connected(N*N,N*N+1);
    }
}

```

引起backwash的根本原因是：在最底部的一排的下面还有一个额外的辅助点，在isFull方法中由于这个辅助点的存在导致了backwash。

按照我们的本意，这个辅助点应该**只在percolate方法中使用**，和最顶部的辅助点一起判断地图是否从上到下连通。isFull方法只需要判断某个点是否与最顶部的辅助点相连，**不需要使用最底部的辅助点**。然而在isFull方法中这个辅助点也被使用了，这与我们的本意相违背。

所以percolate需要底部的辅助点，isFull方法不需要底部的辅助点，即这两个方法对UnionFind的需求不一致，所以我们可以创造两个UnionFind对象，分别适用于两个不同的方法。

```java
import edu.princeton.cs.algs4.WeightedQuickUnionUF;

public class Percolation {
    private boolean[][]grid;
    private int N;
    private WeightedQuickUnionUF UF;
    //用来避免backwash的UF
    private WeightedQuickUnionUF UFwithoutBackWash;
    private int numberOfOpenSites;
    public Percolation(int N){
        if (N<=0)
            throw new IllegalArgumentException("N must > 0");
        this.N=N;
        this.numberOfOpenSites=0;
        UF=new WeightedQuickUnionUF(N*N+2);
        UFwithoutBackWash=new WeightedQuickUnionUF(N*N+2);
        grid=new boolean[N][N];
    }
    public void open(int row,int col){
        int next[][]=new int[][]{
                {0,1},
                {1,0},
                {0,-1},
                {-1,0}
        };
        if (row<0||row>=N||col<0||col>=N)
            throw new IllegalArgumentException("row and col must between 0 and N-1");
        if (isOpen(row,col))return;
        grid[row][col]=true;
        numberOfOpenSites++;
        //连接周围的open的点
        for (int i=0;i<=3;i++){
            int mx=row+next[i][0];
            int my=col+next[i][1];
            if (my<0||my>=N)
                continue;
            if (mx==-1){
                //两个UnionFind都需要使用最顶部的辅助点
                UFwithoutBackWash.union(xyTo1D(row, col),N*N);
                UF.union(xyTo1D(row, col),N*N);
                continue;
            }
            else if (mx==N){
                //只有一个UnionFind需要使用最底部的辅助点
                UF.union(xyTo1D(row, col),N*N+1);
                continue;
            }
            //判断两个点之间有没有连接不需要底部的辅助点
            if (isOpen(mx,my)&&UFwithoutBackWash.connected(xyTo1D(row,col),xyTo1D(mx,my))==false){
                UF.union(xyTo1D(row, col),xyTo1D(mx,my));
                UFwithoutBackWash.union(xyTo1D(row, col),xyTo1D(mx,my));
            }
        }
    }
    private int xyTo1D(int mx,int my){
        return mx*N+my;
    }
    // is the site (row, col) open?
    public boolean isOpen(int row, int col){
        if (row<0||row>=N||col<0||col>=N)
            throw new IllegalArgumentException("row and col must between 0 and N-1");
        return grid[row][col];
    }
    // is the site (row, col) full?
    public boolean isFull(int row, int col){
        if (row<0||row>=N||col<0||col>=N)
            throw new IllegalArgumentException("row and col must between 0 and N-1");
        //在isFull中使用UFwithoutBackWash
        return UFwithoutBackWash.connected(xyTo1D(row, col), N * N);
    }
    // number of open sites
    public int numberOfOpenSites(){
        return numberOfOpenSites;
    }
    // does the system percolate?
    public boolean percolates(){
        //
        return UF.connected(N*N,N*N+1);
    }
    // use for unit testing (not required)
}

```

`PercolationStats`：

```java
import edu.princeton.cs.algs4.StdRandom;
import edu.princeton.cs.algs4.StdStats;

public class PercolationStats {
    private int T;
    private double[]x;
    // perform T independent experiments on an N-by-N grid
    public PercolationStats(int N, int T, PercolationFactory pf){
        if (N<=0||T<=0)
            throw new IllegalArgumentException("N and T must > 0");
        this.T=T;
        x=new double[T];

        for (int i=0;i<T;i++){
            Percolation test=pf.make(N);
            while (!test.percolates()){
                int randomRow= StdRandom.uniform(N);
                int randomCol = StdRandom.uniform(N);
                test.open(randomRow,randomCol);
            }
            x[i]=(double) test.numberOfOpenSites()/(N*N);
        }
    }
    // sample mean of percolation threshold
    public double mean(){
        return StdStats.mean(x);
    }
    // sample standard deviation of percolation threshold
    public double stddev(){
        return StdStats.stddev(x);
    }
    // low endpoint of 95% confidence interval
    public double confidenceLow(){
        return mean()-1.96*stddev()/Math.sqrt(T);
    }
    // high endpoint of 95% confidence interval
    public double confidenceHigh(){
        return mean() + 1.96 * stddev() / Math.sqrt(T);
    }
}

```

Autograder分数：

![image-20210404205233752](https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210404205233752.png)

## lab 9

`BSTmap`：

```java
package lab9;

import java.util.Iterator;
import java.util.Set;
import java.util.TreeSet;

/**
 * Implementation of interface Map61B with BST as core data structure.
 *
 * @author Your name here
 */
public class BSTMap<K extends Comparable<K>, V> implements Map61B<K, V> {

    private class Node {
        /* (K, V) pair stored in this Node. */
        private K key;
        private V value;

        /* Children of this Node. */
        private Node left;
        private Node right;

        private Node(K k, V v) {
            key = k;
            value = v;
        }
    }
    private class Entry{
        private K key;
        private V value;
        public Entry(K k,V v){
            key=k;
            value=v;
        }
    }
    private Node root;  /* Root node of the tree. */
    private int size; /* The number of key-value pairs in the tree */
    /* Creates an empty BSTMap. */
    public BSTMap() {
        this.clear();
    }

    /* Removes all of the mappings from this map. */
    @Override
    public void clear() {
        root = null;
        size = 0;
    }

    /** Returns the value mapped to by KEY in the subtree rooted in P.
     *  or null if this map contains no mapping for the key.
     */
    private V getHelper(K key, Node p) {
        if(p==null)return null;
        if (key.compareTo(p.key)<0)return getHelper(key,p.left);
        else if (key.compareTo(p.key)>0)return getHelper(key, p.right);
        return p.value;
    }

    /** Returns the value to which the specified key is mapped, or null if this
     *  map contains no mapping for the key.
     */
    @Override
    public V get(K key) {
        return getHelper(key,root);
    }

    /** Returns a BSTMap rooted in p with (KEY, VALUE) added as a key-value mapping.
      * Or if p is null, it returns a one node BSTMap containing (KEY, VALUE).
     */
    private Node putHelper(K key, V value, Node p) {
        if (p==null){
            size++;
            return new Node(key, value);
        }
        if (key.compareTo(p.key)<0)p.left=putHelper(key,value,p.left);
        else if (key.compareTo(p.key)>0)p.right=putHelper(key,value,p.right);
        else p.value=value;
        return p;
    }

    /** Inserts the key KEY
     *  If it is already present, updates value to be VALUE.
     */
    @Override
    public void put(K key, V value) {
        root=putHelper(key,value,root);
    }

    /* Returns the number of key-value mappings in this map. */
    @Override
    public int size() {
        return size;
    }

    //////////////// EVERYTHING BELOW THIS LINE IS OPTIONAL ////////////////

    /* Returns a Set view of the keys contained in this map. */
    @Override
    public Set<K> keySet() {
        Set<K>keySet=new TreeSet<>();
        keySet(root,keySet);
        return keySet;
    }
    private void keySet(Node x,Set<K>keySet){
        if (x==null)return;
        keySet(x.left,keySet);
        keySet.add(x.key);
        keySet(x.right,keySet);
    }
//    private Set<Entry> EntrySet(){
//        Set<Entry>EntrySet=new TreeSet<>();
//        EntrySet(root,EntrySet);
//        return EntrySet;
//    }
//    private void EntrySet(Node x,Set<Entry>EntrySet){
//        if (x==null)return;
//        EntrySet(x.left,EntrySet);
//        EntrySet.add(new Entry(x.key,x.value));
//        EntrySet(x.right,EntrySet);
//    }
    /** Removes KEY from the tree if present
     *  returns VALUE removed,
     *  null on failed removal.
     */
    @Override
    public V remove(K key) {
        root=remove(key,root);
        return valueRemoved;
    }
    private V valueRemoved;
    private Node remove(K key,Node x){
        if (x==null)return null;
        if (key.compareTo(x.key)<0)x.left=remove(key,x.left);
        else if(key.compareTo(x.key)>0)x.right=remove(key,x.right);
        else{
            if (x.right==null)return x.left;
            if (x.left==null)return x.right;
            Node t=Min(x.right);
            t.right=removeMin(x.right);
            t.left=x.left;
            valueRemoved=x.value;
            return t;
        }
        return x;
    }
    private Node Min(Node x){
        if (x.left==null)return x;
        return Min(x.left);
    }
    private Node removeMin(Node x){
        if (x.left==null)return x.right;
        x.left=removeMin(x.left);
        size--;
        return x;
    }
    /** Removes the key-value entry for the specified key only if it is
     *  currently mapped to the specified value.  Returns the VALUE removed,
     *  null on failed removal.
     **/
    @Override
    public V remove(K key, V value) {
        if (get(key)!=value)return null;
        return remove(key);
    }

    @Override
    public Iterator<K> iterator() {
        return new BSTIterator();
    }
    private class BSTIterator implements Iterator<K>{
        private Iterator<K> cur;
        public BSTIterator(){
            cur=keySet().iterator();
        }
        @Override
        public boolean hasNext() {
            return cur.hasNext();
        }

        @Override
        public K next() {
            return cur.next();
        }
    }
}

```

`Myhashmap`：

```java
package lab9;

import java.lang.reflect.Array;
import java.util.Iterator;
import java.util.Set;

/**
 *  A hash table-backed Map implementation. Provides amortized constant time
 *  access to elements via get(), remove(), and put() in the best case.
 *
 *  @author Your name here
 */
public class MyHashMap<K, V> implements Map61B<K, V> {

    private static final int DEFAULT_SIZE = 16;
    private static final double MAX_LF = 0.75;

    private ArrayMap<K, V>[] buckets;
    private int size;

    private int loadFactor() {
        return size / buckets.length;
    }

    public MyHashMap() {
        buckets = new ArrayMap[DEFAULT_SIZE];
        this.clear();
    }

    /* Removes all of the mappings from this map. */
    @Override
    public void clear() {
        this.size = 0;
        for (int i = 0; i < this.buckets.length; i += 1) {
            this.buckets[i] = new ArrayMap<>();
        }
    }

    /** Computes the hash function of the given key. Consists of
     *  computing the hashcode, followed by modding by the number of buckets.
     *  To handle negative numbers properly, uses floorMod instead of %.
     */
    private int hash(K key) {
        if (key == null) {
            return 0;
        }

        int numBuckets = buckets.length;
        return Math.floorMod(key.hashCode(), numBuckets);
    }

    /* Returns the value to which the specified key is mapped, or null if this
     * map contains no mapping for the key.
     */
    @Override
    public V get(K key) {
        return buckets[hash(key)].get(key);
    }

    /* Associates the specified value with the specified key in this map. */
    @Override
    public void put(K key, V value) {
        if (loadFactor()>MAX_LF)
            resize(buckets.length*2);
        if (!containsKey(key))
            size++;
        buckets[hash(key)].put(key,value);
    }
    private void resize(int capacity){
        ArrayMap<K,V>[]temp=buckets;
        buckets=(ArrayMap<K,V>[])new ArrayMap[capacity];
        this.clear();
        for (int i=0;i<temp.length;i++){
            Set<K>set=temp[i].keySet();
            Iterator<K>it=set.iterator();
            while (it.hasNext()){
                K nextkey=it.next();
                put(nextkey,temp[i].get(nextkey));
            }
        }
    }
    /* Returns the number of key-value mappings in this map. */
    @Override
    public int size() {
        return size;
    }

    //////////////// EVERYTHING BELOW THIS LINE IS OPTIONAL ////////////////

    /* Returns a Set view of the keys contained in this map. */
    @Override
    public Set<K> keySet() {
        throw new UnsupportedOperationException();
    }

    /* Removes the mapping for the specified key from this map if exists.
     * Not required for this lab. If you don't implement this, throw an
     * UnsupportedOperationException. */
    @Override
    public V remove(K key) {
        throw new UnsupportedOperationException();
    }

    /* Removes the entry for the specified key only if it is currently mapped to
     * the specified value. Not required for this lab. If you don't implement this,
     * throw an UnsupportedOperationException.*/
    @Override
    public V remove(K key, V value) {
        throw new UnsupportedOperationException();
    }

    @Override
    public Iterator<K> iterator() {
        throw new UnsupportedOperationException();
    }
}

```

![image-20210405013612011](https://cdn.jsdelivr.net/gh/BoL0150/imgbed@main/image-20210405013612011.png)