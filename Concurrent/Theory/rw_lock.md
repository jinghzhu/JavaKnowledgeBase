# <center>Read-Write Lock</center>

<br>



#### 信号量实现的写进程优先
--------------
``` java
int readcount, writecount;
semaphore x = 1, y = 1, z = 1, wsem = 1, rsem = 1;

void reader() 
{
	while(true)
	{
		wait(z); // 不允许在rsem上创建长的读进程等待队列，否则可能饿死写进程
		wait(rsem);
		wait(x); // 控制readcount++互斥访问
		readcount++;
		if (readcount == 1)
			wait(wsem);
		signal(x);
		signal(rsem);
		signal(z);
		// do reader work...
		wait(x);
		readcount--;
		if (readcount == 0)
			signal(wsem);
		signal(x);
	}
}

void writer()
{
	while(true)
	{
		wait(y); // 控制writecount++互斥访问
		writecount++;
		if (writecount == 1)
			wait(rsem); // 当有写进程时，禁止所有读进程
		signal(y);
		wait(wsem);
		// do write work...
		signal(wsem);
		wait(y);
		writecount--;
		if (writecount == 0)
			signal(wsem);
		signal(y);
	}
}

void main()
{
	readcount = writecount = 0;
	begin(reader, writer);
}
```

<br>



#### 读写锁可重入的Java代码
-------------------
##### 最简单的读写锁（写锁高优先级）
&#12288;&#12288;如果读操作较频繁，又没有提升写操作优先级就会产生“饥饿”。写线程一直阻塞，直到所有读线程都从ReadWriteLock上解锁。因此，只有当没有线程正锁住ReadWriteLock进行写操作，且没有线程请求锁准备写操作时，才保证读操作继续。

``` java
public class ReadWriteLock{
    private int readers = 0;
    private int writers = 0;
    private int writeRequests = 0;

    public synchronized void lockRead() throws InterruptedException{
        // writeRequests用来防止频繁的读操作以至于写操作饿死
        while(writers > 0 || writeRequests > 0){ 
            wait();
        }
        readers++;
    }

    public synchronized void unlockRead(){
        readers--;
        notifyAll();
    }

    public synchronized void lockWrite() 
        throws InterruptedException{
        writeRequests++;

        while(readers > 0 || writers > 0){
            wait();
        }
        writeRequests--;
        writers++;
    }

    public synchronized void unlockWrite() throws InterruptedException{
        writers--;
        notifyAll();
    }
}
```

&#12288;&#12288;在两个释放锁的方法（unlockRead，unlockWrite）中，都调用`notifyAll()`方法，而不是`notify()`，因为：

* 如果有线程在等待获取读锁，同时又有线程在等待获取写锁。如果这时一个等待读锁的线程被`notify()`方法唤醒，但因为此时仍有请求写锁的线程存在（`writeRequests > 0`），所以被唤醒的线程会再次进入阻塞状态。然而，等待写锁的线程一个也没被唤醒，就像什么也没发生过一样（信号丢失现象）。如果用`notifyAll()`方法，所有的线程都会被唤醒，然后判断能否获得其请求的锁。

* 如果有多个读线程等待读锁且没有线程在等待写锁时，调用`unlockWrite()`后，所有等待读锁的线程都能立马成功获取读锁，而不是一次只允许一个。

<br>


##### 读锁重入
&#12288;&#12288;要保证某个读锁可重入，要么没有写或写请求，要么已持有读锁（不管是否有写请求）。 可以用一个map存储已持有读锁的线程以及对应线程获取读锁的次数。当需要判断某个线程能否获得读锁时，就用map中数据进行判断。

``` java
public class ReadWriteLock{
    private Map<Thread, Integer> readingThreads =
        new HashMap<Thread, Integer>();

    private int writers = 0;
    private int writeRequests = 0;

    public synchronized void lockRead() throws InterruptedException{
        Thread callingThread = Thread.currentThread();
        while(!canGrantReadAccess(callingThread)){
            wait();                                                                   
        }

        readingThreads.put(callingThread,
            (getAccessCount(callingThread) + 1));
    }

    public synchronized void unlockRead(){
        Thread callingThread = Thread.currentThread();
        int accessCount = getAccessCount(callingThread);
        if(accessCount == 1) { 
            readingThreads.remove(callingThread); 
        } else {
            readingThreads.put(callingThread, (accessCount -1)); 
        }
        notifyAll();
    }

    private boolean canGrantReadAccess(Thread callingThread){
        if(writers > 0) return false;
        if(isReader(callingThread) return true;
        if(writeRequests > 0) return false;
        return true;
    }

    private int getReadAccessCount(Thread callingThread){
        Integer accessCount = readingThreads.get(callingThread);
        if(accessCount == null) return 0;
        return accessCount.intValue();
    }

    private boolean isReader(Thread callingThread){
        return readingThreads.get(callingThread) != null;
    }
}
```

&#12288;&#12288;注意，重入的读锁比写锁优先级高。

<br>


##### 写锁重入
&#12288;&#12288;仅当一个线程已持有写锁，才允许写锁重入（再次获得写锁）。

``` java
public class ReadWriteLock{
    private Map<Thread, Integer> readingThreads =
        new HashMap<Thread, Integer>();

    private int writeAccesses    = 0;
    private int writeRequests    = 0;
    private Thread writingThread = null;

    public synchronized void lockWrite() throws InterruptedException{
        writeRequests++;
        Thread callingThread = Thread.currentThread();
        while(!canGrantWriteAccess(callingThread)){
            wait();
        }
        writeRequests--;
        writeAccesses++;
        writingThread = callingThread;
    }

    public synchronized void unlockWrite() throws InterruptedException{
        writeAccesses--;
        if(writeAccesses == 0){
            writingThread = null;
        }
        notifyAll();
    }

    private boolean canGrantWriteAccess(Thread callingThread){
        if(hasReaders()) return false;
        if(writingThread == null)    return true;
        if(!isWriter(callingThread)) return false;
        return true;
    }

    private boolean hasReaders(){
        return readingThreads.size() > 0;
    }

    private boolean isWriter(Thread callingThread){
        return writingThread == callingThread;
    }
}
```

