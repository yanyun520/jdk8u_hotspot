# HotSpot 线程模型深度分析

> 基于 OpenJDK 8u HotSpot 源码的线程架构解析

---

## 一、线程体系架构

```
┌──────────────────────────────────────────────────────────────────┐
│                     HotSpot 线程体系                              │
│                                                                  │
│                      Thread (C++ 基类)                            │
│                         │                                        │
│              ┌──────────┴──────────┐                              │
│              │                     │                              │
│        JavaThread              NonJavaThread                      │
│        (Java业务线程)          (JVM内部线程)                       │
│              │                     │                              │
│    ┌────────┼────────┐    ┌───────┼────────┐                     │
│    │        │        │    │       │        │                     │
│  用户线程  Daemon线程  │  VMThread  CGCThread ServiceThread         │
│                      │    │                         │              │
│                      │    │                  WorkerThread          │
│                CompilerThread  WatcherThread       │              │
│                                                  │              │
│                                           G1RefineThread          │
│                                           G1ConcurrentMarkThread │
│                                           G1FullGCTask            │
└──────────────────────────────────────────────────────────────────┘
```

### 1.1 Thread 基类

```cpp
// 源码位置: src/share/vm/runtime/thread.hpp

class Thread: public ThreadShadow {
 private:
  // 线程本地分配缓冲区 (TLAB)
  char*         _tlab_start;
  char*         _tlab_end;
  unsigned int  _tlab_top;  // 下一个空闲位置
  
  // 线程标识
  jlong         _resource_area;  // 资源区
  int           _thread_state;    // 线程状态
  
  // 句柄区
  HandleArea*   _handle_area;    // JNI 句柄区
  
  // 安全点支持
  JavaThread*   _pending_async_exception;  // 待处理的异步异常
  
 public:
  // 线程状态枚举
  enum ThreadState {
    _thread_uninitialized,     // 未初始化
    _thread_new,               // 新建
    _thread_new_trans,         // 新建→运行中（过渡状态）
    _thread_in_native,         // 在本地方法中
    _thread_in_native_trans,   // 本地方法→运行中（过渡状态）
    _thread_in_vm,             // 在 VM 内部
    _thread_in_vm_trans,       // VM内部→运行中（过渡状态）
    _thread_in_Java,           // 在 Java 代码中
    _thread_in_Java_trans,     // Java→运行中（过渡状态）
    _thread_blocked,           // 阻塞（等待锁）
    _thread_blocked_trans,     // 阻塞→运行中（过渡状态）
    ...
  };
};
```

### 1.2 JavaThread 类

```cpp
// 源码位置: src/share/vm/runtime/thread.hpp

class JavaThread: public Thread {
 private:
  // Java 栈帧指针
  intptr_t*        _stack_base;    // 栈底
  intptr_t*        _stack_limit;   // 栈限制
  
  // 线程对象
  oop              _threadObj;      // java.lang.Thread 对象
  
  // 锁相关
  Parker*          _parker;         // JUC park/unpark 支持
  
  // Monitor 缓存
  ObjectMonitor*   _current_pending_monitor;      // 等待获取的 Monitor
  ObjectMonitor*   _current_pending_monitor_is_from_java;  // 是否来自 Java
  
  // 安全点
  volatile bool    _at_poll_safepoint;  // 是否在安全点轮询处
  
  // 线程局部变量
  int              _vm_result;      // VM 操作结果
  
 public:
  Parker*          parker() { return _parker; }
  
  // 线程状态转换
  void set_thread_state(JavaThreadState state);
  JavaThreadState thread_state() const;
};
```

---

## 二、线程状态机

### 2.1 Java 线程状态 vs JVM 线程状态

