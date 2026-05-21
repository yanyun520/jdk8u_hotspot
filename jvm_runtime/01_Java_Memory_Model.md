# Java 内存模型（JMM）与 HotSpot 实现

> 基于 OpenJDK 8u HotSpot 源码的深度分析

---

## 一、JMM 是什么？

Java 内存模型（Java Memory Model, JMM）定义了 Java 程序中**多线程之间如何通过内存进行交互**的规范。它不是 JVM 内存结构（堆、栈等），而是**一组规则**，确保多线程环境下的内存可见性、有序性和原子性。

### 1.1 JMM 的三大特性

| 特性 | 含义 | Java 关键字/机制 |
|------|------|-----------------|
| **原子性** | 操作不可分割 | `synchronized`、`AtomicXxx` |
| **可见性** | 线程修改后其他线程立即可见 | `volatile`、`synchronized`、`final` |
| **有序性** | 指令不被意外重排 | `volatile`、`synchronized`、`happens-before` |

### 1.2 Happens-Before 规则

```
happens-before 规则（保证前一个操作的结果对后一个操作可见）：

1. 程序顺序规则：同一线程中，按代码顺序，前面的操作 happens-before 后面的操作
2. 监视器锁规则：对同一个锁的 unlock happens-before 后续的 lock
3. volatile 规则：对 volatile 变量的写 happens-before 后续的读
4. 线程启动规则：Thread.start() happens-before 该线程的所有操作
5. 线程终止规则：线程的所有操作 happens-before Thread.join() 返回
6. 线程中断规则：Thread.interrupt() happens-before 被中断线程检测到中断
7. 对象终结规则：构造函数执行结束 happens-before finalize() 开始
8. 传递性：如果 A happens-before B，B happens-before C，则 A happens-before C
```

---

## 二、JMM 在 HotSpot 中的实现架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Java 层面                                    │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │  volatile    │  │ synchronized │  │   AtomicXXX   │              │
│  │   读/写      │  │  lock/unlock  │  │  compareAndSet│              │
│  └──────┬───────┘  └──────┬───────┘  └──────┬────────┘              │
│         │                 │                  │                       │
│  ┌──────▼─────────────────▼──────────────────▼────────────────┐     │
│  │                    字节码层                                  │     │
│  │  putfield/getfield + volatile flag                          │     │
│  │  monitorenter/monitorexit                                    │     │
│  │  Unsafe.compareAndSwapXxx                                    │     │
│  └──────┬─────────────────┬──────────────────┬────────────────┘     │
└─────────┼─────────────────┼──────────────────┼─────────────────────┘
          │                 │                  │
┌─────────▼─────────────────▼──────────────────▼─────────────────────┐
│                      HotSpot JVM 层                                │
│                                                                    │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐  │
│  │   OrderAccess     │  │ ObjectSynchronizer│  │   Atomic::cmpxchg │ │
│  │  (内存屏障)       │  │  (锁膨胀/ deflate)│  │  (CAS 原子操作)    │  │
│  └────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘  │
│           │                     │                       │            │
│  ┌────────▼─────────────────────▼───────────────────────▼────────┐  │
│  │                    CPU 指令层                                  │  │
│  │  x86: lock cmpxchg / mfence / lock addl                      │  │
│  │  ARM: dmb / dsb / ldxr+stxr                                  │  │
│  │  SPARC: membar #StoreLoad/#LoadStore/#StoreStore/#LoadLoad     │  │
│  └───────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────┘
```

---

## 三、内存屏障（Memory Barrier）实现

### 3.1 四种内存屏障

HotSpot 在 `orderAccess.hpp` 中定义了四种内存屏障原语：

```cpp
// 源码位置: src/share/vm/runtime/orderAccess.hpp

// 1. LoadLoad 屏障
//    Load1; LoadLoad; Load2
//    确保 Load1 在 Load2 之前完成
static void     loadload();

// 2. StoreStore 屏障  
//    Store1; StoreStore; Store2
//    确保 Store1 的结果对其他处理器可见后再执行 Store2
static void     storestore();

// 3. LoadStore 屏障
//    Load1; LoadStore; Store2
//    确保 Load1 在 Store2 之前完成
static void     loadstore();

