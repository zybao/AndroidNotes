# 多线程

线程状态：

新建、就绪(Runnable)、阻塞(Blocked)、死亡(Dead)
## synchronized

## interrupt
当Thread收到interrupt信号时，可能的两种结果：要么其线程对象中的isinterrupt属性被置为true；要么抛出InterruptedException异常。注意，如果抛出了InterruptedException异常，那么其isinterrupt属性不会被置为true。

* thread.isInterrupted()和Thread.interrupted()的区别
```java
    public static boolean interrupted() {
        return currentThread().isInterrupted(true);
    }

    public boolean isInterrupted() {
        return isInterrupted(false);
    }

    /**
     * Tests if some Thread has been interrupted.  The interrupted state
     * is reset or not based on the value of ClearInterrupted that is
     * passed.
     */
    private native boolean isInterrupted(boolean ClearInterrupted);
```

可以看到，对象方法的thread.isInterrupted()和静态方法的Thread.interrupted()都是调用的JNI底层的isInterrupted()方法。但是区别在于这个ClearInterrupted参数，前者传入的false，后者传入的是true。相信各位读者都已经猜出其中的含义了，ClearInterrupted参数向操作系统层指明是否在获取状态后将当前线程的isInterrupt属性重置为（或者叫恢复，或者叫清除）false。

这就意味着当某个线程的isInterrupt属性成功被置为true后，如果您使用对象方法thread.isInterrupted()获取值，无论您获取多少次得到的返回值都是true；但是如果您使用静态方法Thread.interrupted()获取值，那么只有第一次获取的结果是true，随后线程的isInterrupt属性将被恢复成false，后续无论使用Thread.interrupted()调用还是使用thread.isInterrupted()调用，获取的结果都是false。

## wait(), notify(), notifyAll()
wait()的作用是让当前线程进入等待状态，同时，wait()也会让当前线程释放它所持有的锁。而notify()和notifyAll()的作用，则是唤醒当前对象上的等待线程；notify()是唤醒单个线程，而notifyAll()是唤醒所有的线程。

## yield()
yield()的作用是让步。它能让当前线程由“运行状态”进入到“就绪状态”，从而让其它具有相同优先级的等待线程获取执行权；但是，并不能保证在当前线程调用yield()之后，其它具有相同优先级的线程就一定能获得执行权；也有可能是当前线程又进入到“运行状态”继续运行！

我们知道，wait()的作用是让当前线程由“运行状态”进入“等待(阻塞)状态”的同时，也会释放同步锁。而yield()的作用是让步，它也会让当前线程离开“运行状态”。它们的区别是：

* wait()是让线程由“运行状态”进入到“等待(阻塞)状态”，而不yield()是让线程由“运行状态”进入到“就绪状态”。

* wait()是会线程释放它所持有对象的同步锁，而yield()方法不会释放锁。

## join
* join: 一直等到目标线程运行结束，当前调用线程才能继续执行
* join(millis): 调用线程等待millis毫秒后，无论目标线程执行是否完成，当前调用线程都会继续执行
* 调用线程等待 millis毫秒 + nanos 纳秒 时间后，无论无论目标线程执行是否完成，当前调用线程都会继续执行；实际上这个join方法的描述并不准确：第二个参数nanos只是一个参考值（修正值），且只有大于等于500000时，第二个参数才会起作用（纳秒是一秒的十亿分之一）
```java
public final synchronized void join(long millis, int nanos) throws InterruptedException {
    if (millis < 0) {
        throw new IllegalArgumentException("timeout value is negative");
    }

    if (nanos < 0 || nanos > 999999) {
        throw new IllegalArgumentException("nanosecond timeout value out of range");
    }

    if (nanos >= 500000 || (nanos != 0 && millis == 0)) {
        millis++;
    }

    join(millis);
}
```

## sleep()
Causes the **currently executing thread** to sleep (temporarily cease
execution) for the specified number of milliseconds, subject to
the precision and accuracy of system timers and schedulers. The thread
does not lose ownership of any monitors.

