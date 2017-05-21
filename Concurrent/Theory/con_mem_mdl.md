# <center>Concurrent Memory Model</center>

<br></br>



### 内存模型内部原理
----------------
&#12288;&#12288;JVM中存在一个主内存（Main Memory），所有的变量、所有实例和实例的字段都存储在主内存中，对于所有的线程是共享的（相当于黑板，其他人都可以看到的）。每个线程都有自己的工作内存（Working Memory），工作内存中保存的是主存中变量的拷贝（相当于自己笔记本，只能自己看到），工作内存由缓存和堆栈组成，其中缓存保存的是主存中的变量的copy，堆栈保存的是线程局部变量。

&#12288;&#12288;关于线程栈和堆。每个运行在JVM里的线程都拥有自己的线程栈。线程栈包含了线程调用的方法当前执行点相关的信息。一个线程仅能访问自己的线程栈。一个线程创建的本地变量对其它线程不可见，仅自己可见。即使两个线程执行同样的代码，这两个线程任然在在自己的线程栈中的代码来创建本地变量。因此，每个线程拥有每个本地变量的独有版本。

&#12288;&#12288;所有原始类型的本地变量都存放在线程栈上，对其它线程不可见。一个线程可能向另一个线程传递一个原始类型变量的拷贝，但是它不能共享这个原始类型变量自身。

&#12288;&#12288;堆上包含创建的所有对象，无论是哪一个对象创建的。这包括原始类型的对象版本。如果一个对象被创建然后赋值给一个局部变量，或者用来作为另一个对象的成员变量，这个对象任然是存放在堆上。

<p align="center">
  <img src="./Images/mem_model1.png"/>
</p>

<br>

&#12288;&#12288;线程对所有变量的操作都是在工作内存中进行的，线程之间无法直接互相访问工作内存，变量的值得变化的传递需要主存来完成。在JMM中通过并发线程修改的变量值，必须通过线程变量同步到主存后，其他线程才能访问到。

&#12288;&#12288;线程对某个变量的操作步骤： 
1. 从主内存中复制数据到工作内存；
2. 执行代码，对数据进行各种操作和计算；
3. 把操作后的变量值重新写回主内存中。

&#12288;&#12288;这三个步骤顺序是我们希望的，但JVM不保证第1步和第3步会按照上述次序执行。由于JVM可以对特征代码进行调优，也就改变了某些运行步骤的次序的颠倒，那么每次线程调用变量时是直接取自己的工作存储器中的值还是先从主存储器复制再取是没有保证的。同样的，线程改变变量的值之后，是否马上写回到主存储器上也是不可保证的 。 

&#12288;&#12288;还好有`synchronized`和`volatile`： 
* 多个线程共有的字段应该用`synchronized`或`volatile`来保护. 
* `synchronized`负责线程间互斥。即同时只有一个线程可以执行`synchronized`中的代码. 

&#12288;&#12288;synchronized还有另外的作用：
* 在线程进入`synchronized`块之前，会把工作存内存中的所有内容映射到主内存上，然后把工作内存清空再从主存储器上拷贝最新的值。
* 线程退出`synchronized`块时，会把工作内存中的值映射到主内存，此时并不会清空工作内存。强制其按照上面的顺序运行，以保证线程在执行完代码块后，工作内存中的值和主内存中的值是一致的，保证了数据的一致性。 
* `volatile`负责线程中的变量与主存储区同步.但不负责每个线程之间的同步. 

<br></br>



### 各种类型变量存放位置
-------------
* 原始类型（primitive type），它总是“呆在”线程栈上。

* 指向一个对象的引用。引用（这个本地变量）存放在线程栈上，但是对象本身存放在堆上。（A local variable may also be a reference to an object. In that case the reference (the local variable) is stored on the thread stack, but the object itself if stored on the heap.）

* 一个对象可能包含方法，这些方法可能包含本地变量。这些本地变量任然存放在线程栈上，即使这些方法所属的对象存放在堆上。

