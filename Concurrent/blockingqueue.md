# <center>BlockingQueue</center>

<br></br>



> 默认非公平锁

<br></br>



BlockingQueue doesn’t accept `null` values and throw `NullPointerException` if store `null` value in the queue.

BlockingQueue是接口，使用它应类似`BlockingQueue q = new LinkedBlockingQueue(5);`。常用的阻塞队列有:
1. ArrayBlockingQueue：由数组支持的有界阻塞队列，其构造函数必须带一个参数来指明其大小.其所含的对象是以FIFO顺序排序的。之所以采用阻塞队列，是因为如果生产者的速度比消费者快很多，很容易就占用大量内存空间.
2. LinkedBlockingQueue：其构造函数可带也可不带规定大小的参数。 若不带大小参数,所生成的大小由`Integer.MAX_VALUE`决定.所含对象以FIFO排序。
3. PriorityBlockingQueue：类似于LinkedBlockQueue,但其所含对象的排序是依据对象的自然排序顺序或者是构造函数的Comparator决定的顺序。
4. SynchronousQueue：对其的操作必须是放和取交替完成的。 

其入队出队方法：
1. `add(Object)`: 如果BlockingQueue可以容纳,则返回`true`,否则异常
2. `offer(Object)`: 如果BlockingQueue可以容纳,则返回`true`,否则返回`false`.
3. `put(Object)`: 如果BlockQueue没有空间,则调用此方法的线程被阻断直到BlockingQueue里面有空间再继续.
4. `poll(time)`:取走排在首位的对象,若不能立即取出,则可以等time参数规定的时间,取不到时返回`null`。
5. `take()`:取走排在首位的对象,若BlockingQueue为空,阻断进入等待状态直到有新的对象被加入为止

原理：
* ArrayBlockingQueue是一个由数组支持的有界阻塞队列。创建时默认非公平锁，不过可在它的构造函数里指定,因为调用ReentrantLock的构造函数创建锁。
* SynchronousQueue同步队列没有任何内部容量。不能在同步队列上进行`peek()`，因为仅在试图要取得元素时，该元素才存在；除非另一个线程试图移除某个元素，否则也不能使用任何方法添加元素；也不能迭代队列，因为其中没有元素可用于迭代。
* LinkedBlockingQueue是一个基于已链接节点的、范围任意的blocking queue。队列的头部是在队列中时间最长的元素。队列的尾部是在队列中时间最短的元素。新元素插入到队列的尾部，队列检索操作会获得位于队列头部的元素。链接队列的吞吐量通常要高于基于数组的队列，但是在大多数并发应用程序中，其可预知的性能要低。

> LinkedBlcokingQueue和ArrayBlockingQueue用ReentrantLock，默认非公平锁，即阻塞式队列。其中LinkedBlockingQueue使用了2个lock，takelock和putlock，读和写用不同的lock来控制，这样并发效率更高。

``` java
/** Main lock guarding all access **/
    final ReentrantLock lock;

    public void put(E e) throws InterruptedException {
        checkNotNull(e);
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == items.length)
                notFull.await();
            insert(e);
        } finally {
            lock.unlock();
        }
    }

    private void insert(E x) {
        items[putIndex] = x;
        putIndex = inc(putIndex);
        ++count;
        notEmpty.signal();
    }

    final int inc(int i) {
        return (++i == items.length) ? 0 : i;
    }

    public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
            while (count == 0)
                notEmpty.await();
            return extract();
        } finally {
            lock.unlock();
        }
    }

    private E extract() {
        final Object[] items = this.items;
        E x = this.<E>cast(items[takeIndex]);
        items[takeIndex] = null;
        takeIndex = inc(takeIndex);
        --count;
        notFull.signal();
        return x;
    }

    final int dec(int i) {
        return ((i == 0) ? items.length : i) - 1;
    }

    @SuppressWarnings("unchecked")
    static <E> E cast(Object item) {
        return (E) item;
    }
```
