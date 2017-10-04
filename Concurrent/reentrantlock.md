# <center>ReentrantLock</center>



<br></br>

* ReentrantLock是悲观锁。
* synchronized是可重入的，ReentrantLock不是。
* ReentrantLock实例化内部类Sync（fair或nofair），Sync类基于AQS。
* 公平锁和非公平锁释放时，最后都要写一个volatile变量`state`。
* 公平锁获取时，首先会去读这个volatile变量。
* 非公平锁获取时，首先会用CAS更新这个volatile变量（`compareAndSetState(int expect, int update)`），CAS操作同时具有volatile读和volatile写的内存语义。

![ReentrantLock类图](./Images/lock_uml.png)

<br></br>



## 使用场景
-----
* 场景1：如果发现该操作已经在执行中则不再执行（有状态执行）

    1. 用在定时任务，如果任务执行时间可能超过下次计划执行时间，确保该有状态任务只有一个在执行，忽略重复触发。
    2. 用在界面交互时点击执行较长时间请求操作时，防止多次点击导致后台重复执行（忽略重复触发）。

```java
private ReentrantLock lock = new ReentrantLock();
    if (lock.tryLock()) {  // 如果已被lock，则不会等待，达到忽略操作的效果 
        try {
            //操作
        } finally {
            lock.unlock();
        }
    }
}
```

* 场景2：如果发现该操作已在执行，等待一个一个执行（同步执行，类似synchronized）

    主要防止资源使用冲突，保证同一时间内只有一个操作可以使用该资源。但与synchronized区别是性能优势，同时Lock有更灵活的锁定方式，公平锁与不公平锁，而synchronized是公平的。

    这种情况主要用于对资源的争抢，如文件操作，同步消息发送和有状态的操作等。

* 场景3：如果发现该操作已经在执行，则尝试等待一段时间，超时则不执行。

    其实属于场景2的改进，等待获得锁的操作有一个时间的限制，如果超时则放弃执行。主要防止由于资源处理不当长时间占用导致死锁情况。

```java
try {
    if (lock.tryLock(5, TimeUnit.SECONDS)) {  
        try {
            //操作
        } finally {
            lock.unlock();
        }
    }
} catch (InterruptedException e) {
    e.printStackTrace(); //当前线程被中断时(interrupt)，会抛InterruptedException                 
}
```

* 场景4：如果发现该操作已经在执行，等待执行。这时可中断正在进行的操作立刻释放锁继续下一操作。

    synchronized与Lock默认情况是不会响应中断，会继续执行完。`lockInterruptibly()`提供了可中断锁来解决此问题。

```java
try {
    lock.lockInterruptibly();
} catch (InterruptedException e) {
    e.printStackTrace();
} finally {
    lock.unlock();
}
```

<br></br>



## 锁的可重入性
----
synchronized同步块是可重入的，这意味着如果一个线程进入synchronized同步块，并因此获得了该同步块使用的同步对象对应的管程上的锁，那么这个线程可以进入由同一个管程对象所同步的另一个代码块：

``` java
public class Reentrant{
    public synchronized outer(){
        inner();
    }

    public synchronized inner(){
        //do something
    }
}
```

`outer()`和`inner()`都被声明为synchronized，这和`synchronized(this)`块等效。如果一个线程调用`outer()`，在`outer()`里调用`inner()`就没有问题，因为这两个方法（代码块）都由同一个管程对象（`this`)同步。如果一个线程已拥有一个管程对象上的锁，那么它就有权访问被这个管程对象同步的所有代码块。这就是可重入。

下面的锁实现不是可重入的。当线程调用`outer()`时，会在`inner()`方法的`lock.lock()`处阻塞住。

``` java
public class Reentrant2{
    Lock lock = new Lock();

    public outer(){
        lock.lock();
        inner();
        lock.unlock();
    }

    public synchronized inner(){
        lock.lock();
        //do something
        lock.unlock();
    }
}
```

