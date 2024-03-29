# <center>Garbage Collector</center>



<br></br>

| GC算法    |   优点     | 缺点              | 存活对象移动 | 内存碎片 | 适用场景 |
| :------: | :--------  | :--------------- | :--------: | :-----: | :----- |
| 引用计数  | 简单        | 不能解决循环引用    |     N      |   Y     |        |
| 标记清除 | 无需额外空间  |  两次扫描，耗时严重 | N          | Y      | 旧生代   |
| 复制     | 没有标记和清除 | 需额外空间        | Y          | N      | 新生代   |
| 标记整理  | 无内存碎片   | 有移动对象的成本    | Y          | N      | 旧生代   |

<br></br>



## 引用计数
----
在每个对象内部维护一个整数值引用计数。对象被引用时加一，不被引用时减一。为0时，自动销毁对象。

主要用在C++标准库的*std::shared_ptr*、微软的COM、Objective-C和PHP中。

优点：
1. 渐进式。内存管理与程序执行在一起，将GC代价分散到整个程序。不像标记-清扫需STW。
2. 易于实现。
3. 内存单元能很快被回收。相比于其他GC，堆被耗尽或达到某个阈值才进行GC。

缺点：
1. 循环引用问题。
2. 每次对象赋值都要将引用计数加一，增加消耗。
3. 单元池free list实现不是cache-friendly。导致频繁cache miss，降低效率。

<br></br>



## 标记清除 Mark-Sweep
----
内存单元不会在变成垃圾立刻回收，而是保持不可达状态，直到到达某个阈值或固定时间长度。这时候系统会STW，转而执行GC。步骤：
1. 标记：根据可达性分析对不可达对象进行标记
2. 清理：回收被标记的对象所占用的空间

优点：实现简单

缺点：
1. 标记和清理的效率不高。因为垃圾对象比较少，大部分都不是垃圾。并且对象增多时，递归遍历整个对象树消耗很多时间。
2. 产生内存碎片，导致后续需要为大对象分配空间时无法找到足够的空间而提前触发GC。
3. STW。因为在标记时须暂停程序，否则其他线程代码可能改变对象状态，从而可能把不该回收的对象当做垃圾。

golang 1.5以前使用这个算法。

适用场景：老年代。

 ![Mark-Sweep](./Images/mark_sweep.png)

<br></br>



## 复制 Copying
-----
Copying算法是基于追踪的算法，为解决Mark-Sweep缺陷。

步骤：
1. 将堆内分成两个相同空间。
2. 从根开始访问每一个关联的活跃对象，将空间A的活跃对象全部复制到空间B。
3. 一次性回收整个空间A。

优点：
1. 所有存活数据结构都缩并地排列在Tospace底部，不存在内存碎片问题。
2. 获取新内存可简单通过递增自由空间指针实现。

缺点：
1. 耗时高。
2. 效率跟存活对象数目多少有关系。
3. 能够使用的内存缩减到原来的一半。

适用场景：新生代

 ![Copying](./Images/copying.png)

<br></br>



## 标记-整理-压缩 Mark-Sweep-Compact
----
Mark-Sweep-Compact算法为了解决Copying算法缺陷。

综合上述两者做法和优点，先标记活跃对象，完成标记之后，不直接清理可回收对象，而是将存活对象向一端移动。然后清理掉端边界以外内存，将其合并成较大内存块.

 ![Mark-Sweep-Compact](./Images/mark_sweep_compact.png)

<br></br>



## 三色标记
----
三色标记是Mark-Sweep改进，是并发的GC算法。

步骤：
1. 创建三个集合：白、灰、黑。
2. 将所有对象放入白色集合。
3. 从根节点遍历所有对象（注意不递归遍历），把遍历到的对象从白色集合放入灰色集合。
4. 遍历灰色集合，将灰色对象引用的对象从白色集合放入灰色集合。之后将此灰色对象放入黑色集合。
5. 重复4直到灰色集合无对象。
6. 通过write-barrier检测对象如果有变化，重复以上操作。
7. 收集所有白色对象（垃圾）。

<p align="center">
  <iframe height=380 width=500 src="./Images/gc1.gif">
</p>

优点：可实现*on-the-fly*，即程序运行同时进行收集，不需要暂停整个程序。

