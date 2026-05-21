# GC 参数调优实战手册

> 基于 OpenJDK 8u HotSpot 源码的深度调优参考

---

## 一、GC 参数源码位置

所有 GC 参数定义在 `src/share/vm/runtime/globals.hpp` 中，使用 `product`、`diagnostic`、`develop`、`manageable` 等类型修饰符。

```cpp
// globals.hpp 中的参数定义示例
product(uintx, MaxGCPauseMillis, max_uintx,                       \
  "Adaptive size policy maximum GC pause time goal in millisecond")\
                                                                  \
product(uintx, GCTimeRatio, 99,                                   \
  "Adaptive size policy application time to GC time ratio")       \
                                                                  \
product(bool, UseG1GC, false,                                     \
  "Use the Garbage-First garbage collector")                      \
                                                                  \
product(uintx, CMSInitiatingOccupancyFraction, -1,                \
  "Percentage CMS generation occupancy to start a CMS collection")\
```

**参数类型含义**：
| 类型 | 说明 |
|------|------|
| `product` | 产品参数，可通过 `-XX:` 设置 |
| `diagnostic` | 诊断参数，需要 `-XX:+UnlockDiagnosticVMOptions` |
| `develop` | 开发参数，需要 `-XX:+UnlockExperimentalVMOptions` |
| `manageable` | 可通过 JMX 动态修改 |
| `notproduct` | 仅在非产品构建中可用 |

---

## 二、GC 选择决策树

```
选择 GC 收集器：
┌─────────────────────────────────────────────┐
│ 数据量 < 100MB？                              │
│   → Serial GC (-XX:+UseSerialGC)            │
├─────────────────────────────────────────────┤
│ 关注吞吐量？批处理？后台计算？                  │
│   → Parallel GC (-XX:+UseParallelGC)        │
├─────────────────────────────────────────────┤
│ 关注延迟？交互式？Web 服务？                    │
│   → G1 GC (-XX:+UseG1GC)                    │
│   → CMS (-XX:+UseConcMarkSweepGC) [已弃用]  │
├─────────────────────────────────────────────┤
│ 超低延迟？超大堆？                             │
│   → ZGC / Shenandoah (JDK 11+)              │
└─────────────────────────────────────────────┘
```

---

## 三、堆大小调优

### 3.1 堆大小参数

| 参数 | 默认值 | 源码位置 | 说明 |
|------|--------|---------|------|
| `-Xms` | 物理内存/64 | `globals.hpp:InitialHeapSize` | 初始堆大小 |
| `-Xmx` | 物理内存/4 | `globals.hpp:MaxHeapSize` | 最大堆大小 |
| `-Xmn` | 自适应 | `globals.hpp:NewSize` | 新生代大小 |
| `-XX:NewRatio` | 2 | `globals.hpp:NewRatio` | 老年代/新生代比例 |
| `-XX:SurvivorRatio` | 8 | `globals.hpp:SurvivorRatio` | Eden/Survivor比例 |
| `-XX:MetaspaceSize` | 21MB | `globals.hpp:MetaspaceSize` | 初始元空间大小 |
| `-XX:MaxMetaspaceSize` | 无限 | `globals.hpp:MaxMetaspaceSize` | 最大元空间大小 |

### 3.2 堆大小计算

```
堆大小自适应计算（arguments.cpp:ergonomic_initialize()）：

初始堆大小 = min(MaxRAMFraction * 物理内存, MaxHeapSize)
  默认: MaxRAMFraction = 4 → 初始堆 = 25% 物理内存

最大堆大小受以下参数影响：
  -XX:MaxRAMPercentage (默认 25%)
  -XX:InitialRAMPercentage (默认 1.5625%)
  -XX:MinRAMPercentage (默认 50%)

推荐：
  -Xms = -Xmx（避免堆动态调整的开销）
  堆大小 ≤ 物理内存的 50%（留空间给 OS 和其他进程）
```

### 3.3 新生代大小调优