<br>


##### 读锁升级到写锁
&#12288;&#12288;有时希望拥有读锁的线程也能获得写锁。要允许这操作，要求这个线程是唯一一个拥有读锁的线程。`writeLock()`做点改动：

``` java
public class ReadWriteLock{
    private Map<Thread, Integer> readingThreads =
        new HashMap<Thread, Integer>();

    private int writeAccesses    = 0;
    private int writeRequests    = 0;
    private Thread writingThread = null;

    public synchronized void lockWrite() throws InterruptedException{
        writeRequests++;
        Thread callingThread = Thread.currentThread();
        while(!canGrantWriteAccess(callingThread)){
            wait();
        }
        writeRequests--;
        writeAccesses++;
        writingThread = callingThread;
    }

    public synchronized void unlockWrite() throws InterruptedException{
        writeAccesses--;
        if(writeAccesses == 0){
            writingThread = null;
        }
        notifyAll();
    }

    private boolean canGrantWriteAccess(Thread callingThread){
        if(isOnlyReader(callingThread)) return true; // 升级读锁
        if(hasReaders()) return false;
        if(writingThread == null) return true;
        if(!isWriter(callingThread)) return false;
        return true;
    }

    private boolean hasReaders(){
        return readingThreads.size() > 0;
    }

    private boolean isWriter(Thread callingThread){
        return writingThread == callingThread;
    }

    private boolean isOnlyReader(Thread thread){
        return readers == 1 && readingThreads.get(callingThread) != null;
    }
}
```

<br>


##### 写锁降级到读锁
&#12288;&#12288;有时拥有写锁的线程希望得到读锁。如果线程拥有写锁，那么其它线程不可能拥有读锁或写锁。所以对于拥有写锁的线程，再获得读锁，不会有什么危险。仅需对`canGrantReadAccess()`方法修改：

``` java
public class ReadWriteLock{
    private boolean canGrantReadAccess(Thread callingThread){
        if(isWriter(callingThread)) return true; // 获得读锁
        if(writingThread != null) return false;
        if(isReader(callingThread) return true;
        if(writeRequests > 0) return false;
        return true;
    }
}
```

<br>


#### 可重入ReadWriteLock完整实现
------------------------
``` java
public class ReadWriteLock{
    private Map<Thread, Integer> readingThreads =
        new HashMap<Thread, Integer>();

    private int writeAccesses    = 0;
    private int writeRequests    = 0;
    private Thread writingThread = null;

    /************* Main Reader Lock *************/

    public synchronized void lockRead() throws InterruptedException{
        Thread callingThread = Thread.currentThread();
        while(! canGrantReadAccess(callingThread)){
            wait();
        }

        readingThreads.put(callingThread,
            (getReadAccessCount(callingThread) + 1));
    }

    public synchronized void unlockRead(){
        Thread callingThread = Thread.currentThread();
        if(!isReader(callingThread)){
            throw new IllegalMonitorStateException(
                "Calling Thread does not" +
                " hold a read lock on this ReadWriteLock");
        }
        int accessCount = getReadAccessCount(callingThread);
        if(accessCount == 1){ 
            readingThreads.remove(callingThread); 
        } else { 
            readingThreads.put(callingThread, (accessCount -1));
        }
        notifyAll();
    }

    /************* Reader Lock Helper *************/

    private boolean canGrantReadAccess(Thread callingThread){
        if(isWriter(callingThread)) return true;
        if(hasWriter()) return false;
        if(isReader(callingThread)) return true;
        if(hasWriteRequests()) return false;
        return true;
    }

    private int getReadAccessCount(Thread callingThread){
        Integer accessCount = readingThreads.get(callingThread);
        if(accessCount == null) return 0;
        return accessCount.intValue();
    }

    private boolean hasReaders(){
        return readingThreads.size() > 0;
    }

    private boolean isReader(Thread callingThread){
        return readingThreads.get(callingThread) != null;
    }

    private boolean isOnlyReader(Thread callingThread){
        return readingThreads.size() == 1 &&
            readingThreads.get(callingThread) != null;
    }

    /************* Main Writer Lock *************/

    public synchronized void lockWrite() throws InterruptedException{
        writeRequests++;
        Thread callingThread = Thread.currentThread();
        while(!canGrantWriteAccess(callingThread)){
            wait();
        }
        writeRequests--;
        writeAccesses++;
        writingThread = callingThread;
    }

    public synchronized void unlockWrite() throws InterruptedException{
        if(!isWriter(Thread.currentThread()){
        throw new IllegalMonitorStateException(
            "Calling Thread does not" +
            " hold the write lock on this ReadWriteLock");
        }
        writeAccesses--;
        if(writeAccesses == 0){
            writingThread = null;
        }
        notifyAll();
    }

    /************* Writer Lock Helper *************/

    private boolean canGrantWriteAccess(Thread callingThread){
        if(isOnlyReader(callingThread)) return true;
        if(hasReaders()) return false;
        if(writingThread == null) return true;
        if(!isWriter(callingThread)) return false;
        return true;
    }

    private boolean hasWriter(){
        return writingThread != null;
    }

    private boolean isWriter(Thread callingThread){
        return writingThread == callingThread;
    }

    private boolean hasWriteRequests(){
        return this.writeRequests > 0;
    }
}
```