# Java 对象头与 Mark Word 深度分析

> 基于 OpenJDK 8u HotSpot 源码的对象内存布局解析

---

## 一、对象内存布局概览

```
┌──────────────────────────────────────────────────────────────────┐
│                    Java 对象在堆中的布局                          │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    对象头 (Object Header)                │    │
│  │  ┌─────────────────────────────────────────────────────┐│    │
│  │  │           Mark Word (32/64 bit)                     ││    │
│  │  │  存储哈希码、GC年龄、锁状态、偏向线程ID等              ││    │
│  │  └─────────────────────────────────────────────────────┘│    │
│  │  ┌─────────────────────────────────────────────────────┐│    │
│  │  │     Klass Pointer (32/64 bit, 可压缩为 32 bit)       ││    │
│  │  │  指向对象所属类的元数据（InstanceKlass）              ││    │
│  │  └─────────────────────────────────────────────────────┘│    │
│  │  ┌─────────────────────────────────────────────────────┐│    │
│  │  │  Array Length (32 bit, 仅数组对象有)                 ││    │
│  │  └─────────────────────────────────────────────────────┘│    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              实例数据 (Instance Data)                     │    │
│  │  对象真正存储的有效信息——各字段的内容                      │    │
│  │  包括从父类继承的字段和自身定义的字段                      │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              对齐填充 (Padding)                           │    │
│  │  保证对象大小是 8 字节的整数倍                              │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘

对象大小计算：
  32位 JVM:  Mark Word(4B) + Klass Ptr(4B) = 8B 对象头
  64位 JVM:  Mark Word(8B) + Klass Ptr(8B) = 16B 对象头
  64位+压缩: Mark Word(8B) + Klass Ptr(4B) = 12B 对象头 + 4B padding = 16B
```

---

## 二、Mark Word 位布局详解

### 2.1 源码定义

```cpp
// 源码位置: src/share/vm/oops/markOop.hpp

class markOopDesc: public oopDesc {
 public:
  // 常量定义
  enum { age_bits                 = 4,     // GC 年龄占 4 位
         lock_bits                = 2,     // 锁标志占 2 位
         biased_lock_bits         = 1,     // 偏向锁标志占 1 位
         max_hash_bits            = BitsPerWord - age_bits - lock_bits - biased_lock_bits,
         hash_bits               = max_hash_bits > 31 ? 31 : max_hash_bits,
         cms_bits                = LP64_ONLY(1) NOT_LP64(0),  // 64位有CMS位
         epoch_bits              = 2      // 偏向锁时间戳占 2 位
  };

  // 锁状态值
  enum { locked_value             = 0,    // 00 - 轻量级锁
         unlocked_value           = 1,    // 01 - 无锁
         monitor_value            = 2,    // 10 - 重量级锁
         marked_value             = 3,    // 11 - GC标记
         biased_lock_pattern      = 5     // 101 - 偏向锁
  };
};
```

### 2.2 32 位 Mark Word 布局

