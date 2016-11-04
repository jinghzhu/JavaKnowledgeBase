# AQS(AbstractQueuedSynchronizer)

## 1.  基于FIFO的CLH变种队列，内部有个静态类Node

```java
    static final class Node {
        static final int CANCELLED =  1;
        static final int SIGNAL    = -1;
        static final int CONDITION = -2;
        static final int PROPAGATE = -3;
       volatile int waitStatus;
        volatile Node prev;
        volatile Node next;
       volatile Thread thread;
        Node nextWaiter;

       Node(Thread thread, Node mode) {     // Used by addWaiter
            this.nextWaiter = mode;
            this.thread = thread;
        }

        Node(Thread thread, int waitStatus) { // Used by Condition
            this.waitStatus = waitStatus;
            this.thread = thread;
        }
    }
```

&#12288;&#12288;其中借助了`volatile`，通过构造函数可以看出每个`Node`保存了Thread的信息。

![AQS Node](./Images/aqs_node.png)

&#12288;&#12288;AQS部分自己的成员也借助了`volatile`:

```java
private transient volatile Node head;
private transient volatile Node tail;
private volatile int state;
```

&#12288;&#12288;对于首尾结点（即获取释放锁和阻赛线程）和结点status设置都是类似CAS语义

## 2. 尝试获取

![Try to get spin](./Images/aqs_acquire.png)

```java
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```

&#12288;&#12288;上述逻辑主要包括：
1. 尝试获取（调用`tryAcquire`更改状态，需要保证原子性）: 在`tryAcquire`中利用`compareAndSet`保证只有一个线程能够对状态进行成功修改，而没有成功修改的线程将进入`sync`队列排队。
2. 如果获取不到，将当前线程构造成节点`Node`并加入`sync`队列；进入队列的每个线程都是一个节点`Node`，从而形成了一个类似CLH双向队列， 这样做的目的是线程间的通信会被限制在较小规模（也就是两个节点左右）。
3. 再次尝试获取，如果没有获取，那么将当前线程从线程调度器上摘下，进入等待状态。使用`LockSupport`将当前线程unpark。


### 2.1 如果尝试获取失败，添加进队列

```java
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }
    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
```

&#12288;&#12288;上述逻辑主要包括：
1. 使用当前线程构造Node；将当节点前驱节点指向尾节点（`current.prev = tail`），尾节点指向它（`tail = current`），原有的尾节点的后继节点指向它（`t.next = current`）而这些操作要求是原子的。上面的操作是利用尾节点的设置来保证的，也就是`compareAndSetTail`来完成的。
2. 先行尝试在队尾添加`Node`；如果尾节点已经有了：
    1. 分配引用T指向尾节点；
    2. 将节点的前驱节点更新为尾节点（`current.prev = tail`）；
    3. 如果尾节点是T，将尾节点设置为该节点（`tail = current`，原子更新）；
    4. T的后继节点指向当前节点（`T.next = current`）。
3. 如果队尾添加失败或者是第一个入队的节点：如果是第1个节点，会进入到`enq`方法，进入的线程可能有多个，或者说在`addWaiter`中没有成功入队的线程都将进入`enq`方法。`enq`的逻辑是确保进入的`Node`都会有机会顺序的添加到`sync`队列中，而加入的步骤如下：
    1. 如果尾节点为空，那么原子化的分配一个头节点，并将尾节点指向头节点，这一步是初始化；
    2. 然后是重复在addWaiter中做的工作，但是在一个while(true)的循环中，直到当前节点入队为止。
  

&#12288;&#12288;进入`sync`队列之后，接下来就是要进行锁的获取，只有一个线程能够在同一时刻继续的运行，而其他的进入等待状态。而每个线程都是一个独立的个体，它们自省的观察，当条件满足的时候（自己的前驱是头结点并且原子性的获取了状态），那么这个线程能够继续运行。


## 2.2 再次尝试获取

```java
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor();
                if (p == head && tryAcquire(arg)) {
                    setHead(node);
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }
```

&#12288;&#12288;上述逻辑主要包括：
1. 获取当前节点的前驱节点；需获取当前节点的前驱节点，而头结点所对应的含义是当前站有锁且正在运行。
2. 当前驱节点是头结点并且能够获取状态，代表该当前节点占有锁；如果满足上述条件，那么代表能够占有锁，根据节点对锁占有的含义，设置头结点为当前节点。
3. 否则进入等待状态。如果没有轮到当前节点运行，那么将当前线程从线程调度器上摘下，也就是进入等待状态。


## 3. 尝试获取总结
1. 状态的维护；需要在锁定时，需要维护一个状态(int类型)，而对状态的操作是原子和非阻塞的，利用`compareAndSet`来确保原子性的修改。
2. 状态的获取；一旦成功的修改了状态，当前线程或者说节点，就被设置为头节点。
3. `sync`队列的维护。在获取资源未果的过程中条件不符合的情况下(不该自己，前驱节点不是头节点或者没有获取到资源)进入睡眠状态，停止线程调度器对当前节点线程的调度。这时引入的一个释放的问题，也就是说使睡眠中的`Node`或者说线程获得通知的关键，就是前驱节点的通知，而这一个过程就是释放，释放会通知它的后继节点从睡眠中返回准备运行。


## 4. 释放结点

```java
    public final boolean release(int arg) {
        if (tryRelease(arg)) {
            Node h = head;
            if (h != null && h.waitStatus != 0)
                unparkSuccessor(h);
            return true;
        }
        return false;
    }
```
&#12288;&#12288;上述逻辑主要包括：
1. 尝试释放状态；`tryRelease`保证原子化的将状态设置回去，需使用`compareAndSet`保证。如果释放状态成功过之后，将会进入后继节点的唤醒过程。
2. 唤醒当前节点的后继节点所包含的线程。通过`LockSupport`的`unpark`方法将休眠中的线程唤醒，让其继续`acquire`状态。

```java
   private void unparkSuccessor(Node node) {
        /* 将状态设置为同步状态 */
        int ws = node.waitStatus;
        if (ws < 0)
            compareAndSetWaitStatus(node, ws, 0);

        /* 获取当前结点的后继结点，如果满足状态进行唤醒 
         * 如果没满足，则从尾部尾部开始寻找符合要求的结点进行唤醒
         */
        Node s = node.next;
        if (s == null || s.waitStatus > 0) {
            s = null;
            for (Node t = tail; t != null && t != node; t = t.prev)
                if (t.waitStatus <= 0)
                    s = t;
        }
        if (s != null)
            LockSupport.unpark(s.thread);
}
```