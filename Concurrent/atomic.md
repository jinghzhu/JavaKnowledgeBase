# <center>Atomic</center>

<br></br>



Java可以原子更新的基本类型有三个：
1. AtomicBoolean
2. AtomicInteger
3. AtomicLong

Java提供的原子操作，还可以原子更新以下类型的值：
1. 原子更新数组，`Atomic`包提供了以下几个类：`AtomicIntegerArray`、`AtomicLongArray`、`AtomicReferenceArray`。
2. 原子更新引用类型，也就是更新实体类的值，比如`AtomicReference`（原子更新引用类型的值）; `AtomicReferenceFieldUpdater`（原子更新引用类型里的字）; `AtomicMarkableReference`（原子更新带有标记位的引用类型）。

Atomic是乐观锁，原理是基于CAS：

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