```
┌─────────────────────────────────────────────────────────────────────┐
│              java.lang.Thread.State (Java层面)                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  NEW          ──→  RUNNABLE  ──→  BLOCKED  ──→  RUNNABLE           │
│  (新建)            (可运行)       (阻塞)          (可运行)            │
│                                                                     │
│  RUNNABLE  ──→  WAITING  ──→  RUNNABLE                              │
│  (可运行)      (等待)      (可运行)                                   │
│                                                                     │
│  RUNNABLE  ──→  TIMED_WAITING  ──→  RUNNABLE                         │
│  (可运行)      (限时等待)        (可运行)                              │
│                                                                     │
│  RUNNABLE  ──→  TERMINATED                                           │
│  (可运行)      (终止)                                                │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│              HotSpot ThreadState (JVM层面)                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  _thread_new           → Java NEW                                   │
│  _thread_in_Java       → Java RUNNABLE (正在执行Java代码)            │
│  _thread_in_native     → Java RUNNABLE (正在执行本地方法)            │
│  _thread_in_vm         → Java RUNNABLE (正在执行VM内部操作)          │
│  _thread_blocked       → Java BLOCKED/WAITING/TIMED_WAITING         │
│                                                                     │
│  _thread_uninitialized → 线程对象创建但未启动                         │
│                                                                     │
│  过渡状态（_trans 后缀）确保安全点检查的正确性：                       │
│  _thread_new_trans      → _thread_new 到 _thread_in_Java 的过渡     │
│  _thread_in_native_trans → _thread_in_native 到 _thread_in_Java     │
│  _thread_in_vm_trans    → _thread_in_vm 到 _thread_in_Java          │
│  _thread_blocked_trans  → _thread_blocked 到 _thread_in_Java        │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 状态转换的安全点保证

```
线程状态转换与安全点的关系：

  任何状态转换到 _thread_in_Java 之前，必须检查安全点！

  ┌──────────┐    _thread_in_Java_trans    ┌──────────┐
  │ native/  │ ──────────────────────────→ │ _thread_ │
  │ vm/      │    检查 SafepointSynchronize│ in_Java  │
  │ blocked  │    如果有挂起的安全点请求，   │          │
  └──────────┘    则阻塞等待安全点结束     └──────────┘

  安全点机制保证：
  1. 所有线程在安全点处暂停
  2. GC、偏向锁撤销等操作在安全点执行
  3. 状态转换时检查是否需要进入安全点
```

---

## 三、线程栈与栈帧

### 3.1 Java 线程栈结构

```
线程栈内存布局（从高地址到低地址）：

高地址 ┌─────────────────────────────────────────────────────┐
       │                   栈保护页                          │
       │              (Stack Guard Pages)                    │
       │              用于检测栈溢出                           │
       ├─────────────────────────────────────────────────────┤
       │                   栈帧区域                          │
       │                                                   │
       │  ┌───────────────────────────────────────────┐    │
       │  │  栈帧 N (当前方法)                          │    │
       │  │  ┌─────────────────────────────────────┐ │    │
       │  │  │  局部变量表 (Local Variables)          │ │    │
       │  │  ├─────────────────────────────────────┤ │    │
       │  │  │  操作数栈 (Operand Stack)             │ │    │
       │  │  ├─────────────────────────────────────┤ │    │
       │  │  │  帧数据 (Frame Data / Dynamic Link)  │ │    │
       │  │  ├─────────────────────────────────────┤ │    │
       │  │  │  Lock Records (同步块)               │ │    │
       │  │  │  BasicObjectLock { _lock, _obj }    │ │    │
       │  │  └─────────────────────────────────────┘ │    │
       │  └───────────────────────────────────────────┘    │
       │                                                   │
       │  ┌───────────────────────────────────────────┐    │
       │  │  栈帧 N-1 (调用者方法)                      │    │
       │  │  ...                                       │    │
       │  └───────────────────────────────────────────┘    │
       │  ...                                               │
       │  ┌───────────────────────────────────────────┐    │
       │  │  栈帧 0 (main 方法)                        │    │
       │  └───────────────────────────────────────────┘    │
       ├─────────────────────────────────────────────────────┤
       │                TLAB (线程本地分配缓冲区)            │
       └─────────────────────────────────────────────────────┘
低地址

线程栈参数：
  -Xss<size>           每个线程的栈大小（默认 512K-1M）
  -XX:StackShadowPages 栈保护页数（默认 20）
  -XX:ThreadStackSize  线程栈大小（字节）
```

### 3.2 TLAB（Thread-Local Allocation Buffer）

```
TLAB 是每个线程在 Eden 区中的私有分配缓冲区，用于快速无锁分配对象。

┌────────────────────────────────────────────────────────────────┐
│                        Eden 区                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │
│  │ Thread 1 │  │ Thread 2 │  │ Thread 3 │  │  未分配区域    │  │
│  │  TLAB    │  │  TLAB    │  │  TLAB    │  │              │  │
│  │ ┌──────┐ │  │ ┌──────┐ │  │ ┌──────┐ │  │              │  │
│  │ │next  │ │  │ │next  │ │  │ │next  │ │  │              │  │
│  │ │  ↓   │ │  │ │  ↓   │ │  │ │  ↓   │ │  │              │  │
│  │ │free  │ │  │ │free  │ │  │ │free  │ │  │              │  │
│  │ └──────┘ │  │ └──────┘ │  │ └──────┘ │  │              │  │
│  └──────────┘  └──────────┘  └──────────┘  └──────────────┘  │
│                                                                │
│  每个 TLAB 独立分配，无需同步！                                  │
│  当 TLAB 用完时，向 Eden 申请新的 TLAB                           │
└────────────────────────────────────────────────────────────────┘

