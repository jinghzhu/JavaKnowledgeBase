# <center>Synchronized</center>



<br></br>

* 底层是基于CAS操作的等待队列，把等待队列分为ContentionList和EntryList。
* 由JVM实现，不像ReentrantLock由上层类实现。
* 非公平悲观锁。

![Synchronized](./Images/synchronized.png)

<br></br>



## 内部状态和队列
----
* Contention List：所有请求锁线程首先放置到该竞争队列。
* Entry List：Contention List中有资格成为候选人线程移到Entry List。
* Wait Set：调用`wait()`方法被阻塞的线程放置到Wait Set。
* OnDeck：任何时刻最多只能有一个线程竞争锁，该线程称为OnDeck。
* Owner：获得锁线程称为Owner。
* !Owner：释放锁的线程。

![Synchronized](./Images/synchronized.png)

<br></br>



## 工作流程
----
新请求锁线程将先被加入ContentionList：
1. ContentionList是一个虚拟队列。原因在于ContentionList是由`Node`及`next`指针逻辑构成，不存在Queue数据结构。
2. ContentionList是LIFO队列。每次新加入Node时都在队头，通过CAS改变第一个节点的指针为新增节点。同时设置新增节点`next`指向后续节点。取得操作发生在队尾。
3. 只有Owner线程才能从队尾取元素，即线程出列操作无争用，避免CAS的ABA问题。

当拥有锁的线程（Owner状态）调用`unlock()`之后，如果发现EntryList为空，则从ContentionList中移动线程到EntryList：
1. EntryList与ContentionList逻辑上同属等待队列。ContentionList被线程并发访问，为降低对ContentionList队尾争用，而建立EntryList。
2. Owner线程在`unlock()`时，从ContentionList中迁移线程到EntryList，并指定EntryList中某个线程（一般为Head）为Ready（OnDeck）线程。
3. Owner线程不是把锁传递给OnDeck线程，只是把竞争锁权利交给OnDeck。OnDeck线程要重新竞争锁。
4. OnDeck线程获得锁后变为Owner线程。无法获得锁则依然留在EntryList中。考虑到公平性，在EntryList中位置不发生变化（依然在队头）。如果Owner线程被`wait()`阻塞，则转移到WaitSet队列；如果在某时刻被`notify()`或`notifyAll()`唤醒，则再次移到EntryList。

处于ContetionList、EntryList和WaitSet中线程均处于阻塞状态，阻塞操作由操作系统完成。线程阻塞后进入内核调度状态。缓解办法是自旋。synchronized在线程进入ContentionList时，也即第一步操作前，采取自旋。
 
<br></br>



## synchronized vs ReentrantLock
----
* synchronized是Java语言特性，得到虚拟机直接支持。ReentrantLock是concurrent包下的类。
* synchronized进入退出同步方法代码块时会自动获取释放锁。ReentrantLock须显式获取锁，且要在finally中显式释放锁。
* 在资源竞争不是很激烈的情况下，synchronized性能优于ReetrantLock。但在资源竞争激烈情况下，synchronized性能会下降几十倍，但是ReetrantLock性能维持常态。
* ReentrantLock提供更大灵活性：
    * 可通过`tryLock()`实现轮询或定时获取锁，避免死锁发生；
    * `lockInterruptibly()`能在获取锁过程中保持对中断响应；
    * synchronized方法和synchronized块都基于块结构加锁，ReentrantLock可用于非块结构加锁（例如ConcurrentHashMap中分段锁）；
    * synchronized使用内置锁和ReentrantLock默认都是非公平，ReentrantLock可选择公平锁。
* Lock对应synchronized，使用之前都要先获取锁：

|            |    Object  | Condition  |
| ---------- | :--------: | :--------: |
| 休眠        |   wait     |  await     |
| 唤醒单个线程 |   notify   |  signal    |
| 唤醒所有线程 |  notifyAll | signalAll  |
                                     
<br></br>



## 可重入规则
----
* 如果另一个方法不是synchronized则可进入。
* 如果另一个方法是静态方法，则可进入。因为对静态方法，获取的是类class对象（字节码文件）锁，而非同一个实例对象锁。
* 如果正在进入对象可通过`wait()`释放锁，那么该线程还可进入这个对象其他synchronized方法。
* 如果另一个方法也是synchronized的，且已进入方法中不能`wait()`释放锁，那么不能进入那个方法。

<br></br>



## 同步快应用场景
----
修饰对象有4种： 
1. 代码块：被修饰代码块称为同步语句块，作用范围是大括号{}括起来的代码，作用对象是调用这个代码块的对象； 
2. 方法：被修饰方法称为同步方法，作用范围是整个方法，作用对象是调用这个方法的对象； 
3. 静态方法：作用范围是整个静态方法，作用对象是这个类所有对象； 
4. 类：作用范围是synchronized后面括号括起来部分，作用对象是这个类所有对象。

<br>


### 实例方法同步

``` java
public synchronized void add(int value) {
    this.count += value;
}
```

同步在拥有该方法对象上。每个实例其方法同步都同步在不同对象上，即该方法所属的实例。

<br>


### 静态方法同步

``` java
public static synchronized void add(int value) {
    count += value;
}
```

同步在该方法所在类对象上。一个类只能对应一个类对象，所以同时只允许一个线程执行同一个类中静态同步方法。

<br>


### 实例方法同步块

``` java
public void add(int value) {
    synchronized(this) {
       this.count += value;
    }
}
```

使用`this`，即调用`add()`方法实例本身。在同步构造器中用括号括起来对象叫监视器对象。上述代码使用监视器对象同步，同步实例方法使用调用方法本身实例作为监视器对象。

下面两个例子都同步所调用实例对象上，在同步效果上等效：
``` java
public class MyClass {
    public synchronized void log1(String msg1) {
       log.writeln(msg1);
    }

    public void log2(String msg1) {
       synchronized(this) {
          log.writeln(msg1);
       }
    }
}
```

每次只有一个线程能在两个同步块中任意一个方法内执行。如果第二个同步块不是同步`this`实例对象上，那么两个方法可被线程同时执行。

<br>


### 静态方法同步块
同步在该方法所属类对象上。下面两个方法不允许同时被线程访问：

``` java
public class MyClass {
    public static synchronized void log1(String msg1) {
       log.writeln(msg1);
    }

    public static void log2(String msg1) {
       synchronized(MyClass.class) {
          log.writeln(msg1);
       }
    }
}
```

如果第二个同步块不是同步在`MyClass.class`对象上，那么这两个方法可同时被线程访问。

<br></br>
