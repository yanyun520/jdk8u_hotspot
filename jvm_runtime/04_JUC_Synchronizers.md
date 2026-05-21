# JUC 核心同步器与 HotSpot 底层交互分析

> 基于 OpenJDK 8u HotSpot 源码，分析 JUC 同步器如何调用 JVM 底层代码

---

## 一、JUC 同步器与 HotSpot 的调用关系总览

```
┌──────────────────────────────────────────────────────────────────────┐
│                        Java 层面 (JUC)                               │
│                                                                      │
│  ┌─────────────────┐  ┌─────────────────┐  ┌──────────────────┐   │
│  │  ReentrantLock    │  │  Semaphore      │  │  CountDownLatch   │   │
│  │  ReentrantRWLock  │  │  CyclicBarrier  │  │  Phaser           │   │
│  │  StampedLock      │  │  Exchanger      │  │  BlockingQueue    │   │
│  └────────┬─────────┘  └────────┬────────┘  └────────┬──────────┘   │
│           │                      │                     │            │
│  ┌────────▼──────────────────────▼─────────────────────▼──────────┐  │
│  │              AbstractQueuedSynchronizer (AQS)                  │  │
│  │  ┌─────────────────────────────────────────────────────────┐  │  │
│  │  │  - state (volatile int)  ← 状态变量                     │  │  │
│  │  │  - CLH 队列（双向链表）                                   │  │  │
│  │  │  - compareAndSetState()  ← CAS 操作                     │  │  │
│  │  │  - LockSupport.park()   ← 阻塞线程                     │  │  │
│  │  │  - LockSupport.unpark() ← 唤醒线程                     │  │  │
│  │  └─────────────────────────────────────────────────────────┘  │  │
│  └─────────────────────────┬──────────────────────────────────────┘  │
│                            │                                         │
│  ┌─────────────────────────▼──────────────────────────────────────┐  │
│  │                    LockSupport                                    │  │
│  │  park(Object blocker)  ─→ Unsafe.park(false, 0)                 │  │
│  │  unpark(Thread thread) ─→ Unsafe.unpark(thread)                 │  │
│  └─────────────────────────┬──────────────────────────────────────┘  │
│                            │                                         │
│  ┌─────────────────────────▼──────────────────────────────────────┐  │
│  │                    sun.misc.Unsafe                               │  │
│  │  compareAndSwapInt()    ─→ Atomic::cmpxchg()                    │  │
│  │  compareAndSwapLong()   ─→ Atomic::cmpxchg()                    │  │
│  │  compareAndSwapObject() ─→ oopDesc::atomic_compare_exchange()  │  │
│  │  park()                 ─→ Parker::park()                        │  │
│  │  unpark()               ─→ Parker::unpark()                     │  │
│  │  getObjectVolatile()    ─→ OrderAccess::load_acquire()          │  │
│  │  putObjectVolatile()   ─→ OrderAccess::release_store()         │  │
│  │  putOrderedObject()    ─→ OrderAccess::release_store()           │  │
│  │  getAndAddInt()        ─→ Atomic::add()                         │  │
│  └─────────────────────────┬──────────────────────────────────────┘  │
│                            │ JNI                                     │
└────────────────────────────┼────────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────────┐
│                     HotSpot JVM 层                                  │
│                                                                     │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐  │
│  │ Atomic::cmpxchg() │  │ Parker::park()   │  │ OrderAccess      │  │
│  │ (CPU CAS 指令)    │  │ Parker::unpark()  │  │ (内存屏障)        │  │
│  │ lock cmpxchg      │  │ (OS 条件变量)     │  │ mfence/lock addl │  │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘  │
│                                                                     │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐  │
│  │ ObjectSynchronizer│  │ ObjectMonitor    │  │ markOop          │  │
│  │ (锁膨胀/释放)     │  │ (重量级监视器)    │  │ (Mark Word)      │  │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 二、AQS 与 HotSpot 的交互

### 2.1 AQS 核心字段与 CAS

```java
// java/util/concurrent/locks/AbstractQueuedSynchronizer.java

