<h1>Java 内存模型JMM</h1>

<span id = "目录"> <b>目录</b></span>

[toc]

## JMM概述

参考[JMM和底层实现原理](https://www.jianshu.com/p/8a58d8335270)和[JMM概述](https://blog.csdn.net/zjcjava/article/details/78406330)

### 普遍内存模型

现代计算机的物理内存模型中，由于内存IO读写与CPU处理时间相差极大，为了提高运算效率，在处理器和内存之间，加入了一层速度较快的高速缓存（cache）。

为了处理高速缓存，计算机系统加入了缓存一致性协议来处理多处理器系统的不同处理器得到的高速缓存不一致的情况。常见的有MSI，MESI，MOSI等。



<img src="img\Java内存模型\1.png" alt="image-20200813104642253" style="zoom:50%;" />

然而这个模型也会带来一个问题，不同线程独立拥有各自的缓存，不同线程的缓存之间并不可见。这导致实际内存的读写顺序未必与处理器处理的读写顺序一致。

```java
void test(){
    int a = 0,
    	b = 0,
    	x = 0,
    	y = 0;
    Thread(() -> {
    	a = 1;
        x = b;
	}).start();
    
    Thread(() -> {
    	b = 2;
        y = a;
	}).start();
    
    Thread.sleep(1000);
    System.out.println("X = " + x + "\ty = " + y);
}
```

逻辑上应该最后得出x = 2, y =1。但在实际情况中，x和y可能会读取到a和b赋值前的初始值0。



***

### JMM

JMM定义了Java 虚拟机(JVM)在计算机内存(RAM)中的工作方式。JVM是整个计算机虚拟模型，所以JMM是隶属于JVM的。从抽象的角度来看，JMM定义了线程和主内存之间的抽象关系：线程之间的共享变量存储在主内存（Main Memory）中，每个线程都有一个私有的本地内存（Local Memory），本地内存中存储了该线程以读/写共享变量的副本。本地内存是JMM的一个抽象概念，并不真实存在。它涵盖了缓存、写缓冲区、寄存器以及其他的硬件和编译器优化。

在不同的硬件生产商和不同的操作系统下，内存的访问逻辑有一定的差异，结果就是当代码在某个系统环境下运行良好，并且线程安全，但是换了个系统就出现各种问题。JMM，就是为了屏蔽系统和硬件的差异，让一套代码在不同平台下能到达相同的访问结果。JMM从java 5开始的JSR-133发布后，已经成熟和完善起来。

JMM规定了内存主要划分为主内存和工作内存两种。此处的主内存和工作内存跟JVM内存划分（堆、栈、方法区）是在不同的层次上进行的，如果非要对应起来，主内存对应的是Java堆中的对象实例部分，工作内存对应的是栈中的部分区域，从更底层的来说，主内存对应的是硬件的物理内存，工作内存对应的是寄存器和高速缓存。

<img src="img\Java内存模型\2.png" alt="image-20200813142823330" style="zoom:50%;" />

- 所有原始类型(boolean,byte,short,char,int,long,float,double)的局部变量都直接保存在线程栈当中，对于它们的值各个线程之间都是独立的。对于原始类型的局部变量，一个线程可以传递一个副本给另一个线程，当它们之间是无法共享的。
- 堆区包含了Java应用创建的所有对象信息，不管对象是哪个线程创建的，其中的对象包括原始类型的封装类（如Byte、Integer、Long等等）。不管对象是属于一个成员变量还是方法中的局部变量，它都会被存储在堆区。
-  一个局部变量如果是原始类型，那么它会被完全存储到栈区。 一个局部变量也有可能是一个对象的引用，这种情况下，这个本地引用会被存储到栈中，但是对象本身仍然存储在堆区。
-  对于一个对象的成员方法，这些方法中包含局部变量，仍需要存储在栈区，即使它们所属的对象在堆区。 对于一个对象的成员变量，不管它是原始类型还是包装类型，都会被存储到堆区。Static类型的变量以及类本身相关信息都会随着类本身存储在堆区。

<img src="img\Java内存模型\3.png" alt="image-20200813144356090" style="zoom:50%;" />



***

## JMM的三个特征

Java内存模型是围绕着并发编程中有序性、可见性、原子性这三个特征来建立的

首先要提到并发编程中的几个问题：**脏读，幻读，不可重复读**

- 脏读

  读取已修改但未提交的数据，通常出现在回滚操作

  |      | Thread-A             | Thread-B            |
  | ---- | -------------------- | ------------------- |
  | 1    | start                |                     |
  | 2    |                      | start               |
  | 3    | 读取value = 10       |                     |
  | 4    | 修改value = 20       |                     |
  | 5    |                      | 读取value = 20      |
  | 6    | 撤销修改，value = 10 |                     |
  | 7    |                      | 修改value + 10 = 30 |
  | 8    |                      | commit              |
  | 9    | commit               |                     |

  在Thread-B中value就从原本理应的10+10变成了20+10，从而出现了脏读


- 幻读

  前后读取得到的数据条目不同

|      | Thread-A                       | Thread-B                   |
| ---- | ------------------------------ | -------------------------- |
| 1    | start                          |                            |
| 2    |                                | start                      |
| 3    | 读取value == 10的数据，有100条 |                            |
| 4    |                                | 添加100条value == 10的数据 |
| 5    |                                | commit                     |
| 6    | 读取value == 10的数据，有200条 |                            |

​	   在Thread-A中第一次读取和第二次读取得到的数据条目不相同，这种情况就叫幻读

- 不可重复读

  前后读取得到的数据内容不同

  |      | Thread-A       | Thread-B       |
  | ---- | -------------- | -------------- |
  | 1    | start          |                |
  | 2    |                | start          |
  | 3    | 读取value = 10 |                |
  | 4    |                | 读取value = 10 |
  | 5    |                | 修改value = 20 |
  | 6    |                | commit         |
  | 7    | 读取value = 20 |                |

  在Thread-A中value第一次读取和第二次读取得到的value值不同，这种情况就叫不可重复读

针对这三个例子，我们引出JMM中处理有序性，可见性和原子性的方法。



***

### 有序性

- 在本线程内观察，操作都是有序的，即 **线程内表现为串行语义（WithIn Thread As-if-Serial Semantics）**
- 如果在一个线程中观察另外一个线程，所有的操作都是无序的，即在多线程中存在 **重排序** 和 **工作内存和主内存之间的同步延迟** 现象

#### 重排序

在执行程序时，为了提高运行效率，编译器和处理器常常会对指令的执行顺序进行重新排序

<img src="img\Java内存模型\4.png" alt="image-20200814142530035" style="zoom:50%;" />

重排序分三个步骤

- 编译器优化重排序：编译器在不改变单线程程序语义的情况下，可重新排序语句的执行顺序

- 指令级并行重排序：现代处理器采用了指令级并行技术（Instruction-LevelParallelism，ILP）来将多条指令重叠执行。如果不存在**依赖性**，处理器可以改变语句对应机器指令的执行顺序。

  - 依赖性指前后的操作存在逻辑关系，比如对一个变量读后写，写后读，写后写（数据依赖性），或者是变量作为if-else判断条件（控制依赖性）等

  - 当代码中存在控制依赖性时，会影响指令序列执行的并行度。为此，编译器和处理器会采用猜测（Speculation）执行来克服控制相关性对并行度的影响。对于下面的代码，处理器可以提前读取并计算a*a，然后把计算结果临时保存到一个名为重排序缓冲（Reorder Buffer，ROB）的硬件缓存中。当条件判断为真时，就把该计算结果写入变量i中。

    ```java
    void main(String[] args){
        if(flag){
            i = a*a;
        }
    }
    ```

- 内存系统重排序：由于处理器使用cache和读写缓冲区，使加载Load和存储Store操作看上去是乱序在执行

#### 内存屏障

Java编译器在生成指令序列的适当位置会插入内存屏障指令来禁止特定类型的处理器重排序，从而让程序按我们预想的流程去执行。
 1、保证特定操作的执行顺序。
 2、影响某些数据（或则是某条指令的执行结果）的内存可见性。

| 屏障类型           | 指令示例                 | 说明                                                         |
| ------------------ | ------------------------ | ------------------------------------------------------------ |
| LoadLoadBarriers   | Load1;LoadLoad;Load2     | 确保Load1的数据装载先于Load2及其后所有装载指令的操作         |
| StoreStoreBarriers | Store1;StoreStore;Store2 | 确保Store1立即刷新数据到内存（使其对其他处理器可见）的操作先于Store2及其后存储指令的操作 |
| LoadStoreBarriers  | Load1;LoadStore;Store2   | 确保Load1的数据装载先于Store2及其后存储指令刷新数据到内存的操作 |
| StoreLoadBarriers  | Store1;StoreLoad;Load2   | 确保Store1的刷新数据到内存操作先于Load2及其后装载指令的操作。它会使在屏障前的所有内存访问指令执行结束后再执行屏障后的。 |

StoreLoadBarriers同时具备其他三个屏障的效果，因此也称之为全能屏障，是目前大多数处理器所支持的，但是相对其他屏障，该屏障的开销相对昂贵，以为当前处理器要将其缓冲区的数据全部刷新到内存（buffer fully flush)。在x86架构的处理器的指令集中，lock指令可以触发StoreLoadBarriers。

