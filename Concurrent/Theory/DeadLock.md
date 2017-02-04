# <center>Dead Lock</center>



## 1. 如何避免死锁
&#12288;&#12288;三种用于避免死锁的技术：

1. 加锁顺序
2. 加锁时限
3. 死锁检测

<br></br>



### 1.1 加锁顺序
&#12288;&#12288;如果能确保所有线程都按照相同顺序获得锁，那么死锁就不会发生。

```
Thread 1:
  lock A 
  lock B

Thread 2:
   wait for A
   lock C (when A locked)

Thread 3:
   wait for A
   wait for B
   wait for C
```

&#12288;&#12288;如果一个线程（比如线程 3）需要一些锁，那么它须按照确定的顺序获取锁。它只有获得了从顺序上排在前面的锁之后，才能获取后面的锁。

&#12288;&#12288;例如，线程2和3只有在获取了锁A后才能尝试获取锁C。因为线程1已拥有锁A，所以线程2和3需一直等到锁A被释放。然后在尝试对B或C加锁前，必须成功地对A加锁。

&#12288;&#12288;按照顺序加锁是一种有效的死锁预防机制。但需事先知道所有可能会用到的锁并对这些锁做适当的排序，但总有些时候是无法预知的。

<br>


### 1.2 加锁时限
&#12288;&#12288;另外一个避免死锁的方法是在尝试获取锁的时候加一个超时时间。若一个线程没有在给定的时限内成功获得所有需要的锁，则会进行回退并释放所有已经获得的锁，然后等待一段随机的时间再重试。这段随机的等待时间让其它线程有机会尝试获取相同的这些锁，并且让该应用在没有获得锁的时候可以继续运行。

&#12288;&#12288;以下展示了两个线程以不同的顺序尝试获取相同的两个锁，在发生超时后回退并重试的场景：

```
Thread 1 locks A
Thread 2 locks B

Thread 1 attempts to lock B but is blocked
Thread 2 attempts to lock A but is blocked

Thread 1's lock attempt on B times out
Thread 1 backs up and releases A as well
Thread 1 waits randomly (e.g. 257 millis) before retrying.

Thread 2's lock attempt on A times out
Thread 2 backs up and releases B as well
Thread 2 waits randomly (e.g. 43 millis) before retrying.
```

&#12288;&#12288;如果有非常多线程同时去竞争同一批资源，就算有超时和回退机制，还可能会导致线程重复地尝试但却始终得不到锁。

&#12288;&#12288;另一个问题，在Java中不能对synchronized同步块设置超时时间。需要创建一个自定义锁，增加了代码复杂度。

<br>


### 1.3 死锁检测
&#12288;&#12288;主要针对那些不可能实现按序加锁并且锁超时也不可行的场景。

&#12288;&#12288;每当一个线程获得了锁，会在线程和锁相关的数据结构中（map、graph 等等）将其记下。除此之外，每当有线程请求锁，也需要记录在这个数据结构中。

&#12288;&#12288;当一个线程请求锁失败时，这个线程可以遍历锁的关系图看看是否有死锁发生。例如，线程A请求锁7，但是锁7被线程B持有，这时线程A可以检查线程B是否已请求线程A当前所持有的锁。如果线程B确实有这样的请求，就发生了死锁。当检测出死锁时，该做些什么呢？

&#12288;&#12288;一个做法是释放所有锁，回退，且等待一段随机时间后重试。这个加锁超时类似，不一样的是只有死锁已经发生了才回退，而不会是因为加锁的请求超时了。虽然有回退和等待，但是如果有大量的线程竞争同一批锁，它们还是会重复地死锁（原因同超时类似，不能从根本上减轻竞争）。

&#12288;&#12288;更好的方案是给这些线程设置优先级，让一个（或几个）线程回退，剩下的线程就像没发生死锁一样继续保持着它们需要的锁。如果赋予这些线程的优先级是固定不变的，同一批线程总是会拥有更高的优先级。为避免这个问题，可以在死锁发生的时候设置随机的优先级。

<br></br>



## 2. 非公平锁
&#12288;&#12288;用Lock类做的一个实现：

