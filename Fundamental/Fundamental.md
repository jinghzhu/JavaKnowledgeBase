# <center>Java Fundamental</center>

## 1. JDK, JVM and JRE
* Java Development Kit (JDK) is for development purpose and JVM is a part of it to execute the java programs. JDK provides all the tools, executables and binaries required to compile, debug and execute a Java Program. The execution part is handled by JVM to provide machine independence.
* Java Runtime Environment (JRE) is the implementation of JVM. JRE consists of JVM and java binaries and other classes to execute any program successfully. JRE doesn’t contain any development tools like java compiler, debugger etc. The task of java compiler is to convert java program into bytecode, we have javac executable for that. So it must be stored in JDK, we don’t need it in JRE and JVM is just the specs.


## 2. What does super keyword do? 
We can use super keyword to invoke super class constructor in child class constructor but in this case it should be the first statement in the constructor method.

```java
public class SuperClass { 
    public SuperClass(){ 
    } 

    public void test(){ 
        System.out.println("super class test method"); 
    } 
} 
```

Use of super keyword can be seen in below child class implementation.

```java
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


## 3. Can we overload main method? 
Yes, we can have multiple methods with name “main” in a single class. However if we run the class, java runtime environment will look for main method with syntax as `public static void main(String args[])`.



## 5. newInstance与new区别
* newInstance()是方法，new是关键字一个是方法
* 创建对象的方式不一样，前者是使用类加载机制，后者是创建一个新类。 
* 从JVM的角度看，使用关键字new创建一个类的时候，这个类可以没有被加载。但是使用newInstance()方法就须保证：1、这个类已经加载；2、这个类已经连接了。而完成上面两个步骤的正是Class的静态方法forName()所完成的，这个静态方法调用了启动类加载器，即加载java API的那个加载器。 
* newInstance()实际上是把new分解为两步，即首先调用Class加载方法加载某个类，然后实例化。这样分步的好处是显而易见的。我们可以在调用class的静态加载方法forName时获得更好的灵活性，提供给了一种降耦的手段。 



## 6. JAVA范型
* 泛型的类型参数只能是类类型（包括自定义类），不能是简单类型。
* 泛型的参数类型可以使用extends语句，例如`<T extends superclass>`。
* 泛型的参数类型还可以是通配符类型。例如`Class<?> classType = Class.forName(java.lang.String);`

### 6.1 范型类
```java
public class Gen<T> {
    private T t;
    public Gen(T t) {
        this.t = t;
    }
}

Gen<String> gen1=new Gen<String>("");
Gen<Integer> gen2=new Gen<Integer>(1);
```

### 6.2范型方法
&#12288;&#12288;必须在方法的修饰符（public, static, final, abstract）之后，返回值声明之前。
正确：`public static <E> void printArray(E[] a)`
错误：`public static void <E> printArray(E[] a)`


## 7. What is final keyword? 
* Class: no other class can extend it, for example we can’t extend String class. 
* Method: child classes can’t override it.
* Variable: 对于一个`final`变量，如果是基本数据类型的变量，则其数值一旦在初始化之后便不能更改；如果是引用类型的变量，则在对其初始化之后便不能再让其指向另一个对象。


## 8. What is static keyword? 
&#12288;&#12288;static作用是方便在未创建对象的情况下来调用方法／变量。
* Variable: static keyword can be used with class level variables to make it global i.e. all the objects will share the same variable. 
* Method: a static method can access only static variable of class and invoke only static methods of the class，但反过来是可以的.而且可以在没创建任何对象的前提下，仅通过类本身来调用static方法。static方法就是没有this的方法.另外，即使没有显示声明为static，类的构造器也是静态方法。
* Block: is the group of statements that gets executed when the class is loaded into memory by Java ClassLoader. It is used to initialize static variables of the class. Mostly it’s used to create static resources when class is loaded.


## 9. 接口
接口有以下特性：
* 接口是隐式抽象的，当声明一个接口的时候，不必使用abstract关键字。
* 接口中每一个方法也是隐式抽象的，声明时同样不需要abstract关键子。
* 接口中的方法都是公有的。
* 当类实现接口的时候，类要实现接口中所有的方法。否则，类必须声明为抽象的类。


## 10. 抽象方法
&#12288;&#12288;如果你想设计这样一个类，该类包含一个特别的成员方法，该方法的具体实现由它的子类确定，那么你可以在父类中声明该方法为抽象方法。Abstract关键字同样可以用来声明抽象方法，抽象方法只包含一个方法名，而没有方法体。
&#12288;&#12288;抽象方法没有定义，方法名后面直接跟一个分号，而不是花括号。

```java
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

