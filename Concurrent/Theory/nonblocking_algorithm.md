# <center>(Non-)Blocking Algorithm</center>



<br></br>

在并发上下文中，非阻塞算法是一种允许线程在阻塞其他线程的情况下访问共享状态的算法。非阻塞算法和阻赛算法的不同在于当请求操作不能够执行时阻塞算法和非阻塞算法会怎么做。阻塞算法会阻塞线程知道请求操作可以被执行。非阻塞算法会通知请求线程操作不能够被执行，并返回。

<br></br>



## 阻塞并发算法
----
一个阻塞并发算法分下面两步：
1. 执行线程请求的操作
2. 阻塞线程直到可以安全地执行操作

很多算法和并发数据结构都是阻塞的。例如，java.util.concurrent.BlockingQueue的不同实现都是阻塞数据结构。如果一个线程要往一个阻塞队列中插入一个元素，队列中没有足够的空间，执行插入操作的线程就会阻塞直到队列中有了可以存放插入元素的空间。

<p align="center">
  <img src="./Images/blocking1.png"/>
</p>

<center><i>一个阻塞算法保证一个共享数据结构的行为</i></center>

<br></br>



## 非阻塞并发算法
一个非阻塞并发算法包含下面两步：

1. 执行线程请求的操作
2. 通知请求线程操作不能被执行

Java包含几个非阻塞数据结构：AtomicBoolean，AtomicInteger，AtomicLong，AtomicReference 都是非阻塞数据结构的例子。

<p align="center">
  <img src="./Images/nonblocking1.png"/>
</p>

<center><i>一个非阻塞算法保证一个共享数据结构的行为</i></center>

<br></br>



## 3. 非阻塞并发数据结构

> 如果算法确保并发数据结构是阻塞的，就称为阻塞算法。这个数据结构称为阻塞，并发数据结构。
>   
> 如果算法确保并发数据结构是非阻塞的，就称为非阻塞算法。这个数据结构称为非阻塞，并发数据结构。


### 3.1 Volatile变量
**volatile变量是非阻塞的**。修改volatile变量的值是原子操作。不过，在volatile变量上的read-update-write 顺序的操作不是原子的。

#### 3.1.1 单个写线程的情景
当仅有一个线程在更新一个变量，不管有多少线程在读这个变量，都不会发生竞态条件。因此，当仅有一个线程在写一个共享变量时，可以把这个变量声明为volatile。

当多个线程在一个共享变量上执行read-update-write操作时会发生竞态条件。如果只有一个线程在执行raed-update-write 操作，其他线程都在执行读操作，将不会发生竞态条件。


#### 3.1.2 基于volatile高级数据结构
使用多个volatile变量去创建数据结构是可以的，构建出的数据结构中每个volatile变量仅被单个线程写，被多个线程读。每个volatile变量可被一个不同的线程写（但仅有一个）。使用像这样的数据结构多个线程可以使用这些volatile变量以一个非阻塞的方法彼此发送信息。

下面是一个简单的例子：

``` java
public class DoubleWriterCounter{
    private volatile long countA = 0;
    private volatile long countB = 0;

    /**
     *Only one (and the same from thereon) thread may ever call this method,
     *or it will lead to race conditions.
     */
     public void incA(){
         this.countA++;
     }

     /**
      *Only one (and the same from thereon) thread may ever call this method, 
      *or it will  lead to race conditions.
      */
      public void incB(){
          this.countB++;
      }

      /**
       *Many reading threads may call this method
       */
      public long countA(){
          return this.countA;
      }

     /**
      *Many reading threads may call this method
      */
      public long countB(){
          return this.countB;
      }
}
```

DoubleWriterCoounter包含两个volatile变量以及两对自增和读方法。在某刻，仅单个线程可以调用`inc()`，仅单个线程可以访问`incB()`。不同线程可同时调用`incA()`和`ncB()`。`countA()`和`countB()`可被多个线程调用。不会引发竞态条件。

DoubleWriterCoounter可被用来比如线程间通信。`countA()`和`countB()`可分别用来存储生产的任务数和消费的任务数。下图展示了两个线程通过类似于上面的一个数据结构进行通信的：

