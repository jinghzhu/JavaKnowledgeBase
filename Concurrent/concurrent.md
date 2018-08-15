# <center>Concurrent</center>



<br></br>

如果分析concurrent包源码，会发现通用化实现模式：
* 首先，声明共享变量为volatile；
* 然后，使用CAS原子条件更新来实现线程间同步；
* 同时，配合以volatile读写和CAS具有的volatile读写内存语义实现线程通信。

<p align="center">
  <img src="./Images/overall.png" alt="concurrent包的实现示意图" width=500 />
</p>

<center><i>concurrent包实现示意图</i></center>

<br></br>



## 线程状态
----

![Thread Lifecycle](./Images/thread_lifecycle.jpg)

1. 可运行 runnable

2. 阻塞 block
    
    阻塞情况分三种： 
    
    1. 等待阻塞：线程执行`wait()`，JVM把该线程放入等待队列。
    2. 同步阻塞：线程获取对象同步锁时，若锁被别的线程占用，则JVM把线程放入锁池。
    3. 其他阻塞：线程执行`Thread.sleep(long ms)`或`t.join()`，或发出I/O请求时，JVM把该线程置为阻塞状态。当`sleep()`超时、`join()`等待线程终止或超时、或I/O处理完毕，线程重新转入可运行状态。

3. 死亡

`wait()`与`sleep()`区别：
* `wait()`是Object方法，而`sleep()`是Thread类静态方法。
* `sleep()`使线程阻塞指定时间，当前线程让出CPU时间，时间结束后继续执行。**该过程不释放线程持有对象锁。**
* `wait()`释放锁并进入等待队列。收到持有锁的其它线程`notify()`或`notifyAll()`后，`wait()`返回。

`run()`与`start()`区别：
* 通过调用`Thread`类的`start()`启动线程，这时线程处于就绪状态，没有运行，一旦得到时间片开始执行`run()`。
* `run()`只是类的一个普通方法。如果直接调用`run()`，程序中依然只有主线程这一个线程，没有达到写线程的目的。

<br></br>



## 常用数据结构原理
----
### AtomicInteger, AtomicBoolean & AtomicLong
基于**CAS**，是**乐观锁**。

<br>


### ConcurrentHashMap
JDK 1.8前基于**segment的lock**，JDK1.8是对node使用了`volatile`保证读的**happens-before**。在写数据时，如果是新的结点，使用**CAS**，其它则用**synchronized**。

<br>


### BlockingQuque
原理：
* ArrayBlockingQueue - 由数组支持的有界阻塞队列。默认非公平锁，因为调用ReentrantLock构造函数创建锁。
* SynchronousQueue - 同步队列没有任何内部容量。不能在同步队列上`peek()`；也不能迭代队列，因为其中没有元素可用于迭代。
* LinkedBlockingQueue - 基于已链接节点的、范围任意的阻赛队列。队列头部是在队列中时间最长的元素。尾部是在队列中时间最短的元素。新元素插入队列尾部，队列检索操作会获得头部元素。

> LinkedBlcokingQueue和ArrayBlockingQueue用ReentrantLock，默认非公平锁，即阻塞式队列。其中LinkedBlockingQueue使用了2个lock，takelock和putlock，读和写用不同的lock来控制，这样并发效率更高。

<br>


### ConcurrentLinkedQueue
**CAS**，即lock-free非阻塞算法。

> 一个lock-free程序能够确保执行它的所有线程中至少有一个能够继续往下执行。

<br>


### CopyOnWriteArrayList & CopyOnWriteArraySet
复制时用lock。

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
```

<br>


### AQS - Abstract Queued Synchronizer
AQS为实现依赖于FIFO等待队列的阻塞锁和同步器（信号量、事件等）提供一个框架。

设计目标是成为依靠单个原子int值表示状态的大多数同步器基础。子类须定义更改此状态受保护方法，并定义哪种状态对于此对象意味着被获取或被释放。假定这些条件后，此类中其他方法可实现所有排队和阻塞机制。子类可维护其他状态字段，但只为了获得同步而只追踪使用`getState()`、`setState(int)`和`compareAndSetState(int, int)`方法来操作以原子方式更新的int值。

队列节点为：
```java
static final class Node {
	static final int CANCELLED = 1;
	static final int SIGNAL = -1;
	static final int CONDITION = -2;
	static final int PROPAGATE = -3;