// 4. StoreLoad 屏障
//    Store1; StoreLoad; Load2
//    确保 Store1 的结果对其他处理器可见后再执行 Load2
//    这是最强的屏障，也是代价最高的屏障
static void     storeload();
```

### 3.2 内存屏障的 CPU 指令映射

```
┌─────────────────┬──────────────────────────────────────────────────┐
│     屏障类型     │              CPU 实现                            │
├─────────────────┼──────────────────────────────────────────────────┤
│ LoadLoad         │ x86: 无需指令（TSO 保证）                        │
│                  │ ARM: dmb ish                                     │
│                  │ SPARC: membar #LoadLoad | #LoadStore              │
├─────────────────┼──────────────────────────────────────────────────┤
│ StoreStore       │ x86: 无需指令（TSO 保证）                        │
│                  │ ARM: dmb ishst                                   │
│                  │ SPARC: membar #StoreStore                         │
├─────────────────┼──────────────────────────────────────────────────┤
│ LoadStore        │ x86: 无需指令（TSO 保证）                        │
│                  │ ARM: dmb ish                                     │
│                  │ SPARC: membar #LoadStore                          │
├─────────────────┼──────────────────────────────────────────────────┤
│ StoreLoad        │ x86: lock addl $0, (rsp) 或 mfence              │
│                  │ ARM: dmb ish                                     │
│                  │ SPARC: membar #StoreLoad                          │
└─────────────────┴──────────────────────────────────────────────────┘

关键理解：
- x86 是强内存模型（TSO，Total Store Order），大多数屏障是空操作
- x86 唯一需要硬件屏障的是 StoreLoad（因为写缓冲区的存在）
- ARM/AArch64 是弱内存模型，所有屏障都需要硬件指令
- volatile 写在 x86 上只需 StoreLoad 屏障（lock addl 前缀）
```

### 3.3 volatile 的内存屏障实现

```
volatile 读（getfield/getstatic 字节码）：
  ┌─────────────────────┐
  │  LoadLoad 屏障      │  ← 确保前面的读不会重排到后面
  │  LoadStore 屏障     │  ← 确保前面的读不会重排到后面的写
  │  读取 volatile 变量  │
  │  (x86: 无需额外指令) │
  └─────────────────────┘

volatile 写（putfield/putstatic 字节码）：
  ┌─────────────────────┐
  │  StoreStore 屏障    │  ← 确保前面的写在后面写之前对其他处理器可见
  │  写入 volatile 变量  │
  │  StoreLoad 屏障     │  ← 确保写的结果对后续读可见
  │  (x86: lock addl)   │  ← 这是代价最高的屏障！
  └─────────────────────┘
```

### 3.4 release/acquire 语义

```cpp
// 源码位置: src/share/vm/runtime/orderAccess.hpp

// release 操作：确保之前的读写不会重排到 release 之后
// 等价于 LoadStore + StoreStore 屏障
static void   release();
// 实现：x86 上是空操作（TSO 已保证）

// acquire 操作：确保之后的读写不会重排到 acquire 之前
// 等价于 LoadLoad + LoadStore 屏障
static void   acquire();
// 实现：x86 上是空操作（TSO 已保证）

// fence 操作：全功能屏障，等于 LoadLoad + StoreStore + LoadStore + StoreLoad
static void   fence();
// 实现：x86 上是 lock addl $0, (rsp) 或 mfence

// 组合操作（更高效）：
// release_store = release + store（先屏障再存储）
// load_acquire = load + acquire（先加载再屏障）
static void   release_store(volatile jbyte*   p, jbyte   v);
static jint    load_acquire(volatile jint*     p);
```

---

## 四、CAS 原子操作实现

### 4.1 Atomic 类

```cpp
// 源码位置: src/share/vm/runtime/atomic.hpp