```
新生代大小影响 Minor GC 频率和晋升：

新生代太小 → Minor GC 频繁，但单次时间短
新生代太大 → Minor GC 少，但停顿时间长
老年代太小 → 频繁 Full GC

经验法则：
  - 新生代 = 堆的 1/3 到 1/2
  - SurvivorRatio = 8（Eden:S0:S1 = 8:1:1）
  - 让自适应策略调整（不设 -Xmn）

特殊场景：
  - 大量短命对象：增大 Eden → -XX:SurvivorRatio=12
  - 对象存活较长：减小 Eden → -XX:SurvivorRatio=6
```

---

## 四、晋升与年龄调优

### 4.1 晋升参数

| 参数 | 默认值 | 源码位置 | 说明 |
|------|--------|---------|------|
| `-XX:MaxTenuringThreshold` | 15 | `globals.hpp` | 最大晋升年龄 |
| `-XX:InitialTenuringThreshold` | 7 | `globals.hpp` | 初始晋升年龄 |
| `-XX:TargetSurvivorRatio` | 50 | `globals.hpp` | Survivor 目标使用率(%) |
| `-XX:PretenureSizeThreshold` | 0 | `globals.hpp` | 直接晋升老年代的对象大小阈值 |
| `-XX:AlwaysTenure` | false | `globals.hpp` | 新生代对象直接晋升 |

### 4.2 晋升年龄动态计算

```
源码位置：ageTable.cpp

动态年龄计算逻辑：
  1. 每次 Minor GC 后，计算每个年龄的存活对象总大小
  2. 找到满足以下条件的最小年龄 age：
     sum(ages 0..age) > TargetSurvivorRatio% * SurvivorSize
  3. 当前晋升年龄 = min(age, MaxTenuringThreshold)

示例：
  TargetSurvivorRatio = 50
  SurvivorSize = 10MB
  
  年龄分布：
    年龄1: 4MB
    年龄2: 3MB
    年龄3: 2MB
    年龄4: 1MB
  
  累计：
    年龄1: 4MB (40% < 50%) → 不晋升
    年龄1-2: 7MB (70% > 50%) → 晋升年龄 ≥ 2 的对象
  
  实际晋升年龄 = min(2, MaxTenuringThreshold)
```

### 4.3 大对象直接晋升

```
-XX:PretenureSizeThreshold=<bytes>

设置大于此阈值的对象直接在老年代分配（跳过新生代）：
  - 适合已知大对象频繁创建的场景
  - 避免大对象在新生代复制开销
  - 阈值单位是字节

示例：
  -XX:PretenureSizeThreshold=1048576  # 1MB以上的对象直接在老年代分配
```

---

## 五、各收集器调优

### 5.1 Serial GC 调优

```bash
# 适用场景：小应用、Client模式、单核环境
java -XX:+UseSerialGC \
     -Xms256m -Xmx256m \
     -XX:NewRatio=2 \
     -XX:SurvivorRatio=8 \
     -XX:MaxTenuringThreshold=15 \
     -jar app.jar
```

### 5.2 CMS 调优

```bash
# 适用场景：延迟敏感、中等堆（4-8GB）
java -XX:+UseConcMarkSweepGC \
     -XX:+UseParNewGC \
     -Xms8g -Xmx8g \
     -XX:NewRatio=2 \
     -XX:SurvivorRatio=8 \
     -XX:ParallelGCThreads=8 \
     -XX:ConcGCThreads=2 \
     -XX:CMSInitiatingOccupancyFraction=68 \
     -XX:+UseCMSInitiatingOccupancyOnly \
     -XX:+CMSScavengeBeforeRemark \
     -XX:+CMSClassUnloadingEnabled \
     -XX:+UseCMSCompactAtFullCollection \
     -XX:CMSFullGCsBeforeCompaction=2 \
     -XX:MaxTenuringThreshold=6 \
     -XX:+ExplicitGCInvokesConcurrent \
     -XX:+PrintGCDetails \
     -Xloggc:gc.log \
     -jar app.jar

# 关键参数说明：
# CMSInitiatingOccupancyFraction=68
#   → 老年代使用68%时开始CMS
#   → 计算：100 - (CMS周期期间老年代增量% + 安全余量)
#   → 假设CMS周期期间增长20%，安全余量12% → 100-20-12=68
# CMSScavengeBeforeRemark
#   → Remark前先做一次Minor GC，减少Remark扫描量
# ExplicitGCInvokesConcurrent
#   → System.gc()触发并发收集而非Full GC
```

