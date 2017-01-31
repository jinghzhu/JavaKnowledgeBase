# <center>Concurrent</center>

![concurrent包的实现示意图](./Images/overall.png)



## 1. 线程的五个状态wait和sleep区别：
![Thread Lifecycle](./Images/thread_lifecycle.jpg)
1. 新建(new)：新创建了一个线程对象。
2. 可运行(runnable)：线程对象创建后，其他线程调用了该对象的`start()`，该状态的线程位于可运行线程池中，等待被线程调度选中，获取cpu使用权 。
3. 运行(running)：可运行状态(runnable)的线程获得了cpu时间片（timeslice），执行程序。
4. 阻塞(block)：线程因为某种原因放弃了cpu使用权，暂时停止运行，阻塞的情况分三种： 
    1. 等待阻塞：运行的线程执行`wait()`，JVM把该线程放入等待队列(waitting queue)中。
    2. 同步阻塞：运行的线程在获取对象同步锁时，若该同步锁被别的线程占用，则JVM会把该线程放入锁池(lock pool)中。
    3. 其他阻塞：运行的线程执行`Thread.sleep(long ms)`或`t.join()`，或者发出I/O请求时，JVM会把该线程置为阻塞状态。当`sleep()`状态超时、`join()`等待线程终止或者超时、或者I/O处理完毕时，线程重新转入可运行(runnable)状态。
5. 死亡(dead)：线程`run()`、`main() `方法执行结束，或者因异常退出了`run()`，则该线程结束生命周期。

&#12288;&#12288;另外，关于`wait()`与`sleep()`区别：
* `wait()是`Object`方法，而`sleep()`是`Thread`类的静态方法
*  `sleep()`使线程阻塞指定时间，这段时间当前线程让出CPU时间，时间结束后继续执行，该过程不释放线程持有的对象锁；`wait()`调用后线程释放持有的锁并进入该锁等待队列，当收到持有锁的其它线程`notify()`或`notifyAll()`信号后，`wait()`方法返回。



## 2. 常用数据结构原理
### 2.1 AtomicInteger
&#12288;&#12288;基于*CAS*。

### 2.2 ConcurrentHashMap
&#12288;&#12288;JDK 1.8之前基于segment的lock，JDK1.8是对node使用了`volatile`保证读的*happens-before*。在写数据时，如果是新的结点，使用*CAS*，其它则用*synchronized*。

### 2.3 BlockingQuque
&#12288;&#12288;`LinkedBlcokingQueue`和`ArrayBlockingQueue`用*ReentrantLock*，默认非公平锁，即阻塞式队列。其中`LinkedBlockingQueue`使用了2个lock，takelock和putlock，读和写用不同的lock来控制，这样并发效率更高。

```java
/** Main lock guarding all access */
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

### 2.4 ConcurrentLinkedQueue
&#12288;&#12288;使用*CAS*，即”lock-free”的非阻塞式算法。

### 2.5 CopyOnWriteArrayList & CopyOnWriteArraySet
&#12288;&#12288;复制时候用lock。

```java
public boolean add(T e) {
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        // copy data to new array
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        // add new data into new array
        newElements[len] = e;
        // change old array reference
        setArray(newElements);

        return true;
    } finally {
        lock.unlock();
    }
}

final void setArray(Object[] a) {
    array = a;
}
```

### 2.6 AbstractQueuedSynchronizer
&#12288;&#12288;AQS为实现依赖于先进先出 (FIFO) 等待队列的阻塞锁和相关同步器（信号量、事件，等等）提供一个框架。设计目标是成为依靠单个原子`int`值来表示状态的大多数同步器的一个基础。子类必须定义更改此状态的受保护方法，并定义哪种状态对于此对象意味着被获取或被释放。假定这些条件之后，此类中的其他方法就可以实现所有排队和阻塞机制。

&#12288;&#12288;子类可以维护其他状态字段，但只是为了获得同步而只追踪使用`getState()`、`setState(int)`和`compareAndSetState(int, int)`来操作以原子方式更新的`int`值。