# 线程池
```java
   /**
     * Creates a new {@code ThreadPoolExecutor} with the given initial
     * parameters.
     *
     * @param corePoolSize the number of threads to keep in the pool, even
     *        if they are idle, unless {@code allowCoreThreadTimeOut} is set
     * @param maximumPoolSize the maximum number of threads to allow in the
     *        pool
     * @param keepAliveTime when the number of threads is greater than
     *        the core, this is the maximum time that excess idle threads
     *        will wait for new tasks before terminating.
     * @param unit the time unit for the {@code keepAliveTime} argument
     * @param workQueue the queue to use for holding tasks before they are
     *        executed.  This queue will hold only the {@code Runnable}
     *        tasks submitted by the {@code execute} method.
     * @param threadFactory the factory to use when the executor
     *        creates a new thread
     * @param handler the handler to use when execution is blocked
     *        because the thread bounds and queue capacities are reached
     * @throws IllegalArgumentException if one of the following holds:<br>
     *         {@code corePoolSize < 0}<br>
     *         {@code keepAliveTime < 0}<br>
     *         {@code maximumPoolSize <= 0}<br>
     *         {@code maximumPoolSize < corePoolSize}
     * @throws NullPointerException if {@code workQueue}
     *         or {@code threadFactory} or {@code handler} is null
     */
public ThreadPoolExecutor(int corePoolSize,
                            int maximumPoolSize,
                            long keepAliveTime,
                            TimeUnit unit,
                            BlockingQueue<Runnable> workQueue,
                            ThreadFactory threadFactory,
                            RejectedExecutionHandler handler)
```

* corePoolSize（线程池的基本大小）：当提交一个任务到线程池时，线程池会创建一个线程来执行任务，即使其他空闲的基本 线程能够执行新任务也会创建线程，等到需要执行的任务数大于线程池基本大小时就不再创建。如果调用了线程池的prestartAllCoreThreads方法，线程池会提前创建并启动所有基本线程。

* maximumPoolSize（线程池最大大小）：线程池允许创建的最大线程数。如果队列满了，并且已创建的线程数小于最大线程数，则线程池会再创建新的线程执行任务。值得注意的是如果使用了无界的任务队列这个参数就没什么效果。

* keepAliveTime（线程活动保持时间）：线程池的工作线程空闲后，保持存活的时间。所以如果任务很多，并且每个任务执行的时间比较短，可以调大这个时间，提高线程的利用率。

* TimeUnit（线程活动保持时间的单位）：可选的单位有天（DAYS），小时（HOURS），分钟（MINUTES），毫秒(MILLISECONDS)，微秒(MICROSECONDS, 千分之一毫秒)和毫微秒(NANOSECONDS, 千分之一微秒)。

* workQueue（任务队列）：用于保存等待执行的任务的阻塞队列。 可以选择以下几个阻塞队列。

    ArrayBlockingQueue：是一个基于数组结构的有界阻塞队列，此队列按 FIFO（先进先出）原则对元素进行排序。
    
    LinkedBlockingQueue：一个基于链表结构的阻塞队列，此队列按FIFO （先进先出） 排序元素，吞吐量通常要高于
    ArrayBlockingQueue。静态工厂方法Executors.newFixedThreadPool()使用了这个队列。
    
    SynchronousQueue：一个不存储元素的阻塞队列。每个插入操作必须等到另一个线程调用移除操作，否则插入操作一直处于阻塞状态，吞吐量通常要高于LinkedBlockingQueue，静态工厂方法Executors.newCachedThreadPool使用了这个队列。
    
    PriorityBlockingQueue：一个具有优先级的无限阻塞队列。

* RejectedExecutionHandler（饱和策略）：当队列和线程池都满了，说明线程池处于饱和状态，那么必须采取一种策略处理提交的新任务。这个策略默认情况下是AbortPolicy，表示无法处理新任务时抛出异常。以下是JDK1.5提供的四种策略。

    AbortPolicy：直接抛出异常。
    
    CallerRunsPolicy：只用调用者所在线程来运行任务。
    
    DiscardOldestPolicy：丢弃队列里最近的一个任务，并执行当前任务。
    
    DiscardPolicy：不处理，丢弃掉。
    
    当然也可以根据应用场景需要来实现RejectedExecutionHandler接口自定义策略。如记录日志或持久化不能处理的任务。

## execute方法和submit方法的区别

* execute方法：所有实现了Runnable接口的任务都可以使用execute方法进行提交。而实现了Runnable接口的任务，并没有提供任何“标准”的方式为我们返回任务的执行结果（这是我们还没有讲到的知识点）。线程在线程池中运行结束了，就结束了。所以，使用execute方法提交的任务，程序员并不能在任务执行完成后，获得一个“标准”的执行结果。

* submit方法：submit方法提交的任务是实现了Callable接口的任务（这是我们还没有讲到的知识点）。Callable接口的特性是，在其运行完成后，会返回一个“标准”的执行结果。

## Semaphore：信号量

Semaphore信号量，是concurrent包的一个重要工具类，它通过申请和回收“证书”，实现多个线程对同一资源的访问控制。具体的做法是，某个线程在访问某个（可能出现资源抢占的）资源的时候，首先向Semaphore对象申请“证书”，如果没有拿到“证书”就一直阻塞；当拿到“证书”后，线程就解除阻塞状态，然后访问资源；在完成资源操作后，再向Semaphore对象归还“证书”

