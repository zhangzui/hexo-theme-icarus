---
title: ThreadPoolExecutors源码解析
---
![](/images/ThreadPoolExecutor.jpg "")

频繁的创建和销毁线程，会大大降低系统的效率。可能出现服务器在为每个请求创建新线程和销毁线程上花费的时间和消耗的系统资源要比处理实际的用户请求的时间和资源更多。
所以线程池可以缓存线程，需要的时候会从先从线程池中分配。其次根据不同的策略来创建线程和缓存线程。
<!--more-->
>微信公众号：MyClass社区
如有问题或建议，请公众号留言
#ThreadPoolExecutors源码解析
>微信公众号：MyClass社区
如有问题或建议，请公众号留言

###ThreadPool状态属性
线程池中维护着不同阶段的状态（ctl）是原子整数。在ctl中，低位的29位表示工作线程的数量，高位用来表示RUNNING、SHUTDOWN、STOP等状态。因此一个线程池的数量也就变成了(2^29)-1，而不是(2^31)-1
下面是源码中很重要的两个概念：
workerCount，表示有效的线程数
runState，表示是否正在运行、关闭、等状态，维护着线程池的生命周期。下面是具体的状态：

>RUNNING：运行状态：接受新任务并处理排队任务
 SHUTDOWN：关闭状态：不接受新任务，但处理排队任务
 STOP：停止状态：不接受新任务，不处理排队任务，并中断正在进行的任务
 TIDYING：(整理)所有任务都已终止，workerCount=0，线程转换到状态TIDYING,将运行terminate（）转为TERMINATED
 TERMINATED：终止状态：terminate方法已完成之后