&#12288;&#12288;如果Salary类继承了Employee类，那么它必须实现`computePay()`方法：
```java
public class Salary extends Employee {
   private double salary; // Annual salary
  
   public double computePay() {
      System.out.println("Computing salary pay for " + getName());
      return salary/52;
   }
}
```


## 11. JAVA 使用相对路径读取文件
&#12288;&#12288;java project环境，使用java.io用相对路径读取文件的例子：
&#12288;&#12288;目录结构：
  DecisionTree
            |___src
                 |___com.decisiontree.SamplesReader.java
            |___resource
                 |___train.txt,test.txt
&#12288;&#12288;SamplesReader.java:

```java
String filepath="resource/train.txt"; //注意filepath的内容；
File file=new File(filepath);
```

&#12288;&#12288;java.io默认定位到当前用户目录下，即：工程根目录"`D:\DecisionTree`下.因此，此时的相对路径为`resource/train.txt`。这样，JVM就可以根据"user.dir"与"resource/train.txt"得到完整的路径（即绝对路径）`D:\DecisionTree\resource\train.txt`，找到train.txt文件。


## 12. Override重写，即子类对父类； Overlode重载，即一个类里面
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


## 13. Is Java pass by value or pass by reference?
&#12288;&#12288;java中方法参数传递方式是按值传递。如果参数是基本类型，传递的是基本类型的字面量值的拷贝。如果参数是引用类型，传递的是该参量所引用的对象在堆中地址值的拷贝。

```java 
public class Balloon { 
    private String color;
 
    public Balloon(){}
     
    public Balloon(String c){
        this.color=c;
    }
     
    public String getColor() {
        return color;
    }
 
    public void setColor(String color) {
        this.color = color;
    }
}
```

```java
public class Test {
 
    public static void main(String[] args) {
        Balloon red = new Balloon("Red"); //memory reference 50
        Balloon blue = new Balloon("Blue"); //memory reference 100
         
        swap(red, blue);
        System.out.println("red color="+red.getColor());
        System.out.println("blue color="+blue.getColor());
         
        foo(blue);
        System.out.println("blue color="+blue.getColor());      
    }
 
    private static void foo(Balloon balloon) { //baloon=100
        balloon.setColor("Red"); //baloon=100
        balloon = new Balloon("Green"); //baloon=200
        balloon.setColor("Blue"); //baloon = 200
    }
 
    //Generic swap method
    public static void swap(Object o1, Object o2){
        Object temp = o1;
        o1=o2;
        o2=temp;
    }
}
```

When we execute above program, we get following output.
> red color=Red

> blue color=Blue

> blue color=Red

If you look at the first two lines of the output, it’s clear that swap method didn’t worked. This is because Java is pass by value, this `swap()` method test can be used with any programming language to check whether it’s pass by value or pass by reference.

Let’s analyze the program execution step by step.

```java
Balloon red = new Balloon("Red");
Balloon blue = new Balloon("Blue");
```

When we use `new` operator to create an instance of a class, the instance is created and the variable contains the reference location of the memory where object is saved. For our example, let’s assume that `red` is pointing to `50` and `blue` is pointing to `100` and these are the memory location of both Balloon objects.

Now when we are calling `swap()`, two new variables `o1` and `o2` are created pointing to `50` and `100` respectively.

So below code snippet explains what happened in the `swap()` method execution.

