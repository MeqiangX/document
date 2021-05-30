# Java-basic
___
基础语法和sdk知识-记录java核心技术卷笔记

# 多线程

Cpu和线程数的关系：

> 计算机五大组成：控制器，运算器，存储器，输入设备，输出设备。
>
> 其中控制器主要是指指令计数器和指令寄存器，存放各种指令二进制，指挥整个计算机的有序运行，执行方法后如何回到原来的代码位置继续执行剩余方法（指令计数器的入栈出栈）；多线程之间的切换（指令计数器保存各就绪线程的锚指令）等等，都依赖于控制器
>
> 运算器即我们常说的cpu（包含内部的缓存寄存器属于存储部分），以前的cpu都是单核的，即一个cpu一个内核，同一时间只能处理一个线程，也就不存在所谓的并发和多线程执行了，这里说的cpu和内核都是物理上的区分，即你拆开主机板就可以看到cpu的个数，而后面随着发展，cpu还可以分为多个内核，常说的1cpu内部还可以细分为多个内核，这里的拆分也是物理的拆分，后面一个内核又可以分为多个线程（超线程技术），常说的单核4线程，双核4线程，四核8线程等等，这种属于逻辑拆分。

进程和线程的关系：

> 进程是操作系统进行资源（cpu，内存，磁盘，io，带宽）等分配的最小单位
>
> 而线程是cpu调度和分配的基本单位
>
> 举个例子：打开一个浏览器，一个聊天窗口分别是一个进程，而进程可以有多个子任务，聊天工具接受消息，发送消息，浏览器渲染图片，加载视频等是线程，cpu执行调度分配的最小单位都可以是一个线程来单独处理，而进程是一个完整的模块

java中多线程的创建方式：

```java
1、继承Thread类（java单继承，扩展性差，放弃）
  public MyThread extends Thread{}
2、实现Runnable(接口好扩展，可行)，重写run方法
  public MyThread implements Runnable{}
3、java8后使用函数式编程，通过匿名类的方式创建（推荐）
  Runnable r = () -> {//do sth};
创建好线程后并没有启动，需要通过Thread t = new Thread(r); t.start(); 来启动线程
```

多线程竞争问题：

1. synchronized关键字：

> 1. 对方法或者代码块进行显示加锁，底层是基于jvm的
> 2. 用法
>
> ```java
> //1.方法使用synchronized修饰
> public synchronized void syn(){}
> // 使用同步锁关键字修饰的方法为同步方法，并发操作同时只有一个线程进入，synchronized是隐式释放锁的，在块区域结束后自动释放锁。
> 
> //2.代码块中使用synchronized
> synchronized {}
> // 和修饰方法是一样的，并发操作的多个线程只有一个获取到对象的内部锁的线程才执行这段代码，执行完后释放锁，唤醒其他线程执行
> ```
>
> 3. 锁可以有一个或多个相关的条件对象
>
> ```java
> public void transfer(double[] accounts,int from,int to,double amount){
>   synchronized (accounts){
>     
>   } 
> }
> ```
>
> 4. 如果需要在锁代码块内部条件控制，可以调用object的wait()，notifyAll()
>
> ```java
> public synchronized void conditionSynTest(){
>   while (conditon == false){
>     wait();//当前线程由运行态 转变为等待态
>   }
>   // do sth
>   
>   // 唤醒
>   notifyAll();
> }
> ```

2. ReentrantLock重入锁

> 可重入的定义为对象内部锁有一个计数，在锁的块区域中每次方法压栈内部锁计数都会+1，进入锁区域也会+1，弹出栈计数-1，直到结束锁块，计数=0，释放锁；事实上，synchronized和ReentrantLock都是可重入锁，只是一个基于jvm（synchronized），一个是基于他的封装，基于jdk，两者功能类似，一个隐式，一个显式释放锁
>
> ___
>
> 1. 使用
>
> ```java
> public class Demo{
>   private Lock blockLock = new ReentrantLock();
>   public void lockTest(){
>      try{
>        blockLock.lock();
>        // do sth
>      }finally{
>        blockLock.unLock();
>        //记得在finally中释放锁
>      }
>   }
> }
> ```

