---
title: synchronized底层原理
date: 2018-12-31 00:40:44
categories:
    - synchronized底层原理
tags:
  - 网络应用层
description: synchronized底层原理
---
synchronized如何实现同步的，底层实现原理是啥？什么是对象的内置锁，保存在哪里，等待和唤醒都是如何实现的，今天我们走进jvm锁的世界。
![](/images/lock/monitor.png "")
<!--more-->

# 问题
1.synchronized如何实现同步的？
2.synchronized对象锁保存在哪里？
3.如何获得对象锁和释放锁，以及1.6之后做了哪些优化？
3.如何将线程挂起等待和唤醒？

### synchronized 使用
测试类
```
public class SynchronizedTest {
    static SynchronizedTest synchronizedTest = new SynchronizedTest();
    public static synchronized void methodA(){
        try {
            Thread.sleep(1000);
            System.out.println("static methodA synchronized");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

    }
    public synchronized void methodB(){
        try {
            Thread.sleep(1000);
            System.out.println("public methodB synchronized");
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
    public void methodC(){

        synchronized (SynchronizedTest.class){
            try {
                Thread.sleep(1000);
                System.out.println("synchronized methodC synchronized");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
    public void methodD(){
        synchronized (synchronizedTest){
            try {
                Thread.sleep(1000);
                System.out.println("synchronizedTest methodD synchronized");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```
反编译：javap -verbose SynchronizedTest.class >sync.txt
可以看出使用synchronized之后的一些细微的变化，方法上的同步关键字，class文件反编译后会发现flags中有ACC_SYNCHRONIZED标识，
1.方法级的同步是隐式的。同步方法的常量池中会有一个ACC_SYNCHRONIZED标志。当某个线程要访问某个方法的时候，会检查是否有ACC_SYNCHRONIZED，
如果有设置，则需要先获得监视器锁，然后开始执行方法，方法执行之后再释放监视器锁。
2.同步关键字则是monitorenter和monitorexit相关指令操作来加锁和释放锁。
3.简单说一下1.5synchronized的实现。首先ObjectMonitor中维护了_EntryList队列，_owner和_WaitSet，EntryList是当前请求获取锁资源的所有竞争线程对象，然后owner对象是当前获得锁的线程，当调用exit方法后，也就是出同步方法或者代码块之后，会将monitor对象进行状态和锁记录信息变更，然后其他线程又可以重新进行锁竞争。如果调用了wait方法， 将会将该线程封装成等待节点信息，并存放到WaitSet中等待，并且将线程挂起，释放对象锁，此时owner对象会转到WaitSet中，当调用唤醒操作的时候，WaitSet等待的线程又会重新的请求锁资源。具体的可以看一下源码如何将线程打包成节点，存入WaitSet链表中，出队的时候调用DequeueWaiter方法移出_waiterSet第一个结点，只会唤醒第一个入队的线程。所以notify()方法并非是随机唤醒。下面是objectMonitor.hpp具体实现类objectMonitor.cpp的源码
![synchronized](/images/lock/java_synchronized.png "synchronized")

### 对象锁记录在哪？
![对象头](/images/lock/markworld.png "对象头")
>对象头信息简介：首先需要了解对象头信息，首先每一个对象在堆内存中，实例对象会有对象头信息，头信息中包含的信息又，锁状态，GC年纪，Hash值和监控monitor对象的引用等信息，具体可以参考下面的图。
synchronzied实现同步用到了对象的内置锁(ObjectMonitor),锁相关的计数器和线程竞争情况都会依赖monitor对象。下面看一下JVM中改对象头中的监控对象的具体实现，该对象是在HotSpot底层C++语言编写的。

