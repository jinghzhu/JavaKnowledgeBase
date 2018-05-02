# <center>Serializable</center>



<br></br>

* 序列化时，只对对象状态进保存，不管对象方法。
* 当父类实现序列化，子类自动实现序列化，不需显式实现Serializable接口。
* 当对象实例变量引用其他对象，序列化该对象时也把引用对象序列化。
* `static`成员（序列化保存对象状态，静态变量属于类状态）和`transient`修饰的字段不被序列化。
* 如果可序列化对象包含某个不可序列化对象引用，那么序列化操作失败，抛出`NotSerializableException`。
* 父类如果不可序列化，子类不会序列化父类成员，除非在子类中显式序列化。父类可序列化，子类也需调用super的序列化方法。

<br></br>



## Serializable原理
----
为什么实现Serializable接口就可被序列化？之前使用ObjectOutputStream来持久化对象，在该类中有如下代码：

```java
private void writeObject0(Object obj, boolean unshared) throws IOException {  
    if (obj instanceof String) {
        writeString((String) obj, unshared);
    } else if (cl.isArray()) {
        writeArray(obj, desc, unshared);
    } else if (obj instanceof Enum) {
        writeEnum((Enum) obj, desc, unshared);
    } else if (obj instanceof Serializable) {
        writeOrdinaryObject(obj, desc, unshared);
    } else {
        if (extendedDebugInfo) {
            throw new NotSerializableException(cl.getName() + "\n"
                    + debugInfoStack.toString());
        } else {
            throw new NotSerializableException(cl.getName());
        }
    } 
}
```

如果被写对象的类型是`String`或数组或`Enum`或Serializable，那么就可以对该对象进行序列化，否则将抛出`NotSerializableException`。

<br></br>



## Externalizable
----
使用该接口后，之前基于Serializable接口的序列化机制将失效。Externalizable继承Serializable，当使用该接口时，序列化细节由程序员完成。

另外，若使用Externalizable序列化，当读取对象时，会调用被序列化类无参构造器去创建新对象，然后再将被保存对象的字段的值分别填充到新对象中。因此，实现Externalizable接口类须提供无参构造器，且访问权限为`public`。

```java
public class Person implements Externalizable {
    private String name = null;
    transient private Integer age = null;
    private Gender gender = null;

    public Person() {
        System.out.println("none-arg constructor");
    }

    public Person(String name, Integer age, Gender gender) {
        System.out.println("arg constructor");
        this.name = name;
        this.age = age;
        this.gender = gender;
    }

    private void writeObject(ObjectOutputStream out) throws IOException {
        out.defaultWriteObject();
        out.writeInt(age);
    }

    private void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {
        in.defaultReadObject();
        age = in.readInt();
    }

    @Override
    public void writeExternal(ObjectOutput out) throws IOException {
        out.writeObject(name);
        out.writeInt(age);
    }

    @Override
    public void readExternal(ObjectInput in) throws IOException, ClassNotFoundException {
        name = (String) in.readObject();
        age = in.readInt();
    }
}
```

执行`SimpleSerial`结果：
```
arg constructor
none-arg constructor
[John, 31, null]
```

<br></br>



## Serializable vs Externalizable
----

|   接口          |   优点             |   缺点   |
| ---------      | ----------         | ---------- |
| Serializable   | 内建支持，易于实现    | 占用空间大，由于额外的开销导致速度慢 |
| Externalizable | 自由，自己决定存储什么 | JVM不提供帮忙，开发量大 |


Serializable实现方法：
```java
    private void writeObject(ObjectOutputStream oos) {
//        oos.defaultWriteObject();
        // Write/save additional fields
        oos.writeUTF(value);
    }

    private void readObject(ObjectInputStream ois) {
//        ois.defaultReadObject();
        // Read/initialize additional fields
        value = ois.readUTF()
    }
```

Externalizable必须实现方法：
```java
    void writeExternal(ObjectOutput out) {
        out.writeUTF(value)
    }

    void readExternal(ObjectInput input) {
        value = input.readUTF()
    }
```

<br></br>



## 序列化ID问题
----
**情境：**两个客户端A和B通过网络传递对象数据，A将对象C序列化传给B，B反序列化得到C。

**问题：**C对象路径为`com.inout.Test`。在A和B端都有这么一个类文件，功能代码一致，也都实现了Serializable接口，但反序列化不成功。

**解决：**JVM是否允许反序列化，不仅取决于类路径和功能代码是否一致，更要求两个类的序列化ID是否一致（`private static final long serialVersionUID = 1L`）。

```java
 import java.io.Serializable; 

 public class A implements Serializable { 
     private static final long serialVersionUID = 1L; 
     private String name; 
 } 
```

```java
 import java.io.Serializable; 

 public class A implements Serializable { 
     private static final long serialVersionUID = 2L; 
     private String name; 
 }
```

序列化ID提供两种生成策略：
1. 固定的_1L_。
2. 随机生成一个不重复的long数据。

<br></br>



## 父类序列化与transient关键字
----
**情境：**子类实现了Serializable接口，父类没有实现Serializable接口。序列化子类对象，然后反序列化后输出父类定义的某变量数值，该变量数值与序列化时数值不同。

**解决：**要将父类对象也序列化，就需让父类也实现Serializable接口。如果父类不实现，要有默认无参构造函数。

