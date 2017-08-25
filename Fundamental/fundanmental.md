# <center>Fundanmental</center>

<br></br>



## 常用类型
----

| 类型  | 存储需求 | bit数  |    存储数据量     |        取值范围   |
| :--: | :------:| :---: | :--------------: | :-----------------: |
| byte  | 1字节  | 1 * 8  |  $$256=2^{8}$$   | -128 ~ 127         |
| short | 2字节  | 2 * 8  | $$65536=2^{16}$$ | $$-32768 = -2^{15}$$ ~ $$32767 = 2^{15} - 1$$ |
| int   |  4字节 | 4 * 8  | $$2^{32}$$       | $$-2^{31}$$ ~ $$2^{31} - 1$$ (0x8000 0000 ~ 0x7FFF FFFF) |
| long   |  8字节 | 8 * 8 | $$2^{64}$$       | $$-2^{63}$$ ~ $$2^{63} - 1 $$ |
| float  | 4字节  | 4 * 8 |                  | float类型数值有一个后缀F(例如：3.14F) |
| double | 8字节  | 8 * 8 |                  |                    |
| char   | 2字节  | 2 * 8 |                  |                      |
| boolean | 1字节 | 1 * 8 |                  |                     |

``` java
for(int i=0;i<t;i++) {
                long x=sc.nextLong();
                System.out.println(x+" can be fitted in:");
                if(x>=-128 && x<=127)
                	System.out.println("* byte");
                if(x >= -Math.pow(2, 15) && x <= Math.pow(2, 15) - 1)
                	System.out.println("* short");
                if(x >= -Math.pow(2, 31) && x <= Math.pow(2, 31) - 1)
                	System.out.println("* int");
                if(x >= -Math.pow(2, 63) && x <= Math.pow(2, 63) - 1)
                	System.out.println("* long");
        }
```

ASCII码：
``` java
public class ASCII {
	public static void main(String[] args) {
		System.out.println('A' - 65);
		System.out.println('Z' - 90);
		System.out.println('a' - 97);
	}
}
```

<br></br>



## Object Methods
----
* toString
* equals 
* getClass
* hashCode
* notify
* notifyAll
* wait

<br></br>



## 逻辑运算符
----
* `&`，两个操作数中位都为1则为1，否则0。
* `|`, 两个位只要有一个为1就是1。
* `~`，自反。~a
* `^`，异或，两个操作数的位相同则为0，否则为1。

<br></br>



## newInstance与new
----
* `newInstance()`是方法，`new`是关键字
* 创建对象的方式不一样，前者是使用类加载机制，后者是创建一个新类。 
* 使用`new`创建类时，这个类可以没有被加载。但用`newInstance()`方法须保证已加载且这个类已经连接。而完成上面两个步骤的是`Class`的静态方法`forName()`，调用了启动类加载器，即加载Java API的那个加载器。 
* `newInstance()`是把`new`分解为两步，先调用`Class`加载方法加载某个类，然后实例化。 

<br></br>



## for vs foreach
----
1. foreach遍历集合时，集合须实现Iterator接口
2. foreach遍历集合时不能向集合增加删除元素，否则会抛出ConcurrentModificationException异常。
3. for效率最好，尤其是实现RandomAccess接口的collection。但在LinkedList中，Iterator效果更好；foreach效率最差，因为其实现一个Enum，每次都要调用。
4. foreach遍历对象时可以修改对象属性值，但不能修改对象引用。

修改基本类型的值（原集合中的值没有变化，因为`str`是集合中变量的一个副本）：

``` java
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
    public String name;
    public int age;
    
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
            f.age(f.age * 2);
        }
        System.out.println(list);
    }
}
```

<br></br>



## 面向对象特征
---- 
1. 抽象(abstraction)：抽象是忽略一个主题中与当前目标无关的那些方面，以便更充分地注意与当前目标有关的方面。
2. 继承(inheritance)
3. 封装(encapsulation)：封装是把过程和数据包围起来，对数据的访问只能通过已定义的界面。
4. 多态(polymorphism)：允许不同类的对象对同一消息作出响应。多态性包括参数化多态性和包含多态性。

<br></br>



## Composition
----
通过在类中直接使用另一个类的对象来达到重用代码的目的。Composition通常用在新类中使用已有类的功能，而不是使用已有类的接口时。这时，可以在新类中嵌入已有类的对象，完成想要的功能。Composition通常表述为“has a”的关系。

One of the best practices is to “favor composition over inheritance”:
1. Any change in the superclass might affect subclass. If we have a method `test()` in subclass and somebody introduces a method `test()` in superclass, we will get compilation errors in subclass. Composition will never face this issue because we are using only what methods we need.

