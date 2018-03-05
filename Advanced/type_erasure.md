# <center>Type Erasure</center>

<br></br>



## Introduction
----
Java的泛型是伪泛型。因为，在编译期间，所有的泛型信息都会被擦除掉。Java泛型都是在编译器这个层次实现的。生成的Java字节码不包含泛型中的类型信息。使用泛型时加上的类型参数，会在编译器在编译的时候去掉。这个过程就称为类型擦除。

如在代码中定义的`List<Object>`和`List<String>`等类型，在编译后都会变成List。JVM看到的只是List，而由泛型附加的类型信息对JVM来说是不可见的。Java编译器会在编译时尽可能的发现可能出错的地方，但是仍然无法避免在运行时刻出现类型转换异常的情况。类型擦除也是Java的泛型实现方法与C++模版机制实现方式之间的重要区别。

```java
public class Test4 {  
    public static void main(String[] args) {  
        ArrayList<String> arrayList1 = new ArrayList<String>();  
        arrayList1.add("abc");  
        ArrayList<Integer> arrayList2 = new ArrayList<Integer>();  
        arrayList2.add(123);  

        // 说明泛型类型String和Integer都被擦除掉了，只剩下了原始类型。
        System.out.println(arrayList1.getClass() == arrayList2.getClass()); // true  
    }  
} 
``` 

```java
public class Test4 {  
    public static void main(String[] args) throws 
        IllegalArgumentException, SecurityException, IllegalAccessException, InvocationTargetException, NoSuchMethodException {  
        ArrayList<Integer> arrayList3 = new ArrayList<Integer>();  
        arrayList3.add(1); // 这样调用add方法只能存储整形，因为泛型类型的实例为Integer  
        arrayList3.getClass().getMethod("add", Object.class).invoke(arrayList3, "asd");  
        for (int i=0;i<arrayList3.size();i++) {  
            System.out.println(arrayList3.get(i));  
        }  
    }  
```
如果直接调用`add()`方法，那么只能存储整形的数据。利用反射调用`add()`方法时，可存储字符串。说明Integer泛型实例在编译之后被擦除了，只保留了原始类型。

<br></br>



## 原始类型
----
原始类型（raw type）是擦除去了泛型信息，最后在字节码中的类型变量的真正类型。无论何时定义一个泛型类型，相应的原始类型都会被自动地提供。类型变量被擦除，并使用其限定类型（无限定的变量用Object）替换。

```java
class Pair<T> {  
    private T value;  
    public T getValue() {  
        return value;  
    }  
    public void setValue(T  value) {  
        this.value = value;  
    }  
} 
```

`Pair<T>`的原始类型为：
```java
class Pair {  
    private Object value;  
    public Object getValue() {  
        return value;  
    }  
    public void setValue(Object  value) {  
        this.value = value;  
    }  
}  
```

因为在`Pair<T>`中，`T`是一个无限定的类型变量，所以用Object替换。在程序中可以包含不同类型的Pair，如`Pair<String>`或`Pair<Integer>`，但擦除类型后它们就成为原始的Pair类型了，原始类型都是Object。

如果类型变量有限定，那么原始类型就用第一个边界的类型变量来替换。比如Pair这样声明：
```java
public class Pair<T extends Comparable & Serializable> {}
```  
那么原始类型就是Comparable。注意如果Pair这样声明`public class Pair<T extends Serializable & Comparable> `，那么原始类型就用Serializable替换。为提高效率，应将标签（tagging）接口（即没有方法的接口）放在边界限定列表的末尾。

<br>


### 区分原始类型和泛型变量的类型
在调用泛型方法的时候，可以指定泛型，也可以不指定泛型：
* 不指定泛型时，泛型变量的类型为该方法中的几种类型的同一个父类的最小级，直到Object。
* 指定泛型时，该方法中的几种类型必须是该泛型实例类型或者其子类。

```java
public class Test2{  
    public static void main(String[] args) {  
        /**不指定泛型的时候*/  
        int i = Test2.add(1, 2); // 两个参数都是Integer，所以T为Integer类型  
        Number f = Test2.add(1, 1.2); // 一个是Integer，一个是Float，所以取同一父类的最小级，为Number  
        Object o = Test2.add(1, "asd"); // 一个是Integer，一个是Float，所以取同一父类的最小级，为Object  
  
        /**指定泛型的时候*/  
        int a = Test2.<Integer>add(1, 2); // 指定Integer，所以只能为Integer类型或者其子类  
        int b = Test2.<Integer>add(1, 2.2); // 编译错误，指定了Integer，不能为Float  
        Number c = Test2.<Number>add(1, 2.2); // 指定为Number，所以可以为Integer和Float  
    }  
      
    // 泛型方法  
    public static <T> T add(T x,T y){  
        return y;  
    }  
}  
```