TLAB 分配流程（bump-the-pointer）：
  1. 检查 TLAB 中是否有足够空间
  2. 如果有，移动 _tlab_top 指针，直接分配（无需 CAS）
  3. 如果没有，向 Eden 申请新的 TLAB（需要 CAS）
  4. 如果 Eden 也不够，触发 Minor GC

源码位置：
  src/share/vm/runtime/thread.hpp (TLAB 字段)
  src/share/vm/memory/threadLocalAllocBuffer.hpp (TLAB 类)
  src/share/vm/memory/threadLocalAllocBuffer.cpp (TLAB 实现)
```

---

## 四、Parker 与 Park/Unpark 机制

### 4.1 Parker 类

```cpp
// 源码位置: src/share/vm/runtime/park.hpp

class Parker : public os::PlatformParker {
 private:
  volatile int _counter;      // 许可计数器（0=无许可，1=有许可）
  Parker*      FreeNext;      // 空闲链表指针
  JavaThread*  AssociatedWith;// 关联的 Java 线程

 public:
  Parker() : PlatformParker() {
    _counter       = 0;       // 初始无许可
    FreeNext       = NULL;
    AssociatedWith = NULL;
  }

  void park(bool isAbsolute, jlong time);  // 阻塞线程
  void unpark();                           // 唤醒线程

  static Parker* Allocate(JavaThread* t);  // 分配 Parker
  static void    Release(Parker* e);      // 释放 Parker
};
```

### 4.2 Park/Unpark 工作原理

```
Park/Unpark 是 JUC 中 LockSupport 的底层实现：

  ┌─────────────────────────────────────────────────────────┐
  │  LockSupport.park()  ←→  Unsafe_Park() ←→ Parker::park()    │
  │  LockSupport.unpark(t) ←→ Unsafe_Unpark() ←→ Parker::unpark()│
  └─────────────────────────────────────────────────────────┘

Parker::park() 伪代码：
  ┌─────────────────────────────────────────────────────────┐
  │  1. 原子地将 _counter 置为 0                               │
  │  2. 如果 _counter > 0（已收到 unpark许可），立即返回       │
  │  3. 否则，在操作系统条件变量上等待                          │
  │  4. 被唤醒后，将 _counter 置为 0，返回                     │
  └─────────────────────────────────────────────────────────┘

Parker::unpark() 伪代码：
  ┌─────────────────────────────────────────────────────────┐
  │  1. 将 _counter 置为 1                                    │
  │  2. 如果目标线程正在等待，唤醒它（条件变量 signal）         │
  └─────────────────────────────────────────────────────────┘

关键特性：
  - unpark 可以在 park 之前调用（先发放"许可"）
  - unpark 是精确唤醒某个线程（不是 notifyAll）
  - park 可能被虚假唤醒（spurious wakeup）
  - park 只消耗一个许可（多次 unpark 也只等于一次）
```

### 4.3 Unsafe_Park/Unsafe_Unpark 的 JNI 实现

```cpp
// 源码位置: src/share/vm/prims/unsafe.cpp

UNSAFE_ENTRY(void, Unsafe_Park(JNIEnv *env, jobject unsafe, 
                                jboolean isAbsolute, jlong time))
  UnsafeWrapper("Unsafe_Park");
  EventThreadPark event;
  JavaThreadParkedState jtps(thread, time != 0);
  // 调用当前线程的 Parker 对象的 park 方法
  thread->parker()->park(isAbsolute != 0, time);
  if (event.should_commit()) {
    oop obj = thread->current_park_blocker();
    event.set_klass((obj != NULL) ? obj->klass() : NULL);
    event.set_timeout(time);
    event.set_address((obj != NULL) ? (TYPE_ADDRESS) cast_from_oop<uintptr_t>(obj) : 0);
    event.commit();
  }
UNSAFE_END

