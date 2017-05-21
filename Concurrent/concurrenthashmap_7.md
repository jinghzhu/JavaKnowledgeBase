# <center>ConcurrentHashMap (before JDK 1.8)</center>



<br></br>

![ConcurrentHashMap before JDK 1.8](./Images/chm_before1.8.png)

<br></br>



## 内部数据结构
--------------
采用分段锁实现并发操作，底层采用数组+链表+红黑树的存储结构。包含两个核心静态内部类Segment和HashEntry:
1. Segment继承ReentrantLock充当锁，每个Segment对象守护每个散列映射表的若干个桶。
2. HashEntry封装映射表的键/值对；

每个桶是由若干HashEntry链接起来的链表。一个ConcurrentHashMap实例中包含由若干Segment对象组成的数组：

``` java
static final class HashEntry<K,V> {   
       final K key;                       // 声明 key 为 final 型  
       final int hash;                   // 声明 hash 值为 final 型   
       volatile V value;                 // 声明 value 为 volatile 型  
       final HashEntry<K,V> next;      // 声明 next 为 final 型   
  
       HashEntry(K key, int hash, HashEntry<K,V> next, V value) {   
           this.key = key;   
           this.hash = hash;   
           this.next = next;   
           this.value = value;   
       }   
}   
```

HashEntry中的`key`，`hash`，`next`都声明为final型。意味着不能把节点添加到链接的中间和尾部，也不能在链接的中间和尾部删除节点。这个特性保证在访问某个节点时，这个节点之后的链接不会被改变。大大降低处理链表时的复杂性。

```java
static final class Segment<K,V> extends ReentrantLock implements Serializable {   
  
        transient volatile int count;  // 在本segment范围内，包含的HashEntry元素个数  
                                       //volatile 型   
        transient int modCount;     //table 被更新的次数  
        transient int threshold;    //默认容量  
    	final float loadFactor;    //装载因子  
  
        /**  
         * table 是由 HashEntry 对象组成的数组 
         * 如果散列时发生碰撞，碰撞的 HashEntry 对象就以链表的形式链接成一个链表 
         * table 数组的数组成员代表散列映射表的一个桶         
         */   
        transient volatile HashEntry<K,V>[] table;   
  
        /**  
         * 根据key的hash值，找到table中对应的桶（table数组的某个成员） 
         * 把散列值与table数组长度减1的值相“与”，得到散列值对应的table数组下标 
         * 然后返回table数组中此下标对应的HashEntry元素,即这个段中链表的第一个元素 
         */   
        HashEntry<K,V> getFirst(int hash) {   
            HashEntry<K,V>[] tab = table;               
            return tab[hash & (tab.length - 1)];   
        }   
  
        Segment(int initialCapacity, float lf) {   
            loadFactor = lf;   
            setTable(HashEntry.<K,V>newArray(initialCapacity));   
        }   
  
        /**  
         * 设置 table 引用到这个新生成的 HashEntry 数组 
         * 只能在持有锁或构造函数中调用本方法 
         */   
        void setTable(HashEntry<K,V>[] newTable) {               
            threshold = (int)(newTable.length * loadFactor);   
            table = newTable;   
        }          
}   
```

<br></br>



## get方法
--------------
首先判断当前桶数据个数是否为0。否则，得到头节点后根据hash和key逐个判断是否是指定的值。如果是并且值非空，就说明找到了，直接返回。注意，返回结果时的`return readValueUnderLock(e)`是为了并发考虑的。当`v`为空时，可能是一个线程正在改变节点，而之前的get操作都未进行锁定，所以这里要对`e`重新上锁再读一遍，以保证得到的是正确值。
 
为什么get需判断`count`不等于_0_？这是用到happens-before原则之volatile变量法则：对volatile写入操作happens-before于每一个后续对同一域的读操作。

虽然线程N是在未加锁的情况下访问链表。Java内存模型可以保证只要之前对链表做结构性修改操作的写线程M在退出写方法前写volatile变量`count`，读线程N读取`count`后，定能看到修改。这个特性和HashEntry对象的不变性相结合，使读线程在读取散列表时，基本不需要加锁就能成功获得需要的值。这两个特性相配合，不仅减少了请求同一个锁的频率（读操作一般不需要加锁就能够成功获得值），也减少了持有同一个锁的时间（只有`value`为`null`, 读线程才要加锁后重读）。

