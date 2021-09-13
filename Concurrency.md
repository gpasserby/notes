### CountDownLatch

#### CountDownLatch是什么？

在CountDownLatch官方源码注释中描述如下：

```
* A synchronization aid that allows one or more threads to wait until
* a set of operations being performed in other threads completes.
```

它是一个同步工具类，它允许一个或多个线程等待，直到在其他线程中执行的一组操作完成为止。可以理解为一场跑步比赛中，裁判需要等所有的运动员都到达了终点，才能进行下一步的操作，否则一直等待。

#### CountDownLatch工作原理

CountDownLatch是通过一个计数器来实现的，计数器的初始值为线程的数量（即实例化时所赋予的值）。每当一个线程完成了自己的任务后，调用CountDownLatch.countDown方法后，计数器的值就会减1，同时该线程进入等待中。当计数器值到达0时，它表示所有的线程已经完成了任务，然后在其上等待的线程就可以继续往下执行。

CountDownLatch的构造函数中的count就是闭锁需要等待的线程数量。这个值只能被设置一次，而且不能重新设置。

主线程必须在启动其他线程后调用CountDownLatch.await方法，这样主线程就会在这个方法上阻塞，直到其他线程完成各自任务。

其他线程完成任务后必须各自通知CountDownLatch对象，使其调用countDown方法。当count值为0时，主线程就能通过await方法恢复自己的任务。

#### 使用场景

1. **实现最大的并行性**：有时我们想同时启动多个线程，实现最大程度的并行性。例如，我们想测试一个单例类。如果我们创建一个初始计数为1的CountDownLatch，并让所有线程都在这个锁上等待，那么我们可以很轻松地完成测试。我们只需调用 一次countDown()方法就可以让所有的等待线程同时恢复执行。

2. **开始执行前等待n个线程完成各自任务**：例如应用程序启动类要确保在处理用户请求前，所有N个外部系统已经启动和运行了。

3. **死锁检测：**一个非常方便的使用场景是，你可以使用n个线程访问共享资源，在每次测试阶段的线程数目是不同的，并尝试产生死锁。

#### 使用示例

```java
public static void main(String[] args) {
    CountDownLatch countDownLatch = new CountDownLatch(10);

    IntStream.range(0, 10).forEach(i -> new Thread(() -> {
        try {
            Thread.sleep(2000);
            System.out.println("hello world!");
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            countDownLatch.countDown();
        }
    }).start());

    System.out.println("启动子线程完毕！");

    try {
        countDownLatch.await();
    } catch (InterruptedException e) {
        e.printStackTrace();
    }

    System.out.println("主线程执行完毕！");
}
```

运行结果：

>启动子线程完毕！
>hello world!
>hello world!
>hello world!
>hello world!
>hello world!
>hello world!
>hello world!
>hello world!
>hello world!
>hello world!
>主线程执行完毕！

示例运行可以看出，主线程在等待所有子线程输出完成之后，再进行输出。

#### 底层实现

先看看CountDownLatch的构造函数

```java
public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0"); // 确保传入的count大于0
    this.sync = new Sync(count); // 实例化Sync对象
}
```

Sync是位于CountDownLatch的静态内部类，再看看这个Sync是何方神圣

```java
private static final class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = 4982264981922014374L;

    Sync(int count) {
        setState(count);
    }

    int getCount() {
        return getState();
    }

    protected int tryAcquireShared(int acquires) {
        return (getState() == 0) ? 1 : -1;
    }

    protected boolean tryReleaseShared(int releases) {
        // Decrement count; signal when transition to zero
        for (;;) {
            int c = getState();
            if (c == 0)
                return false;
            int nextc = c-1;
            if (compareAndSetState(c, nextc))
                return nextc == 0;
        }
    }
}
```

Sync继承了AbstractQueuedSynchronizer(即AQS)抽象类, 在Sync的构造方法中,调用了setState方法,可以视作初始化了一个标记来记录当前计数器的数量。

再看看CountDownLatch的两个核心方法，CountDownLatch.await以及CountDownLatch.countDown.

```java
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
```

```java
public void countDown() {
    sync.releaseShared(1);
}
```

调用了Sync的方法，再进去看看

```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```

tryReleaseShared经过Sync的重写，看看是怎么实现的

```java
protected boolean tryReleaseShared(int releases) {
    // Decrement count; signal when transition to zero
    for (;;) {
        // 获取状态值
        int c = getState();
        if (c == 0)
            return false;
        // 调用countDown即 -1 
        int nextc = c-1;
        // 通过CAS操作更新值，返回countDownLatch是否已经为0
        if (compareAndSetState(c, nextc))
            return nextc == 0;
    }
}
```

当 next 值等于0后，表示countDownLatch已经为0，调用doReleaseShared方法释放等待中的线程，让其继续往下执行。