```
二进制-RUNNING:11100000000000000000000000000000
二进制-SHUTDOWN:0
二进制-STOP:100000000000000000000000000000
二进制-TIDYING:1000000000000000000000000000000
二进制-TERMINATED:1100000000000000000000000000000
// runState存储在高位中
private static final int RUNNING    = -1 << COUNT_BITS;
private static final int SHUTDOWN   =  0 << COUNT_BITS;
private static final int STOP       =  1 << COUNT_BITS;
private static final int TIDYING    =  2 << COUNT_BITS;
private static final int TERMINATED =  3 << COUNT_BITS;
```
###execute()
execute主要干了下面几件事：
1. 如果正在运行少于corePoolSize的线程，请尝试使用给定命令作为其第一个任务启动新线程。对addWorker的调用以原子方式检查runState和workerCount，因此通过返回false来防止在不应该添加线程时发生的错误警报。
2. 如果任务可以成功排队，那么我们仍然需要仔细检查是否应该添加一个线程（因为自上次检查后现有的线程已经死亡），或者自从进入此方法后池关闭了。 所以我们重新检查状态，如果必要的话，如果没有则回滚入队，或者如果没有，则启动新的线程。.
3. 如果我们不能排队任务，那么我们尝试添加一个新线程。 如果失败，我们知道我们已关闭或饱和，因此拒绝该任务。
```
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    //获取线程池的状态值，用于状态的判断，下面会多次检验这个值，因为线程池运行过程中，状化时刻在变化（可能被shotdown，或者其他线程已经改变了核心线程数）
    int c = ctl.get();
    //1.如果有效线程数 小于corePoolSize的线程
    if (workerCountOf(c) < corePoolSize) {
        //如果添加Worker成功则直接return，其中true标识小于corePoolSize数的标识
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    //2.如果是Running状态并且入队成功
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        //3.首先判断状态是否还是Running，如果不是则回滚入队，再拒绝任务
        if (! isRunning(recheck) && remove(command)){
            reject(command);
        }else if (workerCountOf(recheck) == 0)
            //4.如果有效线程数 == 0,添加null Worker，并core=false标识小于最大线程数 ，因为已经入队了，就不再回滚，直接回创建新线程处理队列的任务
            addWorker(null, false);
    }else if (!addWorker(command, false))
        //5.如果不是Running状态，入队也失败，直接走addWorker，创建线程，直到大于最大线程数，导致addWorker失败，直接拒绝
        reject(command);
}
```
###addWorker()
addWorker方法的主要工作是在线程池中创建一个新的Worker对象，其中会初始化调用getThreadFactory进行创建：thread = getThreadFactory().newThread(this)。
firstTask：用于指定新增的线程执行的第一个任务。
core：true表示有效线程数小于corePoolSize，false表示有效线程数小于maximumPoolSize，
addWorker大概干了这几件事：
1.检查runState 运行状态，获取有效线程数workerCountOf，进行判断是否可以add新的任务
2.然后如果检查没有问题则CAS增加ctl的workerCount，失败则重试，这样就可以原子进行控制核心线程数和最大线程数来保证线程池的运行。
3.新建任务执行对象worker，采用ReentrantLock同步workers.add(w)添加worker
4.添加失败则调用addWorkerFailed回滚一些操作，workers剔除新建的worker对象，减少ctl的workerCount有效线程数，并且调用尝试终止方法
```
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        //检查runState 运行状态
        int c = ctl.get();
        int rs = runStateOf(c);

        // 仅在必要时检查队列是否为空，状态值要在SHUTDOWN以上，只有RUNNING，或者关闭状态下，firstTask == null并且workQueue不为空才能继续addworker
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;

        for (;;) {
            //获取有效线程数
            int wc = workerCountOf(c);
            //如果有效线程数超过容量，或者大于标识位（核心数或者最大数）时，直接拒绝添加
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            //CAS增加ctl的workerCount，失败则重试
            if (compareAndIncrementWorkerCount(c))
                break retry;
            //重新获取ctl状态值
            c = ctl.get();
            //如果和开始的值不相等，则重试
            if (runStateOf(c) != rs)
                continue retry;
        }
    }
    //当上面的状态检查和compareAndIncrementWorkerCount成功之后，则可以安心的执行任务
    ///worker开始状态
    boolean workerStarted = false;
    //worker是否已添加状态
    boolean workerAdded = false;
    Worker w = null;
    try {
        //新建任务执行对象worker
        w = new Worker(firstTask);
        //worker工作程序正在运行的线程
        final Thread t = w.thread;

        if (t != null) {
            //可重入锁
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                // 按住锁定时重新检查。
                // 退出ThreadFactory失败或如果
                // 在获得锁定之前关闭。
                int rs = runStateOf(ctl.get());

                //如果线程池状态已关闭，或者firstTask==null，
                if (rs < SHUTDOWN || (rs == SHUTDOWN && firstTask == null)) {
                    //预先检查t是否可以开始，如果是活跃状态，则抛出非法线程状态异常
                    if (t.isAlive())
                        throw new IllegalThreadStateException();
                    //添加worker，并复制largestPoolSize和 workerAdded = true;
                    workers.add(w);
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                //释放锁
                mainLock.unlock();
            }
            //添加成功之后，则启动线程
            if (workerAdded) {
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        //失败则调用addWorkerFailed回滚一些操作，workers剔除新建的worker对象，减少ctl的workerCount有效线程数，并且调用尝试终止方法
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}
```
###Worker类
1.Worker主要维护运行任务的线程的中断控制状态，以及其他属性记录。
2.这个类机会性地扩展AbstractQueuedSynchronizer以简化获取和释放围绕每个任务执行的锁。
3.这可以防止意图唤醒等待任务的工作线程而不是中断正在运行的任务的中断。
4.采用非重入互斥锁而不是使用ReentrantLock，
    因为我们不希望工作任务在调用setCorePoolSize等池控制方法时能够重新获取锁。此外，为了在线程实际开始运行任务之前禁止中断，我们将锁定状态初始化为负值，并在启动时清除它（在runWorker中）。