#### 临界区

临界区指同一时刻只能有一个任务访问的代码区，例如被synchronized所修饰的函数。

<img src="img\Java内存模型\5.png" alt="image-20200814160310826" style="zoom:50%;" />

临界区内的代码可以重排序（但JMM不允许临界区内的代码“逸出”到临界区之外，那样会破坏监视器的语义）。JMM会在退出临界区和进入临界区这两个关键时间点做一些特别处理，虽然线程A在临界区内做了重排序，但由于监视器互斥执行的特性，这里的线程B根本无法“观察”到线程A在临界区内的重排序。这种重排序既提高了执行效率，又没有改变程序的执行结果。

#### as-if-serial规则

as-if-serial规则的意思是，不管怎么重排序（编译器和处理器为了提高并行度），（单线程）程序的执行结果不能被改变。编译器，runtime 和处理器都必须遵守as-if-serial语义。

为了遵守as-if-serial语义，编译器和处理器不会对**存在数据依赖关系**的操作做重排序，因为这种重排序会改变执行结果。（强调一下，这里所说的数据依赖性仅针对单个处理器中执行的指令序列和单个线程中执行的操作，不同处理器之间和不同线程之间的数据依赖性不被编译器和处理器考虑。）但是，如果操作之间不存在数据依赖关系，这些操作依然可能被编译器和处理器重排序。

