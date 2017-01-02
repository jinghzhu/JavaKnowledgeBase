# <center>Class Loader</center>


## 1. Java中的类分类 
1. 系统类 Bootstrap Loader
2. 扩展类 ExtClass Loader
3. 由程序员自定义的类 AppClass Loader

&#12288;&#12288;类装载器实质上也是类，功能是把类载入jvm中， 层次结构如下： 
  BootstrapLoader - 加载系统类 
    | 
   - - ExtClassLoader - 加载扩展类 
                | 
               - - AppClassLoader - 加载应用类 



## 2.类装载方式
1. 隐式装载，程序在运行过程中当碰到通过new 等方式生成对象时，隐式调用类装载器加载对应的类到jvm中， 
2. 显式装载，通过`class.forname()`等方法，显式加载需要的类 
   


## 3.类加载的动态性体现 
&#12288;&#12288;Java程序启动时，不是一次把所有类全部加载后再运行。先把保证程序运行的基础类一次性加载到jvm中，其它类等到jvm用时再加载，这样的好处是节省了内存的开销。因为java最早为嵌入式系统而设计。用到时再加载是java动态性的体现。



## 4．类加载器之间是如何协调工作的 
&#12288;&#12288;采用委托模型机制，就是类装载器有载入类的需求时，先请示其Parent使用其搜索路径帮忙载入，如果Parent 找不到,才依照自己的搜索路径搜索类。这句话具有递归性。

```java
public class TestClass {  
 
  public static void main(String[] args)  throws Exception{
        //调用class加载器  
        ClassLoader cl = TestClass.class.getClassLoader();  
        System.out.println(cl);  
        //调用上一层Class加载器  
        ClassLoader clParent = cl.getParent();  
        System.out.println(clParent);  
        //调用根部Class加载器  
        ClassLoader clRoot = clParent.getParent();  
        System.out.println(clRoot);          
    }  
} 
```
 
&#12288;&#12288;log信息如下：
```  
sun.misc.Launcher$AppClassLoader@7259da  
sun.misc.Launcher$ExtClassLoader@16930e2  
null  
```

&#12288;&#12288;可以看出TestClass是由AppClassLoader加载器加载.AppClassLoader的Parent加载器是ExtClassLoader.但是ExtClassLoader的Parent为`null`，因为Bootstrap Loader是C++写的,对于Java逻辑上并不存在Bootstrap Loader的类实体，所以在java程序代码里试图打印出其内容时，就会看到输出为`null`。



## 5. 预先加载与依需求加载 
&#12288;&#12288;为了优化系统,在JRE运行开始会将Java运行所需要的基本类采用预先加载(pre-loading)方法加载。因为这些单元在Java运行过程中经常要使用，主要包括JRE的rt.jar文件里面所有的.class文件。 

&#12288;&#12288;当java.exe虚拟机开始运行后，它会找到JRE环境然后把控制权交给JRE。JRE的类加载器会将 lib目录下的rt.jar基础类别文件库加载进内存，这些文件是Java程序执行所必须的，所以系统在开始就将这些文件加载，避免以后的多次IO操作。 

&#12288;&#12288;相对于预先加载，在程序中需使用自定义类时要使用依需求加载方法(load-on-demand。



## 6. JVM加载class文件的原理
&#12288;&#12288;类装载器ClassLoader是寻找类或接口字节码文件进行解析并构造JVM内部对象表示的组件，装载类到JVM的步骤为：
1. 装载：查找和导入class文件
2. 连接： 
    1. 检查：检查载入的class文件数据的正确性
    2. 准备：给类的静态变量分配存储空间
    3. 解析：将符号引用转换成直接引用（可选）
3. 初始化：对静态变量，静态代码块执行初始化



## 7. 为何java装载类使用全盘负责委托机制
&#12288;&#12288;**全盘负责委托机制**是指一个ClassLoader装载类时，除非显示使用另一个ClassLoader，该类所依赖及引用的类也由这个ClassLoader载入。

&#12288;&#12288;先委托父类装载器寻找目标类，只有在找不到的情况下才从自己的路径中查找并载入。这一点是从安全的方面考虑的。如果有人写了一个恶意的基础类（如`java.lang.String`）并加载到JVM将引起严重后果，但是全盘负责制，`java.lang.String`永远是由根装载器来装载的，避免以上情况发生。