UNSAFE_ENTRY(void, Unsafe_Unpark(JNIEnv *env, jobject unsafe, jobject jthread))
  UnsafeWrapper("Unsafe_Unpark");
  Parker* p = NULL;
  if (jthread != NULL) {
    oop java_thread = JNIHandles::resolve_non_null(jthread);
    if (java_thread != NULL) {
      jlong lp = java_lang_Thread::park_event(java_thread);
      if (lp != 0) {
        p = (Parker*)addr_from_java(lp);
      } else {
        // 首次 unpark，需要获取线程的 Parker
        MutexLocker mu(Threads_lock);
        java_thread = JNIHandles::resolve_non_null(jthread);
        if (java_thread != NULL) {
          JavaThread* thr = java_lang_Thread::thread(java_thread);
          if (thr != NULL) {
            p = thr->parker();
            if (p != NULL) {
              // 缓存 Parker 地址到 Thread 对象中
              java_lang_Thread::set_park_event(java_thread, addr_to_java(p));
            }
          }
        }
      }
    }
  }
  if (p != NULL) {
    p->unpark();  // 调用 Parker::unpark()
  }
UNSAFE_END
```

---

## 五、VMThread 与安全点

### 5.1 VMThread

```
VMThread 是 JVM 内部的核心线程，负责执行需要全局暂停的操作：

  ┌─────────────────────────────────────────────────────────┐
  │  VMThread 职责                                           │
  │  ┌──────────────────────────────────────────────────┐  │
  │  │  - GC 操作（STW 暂停）                             │  │
  │  │  - 偏向锁批量撤销/重偏向                             │  │
  │  │  - 类卸载                                           │  │
  │  │  - 代码反优化                                       │  │
  │  │  - 线程栈转储                                       │  │
  │  │  - JVMTI 操作                                      │  │
  │  │  - 打印线程信息                                     │  │
  │  └──────────────────────────────────────────────────┘  │
  │                                                         │
  │  VMThread 使用 VM_Operation 队列来执行操作：             │
  │  ┌──────────────────────────────────────────────────┐  │
  │  │  VM_Operation 队列                                 │  │
  │  │  ┌───────┐  ┌───────┐  ┌───────┐                │  │
  │  │  │ GC op │→│Deopt  │→│Print  │→ ...              │  │
  │  │  └───────┘  └───────┘  └───────┘                │  │
  │  └──────────────────────────────────────────────────┘  │
  └─────────────────────────────────────────────────────────┘

源码位置：
  src/share/vm/runtime/vmThread.hpp
  src/share/vm/runtime/vmThread.cpp
  src/share/vm/runtime/vm_operations.hpp
```

### 5.2 安全点（Safepoint）

```
安全点是所有线程暂停执行的位置，VM 可以安全地执行全局操作。

安全点触发流程：
  ┌──────────────────────────────────────────────────────┐
  │  1. VMThread 请求安全点                               │
  │     SafepointSynchronize::begin()                     │
  │                                                      │
  │  2. 设置全局标志 _pending = 1                         │
  │                                                      │
  │  3. 等待所有线程到达安全点                              │
  │     每个线程在以下位置检查安全点标志：                   │
  │     - 方法返回前                                      │
  │     - 循环回跳前                                      │
  │     - 本地方法调用前后                                 │
  │     - 状态转换时                                      │
  │                                                      │
  │  4. 所有线程到达安全点后，执行 VM 操作                  │
  │                                                      │
  │  5. 执行完毕，SafepointSynchronize::end()             │
  │     唤醒所有线程继续执行                                │
  └──────────────────────────────────────────────────────┘

安全点检查的编译器实现（x86）：
  ┌──────────────────────────────────────────────────────┐
  │  // 解释器中的安全点轮询                               │
  │  // 每个方法入口和循环回跳前插入                       │
  │  test [polling_page], eax  // 访问保护页              │
  │  // 如果安全点请求已设置，此指令会触发 SIGSEGV         │
  │  // JVM 的信号处理器捕获后进入安全点                   │
  └──────────────────────────────────────────────────────┘

源码位置：
  src/share/vm/runtime/safepoint.cpp
  src/share/vm/runtime/interfaceSupport.hpp
```

---

## 六、线程同步原语

### 6.1 Mutex 与 Monitor（JVM 内部）

```
HotSpot 内部使用的同步原语（不是 Java 层面的 Monitor）：