```
32 位 JVM Mark Word (共 32 bit):

┌─────────┬─────────────────────────────┬──────┬──────────┬───────────────┐
│  位域    │         内容                │ 大小  │  说明     │   状态         │
├─────────┼─────────────────────────────┼──────┼──────────┼───────────────┤
│         │ hashcode:25                 │ 25bit │ 身份哈希码│               │
│         │ age:4                       │ 4bit │ GC年龄    │ 无锁(unlocked)│
│         │ biased_lock:1               │ 1bit │ 偏向标志0 │ lock: 01      │
│         │ lock:2                      │ 2bit │ 锁标志01  │               │
├─────────┼─────────────────────────────┼──────┼──────────┼───────────────┤
│         │ JavaThread*:23              │ 23bit│ 偏向线程ID│               │
│         │ epoch:2                     │ 2bit │ 偏向时间戳│ 偏向锁        │
│         │ age:4                       │ 4bit │ GC年龄    │ biased_lock:1 │
│         │ biased_lock:1               │ 1bit │ 偏向标志1 │ lock: 01      │
│         │ lock:2                      │ 2bit │ 锁标志01  │               │
├─────────┼─────────────────────────────┼──────┼──────────┼───────────────┤
│         │ ptr_to_lock_record:30       │ 30bit│ 锁记录指针│ 轻量级锁      │
│         │ lock:2                      │ 2bit │ 锁标志00  │               │
├─────────┼─────────────────────────────┼──────┼──────────┼───────────────┤
│         │ ptr_to_objectMonitor:30     │ 30bit│ 监视器指针│ 重量级锁      │
│         │ lock:2                      │ 2bit │ 锁标志10  │               │
├─────────┼─────────────────────────────┼──────┼──────────┼───────────────┤
│         │ 空/GC信息:30               │ 30bit│ GC信息    │ GC标记        │
│         │ lock:2                      │ 2bit │ 锁标志11  │               │
└─────────┴─────────────────────────────┴──────┴──────────┴───────────────┘

图解：无锁状态 (biased_lock=0, lock=01)
  ┌──────────────────────────┬────┬─┬─┬─┐
  │    hashcode (25bit)      │age │0│01│
  └──────────────────────────┴────┴─┴─┴─┘
  31                        7   3 2 1 0

偏向锁状态 (biased_lock=1, lock=01)
  ┌───────────────────┬──┬────┬─┬─┬─┐
  │ JavaThread* (23bit)│ep│age │1│01│
  └───────────────────┴──┴────┴─┴─┴─┘
  31                 8  6  2  1  0
```

### 2.3 64 位 Mark Word 布局

```
64 位 JVM Mark Word (共 64 bit):

无锁状态 (biased_lock=0, lock=01):
  ┌─────────────────────────────────┬──────┬─┬────┬─┬──┐
  │    unused (25bit)               │hash  │u│age │0│01│
  │                                 │(31) │ │(4) │ │  │
  └─────────────────────────────────┴──────┴─┴────┴─┴──┘
  63                               31     0

偏向锁状态 (biased_lock=1, lock=01):
  ┌─────────────────────────────────┬──────┬─┬────┬─┬──┐
  │    JavaThread* (54bit)          │epoch │u│age │1│01│
  │                                 │(2)   │ │(4) │ │  │
  └─────────────────────────────────┴──────┴─┴────┴─┴──┘
  63                               9      7 6  2  1  0

轻量级锁 (lock=00):
  ┌────────────────────────────────────────────────────┬──┐
  │    ptr_to_lock_record (62bit)                      │00│
  └────────────────────────────────────────────────────┴──┘
  63                                                  1 0

重量级锁 (lock=10):
  ┌────────────────────────────────────────────────────┬──┐
  │    ptr_to_objectMonitor (62bit)                    │10│
  └────────────────────────────────────────────────────┴──┘
  63                                                  1 0

GC标记 (lock=11):
  ┌────────────────────────────────────────────────────┬──┐
  │    GC信息 (62bit)                                  │11│
  └────────────────────────────────────────────────────┴──┘

注意：64位模式下有 CMS free bit (cms_free)，用于 CMS 的空闲块管理
```

---

## 三、锁状态转换全景图

