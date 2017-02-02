# <center>CAS (Compare and Swap)</center>



## 1. What is CAS?

* 是一种非阻塞的乐观加锁策略。 
* CAS通过调用JNI的代码实现的。
* `java.util.concurrent.atomic`包下的原子操作类都是基于CAS实现的。

```java
private static final Unsafe unsafe = Unsafe.getUnsafe();
private static final long valueOffset;
private volatile int value;
 
static {
  try {
     valueOffset = unsafe.objectFieldOffset(AtomicInteger.class.getDeclaredField("value"));
  } catch (Exception ex) { throw new Error(ex); }
}
```

&#12288;&#12288;Java无法直接访问底层操作系统，而是通过native来访问。类Unsafe提供了硬件级别的原子操作。

<br></br>



## 2. 原理
&#12288;&#12288;CAS通过调用JNI的代码实现的，compareAndSwapInt借助C来调用CPU底层指令实现的。下面是sun.misc.Unsafe类的compareAndSwapInt()方法的源代码：



&#12288;&#12288;组合这两个条件，意味着为了同时实现volatile读和volatile写的内存语义下面分析在intel x86处理器中，CAS是如何同时具有volatile读和volatile写的内存语义的。

``` java
public final native boolean compareAndSwapInt(Object o, long offset, int expected, int x)
```

&#12288;&#12288;这是个本地方法调用，在openjdk中依次调用的c++代码为：unsafe.cpp，atomic.cpp和atomicwindowsx86.inline.hpp。下面是对应于intel x86处理器的源代码的片段：

```
// Adding a lock prefix to an instruction on MP machine
// VC++ doesn't like the lock prefix to be on a single line
// so we can't insert a label after the lock prefix.
// By emitting a lock prefix, we can define a label after it.
#define LOCK_IF_MP(mp) __asm cmp mp, 0  \
                       __asm je L0      \
                       __asm _emit 0xF0 \
                       __asm L0:

inline jint     Atomic::cmpxchg    (jint     exchange_value, volatile jint*     dest, jint     compare_value) {
  // alternative for InterlockedCompareExchange
  int mp = os::is_MP();
  __asm {
    mov edx, dest
    mov ecx, exchange_value
    mov eax, compare_value
    LOCK_IF_MP(mp)
    cmpxchg dword ptr [edx], ecx
  }
}
```

&#12288;&#12288;程序会根据当前处理器的类型来决定是否为`cmpxchg`指令添加lock前缀。如果在多处理器上运行，就为`cmpxchg`指令加上lock前缀（`lock cmpxchg`）。反之，就省略lock前缀（单处理器自身会维护单处理器内的顺序一致性，不需要lock前缀提供的内存屏障效果）。

&#12288;&#12288;intel的手册对lock前缀的说明如下：

1. 确保对内存的读-改-写操作原子执行。在Pentium及Pentium之前的处理器中，带有lock前缀的指令在执行期间会锁住总线。从Pentium 4，Intel Xeon及P6处理器开始，intel做了一个优化：如果要访问的内存区域（area of memory）在lock前缀指令执行期间已经在处理器内部的缓存中被锁定（即包含该内存区域的缓存行当前处于独占或以修改状态），并且该内存区域被完全包含在单个缓存行（cache line）中，那么处理器将直接执行该指令。由于在指令执行期间该缓存行会一直被锁定，其它处理器无法读/写该指令要访问的内存区域，因此能保证指令执行的原子性。这个操作过程叫做缓存锁定（cache locking）。但当多处理器之间的竞争程度很高或者指令访问的内存地址未对齐时，仍然会锁住总线。
2. 禁止该指令与之前和之后的读和写指令重排序。
3. 把写缓冲区中的所有数据刷新到内存中。

&#12288;&#12288;第2点和第3点所具有的内存屏障效果，足以同时实现volatile读和volatile写的内存语义，，编译器不能对CAS与CAS前面和后面的任意内存操作重排序。

> volatile读和volatile写内存语义:
> 1. 编译器不会对volatile读与volatile读后面的任意内存操作重排序；
> 2. 编译器不会对volatile写与volatile写前面的任意内存操作重排序。

&#12288;&#12288;备注知识，关于CPU的锁有如下3种：

1. 处理器自动保证基本内存操作的原子性

    首先处理器会保证基本内存操作的原子性。处理器保证从系统内存当中读取或者写入一个字节是原子的，意思是当一个处理器读取一个字节时，其他处理器不能访问这个字节的内存地址。

2. 使用总线锁保证原子性

    第一个机制是通过总线锁保证原子性。如果多个处理器同时对共享变量进行读改写（比如`i++`），那么共享变量操作完之后共享变量的值会和期望的不一致，处理器使用总线锁就是解决这个问题的。所谓总线锁就是使用处理器提供的一个LOCK＃信号，当一个处理器在总线上输出此信号时，其他处理器的请求将被阻塞住，那么该处理器可以独占使用共享内存。