``` java
public class Synchronizer{
    Lock lock = new Lock();
    public void doSynchronized() throws InterruptedException{
        this.lock.lock();
        //critical section, do a lot of work which takes a long time
        this.lock.unlock();
    }
}

public class Lock{
    private boolean isLocked = false;
    private Thread lockingThread = null;

    public synchronized void lock() throws InterruptedException{
    	while(isLocked){
        	wait();
    	}

    isLocked = true;
    lockingThread = Thread.currentThread();
}

public synchronized void unlock(){
    if(this.lockingThread != Thread.currentThread()){
         throw new IllegalMonitorStateException(
			"Calling thread has not locked this lock");
    }

    isLocked = false;
    lockingThread = null;
    notify();
    }
}
```

&#12288;&#12288;如果多线程并发访问`lock()`，这些线程将阻塞。另外，如果锁已经锁上（指`isLocked`等于`true`），这些线程将阻塞在 `while(isLocked)`循环的`wait()`调用里。当线程正在等待进入`lock()`时，可以调用`wait()`释放其锁实例对应的同步锁，使得其他多个线程可以进入`lock()`方法，并调用`wait()`方法。

&#12288;&#12288;对于`doSynchronized()`，在`lock()`和`unlock()`间的注释说明这两个调用之间的代码将运行很长一段时间。意味着大部分时间用在等待进入锁和进入临界区的过程是用在`wait()`的等待中，而不是被阻塞在试图进入`lock()`方法中。

&#12288;&#12288;因为同步块不会对等待进入的多个线程谁能获得访问做任何保障，当调用`notify()`时，`wait()`也不会做保障一定能唤醒某个线程。因此，这个Lock类是非公平的。

<br></br>



## 3. 公平锁
&#12288;&#12288;每一个调用`lock()`的线程都会进入一个队列，当解锁后，只有队列里的第一个线程被允许锁住Farlock实例，所有其它的线程都将处于等待状态，直到他们处于队列头部。

``` java
public class FairLock {
    private boolean           isLocked       = false;
    private Thread            lockingThread  = null;
    private List<QueueObject> waitingThreads =
            new ArrayList<QueueObject>();

  public void lock() throws InterruptedException {
    QueueObject queueObject           = new QueueObject();
    boolean     isLockedForThisThread = true;
    synchronized(this){
        waitingThreads.add(queueObject);
    }

    while(isLockedForThisThread){
      synchronized(this){
        isLockedForThisThread =
            isLocked || waitingThreads.get(0) != queueObject;
        if(!isLockedForThisThread){
           isLocked = true;
           waitingThreads.remove(queueObject);
           lockingThread = Thread.currentThread();
           return;
         }
      }
      try{
        queueObject.doWait();
      }catch(InterruptedException e){
        synchronized(this) { waitingThreads.remove(queueObject); }
        throw e;
      }
    }
  }

  public synchronized void unlock(){
    if(this.lockingThread != Thread.currentThread()){
      throw new IllegalMonitorStateException(
        "Calling thread has not locked this lock");
    }
    isLocked      = false;
    lockingThread = null;
    if(waitingThreads.size() > 0) {
      waitingThreads.get(0).doNotify();
    }
  }
}

public class QueueObject {

    private boolean isNotified = false;

    public synchronized void doWait() throws InterruptedException {
    	while(!isNotified){
        	this.wait();
   	 	}
    	this.isNotified = false;
	}

	public synchronized void doNotify() {
    	this.isNotified = true;
    	this.notify();
	}

	public boolean equals(Object o) {
    	return this == o;
	}
}
```

&#12288;&#12288;`lock()`方法不在声明为synchronized，取而代之的是对必需同步的代码，在synchronized中进行嵌套。

&#12288;&#12288;FairLock创建了一个QueueObject实例，并对每个调用`lock()`的线程进行入队。调用`unlock()`的线程将从队列头部获取 QueueObject，并对其调用`doNotify()`以唤醒在该对象上等待的线程。通过这种方式，在同一时间仅有一个等待线程获得唤醒，而不是所有的等待线程。

&#12288;&#12288;QueueObject实际是一个semaphore。`doWait()`和`doNotify()`方法在QueueObject中保存着信号。这样可以避免一个线程在调用`queueObject.doWait()`之前被另一个调用`unlock()`并随之调用`queueObject.doNotify()`的线程重入，从而导致信号丢失。`queueObject.doWait()`调用放置在`synchronized(this)`块之外，以避免被 monitor嵌套锁死，所以另外的线程可以解锁，只要当没有线程在`lock()`方法的`synchronized(this)`块中执行即可。

&#12288;&#12288;最后，注意`queueObject.doWait()`在try–catch块中是怎样调用的。在InterruptedException抛出的情况下，线程得以离开`lock()`，并需让它从队列中移除。