```java
private void doReleaseShared() {
    /*
     * Ensure that a release propagates, even if there are other
     * in-progress acquires/releases.  This proceeds in the usual
     * way of trying to unparkSuccessor of head if it needs
     * signal. But if it does not, status is set to PROPAGATE to
     * ensure that upon release, propagation continues.
     * Additionally, we must loop in case a new node is added
     * while we are doing this. Also, unlike other uses of
     * unparkSuccessor, we need to know if CAS to reset status
     * fails, if so rechecking.
     */
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}
```

---------------------



### CyclicBarrier

#### CyclicBarrier是什么？

在CyclicBarrier官方源码注释中描述如下：

```
* A synchronization aid that allows a set of threads to all wait for
* each other to reach a common barrier point.
```

也是一个同步工具类，它是让一组线程相互等待进入barrier状态，然后这组线程再执行。 可以理解为在一场跑步比赛中，已经准备就绪的运动员需要等待其他运动员准备就绪，所有运动员就绪后，才能开始比赛跑步。 

#### CyclicBarrier工作原理

相对于CountDownLatch是**一次性对象，一旦进入终止状态，就不能被重置**，CyclicBarrier可以反复使用。CyclicBarrier类似于闭锁，与**闭锁的关键区别在于，闭锁用于等待事件，栅栏用于等待其他线程**，其作用是让一组线程到达一个屏障（也可以叫同步点）时被阻塞，直到最后一个线程到达屏障时，屏障才会开门，所有被屏障拦截的线程才会继续运行。

#### 使用场景

当并行计算时，下一步的运算需要当前并行运算的结果进行汇总处理的结果，就需要使用CyclicBarrier等待其他线程的运算结果输出。

#### 使用示例

```java
public static void main(String[] args) {
    CyclicBarrier cyclicBarrier = new CyclicBarrier(10);

    IntStream.range(0,10).forEach(i -> new Thread(() -> {
        try {
            Thread.sleep((long) (Math.random() * 10000));
            System.out.println("Thread" + i + " ready！");
            cyclicBarrier.await();
            System.out.println("All ready! Thread" + i + " go!!");
        } catch (InterruptedException | BrokenBarrierException e) {
            e.printStackTrace();
        }
    }).start());
}
```

运行结果：

>Thread2 ready！
>Thread3 ready！
>Thread1 ready！
>Thread5 ready！
>Thread9 ready！
>Thread4 ready！
>Thread8 ready！
>Thread0 ready！
>Thread7 ready！
>Thread6 ready！
>All ready! Thread6 go!!
>All ready! Thread2 go!!
>All ready! Thread3 go!!
>All ready! Thread9 go!!
>All ready! Thread8 go!!
>All ready! Thread5 go!!
>All ready! Thread1 go!!
>All ready! Thread7 go!!
>All ready! Thread0 go!!
>All ready! Thread4 go!!

示例运行可以看出，当线程到达之后，调用cyclicBarrier.await方法，使其等待其它线程到达。当全部线程到达后，继续往下执行。

#### 底层实现

先看看CyclicBarrier的构造函数

```java
public CyclicBarrier(int parties) {
    this(parties, null);
}

public CyclicBarrier(int parties, Runnable barrierAction) {
    if (parties <= 0) throw new IllegalArgumentException();
    this.parties = parties;
    this.count = parties;
    this.barrierCommand = barrierAction;
}
```

参数parties为拦截的线程的数量，barrierAction为当最后一个线程到达屏障时，执行的动作（等待的线程恢复运行之前执行的动作）。由此，我们可以将刚刚的示例修改成如下版本：

```java
public static void main(String[] args) {
    CyclicBarrier cyclicBarrier = new CyclicBarrier(10, () -> {
        // 在最后一个线程到达栅栏时输出
        System.out.println("All ready!!!");
    });

    // CyclicBarrier是可以重复使用的。
    for (int j = 0; j < 3; j++) {
        IntStream.range(0,10).forEach(i -> new Thread(() -> {
            try {
                Thread.sleep((long) (Math.random() * 10000));
                System.out.println("Thread" + i + " ready！");
                cyclicBarrier.await();
                System.out.println("Thread" + i + " go!!");
            } catch (InterruptedException | BrokenBarrierException e) {
                e.printStackTrace();
            }
        }).start());
    }
}
```

输出结果与上面的示例类似。再来看看CyclicBarrier.await方法。

```java
public int await() throws InterruptedException, BrokenBarrierException {
    try {
        return dowait(false, 0L);
    } catch (TimeoutException toe) {
        throw new Error(toe); // cannot happen
    }
}
```

