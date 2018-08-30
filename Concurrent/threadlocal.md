# <center>Thread Local</center>



<br></br>

* 类似全局Map，key是线程。不同线程get时拿到的都是属于自己的对象，互相隔离，不存在并发问题。
* ThreadLocal对象都是static，全局共享。
* 所有ThreadLocal对象共享一个AtomicInteger对象`nextHashCode`用于计算hashcode。这是ThreadLocal唯一需同步的地方。
* `ThreadLocalMap.Entry`继承自WeakReference，和java.util.WeakHashMap实现一样。

<br></br>



## 典型使用方式
-----

``` java
// ThreadLocal对象都是static，全局共享。
private static final ThreadLocal<ThreadLocalRandom> localRandom =  
    new ThreadLocal<ThreadLocalRandom>() {      // 初始值
        protected ThreadLocalRandom initialValue() {
            return new ThreadLocalRandom();
        }
}

localRandom.get();      // 拿当前线程对应的对象
localRandom.put(...);   // put
```

典型使用场景：
* 用空间换并发度；
* 在线程范围内传参，如Hibernate的session。

<br></br>



## 实现
----
**每个Thread对象内都存在一个`ThreadLocal.ThreadLocalMap`对象，保存该线程所有用到的ThreadLocal及value。ThreadLocalMap是定义在ThreadLocal类内部的私有类，采用开放地址法解决冲突的hashmap。key是ThreadLocal对象。当调用ThreadLocal对象`get()`或`put()`方法时，先会从当前线程中取出ThreadLocalMap，然后找对应的value：**

ThreadLocal通过`threadLocalHashCode`标识每一个ThreadLocal唯一性。`threadLocalHashCode`通过CAS更新，每次hash增量为0x61c88647。
所有ThreadLocal对象共享一个AtomicInteger对象，`nextHashCode`用于计算hashcode。算法是从0开始，以`HASH_INCREMENT = 0x61c88647`为间隔递增。这是ThreadLocal唯一需要同步的地方。根据Hashcode定位的算法将其与数组长度-1进行与操作：`key.threadLocalHashCode & (table.length - 1)`。

> _0x61c88647_这个魔数是怎么确定的呢？
> 
> ThreadLocalMap初始长度为16，每次扩容都增长为原来2倍，即它的长度始终是 $$ 2^n $$。使用_0x61c88647_可让hash结果在 $$ 2^n $$ 内尽可能均匀分布，减少冲突概率。

<br>


### Source Code
ThreadLocal类定义：
```java
public class ThreadLocal<T> {
    private final int threadLocalHashCode = nextHashCode();

    private static AtomicInteger nextHashCode =
        new AtomicInteger();

    private static final int HASH_INCREMENT = 0x61c88647;

    private static int nextHashCode() {
        return nextHashCode.getAndAdd(HASH_INCREMENT);
    }
}
```

set方法：
```java
public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
}

ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
}
```

通过`Thread.currentThread()`获取当前线程引用，并传给`getMap(Thread)`方法获取ThreadLocalMap实例。`getMap(Thread)`方法返回Thread实例成员变量`threadLocals`，它定义在Thread内部，访问级别为package：

```java
public class Thread implements Runnable {
    /* Make sure registerNatives is the first thing <clinit> does. */
    private static native void registerNatives();
    static {
        registerNatives();
    }

    /* ThreadLocal values pertaining to this thread. This map is maintained
     * by the ThreadLocal class. */
    ThreadLocal.ThreadLocalMap threadLocals = null;
    
}
```

可看出，每个Thread都有一个`ThreadLocal.ThreadLocalMap`成员变量，这样确保每个线程访问的thread-local variable都是本线程的。

get方法：
```java
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null)
                return (T)e.value;
        }
        return setInitialValue();
    }

    private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
        return value;
    }
```

通过`Thread.currentThread()`方法获取当前线程引用，并传给`getMap(Thread)`方法获取ThreadLocalMap实例。`setInitialValue()`方法首先调用`initialValue()`方法初始化，然后通过`Thread.currentThread()`方法获取当前线程引用，并传给`getMap(Thread)`方法获取ThreadLocalMap实例，并将初始化值存到ThreadLocalMap中。

<br>


### ThreadLocalMap
ThreadLocalMap是ThreadLocal静态内部类：

```java
public class ThreadLocal<T> {

    static class ThreadLocalMap {

        static class Entry extends WeakReference<ThreadLocal> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal k, Object v) {
                super(k);
                value = v;
            }
        }

        /**
         * The initial capacity -- MUST be a power of two.
         */
        private static final int INITIAL_CAPACITY = 16;

        /**
         * The table, resized as necessary.
         * table.length MUST always be a power of two.
         */
        private Entry[] table;

        /**
         * The number of entries in the table.
         */
        private int size = 0;

        /**
         * The next size value at which to resize.
         */
        private int threshold; // Default to 0

        ThreadLocalMap(ThreadLocal firstKey, Object firstValue) {
            table = new Entry[INITIAL_CAPACITY];
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
            table[i] = new Entry(firstKey, firstValue);
            size = 1;
            setThreshold(INITIAL_CAPACITY);
        }
    }
}
```

`INITIAL_CAPACITY`代表Map初始容量；`size`代表表中存储数目；`threshold`代表需扩容时对应`size`的阈值。

Entry类是ThreadLocalMap静态内部类，用于存储数据。继承`WeakReference<ThreadLocal<?>>`，即每个Entry对象都有一个ThreadLocal弱引用，防止内存泄露。一旦线程结束，key变为不可达对象，Entry就可被GC。

<br></br>



## 内存管理
-----
`ThreadLocalMap.Entry`继承自WeakReference：

``` java
static class Entry extends WeakReference<ThreadLocal> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal k, Object v) {
        super(k);
        value = v;
    }
}
```

一旦某个ThreadLocal对象没有强引用，它在所有线程内部ThreadLocalMap中key都将被GC。在Map后续get／set中会探测到key被回收的entry，将其value置为`null`以帮助GC。和java.util.WeakHashMap实现一样。

一旦线程退出，Thread对象被回收，内部ThreadLocalMap中value也可被回收。

但线程池场景需注意，线程池中线程可能永远不会退出，只会阻塞。如果队列中某个任务向ThreadLocal对象`put()`一个对象，但任务结束后并未清空。这个对象在ThreadLocal下一次`put()`或`clear()`前不会被GC。