	volatile int waitStatus;
	volatile Node prev;
	volatile Node next;
	volatile Thread thread;

	Node nextWaiter;

	Node(Thread thread, Node mode) { // Used by addWaiter
		this.nextWaiter = mode;
		this.thread = thread;
	}

	Node(Thread thread, int waitStatus) { // Used by Condition
		this.waitStatus = waitStatus;
		this.thread = thread;
	}
}
```

对于首尾结点（即获取释放锁和阻赛线程）和结点status设置都是类似CAS语义。


<br>


### HashTable
**sychronized**

<br>


### ReentrantLock
ReentrantLock由最近成功获取锁，还没有释放的线程所拥有。当锁被另一个线程拥有时，调用`lock()`方法线程可成功获取锁。如果锁已被当前线程拥有，当前线程立即返回。
        
在AQS里有一个`state`字段，在ReentrantLock中表示锁被持有的次数，是`volatile`类型整型值。一个线程持有锁，`state = 1`。如果再次调用`lock()`，那么`state = 2`。当前可重入锁要完全释放，调用多少次`lock()`，还得调用等量的`unlock()`释放锁。

ReentrantLock**默认是nonfair**，当中的`lock()`是通过`static`内部类`sync`来进行锁操作：

``` java
public void lock() {
     sync.lock();
}

// 定义成final型的成员变量，在构造方法中进行初始化 
private final Sync sync;

// 无参数默认非公平锁
public ReentrantLock() {
    sync = new NonfairSync();
}

// 根据参数初始化为公平锁或者非公平锁 
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

<br>


### Synchronized
**是fair**。

<br></br>



## sleep(), join(), and yield()
----
### yield()
在Thread.java中`yield()`定义如下：
```java
/**
  * A hint to the scheduler that the current thread is willing to yield its current use of a processor. The scheduler is free to ignore this hint. Yield is a heuristic attempt to improve relative progression between threads that would otherwise over-utilize a CPU.
  * Its use should be combined with detailed profiling and benchmarking to ensure that it actually has the desired effect.
  */
public static native void yield();
```

* 是静态原生（native）方法。
* 告诉当前执行的线程把运行机会交给线程池中有相同优先级的线程。
* 仅使一个线程从运行状态转到可运行状态，而不是等待或阻塞状态。
* 与`sleep()`类似，但不能由用户指定暂停时间，且`yield()`方法只能让同优先级的线程有执行的机会。

<br>


### join()
可使一个线程在另一个线程结束后再执行。如果`join()`方法在一个线程实例上调用，当前运行着的线程将阻塞直到这个线程实例完成了执行。

在`join()`方法内设定超时，使得`join()`方法的影响在特定超时后无效。当超时时，主方法和任务线程申请运行的时候是平等的。然而，当涉及sleep时，`join()`方法依靠操作系统计时，所以不应假定`join()`方法会等待指定的时间。`sleep()`和`join()`通过抛出InterruptedException对中断做出回应。

<br>


### sleep()
使当前线程（即调用该方法的线程）暂停执行一段时间，让其他线程有机会执行，但不释放对象锁。注意该方法要捕捉异常。

例如有两个线程同时执行(没有synchronized)，一个线程优先级高过另一个。如果没有`sleep()`方法，只有高优先级线程执行完后，低优先级线程才能执行；但是高优先级的线程`sleep(500)`后，低优先级就有机会执行了。

总之，`sleep()`可使低优先级线程得到执行机会，当然也可以让同优先级、高优先级的线程有执行的机会。

<br></br>



## CPU密集型／IO密集型
----
首先考虑可用的处理器核心数：`Runtime.getRuntime().availableProcessors()`。

如果是CPU密集型，则创建处理器可用核心数这么多个线程就可以 。创建更多线程对于性能不利，因为多个线程间频繁上下文切换性能损耗大。

如果是IO密集型，需创建比处理器核心数大几倍的线程。

因此，线程数与任务处于阻塞状态时间比例相关。任务有50%时间处于阻塞状态，那程序所需线程数是处理器核心数两倍。计算程序所需的线程数公式如下：
_线程数=CPU可用核心数/（1 - 阻塞系数），阻塞系数在0到1内（CPU密集型阻塞系数为0，IO密集型程阻塞系数接近1）_

<br></br>