public abstract class AbstractQueuedSynchronizer extends AbstractOwnableSynchronizer {
    // 等待队列节点
    static final class Node {
        volatile int waitStatus;      // 节点状态 (SIGNAL, CANCELLED, CONDITION, PROPAGATE)
        volatile Node prev;           // 前驱节点
        volatile Node next;           // 后继节点
        volatile Thread thread;       // 等待的线程
        Node nextWaiter;              // 条件队列后继

        // CAS 操作（使用 Unsafe）
        private boolean compareAndSetWaitStatus(int expect, int update) {
            return unsafe.compareAndSwapInt(this, waitStatusOffset, expect, update);
        }
        private boolean compareAndSetNext(Node expect, Node update) {
            return unsafe.compareAndSwapObject(this, nextOffset, expect, update);
        }
    }

    // 等待队列头尾
    private volatile Node head;
    private volatile Node tail;
    
    // 同步状态
    private volatile int state;

    // CAS 操作
    protected final boolean compareAndSetState(int expect, int update) {
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }
    
    private boolean compareAndSetHead(Node expect, Node update) {
        return unsafe.compareAndSwapObject(this, headOffset, expect, update);
    }
    
    private boolean compareAndSetTail(Node expect, Node update) {
        return unsafe.compareAndSwapObject(this, tailOffset, expect, update);
    }
}
```

### 2.2 AQS CAS 调用链

```
AQS.compareAndSetState(expect, update)
  │
  ▼
Unsafe.compareAndSwapInt(this, stateOffset, expect, update)
  │
  ▼ (JNI)
Unsafe_CompareAndSwapInt(JNIEnv *env, jobject unsafe, jobject obj, jlong offset, jint e, jint x)
  │  源码: src/share/vm/prims/unsafe.cpp
  │
  ▼
oop p = JNIHandles::resolve(obj);                    // 解析 Java 对象
jint* addr = (jint *) index_oop_from_field_offset_long(p, offset);  // 计算字段地址
return (jint)(Atomic::cmpxchg(x, addr, e)) == e;     // CPU CAS 指令
  │
  ▼
x86: lock cmpxchg [addr], eax  ← 硬件级原子操作

调用链图解：
  ┌────────────────┐     ┌─────────────┐     ┌──────────────┐     ┌───────────┐
  │  AQS (Java)    │────→│ Unsafe (JNI) │────→│ Atomic::cmpxchg│────→│ CPU CAS   │
  │  CAS操作       │     │ unsafe.cpp   │     │ atomic.hpp    │     │ lock cmpxchg│
  └────────────────┘     └─────────────┘     └──────────────┘     └───────────┘
```

### 2.3 AQS park/unpark 调用链

```
AQS 中线程阻塞与唤醒的调用链：

LockSupport.park(this)
  │
  ▼
Unsafe.park(false, 0L)
  │
  ▼ (JNI)
Unsafe_Park(JNIEnv *env, jobject unsafe, jboolean isAbsolute, jlong time)
  │  源码: src/share/vm/prims/unsafe.cpp
  │
  ▼
thread->parker()->park(isAbsolute != 0, time)
  │  源码: src/share/vm/runtime/park.cpp
  │
  ▼
Parker::park(bool isAbsolute, jlong time)
  │  1. 原子检查 _counter > 0（unpark 已先调用）
  │  2. 如果 _counter > 0，重置为 0 并返回
  │  3. 否则在操作系统条件变量上等待
  │
  ▼
os::PlatformParker::park()  ← 操作系统层
  │  Linux: pthread_cond_timedwait()
  │  Windows: WaitForSingleObject()
  │  macOS: pthread_cond_timedwait()


LockSupport.unpark(thread)
  │
  ▼
Unsafe.unpark(thread)
  │
  ▼ (JNI)
Unsafe_Unpark(JNIEnv *env, jobject unsafe, jobject jthread)
  │  源码: src/share/vm/prims/unsafe.cpp
  │
  ▼
p = thr->parker();  // 获取线程的 Parker 对象
p->unpark();
  │
  ▼
Parker::unpark()
  │  1. 设置 _counter = 1
  │  2. 如果线程正在等待，signal 唤醒
  │
  ▼
