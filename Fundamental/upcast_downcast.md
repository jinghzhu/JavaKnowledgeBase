# <center>多态，向上转型和向下转型</center>



<br></br>

Java多态调用的详细流程:

1. 编译器查看对象的声明类型和方法名。

    假设调用`obj.func(param)`，`obj`为`Cat`类对象。可能存在多个名为`func`但参数签名不一样的方法。例如，可能存在`func(int)`和`func(String)`。编译器会列举所有`Cat`类中名为`func`的方法和其父类`Animal`中访问属性为`public`且名为`func`的方法。
    
    这样，编译器就获得了所有可能被调用的候选方法列表。

2. 编泽器检查调用方法时提供的参数签名。

    如果在所有名为`func`方法中存在一个与提供的参数签名完全匹配的方法，那么就选择这个方法。这个过程被称为重载解析（overloading resolution）。
    
    例如，如果调用`func("hello")`，编译器选择`func(String)`，而不是`func(int)`。由于自动类型转换的存在，如果没找到与调用方法参数签名相同的，就进行类型转换后再查找。如果最终没有匹配的类型或有多个方法与之匹配，那么编译错误。

3. 静态绑定与动态绑定：

    如果方法修饰符是`privat`、`static`、`final`或是构造方法，那么编译器可知道调用哪个方法，称为静态绑定（static binding）。
    
    当程序运行且釆用动态绑定调用方法时，JVM调用与`obj`所引用对象实际类型最合适的那个类的方法。假设`obj`类型是`Cat`，是`Animal`子类。如果`Cat`中定义`func(String)`，就调用它，否则将在`Animal`类及其父类中寻找。

每次调用方法都要进行搜索，时间开销大。因此，JVM预先为每个类创建方法表（method lable），列出所有方法名称、参数签名和所属的类。真正调用方法时，JVM查找这个表就行了。
 
```java    
abstract class Animal {  
    abstract void eat();  
}  
  
class Cat extends Animal {  
    public void eat() {  
        System.out.println("吃鱼");  
    }  

    public void work() {  
        System.out.println("抓老鼠");  
    }  
}  
  
class Dog extends Animal {  
    public void eat() {  
        System.out.println("吃骨头");  
    }  
    
    public void work() {  
        System.out.println("看家");  
    }  
}

public class Test {
    public static void main(String[] args) {
      show(new Cat());  // 以Cat对象调用
      show(new Dog());  // 以Dog对象调用
                
      Animal a = new Cat();  // 向上转型  
      a.eat();               // 调用Cat的eat
      Cat c = (Cat)a;        // 向下转型  
      c.work();        // 调用Cat的work
    }  
            
    public static void show(Animal a)  {
        a.eat();  
        // 类型判断
        if (a instanceof Cat)  {  // 猫做的事情 
            Cat c = (Cat)a;  
            c.work();  
        } else if (a instanceof Dog) { // 狗做的事情 
            Dog c = (Dog)a;  
            c.work();  
        }  
    }  
}  
```

<br></br>



## 向下转型
---
把指向子类对象的父类引用，赋给子类引用叫向下转型，要强制转换。

```java
Father f2 = new Father();
Son s1 = (Son)f2; // 编译出错，子类引用不能指向父类对象
```

安全的向下转型是先向上，再向下：

```java
Base b = new Agg(); // 先向上
Agg a= (Agg)b;  // 再向下
```