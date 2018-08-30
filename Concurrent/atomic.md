# <center>Atomic</center>

<br></br>



## What
----
**Atomic是乐观锁，原理是基于CAS：**

``` java
public final int getAndIncrement() {
    for (;;) {
        int current = get();
        int next = current + 1;
        if (compareAndSet(current, next))
            return current;
    }
}

public final boolean compareAndSet(int expect, int update) {
    return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
}
```

Java可原子更新基本类型有三个：
1. AtomicBoolean
2. AtomicInteger
3. AtomicLong

Java提供的原子操作，还可以原子更新以下类型的值：
1. 原子更新数组。`Atomic`包提供类`AtomicIntegerArray`、`AtomicLongArray`、`AtomicReferenceArray`。
2. 原子更新引用类型，也就是更新实体类的值，比如`AtomicReference`（原子更新引用类型的值）；`AtomicReferenceFieldUpdater`（原子更新引用类型里的字）；`AtomicMarkableReference`（原子更新带有标记位的引用类型）。

<br></br>



## How
----
AtomicInteger通过CAS操作实现线程安全。Java的CAS操作会调用底层的unsafe类通过硬件机制实现原子操作，并且其中操作的内存地址存储的值声明为violate类型，保证了更新后可以第一时间获得最新值。

```java
public final int getAndIncrement() {
    for (;;) {
        int current = get();  // 取得AtomicInteger里存储的数值
        int next = current + 1;  // 加1
        if (compareAndSet(current, next))   // 调用compareAndSet执行原子更新操作
            return current;
    }
}
         
public final boolean compareAndSet(int expect, int update) {
    return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
}
```

AtomicInteger关键域有3个：
```java
// setup to use Unsafe.compareAndSwapInt for updates
private static final Unsafe unsafe = Unsafe.getUnsafe();
private static final long valueOffset;
static {    
         try {        
                valueOffset = unsafe.objectFieldOffset (AtomicInteger.class.getDeclaredField("value"));   
        } catch (Exception ex) { 
               throw new Error(ex); 
        }
    }
private volatile int value;
```