2. Inheritance exposes all the super class methods and variables to client, it can lead to security holes. Composition allows us to provide restricted access to the methods and hence more secure.

3. We can get runtime binding in composition where inheritance binds the classes at compile time. So composition provides flexibility in invocation of methods.

什么时候该用继承，什么时候该用组合？
1. 如果存在一种IS-A的关系（比如Bee“是一个”Insect），且一个类需要向另一个类暴露所有的方法接口，那么更应该用继承的机制。
2. 如果存在一种HAS-A的关系（比如Bee“有一个”attack功能），那么更应该运用组合。

<br></br>



## Java 8 新功能
----
### Parallel Streams
* Streams is about possibly-parallel, aggregate operations on datasets
* Sources can be collections, arrays, generator functions, IO…
* Pipelines built from basic primitives – filter, map, reduce, sort
* All operations can be executed in parallel
* Couldn’t get to a library like this without lambdas

<p align="center">
  <img src="./Images/jdk8_streams_example.png" width="800" />
</p>

<br>


### Lambda(from Scala)

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

<br>


### Data/Time changes
Most classes are thread safe and immutable.

<br>


### Parallel Array Sorting
This feature adds the same set of sorting operations currently provided by the Arrays class, but with a parallel implementation that utilizes the Fork/Join framework. The methods are called `parallelSort()` and are overloaded for all the primitive data types and Comparable objects.

<br>


### Concurrency
`ConcurrentHashMap` class introduces over 30 new methods in this release. These include various forEach methods (`forEach`, `forEachKey`, `forEachValue`, and `forEachEntry`), and search methods (`search`, `searchKeys`, `searchValues`, and `searchEntries`).

<br>


### Performance Improvement for HashMaps with Key Collisions
Hash bins containing a large number of colliding keys improve performance by storing their entries in a balanced tree instead of a linked list. JDK 8 change applies only to HashMap, LinkedHashMap, and ConcurrentHashMap.

<br></br>



## Comparable vs Comparator
----
### Comparable
Comparable interface should be implemented by any custom class if we want to use Arrays or Collections sorting methods. 

Comparable interface has `compareTo(T obj)` method which is used by sorting methods. We should override this method.

<br>


### Comparator
`compareTo(Object o)` method implementation can sort based on one field only and we can’t chose the field on which we want to sort the Object. Comparator interface `compare(Object o1, Object o2)` method needs to be implemented that takes two Object argument.


<br>


### Example

```java
public class PersonComparable implements Comparable<PersonComparable>{
    private int age;
    private String name;
    
    public int compareTo(PersonComparable another) {
        if (another != null)
            return age - another.getAge();
        else
            throw new NullPointerException();
   }
}

public static void main(String[] args) {
        PersonComparable[] pArr = new PersonComparable[2];
        pArr[0] = new PersonComparable("test1", 26);
        pArr[1] = new PersonComparable("test2", 19);
        Arrays.sort(pArr);
}
```

<br>


### Comparator Example

```java
public class PersonComparator implements Comparator<Person>{
    public int compare(Person p1, Person p2){
        if(p1 != null && p2 != null)
            return p1.getAge() - p2.getAge();
        else
            throw new NullPointerException();
    }
}

public static void main(String[] args) {
        Person[] pArr = new Person[2];
        pArr[0] = new Person("test1", 26);
        pArr[1] = new Person("test2", 19);
        Arrays.sort(pArr, new PersonComparator());
}
```

<br></br>



## Shallow Copy and Deep Copy
----
### Shallow Copy

![ShallowCopy](./Images/shallow_copy.png)

浅拷贝是按位拷贝对象，会创建一个新对象。如果原始对象属性是基本类型，拷贝的就是基本类型的值；如果原始对象属性是内存地址（引用类型），拷贝的就是内存地址 ，因此如果其中一个对象改变了这个地址，就会影响到另一个对象。

图中，`SourceObject`有一个int类型的属性 `field1`和一个引用类型属性`refObj`。对`SourceObject`浅拷贝时，创建了`CopiedObject`。由于`field1`是基本类型，所以将它的值拷贝给`field2`，但由于`refObj`是引用类型, 所以`CopiedObject`指向`refObj`相同的地址。因此对`SourceObject`中的`refObj`所做的任何改变都会影响到`CopiedObject`。

```java
public class Subject {
    private String name; 

    public Subject(String s) { 
      name = s; 
   } 
}

public class Student implements Cloneable { 
   private Subject subj;  // 对象引用
   private String name; 

   public Student(String s, String sub) { 
      name = s; 
      subj = new Subject(sub); 
   } 

   // 重写clone()方法, 浅拷贝
   public Object clone() { 
      try { 
         return super.clone();  // 直接调用父类的clone()方法
      } catch (CloneNotSupportedException e) { 
         return null; 
      } 
   } 
}
```

