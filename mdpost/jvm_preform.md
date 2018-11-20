---
title: JVM优化
---
![](/images/jvmpreform.jpg "")
JVM优化
<!--more-->
###jvm参数：
https://www.cnblogs.com/edwardlauxh/archive/2010/04/25/1918603.html
###调优：
http://blog.csdn.net/lifuxiangcaohui/article/details/37992725
#一：基本参数
参数	描述

  >-Xms	最小堆大小
  -Xmx	最大堆大小
  -Xmn	新生代大小
  -XX:PermSize	永久代大小
  -XX:MaxPermSize	永久代最大大小
  -XX:+PrintGC	输出GC日志
  -verbose:gc	-
  -XX:+PrintGCDetails	输出GC的详细日志
  -XX:+PrintGCTimeStamps	输出GC时间戳(以基准时间的形式)
  -XX:+PrintHeapAtGC	在进行GC的前后打印出堆的信息
  -Xloggc:/path/gc.log	日志文件的输出路径
  -XX:+PrintGCApplicationStoppedTime	打印由GC产生的停顿时间
#二：获取JVM的dump文件的两种方式
1. JVM启动时增加两个参数:
a.#出现 OOME 时生成堆 dump:-XX:+HeapDumpOnOutOfMemoryError
  #生成堆文件地址：-XX:HeapDumpPath=/home/liuke/jvmlogs/
2.发现程序异常前通过执行指令，直接生成当前JVM的dmp文件，6214是指JVM的进程号
  jmap -dump:format=b,file=serviceDump.dat 6214
备注：由于第一种方式是一种事后方式，需要等待当前JVM出现问题后才能生成dmp文件，实时性不高;
   第二种方式在执行时，JVM是暂停服务的，所以对线上的运行会产生影响。所以建议第一种方式。
1:大对象直接在老年代分配：
Serial和ParNew两款收集器提供了-XX:PretenureSizeThreshold的参数, 令大于该值的大对象直接在老年代分配, 这样做的目的是避免在Eden区和Survivor区之间产生大量的内存复制(大对象一般指 需要大量连续内存的Java对象, 如很长的字符串和数组), 因此大对象容易导致还有不少空闲内存就提前触发GC以获取足够的连续空间.

>-Xms768m -Xmx1280m  jvm堆的最小值和最大值设置，一般设成相同值，避免频繁分配堆空间
-XX:NewSize=128m -XX:MaxNewSize=128m  年轻代最小值和最大值设置（年轻代设定了，年老代也就定了），也可以用参数-XX:NewRatio=4，年老代和年轻代的大小比，这里128m有点小了，官方建议的是heap的3/8，差不多280m
-XX:PermSize=96m -XX:MaxPermSize=128m 持久代最小值和最大值设置
-XX:MaxTenuringThreshold=0  经过多少次minor gc 后进入年老代,设置为0的话直接进入年老代，这是不太合理的，正常应该在年轻代多呆一段时间，真正需要到年老代的才转过去
-XX:SurvivorRatio=20000  年轻代中eden和一块suvivor区的空间比例，这里设置成20000有问题，suvivor区空间几乎为0，一次minor gc后基本都转到年老代了，年轻代没有起到过滤左右
-XX:+UseParNewGC  年轻代采用并行gc策略，JDK5.0以上,JVM会根据系统配置自行设置,所以无需再设置此值。使用多线程收集，提高吞吐量（-XX:ParallelGCThreads 并行收集器的线程数，此值最好配置与处理器数目相等，如果超过当前cpu数，会加大机器负载）
-XX:+UseConcMarkSweepGC  年老代采用并发gc策略，和应用程序并发执行，减少pause time，但是需要更大的堆区，因为并发执行，有碎片（-XX:+UseParallelOldGC 年老代垃圾收集方式为并行收集，这个是Java6出现的参数选项）
-XX:+CMSPermGenSweepingEnabled  为了避免Perm区满引起的full gc，建议开启CMS回收Perm区选项
-verbose:gc -XX:+PrintGCDetails -XX:+PrintGCTimeStamps  打印gc日志
-XX:CMSInitiatingOccupancyFraction=1 年老代使用空间比达到这个值时开始cms gc,默认是在年老代占满68%的时候开始进行CMS收集，这里设置成1是不合理的，会导致CMS GC频繁发生，从gc日志里可以看出来，CMS GC和minor GC几乎一样多
-XX:+CMSIncrementalMode 启动i-CMS模式，增量模式，将cms gc过程分成6个阶段，其中阶段initial Mark和remark时需要pause，这6个阶段在两次minor gc的间隔期执行，具体执行起止时间由下面两个参数决定。拆分成小阶段增量执行时，可以避免应用被中断时间过长，极端情况是如果只有一个cpu，那么得等全部做完这6个阶段才能释放cpu，如果是多cpu这个模式开启与否应该影响不大。
-XX:CMSIncrementalDutyCycleMin=10 默认值10 启动CMS的下线
-XX:CMSIncrementalDutyCycle=30 默认值50 启动CMS的上线
-XX:+UseCMSCompactAtFullCollection  在FULL GC的时候， 对年老代的压缩。CMS是不会移动内存的， 因此这个非常容易产生碎片， 导致内存不够用， 因此， 内存的压缩这个时候就会被启用。 可能会影响性能,但是可以消除碎片，增加这个参数是个好习惯。
-XX:CMSFullGCsBeforeCompaction=0  上面配置开启的情况下,这里设置多少次Full GC后,对年老代进行压缩，这里设置成0不知道什么意思，可以根据线上full gc 的频率确定，频率高，这个值可以大点，比如5，反之频率低，这个值可以小点，比如1
-XX:CMSMarkStackSize=8M
-XX:CMSMarkStackSizeMax=32M
