# <center>Thread Communication</center>



## 1. 通过共享对象通信
线程间发送信号的简单方式是在共享对象的变量里设置信号值。线程A在一个同步块里设置boolean型成员变量`hasDataToProcess`为true，线程B在同步块里读取`hasDataToProcess`变量。这个例子使用了一个持有信号的对象，并提供了set和check方法:

``` java
public class MySignal{

  protected boolean hasDataToProcess = false;

  public synchronized boolean hasDataToProcess(){
    return this.hasDataToProcess;
  }

  public synchronized void setHasDataToProcess(boolean hasData){
    this.hasDataToProcess = hasData;
  }

}
```

<br></br>



## 2. wait()，notify()和notifyAll()
MySingal修改版本——使用了`wait()`和`notify()`：

``` java
public class MonitorObject{}

public class MyWaitNotify{

  MonitorObject myMonitorObject = new MonitorObject();

  public void doWait(){
    synchronized(myMonitorObject){
      try{
        myMonitorObject.wait();
      } catch(InterruptedException e){...}
    }
  }

  public void doNotify(){
    synchronized(myMonitorObject){
      myMonitorObject.notify();
    }
  }
}
```

一个线程如果没有持有对象锁，不能调用	`wait()`，`notify()`或`notifyAll()`。否则会抛出IllegalMonitorStateException。因为一旦线程调用了`wait()`方法，它就释放了所持有的监视器对象上的锁。

一旦一个线程被唤醒，不能立刻就退出`wait()`方法调用，直到调用`notify()`线程退出了它自己的同步块。换句话说：被唤醒的线程必须重新获得监视器对象的锁，才可以退出`wait()`方法调用，因为`wait()`方法调用运行在同步块里面。如果多个线程被`notifyAll()`唤醒，那么在同一时刻将只有一个线程可以退出`wait()`方法，因为每个线程在退出` wait()`前须获得监视器对象的锁。

<br></br>



## 3. 丢失的信号（Missed Signals）
`notify()`和`notifyAll()`方法不会保存调用它们的方法，因为当这两个方法被调用时，有可能没有线程处于等待状态。通知信号过后便丢弃了。因此，如果一个线程先于被通知线程调用`wait()`前调用了`notify()`，等待的线程将错过这个信号。

为避免丢失信号，须把它们保存在信号类里。在MyWaitNotify中，通知信号应被存储在MyWaitNotify实例的一个成员变量里：

``` java
public class MyWaitNotify2{

  MonitorObject myMonitorObject = new MonitorObject();
  boolean wasSignalled = false;

  public void doWait(){
    synchronized(myMonitorObject){
      if(!wasSignalled){
        try{
          myMonitorObject.wait();
         } catch(InterruptedException e){...}
      }
      //clear signal and continue running.
      wasSignalled = false;
    }
  }

  public void doNotify(){
    synchronized(myMonitorObject){
      wasSignalled = true;
      myMonitorObject.notify();
    }
  }
}
```

<br></br>



## 4. 假唤醒
线程可能在没有调用过`notify()`和`notifyAll()`的情况下醒来，这就是假唤醒（spurious wakeups）。

如果在MyWaitNotify2的`doWait()`方法里发生了假唤醒，等待线程即使没有收到正确的信号，也能够执行后续的操作。为防止假唤醒，保存信号的成员变量将在一个while循环里接受检查，而不是在if表达式里。这样的一个while循环叫做自旋锁（这种做法要慎重，JVM实现自旋会消耗CPU）。被唤醒的线程会自旋直到自旋锁（while循环）里的条件变为false。

``` java
public class MyWaitNotify3{

  MonitorObject myMonitorObject = new MonitorObject();
  boolean wasSignalled = false;

  public void doWait(){
    synchronized(myMonitorObject){
      while(!wasSignalled){
        try{
          myMonitorObject.wait();
         } catch(InterruptedException e){...}
      }
      //clear signal and continue running.
      wasSignalled = false;
    }
  }

  public void doNotify(){
    synchronized(myMonitorObject){
      wasSignalled = true;
      myMonitorObject.notify();
    }
  }
}
```

<br></br>



## 5. 不要在字符串常量或全局对象中调用wait()

> 字符串常量指的是值为常量的变量

下面的MyWaitNotify里使用字符串常量（””）作为管程对象：

``` java
public class MyWaitNotify{

  String myMonitorObject = "";
  boolean wasSignalled = false;

  public void doWait(){
    synchronized(myMonitorObject){
      while(!wasSignalled){
        try{
          myMonitorObject.wait();
         } catch(InterruptedException e){...}
      }
      //clear signal and continue running.
      wasSignalled = false;
    }
  }

  public void doNotify(){
    synchronized(myMonitorObject){
      wasSignalled = true;
      myMonitorObject.notify();
    }
  }
}
```

在空字符串作为锁的同步块(或者其他常量字符串)里调用`wait()`和`notify()`产生的问题是，JVM/编译器会把常量字符串转换成同一个对象。意味着即使有2个不同的MyWaitNotify实例，它们都引用了相同的空字符串实例。所以，存在这样的风险：在第一个 MyWaitNotify实例上调用`doWait()`的线程会被在第二个MyWaitNotify实例上调用`doNotify()`的线程唤醒：

<p align="center">
  <img src="./Images/string_comm.png"/>
</p>

<br>

**这种情况相当于假唤醒**。线程A或B在信号没有更新的情况下唤醒。但是代码处理了这种情况，所以线程回到了等待状态。在 MyWaitNotify1上的`doNotify()`调用可能唤醒MyWaitNotify2的线程，但信号值只保存在MyWaitNotify1里。

由于`doNotify()`是调用`notify()`而不是`notifyAll()`。所以，如果线程A或B被发给C或D的信号唤醒，它会检查自己的信号值，看有没有信号被接收到，然后回到等待状态。而C和D都没被唤醒来检查它们实际上接收到的信号值，这样**信号便丢失了**。

如果`doNotify()`方法调用`notifyAll()`，所有等待线程都会被唤醒并依次检查信号值。线程A和B将回到等待状态，但C或D只有一个线程注意到信号，并退出`doWait()`方法。

> 你可能会设法使用`notifyAll()`代替`notify(，但这在性能上是个坏主意。在只有一个线程能对信号进行响应的情况下，没有理由每次都去唤醒所有线程。

所以在wait/notify机制中，不要使用全局对象，字符串常量等。应该使用对应唯一的对象。例如，每一个MyWaitNotify3的实例拥有一个属于自己的监视器对象。

