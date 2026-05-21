# Serial 收集器 — 最简单的 GC 实现

> 源码路径：`src/share/vm/memory/defNewGeneration.cpp`, `src/share/vm/memory/tenuredGeneration.cpp`, `src/share/vm/memory/markSweep.cpp`

---

## 一、Serial 收集器概述

Serial 收集器是 HotSpot 最基础的垃圾收集器，使用**单线程**完成所有 GC 工作。它是理解所有 GC 算法的起点。

**通俗比喻**：Serial 收集器就像一个清洁工，独自打扫整栋公寓楼。简单、可靠，但速度慢。

### 1.1 适用场景

- Client 模式默认收集器
- 小堆内存应用（< 100MB）
- 单核 CPU 环境
- 需要最简单、最可预测的 GC 行为

### 1.2 选用参数

```bash
-XX:+UseSerialGC    # 启用 Serial 收集器
```

---

## 二、新生代收集：DefNewGeneration

**核心源码**：`src/share/vm/memory/defNewGeneration.cpp/.hpp`

### 2.1 内存布局

```
新生代 (DefNewGeneration)
┌─────────────────────────────────────────────────────────┐
│                    Eden Space                            │
│  ┌───────────────────────────────────────────────────┐ │
│  │  新对象在这里分配                                    │ │
│  └───────────────────────────────────────────────────┘ │
│                                                         │
│  From Survivor          To Survivor                     │
│  ┌──────────────────┐  ┌──────────────────┐           │
│  │  存活对象暂存     │  │  存活对象暂存     │           │
│  └──────────────────┘  └──────────────────┘           │
└─────────────────────────────────────────────────────────┘

Eden : Survivor = 由 -XX:SurvivorRatio 控制，默认 8:1
即 Eden = 8 份，每个 Survivor = 1 份
```

### 2.2 Minor GC 流程（复制算法）

**源码入口**：`DefNewGeneration::collect()` → `DefNewGeneration::compute_new_size()`

```
Minor GC 流程（复制算法）：

步骤1: 标记 — 从 GC Roots 出发，标记所有可达对象
        ┌─────────────────────────────────────────────────┐
        │ Eden + From 中:                                 │
        │   绿色=存活  红色=死亡                           │
        │   🟢🔴🔴🟢🔴🟢🔴🔴🟢🔴                        │
        └─────────────────────────────────────────────────┘

步骤2: 复制 — 将存活对象复制到 To Survivor
        ┌─────────────────────────────────────────────────┐
        │ To Survivor:                                    │
        │   🟢🟢🟢🟢🟢  ← 存活对象紧凑排列               │
        └─────────────────────────────────────────────────┘

步骤3: 清空 — 清空 Eden + From
        ┌─────────────────────────────────────────────────┐
        │ Eden + From: 全部清空                           │
        └─────────────────────────────────────────────────┘

步骤4: 交换 — From 和 To 角色互换
        下次 GC 时，新的 To 就是刚清空的区域
```

### 2.3 对象晋升规则

**源码位置**：`DefNewGeneration::copy_to_survivor_space()`

```
对象年龄判断：
- 年龄 < MaxTenuringThreshold → 复制到 To Survivor
- 年龄 >= MaxTenuringThreshold → 晋升到老年代
- To Survivor 空间不足 → 直接晋升到老年代

年龄递增：每次 Minor GC 存活 → 年龄 +1
默认 MaxTenuringThreshold = 15

动态年龄计算：
- TargetSurvivorRatio = 50（期望 Survivor 使用率 50%）
- 如果同年龄对象总大小 > Survivor * TargetSurvivorRatio%
  → 提前晋升（年龄阈值自动降低）
```

### 2.4 核心源码分析

**`DefNewGeneration::collect()`** 核心逻辑：

```cpp
// defNewGeneration.cpp 简化伪代码
void DefNewGeneration::collect(bool   full,
                                bool   clear_all_soft_refs,
                                size_t size,
                                bool   is_tlab) {
  // 1. 检查是否需要 GC
  // 2. 交换 from/to survivor 空间
  // 3. 根据年龄复制存活对象到 to 或老年代
  //    - 使用 FastScanClosure 扫描根
  //    - 使用 DefNewScanClosure 扫描引用
  // 4. 处理晋升失败（promotion failure）
  // 5. 更新年龄表和大小信息
}
```

---

## 三、老年代收集：MarkSweep-Compact

**核心源码**：`src/share/vm/memory/markSweep.cpp/.hpp`，`src/share/vm/memory/tenuredGeneration.cpp/.hpp`

### 3.1 标记-清除-压缩算法

