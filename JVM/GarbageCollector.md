# <center>Garbage Collector</center>

| GC算法    |   优点 | 缺点  | 存活对象移动 | 内存碎片 | 适用场景 |
| :--------: | :--------| :-- | :---: | :--: | :-- |
| 引用计数  | 简单 | 不能解决循环引用 | | | |
| 标记-清除 | 无需额外空间 |  两次扫描，耗时严重| N | Y | 旧生代 |
| 复制 | 没有标记和清除 | 需额外空间 | Y | N | 新生代 |
| 标记整理 | 无内存碎片 | 有移动对象的成本  | Y | N | 旧生代 |



## 1. GC收集算法 
### 1.1 标记-清除(Mark-Sweep)

&#12288;&#12288;步骤：
1. 标记阶段：根据可达性分析对不可达对象进行标记，即标记出所有需要被回收的对象
2. 清理阶段：标记完成后统一清理这些对象，即回收被标记的对象所占用的空间

&#12288;&#12288;缺点：
* 标记和清理的效率不高（因为垃圾对象比较少，大部分都不是垃圾）
* 产生大量内存碎片，导致后续需要为大对象分配空间时无法找到足够的空间而提前触发GC

&#12288;&#12288;适用场景：基于Mark-Sweep的GC多用于老年代。

&#12288;&#12288; ![Mark-Sweep](./Images/mark_sweep.png)

<br></br>


### 1.2 复制(Copying)

&#12288;&#12288;为了解决Mark-Sweep算法的缺陷，Copying算法就被提了出来。将堆内分成两个相同空间，从根(ThreadLocal的对象，静态对象)开始访问每一个关联的活跃对象，将空间A的活跃对象全部复制到空间B，然后一次性回收整个空间A。 因为只访问活跃对象，将所有活动对象复制走之后就清空整个空间，不用去访问死对象，所以实现简单，运行高效且不容易产生内存碎片，但对内存空间的使用做出了高昂的代价，因为能够使用的内存缩减到原来的一半。

&#12288;&#12288;显然，Copying算法的效率跟存活对象的数目多少有很大的关系，如果存活对象很多，那么Copying算法的效率将会大大降低。

&#12288;&#12288; ![Copying](./Images/copying.png)

<br></br>


### 1.3 标记-压缩(Mark-Compact)
&#12288;&#12288;步骤：在完成标记之后，它不是直接清理可回收对象，而是将存活对象都向一端移动，然后清理掉端边界以外的内存
1. 在标记好待回收对象后，将存活的对象移至一端
2. 然后对剩余的部分进行回收

&#12288;&#12288;优点：
* 可以解决内存碎片的问题

&#12288;&#12288;适用场景：基于Mark-Compact的GC多用于老年代

&#12288;&#12288; ![Mark-Compact](./Images/mark_compact.png)

<br></br>


### 1.4 标记-整理-压缩(Mark-Sweep-Compact)

&#12288;&#12288;为了解决Copying算法缺陷，提出了Mark-Sweep-Compact算法。综合了上述两者的做法和优点，先标记活跃对象，在完成标记之后，不直接清理可回收对象，而是将存活对象向一端移动，然后清理掉端边界以外的内，将其合并成较大的内存块.

&#12288;&#12288; ![Mark-Sweep-Compact](./Images/mark_sweep_compact.png)

<br></br>


### 1.4 分代收集(Generational Collection)

 ![Generation1](./Images/generation1.png)

 ![Generation2](./Images/generation2.png)


#### 1.4.1 新生代(Young)

 ![Young Generation](./Images/young.png)

&#12288;&#12288;新建的对象都是用新生代分配内存，新生代又进一步分为Eden和Survivor区，Survivor由FromSpace和ToSpace组成。

&#12288;&#12288;Eden空间不足的时候，会把存活的对象转移到Survivor中。新生代通常存活时间较短，因此基于Copying算法来进行回收，就是在Eden和FromSpace或ToSpace之间copy。`-XX:NewRatio=`参数可以设置Young与Old的大小比例，`-XX:SurvivorRatio=`参数可以设置Eden与Survivor的比例.

