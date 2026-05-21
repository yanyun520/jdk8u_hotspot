# HotSpot JVM GC 垃圾回收 — 总体架构与底层原理

> 基于 OpenJDK 8u HotSpot 源码的深度分析

---

## 一、GC 的本质：为什么要垃圾回收？

**通俗比喻**：想象你住在一栋公寓楼（堆内存）里。每个房间（对象）住着人（数据引用）。当一个人不再被任何邻居提起（没有引用指向他），他就应该搬走（被回收），腾出房间给新住户（新对象）。

垃圾回收的核心问题：
1. **哪些对象还活着？** → 可达性分析（Reachability Analysis）
2. **如何高效回收死对象？** → 各种收集算法
3. **何时回收？** → 分配失败 / 阈值触发
4. **回收后如何安排存活对象？** → 压缩 / 整理

---

## 二、HotSpot GC 架构总览

### 2.1 类继承体系

```
CollectedHeap (gc_interface/collectedHeap.hpp)
├── SharedHeap (memory/sharedHeap.hpp)
│   ├── GenCollectedHeap (memory/genCollectedHeap.hpp)
│   │   └── 使用 Serial/ParNew + CMS/Serial Old 组合
│   └── G1CollectedHeap (g1/g1CollectedHeap.hpp)
│       └── 使用 G1 收集器
└── ParallelScavengeHeap (parallelScavenge/parallelScavengeHeap.hpp)
    └── 使用 Parallel Scavenge + Parallel Old 组合
```

### 2.2 分代模型（Generational Model）

HotSpot 采用**分代收集**思想：不同生命周期的对象放在不同区域，用不同策略回收。

```
┌─────────────────────────────────────────────────────────────┐
│                        Java Heap                             │
│  ┌───────────────────────────────────────┐                  │
│  │           Young Generation             │                  │
│  │  ┌────────────┬──────────┬──────────┐ │                  │
│  │  │    Eden    │ Survivor │ Survivor │ │                  │
│  │  │   Space    │   From   │    To    │ │                  │
│  │  └────────────┴──────────┴──────────┘ │                  │
│  └───────────────────────────────────────┘                  │
│  ┌───────────────────────────────────────┐                  │
│  │            Old Generation              │                  │
│  │  (Tenured / CMS / Parallel Old)       │                  │
│  └───────────────────────────────────────┘                  │
└─────────────────────────────────────────────────────────────┘
```

**为什么分代？** 
- 统计发现：**98% 的对象都是朝生夕死**（弱代假说）
- 新对象分配在 Eden，Minor GC 频繁但快速
- 存活多次的对象晋升到 Old，Major GC 少但慢

### 2.3 Generation 继承体系

```
Generation (memory/generation.hpp)
├── DefNewGeneration (memory/defNewGeneration.hpp)
│   └── ParNewGeneration (parNew/parNewGeneration.hpp)
├── CardGeneration (memory/generation.hpp)
│   ├── OneContigSpaceCardGeneration
│   │   └── TenuredGeneration (memory/tenuredGeneration.hpp)
│   └── ConcurrentMarkSweepGeneration (concurrentMarkSweep/concurrentMarkSweepGeneration.hpp)
│       └── ASConcurrentMarkSweepGeneration
├── PSYoungGen (parallelScavenge/psYoungGen.hpp)
└── PSOldGen (parallelScavenge/psOldGen.hpp)
```

### 2.4 收集器组合矩阵

| 年轻代收集器 | 老年代收集器 | 组合描述 |
|:---:|:---:|:---|
| Serial (DefNew) | Serial Old (Tenured) | 单线程，Client 模式默认 |
| ParNew | CMS | 多线程年轻代 + 并发老年代 |
| ParNew | Serial Old | ParNew + Serial Old（CMS 后备） |
| Parallel Scavenge | Serial Old | 吞吐量优先年轻代 + Serial Old |
| Parallel Scavenge | Parallel Old | 吞吐量优先全并行 |
| G1 | G1 | 区域化收集器，统一管理年轻代和老年代 |

---

## 三、可达性分析（Root Tracing）

所有 GC 算法的第一步都是确定**哪些对象还活着**。

### 3.1 GC Roots

**源码位置**：`src/share/vm/memory/sharedHeap.cpp` → `process_roots()`

```
GC Roots:
├── 虚拟机栈帧中的局部变量表（线程栈）
├── 方法区中的类静态属性（Klass::_java_mirror）
├── 方法区中的常量（ConstantPool）
├── 本地方法栈中的 JNI 引用（JNIHandles）
├── 同步锁持有的对象（ObjectMonitor）
├── JMX / JVMTI 暂存引用
└── 待终结队列（Finalizer）
```

### 3.2 引用类型

| 引用类型 | 类 | GC 行为 | 典型用途 |
|----------|-----|---------|---------|
| 强引用 | `Object ref = obj` | 永不回收 | 正常编程 |
| 软引用 | `SoftReference` | 内存不足时回收 | 缓存 |
| 弱引用 | `WeakReference` | 下次 GC 回收 | WeakHashMap |
| 虚引用 | `PhantomReference` | 随时回收，仅通知 | 跟踪 GC |

**源码位置**：`src/share/vm/memory/referenceProcessor.cpp`

---

## 四、安全点（Safepoint）

GC 必须在**安全点**执行，确保所有线程的引用状态一致。

