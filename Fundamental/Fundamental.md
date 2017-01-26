# <center>Java Fundamental</center>



## 1. What does super keyword do? 
We can use super keyword to invoke super class constructor in child class constructor but in this case it should be the first statement in the constructor method.

``` java
public class SuperClass { 
    public SuperClass(){ 
    } 

    public void test(){ 
        System.out.println("super class test method"); 
    } 
} 
```

Use of super keyword can be seen in below child class implementation.

``` java
public class ChildClass extends SuperClass { 
    public ChildClass(String str){ 
        //access super class constructor with super keyword 
        super(); 
        //access child class method 
        test(); 
        //use super to access super class method 
        super.test(); 
    } 

    @Override 
    public void test(){ 
        System.out.println("child class test method"); 
    } 
}
```

<br></br>



## 2. What is final keyword? 
* Class: no other class can extend it, for example we can’t extend String class. 
* Method: child classes can’t override it.
* Variable: 对于`final`变量，如果是基本数据类型，则其数值在初始化后便不能更改；如果是引用类型，则初始化之后不能指向另一个对象。

<br></br>



## 3. What is static keyword? 
&#12288;&#12288;static作用是方便在未创建对象的情况下来调用方法／变量。
* Variable: static keyword can be used with class level variables to make it global i.e. all the objects will share the same variable. 
* Method: a static method can access only static variable of class and invoke only static methods of the class，但反过来是可以的.而且可以在没创建任何对象的前提下，仅通过类本身来调用static方法。static方法就是没有this的方法.另外，即使没有显示声明为static，类的构造器也是静态方法。
* Block: is the group of statements that gets executed when the class is loaded into memory by Java ClassLoader. It is used to initialize static variables of the class. Mostly it’s used to create static resources when class is loaded.

<br></br>



## 4. 接口
接口有以下特性：
* 接口是隐式抽象的，当声明一个接口的时候，不必使用abstract关键字。
* 接口中每一个方法也是隐式抽象的，声明时同样不需要abstract关键子。
* 接口中的方法都是公有的。
* 当类实现接口的时候，类要实现接口中所有的方法。否则，类必须声明为抽象的类。

<br></br>



## 5. 抽象方法
&#12288;&#12288;如果你想设计这样一个类，该类包含一个特别的成员方法，该方法的具体实现由它的子类确定，那么你可以在父类中声明该方法为抽象方法。Abstract关键字同样可以用来声明抽象方法，抽象方法只包含一个方法名，而没有方法体。
&#12288;&#12288;抽象方法没有定义，方法名后面直接跟一个分号，而不是花括号。

``` java
public abstract class Employee{
   private String name;
   private String address;
   private int number;
   
   public abstract double computePay();
}
```

&#12288;&#12288;声明抽象方法会造成以下两个结果：
* 如果一个类包含抽象方法，那么该类必须是抽象类。
* 任何子类必须重写父类的抽象方法，或者声明自身为抽象类。

&#12288;&#12288;如果Salary类继承Employee类，那么它必须实现`computePay()`方法：
``` java
public class Salary extends Employee {
   private double salary; // Annual salary
  
   public double computePay() {
      System.out.println("Computing salary pay for " + getName());
      return salary/52;
   }
}
```

<br></br>



## 6. Override重写，即子类对父类； Overlode重载，即一个类里面
* 重写Override规则
    * 参数列表必须完全与被重写方法的相同；
    * 返回类型必须完全与被重写方法的返回类型相同；
    * 访问权限不能比父类中被重写的方法的访问权限更低。例如：如果父类的一个方法被声明为public，那么在子类中重写该方法就不能声明为protected。
    * 父类的成员方法只能被它的子类重写。
    * 声明为final和static的方法不能被重写。
    * 重写的方法能够抛出任何非强制异常，无论被重写的方法是否抛出异常。但是，重写的方法不能抛出新的强制性异常，或者比被重写方法声明的更广泛的强制性异常，反之则可以。
    * 构造方法不能被重写。
    * 如果不能继承一个方法，则不能重写这个方法。
* 重载Overload规则：
    * 被重载的方法必须改变参数列表；
    * 被重载的方法可以改变返回类型；
    * 被重载的方法可以声明新的或更广的检查异常；
    * 方法能够在同一个类中或者在一个子类中被重载。
* 重写与重载之间的区别

区别点      |    重载Overload | 重写Override  
-------- | -------- | ----------
参数列表  | 必须修改 |  不能修改   
返回类型 |   可以修改 |  不能修改 
异常      |    可以修改 | 可以减少，但不能抛出新的或更广的异常  
访问      |    可以修改 | 可以降低限制，但不能做更严格限制  

<br></br>



## 7. Is Java pass by value or pass by reference?
&#12288;&#12288;java中方法参数传递方式是按值传递。总结一下java中方法参数的使用情况：

* 方法不能修改基本数据类型的参数(即数值型和布尔型)
* 方法可以改变对象参数的状态
* 方法不能让对象参数引用新对象

