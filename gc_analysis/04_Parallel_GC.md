# Parallel Scavenge + Parallel Old — 吞吐量优先收集器

> 源码路径：`src/share/vm/gc_implementation/parallelScavenge/`

---

## 一、概述

Parallel Scavenge（并行清除）收集器是 HotSpot 专为**吞吐量优先**场景设计的垃圾收集器。与 CMS 的"低延迟"目标不同，Parallel 的目标是**最大化应用吞吐量**。

**通俗比喻**：如果 CMS 是"悄悄打扫、不打扰住户"，那 Parallel 就是"全体住户暂停，所有清洁工同时上阵，快速打扫完恢复"——总停顿时间可能较长，但应用运行时间占比更高。

---

## 二、Parallel Scavenge — 年轻代并行收集

**核心源码**：`src/share/vm/gc_implementation/parallelScavenge/psScavenge.cpp/.hpp`

### 2.1 算法：复制算法（多线程并行）

```
与 ParNew 相同的复制算法，但有以下区别：
1. 自适应大小策略（PSAdaptiveSizePolicy）
2. 吞吐量目标导向（MaxGCPauseMillis / GCTimeRatio）
3. 更精确的 Eden/Survivor/Old 大小调整

 ┌───────────────────────────────────────────────┐
 │ 年轻代（PSYoungGen）                            │
 │ ┌─────────────────┬──────┬──────┐            │
 │ │     Eden         │ From │  To  │            │
 │ │  ████████████    │ ████ │      │            │
 │ │  对象分配区      │存活区│空闲区│            │
 │ └─────────────────┴──────┴──────┘            │
 └───────────────────────────────────────────────┘
         ↓ Minor GC（并行复制）
 ┌───────────────────────────────────────────────┐
 │ 存活对象 → To Survivor 或 Old Gen             │
 └───────────────────────────────────────────────┘
```

### 2.2 核心类

```
ParallelScavengeHeap : public CollectedHeap
  ├── PSYoungGen          → 年轻代（Eden + 2 Survivor）
  ├── PSOldGen             → 老年代
  ├── PSAdaptiveSizePolicy → 自适应大小策略
  └── GCTaskManager        → GC 任务管理器

PSScavenge（静态工具类）
  ├── invoke()             → 入口
  ├── invoke_no_policy()   → 核心清除逻辑
  └── should_scavenge()    → 判断对象是否应被清除
```

### 2.3 核心源码分析

**`PSScavenge::invoke_no_policy()`** 简化流程：

```cpp
void PSScavenge::invoke_no_policy(bool clear_all_softrefs) {
  // 1. 设置引用处理器
  // 2. 初始化扫描闭包
  // 3. 并行扫描根集
  //    - 使用 PSPromotionManager 管理每个线程的晋升
  //    - 每个 GC 线程独立的 PSPromotionLAB
  // 4. 复制存活对象到 To Survivor 或晋升到老年代
  // 5. 处理引用（Soft/Weak/Phantom）
  // 6. 更新年龄表
  // 7. 交换 From/To
  // 8. 更新自适应策略统计
}
```

### 2.4 自适应大小策略

**源码**：`psAdaptiveSizePolicy.cpp/.hpp`

```
PSAdaptiveSizePolicy 根据三个目标自动调整堆大小：

1. 最大 GC 停顿时间目标（MaxGCPauseMillis）
   → 调整年轻代大小，使 GC 停顿不超过目标

2. 吞吐量目标（GCTimeRatio）
   → GC 时间占比 = 1/(1+GCTimeRatio)
   → 默认 GCTimeRatio=99 → GC 占比 ≤ 1%

3. 最大堆使用率（MaxHeapFreeRatio/MinHeapFreeRatio）
   → 控制堆的扩展与收缩

调整策略：
┌─────────────────────────────────────────────────┐
│ GC 停顿 > 目标 → 缩小年轻代                     │
│ GC 停顿 < 目标 → 增大年轻代（提升吞吐量）        │
│ GC 时间比 > 目标 → 增大总堆                      │
│ GC 时间比 < 目标 → 可以缩小总堆                  │
└─────────────────────────────────────────────────┘
```

---

## 三、Parallel Old — 老年代并行压缩

**核心源码**：`src/share/vm/gc_implementation/parallelScavenge/psParallelCompact.cpp/.hpp`

### 3.1 算法：标记-压缩（并行）

