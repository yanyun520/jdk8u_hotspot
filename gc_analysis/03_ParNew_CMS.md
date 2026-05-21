# ParNew + CMS 收集器 — 低延迟并发收集

> 源码路径：`src/share/vm/gc_implementation/parNew/`, `src/share/vm/gc_implementation/concurrentMarkSweep/`

---

## 一、CMS 收集器概述

CMS（Concurrent Mark Sweep）是 HotSpot 第一个**并发**垃圾收集器，目标是**最小化 GC 停顿时间**。

**通俗比喻**：CMS 就像一个清洁团队，在住户（应用线程）正常活动的同时，悄悄标记哪些房间已经空了（并发标记），然后在短暂封楼（STW）时快速确认和打扫。

### 1.1 设计目标

- **最短停顿时间**：老年代收集尽量与应用并发执行
- **适合响应敏感应用**：Web 服务器、交互式系统
- **代价**：吞吐量下降、内存碎片

### 1.2 选用参数

```bash
-XX:+UseConcMarkSweepGC    # 启用 CMS（年轻代自动使用 ParNew）
-XX:+UseParNewGC           # 年轻代使用 ParNew（CMS 时自动启用）
```

---

## 二、ParNew 收集器 — 年轻代并行收集

**核心源码**：`src/share/vm/gc_implementation/parNew/parNewGeneration.cpp/.hpp`

### 2.1 ParNew vs DefNew

ParNew 是 DefNewGeneration 的多线程并行版本，算法完全相同（复制算法），只是使用多个 GC 线程并行执行。

```
DefNew（串行）:  🧹 → 🧹 → 🧹 → 🧹  （单线程）
ParNew（并行）:  🧹🧹🧹🧹 → 🧹🧹🧹🧹  （多线程并行）
```

### 2.2 类继承

```
DefNewGeneration
  └── ParNewGeneration
        - ParScanThreadState: 每个线程的扫描状态
        - ParNewGenTask: 并行扫描任务
        - ParNewRefProcTaskExecutor: 并行引用处理
```

### 2.3 核心源码

**`ParNewGeneration::collect()`**：

```cpp
// parNewGeneration.cpp 简化伪代码
void ParNewGeneration::collect(bool full, ...) {
  // 1. 确定活跃 GC 线程数
  // 2. 初始化 ParScanThreadState（每个线程独立的状态）
  // 3. 并行扫描根集 → Eden/From 存活对象
  // 4. 并行复制存活对象到 To Survivor 或老年代
  // 5. 处理晋升失败
  // 6. 更新年龄表
}
```

### 2.4 关键参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `-XX:ParallelGCThreads` | CPU核数 | GC 并行线程数 |
| `-XX: SurvivorRatio` | 8 | Eden/Survivor 比例 |

---

## 三、CMS 收集器 — 老年代并发标记-清除

**核心源码**：`src/share/vm/gc_implementation/concurrentMarkSweep/concurrentMarkSweepGeneration.cpp/.hpp`

### 3.1 CMS 六个阶段详解

CMS 的一个完整收集周期分为**六个阶段**，其中四个与应用线程并发执行：

```
时间线：
应用线程: ████████████████████████████████████████████████████████
CMS线程:          ┌───┐   ┌──────────┐┌────┐┌───┐  ┌──────────┐┌─────┐
                   │STW│   │并发标记   ││STW ││STW│  │并发清除  ││并发 │
                   │① │   │②       ││④  ││⑤ │  │⑥       ││重置│
                   └───┘   └──────────┘└────┘└───┘  └──────────┘└─────┘

阶段说明：
① 初始标记 (Initial Mark)     — STW，极短
② 并发标记 (Concurrent Mark)  — 并发，最耗时
③ 预清理 (Preclean)            — 并发（可选）
④ 可中止预清理 (Abortable Preclean) — 并发
⑤ 最终标记 (Final Remark)      — STW，较短
⑥ 并发清除 (Concurrent Sweep)  — 并发
⑦ 并发重置 (Concurrent Reset)  — 并发
```

### 3.2 各阶段详细分析

#### 阶段①：初始标记（Initial Mark）

**源码**：`CMSCollector::checkpointRootsInitialWork()`