具体可以在https://github.com/openjdk-mirror/jdk7u-hotspot/blob/50bdefc3afe944ca74c3093e7448d6b889cd20d1/src/share/vm/runtime/objectMonitor.cpp中查看hotspot-JVM具体源码的实现，其中objectMonitor.hpp中是该对象的结构的一些定义信息，包括对象属性和方法定义等，objectMonitor.cpp是具体的实现。首先我们看一下hpp文件中objectMonitor监控对象的结构。
![同步过程](/images/lock/monitor.png "同步过程")
```
ObjectMonitor() {
    _header       = NULL;//头结点
    _count        = 0;//锁计数器
    _waiters      = 0,//等待的线程数
    _recursions   = 0;//线程的重入次数
    _object       = NULL;
    _owner        = NULL;//当前持有锁的线程对象
    _WaitSet      = NULL;//等待线程组成的双向循环链表，_WaitSet是第一个节点
    _WaitSetLock  = 0 ;
    _Responsible  = NULL ;
    _succ         = NULL ;
    _cxq          = NULL ;//多线程竞争锁进入时的单向链表
    FreeNext      = NULL ;
    _EntryList    = NULL ;//_owner从该双向循环链表中唤醒线程结点，_EntryList是第一个节点
    _SpinFreq     = 0 ;
    _SpinClock    = 0 ;
    OwnerIsThread = 0 ;
  }
```
objectMonitor.hpp中相关方法，看到这些大家可能并不陌生，每个对象都会有相应Object中的方法，其中wait和notify，就是超类中的方法。其实底层实现就在这里。
```
  bool      try_enter (TRAPS) ;//尝试获取锁，返回Boolean
  void      enter(TRAPS);//获取锁，jvm触发monitorenter指令的时候，底层将调用该方法
  void      exit(TRAPS);//释放锁，jvm触发monitorexit指令的时候调用
  void      wait(jlong millis, bool interruptable, TRAPS);//获得对象锁，调用对象的wait()方法，将挂起线程
  void      notify(TRAPS);//调用对象的notify()方法，唤醒其中的一个等待线程
  void      notifyAll(TRAPS);//调用对象的notifyAll()方法，将唤醒所有的等待线程
```
### 重量级加锁过程
说明：jdk1.6版本之前，同步关键字采用的是重锁，所以先讲一下重锁的获取和释放，后比在谈谈1.6以后的锁升级优化
>1.enter(TRAPS)是重量级锁的获锁过程，属于1.6之前的做法，1.6 jvm进行锁升级相关的优化，下面会针对jvm-hotspot锁升级源码来分析锁升级的过程。
2.cmpxchg(void* ptr, int old, int new)，如果ptr和old的值一样，则把new写到ptr内存，否则返回ptr的值，整个操作是原子的。
3.参考：https://www.cnblogs.com/kundeg/p/8422557.html
![重锁加锁过程](/images/lock/锁entry.png "重锁加锁过程")
```
void ATTR ObjectMonitor::enter(TRAPS) {
  // The following code is ordered to check the most common cases first
  // and to reduce RTS->RTO cache line upgrades on SPARC and IA32 processors.
  //首先检查最常见的情况 并减少SPARC和IA32处理器上的RTS-> RTO缓存线升级。
  //self 为当前线程
  Thread * const Self = THREAD ;
  void * cur ;
  //调用cmpxchg函数进行cas原子替换
  cur = Atomic::cmpxchg_ptr (Self, &_owner, NULL) ;
  //如果当前cur为null，则获取锁失败
  if (cur == NULL) {
     // Either ASSERT _recursions == 0 or explicitly set _recursions = 0.
     assert (_recursions == 0   , "invariant") ;
     assert (_owner      == Self, "invariant") ;
     // CONSIDER: set or assert OwnerIsThread == 1
     return ;
  }
  //如果cas比较替换之后，如果之前的_owner指向该THREAD，那么该线程是重入，_recursions++
  if (cur == Self) {
     // TODO-FIXME: check for integer overflow!  BUGID 6557169.
     _recursions ++ ;//重入锁次数
     return ;
  }
  //如果当前线程是第一次进入该monitor，设置_recursions为1，并且把当前线程赋值给_owner，标识当前线程已经获得锁
  if (Self->is_lock_owned ((address)cur)) {
    assert (_recursions == 0, "internal state error");
    _recursions = 1 ;
    // Commute owner from a thread-specific on-stack BasicLockObject address to
    // a full-fledged "Thread *".
    _owner = Self ;
    OwnerIsThread = 1 ;
    return ;
  }

  // We've encountered genuine contention.
  assert (Self->_Stalled == 0, "invariant") ;
  Self->_Stalled = intptr_t(this) ;

  // Try one round of spinning *before* enqueueing Self
  // and before going through the awkward and expensive state
  // transitions.  The following spin is strictly optional ...
  // Note that if we acquire the monitor from an initial spin
  // we forgo posting JVMTI events and firing DTRACE probes.
  //尝试一轮旋转*之前*排队Self并在经历尴尬和昂贵的状态转换之前。
  //以下旋转是严格可选的...请注意，如果我们从初始旋转中获取监视器，我们将放弃发布JVMTI事件并触发DTRACE探测器。
  if (Knob_SpinEarly && TrySpin (Self) > 0) {
     assert (_owner == Self      , "invariant") ;
     assert (_recursions == 0    , "invariant") ;
     assert (((oop)(object()))->mark() == markOopDesc::encode(this), "invariant") ;
     Self->_Stalled = 0 ;
     return ;
  }

  // Prevent deflation at STW-time.  See deflate_idle_monitors() and is_busy().
  // Ensure the object-monitor relationship remains stable while there's contention.
  //获得锁，则原子增加_count值，表示锁的计数器
  Atomic::inc_ptr(&_count);

  { // Change java thread status to indicate blocked on monitor enter.
    //更改java线程状态，并且阻止其他线程进入监视器。
    JavaThreadBlockedOnMonitorEnterState jtbmes(jt, this);

    DTRACE_MONITOR_PROBE(contended__enter, this, object(), jt);
    if (JvmtiExport::should_post_monitor_contended_enter()) {
      JvmtiExport::post_monitor_contended_enter(jt, this);
    }

    OSThreadContendState osts(Self->osthread());
    ThreadBlockInVM tbivm(jt);

    Self->set_current_pending_monitor(this);

    //自旋执行ObjectMonitor::EnterI方法等待锁的释放
    for (;;) {
      jt->set_suspend_equivalent();
      // cleared by handle_special_suspend_equivalent_condition()
      // or java_suspend_self()
      //没有获得锁的线程，放入_EntryList中
      EnterI (THREAD) ;

      if (!ExitSuspendEquivalent(jt)) break ;

      // We have acquired the contended monitor, but while we were
      // waiting another thread suspended us. We don't want to enter
      // the monitor while suspended because that would surprise the
      // thread that suspended us.
      //已经获得锁的线程，如果被其他线程暂停，当前线程就需要退出监视器
          _recursions = 0 ;
      _succ = NULL ;
      exit (Self) ;

      jt->java_suspend_self();
    }
    Self->set_current_pending_monitor(NULL);
  }
  //锁如果释放，则原子递减_count值
  Atomic::dec_ptr(&_count);
  assert (_count >= 0, "invariant") ;
  Self->_Stalled = 0 ;

  DTRACE_MONITOR_PROBE(contended__entered, this, object(), jt);
  if (JvmtiExport::should_post_monitor_contended_entered()) {
    JvmtiExport::post_monitor_contended_entered(jt, this);
  }
  if (ObjectMonitor::_sync_ContendedLockAttempts != NULL) {
     ObjectMonitor::_sync_ContendedLockAttempts->inc() ;
  }
}
```
### 重量级锁释放
锁的释放，需要做哪些事情？也就是当方法执行同步代码块之后，退出锁的逻辑，下面是具体的释放锁的过程。
就是唤醒正在等待竞争的线程，也就是需要唤醒内置锁ObjectMonitor中的_EntryList 中的线程，重新获取锁，并且执行同步代码。
![重锁出锁过程](/images/lock/锁exit.png "重锁出锁过程")
下面是具体的代码
```
void ATTR ObjectMonitor::exit(TRAPS) {
   Thread * Self = THREAD ;
   if (THREAD != _owner) {
     if (THREAD->is_lock_owned((address) _owner)) {
       assert (_recursions == 0, "invariant") ;
       _owner = THREAD ;
       _recursions = 0 ;
       OwnerIsThread = 1 ;
     } else {
       TEVENT (Exit - Throw IMSX) ;
       assert(false, "Non-balanced monitor enter/exit!");
       if (false) {
          THROW(vmSymbols::java_lang_IllegalMonitorStateException());
       }
       return;
     }
   }

   if (_recursions != 0) {
     _recursions--;        // this is simple recursive enter
     TEVENT (Inflated exit - recursive) ;
     return ;
   }

   if ((SyncFlags & 4) == 0) {
      _Responsible = NULL ;
   }

   for (;;) {
      assert (THREAD == _owner, "invariant") ;
      if (Knob_ExitPolicy == 0) {
         OrderAccess::release_store_ptr (&_owner, NULL) ;   // drop the lock
         OrderAccess::storeload() ;                         // See if we need to wake a successor
         if ((intptr_t(_EntryList)|intptr_t(_cxq)) == 0 || _succ != NULL) {
            TEVENT (Inflated exit - simple egress) ;
            return ;
         }
         TEVENT (Inflated exit - complex egress) ;

         if (Atomic::cmpxchg_ptr (THREAD, &_owner, NULL) != NULL) {
            return ;
         }
         TEVENT (Exit - Reacquired) ;
      } else {
         if ((intptr_t(_EntryList)|intptr_t(_cxq)) == 0 || _succ != NULL) {
            OrderAccess::release_store_ptr (&_owner, NULL) ;   // drop the lock
            OrderAccess::storeload() ;
            // Ratify the previously observed values.
            if (_cxq == NULL || _succ != NULL) {
                TEVENT (Inflated exit - simple egress) ;
                return ;
            }

            if (Atomic::cmpxchg_ptr (THREAD, &_owner, NULL) != NULL) {
               TEVENT (Inflated exit - reacquired succeeded) ;
               return ;
            }
            TEVENT (Inflated exit - reacquired failed) ;
         } else {
            TEVENT (Inflated exit - complex egress) ;
         }
      }

      guarantee (_owner == THREAD, "invariant") ;

      ObjectWaiter * w = NULL ;
      int QMode = Knob_QMode ;

      //QMode == 2：从_cxq队列中取出线程，并且调用unpark唤醒
      if (QMode == 2 && _cxq != NULL) {
          w = _cxq ;
          assert (w != NULL, "invariant") ;
          assert (w->TState == ObjectWaiter::TS_CXQ, "Invariant") ;
          ExitEpilog (Self, w) ;//这里会唤醒，unpark _EntryList中等待的线程
          return ;
      }
      //QMode == 3:将_cxq队列尾插入_EntryList中
      if (QMode == 3 && _cxq != NULL) {
          w = _cxq ;
          //省去一些代码
          ObjectWaiter * Tail ;
          for (Tail = _EntryList ; Tail != NULL && Tail->_next != NULL ; Tail = Tail->_next) ;
          if (Tail == NULL) {
              _EntryList = w ;
          } else {
              Tail->_next = w ;
              w->_prev = Tail ;
          }
      }
      //QMode == 4：将_cxq队列头插入_EntryList中
      if (QMode == 4 && _cxq != NULL) {
          w = _cxq ;

          if (_EntryList != NULL) {
              q->_next = _EntryList ;
              _EntryList->_prev = q ;
          }
          _EntryList = w ;

      }

      w = _EntryList  ;
      if (w != NULL) {
          assert (w->TState == ObjectWaiter::TS_ENTER, "invariant") ;
          ExitEpilog (Self, w) ;//这里会唤醒，unpark _EntryList中等待的线程
          return ;
      }

      w = _cxq ;
      if (w == NULL) continue ;

      // Drain _cxq into EntryList - bulk transfer.
      // First, detach _cxq.
      // The following loop is tantamount to: w = swap (&cxq, NULL)
      for (;;) {
          assert (w != NULL, "Invariant") ;
          ObjectWaiter * u = (ObjectWaiter *) Atomic::cmpxchg_ptr (NULL, &_cxq, w) ;
          if (u == w) break ;
          w = u ;
      }
      TEVENT (Inflated exit - drain cxq into EntryList) ;

      //QMode == 1? _EntryList为倒置后的cxq队列
      if (QMode == 1) {
         // QMode == 1 : drain cxq to EntryList, reversing order
         // We also reverse the order of the list.
         ObjectWaiter * s = NULL ;
         ObjectWaiter * t = w ;
         ObjectWaiter * u = NULL ;
         while (t != NULL) {
             guarantee (t->TState == ObjectWaiter::TS_CXQ, "invariant") ;
             t->TState = ObjectWaiter::TS_ENTER ;
             u = t->_next ;
             t->_prev = u ;
             t->_next = s ;
             s = t;
             t = u ;
         }
         _EntryList  = s ;
         assert (s != NULL, "invariant") ;
      } else {
         // QMode == 0 or QMode == 2
         _EntryList = w ;
         ObjectWaiter * q = NULL ;
         ObjectWaiter * p ;
         for (p = w ; p != NULL ; p = p->_next) {
             guarantee (p->TState == ObjectWaiter::TS_CXQ, "Invariant") ;
             p->TState = ObjectWaiter::TS_ENTER ;
             p->_prev = q ;
             q = p ;
         }
      }

      if (_succ != NULL) continue;

      w = _EntryList  ;
      if (w != NULL) {
          guarantee (w->TState == ObjectWaiter::TS_ENTER, "invariant") ;
          ExitEpilog (Self, w) ;//唤醒unpark _EntryList中等待的线程
          return ;
      }
   }
}
```
### 锁升级-偏向锁
1.6之后 synchronized进行优化，加入了锁升级相关内容，出现了偏向锁，轻量级锁，重量级锁，自旋等优化，不会像之前那样直接调用enter方法获取重量级锁。
#####偏向锁加锁过程
>1. 首先判断是否开启偏向锁设置，然后调用revoke_and_rebias
2. 获取对象头中的markword信息，判断偏向锁标识位01，如果不是偏向锁，则尝试轻量级锁获取
3. 如果是偏向锁，判断偏向锁ID，为null则cas替换线程ID，成功则获得锁，并执行同步代码，如果线程ID和当前线程ID一致，直接获得锁成功执行代码
4. 第三步cas替换线程ID失败，则撤销偏向锁，升级为轻量级获取锁
5. 线程竞争，判断线程是否退出同步代码，没有则升级为轻量级锁。