<p align="center">
  <img src="./Images/nonblocking2.png"/>
</p>

<br>


### 3.2 使用CAS的乐观锁
如果需要多线程写同一个共享变量，volatile是不合适的。需要一些类型的排它锁（悲观锁）访问这个变量。下面演示了 Java中的同步块进行排他访问的：

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

可以使用原子变量来代替同步块，比如AtomicLong：

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

这些代码不是原子操作。也就是说，两个不同的线程调用`inc()`方法，然后执行`long prevCount = this.count.get()`，获得了这个计数器的上一个`count`。但是，上面的代码并没有包含任何的竞态条件。秘密在于`compareAndSet()`方法是原子操作。`compareAndSet()`被CPU的compare-and-swap指令直接支持。因此，不需要去同步，也不需要挂起线程。


#### 3.2.1 为什么称它为乐观锁

**上一部分展现的代码被称为乐观锁（optimistic locking）**。乐观锁区别于传统的锁（悲观锁），传统的锁会使用同步块或其他类型的锁阻塞对临界区域的访问。一个同步块或锁可能会导致线程挂起。乐观锁允许所有线程在不发生阻塞的情况下创建一份共享内存的拷贝。这些线程接下来可能对拷贝进行修改，并企图把修改后的版本写回到共享内存中。如果没有其它线程对共享内存做任何修改，CAS操作允许线程将它的变化写回到共享内存中去。如果，另一个线程已经修改了共享内存，这个线程将不得不再次获得一个新的拷贝，在新的拷贝上做出修改，并尝试再次把它们写回到共享内存中去。

称之为“乐观锁”的原因是，线程获得想修改的数据的拷贝并做出修改，在乐观的假在此期间没有线程对共享内存做出修改的情况下。当这个乐观假设成立时，这个线程仅仅在无锁的情况下完成共享内存的更新。当这个假设不成立时，线程所做的工作就会被丢弃，但任然不使用锁。

乐观锁使用于共享内存竞用不是非常高的情况。如果共享内存上的内容非常多，仅仅因为更新共享内存失败，就浪费大量CPU 周期用在拷贝和修改上。但是，如果共享内存上有大量的内容，无论如何，你都要把你的代码设计的产生的争用更低。

**乐观锁是非阻塞的**。如果一个线程获得共享内存的拷贝，当尝试修改时，发生了阻塞，其它线程去访问这块内存区域不会发生阻塞。对于传统的加/解锁，当线程持有锁时，其它所有的线程都会阻塞直到持有锁的线程释放锁。



#### 3.2.2 不可替换的数据结构
简单的CAS乐观锁可用于共享数据结果，这样整个数据结构可以通过单个的CAS操作被替换成为新的数据结构。

假设，这个共享数据结构是队列。每当线程尝试从向队列中插入或从队列中取出元素时，都必须拷贝这个队列然后在拷贝上做出期望的修改。可以使用AtomicReference来达到同样的目的。拷贝引用，拷贝和修改队列，尝试替换在AtomicReference中的引用让它指向新创建的队列。然而，一个大的数据结构可能会需要大量的内存和CPU周期来复制。这会占用大量的内存和浪费大量的时间再拷贝操作上。



<br></br>



## 4. 一种实现非阻塞数据结构的方法
这种数据结构可以被并发修改，而不仅是拷贝和修改。

### 4.1 共享预期的修改
用来替换拷贝和修改整个数据结构，一个线程可以共享它们对共享数据结构预期的修改。一个线程向对修改某个数据结构的过程变成了下面这样：
1. 检查是否另一个线程已经提交了对这个数据结构提交了修改
2. 如果没有其他线程提交了一个预期的修改，创建一个预期的修改，然后向这个数据结构提交预期的修
3. 执行对共享数据结构的修改
4. 移除对这个预期的修改的引用，向其它线程发送信号，告诉它们这个预期的修改已经被执行

第二步可以阻塞其他线程提交一个预期的修改。因此，第二步实际的工作是作为这个数据结构的一个锁。如果一个线程成功提交了一个预期的修改，其他线程就不可以再提交一个预期的修改直到第一个预期的修改执行完毕。如果一程提交预期的修改，然后做其它的工作时发生阻塞，这时这个数据结构是被锁住的。其它线程可以检测到不能够提交预期的修改，然后回去做一些其它的事情。

