# <center>Theory of Concurrent Programming</center>

<br></br>



## CPU锁
----
intel对lock前缀说明如下：

1. 确保对内存的读-改-写操作原子执行。如果要访问的内存区域在lock前缀指令执行期间已在处理器内部缓存中被锁定（即包含该内存区域的缓存行当前处于独占或修改状态），并且该内存区域被完全包含在单个缓存行中，那么处理器将直接执行该指令。由于在指令执行期间该缓存行会一直被锁定，其它处理器无法读/写该指令要访问的内存区域，因此能保证指令执行的原子性。这个操作过程叫做缓存锁定（cache locking）。但当多处理器之间的竞争程度很高或者指令访问的内存地址未对齐时，仍然会锁住总线。
2. 禁止该指令与之前和之后的读和写指令重排序。
3. 把写缓冲区中的所有数据刷新到内存中。

第2点和第3点所具有的内存屏障效果，足以同时实现volatile读写的内存语义，编译器不能对CAS与CAS前面和后面的任意内存操作重排序。

> volatile读写内存语义:
> 1. 编译器不会对volatile读与volatile读后面任意内存操作重排序；
> 2. 编译器不会对volatile写与volatile写前面任意内存操作重排序。

CPU锁有3种：
1. 处理器自动保证基本内存操作的原子性

    首先处理器保证基本内存操作原子性。处理器保证从系统内存当中读取或写入一个字节是原子的，意思是当处理器读取一个字节时，其他处理器不能访问这个字节内存地址。

2. 使用总线锁保证原子性

    如果多处理器同时对共享变量进行读改写（比如`i++`），那么共享变量操作完之后共享变量的值会和期望的不一致，处理器使用总线锁解决这个问题。所谓总线锁是使用处理器提供的LOCK＃信号，当处理器在总线上输出此信号时，其他处理器请求被阻塞，那么该处理器可独占使用共享内存。

3. 使用缓存锁保证原子性

    同一时刻只需保证对某个内存地址操作原子即可，但总线锁把CPU和内存通信锁住，使得锁定期间，其他处理器不能操作其他内存地址数据，所以某些场合下使用缓存锁定代替总线锁定进行优化。

    频繁使用内存会缓存在处理器的L1，L2 和L3里，那么原子操作可在处理器内部缓存中进行，并不需要总线锁。所谓缓存锁定是如果缓存在处理器缓存行中内存区域在LOCK操作期间被锁定，当它执行锁操作回写内存时，处理器不在总线上声言LOCK＃信号，而是修改内部的内存地址，并允许它的缓存一致性机制来保证操作的原子性。因为缓存一致性会阻止同时修改被两个以上处理器缓存的内存区域数据，当其他处理器回写已被锁定的缓存行的数据时会起缓存行无效。

    但是有两种情况处理器不会使用缓存锁定。第一种情况是当操作的数据不能被缓存在处理器内部，或操作的数据跨多个缓存行，则处理器会调用总线锁定。第二种情况是有些处理器不支持缓存锁定。

<br></br>



## 乐观锁
----
悲观锁使用同步块或其他类型的锁阻塞对临界区域的访问。一个同步块或锁可能会导致线程挂起。乐观锁允许所有线程在不发生阻塞的情况下创建一份共享内存的拷贝。这些线程接下来可能对拷贝进行修改，并企图把修改后的版本写回到共享内存中。如果没有其它线程对共享内存做任何修改，CAS操作允许线程将它的变化写回到共享内存中去。如果，另一个线程已经修改了共享内存，这个线程将不得不再次获得一个新的拷贝，在新的拷贝上做出修改，并尝试再次把它们写回到共享内存中去。

乐观锁使用于共享内存竞用不是非常高的情况。如果共享内存上的内容非常多，仅仅因为更新共享内存失败，就浪费大量CPU周期用在拷贝和修改上。但是，如果共享内存上有大量的内容，无论如何，你都要把你的代码设计的产生的争用更低。

乐观锁是非阻塞的，因为如果线程获得共享内存拷贝，尝试修改时发生了阻塞，其它线程去访问这块内存区域不会发生阻塞。对于传统的加/解锁，当线程持有锁时，其它所有的线程都会阻塞直到持有锁的线程释放锁。

<br></br>


 
## Happens-Before
----
重排序需要遵守happens-before规则:
* **程序次序法则** 如果A一定在B之前发生，则happen before
* **监视器法则** 对一个监视器的解锁一定发生在后续对同一监视器加锁之前
* **volatie变量法则** 写volatile变量一定发生在后续对它的读之前 
* **线程启动法则** `Thread.start`一定发生在线程中的动作 
* **终结法则** 对象的构造函数结束一定发生在对象的finalizer前 
* **传递性** A发生在B之前，B发生在C之前，A一定发生在C之前

两个操作间具有happens-before关系，不意味着前一个操作要在后一个操作之前执行。happens-before仅要求前一个操作（执行的结果）对后一个操作可见，且前一个操作按顺序排在第二个操作之前。

<p align="center">
  <img src="./Images/happens-before_JMM.png" />