<br></br>



## 类型擦除引起的问题及解决方法
----
### 先检查再编译，以及检查编译的对象和引用传递的问题
既然类型变量会在编译时擦除，为什么往`ArrayList<String> arrayList=new ArrayList<String>();`所创建的数组列表arrayList中，不能使用`add()`方法添加整形呢？不是说泛型变量Integer会在编译时候擦除变为原始类型Object吗？为什么不能存别的类型呢？既然类型擦除了，如何保证只能使用泛型变量限定的类型呢？

Java编译器是通过先检查代码中泛型的类型，然后再进行类型擦除，在进行编译的。

```java
public static  void main(String[] args) {  
        ArrayList<String> arrayList=new ArrayList<String>();  
        arrayList.add("123");  
        arrayList.add(123); // 编译错误  
}  
```
报错说明这是在编译之前的检查。因为如果在编译之后检查，类型擦除后，原始类型为Object，是应该运行任意引用类型的添加的。

那么，类型检查是针对谁呢？先看看参数化类型与原始类型的兼容。

以ArrayList举例子，以前的写法：
```java
ArrayList arrayList = new ArrayList();  
```
现在的写法：
```java
ArrayList<String> arrayList = new ArrayList<String>();  
```

如果与以前的代码兼容，各种引用传值之间，必然会出现如下的情况：
```java
ArrayList<String> arrayList1 = new ArrayList(); // 第一种情况 
ArrayList arrayList2 = new ArrayList<String>(); // 第二种情况  
```

这样是没有错误的，不过会有个编译时警告。不过在第一种情况，可以实现与完全使用泛型参数一样的效果，第二种则完全没效果。

因为，类型检查就是编译时完成的。`new ArrayList()`只是在内存中开辟一个存储空间，可以存储任何的类型对象。而真正涉及类型检查的是它的引用，所以_arrayList1_引用能完成泛型类型的检查。而引用_arrayList2_没有使用泛型，所以不行。

举例：
```java
public class Test10 {  
    public static void main(String[] args) {   
        ArrayList<String> arrayList1 = new ArrayList();  
        arrayList1.add("1"); // 编译通过  
        arrayList1.add(1); // 编译错误  
        String str1 = arrayList1.get(0); // 返回类型就是String  
          
        ArrayList arrayList2=new ArrayList<String>();  
        arrayList2.add("1"); // 编译通过  
        arrayList2.add(1); // 编译通过  
        Object object=arrayList2.get(0); // 返回类型就是Object  
          
        new ArrayList<String>().add("11"); // 编译通过  
        new ArrayList<String>().add(22); // 编译错误  
        String strin g= new ArrayList<String>().get(0); // 返回类型就是String  
    }  
}  
```
通过例子，可以明白类型检查是针对引用的，谁是一个引用，用这个引用调用泛型方法，就会对这个引用调用的方法进行类型检测，而无关它真正引用的对象。

<br>


### 自动类型转换
因为类型擦除的问题，所以所有的泛型类型变量最后都会被替换为原始类型。这样就引起了一个问题，既然都被替换为原始类型，为什么在获取时不需强制类型转换呢？

看下ArrayList和`get()`方法：
```java
public E get(int index) {  
    RangeCheck(index);  
    return (E) elementData[index];  
}  
```
看以看到，在return之前，会根据泛型变量进行强转。

写了个简单的测试代码：
```java
public class Test {  
    public static void main(String[] args) {  
        ArrayList<Date> list = new ArrayList<Date>();  
        list.add(new Date());  
        Date myDate=list.get(0);  
}  
```

<br>


### 类型擦除与多态的冲突和解决方法
现在有这样一个泛型类：
```java
class Pair<T> {  
    private T value;  
    public T getValue() {  
        return value;  
    }  
    public void setValue(T value) {  
        this.value = value;  
    }  
}  
```

然后想要一个子类继承它：
```java
class DateInter extends Pair<Date> {  
    @Override  
    public void setValue(Date value) {  
        super.setValue(value);  
    }  
    @Override  
    public Date getValue() {  
        return super.getValue();  
    }  
} 
```

