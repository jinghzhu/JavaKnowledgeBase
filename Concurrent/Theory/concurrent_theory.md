# <center>Theory of Concurrent Programming</center>

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
