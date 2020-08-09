<h1>Java 多线程</h1>

<span id="目录"><b>目录</b></span>

[toc]

## 创建线程的三类方法

参考CSDN博客[多线程(一)：创建线程和线程的常用方法](https://blog.csdn.net/vbirdbest/article/details/81282163)

- Thread类是实现Runnable
- Callable封装成FutureTask，FutureTask实现RunnableFuture，RunnableFuture继承Runnable，所以Callable也算是一种Runnable
- 所以三种实现方式本质上都是Runnable实现

### Thread

通过继承Thread类来实现创建线程

实际功能通过重载 `run()` 来实现

调用 `start()` 函数即可等待CPU调度线程，然后执行 `run()` ，具体的后面会细讲

```java
public class TestThread{
    public static void main(String[] args) {
        new MyThread("Test").start();
    }
}

class MyThread extends Thread {

    // 传参通过重载构造函数实现
    private String sth = null;

    MyThread(String sth){
        this.sth = sth;
    }

    @Override
    public void run(){
        System.out.println("MyThread - " + sth);
    }
}
```

#### Supplier创建对象

Java 8 后有一种新的创建对象方法：Supplier接口

Supplier源码

```java
@FunctionalInterface
public interface Supplier<T> {
    /**
     * Gets a result.
     *
     * @return a result
     */
    T get();
}
```

即每次调用 `get()` 就返回一个对象

```java
public class TestThread{
    public static void main(String[] args) {
        Supplier<MyThread> sp = MyThread::new;
        MyThread t1 = sp.get();
        t1.start();
        //先执行完t1线程
        try{
            t1.join();
        }catch(InterruptedException e){
            e.printStackTrace();
        }
        sp.get().start();
    }
}

class MyThread extends Thread {

    @Override
    public void run(){
        System.out.println("MyThread");
    }
}
```

可以看到就是代替了 `MyThread t = new MyThread()` 语句

当然也可以通过重写Supplier的 `get()` 方法实现更多操作，这边就不再举例



***

### Runnable

通过Runnable接口实现创建线程

原理是Thread类本身就是连接Runnable接口实现的 `run()`

Thread类源码

```java
package java.lang;
public class Thread implements Runnable {
	// 构造方法
	public Thread(Runnable target);
	public Thread(Runnable target, String name);
	
	public synchronized void start();
}
```

Runnable接口源码

```java
package java.lang;

@FunctionalInterface
public interface Runnable {
    pubic abstract void run();
}
```

通过Runnable接口创建线程一般写法

```java
public class TestThread {
    public static void main(String[] args) {
    	 // 将Runnable实现类作为Thread的构造参数传递到Thread类中，然后启动Thread类
        MyRunnable runnable = new MyRunnable();
        new Thread(runnable).start();
    }
}

class MyRunnable implements Runnable {
    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName() + "\t" + Thread.currentThread().getId());
    }
}
```

使用匿名内部类写法重写接口

```java
public class TestThread {
    public static void main(String[] args) {
		//通过匿名内部类重写接口
        new Thread(new Runnable() {
        	@Override
            public void run(){
                System.out.println(Thread.currentThread().getName() + "\t" +Thread.currentThread().getId());
            }
        }).start();
        //匿名内部类的尾部代码方式
        new Thread(){
        	@Override
            public void run(){
                System.out.println(Thread.currentThread().getName() + "\t" +Thread.currentThread().getId());
            }
        }.start();
    }
}
```

Java 8 后出现的Lambda表达式使重写接口方式更简便

```java
public class TestThread {
    public static void main(String[] args) {
		//通过lambda表达式重写接口
        new Thread(() -> {
            System.out.println(Thread.currentThread().getName() + "\t" + Thread.currentThread().getId());
        }).start();
    }
}
```



***

### Callable

采用重写callable接口中的 `call()` 方法，然后包装入FutureTask然后包装成Thread的方式实现线程

与上面不同的是callable是 **有返回值** 的线程，能够取消线程，抛出异常，还能判断线程是否执行完毕

Callable源码

```java
@FunctionalInterface
public interface Callable<V> {
    V call() throws Exception;
}
```

FutureTask源码

```java
public class FutureTask<V> implements RunnableFuture<V> {
	// 构造函数
	public FutureTask(Callable<V> callable);
	
	// 取消线程
	public boolean cancel(boolean mayInterruptIfRunning);
	// 判断线程
	public boolean isDone();
	// 获取线程执行结果
	public V get() throws InterruptedException, ExecutionException;
}
```

FutureTask使用的接口RunnableFuture源码

```java
public interface RunnableFuture<V> extends Runnable, Future<V> {
    void run();
}
```

实例

```java
public class TestThread {
    public static void main (String[] args) throws Exception {
    	 // 将Callable包装成FutureTask，FutureTask也是一种Runnable
        MyCallable callable = new MyCallable();
        FutureTask<Integer> futureTask = new FutureTask<>(callable);
        new Thread(futureTask).start();

        // get方法会阻塞调用的线程, 类似Thread的join
        Integer sum = futureTask.get();
        System.out.println(Thread.currentThread().getName() + Thread.currentThread().getId() + "=" + sum);
    }
}

//实现计算0到100000的累加
class MyCallable implements Callable<Integer> {
    
    @Override
    public Integer call() throws Exception {
        System.out.println(Thread.currentThread().getName() + "\t" + Thread.currentThread().getId() + "\t" + new Date() + " \tstarting...");

        int sum = 0;
        for (int i = 0; i <= 100000; i++) {
            sum += i;
        }
        Thread.sleep(5000);

        System.out.println(Thread.currentThread().getName() + "\t" + Thread.currentThread().getId() + "\t" + new Date() + " \tover...");
        return sum;
    }
}
```



***



## 多线程常用操作

### start() 和 run()

这里把两者放在一起作区分

- `start()` 是启动一个线程，使得CPU调度分配内存来处理线程

- `run()` 只是一个普通的方法，单纯的函数调用

这边给出主函数中调用子线程的 `start()` 和 `run()` 的区别

```java
public static void main(String[] args) throws Exception {
   //使用start
   new Thread(()-> {
       for (int i = 0; i < 5; i++) {
           System.out.println(Thread.currentThread().getName() + " " + i);
           try { Thread.sleep(200); } catch (InterruptedException e) { }
       }
   }, "Thread-A").start();

   new Thread(()-> {
       for (int j = 0; j < 5; j++) {
           System.out.println(Thread.currentThread().getName() + " " + j);
           try { Thread.sleep(200); } catch (InterruptedException e) { }
       }
   }, "Thread-B").start();
   
   //保证先执行完上面的
   Thread.sleep(2000);
   
   //不使用start，直接调用run
   new Thread(()-> {
       for (int i = 0; i < 5; i++) {
           System.out.println(Thread.currentThread().getName() + " " + i);
           try { Thread.sleep(200); } catch (InterruptedException e) { }
       }
   }, "Thread-A").run();

   new Thread(()-> {
       for (int j = 0; j < 5; j++) {
           System.out.println(Thread.currentThread().getName() + " " + j);
           try { Thread.sleep(200); } catch (InterruptedException e) { }
       }
   }, "Thread-B").run();
}
```

运行结果

<img src="C:\Users\lap\AppData\Roaming\Typora\typora-user-images\image-20200804162524439.png" alt="image-20200804162524439" style="zoom:50%; float:left" />

可以看到后面执行run方法是顺序执行，且线程名是main即主函数，也就是实质是在主线程中顺序调用执行run中的操作

要启动一个线程来运行线程体中的代码 _( run()方法 )_ 还是通过start()方法来实现，调用run()方法就是一种顺序编程不是并发编程

***

### sleep()

- `sleep(long millis)` :  睡眠指定时间，程序暂停运行，睡眠期间会让出CPU的执行权，去执行其它线程，同时CPU也会监视睡眠的时间，一旦睡眠时间到就会立刻执行(因为睡眠过程中仍然保留着锁，有锁的情况下只要设定的睡眠时间到就能立刻执行)。

```java
public static void main(String[] args) throws InterruptedException {
    Object lock = new Object();
	//线程中设置同步锁，即所有线程同一时间只有一个可以访问lock
    new Thread(() -> {
        synchronized (lock) {
            for (int i = 0; i < 5; i++) {
                System.out.println(new Date() + "\t" + Thread.currentThread().getName() + "\t" + i);
                try { Thread.sleep(1000); } catch (InterruptedException e) { }
            }
        }
    }).start();
    
	//主线程睡眠，保证后面一个线程后执行
    Thread.sleep(1000);
	
    //由于新的线程中也设置了同步锁，新的线程需等待上面的线程释放锁才能执行对应操作
    //在上面一个线程的睡眠中新线程并未插入操作，可见sleep()并不会释放锁
    new Thread(() -> {
        synchronized (lock) {
            for (int i = 0; i < 5; i++) {
                System.out.println(new Date() + "\t" + Thread.currentThread().getName() + "\t" + i);
            }
        }
    }).start();
}
```



***

### wait() 

- `wait()` :  阻塞线程，直到受到notify，notifyAll或interrupt唤醒。该方法只能在同步方法中调用。如果当前线程不是锁的持有者，该方法会抛出一个IllegalMonitorStateException异常。`wait(long millis)` ：阻塞指定时间，类似 `sleep(long millis)`

**与sleep很大的不同是，当线程处于wait状态时会释放锁，而非一直阻塞**



***

### notify()

- `notify()` : 该方法只能在同步方法或同步块内部调用，**随机** 选择一个 **(注意：只会通知一个)** 在该对象上调用wait方法的线程，解除其阻塞状态

- `notifyAll()` : 解除所有wait对象的阻塞状态

注意 `wait()` ，`notify()`，`notifyAll()`都 **不能被子类重写**，因为其都是Object类的final native方法。



***

### interrupt()

- `interrupt()` : 调用interrupt()方法，会使得方法抛出InterruptedException异常，当抛出异常就中断了方法，从而让程序继续运行下去

interrupt实质上是改变线程中判断是否中止的boolean值，从而在函数执行过程中能够根据boolean值抛出InterruptExecption

而这个boolean值只有通过 `interrupted()` 才能重新设置为未打断状态

- `interrupt()` ：打断线程，将中断状态修改为true
- `interrupted()` : 不打断线程，获取线程的中断状态，并将中断状态设置为false
- `isInterrupted()` : 获取线程的中断状态，但不改变状态值

```java
public class InterruptTest {
    public static void main(String[] args) {
        MyRunnable mr = new MyRunnable();

        Thread t1 = new Thread(mr, "Thread 1");
        t1.start();

        //访问同步代码块，被阻塞
        Thread t2 = new Thread(mr, "Thread 2");
        t2.start();

        //保证先执行上面语句
        try{
            Thread.sleep(1000);
        }catch(InterruptedException e){
            e.printStackTrace();
        }

        //未打断，false
        boolean interrupted1 = t1.isInterrupted();
        System.out.println(new Date() + "\t" + "interrupted1 = " + interrupted1);

        t1.interrupt();

        //打断，true
        boolean interrupted2 = t1.isInterrupted();
        System.out.println(new Date() + "\t" + "interrupted2 = " + interrupted2);

        //未打断，false
        boolean interrupted3 = t1.isInterrupted();
        System.out.println(new Date() + "\t" + "interrupted3 = " + interrupted3);

        try{
            Thread.sleep(5000);
        }catch(InterruptedException e){
            e.printStackTrace();
        }

        t2.interrupt();
    }
}

class MyRunnable implements Runnable {
    public void run() {
        synchronized (this) {
            try {
                System.out.println(new Date() + "\t" + Thread.currentThread().getName() + " waiting...");
                wait();
            }catch(InterruptedException e) {
                System.out.println(new Date() + "\t" +  Thread.currentThread().getName() + "\t" + Thread.currentThread().isInterrupted());
            }
        }
    }
}
```

运行结果

<img src="C:\Users\lap\AppData\Roaming\Typora\typora-user-images\image-20200804224354287.png" alt="image-20200804224354287" style="zoom:50%; float:left" />

可以看到Thread 2在Thread 1等待期间访问同步代码块，可见wait是释放锁的

将wait换成sleep，即不释放锁的情况，则运行结果如下

<img src="C:\Users\lap\AppData\Roaming\Typora\typora-user-images\image-20200804224625108.png" alt="image-20200804224625108" style="zoom:50%; float:left" />



***

### join()

让主线程在这条语句处加入指定子线程，并阻塞主线程直到子线程执行完毕

也可以通过 `join(long millis)` _( 等待该线程终止的时间最长为 millis 毫秒  )_ 或者 `join(long millis, int nanos)` _( 等待该线程终止的时间最长为 millis 毫秒 + nanos 纳秒 )_ 来设定主线程的 **最大等待时间**

实例

```java
//多线程测试ip是否可用
public class TestIp {
    public static void main(String[] args) throws IOException{
        Thread[] threads = new Thread[256];
        //获取本机地址
        InetAddress host = null;
        try{
            host = InetAddress.getLocalHost();
        }catch(IOException e){
            e.printStackTrace();
        } 
        String ip = host.getHostAddress();
        System.out.println("本机地址：" + ip);
        ip = ip.substring(0, ip.lastIndexOf('.') + 1);
        //多线程测试
        for(int i = 0; i < 256; i++){
            Thread t = new MyIp(ip + i);
            t.start();
            //子类转父类
            threads[i] = t;
        }
        //join回收进程
        for(Thread tt : threads){
            try{
                tt.join();
            }catch(InterruptedException e){
                e.printStackTrace();
            }
        }
    }    
}

//第一类方法
class MyIp extends Thread{
    private String ip = null;

    public MyIp(String ip){
        this.ip = ip;
    }
	
    @Override
    public void run(){   
        String line = null;
        Process p = null;
        try{
            p = Runtime.getRuntime().exec("ping " + ip);
        }catch(IOException e){
            e.printStackTrace();
        }

        BufferedReader br = new BufferedReader(new InputStreamReader(p.getInputStream()));
        boolean connected = false;
        try{
            while((line = br.readLine()) != null){
                if(line.contains("TTL")){
                    connected = true;
                    break;
                }
            }
        }catch(IOException e){
            e.printStackTrace();
        }
        if(connected){
            System.out.println("<!---可连接 " + ip);
        }  
        System.out.println("已完成ip测试 " + ip);
    }
}
```

#### 使用CountDownLatch

参考博客[java多线程对CountDownLatch的使用实例](https://www.cnblogs.com/kaituorensheng/p/9043494.html)

- 介绍

CountDownLatch是一个同步辅助类，它允许一个或多个线程一直等待直到其他线程执行完毕才开始执行

用传入的 **计数参数** 初始化CountDownLatch，其含义是要被等待执行完的线程个数

每次调用CountDown()，计数减1

主线程执行到await()函数会阻塞，并等待线程的执行，直到计数为0

- 实现原理

计数器通过使用锁（共享锁、排它锁）实现

- 实例

```java
//多线程测试ip是否可用
public class TestIp {
    public static void main(String[] args) throws IOException{
        InetAddress host = InetAddress.getLocalHost();
        String ip = host.getHostAddress();
        System.out.println("本机地址：" + ip);
        String ipHead = ip.substring(0, ip.lastIndexOf('.')+1);
        //采用synchronized保证iplist线程安全
        List<String> iplist = Collections.synchronizedList(new ArrayList<String>());
        /*
         * CountDownLatch类用于同步辅助
         */
        final CountDownLatch cdt = new CountDownLatch(255);
        for(int i = 1; i < 256; i++){
            int j = i;
            new Thread(()->{
                String item = ipHead + j;
                String line = null;
                boolean connected = false;
                Process p = null;
                try{
                    //cmd>ping 192.168.3.x 
                    p = Runtime.getRuntime().exec("ping " + item);
                }catch(IOException e){
                    e.printStackTrace();
                }
                
                BufferedReader br = new BufferedReader(new InputStreamReader(p.getInputStream()));
                
                try{
                    while((line = br.readLine()) != null){
                        //当打印信息包含“TTL”时连接可用
                        if(line.contains("TTL")){
                            connected = true;
                            break;
                        }
                    }
                }catch(IOException e){
                    e.printStackTrace();
                }
                if(connected){
                    iplist.add(item);
                }
                System.out.println("已完成" + item);
                //计数减一
                cdt.countDown();
            }).start();
        }
        /*
         * 同步等待所有线程结束
         */
        try{
            cdt.await();
        }catch(Exception e){
            e.printStackTrace();
        }
        System.out.println("\n可连接ip:\n");
        for(int i = 0; i < iplist.size(); i++){
            System.out.println(iplist.get(i));
        }
    }
}
```

#### 异同点

如上两个实例可见，`join()` 可以做的 `CountDownLatch()` 也可以做到

但当我们需求主线程在子进程中执行到一定地方时进行某些操作，`join()` 就难以做到了

网上找到的例子，三名工人做一项工作，工作分两个步骤，工人3需要在工人1和工人2在执行完步骤一后再开始处理工作，这时候显然应该使用 `CountDownLatch()`

实例

```java
public class CountDownLatchTest {
    public static void main() throws InterruptedException {
        int COUNT = 2;
        final CountDownLatch countDownLatch = new CountDownLatch(COUNT);
        WorkerCount2 worker0 = new WorkerCount2("工人-1", (long)(Math.random() * 10000), countDownLatch);
        WorkerCount2 worker1 = new WorkerCount2("工人-2", (long)(Math.random() * 10000), countDownLatch);
        worker0.start();
        worker1.start();
        //等待前两个工人做完步骤一
        countDownLatch.await();
        System.out.println("准备工作就绪");

        WorkerCount2 worker2 = new WorkerCount2("工人-3", (long)(Math.random() * 10000), countDownLatch);
        worker2.start();
        Thread.sleep(10000);
    }
}

public class WorkerCount2 extends Thread {
    private String name;
    private long time;
    private CountDownLatch countDownLatch;

    public WorkerCount2(String name, long time, CountDownLatch countDownLatch) {
        this.name = name;
        this.time = time;
        this.countDownLatch = countDownLatch;
    }

    @Override
    public void run() {
        try {
            System.out.println(name + "开始阶段1工作");
            Thread.sleep(time);
            System.out.println(name + "阶段1完成, 耗时："+ time);
            
            //计数减1
            countDownLatch.countDown();

            System.out.println(name + "开始阶段2工作");
            Thread.sleep(time);
            System.out.println(name + "阶段2完成, 耗时："+ time);

        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```



***

### yield()

类似不定时的 `sleep()`，交出CPU执行时间，并让线程进入就绪状态，**不释放锁**，等待系统自动下次分配，获取CPU的执行时间。

- yield会把CPU的执行权交出去，所以可以用yield来控制线程的执行速度，当一个线程执行的比较快，此时想让它执行的稍微慢一些可以使用该方法，想让线程变慢可以使用sleep和wait,但是这两个方法都需要指定具体时间，而yield不需要指定具体时间，让CPU决定什么时候能再次被执行，当放弃到下次再次被执行的中间时间就是间歇等待的时间

可以通过 `setPriority(int)` 设置线程的优先级来控制线程获取CPU调度的优先级，默认线程优先级为5（normal），最大10，最小1，若超过这个范围，则会抛出IllegalArgumentException异常

虽说是优先级，但是也只是获取CPU调度的概率大小，不是一定的

线程的优先级具有继承性，A进程启动的子进程A-1具有的优先级与A是相同的

***

### setDaemon(boolean on)

- 设置守护线程，保证主线程会等待其全部执行结束再结束。
- 当主线程结束时，非守护线程也会随之消亡。

该方法必须在启动线程前调用。

***

## 线程组

可以对线程进行分组，从而统一管理同组内的线程，同一线程组内的线程共享数据资源，不同线程组相互独立

线程组与线程组间有父子关系，创建线程时若未指定线程组，则默认属于当前线程所在的线程组

在线程运行期间，不能修改其线程组，即一旦创建线程指定线程组，之后就不能更改

```java
public class TestThreadGroup {
    public static void main(String[] args) throws IOException{
        //默认属于main线程组
        Thread t1 = new Thread(() -> {}, "Thread-1");
        t1.start();

        //创建tg线程组
        ThreadGroup tg = new ThreadGroup("Group-1");

        //指定t2线程属于tg线程组
        Thread t2 = new Thread(tg, () -> {}, "Thread-2");
        t2.start();

        Thread t3 = new Thread(tg, () -> {
            ThreadGroup tg2 = new ThreadGroup("Group-2");

             //指定t4线程属于tg2线程组
            Thread t4 = new Thread(tg2, () -> {}, "Thread-4");
            t4.start();

        }, "Thread-3");
        t3.start();

        //中止tg线程组内的线程
        tg.interrupt();
    }
}
/*
system线程组 -> {

	其他线程
	
	其他线程组

	main线程组 -> {
	
        main线程
        
		Thread-1线程
		
		Group-1线程组 -> {
            Thread-2线程
            Thread-3线程

            Group-2线程组 -> {
                Thread-4线程
            }
        }
	}	
}
*/
```

**目前程序开发中已不推荐使用线程组**

- 线程组ThreadGroup对象中比较有用的方法是stop、resume、suspend等方法，由于这几个方法会导致线程的安全问题（主要是死锁问题），已经被官方废弃掉了，所以线程组本身的应用价值就大打折扣了。
- 线程组ThreadGroup不是线程安全的，这在使用过程中获取的信息并不全是及时有效的，这就降低了它的统计使用价值。



***

## 线程池

参考博客[Java线程池详解](https://www.jianshu.com/p/7726c70cdc40)

线程会占用内存资源，若创建线程数目过多，会极大地消耗系统资源，降低系统的稳定性

通过线程池管理线程，能够通过重复利用已创建的线程降低线程创建和销毁造成的消耗，从而降低资源消耗

Java通过Executors提供了四种线程池

- newCachedThreadPool : 可以创建一个线程数量不限的线程池，当负载较轻时可以使任务快速执行
- newFixedThreadPool : 可以创建一个线程数量固定的线程池，采用无界的阻塞队列，对线程数量有效控制，适用于负债较重的情况
- newScheduledThreadPool ：适用于延时任务或周期性任务
- newSingleThreadExecutor ： 可以创建一个线程数量固定为1的线程池，适用于顺序作业任务

任务类型

- CPU密集型 : 比如计算任务，对CPU负载较重，**一般设置线程数量与CPU核数相同**
- IO密集型 : 比如读写任务，对CPU负载较轻，大部分时间CPU在等待IO操作，负载较轻，**一般设置线程数量为CPU核数的两倍**

可以通过 `n = Runtime.getRuntime().availableProcessors()` 获得机器的CPU核数



***

### execute()和submit()

创建线程有两种方式，`execute()` 和 `submit()` 

- execute和submit都属于线程池的方法，execute只能提交Runnable类型的任务，而submit既能提交Runnable类型任务也能提交Callable类型任务。

- execute会直接抛出任务执行时的异常，submit会吃掉异常，可通过Future的get方法将任务执行时的异常重新抛出。

- execute所属顶层接口是Executor,submit所属顶层接口是ExecutorService，实现类ThreadPoolExecutor重写了execute方法,抽象类AbstractExecutorService重写了submit方法。

```java
void execute(Runnable command);

<T> Future<T> submit(Callable<T> task);
<T> Future<T> submit(Runnable task, T result);
Future<?> submit(Runnable task);
```

下面的代码中把execute替换成submit效果相同，但是这种情况下submit返回值为null

```java
public class TestThreadPool {
    public static void main(String[] args) throws IOException{
        //缓存线程池创建不需指定线程数，系统自动创建线程
        //当一个线程执行完任务后有60s的存活时间，过了这个时间线程自动消亡
        //线程数目最多为Integer.MAX_VALUE
        ExecutorService es1 = Executors.newCachedThreadPool();
        for(int i = 0; i < 10; i++){
            es1.execute(() -> {                
                System.out.println(Thread.currentThread().getName());
            });
        }
        //关闭线程池
        es1.shutdown();

        //固定线程池创建时指定最大线程数
        //这边采用Runtime.getRuntime().availableProcessors()获取CPU核数，创建对应数量的线程
        ExecutorService es2 = Executors.newFixedThreadPool(Runtime.getRuntime().availableProcessors());
        for(int i = 0; i < 10; i++){
            es2.execute(() -> {                
                System.out.println(Thread.currentThread().getName());
            });
        }
        //关闭线程池
        es2.shutdown();

        //单线程线程池
        ExecutorService es3 = Executors.newSingleThreadExecutor();
        for(int i = 0; i < 10; i++){
            es3.execute(() -> {                
                System.out.println(Thread.currentThread().getName());
            });
        }
        //关闭线程池
        es3.shutdown();
    } 
}
```

submit提交Callable类型的任务可以通过Future或FutureTask的方式提交

```java
public class TestThreadPoolSubmit {
    public static void main(String[] args) throws IOException{
        ExecutorService es = Executors.newCachedThreadPool();

        //使用Future提交Callable任务
        Future<Integer> f = es.submit(new MyCallable()); 
        Integer num = null;
        //submit的异常可以通过get抛出
        try{
            num = f.get();
        }catch(Exception e){
            e.printStackTrace();
        }
        System.out.println(num);

        //使用FutureTask提交Callable任务
        FutureTask<Integer> ft = new FutureTask<>(new MyCallable());
        es.submit(ft);
        try{
            num = ft.get();
        }catch(Exception e){
            e.printStackTrace();
        }
        System.out.println(num);

        //通过在submit中设置返回值，使用Future提交Runnable任务
        //不设置返回值的话默认返回null
        Future<Integer> f2 = es.submit(() -> {
            System.out.println(new Date() + " " + Thread.currentThread().getName());
        }, 777);
        try{
            num = f2.get();
        }catch(Exception e){
            e.printStackTrace();
        }
        //num = 777
        System.out.println(num);

        es.shutdown();
    }
}

class MyCallable implements Callable<Integer> {
    @Override
    public Integer call(){
        System.out.println(new Date() + " " + Thread.currentThread().getName());
        return 666;
    }
}
```

当采用newScheduledThreadPool线程池时往往采用 `schedule(command, delay, unit)` 来执行任务

其中三个参数分别对应任务，延迟时间，时间单位

```java
public class TestThreadPool {
    public static void main(String[] args) throws IOException{
        //计划线程池创建时也需指定线程数，算是固定线程池的升级版
        ScheduledExecutorService es = Executors.newScheduledThreadPool(Runtime.getRuntime().availableProcessors());
        for(int i = 0; i < 10; i++){
            //五秒后开始执行
            es.schedule(() -> {       
                System.out.println(new Date() + " " + Thread.currentThread().getName());
            }, 5, TimeUnit.SECONDS);
        }
        //关闭线程池
        es.shutdown();
    } 
}
```



***

### shutdown()

当调用了 shutdown方法时，线程池便进入关闭状态，此时意味着 ExecutorService 不再接受新的任务，但它还在执行已经提交了的任务，当素有已经提交了的任务执行完后，便到达终止状态。如果不调用 shutdown()方法，ExecutorService 会一直处在运行状态，不断接收新的任务，执行新的任务，服务器端一般不需要关闭它，保持一直运行即可。

```java
public class TestThreadPool {
    public static void main(String[] args) throws IOException{
        //缓存线程池创建不需指定线程数，系统自动创建线程
        //当一个线程执行完任务后有60s的存活时间，过了这个时间线程自动消亡
        //线程数目最多为Integer.MAX_VALUE
        ExecutorService es = Executors.newCachedThreadPool();
        for(int i = 0; i < 10; i++){
            es.execute(() -> {
                try{
                    Thread.sleep(1000);
                }catch(InterruptedException e){
                    e.printStackTrace();
                }
                
                System.out.println(Thread.currentThread().getName());
            });
        }
        //关闭线程池
        System.out.println("Now Shut Down !");
        es.shutdown();
    } 
}
```

运行结果

<img src="C:\Users\lap\AppData\Roaming\Typora\typora-user-images\image-20200805162927290.png" alt="image-20200805162927290" style="zoom:50%; float:left" />

可以看到，当执行shutdown语句后，线程池仍然会等待其中的任务全部执行完毕才关闭线程池



***

### 阿里对创建线程池的规范 : 禁用Executors !

参考博客[为什么不推荐通过Executors直接创建线程池](https://blog.csdn.net/u010321349/article/details/83927012?utm_medium=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.channel_param&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.channel_param)

- SingleThreadPool和FixedThreadPool的 **允许请求队列长度** 为 Integer.MAX_VALUE，可能会堆积大量请求，导致OOM
- CachedThreadPool和ScheduledThreadPool 的 **允许创建线程数量** 为Integer.MAX_VALUE，可能会创建大量线程，导致OOM

线程池创建参数

```java
ThreadPoolExecutor(int corePoolSize,
				   int maximumPoolSize,
                   long keepAliveTime,
                   TimeUnit unit,
                   BlockingQueue<Runnable> workQueue)
```

创建线程池的正确方法：

避免使用Executors创建线程池，主要是避免使用其中的默认实现，那么我们可以自己直接调用ThreadPoolExecutor的构造函数自己创建线程池。在创建的同时，给BlockQueue指定容量就可以了。

```java
ExecutorService executor = new ThreadPoolExecutor(10, 10, 
                                                  60L, TimeUnit.SECONDS,
                                                  new ArrayBlockingQueue(10));
```



***

[回到顶部](#目录)