#### happenes-before规则

在JMM中，如果一个操作执行的结果需要对另一个操作可见，那么这两个操作之间必须要存在happens-before关系 。

两个操作之间具有happens-before关系，并不意味着前一个操作必须要在后一个操作之前执行！happens-before仅仅要求前一个操作（执行的结果）对后一个操作可见，且前一个操作按顺序排在第二个操作之前。

- 对程序员来说：如果一个操作happens-before另一个操作，那么第一个操作的执行结果将对第二个操作可见，而且第一个操作的执行顺序排在第二个操作之前。
- 对编译器和处理器来说：两个操作之间存在happens-before关系，并不意味着Java平台的具体实现必须要按照happens-before关系指定的顺序来执行。如果重排序之后的执行结果，与按happens-before关系来执行的结果一致，那么这种重排序是允许的

我们程序员对操作是否被重排序并不关心，我们只关心程序执行时的语义正确。

**HB规则具体包含**

- 程序顺序规则：一个线程中的每个操作，happens-before于该线程中的任意后续操作。
- 监视器锁规则：对一个锁的解锁，happens-before于随后对这个锁的加锁。
- volatile变量规则：对一个volatile域的写，happens-before于任意后续对这个volatile域的读。
- 传递性：如果A happens-before B，且B happens-before C，那么A happens-before C。
- start()规则：如果线程A执行操作ThreadB.start()（启动线程B），那么A线程的ThreadB.start()操作happens-before于线程B中的任意操作。
- join()规则：如果线程A执行操作ThreadB.join()并成功返回，那么线程B中的任意操作happens-before于线程A从ThreadB.join()操作成功返回。
- interrupt()规则：对线程interrupt方法的调用happens-before于被中断线程的代码检测到中断事件的发生。
- finalize()规则：一个对象的初始化完成先行发生于他的finalize()方法的开始。

**happens-before关系本质上和as-if-serial语义是一回事，都是保证操作执行的有序性。as-if-serial语义保证单线程内程序的执行结果不被改变，happens-before关系保证正确同步的多线程程序的执行结果不被改变。**



***

### 可见性

- 一个线程对共享变量做了修改之后，其他的线程立即能够看到（感知到）该变量的这种修改（变化）。