### 5.3 Parallel GC 调优

```bash
# 适用场景：批处理、科学计算、吞吐量优先
java -XX:+UseParallelGC -XX:+UseParallelOldGC \
     -Xms16g -Xmx16g \
     -XX:GCTimeRatio=49 \
     -XX:MaxGCPauseMillis=500 \
     -XX:+UseAdaptiveSizePolicy \
     -XX:ParallelGCThreads=8 \
     -XX:MaxTenuringThreshold=15 \
     -jar app.jar

# 关键参数说明：
# GCTimeRatio=49 → GC时间占比 ≤ 2%
# MaxGCPauseMillis=500 → 单次GC停顿目标500ms
# UseAdaptiveSizePolicy → 让JVM自动调整年轻代/老年代大小
# 不设-Xmn，让自适应策略决定年轻代大小
```

### 5.4 G1 GC 调优

```bash
# 适用场景：延迟敏感、大堆（6GB+）
java -XX:+UseG1GC \
     -Xms16g -Xmx16g \
     -XX:MaxGCPauseMillis=200 \
     -XX:G1HeapRegionSize=8m \
     -XX:ParallelGCThreads=8 \
     -XX:ConcGCThreads=2 \
     -XX:InitiatingHeapOccupancyPercent=45 \
     -XX:G1MixedGCLiveThresholdPercent=85 \
     -XX:G1MixedGCCountTarget=8 \
     -XX:G1ReservePercent=10 \
     -XX:+UseStringDeduplication \
     -XX:MaxTenuringThreshold=15 \
     -XX:+PrintGCDetails \
     -XX:+PrintGCDateStamps \
     -XX:+PrintGCTimeStamps \
     -XX:+PrintGCApplicationStoppedTime \
     -XX:+PrintHeapAtGC \
     -XX:+PrintTenuringDistribution \
     -Xloggc:gc.log \
     -jar app.jar

# 关键参数说明：
# MaxGCPauseMillis=200 → 目标每次GC不超过200ms
# G1HeapRegionSize=8m → 16GB堆约2048个Region
# InitiatingHeapOccupancyPercent=45 → 堆使用45%时开始并发标记
# G1MixedGCLiveThresholdPercent=85 → 存活率>85%的Old Region不进入混合收集
# UseStringDeduplication → 字符串去重
```

---

## 六、高级调优技巧

### 6.1 GC 日志分析

```bash
# JDK 8 推荐的GC日志参数
-XX:+PrintGCDetails              # 详细GC日志
-XX:+PrintGCDateStamps           # 日期时间戳
-XX:+PrintGCTimeStamps           # JVM启动后的时间戳
-XX:+PrintGCApplicationStoppedTime  # 应用暂停时间
-XX:+PrintHeapAtGC               # GC前后堆信息
-XX:+PrintTenuringDistribution   # 年龄分布
-XX:+PrintPromotionFailure       # 晋升失败
-XX:+PrintFLSStatistics          # 空闲列表统计（CMS）
-XX:+PrintAdaptiveSizePolicy     # 自适应策略决策
-XX:+PrintReferenceGC            # 引用处理时间
-Xloggc:gc.log                   # GC日志文件
-XX:+UseGCLogFileRotation        # 日志轮转
-XX:NumberOfGCLogFiles=10        # 日志文件数
-XX:GCLogFileSize=50M            # 单个日志文件大小
```

### 6.2 内存泄漏排查

```bash
# 堆转储参数
-XX:+HeapDumpOnOutOfMemoryError  # OOM时自动堆转储
-XX:HeapDumpPath=/path/to/dump   # 堆转储文件路径
-XX:+HeapDumpBeforeFullGC        # Full GC前堆转储（谨慎使用）
-XX:+HeapDumpAfterFullGC         # Full GC后堆转储（谨慎使用）
```

