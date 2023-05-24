___
# 原理
## 工作流程
1. 先判断线程池运行的线程是否达到`corePoolSize`，如果没有则新创建一个线程执行刚提交的任务，否则，则进入第 2 步；
2. 将当前任务加入阻塞队列，；否则，则进入第 3 步；
3. 判断线程池的线程数量是否达到`maximumPoolSize`，如果没有，则创建一个新的线程来执行任务，否则，则交给拒绝策略进行处理

___
# 源码分析

## 构造函数
```java
public class ThreadPoolExecutor extends AbstractExecutorService {
    ...
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
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.acc = System.getSecurityManager() == null ?
                null :
                AccessController.getContext();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
    ...
}
```

### int corePoolSize 核心线程数
核心线程会一直存活，及时没有任务需要执行
当线程数小于核心线程数时，即使有线程空闲，线程池也会优先创建新线程处理
设置allowCoreThreadTimeout=true（默认false）时，核心线程会超时关闭

### int maximumPoolSize 线程池最大线程数量
任务被提交到线程池以后，首先会找有没有空闲存活线程，如果有则直接将任务交给这个空闲线程来执行，如果没有则会缓存到工作队列(后面会介绍)中，如果工作队列满了，才会创建一个新线程，然后从工作队列的头部取出一个任务交由新线程来处理，而将刚提交的任务放入工作队列尾部。线程池不会无限制的去创建新线程，它会有一个最大线程数量的限制，这个数量即由maximunPoolSize指定

### long keepAliveTime 线程空闲时间
超过 corePoolSize 数量的线程会等待任务的到来，如果一直空闲，当线程空闲时间达到 keepAliveTime 时，线程会 terminate，直到线程数量 =corePoolSize
如果 allowCoreThreadTimeout=true，则会直到线程数量=0
allowCoreThreadTimeout 默认是 false，需要通过方法 allowCoreThreadTimeOut 设置
```java
public class ThreadPoolExecutor extends AbstractExecutorService {
    ...
    /**
     * If false (default), core threads stay alive even when idle.
     * If true, core threads use keepAliveTime to time out waiting
     * for work.
     */
    private volatile boolean allowCoreThreadTimeOut;
    ...
    /**
     * Sets the policy governing whether core threads may time out and
     * terminate if no tasks arrive within the keep-alive time, being
     * replaced if needed when new tasks arrive. When false, core
     * threads are never terminated due to lack of incoming
     * tasks. When true, the same keep-alive policy applying to
     * non-core threads applies also to core threads. To avoid
     * continual thread replacement, the keep-alive time must be
     * greater than zero when setting {@code true}. This method
     * should in general be called before the pool is actively used.
     *
     * @param value {@code true} if should time out, else {@code false}
     * @throws IllegalArgumentException if value is {@code true}
     *         and the current keep-alive time is not greater than zero
     *
     * @since 1.6
     */
    public void allowCoreThreadTimeOut(boolean value) {
        if (value && keepAliveTime <= 0)
            throw new IllegalArgumentException("Core threads must have nonzero keep alive times");
        if (value != allowCoreThreadTimeOut) {
            allowCoreThreadTimeOut = value;
            if (value)
                interruptIdleWorkers();
        }
    }
    ...
}
```

### TimeUnit unit 的时间单位

### `BlockingQueue<Runnable> workQueue` 任务队列
当核心线程数达到最大时，新任务会放在队列中排队等待执行
直到队列满了才会开启新线程，直到数量达到 maximumPoolSize

### ThreadFactory threadFactory 线程创建工厂
用于设置创建线程的工厂，可以通过线程工厂给每个创建出来的线程做些更有意义的事情，比如 设定线程名、设置daemon和优先级等等

### RejectedExecutionHandler handler 拒绝策略
两种情况会拒绝处理任务：
1. 当线程数已经达到 maxPoolSize，队列已满，没有空闲线程可用，会执行拒绝策略
2. 当线程池被调用 shutdown() 后，会等待线程池里的任务执行完毕，再 shutdown。如果在这之间再提交任务，会拒绝新任务

ThreadPoolExecutor 中内置四个实现，我们也可以自定义实现 RejectedExecutionHandler 接口，下面看看四个内置实现
#### 1 AbortPolicy
丢弃任务，抛运行时异常。这是默认处理
```java
    /**
     * A handler for rejected tasks that throws a
     * {@code RejectedExecutionException}.
     */
    public static class AbortPolicy implements RejectedExecutionHandler {
        /**
         * Creates an {@code AbortPolicy}.
         */
        public AbortPolicy() { }

        /**
         * Always throws RejectedExecutionException.
         *
         * @param r the runnable task requested to be executed
         * @param e the executor attempting to execute this task
         * @throws RejectedExecutionException always
         */
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            throw new RejectedExecutionException("Task " + r.toString() +
                                                 " rejected from " +
                                                 e.toString());
        }
    }
```
#### 2 CallerRunsPolicy 
由当前提交任务的线程直接执行任务
```java
    /**
     * A handler for rejected tasks that runs the rejected task
     * directly in the calling thread of the {@code execute} method,
     * unless the executor has been shut down, in which case the task
     * is discarded.
     */
    public static class CallerRunsPolicy implements RejectedExecutionHandler {
        /**
         * Creates a {@code CallerRunsPolicy}.
         */
        public CallerRunsPolicy() { }

        /**
         * Executes task r in the caller's thread, unless the executor
         * has been shut down, in which case the task is discarded.
         *
         * @param r the runnable task requested to be executed
         * @param e the executor attempting to execute this task
         */
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            if (!e.isShutdown()) {
                r.run();
            }
        }
    }
```
#### 3 DiscardOldestPolicy
把等待队列中的第一个任务删掉
```java
    /**
     * A handler for rejected tasks that discards the oldest unhandled
     * request and then retries {@code execute}, unless the executor
     * is shut down, in which case the task is discarded.
     */
    public static class DiscardOldestPolicy implements RejectedExecutionHandler {
        /**
         * Creates a {@code DiscardOldestPolicy} for the given executor.
         */
        public DiscardOldestPolicy() { }

        /**
         * Obtains and ignores the next task that the executor
         * would otherwise execute, if one is immediately available,
         * and then retries execution of task r, unless the executor
         * is shut down, in which case task r is instead discarded.
         *
         * @param r the runnable task requested to be executed
         * @param e the executor attempting to execute this task
         */
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            if (!e.isShutdown()) {
                e.getQueue().poll();
                e.execute(r);
            }
        }
    }
```
#### 4 DiscardPolicy
空实现，忽视这个提交，什么都不会发生
```java
    /**
     * A handler for rejected tasks that silently discards the
     * rejected task.
     */
    public static class DiscardPolicy implements RejectedExecutionHandler {
        /**
         * Creates a {@code DiscardPolicy}.
         */
        public DiscardPolicy() { }

        /**
         * Does nothing, which has the effect of discarding task r.
         *
         * @param r the runnable task requested to be executed
         * @param e the executor attempting to execute this task
         */
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        }
    }
```