3. 使用缓存锁保证原子性

    第二个机制是通过缓存锁定保证原子性。在同一时刻只需保证对某个内存地址的操作是原子性即可，但总线锁定把CPU和内存之间通信锁住了，这使得锁定期间，其他处理器不能操作其他内存地址的数 据，所以总线锁定开销较大，最近的处理器在某些场合下使用缓存锁定代替总线锁定来进行优化。

    频繁使用的内存会缓存在处理器的L1，L2 和L3里，那么原子操作可以在处理器内部缓存中进行，并不需要总线锁。所谓“缓存锁定”就是如果缓存在处理器缓存行中内存区域在LOCK操作期间被锁定，当它执行锁操作回写内存时，处理器不在总线上声言LOCK＃信号，而是修改内部的内存地址，并允许它的缓存一致性机制来保证操作的原子性。因为缓存一致性会阻止同时修改被两个以上处理器缓存的内存区域数据，当其他处理器回写已被锁定的缓存行的数据时会起缓存行无效。

    但是有两种情况处理器不会使用缓存锁定。第一种情况是：当操作的数据不能被缓存在处理器内部，或操作的数据跨多个缓存行（cache line），则处理器会调用总线锁定。第二种情况是：有些处理器不支持缓存锁定。

<br></br>



## 2. CAS缺点
### 2.1 ABA问题
&#12288;&#12288;如果一个值原来是A，变成了B，又变成了A，那么使用CAS进行检查时会发现它的值没有发生变化，但是实际上却变化了。解决思路就是使用版本号。在变量前面追加上版本号，每次变量更新的时候把版本号加一，那么A－B－A会变成1A-2B－3A。

&#12288;&#12288;A-B-A解决方法是不再仅替换指向一个预期修改对象的指针，而是指针结合一个计数器，然后使用单个CAS操作来替换指针 + 计数器。在Java中，不能将一个引用和一个计数器归并在一起形成单个变量。不过Java 1.5提供了AtomicStampedReference 类，利用这个类可以使用一个CAS操作（`compareAndSet()`方法）自动替换一个引用和一个标记（stamp）。

<br>



## 2.2 循环时间长开销大

&#12288;&#12288;自旋CAS如果长时间不成功，会给CPU带来非常大开销。如果JVM能支持处理器提供的pause指令那么效率会有一定的提升.pause指令有两个作用，第一它可以延迟流水线执行指令（de-pipeline）,使CPU不会消耗过多的执行资源，延迟的时间取决于具体实现的版本 。第二它可以避免在退出循环的时因内存顺序冲突（memory order violation）而引起CPU流水线清空（CPU pipeline flush），提高CPU的执行效率。

<br>



## 2.3 只能保证一个共享变量的原子操作

&#12288;&#12288;当对一个共享变量执行操作时，可使用循环CAS的方式来保证原子操作，但是对多个共享变量操作时，循环CAS就无法保证操作的原子性，这个时候就可以用锁，或者把多个共享变量合并成一个共享变量来操作。
&#12288;&#12288;比如有两个共享变量i＝2,j=a，合并一下ij=2a，然后用CAS来操作ij。从Java1.5开始提供了`AtomicReference`类来保证引用对象之间的原子性。 

<br>



## 2.4 本地延迟
    
![CPU Bus](./Images/cpu_bus.png)

&#12288;&#12288;所有CPU会共享一条系统总线（BUS），靠此总线连接主存。每个核都有自己的一级缓存。

&#12288;&#12288;Core1和Core2可能会同时把主存中某个位置的值Load到自己的L1 Cache中，当Core1在自己的L1 Cache中修改这个位置的值时，会通过总线，使Core2中L1 Cache对应的值“失效”，而Core2一旦发现自己L1 Cache中的值失效（称为Cache命中缺失）则会通过总线从内存中加载该地址最新的值，大家通过总线的来回通信称为“Cache一致性流量”。

&#12288;&#12288;如果Cache一致性流量过大，总线将成为瓶颈。而当Core1和Core2中的值再次一致时，称为“Cache一致性”，从这个层面来说，锁设计的终极目标便是减少Cache一致性流量。而CAS恰好会导致Cache一致性流量，如果有很多线程都共享同一个对象，当某个Core CAS成功时必然会引起总线风暴，这就是所谓的本地延迟，本质上偏向锁就是为了消除CAS，降低Cache一致性流量。

<br>



## 3. concurrent包的实现

&#12288;&#12288;由于CAS同时具有volatile读和volatile写的内存语义，因此线程间通信有下面四种方式：

1. A线程写volatile变量，随后B线程读这个volatile变量。
2. A线程写volatile变量，随后B线程用CAS更新这个volatile变量。
3. A线程用CAS更新一个volatile变量，随后B线程用CAS更新这个volatile变量。
4. A线程用CAS更新一个volatile变量，随后B线程读这个volatile变量。

&#12288;&#12288;Java的CAS会使用处理器上提供的高效机器级别原子指令，这些原子指令以原子方式对内存执行读-改-写操作，这是在多处理器中实现同步的关键（从本质上来说，能够支持原子性读-改-写指令的计算机器，是顺序计算图灵机的异步等价机器，因此任何现代的多处理器都会去支持某种能对内存执行原子性读-改-写操作的原子指令）。同时，volatile变量的读/写和CAS可以实现线程之间的通信。把这些特性整合在一起，就形成了整个concurrent包得以实现的基石。

&#12288;&#12288;如果分析concurrent包的源代码实现，会发现一个通用化的实现模式：

* 首先，声明共享变量为volatile；
* 然后，使用CAS的原子条件更新来实现线程之间的同步；
* 同时，配合以volatile的读/写和CAS所具有的volatile读和写的内存语义来实现线程之间的通信。

&#12288;&#12288;AQS，非阻塞数据结构和原子变量类（java.util.concurrent.atomic包中的类），这些concurrent包中的基础类都是使用这种模式来实现的，而concurrent包中的高层类又是依赖于这些基础类来实现的：

![concurrent包的实现示意图](./Images/overall.png)