```
private final class Worker extends AbstractQueuedSynchronizer implements Runnable {
    /**
     * 这个类永远不会被序列化，但我们提供了一个serialVersionUID来抑制javac警告。
     */
    private static final long serialVersionUID = 6138294804551838833L;

    /** 此工作程序正在运行的线程。如果工厂失败，则为空。 */
    final Thread thread;
    /** 要运行的初始任务。 可能是空的. */
    Runnable firstTask;
    /** 完成任务计数器 */
    volatile long completedTasks;

    /**
     * 使用给定的第一个任务和ThreadFactory中的线程创建.
     */
    Worker(Runnable firstTask) {
        //在runWorker之前禁止中断
        setState(-1);
        this.firstTask = firstTask;
        this.thread = getThreadFactory().newThread(this);
    }

    /** 启动时调用 */
    public void run() {
        runWorker(this);
    }

    protected boolean isHeldExclusively() {
        return getState() != 0;
    }

    protected boolean tryAcquire(int unused) {
        if (compareAndSetState(0, 1)) {
            setExclusiveOwnerThread(Thread.currentThread());
            return true;
        }
        return false;
    }

    protected boolean tryRelease(int unused) {
        setExclusiveOwnerThread(null);
        setState(0);
        return true;
    }

    public void lock()        { acquire(1); }
    public boolean tryLock()  { return tryAcquire(1); }
    public void unlock()      { release(1); }
    public boolean isLocked() { return isHeldExclusively(); }

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
###runWorker(Worker w)
runWorker从队列中反复获取任务并执行它们，同时这里提供了很多处理hook，帮助我们扩展一些功能，下面是方法上的元源码注解：
 >1.我们可以从一个初始任务开始，在这种情况下，我们不需要获得第一个任务。否则，只要pool正在运行，我们就会从getTask获取任务。如果它返回null，则由于池状态或配置参数的更改而退出工作线程。其他退出是由外部代码中的异常抛出引起的，在这种情况下，completedAbruptly持有，这通常会导致processWorkerExit替换此线程。
 2.在运行任何任务之前，获取锁以防止在任务执行期间其他池中断，然后我们确保除非线程池停止，否则此线程没有设置其中断。
 3.任务运之前都会调用beforeExecute，这可能会抛出异常，在这种情况下，我们会导致线程死亡（使用completedAbruptly打破循环为true）而不处理任务。
 4.假设beforeExecute正常完成，我们运行任务，收集任何抛出的异常以发送到afterExecute。我们分别处理RuntimeException，Error（两个规范保证我们陷阱）和任意Throwables。因为我们无法在Runnable.run中重新抛出Throwables，所以我们将它们包含在出错的Errors中（到线程的UncaughtExceptionHandler）。任何抛出的异常也会保守地导致线程死亡。
 5.在task.run完成之后，我们调用afterExecute，这也可能抛出异常，这也会导致线程死亡。根据JLS Sec 14.20，即使task.run抛出，该异常也将生效。异常机制的主要效果是afterExecute和线程的UncaughtExceptionHandler具有我们可以提供的关于用户代码遇到的任何问题的准确信息。
```
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    //允许中断
    w.unlock();

    boolean completedAbruptly = true;
    try {
        while (task != null || (task = getTask()) != null) {
            //这里
            w.lock();
            // 如果池正在停止，请确保线程被中断;如果没有，确保线程不被中断。 这个需要在第二种情况下重新检查才能处理
            if ((runStateAtLeast(ctl.get(), STOP) || (Thread.interrupted() && runStateAtLeast(ctl.get(), STOP))) && !wt.isInterrupted()){
                wt.interrupt();
            }
            try {
                //hook方法，可以自定义实现
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    //执行任务
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    //同样自己实现
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                //计数器
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        //为已处理过的的worker执行清理工作
        //此方法从工作集中删除线程，并且如果由于用户任务异常退出或者少于corePoolSize工作正在运行或者队列非空但没有工作程序，
       //则可能终止池或替换工作程序
        processWorkerExit(w, completedAbruptly);
    }
}
```
###getTask()
getTask方法用来从阻塞队列中取任务
以下情况worker必须退出，且返回null
1、有多个maximumPoolSize工作者（由于调用setMaximumPoolSize）
2、线程池被停止了
3、线程池被关闭shutdown，并且workQueue队列为空，并且线程等待任务超时
```
private Runnable getTask() {
    //获取超时标识位
    boolean timedOut = false;
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // 运行状态是running状态 并且 (运行状态已经停止 || 队列为空)，减少ctl的workerCount，并且返回null.
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }

        int wc = workerCountOf(c);

        // worker是否可以超时剔除或者活跃线程大于最大线程，允许阻塞标志位
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;
        //条件：1.活跃线程数大于最大线程数 或者 是否允许超时和超时标志位都为 true
        //条件：2.活跃线程大于1，或者队列为空
        //满足条件的cas减少工作线程数，成功扣减return null，失败继续循环获取
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {

            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

        try {
            //如果允许阻塞标志位，则检索并删除此队列的头部，如果元素可用，则等待指定的等待时间。
            //否则直接从队列中获取元素处理
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            //如果任务为空，赋值获取任务超时标识位为ture
            timedOut = true;
        } catch (InterruptedException retry) {
            //失败则赋值为false
            timedOut = false;
        }
    }
}
```
###线程池关闭相关操作：shutdown()

shutdown之前被执行的已提交任务，新的任务不会再被接收了。如果线程池已经被shutdown了，该方法的调用没有其他任何效果了。
```
 /**
     * 启动有序关闭，其中先前提交的任务将被执行，但不会接受任何新任务。 如果已经关闭，调用没有其他影响。
     * 方法不会等待先前提交的任务完成执行。 使用{@link #awaitTermination awaitTermination}来做到这一点。
     *
     * @throws SecurityException {@inheritDoc}
     */
    public void shutdown() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            advanceRunState(SHUTDOWN);
            interruptIdleWorkers();
            onShutdown(); // hook for ScheduledThreadPoolExecutor
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
    }
