AQS 

1、是什么？

> AbstractQueueSynchronizer，同步队列，是一个同步工具，也是Lock用来实现线程同步的核心组件

2、AQS内部实现：

```java
AQS{
  volatile node head；// 头节点
  volatile node tail; //尾节点
  volatile int state； //同步状态，独占锁是0/1，>1表示共享锁，线程可重入,0表示锁未被线程占有
  volatile Thread exclusiveOwnerThread; //当前持有独占锁的线程
  node{
    node shared;// 共享节点
    node exclusive; // 独占节点
    volatile int waitStatus{
      CANCELLED(1) 当前节点的线程取消获取锁的请求
      SIGNAL(-1) 当前节点存在有效的后继节点，但前节点释放锁后需要唤醒后继节点持有锁
      CONDITION(-2) 当前节点在阻塞队列中
      PROPAGETE（-3） 共享锁模式下需要进行传播唤醒后继共享节点
    }
    volatile prev;前一个节点
    volatile next:下一个节点
    volatile thread:线程
    nextWaiter:条件队列中下一个等待的线程
  }
  ConditionOBject:{
    Node firstWaiter:条件队列第一个节点
    Node lastWaiter: 条件队列最后一个节点
  }
}
```



​	内部维护一个FIFO的双向链表，每个节点是阻塞的线程，当线程抢锁失败后会封装成node追加到AQS队列中去；当获得锁的线程释放锁后，会从队列中唤醒一个阻塞的节点

​	内部有一个共享资源state，AQS有两种功能：独占锁和共享重入锁，对于独占锁（比如ReentrantLock），判断能否获得锁还是要加入到等待队列的判断标识就是state=1或者是0，如果1是没有线程获取到锁，那么内部使用CAS判断state=1时，即可以获取到锁，否则加入到阻塞队列；而对于共享重入锁（比如Semaphore/CountDownLatch/ReentrantReadWriteLock），state设置一个同时可获取到锁的最大线程数，当前线程是否能获取到锁或者需要加入到阻塞队列的判断是state是否>=阀值，实现并发访问共享资源。

​	CAS+volatile来判断state的状态，从而获取锁或者加入到等待队列

<img src='https://cdn.jsdelivr.net/gh/MeqiangX/cloud-img@master/AQSstruct.png'></img>

3、AbstructQueuedSynchronizer源码分析

独占锁主要是acquire(获取锁)<---->release（释放锁）

共享锁主要是acquireShared(获取资源)<---->releaseShared(释放资源)

3.1、acquire(int)

> 独占锁模式下的线程获取共享资源的顶层入口，如果获取到，线程返回，否则加入到等待队列，lock()的底层实现，这里是没有中断时间，忽略中断的影响

```java
public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }

private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);
        return node;
    }

 private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // Must initialize
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }
// tryAcquire 尝试获取资源，这里是非公平锁，线程获取锁的时候会加塞一次(fifo队列)，队列中可能会其他的线程等待
// addWaiter() 将该线程加入到等待队列尾部，标记为独占模式，addWaiter中，加入到尾部的步骤如果没被其他线程打断，就可以正常的插入到尾部，如果被其他线程加塞，那么它会调用enq来通过CAS自旋(for+CAS)插入到队尾,确保插入的是尾部
//acquireQueued()使线程阻塞在等待队列中获取资源，一直到获取到资源才返回，如果在中间被中断过，则返回true,否则返回false
// 如果线程在等待中被中断，是不会处理的，返回true/false，然后再进行自我中断selfInterrupt()，将中断补上
```

3.2、tryAcquire(int)

> ```java
> protected boolean tryAcquire(int arg) {
>     throw new UnsupportedOperationException();
> }
> ```

tryLock的底层语意，这里没有定义成abstruct，因为aqs提供两种功能，独占和共享，独占只要tryAcquire--tryRelease，共享只要tryAcquireShared--tryRelaaseShared，如果都定义成abstract，那么任何实现的都要实现两套，所以作者站在开发者的角度，较少不必要的工作量。而第二这里没有具体的实现是因为AQS只是提供一个框架（模版方法），具体的获取/释放的实现需需要具体自定义同步器去实现。

3.3、acquireQueued(node,int)

> 通过tryAcquire()和addWaiter()，没有获取到资源的线程已经被加入到等待队列的队尾了，那么下一步就是要进入等待休息，直到资源被释放后唤醒自己，自己再拿到资源，就可以处理临界代码了，这和排队等待资源一样