在这个子类中，设定父类的泛型类型为`Pair<Date>`，在子类中，覆盖了父类的两个方法，原意是将父类的泛型类型限定为Date，那么父类里面的两个方法的参数都为Date类型。
```java
public Date getValue() {  
    return value;  
}  
public void setValue(Date value) {  
    this.value = value;  
}  
```
所以，在子类中重写这两个方法一点问题也没有，实际上，从@Override标签中也可以看到，一点问题也没有，实际上是这样的吗？

实际上，类型擦除后父类的的泛型类型全部变为了原始类型Object，所以父类编译之后会变成下面的样子：
```java
class Pair {  
    private Object value;  
    public Object getValue() {  
        return value;  
    }  
    public void setValue(Object  value) {  
        this.value = value;  
    }  
}  
```

再看子类的两个重写的方法的类型：
```java
@Override  
public void setValue(Date value) {  
    super.setValue(value);  
}  
@Override  
public Date getValue() {  
    return super.getValue();  
}  
```

先分析`setValue()`方法，父类的类型是Object，而子类的类型是Date，参数类型不一样，似乎这不是重写，而是重载。

在一个main方法测试一下：
```java
public static void main(String[] args) throws ClassNotFoundException {  
        DateInter dateInter = new DateInter();  
        dateInter.setValue(new Date());                  
        dateInter.setValue(new Object()); // 编译错误  
}  
```

如果是重载，那么子类中两个`setValue()`方法，一个是参数Object类型，一个是Date类型。可没有这样的一个子类继承自父类的Object类型参数的方法。所以说，却是是重写了，而不是重载。

由于种种原因，虚拟机并不能将泛型类型变为Date，只能将类型擦除掉，变为原始类型Object。这样，本意是进行重写，实现多态。可类型擦除后变为了重载。这样，类型擦除就和多态有了冲突。JVM知道你的本意吗？知道！它能直接实现吗？不能！

于是JVM采用了一个特殊的方法，来完成这项功能，那就是桥方法。
首先，用`javap -c className`方式反编译下DateInter子类的字节码：
```
class com.tao.test.DateInter extends com.tao.test.Pair<java.util.Date> {  
  com.tao.test.DateInter();  
    Code:  
       0: aload_0  
       1: invokespecial #8                  // Method com/tao/test/Pair."<init>"  
:()V  
       4: return  
  
  public void setValue(java.util.Date);  // 重写的setValue方法  
    Code:  
       0: aload_0  
       1: aload_1  
       2: invokespecial #16                 // Method com/tao/test/Pair.setValue  
:(Ljava/lang/Object;)V  
       5: return  
  
  public java.util.Date getValue();    // 重写的getValue方法  
    Code:  
       0: aload_0  
       1: invokespecial #23                 // Method com/tao/test/Pair.getValue  
:()Ljava/lang/Object;  
       4: checkcast     #26                 // class java/util/Date  
       7: areturn  
  
  public java.lang.Object getValue();     // 编译时由编译器生成的巧方法  
    Code:  
       0: aload_0  
       1: invokevirtual #28                 // Method getValue:()Ljava/util/Date 去调用我们重写的getValue方法  
;  
       4: areturn  
  
  public void setValue(java.lang.Object);   // 编译时由编译器生成的巧方法  
    Code:  
       0: aload_0  
       1: aload_1  
       2: checkcast     #26                 // class java/util/Date  
       5: invokevirtual #30                 // Method setValue:(Ljava/util/Date;   去调用我们重写的setValue方法  
)V  
       8: return  
}  
```

从编译结果看，本意重写`setValue()`和`getValue()`方法的子类，有4个方法。最后的两个方法，是编译器生成的桥方法。桥方法的参数类型都是Object，子类中真正覆盖父类两个方法的就是这两个看不到的桥方法。桥方法的内部实现，是调用重写的那两个方法。

所以，虚拟机巧妙的使用了巧方法，来解决了类型擦除和多态的冲突。

<br>


### 泛型类型变量不能是基本数据类型
比如，没有`ArrayList<double>`，只有`ArrayList<Double>`。因为当类型擦除后，ArrayList的原始类型变为Object，但是Object类型不能存储double值，只能引用Double的值。

<br>


### 运行时类型查询
举个例子:
```java
ArrayList<String> arrayList=new ArrayList<String>();    
```

类型擦除之后，`ArrayList<String>`只剩下原始类型，泛型信息String不存在了。那么，运行时进行类型查询的时候使用下面的方法是错误的：
```java
if(arrayList instanceof ArrayList<String>)    
```

Java限定了这种类型查询的方式：
```java
if(arrayList instanceof ArrayList<?>) // ？是通配符的形式
``` 

<br>