```
目的：标记从 GC Roots 直接可达的对象
停顿：STW（Stop-The-World），极短
特点：只扫描根集，不递归

 ┌───────────────────────────┐
 │  GC Roots                  │
 │  ┌──┐ ┌──┐ ┌──┐ ┌──┐    │
 │  │R1│ │R2│ │R3│ │R4│    │ ← 只标记这些
 │  └─┬┘ └─┬┘ └─┬┘ └─┬┘    │
 │    │    │    │    │      │
 │    ▼    ▼    ▼    ▼      │
 │   🟢   🟢   🟢   🟢     │ ← 直接可达对象
 └───────────────────────────┘
```

#### 阶段②：并发标记（Concurrent Mark）

**源码**：`CMSCollector::markFromRootsWork()`

```
目的：从初始标记的对象出发，遍历整个对象图
停顿：无（与应用线程并发执行）
特点：最耗时的阶段，并发执行

 ┌───────────────────────────────────────────┐
 │  从 🟢 出发，递归遍历所有引用              │
 │  🟢→🟢→🟢→🟢→🟢    完整可达图           │
 │     ↘🟢→🟢              标记完毕           │
 │                                         │
 │  同时应用线程可能修改引用：                │
 │  写屏障记录脏卡（Card Table / ModUnion） │
 └───────────────────────────────────────────┘

并发标记使用 mark bitmap（标记位图）：
- 每个 bit 对应堆中一个最小分配单位
- 标记 = 设置对应 bit 为 1
```

#### 阶段③：预清理（Preclean）

**源码**：`CMSCollector::preclean()`

```
目的：减少最终标记阶段的工作量
停顿：无（并发执行）
特点：扫描脏卡和新生代，提前处理部分引用变更

预清理内容：
1. 处理 Reference 对象（Soft/Weak/Phantom）
2. 扫描新生代存活对象（减少 remark 扫描量）
3. 扫描脏卡（Card Table 中标记为脏的卡片）
```

#### 阶段④：可中止预清理（Abortable Preclean）

**源码**：`CMSCollector::abortable_preclean()`

```
目的：在最终标记前尽量多做工作，减少 STW 时间
停顿：无（并发执行）
退出条件：
1. 达到最大迭代次数
2. 足够多的工作已完成
3. 新生代使用率超过阈值
4. 超时
```

#### 阶段⑤：最终标记（Final Remark / Remark）

**源码**：`CMSCollector::do_remark_parallel()` / `do_remark_non_parallel()`

```
目的：重新扫描并发标记期间发生变更的引用，完成最终标记
停顿：STW（通常较短，因为 preclean 已处理大部分工作）
特点：最关键的 STW 阶段

扫描内容：
1. 重新扫描根集
2. 处理脏卡（Mod Union Table + Card Table）
3. 处理引用队列
4. 扫描新生代（eden + survivor）

 ┌───────────────────────────────────────────┐
 │  并发标记期间被修改的引用（脏卡）：         │
 │  ┌──┐ ┌──┐ ┌──┐                           │
 │  │脏│ │脏│ │脏│  ← 只扫描这些区域的引用    │
 │  └──┘ └──┘ └──┘                           │
 └───────────────────────────────────────────┘
```

#### 阶段⑥：并发清除（Concurrent Sweep）

**源码**：`CMSCollector::sweep()`

```
目的：回收未被标记的对象占用的空间
停顿：无（并发执行）
特点：使用空闲列表（Free List）管理回收的空间

 ┌───────────────────────────────────────────┐
 │  标记位图：                                │
 │  1 0 0 1 0 1 0 0 1 0 1 0 0 1 0 1 0 0    │
 │  ↓           ↓     ↓           ↓         │
 │  🟢 留存    🟢    🟢         🟢          │
 │     回收 ↗      回收 ↗        回收 ↗     │
 │                                           │
 │  空闲列表（Free List）：                    │
 │  [块1: 64B] → [块2: 128B] → [块3: 32B]   │
 └───────────────────────────────────────────┘

注意：CMS 使用 Free List 而非压缩！
→ 产生内存碎片
```

### 3.3 CMS 类层次

```
CMSCollector (concurrentMarkSweepGeneration.hpp)
  ├── ConcurrentMarkSweepGeneration : CardGeneration
  │     └── ASConcurrentMarkSweepGeneration (自适应大小)
  ├── CompactibleFreeListSpace : CompactibleSpace
  │     ├── 空闲列表管理（分配/合并/分裂）
  │     └── 标记位图（markBitMap）
  ├── ConcurrentMarkSweepThread : ConcurrentGCThread
  │     └── 后台 CMS 线程
  └── CMS Oop Closures
        ├── MarkRefsIntoClosure          — 初始标记
        ├── PushOrMarkClosure            — 并发标记
        ├── MarkRefsIntoAndScanClosure   — 最终标记
        └── CMSKeepAliveClosure          — 引用处理

CMSCollector 状态机：
  Idling → InitialMarking → Marking → Precleaning
  → AbortablePreclean → FinalMarking → Sweeping → Resizing → Resetting → Idling
```