### 6.3 并发线程数调优

```
ParallelGCThreads 计算公式（globals.hpp）：
  CPU ≤ 8:  ParallelGCThreads = CPU核数
  CPU > 8:  ParallelGCThreads = 8 + (CPU-8) * 5/8

ConcGCThreads 计算公式：
  ConcGCThreads = max(1, ParallelGCThreads / 4)

建议：
  - CPU密集型应用：减少GC线程数，留更多CPU给应用
  - GC频繁的应用：增加GC线程数
  - 容器环境：明确设置线程数，避免自动检测过多
```

### 6.4 TLAB 调优

```
Thread-Local Allocation Buffer (TLAB)：
  每个线程在Eden中分配的私有缓冲区
  快速分配无需同步

参数：
  -XX:TLABSize=<bytes>           # TLAB大小（0=自适应）
  -XX:MinTLABSize=2k             # 最小TLAB大小
  -XX:TLABAllocationWeight=35    # TLAB分配权重
  -XX:TLABWasteTargetPercent=1   # Eden中可浪费的百分比
  -XX:ResizeTLAB=true            # 动态调整TLAB大小

建议：
  通常不需要手动调整TLAB
  如果GC日志中TLAB refill频率过高，可增大TLABSize
```

### 6.5 字符串去重（G1 专用）

```bash
# G1 字符串去重
-XX:+UseStringDeduplication

# 效果：
# - 减少堆中 String 对象的 char[] 重复
# - 典型节省 10-25% 堆内存
# - 适合大量重复字符串的应用（Web应用、日志处理等）

# 监控：
-XX:+PrintStringDeduplicationStatistics
```

---

## 七、常见问题与解决方案

### 7.1 CMS 并发模式失败

```
问题：CMS 并发标记未完成，老年代已满
表现：退化为 Serial Old Full GC，停顿时间极长

解决方案：
1. 降低 CMSInitiatingOccupancyFraction
   -XX:CMSInitiatingOccupancyFraction=68 -XX:+UseCMSInitiatingOccupancyOnly
2. 增大老年代
   -XX:NewRatio=3（减少新生代，增加老年代）
3. 减少 CMS 期间的对象晋升
   -XX:MaxTenuringThreshold=6（更早晋升，减少Survivor压力）
4. 如果碎片严重，考虑切换到 G1
```

### 7.2 晋升失败（Promotion Failed）

```
问题：Minor GC时存活对象无法晋升到老年代（老年代空间不足或碎片化）
表现：Full GC

CMS解决方案：
1. 增大老年代
2. -XX:CMSInitiatingOccupancyFraction=68（更早触发CMS）
3. -XX:+UseCMSCompactAtFullCollection（Full GC时压缩）
4. -XX:CMSFullGCsBeforeCompaction=2（每2次Full GC压缩一次）

G1解决方案：
1. 切换到G1（天然无碎片）
2. 增大堆
```

### 7.3 元空间溢出

```
问题：Metaspace 不足
表现：java.lang.OutOfMemoryError: Metaspace

解决方案：
1. 增大元空间
   -XX:MetaspaceSize=256m -XX:MaxMetaspaceSize=512m
2. CMS 下启用类卸载
   -XX:+CMSClassUnloadingEnabled
3. 检查是否有大量动态代理/反射生成类
```

### 7.4 G1 混合收集停顿过长

```
问题：Mixed GC 停顿时间超过目标
表现：GC日志中Mixed GC停顿时间过长

解决方案：
1. 减少每次混合收集的Old Region数量
   -XX:G1OldCSetRegionThresholdPercent=5（从10降到5）
2. 增加混合收集次数
   -XX:G1MixedGCCountTarget=16（从8增到16）
3. 提高混合收集存活率阈值
   -XX:G1MixedGCLiveThresholdPercent=90（从85提到90）
4. 适当降低IHOP
   -XX:InitiatingHeapOccupancyPercent=40
```