```
###interruptIdleWorkers(false)：
```
/**
     * 中断可能正在等待任务的线程（如未锁定所示），以便它们可以检查终止或配置更改。
     * 忽略SecurityExceptions（在这种情况下，某些线程可能保持不间断）。
     *
     *  @param onlyOne如果为true，则最多中断一个worker。
     *  只有在启用终止但仍有其他工作程序时，才会从tryTerminate调用此方法。 在这种情况下，
     *  所有线程当前正在等待的情况下，至多一个等待工作者被中断以传播关闭信号。 中断任意线程可确保自关闭后新到达的工作人员最终也将退出。 为了保证最终终止，
     *   只需要中断一个空闲工作程序就足够了，但shutdown（）会中断所有空闲工作程序，以便冗余工作人员立即退出，而不是等待拖延任务完成。
     */
    private void interruptIdleWorkers(boolean onlyOne) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (Worker w : workers) {
                Thread t = w.thread;
                if (!t.isInterrupted() && w.tryLock()) {
                    try {
                        t.interrupt();
                    } catch (SecurityException ignore) {
                    } finally {
                        w.unlock();
                    }
                }
                if (onlyOne)
                    break;
            }
        } finally {
            mainLock.unlock();
        }
    }
```
###interruptWorkers()：
中断所有线程，即使线程还是active的。忽略所有SecurityExceptions异常。和interruptIdleWorkers和区别从
代码上看就是后者在进行中断之前进行了一个而判断：
```
/**
     * 即使处于活动状态，也会中断所有线程。 忽略SecurityExceptions（在这种情况下，某些线程可能保持不间断）。
     */
    private void interruptWorkers() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (Worker w : workers)
                w.interruptIfStarted();
        } finally {
            mainLock.unlock();
        }
    }
```
###shutdownNow()：
尝试stop所有actively executing线程，halt所有正处于等待状态的任务，并返回一个等待执行的task列表
```/**
     * 尝试停止所有正在执行的任务，停止等待任务的处理，并返回等待执行的任务列表。
     * 这些任务耗尽（从此方法返回时从任务队列中删除。<p>此方法不等待主动执行任务终止。
     * 使用{@link #awaTermination awaitTermination}来执行此操作。
     * 除了尽力而为的尝试停止处理主动执行任务之外，没有任何保证。 此实现通过{@link Thread＃interrupt}取消任务，
     * 因此任何无法响应中断的任务都可能永远不会终止。
     *
     * @throws SecurityException {@inheritDoc}
     */
    public List<Runnable> shutdownNow() {
        List<Runnable> tasks;
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            advanceRunState(STOP);
            interruptWorkers();
            tasks = drainQueue();
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
        return tasks;
    }
```
###tryTerminate()：
在线程状态是SHUTDOWN而且线程池和任务队列都是空的，或者线程池处于STOP状态，并且线程池是空的，把线程池的状态改为TERMINATED。该方法一定要跟在任何使termination可行的操作之后——减少wc的值或者在shutdown过程中从任务队列中移除任务
```
/**
     * 如果（SHUTDOWN，池和队列为空）或（STOP和池为空），则转换为TERMINATED状态。
     * 如果否则有资格终止但workerCount非零，则中断空闲工作程序以确保关闭信号传播。
     * 必须在可能使终止成为可能的任何操作之后调用此方法 -在关闭期间减少工作器数或从队列中删除任务。
     * 该方法是非私有的，以允许从ScheduledThreadPoolExecutor访问
     */
    final void tryTerminate() {
        for (;;) {
            int c = ctl.get();
            if (isRunning(c) ||
                runStateAtLeast(c, TIDYING) ||
                (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
                return;
            if (workerCountOf(c) != 0) { // Eligible to terminate
                interruptIdleWorkers(ONLY_ONE);
                return;
            }

            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                    try {
                        terminated();
                    } finally {
                        ctl.set(ctlOf(TERMINATED, 0));
                        termination.signalAll();
                    }
                    return;
                }
            } finally {
                mainLock.unlock();
            }
            // else retry on failed CAS
        }
    }
```