```java
public static void swap(Object o1, Object o2){ //o1=50, o2=100
    Object temp = o1; //temp=50, o1=50, o2=100
    o1=o2; //temp=50, o1=100, o2=100
    o2=temp; //temp=50, o1=100, o2=50
} //method terminated
```

Notice that we are changing values of `o1` and `o2` but they are copies of `red` and `blue` reference locations, so actually there is no change in the values of `red` and `blue` and hence the output.

Now let’s analyze `foo()` method execution.

```java
private static void foo(Balloon balloon) { //baloon=100
    balloon.setColor("Red"); //baloon=100
    balloon = new Balloon("Green"); //baloon=200
    balloon.setColor("Blue"); //baloon = 200
}
```

The first line is the important one, when we call a method the method is called on the Object at the reference location. At this point, balloon is pointing to `100` and hence it’s color is changed to Red.

In the next line, ballon reference is changed to `200` and any further methods executed are happening on the object at memory location `200` and not having any effect on the object at memory location `100`. This explains the third line of our program output printing blue color=Red.


## 14. 作用域   

修饰词   | 当前类      |    同一package | 子孙类 | 其他package
:--------: | :--------: | :----------: | :----: |
public  | √ |  √ |  √ | √ 
protected  | √ |  √ | √| × 
friendly   | √ | √ | × | × 
private    | √ | × | × | × 


## 15. What are the Exception Handling Keywords in Java? 
There are four keywords used in java exception handling. 
1. throw: Sometimes we explicitly want to create exception object and then throw it to halt the normal processing of the program. throw keyword is used to throw exception to the runtime to handle it. 
2. throws: When we are throwing any checked exception in a method and not handling it, then we need to use throws keyword in method signature to let caller program know the exceptions that might be thrown by the method. The caller method might handle these exceptions or propagate it to its caller method using throws keyword. We can provide multiple exceptions in the throws clause and it can be used with main() method also. 
3. try-catch: We use try-catch block for exception handling in our code. try is the start of the block and catch is at the end of try block to handle the exceptions. We can have multiple catch blocks with a try and try-catch block can be nested also. catch block requires a parameter that should be of type Exception. 
4. finally: finally block is optional and can be used only with try-catch block. Since exception halts the process of execution, we might have some resources open that will not get closed, so we can use finally block. Finally block gets executed always, whether exception occurrs or not. 


## 16. Explain JAVA Exception Hierarchy
* Errors are exceptional scenarios that are out of scope of application and it’s not possible to anticipate and recover from them, for example hardware failure, JVM crash or out of memory error. 
* Checked Exceptions are exceptional scenarios that we can anticipate in a program and try to recover from it, for example FileNotFoundException. We should catch this exception and provide useful message to user and log it properly for debugging purpose. Exception is the parent class of all Checked Exceptions. 
* Runtime Exceptions are caused by bad programming, for example trying to retrieve an element from the Array. We should check the length of array first before trying to retrieve the element otherwise it might throw ArrayIndexOutOfBoundException at runtime. RuntimeException is the parent class of all runtime exceptions.

![ExceptionHierarchu](./Images/exception_hierarchy.png)


## 17. Java 8 新功能
### 17.1 Lambda(learn from Scala)
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

### 17.2 Data/Time changes
The Date/Time API is moved to `java.time` package. Another goodie is that most classes are Threadsafe and immutable.

### 17.3 Parallel Array Sorting
This feature adds the same set of sorting operations currently provided by the Arrays class, but with a parallel implementation that utilizes the Fork/Join framework. Additional utility methods were added to java.util.Arrays that use the JSR 166 Fork/Join parallelism common pool to provide sorting of arrays in parallel. The methods are called `parallelSort()` and are overloaded for all the primitive data types and Comparable objects.

### 17.4 Concurrency
The `ConcurrentHashMap` class introduces over 30 new methods in this release. These include various forEach methods (forEach, forEachKey, forEachValue, and forEachEntry), and search methods (search, searchKeys, searchValues, and searchEntries).