class Atomic : AllStatic {
 public:
  // CAS 操作：比较并交换
  // 如果 *dest == compare_value，则将 *dest 替换为 exchange_value
  // 返回 *dest 的旧值
  // 保证全功能内存屏障（fence_cmpxchg_acquire）
  static jbyte  cmpxchg(jbyte  exchange_value, volatile jbyte*  dest, jbyte  compare_value);
  static jint   cmpxchg(jint   exchange_value, volatile jint*   dest, jint   compare_value);
  static jlong  cmpxchg(jlong  exchange_value, volatile jlong*  dest, jlong  compare_value);
  
  // 原子交换
  static jint   xchg(jint   exchange_value, volatile jint*   dest);
  
  // 原子加
  static jint   add(jint  add_value, volatile jint*  dest);
};
```

### 4.2 CAS 的 CPU 指令实现

```
┌──────────┬──────────────────────────────────────────────────────┐
│  架构    │  CAS 指令实现                                        │
├──────────┼──────────────────────────────────────────────────────┤
│  x86     │  lock cmpxchg  (LOCK 前缀 + CMPXCHG 指令)            │
│          │  lock cmpxchg8b (64位 CAS 在 32位 JVM 上)            │
│          │  lock cmpxchg   (64位 JVM 原生支持 64位 CAS)          │
├──────────┼──────────────────────────────────────────────────────┤
│  ARM     │  ldxr + stxr   (独占加载 + 独占存储，LL/SC 模式)       │
│          │  带 dmb 屏障保证可见性                                  │
├──────────┼──────────────────────────────────────────────────────┤
│  SPARC   │  cas  (Compare-And-Swap 指令)                        │
│          │  casx (64位 CAS)                                      │
└──────────┴──────────────────────────────────────────────────────┘

通俗理解 CAS：
  CAS 就像去银行取钱时的"乐观锁"：
  1. 你记住账户余额是 1000 元（compare_value）
  2. 你要存入 500 元（exchange_value）
  3. 到柜台时，先看余额还是不是 1000 元
  4. 如果是，存入 500，余额变为 1500
  5. 如果不是（别人已改过），返回失败，重新来一次
```

### 4.3 Unsafe CAS 的 JNI 实现

```cpp
// 源码位置: src/share/vm/prims/unsafe.cpp

// Unsafe.compareAndSwapInt() 的 JVM 实现
UNSAFE_ENTRY(jboolean, Unsafe_CompareAndSwapInt(JNIEnv *env, jobject unsafe, 
    jobject obj, jlong offset, jint e, jint x))
  UnsafeWrapper("Unsafe_CompareAndSwapInt");
  oop p = JNIHandles::resolve(obj);
  jint* addr = (jint *) index_oop_from_field_offset_long(p, offset);
  // 调用 Atomic::cmpxchg，底层使用 CPU 的 CAS 指令
  return (jint)(Atomic::cmpxchg(x, addr, e)) == e;
UNSAFE_END

// Unsafe.compareAndSwapLong() 的 JVM 实现
UNSAFE_ENTRY(jboolean, Unsafe_CompareAndSwapLong(JNIEnv *env, jobject unsafe, 
    jobject obj, jlong offset, jlong e, jlong x))
  UnsafeWrapper("Unsafe_CompareAndSwapLong");
  Handle p (THREAD, JNIHandles::resolve(obj));
  jlong* addr = (jlong*)(index_oop_from_field_offset_long(p(), offset));
#ifdef SUPPORTS_NATIVE_CX8
  return (jlong)(Atomic::cmpxchg(x, addr, e)) == e;  // 原子 CAS
#else
  // 32位平台可能不支持 64位原子操作，需要加锁
  if (VM_Version::supports_cx8())
    return (jlong)(Atomic::cmpxchg(x, addr, e)) == e;
  else {
    jboolean success = false;
    MutexLockerEx mu(UnsafeJlong_lock, Mutex::_no_safepoint_check_flag);
    jlong val = Atomic::load(addr);
    if (val == e) { Atomic::store(x, addr); success = true; }
    return success;
  }
#endif
UNSAFE_END