### 7.1 基本数据类型为参数
``` java
public class ParamTest {
    public static void main(String[] args) {
        int price = 5;
        doubleValue(price);
        System.out.print(price); // 5
    }

    public static void doubleValue(int x) {
        x = 2 * x;
    }
}
```

Output: 5

<p align="center">
  <img src="./Images/pass_by_value1.jpg" />
</p>


### 7.2 对象引用为参数
``` java
class Student {
    private float score;

    public Student(float score) {
        this.score = score;
    }

    public void setScore(float score) {
        this.score = score;
    }

    public float getScore() {
        return score;
    }
}

public class ParamTest {
    public static void main(String[] args) {
        Student stu = new Student(80);
        raiseScore(stu);
        System.out.print(stu.getScore()); // 90
    }

    public static void raiseScore(Student s) {
        s.setScore(s.getScore() + 10);
    }
}
```

Output: 90

<p align="center">
  <img src="./Images/pass_by_value2.jpg" />
</p>


### 7.3 对对象是值调用还是引用传递？

``` java
public static void swap(Student x, Student y) {
    Student temp = x;
    x = y;
    y = temp;
 } 

class Student {
    private float score;

    public Student(float score) {
        this.score = score;
    }

    public void setScore(float score) {
        this.score = score;
    }

    public float getScore() {
        return score;
    }
}

public class ParamTest {
    public static void main(String[] args) {
        Student a = new Student(0);
        Student b = new Student(100);

        System.out.println("交换前：");
        System.out.println("a的分数：" + a.getScore() + "--- b的分数：" + b.getScore());

        swap(a, b);

        System.out.println("交换后：");
        System.out.println("a的分数：" + a.getScore() + "--- b的分数：" + b.getScore());
    }

    public static void swap(Student x, Student y) {
        Student temp = x;
        x = y;
        y = temp;
    }
}
```

Output:

``` java
交换前：
a的分数：0.0--- b的分数：100.0
交换后：
a的分数：0.0--- b的分数：100.0
```

&#12288;&#12288;看swap调用的过程：
1. 将对象`a`，`b`的拷贝赋给`x`，`y`，此时`a`和`x`指向同一对象，`b`和`y`指向同一对象
2. `swap`方法完成`x`，`y`交换，此时`a`，`b`没有变化
3. 方法执行完成，`x`和`y`不再使用，`a`依旧指向`Student(0)`，`b`指向`Student(100)`

&#12288;&#12288;首先，创建两个对象：
<p align="center">
  <img src="./Images/pass_by_value3.jpg" />
</p>

&#12288;&#12288;然后，进入`swap`，将对象`a`，`b`的拷贝赋给`x`，`y`：
<p align="center">
  <img src="./Images/pass_by_value4.jpg" />
</p>

&#12288;&#12288;接着，交换`x`，`y`的值：
<p align="center">
  <img src="./Images/pass_by_value5.jpg" />
</p>

<br></br>



## 8. What are the Exception Handling Keywords in Java? 
There are four keywords used in java exception handling. 
1. throw: Sometimes we explicitly want to create exception object and then throw it to halt the normal processing of the program. throw keyword is used to throw exception to the runtime to handle it. 
2. throws: When we are throwing any checked exception in a method and not handling it, then we need to use throws keyword in method signature to let caller program know the exceptions that might be thrown by the method. The caller method might handle these exceptions or propagate it to its caller method using throws keyword. We can provide multiple exceptions in the throws clause and it can be used with main() method also. 
3. try-catch: We use try-catch block for exception handling in our code. try is the start of the block and catch is at the end of try block to handle the exceptions. We can have multiple catch blocks with a try and try-catch block can be nested also. catch block requires a parameter that should be of type Exception. 
4. finally: finally block is optional and can be used only with try-catch block. Since exception halts the process of execution, we might have some resources open that will not get closed, so we can use finally block. Finally block gets executed always, whether exception occurrs or not. 

<br></br>



## 9. Explain JAVA Exception Hierarchy
* Errors are exceptional scenarios that are out of scope of application and it’s not possible to anticipate and recover from them, for example hardware failure, JVM crash or out of memory error. 
* Checked Exceptions are exceptional scenarios that we can anticipate in a program and try to recover from it, for example FileNotFoundException. We should catch this exception and provide useful message to user and log it properly for debugging purpose. Exception is the parent class of all Checked Exceptions. 
* Runtime Exceptions are caused by bad programming, for example trying to retrieve an element from the Array. We should check the length of array first before trying to retrieve the element otherwise it might throw ArrayIndexOutOfBoundException at runtime. RuntimeException is the parent class of all runtime exceptions.

<p align="center">
  <img src="./Images/exception_hierarchy.png" />
</p>

<br></br>



## 10. Java 8 新功能
### 10.1 Lambda(learn from Scala)
What it does is that it reduces the code where it is obvious, such as in an anonymous innerclass. So, a thread can be changed as:

