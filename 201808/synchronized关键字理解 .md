## synchronized关键字的作用
多线程访问共享变量时，多个线程对同一个变量进行操作时，有可能会发生数据的脏读。比如：
![线程不安全-脏读](https://img-blog.csdn.net/20180808173808856?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2FzZDUwMTgyMzIwNg==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

输出结果小于200000的原因是，两个线程同时对index进行增加时，当线程1对读取了index的值，对index进行了自增，还未返回时，线程2也读取了index的值。线程2返回的index值会将之前的值覆盖掉，就出现了与预计不同的结果。

synchronized关键字的作用就是在多线程情况下，对共享数据的访问进行同步，能够保证在同一时间下只有一个线程会对共享数据进行操作。实现这种同步的方式就是使用synchronized关键字进行加锁。

---

## synchronized关键字的使用
synchronized关键字同步的方式有三种：

·修饰实例方法：相当于对当前对象进行加锁
·修饰静态方法：相当于对当前对象的类对象进行加锁
·修饰代码块   ：指定加锁对象，相当于指定对象加锁

####  修饰实例方法

```
public class SynchronizedTest01 implements Runnable {
    private static int index = 0;

    private static SynchronizedTest01 instance = new SynchronizedTest01();

	//实例方法同步
    private synchronized void increase() {
        index ++;
    }

    @Override
    public void run() {
        for (int i = 0; i < 100000; i ++)
            increase();
    }

    public static void main(String[] args) throws InterruptedException {
	    //对同一对象进行操作
        Thread thread1 = new Thread(instance);
        Thread thread2 = new Thread(instance);
        thread1.start();
        thread2.start();
        thread1.join();
        thread2.join();
        System.out.println("index = " + index);
    }
}

/**
 * index = 200000
 * Process finished with exit code 0
*/
```

Thread1，2调用同一个Runnable对象的run方法，而run方法中的increase函数是实例同步，在同一时间内只有一个线程获取到instance实例的锁，调用increase函数；而另一个线程会在等待队列中尝试获取锁。

如果线程1，2调用的是不同的Runnable对象，index的值不能保证为200000：因为两个线程调用的同步方法，获取的是各自实例对象的锁，并不冲突，运行效果与最上面的截图类似。

---

#### 修饰静态方法

```
public class SynchronizedTest01 implements Runnable {
    private static int index = 0;

    private static SynchronizedTest01 instance = new SynchronizedTest01();

	//静态同步方法
    private synchronized static void increase() {
        index ++;
    }

    @Override
    public void run() {
        for (int i = 0; i < 100000; i ++)
            increase();
    }

    public static void main(String[] args) throws InterruptedException {
	    //传入不同的Runnable对象
        Thread thread1 = new Thread(new SynchronizedTest01());
        Thread thread2 = new Thread(new SynchronizedTest01());
        thread1.start();
        thread2.start();
        thread1.join();
        thread2.join();
        System.out.println("index = " + index);
    }
}

/**
 * index = 200000
 * Process finished with exit code 0
*/
```

线程1，2传入不同的Runnable对象，increase方法修改为静态同步方法，index结果为200000。因为当synchronized关键字修饰静态方法时，线程尝试获取的锁是当前实例对象对应的类对象的锁(这句话有些绕：class对象的锁，不是实例对象的锁)。而静态方法同步时获取的就是类对象的锁，相同类的不同对象，获取到的也是同一个锁，能保证线程安全。

---

#### 修饰代码块

用synchronized关键字修饰方法时，是对函数整体的访问添加限制。当一个函数体结构很大，而需要同步的执行语句只有一小部分时，这时对函数整体添加同步，会降低代码的效率。这时我们可以用synchronized关键字直接修饰代码块，指明获取锁的对象，如下：

```
public class SynchronizedTest01 implements Runnable {
    private static int index = 0;

    private static SynchronizedTest01 instance = new SynchronizedTest01();

    private void increase() {
        index ++;
    }

    @Override
    public void run() {
        for (int i = 0; i < 100000; i ++) {
	        //修饰代码块
	        //修饰实例对象
            synchronized (this) {
                increase();
            }
            //修饰类对象
            //synchronized (SynchronizedTest01.class) {
            //    increase();
            //}
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Thread thread1 = new Thread(instance);
        Thread thread2 = new Thread(instance);
        thread1.start();
        thread2.start();
        thread1.join();
        thread2.join();
        System.out.println("index = " + index);
    }
}
```

实例对象的锁 -> synchronized(instance)
类对象的锁   -> synchronized(TestSynchronized.class)

---

## synchronized关键字和volatile关键字

共同点：
用于多线程访问中，对共享数据的同步，避免出现对线程访问时修改造成的数据异常。

不同点：
synchronized关键字如上所述，可用于修饰方法和代码块，通过获取不同对象的锁，来保证代码运行时的同步。这里再说一点：synchronized在代码运行之前就获取了锁，默认有其他线程与其争夺资源，是一种悲观锁；而CAS操作是在比较是否不同后再进行不同的操作，属于一种乐观锁。
volatile关键字用于修饰实例对象，通过jvm实现的规则来保证多线程访问数据的一致性：写操作优先于读操作；数据改变时会通知所有工作线程的本地副本失效，从内存中重新获取数据。volatile只能保证原子性操作的同步，如 i ++ 这类非原子性操作，即使用volatile修饰也无法保证同步，这时就需要用到synchronized关键字来修饰代码块了。

---

## synchronized的内部优化

java锁的状态有四种：无锁状态，偏向锁，轻量级锁，重量级锁。锁的状态会随着竞争而升级，但无法降级。偏向锁和轻量级锁是java 1.6时引入的，目的是减少获得锁和释放锁带来的性能消耗。下面让我们一起看一看：

#### 偏向锁

偏向锁使用了一种等到竞争出现时才会释放的机制，当其他线程尝试竞争偏向锁时，持有偏向锁的线程才会释放锁。

为什么要使用偏向锁呢？我们来想一下：当一个对象的方法设计为支持多线程调用时，使用synchronized关键字对代码块进行同步。如果对象目前只被线程A调用，此时如果线程A每次访问都需要获得对象的锁，执行完再释放锁，并没有实际的作用，浪费了性能。
所以此时，线程A会持有对象的偏向锁。当没有其他线程竞争时，线程A会一直持有该锁，不去释放；当线程A再次访问对象时，由于已经持有了偏向锁，进入和退出同步块时不需要进行任何同步操作和CAS操作。

#### 轻量级锁

轻量级锁加锁：线程在执行同步块之前，JVM会先在当前线程的栈桢中创建用于存储锁记录的空间，并将对象头中的Mark Word复制到锁记录中，官方称为Displaced Mark Word。然后线程尝试使用CAS将对象头中的Mark Word替换为指向锁记录的指针。如果成功，当前线程获得锁，如果失败，表示其他线程竞争锁，当前线程便尝试使用自旋来获取锁。

书接上文，当线程B开始争夺对象的锁时，发现偏向锁已被持有，这时会通过CAS来替换Mark Word，从而获取轻量级锁；而线程A持有的偏向锁也会被收回，与B一同竞争轻量级锁。获取到锁开始执行同步体，为获取到锁则进行自旋，等待锁释放。如果自旋的时间过长(自旋消耗cpu)，或是在自旋的过程中出现了第三个线程来进行争夺，轻量级锁就会膨胀为重量级锁。重量级锁使除了持有锁之外的线程都阻塞，防止轻量级锁时等待过程中的CPU空转(自旋)。

---

#### 参考资料

1.[(推荐) zejian_ -  深入理解Java并发之synchronized实现原理][1]
2.[存在mornning -【Java多线程 锁优化】锁的三种状态切换][2]
3.[深入理解java虚拟机]
4.[java并发编程艺术]

[1]: https://blog.csdn.net/javazejian/article/details/72828483
[2]: https://blog.csdn.net/sinat_33087001/article/details/77849575