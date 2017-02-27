# <center>FAQ</center>

<br></br>



## 1. JDK and JRE
* Java Development Kit (JDK) is for development purpose and JVM is a part of it to execute the java programs. JDK provides all the tools, executables and binaries required to compile, debug and execute a Java Program. The execution part is handled by JVM to provide machine independence.
* Java Runtime Environment (JRE) is the implementation of JVM. JRE consists of JVM and java binaries and other classes to execute any program successfully. JRE doesn’t contain any development tools like java compiler, debugger etc. The task of java compiler is to convert java program into bytecode, we have javac executable for that. So it must be stored in JDK, we don’t need it in JRE and JVM is just the specs.

<br></br>



## 2. newInstance与new区别
* `newInstance()`是方法，`new`是关键字一个是方法
* 创建对象的方式不一样，前者是使用类加载机制，后者是创建一个新类。 
* 使用`new`创建类时，这个类可以没有被加载。但用`newInstance()`方法就须保证：1、这个类已经加载；2、这个类已经连接了。而完成上面两个步骤的是`Class`的静态方法`forName()`完成的，这个静态方法调用了启动类加载器，即加载java API的那个加载器。 
* newInstance()是把`new`分解为两步，先调用`Class`加载方法加载某个类，然后实例化。 

<br></br>



## 3. JAVA范型
* 泛型的类型参数只能是类类型（包括自定义类），不能是简单类型。
* 泛型的参数类型可以使用extends语句，例如`<T extends superclass>`。
* 泛型的参数类型还可以是通配符类型。例如`Class<?> classType = Class.forName(java.lang.String);`


### 3.1 范型类
``` java
public class Gen<T> {
    private T t;
    public Gen(T t) {
        this.t = t;
    }
}

Gen<String> gen1=new Gen<String>("");
Gen<Integer> gen2=new Gen<Integer>(1);
```


### 3.2范型方法
&#12288;&#12288;必须在方法的修饰符（public, static, final, abstract）之后，返回值声明之前:
* 正确：`public static <E> void printArray(E[] a)`
* 错误：`public static void <E> printArray(E[] a)`

<br></br>



## 4. JAVA使用相对路径读取文件
&#12288;&#12288;java project环境，使用java.io用相对路径读取文件的例子：
&#12288;&#12288;目录结构：
  DecisionTree
            |___src
                 |___com.decisiontree.SamplesReader.java
            |___resource
                 |___train.txt,test.txt
&#12288;&#12288;SamplesReader.java:

``` java
String filepath="resource/train.txt"; //注意filepath的内容；
File file=new File(filepath);
```

&#12288;&#12288;java.io默认定位到当前用户目录下，即：工程根目录"`D:\DecisionTree`下.因此，此时的相对路径为`resource/train.txt`。这样，JVM就可以根据"user.dir"与"resource/train.txt"得到完整的路径（即绝对路径）`D:\DecisionTree\resource\train.txt`，找到train.txt文件。

<br></br>



## 5. 作用域   

修饰词   | 当前类      |    同一package | 子孙类 | 其他package
:--------: | :--------: | :----------: | :----: |
public  | √ |  √ |  √ | √ 
protected  | √ |  √ | √| × 
friendly   | √ | √ | × | × 
private    | √ | × | × | × 

<br></br>



## 6. 字和字节的区别 
Byte(字节) = 8 bits

<br></br>



## 7. for vs foreach
1. 使用foreach来遍历集合时，集合必须实现Iterator接口，foreach就是使用Iterator接口来实现对集合的遍历的
2. 在用foreach循环遍历一个集合时不能向集合中增加元素，不能从集合中删除元素，否则会抛出ConcurrentModificationException异常。抛出该异常是因为在集合内部有一个modCount变量用于记录集合中元素的个数，当向集合中增加或删除元素时，modCount也会随之变化，在遍历开始时会记录modCount的值，每次遍历元素时都会判断该变量是否发生了变化，如果发生了变化则抛出ConcurrentModificationException异常
3. for效率最好，尤其是实现RandomAccess接口的collection，但在linkedlist中，iterator效果更好;foreach效率最差，因为其实现一个Enum，每次都要掉用。
4. 当使用foreach循环基本类型时变量时不能修改集合中的元素的值，遍历对象时可以修改对象的属性的值，但是不能修改对象的引用
修改基本类型的值（原集合中的值没有变化，因为str是集合中变量的一个副本）：

``` java
    public static void main(String[] args) {
        List<String> list = new ArrayList<String>();
        list.add("0");
        list.add("1");
        list.add("2");
        for(String str:list){
            str=str+"0";
            System.out.println(str);
        }
        System.out.println(list);
    }

// 修改对象的值（可以修改，因为f是一个指针）：
public class ForE {
    private String name;
    private int age;
    
    public ForE(String name, int age) {
        super();
        this.name = name;
        this.age = age;
    }

    public static void main(String[] args) {
        List<ForE> list = new ArrayList<ForE>();
        list.add(new ForE("apple", 10));
        list.add(new ForE("banana", 20));
        list.add(new ForE("orange", 30));
        for(ForE f:list){
            f.setAge(f.getAge()*2);
        }
        System.out.println(list);
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    @Override
    public String toString() {
        return "ForE [name=" + name + ", age=" + age + "]";
    }
    
}
```

<br></br>



## 8. Can we have try without catch block? 
Yes, we can have try-finally statement and hence avoiding catch block.

<br></br>