os::PlatformParker::unpark()  ← 操作系统层
  │  Linux: pthread_cond_signal()
  │  Windows: SetEvent()
```

---

## 三、ReentrantLock 与 HotSpot 的交互

### 3.1 ReentrantLock 获取锁流程

```
ReentrantLock.lock()
  │
  ▼
Sync.acquire(1)  ← AQS 模板方法
  │
  ├── tryAcquire(1)  ← NonfairSync/FairSync 实现
  │     │
  │     ▼
  │   compareAndSetState(0, 1)  ← CAS 尝试获取锁
  │     │
  │     ├── 成功 → 获取锁，设置 exclusiveOwnerThread = currentThread
  │     │
  │     └── 失败 → 检查是否当前线程重入
  │           │
  │           ├── 是 → state++  ← 不需要 CAS（已持有锁）
  │           │
  │           └── 否 → 返回 false，进入 acquire 流程
  │
  └── tryAcquire 失败 → acquireQueued(addWaiter(Node.EXCLUSIVE), 1)
        │
        ├── addWaiter()  ← 创建节点加入 CLH 队列
        │     │
        │     ▼
        │   compareAndSetTail(tail, node)  ← CAS 入队
        │
        └── acquireQueued()  ← 自旋获取锁或阻塞
              │
              ├── shouldParkAfterFailedAcquire()  ← 检查前驱状态
              │     将前驱的 waitStatus 设为 SIGNAL (CAS)
              │
              └── parkAndCheckInterrupt()  ← 阻塞当前线程
                    │
                    ▼
                  LockSupport.park(this)
                    │
                    ▼
                  Unsafe.park(false, 0L)
                    │
                    ▼
                  Parker::park()  ← HotSpot 底层
```

### 3.2 ReentrantLock 释放锁流程

```
ReentrantLock.unlock()
  │
  ▼
Sync.release(1)  ← AQS 模板方法
  │
  ├── tryRelease(1)  ← Sync 实现
  │     │
  │     ▼
  │   state -= 1  ← 不需要 CAS（已持有锁）
  │     │
  │     ├── state == 0 → 设置 exclusiveOwnerThread = null, state = 0
  │     │                 返回 true（完全释放）
  │     │
  │     └── state > 0 → 返回 false（还有重入）
  │
  └── tryRelease 成功 → unparkSuccessor(head)  ← 唤醒后继节点
        │
        ▼
      LockSupport.unpark(thread)
        │
        ▼
      Unsafe.unpark(thread)
        │
        ▼
      Parker::unpark()  ← HotSpot 底层
        │
        ▼
      被唤醒线程从 parkAndCheckInterrupt() 返回
        │
        ▼
      继续自旋 tryAcquire() → 获取锁成功
```

### 3.3 ReentrantLock vs synchronized

```
┌───────────────────┬──────────────────────┬───────────────────────┐
│     特性           │   synchronized        │   ReentrantLock        │
├───────────────────┼──────────────────────┼───────────────────────┤
│ 锁实现            │ HotSpot ObjectMonitor │ AQS (CLH 队列 + CAS)  │
│ 阻塞方式          │ ObjectMonitor::enter()│ LockSupport.park()     │
│ 唤醒方式          │ ObjectMonitor::exit() │ LockSupport.unpark()   │
│ CAS 实现          │ Atomic::cmpxchg_ptr() │ Unsafe.compareAndSwap  │
│ 可中断            │ 不可中断              │ 可中断 lockInterruptibly│
│ 公平性            │ 非公平                │ 公平/非公平可选          │
│ 条件变量          │ wait/notify           │ Condition.await/signal  │
│ 超时获取          │ wait(millis)          │ tryLock(timeout)       │
│ 锁状态查询        │ 不支持                │ getQueueLength()等      │
│ 锁绑定条件        │ 单一条件              │ 多条件 (Condition)      │
└───────────────────┴──────────────────────┴───────────────────────┘

