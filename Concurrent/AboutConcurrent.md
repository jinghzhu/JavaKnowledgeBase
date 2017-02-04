# <center>About Concurrent</center>



## 1. 如何让一段程序并发执行，并最终汇总结果？
* CountDownLatch：允许一个或者多个线程等待前面的一个或多个线程完成，构造一个CountDownLatch时指定需要CountDown的点的数量，每完成一点就count down一下，当所有点都完成，latch.wait就解除阻塞。
* CyclicBarrier：可循环使用的Barrier，它的作用是让一组线程到达一个Barrier后阻塞，直到所有线程都到达 Barrier后才能继续执行。CountDownLatch的计数值只能使用一次，CyclicBarrier可以通过使用reset重置；还可以指定 到达栅栏后优先执行的任务。
fork/join框架，fork把大任务分解成多个小任务，然后汇总小任务的结果得到最终结果。使用一个双端队列，当线程空闲时从双端队列的另一端领取任务。

<br></br>



## 2. 为什么wait(), notify()和notifyAll()须在同步方法或同步块中被调用？
&#12288;&#12288;当一个线程需要调用对象的`wait()`时，这个线程必须拥有该对象的锁，接着它就会释放这个对象锁并进入等待状态直到其他线程调用这个对象上的`notify()`。

&#12288;&#12288;同样的，当一个线程需要调用对象的`notify()`时，它会释放这个对象的锁，以便其他在等待的线程就可以得到这个对象锁。由 于所有的这些方法都需要线程持有对象的锁，就只能通过同步来实现，所以他们只能在同步方法或者同步块中被调用。

<br></br>



## 3. 确定CPU密集型／IO密集型线程数
&#12288;&#12288;首先考虑可用的处理器核心数：`Runtime.getRuntime().availableProcessors()`。应用程序最小线程数等于可用的处理器核数。

&#12288;&#12288;如果所有任务是计算密集型的，则创建处理器可用核心数这么多个线程就可以 。创建更多的线程对于程序性能是不利的，因为多个线程间频繁进行上下文切换对于程序性能损耗较大。

&#12288;&#12288;如果任务都是IO密集型的，需要创建比处理器核心数大几倍的线程。当一个任务执行IO操作时，线程将被阻塞，处理器可以立即进行上下文切换以便处理其他就绪线程。如果只有处理器核心数那么多个线程，即使有待执行的任务也无法调度处理。

&#12288;&#12288;因此，线程数与任务处于阻塞状态的时间比例相关。任务有50%时间处于阻塞状态，那程序所需线程数是处理器核心数的两倍。计算程序所需的线程数公式如下：
<i>线程数=CPU可用核心数/（1 - 阻塞系数），阻塞系数在0到1内（CPU密集型阻塞系数为0，IO密集型程阻塞系数接近1）</i>

You really can't improve on having one thread reading the file sequentially. With one thread, you do one seek and then one long sequential read; with multiple threads you're going to have the threads causing multiple seeks as each gains control of the disk head.

This is a way to parallelize the line processing while still using serial I/O to read the lines. It uses a BlockingQueue (Reading big file from disk like producer/consumer pattern) to communicate between threads; the FileTask adds lines to the queue, and the CPUTask reads them and processes them. You're using put(E e) to add strings to the queue, so if the queue is full, the FileTask blocks until space frees up; likewise you're using take() to remove items from the queue, so the CPUTask will block until an item is available.

```java
public class ReadingFile {
    public static void main(String[] args) {
        final int threadCount = 10;
        BlockingQueue<String> queue = new ArrayBlockingQueue<>(200);

        // create thread pool with given size
        ExecutorService service = Executors.newFixedThreadPool(threadCount);

        for (int i = 0; i < (threadCount - 1); i++) {
            service.submit(new CPUTask(queue));
        }

        // Wait til FileTask completes
        service.submit(new FileTask(queue)).get();

        service.shutdownNow();  // interrupt CPUTasks

        // Wait til CPUTasks terminate
        service.awaitTermination(365, TimeUnit.DAYS);
    }
}

class FileTask implements Runnable {
    private final BlockingQueue<String> queue;

    public FileTask(BlockingQueue<String> queue) {
        this.queue = queue;
    }

    @Override
    public void run() {
        BufferedReader br = null;
        try {
            br = new BufferedReader(new FileReader("D:/abc.txt"));
            String line;
            while ((line = br.readLine()) != null) {
                // block if the queue is full
                queue.put(line);
            }
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                br.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }
}

class CPUTask implements Runnable {
    private final BlockingQueue<String> queue;

    public CPUTask(BlockingQueue<String> queue) {
        this.queue = queue;
    }

    @Override
    public void run() {
        String line;
        while(true) {
            try {
                // block if the queue is empty
                line = queue.take(); 
                // do things with line
            } catch (InterruptedException ex) {
                break; // FileTask has completed
            }
        }
        // poll() returns null if the queue is empty
        while((line = queue.poll()) != null) {
            // do things with line;
        }
    }
}
```

