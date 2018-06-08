# <center>Memory Model</center>

<br></br>



## Why
----
每个CPU都有自己的缓存，但不能实时和内存信息交换，且分在不同CPU执行的不同线程对同一个变量缓存值不同。

用`volatile`修饰变量可解决问题，其原理是使用内存屏障。内存屏障是硬件层概念，不同硬件实现内存屏障不一样。Java屏蔽差异，统一由JVM生成内存屏障指令。

<br></br>



## What
----
硬件层内存屏障分两种：
1. Load Barrier 读屏障
2. Store Barrier 写屏障

内存屏障有两个作用：
1. 阻止屏障两侧指令重排序；
2. 强制把写缓冲区/高速缓存中脏数据写回主内存，让缓存中相应数据失效。

    Load Barrier在指令前插入屏障，让高速缓存中数据失效，强制从主内存加载数据；
    Store Barrier在指令后插入屏障，让写入缓存中最新数据更新写入主内存，让其他线程可见。

<br></br>



## Java内存屏障
----
Java内存屏障有四种：
1. LoadLoad

    对语句`Load1; LoadLoad; Load2`，在`Load2`及后续读取操作要读取的数据访问前，保证`Load1`要读取的数据被读取完毕。

2. StoreStore

    对语句`Store1; StoreStore; Store2`，在`Store2`及后续写入操作执行前，保证`Store1`写入操作对其它处理器可见。

3. LoadStore

    对语句`Load1; LoadStore; Store2`，在`Store2`及后续写入操作被刷出前，保证`Load1`要读取的数据被读取完毕。

4. StoreLoad

    对语句`Store1; StoreLoad; Load2`，在`Load2`及后续所有读取操作执行前，保证`Store1`写入对所有处理器可见。它的开销是最大的。在多数处理器，这个屏障是万能屏障，兼具其它三种屏障功能。

<br></br>



## volatile语义内存屏障
----
`volatile`内存屏障策略非常保守悲观且无安全感心态：
* 在每个`volatile`写操作前插入StoreStore屏障，在写操作后插入StoreLoad屏障；
* 在每个`volatile`读操作前插入LoadLoad屏障，在读操作后插入LoadStore屏障；

由于内存屏障作用，避免了`volatile`变量和其它指令重排序，使`volatile`表现出锁的特性。

<br></br>



## final语义内存屏障
----
对`final`域，编译器和CPU遵循两个排序规则：
1. 新建对象过程中，构造体中对`final`域初始化写入和对象赋值给其他引用变量，这两个操作不重排序；
2. 初次读包含`final`域对象引用和读取这个`final`域，这两个操作不能重排序。即先赋值引用，再调用`final`值。

总之，需保证一个对象所有`final`域写入完毕后才能引用和读取。这也是内存屏障起的作用：
1. 写`final`域：编译器写`final`域完毕，构造体结束前，插入StoreStore屏障，保证前面对`final`写入对其他线程可见，并阻止重排序。
2. 读`final`域：在上述规则2中，两步操作不能重排序机理是在读`final`域前插入了LoadLoad屏障。
