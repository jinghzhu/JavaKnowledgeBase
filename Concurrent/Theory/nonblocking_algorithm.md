# <center>(Non-)Blocking Algorithm</center>



<br></br>

* 非阻塞算法是一种允许线程在阻塞其他线程的情况下访问共享状态的算法。非阻塞算法和阻赛算法的不同在于当请求操作不能够执行时阻塞算法和非阻塞算法会怎么做。阻塞算法会阻塞线程知道请求操作可以被执行。非阻塞算法会通知请求线程操作不能够被执行，并返回。
* volatile变量是非阻塞的。

<br></br>



## 阻塞并发算法
----
阻塞并发算法分下面两步：
1. 执行线程请求的操作
2. 阻塞线程直到可以安全地执行操作

很多算法和并发数据结构都是阻塞的。例如，java.util.concurrent.BlockingQueue的不同实现都是阻塞数据结构。

<br></br>



## 非阻塞并发算法
----
非阻塞并发算法包含下面两步：
1. 执行线程请求的操作
2. 通知请求线程操作不能被执行

AtomicBoolean，AtomicInteger，AtomicLong，AtomicReference都是非阻塞数据结构的例子。

<br></br>



## 非阻塞并发数据结构
----
### volatile
volatile变量是非阻塞的。修改volatile变量的值是原子操作。不过，在volatile变量上的read-update-write 顺序的操作不是原子的。

当仅有一个线程更新一个变量，不管有多少线程在读这个变量，都不会发生竞态条件。因此，当仅有一个线程在写一个共享变量时，可以把这个变量声明为volatile。

当多个线程在一个共享变量上执行read-update-write操作时会发生竞态条件。如果只有一个线程在执行raed-update-write 操作，其他线程都在执行读操作，将不会发生竞态条件。

<br>


### 使用CAS的乐观锁
如果需要多线程写同一个共享变量，volatile是不合适的。需要一些类型的排它锁（悲观锁）访问这个变量：

``` java
public class SynchronizedCounter{
    long count = 0;

    public void inc(){
        synchronized(this){
            count++;
        }
    }

    public long count(){
        synchronized(this){
            return this.count;
        }
    }
}
```

可使用原子变量来代替同步块，比如AtomicLong：
``` java
import java.util.concurrent.atomic.AtomicLong;

public class AtomicLong{
    private AtomicLong count = new AtomicLong(0);

    public void inc(){
        boolean updated = false;

        while(!updated){
            long prevCount = this.count.get();
            updated = this.count.compareAndSet(prevCount, prevCount + 1);
        }
    }

    public long count(){
        return this.count.get();
    }
}
```

这个版本仅是上一个的线程安全版本。`inc()`方法中不再含有一个同步块。而是被下面这些代码替代：

``` java
boolean updated = false;
while(!updated){
    long prevCount = this.count.get();
    updated = this.count.compareAndSet(prevCount, prevCount + 1);
}
```

没有包含任何的竞态条件，在于`compareAndSet()`方法是原子操作。

<br></br>



## 非阻塞算法模板
----
线程对修改某个数据结构的过程变成了下面这样：
1. 检查是否另一个线程已经提交了对这个数据结构提交了修改
2. 如果没有其他线程提交了一个预期的修改，创建一个预期的修改，然后向这个数据结构提交预期的修
3. 执行对共享数据结构的修改
4. 移除对这个预期的修改的引用，向其它线程发送信号，告诉它们这个预期的修改已经被执行

第二步可以阻塞其他线程提交一个预期的修改。因此，第二步实际的工作是作为这个数据结构的一个锁。如果一个线程成功提交了一个预期的修改，其他线程就不可以再提交一个预期的修改直到第一个预期的修改执行完毕。如果一程提交预期的修改，然后做其它的工作时发生阻塞，这时这个数据结构是被锁住的。其它线程可以检测到不能够提交预期的修改，然后回去做一些其它的事情。

很明显，需要解决这个问题。

为避免已提交的预期修改锁住共享数据结构，已提交的预期修改须包含足够的信息让其他线程来完成这次修改。因此，如果一个提交了预期修改的线程从未完成这次修改，其他线程可以在它的支持下完成这次修改，保证这个共享数据结构对其他线程可用。

下图说明了上面描述的非阻塞算法的蓝图：

<p align="center">
  <img src="./Images/nonblocking3.png"/>
</p>

<br>

修改须被当做一个或多个CAS操作来执行。如果两个线程尝试完成同一个预期修改，仅有一个线程可以执行所有的CAS操作。一旦一条CAS操作完成后，再次企图完成这个CAS操作都不会得逞。

``` java
import java.util.concurrent.atomic.AtomicBoolean;
import java.util.concurrent.atomic.AtomicStampedReference;

public class NonblockingTemplate{
    public static class IntendedModification{
        public AtomicBoolean completed = new AtomicBoolean(false);
    }

    private AtomicStampedReference<IntendedModification> ongoinMod = 
                new AtomicStampedReference<IntendedModification>(null, 0);
    //declare the state of the data structure here.

    public void modify(){
        while(!attemptModifyASR());
    }

    public boolean attemptModifyASR(){
        boolean modified = false;

        IntendedMOdification currentlyOngoingMod = ongoingMod.getReference();
        int stamp = ongoingMod.getStamp();

        if(currentlyOngoingMod == null){
            //copy data structure - for use in intended modification
            //prepare intended modification
            IntendedModification newMod = new IntendModification();
            boolean modSubmitted = ongoingMod.compareAndSet(null, newMod, stamp, stamp + 1);

            if(modSubmitted){
                //complete modification via a series of compare-and-swap operations.
                //note: other threads may assist in completing the compare-and-swap
                // operations, so some CAS may fail
                modified = true;
            }
        } else {
            //attempt to complete ongoing modification, so the data structure is freed up
            //to allow access from this thread.
            modified = false;
        }

        return modified;
    }
}
```

<br></br>