### 17.5 Performance Improvement for HashMaps with Key Collisions
Hash bins containing a large number of colliding keys improve performance by storing their entries in a balanced tree instead of a linked list. This JDK 8 change applies only to HashMap, LinkedHashMap, and ConcurrentHashMap.


## 18. 字和字节的区别 
Byte(字节) = 8 bits


## 19. 


## 20. Comparable vs Comparator
* Comparable

Comparable interface should be implemented by any custom class if we want to use Arrays or Collections sorting methods. 
Comparable interface has `compareTo(T obj)` method which is used by sorting methods. We should override this method in such a way that it returns a negative integer, zero, or a positive integer if “this” object is less than, equal to, or greater than the object passed as argument.

* Comparator

But, in most real life scenarios, we want sorting based on different parameters. For example, as a CEO, I would like to sort the employees based on Salary, an HR would like to sort them based on the age. This is the situation where we need to use Comparator interface because Comparable.
`compareTo(Object o)` method implementation can sort based on one field only and we can’t chose the field on which we want to sort the Object. 
Comparator interface `compare(Object o1, Object o2)` method needs to be implemented that takes two Object argument, it should be implemented in such a way that it returns negative int if first argument is less than the second one and returns zero if they are equal and positive int if first argument is greater than second one.


## 21. methods of Object?
toString, equals. getClass, hashCode (return Hash Code), notify, notifyAll, wait


## 22. for vs foreach
1. 使用foreach来遍历集合时，集合必须实现Iterator接口，foreach就是使用Iterator接口来实现对集合的遍历的
2. 在用foreach循环遍历一个集合时不能向集合中增加元素，不能从集合中删除元素，否则会抛出ConcurrentModificationException异常。抛出该异常是因为在集合内部有一个modCount变量用于记录集合中元素的个数，当向集合中增加或删除元素时，modCount也会随之变化，在遍历开始时会记录modCount的值，每次遍历元素时都会判断该变量是否发生了变化，如果发生了变化则抛出ConcurrentModificationException异常
3. for效率最好，尤其是实现RandomAccess接口的collection，但在linkedlist中，iterator效果更好;foreach效率最差，因为其实现一个Enum，每次都要掉用。
4. 当使用foreach循环基本类型时变量时不能修改集合中的元素的值，遍历对象时可以修改对象的属性的值，但是不能修改对象的引用
修改基本类型的值（原集合中的值没有变化，因为str是集合中变量的一个副本）：

```java
    public static void main(String[] args) {
        List<String> list = new ArrayList<String>();
        list.add("0");
        list.add("1");
        list.add("2");
        for(String str:list){
            str=str+"0";
            System.out.println(str);
        }
        System.out.println(list);
    }

// 修改对象的值（可以修改，因为f是一个指针）：
public class ForE {
    private String name;
    private int age;
    
    public ForE(String name, int age) {
        super();
        this.name = name;
        this.age = age;
    }

    public static void main(String[] args) {
        List<ForE> list = new ArrayList<ForE>();
        list.add(new ForE("apple", 10));
        list.add(new ForE("banana", 20));
        list.add(new ForE("orange", 30));
        for(ForE f:list){
            f.setAge(f.getAge()*2);
        }
        System.out.println(list);
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    @Override
    public String toString() {
        return "ForE [name=" + name + ", age=" + age + "]";
    }
    
}
```


## 23. 为何需要序列化
&#12288;&#12288;Java允许在内存中创建可复用的Java对象，但只有当JVM处于运行时，这些对象才可能存在，即这些对象生命周期不会比JVM的生命周期更长。但在现实应用中，要求JVM停止后能够保存(持久化)对象。Java对象序列化就能够帮助我们实现该功能。当使用RMI(远程方法调用)或网络中传递对象时，都会用到对象序列化。

&#12288;&#12288;使用序列化保存对象时，把其状态保存为一组字节。在未来，再将这些字节组装成对象。对象序列化保存的是对象的"状态"，即它的成员变量。由此可知，对象序列化不会关注类中的静态变量。


