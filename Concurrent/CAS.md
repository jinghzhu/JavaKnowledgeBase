# CAS (Compare and Swap)

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
     valueOffset = unsafe.objectFieldOffset
         (AtomicInteger.class.getDeclaredField("value"));
   } catch (Exception ex) { throw new Error(ex); }
 }
```

&#12288;&#12288;Java无法直接访问底层操作系统，而是通过native来访问。类Unsafe提供了硬件级别的原子操作。


## 2.ABA问题

&#12288;&#12288;如果一个值原来是A，变成了B，又变成了A，那么使用CAS进行检查时会发现它的值没有发生变化，但是实际上却变化了。
&#12288;&#12288;解决思路就是使用版本号。在变量前面追加上版本号，每次变量更新的时候把版本号加一，那么A－B－A 就会变成1A-2B－3A。
&#12288;&#12288;从Java1.5开始`atomic`包里提供了一个类`AtomicStampedReference`来解决ABA问题。这个类的`compareAndSet`方法作用是首先检查当前引用是否等于预期引用，并且当前标志是否等于预期标志，如果全部相等，则以原子方式将该引用和该标志的值设置为给定的更新值。


## 3. 循环时间长开销大

&#12288;&#12288;自旋CAS如果长时间不成功，会给CPU带来非常大开销。如果JVM能支持处理器提供的pause指令那么效率会有一定的提升.pause指令有两个作用，第一它可以延迟流水线执行指令（de-pipeline）,使CPU不会消耗过多的执行资源，延迟的时间取决于具体实现的版本 。第二它可以避免在退出循环的时因内存顺序冲突（memory order violation）而引起CPU流水线清空（CPU pipeline flush），提高CPU的执行效率。


## 4. 只能保证一个共享变量的原子操作

&#12288;&#12288;当对一个共享变量执行操作时，可使用循环CAS的方式来保证原子操作，但是对多个共享变量操作时，循环CAS就无法保证操作的原子性，这个时候就可以用锁，或者把多个共享变量合并成一个共享变量来操作。
&#12288;&#12288;比如有两个共享变量i＝2,j=a，合并一下ij=2a，然后用CAS来操作ij。从Java1.5开始提供了`AtomicReference`类来保证引用对象之间的原子性。 

## 5. 本地延迟
    
![CPU Bus](./Images/cpu_bus.png)

&#12288;&#12288;所有CPU会共享一条系统总线（BUS），靠此总线连接主存。每个核都有自己的一级缓存。

&#12288;&#12288;Core1和Core2可能会同时把主存中某个位置的值Load到自己的L1 Cache中，当Core1在自己的L1 Cache中修改这个位置的值时，会通过总线，使Core2中L1 Cache对应的值“失效”，而Core2一旦发现自己L1 Cache中的值失效（称为Cache命中缺失）则会通过总线从内存中加载该地址最新的值，大家通过总线的来回通信称为“Cache一致性流量”。

&#12288;&#12288;如果Cache一致性流量过大，总线将成为瓶颈。而当Core1和Core2中的值再次一致时，称为“Cache一致性”，从这个层面来说，锁设计的终极目标便是减少Cache一致性流量。 
        而CAS恰好会导致Cache一致性流量，如果有很多线程都共享同一个对象，当某个Core CAS成功时必然会引起总线风暴，这就是所谓的本地延迟，本质上偏向锁就是为了消除CAS，降低Cache一致性流量。