**源码位置**：`src/share/vm/runtime/safepoint.cpp`

```
安全点触发方式：
1. 解释执行：检查方法返回时的安全点轮询
2. 编译代码：在方法调用/循环回边插入安全点轮询
3. JNI 代码：从 JNI 返回时检查
```

**通俗比喻**：安全点就像是高速路上的服务区——所有车（线程）必须在服务区停下，才能统一清点人数（GC）。

---

## 五、内存屏障与写屏障

GC 需要跟踪跨代引用，这依赖**写屏障（Write Barrier）**。

### 5.1 Card Table（卡表）

**源码位置**：`src/share/vm/memory/cardTableRS.hpp`, `src/share/vm/memory/cardTableModRefBS.hpp`

```
堆被划分为 512 字节的"卡片"（Card）
每个卡片对应卡表中 1 字节

Card Table:
┌───┬───┬───┬───┬───┬───┬───┬───┐
│ 0 │ 0 │ 1 │ 0 │ 0 │ 1 │ 0 │ 0 │  ← 1=脏卡，有跨代引用
└───┴───┴───┴───┴───┴───┴───┴───┘
  ↑       ↑       ↑
  │       │       │
  卡片    卡片    卡片
  干净    脏     脏
```

写屏障在每次引用写入时标记对应卡片为脏，Minor GC 时只需扫描脏卡。

### 5.2 SATB 屏障（G1 使用）

**源码位置**：`src/share/vm/gc_implementation/g1/g1SATBCardTableModRefBS.hpp`

```
SATB (Snapshot At The Beginning) 写屏障：
- 在引用被覆盖前，将旧引用记录到 SATB 队列
- 保证并发标记开始时的对象图快照一致性
- 源码：satbQueue.cpp / satbQueue.hpp
```

---

## 六、对象分配流程

**源码位置**：`src/share/vm/memory/defNewGeneration.cpp` → `allocate()`, `src/share/vm/gc_implementation/g1/g1CollectedHeap.cpp` → `attempt_allocation()`

```
对象分配流程：
┌──────────────┐
│  new Object() │
└──────┬───────┘
       │
       ▼
┌──────────────────────────┐
│ TLAB 快速分配（线程本地） │ ← src/share/vm/memory/threadLocalAllocBuffer.cpp
└──────┬───────────────────┘
       │ TLAB 用完
       ▼
┌──────────────────────────┐
│ Eden 区 Bump-the-pointer  │ ← 快速指针碰撞分配
└──────┬───────────────────┘
       │ Eden 空间不足
       ▼
┌──────────────────────────┐
│ 触发 Minor GC             │
└──────┬───────────────────┘
       │ 仍然不足
       ▼
┌──────────────────────────┐
│ 扩展年轻代 / 分配到老年代 │
└──────┬───────────────────┘
       │ 仍然不足
       ▼
┌──────────────────────────┐
│ 触发 Full GC              │
└──────┬───────────────────┘
       │ 仍然不足
       ▼
    OOM!
```

---

## 七、源码文件索引

### 7.1 GC 接口层

| 文件 | 路径 | 说明 |
|------|------|------|
| `collectedHeap.cpp/.hpp` | `gc_interface/` | 堆抽象基类 |
| `gcCause.cpp/.hpp` | `gc_interface/` | GC 原因枚举 |
| `allocTracer.cpp/.hpp` | `gc_interface/` | 分配跟踪 |

### 7.2 内存管理层

| 文件 | 路径 | 说明 |
|------|------|------|
| `generation.cpp/.hpp` | `memory/` | 分代基类 |
| `genCollectedHeap.cpp/.hpp` | `memory/` | 分代收集堆 |
| `defNewGeneration.cpp/.hpp` | `memory/` | 默认新生代 |
| `tenuredGeneration.cpp/.hpp` | `memory/` | 老年代 |
| `space.cpp/.hpp` | `memory/` | 内存空间抽象 |
| `cardTableRS.cpp/.hpp` | `memory/` | 卡表记忆集 |
| `collectorPolicy.cpp/.hpp` | `memory/` | 收集器策略选择 |

### 7.3 各收集器源码

| 收集器 | 路径 | 核心文件 |
|--------|------|---------|
| CMS | `gc_implementation/concurrentMarkSweep/` | `concurrentMarkSweepGeneration.cpp/.hpp`, `compactibleFreeListSpace.cpp/.hpp`, `cmsCollectorPolicy.cpp/.hpp` |
| G1 | `gc_implementation/g1/` | `g1CollectedHeap.cpp/.hpp`, `g1CollectorPolicy.cpp/.hpp`, `concurrentMark.cpp/.hpp`, `heapRegion.cpp/.hpp` |
| Parallel Scavenge | `gc_implementation/parallelScavenge/` | `psScavenge.cpp/.hpp`, `psParallelCompact.cpp/.hpp`, `parallelScavengeHeap.cpp/.hpp` |
| ParNew | `gc_implementation/parNew/` | `parNewGeneration.cpp/.hpp` |
| Shared | `gc_implementation/shared/` | `adaptiveSizePolicy.cpp/.hpp`, `markSweep.cpp/.hpp`, `ageTable.cpp/.hpp` |

---

*下一文档：[02_Serial_GC.md](02_Serial_GC.md) — Serial 收集器详解*