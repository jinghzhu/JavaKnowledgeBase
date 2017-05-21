# <center>Synchronized</center>

> 底层是基于CAS操作的等待队列，把等待队列分为`ContentionList`和`EntryList`。由JVM实现，不像ReentrantLock由上层类实现。是**非公平悲观锁**。



## 1. Java同步快应用场景
&#12288;&#12288;有四种不同的同步块：

1. 实例方法
2. 静态方法
3. 实例方法中的同步块
4. 静态方法中的同步块


### 1.1 实例方法同步

``` java
public synchronized void add(int value){
    this.count += value;
}
```

&#12288;&#12288;实例方法同步是同步在拥有该方法的对象上。每个实例其方法同步都同步在不同的对象上，即该方法所属的实例。只有一个线程能够在实例方法同步块中运行。如果有多个实例存在，那么一个线程一次可以在一个实例同步块中执行操作。一个实例一个线程。

<br>


### 1.2 静态方法同步

``` java
public static synchronized void add(int value){
    count += value;
}
```

&#12288;&#12288;静态方法同步是指同步在该方法所在的类对象上。一个类只能对应一个类对象，所以同时只允许一个线程执行同一个类中的静态同步方法。

&#12288;&#12288;对于不同类中的静态同步方法，一个线程可以执行每个类中的静态同步方法而无需等待。不管类中的那个静态同步方法被调用，一个类只能由一个线程同时执行。

<br>


### 1.3 实例方法中的同步块

``` java
public void add(int value){
    synchronized(this){
       this.count += value;
    }
}
```

&#12288;&#12288;Java 同步块构造器用括号将对象括起来。在上例中，使用了“this”，即为调用`add()`方法的实例本身。在同步构造器中用括号括起来的对象叫做监视器对象。上述代码使用监视器对象同步，同步实例方法使用调用方法本身的实例作为监视器对象。

> 一次只有一个线程能够在同步于同一个监视器对象的 Java 方法内执行。

&#12288;&#12288;下面两个例子都同步他们所调用的实例对象上，因此他们在同步的执行效果上是等效的。

``` java
public class MyClass {
    public synchronized void log1(String msg1, String msg2){
       log.writeln(msg1);
       log.writeln(msg2);
    }

    public void log2(String msg1, String msg2){
       synchronized(this){
          log.writeln(msg1);
          log.writeln(msg2);
       }
    }
}
```

&#12288;&#12288;上例中，每次只有一个线程能够在两个同步块中任意一个方法内执行。如果第二个同步块不是同步`this`实例对象上，那么两个方法可以被线程同时执行。

<br>


### 1.4 静态方法中的同步块
&#12288;&#12288;下面是两个静态方法同步的例子。这些方法同步在该方法所属的类对象上。这两个方法不允许同时被线程访问。

``` java
public class MyClass {
    public static synchronized void log1(String msg1, String msg2){
       log.writeln(msg1);
       log.writeln(msg2);
    }

    public static void log2(String msg1, String msg2){
       synchronized(MyClass.class){
          log.writeln(msg1);
          log.writeln(msg2);
       }
    }
}
```

&#12288;&#12288;如果第二个同步块不是同步在`MyClass.class`对象上。那么这两个方法可以同时被线程访问。

<br></br>



## 2.内部状态和队列

* Contention List：所有请求锁的线程将被首先放置到该竞争队列
* Entry List：Contention List中那些有资格成为候选人的线程被移到Entry List
* Wait Set：那些调用wait方法被阻塞的线程被放置到Wait Set
* OnDeck：任何时刻最多只能有一个线程正在竞争锁，该线程称为OnDeck
* Owner：获得锁的线程称为Owner
* !Owner：释放锁的线程

![Synchronized](./Images/synchronized.png)

<br></br>



## 3. 工作流程
* 新请求锁的线程将首先被加入到ContentionList中

1. `ContentionList`只是一个虚拟队列，原因在于`ContentionList`是由`Node`及其`next`指针逻辑构成，并不存在一个Queue的数据结构。
2. `ContentionList`是一个后进先出（LIFO）的队列，每次新加入Node时都会在队头进行，通过CAS改变第一个节点的的指针为新增节点，同时设置新增节点的next指向后续节点，而取得操作则发生在队尾。显然，该结构其实是个Lock-Free的队列。
3. 因为只有Owner线程才能从队尾取元素，也即线程出列操作无争用，当然也就避免了CAS的ABA问题。

* 当某个拥有锁的线程（Owner状态）调用unlock之后，如果发现EntryList为空则从ContentionList中移动线程到EntryList

