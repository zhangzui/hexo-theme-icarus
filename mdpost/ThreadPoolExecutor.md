---
title: ThreadPoolExecutor源码注释
date: 2018-07-26
categories:
    - java
tags:
  - 线程池
description: ThreadPoolExecutor源码解析
---
![](/images/thumbnail.17.jpg "")
ThreadPoolExecutor源码解析,源码注解翻译
<!--more-->
# ThreadPoolExecutor源码解析

```
import java.util.concurrent.*;
import java.util.concurrent.locks.AbstractQueuedSynchronizer;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.*;

/**
 * 一个{@link ExecutorService}，它使用可能的几个池化线程之一执行每个提交的任务，通常使用{@link Executors}工厂方法配置。
 *
 * 线程池解决了两个不同的问题：由于减少了每个任务的调用开销，它们通常在执行大量异步任务时提供更高的性能，
 * 并且它们提供了一种绑定和管理资源（包括执行时消耗的线程）的方法 一系列任务。 每
 * 个{@code ThreadPoolExecutor}还维护一些基本统计信息，例如已完成任务的数量。
 *
 * 为了在各种上下文中有用，这个类提供了许多可调参数和可扩展性钩子。
 * 但是，程序员应该使用更方便的{@link Executors}工厂方法{@link Executors＃newCachedThreadPool}（无界线程池，自动线程回收），
 * {@link Executors＃newFixedThreadPool}（固定大小的线程池）和{ @link Executors＃newSingleThreadExecutor}（单一后台线程），
 * 为最常见的使用场景预配置设置。 否则，在手动配置和调整此类时，请使用以下指南：
 *
 * <dt>Core and maximum pool sizes</dt>
 *
 * 当在方法{@link #execute（Runnable）}中提交新任务并且运行的线程少于corePoolSize时，
 * 即使其他工作线程处于空闲状态，也会创建一个新线程来处理该请求。 如果有多个corePoolSize但运行的maximumPoolSize线程少于maximumPoolSize，
 * 则只有在队列已满时才会创建新线程。 通过设置corePoolSize和maximumPoolSize相同，您可以创建固定大小的线程池。
 * 通过将maximumPoolSize设置为基本无限制的值（例如{@code Integer.MAX_VALUE}），您可以允许池容纳任意数量的并发任务。
 * 最典型的情况是，核心和最大池大小仅在构造时设置，但也可以使用{@link #setCorePoolSize}和{@link #setMaximumPoolSize}动态更改。
 *
 * <dt>On-demand construction</dt>
 *
 * <dd>By default, even core threads are initially created and
 * started only when new tasks arrive, but this can be overridden
 * dynamically using method {@link #prestartCoreThread} or {@link
 * #prestartAllCoreThreads}.  You probably want to prestart threads if
 * you construct the pool with a non-empty queue. </dd>
 *
 * <dt>Creating new threads</dt>
 *
 * 使用{@link ThreadFactory}创建新线程。 如果没有另外指定，则使用{@link Executors＃defaultThreadFactory}，
 * 它创建的线程都在同一个{@link ThreadGroup}中，并且具有相同的{@code NORM_PRIORITY}优先级和非守护进程状态。
 * 通过提供不同的ThreadFactory，您可以更改线程的名称，线程组，优先级，守护程序状态等。
 * 如果{@code ThreadFactory}在通过从{@code newThread}返回null而无法创建线程时，执行程序将 继续，但可能无法执行任何任务。
 * 线程应该拥有“modifyThread”{@code RuntimePermission}。 如果使用池的工作线程或其他线程不具有此权限，则服务可能会降级：配置更改可能不会及时生效，
 * 并且关闭池可能保持可以终止但未完成的状态。
 *
 * <dt>Keep-alive times</dt>
 *
 * <dd>如果池当前具有多个corePoolSize线程，则多余的线程如果空闲时间超过keepAliveTime，
 * 则将终止（请参阅{@link #getKeepAliveTime（TimeUnit）}）。 这提供了一种在不主动使用池时减少资源消耗的方法。
 * 如果池稍后变得更活跃，则将构造新线程。 也可以使用方法{@link #setKeepAliveTime（long，TimeUnit）}动态更改此参数。
 * 使用值{@code Long.MAX_VALUE} {@link TimeUnit＃NANOSECONDS}可以有效地禁止空闲线程在关闭之前终止。
 * 默认情况下，仅当存在多个corePoolSize线程时，保持活动策略才适用。
 * 但是方法{@link #allowCoreThreadTimeOut（boolean）}也可用于将此超时策略应用于核心线程，只要keepAliveTime值为非零。</dd>
 *
 * <dt>Queuing</dt>
 *
 * <dd>任何{@link BlockingQueue}都可用于转移和保留
 * 提交的任务。 此队列的使用与池大小调整交互：
 *
 * <li>如果运行的corePoolSize线程少于Executor
 * 总是喜欢添加新线程
 * 而不是排队。</ li>
 *
 * <li>如果正在运行corePoolSize或更多线程，则执行程序
 * 总是喜欢排队请求而不是添加新请求
 * 线程。</ LI>
 *
 * <li>如果请求无法排队，则创建新线程，除非
 * 这将超过maximumPoolSize，在这种情况下，任务将是
 * 拒绝。</ LI>
 *
 * </ul>
 *
 * There are three general strategies for queuing:
 * 直接切换:工作队列的一个很好的默认选择是{@link SynchronousQueue}，它将任务交给线程而不另外保存它们。在这里，如果没有线程立即可用于运行它，则尝试对任务进行排队将失败，因此将构造新线程。此策略在处理可能具有内部依赖性的请求集时避免了锁定。直接切换通常需要无限制的maximumPoolSizes以避免拒绝新提交的任务。这反过来承认，当命令继续以比处理它们更快的速度到达时，无限制的线程增长的可能性。
 * 无界队列:使用无界队列（例如{@link LinkedBlockingQueue}而没有预定义的容量）将导致新任务在所有corePoolSize线程忙时在队列中等待。因此，只会创建corePoolSize线程。 （并且maximumPoolSize的值因此没有任何影响。）当每个任务完全独立于其他任务时，这可能是适当的，因此任务不会影响彼此的执行;例如，在网页服务器中。虽然这种排队方式可以有助于平滑瞬态突发请求，但它承认，当命令继续平均到达的速度超过可处理速度时，无限制的工作队列增长的可能性。
 * 有界队列:有限队列（例如，{@link ArrayBlockingQueue}）与有限maximumPoolSizes一起使用时有助于防止资源耗尽，但可能更难以调整和控制。队列大小和最大池大小可以相互交换：使用大型队列和小型池最小化CPU使用率，OS资源和上下文切换开销，但可能导致人为的低吞吐量。如果任务经常阻塞（例如，如果它们是I / O绑定的），则系统可能能够为您提供比您允许的更多线程的时间。使用小队列通常需要更大的池大小，这会使CPU更加繁忙，但可能会遇到不可接受的调度开销，这也会降低吞吐量。
 * <dt>Rejected tasks</dt>
 *
 * <dd>当Executor关闭时，以及当Executor对最大线程和工作队列容量使用有限边界时，方法{@link #execute（Runnable）}
 * 中提交的新任务将被<em>拒绝</ em> 已经饱和了。 在任何一种情况下，{@code execute}方法都会调用其{@link RejectedExecutionHandler}
 * 的{@link RejectedExecutionHandler rejectedExecution（Runnable，ThreadPoolExecutor）}方法。 提供了四种预定义的处理程序策
 *
 * <ol>
 *
 * <li>在默认的{@link ThreadPoolExecutor.AbortPolicy}中
 * 处理程序抛出运行时{@link RejectedExecutionException}
 * 拒绝。</ LI>
 *
 * <li>在{@link ThreadPoolExecutor.CallerRunsPolicy}中，线程
 * 调用{@code execute}本身运行任务。 这提供了一个
 * 简单的反馈控制机制，将减慢速度
 * 新任务已提交。</ LI>
 *
 * <li>在{@link ThreadPoolExecutor.DiscardPolicy}中，一个任务
 * 简单地删除不能执行。</ LI>
 *
 * <li>在{@link ThreadPoolExecutor.DiscardOldestPolicy}中，如果是
 * 执行程序没有关闭，任务在工作队列的头部
 * 被删除，然后重试执行（可能再次失败，
 * 导致这种情况重复。）</ li>
 *
 * </ol>
 *
 * 可以定义和使用其他类型的{@link zRejectedExecutionHandler}类。 这样做需要特别小心，特别是当策略设计为仅在特定的zcapacity或排队策略下工作时。
 *
 * <dt>Hook methods</dt>
 *
 * <dd>此类提供在执行每个任务之前和之后调用的{@code protected}可覆盖{@link #beforeExecute（Thread，Runnable）}
 * 和{@link #afterExecute（Runnable，Throwable）}方法。 这些可以用来操纵执行环境; 例如，重新初始化ThreadLocals，收集统计信息或添加日志条目。
 * 此外，可以重写方法{@link #terminated}以执行Executor完全终止后需要执行的任何特殊处理。
 *
 * <p>如果钩子或回调方法抛出异常，内部工作线程可能会失败并突然终止。</dd>
 *
 * <dt>Queue maintenance</dt>
 *
 * 方法{@link #getQueue（）}允许访问工作队列以进行监视和调试。
 * 强烈建议不要将此方法用于任何其他目的。 当大量排队的任务被取消时，两个提供的方法{@link #remove（Runnable）}
 * 和{@link #purge}可用于协助存储回收。
 *
 * <dt>Finalization</dt>
 *
 * 程序<em> AND </ em>中不再引用的池没有剩余的线程将自动{@code shutdown}。
 * 如果您希望确保即使用户忘记调用{@link #shutdown}也会回收未引用的池，
 * 那么您必须通过设置适当的保持活动时间，使用零核心线程的下限来安排未使用的线程最终死亡 和/
 * 或设置{@link #allowCoreThreadTimeOut（boolean）}。
 *
 */
public class ThreadPoolExecutor extends AbstractExecutorService {
    /**
     * 主池控制状态ctl是原子整数打包
     * 两个概念领域
     *    workerCount，表示有效的线程数
     *    runState，表示是否正在运行，关闭等
     *
     * 为了将它们打包成一个int，我们将workerCount限制为（2 ^ 29）-1（约5亿）个线程，而不是（2 ^ 31）-1（20亿），
     * 否则可以表示。 如果以后这是一个问题，可以将变量更改为AtomicLong，并调整以下移位/掩码常量。
     * 但是在需要之前，使用int这个代码更快更简单。
     *
     * workerCount是允许启动但不允许停止的worker数。 该值可能与实际线程的实际数量暂时不同，
     * 例如，当ThreadFactory在被询问时无法创建线程时，以及退出线程在终止之前仍在执行簿记时。 用户可见的池大小将报告为工作集的当前大小。
     * runState提供主生命周期控件，具有以下值：
     *
     *   RUNNING：接受新任务并处理排队任务
     *   SHUTDOWN：不接受新任务，但处理排队任务
     *   STOP：不接受新任务，不处理排队任务，
     *              并中断正在进行的任务
     *   TIDYING：(整理)所有任务都已终止，workerCount为零，
     *              线程转换到状态TIDYING
     *              将运行terminate（）钩子方法
     *   TERMINATED：terminate（）已完成
     *
     * 这些值之间的数字顺序很重要，以允许有序比较。 runState随着时间的推移单调增加，但不需要命中每个状态。 过渡是：
     * RUNNING -> SHUTDOWN
     *      在调用shutdown（）时，可能隐含在finalize（）中
     * (RUNNING or SHUTDOWN) -> STOP
     *   关于调用shutdownNow（）
     * SHUTDOWN -> TIDYING
     *    当队列和池都为空时
     * STOP -> TIDYING
     *    池空时
     * TIDYING -> TERMINATED
     *     当terminate（）钩子方法完成时
     *
     * 当状态达到TERMINATED时，在awaitTermination（）中等待的线程将返回。
     *
     * 检测从SHUTDOWN到TIDYING的转换不如你想要的那么简单，因为在SHUTDOWN状态期间队列可能在非空后变为空，
     * 反之亦然，但我们只能在看到它为空后看到workerCount时终止 是0（有时需要重新检查 - 见下文）。
     *
     */
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    private static final int COUNT_BITS = Integer.SIZE - 3;
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

    // runState存储在高位中
    private static final int RUNNING    = -1 << COUNT_BITS;
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    private static final int STOP       =  1 << COUNT_BITS;
    private static final int TIDYING    =  2 << COUNT_BITS;
    private static final int TERMINATED =  3 << COUNT_BITS;

    public ThreadPoolExecutor(BlockingQueue<Runnable> workQueue) {
        this.workQueue = workQueue;
    }

    // 包装和拆包ctl
    private static int runStateOf(int c)     { return c & ~CAPACITY; }
    private static int workerCountOf(int c)  { return c & CAPACITY; }
    private static int ctlOf(int rs, int wc) { return rs | wc; }

    /*
     * 不需要解压缩ctl的位字段访问器。
     * 这些取决于位布局，而workerCount永远不会消极。
     */

    private static boolean runStateLessThan(int c, int s) {
        return c < s;
    }

    private static boolean runStateAtLeast(int c, int s) {
        return c >= s;
    }

    private static boolean isRunning(int c) {
        return c < SHUTDOWN;
    }

    /**
     * 尝试CAS增加ctl的workerCount字段。
     */
    private boolean compareAndIncrementWorkerCount(int expect) {
        return ctl.compareAndSet(expect, expect + 1);
    }

    /**
     * 尝试CAS减少ctl的workerCount字段。
     */
    private boolean compareAndDecrementWorkerCount(int expect) {
        return ctl.compareAndSet(expect, expect - 1);
    }

    /**
     * 减少ctl的workerCount字段。 只有在线程突然终止时才会调用此方法（请参阅processWorkerExit）。 其他减量在getTask中执行。
     */
    private void decrementWorkerCount() {
        do {} while (! compareAndDecrementWorkerCount(ctl.get()));
    }

    /**
     * 用于保存任务和切换到工作线程的队列。 我们不要求workQueue.poll（）返回null必然意味着workQueue.isEmpty（），
     * 因此只依赖于isEmpty来查看队列是否为空（我们必须这样做，例如在决定是否从SHUTDOWN转换为TIDYING时）。
     * 这适用于特殊用途的队列，例如DelayQueues，即使在延迟到期后它可能稍后返回非null，也允许poll（）返回null。
     */
    private final BlockingQueue<Runnable> workQueue;

    /**
     * 锁定访问worker集和相关的簿记。 虽然我们可以使用某种并发集合，但通常最好使用锁定。
     * 其中的原因是这会使interruptIdleWorkers序列化，从而避免不必要的中断风暴，特别是在关机期间。
     * 否则退出线程会同时中断那些尚未中断的线程。 它还简化了largePoolSize等的一些相关统计簿记。
     * 我们还在shutdown和shutdownNow上保存mainLock，以确保工作集稳定，同时单独检查中断和实际中断的权限。
     */
    private final ReentrantLock mainLock = new ReentrantLock();

    /**
     * 设置包含池中的所有工作线程。 仅在持有mainLock时访问。.
     */
    private final HashSet<Worker> workers = new HashSet<Worker>();

    /**
     * 等待条件以支持awaitTermination
     */
    private final Condition termination = mainLock.newCondition();

    /**
     * 跟踪最大的游泳池大小。 仅在mainLock下访问。.
     */
    private int largestPoolSize;

    /**
     * 已完成的任务数
     */
    private long completedTaskCount;


    private volatile ThreadFactory threadFactory;
    private volatile RejectedExecutionHandler handler;


    private volatile long keepAliveTime;


    private volatile boolean allowCoreThreadTimeOut;

    private volatile int corePoolSize;

    private volatile int maximumPoolSize;

    private static final RejectedExecutionHandler defaultHandler =
        new AbortPolicy();

    /**
     * Permission required for callers of shutdown and shutdownNow.
     * We additionally require (see checkShutdownAccess) that callers
     * have permission to actually interrupt threads in the worker set
     * (as governed by Thread.interrupt, which relies on
     * ThreadGroup.checkAccess, which in turn relies on
     * SecurityManager.checkAccess). Shutdowns are attempted only if
     * these checks pass.
     *
     * All actual invocations of Thread.interrupt (see
     * interruptIdleWorkers and interruptWorkers) ignore
     * SecurityExceptions, meaning that the attempted interrupts
     * silently fail. In the case of shutdown, they should not fail
     * unless the SecurityManager has inconsistent policies, sometimes
     * allowing access to a thread and sometimes not. In such cases,
     * failure to actually interrupt threads may disable or delay full
     * termination. Other uses of interruptIdleWorkers are advisory,
     * and failure to actually interrupt will merely delay response to
     * configuration changes so is not handled exceptionally.
     */
    private static final RuntimePermission shutdownPerm =
        new RuntimePermission("modifyThread");

    /**
     * Class Worker主要维护运行任务的线程的中断控制状态，以及其他小型簿记。
     * 这个类机会性地扩展AbstractQueuedSynchronizer以简化获取和释放围绕每个任务执行的锁。
     * 这可以防止意图唤醒等待任务的工作线程而不是中断正在运行的任务的中断。 我们实现了一个简单的非重入互斥锁而不是使用ReentrantLock，
     * 因为我们不希望工作任务在调用setCorePoolSize等池控制方法时能够重新获取锁。
     * 此外，为了在线程实际开始运行任务之前禁止中断，我们将锁定状态初始化为负值，并在启动时清除它（在runWorker中）。
     */
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

        /**
         * Creates with given first task and thread from ThreadFactory.
         * @param firstTask the first task (null if none)
         */
        Worker(Runnable firstTask) {
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }

        /** Delegates main run loop to outer runWorker  */
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

    /*
     * Methods for setting control state
     */

    /**
     * Transitions runState to given target, or leaves it alone if
     * already at least the given target.
     *
     * @param targetState the desired state, either SHUTDOWN or STOP
     *        (but not TIDYING or TERMINATED -- use tryTerminate for that)
     */
    private void advanceRunState(int targetState) {
        for (;;) {
            int c = ctl.get();
            if (runStateAtLeast(c, targetState) ||
                ctl.compareAndSet(c, ctlOf(targetState, workerCountOf(c))))
                break;
        }
    }

    /**
     * Transitions to TERMINATED state if either (SHUTDOWN and pool
     * and queue empty) or (STOP and pool empty).  If otherwise
     * eligible to terminate but workerCount is nonzero, interrupts an
     * idle worker to ensure that shutdown signals propagate. This
     * method must be called following any action that might make
     * termination possible -- reducing worker count or removing tasks
     * from the queue during shutdown. The method is non-private to
     * allow access from ScheduledThreadPoolExecutor.
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

    /*
     * Methods for controlling interrupts to worker threads.
     */

    /**
     * If there is a security manager, makes sure caller has
     * permission to shut down threads in general (see shutdownPerm).
     * If this passes, additionally makes sure the caller is allowed
     * to interrupt each worker thread. This might not be true even if
     * first check passed, if the SecurityManager treats some threads
     * specially.
     */
    private void checkShutdownAccess() {
        SecurityManager security = System.getSecurityManager();
        if (security != null) {
            security.checkPermission(shutdownPerm);
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                for (Worker w : workers)
                    security.checkAccess(w.thread);
            } finally {
                mainLock.unlock();
            }
        }
    }

    /**
     * Interrupts all threads, even if active. Ignores SecurityExceptions
     * (in which case some threads may remain uninterrupted).
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

    /**
     * Interrupts threads that might be waiting for tasks (as
     * indicated by not being locked) so they can check for
     * termination or configuration changes. Ignores
     * SecurityExceptions (in which case some threads may remain
     * uninterrupted).
     *
     * @param onlyOne If true, interrupt at most one worker. This is
     * called only from tryTerminate when termination is otherwise
     * enabled but there are still other workers.  In this case, at
     * most one waiting worker is interrupted to propagate shutdown
     * signals in case all threads are currently waiting.
     * Interrupting any arbitrary thread ensures that newly arriving
     * workers since shutdown began will also eventually exit.
     * To guarantee eventual termination, it suffices to always
     * interrupt only one idle worker, but shutdown() interrupts all
     * idle workers so that redundant workers exit promptly, not
     * waiting for a straggler task to finish.
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

    /**
     * Common form of interruptIdleWorkers, to avoid having to
     * remember what the boolean argument means.
     */
    private void interruptIdleWorkers() {
        interruptIdleWorkers(false);
    }

    private static final boolean ONLY_ONE = true;

    /*
     * Misc utilities, most of which are also exported to
     * ScheduledThreadPoolExecutor
     */

    /**
     * Invokes the rejected execution handler for the given command.
     * Package-protected for use by ScheduledThreadPoolExecutor.
     */
    final void reject(Runnable command) {
        handler.rejectedExecution(command, null);
    }

    /**
     * Performs any further cleanup following run state transition on
     * invocation of shutdown.  A no-op here, but used by
     * ScheduledThreadPoolExecutor to cancel delayed tasks.
     */
    void onShutdown() {
    }

    /**
     * State check needed by ScheduledThreadPoolExecutor to
     * enable running tasks during shutdown.
     *
     * @param shutdownOK true if should return true if SHUTDOWN
     */
    final boolean isRunningOrShutdown(boolean shutdownOK) {
        int rs = runStateOf(ctl.get());
        return rs == RUNNING || (rs == SHUTDOWN && shutdownOK);
    }

    /**
     * Drains the task queue into a new list, normally using
     * drainTo. But if the queue is a DelayQueue or any other kind of
     * queue for which poll or drainTo may fail to remove some
     * elements, it deletes them one by one.
     */
    private List<Runnable> drainQueue() {
        BlockingQueue<Runnable> q = workQueue;
        ArrayList<Runnable> taskList = new ArrayList<Runnable>();
        q.drainTo(taskList);
        if (!q.isEmpty()) {
            for (Runnable r : q.toArray(new Runnable[0])) {
                if (q.remove(r))
                    taskList.add(r);
            }
        }
        return taskList;
    }

    /**
     * 检查是否可以根据当前池状态和给定边界（核心或最大）添加新工作线程。如果是，则相应地调整工作者计数，并且如果可能，
     * 创建并启动新工作者，将firstTask作为其第一个任务运行。如果池已停止或有资格关闭，则此方法返回false。
     * 如果线程工厂在询问时无法创建线程，它也会返回false。如果线程创建失败，或者由于线程工厂返回null，
     * 或者由于异常（通常是Thread.start（）中的OutOfMemoryError），我们会干净地回滚。
     * @param firstTask新线程应先运行的任务（如果没有则为null）。使用初始第一个任务（在方法execute（）中）创建工作程序，
     *                           以便在少于corePoolSize线程时（在这种情况下我们始终启动一个）或队列已满（在这种情况下我们必须绕过队列）时绕过排队。
     *                           最初空闲线程通常通过prestartCoreThread创建或替换其他垂死的工作者。
     * @param core如果为true则使用corePoolSize作为绑定，否则为maximumPoolSize。 （此处使用布尔指示符而不是值，以确保在检查其他池状态后读取新值）。
     * @return如果成功则为true
     */
    private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
        }

        boolean workerStarted = false;
        boolean workerAdded = false;
        Worker w = null;
        try {
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());

                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }

    /**
     * Rolls back the worker thread creation.
     * - removes worker from workers, if present
     * - decrements worker count
     * - rechecks for termination, in case the existence of this
     *   worker was holding up termination
     */
    private void addWorkerFailed(Worker w) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            if (w != null)
                workers.remove(w);
            decrementWorkerCount();
            tryTerminate();
        } finally {
            mainLock.unlock();
        }
    }

    /**
     * Performs cleanup and bookkeeping for a dying worker. Called
     * only from worker threads. Unless completedAbruptly is set,
     * assumes that workerCount has already been adjusted to account
     * for exit.  This method removes thread from worker set, and
     * possibly terminates the pool or replaces the worker if either
     * it exited due to user task exception or if fewer than
     * corePoolSize workers are running or queue is non-empty but
     * there are no workers.
     *
     * @param w the worker
     * @param completedAbruptly if the worker died due to user exception
     */
    private void processWorkerExit(Worker w, boolean completedAbruptly) {
        if (completedAbruptly) // If abrupt, then workerCount wasn't adjusted
            decrementWorkerCount();

        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            completedTaskCount += w.completedTasks;
            workers.remove(w);
        } finally {
            mainLock.unlock();
        }

        tryTerminate();

        int c = ctl.get();
        if (runStateLessThan(c, STOP)) {
            if (!completedAbruptly) {
                int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
                if (min == 0 && ! workQueue.isEmpty())
                    min = 1;
                if (workerCountOf(c) >= min)
                    return; // replacement not needed
            }
            addWorker(null, false);
        }
    }

    /**
     * 执行阻塞或定时等待任务，具体取决于
     * 当前配置设置，如果此worker，则返回null
     * 必须退出，因为任何：
     * 1.有多个maximumPoolSize工作者（由于调用setMaximumPoolSize）。
     * 2.池停止了。
     * 3.池已关闭，队列为空。这名worker超时等待任务，并超时
     *     worker受到终止（即{@code allowCoreThreadTimeOut || workerCount> corePoolSize}）
     *     在定时等待之前和之后，如果队列是非空，此worker不是池中的最后一个线程。
     *
     * return 任务，如果worker必须退出，则返回null，在这种情况下，workerCount递减
     */
    private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }

            int wc = workerCountOf(c);

            // Are workers subject to culling?
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

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

    /**
     *主要worker运行循环。从队列中反复获取任务并执行它们，同时解决许多问题：
     * 1.我们可以从一个初始任务开始，在这种情况下，我们不需要获得第一个任务。否则，只要pool正在运行，我们就会从getTask获取任务。如果它返回null，则由于池状态或配置参数的更改而退出工作线程。其他退出是由外部代码中的异常抛出引起的，在这种情况下，completedAbruptly持有，这通常会导致processWorkerExit替换此线程。
     * 2.在运行任何任务之前，获取锁以防止在任务执行期间其他池中断，然后我们确保除非池停止，否则此线程没有设置其中断。
     * 3.每个任务运行之前都会调用beforeExecute，这可能会抛出异常，在这种情况下，我们会导致线程死亡（使用completedAbruptly打破循环为true）而不处理任务。
     * 4.假设beforeExecute正常完成，我们运行任务，收集任何抛出的异常以发送到afterExecute。我们分别处理RuntimeException，Error（两个规范保证我们陷阱）和任意Throwables。因为我们无法在Runnable.run中重新抛出Throwables，所以我们将它们包含在出错的Errors中（到线程的UncaughtExceptionHandler）。任何抛出的异常也会保守地导致线程死亡。
     * 5.在task.run完成之后，我们调用afterExecute，这也可能抛出异常，这也会导致线程死亡。根据JLS Sec 14.20，即使task.run抛出，该异常也将生效。异常机制的净效果是afterExecute和线程的UncaughtExceptionHandler具有我们可以提供的关于用户代码遇到的任何问题的准确信息。
     */
    final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            while (task != null || (task = getTask()) != null) {
                w.lock();
                // 如果池正在停止，请确保线程被中断;
                // 如果没有，确保线程不被中断。 这个
                // 需要在第二种情况下重新检查才能处理
                // shutdownNow在清除中断时进行比赛
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);
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


    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        /*
         * 执行3步:
         *
         * 1. 如果正在运行少于corePoolSize的线程，请尝试使用给定命令作为其第一个任务启动新线程。
         * 对addWorker的调用以原子方式检查runState和workerCount，因此通过返回false来防止在不应该添加线程时发生的错误警报。
         *
         * 2. 如果任务可以成功排队，那么我们仍然需要仔细检查是否应该添加一个线程（因为自上次检查后现有的线程已经死亡），
         * 或者自从进入此方法后池关闭了。 所以我们重新检查状态，如果必要的话，如果没有则回滚入队，或者如果没有，则启动新的线程。.
         *
         * 3. 如果我们不能排队任务，那么我们尝试添加一个新线程。 如果失败，我们知道我们已关闭或饱和，因此拒绝该任务。
         */
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        else if (!addWorker(command, false))
            reject(command);
    }

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

    /**
     *
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

    public boolean isShutdown() {
        return ! isRunning(ctl.get());
    }

    /**
     * 是否终止
     *
     * 如果此执行程序在{@link #shutdown}或{@link #shutdownNow}之后终止但尚未完全终止，则返回true。
     *  此方法可用于调试。 返回{@code true}报告关闭后足够的时间段可能表示提交的任务已忽略或抑制中断，导致此执行程序无法正确终止
     * @return {@code true} if terminating but not yet terminated
     */
    public boolean isTerminating() {
        int c = ctl.get();
        return ! isRunning(c) && runStateLessThan(c, TERMINATED);
    }

    public boolean isTerminated() {
        return runStateAtLeast(ctl.get(), TERMINATED);
    }

    public boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException {
        long nanos = unit.toNanos(timeout);
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (;;) {
                if (runStateAtLeast(ctl.get(), TERMINATED))
                    return true;
                if (nanos <= 0)
                    return false;
                nanos = termination.awaitNanos(nanos);
            }
        } finally {
            mainLock.unlock();
        }
    }

    /**
     * 调用shutdown，如果excutors没有更长的引用并且没有其他线程
     * Invokes {@code shutdown} when this executor is no longer
     * referenced and it has no threads.
     */
    protected void finalize() {
        shutdown();
    }


    public void setThreadFactory(ThreadFactory threadFactory) {
        if (threadFactory == null)
            throw new NullPointerException();
        this.threadFactory = threadFactory;
    }


    public ThreadFactory getThreadFactory() {
        return threadFactory;
    }


    public void setRejectedExecutionHandler(RejectedExecutionHandler handler) {
        if (handler == null)
            throw new NullPointerException();
        this.handler = handler;
    }

    public RejectedExecutionHandler getRejectedExecutionHandler() {
        return handler;
    }


    public void setCorePoolSize(int corePoolSize) {
        if (corePoolSize < 0)
            throw new IllegalArgumentException();
        int delta = corePoolSize - this.corePoolSize;
        this.corePoolSize = corePoolSize;
        if (workerCountOf(ctl.get()) > corePoolSize)
            interruptIdleWorkers();
        else if (delta > 0) {
            // We don't really know how many new threads are "needed".
            // As a heuristic, prestart enough new workers (up to new
            // core size) to handle the current number of tasks in
            // queue, but stop if queue becomes empty while doing so.
            int k = Math.min(delta, workQueue.size());
            while (k-- > 0 && addWorker(null, true)) {
                if (workQueue.isEmpty())
                    break;
            }
        }
    }


    public int getCorePoolSize() {
        return corePoolSize;
    }

    /**
     * 启动一个核心线程，导致它无所事事地等待工作。
     * 这将覆盖仅在执行新任务时启动核心线程的默认策略。
     * 如果所有核心线程都已启动，则此方法将返回{@code false}。
     *
     */
    public boolean prestartCoreThread() {
        return workerCountOf(ctl.get()) < corePoolSize &&
            addWorker(null, true);
    }

    /**
     * 与prestartCoreThread相同，至少启动一个线程，即使corePoolSize为0。
     */
    void ensurePrestart() {
        int wc = workerCountOf(ctl.get());
        if (wc < corePoolSize)
            addWorker(null, true);
        else if (wc == 0)
            addWorker(null, false);
    }

    /**
     * 启动所有核心线程，导致它们无所事事地等待工作。 这将覆盖仅在执行新任务时启动核心线程的默认策略。
     *
     */
    public int prestartAllCoreThreads() {
        int n = 0;
        while (addWorker(null, true))
            ++n;
        return n;
    }

    public boolean allowsCoreThreadTimeOut() {
        return allowCoreThreadTimeOut;
    }


    public void allowCoreThreadTimeOut(boolean value) {
        if (value && keepAliveTime <= 0)
            throw new IllegalArgumentException("Core threads must have nonzero keep alive times");
        if (value != allowCoreThreadTimeOut) {
            allowCoreThreadTimeOut = value;
            if (value)
                interruptIdleWorkers();
        }
    }


    public void setMaximumPoolSize(int maximumPoolSize) {
        if (maximumPoolSize <= 0 || maximumPoolSize < corePoolSize)
            throw new IllegalArgumentException();
        this.maximumPoolSize = maximumPoolSize;
        if (workerCountOf(ctl.get()) > maximumPoolSize)
            interruptIdleWorkers();
    }


    public int getMaximumPoolSize() {
        return maximumPoolSize;
    }

    public void setKeepAliveTime(long time, TimeUnit unit) {
        if (time < 0)
            throw new IllegalArgumentException();
        if (time == 0 && allowsCoreThreadTimeOut())
            throw new IllegalArgumentException("Core threads must have nonzero keep alive times");
        long keepAliveTime = unit.toNanos(time);
        long delta = keepAliveTime - this.keepAliveTime;
        this.keepAliveTime = keepAliveTime;
        if (delta < 0)
            interruptIdleWorkers();
    }


    public long getKeepAliveTime(TimeUnit unit) {
        return unit.convert(keepAliveTime, TimeUnit.NANOSECONDS);
    }


    public BlockingQueue<Runnable> getQueue() {
        return workQueue;
    }


    public boolean remove(Runnable task) {
        boolean removed = workQueue.remove(task);
        tryTerminate(); // In case SHUTDOWN and now empty
        return removed;
    }

    /**
     * 尝试从工作队列中删除已取消的所有{@link Future}任务。 此方法可用作存储回收操作，对功能没有其他影响。 取消的任务永远不会执行，
     * 但可能会累积在工作队列中，直到工作线程可以主动删除它们。 现在调用此方法会尝试删除它们。
     * 但是，此方法可能无法在存在其他线程干扰的情况下删除任务。
     */
    public void purge() {
        final BlockingQueue<Runnable> q = workQueue;
        try {
            Iterator<Runnable> it = q.iterator();
            while (it.hasNext()) {
                Runnable r = it.next();
                if (r instanceof Future<?> && ((Future<?>)r).isCancelled())
                    it.remove();
            }
        } catch (ConcurrentModificationException fallThrough) {
            //如果我们在遍历期间遇到干扰，请采取慢速路径。
            //为遍历创建副本，并为已取消的条目调用删除。
            //慢路径更可能是O（N * N）。
            for (Object r : q.toArray())
                if (r instanceof Future<?> && ((Future<?>)r).isCancelled())
                    q.remove(r);
        }

        tryTerminate(); // In case SHUTDOWN and now empty
    }

    public int getPoolSize() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            // Remove rare and surprising possibility of
            // isTerminated() && getPoolSize() > 0
            return runStateAtLeast(ctl.get(), TIDYING) ? 0
                : workers.size();
        } finally {
            mainLock.unlock();
        }
    }


    public int getActiveCount() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            int n = 0;
            for (Worker w : workers)
                if (w.isLocked())
                    ++n;
            return n;
        } finally {
            mainLock.unlock();
        }
    }

    public int getLargestPoolSize() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            return largestPoolSize;
        } finally {
            mainLock.unlock();
        }
    }

    /**
     * 返回已安排执行的大致任务总数。 因为任务和线程的状态可能在计算期间动态改变，所以返回
     *
     * @return the number of tasks
     */
    public long getTaskCount() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            long n = completedTaskCount;
            for (Worker w : workers) {
                n += w.completedTasks;
                if (w.isLocked())
                    ++n;
            }
            return n + workQueue.size();
        } finally {
            mainLock.unlock();
        }
    }

    /**
     * 返回已完成执行的大致任务总数。 因为任务和线程的状态可能在计算期间动态地改变，所以返回的值仅是近似值，而是在连续调用期间不会减少的值。
     * @return the number of tasks
     */
    public long getCompletedTaskCount() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            long n = completedTaskCount;
            for (Worker w : workers)
                n += w.completedTasks;
            return n;
        } finally {
            mainLock.unlock();
        }
    }

    public String toString() {
        long ncompleted;
        int nworkers, nactive;
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            ncompleted = completedTaskCount;
            nactive = 0;
            nworkers = workers.size();
            for (Worker w : workers) {
                ncompleted += w.completedTasks;
                if (w.isLocked())
                    ++nactive;
            }
        } finally {
            mainLock.unlock();
        }
        int c = ctl.get();
        String rs = (runStateLessThan(c, SHUTDOWN) ? "Running" :
                     (runStateAtLeast(c, TERMINATED) ? "Terminated" :
                      "Shutting down"));
        return super.toString() +
            "[" + rs +
            ", pool size = " + nworkers +
            ", active threads = " + nactive +
            ", queued tasks = " + workQueue.size() +
            ", completed tasks = " + ncompleted +
            "]";
    }

    /* Extension hooks */


    protected void beforeExecute(Thread t, Runnable r) { }

    /**
     * 完成给定Runnable的执行后调用的方法。 执行任务的线程调用此方法。 ‘如果非null，则Throwable是未被捕获的{@code RuntimeException}或{@code Error}，导致执行突然终止。
     *
     * <p>此实现不执行任何操作，但可以在子类中进行自定义。 注意：要正确嵌套多个重叠，子类通常应在此方法的开头调用{@code super.afterExecute}。
     *
     * <p> <b>注意：</ b>当任务明确地或通过{@code submit}等方法包含在任务（例如{@link FutureTask}）中时，
     * 这些任务对象会捕获并维护计算异常，并且 因此它们不会导致突然终止，并且内部异常<em>不</ em>传递给此方法。
     * 如果您希望在此方法中捕获两种类型的故障，则可以进一步探测此类情况，如此示例子类中，如果任务已中止，则会打印直接原因或基础异常：
     *  <pre> {@code
     * class ExtendedExecutor extends ThreadPoolExecutor {
     *   // ...
     *   protected void afterExecute(Runnable r, Throwable t) {
     *     super.afterExecute(r, t);
     *     if (t == null && r instanceof Future<?>) {
     *       try {
     *         Object result = ((Future<?>) r).get();
     *       } catch (CancellationException ce) {
     *           t = ce;
     *       } catch (ExecutionException ee) {
     *           t = ee.getCause();
     *       } catch (InterruptedException ie) {
     *           Thread.currentThread().interrupt(); // ignore/reset
     *       }
     *     }
     *     if (t != null)
     *       System.out.println(t);
     *   }
     * }}</pre>
     *
     */
    protected void afterExecute(Runnable r, Throwable t) { }

    /**
     * Executor终止时调用的方法。 默认实现什么都不做。
     * 注意：要正确嵌套多个重叠，子类通常应在此方法中调用{@code super.terminated}。
     */
    protected void terminated() { }

    /* Predefined RejectedExecutionHandlers */

    /**
     * 被拒绝任务的处理程序，它直接在{@code execute}方法的调用线程中运行被拒绝的任务，
     * 除非执行程序已关闭，在这种情况下，任务将被丢弃。
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
        public void rejectedExecution(Runnable r, java.util.concurrent.ThreadPoolExecutor e) {
            if (!e.isShutdown()) {
                r.run();
            }
        }
    }

    /**
     * 拒绝策略，throw RejectedExecutionException
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
         * @param executor the executor attempting to execute this task
         * @throws RejectedExecutionException always
         */
        public void rejectedExecution(Runnable r, java.util.concurrent.ThreadPoolExecutor executor) {
            throw new RejectedExecutionException("Task " + r.toString() +
                    " rejected from " +
                    executor.toString());
        }
    }

    /**
     * 拒绝策略，直接放弃任务，不做任何处理
     */
    public static class DiscardPolicy implements RejectedExecutionHandler {
        /**
         * Creates a {@code DiscardPolicy}.
         */
        public DiscardPolicy() { }

        @Override
        public void rejectedExecution(Runnable r, java.util.concurrent.ThreadPoolExecutor executor) {

        }
    }

    /**
     * 丢弃最老的任务
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
        @Override
        public void rejectedExecution(Runnable r, java.util.concurrent.ThreadPoolExecutor e) {
            if (!e.isShutdown()) {
                e.getQueue().poll();
                e.execute(r);
            }
        }
    }
}

```