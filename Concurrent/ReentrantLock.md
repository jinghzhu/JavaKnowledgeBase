# ReentrantLock

![ReentrantLock](./Images/lock_uml.png)


## 1. ReentrantLock中有一个抽象类Sync
```java
private final Sync sync;
abstract static class Sync extends AbstractQueuedSynchronizer{
    ...
    }
```
&#12288;&#12288;ReentrantLock根据传入构造方法的布尔型参数实例化出Sync的实现类`FairSync`和`NonfairSync`，默认Nonfair。


## 2. Nonfair获取锁 （ReentrantLock是悲观锁）
```java
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0) // overflow
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```

&#12288;&#12288;流程：
1. 首先判断当前状态: 如果`c == 0`说明没有线程正在竞争该锁，如果不`c != 0`说明有线程拥有了锁。
2. 如果`c == 0`，通过CAS设置该状态值为`acquires`, `acquires`的初始调用值为1，每次线程重入该锁都会+1，每次unlock都会-1，但为0时释放锁。如果CAS设置成功，则可以预计其他任何线程调用CAS都不会再成功，也就认为当前线程得到了该锁 。
3. 如果`c != 0`，但发现已拥有锁，只是简单地`++acquires`，并修改`status`值。但因为没有竞争，所以通过`setStatus`修改，而非CAS，也就是说这段代码实现了偏向锁的功能 。


## 3. 解锁
```java
        protected final boolean tryRelease(int releases) {
            int c = getState() - releases;
            if (Thread.currentThread() != getExclusiveOwnerThread())
                throw new IllegalMonitorStateException();
            boolean free = false;
            if (c == 0) {
                free = true;
                setExclusiveOwnerThread(null);
            }
            setState(c);
            return free;
        }
```

&#12288;&#12288;`tryRelease`与`tryAcquire`语义相同，把如何释放的逻辑延迟到子类中。

&#12288;&#12288;`tryRelease`语义很明确：如果线程多次锁定，则进行多次释放，直至`status == 0`则真正释放锁，所谓释放锁即设置`status`为0，因为无竞争所以没有使用CAS。
