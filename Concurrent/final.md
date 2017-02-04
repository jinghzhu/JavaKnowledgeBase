# <center>final</center>




&#12288;&#12288;对于final域，编译器和处理器要遵守两个重排序规则：
1. 在构造函数内对一个final域的写入，与随后把这个被构造对象的引用赋值给一个引用变量，这两个操作之间不能重排序。
2. 初次读一个包含final域的对象的引用，与随后初次读这个final域，这两个操作之间不能重排序。

&#12288;&#12288;通过代码来说明这两个规则：

``` java
public class FinalExample {
    int i;                            //普通变量
    final int j;                      //final变量
    static FinalExample obj;

    public void FinalExample () {     //构造函数
        i = 1;                        //写普通域
        j = 2;                        //写final域
    }

    public static void writer () {    //写线程A执行
        obj = new FinalExample ();
    }

    public static void reader () {       //读线程B执行
        FinalExample object = obj;       //读对象引用
        int a = object.i;                //读普通域
        int b = object.j;                //读final域
    }
}
```

&#12288;&#12288;假设线程A执行`writer()`方法，随后线程B执行`reader()`方法。下面我们通过这两个线程的交互来说明这两个规则。

<br></br>



## 1. 写final域的重排序规则

&#12288;&#12288;写final域的重排序规则禁止把final域的写重排序到构造函数之外。这个规则包含2方面：
1. JMM禁止编译器把final域的写重排序到构造函数之外。
2. 编译器会在final域的写之后，构造函数return之前，插入一个StoreStore屏障。这个屏障禁止处理器把final域的写重排序到构造函数之外。

&#12288;&#12288;`writer()`方法只包含一行代码：`finalExample = new FinalExample ()`。这行代码包含两个步骤：
1. 构造一个`FinalExample`类型的对象；
2. 把这个对象的引用赋值给引用变量`obj`。

&#12288;&#12288;假设线程B读对象引用与读对象的成员域之间没有重排序，下图是一种可能的执行时序：

![](./Images/final1.png)

&#12288;&#12288;写普通域的操作被编译器重排序到了构造函数外，读线程B错误的读取了普通变量`i`初始化之前的值。而写final域的操作，被写final域的重排序规则“限定”在了构造函数之内，读线程B正确的读取了final变量初始化之后的值。

&#12288;&#12288;写final域的重排序规则可以确保：在对象引用为任意线程可见之前，对象的final域已经被正确初始化过了，而普通域不具有这个保障。以上图为例，在读线程B“看到”对象引用`obj`时，很可能`obj`对象还没有构造完成（对普通域`i`的写操作被重排序到构造函数外，此时初始值2还没有写入普通域`i`）。

<br></br>



## 2. 读final域的重排序规则

&#12288;&#12288;在线程中，初次读对象引用与初次读该对象包含的final域，JMM禁止处理器重排序这两个操作（注意，这个规则仅针对处理器）。编译器会在读final域操作的前面插入一个LoadLoad屏障。

&#12288;&#12288;初次读对象引用与初次读该对象包含的final域，这两个操作之间存在间接依赖关系。由于编译器遵守间接依赖关系，因此编译器不会重排序这两个操作。大多数处理器也遵守间接依赖，大多数处理器也不会重排序这两个操作。但有少数处理器允许对存在间接依赖关系的操作做重排序（比如alpha处理器），这个规则就是专门用来针对这种处理器。

&#12288;&#12288;`reader()`方法包含三个操作：
1. 初次读引用变量`obj`;
2. 初次读引用变量`obj`指向对象的普通域`j`。
3. 初次读引用变量`obj`指向对象的final域`i`。

&#12288;&#12288;假设写线程A没发生重排序，同时程序在不遵守间接依赖的处理器上执行，下面是一种可能的执行时序：

![](./Images/final2.png)

&#12288;&#12288;在上图中，读对象的普通域的操作被处理器重排序到读对象引用之前。读普通域时，该域还没有被写线程A写入，这是一个错误的读取操作。而读final域的重排序规则会把读对象final域的操作“限定”在读对象引用之后，此时该final域已经被A线程初始化过了，这是一个正确的读取操作。

&#12288;&#12288;读final域的重排序规则可以确保：在读一个对象的final域之前，一定会先读包含这个final域的对象的引用。在示例程序中，如果该引用不为`null`，那么引用对象的final域一定已经被A线程初始化过了。