```java
final boolean acquireQueued(final Node node, int arg) {
  			// 是否拿到资源
        boolean failed = true;
        try {
          	// 是否被中断
            boolean interrupted = false;
            for (;;) {
                final Node p = node.predecessor(); //获取前驱节点
                if (p == head && tryAcquire(arg))  //只有当前节点是第一个等待节点（除去head头空节点）才能竞争资源，这里又用了自旋，来获取资源
                    setHead(node); //获取到资源，设置首节点为当前节点
                    p.next = null; // help GC  --之前的头节点指向了当前节点，head为空,next置为空，方便gc回收，（没有引用标记）
                    failed = false; //成功获取资源
                    return interrupted; // 返回多次自旋过程中是否有被中断。
                }
          
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                  	//正常等待资源，并且是被中断唤醒的
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
    }

// 循环看前置节点的状态：
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
          // signal的状态表示，后继结点插入的时候会将前置节点的状态更新为signal，当前节点在等待前置节点唤醒它，这时候当前节点可以安全的休息等待 true
            /*
             * This node has already set status asking a release
             * to signal it, so it can safely park.
             */
            return true;
        if (ws > 0) {
          // >0的状态=CANCELLED（1），表示这个前置节点已经取消调度，timeout中断或者其他，这时候指针向前，查找上一个等待资源的线程节点
            /*
             * Predecessor was cancelled. Skip over predecessors and
             * indicate retry.
             */
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
          //除了cancelled取消了排队，或者signal找到了前一个等待节点，其余的都是在初始化(0)，或者condition(-2)，或者propagate(-3)共享模式下的唤醒（可唤醒多个，可能不仅是后一个，可能是后几个）
          // 那都是正常等待的，将信号置为signal,在处理完后好通知唤醒后置节点，在自旋下次进来的时候，走true,但是注意，这里没有进行一个cas的自旋，也就是有可能其他线程改变了这个状态，那么将会更新失败，下次再次判断
            /*
             * waitStatus must be 0 or PROPAGATE.  Indicate that we
             * need a signal, but don't park yet.  Caller will need to
             * retry to make sure it cannot acquire before parking.
             */
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;
    }
//这个方法的目的就是为了让当前线程节点的前置节点的状态为signal，这样就可以好好的去休息了，（&&后面的park（）,将线程挂起），同时尝试看有没有轮到自己拿到资源

//挂起
private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);
        return Thread.interrupted();
    }
  // park是unsafe的本地操作，线程进入waiting状态，有两种方式唤醒线程：1、unpark()正常唤醒 2、interrupt()中断唤醒，LockSupport.park(this);进入等待，return Thread.interrupted(); 看是否是被中断的
```

到这里，一个线程节点等待获取资源的流程就出来了：

1、进入队尾，找到最后一个前置等待节点signal，进入安全休息点，等待

2、park() 进入等待状态，等待unpack()或者interrupt()唤醒自己

3、唤醒后，看是否具备资格能拿到资源，如果拿到，头节点指向自己，并返回是否被中断过；如果没拿到，则 重复1

<img src="https://cdn.jsdelivr.net/gh/MeqiangX/cloud-img@master/acquire.png"></img>

> acquireShared和releaseShared的流程和 独占功能的流程类似，只是共享情况的获取资源不需要state=0全部释放完，因为独占的情况同一时间只能一个线程获取全部资源，其他线程要获取只能等获取到的线程将资源全部释放，就是如果有一堆人在商场娃娃机前排队，但是娃娃机却只有一台，如果将state类比成游戏次数，每个人虽然有三次机会，但是同时间只能有一个人在抓，并且抓完三次才能被释放，然后在队伍中取下一个等待就绪的人
>
> 共享锁是每个线程可以获取到不同的资源所需，如果state>0，并且满足剩余的可用资源 > 所需资源，那么就可以有竞争的资格，当前线程节点获取到资源后，如果还有可用资源，会继续向后唤醒其他等待线程，因为同步队列是严格按照顺序的，所以就算下一个就绪线程节点所需的资源大于state，后面就算有满足state>当前线程所需资源数的线程节点，也是会park阻塞，直到被unpark或中断

​	AQS顶层已经通过模板方法实现了线程的排队，等待，唤醒，以及执行的流程框架，开发人员只需要在这个基础上实现流程中的业务资源共享控制方法即可（tryAcquire，tryRelease，tryAcquireShared，tryReleaseShared），一般自定义同步器都会在内部定义一个实现aqs的工具内部类，同步器本身实现接口，对外提供服务，

​	应用例子参考：

​	mutex、reentrant lock、countdown latch、semaphore



练习：自己实现一个自定义同步器