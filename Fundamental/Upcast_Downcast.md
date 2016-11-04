# 多态，向上转型和向下转型

## 1. 多态的三个前提条件：
1. 要有继承； 
2. 要有重写； 
3. 父类变量引用子类对象。

## 2. 例子
当使用多态方式调用方法时，首先检查父类中是否有该方法，如果没有，则编译错误；如果有，再去调用子类的同名方法。
 
```java    
public class Test {
    public static void main(String[] args) {
      show(new Cat());  // 以 Cat 对象调用 show 方法
      show(new Dog());  // 以 Dog 对象调用 show 方法
                
      Animal a = new Cat();  // 向上转型  
      a.eat();               // 调用的是 Cat 的 eat
      Cat c = (Cat)a;        // 向下转型  
      c.work();        // 调用的是 Cat 的 catchMouse
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
```

&#12288;&#12288;输出结果为：

> 吃鱼

> 抓老鼠

> 吃骨头

> 看家

> 吃鱼

> 抓老鼠


## 3. upcast
&#12288;&#12288;向上转型(upcast)存在一些缺憾，那就是它必定会导致一些方法和属性的丢失，而导致我们不能够获取它们。所以父类类型的引用可以调用父类中定义的所有属性和方法，对于只存在与子类中的方法和属性它就望尘莫及了.
&#12288;&#12288;从上面的例子可以看出，多态的一个好处是：当子类比较多时，也不需要定义多个变量，可以只定义一个父类类型的变量来引用不同子类的实例。
&#12288;&#12288;请再看下面的一个例子：

```java
public class Demo {
  public static void main(String[] args){
    // 借助多态，主人可以给很多动物喂食
    Master ma = new Master();
    ma.feed(new Animal(), new Food());
    ma.feed(new Cat(), new Fish());
    ma.feed(new Dog(), new Bone());
    }
}
 
// Animal类及其子类
class Animal{
  public void eat(Food f){
    System.out.println("我是一个小动物，正在吃" + f.getFood());
  }
}
 
class Cat extends Animal{
  public void eat(Food f){
    System.out.println("我是一只小猫咪，正在吃" + f.getFood());
  }
}
 
class Dog extends Animal{
  public void eat(Food f){
    System.out.println("我是一只狗狗，正在吃" + f.getFood());
  }
}
 
// Food及其子类
class Food{
  public String getFood(){
    return "事物";
  }
}
 
class Fish extends Food{
  public String getFood(){
    return "鱼";
  }
}
 
class Bone extends Food{
  public String getFood(){
    return "骨头";
  }
}
 
// Master类
class Master{
  public void feed(Animal an, Food f){
    an.eat(f);
  }
}
```

&#12288;&#12288;运行结果：
```
我是一个小动物，正在吃事物
我是一只小猫咪，正在吃鱼
我是一只狗狗，正在吃骨头
```

## 4. 多态的本质

&#12288;&#12288;Java调用的详细流程:

### 4.1 编译器查看对象的声明类型和方法名。
&#12288;&#12288;假设调用`obj.func(param)`，`obj`为`Cat`类的对象。有可能存在多个名字为`func`但参数签名不一样的方法。例如，可能存在`func(int)`和`func(String)`。编译器会列举所有`Cat`类中名为`func`的方法和其父类`Animal`中访问属性为`public`且名为`func`的方法。

&#12288;&#12288;这样，编译器就获得了所有可能被调用的候选方法列表。

### 4.2 接下来，编泽器将检查调用方法时提供的参数签名。
&#12288;&#12288;如果在所有名为`func`的方法中存在一个与提供的参数签名完全匹配的方法，那么就选择这个方法。这个过程被称为重载解析(overloading resolution)。

&#12288;&#12288;例如，如果调用`func("hello")`，编译器会选择`func(String)`，而不是`func(int)`。由于自动类型转换的存在，如果没有找到与调用方法参数签名相同的方法，就进行类型转换后再继续查找，如果最终没有匹配的类型或者有多个方法与之匹配，那么编译错误。

