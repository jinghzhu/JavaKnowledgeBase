# <center>Concurrent Memory Model</center>



## 1. 内存模型内部原理
&#12288;&#12288;Java内存模型把JVM内部划分为线程栈和堆。每个运行在JVM里的线程都拥有自己的线程栈。线程栈包含了线程调用的方法当前执行点相关的信息。一个线程仅能访问自己的线程栈。一个线程创建的本地变量对其它线程不可见，仅自己可见。即使两个线程执行同样的代码，这两个线程任然在在自己的线程栈中的代码来创建本地变量。因此，每个线程拥有每个本地变量的独有版本。

&#12288;&#12288;所有原始类型的本地变量都存放在线程栈上，对其它线程不可见。一个线程可能向另一个线程传递一个原始类型变量的拷贝，但是它不能共享这个原始类型变量自身。

&#12288;&#12288;堆上包含创建的所有对象，无论是哪一个对象创建的。这包括原始类型的对象版本。如果一个对象被创建然后赋值给一个局部变量，或者用来作为另一个对象的成员变量，这个对象任然是存放在堆上。

<p align="center">
  <img src="./Images/mem_model1.png"/>
</p>

<br>

Difference between Java Heap Space and Stack Memory
* Whenever an object is created, it’s always stored in the Heap space and stack memory contains the reference to it. Stack memory only contains local primitive variables and reference variables to objects in heap space.
* Memory management in stack is done in LIFO manner whereas it’s more complex in Heap memory because it’s used globally. Heap memory is divided into Young-Generation, Old-Generation etc.
* Stack memory is short-lived whereas heap memory lives from the start till the end of application execution.
* We can use -Xms and -Xmx JVM option to define the startup size and maximum size of heap memory. We can use -Xss to define the stack memory size.
* When stack memory is full, Java runtime throws java.lang.StackOverFlowError whereas if heap memory is full, it throws java.lang.OutOfMemoryError: Java Heap Space error.
* Stack memory size is very less when compared to Heap memory. Because of simplicity in memory allocation (LIFO), stack memory is very fast when compared to heap memory.

<br></br>



## 2. 各种类型变量存放位置

* 原始类型（primitive type），它总是“呆在”线程栈上。

* 指向一个对象的引用。引用（这个本地变量）存放在线程栈上，但是对象本身存放在堆上。（A local variable may also be a reference to an object. In that case the reference (the local variable) is stored on the thread stack, but the object itself if stored on the heap.）

* 一个对象可能包含方法，这些方法可能包含本地变量。这些本地变量任然存放在线程栈上，即使这些方法所属的对象存放在堆上。

* 一个对象的成员变量可能随着这个对象自身存放在堆上，不管这个成员变量是原始类型还是引用类型。（An object's member variables are stored on the heap along with the object itself. That is true both when the member variable is of a primitive type, and if it is a reference to an object. ）

* 静态成员变量跟随着类一起也存放在堆上。（Static class variables are also stored on the heap along with the class definition. ）

<br></br>



## 3. Java Memory Model And Hardware Memory Architecture

&#12288;&#12288;Java内存模型与硬件内存架构之间存在差异。硬件内存架构没有区分线程栈和堆。对于硬件，所有的线程栈和堆都分布在主内中。部分线程栈和堆可能有时候会出现在CPU缓存中和CPU内部的寄存器中。

![JVM & Hardware](./Images/jvm_hardware.png)

&#12288;&#12288;当对象和变量被存放在计算机中各种不同的内存区域中时，就会出现一些问题：

1. 线程对共享变量修改的可见性
2. 当读，写和检查共享变量时出现race conditions

<br>


### 3.1 Visibility of Shared Objects

The shared object is initially stored in main memory. A thread running on CPU one then reads the shared object into its CPU cache. There it makes a change to the shared object. As long as the CPU cache has not been flushed back to main memory, the changed version of the shared object is not visible to threads running on other CPUs. This way each thread may end up with its own copy of the shared object, each copy sitting in a different CPU cache. 

&#12288;&#12288;跑在左边CPU的线程拷贝这个共享对象到它的CPU缓存中，然后将`count`变量的值改为2。这个修改对跑在右边CPU上的其它线程是不可见的，因为修改后的`count`的值还没有被刷新回主存中去。

![Visibility Problem](./Images/visibility_problem.png)

To solve this problem you can use Java's volatile keyword. 

<br>

### 3.2 Race Conditions

> 如果两个或者更多的线程共享一个对象，多个线程在这个共享对象上更新变量，就可能发生race conditions。

If thread A reads the variable count of a shared object into its CPU cache. Thread B does the same, but into a different CPU cache. Now thread A adds one to count, and thread B does the same. Now value has been incremented two times, once in each CPU cache. 

If these increments had been carried out sequentially, the variable count would be been incremented twice and had the original value + 2 written back to main memory. 

However, the two increments have been carried out concurrently without proper synchronization. Regardless of which of thread A and B that writes its updated version of count back to main memory, the updated value will only be 1 higher than the original value, despite the two increments. 

![Race Problem](./Images/race_problem.png)

To solve this problem you can use Java synchronized block. 