调用`outer()`的线程首先锁住Lock实例，然后继续调用`inner()`。`inner()`方法中该线程将再一次尝试锁住Lock实例，结果该动作会失败，因为这个Lock实例已在`outer()`方法中被锁住了。

两次`lock()`间没有调用`unlock()`，第二次调用lock就会阻塞。看过`lock()`实现后，会发现原因：

``` java
public class Lock{
    boolean isLocked = false;

    public synchronized void lock() throws InterruptedException{
        while(isLocked)
            wait();
        isLocked = true;
    }
}
```

一个线程是否被允许退出`lock()`方法是由while循环（自旋锁）中的条件决定。当前的判断条件是只有当`isLocked`为`false`时lock操作才被允许，而没有考虑是哪个线程锁住了它。

为了让这个Lock类具有可重入性，需做一点改动：

``` java
public class Lock{
    boolean isLocked = false;
    Thread  lockedBy = null;
    int lockedCount = 0;

    public synchronized void lock() throws InterruptedException{
        Thread callingThread = Thread.currentThread();
        while(isLocked && lockedBy != callingThread)
            wait();
        isLocked = true;
        lockedCount++;
        lockedBy = callingThread;
    }

    public synchronized void unlock(){
        if(Thread.curentThread() == this.lockedBy) {
            lockedCount--;
            if(lockedCount == 0) {
                isLocked = false;
                notify();
            }
        }
    }
}
```

现在的while循环考虑到了已锁住该Lock实例的线程。如果当前的锁对象没有被加锁(`isLocked = false`)，或当前调用线程已对该Lock实例加锁，那么while循环不会被执行，调用`lock()`的线程就可以退出该方法。

除此之外，需记录同一个线程重复对一个锁对象加锁的次数。否则，一次`unblock()`调用会解除整个锁，即使当前锁已经被加锁过多次。

现在这个Lock类是可重入的了。

<br></br>



## 抽象类Sync
----
ReentrantLock中有一个抽象类Sync：

``` java
private final Sync sync;

abstract static class Sync extends AbstractQueuedSynchronizer{
    ...
    }
```
ReentrantLock根据传入构造方法的布尔型参数实例化出Sync的实现类`FairSync`和`NonfairSync`，默认Nonfair。

<br></br>



## Nonfair获取锁 （ReentrantLock是悲观锁）
-----

```java
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();

            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            } else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```

流程：
1. 首先判断当前状态: 如果`c == 0`说明没有线程正在竞争该锁，如果不`c != 0`说明有线程拥有了锁。
2. 如果`c == 0`，通过CAS设置该状态值为`acquires`，`acquires`的初始调用值为1，每次线程重入该锁会+1，每次unlock会-1，为0时释放锁。如果CAS设置成功，则可以预计其他任何线程调用CAS都不会再成功，也就认为当前线程得到了该锁 。
3. 如果`c != 0`，但发现已拥有锁，只是简单地`++acquires`，并修改`status`值。但因为没有竞争，所以通过`setStatus`修改，而非CAS，也就是说实现了偏向锁的功能 。

<br></br>



## 用ReentrantLock解决死锁
----
synchronized产生死锁：
```java
public class Uninterruptible {
    public static void main(String[] args) throws InterruptedException {
        final Object o1 = new Object(), o2 = new Object();

        Thread t1 = new Thread() {
            public void run() {
                try {
                    synchronized(o1) {
                        Thread.sleep(1000);
                        synchronized(o2) {}
                    }
                } catch (InterruptedException e) {
                    System.out.println("t1 interrupted.");
                }
            }
        };

        Thread t2 = new Thread() {
            public void run() {
                try {
                    synchronized(o2) {
                        Thread.sleep(1000);
                        synchronized(o1) {}
                    } catch (InterruptedException e) {
                        System.out.println("t2 interrupted.");
                    }
                }
            }
        };

        t1.start(); t2.start();
        Thread.sleep(2000);
        t1.interrupt(); t2.interrupt();  // 无法中断死锁线程
    }
}
```

使用ReentrantLock的`lockInterruptibly()`终止死锁线程：