### 4.3 静态绑定与动态绑定
&#12288;&#12288;如果方法的修饰符是`private`、`static`、`final`，或是构造方法，那么编译器可准确知道该调用哪个方法，这种调用方式称为静态绑定(static binding)。

&#12288;&#12288;当程序运行，且釆用动态绑定调用方法时，JVM会调用与`obj`所引用对象的实际类型最合适的那个类的方法。我们假设`obj`实际类型是`Cat`，是`Animal`子类，如果 `Cat`中定义`func(String)`，就调用它，否则将在`Animal`类及其父类中寻找。
<br></br>

&#12288;&#12288;每次调用方法都要进行搜索，时间开销相当大。因此，JVM预先为每个类创建了一个方法表(method lable)，其中列出了所有方法的名称、参数签名和所属的类。在真正调用方法的时候，虚拟机仅查找这个表就行了。

&#12288;&#12288;在上面的例子中，JVM搜索`Cat`类的方法表，以便寻找与调用`func("hello")`相匹配的方法。注意，如果调用`super.func("hello")`，编译器将对父类的方法表迸行搜索。

&#12288;&#12288;假设`Animal`类包含`cry()`、`getName()`、`getAge()`三个方法，那么它的方法表如下：
> cry() -> Animal.cry()

> getName() -> Animal.getName()

> getAge() -> Animal.getAge()

&#12288;&#12288;实际上，`Animal`也有默认的父类`Object`，会继承`Object`方法，所以上面列举的方法并不完整。

&#12288;&#12288;假设`Cat`类覆盖了`Animal`类中的`cry()`方法，且新增了一个方法`climbTree()`，那么它的参数列表为：
> cry() -> Cat.cry()

> getName() -> Animal.getName()

> getAge() -> Animal.getAge()

> climbTree() -> Cat.climbTree()

&#12288;&#12288;在运行的时候，调用`obj.cry()`过程如下：
* JVM 首先访问`obj`实际类型的方法表，可能是`Animal`类的方法表，也可能是`Cat`类及其子类的方法表。
* JVM 在方法表中搜索与`cry()`匹配的方法 。
* JVM 调用该方法。

```java
class Base{
    public static String getField(){
        String name = "Base";
            return name;
    }
}
class Agg extends Base{
    public static String getField(){
        String name = "Agg";
            return name;
    }
}
public class test {
    public static void main(String[] args) {
        Agg a = new Agg();
        Base b =  new Agg();
        System.out.println(""+a.getField());
        System.out.println(""+b.getField());
    }
}
```
output : Agg Base


## 5. 向下转型

### 5.1 介绍
`Son s1 = (Son)f1;`
&#12288;&#12288;把指向子类对象的父类引用，赋给子类引用叫向下转型，要强制转换。

```java
Father f2 = new Father();
Son s1 = (Son)f2; //编译出错，子类引用不能指向父类对象
```

&#12288;&#12288;如:

```java
class Base{
    public String getField(){
        String name = "Base";
            return name;
    }
}

class Agg extends Base{
    public String getField(){
        String name = "Agg";
            return name;
    }
}

public class test {
    public static void main(String[] args) {
        Base a = new Base();
        Agg b = (Agg)a; \\ 不会编译通过，这样向下转型不可以，不安全。
        System.out.println(""+b.getField());
    }
}
```

### 5.2 如何安全的向下转型？

&#12288;&#12288;先向上 - 再向下

```java
Base b = new Agg(); \\先向上
Agg a= (Agg)b; \\再向下
```

&#12288;&#12288;再贴一个向下转型实际应用场景的代码，比较对象：
```java
public class Person {
  private String name;
  private int age;

  public boolean equals(Object anObject) {
     if (anObject == null)
         return false;

     /* The object being passed in is checked
         to see it's class type which is then compared
         to the class type of the current class.  If they
         are not equal then it returns false
     */

     else if (getClass( ) != anObject.getClass()) {
         return false;
     }
         

     else {
        /* 
         this is a downcast since the Object class
         is always at the very top of the inheritance tree
         and all classes derive either directly or indirectly 
         from Object:
        */
        Person aPerson = (Person) anObject;
        return ( name.equals(aPerson.name) 
                && (age == aPerson.age));
     }
  }
}
```