底层差异：
  synchronized:
    偏向锁 → 轻量级锁 → 重量级锁 (ObjectMonitor)
    等待队列: _cxq + _EntryList (ObjectMonitor 内部)
    线程阻塞: ParkEvent (操作系统条件变量)

  ReentrantLock:
    CAS → CLH 队列等待 → LockSupport.park()
    等待队列: AQS 双向链表 (Node 队列)
    线程阻塞: Parker (操作系统条件变量)
```

---

## 四、Atomic 原子类与 HotSpot 的交互

### 4.1 AtomicInteger CAS 调用链

```
AtomicInteger.compareAndSet(expect, update)
  │
  ▼
Unsafe.compareAndSwapInt(this, valueOffset, expect, update)
  │
  ▼ (JNI)
Unsafe_CompareAndSwapInt()
  │
  ▼
Atomic::cmpxchg(update, &value, expect)
  │
  ▼
x86: lock cmpxchg [addr], eax  ← 总线锁 + CAS

AtomicInteger.getAndIncrement()
  │
  ▼
Unsafe.getAndAddInt(this, valueOffset, 1)
  │
  ▼
循环:
  int current = getIntVolatile(this, valueOffset);  // volatile 读
  int next = current + 1;
  if (compareAndSwapInt(this, valueOffset, current, next))  // CAS
    return current;
  // CAS 失败则重试（自旋）
```

### 4.2 Unsafe volatile 读写与内存屏障

```java
// Unsafe 中的 volatile 读写方法

// volatile 读（带 LoadLoad + LoadStore 屏障）
public native int getIntVolatile(Object obj, long offset);

// volatile 写（带 StoreStore + StoreLoad 屏障）
public native void putIntVolatile(Object obj, long offset, int value);

// 普通有序写（只有 StoreStore 屏障，无 StoreLoad）
public native void putOrderedObject(Object obj, long offset, Object value);
public native void putOrderedInt(Object obj, long offset, int value);
public native void putOrderedLong(Object obj, long offset, long value);
```

```
HotSpot 中的实现（unsafe.cpp）：

getIntVolatile():
  1. 调用 OrderAccess::load_acquire() 读取值
  2. 在 x86 上等同于普通读（TSO 保证）
  3. 在 ARM 上需要 dmb 指令

putIntVolatile():
  1. 调用 OrderAccess::release_store() 写入值
  2. 然后调用 OrderAccess::storeload() 屏障
  3. 在 x86 上需要 lock addl $0, (rsp)

putOrderedInt():
  1. 调用 OrderAccess::release_store() 写入值
  2. 不需要 StoreLoad 屏障（比 putIntVolatile 更快）
  3. 适用于不需要立即对其他线程可见的写操作

源码位置：src/share/vm/prims/unsafe.cpp
```

### 4.3 LongAdder 与 Striped64

```
LongAdder 的 CAS 调用链：

Striped64.longAccumulate()
  │
  ├── Cells 为空 → CAS base 值
  │     │
  │     ▼
  │   compareAndSwapLong(this, BASE_OFFSET, b, b + x)
  │
  └── Cells 非空 → CAS Cell 值
        │
        ▼
      compareAndSwapLong(cell, VALUE_OFFSET, v, v + x)
        │
        ├── 成功 → 返回
        │
        └── 失败 → 扩展 Cells 或重试

Striped64 使用 @sun.misc.Contended 注解避免伪共享：
  @sun.misc.Contended
  static final class Cell {
      volatile long value;
      // 每个 Cell 占一个缓存行（64字节），避免伪共享
  }
```

---

## 五、CountDownLatch 与 HotSpot 的交互

### 5.1 CountDownLatch 实现分析

```java
// CountDownLatch 使用 AQS 的共享模式

// await() 调用链
CountDownLatch.await()
  │
  ▼
Sync.acquireSharedInterruptibly(1)
  │
  ├── tryAcquireShared(1) → getState() == 0 ? 1 : -1
  │     如果 state == 0（计数器归零），直接返回
  │
  └── state > 0 → doAcquireSharedInterruptibly(node)
        │
        ├── addWaiter(Node.SHARED)  ← 创建共享节点
        │
        ├── shouldParkAfterFailedAcquire()  ← 设置前驱为 SIGNAL
        │
        └── parkAndCheckInterrupt()  ← LockSupport.park()
              │
              ▼
            Unsafe.park(false, 0L)  ← Parker::park()