&#12288;&#12288;以下是一个非再进入的互斥锁类，它使用值_0_表示未锁定状态，_1_表示锁定状态。还支持一些条件并公开了一个检测方法：

```java
class Mutex implements Lock, java.io.Serializable {
    // Our internal helper class
    private static class Sync extends AbstractQueuedSynchronizer {
      // Report whether in locked state
      protected boolean isHeldExclusively() { 
        return getState() == 1; 
      }

      // Acquire the lock if state is zero
      public boolean tryAcquire(int acquires) {
        assert acquires == 1; // Otherwise unused
       if (compareAndSetState(0, 1)) {
         setExclusiveOwnerThread(Thread.currentThread());
         return true;
       }
       return false;
      }

      // Release the lock by setting state to zero
      protected boolean tryRelease(int releases) {
        assert releases == 1; // Otherwise unused
        if (getState() == 0) throw new IllegalMonitorStateException();
        setExclusiveOwnerThread(null);
        setState(0);
        return true;
      }
       
      // Provide a Condition
      Condition newCondition() { return new ConditionObject(); }

      // Deserialize properly
      private void readObject(ObjectInputStream s) 
        throws IOException, ClassNotFoundException {
        s.defaultReadObject();
        setState(0); // reset to unlocked state
      }
    }

    // The sync object does all the hard work. We just forward to it.
    private final Sync sync = new Sync();

    public void lock()                { sync.acquire(1); }
    public boolean tryLock()          { return sync.tryAcquire(1); }
    public void unlock()              { sync.release(1); }
    public Condition newCondition()   { return sync.newCondition(); }
    public boolean isLocked()         { return sync.isHeldExclusively(); }
    public boolean hasQueuedThreads() { return sync.hasQueuedThreads(); }
    public void lockInterruptibly() throws InterruptedException { 
      sync.acquireInterruptibly(1);
    }
    public boolean tryLock(long timeout, TimeUnit unit) 
        throws InterruptedException {
      return sync.tryAcquireNanos(1, unit.toNanos(timeout));
    }
 }
```

### 2.7 HashTable
&#12288;&#12288;*sychronized*



## 3. 非阻塞算法
&#12288;&#12288;基于锁的算法会带来一些活跃度失败的风险。如果线程在持有锁的时候因为阻塞I/O，页面错误，或其他原因发生延迟，很可能所有的线程都不能前进了。 

&#12288;&#12288;一个线程的失败或挂起不应该影响其他线程的失败或挂起，这样的算法为非阻塞（nonblocking）算法；如果算法的每一个步骤中都有一些线程能够继续执行，那么这样的算法称为锁自由（lock-free）算法。在线程间使用CAS进行协调，这样的算法如果能构建正确的话，它既是非阻塞的，又是锁自由的。非竞争的CAS总是能够成功，如果多个线程以一个CAS竞争，总会有一个胜出并前进。

&#12288;&#12288;非阻塞算法通过使用低层次的并发原语，比如比较交换，取代了锁。 


## 4. 可重入锁ReentrantLock
&#12288;&#12288;ReentrantLock由最近成功获取锁，还没有释放的线程所拥有，当锁被另一个线程拥有时，调用`lock()`方法的线程可以成功获取锁。如果锁已经被当前线程拥有，当前线程会立即返回。
        
&#12288;&#12288;在AQS里面有一个`state`字段，在ReentrantLock中表示锁被持有的次数，它是一个`volatile`类型的整型值。一个线程持有锁，`state = 1`,如果它再次调用`lock()`，那么`state = 2`.当前可重入锁要完全释放，调用了多少次`lock()`，还得调用等量的`unlock()`来释放锁。

&#12288;&#12288;ReentrantLock默认是nonfair，Synchronizer是fair. ReentrantLock当中的`lock()`是通过`static`内部类`sync`来进行锁操作：

```java
public void lock() {
     sync.lock();
}
//定义成final型的成员变量，在构造方法中进行初始化 
private final Sync sync;
//无参数默认非公平锁
public ReentrantLock() {
    sync = new NonfairSync();
}
//根据参数初始化为公平锁或者非公平锁 
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```