## 24. final vs finally vs finalize
final and finally are keywords in java whereas finalize is a method. 
final keyword can be used with class variables so that they can’t be reassigned, with class to avoid extending by classes and with methods to avoid overriding by subclasses, finally keyword is used with try-catch block to provide statements that will always get executed even if some exception arises, usually finally is used to close resources. finalize() method is executed by Garbage Collector before the object is destroyed, it’s great way to make sure all the global resources are closed.



## 25. Can we have try without catch block? 
Yes, we can have try-finally statement and hence avoiding catch block.


## 26. 空字符
&#12288;&#12288;Java没有空字符`’’`，只有`(Character) null`


## 27. 自定义Exception
```java
public class custExpetion1 {
    public static void main(String[] args) {
        int i = 1, j = -1;
        try{
            if(i < 0 || j < 0){
                throw new Exception("error");
            }
        }catch(Exception e){
            System.out.println(e.getMessage());;
        }
        
        Calculator myCalculator = new Calculator();
        try{
            int ans=myCalculator.power(i,j);
            System.out.println(ans);
            
        }
        catch(Exception e)
        {
            System.out.println(e.getMessage());
        }
    }
}
```

```java
class Calculator{
    public int power(int i, int j) throws Exception{
        if(i < 0 || j < 0){
            throw new Exception("n and p should be non-negative");
        }
        …
    }
}
```


## 28. 面向对象的特征有哪些方面 
1. 抽象(abstraction)：抽象是忽略一个主题中与当前目标无关的那些方面，以便更充分地注意与当前目标有关的方面。包括两个方面，一是过程抽象，二是数据抽象。
2. 继承(inheritance)
3. 封装(encapsulation)：封装是把过程和数据包围起来，对数据的访问只能通过已定义的界面。面向对象计算始于这个基本概念，即现实世界可以被描绘成一系列完全自治、封装的对象，这些对象通过一个受保护的接口访问其他对象。
4.  多态性(polymorphism)：指允许不同类的对象对同一消息作出响应。多态性包括参数化多态性和包含多态性。多态性语言具有灵活、抽象、行为共享、代码共享的优势，很好的解决了应用程序函数同名问题。


## 29. 父类和子类的构造函数
```java
class Super {
    String s;

    public Super(String s) {
        this.s = s;
    }
}

public class Sub extends Super {
    int x = 200;

    public Sub(String s) {
    } // error

    public Sub(){
    } // error
}
```

&#12288;&#12288;出现编译错误因为默认的父类构造函数未定义。在Java中，如果一个类没有定义构造函数，编译器会默认插入一个默认的无参数构造函数。如果程序员定义构造函数，编译器将不插入默认的无参数构造函数。上面的代码由于自定义了有参数的构造函数，编译器不再插入无参数的构造函数。子类的构造函数，无论是有参数或无参数，都将调用父类无参构造函数。当子类需要父类的无参数构造函数的时候，就发生了错误。解决这个问题，可以增加一个父类构造函数:
```java
public Super() {}
```


## 30. 逻辑运算符
&#12288;&#12288;&，两个操作数中位都为1则为1，否则0. a & b
&#12288;&#12288;|, 两个位只要有一个为1就是1。a | b
&#12288;&#12288;~,自反。~a
&#12288;&#12288;^,异或，两个操作数的位相同则为0，否则为1. a ^ b


## 31. JAVA vs C++
* JVM虚拟机内部自己管理指针
* 多重继承
* 自动内存管理
* 操作符重载
* JAVA不支持缺省函数参数，C++支持
* JAVA的try／catch方便处理异常，C++没有


## 32. final对象是否有默认值
```java
class Something{
    int i;
    public void doSomething(){
         System.out.println(“i = “ + i);
    }
}
```

Correct, the output is i = 0. because i is instant variable which has default value.

```java
      class Something{
         final int i;
         public void doSomething(){
               System.out.println(“i = “ + i);
         }
      }
```

Wrong, because the instant variable with keyword final has no default value which means we must set a value for it.


