# <center>Reference</center>

<br></br>



## StrongReference强引用
----
强引用是指在程序代码之中普遍存在的，比如：

```java
Object obj = new Object();
String str = "hello";
```

只要某个对象有强引用与之关联，JVM必定不会回收这个对象，即使在内存不足的情况下，JVM宁愿抛出OutOfMemory也不会回收这种对象。如果想中断强引用和某个对象的关联，可以显示地将引用赋值为`null`， 比如Vector类的`clear()`方法：

```java
public synchronized E remove(int index) {
    modCount++;
    if (index >= elementCount)
        throw new ArrayIndexOutOfBoundsExpection(index);
    Object oldValue = elementData(index);
    int numMoved = elementCount - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index + 1, elementData, index, numMoved);
    elementData[--elementCount] = null; // let gc do its work

    return (E)oldValue;
}
```

<br></br>



## SoftReference软引用
----
软引用是一些有用但并不是必需的对象，用`java.lang.ref.SoftReference`类表示。软引用对象，只有内存不足时JVM才会回收。适合用来实现缓存。

软引用可以和一个引用队列（ReferenceQueue）联合使用，如果软引用所引用的对象被JVM回收，这个软引用就会被加入到与之关联的引用队列中：

```java
import java.lang.ref.SoftReference;

public class Main {
    public static void main(String[] args) {
        SoftReference<String> ar = new SoftReference<String>(new String("hello"));
        System.out.println(sr.get());
    }
}
```

![Soft Reference](./Images/soft_reference.png)

<br></br>



## WeakReference弱引用
----
弱引用也是描述非必需对象。GC时，无论内存是否充足，都会回收弱引用对象。用`java.lang.ref.WeakReference`类表示：

```java
import java.lang.ref.WeakReference;

public class Main {
    public static void main(String[] args) {
        WeakReference<String> sr = new WeakReference<String>(new String("hello"));
        System.out.println(sr.get());
        System.gc(); 
        System.out.println(sr.get()); // null
    }
}
```

<br></br>



## PhantomReference虚引用
----
虚引用不影响对象生命周期。用`java.lang.ref.PhantomReference`类表示。如果一个对象与虚引用关联，则跟没有引用与之关联一样，任何时候都可能被回收。

虚引用须和引用队列关联使用，当GC准备回收对象时，如果发现有虚引用，会把虚引用加入到与之关联的引用队列中。程序可通过判断引用队列中是否已经加入了虚引用，来了解被引用的对象是否将要被垃圾回收。如果程序发现虚引用已被加入到引用队列，就可在所引用的对象的内存被回收之前采取行动。

```java
import java.lang.ref.PhantomReference;
import java.lang.ref.ReferenceQueue;

public class Main {
    public static void main(String[] args) {
        ReferenceQueue<String> queue = new ReferenceQueue<String>();
        PhantomReference<String> pr = new PhantomReference<String>(new String("hello"), queue);
        System.out.println(pr.get());
    }
}
```

<br></br>



## WeakHashMap
----
`java.util.WeakHashMap` uses weak references as the key. Therefore, when a particular key is not in use anymore, and it is garbage collected, the corresponding entry will disappear from the map. And the magic relies on ReferenceQueue mechanism to identify when a particular weak reference is to be garbage collected. This is useful when you want to build a cache based on weak references. 

<br></br>



## 各种引用对比
----

![Reference Conclusion](./Images/reference_conclusion.png)

<br></br>



## ReferenceQueue
----
当软可及对象被回收后，虽然这个SoftReference对象的`get()`方法返回`null`，但这个SoftReference对象已经不再具有存在的价值，需要一个适当的清除机制。`java.lang.ref`包里提供ReferenceQueue。如果在创建SoftReference对象时，使用ReferenceQueue作为参数提供给SoftReference的构造方法，如:

```java
ReferenceQueue queue = new ReferenceQueue();
SoftReference ref = new SoftReference(aMyObject, queue);
```

当这个SoftReference所引用的`aMyOhject`被回收时，`ref`所强引用的SoftReference对象被列入`ReferenceQueue`。 

任何时候都可调用ReferenceQueue的`poll()`方法检查是否有它所关心的非强可及对象被回收。如果队列为空，将返回`null`，否则该方法返回队列中前面的一个Reference对象。利用这个方法，可以检查哪个SoftReference所软引用的对象已经被回收。于是可以把失去软引用的对象清除掉。

<br></br>



## 解决OOM
----
JVM的GC线程对软可达对象和其他对象区别对待：软可及对象的清理是由垃圾收集线程根据其特定算法按照内存需求决定的。即垃圾收集线程会在抛出OOM前回收软可及对象，而且虚拟机会尽可能优先回收长时间闲置不用的软可达对象，对那些刚构建或刚使用过的软可达对象会被虚拟机尽可能保留。在回收这些对象前，可通过`MyObject anotherRef =(MyObject) aSoftRef.get()`重新获得对该实例的强引用。而回收之后，调用`get()`方法只能得到`null`。
        
假如需读取大量本地图片，如果每次都从硬盘读取，会严重影响性能。如果全部加载到内存，又可能内存溢出。此时使用软引用可解决这个问题。比如用HashMap保存图片路径和相应图片对象关联的软引用之间的映射关系，内存不足时，JVM会回收缓存图片对象所占用的空间 。

```java
private Map<String, SoftReference<Bitmap>> imageCache = new HashMap<String, SoftReference<Bitmap>>();

public void addBitmapToCache(String path) {
    Bitmap bitmap = BitmapFactory.decodeFile(path); // StrongReference
    SoftReference<Bitmap> softBitmap = new SoftReference<Bitmap>(bitmap); // SoftReference
    imageCache.put(path, softBitmap); // cache it
}

public Bitmap getBitmapByPath(String path) {
    SoftReference<Bitmap> softBitmap = imageCache.get(path); // ger SoftReference object Bitmap from cache
    
    if (softBitmap == null)
        return null;
    Bitmap bitmap = softBitmap.get(); // get object Bitmap, if Bitmap is reclaimed, it will be null

    return bitmap;
}
```

<br></br>



## 对象可达性判断
----

![Reference Object Path](./Images/reference_path.png)

很多时候，一个对象并不是从根集直接引用，而是一个对象被其他对象引用，甚至同时被几个对象所引用，从而构成一个以根集为顶的树形结构。由此带来了一个问题，那就是某个对象的可及性如何判断:
* **单条引用路径**：在这条路径中，**最弱**的一个引用决定对象的可及性。
* **多条引用路径**：几条路径中，**最强**的一条的引用决定对象的可及性。

<br></br>