* 一个对象的成员变量可能随着这个对象自身存放在堆上，不管这个成员变量是原始类型还是引用类型。（An object's member variables are stored on the heap along with the object itself. That is true both when the member variable is of a primitive type, and if it is a reference to an object. ）

* 静态成员变量跟随着类一起也存放在堆上。（Static class variables are also stored on the heap along with the class definition. ）

<br></br>



### Java Memory Model And Hardware Memory Architecture
----------------
&#12288;&#12288;Java内存模型与硬件内存架构之间存在差异。硬件内存架构没有区分线程栈和堆。对于硬件，所有的线程栈和堆都分布在主内中。部分线程栈和堆可能有时候会出现在CPU缓存中和CPU内部的寄存器中。

![JVM & Hardware](./Images/jvm_hardware.png)

&#12288;&#12288;当对象和变量被存放在计算机中各种不同的内存区域中时，就会出现一些问题：

1. 线程对共享变量修改的可见性
2. 当读，写和检查共享变量时出现race conditions

<br>



#### Visibility of Shared Objects
The shared object is initially stored in main memory. A thread running on CPU one then reads the shared object into its CPU cache. There it makes a change to the shared object. As long as the CPU cache has not been flushed back to main memory, the changed version of the shared object is not visible to threads running on other CPUs. This way each thread may end up with its own copy of the shared object, each copy sitting in a different CPU cache. 

&#12288;&#12288;跑在左边CPU的线程拷贝这个共享对象到它的CPU缓存中，然后将`count`变量的值改为2。这个修改对跑在右边CPU上的其它线程是不可见的，因为修改后的`count`的值还没有被刷新回主存中去。

![Visibility Problem](./Images/visibility_problem.png)

To solve this problem you can use Java's volatile keyword. 

<br>


#### Race Conditions
> 如果两个或者更多的线程共享一个对象，多个线程在这个共享对象上更新变量，就可能发生race conditions。

If thread A reads the variable count of a shared object into its CPU cache. Thread B does the same, but into a different CPU cache. Now thread A adds one to count, and thread B does the same. Now value has been incremented two times, once in each CPU cache. 

If these increments had been carried out sequentially, the variable count would be been incremented twice and had the original value + 2 written back to main memory. 

However, the two increments have been carried out concurrently without proper synchronization. Regardless of which of thread A and B that writes its updated version of count back to main memory, the updated value will only be 1 higher than the original value, despite the two increments. 

![Race Problem](./Images/race_problem.png)

To solve this problem you can use Java synchronized block. 

<br></br>



### ThreadLocal
------------
ThreadLocal is used to create thread-local variables. We know that all threads of an `Object` share its variables, so if the variable is not thread safe, we can use synchronization but if we want to avoid synchronization, we can use ThreadLocal variables.

Every thread has its own ThreadLocal variable and they can use its `get()` and `set() to get the default value or change its value local to Thread. ThreadLocal instances are typically private static fields in classes that wish to associate state with a thread.
        
&#12288;&#12288;ThreadLocal是一种以空间换时间的做法，在每个Thread里面维护了一个以开地址法实现的`ThreadLocal.ThreadLocalMap`，把数据进行隔离，数据不共享。

<br></br>



### Difference between Java Heap Space and Stack Memory
------------
* Whenever an object is created, it’s always stored in the Heap space and stack memory contains the reference to it. Stack memory only contains local primitive variables and reference variables to objects in heap space.
* Memory management in stack is done in LIFO manner whereas it’s more complex in Heap memory because it’s used globally. Heap memory is divided into Young-Generation, Old-Generation etc.
* Stack memory is short-lived whereas heap memory lives from the start till the end of application execution.
* We can use -Xms and -Xmx JVM option to define the startup size and maximum size of heap memory. We can use -Xss to define the stack memory size.
* When stack memory is full, Java runtime throws java.lang.StackOverFlowError whereas if heap memory is full, it throws java.lang.OutOfMemoryError: Java Heap Space error.
* Stack memory size is very less when compared to Heap memory. Because of simplicity in memory allocation (LIFO), stack memory is very fast when compared to heap memory.