``` java
V get(Object key, int hash) {  
    if (count != 0) { // read-volatile  
        HashEntry e = getFirst(hash);  
        while (e != null) {  
            if (e.hash == hash && key.equals(e.key)) {  
                V v = e.value;  
                if (v != null)  
                    return v;  
                return readValueUnderLock(e); // recheck  
            }  
            e = e.next;  
        }  
    }  
    return null;  
}  
  
V readValueUnderLock(HashEntry e) {  
    lock();  
    try {  
        return e.value;  
    } finally {  
        unlock();  
    }  
}  
```

<br></br>



## 写操作
--------------
一般情况下不需要加锁就可以完成，对容器做结构性修改的操作才需要加锁。下面以put操作为例说明对ConcurrentHashMap做结构性修改的过程。

首先，根据_key_计算对应的hash值：
```java
public V put(K key, V value) { 
        if (value == null)          //ConcurrentHashMap 中不允许用 null 作为映射值
            throw new NullPointerException(); 
        int hash = hash(key.hashCode());        // 计算键对应的散列码
        // 根据散列码找到对应的 Segment 
        return segmentFor(hash).put(key, hash, value, false); 
 }
 ```

然后，根据_hash_值找到对应的Segment对象：
```java
    /** 
     * 使用 key 的散列码来得到 segments 数组中对应的 Segment 
     */ 
 final Segment<K,V> segmentFor(int hash) { 
    // 将散列值右移 segmentShift 个位，并在高位填充 0 
    // 然后把得到的值与 segmentMask 相“与”
 // 从而得到 hash 值对应的 segments 数组的下标值
 // 最后根据下标值返回散列码对应的 Segment 对象
        return segments[(hash >>> segmentShift) & segmentMask]; 
 }
```

最后，在Segment中执行具体的put操作：
``` java
V put(K key, int hash, V value, boolean onlyIfAbsent) {  
    lock();  // 先锁定整个segment，因为修改数据不能并发进行。
    try {  
        int c = count;  
        if (c++ > threshold) // ensure capacity  
            rehash();  
        HashEntry[] tab = table;  
        int index = hash & (tab.length - 1);  
        HashEntry first = (HashEntry) tab[index];  
        HashEntry e = first;  
        while (e != null && (e.hash != hash || !key.equals(e.key)))  
            e = e.next;  
  
        V oldValue;  
        if (e != null) {  
            oldValue = e.value;  
            if (!onlyIfAbsent)  
                e.value = value;  
        }  
        else {  
            oldValue = null;  
            ++modCount;  
            tab[index] = new HashEntry(key, hash, first, value);  
            count = c; // write-volatile  
        }  
        return oldValue;  
    } finally {  
        unlock();  
    }  
}  
```

线程写入的两种情形：
1. 对散列表做非结构性修改的操作。
2. 对散列表做结构性修改的操作。

非结构性修改操作只更改某个HashEntry的`value`域的值。由于对volatile变量的写入操作与随后对这个变量的读操作同步。当写线程修改了某个HashEntry的`value`域后，另一个读线程读这个值域，Java内存模型保证读取的一定是更新后的值。所以，写线程对链表的非结构性修改能够被后续不加锁的读线程看到。

结构性修改是对某个桶指向的链表做结构性修改。如果能够确保在读线程遍历一个链表期间，写线程对这个链表所做的结构性修改不影响读线程继续正常遍历这个链表。那么读/写线程之间就可以安全并发访问这个ConcurrentHashMap。

结构性修改操作包括put，remove，clear。

clear操作只是把ConcurrentHashMap中所有的桶置空，每个桶之前引用的链表依然存在，只是桶不再引用到这些链表（所有链表的结构并没有被修改）。正在遍历某个链表的读线程依然可以正常执行对该链表的遍历。

put操作如果需插入一个新节点 , 会在链表头部插入这个新节点。此时，链表中的原有节点的链接并没有被修改。也就是说插入新健/值对到链表中的操作不会影响读线程正常遍历这个链表。