#### volatile关键字

volatile作为并发编程中重要的修饰关键字，其作用为**保证被修饰的变量在多线程中的可见性并禁止对相关指令进行重排序**，即若值发生变更，除了当前线程，其他线程也立即可见这一变更，也就是说这可以避免脏读。

**volatile变量的特性**

- 可见性
  - 对一个volatile变量的读，总是能看到（任意线程）对这个volatile变量最后的写入。

- 原子性
  - 对任意单个volatile变量的读/写具有原子性，**但类似于volatile++这种复合操作不具有原子性**。

在上面的脏读例子中，Thead-A在回滚value=10时，Thread-B也能发现这一变更，从而使结果value=30。

**volatile的内存语义**

- 读
  - 当写一个volatile变量时，JMM会把该线程对应的本地内存中的共享变量值刷新到主内存。
- 写
  - 当读一个volatile变量时，JMM会把该线程对应的本地内存置为无效。线程接下来将从主内存中读取共享变量。

**volatile重排序规则**

- volatile读之前，所有volatile读写操作都已完成。
- volatile读之后，所有变量读写操作都不会重排序到其前面。

- volatile写之前，所有变量的读写操作都已完成。
- volatile写之后，volatile变量读写操作都不会重排序到其前面。

| 操作1      | 操作2    |            |            |
| ---------- | -------- | ---------- | ---------- |
|            | 普通读写 | volatile读 | volatile写 |
| 普通读写   |          |            | X          |
| volatile读 | X        | X          | X          |
| volatile写 |          | X          | X          |

**volatile内存语义的实现——JMM对volatile的内存屏障插入策略**

-  在每个volatile读操作的后面插入一个LoadLoad屏障，防止volatile的读操作与后面普通读操作重排序。
-  在每个volatile读操作的后面插入一个LoadStore屏障，防止volatile的读操作与后面普通写操作重排序。
-  在每个volatile写操作的前面插入一个StoreStore屏障，保证在volatile的写操作前所有操作的写都已经写入内存。
- 在每个volatile写操作的后面插入一个StoreLoad屏障，防止volatile的写操作与后面读写操作重排序。



***

### 原子性

- 一个操作不能被打断，要么全部执行完毕，要么不执行。在这点上有点类似于事务操作，要么全部执行成功，要么回退到执行该操作之前的状态。
- 基本类型数据的访问大都是原子操作，long 和double类型的变量是64位，但是在32位JVM中，32位的JVM会将64位数据的读写操作分为2次32位的读写操作来进行，这就导致了long、double类型的变量在32位虚拟机中是非原子操作，数据有可能会被破坏，也就意味着多个线程在并发访问的时候是线程非安全的。

#### CAS

CAS全称Compare And Swap，即比较替换。

CAS机制当中使用了3个基本操作数：内存地址V，旧的预期值A，要修改的新值B。

更新一个变量的时候，只有当变量的预期值A和内存地址V当中的实际值相同时，才会将内存地址V对应的值修改为B。

举个例子，线程1的CAS操作数为V=0，A=0，B=1

<img src="img\Java内存模型\6.png" alt="image-20200816101438052" style="zoom:50%;" />

但在开始比较之前，另一个V=0，A=0，B=2的线程2先完成了CAS操作，将内存地址V中的值改成了2

<img src="img\Java内存模型\7.png" alt="image-20200816102552131" style="zoom:50%;" />

这时线程1中V已经修改成了2，而A和B并没有变化，比较A和V的值不相等，操作失败

<img src="img\Java内存模型\8.png" alt="image-20200816102916596" style="zoom:50%;" />

当操作失败时，CAS机制中有一个处理方法，**自旋**，即重新读取V，A和得出要修改的B，再一次进行CAS操作

<img src="img\Java内存模型\9.png" alt="image-20200816110011781" style="zoom:50%;" />

此时线程1的V更新为2，A更新为2，B更新为4，同时恰好并没有线程修改V值，于是线程1通过CAS操作成功将V修改成4

<img src="img\Java内存模型\10.png" alt="image-20200816110200445" style="zoom:50%;" />



这种CAS机制存在三个问题

- ABA问题

  当我们判断V与A相等时，并不能说明V中值没有被修改。有可能是从值1修改成了值2，然后又修改回值1。这就是ABA问题。

  我们通过 **版本号** 来解决这个问题，也就是给变量绑定一个版本号变量S，每次修改时都给S加1，那么在compare的时候只需再比较版本号就可以知道变量是否被修改了。A-B-A转化为1A-2B-3A。