```java
public class Interruptible {
    public static void main(String[] args) throws InterruptedException {
        final ReentrantLock l1 = new ReentrantLock();
        final ReentrantLock l2 = new ReentrantLock();

        Thread t1 = new Thread() {
            public void run() {
                try {
                    l1.lockInterruptibly(); // 获取可中断的锁l1且为t1互斥访问加锁
                    // 访问t1互斥资源
                    Thread.sleep(1000);
                    l2.lockInterruptibly(); // 获取可中断的锁l2
                    // 访问t2互斥资源
                } catch (InterruptedException e) {
                    System.out.println("t1 interrupted.");
                }
            }
        };

        Thread t2 = new Thread() {
            public void run() {
                try {
                    synchronized(o2) {
                        l2.lockInterruptibly();
                        Thread.sleep(1000);
                        l1.lockInterruptibly();
                    } catch (InterruptedException e) {
                        System.out.println("t2 interrupted.");
                    }
                }
            }
        };

        t1.start(); t2.start();
        Thread.sleep(2000);
        t1.interrupt(); t2.interrupt();  // 可以中断线程
    }
}
```

<br></br>



## 交替锁
----
链表需要插入一个节点，一种做法是锁整个链表，效率太低。交替锁只锁住链表一部分。在链表中交替加锁的过程为不断的加锁和解锁，直到找到要插入的位置对两边的节点加锁。

<p align="center">
  <img src="./Images/alternate_lock.png" alt="交替锁"/>
</p>

<br>

内置锁无法完成这种效果，可用ReentrantLock在需要的地方使用`lock()`和`unlock()`即可。交替锁的有序链表实现如下：

```java
class ConcurrentSortedList {
    private class Node {
        int value;
        Node prev, next;
        ReentrantLock lock = new ReentrantLock();

        Node(int value, Node prev, Node next) {
            this.value =value; this.prev = prev; this.next = next;
        }
    }

    private final Node head;
    private final Node tail;

    public ConcurrentSortedList() {
        head = new Node(); tail = new Node();
        head.next = tail; tail.prev = head;
    }

    public void insert(int value) {
        Node current = head;
        current.lock.lock();
        Node next = current.next;
        try {
            while (true) {
                next.lock.lock();
                try{
                    // 找到插入位置则插入并且跳出while
                    // 否则获取锁判断，再在finally中释放锁
                    if (next == tail || next.value < value) {
                        Node node = new Node(value, current, next);
                        next.prev = node;
                        current.next = node;

                        return;
                    }
                } finally {
                    current.lock.lock();
                }
                current = next;
                next = current.next;
            }
            finally {
                next.lock.unlock();  // 未找到插入位置，则释放锁
            }
        }
    }
}
```

<br></br>



## 公平锁分析（`tryAcquire()` & `tryRelease()`）
----
使用公平锁时，加锁方法`lock()`调用轨迹如下：
1. ReentrantLock : `lock()`
2. FairSync : `lock()`
3. AbstractQueuedSynchronizer : `acquire(int arg)`
4. ReentrantLock : `tryAcquire(int acquires)`

在第4步真正开始加锁：
``` java
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();   //获取锁的开始，首先读volatile变量state

    if (c == 0) {
        if (isFirst(current) && compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    } else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)  
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }

    return false;
}
```

加锁方法先读volatile变量`state`。

解锁方法`unlock()`调用轨迹如下：
1. ReentrantLock : `unlock()`
2. AbstractQueuedSynchronizer : `release(int arg)`
3. Sync : `tryRelease(int releases)`

在第3步真正开始释放锁：
``` java
        protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);

            return free;
        }
```

`tryRelease()`与`tryAcquire()`语义相同，把如何释放的逻辑延迟到子类中。`tryRelease()`语义很明确，如果线程多次锁定，则进行多次释放，直至`status == 0`真正释放锁，所谓释放锁即设置`status`为0，因为无竞争所以没有使用CAS。

公平锁在释放锁的最后写volatile变量`state`，在获取锁时首先读这个volatile变量。

<br></br>