```
                          ┌──────────────┐
                          │   无锁状态    │
                          │ lock=01      │
                          │ biased=0     │
                          │ hashcode=已算│
                          └──────┬───────┘
                                 │
           ┌─────────────────────┼─────────────────────┐
           │                     │                     │
           │ 首次获取锁          │ 计算hashcode后       │  禁用偏向锁
           │ (启用偏向锁)        │ 不可偏向             │  -XX:-UseBiasedLocking
           ▼                     │                     ▼
    ┌──────────────┐             │             ┌──────────────┐
    │   偏向锁      │             │             │ 直接轻量级锁  │
    │ lock=01      │             │             │ lock=00      │
    │ biased=1     │             │             │ (无偏向)      │
    │ threadID=XX  │             │             └──────┬───────┘
    └──────┬───────┘             │                    │
           │                     │                    │
           │ 其他线程            │                    │ 竞争
           │ 尝试获取锁          │                    │ CAS 失败
           │ 偏向撤销            │                    │
           ▼                     │                    ▼
    ┌──────────────┐             │             ┌──────────────┐
    │ 撤销偏向→    │             │             │   轻量级锁    │
    │ 轻量级锁     │◄────────────┘             │ lock=00      │
    │ lock=00      │                           │ 指向Lock Record│
    └──────┬───────┘                           └──────┬───────┘
           │                                          │
           │ CAS 自旋失败                               │ 自旋等待
           │ 或调用 wait()                              │ 适应自旋失败
           ▼                                          ▼
    ┌──────────────────────────────────────────────────────┐
    │                   重量级锁                            │
    │ lock=10                                              │
    │ 指向 ObjectMonitor                                   │
    │ (包含 EntryList、WaitSet、owner 等)                   │
    └──────────────────────────────────────────────────────┘

锁膨胀过程（不可逆）：
  偏向锁 → 轻量级锁 → 重量级锁
  （一旦膨胀就不会降级，除非 GC 时 deflate）
```

---

## 四、偏向锁详解

### 4.1 偏向锁原理

```
偏向锁的核心思想：
  如果一个锁始终只被同一个线程获取，那么就没有必要使用 CAS 操作。
  只需在 Mark Word 中记录线程 ID，后续加锁只需比较线程 ID 即可。

通俗比喻：
  偏向锁就像"专用停车位"。你的车停在专用车位上，下次你直接停。
  但如果别人也想停这个车位，就需要撤销"专用"标记，变成公共车位。

源码位置：src/share/vm/runtime/biasedLocking.hpp
          src/share/vm/runtime/biasedLocking.cpp

偏向锁获取流程（fast_enter）：
  ┌──────────────────────────────────────────────────┐
  │ 1. 检查 mark->has_bias_pattern()                  │
  │    → 判断是否是偏向模式 (biased_lock=1, lock=01) │
  │                                                  │
  │ 2. 如果是匿名偏向 (threadID=0)                    │
  │    → CAS 将当前线程ID写入 Mark Word               │
  │    → 成功则获取锁                                 │
  │                                                  │
  │ 3. 如果是当前线程已偏向                             │
  │    → 直接获取锁（无需 CAS）                        │
  │    → 这是最快路径！                                │
  │                                                  │
  │ 4. 如果偏向其他线程                                 │
  │    → 需要撤销偏向（BiasedLocking::revoke_and_rebias）│
  │    → 撤销后进入轻量级锁流程                         │
  └──────────────────────────────────────────────────┘
```

### 4.2 偏向锁撤销

```cpp
// 源码位置: src/share/vm/runtime/synchronizer.cpp

void ObjectSynchronizer::fast_enter(Handle obj, BasicLock* lock, 
                                     bool attempt_rebias, TRAPS) {
  if (UseBiasedLocking) {
    if (!SafepointSynchronize::is_at_safepoint()) {
      // 非安全点：尝试撤销偏向并可能重偏向
      BiasedLocking::Condition cond = 
          BiasedLocking::revoke_and_rebias(obj, attempt_rebias, THREAD);
      if (cond == BiasedLocking::BIAS_REVOKED_AND_REBIASED) {
        return;  // 成功重偏向，获取锁
      }
    } else {
      // 安全点：只能撤销偏向
      BiasedLocking::revoke_at_safepoint(obj);
    }
    assert(!obj->mark()->has_bias_pattern(), "biases should be revoked by now");
  }
  // 偏向失败，进入轻量级锁
  slow_enter(obj, lock, THREAD);
}
```

### 4.3 批量重偏向与批量撤销