// countDown() 调用链
CountDownLatch.countDown()
  │
  ▼
Sync.releaseShared(1)
  │
  ├── tryReleaseShared(1) → decrement state, CAS
  │     │
  │     ▼
  │   compareAndSetState(c, c - 1)  ← CAS 减计数
  │     │
  │     ├── 成功 && c == 1 → 返回 true（计数器归零）
  │     │
  │     └── 失败 → 重试
  │
  └── 返回 true → doReleaseShared()  ← 唤醒等待线程
        │
        ├── unparkSuccessor(head)  ← 唤醒头节点的后继
        │     │
        │     ▼
        │   LockSupport.unpark(thread)
        │     │
        │     ▼
        │   Parker::unpark()  ← HotSpot 底层
        │
        └── 传播唤醒（共享模式可能唤醒多个线程）
```

---

## 六、Semaphore 与 HotSpot 的交互

```
Semaphore.acquire() 调用链：

Sync.acquireSharedInterruptibly(permits)
  │
  ├── tryAcquireShared(permits)
  │     NonfairSync: compareAndSetState(available, available - permits)
  │     FairSync:     先检查 CLH 队列，再 CAS
  │
  └── CAS 失败 → 加入 CLH 队列 → LockSupport.park()

Semaphore.release() 调用链：

Sync.releaseShared(permits)
  │
  ├── tryReleaseShared(permits)
  │     compareAndSetState(c, c + permits)  ← CAS 增加许可
  │
  └── 成功 → doReleaseShared() → unparkSuccessor() → Parker::unpark()
```

---

## 七、CyclicBarrier 与 HotSpot 的交互

```
CyclicBarrier 使用 ReentrantLock + Condition 实现：

CyclicBarrier.await()
  │
  ▼
lock.lock()  ← ReentrantLock 获取锁
  │
  ▼
int index = --count;  ← 减计数（在锁保护下，不需要 CAS）
  │
  ├── index == 0 (最后一个线程)
  │     │
  │     ▼
  │   trip.signalAll()  ← 唤醒所有等待线程
  │     │
  │     ▼
  │   ConditionObject.signalAll()
  │     │
  │     ▼
  │   遍历条件队列，逐个 LockSupport.unpark(node.thread)
  │     │
  │     ▼
  │   Parker::unpark()  ← HotSpot 底层
  │
  └── index > 0 (等待其他线程)
        │
        ▼
      trip.await()  ← 在条件上等待
        │
        ▼
      ConditionObject.await()
        │
        ├── lock.unlock()  ← 释放锁
        │
        ├── LockSupport.park(this)  ← 阻塞
        │     │
        │     ▼
        │   Parker::park()  ← HotSpot 底层
        │
        └── 被唤醒后 lock.lock()  ← 重新获取锁
```

---

## 八、synchronized 与 JUC 锁的底层对比

### 8.1 等待队列对比

```
synchronized 的 ObjectMonitor 等待队列：

  ┌─────────────────────────────────────────────────────────┐
  │  ObjectMonitor                                          │
  │                                                         │
  │  _owner (持有锁的线程)                                    │
  │                                                         │
  │  _cxq (竞争队列, LIFO)          _EntryList (入口队列, FIFO)│
  │  ┌───┐  ┌───┐              ┌───┐  ┌───┐               │
  │  │ T5│→│ T4│              │ T2│→│ T3│               │
  │  └───┘  └───┘              └───┘  └───┘               │
  │  (新来的线程先入 cxq,        (从 cxq 转移到 EntryList)   │
  │   然后由 _owner 决定                                        │
  │   是否转移到 EntryList)                                     │
  │                                                         │
  │  _WaitSet (等待集合, 双向链表)                             │
  │  ┌───┐  ┌───┐  ┌───┐                                  │
  │  │ T6│←→│ T7│←→│ T8│                                  │
  │  └───┘  └───┘  └───┘                                  │
  │  (调用 wait() 的线程)                                     │
  └─────────────────────────────────────────────────────────┘


