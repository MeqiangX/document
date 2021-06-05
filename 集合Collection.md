集合常见知识点：

> HashMap

1、key-value键值对的hash表数据结构

2、jdk1.8后的版本采用 `数组+链表+红黑树`的数据结构存储

<kbd>数组</kbd>-具有随机访问的特性，堆空间连续，适合做查询，增/删/改的操作涉及数组的挪动，效率较低

<kbd>链表</kbd>-具有顺序连接的特性，堆空间不连续，增/删/改操作不涉及大量的结点挪动，效率较高，但是查询需要遍历随机的地址空间，不利于查找

<kbd>红黑树</kbd>-AVL自平衡二叉树的变种，也称为Self-balancing Binary Search Tree`自平衡二叉搜索树`，叫AVL树的起因是来源于发明这种人的名字（Adelson-Velsky-Landis Tree ）G. M. Adelson-Velsky和E. M. Landis，在BST(Binary-Search-Tree)二叉搜索树的概念上增加的平衡的概念，它规定任意节点的左右子树的高度差不大于1，原因在于如果BST树存在节点的左右子树高度差相差很大，他的搜索时间复杂度会从O(logN)退化成链表的O(N)；但是AVL的自平衡过程也十分消耗性能，所以出现了折中的方案：红黑树（Red-Black-Tree）

他定义：

	- 节点分别红/黑两种
	- 根节点是黑色
	- 红节点的左右两个节点是黑色
	- 任意及诶单的所有路基都包含相同数量的黑色节点

取3，4两点来看，左右子树的高度差会在k（左子树k(全黑)，右子树2k（红黑节点交替出现）），他综合了查询和平衡旋转性能损耗的效率

3、put，hash，扩容

put(key,value)，通过<kbd>hash(key) & (n-1)</kbd> 来获取到需要插入的`数组的下标`，这个&操作等同于 % n ，hash计算后得到32（int 4byte-32bit）位，只有最后的logn位，会用来计算下标，所以这要求map数组的长度必须是2的次幂，定位下标的方法会使得参与决定的只有hash的后4位，而jdk为了让数据更加的分散，并且让keyHash的高位也能参与进来，而有了这个移位异或操作：<kbd>(h = key.hashCode() ^ (h >> 16))</kbd>

```java
static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```

`h >>> 16`：让高位参与运算

`^`：为了让数据平均和更加分散

疑问1：为什么是^异或 而不是&或者|呢

答：考虑2bit的所有逻辑取值：

| 逻辑符号      | 0位  | 1位  | 结果 |
| ------------- | ---- | ---- | ---- |
| &             | 0    | 0    | 0    |
| &             | 0    | 1    | 0    |
| &             | 1    | 0    | 0    |
| &             | 1    | 1    | 1    |
| &趋向0 3/4    |      |      |      |
| \|            | 0    | 0    | 0    |
| \|            | 0    | 1    | 1    |
| \|            | 1    | 0    | 1    |
| \|            | 1    | 1    | 1    |
| \|趋向于1 3/4 |      |      |      |
| ^             | 0    | 0    | 1    |
| ^             | 0    | 1    | 0    |
| ^             | 1    | 0    | 0    |
| ^             | 1    | 1    | 1    |
| ^平均分布 1/2 |      |      |      |

结论：逻辑操作& 趋向0  | 趋向1，而^较平均

---

put的几个属性因子:

```java
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; //8 默认数组长度
static final int DEFAULT_LOAD_FACTOR = 0.75f;// 3/4 默认扩容因子
static final int TREEIFY_THRESHOLD = 8;// 默认树化链表长度，>8 链表->红黑树
static final int MIN_TREEIFY_CAPACITY = 64; // 默认树化的数组长度 >= 64
```

树化条件：

```java
if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);

final void treeifyBin(Node<K,V>[] tab, int hash) {
        int n, index; Node<K,V> e;
        if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
            resize();
// 链表长度 >= 8 && 数组长度 >= 64.  链表->红黑树
// treeifyBin 中  如果不满足 数组长度，会进行resize()
```

疑问1：树化条件。链表长度为何是8

答：

```java
//参考 TREEIFY_THRESHOLD 源码
给出的一个概率论-泊松分布的 基于科学数据统计，放入数组中的链表结点个数的一个出现概率
链表个数和概率的表格如下：
     * 0:    0.60653066
     * 1:    0.30326533
     * 2:    0.07581633
     * 3:    0.01263606
     * 4:    0.00157952
     * 5:    0.00015795
     * 6:    0.00001316
     * 7:    0.00000094
     * 8:    0.00000006
会发现，大部分90%+的概率 链表的结点数不会超过4，而出现8个结点的概率仅仅为 0.000006%
```

---

put的死循环问题：

1.8之前 链表的插入是用的头插法，单线程下的操作都是安全的，在多线程的情况下，有可能

出现线程A在遍历链表，rehash()的时候，另外一个线程也同时在运行，rehash()，重组散列和链表，发生循环引用从而造成cpu100%

```java
// 代码解释：
// 遍历某个链遍，进行rehash，重组，假设开始数组i=2,重组后，链表在i=5
p = head;
newList = list;
while(p != null){
  int i = rehash(p); //1 i 为新插入计算的数组下标
  insertNewList(i,p); //2 将节点插入到新的数组下标所在的链表头节点
  p = p.next;//3
}
rebuild();

// 此时有两个线程a,b同时对散列表进行扩容
// 如果此时链表结构为:A -> B 
// 线程a，执行到2 ，此时rehash完了A，还没插入到新的下标头节点，p = A; CPU线程b抢夺，b执行完重组后，没有被抢夺中断cpu，链表变成了B -> A;
// 此时线程a 获取了cpu执行权，再次循环，新链表[B->A]，线程a经过刚才抢断后，重新执行p=A插入到新链表,新链表[A->B->A]，执行到3，注意，新表中是AB节点的副本，不影响来源的节点结构，所以这里，线程a执行到3，p = B,发现新链表已经有了，而且B.next=null；退出循环，resize完成
// 而此时i=5的数组下标的链表[A->B->A]构成了循环，当通过get(hash(key)=5)来取数据的时候，就会发生死循环
```

而jdk1.8之后优化采用了尾插法+树化，但是也会有这种死循环情况，只是说情况没有头插法的时候出现频率高。无论是尾插还是树化 都不能解决死循环的问题。

伪初始化：new HashMap()并没有初始化，分配空间，真正是在put的时候初始化分配空间的

```java
public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }
// put:init
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
// resize():init/double size
 
    /**
     * Initializes or doubles table size.  If null, allocates in
     * accord with initial capacity target held in field threshold.
     * Otherwise, because we are using power-of-two expansion, the
     * elements from each bin must either stay at same index, or move
     * with a power of two offset in the new table.
     *
     * @return the table
     */
    final Node<K,V>[] resize() {
```

---

线程安全的散列表：

HashTable，集合工具类包装对象Collections.synchronizedMap()，JUC并发包下的ConcurrentHashMap

- HashTable重量锁全表锁不推荐，不算多线程，不使用

- synchronizedMap，内部加的信号量，synchronized (mutex)  也是重量锁，包装类，不使用

- JUC.ConcurrentHashMap 

  `jdk1.7` 分段锁（区间加锁synchronized，实现更多并发）

  `jdk1.8` CAS自旋+synchronized，读不加锁，对写put加锁，而且是针对到每个结点