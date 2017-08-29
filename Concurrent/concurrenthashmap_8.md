# <center>ConcurrentHashMap (JDK 1.8)</center>

<br></br>

* 达到一定阀值，hash桶转换为红黑树。
* PUT操作基于CAS(Unsafe类)+synchronized实现并发插入或更新操作：
    * 插入更新用CAS
    * 红黑树转换用synchronized
* GET是对tab原子读取。不加锁因为用了`Unsafe.getObjectVolatile()`，因为table是volatile，所以对tab原子请求是可见的。根据happens-before原则，对volatile域的写入操作happens-before于每一个后续对同一域的读操作。所以不管其他线程对table链表或树的修改，都对get读取可见。

<br></br>

## 相关概念：
-----------
* table：默认为`null`，初始化发生在第一次插入操作，默认大小为_16_的数组，存储Node节点数据，扩容时大小总是_2_的幂次方。
* nextTable：默认为`null`，扩容时新生成的数组，其大小为原数组的两倍。
* sizeCtl ：默认为_0_，用来控制table的初始化和扩容操作：
    * _-1_代表table正在初始化
    * _-N_表示有_N - 1_个线程正在进行扩容操作
    * 其余情况：
        * 如果table未初始化，表示table需要初始化的大小。
        * 如果table初始化完成，表示table的容量，默认是table大小的0.75倍。
* Node：保存key，value及key的hash值的数据结构。其中value和next都用`volatile`修饰，保证并发的可见性。Node类没有提供修改入口，只能用于只读遍历。
* ForwardingNode：特殊的Node节点，hash值为_-1_，存储nextTable的引用。只有table扩容时，ForwardingNode才会发挥作用，作为一个占位符放在table中表示当前节点为`null`或则已经被移动。

``` java
final class ForwardingNode<K, V> extends Node<K, V> {
    final Node<K, V>[] nextTable;
    ForwardingNode(Node<K, V>[] tab) {
        super(MOVED, null, null, null);
        this.nextTable = tab;
    }
}
```

* TreeBins: 用于封装维护TreeNode。当链表转树时，用于封装TreeNode，即ConcurrentHashMap的红黑树存放的是TreeBin，而不是treeNode。
* 一系列标示：
``` java
 static final int MOVED     = -1; // hash for forwarding nodes
 static final int TREEBIN   = -2; // hash for roots of trees
private transient volatile int transferIndex; // 扩容另一个表索引
private transient volatile int cellsBusy; // 旋转锁
```

<br>



## table初始化
---------------
table初始化操作会延缓到第一次put行为。`sizeCtl`默认为_0_。如果ConcurrentHashMap实例化时有传参数，`sizeCtl`会是一个_2_的幂次方的值。所以执行第一次put操作的线程会执行`Unsafe.compareAndSwapInt()`方法修改`sizeCtl`为_-1_。有且只有一个线程能够修改成功，其它线程通过`Thread.yield()`让出CPU时间片等待table初始化完成。

