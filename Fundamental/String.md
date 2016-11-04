# String

## 1. What is java string pool?
We know that String is special class in java and we can create String object using new operator as well as providing values in double quotes.
Here is a diagram which clearly explains how String Pool is maintained in java heap space and what happens when we use different ways to create Strings.

![StringPool](../Images/string_pool.png)

String Pool is possible only because String is immutable in Java and it’s implementation of String interning concept. *String pool is also example of Flyweight design pattern.*
When we use double quotes to create a String, it first looks for String with same value in the String pool, if found it just returns the reference else it creates a new String in the pool and then returns the reference.
However using `new` operator, we force String class to create a new String object and then we can use `intern()` method to put it into the pool or refer to other String object from pool having same value.



## 2. What does String intern() method do?
When the `intern()` is invoked, if the pool already contains a string equal to this String object as determined by the `equals(Object)` method, then the string from the pool is returned. Otherwise, this String object is added to the pool and a reference to this String object is returned.
This method always returns a String that has the same contents as this string, but is guaranteed to be from a pool of unique strings.


## 3. Why String is immutable or final in Java 
There are several benefits of String because it’s immutable and final:
* String Pool is possible because String is immutable in java. 
* It increases security because any hacker can’t change its value and it’s used for storing sensitive information such as database username, password etc. 
* ince String is immutable, it’s safe to use in multi-threading and we don’t need any synchronization. 
* Strings are used in java classloader and immutability provides security that correct class is getting loaded by Classloader. 


## 4. Difference between String, StringBuffer and StringBuilder?
String is immutable and final in java, so whenever we do String manipulation, it creates a new String. String manipulations are resource consuming, so java provides two utility classes for String manipulations – StringBuffer and StringBuilder. 
StringBuffer and StringBuilder are mutable classes. StringBuffer operations are thread-safe and synchronized where StringBuilder operations are not thread-safe. 


## 5. How many String objects got created in below code snippet? 
```java
String s1 = new String("Hello"); 
String s2 = new String("Hello"); 
```

Answer is 3. First – line 1, “Hello” object in the string pool. Second – line 1, new String with value “Hello” in the heap memory. Third – line 2, new String with value “Hello” in the heap memory. Here “Hello” string from string pool is reused.


## 6. What are different ways to create String Object?
```java
String str = new String("abc"); 
String str1 = "abc";
```

When we create a String using double quotes, JVM looks in the String pool to find if any other String is stored with same value. If found, it returns the reference to that String object else it creates a new String object and stores it in the String pool. When we use new operator, JVM creates the String object but don’t store it into the String Pool. We can use `intern()` method to store the String object into String pool or return the reference if there is already a String with equal value present in the pool.

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

&#12288;&#12288;Java语言使用内存时，栈内存主要保存基本数据类型和对象的引用，而堆内存存储对象，栈内存的速度要快于堆内存。 String类的本质是字符数组`char[]`，其次String类是`final`的，再次String是特殊的封装类型，使用String时可以直接赋值，也可以用new来创建对象，但是这二者的实现机制是不同的。还有一个 String池的概念，Java运行时维护一个String池，池中的String对象不可重复，没有创建，有则作罢。String池不属于堆和栈，而是属于常量池。
&#12288;&#12288;下面分析上方代码的真正含义 
```java 
String str = "abc";  
String str1= "abc";  
```

&#12288;&#12288;第一句的含义是在String池中创建一个对象”abc”，然后引用时str指向池中的对象”abc”。第二句执行时，因为”abc”已经存在于 String池了，所以不再创建，则`str==str1`返回`true`就明白了。 

``String str2 = new String("abc"); ``

&#12288;&#12288;这句话创建了2个String对象，而基于上面两句，只在栈内存创建str2引用，在堆内存上创建一个String对象，内容是”abc”，而str2指向堆内存对象的首地址。

&#12288;&#12288;下面就是str2==”abc”的问题了，显然不对，”abc”是位于String池中的对象，而str2指向的是堆内存的String对象，==判断的是地址，肯定不等了。 

&#12288;&#12288;`str1.equals(str2)`是对的。String类的`equals()`重写了Object类的`equals()`方法，实际就是判断内容是否相同了。 

&#12288;&#12288;`intern()`实际上是这样的：该方法现在String池中查找是否存在一个对象，存在了就返回String池中对象的引用。 那么本例中String池存在”abc”，则调用`intern()`方法时返回的是池中”abc”对象引用，那么和str/str1都是等同的，和str2就不同了，因为str2指向的是堆内存。 


## 7. Can we use String in switch case?
Java 7 extended the capability of switch case to use Strings also, earlier java versions doesn’t support this.
