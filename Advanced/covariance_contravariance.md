# <center>Convariance and Contravariance</center>

<br></br>



Example:
```java
// 定义三个类: Benz -> Car -> Vehicle，它们之间是顺次继承关系
class Vehicle {}
class Car extends Vehicle {}
class Benz extends Car {}

// 定义一个util类，其中用到泛型里的协变和逆变
class Utils<T> {
    T get(List<? extends T> list, int i) {
        return list.get(i);
    }
    
    void put(List<? super T> list, T item) {
        list.add(item);
    }
    
    void copy(List<? super T> to, List<? extends T> from) {
        for(T item : from) {
            to.add(item);
        }
    }
}

// 测试函数
void test() {
    List<Vehicle> vehicles = new ArrayList<>();
    List<Benz> benzs = new ArrayList<>();
    Utils<Car> carUtils = new Utils<>();

    carUtils.put(vehicles, new Car());
    Car car = carUtils.get(benzs, 0);
    carUtils.copy(vehicles, benzs);
}
```

<br></br>



## 定义
----
> Liskov替换原则：所有引用基类（父类）的地方必须能透明地使用其子类的对象。
> LSP包含以下四层含义：
> 1. 子类完全拥有父类的方法，且具体子类必须实现父类的抽象方法。
> 2. 子类中可以增加自己的方法。
> 3. 当子类覆盖或实现父类的方法时，方法的形参要比父类方法的更为宽松。
> 4. 当子类覆盖或实现父类的方法时，方法的返回值要比父类更严格。

如果$$A$$和$$B$$表示类型，$$f(⋅)$$表示类型转换，$$\geqq$$≦表示继承关系（比如，A≦BA表示A由B派生出来的子类），那么：
1. $$f(⋅)$$是逆变（contravariant）的，当A≦B时有$$f(B)$$≦$$f(A)$$成立；
2. $$f(⋅)$$是协变（covariant）的，当A≦B时有$$f(A)$$≦$$f(B)$$成立；
3. $$f(⋅)$$是不变（invariant）的，当A≦B时上述两个式子均不成立，即$$f(A)$$与$$f(B)$$之间没有继承关系。

数组是协变：
```java
Number[] numbers = new Integer[3]; 
```

泛型是不可变的，比如：
```java
List<? extends Number> list = new ArrayList<>(); 
```

 `<? extends Number>`则表示通配符`?`的上界为Number，但代表不了Number的父类（如Object）。于是有`<? extends Number>` ≦ Number，则`List<? extends Number>` ≦ `List< Number>`。那么就有：
 ```java
List<? extends Number> list001 = new ArrayList<Integer>();  
List<? extends Number> list002 = new ArrayList<Float>();  
```

但不能向_list001_和_list002_添加除null以外的任意对象。

`<? super>`实现了泛型的逆变，比如：
```java
List<? super Number> list = new ArrayList<>();  
```

`<? super Number>`表示通配符`?`下界为Number。为了保护类型的一致性，因为`<？ super Number>`可以是Object或其他Number父类，因无法确定其类型，也就不能往`List<? super Number>`添加Number的任意父类对象。但可以向`List<? super Number>`添加Number及其子类。
```java
List<? super Number> list001 = new ArrayList<Number>();  
List<? super Number> list002 = new ArrayList<Object>();  
list001.add(new Integer(3));  
list002.add(new Integer(3));  
```

<br></br>



## PECS(producer-extends, consumer-super)
----
* 要从泛型类取数据时，用extends；
* 要往泛型类写数据时，用super；
* 既要取又要写，就不用通配符（即extends与super都不用）。

比如，一个简单的Stack API：
```java
public class Stack<E>{  
    public Stack();  
    public void push(E e):  
    public E pop();  
    public boolean isEmpty();  
}  
```

要实现`pushAll(Iterable<E> src)`方法，将_src_的元素逐一入栈：
```
public void pushAll(Iterable<E> src){  
    for(E e : src)  
        push(e)  
}  
```
       
假设有一个实例化`Stack<Number>`的对象_stack_，_src_有`Iterable<Integer>`与`Iterable<Float`。在调用`pushAll()`方法时会发生Type Mismatch错误，因为Java中泛型是不可变的，`Iterable<Integer>`与`Iterable<Float>`都不是`Iterable<Number>`的子类型。因此，应改为
```java
// Wildcard type for parameter that serves as an E producer  
public void pushAll(Iterable<? extends E> src) {  
    for (E e : src)  
        push(e);  
}
```  
       
要实现`popAll(Collection<E> dst)`方法，如果不用通配符实现：
```java
// popAll method without wildcard type - deficient!  
public void popAll(Collection<E> dst) {  
    while (!isEmpty())  
        dst.add(pop());     
}
```  
       
同样地，假设有一个实例化`Stack<Number>`的对象_stack_，_dst_为`Collection<Object>`。调用`popAll()`方法会发生Type Mismatch错误，因为`Collection<Object>`不是`Collection<Number>`的子类型。因而，应改为：
```java
// Wildcard type for parameter that serves as an E consumer  
public void popAll(Collection<? super E> dst) {  
    while (!isEmpty())  
        dst.add(pop());  
}
```  
       
在上述例子中，调用`pushAll()`方法时生产了E实例（produces E instances），调用`popAll()`方法时_dst_消费了E实例（consumes E instances）。

java.util.Collections的copy方法(JDK1.7)完美地诠释了PECS：
```java
public static <T> void copy(List<? super T> dest, List<? extends T> src) {  
    int srcSize = src.size();  
    if (srcSize > dest.size())  
        throw new IndexOutOfBoundsException("Source does not fit in dest");  
  
    if (srcSize < COPY_THRESHOLD ||  
        (src instanceof RandomAccess && dest instanceof RandomAccess)) {  
        for (int i=0; i<srcSize; i++)  
            dest.set(i, src.get(i));  
    } else {  
        ListIterator<? super T> di=dest.listIterator();  
        ListIterator<? extends T> si=src.listIterator();  
        for (int i=0; i<srcSize; i++) {  
            di.next();  
            di.set(si.next());  
        }  
    }  
}  
```