AQS 的 CLH 队列（变体）：

  ┌─────────────────────────────────────────────────────────┐
  │  AQS                                                    │
  │                                                         │
  │  exclusiveOwnerThread (持有锁的线程)                      │
  │                                                         │
  │  CLH 双向队列:                                           │
  │  ┌──────┐     ┌──────┐     ┌──────┐                    │
  │  │ head │ ←── │ Node │ ←── │ Node │ ←── tail          │
  │  │(dummy)│ ──→ │  T2  │ ──→ │  T3  │                    │
  │  └──────┘     └──────┘     └──────┘                    │
  │  waitStatus:   SIGNAL       SIGNAL                     │
  │                                                         │
  │  Condition 队列:                                         │
  │  ┌──────┐     ┌──────┐                                 │
  │  │ Node │ ←── │ Node │                                 │
  │  │  T6  │ ──→ │  T7  │                                 │
  │  └──────┘     └──────┘                                 │
  │  (await() 的线程)                                        │
  └─────────────────────────────────────────────────────────┘
```

### 8.2 底层原语调用对比

```
┌──────────────────┬───────────────────────┬───────────────────────┐
│     操作          │  synchronized          │  JUC (AQS)            │
├──────────────────┼───────────────────────┼───────────────────────┤
│ 获取锁(无竞争)   │ 偏向锁: 比较线程ID      │ CAS: Atomic::cmpxchg  │
│                  │ 轻量级锁: CAS 替换      │ Unsafe.compareAndSwap │
│                  │ markOop                │ state字段              │
├──────────────────┼───────────────────────┼───────────────────────┤
│ 获取锁(有竞争)   │ 重量级锁:              │ 入CLH队列:             │
│                  │ ObjectMonitor.enter()  │ CAS 设置 tail          │
│                  │ ParkEvent::park()      │ LockSupport.park()    │
│                  │                        │ Parker::park()        │
├──────────────────┼───────────────────────┼───────────────────────┤
│ 释放锁           │ 偏向锁: 无操作          │ CAS 更新 state        │
│                  │ 轻量级锁: CAS 恢复      │ unpark 后继节点        │
│                  │ markOop                │ LockSupport.unpark()   │
│                  │ 重量级锁:               │ Parker::unpark()      │
│                  │ ObjectMonitor.exit()   │                       │
├──────────────────┼───────────────────────┼───────────────────────┤
│ 条件等待          │ Object.wait()          │ Condition.await()     │
│                  │ ObjectMonitor.wait()   │ LockSupport.park()   │
│                  │ ParkEvent::park()      │ Parker::park()        │
├──────────────────┼───────────────────────┼───────────────────────┤
│ 条件通知          │ Object.notify()        │ Condition.signal()   │
│                  │ ObjectMonitor.notify() │ LockSupport.unpark()  │
│                  │ ParkEvent::unpark()    │ Parker::unpark()      │
├──────────────────┼───────────────────────┼───────────────────────┤
│ 内存屏障          │ monitorenter: acquire  │ volatile 读/写        │
│                  │ monitorexit: release   │ OrderAccess           │
│                  │ (由JVM隐式保证)        │ (由AQS显式保证)        │
└──────────────────┴───────────────────────┴───────────────────────┘

关键区别：
  1. synchronized 使用 ObjectMonitor + ParkEvent（JVM 内部同步原语）
  2. JUC 使用 AQS + Parker（基于 CAS + park/unpark 的 Java 层实现）
  3. 两者最终都使用操作系统条件变量（pthread_cond_t / Windows Event）
  4. synchronized 的锁升级是自动的，JUC 的锁策略是手动选择的