##### 偏向锁源码
源码：https://github.com/openjdk-mirror/jdk7u-hotspot/blob/50bdefc3af/src/share/vm/runtime/synchronizer.cpp
```
//快速进入对象监听器和退出方法，这里理解为偏向锁的概念
void ObjectSynchronizer::fast_enter(Handle obj, BasicLock* lock, bool attempt_rebias, TRAPS) {
//1.判断是否开启偏向锁功能，判断UseBiasedLocking，
 if (UseBiasedLocking) {
    //判断jvm是否在一个安全点，不在安全点则调用revoke_and_rebias进行判断.
    if (!SafepointSynchronize::is_at_safepoint()) {
     //revoke_and_rebias的调用过程，它的意思是撤销和重新安排，如果Condition是3代表偏向线程已经撤销并成功替换，获得锁
      BiasedLocking::Condition cond = BiasedLocking::revoke_and_rebias(obj, attempt_rebias, THREAD);
      if (cond == BiasedLocking::BIAS_REVOKED_AND_REBIASED) {
        return;
      }
    } else {
      //如果是安全点，调用revoke_at_safepoint进行偏向锁撤销
      assert(!attempt_rebias, "can not rebias toward VM thread");
      BiasedLocking::revoke_at_safepoint(obj);
    }
    assert(!obj->mark()->has_bias_pattern(), "biases should be revoked by now");
 }
 //如果获取偏向锁过程中，偏向失败得到偏见撤销状态（BIAS_REVOKED）， 则调用slow_enter 轻量级锁升级.
 slow_enter (obj, lock, THREAD) ;
}
```
判断是否是安全点，is_at_safepoint，其实就是判断SynchronizeState的状态，这里的状态都是实时记录的和变更的。
```
enum SynchronizeState {
    _not_synchronized = 0,                   // Threads not synchronized at a safepoint
                                             // Keep this value 0. See the coment in do_call_back()
    _synchronizing    = 1,                   // Synchronizing in progress
    _synchronized     = 2                    // All Java threads are stopped at a safepoint. Only VM thread is running
};
```
看一下BiasedLocking::revoke_and_rebias，该方法源码在biasedLocking.cpp文件中这里基本上就会展示真正的对象头信息获取和判断等，看不懂不要紧，先mark下来，有空仔细研究。
BiasedLocking中定义的几个条件状态枚举
```
enum Condition {
  NOT_BIASED = 1,//没有偏见
  BIAS_REVOKED = 2,//偏见撤销
  BIAS_REVOKED_AND_REBIASED = 3 //偏见撤销并且重新安排
};
```
### 偏向锁获取
首先了解markOop中锁状态的枚举值，和上面的Mark头信息一致
```
 enum {
         locked_value             = 0,//00偏向锁
         unlocked_value           = 1,//01无锁
         monitor_value            = 2,//10监视器锁，又叫重量级锁
         marked_value             = 3,//11GC标记
         biased_lock_pattern      = 5//101偏向锁
  };
```
Atomic::cmpxchg_ptr hotspot源码文件在Atomic.hpp中
static intptr_t cmpxchg_ptr(intptr_t exchange_value, volatile intptr_t* dest, intptr_t compare_value);
下面是源码的注解翻译：
 1.执行* dest和compare_value的原子比较，并用exchange_value交换* dest
 2.如果比较成功，返回* dest的先前值。 保证双向记忆
 3.跨越cmpxchg的障碍。 即，它真的是'fence_cmpxchg_acquire'。