&#12288;&#12288;两个存活区中始终有一个是空白的。GC时，Eden和其中一个非空存活区中还存活的对象根据其存活时间被复制到当前空白的存活区或年老世代中。经过这一次的复制之后，之前非空的存活区中包含了当前还存活的对象，而Eden和另一个存活区中的内容已经不再需要了，只需把这两个区域清空即可。下一次GC时，这两个存活区的角色就发生了交换。

&#12288;&#12288;新生代采用空闲指针的方式来控制GC触发，指针保持最后一个分配的对象在新生代区间的位置，当有新的对象要分配内存时，用于检查空间是否足够，不够就触发GC(minor GC)。当连续分配对象时，对象会逐渐从Eden到Survivor，最后到旧生代. 

> 年轻代的痛：由于对年轻代的复制收集，须停止所有线程。只能靠多CPU，多线程并发来提高收集速度。所以，暂停时间的瓶颈就落在了年轻代的复制算法上。  
> 
> Minor GC does trigger stop-the-world pauses, suspending the application threads. For most applications, the length of the pauses is negligible latency-wise. Major GC will clean the old gen and full GC will clean whole heap.

 ![Young GC](./Images/young_gc.png)

<br></br>


#### 1.4.2 年老代(Old)

&#12288;&#12288;`-XX:MaxTenuringThreshold=`设置熬过年轻代多少次GC后移入老人区，默认为0，熬过一次GC就转入。Java对象存活周期长命的对象放在老年代
1. 存放新生代中经历多次GC仍然存活的对象
2. 新建的对象也有可能直接在旧生代分配，取决于具体GC的实现
3. C频率相对降低，标记(mark)、清理(sweep)、压缩(compaction)算法的各种结合和优化

&#12288;&#12288;Old常见对象为比如Http请求中的Session对象、线程、Socket连接，这类对象跟业务直接挂钩，因此生命周期比较长。

<br></br>

#### 1.4.3 永久代(Permanent)

&#12288;&#12288;装载Class信息等基础数据，默认64M，如果是类很多的程序，需加大其设置`-XX:MaxPermSize=`，否则满了后会引起Major GC。Spring，Hibernate这类喜欢AOP动态生成类的框架需要更多的持久代内存。

<br></br>

#### 1.4.4 对象提升到老年代
1.对象分配

 ![Step 1](./Images/step1.png)

2.填充到Eden区

 ![Step 2](./Images/step2.png)

3.将Eden区中存活的对象（引用对象）拷贝到其中一个存活区

 ![Step 3](./Images/step3.png)

4.年龄计数器：在Eden中存活的对象其年龄初始=1，从其他存活区存活下来年龄+1

 ![Step 4](./Images/step4.png)

5.增加年龄计数器，图中To存活区有三个对象来自于From存活区，一个对象来自Eden

 ![Step 5](./Images/step5.png)

6.对象提升，这里假设年龄阈值=8，发生GC时，From存活区中=8的对象提升到老年代，其他存活对象移动到To存活区

 ![Step 6](./Images/step6.png)

7.总结下对象提升的过程：对象在新生代分配，每当熬过一次YGC，对象的年龄计数器+1，当达到阈值时仍然存活，提升到老年代

 ![Step 7](./Images/step7.png)

8.总结下GC过程：对象在新生代分配并填充，当新生代满时发生YGC，当对象在存活区熬过一定年龄，提升到老年代

 ![Step 8](./Images/step8.png)

<br></br>


## 2. Java Garbage Collection Types

1. Serial GC (`-XX:+UseSerialGC`): **uses mark-sweep-compact approach for young and old generations gc** i.e Minor and Major GC. Serial GC is useful in client-machines such as our simple stand alone applications and machines with smaller CPU. It is good for small applications with low memory footprint.

2. Parallel GC (`-XX:+UseParallelGC`): **Parallel GC is same as Serial GC except that is spawns N threads for young generation gc where N is the number of CPU cores in the system**. We can control the number of threads using `-XX:ParallelGCThreads = n`. Parallel Garbage Collector is also called throughput collector because it uses multiple CPUs to speed up the GC performance. Parallel GC uses single thread for Old Generation garbage collection.