很明显，需要解决这个问题。

<br>


### 4.2 可完成的预期修改
为避免已提交的预期修改锁住共享数据结构，已提交的预期修改须包含足够的信息让其他线程来完成这次修改。因此，如果一个提交了预期修改的线程从未完成这次修改，其他线程可以在它的支持下完成这次修改，保证这个共享数据结构对其他线程可用。

下图说明了上面描述的非阻塞算法的蓝图：

<p align="center">
  <img src="./Images/nonblocking3.png"/>
</p>

修改须被当做一个或多个CAS操作来执行。如果两个线程尝试完成同一个预期修改，仅有一个线程可以所有的CAS 操作。一旦一条CAS操作完成后，再次企图完成这个CAS操作都不会“得逞”。

<br>


### 4.3 一个非阻塞算法模板

``` java
import java.util.concurrent.atomic.AtomicBoolean;
import java.util.concurrent.atomic.AtomicStampedReference;

public class NonblockingTemplate{
    public static class IntendedModification{
        public AtomicBoolean completed = new AtomicBoolean(false);
    }

    private AtomicStampedReference<IntendedModification> ongoinMod = new AtomicStampedReference<IntendedModification>(null, 0);
    //declare the state of the data structure here.

    public void modify(){
        while(!attemptModifyASR());
    }

    public boolean attemptModifyASR(){
        boolean modified = false;

        IntendedMOdification currentlyOngoingMod = ongoingMod.getReference();
        int stamp = ongoingMod.getStamp();

        if(currentlyOngoingMod == null){
            //copy data structure - for use
            //in intended modification

            //prepare intended modification
            IntendedModification newMod = new IntendModification();

            boolean modSubmitted = ongoingMod.compareAndSet(null, newMod, stamp, stamp + 1);

            if(modSubmitted){
                 //complete modification via a series of compare-and-swap operations.
                //note: other threads may assist in completing the compare-and-swap
                // operations, so some CAS may fail
                modified = true;
            }
        }else{
             //attempt to complete ongoing modification, so the data structure is freed up
            //to allow access from this thread.
            modified = false;
        }

        return modified;
    }
}
```

Java提供了一些非阻塞实现（比如ConcurrentLinkedQueue）。除了Java内置非阻塞数据结构还有很多开源的非阻塞数据结构可以使用。例如，LAMX Disrupter和Cliff Click实现的非阻塞HashMap。

<br></br>



## 5. 非阻塞算法好处
1. 选择：给了线程选择当它们请求的动作不能够被执行时做些什么。不再是被阻塞在那，请求线程关于做什么有了一个选择。有时一个线程什么也不能做。在这种情况下，它可以选择阻塞或自我等待，像这样把CPU的使用权让给其它任务。至少给了请求线程选择的机会。
2. 没有死锁：一个线程的挂起不能导致其它线程挂起。意味着不会发生死锁。两个线程不能互相彼此等待来获得被对方持有的锁。非阻塞算法任可能产生活锁（live lock），两个线程一直请求一些动作，但一直被告知不能够被执行（因为其他线程的动作）。
3. 没有线程挂起：挂起和恢复一个线程的代价是昂贵的。无论什么时候，线程阻塞就会被挂起。由于使用非阻塞算法线程不会被挂起，这种过载就不会发生。意味着CPU可能花更多时间在执行实际的业务逻辑上而不是上下文切换。
4. 降低线程延迟：延迟指一个请求产生到线程实际执行它之间的时间。在非阻塞算法中线程不会被挂起，就不需要付线程激活成本。意味着当一个请求执行时可以得到更快的响应，减少它们的响应延迟。

非阻塞算法通常忙等待直到请求动作可以被执行来降低延迟。在一个非阻塞数据数据结构有着很高的线程争用的系统中，CPU 可能在它们忙等待期间停止消耗大量CPU周期。非阻塞算法可能不是最好的选择如果数据结构有很高的线程争用。