```
Full GC 流程（Mark-Sweep-Compact）：

步骤1: 标记（Mark）
  从 GC Roots 出发，标记所有可达对象
  ┌─────────────────────────────────────────────┐
  │ 🟢🔴🔴🟢🔴🟢🔴🔴🟢🔴🟢🔴🔴🟢🔴🟢🔴🔴 │
  │  绿色=存活  红色=死亡                        │
  └─────────────────────────────────────────────┘

步骤2: 清除（Sweep）
  回收死亡对象占用的空间（产生碎片）
  ┌─────────────────────────────────────────────┐
  │ 🟢  🟢  🟢  🟢  🟢  🟢  🟢  │  ← 碎片化  │
  └─────────────────────────────────────────────┘

步骤3: 压缩（Compact）
  将存活对象移动到一端，消除碎片
  ┌─────────────────────────────────────────────┐
  │ 🟢🟢🟢🟢🟢🟢🟢│          空闲            │
  └─────────────────────────────────────────────┘
```

### 3.2 核心源码分析

**`MarkSweep` 类**（`markSweep.hpp`）：

```cpp
// 标记阶段
class MarkSweep : AllStatic {
  static Stack<oop>          _marking_stack;    // 标记栈
  static Stack<ObjArrayTask>  _objarray_stack;   // 对象数组任务栈
  static Stack<Klass*>        _revisit_klass_stack; // 需要重访的类
  
  // 标记：从 roots 出发，递归标记可达对象
  static void mark_sweep_phase1(...);
  
  // 计算新地址：为每个存活对象计算压缩后的目标地址
  static void mark_sweep_phase2();
  
  // 调整指针：更新所有引用指向新地址
  static void mark_sweep_phase3();
  
  // 移动对象：将对象复制到新地址
  static void mark_sweep_phase4();
};
```

**`TenuredGeneration::collect()`** 入口：

```cpp
// tenuredGeneration.cpp 简化伪代码
void TenuredGeneration::collect(...) {
  // 使用 MarkSweep::invoke_at_safepoint() 执行
  // 四阶段：标记 → 计算新地址 → 调整指针 → 移动
}
```

---

## 四、Serial 收集器的优缺点

| 优点 | 缺点 |
|------|------|
| 实现简单，代码量少 | 单线程，GC 停顿时间长 |
| 内存开销小 | 不适合大堆 |
| 适合 Client 模式 | 不适合延迟敏感应用 |
| 调试方便 | 无法利用多核 |

---

## 五、核心参数调优

### 5.1 堆大小

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `-Xms` | 物理内存/64 | 初始堆大小 |
| `-Xmx` | 物理内存/4 | 最大堆大小 |
| `-Xmn` | - | 新生代大小 |
| `-XX:NewRatio` | 2 | 老年代/新生代比例（2 表示老年代是新生代的2倍） |
| `-XX:SurvivorRatio` | 8 | Eden/Survivor 比例（8 表示 Eden 是一个 Survivor 的8倍） |

### 5.2 晋升与年龄

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `-XX:MaxTenuringThreshold` | 15 | 对象晋升老年代的最大年龄 |
| `-XX:InitialTenuringThreshold` | 7 | 初始晋升年龄阈值 |
| `-XX:TargetSurvivorRatio` | 50 | Survivor 区目标使用率(%) |

### 5.3 调优建议

```bash
# 小应用（Client 模式）
java -XX:+UseSerialGC -Xms64m -Xmx256m -Xmn64m -jar app.jar

# 调优原则：
# 1. -Xms = -Xmx 避免堆动态调整的开销
# 2. 新生代大小约为堆的 1/3 到 1/2
# 3. SurvivorRatio 根据对象存活时间调整
#    - 大量短命对象：增大 Eden（SurvivorRatio=12）
#    - 对象存活稍长：减小 Eden（SurvivorRatio=6）
```

---

## 六、源码文件清单

| 文件 | 路径 | 说明 |
|------|------|------|
| `defNewGeneration.cpp/.hpp` | `memory/` | 新生代实现（Eden + 2 Survivor） |
| `tenuredGeneration.cpp/.hpp` | `memory/` | 老年代实现 |
| `markSweep.cpp/.hpp` | `gc_implementation/shared/` | 标记-清除-压缩算法 |
| `generation.cpp/.hpp` | `memory/` | 分代基类 |
| `genCollectedHeap.cpp/.hpp` | `memory/` | 分代收集堆 |
| `space.cpp/.hpp` | `memory/` | 内存空间抽象 |
| `cardTableRS.cpp/.hpp` | `memory/` | 卡表记忆集 |
| `referenceProcessor.cpp/.hpp` | `memory/` | 引用处理 |

---

*上一篇：[01_GC_Overview.md](01_GC_Overview.md) | 下一篇：[03_ParNew_CMS.md](03_ParNew_CMS.md) — ParNew + CMS 收集器详解*