<br>


### Deep Copy
深拷贝会拷贝所有的属性，并拷贝属性指向的动态分配的内存。当对象和它所引用的对象一起拷贝时即发生深拷贝。深拷贝相比于浅拷贝速度较慢并且花销较大。

![DeepCopy](./Images/deep_copy.png)

```java
public class Student implements Cloneable { 
   private Subject subj;  // 对象引用 
   private String name; 

   public Student(String s, String sub) { 
      name = s; 
      subj = new Subject(sub); 
   } 

   // 重写clone()方法 
   public Object clone() { 
      // 深拷贝，创建拷贝类的一个新对象，这样就和原始对象相互独立
      Student s = new Student(name, subj.getName()); 
      return s; 
   } 
}
```

<br>


### 通过序列化深拷贝
通过序列化进行深拷贝时，必须确保对象图中所有类都是可序列化的。

<br>


### 延迟拷贝
是浅拷贝和深拷贝的组合。当最开始拷贝一个对象时，会使用速度较快的浅拷贝，还会用计数器记录有多少对象共享这个数据。当程序想要修改原始的对象时，它会决定数据是否被共享（通过检查计数器）并根据需要进行深拷贝。 

延迟拷贝从外面看起来就是深拷贝，但是只要有可能它就会利用浅拷贝的速度。当原始对象中的引用不经常改变的时候可以使用延迟拷贝。由于存在计数器，效率下降很高，但只是常量级的开销。而且, 在某些情况下, 循环引用会导致一些问题。

<br></br>



## FAQ 
---- 
### final vs finally vs finalize
* `final` and `finally` are keywords in java whereas `finalize` is a method. 
* `final` keyword can be used with class variables so that they can’t be reassigned, with class to avoid extending by classes and with methods to avoid overriding by subclasses. `finally` keyword is used with try-catch block. `finalize` method is executed by Garbage Collector before the object is destroyed.

`final`使用效果：
* Class: no other class can extend it, for example we can’t extend String class. 
* Method: child classes can’t override it.
* Variable: 如果是基本数据类型，则数值在初始化后不能更改；如果是引用类型，则初始化之后不能指向另一个对象。
* Java interface variables are by default final and static.

<br>


### static
* 变量: A static variable is a class variable and doesn’t belong to Object/instance of the class.

* 方法: Same as static variables, static methods belong to class and not to class instances. A static method can access only static variables of class and invoke only static methods of the class.即使没有声明为static，类的构造器也是静态方法。

* Block: is the group of statements that gets executed when the class is loaded into memory by Java ClassLoader.

* Class: We can use static keyword with nested classes. static keyword can’t be used with top-level classes.

<br>


### Marker Interface
A marker interface is an empty interface without any method but used to force some functionality in implementing classes by Java. Some of the well known marker interfaces are Serializable and Cloneable.

<br>


### Override and Overload
> Override重写，即子类对父类； Overload重载，即一个类里面。

重写Override规则：
* 参数列表必须完全与被重写方法的相同；
* 返回类型必须完全与被重写方法的返回类型相同；
* 访问权限不能比父类中被重写的方法的访问权限更低。
* 父类的成员方法只能被它的子类重写。
* 声明为final和static的方法不能被重写。
* 重写的方法能够抛出任何非强制异常，无论被重写的方法是否抛出异常。但是，重写的方法不能抛出新的强制性异常，或者比被重写方法声明的更广泛的强制性异常，反之则可以。
* 构造方法不能被重写。
* 如果不能继承一个方法，则不能重写这个方法。

重载Overload规则：
* 被重载的方法必须改变参数列表；
* 被重载的方法可以改变返回类型；
* 被重载的方法可以声明新的或更广的检查异常；
* 方法能够在同一个类中或者在一个子类中被重载。

重写与重载区别：

区别点      |    重载Overload | 重写Override  
-------- | -------- | ----------
参数列表  | 必须修改 |  不能修改   
返回类型 |   可以修改 |  不能修改 
异常      |    可以修改 | 可以减少，但不能抛出新的或更广的异常  
访问      |    可以修改 | 可以降低限制，但不能做更严格限制  

<br>


### Java Compiler is stored in JDK, JRE or JVM?
The task of java compiler is to convert java program into bytecode, we have javac executable for that. So it must be stored in JDK, we don’t need it in JRE and JVM is just the specs.

<br>


### Collection和Collections
* Collection是集合类的上级接口，继承与他的接口主要有Set和List。
* Collections是针对集合类的一个帮助类，他提供一系列静态方法实现对各种集合的搜索、排序、线程安全化等操作。

<br>