3. Parallel Old GC (`-XX:+UseParallelOldGC`): *same as Parallel GC except that it uses multiple threads for both Young Generation and Old Generation garbage collection*.

4. Concurrent Mark Sweep (CMS) Collector (`-XX:+UseConcMarkSweepGC`): CMS Collector is also referred as concurrent low pause collector. **It does gc for Old generation. CMS collector tries to minimize the pauses due to gc by doing most of the gc work concurrently with the application threads. CMS collector on young generation uses the same algorithm as that of the parallel collector. This gc is suitable for responsive applications where we can’t afford longer pause times**. We can limit the number of threads in CMS collector using `-XX:ParallelCMSThreads = n`.

5. G1 Garbage Collector (`-XX:+UseG1GC`): The Garbage First or G1 garbage collector is **available from Java 7** and it’s long term goal is to **replace CMS collector**. The G1 collector is a parallel, concurrent, and incrementally compacting low-pause garbage collector. **G1 doesn’t work like other collectors and there is no concept of Young and Old generation space. It divides the heap space into multiple equal-sized heap regions. When gc is invoked, it first collects the region with lesser live data, hence “Garbage First”**.
        


## 3. System.gc(), finalize()
&#12288;&#12288;使用`System.gc()`可以请求Java的垃圾回收。调用`System.gc()`仅是一个请求。JVM并不是立即做gc，而只是对几个gc算法做了加权，使gc容易发生，或回收较多而已。

&#12288;&#12288;之所以用`finalize()`是存在着gc不能处理的特殊情况，例如：
* 由于在分配内存的时候采用了C语言的做法，而非JAVA的new做法。主要发生在native method中。此时除非调用`free()`否则这些内存将不会释放，造成内存泄漏。但由于`free()`在C/C++中的函数，所以`finalize()`中可以用本地方法来调用它。

* 打开的文件资源，这些资源不属于垃圾回收器的回收范围。
         
&#12288;&#12288;每个对象只能调用`finalize()`一次。如果在`finalize()``执行时产生异常，则该对象仍可以被垃圾收集器收集。



## 4. 如何确定某个对象是“垃圾”？
### 4.1 引用计数法
&#12288;&#12288;每个对象都有一个引用计数器，当有对象引用它时，计数器+1；当引用失效时，计数器-1；计数器为0时就是不可能再被使用的。缺点是：
1. 引用和去引用伴随加法和减法，影响性能
2. 对于循环引用的对象无法进行回收

<br></br>


### 4.2 可达性分析法（根搜索）
&#12288;&#12288;为了解决循环引用，Java采取了**可达性分析法**。通过GC Roots对象作为起点进行搜索，如果在GC Roots和一个对象之间没有可达路径，则称该对象是不可达的。被判定为不可达的对象不一定就会成为可回收对象，除非至少经历两次标记过程，如果在这两次标记过程中仍然没有逃脱成为可回收对象的可能性。

<br></br>


### 4.3 GC Roots in Java:
There are many kinds of GC roots:
1. Stack Local - Java方法的local变量或参数。Local variables are kept alive by the stack of a thread. This is not a real object virtual reference and thus is not visible. For all intents and purposes, local variables are GC roots.

2. Thread - 活着的线程。Active Java threads are always considered live objects and are therefore GC roots. This is especially important for thread local variables.

3. Static variables are referenced by their classes. This fact makes them de facto GC roots. Classes themselves can be garbage-collected, which would remove all referenced static variables. This is of special importance when we use application servers, OSGi containers or class loaders in general. We will discuss the related problems in the Problem Patterns section.

4. JNI Local - JNI方法的local变量或参数

5. JNI Global - 全局JNI引用

6. Monitor Used - 用于同步的监控对象

7. Class - 由系统类加载器(system class loader)加载的对象，这些类是不能够被回收的，他们可以以静态字段的方式保存持有其它对象。通过用户自定义的类加载器加载的类，除非相应的java.lang.Class实例以其它的某种（或多种）方式成为roots，否则它们并不是roots，.