```
当大量偏向锁需要撤销时，JVM 使用两种优化：

1. 批量重偏向 (Bulk Rebias):
   - 当某个类的偏向锁撤销次数达到阈值 (BiasedLockingBulkRebiasThreshold=20)
   - JVM 递增该类的 epoch 值
   - 所有旧 epoch 的偏向锁在下次访问时自动重偏向
   
2. 批量撤销 (Bulk Revoke):
   - 当偏向锁撤销次数继续达到阈值 (BiasedLockingBulkRevokeThreshold=40)
   - JVM 完全禁用该类的偏向锁
   - 该类的所有对象都走轻量级锁路径

源码关键参数 (globals.hpp):
  product(intx, BiasedLockingBulkRebiasThreshold, 20, ...)  
  product(intx, BiasedLockingBulkRevokeThreshold, 40, ...)
  product(intx, BiasedLockingDecayTime, 25000, ...)
  product(bool, UseBiasedLocking, true, ...)
```

---

## 五、轻量级锁详解

### 5.1 轻量级锁原理

```
轻量级锁的核心思想：
  当多个线程交替获取锁（没有真正的竞争），使用 CAS 操作避免互斥量开销。
  每个线程在自己的栈帧中创建 Lock Record，通过 CAS 将 Mark Word 
  替换为指向 Lock Record 的指针。

通俗比喻：
  轻量级锁就像"排队取号"。多个人轮流取号，不需要保安（重量级锁）。
  但如果两个人同时取号（竞争），就需要升级为重量级锁。

源码位置：src/share/vm/runtime/synchronizer.cpp (slow_enter)
          src/share/vm/runtime/basicLock.hpp (BasicLock)
```

### 5.2 BasicLock 和 Lock Record

```cpp
// 源码位置: src/share/vm/runtime/basicLock.hpp

class BasicLock VALUE_OBJ_CLASS_SPEC {
 private:
  volatile markOop _displaced_header;  // 保存原来的 Mark Word
 public:
  markOop displaced_header() const     { return _displaced_header; }
  void set_displaced_header(markOop header) { _displaced_header = header; }
};

class BasicObjectLock VALUE_OBJ_CLASS_SPEC {
 private:
  BasicLock _lock;    // 锁记录（包含 displaced header）
  oop       _obj;     // 持有锁的对象
};

// Lock Record 在栈帧中的布局：
// ┌────────────────────────────────┐
// │  method frame                   │
// │  ┌──────────────────────────┐  │
// │  │  BasicObjectLock          │  │
// │  │  _lock._displaced_header  │  │ ← 保存原始 Mark Word
// │  │  _obj                     │  │ ← 指向锁对象
// │  └──────────────────────────┘  │
// │  ┌──────────────────────────┐  │
// │  │  BasicObjectLock          │  │ ← 可能有多层嵌套锁
// │  │  ...                      │  │
// │  └──────────────────────────┘  │
// └────────────────────────────────┘
```

### 5.3 轻量级锁加锁流程

```cpp
// 源码位置: src/share/vm/runtime/synchronizer.cpp

void ObjectSynchronizer::slow_enter(Handle obj, BasicLock* lock, TRAPS) {
  markOop mark = obj->mark();
  assert(!mark->has_bias_pattern(), "should not see bias pattern here");

  if (mark->is_neutral()) {
    // CASE 1: 无锁状态
    // 将 Mark Word 保存到 Lock Record 的 displaced_header
    lock->set_displaced_header(mark);
    // CAS 尝试将 Mark Word 替换为指向 Lock Record 的指针
    if (mark == (markOop) Atomic::cmpxchg_ptr(lock, obj()->mark_addr(), mark)) {
      TEVENT(slow_enter: release stacklock);
      return;  // CAS 成功，获取轻量级锁
    }
    // CAS 失败，有竞争，膨胀为重量级锁
  } else if (mark->has_locker() && THREAD->is_lock_owned((address)mark->locker())) {
    // CASE 2: 当前线程已经持有该锁（重入）
    assert(lock != mark->locker(), "must not re-lock the same lock");
    lock->set_displaced_header(NULL);  // NULL 表示重入
    return;
  }

  // CASE 3: 其他情况（有竞争），膨胀为重量级锁
  lock->set_displaced_header(markOopDesc::unused_mark());
  ObjectSynchronizer::inflate(THREAD, obj())->enter(THREAD);
}
```