```
Parallel Old 使用标记-压缩算法（Mark-Compact），而非 CMS 的标记-清除：

步骤1: 标记（Mark）
  ┌─────────────────────────────────────────────┐
  │ 🟢🔴🔴🟢🔴🟢🔴🔴🟢🔴🟢🔴🔴🟢🔴🟢🔴🔴 │
  └─────────────────────────────────────────────┘

步骤2: 计算压缩目标地址（Summary）
  ┌─────────────────────────────────────────────┐
  │ 🟢→0 🟢→4 🟢→8 🟢→10 🟢→13 🟢→15 🟢→16  │
  └─────────────────────────────────────────────┘

步骤3: 调整指针（Adjust Pointers）
  更新所有引用，指向新地址

步骤4: 移动对象（Compact）
  ┌─────────────────────────────────────────────┐
  │ 🟢🟢🟢🟢🟢🟢🟢│        空闲              │
  └─────────────────────────────────────────────┘
  无碎片！
```

### 3.2 核心类

```
PSParallelCompact（静态工具类）
  ├── ParallelCompactData → 压缩数据（标记位图 + 区域摘要）
  ├── SpaceInfo            → 空间信息（源/目标区域对）
  ├── SplitInfo            → 分裂信息（跨区域对象）
  └── Invoke 流程：
        invoke() → invoke_no_policy()
          → mark_phase()      → 标记
          → summary_phase()   → 计算新地址
          → adjust_roots()    → 调整根指针
          → compact_perm()    → 压缩永久代
          → compact()         → 压缩各代
```

### 3.3 并行标记-压缩的挑战

```
问题：如何并行计算压缩目标地址？

解决方案：区域化（Region-based Summary）

将老年代划分为固定大小的"摘要区域"(dence region)：
┌────┬────┬────┬────┬────┬────┬────┬────┐
│ R0 │ R1 │ R2 │ R3 │ R4 │ R5 │ R6 │ R7 │
│3活 │2活 │4活 │1活 │3活 │0活 │2活 │1活 │
└────┴────┴────┴────┴────┴────┴────┴────┘

步骤1: 并行统计每个区域的存活数据量
步骤2: 前缀和计算每个区域的起始目标地址
步骤3: 每个线程负责一个区域，并行移动对象
```

### 3.4 核心源码分析

**`PSParallelCompact::invoke_no_policy()`**：

```cpp
void PSParallelCompact::invoke_no_policy(bool maximum_heap_compaction) {
  // 1. 标记阶段：并行标记存活对象
  mark_phase();
  
  // 2. 摘要阶段：计算每个对象的新地址
  summary_phase();
  
  // 3. 调整根指针
  adjust_roots();
  
  // 4. 压缩永久代
  compact_perm();
  
  // 5. 压缩各代
  compact();
  
  // 6. 更新引用
  // 7. 重置标记位图
}
```

---

## 四、Parallel 收集器 vs CMS 对比

| 特性 | Parallel Scavenge/Old | CMS |
|------|----------------------|-----|
| 目标 | 吞吐量优先 | 停顿时间优先 |
| 年轻代算法 | 复制（并行） | 复制（ParNew并行） |
| 老年代算法 | 标记-压缩（并行） | 标记-清除（并发） |
| 碎片 | 无（压缩） | 有（不压缩） |
| 停顿时间 | 较长但可控 | 较短但不可控 |
| 吞吐量 | 高 | 较低 |
| 并发 | 否（全程STW） | 是（大部分并发） |
| 自适应 | 是（PSAdaptiveSizePolicy） | 是（CMSAdaptiveSizePolicy） |

---

## 五、核心参数调优

### 5.1 吞吐量目标

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `-XX:MaxGCPauseMillis` | 无限 | 最大 GC 停顿时间目标(ms) |
| `-XX:GCTimeRatio` | 99 | 吞吐量目标（GC时间比 ≤ 1/(1+GCTimeRatio)） |
| `-XX:+UseAdaptiveSizePolicy` | true | 自适应大小策略 |
| `-XX:+UseParallelOldGC` | 自动 | 老年代使用并行压缩 |

**吞吐量计算**：
```
GCTimeRatio = 99 → 目标 GC 时间占比 ≤ 1/(1+99) = 1%
GCTimeRatio = 19 → 目标 GC 时间占比 ≤ 1/(1+19) = 5%
```

