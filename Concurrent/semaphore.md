# <center>Semaphore</center>



<br></br>

* 基于synchronized。

<br></br>



## Semaphore的实现
----

```java
public class Semaphore {
    private boolean signal = false;   // 使用signal避免信号丢失

    public synchronized void take() {
       this.signal = true;
       this.notify();
    }

    public synchronized void release() throws InterruptedException{
       while(!this.signal) // 使用while避免假唤醒
           wait();
        this.signal = false;
    }
}
```

<br></br>



## 可计数Semaphore
----

```java
public class CountingSemaphore {
    private int signals = 0;

    public synchronized void take() {
        this.signals++;
        this.notify();
    }

public synchronized void release() throws InterruptedException{
    while(this.signals == 0) 
        wait();
    this.signals--;
    }
}
```

<br></br>



## 有上限的Semaphone
----

```java
public class BoundedSemaphore {
    private int signals = 0;
    private int bound   = 0;

    public BoundedSemaphore(int upperBound){
        this.bound = upperBound;
    }

    public synchronized void take() throws InterruptedException{
        while(this.signals == bound) 
            wait();
        this.signals++;
        this.notify();
    }
    
    public synchronized void release() throws InterruptedException{
        while(this.signals == 0) 
            wait();
        this.signals--;
        this.notify();
    }
}
```

<br></br>



## 互斥量Mutex
----

```java
public class Mutex {
    private boolean isLocked = false;

    public synchronized void lock() {
        while(this.isLocked) // 使用while避免假唤醒
            wait();
        this.isLocked= true;
    }
}

public synchronized void unlock() throws InterruptedException{
    this.isLocked= false;
    this.notify();
}
```