* 申请/获取证书：

    void acquire()：从此信号量获取一个许可，在Semaphore能够提供一个许可前，当前线程将一直阻塞等待。如果在等待过程中，当前线程收到了interrupt信号，那么将抛出InterruptedException异常。

    void acquire(permits)：从此信号量获取permits个许可，在Semaphore能够提供permits个许可前，当前线程将一直阻塞等待。如果在等待过程中，当前线程收到了interrupt信号，那么将抛出InterruptedException异常。

    void acquireUninterruptibly()：从此信号量获取一个许可，在Semaphore能够提供一个许可前，当前线程将一直阻塞等待。使用这个方法获取许可时，不会受到线程interrupt信号的影响。

    void acquireUninterruptibly(permits)：从此信号量获取permits个许可，在Semaphore能够提供permits个许可前，当前线程将一直阻塞等待。使用这个方法获取许可时，不会受到线程interrupt信号的影响。

    boolean tryAcquire()：从此信号量获取一个许可，如果无法获取，线程并不会阻塞在这里。如果获取到了许可，则返回true，其他情况返回false。

    boolean tryAcquire(permits)：从此信号量获取permits个许可，如果无法获取，线程并不会阻塞在这里。如果获取到了许可，则返回true，其他情况返回false。

    boolean tryAcquire(int permits, long timeout, TimeUnit unit)：从此信号量获取permits个许可，如果无法获取，则当前线程等待设定的时间。如果超过等待时间后，还是没有拿到许可，则解除等待继续执行。如果获取到了许可，则返回true，其他情况返回false。

* 证书状态：

    int availablePermits()：返回此信号量中当前可用的许可数。

    int getQueueLength()：返回正在等待获取的线程的估计数目。该值仅是估计的数字，因为在此方法遍历内部数据结构的同时，线程的数目可能动态地变化。此方法用于监视系统状态，不用于同步控制。

    boolean hasQueuedThreads()：查询是否有线程正在等待获取。注意，因为同时可能发生取消，所以返回 true 并不保证有其他线程等待获取许可。此方法主要用于监视系统状态。

    boolean isFair()：如果此信号量的公平设置为 true，则返回 true。

* 释放/返还证书：

    void release()：释放一个许可，将其返回给信号量。最好将这个方法的调用，放置在finally程序块中执行。

    void release(permits)：释放给定数目的许可，将其返回到信号量。最好将这个方法的调用，放置在finally程序块中执行。

* fair：公平与非公平

    Semaphore一共有两个构造函数，分别是：Semaphore(int permits)和Semaphore(int permits, boolean fair)；permits是指由Semaphore信号量控制的“证书”数量。fair参数是设置这个信号量对象的工作方式。

当fair参数为true时，信号量将以“公平方式”运行。即首先申请证书，并进入阻塞状态的线程，将有权利首先获取到证书；当fair参数为false时，信号量对象将不会保证“先来先得”。默认情况下，Semaphore采用“非公平”模式运行。


## CountDownLatch：同步器


# Callable<V>

```java
/**
 * A task that returns a result and may throw an exception.
 * Implementors define a single method with no arguments called
 * {@code call}.
 *
 * <p>The {@code Callable} interface is similar to {@link
 * java.lang.Runnable}, in that both are designed for classes whose
 * instances are potentially executed by another thread.  A
 * {@code Runnable}, however, does not return a result and cannot
 * throw a checked exception.
 *
 * <p>The {@link Executors} class contains utility methods to
 * convert from other common forms to {@code Callable} classes.
 *
 * @see Executor
 * @since 1.5
 * @author Doug Lea
 * @param <V> the result type of method {@code call}
 */
@FunctionalInterface
public interface Callable<V> {
    /**
     * Computes a result, or throws an exception if unable to do so.
     *
     * @return computed result
     * @throws Exception if unable to compute a result
     */
    V call() throws Exception;
}
```

```java
    ExecutorService executorService = Executors.newCachedThreadPool();
    Callable<String> callable = new Callable<String>() {
        @Override
        public String call() throws Exception {
            return "Result";
        }
    };

    Future<String> future = executorService.submit(callable);
    String result = future.get();
    executorService.shutdown();
```

目前Callable定义的线程任务，只能放入线程池中，由线程池中的任务进行执行。如果不想使用线程池管理执行，则可用FutureTask，如：
```java
    new Thread(new FutureTask<V>(callableThread)).start();
```

Future接口中的get方法，将会是当前线程进入阻塞状态。直到目标线程执行完毕，并且得到目标线程的返回结果。

## ReentrantLock