在父类没有实现Serializable接口时，JVM不会序列化父对象，而Java对象构造必须先有父对象，才有子对象，反序列化也不例外。

所以为了构造父对象，只能调用父类无参构造函数作为默认父对象。因此取父对象变量值时，它的值是调用父类无参构造函数后的值。所以这个情况要在父类无参构造函数中对变量进行初始化，否则父类变量值都是默认声明的值。

`transient`可阻止该变量被序列化到文件中，在被反序列化后，`transient`值被设为初始值，如`int`是_0_。

<br></br>



## 加密
----
**情境：**服务器端对密码字段序列化时进行加密，而客户端如果拥有解密的密钥，只有在客户端进行反序列化时，才可以对密码进行读取 。

**解决：**序列化过程中，JVM会调用对象类`writeObject()`和`readObject()`方法，进行用户自定义的序列化和反序列化。如果没有这样的方法，则默认调用ObjectOutputStream的`defaultWriteObject()`方法及ObjectInputStream的`defaultReadObject()`方法。用户自定义的`writeObject()`和`readObject()`方法允许用户控制序列化过程，比如可以在序列化的过程中动态改变序列化的数值。

基于这个原理，可用于敏感字段的加密工作。

```java
    private static final long serialVersionUID = 1L;
    public String password = "pass";

    private void writeObject(ObjectOutputStream out) {
        try {
            PutField putFields = out.putFields();
            System.out.println("原密码:" + password);
            password = "encryption";//模拟加密
            putFields.put("password", password);
            System.out.println("加密后的密码" + password);
            out.writeFields();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private void readObject(ObjectInputStream in) {
        try {
            GetField readFields = in.readFields();
            Object object = readFields.get("password", "");
            System.out.println("要解密的字符串:" + object.toString());
            password = "pass";//模拟解密,需要获得本地的密钥
        } catch (IOException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }

    }

    public static void main(String[] args) {
        try {
            ObjectOutputStream out = new ObjectOutputStream(
                    new FileOutputStream("result.obj"));
            out.writeObject(new Test());
            out.close();

            ObjectInputStream oin = new ObjectInputStream(new FileInputStream(
                    "result.obj"));
            Test t = (Test) oin.readObject();
            System.out.println("解密后的字符串:" + t.password);
            oin.close();
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
    }
```

<br></br>



##  序列化存储规则
----

```java
    ObjectOutputStream out = new ObjectOutputStream(
                new FileOutputStream("result.obj"));
    Test test = new Test();
    // 试图将对象两次写入文件
    out.writeObject(test);
    out.flush();
    System.out.println(new File("result.obj").length());
    out.writeObject(test);
    out.close();
    System.out.println(new File("result.obj").length());

    // 从文件依次读出两个文件
    ObjectInputStream oin = new ObjectInputStream(new FileInputStream("result.obj"));
    Test t1 = (Test) oin.readObject();
    Test t2 = (Test) oin.readObject();
    oin.close();
            
    // 判断两个引用是否指向同一个对象
    System.out.println(t1 == t2);
```

输出:
```
31
36
true
```

为何第二次写入对象时文件只增加了5字节，并且两个对象是相等的？

因为Java序列化为节省磁盘空间，具有特定的存储规则。当写入文件为同一对象时，不会再将对象内容进行存储，而只是再次存储一份引用。上面增加的5字节的存储空间是新增引用和一些控制信息的空间。反序列化时，恢复引用关系，使`t1`和`t2`指向唯一对象。

<br></br>



## FAQ
----
* How many methods Serializable has? If no method then what is the purpose of Serializable interface?

    Serializable interface doesn't have any method and also called Marker Interface.


* What is `serialVersionUID`? What would happen if you don't define this?

    `serialVersionUID` is `public static final` constant which should define in your class otherwise compiler will throw warning. 

    If do not specify `serialVersionUID`, Java compiler automatically generates it while persisting the object and uses its own algorithm to generate it which is normally based on fields of class and normally represent hash code of object. Consequence of not implementing `serialVersionUID` is that when you add or modify any field in class then already serialized class will not be able to recover because `serialVersionUID` generated for new class and for old serialized object will be different. 

* 为何ArrayList中数组是transient修饰？

    假如实际有5个元素，而`elementData`是10，那么序列化时只需储存5个元素。所以设计为`transient`，在`writeObject()`中手动序列化，且只序列化了实际存储的元素，而不是整个数组。

* 若通过ObjectOutputStream向文件中多次以追加方式写入object，为什么用ObjectInputStream读取这些object时会产生StreamCorruptedException？ 

    使用缺省的serializetion实现时，一个ObjectOutputStream构造和一个ObjectInputStream构造须一一对应。ObjectOutputStream构造函数会向输出流中写入一个标识头，而ObjectInputStream会首先读入标识头。因此，多次以追加方式向文件中写入object时，该文件将会包含多个标识头。所以用ObjectInputStream来deserialize这个ObjectOutputStream时，将产生StreamCorruptedException。 

    解决方法是构造一个ObjectOutputStream子类，并覆盖`writeStreamHeader()`方法。被覆盖后的`writeStreamHeader()`方法应判断是否为首次向文件中写入object。若是，则调用`super.writeStreamHeader()`；若否，即以追加方式写入object时，则应调用`ObjectOutputStream.reset()`方法。