## 9. 空字符
&#12288;&#12288;Java没有空字符`’’`，只有`(Character) null`

<br></br>



## 10. 自定义Exception

```java
public class custExpetion1 {
    public static void main(String[] args) {
        int i = 1, j = -1;
        try{
            if(i < 0 || j < 0){
                throw new Exception("error");
            }
        }catch(Exception e){
            System.out.println(e.getMessage());;
        }
        
        Calculator myCalculator = new Calculator();
        try{
            int ans=myCalculator.power(i,j);
            System.out.println(ans);
            
        }
        catch(Exception e)
        {
            System.out.println(e.getMessage());
        }
    }
}

class Calculator{
    public int power(int i, int j) throws Exception{
        if(i < 0 || j < 0){
            throw new Exception("n and p should be non-negative");
        }
        …
    }
}
```

<br></br>



## 11. 面向对象的特征有哪些方面 
1. 抽象(abstraction)：抽象是忽略一个主题中与当前目标无关的那些方面，以便更充分地注意与当前目标有关的方面。包括两个方面，一是过程抽象，二是数据抽象。
2. 继承(inheritance)
3. 封装(encapsulation)：封装是把过程和数据包围起来，对数据的访问只能通过已定义的界面。面向对象计算始于这个基本概念，即现实世界可以被描绘成一系列完全自治、封装的对象，这些对象通过一个受保护的接口访问其他对象。
4.  多态性(polymorphism)：指允许不同类的对象对同一消息作出响应。多态性包括参数化多态性和包含多态性。多态性语言具有灵活、抽象、行为共享、代码共享的优势，很好的解决了应用程序函数同名问题。

<br></br>



## 12. JAVA vs C++
* JVM虚拟机内部自己管理指针
* 多重继承
* 自动内存管理
* 操作符重载
* JAVA不支持缺省函数参数，C++支持
* JAVA的try／catch方便处理异常，C++没有

<br></br>



## 13. final对象是否有默认值
```java
class Something{
    int i;
    public void doSomething(){
         System.out.println(“i = “ + i);
    }
}
```

Correct, the output is `i = 0`. Because `i` is instant variable which has default value.

``` java
      class Something{
         final int i;
         public void doSomething(){
               System.out.println(“i = “ + i);
         }
      }
```

Wrong, because the instant variable with keyword `final` has no default value which means we must set a value for it.

<br></br>



## 14. Writer和Reader： 
&#12288;&#12288;Writer和Reader用于字符流的写入和读取，也就是说写入和读取的单位是字符，如果读与写的操作不涉及字符，那么是不需要Writer和Reader的。 
* Writer类（抽象类），子类必须实现的方法仅有 write()、flush()、close()。继承了此类的类是用于写操作的 “Writer”。 
* Reader 类（抽象类），子类必须实现的方法仅有 read()、close()。继承了此类的类是用于读操作的“Reader”。
* write()方法是将数据写入到文件（广义概念，包括字节流什么的）中。
* read()方法是将文件（广义概念，包括字节流什么的）中的数据读出到缓冲目标上。 
 
<br></br>



## 15. InputStream和OutputStream： 
* InputStream是表示字节输入流的所有类的超类。字节输入流相当于是一个将要输入目标文件的“流”。InputStream有 read() 方法而没有write()方法，因为它本身代表将要输入目的文件的一个“流” 
* OutputStream：此抽象类是表示输出字节流的所有类的超类。输出流接受输出字节并将这些字节发送到某个接收器。是从文件中将要输出到某个目标的“流”。OutputStream有 write()方法而没有read()方法。
* InputStreamReader是字节流通向字符流的桥梁：它使用指定的 charset 读取字节并将其解码为字符。它使用的字符集可以由名称指定或显式给定，或者可以接受平台默认的字符集。 
* OutputStreamWriter是字符流通向字节流的桥梁：可使用指定的 charset 将要写入流中的字符编码成字节。它使用的字符集可以由名称指定或显式给定，否则将接受平台默认的字符集。

&#12288;&#12288;BufferReader和BufferWriter：缓冲机制是说先把数据存到缓冲内存中，再一次写入文件，减少打开文件的消耗。

<br></br>



## 16. 文件目录操作
### 16.1 如何列出某个目录下的所有文件? 

``` java
File file = new File("e:\\总结"); 
File[] files = file.listFiles(); 
for(int i=0; i<files.length; i++) { 
    if(files[i].isFile()) 
        System.out.println(files[i]);
}
```


### 16.2 如何列出某个目录下的所有子目录?

``` java
File file = new File("e:\\总结"); 
File[] files = file.listFiles(); 
for(int i=0; i<files.length; i++){ 
    if(files[i].isDirectory()) 
        System.out.println(files[i]);
}
```


### 16.3 如何判断一个文件或目录是否存在?
&#12288;&#12288;创建`File`对象,调用其`exsit()`方法即可返回是否存在,如: 
``System.out.println(new File("d:\\t.txt").exists());``


### 16.4 如何读写文件? 

``` java
//读文件:
FileInputStream fin = new FileInputStream("e:\\tt.txt"); 
byte[] bs = new byte[100];

while(true){ 
          int len = fin.read(bs);
          if(len <= 0) 
              break;
          System.out.print(new String(bs,0,len));
}

fin.close();
```

``` java
//写文件:
FileWriter fw = new FileWriter("e:\\test.txt");

fw.write("hello world!" + System.getProperty("line.separator")); 
fw.write("你好!北京!"); 
fw.close(); 
```

<br></br>