缺点：可能垃圾产生速度大于GC速度，导致程序垃圾越来越多无法被收集。

使用这种算法的是Go 1.5及以后。

<p align="center">
  <img src="./Images/gc2.png" width = "400"/>
</p>

注意：
1. 首先从root开始遍历，root包括全局指针和goroutine栈上指针。
2. mark有两个过程：
    1. 从root开始遍历，标记为灰色。遍历灰色队列。
    2. re-scan全局指针和栈。因为mark和用户程序并行，所以在过程1时可能会新对象分配。这时需通过写屏障（write barrier）记录。re-scan再完成检查一下。
3. STW有两个过程：
    1. GC将要开始时，主要是一些准备工作，比如enable write barrier。
    2. 是re-scan过程。如果这时候没有STW，那么mark将无休止。
4. 针对上图各阶段对应GCPhase如下：
    * Off: _GCoff
    * Stack scan ~ Mark: _GCmark
    * Mark termination: _GCmarktermination

<br>


### 写屏障 Write Barrier
GC中的Write Barrier可理解为编译器在写操作时特意插入一段代码。之所以需要Write Barrier，因为对于和用户程序并发运行的GC，用户程序会一直修改内存，所以需记录。

Golang 1.7前Write Barrier使用的经典*Dijkstra-style insertion write barrier [Dijkstra ‘78]*。STW 主要耗时在stack re-scan过程。1.8后采用混合Write Barrier方式 （Yuasa-style deletion write barrier [Yuasa ‘90] 和 Dijkstra-style insertion write barrier [Dijkstra ‘78]）避免re-scan。

<br></br>



## 分代收集 Generational Collection
----
主要用于JVM和.Net。

 ![Generation1](./Images/generation1.png)

 ![Generation2](./Images/generation2.png)

<br>


### 新生代 Young
新建对象用新生代分配内存。新生代进分为Eden和Survivor区，Survivor由From和To组成。

Eden空间不足时，把存活对象转移到Survivor。新生代存活时间短，因此基于Copying算法进行回收，在Eden的From或To之间copy。

> `-XX:NewRatio=`参数可设置Young与Old大小比例，`-XX:SurvivorRatio=`参数可设置Eden与Survivor比例。

新生代采用空闲指针方式控制GC触发。指针保持最后一个分配的对象在新生代区间的位置，当有新对象要分配内存时，用于检查空间是否足够，不够就触发minor GC。当连续分配对象时，对象逐渐从Eden到Survivor。 

> 年轻代的痛：由于对年轻代的复制收集，须停止所有线程。只能靠多CPU，多线程并发来提高收集速度。所以，暂停时间的瓶颈就落在了年轻代的复制算法上。

 ![Young GC](./Images/young_gc.png)

<br>


### 年老代 Old
> `-XX:MaxTenuringThreshold=`设置熬过年轻代多少次GC后移入老人区。默认为0，熬过一次GC就转入。

对象存活周期长的对象放在老年代：
* 存放新生代中经历多次GC仍然存活的对象；
* 新建对象也可能直接在旧生代分配，取决于具体GC实现。

Old常见对象为比如Http请求中的Session对象、线程、Socket连接，这类对象跟业务直接挂钩，因此生命周期比较长。

<br>


### 永久代Permanent
装载Class信息等基础数据，默认64M。如果是类很多的程序，需加大其设置`-XX:MaxPermSize=`，否则满了后引起Major GC。Spring，Hibernate这类喜欢AOP动态生成类的框架需要更多的持久代内存。

<br>


### 对象提升到老年代
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

<br></br>



## GC Type
----
1. Serial GC (`-XX:+UseSerialGC`)

    **Use mark-sweep-compact for Young and Old Generations GC.** i.e Minor and Major GC. Serial GC is useful in client-machines such as our simple stand alone applications and machines with smaller CPU. It is good for small applications with low memory footprint.

2. Parallel GC (`-XX:+UseParallelGC`) 

    **Parallel GC is same as Serial GC except that is spawns N threads for young generation gc where N is the number of CPU cores in the system**. We can control the number of threads using `-XX:ParallelGCThreads = n`. Parallel GC uses single thread for Old Generation GC.

