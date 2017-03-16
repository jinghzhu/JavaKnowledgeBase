# <center>Thread Local</center>

<br></br>

> 类似一个全局的Map，key是线程。不同线程get时拿到的都是属于自己的对象，互相隔离，不存在并发问题。

<br></br>



### 典型的使用方式
-----

``` java
// ThreadLocal对象都是static的，全局共享
private static final ThreadLocal<ThreadLocalRandom> localRandom =  
    new ThreadLocal<ThreadLocalRandom>() {      // 初始值
        protected ThreadLocalRandom initialValue() {
            return new ThreadLocalRandom();
        }
};

localRandom.get();      // 拿当前线程对应的对象
localRandom.put(...);   // put
```


&#12288;&#12288;典型使用场景：
* 用空间换并发度；
* 在线程范围内传参，如hibernate的session。

<br></br>



### 实现
----
&#12288;&#12288;一个非常自然想法是用一个线程安全的`Map<Thread,Object>`实现：

``` java
class ThreadLocal {
  private Map values = Collections.synchronizedMap(new HashMap());

  public Object get() {
    Thread curThread = Thread.currentThread();
    Object o = values.get(curThread);
    if (o == null && !values.containsKey(curThread)) {
      o = initialValue();
      values.put(curThread, o);
    }
    return o;
  }

  public void set(Object newValue) {
    values.put(Thread.currentThread(), newValue);
  }
}
```

&#12288;&#12288;但这是非常naive的：
* `ThreadLocal`是避免并发，用一个全局`Map`显然违背了初衷；
* 用Thread当key，除非手动调用remove，否则即使线程退出了不仅该Thread对象无法回收，而且该线程在所有`ThreadLocal`中对应的value也无法回收。

&#12288;&#12288;JDK实现是反过来的： 
<p align="center">
  <img src="./Images/threadlocal.png" width="400" />
</p>

<br>

&#12288;&#12288;每个Thread对象内都存在一个`ThreadLocal.ThreadLocalMap`对象，保存着该线程所有用到的`ThreadLocal`及其value。`ThreadLocalMap`是定义在`ThreadLocal`类内部的私有类，是采用“开放定址法”解决冲突的hashmap。key是`ThreadLocal`对象。当调用`ThreadLocal`对象的`get()`或`put()`方法时，先会从当前线程中取出`ThreadLocalMap`，然后找对应的value：

``` java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);     //拿到当前线程的ThreadLocalMap
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);    // 以该ThreadLocal对象为key取value
        if (e != null)
            return (T)e.value;
    }
    return setInitialValue();
}

ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```

&#12288;&#12288;所有`ThreadLocal`对象共享一个`AtomicInteger`对象`nextHashCode`用于计算hashcode。算法是从0开始，以`HASH_INCREMENT = 0x61c88647`为间隔递增，这是`ThreadLocal`唯一需要同步的地方。根据hashcode定位桶的算法是将其与数组长度-1进行与操作：`key.threadLocalHashCode & (table.length - 1)`。

&#12288;&#12288;_0x61c88647_这个魔数是怎么确定的呢？

&#12288;&#12288;`ThreadLocalMap`初始长度为16，每次扩容都增长为原来2倍，即它的长度始终是$$ 2^n $$。使用_0x61c88647_可让hash结果在$$ 2^n $$内尽可能均匀分布，减少冲突概率。

<br></br>



### 内存管理
-----
&#12288;&#12288;`ThreadLocalMap.Entry`继承自WeakReference：

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

&#12288;&#12288;一旦某个ThreadLocal对象没有强引用了，它在所有线程内部的`ThreadLocalMap`中的key都将被GC掉（此时value还未回收），在map后续的get/set中会探测到key被回收的entry，将其value置为`null`以帮助GC。因此value在key被GC后还存活一段时间。这个过程和java.util.WeakHashMap实现一样。

&#12288;&#12288;一旦线程退出，Thread对象被回收了，内部`ThreadLocalMap`中的value也可以被回收了。但线程池场景需注意，线程池中线程可能永远不会退出，只会阻塞，如果队列中的某个任务向`ThreadLocal`对象`put()`了一个对象，但任务结束后并未清空，那么这个对象在`ThreadLocal`的下一次`put()`或`clear()`前永远不会被GC；这种情况下，假如线程池有200个线程，那么一个ThreadLocal最多可造成200个对象的内存泄露。