// Unsafe.compareAndSwapObject() 的 JVM 实现
UNSAFE_ENTRY(jboolean, Unsafe_CompareAndSwapObject(JNIEnv *env, jobject unsafe,
    jobject obj, jlong offset, jobject e_h, jobject x_h))
  UnsafeWrapper("Unsafe_CompareAndSwapObject");
  oop x = JNIHandles::resolve(x_h);
  oop e = JNIHandles::resolve(e_h);
  oop p = JNIHandles::resolve(obj);
  HeapWord* addr = (HeapWord *)index_oop_from_field_offset_long(p, offset);
  oop res = oopDesc::atomic_compare_exchange_oop(x, addr, e, true);
  jboolean success = (res == e);
  if (success)
    update_barrier_set((void*)addr, x);  // 写屏障（GC 用）
  return success;
UNSAFE_END
```

---

## 五、volatile 在 HotSpot 中的实现

### 5.1 volatile 字节码处理

```
Java 代码:                    字节码:
volatile int x;               // volatile 标志在 field_info 的 access_flags 中
x = 1;                        // putfield (带 ACC_VOLATILE 标志)
int y = x;                    // getfield (带 ACC_VOLATILE 标志)

HotSpot 解释器处理：
  putfield (volatile) → 调用 OrderAccess::release_store + StoreLoad 屏障
  getfield (volatile) → 调用 OrderAccess::load_acquire
```

### 5.2 volatile 读写流程图

```
volatile 写（putfield）:
  ┌──────────────────────────────────────────────┐
  │  1. StoreStore 屏障                           │
  │     (确保前面的写先完成)                        │
  │  2. 写入 volatile 变量                         │
  │     (将新值写入主内存)                          │
  │  3. StoreLoad 屏障                            │
  │     (x86: lock addl $0, (rsp))                │
  │     (确保写对后续读可见)                        │
  └──────────────────────────────────────────────┘

volatile 读（getfield）:
  ┌──────────────────────────────────────────────┐
  │  1. 读取 volatile 变量                         │
  │     (从主内存读取最新值)                        │
  │  2. LoadLoad 屏障                              │
  │     (确保后面的读不被重排到前面)                  │
  │  3. LoadStore 屏障                             │
  │     (确保后面的写不被重排到前面)                  │
  └──────────────────────────────────────────────┘

注意：在 x86 (TSO) 上：
  - volatile 读不需要额外屏障指令（x86 保证 LoadLoad 和 LoadStore）
  - volatile 写只需要 StoreLoad 屏障（lock addl 指令）
  - 这就是为什么 volatile 在 x86 上读比写快
```

### 5.3 JSR-133 Cookbook 对照表

```
┌──────────────────────────────────────────────────────────────────┐
│                    JSR-133 规则 → HotSpot 实现                    │
├──────────────────────────────────────────────────────────────────┤
│ volatile 读后 → 任意读    →  LoadLoad 屏障  →  x86: 空操作      │
│ volatile 读后 → 任意写    →  LoadStore 屏障 →  x86: 空操作      │
│ volatile 写前 → 任意写    →  StoreStore 屏障 →  x86: 空操作     │
│ volatile 写后 → 任意读    →  StoreLoad 屏障 →  x86: lock addl  │
│                                                                  │
│ monitorEnter 前 → 任意读写 →  LoadLoad + LoadStore + StoreStore │
│ monitorExit 后  → 任意读写 →  StoreLoad                         │
│                                                                  │
│ Thread.start() 前 → 任意读写 →  StoreStore + LoadStore          │
│ Thread.join() 后  → 任意读写 →  LoadLoad + LoadStore            │
└──────────────────────────────────────────────────────────────────┘
```

---

## 六、synchronized 的内存语义

### 6.1 synchronized 的 happens-before 保证

```
synchronized 的内存语义等价于：

进入 synchronized 块（monitorenter）：
  ┌─────────────────────────────────────────────┐
  │  1. 获取监视器锁                              │
  │  2. LoadLoad 屏障（确保后续读不被重排）         │
  │  3. LoadStore 屏障                           │
  │  = acquire 语义                              │
  └─────────────────────────────────────────────┘

退出 synchronized 块（monitorexit）：
  ┌─────────────────────────────────────────────┐
  │  1. StoreStore 屏障（确保之前的写先完成）     │
  │  2. StoreLoad 屏障                           │
  │  3. 释放监视器锁                              │
  │  = release 语义                              │
  └─────────────────────────────────────────────┘