## 5. 如何让一段程序并发执行，并最终汇总结果？
* CountDownLatch：允许一个或者多个线程等待前面的一个或多个线程完成，构造一个CountDownLatch时指定需要CountDown的点的数量，每完成一点就count down一下，当所有点都完成，latch.wait就解除阻塞。
* CyclicBarrier：可循环使用的Barrier，它的作用是让一组线程到达一个Barrier后阻塞，直到所有线程都到达 Barrier后才能继续执行。CountDownLatch的计数值只能使用一次，CyclicBarrier可以通过使用reset重置；还可以指定 到达栅栏后优先执行的任务。
fork/join框架，fork把大任务分解成多个小任务，然后汇总小任务的结果得到最终结果。使用一个双端队列，当线程空闲时从双端队列的另一端领取任务。


## 6. 为什么wait(), notify()和notifyAll()必须在同步方法或同步块中被调用？
&#12288;&#12288;当一个线程需要调用对象的`wait()`时，这个线程必须拥有该对象的锁，接着它就会释放这个对象锁并进入等待状态直到其他线程调用这个对象上的`notify()`。

&#12288;&#12288;同样的，当一个线程需要调用对象的`notify()`时，它会释放这个对象的锁，以便其他在等待的线程就可以得到这个对象锁。由 于所有的这些方法都需要线程持有对象的锁，就只能通过同步来实现，所以他们只能在同步方法或者同步块中被调用。



## 7. AtomicInteger
### 7.1 Example
```java
class Counter {
    private volatile int count = 0;
    public synchronized void increment() {
        count++;  //若要线程安全执行执行count++，需要加锁
    }
}

class Counter {
    private AtomicInteger count = new AtomicInteger(); 
    public void increment() {//使用AtomicInteger之后，不加锁也可实现线程安全
        count.incrementAndGet();
    }
       
}
```

### 7.2 java提供的原子操作可以原子更新的基本类型有以下三个
1. AtomicBoolean
2. AtomicInteger
3. AtomicLong

### 7.3 java提供的原子操作，还可以原子更新以下类型的值
1. 原子更新数组，`Atomic`包提供了以下几个类：`AtomicIntegerArray`、`AtomicLongArray`、`AtomicReferenceArray`。
2. 原子更新引用类型，也就是更新实体类的值，比如`AtomicReference`（原子更新引用类型的值）; `AtomicReferenceFieldUpdater`（原子更新引用类型里的字）; `AtomicMarkableReference`（原子更新带有标记位的引用类型）。

### 7.4 是乐观锁