### 异常中使用泛型的问题
* 不能抛出也不能捕获泛型类的对象。
    事实上，泛型类扩展Throwable都不合法。例如下面的定义将不会通过编译：
    ```java
    public class Problem<T> extends Exception{......}  
    ```

    因为异常都是在运行时捕获和抛出的，而在编译候，泛型信息都会被擦除。假设上面的编译可行，再看下面的定义：
    ```java
    try{  
    }catch(Problem<Integer> e1){  
        ...
    }catch(Problem<Number> e2){  
        ...  
    }   
    ```

    类型信息被擦除后，两个地方的catch都变为原始类型Object，相当于：
    ```java
    try{  
    }catch(Problem<Object> e1){  
        ...  
    }catch(Problem<Object> e2){  
        ...  
    }
    ```

    catch两个一模一样的普通异常，不能通过编译。

* 不能在catch子句中使用泛型变量
    ```java
    public static <T extends Throwable> void doWork(Class<T> t){  
        try{  
            ...  
        }catch(T e){ //编译错误  
            ...  
        }  
    }
    ```

    因为泛型信息在编译的时变为原始类型，也就是说T变为Throwable。假如可在catch子句中使用泛型变量：
    ```java
    public static <T extends Throwable> void doWork(Class<T> t){  
        try{  
            ...  
        }catch(T e){ //编译错误  
            ...  
        }catch(IndexOutOfBounds e){  
            ...
        }                           
    }  
    ```

    根据异常捕获原则，一定是子类在前面，父类在后面，上面就违背了这个原则。所以Java为了避免这样的情况，禁止在catch子句中使用泛型变量。但在异常声明中可使用类型变量：
    ```java
    public static<T extends Throwable> void doWork(T t) throws T{  
        try{  
            ...  
        }catch(Throwable realCause){  
            t.initCause(realCause);  
            throw t;   
        } 
    }
    ``` 

<br>


### 数组
不能声明参数化类型的数组。如：
```java
Pair<String>[] table = newPair<String>(10); 
```
因为擦除后，`table`的类型变为`Pair[]`，可以转化成`Object[]`。
  
数组可以记住自己的元素类型，下面的赋值会抛出一个ArrayStoreException异常：
```java
objarray = "Hello"; 
```

对于泛型而言，擦除降低了这个机制的效率。下面的赋值可以通过数组存储的检测，但仍然会导致类型错误。  
```java
objarray = new Pair<Employee>();  
```

<br>


### 泛型类型实例化 
* 不能实例化泛型类型
    ```java
    first = new T(); // ERROR 
    ``` 

    类型擦除会使这个操作做成`new Object()`。

* 不能建立一个泛型数组
    ```java
    public<T> T[] minMax(T[] a){  
        T[] mm = new T[2]; // ERROR
    }  
    ```

    擦除会使这个方法总是构靠一个`Object[2]`数组。但可用反射构造泛型对象和数组:
    ```java
    public static <T extends Comparable> T[]minmax(T[] a)  {  
        T[] mm == (T[])Array.newInstance(a.getClass().getComponentType(),2);   
        // 替换掉以下代码  
        // Obeject[] mm = new Object[2];
    }
    ```

<br>
  

### 类型擦除后的冲突
泛型类型擦除后，创建条件不能产生冲突。

如果在Pair类中添加下面的`equals()`方法：
```java
class Pair<T>   {  
    public boolean equals(T value) {  
        return null;  
    }    
}  
```

考虑一个`Pair<String>`。概念上，它有两个`equals()`方法：
```java
boolean equals(String); // 在Pair<T>中定义
boolean equals(Object); // 从object中继承
```

这只是一种错觉。实际上，擦除后方法`boolean equals(T)`变成了方法`boolean equals(Object)`。这与`Object.equals()`方法是冲突的。补救办法是重新命名引发错误的方法。

<br>


### 泛型在静态方法和静态类中的问题
泛型类中的静态方法和静态变量不可以使用泛型类所声明的泛型类型参数。

```java
public class Test2<T> {    
    public static T one;   // 编译错误    
    public static  T show(T one){ // 编译错误    
        return null;    
    }    
}    
```
因为泛型类中的泛型参数的实例化是在定义对象的时候指定的，而静态变量和静态方法不需要使用对象来调用。

但是要注意区分下面的情况：
```java
public class Test2<T> {    
    public static <T> T show(T one){ // 正确的    
        return null;    
    }    
}    
```
因为这是一个泛型方法，在泛型方法中使用的T是自己在方法中定义的T，而不是泛型类中的T。