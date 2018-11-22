---
title: 线程池的使用规范和线程池大小设置
date: 2018-06-01 23:47:44
---
![](/images/ThreadPoolExecutor.jpg "")

频繁的创建和销毁线程，会大大降低系统的效率。可能出现服务器在为每个请求创建新线程和销毁线程上花费的时间和消耗的系统资源要比处理实际的用户请求的时间和资源更多。
所以线程池可以缓存线程，需要的时候会从先从线程池中分配。其次根据不同的策略来创建线程和缓存线程。
<!--more-->
>微信公众号：MyClass社区
如有问题或建议，请公众号留言

###线程池的使用案例：
```
//Positive example 1
ScheduledExecutorService executorService = new ScheduledThreadPoolExecutor(1,
    new BasicThreadFactory.Builder().namingPattern("example-schedule-pool-%d").daemon(true).build());

//Positive example 2：
ThreadFactory namedThreadFactory = new ThreadFactoryBuilder()
    .setNameFormat("demo-pool-%d").build();

ExecutorService pool = new ThreadPoolExecutor(
        5,
        200,
        0L,
        TimeUnit.MILLISECONDS,
        new LinkedBlockingQueue<Runnable>(1024),
        namedThreadFactory,
        new ThreadPoolExecutor.AbortPolicy());
pool.execute(()-> System.out.println(Thread.currentThread().getName()));
pool.shutdown();//gracefully shutdown 优雅关闭

//Positive example 3：Spring管理
<bean id="userThreadPool" class="org.springframework.scheduling.semaphore.ThreadPoolTaskExecutor">
    <property name="corePoolSize" value="10" />
    <property name="maxPoolSize" value="100" />
    <property name="queueCapacity" value="2000" />
    <property name="threadFactory" value= threadFactory />
    <property name="rejectedExecutionHandler">
         <ref local="rejectedExecutionHandler" />
    </property>
</bean>
```
上面写法来自阿里规范手册，规范中明确线程池不允许使用Executors去创建，而是通过ThreadPoolExecutor的方式，这样的处理方式让写的同学更加明确线程池的运行规则，规避资源耗尽的风险。当然我觉得在风险可控的情况下，还是可以使用的。
Executors各个方法的弊端说明：
     1）newFixedThreadPool和newSingleThreadExecutor:
       主要问题是堆积的请求处理队列可能会耗费非常大的内存，甚至OOM。
     2）newCachedThreadPool和newScheduledThreadPool:
       主要问题是线程数最大数是Integer.MAX_VALUE，可能会创建数量非常多的线程，甚至OOM。

###首先看一下ThreadPool的几个参数
#####1.corePoolSize:核心线程数。
a.核心线程会一直存活，没有任务也会一直等待接受任务，
b.当线程数小于核心线程数时，即使有线程空闲，线程池也会优先创建新线程处理
c.设置allowCoreThreadTimeout=true（默认false）时，核心线程会超时关闭
#####2.maximumPoolSize：最大线程数
a.任务线程数 >= corePoolSize:线程池不会新建线程去处理请求，会先将task放入阻塞队列。
b.当阻塞队列满了之后，则再创建新的线程，任务线程数 = maxPoolSize，且任务队列已满时,还处理不了的才会走拒绝策略。

#####3.keepAliveTime：线程保持活跃时间（又称线程空闲时间）
a.如果过了任务处理高峰，线程处于空闲状态，过了keepAliveTime时间，线程会被销毁，直到线程数等于核心线程数
b.如果设置了允许核心线程超时（allowCoreThreadTimeout=true），线程超过了keepAliveTime，核心线程也会被销毁，最终线程池中的线程为0
c.设置了allowCoreThreadTimeout=true，这样核心线程空闲之后会销毁，如果活跃时间设置不合理，每次任务来都相当与有一段时间会延时创建核心线程。我觉得适合大时间段任务处理导致线程一直存活带来的资源浪费。比如一些定时worker，一天可能就跑几个小时。

#####4.unit：时间单位。这个是时间单位，支持毫秒，秒，分钟：
>TimeUnit.DAYS;               //天
TimeUnit.HOURS;             //小时
TimeUnit.MINUTES;           //分钟
TimeUnit.SECONDS;           //秒
TimeUnit.MILLISECONDS;      //毫秒
TimeUnit.MICROSECONDS;      //微妙
TimeUnit.NANOSECONDS;       //纳秒

