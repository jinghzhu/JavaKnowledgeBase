# <center>Changelog</center>

<br></br>



## v1.8
----
### Lambda(from Scala)

```java
Runnable oldRunner = new Runner() {
    public void run() {
        // do something
    }
}

Runnable newRunner = () -> {
    // do something
}
```

<br>


### Data/Time changes
Most classes are thread safe and immutable.

<br>


### Parallel Array Sorting
This feature adds the same set of sorting operations currently provided by the Arrays class, but with a parallel implementation that utilizes the Fork/Join framework. The methods are called `parallelSort()` and are overloaded for all the primitive data types and Comparable objects.

<br></br>



## v1.9
----
### 平台级模块系统
模块化JAR文件都包含一个额外的模块描述器。在模块描述器中，对其它模块依赖通过`requires`关键字表示。另外，`exports`语句控制哪些包可被其它模块访问。所有不被导出的包默认封装在模块里面。

如下给出一个示例。模块*com.mycompany.sample*导出Java包*com.mycompany.sample*。该模块依赖于模块*com.mycompany.sample*。该模块也提供服务接口*com.mycompany.common.DemoService*实现类*com.mycompany.sample.DemoServiceImpl*。 

```java
module com.mycompany.sample { 
    exports com.mycompany.sample; 
    requires com.mycompany.common; 
    provides com.mycompany.common.DemoService with
        com.mycompany.sample.DemoServiceImpl; 
}
```

模块系统增加了模块路径概念。模块系统在解析模块时，从模块路径中查找。为保持与之前Java版本兼容，`CLASSPATH`被保留。所有类型在运行时都属于某个特定模块。对于从`CLASSPATH`中加载的类型，它们属于加载它们的类加载器对应的未命名模块。可通过*Class*`getModule()`方法获取表示其所在模块的Module对象。

JVM启动时，从应用根模块开始，根据依赖关系递归的进行解析，直到得到一个表示依赖关系的图。如果解析过程中出现找不到模块情况，或在模块路径同一个地方找到名称相同的模块，模块解析过程终止，JVM退出。

<br>


### JShell
```bash
jshell> int add(int x, int y) { 
    ...> return x + y; 
    ...> } 
 | created method add(int,int)
```

<br>


### Security
新增4个SHA-3哈希算法。也增加了通过*java.security.SecureRandom*生成使用DRBG算法的强随机数。

<br>


### 改进Javadoc
Javadoc现支持在API文档中进行搜索。另外，Javadoc输出符合兼容HTML5标准。

<br>


### HTTP/2
有新方式处理HTTP调用，用于代替`HttpURLConnection` API，并提供对WebSocket和HTTP/2支持。

<br>


### 多版本兼容JAR
当新版本Java出现时，库用户要花费很久才会切换到新版本。意味着库得向后兼容想要支持的老Java版本。如今，多版本兼容JAR功能能创建仅在特定版本Java环境中运行库程序时选择使用的class版本：

```
multirelease.jar
├── META-INF
│   └── versions
│       └── 9
│           └── multirelease
│               └── Helper.class
├── multirelease
    ├── Helper.class
    └── Main.class
```

*multirelease.jar*可在Java 9使用，不过*Helper*类使用的不是顶层*multirelease.Helper*，而是处在`META-INF/versions/9`下面的。这是特别为Java 9准备的class版本。同时，在早期Java中使用这个JAR也能运行，因为老版本Java只会看到顶层的*Helper*类。

<br></br>



## v1.10
----
### 局部变量类型推断（语法糖）
之前：
```java
List<String> list = new ArrayList<String>();
```

现在：
```java
var list = new ArrayList<String>();
```

<br>


### 将JDK生态整合单个代码库

<br>


### GC优化
一个干净的垃圾收集器接口，用来改善垃圾收集器源代码之间的隔离效果。为HotSpot中内部垃圾收集代码提供更好模块化功能，也可更容易向HotSpot添加新的垃圾收集器。

此外，实现并行、完整G1垃圾收集器，通过实现并行性来改善最坏情况下的延迟问题。

<br>


### 内存分配
启用HotSpot将对象堆分配给指定的备用内存设备。预示未来系统可能会采用异构内存架构。