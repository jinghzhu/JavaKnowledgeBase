# <center>Class Loader</center>

<br></br>



## 装载类分类
--- 
1. 系统类 Bootstrap Loader – It loads JDK internal classes, typically loads rt.jar and other core classes. 
2. 扩展类 ExtClass Loader – It loads classes from the JDK extensions directory, usually `$JAVA_HOME/lib/ext` directory.
3. 由程序员自定义的类 AppClass Loader - It loads classes from the current classpath that can be set while invoking a program using -cp or -classpath command line options.

类装载器实质上也是类，功能是把类载入JVM中， 层次结构如下： 
  BootstrapLoader - 加载系统类 
    | 
   - - ExtClassLoader - 加载扩展类 
                | 
               - - AppClassLoader - 加载应用类 

<br></br>



## 类装载方式
----
1. 隐式装载，程序在运行过程中当碰到通过new等方式生成对象时，隐式调用类装载器加载对应的类到JVM中。
2. 显式装载，通过`class.forname()`等方法，显式加载需要的类。 

Java启动时，不是一次把所有类全部加载后再运行。先把保证程序运行的基础类一次性加载到JVM中，其它类等到JVM用时再加载。因为java最早为嵌入式系统而设计。用到时再加载是java动态性的体现。
   
<br></br>



## 类加载器之间是如何协调工作
----
采用委托模型机制，就是类装载器有载入类的需求时，先请示其Parent使用其搜索路径帮忙载入，如果Parent找不到,才依自己的搜索路径搜索类，具有递归性。

这一点是从安全的方面考虑的。如果有人写了一个恶意的基础类（如`java.lang.String`）并加载到JVM将引起严重后果，但是全盘负责制，`java.lang.String`永远是由根装载器来装载的，避免以上情况发生。

```java
public class TestClass {  
  public static void main(String[] args)  throws Exception{
        //调用class加载器  
        ClassLoader cl = TestClass.class.getClassLoader();  
        System.out.println(cl); // sun.misc.Launcher$AppClassLoader@7259da   
        //调用上一层Class加载器  
        ClassLoader clParent = cl.getParent();  
        System.out.println(clParent); // sun.misc.Launcher$ExtClassLoader@16930e2 
        //调用根部Class加载器  
        ClassLoader clRoot = clParent.getParent();  
        System.out.println(clRoot); // null           
    }  
} 
```

TestClass由AppClassLoader加载器加载，AppClassLoader的Parent加载器是ExtClassLoader，但ExtClassLoader的Parent为`null`，因为Bootstrap Loader是C++写的，对于Java逻辑上不存在Bootstrap Loader类实体。

``` java
public class ClassLoaderTest {
    public static void main(String[] args) {
        System.out.println("class loader for HashMap: "
                + java.util.HashMap.class.getClassLoader());

        System.out.println("class loader for DNSNameService: "
                + sun.net.spi.nameservice.dns.DNSNameService.class
                        .getClassLoader());

        System.out.println("class loader for this class: "
                + ClassLoaderTest.class.getClassLoader());
 
        System.out.println(com.mysql.jdbc.Blob.class.getClassLoader());
    }
}
```

The output is:
``` 
class loader for HashMap: null
class loader for DNSNameService: sun.misc.Launcher$ExtClassLoader@7c354093
class loader for this class: sun.misc.Launcher$AppClassLoader@64cbbe37
sun.misc.Launcher$AppClassLoader@64cbbe37
```

`java.util.HashMap` ClassLoader is null that reflects Bootstrap ClassLoader whereas `DNSNameService` ClassLoader is ExtClassLoader. Since the class itself is in `CLASSPATH`, System ClassLoader loads it.

When trying to load `HashMap`, ystem ClassLoader delegates it to the Extension ClassLoader, which delegates it to Bootstrap ClassLoader that found the class and load it in JVM. The same process is followed for DNSNameService class but Bootstrap ClassLoader is not able to locate it since its in `$JAVA_HOME/lib/ext/dnsns.jar` and hence gets loaded by Extensions Class Loader. Note that Blob class is included in the MySql JDBC Connector jar (mysql-connector-java-5.0.7-bin.jar) that I have included in the build path of the project before executing it and its also getting loaded by System Class Loader.

One more important point to note is that Classes loaded by a child class loader have visibility into classes loaded by its parent class loaders. So classes loaded by System ClassLoader have visibility into classes loaded by Extensions and Bootstrap ClassLoader.

<br></br>



## 加载class文件原理
----
类装载器ClassLoader是寻找类或接口字节码文件进行解析并构造JVM内部对象表示的组件，装载类到JVM的步骤为：
1. 装载：查找和导入class文件
2. 连接： 
    1. 检查：检查载入的class文件数据的正确性
    2. 准备：给类的静态变量分配存储空间
    3. 解析：将符号引用转换成直接引用（可选）
3. 初始化：对静态变量，静态代码块执行初始化

<br></br>


## Why write a ClassLoader?
----
Java default ClassLoader can load files from local file system that is good enough for most of the cases. But if you are expecting a class at the runtime or from FTP server or via third party web service at the time of loading the class then you have to extend the existing class loader. For example, AppletViewers load the classes from remote web server.

