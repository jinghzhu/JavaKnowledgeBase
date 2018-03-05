# <center>Pass By Value</center>



<br></br>

* 方法不能修改基本数据类型的参数(即数值型和布尔型)
* 方法可以改变对象参数的状态
* 方法不能让对象参数引用新对象

<br></br>



## 基本数据类型为参数
----
``` java
public class ParamTest {
    public static void main(String[] args) {
        int price = 5;
        doubleValue(price);
        System.out.print(price); // 5
    }

    public static void doubleValue(int x) {
        x = 2 * x;
    }
}
```

<p align="center">
  <img src="./Images/pass_by_value1.jpg" />
</p>

<br></br>



## 对象引用为参数
----
``` java
class Student {
    public float score;

    public Student(float score) {
        this.score = score;
    }
}

public class ParamTest {
    public static void main(String[] args) {
        Student stu = new Student(80);
        raiseScore(stu);
        System.out.print(stu.score); // 90
    }

    public static void raiseScore(Student s) {
        s.setScore(s.score + 10);
    }
}
```

<p align="center">
  <img src="./Images/pass_by_value2.jpg" />
</p>

<br></br>


## 对象是值调用+引用传递
----
> 当一个对象实例作为一个参数被传递到方法中时，参数的值就是对该对象的引用。对象的内容可以在被调用的方法中改变，但对象的引用是永远不会改变的。

``` java
public static void swap(Student x, Student y) {
    Student temp = x;
    x = y;
    y = temp;
 } 

class Student {
    public float score;

    public Student(float score) {
        this.score = score;
    }
}

public class ParamTest {
    public static void main(String[] args) {
        Student a = new Student(0);
        Student b = new Student(100);

        System.out.println("交换前：");
        System.out.println("a的分数：" + a.score + "--- b的分数：" + b.score);

        swap(a, b);

        System.out.println("交换后：");
        System.out.println("a的分数：" + a.score + "--- b的分数：" + b.score);
    }

    public static void swap(Student x, Student y) {
        Student temp = x;
        x = y;
        y = temp;
    }
}
```

Output:
``` 
交换前：
a的分数：0.0--- b的分数：100.0

交换后：
a的分数：0.0--- b的分数：100.0
```

`swap()`调用的过程：
1. 将对象`a`，`b`的拷贝赋给`x`，`y`，此时`a`和`x`指向同一对象，`b`和`y`指向同一对象
2. `swap`方法完成`x`，`y`交换，此时`a`，`b`没有变化
3. 方法执行完成，`x`和`y`不再使用，`a`依旧指向`Student(0)`，`b`指向`Student(100)`

首先，创建两个对象：
<p align="center">
  <img src="./Images/pass_by_value3.jpg" />
</p>

<br>

然后，进入`swap`，将对象`a`，`b`的拷贝赋给`x`，`y`：
<p align="center">
  <img src="./Images/pass_by_value4.jpg" />
</p>

<br>

接着，交换`x`，`y`的值：
<p align="center">
  <img src="./Images/pass_by_value5.jpg" />
</p>

<br></br>