``` java
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

<br>



## PUT操作
------------
采用CAS + synchronized实现并发插入或更新操作:
1. 判断存储的key、value是否为空，若为空，则抛出异常，否则，进入步骤2;
2. 计算key的hash值，随后进入无限循环，该无限循环可以确保成功插入数据，若table表为空或者长度为_0_，则初始化table表，否则，进入步骤3;
3. 根据key的hash值取出table表中结点，若取出结点为空（该桶为空），则用`CAS`将key、value、hash值生成的结点放入桶中。否则，进入步骤4;
4. 若该结点hash值为`MOVED`，则对该桶中结点进行转移，否则，进入步骤5;
5. 对桶中第一个结点（即table表中结点）进行加锁，对该桶遍历，桶中的结点的hash值与key值与给定的hash值和key值相等，则根据标识选择是否进行更新操作（用给定的value值替换该结点的value值），若遍历完桶仍没有找到hash值与key值和指定的hash值与key值相等的结点，则直接新生一个结点并赋值为之前最后一个结点的下一个结点。进入步骤6;
6. 若`binCount`值达到红黑树转化的阈值，则将桶中的结构转化为红黑树存储，最后，增加`binCount`的值。

```java
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null)
        throw new NullPointerException();
    int hash = spread(key.hashCode()), binCount = 0;
    for (Node<K, V>[] tab = table; ;) {
        Node<K, V> f;
        int n, i, fh;
        if (tab == null || (n = tab.length) == 0) {
            tab = initTable();
        } else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null, new Node<K, V>(hash, key, value, null)))
                break;  // no lock when adding to empty bin
        } else if ((fh == f.hash) == MOVED) {
            tab = helpTransfer(tab, f);
            // 省略部分代码
        }
    }
    addCount(1L, binCount);

    return null;
}
```

其中的hash算法为：
``` java
static final int spread(int h) {return (h ^ (h >>> 16)) & HASH_BITS;}
```

获取table中对应索引的元素f采用`Unsafe.getObjectVolatile()`来获取。每个线程都有一个工作内存，存储着table副本，虽然table是`volatile`修饰，但不能保证每次都拿到table中最新元素，`Unsafe.getObjectVolatile`可直接获取指定内存的数据，保证了每次拿到数据都是最新的。

如果f为`null`，说明这个位置第一次插入元素，用`Unsafe.compareAndSwapObject()`方法插入Node节点:
* 如果CAS成功，说明Node节点已插入，随后`addCount(1L, binCount)`会检查当前容量是否需要进行扩容。
* 如果CAS失败，说明有其它线程提前插入节点，自旋重新尝试插入节点。

如果f的hash值为_-1_，说明当前f是ForwardingNode节点，意味有线程正在扩容，则一起进行扩容操作。其余情况把新节点按链表或红黑树方式，采用同步内置锁实现并发:

``` java
synchronized (f) {
    if (tabAt(tab, 1) == f) {
        if (fh >= 0) {
            binCount = 1;
            for (Node<K,V> e = f; ; ++binCount) {
                K ek;
                if (e.hash == hash && ((ek = e.key) == key || 
                    (ek != null && key.equals(ek)))) {
                    oldVal = e.val;
                    if (!onlyIfAbsent)
                        e.val = value;
                    break;
                }
                Node<K, V> pred = e;
                if ((e = e.next) == null) {
                    pred.next = new Node<K, V>(hash, key, value, null);
                    break;
                }
            }
        } else if (f instanseof TreeBin) {
            Node<K, V> p;
            binCount = 2;
            if ((p = ((TreeBin<K, V>)f).putTreeVal(hash, key, value)) != null) {
                oldVal = p.val;
                if (!onlyIfAbsent)
                    p.val = value;
            }
        }
    } // if
}
```

在节点f上同步，插入前，再次利用`tabAt(tab, i) == f`判断，防止其它线程修改:
* 如果`f.hash >= 0`，说明f是链表结构的头结点，遍历链表，如果找到对应的node节点，则修改value，否则在链表尾部加入节点。
* 如果f是TreeBin类型，说明f是红黑树根节点，则在树结构上遍历元素，更新或增加节点。
* 如果链表中节点数`binCount >= TREEIFY_THRESHOLD`(默认是8)，则把链表转化为红黑树结构。

<br>



## putVal中涉及的函数
---------------
### initTable
对于table大小，会根据`sizeCtl`值进行设置，如果没有设置`szieCtl`值，那么默认table大小为_16_：

``` java
private final Node<K,V>[] initTable() {
        Node<K,V>[] tab; int sc;
        while ((tab = table) == null || tab.length == 0) { // 无限循环
            if ((sc = sizeCtl) < 0) // sizeCtl小于0，则进行线程让步等待
                Thread.yield(); // lost initialization race; just spin
            // 比较sizeCtl的值与sc是否相等，相等则用-1替换
            else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) { 
                try {
                    if ((tab = table) == null || tab.length == 0) {
                        // sc的值是否大于0，若是，则n为sc，否则，n为默认初始容量
                        int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                        // 新生结点数组
                        Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n]; 
                        table = tab = nt; // 赋值给table
                        sc = n - (n >>> 2); // sc为n * 3/4
                    }
                } finally {
                    sizeCtl = sc; // 设置sizeCtl的值
                }
                break;
            }
        }
        return tab; // 返回table表
    }