## 线程池状态
```java
    /**
     * 线程池可以接收新的任务和执行已添加的任务。线程池被一旦被创建就是 RUNNING
     */
    private static final int RUNNING    = -1 << COUNT_BITS;
    /**
     * 不接收新任务，但能处理已添加的任务
     * shutdown() 方法
     * RUNNING -> SHUTDOWN
     */
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    // 
    /**
     * 不接收新任务，不处理已添加的任务，并且会中断正在执行的任务
     * shutdownNow() 方法
     * (RUNNING or SHUTDOWN) -> STOP
     */
    private static final int STOP       =  1 << COUNT_BITS;
    /**
     * 当所有的任务已终止，记录的”任务数量”为0，线程池会变为TIDYING状态
     * 当线程池变为TIDYING状态时，会执行钩子函数terminated()
     * SHUTDOWN -> TIDYING 或者 STOP -> TIDYING
     */
    private static final int TIDYING    =  2 << COUNT_BITS;
    /**
     * 当 TIDYING 状态执行的钩子函数 terminated() 被执行完成之后，线程池彻底终止，就变成TERMINATED状态
     * TIDYING -> TERMINATED
     */
    private static final int TERMINATED =  3 << COUNT_BITS;
```

## Worker
Worker 类本身既实现了Runnable，又继承了AbstractQueuedSynchronizer，所以其既是一个可执行的任务，又可以达到锁的效果
```java
    private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
    {
        // 内部维护了线程实例，靠此实例调用 start() 开启线程
        final Thread thread;
        
        // 用户传入的实际工作任务， run 方法会在 runWorker 内被调用
        Runnable firstTask;

        /**
         * Creates with given first task and thread from ThreadFactory.
         * @param firstTask the first task (null if none)
         */
        Worker(Runnable firstTask) {
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }

        // 线程启动后实际上会调用 worker 的这个 run 方法，在 runWorker 中调用 firstTask 或从队列获取 task 来调用 run 方法
        public void run() {
            runWorker(this);
        }

    }
```

## execute
![[Pasted image 20230523200615.png]]
```java
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        int c = ctl.get();
        // 1. 先判断线程池运行的线程是否达到 corePoolSize
        if (workerCountOf(c) < corePoolSize) {
            // 1.1 没有则新建一个线程执行当前任务
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        // 2. 已经达到 corePoolSize 则尝试放入阻塞队列 workQueue
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                // 判断线程池状态，如果不是 RUNNING 则把任务删掉，并执行拒绝策略
                reject(command);
            else if (workerCountOf(recheck) == 0)
                // 如果线程池正常运行，并且 worker 线程数量是 0 ，则添加一个没有任务的 worker
                addWorker(null, false);
        }
        // 插入队列失败，尝试扩充线程数量到 maximumPoolSize
        else if (!addWorker(command, false))
            // 扩充失败，执行拒绝侧脸
            reject(command);
    }
```

## submit

### execute 和 submit 比较
execute 的入参是 Runnable， 没有返回值。任务通过execute提交后就基本和主线程脱离关系了。

而submit的入参可以是Callable(也可以是Runnable)，并且有返回值，返回的是一个Future对象，然后通过对象的get方法获取任务执行的结果。
线程池通过submit方式提交任务，会把Runnable封装成FutrueTask，每次提交都会创建一个新的实例

如果runable中的方法抛出异常，`execute`会终止这个线程。而`submit`不会，异常被set了，可以用get主动拉取到的

使用submit，要么用get拉取异常处理， 要么自己写try catch把任务执行的逻辑包起来

## shutdownNow
```java
    public List<Runnable> shutdownNow() {
        List<Runnable> tasks;
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            advanceRunState(STOP);
            // 仅仅是中断当前执行线程，不保证是马上停止
            interruptWorkers();
            // 清空阻塞队列内的任务
            tasks = drainQueue();
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
        return tasks;
    }
```

## 线程池回收
java.util.concurrent.ThreadPoolExecutor#runWorker 中如果 java.util.concurrent.ThreadPoolExecutor#getTask 取出为空
则进行 processWorkerExit 进行线程回收





# 参考感谢
https://blog.csdn.net/qq_41573860/article/details/123291943
https://blog.csdn.net/wangfenglei123456/article/details/122563597
https://blog.csdn.net/zs18753479279/article/details/123776184