</p>

<br></br>



## 线程安全与共享资源
----
### 局部的对象引用
对象的局部引用和基础类型的局部变量不一样。尽管引用本身没有被共享，但引用所指的对象没有存储在线程的栈内。所有的对象都存在共享堆中。如果在某个方法中创建的对象不会逃逸出该方法，那么它是线程安全的。哪怕将这个对象作为参数传给其它方法，只要别的线程获取不到这个对象，它仍是线程安全的。

``` java
public void someMethod(){
  LocalObject localObject = new LocalObject();

  localObject.callMethod();
  method2(localObject);
}

public void method2(LocalObject localObject){
  localObject.setValue("value");
}
```

LocalObject对象没有被方法返回，也没有传给`someMethod()`方法外的对象。每个执行`someMethod()`的线程都会创建自己的LocalObject对象，并赋值给`localObject`引用。因此，这里的`LocalObject`是线程安全的。事实上，整个`someMethod()`都是线程安全的。即使将LocalObject作为参数传给同一个类的其它方法或其它类的方法时，它仍是线程安全的。如果LocalObject通过某些方法被传给了别的线程，那它就不再是线程安全的了。

<br>


### 对象成员
对象成员存储在堆上。如果两个线程同时更新同一个对象的同一个成员，那就不是线程安全的。

``` java
public class NotThreadSafe{
    StringBuilder builder = new StringBuilder();

    public add(String text){
        this.builder.append(text);
    }    
}
```

如果两个线程同时调用同一个NotThreadSafe实例上的`add()`方法，就会有竞态条件：

``` java
NotThreadSafe sharedInstance = new NotThreadSafe();

new Thread(new MyRunnable(sharedInstance)).start();
new Thread(new MyRunnable(sharedInstance)).start();

public class MyRunnable implements Runnable{
  NotThreadSafe instance = null;

  public MyRunnable(NotThreadSafe instance){
    this.instance = instance;
  }

  public void run(){
    this.instance.add("some text");
  }
}
```

两个MyRunnable共享了同一个NotThreadSafe对象，当它们调用`add()`方法会造成竞态条件。

<br>


### 线程控制逃逸规则
如果一个资源的创建、使用和销毁都在同一个线程内完成，且永远不会脱离该线程的控制，则该资源的使用是线程安全的。

即使对象本身线程安全，但如果该对象中包含其他资源（文件，数据库连接），整个应用也许不再是线程安全。比如2个线程都创建了各自的数据库连接，每个连接自身是线程安全的，但它们所连接到的同一个数据库也许不是线程安全的：

```
检查记录X是否存在，如果不存在，插入X
```

如果两个线程同时执行，且检查同一个记录，那么两个线程最终可能都插入了记录：
```
线程 1 检查记录 X 是否存在。检查结果：不存在
线程 2 检查记录 X 是否存在。检查结果：不存在
线程 1 插入记录 X
线程 2 插入记录 X
```

<br></br>



## 线程安全及不可变性
----

``` java
public class ImmutableValue{
    private int value = 0;

    public ImmutableValue(int value){
        this.value = value;
    }

    public int getValue(){
        return this.value;
    }
}
```

ImmutableValue类成员变量`value`是通过构造函数赋值，在类中没有set方法。一旦ImmutableValue实例被创建，value变量就不能再被修改，这就是不可变性。

> “不变”（Immutable）和“只读”（Read Only）是不同的。当一个变量是“只读”时，变量的值不能直接改变，但可在其它变量发生改变的时候发生改变。比如，一个人的出生年月日是“不变”属性，而一个人的年龄是“只读”属性。随着时间变化，年龄会变化，而出生年月日则不会。

需要对ImmutableValue类实例进行操作，可以通过得到`value`变量后创建一个新的实例来实现：

``` java
public class ImmutableValue{
    private int value = 0;

    public ImmutableValue(int value){
        this.value = value;
    }

    public int getValue(){
        return this.value;
    }

    public ImmutableValue add(int valueToAdd){
        return new ImmutableValue(this.value + valueToAdd);
    }
}
```

注意`add()`方法以加法操作的结果作为一个新的ImmutableValue类实例返回，而不是直接对它自己的`value` 变量进行操作。

**引用不是线程安全的！**即使一个对象是线程安全的不可变对象，指向这个对象的引用也可能不是线程安全的：

``` java
public void Calculator{
    private ImmutableValue currentValue = null;

    public ImmutableValue getValue(){
        return currentValue;
    }

    public void setValue(ImmutableValue newValue){
        this.currentValue = newValue;
    }

    public void add(int newValue){
        this.currentValue = this.currentValue.add(newValue);
    }
}
```

Calculator类有一个指向ImmutableValue实例的引用。通过`setValue()`方法和`add()`方法可能会改变这个引用。因此，即使Calculator类内部使用了一个不可变对象，但Calculator类本身还是可变的，因此Calculator类不是线程安全的。

要使Calculator类线程安全，将`getValue()`、`setValue()`和`add()`方法都声明为同步方法即可。