3. Parallel Old GC (`-XX:+UseParallelOldGC`)

    **Same as Parallel GC except that it uses multiple threads for both Young Generation and Old Generation GC**.

4. Concurrent Mark Sweep (CMS) Collector (`-XX:+UseConcMarkSweepGC`)

    CMS Collector is also referred as concurrent low pause collector. **It does GC for Old Generation. CMS collector tries to minimize pauses due to GC by doing most of GC works concurrently with application threads. CMS collector on Young Generation uses  same algorithm as of Parallel Collector. This GC is suitable for responsive applications where we can’t afford longer pause times**. We can limit the number of threads in CMS collector using `-XX:ParallelCMSThreads = n`.

5. G1 Garbage Collector (`-XX:+UseG1GC`)

    The Garbage First or G1 GC is **available from Java 7** and it’s long term goal is to **replace CMS collector**. The G1 collector is a parallel, concurrent, and incrementally compacting low-pause GC. **G1 doesn’t work like other collectors and there is no concept of Young and Old generation space. It divides heap space into multiple equal-sized heap regions. When GC is invoked, it first collects region with lesser live data, hence “Garbage First”**.
        
<br></br>



## 确定垃圾
----

![根搜索](./Images/root_example.png)

GC roots:
1. **Stack Local - Java方法local变量或参数**

    Local variables are kept alive by the stack of a thread. This is not a real object virtual reference and thus is not visible.

2. **Thread - 活着的线程**

    Active Java threads are always considered live objects and are therefore GC roots. This is especially important for thread local variables.

3. **Static variables** 

    Referenced by their classes. This fact makes them de facto GC roots. Classes themselves can be garbage-collected, which would remove all referenced static variables. 

4. **JNI Local - JNI方法的local变量或参数**

5. **Monitor Used - 用于同步的监控对象**

7. **Class - 由系统类加载器加载的对象**

JVM判定无用的类的条件：
1. 该类所有实例已被回收，堆中不存在该类任何示例。
2. 加载该类的ClassLoader已被回收。
3. 该类对应的`java.lang.Class`对象没有在任何地方被引用。

<br></br>



## Performance
----
1. **不要显式调用`System.gc()`。**

2. **减少临时对象使用。**

3. **对象不用时显式置为`null`。**

4. **使用`StringBuffer`，而不用`String`来累加字符串。**

5. **能用基本类型如`int`，就不用`Integer`对象。**

    基本类型变量占用内存资源比相应对象少得多，如果没有必要，最好使用基本变量。

6. **少用静态对象变量。**

    静态变量属于全局变量，不会被GC回收，会一直占用内存。

7. **分散对象创建或删除的时间。**

<br></br>



## 串行、并行、并发GC
----
串行和并行指GC工作时STW，使用单核CPU（串行）还是多核CPU（并行）。

* 串行（Serial）：使用单核CPU串行地进行垃圾收集。
* 并行（Parallel）：使用多CPU并行地进行垃圾收集，并行是GC线程有多个，但在运行GC线程时，用户线程是阻塞的。
* 并发（Concurrent）：垃圾收集时不会暂停应用程序线程，大部分阶段用户线程和GC线程都在运行。
      
| 概念   |  STW | 单线程／多线程 | 
| :---: | :----: | :--------: | 
| 串行  | Y | GC单线程 | 
| 并行 | Y |  GC多线程 |
| 并发 | N | GC和用户线程是多线程 |

<br></br>



## 增量式GC和普通GC
----
GC由一个或一组进程来实现的，它本身也和用户程序一样占用heap空间，运行时也占用CPU。当GC运行时，应用程序停止运行。增量式GC是指把一个长时间的中断，划分为很多个小的中断，减少GC对用户程序的影响。虽然增量式GC整体性能上不如普通GC的效率高，但是能够减少程序的最长停顿时间。

增量式GC采用TrainGC算法，基本想法是将堆中的所有对象按照创建和使用情况进行分组（分层），将使用频繁高和具有相关性的对象放在一队中，随着程序的运行，不断对组进行调整。当GC运行时，它总是先回收最老的（最近很少访问的）的对象，如果整组都为可回收对象，GC将整组回收。这样，每次GC运行只回收一定比例的不可达对象，保证程序的顺畅运行。