3. 条件锁

> 并发竞争的产生往往是伴随着多个条件产生，a在等待b，b在等待c锁，c在等待a锁，处理不当往往容易产生死锁
>
> Condition 条件，一个锁可以有多个条件对象
>
> ```java
> public class ConditionDemo{
>   private Lock lock = new ReentrantLock();
>   private Condition c1;
>   private Condition c2;
>   
>   ConditionDemo(){
>     c1 = lock.newCondition();
>     c2 = lock.newCondition();
>   }
>   
>   public void transfer(){
>     try{
>       c1.await();
>       //do sth
>       c2.await();;
>       // do sth
>       c2.notifyAll();
>       //release condition lock
>       c1.notifyAll();
>     }catch (InterruptedException e){
>       //
>     }finally{
>       c2.notifyAll();
>       c2.notifyAll();
>     }
>   }
> }
> ```

4. Volatile域关键字

> 缓存一致性：
>
> 内存的访问速度要比高速缓存cache(cpu中的缓存寄存器)要慢的多，所以当程序运行时，会将运算需要的数据从内存复制一份到cpu的高速缓存中。cpu如果要进行计算，就直接从缓存中取数，运算结束后再将结果刷新到主存中，；而每个cpu的cache是不一致的，这就带来了缓存的不一致性；
>
> 有两种解决方案：
>
> 1、加锁Lock
>
> 2、定义缓存一致性协议
>
> 加锁会阻碍其他cpu对其他部件（如内存）的访问，从而使得只有一个cpu能使用这个变量的内存，效率低下
>
> 缓存一致性协议保证每个缓存使用的共享变量的副本都是一致的，核心思想是当cpu写数据的时候，如果发现操作的变量是共享变量，（在其他的cpu缓存中也存在此变量的副本），会发出信号给其他的cpu通知将缓存置为无效状态，当其他的线程需要读取这个变量时，发现当前cpu的缓存是无效的，那么他就回取内存的最新值来覆盖缓存，从而保证每个线程读取到的cpu缓存中的共享变量为最新的
>
> <img src="https://cdn.jsdelivr.net/gh/MeqiangX/cloud-img@master/212219343783699.jpg" width=100% height=300 title='数据流' alt='loadingfail'></img>

Volatile为实例域的同步访问提供了一种免锁机制，如果变量被声明为volatile，那么编译器和jvm就知道这个域是可能被另一个线程并发更新的。

锁也可以实现这个机制，但是繁琐且耗时，并且方法阻塞，而volatile则快很多，也不用为了单独的一个变量而设立独立的Lock

> Volatile的原理和实现机制：
>
> 有volatile关键字时，生成汇编代码后，会多出一个lock的前缀指令，相当于一个内存屏障，会提供三个功能：
>
> 1、确保指令重排的时候不会将后面的指令排到内存屏障之前的位置，也不会把内存屏障之前的指令排到内存屏障的后面，也就是在执行这一行的时候，前面已经都执行完成了
>
> 2、强制对缓存的操作立即写入主存
>
> 3、如果是写操作，会使其他cpu中的缓存行无效

5. 并发编程3个概念

   1. 原子性

   一个操作或者多个操作要么全部执行，并且执行过程不会被任何因素打断，要么就都不执行

   2. 可见性

   值的改变并不会从缓存中直接刷新到内存，而是会在所有运算结束后才被写入主存，而多线程的其他缓存中的共享变量是不会因为一个变量值的改变而同步的，除非当前缓存的值被置为无效状态，即被volatile修饰，才会当值改变时从内存中重新读取

   3. 有序性

   对于程序代码逻辑没有依赖性的几行来说，jvm并不会按照代码行数顺序执行，而是会进行指令冲编排，这是处理器为了提高程序运行效率，而对输入代码的执行顺序进行的优化，前提是不会对程序的最终结果产生变化，保持一致

   要保证多线程的环境下程序正常运行，就要保证这三个条件，否则最终的结果可能不是最终想要达成的结果