1. `EntryList`与`ContentionList`逻辑上同属等待队列，`ContentionList`会被线程并发访问，为了降低对`ContentionList`队尾的争用，而建立`EntryList`。
2. Owner线程在unlock时会从`ContentionList`中迁移线程到`EntryList`，并会指定`EntryList`中的某个线程（一般为Head）为Ready（OnDeck）线程。
3. Owner线程并不是把锁传递给OnDeck线程，只是把竞争锁的权利交给OnDeck，OnDeck线程需要重新竞争锁。
4. OnDeck线程获得锁后即变为owner线程，无法获得锁则会依然留在`EntryList`中，考虑到公平性，在`EntryList`中的位置不发生变化（依然在队头）。如果Owner线程被wait方法阻塞，则转移到WaitSet队列；如果在某个时刻被notify/notifyAll唤醒，则再次转移到`EntryList`。

* 处于ContetionList、EntryList、WaitSet中的线程均处于阻塞状态，阻塞操作由操作系统完成。线程被阻塞后便进入内核（Linux）调度状态
        
&#12288;&#12288;缓解上述问题的办法便是自旋，其原理是：当发生争用时，若Owner线程能在很短的时间内释放锁，则那些正在争用线程可以稍微等一等（自旋），在Owner线程释放锁后，争用线程可能会立即得到锁，从而避免了系统阻塞 。

&#12288;&#12288;那`synchronized`实现何时使用了自旋锁？答案是在线程进入`ContentionList`时，也即第一步操作前。
 
<br></br>



## 4. 当一个线程进入某个对象的一个synchronized的实例方法后，其它线程是否可进入此对象的其它方法？
* 如果另一个方法不是`synchronized`则可以进入。
* 如果另一个方法是`static`方法，则可以进入，因为对于`static`方法，获取的是类的`class`对象（字节码文件）的锁而非同一个实例对象的锁。
* 如果正在进入的对象可以通过`wait()`释放锁，那么该线程还可以进入这个对象的其他`synchronized`方法。
* 如果另一个方法也是`synchronized`的，并且已经进入的方法中不能`wait()`释放锁，那么就不能进入那个方法。

<br></br>



## 6. synchronized vs ReentrantLock
* `synchronized`是Java语言特性，得到虚拟机直接支持，`Lock`是`concurrent`包下的类
* synchronized在进入退出同步方法代码块时会自动获取释放锁，`ReentrantLock`须显式获取锁，且要在`finally`中显式释放锁 。Synchronization code is much cleaner and easy to maintain.
* 在资源竞争不是很激烈的情况下，`Synchronized`性能优于`ReetrantLock`，但在资源竞争激烈情况下，`Synchronized`性能会下降几十倍，但是`ReetrantLock`的性能能维持常态。
* ReentrantLock提供了更大的灵活性：
    * 可以通过`tryLock`实现轮询或定时获取锁，可用于避免死锁的发生
    * `lockInterruptibly`方法能在获取锁的过程中保持对中断的响应
    * `synchronized`方法和`synchronized`块都是基于块结构的加锁，`ReentrantLock`可用于非块结构加锁（例如ConcurrentHashMap中的分段锁）
    * `synchronized`使用的内置锁和`ReentrantLock`默认都是非公平的，`ReentrantLock`在构造时可选择公平锁。

<br></br>



## 7. Condition VS wait/notify
&#12288;&#12288;Lock对应synchronized，使用之前都要先获取锁：

|            |    Object  | Condition  |
| ---------- | :--------: | :--------: |
| 休眠        |   wait     |  await     |
| 唤醒单个线程 |   notify   |  signal    |
| 唤醒所有线程 |  notifyAll | signalAll  |
                                     
&#12288;&#12288;Condition更强大的地方在于能够更加精细的控制多线程的休眠与唤醒。 

&#12288;&#12288;例如，多线程读/写缓冲区：当向缓冲区中写入数据之后，唤醒读线程；当从缓冲区读出数据之后，唤醒写线程。 如果采用`Object`类中的`wait()`, `notify()`, `notifyAll()`实现该缓冲区，当向缓冲区写入数据之后需要唤醒读线程时，不可能通过`notify()`或`notifyAll()`明确的指定唤醒读线程，而只能通过`notifyAll`唤醒所有线程(但是`notifyAll()`无法区分唤醒的线程是读线程，还是写线程)。 通过Condition，就能明确的指定唤醒读线程。

<br></br>



## 8. synchronized与static synchronized区别
&#12288;&#12288;synchronized是对类当前实例加锁，防止其他线程同时访问该类的该实例的synchronized块，注意是“类当前实例”， 类的两个不同实例没有这种约束。
        
&#12288;&#12288;static synchronized是控制类所有实例的访问，static synchronized是限制线程同时访问jvm中该类的所有实例同时访问对应的代码快。



