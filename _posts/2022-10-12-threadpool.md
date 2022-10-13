---
layout: post
title: 深度剖析：线程池运行原理及常见面试题，看完这篇文章彻底懂了！
category: java
tags: [java,interview]
excerpt:  深度剖析：线程池运行原理及常见面试题，看完这篇文章彻底懂了！
keywords: 线程池,线程,java线程,阻塞队列,线程中断
---

你好，我是猿java，一个践行终身学习的程序员。

今天我们开门见山，直奔主题。

> 申明：本文基于 jdk-11.0.15

线程池是 JDK 1.5版本开始引入，由大牛 Doug Lea实现，源码类为 java.util.concurrent.ThreadPoolExecutor。

内容大纲：
![img.png](https://www.yuanjava.cn/assets/md/java/threadpool-outline.png)

## 线程池核心属性

下面给出一张创建线程池最底层构造器源码截图（其他方式创建线程池最终都是调用该构造器）：

![img.png](https://www.yuanjava.cn/assets/md/java/threadpool.png)

通过 ThreadPoolExecutor源码截图我们可以得出线程池中几个核心的概念：corePoolSize、maximumPoolSize、workQueue、keepAliveTime、threadFactory、handler。

理解了这几个核心概念，线程池基本掌握了50%，下面分别对每个属性进行讲解。

### corePoolSize

核心线程数，当提交新任务，如果线程池中的线程数小于 corePoolSize，即便其他的工作线程处于空闲状态，线程池也会创建一个新线程来处理该任务；

### maximumPoolSize

最大线程数，线程池中允许创建的最大线程数阈值。当新任务被提交时：

- 如果线程池中的线程数 countWorker大于 corePoolSize并且小于 maximumPoolSize，任务会被放入队列；
- 如果线程池中的线程数 countWorker大于 maximumPoolSize，假如任务能够放入堵塞队列则放入，否则执行拒绝策略；

### workQueue

工作队列，它是 BlockingQueue 堵塞类型，用来传输和保存由 execute()方法提交的 Runnable任务，当线程池中的线程大于等于
corePoolSize 而小于
maximumPoolSize时，任务会被优先放入队列，任务放入队列有 3种策略:

1. 直接处理：当 workQueue为
   SynchronousQueue（同步队列），则新任务会立即被交给线程池中已有的线程进行处理，如果无法立即获取可用线程，则认为任务放入队列失败，需要重新生成一个线程来处理该任务（如果此时线程数大于
   maximumPoolSize，任务被拒绝），为了避免任务被拒绝，一般会把 maximumPoolSize设置成无限大，带来的问题是线程数疯长，有资源耗尽的风险，不过该策略可以避免死锁。
2. 无界队列：当 workQueue为无界队列，任务会被放入队列中，等待空闲的线程来处理。如果任务之间是相互独立的，适合使用这种队列，但是当线程处理的速度小于新任务放入的速度，队列会无线封装，依然存在资源耗尽的风险。
3. 有界队列：当 workQueue为有界队列，可以规避无界队列资源耗尽的风险，但是这个边界该设置多大比较难把控，如果设置大队列小线程池，虽然可以降低
   CPU使用率和线程切换的开销，但是可能导致低吞吐量。

### keepAliveTime

保留存活时间，当线程数大于 corePoolSize时，假如线程池无需处理任务，超出 corePoolSize的空闲线程会保留 keepAliveTime时长后被
terminated，如果设置了 allowCoreThreadTimeOut(true)，核心线程也会保留 keepAliveTime时长后会被
terminated，keepAliveTime的值可以通过
setKeepAliveTime(long, TimeUnit)方法动态设置；

### threadFactory

线程工厂，线程池中的所有线程都是通过 threadFactory进行创建；使用线程工厂可以避免 new Thread()这种硬码创建新线程，我们可以在
threadFactory中使用特殊的线程子类、优先级等功能。

### handler

拒绝策略，当 Executor关闭时，以及 Executor对最大线程和工作队列容量都使用有限的界限，并且已经饱和时，在方法 execute(Runnable)
中提交的新任务将被拒绝。线程池提供了 4种拒绝策略：

- AbortPolicy：默认的拒绝策略。直接丢弃任务并抛出 RejectedExecutionException这样一个运行时错误；
- CallerRunsPolicy：任务会被转交给调用 execute()方法的线程处理；
- DiscardPolicy：直接丢弃了无法执行的任务；
- DiscardOldestPolicy：如果线程池没有被关闭，则丢弃工作队列头部的任务（最早放入的任务），然后重试执行（可能再次失败，导致重复此操作）；

在实际生产中，我们需要根据具体的业务来自定义拒绝策略：

比如：对于一些不能丢失的数据，可以写数据库、写log或者放入MQ中，当线程池恢复处理能力时，再取出数据进行处理。

比如：在数据库中数据有软状态，可以直接丢弃，当线程池恢复处理能力时，再从数据库中获取数据进行处理。

## 线程池的运行状态

线程池的运行状态主要有 5种：
- RUNNING：接受新任务并处理排队任务；
- SHUTDOWN：不接受新任务，但处理排队任务；
- STOP：不接受新任务，不处理排队任务，并中断正在进行的任务；
- TIDYING：所有任务都已终止，workerCount 为零，转换到状态 TIDYING的线程将运行 terminate()钩子方法；
- TERMINATED：当 terminate()完成，在 awaitTermination()中等待的线程将达到 TERMINATED状态；

5种运行状态的转换关系如下图：

![img.png](https://www.yuanjava.cn/assets/md/java/threadpool-runstate.png)

在 java.util.concurrent.ThreadPoolExecutor 源码中，有一个 ctl 变量可以获取线程池的生命周期，并且通过 advanceRunState()
方法进行生命周期状态值的切换，ctl是一个包含了 workerCount 和 runState两个字段的原子整数，共 32bit。

```java
public class ThreadPoolExecutor extends AbstractExecutorService {
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
}
```

workerCount：指的是线程池中的工作线程数；

runState：指的是线程池的运行状态，包含 RUNNING、SHUTDOWN、STOP、TIDYING、TERMINATED 5种；

ctl 结构如下图：

![img.png](https://www.yuanjava.cn/assets/md/java/threadpool-ctl.png)

## 线程池工作机制
下面整理了一张线程池工作的核心流程图：

![img.png](https://www.yuanjava.cn/assets/md/java/thread-working-principle.png)


## 线程池源码分析

### execute()方法

```java
public class ThreadPoolExecutor extends AbstractExecutorService {
  public void execute(Runnable command) {

    if (command == null)
      throw new NullPointerException();
    /*
     * Proceed in 3 steps:
     *
     * 1. If fewer than corePoolSize threads are running, try to
     * start a new thread with the given command as its first
     * task.  The call to addWorker atomically checks runState and
     * workerCount, and so prevents false alarms that would add
     * threads when it shouldn't, by returning false.
     *
     * 2. If a task can be successfully queued, then we still need
     * to double-check whether we should have added a thread
     * (because existing ones died since last checking) or that
     * the pool shut down since entry into this method. So we
     * recheck state and if necessary roll back the enqueuing if
     * stopped, or start a new thread if there are none.
     *
     * 3. If we cannot queue task, then we try to add a new
     * thread.  If it fails, we know we are shut down or saturated
     * and so reject the task.
     */
    int c = ctl.get();
    /**
     * 步骤1：workerCount < corePoolSize，尝试创建一个新线程执行任务
     * workerCountOf()方法取出 ctl变量低 29位的值，表示当前活动的线程数；
     * 如果当前活动的线程数小于corePoolSize，则新建一个线程放入线程池中，并把该任务放到线程中
     */
    if (workerCountOf(c) < corePoolSize) {
      /**
       * addWorker中的第二个参数表示限制添加线程的数量 是根据据corePoolSize 来判断还是maximumPoolSize来判断；
       * 如果是ture，根据corePoolSize判断
       * 如果是false，根据maximumPoolSize判断
       */
      if (addWorker(command, true))
        return;

      // 如果 addWorker()添加失败，则重新获取 ctl值
      c = ctl.get();
    }
    /**
     * 步骤2：当步骤1的 addWorker()失败，并且线程池是RUNNING，并且任务成功接入阻塞队列
     */
    if (isRunning(c) && workQueue.offer(command)) {
      //double-check，重新获取ctl的值
      int recheck = ctl.get();
      /**
       * 因为线程池的状态一直在变化，所以再次判断线程池的状态，如果不是运行状态，并且移之前已添加到阻塞队列中command
       * 然后进入拒绝策略，流程终止
       */
      if (!isRunning(recheck) && remove(command))
        reject(command);
      /**
       * 线程池为Running状态，获取线程池中的总线程数，如果数量是0，则执行 addWorker()方法；
       * 第一个参数为null，表示在线程池中创建一个线程，但不去启动
       * 第二个参数为false，将线程池的线程数量的上限设置为maximumPoolSize，添加线程时根据maximumPoolSize来判断
       */
      else if (workerCountOf(recheck) == 0)
        addWorker(null, false);
      /**
       * 步骤3：步骤1的 addWorker()失败并且无法进入步骤2，有两种情况：
       * 1、线程池的状态不是RUNNING；
       * 2、线程池状态是RUNNING，但是workerCount >= corePoolSize， workerQueue已满
       * 再次调用addWorker方法，第二个参数传 false，将线程池的线程数上限设置为 maximumPoolSize；
       * 如果失败则执行拒绝策略；
       */
    } else if (!addWorker(command, false))
      reject(command);
  }
}
```

execute(Runnable command) 方法总结：

1. 如果 workerCount < corePoolSize，则尝试创建并启动一个新线程来执行新提交的任务；
2. 如果 workerCount >= corePoolSize，且线程池内的阻塞队列未满，则将任务添 加到该阻塞队列中；
3. 如果 corePoolSize <= workerCount < maximumPoolSize，且线程池内的阻塞队列已满，则尝试创建并启动一个新线程来执行新提交的任务；
4. 如果 workerCount >= maximumPoolSize，并且线程池内的阻塞队列已满, 则根据拒绝策略来处理该任务, 默认的
   AbortPolicy策略是直接抛异常；

### addWorker()方法

```java
public class ThreadPoolExecutor extends AbstractExecutorService {
  /**
   * 给线程池增加新的 worker
   * @param firstTask 当前任务
   * @param core 创建新worker是以 corePoolSize还是 maximumPoolSize作为最大阈值
   * @return 返回增加新的 worker的结果，true or false
   */
  private boolean addWorker(Runnable firstTask, boolean core) {
    // retry 作用域为 for循环
    retry:
    // 自旋
    for (int c = ctl.get(); ; ) {
      // Check if queue empty only if necessary.
      /**
       * 这里的判断比较绕，直白的说：线程池已经关闭 或者 有资格关闭，则无法 addWorker并且返回false；
       * 如果线程池当前的 runState为 SHUTDOWN、STOP、TIDYING、TERMINATED 其中一种；并且下面的结果
       * 三个条件 通过 || (或) 连接，只要有一个为 true，就返回 true；
       * 1. 线程池当前的 runState为 STOP、TIDYING、TERMINATED 其中一种；表示关闭状态，不再接收提交的任务，但却可以继续处理阻塞队列中已经保存的任务；
       * 2. firstTask 不为空
       * 3. Check if queue empty only if necessary.阻塞队列为空
       */
      if (runStateAtLeast(c, SHUTDOWN)
        && (runStateAtLeast(c, STOP)
        || firstTask != null
        || workQueue.isEmpty()))
        return false;

      for (; ; ) {
        // workerCount 大于最大线程阈值，无法 addWorker，返回false
        if (workerCountOf(c)
          >= ((core ? corePoolSize : maximumPoolSize) & COUNT_MASK))
          return false;
        // 通过 CAS 方式来增加线程数，如果成功，则跳出最外层 for
        if (compareAndIncrementWorkerCount(c))
          break retry;
        c = ctl.get();  // Re-read ctl
        // 线程池状态为 非 RUNNING，则重新进入 for循环
        if (runStateAtLeast(c, SHUTDOWN))
          continue retry;
        // else CAS failed due to workerCount change; retry inner loop
      }
    }

    // addWorker() 的核心流程
    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
      /**
       * 根据firstTask来创建Worker对象，每个Worker对象里面会封装一个通过 ThreadFactory创建的线程
       * Worker(Runnable firstTask) {
       *     setState(-1); // inhibit interrupts until runWorker
       *     this.firstTask = firstTask;
       *     this.thread = getThreadFactory().newThread(this);
       *  }
       */

      w = new Worker(firstTask);
      final Thread t = w.thread;
      if (t != null) {
        final ReentrantLock mainLock = this.mainLock;
        // 获取可重入锁
        mainLock.lock();
        try {
          // Recheck while holding lock.
          // Back out on ThreadFactory failure or if
          // shut down before lock acquired.
          int c = ctl.get();
          // 如果线程池的runState为 RUNNING状态，或者 runState < STOP(RUNNING或者SHUTDOWN)并且任务为空
          // 如果线程不为NEW，则抛异常
          if (isRunning(c) ||
            (runStateLessThan(c, STOP) && firstTask == null)) {
            if (t.getState() != Thread.State.NEW)
              throw new IllegalThreadStateException();
            workers.add(w);
            workerAdded = true;
            int s = workers.size();
            if (s > largestPoolSize)
              largestPoolSize = s;
          }
        } finally {
          mainLock.unlock();
        }
        if (workerAdded) {
          // 启动线程，Worker implements Runnable，是一个线程类，此时t.start()会调用 Worker的run()方法，从而调用 runWorker()方法
          t.start();
          workerStarted = true;
        }
      }
    } finally {
      if (!workerStarted)
        addWorkerFailed(w);
    }
    return workerStarted;
  }
}
```

addWorker() 方法总结：

- addWorker()方法的主要作用是在线程池中创建一个新的线程并执行任务；
- firstTask参数用于指定新增的线程执行的第一个任务，
- core参数用来限制创建新线程，是以 corePoolSize还是以 maximumPoolSize作为线程数最大阈值。
- 需要获取 mainLock可重入锁成功才能将 worker加入 HashSet中
- worker添加成功后，调用线程的t.start() 启动线程执行任务;

### Worker类分析

```java
private final class Worker extends AbstractQueuedSynchronizer implements Runnable {
  /**
   * This class will never be serialized, but we provide a
   * serialVersionUID to suppress a javac warning.
   */
  private static final long serialVersionUID = 6138294804551838833L;

  /** Thread this worker is running in.  Null if factory fails. */
  final Thread thread;
  /** Initial task to run.  Possibly null. */
  Runnable firstTask;
  /** Per-thread task counter */
  volatile long completedTasks;

  // TODO: switch to AbstractQueuedLongSynchronizer and move
  // completedTasks into the lock word.

  /**
   * Creates with given first task and thread from ThreadFactory.
   * @param firstTask the first task (null if none)
   */
  Worker(Runnable firstTask) {
    /**
     *  把state设置为 -1，阻止中断直到调用runWorker方法；
     *  因为 AQS默认 state是 0，刚创建一个 Worker对象，在没有执行任务时，不应该被中断
     */
    setState(-1); // inhibit interrupts until runWorker
    this.firstTask = firstTask;
    /**
     * 创建一个线程，newThread()方法入参是 this对象，也就是 Worker对象，
     * 因为 Worker implements Runnable，Worker 是一个线程类；
     * 所以 Worker对象中的 Thread.start()会间接调用 Worker类中 run()方法
     */
    this.thread = getThreadFactory().newThread(this);
  }

  /** Delegates main run loop to outer runWorker.
   * 将主运行循环委托给外部 runWorker
   */
  public void run() {
    runWorker(this);
  }

  // Lock methods
  //
  // The value 0 represents the unlocked state.
  // The value 1 represents the locked state.

  protected boolean isHeldExclusively() {
    return getState() != 0;
  }

  /** 获取锁
   * CAS 方式修改 state，不可重入；
   * state根据 0 来判断，所以 Worker构造方法中state=-1，就是为了禁止在执行任务前对线程进行中断；
   * 因此，在 runWorker()方法中会先调用Worker对象的 unlock()方法将 state设置为 0
   */
  protected boolean tryAcquire(int unused) {
    if (compareAndSetState(0, 1)) {
      setExclusiveOwnerThread(Thread.currentThread());
      return true;
    }
    return false;
  }

  // 释放锁
  protected boolean tryRelease(int unused) {
    setExclusiveOwnerThread(null);
    setState(0);
    return true;
  }

  public void lock() {
    acquire(1);
  }

  public boolean tryLock() {
    return tryAcquire(1);
  }

  public void unlock() {
    release(1);
  }

  public boolean isLocked() {
    return isHeldExclusively();
  }

  void interruptIfStarted() {
    Thread t;
    if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
      try {
        t.interrupt();
      } catch (SecurityException ignore) {
      }
    }
  }
}
```

Worker类总结：

- Worker类实现了 Runnable接口，因此是一个线程类。
- Worker类继承了AQS，使用了 AQS实现独占锁的功能。
- 为什么不使用 ReentrantLock可重入锁来实现？
  从源码的 tryAcquire()方法可以看出它是不允许重入的，而ReentrantLock是允许可重入的：

1、lock方法一旦获取独占锁，表示当前线程正在执行任务中；
2、如果正在执行任务，则不应该中断线程；
3、如果该线程现在不是独占锁的状态，也就是空闲状态，说明它没有处理任务，这时可以对该线程进行中断；
4、线程池中执行shutdown方法或tryTerminate方法时会调用interruptIdleWorkers方法来中断空闲线程，interruptIdleWorkers方法会使用tryLock方法来判断线程池中的线程是否是空闲状态；
5、之所以设置为不可重入的，是因为在任务调用setCorePoolSize这类线程池控制的方法时，不会中断正在运行的线程所以，Worker继承自AQS，用于判断线程是否空闲以及是否处于被中断。

### runWorker()

```java
private final class Worker extends AbstractQueuedSynchronizer implements Runnable {

  final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    // 获取任务
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
      while (task != null || (task = getTask()) != null) {
        w.lock();
        // If pool is stopping, ensure thread is interrupted;
        // if not, ensure thread is not interrupted.  This
        // requires a recheck in second case to deal with
        // shutdownNow race while clearing interrupt
        if ((runStateAtLeast(ctl.get(), STOP) ||
          (Thread.interrupted() &&
            runStateAtLeast(ctl.get(), STOP))) &&
          !wt.isInterrupted())
          wt.interrupt();
        try {
          beforeExecute(wt, task);
          try {
            task.run();
            afterExecute(task, null);
          } catch (Throwable ex) {
            afterExecute(task, ex);
            throw ex;
          }
        } finally {
          task = null;
          w.completedTasks++;
          w.unlock();
        }
      }
      completedAbruptly = false;
    } finally {
      processWorkerExit(w, completedAbruptly);
    }
  }
}
```

runWorker()方法总结：

1. while 死循环通过getTask()方法从阻塞队列中获取任务，当getTask()为null时跳出 while()循环；
2. 正常进入 while死循，先加锁，然后进入任务处理逻辑；
3. 如果线程池正在停止，则中断当前线程，调用 processWorkerExit()方法，runWorker()结束；
4. 调用 task.run() 执行任务；
5. while 循环不论何种原因跳出，最终都需要执行 processWorkerExit()方法；
6. runWorker()方法执行成功，意味着 Worker中的 run()方法执行成功，则释放锁，销毁当前线程；

### getTask()方法

```java
 public class ThreadPoolExecutor extends AbstractExecutorService {

  private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?

    for (; ; ) {
      int c = ctl.get();

      /**
       * 再次判断线程池的 runState，决定能够获取到 task
       */
      // Check if queue empty only if necessary.
      if (runStateAtLeast(c, SHUTDOWN)
        && (runStateAtLeast(c, STOP) || workQueue.isEmpty())) {
        decrementWorkerCount();
        return null;
      }

      int wc = workerCountOf(c);

      /**
       * timed 变量用于判断是否需要进行 keepAliveTime超时处理；
       * allowCoreThreadTimeOut 默认是 false，代表核心线程不允许超时；
       * wc > corePoolSize，表示当前线程数大于核心线程数；
       * 对于超出 corePoolSize的线程，需要 keepAliveTime超时处理；
       */
      // Are workers subject to culling?
      boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

      /**
       * wc > maximumPoolSize 产生的原因是：此方法执行阶段同时执行了 setMaximumPoolSize()方法；
       * timed && timedOut 如果为true，表示当前操作需要进行获取任务超时处理，并且上次从阻塞队列中获取任务发生了超时；
       * 如果池中的线程数大于 1，或者 workQueue为空，则尝试将 workCount减1；如果减1失败，则返回重试；
       */
      if ((wc > maximumPoolSize || (timed && timedOut))
        && (wc > 1 || workQueue.isEmpty())) {
        if (compareAndDecrementWorkerCount(c))
          return null;
        continue;
      }

      /**
       * 从工作队列中获取任务，根据 timed来决定采用超时获取task()还是堵塞获取task
       * timed为 true，则通过workQueue的poll方法进行超时控制，如果在keepAliveTime时间内没有获取任务，则返回null；
       * 否则通过take方法，如果队列为空，则take方法会阻塞直到队列中不为空；
       */
      try {
        Runnable r = timed ?
          workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
          workQueue.take();
        if (r != null)
          return r;
        timedOut = true;
      } catch (InterruptedException retry) {
        timedOut = false;
      }
    }
  }
}
```

getTask()方法总结：

- 判断线程池的 runState决定是否能返回任务；
- 判断是否需要对超出 corePoolSize的线程最超时推出操作；
- 根据 timed来判断 采用什么方式从阻塞队列中获取任务；

### processWorkerExit()方法

```java
 public class ThreadPoolExecutor extends AbstractExecutorService {

  private void processWorkerExit(Worker w, boolean completedAbruptly) {
    // 如果completedAbruptly为 true，则说明线程执行时出现异常，需要将 workerCount 减1
    if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
      decrementWorkerCount();

    final ReentrantLock mainLock = this.mainLock;
    // 获取锁
    mainLock.lock();
    try {
      // 已完成 taskCount计数加1
      completedTaskCount += w.completedTasks;
      // 从 workers hashSet中移除当前 worker
      workers.remove(w);
    } finally {
      mainLock.unlock();
    }

    // 钩子函数，根据线程池的状态来判断是否结束线程池
    tryTerminate();

    int c = ctl.get();
    /**
     * 当前线程 runState为 RUNNING或 SHUTDOWN 时，如果 worker是异常结束，会执行 addWorker()方法；
     * 如果allowCoreThreadTimeOut=true，那么等待队列有任务，至少保留一个worker；
     * 如果allowCoreThreadTimeOut=false，workerCount少于coolPoolSize
     */
    if (runStateLessThan(c, STOP)) {
      if (!completedAbruptly) {
        int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
        if (min == 0 && !workQueue.isEmpty())
          min = 1;
        if (workerCountOf(c) >= min)
          return; // replacement not needed
      }
      addWorker(null, false);
    }
  }
}
```

到此，线程池的核心方法已经分析完毕，那么，我们需要如何创建线程池呢？

## 线程池创建方式

### 手动创建

参数按需设置，比如下面代码就是自己创建的一个线程池：

```java
ThreadPoolExecutor executor=new ThreadPoolExecutor(2,
  4,
  4,
  TimeUnit.SECONDS,
  new LinkedBlockingQueue<>(),
  Executors.defaultThreadFactory(),
  new ThreadPoolExecutor.AbortPolicy());
```

### 使用 Executors提供的方法

- newFixedThreadPool：固定线程数的线程池，corePoolSize = maximumPoolSize，keepAliveTime = 0，堵塞队列为无界的
  LinkedBlockingQueue。适用于为了满足资源管理的需求，而需要限制当前线程数量的场景，适用于负载比较重的服务器。

- newSingleThreadExecutor：只有一个线程的线程池，corePoolSize = maximumPoolSize = 1，keepAliveTime = 0，堵塞队列为无界的
  LinkedBlockingQueue。适用于需要顺序执行的业务场景。

- newCachedThreadPool： 按需要创建新线程的线程池。核心线程数为0，最大线程数为
  Integer.MAX_VALUE，keepAliveTime为60秒，工作队列使用同步移交
  SynchronousQueue。该线程池可以无限扩展，当需求增加时，可以添加新的线程，而当需求降低时会自动回收空闲线程。适用于执行很多的短期异步任务，或者是负载较轻的服务器。

- newScheduledThreadPool：创建一个以延迟或定时的方式来执行任务的线程池，堵塞队列为 DelayedWorkQueue。适用于需要多个后台线程执行周期任务。

- newWorkStealingPool：可窃取的线程池（名字很难直接看下线程池的含义），JDK 1.8 新增，底层使用 ForkJoinPool 实现。

## 线程池终止方式

终止线程池主要有两种方式：

- shutdown()：“温柔”的关闭线程池。不接受新任务，但是在关闭前会将之前提交的任务处理完毕。
- shutdownNow()：“粗暴”的关闭线程池，也就是直接关闭线程池，通过 Thread#interrupt() 方法终止所有线程，不会等待之前提交的任务执行完毕。但是会返回队列中未处理的任务。

## 如何配置线程池

线程池参数的配置，我们会依据服务器的工作类型来设置，参考意见如下：

- 计算密集型，设置 线程数 = CPU数 + 1；

- I/O密集型，线程数 = CPU数 * 2；

但是在实际开发中，服务器很难完全区分是哪一种类型，因此一个比较合理的设置是： 线程数 = CPU数 * CPU利用率 * (任务等待时间 /
任务计算时间 + 1)

比如，有个程序部署在4核的服务器上用于计算任务，假设任务计算时间是 100ms，等待 I/O操作为 400ms，则线程数约为：4 * 1 * (1 + 400
/ 100) = 20个。

不过，上面的都是参考值，在实际生产中的做法是，经过压测得出最佳线程数，并且把 corePoolSize 和 maximumPoolSize
设计成可配置化参数，一旦需要调整大小，可以直接通过配置中心来完成，以免重新发代码。

## 常见面试题

注意：在面试中回答线程池的问题，一定要先说明 jdk版本，因为不同的版本实现可能有差异（jdk修复bug或者性能优化导致的）。有了上面对线程池原理的分析，下面的面试题可以很容易的回答。

### 1.为什么有线程还需要线程池呢？
线程在正常执行或者异常中断时会被销毁，如果频繁的创建很多线程，不仅会消耗系统资源，还会降低系统的稳定性，一不小心把系统搞崩了。

使用线程池可以带来以下几个好处：

- 线程池内部的线程数是可控的；
- 可以灵活的设置参数；
- 线程池内会保留部分线程，当提交新的任务可以直接运行；
- 方便内部线程资源的管理，调优和监控；


### 2.能讲下线程池中几个核心属性吗？
参考上述 核心属性 章节

### 3.能讲下线程池的运行原理吗？
参考上述 运行原理 章节

### 4.线程池中的核心线程会被销毁吗？
  默认情况下不会，但是如果设置了 allowCoreThreadTimeOut(true)，核心线程也会保留 keepAliveTime时长后会被 terminated。

### 5.能讲下线程池的运行状态吗？他们之间是如何转换的？

参考上述 线程池的运行 章节

### 6.能聊聊线程池的拒绝策略吗？

  参考上述 拒绝策略 章节
### 7.能讲讲 ctl的实现机制吗？
  ctl将 runState 和 workerCount 封装成了一个原子 AtomicInteger操作。因为 runState 和 workerCount 是线程池正常运行的2个最重要属性。

  因此无论是查询还是修改，我们必须保证对这2个属性的操作是属于“同一时刻”的，也就是原子操作，否则就会出现错乱的情况。如果我们使用2个变量来分别存储，要保证原子性则需要额外进行加锁操作，这显然会带来额外的开销，而将这2个变量封装成1个
  AtomicInteger 则不会带来额外的加锁开销，而且只需使用简单的位操作就能分别得到 runState 和 workerCount。

由于这个设计，workerCount 的上限 CAPACITY = (1 << 29) - 1，对应的二进制原码为：0001 1111 1111 1111 1111 1111 1111
1111（29个1）。

通过 ctl得到 runState：runState = ctl & ~CAPACITY。（~ 按位取反），于是"~CAPACITY"的值为：1110 0000 0000 0000 0000 0000 0000
0000，只有高3位为1，与 ctl 进行 & 操作，结果为 ctl
高3位的值，也就是 runState。

通过 ctl得到 workerCount：workerCount = c & CAPACITY（位操作）。

### 8.Executors提供了哪些创建线程池的方法？

  参考上述 线程池创建方法 章节
### 9.线程池中的核心线程数和最大线程数该如何设置呢？

### 10.如何终止线程池？

  shutdownNow() 和 shutdown() 都是用来终止线程池的，调用 shutdown()方法程序不会报错，也不会立即终止线程，它会等待线程池中的缓存任务执行完之后再退出，执行了
  shutdown() 之后就不能给线程池添加新任务了；执行 shutdownNow()方法会试图立即停止任务，线程中的任务不会再执行，也无法添加新的任务。

不过，调用 shutdownNow() 和 shutdown()方法，线程池不一定退出，因为线程池会调用

### 11.为什么线程池中要使用堵塞队列？
  - 主要原因：线程从阻塞队列取任务时，如果阻塞队列不为空则立即返回，如果为空，则线程会被阻塞，一直等待，直到队列中有新的任务，这样就充分的利用了阻塞队列
  堵塞和通知线程的功能。如果采用非堵塞队列，则需要写额外的机制来实现通知线程获取任务。
  - 次要原因：当核心线程处理不过来，可以通过堵塞队列进行任务堆积

### 12. 非核心线程能成为核心线程吗？

核心线程和非核心线程是理解上的区分，线程池内部并不区分核心线程和非核心线程的，会根据池的工作线程数 countWorker和 corePoolSize 进行调整，
- 当 countWorker <= corePoolSize，则 countWorker这些线程被理解成核心线程；
- 当  corePoolSize < countWorker < maximumPoolSize，则 countWorker - corePoolSize 这部分线程被理解成非核心线程；

### 13. 线程池中的线程在什么时候创建？

默认情况下，即使是核心线程也只能在新任务到达时才创建和启动。但是可以使用 prestartCoreThread 或 prestartAllCoreThreads 方法来提前启动核心线程，即创建线程池的时候就会创建对应数量的线程。

### 14. 有界队列和无解队列有什么风险？

- 有界队列，当线程池的队列满了后，被拒绝的任务如何处理；
- 无界队列，当任务的提交速度大于线程池的处理速度，可能会导致内存溢出；

### 1.5 为什么不推荐使用Executors包装的线程池？
从上面的线程池的种类可以看出：
- newFixedThreadPool线程池，由于使用了LinkedBlockingQueue，队列的容量默认是无限大，如果任务堆积过多时可能导致内存溢出；
- newCachedThreadPool线程池，由于核心线程数无限大，当任务过多的时候，会导致创建大量的线程，可能机器负载过高，导致服务宕机；

## 总结

到此，我们对线程池进行了全面的分析，也列举了面试中常见的问题，对于线程池的学习，个人极力推荐优先阅读源码，毕竟是原滋原味，然后结合一些优秀的文章，比如本文😁，对照着理解，这样的话就能快速的掌握理论，
但是，光有理论还不行，还应该多参与一些关于线程池的开发工作，也可以多参与一些生产环境线程池的监控和排错任务，加强实战水平，这样才能在工作和面试中游刃有余。

文章总结不易，看到的这里的小伙伴，还请帮忙点赞，关注哦！

## 鸣谢

如果你觉得本文章对你有帮助，感谢转发给更多的好友，关注我：猿java，为你呈现更多的硬核文章。

<img src="https://yuanjava.cn/assets/img/pub.jpg" alt="drawing" style="width:300px;"/>