```
轻量级锁加锁图解：

步骤1：无锁状态
  对象头: [hashcode | age | 0 | 01]
  栈帧:   [空]

步骤2：复制 Mark Word 到 Lock Record
  对象头: [hashcode | age | 0 | 01]  ← 不变
  栈帧:   [displaced_header = hashcode | age | 0 | 01]
           [obj = &object]

步骤3：CAS 替换 Mark Word
  CAS(obj.mark, mark, lock_record_ptr | 00)
  
  成功：
  对象头: [lock_record_ptr | 00]     ← 指向栈中 Lock Record
  栈帧:   [displaced_header = hashcode | age | 0 | 01]  ← 保存原始值
  
  失败（竞争）：
  → 膨胀为重量级锁
```

### 5.4 轻量级锁解锁流程

```cpp
// 源码位置: src/share/vm/runtime/synchronizer.cpp

void ObjectSynchronizer::fast_exit(oop object, BasicLock* lock, TRAPS) {
  markOop dhw = lock->displaced_header();
  markOop mark = object->mark();

  if (dhw == NULL) {
    // 重入锁，displaced_header 为 NULL
    return;
  }

  if (mark == (markOop) lock) {
    // 轻量级锁，CAS 恢复 Mark Word
    if ((markOop) Atomic::cmpxchg_ptr(dhw, object->mark_addr(), mark) == mark) {
      return;  // 解锁成功
    }
  }

  // CAS 失败或已是重量级锁，走重量级锁退出
  ObjectSynchronizer::inflate(THREAD, object)->exit(true, THREAD);
}
```

---

## 六、重量级锁详解

### 6.1 ObjectMonitor 结构

```cpp
// 源码位置: src/share/vm/runtime/objectMonitor.hpp

class ObjectMonitor {
 public:
  // 关键字段
  volatile markOop   _header;       // 保存原始 Mark Word
  void*     volatile _object;       // 反向引用到对象
  void*     volatile _owner;        // 持有锁的线程（或 BasicLock 指针）
  volatile intptr_t  _recursions;   // 重入次数
  ObjectWaiter * volatile _cxq;     // 最近到达的等待线程队列（LIFO）
  ObjectWaiter * volatile _EntryList;// 等待获取锁的线程队列（FIFO）
  ObjectWaiter * volatile _WaitSet;  // 调用 wait() 的线程队列
  volatile intptr_t  _count;        // 引用计数
  volatile intptr_t  _waiters;      // 等待线程数
  Thread * volatile _succ;          // 继任者线程
  volatile int _Spinner;            // 自旋计数
  volatile int _SpinFreq;           // 自旋频率
  volatile int _SpinClock;          // 自旋时钟
  volatile intptr_t _SpinState;     // 自旋状态（CLH/MCS 队列）
};
```

### 6.2 ObjectMonitor 等待队列模型

```
ObjectMonitor 内部队列结构：

                    _owner (持有锁的线程)
                      │
                      ▼
  ┌────────────────────────────────────────────────────────────┐
  │                    ObjectMonitor                             │
  │                                                             │
  │  _cxq (Contention Queue)     _EntryList (Entry Queue)       │
  │  ┌───────┐                  ┌───────┐                      │
  │  │WaiterA│ ──→ │WaiterB│ ──→ │WaiterC│ ──→ │WaiterD│        │
  │  └───────┘                  └───────┘                      │
  │   (新来的线程)               (等待获取锁的线程)               │
  │                                                             │
  │  _WaitSet (Wait Set)                                        │
  │  ┌───────┐  ┌───────┐  ┌───────┐                          │
  │  │WaiterE│→│WaiterF│→│WaiterG│  (调用 wait() 的线程)       │
  │  └───────┘  └───────┘  └───────┘                          │
  │                                                             │
  └────────────────────────────────────────────────────────────┘

锁获取流程：
  1. 新线程先进入 _cxq（竞争队列，LIFO）
  2. 持有锁的线程释放后，将 _cxq 中的线程转移到 _EntryList
  3. 从 _EntryList 头部唤醒线程
  4. 调用 wait() 的线程进入 _WaitSet
  5. 调用 notify() 从 _WaitSet 移到 _EntryList
```