```

<br>


### tabAt
返回table数组中下标为`i`的结点。是通过Unsafe反射获取的，`getObjectVolatile()`的第二项参数为下标为`i`的偏移地址:

``` java
static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i,
                                   Node<K,V> c, Node<K,V> v) {
        return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
}
```

<br>


### casTabAt
用于比较table数组下标为`i`的结点是否为`c`，若为`c`，则用`v`交换。否则，不进行交换。

``` java
    // 利用CAS算法设置i位置上的Node节点（将c和table[i]比较，相同则插入v）。  
    static final <k,v> boolean casTabAt(Node<k,v>[] tab, int i,  
                                        Node<k,v> c, Node<k,v> v) {  
        return U.compareAndSwapObject(tab, ((long)i << ASHIFT) + ABASE, c, v);  
}  
```

<br>


### setTabAt

``` java 
    // 设置节点位置的值，仅在上锁区被调用  
    static final <k,v> void setTabAt(Node<k,v>[] tab, int i, Node<k,v> v) {  
        U.putObjectVolatile(tab, ((long)i << ASHIFT) + ABASE, v);  
    }
```

<br>


### helpTransfer
用于在扩容时将table表中的结点转移到nextTable中:

``` java
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
        Node<K,V>[] nextTab; int sc;
        // table表不为空并且结点类型使ForwardingNode类型，
        // 并且结点的nextTable不为空
        if (tab != null && (f instanceof ForwardingNode) &&
            (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) { 
            int rs = resizeStamp(tab.length);
            while (nextTab == nextTable && table == tab &&
                   (sc = sizeCtl) < 0) { // 条件判断
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || transferIndex <= 0) // 
                    break;
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) { // 比较并交换
                    transfer(tab, nextTab); // 将table的结点转移到nextTab中
                    break;
                }
            }
            return nextTab;
        }
        return table;
    }