### tryAcquire()
公平锁的tryAcquire()在ReentrantLock.java的FairSync类中实现，源码如下：
```java
        protected final boolean tryAcquire(int acquires) {
            // 获取“当前线程”
            final Thread current = Thread.currentThread();
            // 获取“独占锁”的状态
            int c = getState();
            // c=0意味着“锁没有被任何线程锁拥有”
            if (c == 0) {
                // 若“锁没有被任何线程锁拥有”，
                // 则判断“当前线程”是不是CLH队列中的第一个线程线程，
                // 若是的话，则获取该锁，设置锁的状态，并切设置锁的拥有者为“当前线程”。
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            else if (current == getExclusiveOwnerThread()) {
                // 如果“独占锁”的拥有者已经为“当前线程”，
                // 则将更新锁的状态。
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
```

说明：根据代码，我们可以分析出，tryAcquire()的作用就是尝试去获取锁。注意，这里只是尝试！
尝试成功的话，返回true；尝试失败的话，返回false，后续再通过其它办法来获取该锁。后面我们会说明，在尝试失败的情况下，是如何一步步获取锁的。

### hasQueuedPredecessors()

```java
   /**
     * Queries whether any threads have been waiting to acquire longer
     * than the current thread.
     *
     * <p>An invocation of this method is equivalent to (but may be
     * more efficient than):
     *  <pre> {@code
     * getFirstQueuedThread() != Thread.currentThread() &&
     * hasQueuedThreads()}</pre>
     *
     * <p>Note that because cancellations due to interrupts and
     * timeouts may occur at any time, a {@code true} return does not
     * guarantee that some other thread will acquire before the current
     * thread.  Likewise, it is possible for another thread to win a
     * race to enqueue after this method has returned {@code false},
     * due to the queue being empty.
     *
     * <p>This method is designed to be used by a fair synchronizer to
     * avoid <a href="AbstractQueuedSynchronizer#barging">barging</a>.
     * Such a synchronizer's {@link #tryAcquire} method should return
     * {@code false}, and its {@link #tryAcquireShared} method should
     * return a negative value, if this method returns {@code true}
     * (unless this is a reentrant acquire).  For example, the {@code
     * tryAcquire} method for a fair, reentrant, exclusive mode
     * synchronizer might look like this:
     *
     *  <pre> {@code
     * protected boolean tryAcquire(int arg) {
     *   if (isHeldExclusively()) {
     *     // A reentrant acquire; increment hold count
     *     return true;
     *   } else if (hasQueuedPredecessors()) {
     *     return false;
     *   } else {
     *     // try to acquire normally
     *   }
     * }}</pre>
     *
     * @return {@code true} if there is a queued thread preceding the
     *         current thread, and {@code false} if the current thread
     *         is at the head of the queue or the queue is empty
     * @since 1.7
     */
    public final boolean hasQueuedPredecessors() {
        // The correctness of this depends on head being initialized
        // before tail and on head.next being accurate if the current
        // thread is first in queue.
        Node t = tail; // Read fields in reverse initialization order
        Node h = head;
        Node s;
        return h != t &&
            ((s = h.next) == null || s.thread != Thread.currentThread());
    }
```
通过代码能分析出hasQueuedPredecessors() 是通过判断"当前线程"是不是在CLH队列的队首，来返回AQS中是不是有比“当前线程”等待更久的线程。


# 高级类
## ThreadLocal类

用处：保存线程的独立变量。对一个线程类（继承自Thread)
当使用ThreadLocal维护变量时，ThreadLocal为每个使用该变量的线程提供独立的变量副本，所以每一个线程都可以独立地改变自己的副本，而不会影响其它线程所对应的副本。常用于用户登录控制，如记录session信息。

实现：每个Thread都持有一个TreadLocalMap类型的变量（该类是一个轻量级的Map，功能与map一样，区别是桶里放的是entry而不是entry的链表。功能还是一个map。）以本身为key，以目标为value。
主要方法是get()和set(T a)，set之后在map里维护一个threadLocal -> a，get时将a返回。ThreadLocal是一个特殊的容器。

## 原子类（AtomicInteger、AtomicBoolean……）

如果使用atomic wrapper class如atomicInteger，或者使用自己保证原子的操作，则等同于synchronized

```java
//返回值为boolean
AtomicInteger.compareAndSet(int expect,int update)
```

该方法可用于实现乐观锁，考虑文中最初提到的如下场景：a给b付款10元，a扣了10元，b要加10元。此时c给b2元，但是b的加十元代码约为：

```java
if(b.value.compareAndSet(old, value)){
   return ;
}else{
   //try again
   // if that fails, rollback and log
}
```

**AtomicReference**

对于AtomicReference 来讲，也许对象会出现，属性丢失的情况，即oldObject == current，但是oldObject.getPropertyA != current.getPropertyA。

这时候，AtomicStampedReference就派上用场了。这也是一个很常用的思路，即加上版本号

## 容器类

## 管理类

了解到的值得一提的管理类：ThreadPoolExecutor和 JMX框架下的系统级管理类 ThreadMXBean
