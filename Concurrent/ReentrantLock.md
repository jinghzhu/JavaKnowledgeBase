# <center>ReentrantLock</center>

![ReentrantLock类图](./Images/lock_uml.png)



## 1. ReentrantLock中有一个抽象类Sync
``` java
private final Sync sync;

abstract static class Sync extends AbstractQueuedSynchronizer{
    ...
    }
```
&#12288;&#12288;ReentrantLock根据传入构造方法的布尔型参数实例化出Sync的实现类`FairSync`和`NonfairSync`，默认Nonfair。

<br></br>



## 2. Nonfair获取锁 （ReentrantLock是悲观锁）
```java
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```

&#12288;&#12288;流程：
1. 首先判断当前状态: 如果`c == 0`说明没有线程正在竞争该锁，如果不`c != 0`说明有线程拥有了锁。
2. 如果`c == 0`，通过CAS设置该状态值为`acquires`, `acquires`的初始调用值为1，每次线程重入该锁都会+1，每次unlock都会-1，但为0时释放锁。如果CAS设置成功，则可以预计其他任何线程调用CAS都不会再成功，也就认为当前线程得到了该锁 。
3. 如果`c != 0`，但发现已拥有锁，只是简单地`++acquires`，并修改`status`值。但因为没有竞争，所以通过`setStatus`修改，而非CAS，也就是说这段代码实现了偏向锁的功能 。

<br></br>



## 3. synchronized造成死锁并用ReentrantLock解决
&#12288;&#12288;synchronized产生死锁：

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

&#12288;&#12288;使用ReentrantLock的`lockInterruptibly()`终止死锁线程：

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



## 4. 交替锁
&#12288;&#12288;链表需要插入一个节点，一种做法是锁整个链表，显然效率太低。交替锁可以只锁住链表的一部分。在链表中交替加锁的过程如下，即不断的加锁和解锁，直到找到要插入的位置对两边的节点加锁。

<p align="center">
  <img src="./Images/alternate_lock.png" alt="交替锁"/>
</p>

&#12288;&#12288;内置锁无法完成这种效果，可用ReentrantLock在需要的地方使用`lock()和`unlock()`即可。交替锁的有序链表实现如下：

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



## 5. 公平锁分析（`tryAcquire` & `tryRelease`）
&#12288;&#12288;使用公平锁时，加锁方法`lock()`调用轨迹如下：
1. ReentrantLock : `lock()`
2. FairSync : `lock()`
3. AbstractQueuedSynchronizer : `acquire(int arg)`
4. ReentrantLock : `tryAcquire(int acquires)`

&#12288;&#12288;在第4步真正开始加锁：

``` java
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();   //获取锁的开始，首先读volatile变量state

    if (c == 0) {
        if (isFirst(current) &&
            compareAndSetState(0, acquires)) {
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

&#12288;&#12288;可以看出，加锁方法首先读volatile变量`state`。解锁方法`unlock()`调用轨迹如下：
1. ReentrantLock : `unlock()`
2. AbstractQueuedSynchronizer : `release(int arg)`
3. Sync : `tryRelease(int releases)`

&#12288;&#12288;在第3步真正开始释放锁，下面是该方法的源代码：

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

&#12288;&#12288;`tryRelease`与`tryAcquire`语义相同，把如何释放的逻辑延迟到子类中。`tryRelease`语义很明确：如果线程多次锁定，则进行多次释放，直至`status == 0`则真正释放锁，所谓释放锁即设置`status`为0，因为无竞争所以没有使用CAS。

&#12288;&#12288;公平锁在释放锁的最后写volatile变量`state`；在获取锁时首先读这个volatile变量。根据volatile的happens-before规则，释放锁的线程在写volatile变量之前可见的共享变量，在获取锁的线程读取同一个volatile变量后将立即变的对获取锁的线程可见。

<br></br>



## 6. 非公平锁分析
&#12288;&#12288;非公平锁的释放和公平锁一样，所以仅分析非公平锁的获取。加锁方法`lock()`调用轨迹如下：
1. ReentrantLock : `lock()`
2. NonfairSync : `ock()`
3. AbstractQueuedSynchronizer : `compareAndSetState(int expect, int update)`

&#12288;&#12288;在第3步真正开始加锁，下面是该方法的源代码：

``` java
protected final boolean compareAndSetState(int expect, int update) {
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```

&#12288;&#12288;该方法以CAS原子操作的方式更新state变量。

&#12288;&#12288;对公平锁和非公平锁的内存语义做个总结：
* 公平锁和非公平锁释放时，最后都要写一个volatile变量`state`。
* 公平锁获取时，首先会去读这个volatile变量。
* 非公平锁获取时，首先会用CAS更新这个volatile变量，CAS操作同时具有volatile读和volatile写的内存语义。

