┌──────────────────────────────────────────────────────────────┐
│  Mutex（互斥锁）                                              │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  用于保护 JVM 内部数据结构                              │    │
│  │  类型：                                                │    │
│  │  - Mutex::_no_safepoint_check_flag：不需要安全点检查    │    │
│  │  - Mutex::_safepoint_check_flag：需要安全点检查         │    │
│  │                                                        │    │
│  │  常见 Mutex：                                          │    │
│  │  - Threads_lock：线程列表锁                            │    │
│  │  - Heap_lock：堆锁                                     │    │
│  │  - CodeCache_lock：代码缓存锁                          │    │
│  │  - SystemDictionary_lock：系统字典锁                    │    │
│  │  - Safepoint_lock：安全点锁                            │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                              │
│  Monitor（条件变量 + 互斥锁）                                 │
│  ┌──────────────────────────────────────────────────────┐    │
│  │  在 Mutex 基础上增加了 wait/notify 功能                │    │
│  │  用于 JVM 内部的等待/通知机制                          │    │
│  │                                                        │    │
│  │  常见 Monitor：                                        │    │
│  │  - CGC_lock：GC 线程等待/通知                          │    │
│  │  - FullGC_lock：Full GC 等待/通知                      │    │
│  │  - Service_lock：Service Thread 等待                    │    │
│  └──────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘

源码位置：
  src/share/vm/runtime/mutex.hpp
  src/share/vm/runtime/mutex.cpp
  src/share/vm/runtime/mutexLocker.hpp（常用锁定义）
```

### 6.2 ParkEvent

```cpp
// 源码位置: src/share/vm/runtime/park.hpp

// ParkEvent 用于 JVM 内部的线程等待/唤醒
// 与 Parker 类似但用于 synchronized 的 Monitor 等待

class ParkEvent : public os::PlatformEvent {
 private:
  ParkEvent* FreeNext;
  Thread*    AssociatedWith;     // 关联的线程
  intptr_t   RawThreadIdentity;  // LWPID
  volatile int Incarnation;

 public:
  ParkEvent* volatile ListNext;  // CLH 队列链接
  ParkEvent* volatile ListPrev;
  volatile intptr_t OnList;
  volatile int TState;           // 线程状态
  volatile int Notified;         // 是否被通知
  volatile int IsWaiting;        // 是否在等待

  void park();                   // 无限期等待
  void park(jlong millis);       // 限时等待
  void unpark();                 // 唤醒
};

// ParkEvent 与 Parker 的区别：
// ParkEvent  → 用于 synchronized 的 Monitor 等待（ObjectMonitor 内部）
// Parker     → 用于 JUC 的 LockSupport.park/unpark
```

---

## 七、线程创建与启动

### 7.1 Java 线程创建流程

```
Java 代码:                    JNI 调用链:
new Thread(r)        →       Thread.<init>()
t.start()            →       Thread.start0() (native)
                               ↓
                          JVM_StartThread()
                           ↓
                          new JavaThread()
                           ↓
                          os::create_thread()
                           ↓
                          线程启动 → thread_entry()
                           ↓
                          JavaCalls::call_virtual(&ts, threadObj, ...)
                           ↓
                          Thread.run() 方法执行

源码位置：
  src/share/vm/prims/jvm.cpp (JVM_StartThread)
  src/share/vm/runtime/thread.cpp (JavaThread::JavaThread, thread_entry)
```

### 7.2 Thread.start() 的 JVM 实现

```cpp
// 源码位置: src/share/vm/prims/jvm.cpp

JVM_ENTRY(void, JVM_StartThread(JNIEnv* env, jobject jthread))
  JVMWrapper("JVM_StartThread");
  oop java_thread = JNIHandles::resolve_non_null(jthread);
  
  // 获取线程组、栈大小等参数
  oop group = java_lang_Thread::threadGroup(java_thread);
  jlong stack_size = java_lang_Thread::stackSize(java_thread);
  
  // 创建 JavaThread
  JavaThread* native_thread = new JavaThread(&thread_entry, stack_size);
  
  // 将 Java Thread 对象与 native 线程关联
  native_thread->prepare(thread_group);
  
  // 启动底层操作系统线程
  Thread::start(native_thread);
JVM_END