```java
Runnable oldRunner = new Runner() {
    public void run() {
        // do something
    }
}

Runnable newRunner = () -> {
    // do something
}
```


### 10.2 Data/Time changes
The Date/Time API is moved to `java.time` package. Another goodie is that most classes are Threadsafe and immutable.


### 10.3 Parallel Array Sorting
This feature adds the same set of sorting operations currently provided by the Arrays class, but with a parallel implementation that utilizes the Fork/Join framework. Additional utility methods were added to java.util.Arrays that use the JSR 166 Fork/Join parallelism common pool to provide sorting of arrays in parallel. The methods are called `parallelSort()` and are overloaded for all the primitive data types and Comparable objects.


### 10.4 Concurrency
`ConcurrentHashMap` class introduces over 30 new methods in this release. These include various forEach methods (forEach, forEachKey, forEachValue, and forEachEntry), and search methods (search, searchKeys, searchValues, and searchEntries).


### 10.5 Performance Improvement for HashMaps with Key Collisions
Hash bins containing a large number of colliding keys improve performance by storing their entries in a balanced tree instead of a linked list. This JDK 8 change applies only to HashMap, LinkedHashMap, and ConcurrentHashMap.

<br></br>



## 11. Comparable vs Comparator
### 11.1 Comparable

Comparable interface should be implemented by any custom class if we want to use Arrays or Collections sorting methods. 
Comparable interface has `compareTo(T obj)` method which is used by sorting methods. We should override this method in such a way that it returns a negative integer, zero, or a positive integer if “this” object is less than, equal to, or greater than the object passed as argument.


### 11.2 Comparator

But, in most real life scenarios, we want sorting based on different parameters. For example, as a CEO, I would like to sort the employees based on Salary, an HR would like to sort them based on the age. This is the situation where we need to use Comparator interface because Comparable.
`compareTo(Object o)` method implementation can sort based on one field only and we can’t chose the field on which we want to sort the Object. 
Comparator interface `compare(Object o1, Object o2)` method needs to be implemented that takes two Object argument, it should be implemented in such a way that it returns negative int if first argument is less than the second one and returns zero if they are equal and positive int if first argument is greater than second one.

<br></br>



## 12. methods of Object?
toString, equals. getClass, hashCode (return Hash Code), notify, notifyAll, wait

<br></br>



## 13. final vs finally vs finalize
final and finally are keywords in java whereas finalize is a method. 
final keyword can be used with class variables so that they can’t be reassigned, with class to avoid extending by classes and with methods to avoid overriding by subclasses, finally keyword is used with try-catch block to provide statements that will always get executed even if some exception arises, usually finally is used to close resources. finalize() method is executed by Garbage Collector before the object is destroyed, it’s great way to make sure all the global resources are closed.

<br></br>



## 14. 父类和子类的构造函数
``` java
class Super {
    String s;

    public Super(String s) {
        this.s = s;
    }
}

public class Sub extends Super {
    int x = 200;
    public Sub(String s) {} // error
    public Sub(){} // error
}
```

&#12288;&#12288;出现编译错误因为默认的父类构造函数未定义。在Java中，如果一个类没有定义构造函数，编译器会默认插入一个默认的无参数构造函数。如果程序员定义构造函数，编译器将不插入默认的无参数构造函数。上面的代码由于自定义了有参数的构造函数，编译器不再插入无参数的构造函数。子类的构造函数，无论是有参数或无参数，都将调用父类无参构造函数。当子类需要父类的无参数构造函数的时候，就发生了错误。解决这个问题，可以增加一个父类构造函数:
``` java
public Super() {}
```

<br></br>



## 15. 逻辑运算符
&#12288;&#12288;&，两个操作数中位都为1则为1，否则0. a & b
&#12288;&#12288;|, 两个位只要有一个为1就是1。a | b
&#12288;&#12288;~,自反。~a
&#12288;&#12288;^,异或，两个操作数的位相同则为0，否则为1. a ^ b

<br></br>



## 16. 常用类型占用的字节数

| 类型  | 存储需求 | bit数  |             取值范围/备注             |
| :----: | :------:| :--: | :------------------------------: |
| byte  | 1字节   |  1 * 8   | $$ -2^{31} $$ ~ $$ 2^{31} - 1 $$ |
| short | 2字节   |  2 * 8  |  $$ -2^{15} $$ ~ $$ 2^{15} - 1 $$ |
| int   |   4字节 | 4 * 8   | $$ -2^{63} $$ ~ $$ 2^{63} - 1 $$ |
| long   |   8字节 | 8 * 8   | $$ -2^{7} $$ ~ $$ 2^{7} - 1 $$ |
| float  | 4字节   |  4 * 8   | float类型的数值有一个后缀F(例如：3.14F) |
| double | 8字节   |  8 * 8  |  没有后缀F的浮点数值(如3.14)默认为double |
| char | 2字节   |  2 * 8  |    |       
| boolean | 1字节   |  1 * 8  |    |