<br></br>



## 4. daemon线程
A daemon thread runs in background and doesn’t prevent JVM from terminating. When there are no user threads running, JVM shutdown the program and quits.

&#12288;&#12288;必须在Thread启动前调用`setDaemon()`将线程设置为Daemon线程：
* Daemon线程创建的线程也是Daemon线程
* Daemon不应该访问数据库、文件等资源，因为它随时有可能被中断（后台进程在不执行finally语句前就可以中断其`run()`）

<br></br>



## 5. 针对外星方法的保护性锁
&#12288;&#12288;例: 构造一个类从一个URL进行下载，并用`ProgressListeners`监听下载的进度：

``` java
class Downloader extends Thread {
    private InputStream in;
    private OutputStream out;
    private ArrayList<ProgressListener> listeners;

    public Downloader(URL url, String outputFilename) throws IOException {
        in = url.openConnection().getInputStream();
        out = new FileOutputStream(outputFilename);
        listeners = new ArrayList<ProgressListener>();
    }

    public synchronized void addListener(ProgressListener listeners) {
        listeners.add(listener);
    }

    public synchronized void removeListener(ProgressListener listener) {
        listeners.remove(listener);
    }

    private synchronized void updateProgress(int n) {
        for (ProgressListener listener: listeners)
            listener.onProgress(n);   // 这里调用了一个外星方法
    }

    public void run() {
        int n = 0, total = 0;
        byte[] buffer = new byte[1024];

        try {
            while ((n = in.read(buffer)) != -1) {
                out.write(buffer, 0, n);
                total += n;
                updateProgress(total);
            }
            out.flush();
        } catach (IOException e) {}
    }
}
```

* 问题：调用了`onProgress()`这个外星方法，可能引入其他锁导致死锁。
* 解决方案：采用保护性复制（defensive copy），即不对原始对象进行操作，而是对克隆出来的对象进行操作。
* 注意：保护性复制是个好方法。前提是进行只读的操作，不做修改。

``` java
private void updatePrgress(int n) {
    ArrayList<ProgressListener> listernersCopy;
    synchronized(this) {
        listernersCopy = (ArrayList<ProgressListener>)listeners.clone();
    }
    for (ProgressListener listener : listernersCopy)
        listener.onProgress(n);
}
```

<br></br>



## 6. What will happen if don’t override Thread.run() method?
Thread class `run()` method code is as shown below:

``` java
public void run() {
    if (target != null) {
        target.run();
    }
}
```

Above target is set in the `init()` method of Thread class and if we create an instance of Thread class as new `TestThread()`, it’s set to `null`. So nothing will happen.

<br></br>



## 7. 线程类的构造方法、静态块是被哪个线程调用的
&#12288;&#12288;线程类的构造方法、静态块是被new这个线程类所在的线程所调用的，而`run()`里面的代码才是被线程自身所调用的。

&#12288;&#12288;举个例子，假设`Thread2`中new了`Thread1`，`main()`中new了`Thread2`，那么：
* `Thread2`的构造方法、静态块是`main`线程调用的，`Thread2`的`run()`是`Thread2`自己调用的。
* `Thread1`的构造方法、静态块是`Thread2`调用的，`Thread1`的`run()`是`Thread1`自己调用的。

<br></br>



## 8. Java如何获取线程dump文件
&#12288;&#12288;线程dump就是线程堆栈，获取到线程堆栈有两步：
1. 获取到线程的`pid`, 在Linux环境下还可以使用ps -ef | grep java
2. 打印线程堆栈, 在Linux环境下还可以使用kill -3 pid

&#12288;&#12288;Thread类提供了`getStackTrace()`用于获取线程堆栈。这是一个实例方法，因此此方法是和具体线程实例绑定的，每次获取获取到的是具体某个线程当前运行的堆栈。

<br></br>














