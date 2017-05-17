# <center>String</center>

<br></br>



## String Pool
----
Here is a diagram which clearly explains how String Pool is maintained in Java heap space and what happens when we use different ways to create Strings.

![StringPool](../Images/string_pool.png)

String Pool is possible only because String is immutable in Java and it’s implementation of String interning concept. *String pool is also example of Flyweight design pattern.*

When we use double quotes to create a String(`String str = "test"`), it first looks for String with same value in the String pool, if found it just returns the reference else it creates a new String in the pool and then returns the reference.

However using `new` operator, we force String class to create a new String object and then we can use `intern()` method to put it into the pool or refer to other String object from pool having same value.

<br></br>



## String.intern()
----
When the `intern()` is invoked, if the pool already contains a string equal to this String object as determined by the `equals(Object)` method, then the string from the pool is returned. Otherwise, this String object is added to the pool and a reference to this String object is returned.

<br></br>



## Why String is immutable or final?
----
There are several benefits of String because it’s immutable and final:
* String Pool is possible because String is immutable in java. 
* It increases security because any hacker can’t change its value and it’s used for storing sensitive information such as database username, password etc. 
* Since String is immutable, it’s safe to use in multi-threading and we don’t need any synchronization. 
* Strings are used in java classloader and immutability provides security that correct class is getting loaded by Classloader. 

<br></br>



## How many String objects got created in below code snippet? 
----
```java
String s1 = new String("Hello"); 
String s2 = new String("Hello"); 
```

Answer is 3: 
1. First – line 1, “Hello” object in the string pool. 
2. Second – line 1, new String with value “Hello” in the heap memory. 
3. Third – line 2, new String with value “Hello” in the heap memory. Here “Hello” string from string pool is reused.

<br></br>



## What are different ways to create String Object?
----
```java
String str = new String("abc"); 
String str1 = "abc";
```

When we create a String using double quotes, JVM looks in the String pool to find if any other String is stored with same value. If found, it returns the reference to that String object else it creates a new String object and stores it in the String pool. 

When we use new operator, JVM creates the String object but don’t store it into the String Pool. We can use `intern()` method to store the String object into String pool or return the reference if there is already a String with equal value present in the pool.

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

Java语言使用内存时，栈内存主要保存基本数据类型和对象的引用，而堆内存存储对象，栈内存的速度要快于堆内存。String类的本质是字符数组`char[]`，其次String类是`final`。Java运行时维护一个String池，池中的String对象不可重复，没有创建，有则作罢。**String池不属于堆和栈，而是属于常量池。**

下面分析上方代码的真正含义： 
```java 
String str = "abc";  
String str1= "abc";  
```

第一句的含义是在String池中创建一个对象_abc_，然后引用时`str`指向池中的对象_abc_。第二句执行时，因为_abc_已经存在于String池了，所以不再创建，则`str==str1`返回`true`。 

```java
String str2 = new String("abc"); 
```

创建了2个String对象，而基于上面两句，只在栈内存创建`str2`引用，在堆内存上创建一个String对象，内容是_abc_，而`str2`指向堆内存对象的首地址。

`str2 == ”abc”`为false。因为，_abc_是位于String池中的对象，而`str2`指向的是堆内存的String对象，==判断的是地址。 但`str1.equals(str2)`是对的。

`intern()`方法先在String池中查找是否存在一个对象，存在了就返回String池中对象的引用。本例中String池存在_abc_，则调用`intern()`方法时返回的是池中_abc_对象引用，那么和`str`，`str1`都是等同的。和`str2`就不同了，因为`str2`指向的是堆内存。 

<br></br>



## StringBuilder和StringBuffer底层原理
----
都继承自AbstractStringBuilder这个类，底层都是char数组操作，StringBuffer之所以线程安全是所有方法都加了Synchronized。



## 求String输出结果
----
### Case I

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

`str1`和`str3`都在编译期间生成了字面常量和符号引用，运行期间字面常量`"hello world"`存储在运行时常量池。通过这种方式来将String对象跟引用绑定的话，JVM会先在运行时常量池查找是否存在相同的字面常量，如果存在，则直接将引用指向已经存在的字面常量；否则在运行时常量池开辟一个空间来存储该字面常量，并将引用指向该字面常量。

通过new关键字来生成对象是在堆进行的，而在堆对象生成的过程是不会去检测该对象是否已存在的。因此通过new来创建对象，创建出的一定是不同的对象，即使字符串的内容是相同的。

<br>


### Case II

```java
String a = "hello2"; 　　
String b = "hello" + 2; 　　
System.out.println(a == b); // true
```

`b`在编译期间就已经被优化成`"hello2"`，因此在运行期间，变量`a`和变量`b`指向的是同一个对象。

<br>


### Case III

```java
String a = "hello2"; 
String b = "hello";    
String c = b + 2;     
System.out.println(a == c); // false
```

由于有符号引用的存在，所以`c`不会在编译期间被优化，不会把`b + 2`当做字面常量来处理的，因此生成的对象是保存在堆上的。

<br>


### Case IV

```java
String a = "hello2"; 
final String b = "hello"; 
String c = b + 2; 
System.out.println(a == c); // true
```

被`final`修饰的变量，会在class文件常量池中保存一个副本，不会通过连接而进行访问。对`final`变量的访问在编译期间都会直接被替代为真实的值。那么`String c = b + 2`在编译期间就会被优化成`String c = "hello" + 2`。

<br>


### Case V

```java
 String a = "hello2"; 
 final String b = getHello(); 
 String c = b + 2;   
 System.out.println(a == c); // false
```

虽然`b`用`final`修饰，但由于其赋值是通过方法调用返回的，那么它的值只能在运行期间确定，因此`a`和`c`指向的不是同一个对象。

<br></br>



## Why Char array is preferred over String for storing password?
----
String is immutable in java and stored in String pool. Once it’s created it stays in the pool until unless garbage collected, so even though we are done with password it’s available in memory for longer duration and there is no way to avoid it. It’s a security risk because anyone having access to memory dump can find the password as clear text. If we use char array to store password, we can set it to blank once we are done with it. So we can control for how long it’s available in memory that avoids the security threat with String.