// 线程入口函数
static void thread_entry(JavaThread* thread, TRAPS) {
  HandleMark hm(THREAD);
  Handle obj(THREAD, thread->threadObj());
  // 调用 Thread.run()
  JavaCalls::call_virtual(&result, obj, KlassHandle(THREAD, SystemDictionary::Thread_klass()),
                          vmSymbols::run_method_name(), vmSymbols::void_method_signature(), THREAD);
}
```

---

## 八、线程与锁的交互全景图

```
┌────────────────────────────────────────────────────────────────────┐
│                     线程与锁的交互全景图                              │
│                                                                    │
│  ┌─────────────┐      synchronized        ┌──────────────────┐    │
│  │ Java 线程    │ ──── monitorenter ─────→ │ ObjectSynchronizer│    │
│  │             │                           │  fast_enter()     │    │
│  │             │                           │  slow_enter()     │    │
│  │             │                           └────────┬─────────┘    │
│  │             │                                    │              │
│  │             │                    ┌───────────────┼──────────┐   │
│  │             │                    │               │          │   │
│  │             │              偏向锁路径      轻量级锁路径   重量级锁  │
│  │             │                    │               │          │   │
│  │             │              BiasedLocking    CAS+Lock Rec  ObjectMonitor│
│  │             │              ::revoke_and_rebias              │   │
│  │             │                    │               │          │   │
│  │             │              ┌─────┴─────┐  ┌─────┴────┐  ┌──┴──┐│
│  │             │              │比较线程ID   │  │CAS替换   │  │EntryList││
│  │             │              │相等→获取   │  │Mark Word │  │cxq   ││
│  │             │              │不等→撤销   │  │→获取锁   │  │WaitSet││
│  │             │              └───────────┘  └──────────┘  └──────┘│
│  │             │                                            │    │
│  │             │      JUC 同步器                              │    │
│  │             │      ┌──────────────┐                       │    │
│  │             │      │AQS/ReentrantLock│                    │    │
│  │             │      │CountDownLatch    │                    │    │
│  │             │      │Semaphore         │                    │    │
│  │             │      └───────┬──────────┘                    │    │
│  │             │              │                               │    │
│  │             │      LockSupport.park()                      │    │
│  │             │      LockSupport.unpark()                    │    │
│  │             │              │                               │    │
│  │             │      Unsafe_Park/Unsafe_Unpark               │    │
│  │             │              │                               │    │
│  │             │      Parker::park()/unpark()                 │    │
│  │             │      ┌───────┴──────────┐                    │    │
│  │             │      │ 操作系统条件变量    │                    │    │
│  │             │      │ pthread_cond_t     │                    │    │
│  │             │      └────────────────────┘                    │    │
│  └─────────────┘                                              │    │
└────────────────────────────────────────────────────────────────────┘
```

---

## 九、源码文件索引

| 文件 | 路径 | 说明 |
|------|------|------|
| thread.hpp | `src/share/vm/runtime/thread.hpp` | Thread/JavaThread 定义 |
| thread.cpp | `src/share/vm/runtime/thread.cpp` | 线程创建/启动实现 |
| vmThread.hpp | `src/share/vm/runtime/vmThread.hpp` | VMThread 定义 |
| vmThread.cpp | `src/share/vm/runtime/vmThread.cpp` | VMThread 实现 |
| park.hpp | `src/share/vm/runtime/park.hpp` | Parker/ParkEvent 定义 |
| park.cpp | `src/share/vm/runtime/park.cpp` | Park 实现 |
| mutex.hpp | `src/share/vm/runtime/mutex.hpp` | Mutex/Monitor 定义 |
| mutex.cpp | `src/share/vm/runtime/mutex.cpp` | Mutex 实现 |
| mutexLocker.hpp | `src/share/vm/runtime/mutexLocker.hpp` | 常用锁定义 |
| safepoint.cpp | `src/share/vm/runtime/safepoint.cpp` | 安全点实现 |
| interfaceSupport.hpp | `src/share/vm/runtime/interfaceSupport.hpp` | 安全点检查宏 |
| jvm.cpp | `src/share/vm/prims/jvm.cpp` | JVM_Thread_Start 等 |
| unsafe.cpp | `src/share/vm/prims/unsafe.cpp` | Unsafe_Park/Unpark |
| threadLocalAllocBuffer.hpp | `src/share/vm/memory/threadLocalAllocBuffer.hpp` | TLAB 定义 |
| threadLocalAllocBuffer.cpp | `src/share/vm/memory/threadLocalAllocBuffer.cpp` | TLAB 实现 |
| osThread.hpp | `src/share/vm/runtime/osThread.hpp` | 操作系统线程封装 |

---

*上一篇：[02_Object_Header.md](02_Object_Header.md) | 下一篇：[04_JUC_Synchronizers.md](04_JUC_Synchronizers.md) — JUC 核心同步器与 HotSpot 交互分析*