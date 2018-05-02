# <center>Class</center>

<br></br>



## Reflection in Frameworks
----
Because frameworks have no knowledge and access of user defined classes, interfaces, their methods etc. Using reflection we can inspect a class, interface and enum get their structure, methods and fields information at runtime. We can also use reflection to instantiate an object, invoke it’s methods, change field values.

We should not use reflection in normal programming:
* Poor Performance – Since reflection resolve the types dynamically, it involves processing like scanning the classpath to find the class to load, causing slow performance.反射包括了一些动态类型，所以JVM无法对这些代码进行优化。

* Security Issues – Using reflection we can access part of code that we are not supposed to access, for example we can access private fields of a class and change its value. 

* High Maintenance – Reflection code is hard to understand and debug, also any issues with the code can’t be found at compile time because the classes might not be available.

<br></br>



## 抽象类与接口
---- 
* 接口中所有方法都是抽象。抽象类可同时包含抽象和非抽象方法。
* 类可实现多个接口，但只能继承一个抽象类。接口可实现多重继承。
* 类如果要实现一个接口，须实现接口声明所有方法，但可不实现抽象类声明所有方法。
* 接口声明变量是`final`。抽象类包含非final变量。
* 接口成员函数是`public`，不能有私有的方法或变量，因为是用于让别人使用的。抽象类成员函数可以是`private`，`protected`或`public`。

<br></br>



## FAQ
----
1. Can we declare a class as `static`?
    
    We can’t declare a top-level class as `static` however an inner class can be declared as `static`. If inner class is declared as `static`, it’s called `static` nested class. 

    All top-level classes are, by definition, `static`. What the `static` boils down to is that an instance of the class can stand on its own. Or, the other way around: a non-static inner class can’t exist without an instance of the outer class. Since a top-level class does not have an outer class, it can’t be anything but `static`.

2. 抽象类是否可继承实体类 (concrete class)？
    
    可以继承，但实体类须有明确的构造函数。`Object`就是实体类，每个抽象类都直接或间接继承`Object`。

3. Anonymous Inner Class是否可继承其它类，实现接口？

    匿名的内部类是没有名字的内部类。不能继承其它类，但内部类可作为一个接口，由另一个内部类实现。

<br></br>