### 7.5 原理是基于CAS
```java
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


## 8. CopyOnWriteArrayList适用场景？
&#12288;&#12288;`CopyOnWriteArrayList`（写时复制）是一个线程安全的容器，它的线程安全性在于对容器的修改可以和读操作同时进行。从容器中读时不需要加锁，对容器中的元素进行修改时，先复制一份所有元素的副本，然后在新的副本上进行操作。 
&#12288;&#12288;`CopyOnWriteArrayList`适用于读操作多于写操作的场景。例如，缓存。


## 9. Java多线程内存模型Memory Model
&#12288;&#12288;JVM中存在一个主内存（Main Memory），Java中所有的变量存储在主内存中，所有实例和实例的字段都在此区域，对于所有的线程是共享的（相当于黑板，其他人都可以看到的）。

&#12288;&#12288;每个线程都有自己的工作内存（Working Memory），工作内存中保存的是主存中变量的拷贝，（相当于自己笔记本，只能自己看到），工作内存由缓存和堆栈组成，其中缓存保存的是主存中的变量的copy，堆栈保存的是线程局部变量。

&#12288;&#12288;线程对所有变量的操作都是在工作内存中进行的，线程之间无法直接互相访问工作内存，变量的值得变化的传递需要主存来完成。在JMM中通过并发线程修改的变量值，必须通过线程变量同步到主存后，其他线程才能访问到。

![Concurrent Memory Model](./Images/concurrent_memory_model.png)

&#12288;&#12288;线程对某个变量的操作步骤： 
1. 从主内存中复制数据到工作内存；
2. 执行代码，对数据进行各种操作和计算；
3. 把操作后的变量值重新写回主内存中。

&#12288;&#12288;这三个步骤顺序是我们希望的，但JVM不保证第1步和第3步会严格按照上述次序立即执行。由于JVM可以对特征代码进行调优，也就改变了某些运行步骤的次序的颠倒，那么每次线程调用变量时是直接取自己的工作存储器中的值还是先从主存储器复制再取是没有保证的，任何一种情况都可能发生。同样的，线程改变变量的值之后，是否马上写回到主存储器上也是不可保证的 。

&#12288;&#12288;在多线程的应用场景下同时访问同一个代码块，有可能某个线程已经改变了某变量值，当然现在的改变仅仅是局限于工作内存中的改变，此时JVM并不能保证将改变后的值立马写到主内存中去，也就意味着有可能其他线程不能立马得到改变后的值，依然在旧的变量上进行各种操作和运算，最终导致不可预料的结果。 

&#12288;&#12288;还好有`synchronized`和`volatile`： 
* 多个线程共有的字段应该用`synchronized`或`volatile`来保护. 
* `synchronized`负责线程间互斥。即同时只有一个线程可以执行`synchronized`中的代码. 

&#12288;&#12288;`synchronized`还有另外的作用：
* 在线程进入`synchronized`块之前，会把工作存内存中的所有内容映射到主内存上，然后把工作内存清空再从主存储器上拷贝最新的值。
* 线程退出`synchronized`块时，会把工作内存中的值映射到主内存，此时并不会清空工作内存。强制其按照上面的顺序运行，以保证线程在执行完代码块后，工作内存中的值和主内存中的值是一致的，保证了数据的一致性。 
* `volatile`负责线程中的变量与主存储区同步.但不负责每个线程之间的同步. 


## 10. 确定CPU密集型／IO密集型线程数 Multiple CPU/IO Threads Work
&#12288;&#12288;首先要考虑系统可用的处理器核心数：`Runtime.getRuntime().availableProcessors()`。应用程序最小线程数等于可用的处理器核数。

&#12288;&#12288;如果所有的任务都是计算密集型的，则创建处理器可用核心数这么多个线程就可以了 。创建更多的线程对于程序性能反而是不利的，因为多个线程间频繁进行上下文切换对于程序性能损耗较大。

&#12288;&#12288;如果任务都是IO密集型的，就需要创建比处理器核心数大几倍的线程。当一个任务执行IO操作时，线程将被阻塞，处理器可以立即进行上下文切换以便处理其他就绪线程。如果只有处理器核心数那么多个线程，即使有待执行的任务也无法调度处理。

&#12288;&#12288;因此，线程数与任务处于阻塞状态的时间比例相关。任务有50%时间处于阻塞状态，那程序所需线程数是处理器核心数的两倍。计算程序所需的线程数公式如下：
线程数=CPU可用核心数/（1 - 阻塞系数），阻塞系数在0到1内（CPU密集型阻塞系数为0，IO密集型程阻塞系数接近1） 

You really can't improve on having one thread reading the file sequentially. With one thread, you do one seek and then one long sequential read; with multiple threads you're going to have the threads causing multiple seeks as each gains control of the disk head.

This is a way to parallelize the line processing while still using serial I/O to read the lines. It uses a BlockingQueue (Reading big file from disk like producer/consumer pattern) to communicate between threads; the FileTask adds lines to the queue, and the CPUTask reads them and processes them. You're using put(E e) to add strings to the queue, so if the queue is full, the FileTask blocks until space frees up; likewise you're using take() to remove items from the queue, so the CPUTask will block until an item is available.

```java
public class ReadingFile {
    public static void main(String[] args) {
        final int threadCount = 10;
        BlockingQueue<String> queue = new ArrayBlockingQueue<>(200);

        // create thread pool with given size
        ExecutorService service = Executors.newFixedThreadPool(threadCount);

        for (int i = 0; i < (threadCount - 1); i++) {
            service.submit(new CPUTask(queue));
        }

        // Wait til FileTask completes
        service.submit(new FileTask(queue)).get();

        service.shutdownNow();  // interrupt CPUTasks

        // Wait til CPUTasks terminate
        service.awaitTermination(365, TimeUnit.DAYS);
    }
}