```
BiasedLocking::Condition BiasedLocking::revoke_and_rebias(Handle obj, bool attempt_rebias, TRAPS) {
  assert(!SafepointSynchronize::is_at_safepoint(), "must not be called while at safepoint");
  markOop mark = obj->mark();
  //1.判断对象头中的信息is_biased_anonymously，是否偏向锁（也就是判断枚举值是否是101）
  //2.如果是偏向锁则cas更新对象头信息markword，如果CAS偏向撤销失败，则返回状态让调用者处理
  if (mark->is_biased_anonymously() && !attempt_rebias) {
    //注释大致翻译：在此数据类型的批量撤销发生之前，此对象具有偏向，此时更新启发式方法毫无意义，因此只需使用CAS更新标头即可。
    //如果CAS失败，那么对象的偏向已被另一个线程撤销，所以我们只需返回并让调用者处理它。
    //这里cas替换：预期unbiased_prototype，当前值：obj->mark_addr()
    markOop biased_value       = mark;
    markOop unbiased_prototype = markOopDesc::prototype()->set_age(mark->age());
    markOop res_mark = (markOop) Atomic::cmpxchg_ptr(unbiased_prototype, obj->mark_addr(), mark);
    if (res_mark == biased_value) {
      return BIAS_REVOKED;//这里如果cas替换失败，返回偏向锁撤销
    }
  }
  //如果当前对象已经是偏见模式
  else if (mark->has_bias_pattern()) {
    //获取obj->klass类对象
    Klass* k = Klass::cast(obj->klass());
    //类对象头信息属性值
    markOop prototype_header = k->prototype_header();
    //如果类对象不是偏向模式，即该类对象没有同步静态方法或者属性
    if (!prototype_header->has_bias_pattern()) {
      //翻译：
      //由于身份哈希码计算，我们可能试图撤销此对象的偏见。 尝试在没有安全点的情况下撤销偏见。
      //如果我们能够成功地将无偏的头部与对象的标记词进行比较和交换，这是可能的，这意味着没有其他线程竞相获取对象的偏差。
      //首先类对象上没有偏向锁，则CAS替换对象头信息，成功则返回BIAS_REVOKED，偏向已撤销
      markOop biased_value       = mark;
      markOop res_mark = (markOop) Atomic::cmpxchg_ptr(prototype_header, obj->mark_addr(), mark);
      assert(!(*(obj->mark_addr()))->has_bias_pattern(), "even if we raced, should still be revoked");
      return BIAS_REVOKED;//偏见撤销
    }
    //如果prototype_header的偏向统治期 和 实例的对象头信息的统治期不一致
    else if (prototype_header->bias_epoch() != mark->bias_epoch()) {
      //翻译：
      //这种偏见的时代已经过期，表明该对象实际上是无偏的。 根据我们是否需要重新调整或撤销此对象的偏差，我们可以使用CAS来有效地完成它，
      //我们不应该更新启发式算法。这通常在汇编代码中完成，但由于运行时需要撤消偏差的各个点，我们可以达到这一点。
      if (attempt_rebias) {
        //这里如果可以允许尝试重新偏向操作，则 cas 替换rebiased_prototype 新的头信息和老的头信息替换成功，则偏向锁撤销成功并且替换成新的线程信息和年龄信息。
        //成功则返回BIAS_REVOKED_AND_REBIASED：标识偏向撤销并且重新安排其他的线程THREAD
        assert(THREAD->is_Java_thread(), "");
        markOop biased_value       = mark;
        markOop rebiased_prototype = markOopDesc::encode((JavaThread*) THREAD, mark->age(), prototype_header->bias_epoch());
        markOop res_mark = (markOop) Atomic::cmpxchg_ptr(rebiased_prototype, obj->mark_addr(), mark);
        if (res_mark == biased_value) {
          return BIAS_REVOKED_AND_REBIASED;//偏见撤销并且重新安排
        }
      } else {
        // 如果不允许重新偏向，则cas替换其他信息，不会替换偏向线程信息，成功则返回偏向撤销
        markOop biased_value       = mark;
        markOop unbiased_prototype = markOopDesc::prototype()->set_age(mark->age());
        markOop res_mark = (markOop) Atomic::cmpxchg_ptr(unbiased_prototype, obj->mark_addr(), mark);
        if (res_mark == biased_value) {
          return BIAS_REVOKED;//偏见撤销
        }
      }
    }
  }
  //启发式：英文Heuristic；简化虚拟机和简化行为判断引擎的结合。计算机专业术语，这里应该是启发式或者探测式的更新对象中的偏向状态
 HeuristicsResult heuristics = update_heuristics(obj(), attempt_rebias);
 //1.如果是无偏向，直接返回
  if (heuristics == HR_NOT_BIASED) {
    return NOT_BIASED;

  }
  //如果是 HR_SINGLE_REVOKE 表示需要唤醒通知撤销
  else if (heuristics == HR_SINGLE_REVOKE) {
    Klass *k = Klass::cast(obj->klass());
    markOop prototype_header = k->prototype_header();
    if (mark->biased_locker() == THREAD &&
        prototype_header->bias_epoch() == mark->bias_epoch()) {
      //一个线程试图撤销偏向它的对象的偏差，再次可能是由于哈希码
      //在这种情况下，我们可以再次避免使用安全点，因为我们只会走自己的堆栈。 在其他线程中没有发生撤销的比较，
      //因为我们在撤销路径中没有达到安全点。
      //同时检查纪元(epoch)，因为即使线程匹配，另一个线程也可以使用CAS来窃取具有陈旧时期的对象的偏差。
      ResourceMark rm;
      if (TraceBiasedLocking) {
        tty->print_cr("Revoking bias by walking my own stack:");
      }
      //这里撤销偏向，cond=偏见撤销状态
      BiasedLocking::Condition cond = revoke_bias(obj(), false, false, (JavaThread*) THREAD);
      ((JavaThread*) THREAD)->set_cached_monitor_info(NULL);
      assert(cond == BIAS_REVOKED, "why not?");
      return cond;//偏见撤销
    } else {
      //其他的状态，直接撤销偏向，并返回status_code
      VM_RevokeBias revoke(&obj, (JavaThread*) THREAD);
      VMThread::execute(&revoke);
      return revoke.status_code();
    }
  }

  assert((heuristics == HR_BULK_REVOKE) ||
         (heuristics == HR_BULK_REBIAS), "?");
  //启发式更新之后，如果状态时HR_BULK_REVOKE，表示是批量撤销
  VM_BulkRevokeBias bulk_revoke(&obj, (JavaThread*) THREAD,
                                (heuristics == HR_BULK_REBIAS),
                                attempt_rebias);
  VMThread::execute(&bulk_revoke);
  return bulk_revoke.status_code();
}
```
### 偏向锁获取流程
    其实就是偏向锁的撤销和重新偏向过程
    1.主要是判断当前对象头信息中是否是偏向模式，如果不是，直接替换对象头信息，成功，则获得锁成功，失败则返回偏向锁撤销，之后升级轻量级锁
    2.如果当前对象已经有线程在占用，即如果是偏向模式，
        a：需要获取类对象是否是偏向模式，如果不是偏向模式，cas替换该对象的对象头属性值
        b：如果是偏向模式，但是两个对象信息不是一个统治期的，则cas替换当前的对象头信息，包括线程信息，成功则返回BIAS_REVOKED_AND_REBIASED，表示获得锁成功
    3.最后都没有返回值的情况，最终会走到update_heuristics，启发式或者说是探索是更新的方法，获取更新后的状态，然后同样会进行撤销偏向操作