## 33.  Writer和Reader： 
&#12288;&#12288;Writer和Reader用于字符流的写入和读取，也就是说写入和读取的单位是字符，如果读与写的操作不涉及字符，那么是不需要Writer和Reader的。 
* Writer类（抽象类），子类必须实现的方法仅有 write()、flush()、close()。继承了此类的类是用于写操作的 “Writer”。 
* Reader 类（抽象类），子类必须实现的方法仅有 read()、close()。继承了此类的类是用于读操作的“Reader”。
* write()方法是将数据写入到文件（广义概念，包括字节流什么的）中。
* read()方法是将文件（广义概念，包括字节流什么的）中的数据读出到缓冲目标上。 
 


## 34. InputStream和OutputStream： 
* InputStream是表示字节输入流的所有类的超类。字节输入流相当于是一个将要输入目标文件的“流”。InputStream有 read() 方法而没有write()方法，因为它本身代表将要输入目的文件的一个“流” 
* OutputStream：此抽象类是表示输出字节流的所有类的超类。输出流接受输出字节并将这些字节发送到某个接收器。是从文件中将要输出到某个目标的“流”。OutputStream有 write()方法而没有read()方法。
* InputStreamReader是字节流通向字符流的桥梁：它使用指定的 charset 读取字节并将其解码为字符。它使用的字符集可以由名称指定或显式给定，或者可以接受平台默认的字符集。 
* OutputStreamWriter是字符流通向字节流的桥梁：可使用指定的 charset 将要写入流中的字符编码成字节。它使用的字符集可以由名称指定或显式给定，否则将接受平台默认的字符集。

&#12288;&#12288;BufferReader和BufferWriter：缓冲机制是说先把数据存到缓冲内存中，再一次写入文件，减少打开文件的消耗。


## 35. char型变量中能不能存贮一个中文汉字?为什么? 
&#12288;&#12288;是能够定义成为一个中文的，因为java中以unicode编码，一个char占16个字节，所以放一个中文是没问题的


## 36. 文件目录操作
### 36.1 如何列出某个目录下的所有文件? 
```java
File file = new File("e:\\总结"); 
File[] files = file.listFiles(); 
for(int i=0; i<files.length; i++){ 
    if(files[i].isFile()) System.out.println(files[i]);
}
```

### 36.2 如何列出某个目录下的所有子目录?
```java
File file = new File("e:\\总结"); 
File[] files = file.listFiles(); 
for(int i=0; i<files.length; i++){ 
    if(files[i].isDirectory()) 
        System.out.println(files[i]);
}
```

### 36.3 如何判断一个文件或目录是否存在?
&#12288;&#12288;创建`File`对象,调用其`exsit()`方法即可返回是否存在,如: 
``System.out.println(new File("d:\\t.txt").exists());``

### 36.4 如何读写文件? 
```java
//读文件:
FileInputStream fin = new FileInputStream("e:\\tt.txt"); 
byte[] bs = new byte[100];

while(true){ 
          int len = fin.read(bs);
          if(len <= 0) 
              break;
          System.out.print(new String(bs,0,len));
}

fin.close();
```

```java
//写文件:
FileWriter fw = new FileWriter("e:\\test.txt");

fw.write("hello world!" + System.getProperty("line.separator")); 
fw.write("你好!北京!"); 
fw.close(); 
```


## 37. 常用类型占用的字节数
（1）整型
类型              存储需求        bit数                  取值范围      
byte                 1字节           1*8      （-2的31次方到2的31次方-1）
short                2字节           2*8             －32768～32767
int                    4字节           4*8      （-2的63次方到2的63次方-1）
long                 8字节           8*8                 －128～127

（2）浮点型
类型              存储需求         bit数                                  备注
float                  4字节           4*8            float类型的数值有一个后缀F(例如：3.14F)
double              8字节           8*8         没有后缀F的浮点数值(如3.14)默认为double类型

（3）.char类型
类型              存储需求        bit数 
char                  2字节          2*8

（4）.boolean类型
类型              存储需求        bit数                  取值范围  
boolean           1字节           1*8                 false、true
