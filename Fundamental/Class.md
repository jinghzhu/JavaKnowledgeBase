# <center>Class</center>

<br></br>



## 抽象类与接口
---- 
* 接口中所有方法都是抽象。抽象类可同时包含抽象和非抽象方法。
* 类可实现多个接口，但只能继承一个抽象类。接口可实现多重继承。
* 类如果要实现一个接口，须实现接口声明所有方法，但可不实现抽象类声明所有方法。
* 接口声明变量是`final`。抽象类包含非final变量。
* 接口成员函数是`public`，不能有私有的方法或变量，因为是用于让别人使用的。抽象类成员函数可以是`private`，`protected`或`public`。

<br></br>



## Composition 组合
----
Composition用在新类中使用已有类的功能，而不是使用已有类接口。这时，在新类中嵌入已有类对象，完成想要的功能。Composition常表述为*HAS-A*关系。

Why favor composition over inheritance:
1. Any change in the superclass might affect subclass. Composition will never face this issue because we are using only what methods we need.

2. Inheritance exposes all super class methods and variables to client, it can lead to security holes. Composition allows us to provide restricted access to the methods.

什么时候用继承？什么时候用组合？
1. 如果存在*IS-A*关系（比如Bee是一个Insect），且类需向另一个类暴露所有方法接口，用继承。
2. 如果存在*HAS-A*关系（比如Bee有一个attack功能），用组合。

定义一个`Battery`类，并用`power`表示电量。`Battery`可以充电`chargeBattery()`和使用`useBattery()`。`Torch`类定义中使用`Battery`类型对象作为数据成员：
```java
class Battery {
    public void chargeBattery(double p) {
        if (this.power < 1.)
            this.power = this.power + p;
    }
    public boolean useBattery(double p) {
        if (this.power >= p) {
            this.power = this.power - p;
            return true;
        } else {
            this.power = 0.0;
            return false;
        }
    }

    private double power = 0.0;
}

class Torch {
    /** 
     * 10% power per hour use
     * warning when out of power
     */
    public void turnOn(int hours) {
        boolean usable;
        usable = this.theBattery.useBattery( hours*0.1 );
        if (usable != true)
            System.out.println("No more usable, must charge!");
    }

    /**
     * 20% power per hour charge
     */
    public void charge(int hours) {
        this.theBattery.chargeBattery( hours*0.2 );
    }

    /**
     * composition
     */
    private Battery theBattery = new Battery();
}
```

`Torch`类使用`Battery`类型对象`theBattery`作为数据成员。在`Torch`方法中，通过操纵`theBattery`对象接口，实现`Battery`类提供的功能。

<p align="center">
  <img src="./Images/composition1.jpg" />
</p>

通过组合，可复用Battery相关代码。假如还有其他使用Battery类，比如手机，都可将Battery对象组合进去，不用为每个类单独编写相关功能。

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

4. Why can’t we create generic array? 

    We are not allowed to create generic arrays because array carry type information of it’s elements at runtime. This information is used at runtime to throw `ArrayStoreException` if elements type doesn’t match to the defined type. Since generics type information gets erased at runtime by Type Erasure, the array store check would have been passed where it should have failed. 

<br></br>