<br></br>



## 3. final域是引用类型

``` java
public class FinalReferenceExample {
	final int[] intArray;                     //final是引用类型
	static FinalReferenceExample obj;

	public FinalReferenceExample () {        //构造函数
    	intArray = new int[1];              //1
    	intArray[0] = 1;                   //2
	}

	public static void writerOne () {          //写线程A执行
    <object></object>bj = new FinalReferenceExample ();  //3
	}

	public static void writerTwo () {          //写线程B执行
    	obj.intArray[0] = 2;                 //4
	}

	public static void reader () {              //读线程C执行
    	if (obj != null) {                    //5
        	int temp1 = obj.intArray[0];       //6
    	}
	}
}
```

&#12288;&#12288;这里final域为引用int型的数组对象。对于引用类型，写final域的重排序规则对编译器和处理器增加了如下约束：
* 在构造函数内对一个final引用的对象的成员域的写入，与随后在构造函数外把这个被构造对象的引用赋值给一个引用变量，这两个操作之间不能重排序。

&#12288;&#12288;假设首先线程A执行`writerOne()`方法，执行完后线程B执行`writerTwo()`方法，执行完后线程C执行`reader()`方法。下面是一种可能的线程执行时序：

![](./Images/final3.png)

&#12288;&#12288;1是对final域的写入，2是对这个final域引用的对象的成员域的写入，3是把被构造的对象的引用赋值给某个引用变量。这里除了前面提到的1不能和3重排序外，2和3也不能重排序。

&#12288;&#12288;JMM可以确保读线程C至少能看到写线程A在构造函数中对final引用对象的成员域的写入。即C至少能看到数组下标0的值为1。而写线程B对数组元素的写入，读线程C可能看的到，也可能看不到。JMM不保证线程B的写入对读线程C可见，因为写线程B和读线程C之间存在数据竞争，此时的执行结果不可预知。

&#12288;&#12288;如果想要确保读线程C看到写线程B对数组元素的写入，写线程B和读线程C之间需要使用同步原语（lock或volatile）来确保内存可见性。

<br></br>



## 4. final引用不能从构造函数内“逸出”

&#12288;&#12288;前面提到过写final域的重排序规则可以确保：在引用变量为任意线程可见之前，该引用变量指向的对象的final域已经在构造函数中被正确初始化过了。要得到这个效果，还需要一个保证：在构造函数内部，不能让这个被构造对象的引用为其他线程可见，也就是对象引用不能在构造函数中“逸出”。

``` java
public class FinalReferenceEscapeExample {
	final int i;
	static FinalReferenceEscapeExample obj;

	public FinalReferenceEscapeExample () {
    	i = 1;                              //1写final域
    	obj = this;                         //2 this引用在此“逸出”
	}

	public static void writer() {
    	new FinalReferenceEscapeExample ();
	}

	public static void reader {
    	if (obj != null) {                    //3
        	int temp = obj.i;                 //4
    	}
	}
}
```

&#12288;&#12288;假设线程A执行`writer()`方法，线程B执行`reader()`方法。操作2使得对象还未完成构造前就为线程B可见。即使这里的操作2是构造函数的最后一步，且即使在程序中操作2排在操作1后面，执行`read()`方法的线程仍然可能无法看到final域被初始化后的值，因为这里的操作1和操作2之间可能被重排序。实际的执行时序可能如下图所示：

![](./Images/final4.png)

&#12288;&#12288;在构造函数返回前，被构造对象的引用不能为其他线程可见，因为此时的final域可能还没有被初始化。在构造函数返回后，任意线程都将保证能看到final域正确初始化之后的值。

<br></br>



## 5. final语义在处理器中的实现

&#12288;&#12288;写final域的重排序规则会要求译编器在final域的写之后，构造函数return之前，插入一个StoreStore障屏。读final域的重排序规则要求编译器在读final域的操作前面插入一个LoadLoad屏障。

&#12288;&#12288;由于x86处理器不会对写-写操作做重排序，所以在x86处理器中，写final域需要的StoreStore障屏会被省略掉。同样，由于x86处理器不会对存在间接依赖关系的操作做重排序，所以在x86处理器中，读final域需要的LoadLoad屏障也会被省略掉。也就是说在x86处理器中，final域的读/写不会插入任何内存屏障！
