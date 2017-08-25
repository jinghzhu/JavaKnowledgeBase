# <center>Class</center>

<br></br>



## Reflection in Frameworks
----
Because frameworks have no knowledge and access of user defined classes, interfaces, their methods etc. Using reflection we can inspect a class, interface and enum get their structure, methods and fields information at runtime. We can also use reflection to instantiate an object, invoke it’s methods, change field values.

We should not use reflection in normal programming:
* Poor Performance – Since reflection resolve the types dynamically, it involves processing like scanning the classpath to find the class to load, causing slow performance.反射包括了一些动态类型，所以JVM无法对这些代码进行优化。

* Security Restrictions – Reflection requires runtime permissions that might not be available for system running under security manager.

* Security Issues – Using reflection we can access part of code that we are not supposed to access, for example we can access private fields of a class and change its value. 

* High Maintenance – Reflection code is hard to understand and debug, also any issues with the code can’t be found at compile time because the classes might not be available.

<br></br>



## 抽象类与接口
---- 
* 接口中所有方法都是抽象。抽象类可同时包含抽象和非抽象方法。
* 类可以实现多个接口，但只能继承一个抽象类。接口可以实现多重继承。
* 类如果要实现一个接口，须实现接口声明的所有方法。但类可以不实现抽象类声明的所有方法。
* 接口中声明的变量是final。抽象类可以包含非final变量。
* 接口中的成员函数是public, 不能有私有的方法或变量，因为是用于让别人使用的。抽象类成员函数可以是private，protected或public。



## FAQ
----
**Q: Can we declare a class as static?**

**A:**We can’t declare a top-level class as static however an inner class can be declared as static. If inner class is declared as static, it’s called static nested class. 

The reason why the top-level class can’t be added keyword static is all top-level classes are, by definition, static. What the static boils down to is that an instance of the class can stand on its own. Or, the other way around: a non-static inner class (= instance inner class) can’t exist without an instance of the outer class. Since a top-level class does not have an outer class, it can’t be anything but static.

<br>


**Q: 抽象类是否可继承实体类 (concrete class)？**

**A:**可以继承，但实体类须有明确的构造函数。Object就是个实体类，每个抽象类都直接或间接继承自Object，所以这点是没有疑问的。

<br>


**Q: 局部内部类是否可以访问非final变量 ？**

**A:**不能访问局部的，可以访问成员变量（全局的）。 
```java
class Out { 
    private String name = "out.name"; 
    void print() { 
        // 若不是final的则不能被Animal使用。 
        final String work = "out.local.work";
        int age=10; 
        
        // 定义一个局部内部类，只能在print()方法中使用。
        // 局部类中不能使用外部的非final的局部变量，全局的可以。
        class Animal { 
            public void eat() { 
                System.out.println(work);//ok 
                //age=20;error not final 
                System.out.println(name);//ok. 
            } 
        } 
        Animal local = new Animal(); 
        local.eat(); 
    } 
} 
```

<br>


**Q: Anonymous Inner Class是否可继承其它类，实现接口？**

**A:**匿名的内部类是没有名字的内部类。不能extends(继承) 其它类，但一个内部类可以作为一个接口，由另一个内部类实现

<br>