#####5.workQueue：超过核心处理能力之后任务缓冲用的阻塞队列
这个是任务的等待队列，当任务在自己系统的处理范围之内，又支持不了那么多并发的情况，可以将任务进行排队，当然在一些秒杀或者大促的时候，这个阻塞队列一定要设置大小，要不然流量风暴会撑爆队列，将系统打挂。这里队列的策略完全是一个保护措施，正真的高并发场景下，更多用它来完成削锋限流，通过拒绝策略来处理或者落地一些没有来得及处理的一些流量，或者对拒绝的用户进行一次补偿，当然限流还有更好的方式，比如令牌桶等方式。

#####6.handler：拒绝策略
a.如果没有设置handler会构造默认AbortPolicy的策略，当然也可以自定义
b.当线程数已经达到maxPoolSize，切队列已满，会拒绝新任务。默认的策略会直接抛出运行时异常
c.ThreadPoolExcecutor还封装了一个DiscardPolicy，里边啥也没有实现，如果选择这个代表会直接忽略拒绝的任务；
d.DiscardOldestPolicy:会将最先入队的任务剔除
e.CallerRunsPolicy：会使用当前线程来执行任务；

###如何设置线程池大小
大家都知道线程是个好东西，但不是绝对的，线程的创建和销毁需要时间和资源开销，多线程的执行会有cpu上线文切换的开销，如果cpu计算密集型的操作很多，而频繁的采用多线程其实意义不大，因为cpu密集型的程序中使用多线程并没有单线程来的实在，比如我们比较熟悉的redis就是一个单线程，他多有的操作都在内存中完成，在忽略io操作的情况下，选择了单线程。所以使用线程池的时候需要思考这些东西。
今天不过多讨论IO密集型和cpu密集型，这里说一下线程池的大小设置，在平时的开发中经常会用到，一般的我们的业务代码很多都会有网络调用，IO操作，图片处理，存储，查询db等。
这些都多多少少的会让cpu处于一个等待网络响应和IO响应的状态，所以合理的设置线程池的大小来提高程序的吞吐量尤为重要。
1.简单设置，根据CPU的核数，来设置cpu密集型：N+1，IO密集型：2N+1,这种比较粗糙。
2.下面是一个任务执行的时间估值：总处理时间为120ms
    taskTime：程序的处理所需时间100ms
    cpuTime:cpu计算处理时间：20ms
    core:cpu核数：4核
计算公式：最佳的线程池大小：size = (taskTimes+cpuTime)/cpuTime *4 = (taskTimes/cpuTime+1)*4 = (100/20+1)*4 = 24

###Executors创建线程池
1.Executors类：并发包中提供了几种常用的线程池用法，这里可以了解一下，
Executors并不提倡我们直接使用ThreadPoolExecutor，而是使用Executors类中提供的四个静态方法来创建线程池
但是实际的并发高的系统开发中尽量要规避上面说的一些风险，所以阿里线程池的使用还是很有态度的。但是Executors的几个封装处理相应场景的时候是很有价值的。下面是
Executors中的几个常用的线程池创建方法。
（1）、newCachedThreadPool：创建可缓存线程池，如果线程池长度超过处理需要，可灵活回收空闲线程，若无可回收，则新建线程。
```
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
```
（2）、newFixedThreadPool：创建一个固定大小的线程池，可控制线程最大并发数，超出的线程会在队列中等待
```
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
```

（3）、newScheduledThreadPool：创建一个定长线程池，适用定时及周期性任务执行
```
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
        return new ScheduledThreadPoolExecutor(corePoolSize);
    }
```
（4）、newSingleThreadExecutor：创建一个线程容量为1的线程池，它只会用唯一的工作线程来执行任务，保证所有任务按照指定顺序执行。里边使用的是LinkedBlockingQueue
```
public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
```

#总结
我们对于线程池的使用应该是很普遍的，但是要想用好需要结合自己的业务场景合理配置相应的参数，才能发挥它的价值。今天主要总结了一下线程池大小的设置，以及线程池的参数说明和Executor
的几个开辟线程池方式。下一篇是关于线程池源码的一些具体实现，这样可以帮助大家更深刻的理解和使用线程池。