下面分析remove操作。

<br></br>



## remove操作
----------------------
类似put，但要注意一点区别，中间for循环是将定位之后的所有entry克隆并拼回前面去。这点是由entry 的不变性决定的。entry中除了value，其他所有属性都是final修饰。意味着在第一次设置了next域后便不能再改变，取而代之的是将它之前的节点全都克隆一次。entry要设置为不变性，因为不变性的访问不需要同步从而节省时间有关。

<p align="center">
  <img src="./Images/chm7_delete1.png" />
</p>

<center><i>删除元素之前</i></center>

<br>

<p align="center">
  <img src="./Images/chm7_delete2.png" />
</p>

<center><i>删除元素3之后</i></center>

<br>
 
``` java
V remove(Object key, int hash, Object value) {  
    lock();  
    try {  
        int c = count - 1;  
        HashEntry[] tab = table;  
        int index = hash & (tab.length - 1);  // 根据散列码找到table下标值
        HashEntry first = (HashEntry)tab[index];  // 找到散列码对应的桶
        HashEntry e = first;  
        while (e != null && (e.hash != hash || !key.equals(e.key)))  
            e = e.next;  
  
        V oldValue = null;  
        if (e != null) {  
            V v = e.value;  
            if (value == null || value.equals(v)) { // 找到要删除的节点
                oldValue = v;  
                // 所有处于待删除节点之后的节点原样保留在链表中
                // 所有处于待删除节点之前的节点被克隆到新链表中 
                ++modCount;  
                HashEntry newFirst = e.next;  // 待删节点的后继结点
                for (HashEntry p = first; p != e; p = p.next)  // 将定位后的所有entry克隆并拼回前面
                    newFirst = new HashEntry(p.key, p.hash,   
                                             newFirst, p.value);  
                // 把桶链接到新的头结点
                // 新的头结点是原链表中，删除节点之前的那个节点
                tab[index] = newFirst;  
                count = c; // write-volatile  
            }  
        }  
        return oldValue;  
    } finally {  
        unlock();  // 解锁
    }  
} 
``` 

<br></br>



## volatile变量协调读写线程间的内存可见性
----
由于内存可见性问题，未正确同步的情况下，写线程写入的值可能并不为后续的读线程可见。

<p align="center">
  <img src="./Images/chm7_violate1.jpg" />
</p>

<center><i>协调读-写线程间的内存可见性的示意图</i></center>

<br>

假设线程M写入volatile型变量`count`后，线程N读取了`count`。根据happens-before的程序次序法则，A appens-before于B，C happens-before D。根据volatile变量法则，B happens-before C。根据传递性，得到A appens-before 于 B，B appens-before C，C happens-before D。写线程M对链表做的结构性修改，在读线程N读取了同一个volatile变量后，对线程N也是可见的了。

ConcurrentHashMap中，每个Segment都有一个变量`count`。统计Segment中的HashEntry个数。这个变量被声明为volatile。

```java
 transient volatile int count;
```

所有不加锁读方法，在进入读方法时，首先都会去读这个`count`变量。比如get方法：

```java
V get(Object key, int hash) { 
    if(count != 0) {       // 首先读 count 变量
        HashEntry<K,V> e = getFirst(hash); 
        while(e != null) { 
            if(e.hash == hash && key.equals(e.key)) { 
                V v = e.value; 
                if(v != null)            
                    return v; 
                // 如果读到value域为null，说明发生了重排序，加锁后重新读取
                return readValueUnderLock(e); 
            } 
            e = e.next; 
        } 
    } 
    return null; 
}
```

所有执行写操作的方法（put, remove, clear），在对链表做结构性修改之后，在退出写方法前都会去写这个`count`变量。所有未加锁的读操作（get, contains, containsKey）在读方法中，都会首先去读取这个`count`变量。

根据Java内存模型，对同一个volatile变量的写/读操作可确保写线程写入的值，能够被之后未加锁的读线程看到。

这个特性和HashEntry对象的不变性相结合，使得在ConcurrentHashMap中，读线程基本不需加锁就能成功获得值。只有读到`value`域为null时 , 读线程才需要加锁后重读。