6. java内存模型

> jvm定义了一种java内存模型（java Memory Model）JMM，来屏蔽各个平台硬件平台和操作系统的内存访问差异，以此在实现java程序在各种平台下都能达到一致的内存访问效果

> 为了获得较好的执行性能，java内存模型没有限制用寄存器或高速缓存来提升指定执行速度，也没有限制编译器对指令进行重排序，在底层并发的三个条件限制对jmm来说也是一样的，那么他是如何来保证这三个条件的呢：
>
> 1、原子性，只保证基本读取和赋值（只针对确定的具体值赋值给变量，而不是变量赋值给变量，a=b，这里面包含了 1.取b的值，2.赋值给a两部操作）
>
> 2、可见性，通过volatile关键字来保证可见性
>
> 3、有序性，可以通过volatile来保证一定的有序性（禁止重编排），也可以使用synchronized和lock来保证有序性

7. 原子操作类

在大大多数并发的情况下，java sdk编写者都会考虑到，有给开发者们提供原子操作类和线程安全集合来安全的操作基本数据类型和集合，java.util.concurrent.atomic包中包含了常见数据类型的原子操作包装类，如AtomicInteger,AtomicLong，AtomicBoolean，数组类的有AtomicIntegerArray，AtomicLongArray，引用类的有AtomicReference，AtomicReferenceArray等，源码中都是通过CAS+volatile来实现的，下面通过AtomicInteger的源码来分析原理：

```java
public class AtomicInteger extends Number implements java.io.Serializable {
    private static final long serialVersionUID = 6214790243416807050L;

    // setup to use Unsafe.compareAndSwapInt for updates
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long valueOffset;

    static {
        try {
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }

    private volatile int value;

    /**
     * Creates a new AtomicInteger with the given initial value.
     *
     * @param initialValue the initial value
     */
    public AtomicInteger(int initialValue) {
        value = initialValue;
    }
```

几个关键信息属性：

1、private volatile int value;

​	value是当前值，使用volatile修饰，来确保每次访问更新的时候拿到的都是value最新的值

2、static final Unsafe unsafe = Unsafe.getUnsafe();

​	Unsafe类，和C/C++不同，java没有直接对内核和操作系统，都是通过jvm，当然也不能直接操作一块内存区域，Unsafe就是jvm提供给我们操作管理内存的这么一个类，他的全限定名是sun.misc.Unsafe，一般开发应用者不会用到这个类，管理操作内存是比较危险的一件事情，所以在java中他被修饰为final，并且构造器私有化，只能通过反射来得到，当然类中也提供了静态方法通过类加载器来得到，原理相同，这个类中主要包含内存管理的一些api：