### 偏向锁安全点的撤销
偏向锁的释放，需要等待全局安全点（在这个时间点上没有正在执行的字节码），它会首先暂停拥有偏向锁的线程，然后检查持有偏向锁的线程是否还活着，如果线程不处于活动状态，则将对象头设置成无锁状态。如果线程仍然活着，拥有偏向锁的栈会被执行，遍历偏向对象的所记录。栈帧中的锁记录和对象头的Mark Word要么重新偏向其他线程，要么恢复到无锁，或者标记对象不适合作为偏向锁。最后唤醒暂停的线程。
偏向锁在Java 1.6之后是默认启用的，但在应用程序启动几秒钟之后才激活，可以使用-XX:BiasedLockingStartupDelay=0参数关闭延迟，如果确定应用程序中所有锁通常情况下处于竞争状态，可以通过XX:-UseBiasedLocking=false参数关闭偏向锁。
BiasedLocking::revoke_at_safepoint(obj);逻辑和非安全点的撤销逻辑差不多
```
void BiasedLocking::revoke_at_safepoint(Handle h_obj) {
  assert(SafepointSynchronize::is_at_safepoint(), "must only be called while at safepoint");
  oop obj = h_obj();
  HeuristicsResult heuristics = update_heuristics(obj, false);
  if (heuristics == HR_SINGLE_REVOKE) {
    revoke_bias(obj, false, false, NULL);
  } else if ((heuristics == HR_BULK_REBIAS) ||
             (heuristics == HR_BULK_REVOKE)) {
    bulk_revoke_or_rebias_at_safepoint(obj, (heuristics == HR_BULK_REBIAS), false, NULL);
  }
  clean_up_cached_monitor_info();
}
```
### 锁升级-轻量级锁
```
void ObjectSynchronizer::slow_enter(Handle obj, BasicLock* lock, TRAPS) {
  markOop mark = obj->mark();
  assert(!mark->has_bias_pattern(), "should not see bias pattern here");

  //如果是无锁状态,cas设置有锁状态set_displaced_header
  if (mark->is_neutral()) {
    // Anticipate successful CAS -- the ST of the displaced mark must
    // be visible <= the ST performed by the CAS.
    //设置替换头
    lock->set_displaced_header(mark);
    if (mark == (markOop) Atomic::cmpxchg_ptr(lock, obj()->mark_addr(), mark)) {
      TEVENT (slow_enter: release stacklock) ;
      return ;
    }
  }
  //如果有锁 并且当前线程是owner，则设置头对象信息为null
  else if (mark->has_locker() && THREAD->is_lock_owned((address)mark->locker())) {
    assert(lock != mark->locker(), "must not re-lock the same lock");
    assert(lock != (BasicLock*)obj->mark(), "don't relock with same BasicLock");
    lock->set_displaced_header(NULL);
    return;
  }

#if 0
  // The following optimization isn't particularly useful.一下优化不是很有用
  //如果有监控信息，并且是竞争等待等待线程，设置对象头信息为null
  if (mark->has_monitor() && mark->monitor()->is_entered(THREAD)) {
    lock->set_displaced_header (NULL) ;
    return ;
  }
#endif
  //轻量级锁膨胀，升级为重锁，并且调用enter，将线程放入竞争队列中挂起
  lock->set_displaced_header(markOopDesc::unused_mark());
  ObjectSynchronizer::inflate(THREAD, obj())->enter(THREAD);
}
```

