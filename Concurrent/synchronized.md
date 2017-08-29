# <center>Synchronized</center>



<br></br>

* 底层是基于CAS操作的等待队列，把等待队列分为ContentionList和EntryList。
* 由JVM实现，不像ReentrantLock由上层类实现。
* 非公平悲观锁。

![Synchronized](./Images/synchronized.png)

<br></br>



## Java同步快应用场景
----
有四种不同的同步块：
1. 实例方法
2. 静态方法
3. 实例方法中的同步块
4. 静态方法中的同步块

<br>


### 实例方法同步

``` java
public synchronized void add(int value){
    this.count += value;
}
```

实例方法同步是同步在拥有该方法的对象上。每个实例其方法同步都同步在不同的对象上，即该方法所属的实例。只有一个线程能够在实例方法同步块中运行。如果有多个实例存在，那么一个线程一次可以在一个实例同步块中执行操作。一个实例一个线程。

<br>


### 静态方法同步

``` java
public static synchronized void add(int value){
    count += value;
}
```

静态方法同步是指同步在该方法所在的类对象上。一个类只能对应一个类对象，所以同时只允许一个线程执行同一个类中的静态同步方法。

对于不同类中的静态同步方法，一个线程可以执行每个类中的静态同步方法而无需等待。不管类中的那个静态同步方法被调用，一个类只能由一个线程同时执行。

<br>


### 实例方法中的同步块

``` java
public void add(int value){
    synchronized(this){
       this.count += value;
    }
}
```

上例中，使用了`this`，即为调用`add()`方法的实例本身。在同步构造器中用括号括起来的对象叫做监视器对象。上述代码使用监视器对象同步，同步实例方法使用调用方法本身的实例作为监视器对象。

下面两个例子都同步所调用的实例对象上，因此在同步的执行效果上是等效的。
``` java
public class MyClass {
    public synchronized void log1(String msg1){
       log.writeln(msg1);
    }

    public void log2(String msg1){
       synchronized(this){
          log.writeln(msg1);
       }
    }
}
```

上例中，每次只有一个线程能够在两个同步块中任意一个方法内执行。如果第二个同步块不是同步`this`实例对象上，那么两个方法可以被线程同时执行。

<br>


### 静态方法中的同步块
这些方法同步在该方法所属的类对象上。这两个方法不允许同时被线程访问。

``` java
public class MyClass {
    public static synchronized void log1(String msg1){
       log.writeln(msg1);
    }

    public static void log2(String msg1){
       synchronized(MyClass.class){
          log.writeln(msg1);
       }
    }
}
```

如果第二个同步块不是同步在`MyClass.class`对象上。那么这两个方法可以同时被线程访问。

<br></br>



## 内部状态和队列
----
* Contention List：所有请求锁的线程将被首先放置到该竞争队列
* Entry List：Contention List中那些有资格成为候选人的线程被移到Entry List
* Wait Set：那些调用wait方法被阻塞的线程被放置到Wait Set
* OnDeck：任何时刻最多只能有一个线程正在竞争锁，该线程称为OnDeck
* Owner：获得锁的线程称为Owner
* !Owner：释放锁的线程

![Synchronized](./Images/synchronized.png)

<br></br>



## 工作流程
----
新请求锁的线程将首先被加入到ContentionList中：
1. ContentionList是一个虚拟队列，原因在于ContentionList是由`Node`及其`next`指针逻辑构成，并不存在一个Queue的数据结构。
2. ContentionList是LIFO队列，每次新加入Node时都会在队头进行，通过CAS改变第一个节点的的指针为新增节点，同时设置新增节点的next指向后续节点，而取得操作则发生在队尾。
3. 只有Owner线程才能从队尾取元素，也即线程出列操作无争用，避免了CAS的ABA问题。

当某个拥有锁的线程（Owner状态）调用`unlock()`之后，如果发现EntryList为空则从ContentionList中移动线程到EntryList：
1. EntryList与ContentionList逻辑上同属等待队列。ContentionList会被线程并发访问，为了降低对ContentionList队尾的争用，而建立EntryList。
2. Owner线程在`unlock()`时从ContentionList中迁移线程到EntryList，并会指定EntryList中的某个线程（一般为Head）为Ready（OnDeck）线程。
3. Owner线程不是把锁传递给OnDeck线程，只是把竞争锁的权利交给OnDeck，OnDeck线程要重新竞争锁。
4. OnDeck线程获得锁后变为Owner线程，无法获得锁则会依然留在EntryList中。考虑到公平性，在EntryList中的位置不发生变化（依然在队头）。如果Owner线程被`wait()`方法阻塞，则转移到WaitSet队列；如果在某个时刻被`notify()`/`notifyAll()`唤醒，则再次转移到EntryList。

处于ContetionList、EntryList和WaitSet中的线程均处于阻塞状态，阻塞操作由操作系统完成。线程被阻塞后便进入内核调度状态。
        
缓解上述问题的办法是自旋，原理是当发生争用时，若Owner线能在很短的时间内释放锁，则那些正在争用线程可以稍微等一等（自旋），在Owner线程释放锁后，争用线程可能会立即得到锁，从而避免了系统阻塞 。

那synchronized何时使用了自旋锁？是在线程进入ContentionList时，也即第一步操作前。
 
<br></br>



## 可重入规则
----
当一个线程进入某个对象的一个synchronized的实例方法后，其它线程是否可进入此对象的其它方法？
* 如果另一个方法不是synchronized则可以进入。
* 如果另一个方法是`static`方法，则可以进入。因为对于`static`方法，获取的是类的class对象（字节码文件）的锁而非同一个实例对象的锁。
* 如果正在进入的对象可以通过`wait()`释放锁，那么该线程还可以进入这个对象的其他synchronized方法。
* 如果另一个方法也是synchronized的，并且已经进入的方法中不能`wait()`释放锁，那么就不能进入那个方法。

<br></br>



## synchronized vs ReentrantLock
----
* synchronized是Java语言特性，得到虚拟机直接支持，ReentrantLock是concurrent包下的类。
* synchronized在进入退出同步方法代码块时会自动获取释放锁，ReentrantLock`须显式获取锁，且要在`finally`中显式释放锁。
* 在资源竞争不是很激烈的情况下，synchronized性能优于ReetrantLock。但在资源竞争激烈情况下，synchronized性能会下降几十倍，但是ReetrantLock性能维持常态。
* ReentrantLock提供了更大的灵活性：
    * 可以通过`tryLock()`实现轮询或定时获取锁，可用于避免死锁的发生
    * `lockInterruptibly()`方法能在获取锁的过程中保持对中断的响应
    * synchronized方法和synchronized块都是基于块结构的加锁，ReentrantLock可用于非块结构加锁（例如ConcurrentHashMap中的分段锁）
    * synchronized使用的内置锁和ReentrantLock默认都是非公平的，ReentrantLock在构造时可选择公平锁。
* Lock对应synchronized，使用之前都要先获取锁：

|            |    Object  | Condition  |
| ---------- | :--------: | :--------: |
| 休眠        |   wait     |  await     |
| 唤醒单个线程 |   notify   |  signal    |
| 唤醒所有线程 |  notifyAll | signalAll  |
                                     
<br></br>