class FileTask implements Runnable {
    private final BlockingQueue<String> queue;

    public FileTask(BlockingQueue<String> queue) {
        this.queue = queue;
    }

    @Override
    public void run() {
        BufferedReader br = null;
        try {
            br = new BufferedReader(new FileReader("D:/abc.txt"));
            String line;
            while ((line = br.readLine()) != null) {
                // block if the queue is full
                queue.put(line);
            }
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                br.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
}

class CPUTask implements Runnable {
    private final BlockingQueue<String> queue;

    public CPUTask(BlockingQueue<String> queue) {
        this.queue = queue;
    }

    @Override
    public void run() {
        String line;
        while(true) {
            try {
                // block if the queue is empty
                line = queue.take(); 
                // do things with line
            } catch (InterruptedException ex) {
                break; // FileTask has completed
            }
        }
        // poll() returns null if the queue is empty
        while((line = queue.poll()) != null) {
            // do things with line;
        }
    }
}
```


## 11. volatile作用
&#12288;&#12288;`volatile`解决变量的可见性问题。如果一个变量被声明为`volatile`，那么编译器和JVM就会把该变量当做共享变量从而禁止对该变量的一些重排序操作。修改`volatile`变量后立刻对所有线程可见 。

&#12288;&#12288;`volatile`无法解决原子性问题和一致性问题。可以用于确保自身状态的可见性，例如检查一个`volatile`变量以确定是否退出循环，如果该变量被其它变量修改，那么这个线程可以马上检查到。使用 `volatile`可以简化多线程编程，例如`ConcurrentHashMap`中`HashEntry`中的`val`域和`next`通过使 用`volatile`减少锁的获取和释放引起的损失。


## 12. 什么是daemon线程
A daemon thread runs in background and doesn’t prevent JVM from terminating. When there are no user threads running, JVM shutdown the program and quits.

&#12288;&#12288;必须在Thread启动前调用`setDaemon()`将线程设置为Daemon线程：
* Daemon线程创建的线程也是Daemon线程
* Daemon不应该访问数据库、文件等资源，因为它随时有可能被中断（后台进程在不执行finally语句前就可以中断其`run()`）


## 13. 针对外星方法的保护性锁
&#12288;&#12288;例: 构造一个类从一个URL进行下载，并用`ProgressListeners`监听下载的进度：
```java
class Downloader extends Thread {
    private InputStream in;
    private OutputStream out;
    private ArrayList<ProgressListener> listeners;

    public Downloader(URL url, String outputFilename) throws IOException {
        in = url.openConnection().getInputStream();
        out = new FileOutputStream(outputFilename);
        listeners = new ArrayList<ProgressListener>();
    }

    public synchronized void addListener(ProgressListener listeners) {
        listeners.add(listener);
    }

    public synchronized void removeListener(ProgressListener listener) {
        listeners.remove(listener);
    }

    private synchronized void updateProgress(int n) {
        for (ProgressListener listener: listeners)
            listener.onProgress(n);   // 这里调用了一个外星方法
    }

    public void run() {
        int n = 0, total = 0;
        byte[] buffer = new byte[1024];

        try {
            while ((n = in.read(buffer)) != -1) {
                out.write(buffer, 0, n);
                total += n;
                updateProgress(total);
            }
            out.flush();
        } catach (IOException e) {}
    }
}
```

* 问题：调用了`onProgress()`这个外星方法，可能引入其他锁导致死锁。
* 解决方案：采用保护性复制（defensive copy），即不对原始对象进行操作，而是对克隆出来的对象进行操作。
* 注意：保护性复制是个好方法。前提是进行只读的操作，不做修改。

```java
private void updatePrgress(int n) {
    ArrayList<ProgressListener> listernersCopy;
    synchronized(this) {
        listernersCopy = (ArrayList<ProgressListener>)listeners.clone();
    }
    for (ProgressListener listener : listernersCopy)
        listener.onProgress(n);
}
```



## 14. BlockingQueue
BlockingQueue doesn’t accept `null` values and throw `NullPointerException` if store `null` value in the queue.

&#12288;&#12288;BlockingQueue是接口，使用它应类似`BlockingQueue q = new LinkedBlockingQueue(5);`常用的阻塞队列有:
1. ArrayBlockingQueue：由数组支持的有界阻塞队列，其构造函数必须带一个参数来指明其大小.其所含的对象是以FIFO顺序排序的。之所以采用阻塞队列，是因为如果生产者的速度比消费者快很多，很容易就占用大量内存空间.
2. LinkedBlockingQueue：其构造函数可带也可不带规定大小的参数。 若不带大小参数,所生成的大小由`Integer.MAX_VALUE`决定.所含对象以FIFO排序。
3. PriorityBlockingQueue：类似于LinkedBlockQueue,但其所含对象的排序是依据对象的自然排序顺序或者是构造函数的Comparator决定的顺序。
4. SynchronousQueue：对其的操作必须是放和取交替完成的。 

&#12288;&#12288;其入队出队方法：
1. `add(Object)`: 如果BlockingQueue可以容纳,则返回`true`,否则异常
2. `offer(Object)`: 如果BlockingQueue可以容纳,则返回`true`,否则返回`false`.
3. `put(Object)`: 如果BlockQueue没有空间,则调用此方法的线程被阻断直到BlockingQueue里面有空间再继续.
4. `poll(time)`:取走排在首位的对象,若不能立即取出,则可以等time参数规定的时间,取不到时返回`null`。
5. `take()`:取走排在首位的对象,若BlockingQueue为空,阻断进入等待状态直到有新的对象被加入为止

&#12288;&#12288;原理：
* ArrayBlockingQueue是一个由数组支持的有界阻塞队列。 创建时默认非公平锁，不过可在它的构造函数里指定,因为调用ReentrantLock的构造函数创建锁。
* SynchronousQueue同步队列没有任何内部容量。不能在同步队列上进行`peek()`，因为仅在试图要取得元素时，该元素才存在；除非另一个线程试图移除某个元素，否则也不能使用任何方法添加元素；也不能迭代队列，因为其中没有元素可用于迭代。
* LinkedBlockingQueue是一个基于已链接节点的、范围任意的blocking queue。队列的头部是在队列中时间最长的元素。队列的尾部是在队列中时间最短的元素。新元素插入到队列的尾部，队列检索操作会获得位于队列头部的元素。链接队列的吞吐量通常要高于基于数组的队列，但是在大多数并发应用程序中，其可预知的性能要低。



## 15. ThreadLocal?
ThreadLocal is used to create thread-local variables. We know that all threads of an `Object` share its variables, so if the variable is not thread safe, we can use synchronization but if we want to avoid synchronization, we can use ThreadLocal variables.

Every thread has its own ThreadLocal variable and they can use its `get()` and `set() to get the default value or change its value local to Thread. ThreadLocal instances are typically private static fields in classes that wish to associate state with a thread.
        
&#12288;&#12288;ThreadLocal是一种以空间换时间的做法，在每个Thread里面维护了一个以开地址法实现的`ThreadLocal.ThreadLocalMap`，把数据进行隔离，数据不共享。



## 16. run() vs start()
* 用`start()`启动无需等待`run()`执行完毕而直接继续执行下面的代码。通过调用`Thread`类的`start()`启动线程，这时此线程处于就绪状态，并没有运行，一旦得到时间片就开始执行`run()`。
* `run()`只是类的一个普通方法，如果直接调用`run()`，程序中依然只有主线程这一个线程，其程序执行路径还是只有一条，还是要顺序执行，还是要等待`run()`方法体执行完毕后才可继续执行下面的代码，这样就没有达到写线程的目的。



## 17. Thread.sleep(0)的作用
&#12288;&#12288;由于Java采用抢占式的线程调度算法，因此可能会出现某条线程常常获取到CPU控制权的情况，为了让某些优先级较低的线程也能获取CPU控制权，可使用`Thread.sleep(0)`手动触发一次操作系统分配时间片的操作 。



## 18. What will happen if we don’t override Thread class run() method?
Thread class `run()` method code is as shown below:
```java
public void run() {
    if (target != null) {
        target.run();
    }
}
```
Above target set in the `init()` method of Thread class and if we create an instance of Thread class as new `TestThread()`, it’s set to `null`. So nothing will happen.



## 19. Semaphore
### 19.1 Semaphore的实现

```java
public class Semaphore {
    private boolean signal = false;   //使用signal可以避免信号丢失
    public synchronized void take() {
       this.signal = true;
       this.notify();
    }
    public synchronized void release() throws InterruptedException{
       while(!this.signal) //使用while避免假唤醒
           wait();
        this.signal = false;
    }
}
```


### 19.2 可计数的Semaphore

```java
public class CountingSemaphore {
    private int signals = 0;
    public synchronized void take() {
        this.signals++;
        this.notify();
    }

public synchronized void release() throws InterruptedException{
    while(this.signals == 0) 
        wait();
    this.signals--;
    }
}
```


### 19.3 有上限的Semaphone

```java
public class BoundedSemaphore {
    private int signals = 0;
    private int bound   = 0;
    public BoundedSemaphore(int upperBound){
        this.bound = upperBound;
    }
    public synchronized void take() throws InterruptedException{
        while(this.signals == bound) 
            wait();
        this.signals++;
        this.notify();
    }
    public synchronized void release() throws InterruptedException{
        while(this.signals == 0) 
            wait();
        this.signals--;
        this.notify();
    }
}
```


### 19.4 互斥量Mutex

```java
public class Mutex {
    private boolean isLocked = false;
    public synchronized void lock() {
        while(this.isLocked) //使用while可以避免线程 假唤醒
            wait();
        this.isLocked= true;
    }
}

public synchronized void unlock() throws InterruptedException{
    this.isLocked= false;
    this.notify();
    }
}
```



## 20. 线程类的构造方法、静态块是被哪个线程调用的
&#12288;&#12288;线程类的构造方法、静态块是被new这个线程类所在的线程所调用的，而`run()`里面的代码才是被线程自身所调用的。

&#12288;&#12288;举个例子，假设`Thread2`中new了`Thread1`，`main()`中new了`Thread2`，那么：
* `Thread2`的构造方法、静态块是`main`线程调用的，`Thread2`的`run()`是`Thread2`自己调用的。
* `Thread1`的构造方法、静态块是`Thread2`调用的，`Thread1`的`run()`是`Thread1`自己调用的。



## 21. 什么是线程安全
&#12288;&#12288;如果代码在多线程和在单线程下执行永远获得一样的结果，那么就是线程安全。线程安全也有几个级别：
* 不可变：像`String`、`Integer`、`Long`都是`final`类型的类，任何一个线程都改变不了它们的值，因此这些不可变对象不需要任何同步手段就可以直接在多线程环境下使用。
* 绝对线程安全：不管运行时环境如何，调用者都不需要额外的同步措施。要做到这一点需付出额外代价，比方说`CopyOnWriteArrayList`、`CopyOnWriteArraySet`。
* 相对线程安全：就是通常意义上所说的线程安全，像`Vector`，`add()`、`remove()`都是原子操作。如果有个线程在遍历某个`Vector`、有个线程同时在add这个`Vector`，99%的情况下都会出现`ConcurrentModificationException`，也就是fail-fast机制。



## 22. Java如何获取线程dump文件
&#12288;&#12288;线程dump就是线程堆栈，获取到线程堆栈有两步：
1. 获取到线程的`pid`, 在Linux环境下还可以使用ps -ef | grep java
2. 打印线程堆栈, 在Linux环境下还可以使用kill -3 pid

&#12288;&#12288;Thread类提供了`getStackTrace()`用于获取线程堆栈。这是一个实例方法，因此此方法是和具体线程实例绑定的，每次获取获取到的是具体某个线程当前运行的堆栈。