```

---

## 九、Unsafe 方法与 HotSpot 实现映射表

```
┌────────────────────────────────┬──────────────────────────────────────────────┐
│  Unsafe 方法                    │  HotSpot 实现                                │
├────────────────────────────────┼──────────────────────────────────────────────┤
│ compareAndSwapInt()            │ Atomic::cmpxchg() → lock cmpxchg (x86)      │
│ compareAndSwapLong()           │ Atomic::cmpxchg() → lock cmpxchg/cmpxchg8b   │
│ compareAndSwapObject()         │ oopDesc::atomic_compare_exchange_oop()       │
│                                │ + update_barrier_set() (写屏障)               │
├────────────────────────────────┼──────────────────────────────────────────────┤
│ getIntVolatile()               │ OrderAccess::load_acquire()                  │
│ putIntVolatile()               │ OrderAccess::release_store() + storeload()    │
│ getObjectVolatile()            │ OrderAccess::load_acquire()                  │
│ putObjectVolatile()            │ OrderAccess::release_store() + storeload()   │
├────────────────────────────────┼──────────────────────────────────────────────┤
│ putOrderedObject()             │ OrderAccess::release_store() (无 storeload) │
│ putOrderedInt()                │ OrderAccess::release_store() (无 storeload) │
│ putOrderedLong()               │ OrderAccess::release_store() (无 storeload) │
├────────────────────────────────┼──────────────────────────────────────────────┤
│ getAndAddInt()                 │ Atomic::add() → lock xadd (x86)             │
│ getAndSetInt()                 │ Atomic::xchg() → lock xchg (x86)            │
├────────────────────────────────┼──────────────────────────────────────────────┤
│ park()                         │ Parker::park() → pthread_cond_timedwait()   │
│ unpark()                       │ Parker::unpark() → pthread_cond_signal()     │
├────────────────────────────────┼──────────────────────────────────────────────┤
│ loadFence()                    │ OrderAccess::acquire()                      │
│ storeFence()                   │ OrderAccess::release()                      │
│ fullFence()                    │ OrderAccess::fence()                        │
├────────────────────────────────┼──────────────────────────────────────────────┤
│ objectFieldOffset()            │ 反射获取字段偏移量                            │
│ allocateInstance()             │ InstanceKlass::allocate_instance()           │
├────────────────────────────────┼──────────────────────────────────────────────┤
│ monitorEnter()                 │ ObjectSynchronizer::jni_enter()              │
│ monitorExit()                  │ ObjectSynchronizer::jni_exit()               │
│ tryMonitorEnter()              │ ObjectSynchronizer::jni_try_enter()          │
└────────────────────────────────┴──────────────────────────────────────────────┘
```

---

## 十、源码文件索引

| 文件 | 路径 | 说明 |
|------|------|------|
| unsafe.cpp | `src/share/vm/prims/unsafe.cpp` | Unsafe 所有 JNI 实现 |
| jvm.cpp | `src/share/vm/prims/jvm.cpp` | JVM_Thread_Start 等 |
| synchronizer.hpp | `src/share/vm/runtime/synchronizer.hpp` | ObjectSynchronizer 接口 |
| synchronizer.cpp | `src/share/vm/runtime/synchronizer.cpp` | 锁膨胀/释放实现 |
| objectMonitor.hpp | `src/share/vm/runtime/objectMonitor.hpp` | 重量级监视器 |
| objectMonitor.cpp | `src/share/vm/runtime/objectMonitor.cpp` | 重量级监视器实现 |
| park.hpp/cpp | `src/share/vm/runtime/park.hpp/cpp` | Parker/ParkEvent |
| atomic.hpp | `src/share/vm/runtime/atomic.hpp` | 原子操作接口 |
| atomic.inline.hpp | `src/share/vm/runtime/atomic.inline.hpp` | 原子操作内联 |
| orderAccess.hpp | `src/share/vm/runtime/orderAccess.hpp` | 内存屏障接口 |
| markOop.hpp | `src/share/vm/oops/markOop.hpp` | Mark Word 定义 |
| basicLock.hpp | `src/share/vm/runtime/basicLock.hpp` | 轻量级锁记录 |
| biasedLocking.hpp/cpp | `src/share/vm/runtime/biasedLocking.hpp/cpp` | 偏向锁实现 |

---

*上一篇：[03_Thread_Model.md](03_Thread_Model.md) | 返回：[01_Java_Memory_Model.md](01_Java_Memory_Model.md)*