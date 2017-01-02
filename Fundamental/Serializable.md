# <center>Serializable</center>

## 1. Serializable Example

```java
public class  Box implements Serializable  {  
    private int width;  
    private int height;  
  
    public void setWidth(int width){  
        this.width  = width;  
    }  

    public void setHeight(int height){  
        this.height = height;  
    }  
  
    public static void main(String[] args){  
        Box myBox = new Box();  
        myBox.setWidth(50);  
        myBox.setHeight(30);  
  
        try{  
            FileOutputStream fs = new FileOutputStream("foo.ser");  
            ObjectOutputStream os =  new ObjectOutputStream(fs);  
            os.writeObject(myBox);  
            os.close();  
        }catch(Exception ex){  
            ex.printStackTrace();  
        }  
    }     
} 
``` 

&#12288;&#12288;相关注意事项：
* 序列化时，只对对象的状态进行保存，而不管对象的方法；
* 当一个父类实现序列化，子类自动实现序列化，不需要显式实现`Serializable`接口;
* 当一个对象的实例变量引用其他对象，序列化该对象时也把引用对象进行序列化;
* `static`成员(序列化保存的是对象的状态，静态变量属于类的状态) & `transient`修饰的字段不被序列化;
* 如果一个可序列化的对象包含对某个不可序列化的对象的引用，那么整个序列化操作将会失败，并且会抛出一个`NotSerializableException`;
* 父类如果不可序列化，子类不会序列化父类的成员，除非在子类中显式序列化。父类可序列化，子类也需要调用super的序列化方法。


## 2. Serializable的作用
&#12288;&#12288;为什么一个类实现了`Serializable`接口，它就可以被序列化呢？
&#12288;&#12288;在上节的示例中，使用`ObjectOutputStream`来持久化对象，在该类中有如下代码：

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

&#12288;&#12288;如果被写对象的类型是`String`，或数组，或`Enum`，或`Serializable`，那么就可以对该对象进行序列化，否则将抛出`NotSerializableException`。


## 3. Externalizable
&#12288;&#12288;使用该接口之后，之前基于`Serializable`接口的序列化机制就将失效。`Externalizable`继承于`Serializable`，当使用该接口时，序列化的细节需要由程序员去完成。
&#12288;&#12288;另外，若使用`Externalizable`进行序列化，当读取对象时，会调用被序列化类的无参构造器去创建一个新的对象，然后再将被保存对象的字段的值分别填充到新对象中。 由于这个原因，实现`Externalizable`接口的类必须要提供一个无参的构造器，且它的访问权限为`public`。

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

&#12288;&#12288;执行`SimpleSerial`之后会有如下结果：
> arg constructor

> none-arg constructor

> [John, 31, null]


## 4. 两者比较
|   接口   |   优点   |   缺点   |
|---------|----------|----------|
| Serializable | 内建支持，易于实现 | 占用空间大，由于额外的开销导致速度慢 |
| Externalizable | 自由，自己决定存储什么 | JVM不提供帮忙，开发量大 |


&#12288;&#12288;_Serializable_实现方法：

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

&#12288;&#12288;_Externalizable_必须实现方法：

```java
    void writeExternal(ObjectOutput out) {
        out.writeUTF(value)
    }

    void readExternal(ObjectInput input) {
        value = input.readUTF()
    }
```


## 5. 序列化 ID 问题
* 情境：两个客户端A和B通过网络传递对象数据，A将对象C序列化再传给B，B反序列化得到C。

* 问题：C对象的路径假设为`com.inout.Test`，在A和B端都有这么一个类文件，功能代码完全一致。也都实现了`Serializable`接口，但是反序列化总是不成功。

* 解决：虚拟机是否允许反序列化，不仅取决于类路径和功能代码是否一致，重要的一点是两个类的序列化`ID`是否一致（`private static final long serialVersionUID = 1L`）。虽然两个类的功能代码完全一致，但是`ID`不同，他们无法相互序列化和反序列化。

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

&#12288;&#12288;序列化`ID`提供两种生成策略，一个是固定的`1L`，一个是随机生成一个不重复的`long`数据。通过改变`ID`可以用来限制某些用户的使用。


## 6. 父类的序列化与 Transient 关键字
* 情境：一个子类实现了`Serializable`接口，它的父类没有实现`Serializable` 接口，序列化该子类对象，然后反序列化后输出父类定义的某变量的数值，该变量数值与序列化时的数值不同。

* 解决：要想将父类对象也序列化，就需要让父类也实现`Serializable`接口。
如果父类不实现，要有默认的无参构造函数。

&#12288;&#12288;在父类没有实现`Serializable`接口时，虚拟机不会序列化父对象，而Java对象的构造必须先有父对象，才有子对象，反序列化也不例外。