### 轻量级锁撤销
```
void ObjectSynchronizer::fast_exit(oop object, BasicLock* lock, TRAPS) {
  assert(!object->mark()->has_bias_pattern(), "should not see bias pattern here");
  // if displaced header is null, the previous enter is recursive enter, no-op
  markOop dhw = lock->displaced_header();
  markOop mark ;
  //如果置换标头为空，则前一个输入为递归输入，无操作
  if (dhw == NULL) {
     // Recursive stack-lock.
     // Diagnostics -- Could be: stack-locked, inflating, inflated.
     mark = object->mark() ;
     assert (!mark->is_neutral(), "invariant") ;
     if (mark->has_locker() && mark != markOopDesc::INFLATING()) {
        assert(THREAD->is_lock_owned((address)mark->locker()), "invariant") ;
     }
     if (mark->has_monitor()) {
        ObjectMonitor * m = mark->monitor() ;
        assert(((oop)(m->object()))->mark() == mark, "invariant") ;
        assert(m->is_entered(THREAD), "invariant") ;
     }
     return ;
  }

  mark = object->mark() ;

  // If the object is stack-locked by the current thread, try to
  // swing the displaced header from the box back to the mark.
  //如果对象被当前线程堆积锁定，请尝试将移位头信息 从几个中移回标记。
  if (mark == (markOop) lock) {
     assert (dhw->is_neutral(), "invariant") ;
     if ((markOop) Atomic::cmpxchg_ptr (dhw, object->mark_addr(), mark) == mark) {
        TEVENT (fast_exit: release stacklock) ;
        return;
     }
  }
  //调用exit，将线程
  ObjectSynchronizer::inflate(THREAD, object)->exit (THREAD) ;
}
```
### 轻量级锁膨胀
轻量级锁膨胀升级返回一个ObjectMonitor对象，改对象就是一开始讨论的1.5版本的重锁，其中会维护一个当前占用线程Owner对象，其他的都会进入等待队列进行竞争。
```
ObjectMonitor * ATTR ObjectSynchronizer::inflate (Thread * Self, oop object) {
  assert (Universe::verify_in_progress() ||
          !SafepointSynchronize::is_at_safepoint(), "invariant") ;
  for (;;) { // 为后面的continue操作提供自旋
      const markOop mark = object->mark() ; //获得Mark Word结构
      assert (!mark->has_bias_pattern(), "invariant") ;

      //Mark Word可能有以下几种状态:
      // *  Inflated(膨胀完成)     - just return
      // *  Stack-locked(轻量级锁) - coerce it to inflated
      // *  INFLATING(膨胀中)     - busy wait for conversion to complete
      // *  Neutral(无锁)        - aggressively inflate the object.
      // *  BIASED(偏向锁)       - Illegal.  We should never see this

      if (mark->has_monitor()) {//判断是否是重量级锁
          ObjectMonitor * inf = mark->monitor() ;
          assert (inf->header()->is_neutral(), "invariant");
          assert (inf->object() == object, "invariant") ;
          assert (ObjectSynchronizer::verify_objmon_isinpool(inf), "monitor is invalid");
          //Mark->has_monitor()为true，说明已经是重量级锁了，膨胀过程已经完成，返回
          return inf ;
      }
      if (mark == markOopDesc::INFLATING()) { //判断是否在膨胀
         TEVENT (Inflate: spin while INFLATING) ;
         ReadStableMark(object) ;
         continue ; //如果正在膨胀，自旋等待膨胀完成
      }

      if (mark->has_locker()) { //如果当前是轻量级锁
          ObjectMonitor * m = omAlloc (Self) ;//返回一个对象的内置ObjectMonitor对象
          m->Recycle();
          m->_Responsible  = NULL ;
          m->OwnerIsThread = 0 ;
          m->_recursions   = 0 ;
          m->_SpinDuration = ObjectMonitor::Knob_SpinLimit ;//设置自旋获取重量级锁的次数
          //CAS操作标识Mark Word正在膨胀
          markOop cmp = (markOop) Atomic::cmpxchg_ptr (markOopDesc::INFLATING(), object->mark_addr(), mark) ;
          if (cmp != mark) {
             omRelease (Self, m, true) ;
             continue ;   //如果上述CAS操作失败，自旋等待膨胀完成
          }
          m->set_header(dmw) ;
          m->set_owner(mark->locker());//设置ObjectMonitor的_owner为拥有对象轻量级锁的线程，而不是当前正在inflate的线程
          m->set_object(object);
          /**
          *省略了部分代码
          **/
          return m ;
      }
  }
}
```
### wait操作
调用wait之后，当前线程就会被挂起，放弃对象锁，这个过程中JVM底层逻辑是通过调用相关的wait函数，将线程封装成相应的节点然后添加到等待队列中，只有调用notify方法或者超时自己唤醒。下面是jvm的源码方法，部分代码已经省略。
```
void ObjectMonitor::wait(jlong millis, bool interruptible, TRAPS) {
 Thread * const Self = THREAD ;
// check for a pending interrupt
//将self线程封装成node节点
 ObjectWaiter node(Self);
 //添加到Waiter集合中
   AddWaiter (&node) ;
    _waiters++;
   //调用退出监控器
   exit (Self) ;
   //调用线程阻塞park
if (node._notified == 0) {
    if (millis <= 0) {
       Self->_ParkEvent->park () ;
    } else {
       ret = Self->_ParkEvent->park (millis) ;
    }
  }
}
```

### notify操作
当对象触发notify时，首先会根据不同的策略模式操作_cxq队列中的元素和EntryList等待的线程节点，然后唤醒waitSet中的线程。
这样唤醒的线程又可以重新获得_owner,获得并占有内置锁。
```
void ObjectMonitor::notify(TRAPS) {
//策略
 int Policy = Knob_MoveNotifyee ;
//遍历wait集合
ObjectWaiter * iterator = DequeueWaiter() ;
  if (Policy == 1)     // append to EntryList
  if (Policy == 2)       // prepend to cxq
  if (Policy == 3)       // append to cxq
//iterator第一个元素，唤醒改线程
  ParkEvent * ev = iterator->_event ;
  iterator->TState = ObjectWaiter::TS_RUN ;
  OrderAccess::fence() ;
  ev->unpark() ;
}
```
### 总结
以上就是hotspot-jvm中同步关键字加锁流程，以及锁升级和等待唤醒的源码介绍，部分源码读起来很艰涩，有很多地方没有完全读懂，有兴趣的可以上github自行研究。
研究的主要目的是加深jvm锁的概念和原理，从不同的角度去思考问题，技术才会有所提升。

