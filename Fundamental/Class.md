# <center>Class</center>

<br></br>



## FAQ
----
### Can we declare a class as static? 
We can’t declare a top-level class as static however an inner class can be declared as static. If inner class is declared as static, it’s called static nested class. Static nested class is same as any other top-level class and is nested for only packaging convenience.
The reason why the top-level class can’t be added keyword static is all top-level classes are, by definition, static. What the static boils down to is that an instance of the class can stand on its own. Or, the other way around: a non-static inner class (= instance inner class) can’t exist without an instance of the outer class. Since a top-level class does not have an outer class, it can’t be anything but static.

<br>


### Why Java main function and all members in main classes needs to be static?
Because, in Java a function or variable in class in inaccessible until an Object is created from the class, but the 'main' function is supposed to run at startup(without Object initiation) by JVM. So the 'main' is declared public as well as static so that it can be accessed outside class & without even an Object is created.

<br>


### 抽象类是否可继承实体类 (concrete class)
可以继承，但实体类须有明确的构造函数.其实Object就是个实体类，每个抽象类都直接或间接继承自Object，所以这点是没有疑问的。

<br></br>



## 抽象类与接口
---- 
* 接口中所有的方法隐含的都是抽象的。而抽象类则可以同时包含抽象和非抽象的方法。
* 类可以实现很多个接口，但只能继承一个抽象类。接口可以实现多重继承。
* 类如果要实现一个接口，必须实现接口声明的所有方法。但是，类可以不实现抽象类声明的所有方法。
* Java接口中声明的变量默认都是final的。抽象类可以包含非final的变量。
* Java接口中的成员函数默认是public的, 不能有私有的方法或变量，因为是用于让别人使用的。抽象类的成员函数可以是private，protected或public。

<br></br>



## Local Class
----
局部内部类是定义在一个方法或者一个作用域里面的类，访问仅限于方法内或者该作用域内。

Use it if you need to create more than one instance of a class, access its constructor, or introduce a new, named type (because, for example, you need to invoke additional methods later).

```java
class Outter {
    private int age = 12;
    // 局部变量x须为final！因为在方法中定义的局部变量相当于一个常量。
    // 它的生命周期超出方法运行的生命周期，所以不能再内部类中改变局部变量的值。
    public void Print(final int x) {    
        class Inner {
            public void inPrint() {
                System.out.println(x);
                System.out.println(age);
            }
        }
        new Inner().inPrint();
    }
}
```

<br></br>



## Anonymous Class
----
Use it if you need to declare fields or additional methods or need to use a local class only once. The following example uses anonymous classes in the initialization statements of the local variables `frenchGreeting`, but uses a local class for the initialization of the variable `englishGreeting`:

```java
public class HelloWorldAnonymousClasses {
    interface HelloWorld {
        public void greetSomeone(String someone);
    }
  
    public void sayHello() {
        class EnglishGreeting implements HelloWorld {
            String name = "world";
            public void greetSomeone(String someone) {
                name = someone;
                System.out.println("Hello " + name);
            }
        }
      
        HelloWorld englishGreeting = new EnglishGreeting();
        
        HelloWorld frenchGreeting = new HelloWorld() {
            String name = "tout le monde";
            public void greetSomeone(String someone) {
                name = someone;
                System.out.println("Salut " + name);
            }
        };
       
        englishGreeting.greet();
        frenchGreeting.greetSomeone("Fred");
    }   
}
```

<br></br>



## Nested Class
----
Nested classes enable you to logically group classes that are only used in one place, increase the use of encapsulation, and create more readable and maintainable code.

```java
class Outter {
    private int age = 12;
      
    class Inner {
        private int age = 13;
        public void print() {
            System.out.println("内部类变量：" + this.age);
            System.out.println("外部类变量：" + Outter.this.age);
        }
    }
}
```

<br></br>



## Static Nested Class
----
内部类就只能访问外部类的静态成员变量

```java
class Outter {
    private static int age = 12;
    static class Inner {
        public void print() {
            System.out.println(age);
        }
    }
}
```

<br></br>



## Using Reflection in Frameworks
----
Why use reflection in JUNIT, Spring, Tomcat, Struts and Hibernate? but not for normal programming?

Because frameworks have no knowledge and access of user defined classes, interfaces, their methods etc. Using reflection we can inspect a class, interface and enum get their structure, methods and fields information at runtime even though class is not accessible at compile time. We can also use reflection to instantiate an object, invoke it’s methods, change field values.

We should not use reflection in normal programming interfaces because:
* Poor Performance – Since reflection resolve the types dynamically, it involves processing like scanning the classpath to find the class to load, causing slow performance.反射包括了一些动态类型，所以JVM无法对这些代码进行优化。

* Security Restrictions – Reflection requires runtime permissions that might not be available for system running under security manager. This can cause you application to fail at runtime because of security manager.

* Security Issues – Using reflection we can access part of code that we are not supposed to access, for example we can access private fields of a class and change its value. This can be a serious security threat and cause your application to behave abnormally.

* High Maintenance – Reflection code is hard to understand and debug, also any issues with the code can’t be found at compile time because the classes might not be available, making it less flexible and hard to maintain.

<br></br>