### 3.4 CompactibleFreeListSpace — 空闲列表

**源码**：`compactibleFreeListSpace.cpp/.hpp`

CMS 老年代使用**空闲列表**而非 Bump-the-Pointer 分配：

```
空闲列表结构：
┌─────────────────────────────────────────────────┐
│ 已分配: ██████  空闲: ░░░░  已分配: ████████  │
│ ██████░░░░████████░░████░░░░████████░░░░░░████  │
│        ↑              ↑        ↑                │
│     FreeNode         FreeNode  FreeNode         │
│     (64B)            (32B)    (128B)            │
│                                                  │
│ 空闲列表：FreeNode(64B) → FreeNode(32B) → ...   │
└─────────────────────────────────────────────────┘

问题：长期使用后产生大量碎片（碎片化）
解决：定期 Full GC 压缩（UseCMSCompactAtFullCollection）
```

---

## 四、CMS 的并发标记 — 写屏障与 ModUnionTable

### 4.1 卡表（Card Table）

**源码**：`src/share/vm/memory/cardTableRS.cpp`

```
堆内存被划分为 512 字节的卡片：
┌────┬────┬────┬────┬────┬────┬────┬────┐
│ C0 │ C1 │ C2 │ C3 │ C4 │ C5 │ C6 │ C7 │  ← 堆内存卡片
└────┴────┴────┴────┴────┴────┴────┴────┘
  ↓    ↓    ↓    ↓    ↓    ↓    ↓    ↓
┌────┬────┬────┬────┬────┬────┬────┬────┐
│ 0  │ 0  │ 1  │ 0  │ 0  │ 1  │ 0  │ 0  │  ← 卡表（1=脏）
└────┴────┴────┴────┴────┴────┴────┴────┘

写屏障：每次引用写入时，标记对应卡片为脏
  oop.field = newRef;  // 应用代码
  → card_table[oop >> 9] = DIRTY;  // 写屏障自动插入
```

### 4.2 ModUnionTable

**源码**：`concurrentMarkSweepGeneration.hpp` 中的 `_modUnionTable`

```
ModUnionTable 是卡表在 CMS 中的增强版本：
- 在初始标记时记录所有脏卡
- 在并发标记期间，写屏障将变更标记到 ModUnionTable
- Remark 时只需扫描 ModUnionTable 中标记的卡片

┌─────────────────────────────────────────────────┐
│ 初始标记时快照：                                  │
│ [1][0][1][0][0][1][0][0]                        │
│                                                  │
│ 并发标记期间变更：                                │
│ [0][0][1][0][1][0][0][0]  ← 新的脏卡             │
│                                                  │
│ Remark 扫描 = 初始快照 ∪ 并发期间变更              │
│ [1][0][1][0][1][1][0][0]                        │
└─────────────────────────────────────────────────┘
```

---

## 五、CMS 的 Full GC 与压缩

### 5.1 触发条件

**源码**：`CMSCollector::acquire_control_and_collect()`

```
Full GC 触发条件：
1. 老年代使用率超过阈值（CMSInitiatingOccupancyFraction）
2. 显式调用 System.gc() 且未设置 ExplicitGCInvokesConcurrent
3. 晋升失败（Promotion Failed）
4. 并发模式失败（Concurrent Mode Failure）
5. 元空间不足
```

### 5.2 并发模式失败（Concurrent Mode Failure）

```
场景：CMS 并发标记尚未完成，老年代已满

时间线：
应用: ████████████████████████ ████ 堆满! ← 分配失败
CMS:    ┌────并发标记中────┐    ↑ 还没清完!
         ↑ 还没开始清除

结果：退化为 Serial Old（Full GC，串行压缩）→ 长停顿!
```

### 5.3 压缩

**源码**：`CMSCollector::do_compaction_work()` → `GenMarkSweep::invoke_at_safepoint()`

```
CMS 默认不压缩，只使用空闲列表。
当需要压缩时（Full GC）：
  使用 GenMarkSweep（标记-清除-压缩），等同于 Serial Old
```

---

## 六、CMS 核心参数调优

### 6.1 基础参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `-XX:+UseConcMarkSweepGC` | false | 启用 CMS |
| `-XX:ParallelGCThreads` | CPU核数 | 并行 GC 线程数 |
| `-XX:ConcGCThreads` | ParallelGCThreads/4 | 并发标记线程数 |