```

<br>


## table扩容
-----------------
table元素数量达到阈值`sizeCtl`，需扩容，扩容分为两部分：

1. 构建一个nextTable，大小为table的_2_倍。
2. 把table的数据复制到nextTable中。

第一步构建nextTable，只能单线程进行nextTable初始化。通过`Unsafe.compareAndSwapInt()`修改`sizeCtl`值。节点从table移到nextTable：

1. 根据运算得到需要遍历的次数`i`，利用`tabAt()`方法获得`i`位置元素`f`，初始化forwardNode实例`fwd`。
2. 如果`f == null`，在`i`位置放入`fwd`，采用`Unsafe.compareAndSwapObjectf()`方法实现。
3. 如果`f`是链表头节点，构造一个反序链表，把他们放在nextTable的`i`和`i + n`位置，移动完成，用`Unsafe.putObjectVolatile给table()`原位置赋值`fwd`。
4. 如果`f`是`TreeBin`节点，也做反序处理并判断是否需要`untreeify()`，把处理的结果分别放在nextTable的`i`和`i + n`位置上，移动完成，采用`Unsafe.putObjectVolatile()`方法给table原位置赋值`fwd`。

遍历过所有节点以后就完成了复制工作，把table指向nextTable，并更新`sizeCtl`为新数组大小的0.75倍 ，扩容完成。

```java
private final void addCount(long x, int check) {
    // ...省略部分代码
    if (check >= 0) {
        Node<K, V>[] tab, nt;
        int n, sc;
        while (s >= (long)(sc = sizeCtl) && 
               (tab = table) != null && 
               (n = tab.length) < MAXIMUM_CAPACITY) {
            int rs = resizeStamp(n);
            if (sc < 0) {
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || 
                    sc == rs + 1 || sc == rs + MAX_RESIZERS || 
                    (nt = nextTable) == null || transferIndex <= 0)
                    break;
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            } else if (U.compareAndSwapInt(this, SIZECTL, sc, 
                       (rs <<< RESIZE_STAMP_SHIFT) + 2))
                transfer(tab, null);
            s = sumCount();
        }
    }
}
```

<br>


## 红黑树转化
----------
如果链表中元素超过`TREEIFY_THRESHOLD`阈值（默认为8），则把链表转化为红黑树：

``` java
if (bitCount != 0) {
    if (binCount >= TREEIFY_THRESHOLD)
        treeifyBin(tab, 1);
    if (oldVal != null)
        return oldVal;
    break;
}
```

红黑树构造代码如下：

``` java
private final void treeifyBin(Node<K, V>[] tab, int index) {
    Node<K, V> b;
    int n, sc;
    if (tab != null) {
        if ((n = tab.length) < MIN_TREEIFY_CAPACITY)
            tryPresize(n << 1);
        else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
            synchronized (b) {
                if (tabAt(tab, index) == b) {
                    TreeNode<K, V> hd = null, tl = null;
                    for (Node<K, V> e = b; e != nulll; e = e.next) {
                        TreeNode<K, V> p = new TreeNode<K, V>(e.hash, e.key, e.val, null, null);
                        if ((p.prev = tl) == null)
                            hd = p;
                        else
                            tl.next = p;
                        tl = p;
                    } // for
                    setTabAt(tab, index, new TreeBin<K, V>(hd));
                } // if (tabAt(tab, index) == b)
            } // synchronized
        }
    }
}
```

生成树节点的代码块是同步的，进入同步块后再次验证table中`index`位置元素是否被修改过：
1. 根据table中`index`位置Node链表，重新生成一个`hd`为头结点的TreeNode链表。
2. 根据`hd`头结点，生成TreeBin树，并把树`root`节点写到table的`index`位置，具体实现如下：

``` java
TreeBin(TreeNode<K, V> b) {
    super(TREEBIN, null, null, null);
    this.first = b;
    TreeNode<K, V> r = null;
    for (TreeNode<K, V> x = b, next; x != null; x = next) {
        next = (TreeNode<K, V>)x.next;
        x.left = x.right = null;
        if (r == null) {
            x.parent = null;
            x.red = false;
            r = x;
        } else {
            K k = x.key;
            int h = x.hash;
            Class<?> kc = null;
            for (TreeNode<K, V> p = r; ;) {
                int dir, ph;
                K pk = p.key;
                if ((ph = p.hash) > h)
                    dir = -1;
                else if (ph < h)
                    dir = 1;
                else if ((kc == null && (kc = comparableClassFor(k)) == null 
                         || (dir = compareComparables(kc, k, pk)) == 0))
                    dir = tieBreakOrder(k, pk);
                TreeNode<K, V> xp = p;
                if ((p = (dir <= 0) ? p.left : p.right) == null) {
                    x.parent = xp;
                    if (dir <= 0)
                        xp.left = x;
                    else
                        xp.right = x;
                    r = balanceInsetion(r, x);
                    break;
                }
            } // inner for
        } // if-else
    } // out for
    this.root = r;
    assert checkinvariants(root);
}
```

<br>


## get操作
----------------
1. 判断table是否为空，如果为空返回`null`。
2. 计算key的hash值，并获取指定table中指定位置的Node节点，通过遍历链表或则树结构找到对应的节点，返回value。

``` java
public V get(Object key) {
    Node<K, V>[] tab, e, p;
    int n, eh, h = spread(key,hashCode());
    K ek;
    if ((tab = table) != null && (n = tabl.length) > 0 && 
        (e = tabAt(tab, (n - 1) & h)) != null) {
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        } else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;
        while ((e = e.next) != null) {
            if (e.hash == h && ((ek = e.key) == key || 
                (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```

get请求需CAS保证变量原子性。如果`tab[i]`被锁住，那么CAS会失败，失败之后会不断的重试。保证了get在高并发情况下不会出错。到底有多少种情况会导致get在并发情况下可能取不到值?

1. 一个线程在get，另一个线程对同一个key的node进行remove；
2. 一个线程在get，另一个线程正重排table。可能导致旧table取不到值。

`get`是怎么保证同步的呢？注意`e = tabAt(tab, (n - 1) & h)) != null`，`tablAt()`方法是对`tab[i]`进行原子性读取。不需要加锁是因为用了`Unsafe.getObjectVolatile()`，因为table是volatile，所以对`tab[i]`原子请求也是可见的。根据happens-before原则，对volatile域的写入操作happens-before于每一个后续对同一域的读操作。所以不管其他线程对table链表或树的修改，都对get读取可见。 
        