&#12288;&#12288;所以为了构造父对象，只能调用父类的无参构造函数作为默认的父对象。因此取父对象的变量值时，它的值是调用父类无参构造函数后的值。如果考虑到这种序列化的情况，在父类无参构造函数中对变量进行初始化，否则的话，父类变量值都是默认声明的值，如`int`型的默认是_0_。

&#12288;&#12288;`Transient`可以阻止该变量被序列化到文件中，在被反序列化后，`transient`的值被设为初始值，如`int`的是_0_。



## 7. 对敏感字段加密
* 情境：服务器端希望对密码字段在序列化时，进行加密，而客户端如果拥有解密的密钥，只有在客户端进行反序列化时，才可以对密码进行读取 。

* 解决：在序列化过程中，虚拟机会调用对象类里的`writeObject`和`readObject`方法，进行用户自定义的序列化和反序列化，如果没有这样的方法，则默认调用是 `ObjectOutputStream`的`defaultWriteObject`方法以及`ObjectInputStream`的`defaultReadObject`方法。用户自定义的`writeObject`和`readObject`方法允许用户控制序列化过程，比如可以在序列化的过程中动态改变序列化的数值。

&#12288;&#12288;基于这个原理，可以在实际应用中得到使用，用于敏感字段的加密工作。

```java
 private static final long serialVersionUID = 1L;

    private String password = "pass";

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }

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
            System.out.println("解密后的字符串:" + t.getPassword());
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


## 8. 序列化存储规则

```java
 ObjectOutputStream out = new ObjectOutputStream(
                new FileOutputStream("result.obj"));
    Test test = new Test();
    //试图将对象两次写入文件
    out.writeObject(test);
    out.flush();
    System.out.println(new File("result.obj").length());
    out.writeObject(test);
    out.close();
    System.out.println(new File("result.obj").length());

    ObjectInputStream oin = new ObjectInputStream(new FileInputStream("result.obj"));//从文件依次读出两个文件
    Test t1 = (Test) oin.readObject();
    Test t2 = (Test) oin.readObject();
    oin.close();
            
    //判断两个引用是否指向同一个对象
    System.out.println(t1 == t2);
```

&#12288;&#12288;对同一对象两次写入文件，打印出写入一次对象后的存储大小和写入两次后的存储大小，然后从文件中反序列化出两个对象，比较这两个对象是否为同一对象。

&#12288;&#12288;示例程序输出:
> 31

> 36

> true

&#12288;&#12288;为何第二次写入对象时文件只增加了_5_字节，并且两个对象是相等的？

&#12288;&#12288;Java序列化机制为了节省磁盘空间，具有特定的存储规则，当写入文件的为同一对象时，并不会再将对象的内容进行存储，而只是再次存储一份引用。上面增加的_5_字节的存储空间就是新增引用和一些控制信息的空间。反序列化时，恢复引用关系，使得`t1`和`t2`指向唯一的对象，二者相等，输出_true_。



## 9. What is the difference between Serializable and Externalizable interface in Java?
Externalizable provides us `writeExternal()` and `readExternal()` method which gives us flexibility to control java serialization mechanism instead of relying on java's default serialization. Correct implementation of Externalizable interface can improve performance of application drastically.


## 10. How many methods Serializable has? If no method then what is the purpose of Serializable interface?
Serializable interface exists in java.lang package and forms core of java serialization mechanism. It doesn't have any method and also called Marker Interface. When your class implements Serializable interface it becomes Serializable in Java and gives compiler an indication that use Java Serialization mechanism to serialize this object.


## 11. What is serialVersionUID? What would happen if you don't define this?
SerialVersionUID is public static final constant which should define in your class otherwise compiler will throw warning. If you do not specify serialVersionUID in your class Java compiler automatically generates it while persisting the object and uses its own algorithm to generate it which is normally based on fields of class and normally represent hash code of object. Consequence of not implementing serialVersionUID is that when you add or modify any field in class then already serialized class will not be able to recover because serialVersionUID generated for new class and for old serialized object will be different. Java serialization process relies on correct serialVersionUID for recovering state of serialized object.


## 12. What will happen if one of the members in the class doesn't implement Serializable interface?
If you try to serialize an object of a class which implements Serializable, but the object includes a reference to an non-Serializable class then a `NotSerializableException` will be thrown at runtime.


## 13. 为何ArrayList中数组是transient修饰？
&#12288;&#12288;假如实际有_5_个元素，而`elementData`大小是_10_，那么序列化时只需要储存_5_个元素。所以设计为`transient`，在`writeObject`中手动序列化，且只序列化了实际存储的那些元素，而不是整个数组。