等价关系：
  synchronized 块内的操作 happens-before 后续 synchronized 块内的操作
  = monitorenter + acquire + 操作 + release + monitorexit
```

### 6.2 从字节码到 HotSpot 的调用链

```
Java 代码:                           字节码:
synchronized(obj) {                  monitorenter    ← 进入同步块
  // critical section                ...
}                                    monitorexit     ← 退出同步块

HotSpot 调用链:
  monitorenter 字节码
    → InterpreterGenerator::generate_monitorenter_entry()
    → ObjectSynchronizer::fast_enter()
        → if (UseBiasedLocking):
              BiasedLocking::revoke_and_rebias()  ← 偏向锁
          else:
              ObjectSynchronizer::slow_enter()    ← 轻量级锁/重量级锁

  monitorexit 字节码
    → InterpreterGenerator::generate_monitorexit_entry()
    → ObjectSynchronizer::fast_exit()
        → CAS 释放轻量级锁
        或 → ObjectMonitor::exit()                 ← 重量级锁退出
```

---

## 七、final 字段的内存语义

### 7.1 final 的可见性保证

```
JMM 对 final 的保证：
  在构造函数中设置 final 字段的值，
  happens-before 其他线程读取该对象引用。

HotSpot 实现：
  1. 构造函数返回前，编译器会插入 StoreStore 屏障
     确保所有 final 字段写入在对象引用对其他线程可见之前完成
  2. 这是通过 "Freeze" 操作实现的

源码位置：
  src/share/vm/runtime/synchronizer.cpp
  src/share/vm/c1/c1_LIR.cpp（C1 编译器的 final 写入处理）
```

---

## 八、源码文件索引

| 文件 | 路径 | 说明 |
|------|------|------|
| orderAccess.hpp | `src/share/vm/runtime/orderAccess.hpp` | 内存屏障接口定义 |
| orderAccess.inline.hpp | `src/share/vm/runtime/orderAccess.inline.hpp` | 内存屏障内联实现 |
| atomic.hpp | `src/share/vm/runtime/atomic.hpp` | 原子操作接口 |
| atomic.inline.hpp | `src/share/vm/runtime/atomic.inline.hpp` | 原子操作内联实现 |
| atomic.cpp | `src/share/vm/runtime/atomic.cpp` | 原子操作非内联实现 |
| synchronizer.hpp | `src/share/vm/runtime/synchronizer.hpp` | 对象同步器接口 |
| synchronizer.cpp | `src/share/vm/runtime/synchronizer.cpp` | 锁膨胀/释放实现 |
| objectMonitor.hpp | `src/share/vm/runtime/objectMonitor.hpp` | 重量级监视器 |
| objectMonitor.cpp | `src/share/vm/runtime/objectMonitor.cpp` | 重量级监视器实现 |
| unsafe.cpp | `src/share/vm/prims/unsafe.cpp` | Unsafe CAS/park 实现 |
| markOop.hpp | `src/share/vm/oops/markOop.hpp` | Mark Word 定义 |
| biasedLocking.hpp | `src/share/vm/runtime/biasedLocking.hpp` | 偏向锁实现 |
| biasedLocking.cpp | `src/share/vm/runtime/biasedLocking.cpp` | 偏向锁逻辑 |
| park.hpp/cpp | `src/share/vm/runtime/park.hpp/cpp` | Parker/ParkEvent |
| basicLock.hpp | `src/share/vm/runtime/basicLock.hpp` | 轻量级锁记录 |
| x86/orderAccess_x86.inline.hpp | `src/cpu/x86/vm/orderAccess_x86.inline.hpp` | x86 屏障实现 |
| sparc/orderAccess_sparc.inline.hpp | `src/cpu/sparc/vm/orderAccess_sparc.inline.hpp` | SPARC 屏障实现 |
| arm/orderAccess_arm.inline.hpp | `src/cpu/arm/vm/orderAccess_arm.inline.hpp` | ARM 屏障实现 |

---

*下一篇：[02_Object_Header.md](02_Object_Header.md) — Java 对象头与 Mark Word 深度分析*