- CPU消耗问题

  CAS本质上是一种**乐观锁**，即默认线程之间的冲突较少，此时使用CAS对CPU的消耗就会少一些。由于自旋操作会导致线程占用CPU，当冲突较多时，会有很多的自旋操作，使得CPU负载较重

- 原子性问题

  CAS只能保证单一变量的原子性，当涉及同时修改多个变量则只能通过synchronized等锁的形式来保证原子性

  还有一个方法，就是把多个变量合并到一个变量中，比如i=1，j=2合并为ij=12，这样依旧可以保证这几个变量的原子性



#### Atomic包

由于类似i++和++i的操作是非原子性的，java.util.concurrent包中提供了atomic包，对一些初始数据类型的非原子操作进行了原子性处理，比如int类型的AtomicInteger，boolean类型的AtomicBoolean，long类型的AtomicLong，引用类型的AtomicReference，数组类型的AtomicIntegerArray等。

下面举出几种典型类型的例子

- AtomicInteger
  - 提供getAndSet(int)方法实现获取并返回旧值同时设置新值操作。
  - 提供compareAndSet(int)方法实现CAS操作。
  - 提供getAndAdd(int)方法实现先获得再自加一个数的操作，addAndGet(int)方法实现先加再获得的操作。
  - 提供getAndIncrement()方法实现i++操作，而incrementAndGet()方法实现++i操作，而getAndDecrement()和decrementAndGet()方法实现i--和--i操作。本质是在方法中调用了getAndIncrement(1)等方法。

- AtomicIntegerArray

  - 与AtomicInteger的操作类似，只是函数多了个下标参数，如getAndAdd(int i, int newValue)对下标为i的元素先返回值再加上newValue
  - 提供length()方法获得数组的长度

- AtomicReference

  - 相当于C中的取引用的原子性处理，构造时需声明引用类型，如String类型则AtomicReference\<String>

- AtomicStampedReference

  - **解决了ABA问题的CAS操作，加入了stamp版本号**
  - 提供set(V newValue, int newStamp)设置值和版本号
  - 提供compareAndSet(V expectedValue, V newValue, int expectedStamp, int newStamp)通过CAS操作设置值和版本号

- AtomicIntegerFieldUpdater

  - 可实现普通int类型的原子性操作

  - 实现方式如下

    - 首先构造一个对象，包含我们需要原子性处理的int变量，**同时声明该变量为volatile类型**，且不能为private

      ```java
      class Test{
          int id;
          public volatile int score;
      }
      ```

    - 然后构造对应类型的AtomicIntegerFieldUpdater，调用其中方法newUpdater并传参对应对象的类型和待处理的int变量名（字符串）

      ```java
      AtomicIntegerFieldUpdater<Test> updater = AtomicIntegerFieldUpdater.newUpdater(Test.class, "score");
      ```

    - 然后只需通过创建的updater调用所需要的方法即可，第一个传参对应待处理的对象

      ```java
      public class Test {
          public static void main(String[] args){
              final Student stu = new Student();
              Thread[] threads = new Thread[95];
              for(int i = 0; i < 95; i++){
                  threads[i] = new Thread(() -> {
                      //stu.score+=2
                      updater.getAndAdd(stu, 2);
                  });
      
                  threads[i].start();
              }
              while(Thread.activeCount() > 1){
                  Thread.yield();
              }
              System.out.println("id = " + stu.id + "\tscore = " + stu.score);
          }
          //构造的对象
          public static class Student{
              int id;
              public volatile int score;
          }
      	//Updater
          public final static AtomicIntegerFieldUpdater<Student> updater = AtomicIntegerFieldUpdater.newUpdater(Student.class, "score");
      
      }
      ```

      


Atomic包是通过Java中Unsafe类和CAS操作封装实现的，其中Unsafe类是允许java像c一样越过JVM对内存进行操作处理，但不安全，官方不推荐使用，因此叫“Unsafe"。

**在即将推出的Java 9中，Unsafe很有可能被取缔，这恐怕会带来以Unsafe为基础的包括Atomic包在内的几乎绝大多数程序和软件都将不可使用。**



***

[回到顶部](#目录)