<br></br>


### 4.4 Example
&#12288;&#12288; ![引用循环](./Images/count_circle.png)

&#12288;&#12288;最后两句将`obj1`和`obj2`赋为`null`，即`obj1`和`obj2`指向的对象不可能再被访问。但由于它们互相引用对方，导致它们的引用计数都不为0，那么GC永远不会回收它们。

&#12288;&#12288;使用根搜索算法，则可以正常回收:

&#12288;&#12288; ![根搜索](./Images/root_example.png)

<br></br>



## 5. 减少GC开销的措施

* 不要显式调用`System.gc()`
&#12288;&#12288;此函数建议JVM进行Major GC,从而增加主GC的频率,也即增加了间歇性停顿的次数。

* 尽量减少临时对象的使用
&#12288;&#12288;临时对象在跳出函数调用后,会成为垃圾,少用临时变量就相当于减少了垃圾的产生,从而延长了出现上述第二个触发条件出现的时间,减少了Major GC的机会。

* 对象不用时最好显式置为Null

&#12288;&#12288;为Null的对象都会被作为垃圾处理,有利于GC收集器判定垃圾,从而提高了GC的效率。

* 尽量使用StringBuffer,而不用String来累加字符串

&#12288;&#12288;由于String是final对象,累加String对象时,并非在一个String对象中扩增,而是重新创建新的String对象。

* 能用基本类型如Int,Long,就不用Integer,Long对象

&#12288;&#12288;基本类型变量占用的内存资源比相应对象少得多,如果没有必要,最好使用基本变量。

* 尽量少用静态对象变量

&#12288;&#12288;静态变量属于全局变量,不会被GC回收,它们会一直占用内存。

* 分散对象创建或删除的时间

&#12288;&#12288;集中在短时间内大量创建新对象,会导致突然需要大量内存。JVM在面临这种情况时,只能进行Major GC,以回收内存或整合内存碎片,从而增加主GC的频率。集中删除对象,道理也是一样的。它使得突然出现了大量的垃圾对象,空闲空间必然减少,从而大大增加了下一次创建新对象时强制主GC的机会。

<br></br>



## 6. 串行、并行、并发GC
&#12288;&#12288;串行和并行指的是垃圾收集器工作时暂停应用程序（Stop the World），使用单核CPU（串行）还是多核CPU（并行）。

* 串行（Serial）：使用单核CPU串行地进行垃圾收集。
* 并行（Parallel）：使用多CPU并行地进行垃圾收集，并行是GC线程有多个，但在运行GC线程时，用户线程是阻塞的。
* 并发（Concurrent）：垃圾收集时不会暂停应用程序线程，大部分阶段用户线程和GC线程都在运行。
      
| 概念   |  STW | 单线程／多线程 | 
| :---: | :----: | :--------: | 
| 串行  | Y | GC单线程 | 
| 并行 | Y |  GC多线程 |
| 并发 | N | GC和用户线程是多线程 |

&#12288;&#12288; ![Serial, Parallel, and Concurrent](./Images/serial_parallel_concurrent.png)

<br></br>



## 7. 增量式GC和普通GC的区别
&#12288;&#12288;GC由一个或一组进程来实现的，它本身也和用户程序一样占用heap空间，运行时也占用CPU。当GC运行时，应用程序停止运行。

&#12288;&#12288;增量式GC是指把一个长时间的中断，划分为很多个小的中断，减少GC对用户程序的影响。虽然增量式GC整体性能上不如普通GC的效率高，但是能够减少程序的最长停顿时间。增量式GC采用TrainGC算法，基本想法是：将堆中的所有对象按照创建和使用情况进行分组（分层），将使用频繁高和具有相关性的对象放在一队中，随着程序的运行，不断对组进行调整。当GC运行时，它总是先回收最老的（最近很少访问的）的对象，如果整组都为可回收对象，GC将整组回收。这样，每次GC运行只回收一定比例的不可达对象，保证程序的顺畅运行。

