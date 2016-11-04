# ConcurrentHashMap

## 1. before JDK 1.8
&#12288;&#12288;采用分段锁实现并发操作，底层采用数组+链表+红黑树的存储结构。包含两个核心静态内部类 `Segment`和`HashEntry`:
1. `Segment`继承`ReentrantLock`充当锁，每个`Segment`对象守护每个散列映射表的若干个桶。
2. `HashEntry`用来封装映射表的键 / 值对；

&#12288;&#12288;每个桶是由若干个`HashEntry`链接起来的链表。一个ConcurrentHashMap实例中包含由若干个`Segment`对象组成的数组：

![ConcurrentHashMap before JDK 1.8](./Images/chm_before1.8.png)


## 2. JDK 1.8
### 2.1 相关概念：
* table：默认为`null`，初始化发生在第一次插入操作，默认大小为_16_的数组，用来存储Node节点数据，扩容时大小总是_2_的幂次方。
* nextTable：默认为`null`，扩容时新生成的数组，其大小为原数组的两倍。
* sizeCtl ：默认为_0_，用来控制table的初始化和扩容操作：
    * _-1_代表table正在初始化
    * _-N_表示有_N - 1_个线程正在进行扩容操作
    * 其余情况：
        * 如果table未初始化，表示table需要初始化的大小。
        * 如果table初始化完成，表示table的容量，默认是table大小的0.75倍。
* Node：保存key，value及key的hash值的数据结构。其中value和next都用`volatile`修饰，保证并发的可见性。Node类没有提供修改入口，只能用于只读遍历。
* ForwardingNode：特殊的Node节点，hash值为_-1_，存储nextTable的引用。只有table扩容时，ForwardingNode才会发挥作用，作为一个占位符放在table中表示当前节点为`null`或则已经被移动。

```java
final class ForwardingNode<K, V> extends Node<K, V> {
    final Node<K, V>[] nextTable;
    ForwardingNode(Node<K, V>[] tab) {
        super(MOVED, null, null, null);
        this.nextTable = tab;
    }
}
```

* TreeBins: 用于封装维护TreeNode，包含`putTreeVal`、`lookRoot`、`remove`、`balanceInsetion`、`balanceDeletion`等方法。当链表转树时，用于封装TreeNode，即ConcurrentHashMap的红黑树存放的是TreeBin，而不是treeNode。
* 一系列标示：
```java
 static final int MOVED     = -1; // hash for forwarding nodes
 static final int TREEBIN   = -2; // hash for roots of trees
private transient volatile int transferIndex; // 扩容另一个表索引
private transient volatile int cellsBusy; // 旋转锁
```


### 2.2 table初始化：
&#12288;&#12288;table初始化操作会延缓到第一次`put`行为。`sizeCtl`默认为_0_。如果ConcurrentHashMap实例化时有传参数，`sizeCtl`会是一个_2_的幂次方的值。所以执行第一次`put`操作的线程会执行`Unsafe.compareAndSwapInt`方法修改`sizeCtl`为_-1_。有且只有一个线程能够修改成功，其它线程通过`Thread.yield()`让出CPU时间片等待table初始化完成。

```java
private final Node<K, V>[] initTable() {
    Node<K, V>[] tab; int sc;
    while((tab = table) == null || tab.length == 0) {
    // 如果一个线程发现sizeCtl < 0,
    // 意味着另外的线程执行CAS成功，
    // 当前线程只需让出CPU时间片
        if((sc = sizeCtrl) < 0) {
            Thread.yield(); // lost initialization race; just spin
        } else if(U.CompareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                if ((tab = table) = null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K, V>[] nt = (Node<K, V>[])new Node<?, ?>[n];
                    table = tab = nt;
                    sc = n - (n >> 2);
                }
            } finally {
                sizeCtl = src;
            }
            break;
        }
    }

    return tab;
}
```


### 2.3 



