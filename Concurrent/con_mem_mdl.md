# <center>Concurrent Memory Model</center>

<br></br>



## 内存模型内部原理
----------------
JVM中存在一个主内存，所有变量、实例和实例的字段都存储在主内存中。对于所有线程是共享的（相当于黑板，其他人都可以看到）。每个线程都有自己的工作内存，工作内存中保存主存中变量的拷贝（相当于自己笔记本，只能自己看到）。工作内存由缓存和堆栈组成，缓存保存主存中变量的copy，堆栈保存的是线程局部变量。

所有原始类型本地变量都存放在线程栈上，对其它线程不可见。一个线程可能向另一个线程传递一个原始类型变量的拷贝，但是它不能共享这个原始类型变量自身。

堆上包含创建的所有对象，无论是哪一个对象创建的，包括原始类型对象版本。如果对象被创建然后赋值给一个局部变量，或用来作为另一个对象成员变量，这个对象任然放在堆上。

<p align="center">
  <img src="./Images/mem_model1.png"/>
</p>

线程对所有变量操作都在工作内存进行，线程间无法直接互相访问工作内存，变量值变化的传递需主存完成。在JMM中通过并发线程修改的变量值，须通过线程变量同步到主存后，其他线程才能访问到。

由于JVM可对代码进行调优，也就改变某些运行步骤次序，那么每次线程调用变量时是直接取自己的工作存储器中的值还是先从主存储器复制再取是没有保证的。同样，线程改变变量值后，是否马上写回到主存储器上也是不可保证的 。 

还好有`synchronized`和`volatile`： 
* 多个线程共有的字段用`synchronized`或`volatile`保护. 
* `synchronized`负责线程间互斥。即同时只有一个线程可以执行`synchronized`中的代码. 

synchronized还有另外的作用：
* 线程进入`synchronized`块前，把工作存内存内容映射到主内存，然后把工作内存清空再从主存储器上拷贝最新值。
* 线程退出`synchronized`块时，把工作内存值映射到主内存。此时并不清空工作内存。
* `volatile`负责线程中的变量与主存储区同步，但不负责每个线程之间的同步。

<br></br>



## 各种类型变量存放位置
-------------
* 原始类型（primitive type）：总在线程栈上。
* 指向一个对象的引用：引用（这个本地变量）在线程栈上，但对象本身在堆上。
* 对象可能包含方法，这些方法可能包含本地变量。这些本地变量在线程栈上，即使这些方法所属的对象存放在堆上。
* 一个对象的成员变量可能随着这个对象自身存放在堆上，不管这个成员变量是原始类型还是引用类型。
* 静态成员变量跟随着类一起也存放在堆上。

<br></br>



## Java Memory Model And Hardware Memory Architecture
----------------
硬件内存架构没有区分线程栈和堆。对于硬件，所有的线程栈和堆都分布在主内中。部分线程栈和堆可能有时会出现在CPU缓存中和CPU内部寄存器中。

![JVM & Hardware](./Images/jvm_hardware.png)

当对象和变量被存放在计算机中各种不同区域时，会出现一些问题：
1. 线程对共享变量修改的可见性。
2. 当读，写和检查共享变量时出现race conditions。

<br>


### Visibility of Shared Objects
跑在左边CPU的线程拷贝这个共享对象到它的CPU缓存中，然后将`count`变量的值改为2。这个修改对跑在右边CPU上的其它线程是不可见的，因为修改后的`count`的值还没有被刷新回主存中去。

可以使用volatile去解决这个问题。

![Visibility Problem](./Images/visibility_problem.png)

<br>


### Race Conditions
如果两个或更多线程共享一个对象，多个线程在共享对象上更新变量，就可能发生race conditions。可以使用synchronized解决这个问题。

If thread A reads the variable count of a shared object into its CPU cache. Thread B does the same, but into a different CPU cache. Now thread A adds one to count, and thread B does the same. Now value has been incremented two times, once in each CPU cache. 

If these increments had been carried out sequentially, the variable count would be been incremented twice and had the original value + 2 written back to main memory. 

However, the two increments have been carried out concurrently without proper synchronization. Regardless of which of thread A and B that writes its updated version of count back to main memory, the updated value will only be 1 higher than the original value, despite the two increments. 

![Race Problem](./Images/race_problem.png) 

<br></br>