### 6.3 锁膨胀过程

```cpp
// 源码位置: src/share/vm/runtime/synchronizer.cpp

// 膨胀流程：从轻量级锁/偏向锁膨胀为重量级锁
ObjectMonitor* ObjectSynchronizer::inflate(Thread * Self, oop object) {
  for (;;) {
    markOop mark = object->mark();
    
    if (mark->has_monitor()) {
      // CASE 1: 已经是重量级锁，直接返回
      return mark->monitor();
    }
    
    if (mark == markOopDesc::INFLATING()) {
      // CASE 2: 正在膨胀中，等待
      TEVENT(Inflate: INFLATING - yield);
      os::NakedYield(Self);
      continue;
    }
    
    if (mark->has_locker()) {
      // CASE 3: 轻量级锁，需要膨胀
      ObjectMonitor * m = omAlloc(Self);  // 分配 ObjectMonitor
      m->set_header(mark->displaced_mark_helper());  // 保存原始 Mark Word
      m->set_object(object);
      // CAS 将 Mark Word 替换为指向 ObjectMonitor 的指针 + monitor_value(10)
      markOop cmp = object->cas_set_mark(markOopDesc::encode(m), mark);
      if (cmp == mark) {
        return m;  // 膨胀成功
      }
      // CAS 失败，重试
    }
    
    if (mark->is_neutral()) {
      // CASE 4: 无锁状态，直接膨胀
      ObjectMonitor * m = omAlloc(Self);
      m->set_header(mark);
      m->set_object(object);
      markOop cmp = object->cas_set_mark(markOopDesc::encode(m), mark);
      if (cmp == mark) {
        return m;
      }
    }
  }
}
```

---

## 七、Klass Pointer 与压缩指针

### 7.1 对象头中的 Klass Pointer

```cpp
// 源码位置: src/share/vm/oops/oop.hpp

class oopDesc {
 private:
  volatile markOop _mark;           // Mark Word
  union _metadata {
    Klass*      _klass;             // 正常的 Klass 指针（64位）
    narrowKlass _compressed_klass;  // 压缩的 Klass 指针（32位）
  } _metadata;
};
```

### 7.2 压缩指针（Compressed OOPs）

```
64 位 JVM 默认启用压缩指针（-XX:+UseCompressedOops）：

不使用压缩指针：
  Mark Word:  8 bytes
  Klass Ptr:  8 bytes
  对象头总计: 16 bytes
  最大堆内存: 无限制

使用压缩指针：
  Mark Word:          8 bytes
  Compressed Klass:   4 bytes  
  对象头总计:         12 bytes + 4 bytes padding = 16 bytes
  最大堆内存:         32 GB（2^32 * 8 = 32GB，对象按8字节对齐）

压缩指针原理：
  ┌────────────────────────────────────────────────┐
  │  窄 Klass 指针 (32 bit)                        │
  │  实际地址 = 窄指针 << 3 + Klass 基地址           │
  │  （左移3位，因为对象按8字节对齐，低3位总是0）      │
  └────────────────────────────────────────────────┘

  窄 OOP 指针 (32 bit):
  实际地址 = 窄指针 << 3 + 堆基地址
  最大寻址: 2^32 * 8 = 32 GB
```

---

## 八、Identity HashCode 与锁的交互

### 8.1 HashCode 对锁的影响

```
关键规则：一旦计算了 identity hashcode，对象就不能进入偏向锁状态！

原因：偏向锁需要使用 Mark Word 的 hashcode 位来存储线程ID，
     而 hashcode 一旦计算就被存储在 Mark Word 中，
     两者冲突，只能二选一。

流程图：
  ┌──────────────────────────────────────────────┐
  │  调用 System.identityHashCode(obj) 或         │
  │  obj.hashCode() (未重写时)                    │
  │                                              │
  │  → 第一次调用时计算 hashcode                  │
  │  → 写入 Mark Word 的 hashcode 字段            │
  │  → 对象从此不能进入偏向锁状态                   │
  └──────────────────────────────────────────────┘

  源码位置: src/share/vm/runtime/synchronizer.cpp
  FastHashCode() 方法会膨胀锁来存储 hashcode

  hashcode 计算时机：
  - 首次调用 System.identityHashCode() 时
  - 使用 os::random() 生成（不是内存地址！）
  - 存储在 Mark Word 中，之后不再变化
```