```java
private int dowait(boolean timed, long nanos)
    throws InterruptedException, BrokenBarrierException, TimeoutException {
    final ReentrantLock lock = this.lock;
    // 获取锁，使下面的操作不会在并发情况下，发生错误。
    lock.lock();
    try {
        final Generation g = generation;

        if (g.broken)
            throw new BrokenBarrierException();

        if (Thread.interrupted()) {
            breakBarrier();
            throw new InterruptedException();
        }
        // --count并不是原子性操作，如果没有上锁的话，会发生并发错误。
        int index = --count;
        if (index == 0) {  // tripped 为0表示所有线程已经到达栅栏处
            boolean ranAction = false;
            try {
                // CyclicBarrier实例化时传入的Runnable。
                final Runnable command = barrierCommand;
                if (command != null)
                    // 执行Runnable
                    command.run();
                ranAction = true;
                // 产生新的分代
                nextGeneration();
                return 0;
            } finally {
                if (!ranAction)
                    breakBarrier();
            }
        }

        // loop until tripped, broken, interrupted, or timed out
        for (;;) {
            try {
                if (!timed)
                    // trip为lock产生的Condition，trip.await使线程等待。
                    trip.await();
                else if (nanos > 0L)
                    nanos = trip.awaitNanos(nanos);
            } catch (InterruptedException ie) {
                if (g == generation && ! g.broken) {
                    breakBarrier();
                    throw ie;
                } else {
                    // We're about to finish waiting even if we had not
                    // been interrupted, so this interrupt is deemed to
                    // "belong" to subsequent execution.
                    Thread.currentThread().interrupt();
                }
            }

            // 异常处理
            if (g.broken)
                throw new BrokenBarrierException();

            if (g != generation)
                return index;

            if (timed && nanos <= 0L) {
                breakBarrier();
                throw new TimeoutException();
            }
        }
    } finally {
        // 解锁
        lock.unlock();
    }
}
```

其中调用了nextGeneration，看看它的实现。

```java
private void nextGeneration() {
    // signal completion of last generation
    // 发送信号，唤醒所有等待的线程继续往下执行。
    trip.signalAll();
    // set up next generation
    // 重置count值，使CyclicBarrier可以重复使用。
    count = parties;
    // 开启新的分代。
    generation = new Generation();
}
```

### CountDownLatch与CyclicBarrier区别

1. CountDownLatch的计数器只能使用一次，而CyclicBarrier的计数器可以使用reset()方法重置。所以CyclicBarrier能处理更为复杂的业务场景。例如，如果计算发生错误，可以重置计数器，并让线程重新执行一次

2. CyclicBarrier还提供其他有用的方法，比如getNumberWaiting方法可以获得CyclicBarrier阻塞的线程数量。isBroken()方法用来了解阻塞的线程是否被中断。

3. CountDownLatch倾向于一个线程等多个线程，CyclicBarrier倾向于多个线程互相等待。



### CAS

synchronized关键字与lock等锁机制都是悲观锁：无论做何种操作，首先都需要先上锁，接下来再去执行后续操作，从而确保了接下来的所有操作都是由当前这个线程来执行的。

乐观锁：线程在操作之前不会做任何预先的处理，而是直接去执行，当在最后执行变量更新的时候，当前线程需要有一种机制来确保当前被操作的变量时没有被其他线程修改的。CAS时乐观锁的一种极为重要的实现方式。

CAS（Compare And Swap）

比较与交换：这是一个不断循环的过程，一直到变量值被修改成功为止。CAS本身是由硬件指令来提供支持的，因此CAS时可以确保变量操作的原子性的。

对于CAS来说，其操作数主要涉及到如下三个：

1.需要被操作的内存值V

2.需要进行比较的值A

3。需要进行写入的值B

只有当V==A的时候，CAS才会通过原子操作的手段来将V的值更新为B

```java
public class AtomicInteger extends Number implements java.io.Serializable {
    private static final long serialVersionUID = 6214790243416807050L;

    // setup to use Unsafe.compareAndSwapInt for updates
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    // 内存偏移地址
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
    
    public final int getAndSet(int newValue) {
        return unsafe.getAndSetInt(this, valueOffset, newValue);
    }
}
```

```java
// Unsafe
public final int getAndSetInt(Object var1, long var2, int var4) {
    int var5;
    do {
        var5 = this.getIntVolatile(var1, var2);
    //  var1：当前对象的引用(内存地址) var2：要修改的变量的在var1对象中的内存偏移地址
    //  var5预期的值  var4写入的值
    } while(!this.compareAndSwapInt(var1, var2, var5, var4));

    return var5;
}
```

关于CAS的限制或问题

1.循环开销问题：并发量大的情况下会导致线程一直自旋。

2.只能保证一个变量的原子操作：可以通过AtomicReference来实现对多个变量的原子操作

3.ABA问题：1 -> 3 -> 1，通过添加版本号。



### Future

位于J.U.C包。



### ThreadLocal

本质上，ThreadLocal是通过空间来换取时间，从而实现每个线程当中都会有一个变量的副本，这样每个线程都会操作该副本，从而完全规避了多线程并发问题。

Java中存在四种类型的引用：

1.强引用（strong）

2.软引用（soft）

3.弱引用（weak）

4.虚引用（phantom）

防止内存泄漏

### AQS

AQS（AbstractQueuedSynchronizer 即抽象对象同步器）

  

