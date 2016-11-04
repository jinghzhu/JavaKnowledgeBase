# Synchronized


## 1. 原理
&#12288;&#12288;底层也是基于CAS操作的等待队列，把等待队列分为`ContentionList`和`EntryList`。但由JVM实现，不像ReentrantLock由上层类实现。是非公平悲观锁。


## 2.内部状态和队列

* Contention List：所有请求锁的线程将被首先放置到该竞争队列
* Entry List：Contention List中那些有资格成为候选人的线程被移到Entry List
* Wait Set：那些调用wait方法被阻塞的线程被放置到Wait Set
* OnDeck：任何时刻最多只能有一个线程正在竞争锁，该线程称为OnDeck
* Owner：获得锁的线程称为Owner
* !Owner：释放锁的线程

![Synchronized](./Images/synchronized.png)


## 3. 工作流程
* 新请求锁的线程将首先被加入到ContentionList中

1. `ContentionList`只是一个虚拟队列，原因在于`ContentionList`是由`Node`及其`next`指针逻辑构成，并不存在一个Queue的数据结构。
2. `ContentionList`是一个后进先出（LIFO）的队列，每次新加入Node时都会在队头进行，通过CAS改变第一个节点的的指针为新增节点，同时设置新增节点的next指向后续节点，而取得操作则发生在队尾。显然，该结构其实是个Lock-Free的队列。
3. 因为只有Owner线程才能从队尾取元素，也即线程出列操作无争用，当然也就避免了CAS的ABA问题。

* 当某个拥有锁的线程（Owner状态）调用unlock之后，如果发现EntryList为空则从ContentionList中移动线程到EntryList

1. `EntryList`与`ContentionList`逻辑上同属等待队列，`ContentionList`会被线程并发访问，为了降低对`ContentionList`队尾的争用，而建立`EntryList`。
2. Owner线程在unlock时会从`ContentionList`中迁移线程到`EntryList`，并会指定`EntryList`中的某个线程（一般为Head）为Ready（OnDeck）线程。
3. Owner线程并不是把锁传递给OnDeck线程，只是把竞争锁的权利交给OnDeck，OnDeck线程需要重新竞争锁。
4. OnDeck线程获得锁后即变为owner线程，无法获得锁则会依然留在`EntryList`中，考虑到公平性，在`EntryList`中的位置不发生变化（依然在队头）。如果Owner线程被wait方法阻塞，则转移到WaitSet队列；如果在某个时刻被notify/notifyAll唤醒，则再次转移到`EntryList`。

* 处于ContetionList、EntryList、WaitSet中的线程均处于阻塞状态，阻塞操作由操作系统完成。线程被阻塞后便进入内核（Linux）调度状态
        
&#12288;&#12288;缓解上述问题的办法便是自旋，其原理是：当发生争用时，若Owner线程能在很短的时间内释放锁，则那些正在争用线程可以稍微等一等（自旋），在Owner线程释放锁后，争用线程可能会立即得到锁，从而避免了系统阻塞 。

&#12288;&#12288;那`synchronized`实现何时使用了自旋锁？答案是在线程进入`ContentionList`时，也即第一步操作前。
 

## 4. 偏向锁
&#12288;&#12288;在JVM1.6中引入了偏向锁，偏向锁主要解决无竞争下的锁性能问题，首先看下无竞争下锁存在什么问题.

&#12288;&#12288;现在几乎所有的锁都是可重入的，也即已经获得锁的线程可以多次锁住/解锁监视对象。每次加锁/解锁都会涉及到一些CAS操作（比如对等待队列的CAS操作），CAS操作会延迟本地调用，因此偏向锁的想法是一旦线程第一次获得了监视对象，之后让监视对象“偏向”这个线程，之后的多次调用则可以避免CAS操作，说白了就是置个变量，如果发现为true则无需再走各种加锁/解锁流程