### 6.2 触发阈值

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `-XX:CMSInitiatingOccupancyFraction` | -1(自适应) | 老年代使用率触发阈值(%) |
| `-XX:+UseCMSInitiatingOccupancyOnly` | false | 只使用固定阈值，不使用启发式 |
| `-XX:CMSBootstrapOccupancy` | 50 | CMS 启动前的初始阈值(%) |

**调优建议**：
```bash
# 设置 CMS 在老年代 70% 满时触发，避免并发模式失败
-XX:CMSInitiatingOccupancyFraction=70 -XX:+UseCMSInitiatingOccupancyOnly

# 计算公式：
# CMSInitiatingOccupancyFraction = 100 - (CMS 周期期间老年代增长量% + 安全余量)
# 例如：CMS 周期期间老年代增长 20%，安全余量 10%
# → 阈值 = 100 - 20 - 10 = 70
```

### 6.3 预清理与 Remark

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `-XX:+CMSPrecleaningEnabled` | true | 启用预清理 |
| `-XX:CMSScavengeBeforeRemark` | false | Remark 前 Minor GC |
| `-XX:CMSMaxAbortablePrecleanTime` | 5000 | 最大可中止预清理时间(ms) |

**调优建议**：
```bash
# Remark 前 Minor GC 可以减少 Remark 扫描新生代的工作量
-XX:+CMSScavengeBeforeRemark
```

### 6.4 压缩与碎片

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `-XX:+UseCMSCompactAtFullCollection` | true | Full GC 时压缩 |
| `-XX:CMSFullGCsBeforeCompaction` | 0 | 多少次 Full GC 后压缩（0=每次） |
| `-XX:CMSFullGCsBeforeCompaction` | 0 | 同上 |

### 6.5 类卸载

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `-XX:+CMSClassUnloadingEnabled` | false | CMS 阶段卸载类 |
| `-XX:+CMSPermGenSweepingEnabled` | false | 扫描永久代 |

**调优建议**：
```bash
# JDK8 推荐开启类卸载（减少永久代/元空间压力）
-XX:+CMSClassUnloadingEnabled
```

### 6.6 完整 CMS 调优示例

```bash
# 典型 CMS 配置（4核 8GB 堆）
java -XX:+UseConcMarkSweepGC \
     -XX:ParallelGCThreads=4 \
     -XX:ConcGCThreads=1 \
     -Xms8g -Xmx8g \
     -XX:NewRatio=2 \
     -XX:SurvivorRatio=8 \
     -XX:CMSInitiatingOccupancyFraction=70 \
     -XX:+UseCMSInitiatingOccupancyOnly \
     -XX:+CMSScavengeBeforeRemark \
     -XX:+CMSClassUnloadingEnabled \
     -XX:+UseCMSCompactAtFullCollection \
     -XX:CMSFullGCsBeforeCompaction=2 \
     -XX:MaxTenuringThreshold=6 \
     -jar app.jar
```

---

## 七、CMS 源码文件清单

| 文件 | 说明 |
|------|------|
| `concurrentMarkSweepGeneration.cpp/.hpp` | CMS 核心类，收集器状态机 |
| `concurrentMarkSweepThread.cpp/.hpp` | CMS 后台线程 |
| `compactibleFreeListSpace.cpp/.hpp` | 空闲列表空间管理 |
| `cmsCollectorPolicy.cpp/.hpp` | CMS 收集策略 |
| `cmsAdaptiveSizePolicy.cpp/.hpp` | CMS 自适应大小策略 |
| `adaptiveFreeList.cpp/.hpp` | 自适应空闲列表 |
| `freeChunk.cpp/.hpp` | 空闲块管理 |
| `promotionInfo.cpp/.hpp` | 晋升信息 |
| `cmsOopClosures.hpp/.inline.hpp` | CMS Oop 闭包（标记/扫描） |
| `cmsLockVerifier.cpp/.hpp` | CMS 锁验证 |
| `cmsGCAdaptivePolicyCounters.cpp/.hpp` | GC 自适应策略计数器 |
| `vmCMSOperations.cpp/.hpp` | CMS VM 操作 |
| `vmStructs_cms.hpp` | CMS VM 结构导出 |

---

*上一篇：[02_Serial_GC.md](02_Serial_GC.md) | 下一篇：[04_Parallel_GC.md](04_Parallel_GC.md) — Parallel Scavenge + Parallel Old 收集器详解*