### 7.5 大对象分配失败

```
问题：Humongous 对象分配失败
表现：Full GC 或 OOM

解决方案：
1. 增大Region大小
   -XX:G1HeapRegionSize=16m（从8m增到16m）
2. 减少大对象创建
3. 增大堆

CMS/Parallel解决方案：
1. 增大老年代
2. -XX:PretenureSizeThreshold=1m（大对象直接在老年代分配）
```

---

## 八、GC 参数完整参考表

### 8.1 通用参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `-Xms` | 物理内存/64 | 初始堆大小 |
| `-Xmx` | 物理内存/4 | 最大堆大小 |
| `-Xmn` | 自适应 | 新生代大小 |
| `-XX:NewRatio` | 2 | 老年代/新生代比 |
| `-XX:SurvivorRatio` | 8 | Eden/Survivor比 |
| `-XX:MaxTenuringThreshold` | 15 | 最大晋升年龄 |
| `-XX:InitialTenuringThreshold` | 7 | 初始晋升年龄 |
| `-XX:TargetSurvivorRatio` | 50 | Survivor目标使用率 |
| `-XX:MaxHeapFreeRatio` | 70 | 最大空闲堆比(%) |
| `-XX:MinHeapFreeRatio` | 40 | 最小空闲堆比(%) |
| `-XX:MetaspaceSize` | 21MB | 初始元空间 |
| `-XX:MaxMetaspaceSize` | 无限 | 最大元空间 |

### 8.2 CMS 参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `-XX:+UseConcMarkSweepGC` | false | 启用CMS |
| `-XX:CMSInitiatingOccupancyFraction` | -1(自适应) | CMS触发阈值(%) |
| `-XX:+UseCMSInitiatingOccupancyOnly` | false | 只用固定阈值 |
| `-XX:+CMSPrecleaningEnabled` | true | 启用预清理 |
| `-XX:+CMSScavengeBeforeRemark` | false | Remark前Minor GC |
| `-XX:+CMSClassUnloadingEnabled` | false | CMS类卸载 |
| `-XX:+UseCMSCompactAtFullCollection` | true | Full GC压缩 |
| `-XX:CMSFullGCsBeforeCompaction` | 0 | N次Full GC后压缩 |
| `-XX:ConcGCThreads` | ParallelGCThreads/4 | 并发线程数 |
| `-XX:+ExplicitGCInvokesConcurrent` | false | System.gc()触发并发 |

### 8.3 G1 参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `-XX:+UseG1GC` | false | 启用G1 |
| `-XX:MaxGCPauseMillis` | 200 | 最大停顿目标(ms) |
| `-XX:G1HeapRegionSize` | 自动 | Region大小 |
| `-XX:InitiatingHeapOccupancyPercent` | 45 | 并发标记触发阈值(%) |
| `-XX:G1NewSizePercent` | 5 | 年轻代最小占比(%) |
| `-XX:G1MaxNewSizePercent` | 60 | 年轻代最大占比(%) |
| `-XX:G1MixedGCLiveThresholdPercent` | 85 | 混合收集存活率阈值(%) |
| `-XX:G1MixedGCCountTarget` | 8 | 混合收集次数目标 |
| `-XX:G1ReservePercent` | 10 | 保留空间(%) |
| `-XX:+UseStringDeduplication` | false | 字符串去重 |

### 8.4 Parallel 参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `-XX:+UseParallelGC` | false | 启用Parallel |
| `-XX:+UseParallelOldGC` | 自动 | 启用Parallel Old |
| `-XX:GCTimeRatio` | 99 | 吞吐量目标 |
| `-XX:MaxGCPauseMillis` | 无限 | 最大停顿目标(ms) |
| `-XX:+UseAdaptiveSizePolicy` | true | 自适应大小策略 |
| `-XX:ParallelGCThreads` | CPU核数 | 并行线程数 |

---

*上一篇：[05_G1_GC.md](05_G1_GC.md) | 返回：[01_GC_Overview.md](01_GC_Overview.md)*