### 8.2 不同锁状态下的 Mark Word 变化

```
以 64 位 JVM 为例，对象 new Object() 的 Mark Word 变化过程：

初始状态（无锁，未计算hashcode）：
  ┌──────────────────────────────────────────────────────┐
  │unused:25│          0         │u│age:4│0│    01       │
  │         │  (hash=0,表示未算)  │ │ 0   │ │ lock=01    │
  └──────────────────────────────────────────────────────┘

计算hashcode后（无锁，已计算hashcode）：
  ┌──────────────────────────────────────────────────────┐
  │unused:25│    hashcode:31    │u│age:4│0│    01       │
  │         │  (0x12345678)      │ │ 0   │ │ lock=01    │
  └──────────────────────────────────────────────────────┘

偏向锁（线程 0x1234 获取锁）：
  ┌──────────────────────────────────────────────────────┐
  │      JavaThread*:54       │ep:2│age:4│1│    01       │
  │  (0x1234)                 │ 0  │ 0   │ │ lock=01    │
  └──────────────────────────────────────────────────────┘
  注意：hashcode 字段被线程ID覆盖！

轻量级锁（指向栈中 Lock Record）：
  ┌──────────────────────────────────────────────────────┐
  │          ptr_to_lock_record:62                    │00 │
  │  (0x7fff1234)                                     │   │
  └──────────────────────────────────────────────────────┘
  注意：原始 Mark Word 保存在 Lock Record 的 displaced_header 中

重量级锁（指向 ObjectMonitor）：
  ┌──────────────────────────────────────────────────────┐
  │          ptr_to_objectMonitor:62                  │10 │
  │  (0x7fff5678)                                     │   │
  └──────────────────────────────────────────────────────┘
  注意：原始 Mark Word 保存在 ObjectMonitor 的 _header 中
```

---

## 九、源码文件索引

| 文件 | 路径 | 说明 |
|------|------|------|
| markOop.hpp | `src/share/vm/oops/markOop.hpp` | Mark Word 位布局定义 |
| markOop.cpp | `src/share/vm/oops/markOop.cpp` | Mark Word 方法实现 |
| markOop.inline.hpp | `src/share/vm/oops/markOop.inline.hpp` | Mark Word 内联方法 |
| oop.hpp | `src/share/vm/oops/oop.hpp` | 对象头定义（markOop + Klass*） |
| oop.inline.hpp | `src/share/vm/oops/oop.inline.hpp` | 对象头内联方法 |
| basicLock.hpp | `src/share/vm/runtime/basicLock.hpp` | 轻量级锁记录 |
| basicLock.cpp | `src/share/vm/runtime/basicLock.cpp` | 轻量级锁实现 |
| synchronizer.hpp | `src/share/vm/runtime/synchronizer.hpp` | 对象同步器接口 |
| synchronizer.cpp | `src/share/vm/runtime/synchronizer.cpp` | 锁膨胀/释放实现 |
| objectMonitor.hpp | `src/share/vm/runtime/objectMonitor.hpp` | 重量级监视器 |
| objectMonitor.cpp | `src/share/vm/runtime/objectMonitor.cpp` | 重量级监视器实现 |
| biasedLocking.hpp | `src/share/vm/runtime/biasedLocking.hpp` | 偏向锁接口 |
| biasedLocking.cpp | `src/share/vm/runtime/biasedLocking.cpp` | 偏向锁实现 |
| instanceKlass.hpp | `src/share/vm/oops/instanceKlass.hpp` | 类元数据 |

---

*上一篇：[01_Java_Memory_Model.md](01_Java_Memory_Model.md) | 下一篇：[03_Thread_Model.md](03_Thread_Model.md) — 线程模型深度分析*