### 5.2 线程数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `-XX:ParallelGCThreads` | CPU核数 | 并行 GC 线程数 |
| `-XX:UseDynamicNumberOfGCThreads` | false | 动态调整 GC 线程数 |

### 5.3 年轻代大小

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `-Xmn` | 自适应 | 新生代大小（固定值会禁用自适应） |
| `-XX:NewRatio` | 2 | 老年代/新生代比例 |
| `-XX:SurvivorRatio` | 8 | Eden/Survivor 比例 |
| `-XX:MaxTenuringThreshold` | 15 | 最大晋升年龄 |

**重要**：设置 `-Xmn` 固定值会禁用 `UseAdaptiveSizePolicy` 的年轻代调整！

### 5.4 晋升与 PLAB

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `-XX:YoungPLABSize` | 4096 | 年轻代 PLAB 大小 |
| `-XX:OldPLABSize` | 1024 | 老年代 PLAB 大小 |
| `-XX:ResizePLAB` | true | 动态调整 PLAB 大小 |

### 5.5 完整调优示例

```bash
# 吞吐量优先配置（8核 16GB 堆，批处理应用）
java -XX:+UseParallelGC -XX:+UseParallelOldGC \
     -XX:ParallelGCThreads=8 \
     -Xms16g -Xmx16g \
     -XX:GCTimeRatio=49 \
     -XX:MaxGCPauseMillis=200 \
     -XX:+UseAdaptiveSizePolicy \
     -XX:MaxTenuringThreshold=15 \
     -XX:SurvivorRatio=8 \
     -jar app.jar

# 说明：
# - GCTimeRatio=49 → GC 时间占比 ≤ 2%
# - MaxGCPauseMillis=200 → 单次 GC 停顿目标 200ms
# - UseAdaptiveSizePolicy → JVM 自动调整年轻代/老年代大小
# - 不设 -Xmn，让自适应策略决定年轻代大小
```

---

## 六、源码文件清单

### Parallel Scavenge 目录（60 个文件）

| 文件 | 说明 |
|------|------|
| `parallelScavengeHeap.cpp/.hpp` | 堆实现 |
| `psScavenge.cpp/.hpp` | 年轻代并行清除 |
| `psMarkSweep.cpp/.hpp` | 老年代标记-清除（旧版） |
| `psParallelCompact.cpp/.hpp` | 老年代并行压缩 |
| `psAdaptiveSizePolicy.cpp/.hpp` | 自适应大小策略 |
| `psYoungGen.cpp/.hpp` | 年轻代实现 |
| `psOldGen.cpp/.hpp` | 老年代实现 |
| `psPromotionManager.cpp/.hpp` | 晋升管理器 |
| `psPromotionLAB.cpp/.hpp` | 晋升本地分配缓冲 |
| `gcTaskManager.cpp/.hpp` | GC 任务管理器 |
| `gcTaskThread.cpp/.hpp` | GC 任务线程 |
| `psTasks.cpp/.hpp` | GC 任务定义 |
| `pcTasks.cpp/.hpp` | 并行压缩任务 |
| `psCompactionManager.cpp/.hpp` | 压缩管理器 |
| `parMarkBitMap.cpp/.hpp` | 并行标记位图 |
| `objectStartArray.cpp/.hpp` | 对象起始数组 |
| `cardTableExtension.cpp/.hpp` | 卡表扩展 |
| `generationSizer.cpp/.hpp` | 分代大小器 |
| `adjoiningGenerations.cpp/.hpp` | 相邻分代 |
| `adjoiningVirtualSpaces.cpp/.hpp` | 相邻虚拟空间 |
| `asPSYoungGen.cpp/.hpp` | 自适应年轻代 |
| `asPSOldGen.cpp/.hpp` | 自适应老年代 |
| `psGCAdaptivePolicyCounters.cpp/.hpp` | GC 自适应策略计数器 |
| `psGenerationCounters.cpp/.hpp` | 分代计数器 |
| `psMarkSweepDecorator.cpp/.hpp` | 标记-清除装饰器 |
| `psVirtualspace.cpp/.hpp` | 虚拟空间 |
| `vmPSOperations.cpp/.hpp` | VM 操作 |
| `vmStructs_parallelgc.hpp` | VM 结构导出 |

---

*上一篇：[03_ParNew_CMS.md](03_ParNew_CMS.md) | 下一篇：[05_G1_GC.md](05_G1_GC.md) — G1 收集器详解*