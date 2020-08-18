# Java锁

参考博客[多线程：锁](https://blog.csdn.net/vbirdbest/article/details/81511570)和[Java并发之彻底搞懂偏向锁升级为轻量级锁](https://www.cnblogs.com/tiancai/p/9382542.html)

<span id = "目录"><b>目录</b></span>

[toc]

## synchronized

java中提供的synchronized关键字实际上是一种重量级锁，被修饰的对象呈现原子性

synchronized有三种锁的方式

- 对象锁

  顾名思义，对象锁即为锁住一个对象

  常见使用方法

  ```java
  class MyRunnable implements Runnable{
      void run(){
          //this锁，锁定当前myRunnable对象
          synchronized(this){
              //操作
          }
      }
  }
  
  Object lock = new Object();
  public void test(){
      //锁lock对象
      synchronized(lock){
          //操作
      }
  }
  ```

- 修饰方法的对象锁

  即在方法前加上synchronized关键字，本质上是对调用该方法的对象的对象锁

  ```java
  class MyRunnable implements Runnable{
      //本质是对象锁，锁为this
      synchronized void myRun(){
          //操作
      }
  }
  ```

- 类锁

  类锁很容易与对象锁相混淆，简单来说，类锁是全体学生不得离校，而对象锁是小明不得离校

  当static修饰对象中的synchronized方法时，锁定的是对应类Test.class

  ```java
  class Test{
      //本质是对象锁，锁为this
      public synchronized void myRun1(){
          //操作
      }
      
      //本质是类锁，锁为Test.class
      public synchronized static void myRun2(){
          //操作
      }
  }
  ```

  当两个线程分别调用对象test的myRun1和myRun2方法时，呈现并行运行而非同步运行，因为锁的类型不同



***

## Lock()

java.util.concurrent中提供locks包，提供synchronized之外的上锁方式

与synchronized不同的是，Lock()有以下几点特征

- 使用灵活，但是必须手动开启和释放锁
- 扩展性好，有时间锁等候（tryLock( )），可中断锁等候（lockInterruptibly( )），锁投票等，适合用于高度竞争锁和多个条件变量的地方
- 提供了可轮询的锁请求，可以尝试去获取锁（tryLock( )），如果失败，则会释放已经获得的锁。有完善的错误恢复机制，可以避免死锁的发生。

最常用的是ReentrantLock可重入锁和ReadWriteLock

```java
public class ReentrantLockTest {
    public static void main(String[] args) {
        //可重入锁
        Lock lock = new ReentrantLock();

        Runnable runnable = () -> {
            System.out.println(new Date() + "\t" + Thread.currentThread().getName() + "\trunning...");
            //上锁
            lock.lock();
            try {
                for(int i = 0; i < 4; i++) {
                    System.out.println(new Date() + "\t" + Thread.currentThread().getName() + "\ti = " + i);
                    try {
                        Thread.sleep(1000);
                    }catch(InterruptedException e){}
                }
            }catch(Exception e){
                e.printStackTrace();
            }finally{
                //释放锁，一般放在finally
                lock.unlock();
            }
        };

        new Thread(runnable, "Thread-A").start();
        new Thread(runnable, "Thread-B").start();
    }
}
```

由于Object.wait()和Object.notify()只能用于synchronized，Lock()对象只能通过Condition()对象来实现wait和notify操作

```java
public interface Condition {
    // 等待，相当于Object.wait()
    void await() throws InterruptedException;
    long awaitNanos(long nanosTimeout) throws InterruptedException;
    boolean await(long time, TimeUnit unit) throws InterruptedException;

    // 唤醒，相当于Object.notify()
    void signal();  
    void signalAll();
}
```

实例

```java
public class ConditionTest {
    public static void main(String[] args) throws InterruptedException{
        //使用相同的锁和条件
        Lock lock = new ReentrantLock();
        Condition condition = lock.newCondition();
        System.out.println(condition);

        new Thread(new MyRunnable1(lock, condition), "Thread-A").start();
        Thread.sleep(1000);
        new Thread(new MyRunnable2(lock, condition), "Thread-B").start();
    }

}

class MyRunnable1 implements Runnable {
    private Lock lock = null;
    private Condition condition = null;
    public MyRunnable1(Lock lock, Condition condition) {
        this.lock = lock;
        this.condition = condition;
    }

    @Override
    public void run() {
        System.out.println(new Date() + "\t" + Thread.currentThread().getName() + "\t开始等待>>>");
        lock.lock();
        try {
            condition.await();
        }catch(InterruptedException e){
            e.printStackTrace();
        }finally{
            lock.unlock();
        }
        System.out.println(new Date() + "\t" + Thread.currentThread().getName() + "\t已苏醒");
    }
}

class MyRunnable2 implements Runnable {
    private Lock lock = null;
    private Condition condition = null;
    public MyRunnable2(Lock lock, Condition condition) {
        this.lock = lock;
        this.condition = condition;
    }

    @Override
    public void run() {
        lock.lock();
        try {
            System.out.println(new Date() + "\t" + Thread.currentThread().getName() + "\t开始通知...");
            condition.signal();
            System.out.println(new Date() + "\t" + Thread.currentThread().getName() + "\t通知完毕。。。");
        }catch(Exception e){
            e.printStackTrace();
        }finally{
            lock.unlock();
        }
    }
}
```



***

## 锁的四种等级

Java中锁从低到高分为四个等级：无锁，偏向锁，轻量级锁，重量级锁

锁只能升级而不能降级，消耗的CPU资源随等级递增

下图是用于区分锁类型的对象头（Markword）内容

<img src="img\Java锁\1.png" alt="image-20200817223004662" style="zoom:67%;" />

- 无锁
  - 不存在多线程竞争的同步代码
- 偏向锁
  - 当线程之间**不存在竞争**，获得锁的总是同一个线程时，会采用偏向锁，即偏向该线程
  - 原理是在线程访问同步块并获得锁时，同步块会在对象头中记录对应的线程id，当下一次线程访问同步块时只需比较线程id是否相同，若相同则不需要再获得锁和释放锁
  - 由于线程在执行完毕偏向锁中的同步块后并不会主动释放偏向锁，当新的不同线程尝试获取偏向锁时需先重置对象头。
- 轻量级锁
  - 当线程间存在竞争，但竞争不激烈，不会出现阻塞时会采用轻量级锁
  - 加锁：线程在执行同步块的时候，JVM会现在栈帧中开辟一个存储锁记录的空间，并将对象头中的Markword复制到锁记录中，然后线程会使用CAS将对象头中的MarkWord替换为指向锁记录的指针，如果成功，线程获取锁，如果失败，则表示其他线程占有该锁，当前线程会通过**自旋**来再次尝试获取锁
  - 释放锁：解锁时候会通过CAS将加锁时栈帧中复制的Markword替换回到对象头中，如果成功，表示没有竞争，失败表示存在竞争关系，锁会膨胀成重量级锁
  - **volatile是一种轻量级锁，在一写多读的情况上很有用**
- 重量级锁
  - 当线程间存在激烈竞争时会采用重量级锁，直接阻塞等待线程
  - 未获取到锁的线程会阻塞，直到锁释放被唤醒后再获得锁
  - synchronized和Lock()都是重量级锁



***

## 锁的膨胀

当设置的偏向锁遇到线程竞争获得锁的情况，会升级为轻量级锁。当轻量级锁遇到线程自旋后仍然不能获得锁的情况，会升级为重量级锁，以避免持续的自旋带来的极大CPU资源消耗。这一过程被称为锁的膨胀（锁的升级）。

- 偏向锁 ~ 轻量级锁
  - 初始锁为无锁状态，Markword为`hashCode|age|0|01`
  - 线程T1获得偏向锁，通过CAS替换设置Markword为`T1|Epoch|age|1|01`，开始执行同步块代码
  - 线程T2通过锁的Markword判断锁为偏向锁，试图获得偏向锁，由于此时线程T1仍然占用着锁**（线程T2执行start方法时，传入的thread_list表示存活着的线程，只需在CAS时判断线程1是否存在于thread_list即可）**，获得失败，**暂停线程T1**，将Markword的锁标志位设置为00，表示膨胀为轻量级锁
  - 在线程T1栈帧的LockRecord中读取地址，然后更新Markword为`T1 address|00`，然后继续执行线程1的同步代码块
  - 于是锁膨胀为了轻量级锁，同时线程2在CAS尝试替换失败后开始**自旋**尝试获得锁
  - 当线程T1执行完毕后，发现Markword已被更改为轻量级锁，于是释放Markword中的指针，更新Markword为`0|00`释放锁
- 轻量级锁 ~ 重量级锁
  - 初始锁为无锁状态，Markword为`hashCode|age|0|01`
  - 线程T1尝试获得锁，在栈帧中开辟空间LockRecord来存储Markword，然后通过CAS替换Markword为`T1 address|00`，表示轻量级锁，然后开始执行同步代码
  - 线程T2尝试获得锁，在栈帧中开辟空间LockRecord来存储Markword，在CAS时发现Markword为`T1 address|00`而非预期的`hashCode|age|0|01`，CAS失败，进入自旋再次尝试
  - 线程T2自旋结束后，进入CAS发现锁标志仍为00，即锁依然为轻量级锁并被占用，于是线程T2通过CAS将Markword替换为`monitor address|10`，即锁膨胀为重量级锁，线程T2进入阻塞状态
  - 线程T1在执行完毕后，尝试通过CAS将Markword替换回`hashCode|age|0|01`来释放锁，但由于Markword被线程T2修改为`monitor address|10`，CAS失败，于是释放monitor address，将Markword替换为`0|10`释放锁，同时唤醒其他阻塞线程来获得锁。



***

## 锁的其他机制

可重入锁和不可重入锁

- 可重入锁是指当一个对象已获取到某个锁后，该对象可以再次获取该锁，而其他对象不可以。
- 不可重入锁与可重入锁相对，一个对象已获取到某个锁后，任何对象都不可再获取该锁直到该对象释放锁。
- 可重入锁可以**防止死锁**，其实现原理是：在锁中设置一个计数器，在获得锁的方法中判断获得锁的对象是否是一开始获得锁的对象，若是的话计数器加一，若不是则获取锁失败。释放锁同理计数器减一。当计数器为0时锁即被释放。
- java中的synchronized关键字与提供的ReentrantLock()对象都是可重入锁

公平锁和非公平锁

- 公平锁指线程获取锁的顺序与线程创建顺序相同，采用先来先得的顺序。
- 非公平锁指线程获取锁的顺序与线程创建顺序不一定相同，即后创建的线程获取锁时可以插队到先创建的线程之前。
- 在构造ReentrantLock()对象时可传参一个布尔变量isFair，表示是否为公平锁。默认为非公平锁。

乐观锁和悲观锁

- 进入一个方法前必须加锁，这种锁为悲观锁，而乐观锁为一开始可以不加锁，当出现冲突时再加锁，如CAS机制
- 重量级锁为悲观锁的体现，轻量级锁为乐观锁的体现



***

[回到顶部](#目录)