![alt 加载失败](https://cdn.jsdelivr.net/gh/MeqiangX/cloud-img@master/20210530101914.jpg "Unsafe常用功能")

原子操作类和线程安全集合底层都是操作unsafe，依靠unsafe的cas和volatile来完成

3、private static final long valueOffset

这是个比较有意思的属性，可以看到，在AtomicInteger的static代码块中随着AtomicInteger.class加载进jvm，也被初始化，

```java
static {
        try {
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }
```

即：在类加载后就固定了，思考下这个属性的作用，通过unsafe.objectFieldOffse()方法得到value在内存中相对AtomicInteger实例的偏移量，通过valueOffset来得到value在AtomicInteger对象中的位置是这个属性的核心作用，图解：

![alt loading](https://cdn.jsdelivr.net/gh/MeqiangX/cloud-img@master/20210530101930.png "valueOffset图解")

- valueOffset可以定位到AtomicInteger中value的位置
- AtomicInteger中valueOffset是固定的（static final）,不因不同实例而改变，随着类文件被加载的jvm，相对于value的偏移量就确定了

4、方法实例解析：

`getAndAdd`方法

```java

    /**
     * Atomically adds the given value to the current value.
     *
     * @param delta the value to add
     * @return the previous value
     */
    public final int getAndAdd(int delta) {
        return unsafe.getAndAddInt(this, valueOffset, delta);
    }
```

Unsafe.getAndAddInt：

源码中的变量名不太好理解，替换成好理解的变量名来看

```java
public final int getAndAddInt(Object atomicInstance, long valueOffset, int delta) {
        int result;
        do {
            result = this.getIntVolatile(atomicInstance, valueOffset);
        } while(!this.compareAndSwapInt(atomicInstance, valueOffset, result, result + delta));

        return result;
    }
```

unsafe.compareAndSwapInt：

```java
/**
* unsafe中基本都是native本地方法来对内存直接操作
* var1 需要更新的原子对象
* valueOffset 原子对象中value相对于对象首地址的偏移量
* except 希望field中的value值
* 如果期望值except和filed-value的当前值相同，则设置field-value为这个值
**/
public final native boolean compareAndSwapInt(Object var1, long valueOffset, int except, int update);
```

这就是CAS的核心，他会以原子性的操作来完成三个操作:

- 获取原子对象的value相对对象首地址的偏移量offset
- 通过偏移量来获取value，比较value和期望值是否相同
- 相同则更新value为update，否则不更新，循环直到工作内存中的值和主存中的同步

现在思考模拟一下多线程（A,B两个线程）的情况下对AtomicInteger执行从1加到100的操作：

- 线程A首先执行`getIntVolatile()` 方法，第一次执行，所以`getIntVolatile()`得到的一定是内存中最性的值，即为1
- 线程下一步要执行CAS比较，假如这时线程A被阻塞了（`getAndAddInt()`并不是原子性），线程B开始执行
- 线程B执行`getIntVolatile()`，工作内存和主存中的值都为1，获取工作内存的值为1
- 线程B执行CAS，expect和value的值相同，执行update，更新值为1+1=2，volatile将工作内存的2刷新到主存2，线程B执行结束
- 此时线程A恢复，继续执行，执行到CAS时，此时expext和value（获取到的为B线程更新后的最新值）的值不相同，不执行更新，进行下一轮循环
- 线程A进入下一次循环，执行`getIntVolatile()`，此时获取到的期望值更新为最新的2
- 线程A执行CAS，期望值和value值相同=2，执行更新操作2+1=3，工作内存为3刷新到主存，线程A结束。

---

8. 线程类和线程池

   1. Runnable、Future、Callable

      Runnable无返回结果，执行方法是`run()`，Callable有返回结果，执行方法是`call()`，Future保存线程执行状态和计算结果  

   2. FutureTask包装器

      实现RunnableFuture(同时继承Runnable和Future的接口)，可以将Runnable和Callable都转为Futur

   3. 执行器Executors和线程池

      1. 构建一个新的线程是有一定代价的，涉及到操作系统的交互和分配资源调度，正确的做法应该是创建一个线程池，而不是频繁的去创建和销毁线程（如果涉及到需要创建许多短周期的线程），将线程对象交给线程池，就会调用run/call方法，执行完成后，线程会回到线程池准备下一次的执行，而不是销毁，执行器（Executors）包含了很多静态工厂方法来构建不同需求的线程池：

         |               方法               |                         描述                          |
         | :------------------------------: | :---------------------------------------------------: |
         |       newCachedThreadPool        | 创建缓存线程池，空闲线程会被保留60s，必要时创建新线程 |
         |        newFixedThreadPool        |       创建固定线程数的线程池，空闲线程会被保留        |
         |     newSingleThreadExecutor      |   只有一个线程的线程池，会按照提交任务的顺序来执行    |
         |      newScheduledThreadPool      |           预定时间间隔/周期执行任务的线程池           |
         | newSingleThreadScheduledExecutor |          预定时间/周期执行的单个线程的线程池          |

---

9. 同步器（AQS AbstracuQueueSynchronizer为基础原理）

   java.util.concurrent（JUC）包包含了几个能帮助人们管理相互合作的线程集，这些机制具有为线程之间的共用集结点模式

   | 类               | 作用                                                         | 说明                                                         |
   | ---------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
   | CyclicBarrier    | 让线程集等待直至其中预定数目的线程都到达一个公共障栅（barrier），然后可以选择性的执行一个处理障栅的动作 | 当障栅后续的操作需要得到前面线程集的操作结果和的时候         |
   | CountDownLatch   | 允许线程集等待直到计数器减为0                                | 当一个线程或多个线程需要等待直到指定数目的事件发生           |
   | Semaphore        | 流量控制，限制最大的并发访问线程数                           | 为线程操作设置信号量，数量为最大的并发访问线程数，数量之内的线程可以获取信号-1，当信号量为0时，达到最大并发，不允许另外的线程进入直到拿到信号量的线程执行完释放信号量，这只是限制并发数量，限流，并不能保证线程安全，要达到线程安全还需配合锁来完成 |
   | SynchronousQueue | 允许一个线程把对象交给另一个线程                             | 生产者消费者，在没有显示同步的情况下，把一个对象从一个线程传递给另外一个对象（数据流的方向是确定单向的，不同与Exchanger是双向的） |
   | Exchanger        | 允许两个线程在要交换的对象准备好时交换对象                   | 当一个线程到达exchange调用点时，如果其他线程此前调用了此方法，则其他线程会被调度唤醒并与之进行对象交换，然后各自返回，如果其他线程还没达到交换点，则当前线程会被挂起，直到其他线程到达调用才会完成交换并正常返回，或者有超时中断返回 |
   | Phaser           | 类似于cyclicbarrier，更加灵活，计数器是可以手动改变的        | （`register()`为part(phaser中计数器的叫法)+1，`arriveAndDeregister()` part-1）另外可以自定义各个周期到达后的触发事件 |

10. MESI缓存一致性协议

    参考博客：https://www.cnblogs.com/ynyhl/p/12119690.html

    背景：

    ​	在单线程模式下，一块内存只对应一个cpu核心的缓存，且只被一个线程访问。缓存独占，不会出现访问冲突等问题。  在多线程模式下，一个CPU，且CPU单核，进程中的多个线程会同时访问进程中的共享数据，CPU将某块内存加载到缓存后，**不同线程在访问相同的物理地址的时候，都会映射到相同的缓存位置，这样即使发生线程的切换，缓存仍然不会失效**。但由于任何时刻只能有一个线程在执行，因此不会出现缓存访问冲突。当然，由于CPU内部寄存器的存在，会有多线程下的 A++ 非原子性语句导致的问题。 
    （ 
      A++ ==> 
      reg = *(&A); 
      reg ++; 
    \*  (&A) = reg; 
    在发生线程切换的时候，线程使用的内部寄存器会被作为现场进行保存。 
    ） 
     	多线程模式下，一个CPU，且CPU有多核，每个核都至少有一个L1 cache。多个线程访问进程中的某个共享内存，且这多个线程分别在不同的核心上执行，则每个核心都会在各自的caehe中保留一份共享内存的缓冲。由于多核可以做到真并行，可能会出现多个线程同时写各自的cache， 
      因此CPU有“缓存一致性”原则，即每个处理器（核）都会通过嗅探在总线上传播的数据来检查自己的缓存值是不是过期了，当处理器发现自己缓存行对应的内存地址被修改，就会将当前处理器（核）。因此，我们经常看到在多核多线程的场景下，在声明变量时候使用volatile，volatile变量要求在更新了缓存之后立即写入到系统内存，而非volatile变量，则是CPU修改缓存，缓存在适当的时候（不知道什么时候）将缓存数据写入内存。写入内存的操作会出发其他处理器（核）将自己已经缓存的那块正在被写入的内存失效，并在下次需要使用到该内存的时候重新从内存读取。

    ​	总结：因为目前的cpu大都是多核，每个核心都有自己对应独立的缓存，所以在多线程的环境下，如果多个线程同时多一个变量进行修改时，会造成结果的随机性，而解决这个问题的一个方法就是总线加锁，即对内存进行加锁，在一个线程核心对变量修改的过程中，其他的线程无法操作内存的其他数据，这样会导致cpu的执行效率直线下降，而MESI缓存一致性协议提供了一种高效的内存数据管理解决方案，他只会对缓存行（缓存行是缓存中数据存储的基本单元）的数据进行加锁（eg：一个变量加锁），不会影响内存中其他数据的读写，参考代码中对变量的加锁。

    MESI分别代表缓存行数据所处的四种状态，通过对这四种状态的切换，来达到对缓存数据进行管理的目的

    | 状态                     | 描述                                                         | 监听任务                                                     |
    | ------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
    | M 修改(Modify)           | 该缓存行有效，数据被修改了，和内存中的数据不一样，数据只存在于本缓存行中 | 缓存行必须时刻监听所有试图读该缓存行相应的内存的操作，其他缓存须在本缓存行写回内存并将状态置为E之后才能操作该缓存行所对应的内存数据 |
    | E 独占 互斥（Exclusive） | 该缓存行有效，缓存行的数据和内存中的数据一致，数据只存在于本缓存行中 | 缓存行必须监听其他缓存读取内存中该缓存行对应的内存的操作，一旦有这种操作，该缓存行需要变成S状态 |
    | S 共享（Share）          | 该缓存行有效，并且该缓存行的数据和内存中的一致，数据同时存在其他缓存行中 | 缓存必须监听其他缓存使该缓存行无效或者独享该缓存行的请求，并将该缓存行置为I |
    | I 无效 （Invalid）       | 该缓存行数据无效                                             | 无                                                           |

    MESI工作流程原理：

    > 1、两个cpu(或者一个cpu两核，这里为了简单就使用单核的情景)cpu1和cpu2，cpu1从内存中读取变量a加载到缓存中，并将变量设为E独占，并通过总线嗅探机制对变量a的操作进行嗅探（监听）
    >
    > 2、cpu2读取变量a，总线的监听监听到了这个操作，将cpu1的缓存行数据状态置为S 共享，并将a加载到cpu2的缓存中，状态为S 
    >
    > 3、cpu1对变量a进行修改，总线嗅探会将cpu1的缓存行数据a状态置为M 修改，而此时嗅探会将cpu2的a状态置为I 无效，但是要注意，cpu2的a状态I 只是保证cpu2对变量a的修改操作不会被写回内存，有可能会在cpu1修改完a写回内存并且置为E之前，重新读取内存中a的值，它只能保证并发编程的可见性，并不能保证原子性和有序性
    >
    > 4、cpu1将修改好的数据写回内存，并将变量a置为E，嗅探机制得知变量a修改完成，cpu2会重新去内存中加载变量a，并同时将cpu1，cpu2的变量a置为S

---

# jvm

1. java类加载过程

   类加载器：

   自定义类加载器 --> AppClassLoader  --> ExtClassLoader --> BootStrapClassLoader

   <img src='https://cdn.jsdelivr.net/gh/MeqiangX/cloud-img@master/20210530102055.jpg' width=400 height=200 alt='error' title='类加载器'></img>

   java类从加载到卸载拢共7个步骤：

   加载ClassLoader.loadClass --> 链接ClassLoader.resolveClass（验证，准备，解析）--> 初始化 --> 使用 --> 卸载

   类加载加载：ClassLoader双亲委派机制将java文件加载到jvm中（Class对象）

   验证：编译后的class字节码文件是否符合jvm规范

   准备：创建接口或者类中的静态变量，并且初始化值，侧重点在于分配所需的内存空间，而不执行进一步的JVM指令

   解析：将被加载类对象里的符号引用转化成直接引用，类似于ioc依赖注入，将依赖的变量（常量池，指向堆或栈中的内存空间）赋值为确切的内存地址，在编译加载类时，类对象并不知道所引用的类的实际地址，所以只能用符号引用来替代（例如：CONSTANT_Methodref_info等类型的常量出现），在解析阶段便将这些符号引用转化为确切的指向地址

   初始化：对静态变量，静态代码块执行 初始化，不同于准备阶段的初始化，这里是执行确切的赋值和执行代码指令，而准备阶段的初始化是只是分配空间和默认值

   使用：加载链接初始化后，类就成功加载到了jvm中，可以通过jvm的类模版来创建对象使用了

   卸载：

   满足类可卸载的条件：

   > 1、该类的所有实例都被回收，并且堆中也没有该类的任何实例
   >
   > 2、加载该类的ClassLoader类已经被卸载
   >
   > 3、该类对应的Class对象没有任何地方被引用，无法在任何地方通过反射访问该类的方法

   上述三个条件全部满足，jvm就会在方法区垃圾回收的时候对该类进行卸载，类的卸载过程是在方法区中清空类信息，java类的整个生命周期就结束了。

2. 双亲委派机制

   背景：避免重复加载+避免核心类的篡改

   双亲委派机制的实现非常简单，可以查看ClassLoader里loadClass源码，向上传递寻找，看是否当前限定类名的类已经被加载，如果没有，则parent加载，如果父类加载器在搜索范围内都没有找到，则类的加载器再加载，如果都没有的话，报ClassNotFound异常

   Class.getClassLoader().getParent()，传递继承关系如上图

3. jvm内存模型

   jvm内存可分为程序计数器，虚拟机栈，方法区，堆区，运行时常量池，本地方法栈，其中程序计数器，虚拟机栈，本地方法栈，是每个线程各自区分的，程序计数器存放的是线程执行的jvm指令地址，虚拟机栈就是我们常说的栈区，存放的是栈桢，栈桢指向的内存结构包括（局部变量表，操作数栈，动态链接，返回的代码位置（方法正常退出或者异常退出的定义））本地方法栈和虚拟机栈类似，但是本地方法栈中存的是程序对本地方法native的调用栈桢；方法区和堆区，运行区常量池是线程共享的，方法区中就是所有加载进jvm的类对象模版，堆区是存放的引用类型的存储空间和引用，运行区常量池存放各种常量

   <img src='https://cdn.jsdelivr.net/gh/MeqiangX/cloud-img@master/20210530102128.png'></img>

4. GC垃圾回收 

   背景：

   ​		GC（Garbege Collection）垃圾回收机制，不同于c/c++面向系统的编程语言，对程序员手动操作内存的要求很高，java的内存回收有专门的机制，即GC，通过对分配的堆内存的监控和回收器的回收算法，来完成内存的动态释放

   工作流程和原理：

   ​		回收算法，复制算法，标记-清除（mark-sweep）算法，标记-整理（mark-compact）三种算法

    		年轻代：分为eden区和survivor（from和to）区，eden是大部分新建对象的内存分配区域，如果对象比较大，则会直接分配到old老年代区；当eden区满了时，会发生第一次minor (次要的) gc，将eden中存活对象复制到from区，将没有被引用的对象回收，并且存活被复制到from区的对象会被标记上1，表示存活对象的年龄；经过一次minor gc，eden区会空闲下来，直到触达第二次minor gc，这时候eden和from区的存活对象都会被复制到to区域，大对象到old区，并且from和to区的存活对象的标记年龄都加1；这个过程会发生很多次，直到存活对象的标记年龄达到阀值，超过阀值的对象会晋升到老年代；

   ​		老年代：到多次minor gc后，通过晋升或者大对象分配而old年代区域也满了，则会发生老年代gc，即Major GC，标记-清除算法之后会有很多零散的内存空间，不利于整理分配需要连续空间的对象，而标记-整理算法会将标记需要回收的对象聚合回收，释放后的内存是一块连续的内存空间，根据不同回收器的回收算法而异

   ​		Full GC：当整个堆区都满时，触发full gc，minor gc + major gc

   什么时候会发生GC：

   GC的类型和常见的垃圾回收器：

   > 类型：
   >
   > Minor gc：eden区满时触发
   >
   > Major gc：old区满时触发，
   >
   > Full gc：minor gc + major gc
   >
   > 当准备要触发一次young GC时，如果发现统计数据之前young GC的平均晋升大小比目前old gen剩余的空间大，则不会触发young GC而是转为触发full GC
   >
   > 
   >
   > 实际上minor gc 和 major gc 是俗称，在hotspt jvm实现的Serial GC，Parallel GC，CMS，G1 GC中大致可以对应到某个区域（young gc 和 old gc）的算法组合，而Full GC的定义是相对明确的，就是针对整个新生代，老生代的全局范围的gc
   >
   > 
   >
   > Major GC通常是跟full GC是等价的，收集整个GC堆。但因为HotSpot VM发展了这么多年，外界对各种名词的解读已经完全混乱了

   常见垃圾收集器：

   > 1、Serial GC：全年代收集器，收集工作是单线程的，收集过程中，会进入[^stop-the-world]状态，单线程意味着精简的GC实现，简单，从年代的角度，新生代使用复制算法，老年代使用标记-整理算法，jvm的参数是：-XX:+UseSerialGC。
   >
   > 2、ParNew GC：新生代GC收集器，是Serial GC中 Serial Young的多线程版本，常见的场景是配合老年代的CMS GC工作，对应jvm参数：-XX:+UseConcMarkSweepGC  -XX:+UseParNewGC。
   >
   > 3、CMS（Concurrent Mark Sweep） GC，老年代GC收集器，基于标记-清除（Mark-Sweep）算法，尽量的减少停顿时间（stop-the-world的停顿现象在各种gc中都是存在的，各gc收集器需要尽量少的触发stw现象来维持系统的稳定运行），但是标记-清除算法意味着会存在内存碎片化的问题，所以难免会在长时间的运行等情况下发生full gc，导致stw，另外，并发的多线程会占用更多的cpu资源，和用户线程争夺资源。
   >
   > 4、Parallel GC，在jdk8以及之前是默认的gc收集器，算法和Serial GC相似，实现更为复杂，也是全年代gc收集器，新生代和老年代gc并行，常见的服务器环境更加高效（吞吐量优先），开启的jvm参数：-XX:+UseParallelGC
   >
   > 可以设置暂停时间或吞吐量，-XX:MaxGCPauseMillis=value；-XX:gun:=N //GC时间和用户时间比例 = 1/（N+1）
   >
   > 5、G1 GC：兼顾吞吐和停顿时间的GC实现，整体可看成是标记-整理算法，可以避免内存碎片化，jdk9之后的默认gc

   

   

   

   

   ---

   #### 脚注：

   

   [^stop-the-world]: 在GC过程中经常涉及到对对象的挪动（如对象在from和to之间复制，to和old之间复制，eden和from之间复制等），进而导致需要对对象引用进行更新，为了保证引用更新的正确性，java会暂停其他的线程，这种现象被称为"stop-the-world"。----java中一种全局暂停的现象，jvm挂起状态；全局停顿，所有java代码停止，native代码可以执行，但不能和jvm交互；多半由于jvm的GC引起，如：1、老年代空间不足。2、元数据空间不足（方法区中的空间）3、System.gc()。4、young gc晋升老年代的内存平均值大于老年代的剩余空间。5、连续的大对象需要分配内存。

   

   ​		

   