# <center>String</center>



<br></br>

* String池不属于堆和栈，属于常量池。
* StringBuilder和StringBuffer都继承自AbstractStringBuilder类，底层都是char数组操作。StringBuffer线程安全是所有方法都加Synchronized。

<br></br>



## String Pool
----
String Pool is possible only because String is immutable and it’s the implementation of String interning concept. 

> String pool is also example of Flyweight design pattern.

When we use double quotes to create a String(`String str = "test"`), it firstly looks for String with same value in String pool. If found, it returns the reference else it creates a new String in pool and then returns the reference.

However using `new` operator, we force String class to create a new String object in heap and then we can use `intern()` method to put it into pool or refer to other String object from pool having same value.

<br></br>



## String.intern()
----
When `intern()` is invoked, if the pool already contains a string equal to this String object as determined by the `equals()` method, then the string from pool is returned. Otherwise, this String object is added to pool and a reference to this String object is returned.

<br></br>



## 测试
----
### Case I
```java
public class Test {  
    public static void main(String[] args) {  
        String str = "abc";  
        String str1 = "abc";  
        String str2 = new String("abc");  
        System.out.println(str == str1);  
        System.out.println(str1 == "abc");  
        System.out.println(str2 == "abc");  
        System.out.println(str1 == str2);  
        System.out.println(str1.equals(str2));  
        System.out.println(str1 == str2.intern());  
        System.out.println(str2 == str2.intern());  
    }  
} 
``` 

栈内存保存基本数据类型和对象引用。String类本质是字符数组`char[]`，其次String类是`final`。Java运行时维护一个String池，池中String对象不可重复，没有创建，有则作罢。
 
```java 
String str = "abc";  
String str1= "abc";  
```

第一句在String池中创建一个对象`abc`，`str`指向池中对象`abc`。第二句，因为`abc`已存在String池，所以不再创建，则`str==str1`返回true。 

```java
String str2 = new String("abc"); 
```

创建2个String对象，而基于上面两句，只在栈内存创建`str2`引用，在堆上创建一个String对象，内容是`abc`，`str2`指向堆内存对象首地址。

`str2 == ”abc”`为false。因为，`abc`是位于String池中对象，而`str2`指堆内存String对象，`==`判断地址。但`str1.equals(str2)`是true。

`intern()`先在String池中查找是否存在一个对象，存在就返回String池中对象引用。本例中String池存在`abc`，则调用`intern()`时返回的是池中`abc`对象引用，那么和`str`，`str1`等同。和`str2`不同，因为`str2`指堆内存。 

<br>


### Case II

```java
public static void main(String[] args) {
    String str1 = "hello world";
    String str2 = new String("hellop world");
    String str3 = "hello world";
    String str4 = new String("hello world");

    System.out.println(str1 == str2); // false
    System.out.println(str1 == str3); // true
    System.out.println(str4 == str2); // false
}
```

`str1`和`str3`都在编译期间生成字面常量和符号引用，运行期间字面常量`"hello world"`存储在运行时常量池。通过这种方式将String对象跟引用绑定，JVM先在运行时常量池查找是否存在相同字面常量。如果存在，则将引用指向已存在字面常量；否则在运行时常量池开辟一个空间存储该字面常量，并将引用指向该字面常量。

通过`new`关键字生成对象是在堆进行。而堆对象生成过程是不会去检测该对象是否已存在。因此通过`new`创建对象，创建出的一定是不同的对象，即使字符串内容相同。

<br>


### Case III

```java
String a = "hello2"; 　　
String b = "hello" + 2; 　　
System.out.println(a == b); // true
```

`b`在编译期间已被优化成`"hello2"`，因此在运行期间，变量`a`和变量`b`指同一个对象。

<br>


### Case IV

```java
String a = "hello2"; 
String b = "hello";    
String c = b + 2;     
System.out.println(a == c); // false
```

由于有符号引用存在，所以`c`不会在编译期间被优化，不会把`b + 2`当做字面常量处理。因此生成的对象是在堆上。

<br>


### Case V

```java
String a = "hello2"; 
final String b = "hello"; 
String c = b + 2; 
System.out.println(a == c); // true
```

被`final`修饰的变量，会在class文件常量池中保存一个副本，不会通过连接而进行访问。对`final`变量访问在编译期间会被替代为真实值。`String c = b + 2`在编译期间会被优化成`String c = "hello" + 2`。

<br>


### Case VI

```java
 String a = "hello2"; 
 final String b = getHello(); 
 String c = b + 2;   
 System.out.println(a == c); // false
```

虽然`b`用`final`修饰，但由于其赋值是通过方法调用返回，那么它值只能在运行期间确定。因此`a`和`c`指向不是同一个对象。

<br>


### Case VII

```java
String s1 = new String("Hello"); 
String s2 = new String("Hello"); 
```

Answer is 3: 
1. First – line 1, “Hello” object in the string pool. 
2. Second – line 1, new String with value “Hello” in the heap memory. 
3. Third – line 2, new String with value “Hello” in the heap memory. Here “Hello” string from string pool is reused.

<br></br>
