# HotSpot JVM (JDK 8u) 项目工程结构详细分析

> 本文档对 `jdk8u_hotspot` 项目（OpenJDK 8 HotSpot 虚拟机源码）的目录与文件进行逐层、逐文件的详细说明。

---

## 📁 顶层目录概览

```
jdk8u_hotspot/
├── ASSEMBLY_EXCEPTION    # OpenJDK 汇编例外声明
├── LICENSE               # GPLv2 许可证
├── README                # 项目简介与构建指引
├── THIRD_PARTY_README    # 第三方库许可声明
├── agent/                # Serviceability Agent（SA）调试工具
├── make/                 # 构建系统（跨平台 Makefile）
├── src/                  # HotSpot VM 核心源码
└── test/                 # 测试套件
```

---

## 📄 顶层文件详解

| 文件 | 说明 |
|------|------|
| `ASSEMBLY_EXCEPTION` | OpenJDK 许可例外条款，说明在 GPLv2 之下对特定模块授予链接例外 |
| `LICENSE` | GNU General Public License version 2 全文，是本代码库的主许可证 |
| `README` | 简短说明文件，指出此为 HotSpot Mercurial 仓库顶层，构建命令为 `cd make && gnumake` |
| `THIRD_PARTY_README` | 第三方组件的许可声明，涵盖 ASM、BSDiff、Cryptix AES、CUP、DejaVu 字体等 |

---

## 📁 agent/ — Serviceability Agent 调试工具

Serviceability Agent (SA) 是 HotSpot 的进程外调试与分析框架，支持本地进程附加、核心转储分析和远程调试。

```
agent/
├── doc/        # SA 文档
├── make/       # SA 构建与运行脚本
├── src/        # SA 源码（Java + Native）
└── test/       # SA 测试
```

### agent/doc/

SA 的用户文档与使用指南。

| 文件 | 说明 |
|------|------|
| `index.html` | SA 总览文档，涵盖 HSDB GUI、CLHSDB 命令行、JSDB、远程调试服务器等 |
| `hsdb.html` | HSDB（HotSpot Debugger）图形界面使用文档 |
| `clhsdb.html` | CLHSDB（Command-Line HotSpot Debugger）命令行调试器文档 |
| `jsdb.html` | JSDB（JavaScript Debugger）基于 JavaScript 的 SA 交互界面文档 |
| `transported_core.html` | 远程/核心转储调试的详细说明 |
| `cireplay.html` | C1 编译器回放（CI Replay）调试相关文档 |
| `ReadMe-JavaScript.text` | JavaScript 支持说明 |

### agent/make/

SA 的构建系统与运行时封装脚本。

| 文件 | 说明 |
|------|------|
| `Makefile` | SA 的 GNU Make 构建入口，编译 Java 源码、生成 `sa.jar`、运行 `rmic` 生成 RMI 桩 |
| `build.xml` | Ant 构建文件，编译源码到 `../build/classes`，打包 `sa.jar` |
| `build-filelist` | 源文件列表（自动生成），供 Makefile 使用 |
| `build-pkglist` | 包列表（自动生成），供 Makefile 使用 |
| `mkinstall` | 安装辅助脚本 |
| `README.txt` | 说明此目录包含 SA Java 源码，构建命令为 `gnumake all` |
| `grantAll.policy` | Java 安全策略文件，用于 RMI 远程调试授权 |

**Unix 运行脚本（`.sh`）：**

| 脚本 | 说明 |
|------|------|
| `hsdb.sh` / `hsdbproc.sh` / `hsdbproc64.sh` | 启动 HSDB 图形调试器（32/64位） |
| `clhsdbproc.sh` / `clhsdbproc64.sh` | 启动 CLHSDB 命令行调试器（32/64位） |
| `jsdbproc.sh` / `jsdbproc64.sh` | 启动 JSDB JavaScript 调试器（32/64位） |
| `jstackproc.sh` / `jstackproc64.sh` | SA 版 jstack 线程栈打印 |
| `jhistoproc.sh` / `jhistoproc64.sh` | SA 版 jhisto 类直方图打印 |
| `heapdumpproc.sh` / `heapdumpproc64.sh` | SA 版堆转储 |
| `heapsumproc.sh` / `heapsumproc64.sh` | SA 版堆摘要 |
| `pstackproc.sh` / `pstackproc64.sh` | SA 版 pstack 进程栈打印 |
| `pmapproc.sh` / `pmapproc64.sh` | SA 版 pmap 内存映射打印 |
| `dumpflagsproc.sh` / `dumpflagsproc64.sh` | 打印 VM 标志 |
| `dumpsyspropsproc.sh` / `dumpsyspropsproc64.sh` | 打印系统属性 |
| `permstatproc.sh` / `permstatproc64.sh` | 打印永久代统计 |
| `finalizerinfoproc.sh` / `finalizerinfoproc64.sh` | 打印终结器信息 |
| `jcoreproc.sh` / `jcoreproc64.sh` | 生成核心转储 |
| `saenv.sh` / `saenv64.sh` | SA 环境变量设置脚本 |

**Windows 运行脚本（`.bat`）：** 与 Unix 脚本一一对应，如 `hsdb.bat`、`clhsdbwindbg.bat`、`clhsdbwindbg64.bat`、`jstackwindbg.bat` 等。

### agent/src/

SA 的源代码，分为平台原生支持和共享 Java 代码。

```
agent/src/
├── os/          # 平台原生支持
│   ├── bsd/     # BSD 平台原生代码
│   ├── linux/   # Linux 平台原生代码
│   ├── solaris/ # Solaris 平台原生代码
│   └── win32/   # Windows 平台原生代码
├── scripts/     # 启动脚本
└── share/       # 共享 Java 源码
    ├── classes/ # Java 源码树
    └── native/  # JNI 原生接口支持
```

**`agent/src/scripts/`** — 远程调试启动脚本：

| 文件 | 说明 |
|------|------|
| `start-debug-server.sh` / `.bat` / `64.sh` | 启动远程调试服务器 |
| `start-rmiregistry.sh` / `.bat` / `64.sh` | 启动 RMI 注册表 |
| `README` | 脚本使用说明 |

**`agent/src/share/classes/`** — Java 源码树，包结构为 `sun.jvm.hotspot.*`：

| 包 | 说明 |
|----|------|
| `sun.jvm.hotspot` | 顶层 SA API 与启动入口 |
| `sun.jvm.hotspot.debugger` | 调试器后端（进程附加、核心转储读取） |
| `sun.jvm.hotspot.debugger.proc` | 基于 `/proc` 的调试器后端 |
| `sun.jvm.hotspot.debugger.remote` | 远程 RMI 调试器后端 |
| `sun.jvm.hotspot.debugger.windbg` | Windows Windbg 调试器后端 |
| `sun.jvm.hotspot.tools` | 命令行 SA 工具（jmap、jstack、jcore 等） |
| `sun.jvm.hotspot.tools.jcore` | 核心转储类提取工具 |
| `sun.jvm.hotspot.tools.soql` | SOQL（Simple Object Query Language）查询引擎 |
| `sun.jvm.hotspot.ui` | HSDB GUI 组件 |
| `sun.jvm.hotspot.utilities` | 通用工具库 |
| `sun.jvm.hotspot.jdi` | JDI（Java Debug Interface）连接器 |
| `sun.jvm.hotspot.runtime` | VM 运行时模型 |
| `sun.jvm.hotspot.oops` | 对象模型（Klass、Oop 等） |
| `sun.jvm.hotspot.memory` | 内存模型 |
| `sun.jvm.hotspot.interpreter` | 解释器模型 |
| `sun.jvm.hotspot.gc_implementation` | GC 实现模型 |
| `sun.jvm.hotspot.gc_interface` | GC 接口模型 |
| `sun.jvm.hotspot.code` | 代码缓存模型 |
| `sun.jvm.hotspot.compiler` | 编译器模型 |
| `sun.jvm.hotspot.opto` | C2 优化器模型 |
| `sun.jvm.hotspot.c1` | C1 编译器模型 |
| `sun.jvm.hotspot.ci` | 编译器接口模型 |
| `sun.jvm.hotspot.asm` | 汇编器模型 |
| `sun.jvm.hotspot.prims` | 原语支持模型 |

**关键 Java 入口类：**

| 类 | 说明 |
|----|------|
| `HotSpotAgent.java` | SA 核心，负责本地进程附加、核心转储附加、远程调试服务器连接 |
| `HSDB.java` | HSDB 图形调试器主入口 |
| `CLHSDB.java` | CLHSDB 命令行调试器主入口 |
| `DebugServer.java` | 远程调试服务器入口 |
| `StackTrace.java` | 独立栈跟踪打印工具 |

### agent/test/

SA 的测试代码。

| 目录 | 说明 |
|------|------|
| `jdi/` | JDI 连接器测试，含 `sagclient.java`、`sagdoit.java`、`sagtest.java`、`SASanityChecker.java` 及多个 shell 测试脚本 |
| `libproc/` | libproc 原生接口测试，含 `LibprocClient.java`、`LibprocTest.java` 及脚本 |

---

## 📁 make/ — 构建系统

HotSpot 使用基于 GNU Make 的构建系统，按操作系统和 CPU 架构组织。

```
make/
├── Makefile               # 顶层构建入口
├── build.sh               # Unix 构建引导脚本
├── defs.make              # 共享构建定义
├── altsrc.make             # 替代源码路径支持
├── excludeSrc.make         # 排除源码列表
├── cscope.make             # cscope 索引生成
├── scm.make                # SCM 辅助
├── pic.make                # 位置无关代码编译选项
├── jprt.gmk               # JPRT（Java Putback Release Tool）配置
├── hotspot.script          # HotSpot 脚本配置
├── hotspot_version         # 版本元数据
├── hotspot_distro          # HotSpot 发行版标识
├── openjdk_distro          # OpenJDK 发行版标识
├── jdk6_hotspot_distro    # JDK6 兼容发行版标识
├── sa.files                # SA 源文件列表
├── templates/              # 许可证头模板
│   ├── gpl-header          # GPL 头模板
│   └── gpl-cp-header       # GPL+CP 头模板
├── aix/                    # AIX 平台构建
├── bsd/                    # BSD 平台构建
├── linux/                  # Linux 平台构建
├── solaris/                # Solaris 平台构建
└── windows/                # Windows 平台构建
```

### 顶层构建文件详解

| 文件 | 说明 |
|------|------|
| `Makefile` | 顶层 GNU Make 入口，定义 `all`、`product`、`fastdebug`、`optimized`、`debug`、`zero`、`shark`、`core`、`minimal1` 等构建目标，分派到各平台 Makefile |
| `build.sh` | Unix 构建引导脚本，验证 `JAVA_HOME`、解析 `ALT_BOOTDIR`/`ALT_OUTPUTDIR`、定位 `gnumake` 并执行构建 |
| `defs.make` | 跨平台共享定义，确定 `GAMMADIR`、`HS_SRC_DIR`、`HS_BUILD_DIR`，通过 `uname` 检测 OS，定义通用构建宏 |
| `altsrc.make` | 替代源码路径支持，允许覆盖源码位置 |
| `excludeSrc.make` | 排除特定源文件的配置 |
| `cscope.make` | 生成 cscope 代码索引 |
| `scm.make` | 源码管理辅助 |
| `pic.make` | 位置无关代码（PIC）编译选项 |
| `jprt.gmk` | JPRT 构建系统配置 |
| `hotspot_version` | HotSpot 版本号与发布元数据，供 Windows 和 Unix 构建使用 |
| `hotspot_distro` | HotSpot 发行版产品标识 |
| `openjdk_distro` | OpenJDK 发行版标识 |
| `jdk6_hotspot_distro` | JDK6 兼容发行版标识 |
| `sa.files` | SA 源文件列表 |

### make/linux/ — Linux 平台构建

```
linux/
├── Makefile               # Linux 构建入口
├── README                 # Linux 构建说明
├── adlc_updater           # ADLC 更新脚本
├── platform_amd64         # AMD64 平台配置
├── platform_i486         # i486 平台配置
├── platform_ia64         # IA-64 平台配置
├── platform_ppc64        # PPC64 平台配置
├── platform_sparc         # SPARC 平台配置
├── platform_sparcv9      # SPARCv9 平台配置
├── platform_zero.in      # Zero 解释器平台模板
├── platform_*.suncc      # Sun CC 编译器变体
└── makefiles/             # 构建规则子目录
```

**`make/linux/makefiles/`** 核心文件：

| 文件 | 说明 |
|------|------|
| `buildtree.make` | 构建树生成，创建目录结构与 Makefile 层次 |
| `defs.make` | Linux 特定构建定义 |
| `rules.make` | 通用构建规则 |
| `top.make` | 顶层规则，包含其他 makefile |
| `vm.make` | VM 核心编译规则 |
| `adlc.make` | ADL 编译器（Architecture Description Language Compiler）构建规则 |
| `sa.make` | Serviceability Agent 构建规则 |
| `saproc.make` | SA 原生过程（proc）库构建规则 |
| `gcc.make` | GCC 编译器选项与规则 |
| `sparcWorks.make` | Sun Studio 编译器选项 |
| `compiler1.make` | C1 编译器（Client Compiler）构建配置 |
| `compiler2.make` | C2 编译器（Server Compiler）构建配置 |
| `tiered.make` | 分层编译构建配置 |
| `core.make` | 核心VM构建配置（无编译器） |
| `minimal1.make` | 最小化 VM 构建配置 |
| `zero.make` | Zero 解释器构建配置 |
| `shark.make` | Shark/LLVM JIT 构建配置 |
| `amd64.make` / `i486.make` / `ia64.make` / `sparc.make` / `sparcv9.make` | 各 CPU 架构特定规则 |
| `jvmti.make` | JVMTI 生成规则 |
| `jsig.make` | 信号处理库构建规则 |
| `dtrace.make` | DTrace 探针构建规则 |
| `trace.make` | 跟踪事件生成规则 |
| `debug.make` / `fastdebug.make` / `optimized.make` / `product.make` | 各构建变体（调试/快速调试/优化/产品）规则 |
| `mapfile-vers-*` | 链接器版本脚本（符号导出控制） |
| `adjust-mflags.sh` | Make 标志调整脚本 |
| `build_vm_def.sh` | VM 定义文件生成脚本 |

### make/bsd/ — BSD 平台构建

结构类似 `make/linux/`，包含：

| 特殊文件 | 说明 |
|----------|------|
| `universal.gmk` | macOS Universal Binary 构建支持 |

### make/solaris/ — Solaris 平台构建

结构类似 `make/linux/`，额外包含：

| 特殊文件 | 说明 |
|----------|------|
| `platform_*.gcc` | GCC 编译器变体的平台配置 |
| `mapfile-vers-CORE` / `COMPILER1` / `COMPILER2` / `TIERED` | 各 VM 变体的链接器版本脚本 |
| `mapfile-vers-jvm_dtrace` / `jvm_db` | DTrace 相关版本脚本 |
| `reorder_CORE_amd64` | 函数重排序优化文件 |

### make/aix/ — AIX 平台构建

| 特殊文件 | 说明 |
|----------|------|
| `xlc.make` | IBM XL C/C++ 编译器规则 |
| `ppc64.make` | PowerPC 64 位架构规则 |

### make/windows/ — Windows 平台构建

```
windows/
├── build.bat              # Windows 构建脚本
├── build.make             # Windows 构建规则
├── build_vm_def.sh        # VM 定义文件生成
├── create.bat             # 项目文件创建脚本
├── create_obj_files.sh    # 目标文件列表生成
├── cross_build.bat        # 交叉构建脚本
├── get_msc_ver.sh         # MSVC 版本检测
├── jvmexp.lcf             # JVM 导出链接命令文件（Release）
├── jvmexp_g.lcf           # JVM 导出链接命令文件（Debug）
├── makefiles/             # 构建子规则
└── projectfiles/          # Visual Studio 项目文件
    ├── common/Makefile
    ├── compiler1/         # C1 编译器项目
    ├── compiler2/         # C2 编译器项目
    ├── tiered/            # 分层编译项目
    └── core/              # 核心 VM 项目
```

**`make/windows/makefiles/`** 核心文件：

| 文件 | 说明 |
|------|------|
| `defs.make` | Windows 特定构建定义 |
| `compile.make` | 编译规则 |
| `rules.make` | 通用构建规则 |
| `top.make` | 顶层规则 |
| `vm.make` | VM 核心编译规则 |
| `adlc.make` | ADL 编译器构建规则 |
| `sa.make` | SA 构建规则 |
| `jvmti.make` | JVMTI 生成规则 |
| `trace.make` | 跟踪事件生成规则 |
| `generated.make` | 生成代码规则 |
| `shared.make` | 共享编译规则 |
| `projectcreator.make` | Visual Studio 项目生成规则 |
| `sanity.make` | 构建前检查规则 |
| `debug.make` / `fastdebug.make` / `product.make` | 各构建变体规则 |

**`make/windows/projectfiles/`** — Visual Studio 项目文件：

| 目录 | 说明 |
|------|------|
| `common/` | 公共 Makefile |
| `compiler1/` | C1 编译器 VS 项目（`.dsp`、`.dsw`） |
| `compiler2/` | C2 编译器 VS 项目，含 ADL 编译器子项目 |
| `tiered/` | 分层编译 VS 项目，含 ADL 编译器子项目 |
| `core/` | 核心 VM VS 项目 |

---

## 📁 src/ — HotSpot VM 核心源码

HotSpot VM 源码按架构、操作系统和共享代码三层组织。

```
src/
├── cpu/      # CPU 架构特定代码
├── os/       # 操作系统特定代码
├── os_cpu/   # OS+CPU 组合特定代码
└── share/    # 跨平台共享代码
    ├── tools/ # 工具
    └── vm/    # VM 核心
```

### src/cpu/ — CPU 架构特定代码

每个 CPU 架构下有 `vm/` 子目录，包含该架构的汇编器、解释器、运行时、寄存器定义和代码生成支持。

```
cpu/
├── ppc/     # PowerPC 架构
│   └── vm/
├── sparc/   # SPARC 架构
│   └── vm/
├── x86/     # x86 架构（含 x86_32 和 x86_64）
│   └── vm/
└── zero/    # Zero 解释器（通用参考后端）
    └── vm/
```

**`src/cpu/ppc/vm/`** — PowerPC 架构支持：

| 关键文件 | 说明 |
|----------|------|
| `assembler_ppc.cpp` / `.hpp` | PowerPC 汇编器实现 |
| `macroAssembler_ppc.cpp` / `.hpp` | PowerPC 宏汇编器 |
| `interpreter_ppc.cpp` / `.hpp` | PowerPC 模板解释器 |
| `runtime_ppc.cpp` | PowerPC 运行时支持（信号处理、栈帧等） |
| `vmreg_ppc.cpp` / `.hpp` | PowerPC 虚拟寄存器定义 |
| `c1_CodeStubs_ppc.cpp` | C1 编译器 PowerPC 代码桩 |
| `c1_FrameMap_ppc.cpp` | C1 编译器 PowerPC 栈帧映射 |
| `c1_LIRAssembler_ppc.cpp` | C1 编译器 PowerPC LIR 汇编器 |
| `c1_Runtime1_ppc.cpp` | C1 编译器 PowerPC 运行时支持 |
| `c2_MacroAssembler_ppc.cpp` | C2 编译器 PowerPC 宏汇编器 |
| `interp_masm_ppc_64.cpp` | 解释器 PowerPC 64 位宏汇编器 |
| `nativeInst_ppc.cpp` | PowerPC 原生指令 |
| `relocInfo_ppc.hpp` | PowerPC 重定位信息 |
| `register_ppc.cpp` / `.hpp` | PowerPC 寄存器定义 |
| `stubGenerator_ppc_64.cpp` | PowerPC 64 位桩代码生成器 |
| `templateInterpreterGenerator_ppc_64.cpp` | PowerPC 64 位模板解释器生成器 |
| `templateTable_ppc_64.cpp` | PowerPC 64 位字节码模板表 |
| `vtableStubs_ppc_64.cpp` | PowerPC 64 位虚表桩 |

**`src/cpu/sparc/vm/`** — SPARC 架构支持：

| 关键文件 | 说明 |
|----------|------|
| `assembler_sparc.cpp` / `.hpp` | SPARC 汇编器实现 |
| `macroAssembler_sparc.cpp` / `.hpp` | SPARC 宏汇编器 |
| `interpreter_sparc.cpp` | SPARC 解释器 |
| `runtime_sparc.cpp` | SPARC 运行时支持 |
| `vmreg_sparc.cpp` / `.hpp` | SPARC 虚拟寄存器 |
| `c1_*_sparc.cpp` | C1 编译器 SPARC 相关文件 |
| `c2_MacroAssembler_sparc.cpp` | C2 编译器 SPARC 宏汇编器 |

**`src/cpu/x86/vm/`** — x86 架构支持（32位和64位）：

| 关键文件 | 说明 |
|----------|------|
| `assembler_x86.cpp` / `.hpp` | x86 汇编器实现（同时支持 32/64 位） |
| `macroAssembler_x86.cpp` / `.hpp` | x86 宏汇编器 |
| `interpreter_x86_32.cpp` / `interpreter_x86_64.cpp` | x86 解释器（分 32/64 位） |
| `runtime_x86_32.cpp` / `runtime_x86_64.cpp` | x86 运行时（分 32/64 位） |
| `vmreg_x86.cpp` / `.hpp` | x86 虚拟寄存器 |
| `c1_*_x86.cpp` | C1 编译器 x86 相关文件 |
| `c2_MacroAssembler_x86.cpp` | C2 编译器 x86 宏汇编器 |
| `templateTable_x86_32.cpp` / `templateTable_x86_64.cpp` | x86 字节码模板表（分 32/64 位） |
| `vtableStubs_x86_32.cpp` / `vtableStubs_x86_64.cpp` | x86 虚表桩（分 32/64 位） |

**`src/cpu/zero/vm/`** — Zero 解释器（架构无关参考实现）：

| 关键文件 | 说明 |
|----------|------|
| `assembler_zero.cpp` / `.hpp` | Zero 汇编器（最小实现） |
| `interpreter_zero.cpp` / `.hpp` | Zero 解释器 |
| `runtime_zero.cpp` | Zero 运行时 |
| `vmreg_zero.cpp` / `.hpp` | Zero 虚拟寄存器 |
| `stubGenerator_zero.cpp` | Zero 桩代码生成器 |

### src/os/ — 操作系统特定代码

```
os/
├── aix/      # AIX 操作系统
│   └── vm/
├── bsd/      # BSD/macOS 操作系统
│   ├── dtrace/  # BSD DTrace 支持
│   ├── launcher/ # BSD JVM 启动器
│   └── vm/
├── linux/    # Linux 操作系统
│   └── vm/
├── posix/    # POSIX 共享代码
│   └── vm/
├── solaris/  # Solaris 操作系统
│   ├── dtrace/  # Solaris DTrace 支持
│   └── vm/
└── windows/  # Windows 操作系统
    └── vm/
```

**各 `os/*/vm/` 目录的关键文件：**

| 通用文件名模式 | 说明 |
|----------------|------|
| `os_<platform>.cpp` | 操作系统接口实现（线程、内存、信号、文件IO等） |
| `thread_<platform>.cpp` / `.hpp` | 平台线程实现 |
| `mutex_<platform>.cpp` | 平台互斥锁实现 |
| `perf_<platform>.cpp` | 平台性能计数器 |
| `vmStructs_<platform>.cpp` | 平台 VM 结构定义（用于 SA） |
| `attachListener_<platform>.cpp` | 平台附加监听器 |
| `jvm_<platform>.cpp` | JVM 接口的平台实现 |
| `c1_globals_<platform>.hpp` / `c2_globals_<platform>.hpp` | 平台特定编译器全局参数 |

**`src/os/bsd/dtrace/`** 和 **`src/os/solaris/dtrace/`** — DTrace 探针：

| 文件 | 说明 |
|------|------|
| `jvm_dtrace.c` / `.h` | JVM DTrace 探针实现 |
| `jvm_dtrace_helper.c` | DTrace 辅助函数 |

**`src/os/bsd/launcher/`** — BSD JVM 启动器：

| 文件 | 说明 |
|------|------|
| `java.c` | JVM 启动器主入口 |
| `java.h` | 启动器头文件 |
| `main.c` | 主函数 |

**`src/os/posix/vm/`** — POSIX 共享代码：

| 文件 | 说明 |
|------|------|
| `os_posix.cpp` | POSIX 操作系统共享实现 |

### src/os_cpu/ — OS+CPU 组合特定代码

这些目录包含操作系统与 CPU 架构组合的粘合代码。

```
os_cpu/
├── aix_ppc/       # AIX + PowerPC
│   └── vm/
├── bsd_x86/       # BSD + x86
│   └── vm/
├── bsd_zero/      # BSD + Zero
│   └── vm/
├── linux_ppc/     # Linux + PowerPC
│   └── vm/
├── linux_sparc/   # Linux + SPARC
│   └── vm/
├── linux_x86/     # Linux + x86
│   └── vm/
├── linux_zero/    # Linux + Zero
│   └── vm/
├── solaris_sparc/ # Solaris + SPARC
│   └── vm/
├── solaris_x86/   # Solaris + x86
│   └── vm/
└── windows_x86/   # Windows + x86
    └── vm/
```

**各 `os_cpu/*/vm/` 目录的典型文件：**

| 文件 | 说明 |
|------|------|
| `atomic_<os>_<cpu>.inline.hpp` | 原子操作内联实现 |
| `bytes_<os>_<cpu>.inline.hpp` | 字节序处理 |
| `orderAccess_<os>_<cpu>.inline.hpp` | 内存访问排序 |
| `os_<os>_<cpu>.cpp` | OS+CPU 特定运行时 |
| `prefetch_<os>_<cpu>.inline.hpp` | 预取指令 |
| `thread_<os>_<cpu>.cpp` / `.hpp` | 线程平台实现 |
| `vmStructs_<os>_<cpu>.hpp` | VM 结构定义 |
| `gc_<os>_<cpu>.cpp` | GC 平台支持 |

### src/share/ — 跨平台共享代码

```
share/
├── tools/    # 开发工具
│   ├── hsdis/                 # 反汇编器插件
│   ├── IdealGraphVisualizer/  # 理想图可视化工具
│   ├── LogCompilation/        # 编译日志分析工具
│   └── ProjectCreator/        # VS 项目生成器
└── vm/       # VM 核心源码（最大子目录）
```

#### src/share/tools/

| 目录 | 说明 |
|------|------|
| `hsdis/` | HotSpot 反汇编器插件，使用 GNU binutils 或 LLVM 进行代码反汇编 |
| `IdealGraphVisualizer/` | C2 编译器理想图（Ideal Graph）可视化工具，基于 Java Swing |
| `LogCompilation/` | 编译日志（`-XX:+LogCompilation`）分析工具 |
| `ProjectCreator/` | Windows Visual Studio 项目文件生成器 |

#### src/share/vm/ — HotSpot VM 核心源码

这是整个项目最核心、最大的源码目录，按子系统组织。

```
vm/
├── adlc/          # ADL 编译器
├── asm/           # 汇编器抽象层
├── c1/            # C1 编译器（Client Compiler）
├── ci/            # 编译器接口
├── classfile/     # 类文件加载与验证
├── code/          # 代码缓存与生成代码管理
├── compiler/      # 编译器协调与策略
├── gc_implementation/ # GC 算法实现
├── gc_interface/  # GC 接口
├── interpreter/  # 字节码解释器
├── libadt/        # 基本数据结构
├── memory/        # 内存管理
├── oops/          # 对象模型
├── opto/          # C2 编译器（Server Compiler）
├── precompiled/   # 预编译头
├── prims/         # JNI/JVMTI/原生方法
├── runtime/       # VM 运行时核心
├── services/      # 诊断与管理服务
├── shark/         # Shark/LLVM JIT 后端
├── trace/         # 跟踪事件
├── utilities/     # 通用工具
└── Xusage.txt     # -X 参数帮助文本
```

##### vm/adlc/ — ADL 编译器（Architecture Description Language Compiler）

ADL 编译器从 `.ad` 架构描述文件生成 C++ 代码，是 HotSpot 的代码生成基础设施。

| 关键文件 | 说明 |
|----------|------|
| `adlparse.cpp` / `.hpp` | ADL 语法解析器 |
| `archDesc.cpp` / `.hpp` | 架构描述类 |
| `output_c.cpp` / `output_h.cpp` | C++ 代码生成器 |
| `form.cpp` / `.hpp` | 指令/操作数格式基类 |
| `formsopt.cpp` / `.hpp` | 优化相关格式 |
| `forms.cpp` / `.hpp` | 格式管理 |
| `dict2.cpp` / `.hpp` | 字典数据结构 |
| `filebuff.cpp` / `.hpp` | 文件缓冲区 |
| `opto/adlc.hpp` | ADLC 配置头 |

##### vm/asm/ — 汇编器抽象层

| 关键文件 | 说明 |
|----------|------|
| `assembler.cpp` / `.hpp` | 汇编器基类 |
| `codeBuffer.cpp` / `.hpp` | 代码缓冲区管理 |
| `macroAssembler.hpp` | 宏汇编器基类（平台特定实现在 cpu/ 下） |
| `register.cpp` / `.hpp` | 寄存器抽象 |

##### vm/c1/ — C1 编译器（Client Compiler）

C1 是 HotSpot 的客户端编译器，专注于快速编译。

| 关键文件 | 说明 |
|----------|------|
| `c1_Compiler.cpp` / `.hpp` | C1 编译器主入口 |
| `c1_Optimizer.cpp` / `.hpp` | C1 优化器 |
| `c1_LIR.cpp` / `.hpp` | 低级中间表示（LIR） |
| `c1_HIR.cpp` / `.hpp` | 高级中间表示（HIR） |
| `c1_FrameMap.cpp` / `.hpp` | 栈帧映射 |
| `c1_Runtime1.cpp` / `.hpp` | C1 运行时支持 |
| `c1_CodeStubs.cpp` / `.hpp` | 代码桩 |
| `c1_GraphBuilder.cpp` / `.hpp` | 图构建器 |
| `c1_IR.cpp` / `.hpp` | 中间表示管理 |
| `c1_Instruction.cpp` / `.hpp` | HIR 指令 |
| `c1_LinearScan.cpp` / `.hpp` | 线性扫描寄存器分配 |
| `c1_ValueType.cpp` / `.hpp` | 值类型系统 |
| `c1_RangeCheckElimination.cpp` | 范围检查消除 |

##### vm/ci/ — 编译器接口（Compiler Interface）

编译器接口层，为 C1/C2 提供对 VM 内部结构的统一访问。

| 关键文件 | 说明 |
|----------|------|
| `ciEnv.cpp` / `.hpp` | 编译器环境，管理编译请求 |
| `ciKlass.cpp` / `.hpp` | 类元数据表示 |
| `ciMethod.cpp` / `.hpp` | 方法元数据表示 |
| `ciField.cpp` / `.hpp` | 字段元数据表示 |
| `ciObjectFactory.cpp` / `.hpp` | 编译器对象工厂 |
| `ciInstanceKlass.cpp` / `.hpp` | 实例类表示 |
| `ciArrayKlass.cpp` / `.hpp` | 数组类表示 |
| `ciTypeFlow.cpp` / `.hpp` | 类型流分析 |

##### vm/classfile/ — 类文件加载与验证

| 关键文件 | 说明 |
|----------|------|
| `classFileParser.cpp` / `.hpp` | 类文件解析器，解析 .class 文件格式 |
| `classLoader.cpp` / `.hpp` | 类加载器实现 |
| `verifier.cpp` / `.hpp` | 字节码验证器 |
| `systemDictionary.cpp` / `.hpp` | 系统字典，管理已加载类 |
| `symbolTable.cpp` / `.hpp` | 符号表，管理字符串常量 |
| `stringTable.cpp` / `.hpp` | 字符串表，管理 intern 字符串 |
| `dictionary.cpp` / `.hpp` | 类加载器字典 |
| `loaderConstraints.cpp` / `.hpp` | 加载器约束 |
| `resolutionErrors.cpp` / `.hpp` | 解析错误表 |
| `javaClasses.cpp` / `.hpp` | Java 核心类映射 |
| `vmSymbols.cpp` / `.hpp` | VM 符号常量 |

##### vm/code/ — 代码缓存与生成代码管理

| 关键文件 | 说明 |
|----------|------|
| `codeBlob.cpp` / `.hpp` | 代码块抽象 |
| `codeCache.cpp` / `.hpp` | 代码缓存管理 |
| `nmethod.cpp` / `.hpp` | 已编译方法（nmethod）管理 |
| `debugInfo.cpp` / `.hpp` | 调试信息 |
| `relocInfo.cpp` / `.hpp` | 重定位信息 |
| `stubs.cpp` / `.hpp` | 桩代码管理 |
| `pcDesc.cpp` / `.hpp` | PC 描述符 |
| `scopeDesc.cpp` / `.hpp` | 作用域描述 |
| `exceptionHandlerTable.cpp` / `.hpp` | 异常处理表 |
| `icBuffer.cpp` / `.hpp` | 内联缓存缓冲区 |
| `vtableStubs.cpp` / `.hpp` | 虚表桩 |

##### vm/compiler/ — 编译器协调与策略

| 关键文件 | 说明 |
|----------|------|
| `compileBroker.cpp` / `.hpp` | 编译请求代理，管理编译任务队列 |
| `compilerOracle.cpp` / `.hpp` | 编译器指令/策略（CompileCommand） |
| `compileLog.cpp` / `.hpp` | 编译日志 |
| `oopMap.cpp` / `.hpp` | Oop 映射（GC 根集） |
| `compileTask.cpp` / `.hpp` | 编译任务 |
| `methodLiveness.cpp` / `.hpp` | 方法活跃性分析 |
| `disassembler.cpp` / `.hpp` | 反汇编器接口 |

##### vm/gc_implementation/ — GC 算法实现

```
gc_implementation/
├── concurrentMarkSweep/ # CMS 垃圾收集器
├── g1/                  # G1 垃圾收集器
├── parallelScavenge/    # Parallel Scavenge 收集器
├── parNew/              # ParNew 收集器
└── shared/              # GC 共享代码
```

**`concurrentMarkSweep/`** — CMS 收集器：

| 关键文件 | 说明 |
|----------|------|
| `concurrentMarkSweepGeneration.cpp` | CMS 代实现 |
| `cmsCollectorPolicy.cpp` | CMS 收集策略 |
| `cmsOopClosures.hpp` | CMS Oop 闭包 |
| `compactibleFreeListSpace.cpp` | 可压缩空闲列表空间 |
| `concurrentMarkSweepThread.cpp` | CMS 后台线程 |

**`g1/`** — G1 收集器：

| 关键文件 | 说明 |
|----------|------|
| `g1CollectedHeap.cpp` | G1 堆实现 |
| `g1CollectorPolicy.cpp` | G1 收集策略 |
| `g1Allocator.cpp` | G1 分配器 |
| `g1BlockOffsetTable.cpp` | 块偏移表 |
| `g1HeapRegion.cpp` | G1 堆区域 |
| `g1RemSet.cpp` | G1 记忆集 |
| `g1SATBCardTableModRefBS.cpp` | SATB 屏障 |
| `heapRegionSet.cpp` | 堆区域集合 |
| `suspiciousSemantics.cpp` | 可疑语义检查 |

**`parallelScavenge/`** — Parallel Scavenge 收集器：

| 关键文件 | 说明 |
|----------|------|
| `parallelScavengeHeap.cpp` | Parallel Scavenge 堆 |
| `psScavenge.cpp` | PS 收集器 |
| `psMarkSweep.cpp` | PS 标记-清除 |
| `psParallelCompact.cpp` | PS 并行压缩 |
| `psAdaptiveSizePolicy.cpp` | 自适应大小策略 |
| `aspaceCounters.cpp` | 空间计数器 |

**`parNew/`** — ParNew 收集器：

| 关键文件 | 说明 |
|----------|------|
| `parNewGeneration.cpp` | ParNew 代实现 |

**`shared/`** — GC 共享代码：

| 关键文件 | 说明 |
|----------|------|
| `adaptiveSizePolicy.cpp` | 自适应大小策略 |
| `ageTable.cpp` | 年龄表 |
| `collectorCounters.cpp` | 收集器计数器 |
| `gcUtil.cpp` | GC 工具 |
| `markSweep.cpp` | 标记-清除算法 |
| `spaceCounters.cpp` | 空间计数器 |

##### vm/gc_interface/ — GC 接口

| 关键文件 | 说明 |
|----------|------|
| `collectedHeap.cpp` / `.hpp` | 收集堆基类 |
| `gcCause.cpp` / `.hpp` | GC 原因枚举 |
| `allocTracer.cpp` / `.hpp` | 分配跟踪器 |

##### vm/interpreter/ — 字节码解释器

| 关键文件 | 说明 |
|----------|------|
| `bytecodeInterpreter.cpp` / `.hpp` | 字节码解释器核心实现 |
| `templateInterpreter.cpp` / `.hpp` | 模板解释器 |
| `templateTable.cpp` / `.hpp` | 字节码模板表 |
| `interpreter.cpp` / `.hpp` | 解释器抽象层 |
| `interpreterRuntime.cpp` / `.hpp` | 解释器运行时支持 |
| `linkResolver.cpp` / `.hpp` | 链接解析器 |
| `oopMapCache.cpp` / `.hpp` | Oop 映射缓存 |
| `bytecode.cpp` / `.hpp` | 字节码表示 |
| `bytecodeStream.cpp` / `.hpp` | 字节码流 |
| `rewriter.cpp` / `.hpp` | 字节码重写器 |
| `invocationCounter.cpp` / `.hpp` | 调用计数器 |

##### vm/libadt/ — 基本数据结构

| 关键文件 | 说明 |
|----------|------|
| `dict2.cpp` / `.hpp` | 字典 |
| `set.cpp` / `.hpp` | 集合 |
| `vectset.cpp` / `.hpp` | 位向量集合 |

##### vm/memory/ — 内存管理

| 关键文件 | 说明 |
|----------|------|
| `allocation.cpp` / `.hpp` | 内存分配器（Arena、ResourceArea） |
| `heap.cpp` / `.hpp` | 代码堆管理 |
| `generation.cpp` / `.hpp` | 分代管理 |
| `genCollectedHeap.cpp` / `.hpp` | 分代收集堆 |
| `metaspace.cpp` / `.hpp` | 元空间（Metaspace）管理 |
| `universe.cpp` / `.hpp` | VM 宇宙（全局单例） |
| `space.cpp` / `.hpp` | 内存空间抽象 |
| `edenSpace.cpp` / `.hpp` | Eden 空间 |
| `defNewGeneration.cpp` / `.hpp` | 默认新生代 |
| `tenuredGeneration.cpp` / `.hpp` | 老年代 |
| `binaryTreeDictionary.cpp` / `.hpp` | 二叉树字典（空闲列表） |
| `freeList.cpp` / `.hpp` | 空闲列表 |
| `blockOffsetTable.cpp` / `.hpp` | 块偏移表 |
| `cardTableRS.cpp` / `.hpp` | 卡表记忆集 |
| `resourceArea.cpp` / `.hpp` | 资源区域 |
| `threadLocalAllocBuffer.cpp` / `.hpp` | 线程本地分配缓冲区（TLAB） |

##### vm/oops/ — 对象模型

| 关键文件 | 说明 |
|----------|------|
| `oop.cpp` / `.hpp` | 对象头（oop）基本抽象 |
| `klass.cpp` / `.hpp` | 类元数据基类（Klass） |
| `instanceKlass.cpp` / `.hpp` | 实例类元数据 |
| `arrayKlass.cpp` / `.hpp` | 数组类元数据 |
| `objArrayKlass.cpp` / `.hpp` | 对象数组类 |
| `typeArrayKlass.cpp` / `.hpp` | 类型数组类 |
| `method.cpp` / `.hpp` | 方法元数据 |
| `constMethod.cpp` / `.hpp` | 常量方法数据 |
| `methodData.cpp` / `.hpp` | 方法性能数据（Profile） |
| `constantPool.cpp` / `.hpp` | 常量池 |
| `symbol.cpp` / `.hpp` | 符号 |
| `metadata.cpp` / `.hpp` | 元数据基类 |
| `cpCache.cpp` / `.hpp` | 常量池缓存 |
| `generateOopMap.cpp` / `.hpp` | Oop 映射生成器 |
| `markOop.cpp` / `.hpp` | 对象标记字（Mark Word） |
| `compressedOops.cpp` / `.hpp` | 压缩 Oop |
| `fieldInfo.cpp` / `.hpp` | 字段信息 |
| `annotations.cpp` / `.hpp` | 注解 |

##### vm/opto/ — C2 编译器（Server Compiler）

C2 是 HotSpot 的服务端编译器，基于理想图（Ideal Graph）进行深度优化。

| 关键文件 | 说明 |
|----------|------|
| `compile.cpp` / `.hpp` | C2 编译主入口 |
| `node.cpp` / `.hpp` | 理想图节点基类 |
| `phaseX.cpp` | 优化阶段基类 |
| `phase.cpp` | 阶段管理 |
| `phaseaggressivecombine.cpp` | 激进组合优化 |
| `phaseidealloop.cpp` | 理想循环优化 |
| `phaselivenodes.cpp` | 活节点分析 |
| `phaseregalloc.cpp` | 寄存器分配 |
| `regalloc.cpp` / `.hpp` | 寄存器分配器 |
| `regmask.cpp` / `.hpp` | 寄存器掩码 |
| `graphKit.cpp` / `.hpp` | 图构建工具包 |
| `loopnode.cpp` / `.hpp` | 循环节点 |
| `loopopts.cpp` | 循环优化 |
| `superword.cpp` / `.hpp` | 自动向量化（SuperWord） |
| `vectornode.cpp` / `.hpp` | 向量节点 |
| `callnode.cpp` / `.hpp` | 调用节点 |
| `cfgnode.cpp` / `.hpp` | 控制流节点 |
| `connode.cpp` / `.hpp` | 常量节点 |
| `divnode.cpp` / `.hpp` | 除法节点 |
| `mulnode.cpp` / `.hpp` | 乘法节点 |
| `addnode.cpp` / `.hpp` | 加法节点 |
| `subnode.cpp` / `.hpp` | 减法节点 |
| `castnode.cpp` / `.hpp` | 类型转换节点 |
| `memnode.cpp` / `.hpp` | 内存节点 |
| `locknode.cpp` / `.hpp` | 锁节点 |
| `output.cpp` / `.hpp` | 代码生成输出 |
| `matcher.cpp` / `.hpp` | 指令选择匹配器 |
| `chaitin.cpp` / `.hpp` | Chaitin 图着色寄存器分配 |
| `buildOopMap.cpp` | Oop 映射构建 |
| `bytecodeInfo.cpp` | 字节码信息 |
| `escape.cpp` / `.hpp` | 逃逸分析 |
| `idealGraphPrinter.cpp` / `.hpp` | 理想图打印器（连接 IdealGraphVisualizer） |
| `stringopts.cpp` / `.hpp` | 字符串优化 |
| `arraycopynode.cpp` / `.hpp` | 数组拷贝节点 |

##### vm/prims/ — JNI/JVMTI/原生方法

| 关键文件 | 说明 |
|----------|------|
| `jni.cpp` / `.hpp` | JNI（Java Native Interface）实现 |
| `jvm.cpp` / `.hpp` | JVM 接口实现 |
| `jvmtiEnv.cpp` / `.hpp` | JVMTI 环境实现 |
| `jvmtiEnvBase.cpp` / `.hpp` | JVMTI 环境基类 |
| `jvmtiExport.cpp` / `.hpp` | JVMTI 导出 |
| `jvmtiManageCapabilities.cpp` | JVMTI 能力管理 |
| `jvmtiRedefineClasses.cpp` | JVMTI 类重定义 |
| `jvmtiTagMap.cpp` | JVMTI 标签映射 |
| `nativeLookup.cpp` / `.hpp` | 原生方法查找 |
| `unsafe.cpp` / `.hpp` | Unsafe API 实现 |
| `whitebox.cpp` / `.hpp` | WhiteBox 测试 API |
| `forte.cpp` / `.hpp` | Forte 性能分析接口 |
| `perf.cpp` / `.hpp` | 性能计数器原生接口 |
| `stackwalk.cpp` / `.hpp` | 栈遍历（StackWalk） |
| `vmStructs.cpp` / `.hpp` | VM 结构导出（用于 SA） |

##### vm/runtime/ — VM 运行时核心

| 关键文件 | 说明 |
|----------|------|
| `thread.cpp` / `.hpp` | Java 线程实现 |
| `safepoint.cpp` / `.hpp` | 安全点机制 |
| `os.cpp` / `.hpp` | 操作系统抽象层 |
| `frame.cpp` / `.hpp` | 栈帧实现 |
| `sharedRuntime.cpp` / `.hpp` | 共享运行时（出栈、异常处理等） |
| `vm_operations.cpp` / `.hpp` | VM 操作 |
| `vmThread.cpp` / `.hpp` | VM 线程 |
| `reflection.cpp` / `.hpp` | 反射支持 |
| `reflectionUtils.cpp` / `.hpp` | 反射工具 |
| `signature.cpp` / `.hpp` | 类型签名 |
| `handles.cpp` / `.hpp` | 句柄（Handle） |
| `monitorCache.cpp` / `.hpp` | 监视器缓存 |
| `objectMonitor.cpp` / `.hpp` | 对象监视器（同步） |
| `synchronizer.cpp` / `.hpp` | 同步器 |
| `mutex.cpp` / `.hpp` | 互斥锁 |
| `mutexLocker.cpp` / `.hpp` | 互斥锁便捷封装 |
| `perfData.cpp` / `.hpp` | 性能数据 |
| `vframe.cpp` / `.hpp` | 虚拟栈帧 |
| `deoptimization.cpp` / `.hpp` | 逆优化（Deoptimization） |
| `relocator.cpp` / `.hpp` | 代码重定位 |
| `stubRoutines.cpp` / `.hpp` | 桩例程 |
| `rtmLocking.cpp` / `.hpp` | RTM 锁（受限事务内存） |
| `compilationPolicy.cpp` / `.hpp` | 编译策略 |
| `fieldDescriptor.cpp` / `.hpp` | 字段描述符 |
| `java.cpp` / `.hpp` | JVM 初始化与关闭 |
| `javaCalls.cpp` / `.hpp` | Java 调用 |
| `jniHandles.cpp` / `.hpp` | JNI 句柄 |
| `tlab.cpp` / `.hpp` | 线程本地分配缓冲 |
| `globals.cpp` / `.hpp` | 全局参数定义 |
| `globals_x86.hpp` / `globals_sparc.hpp` / `globals_ppc.hpp` | 平台特定全局参数 |
| `arguments.cpp` / `.hpp` | 命令行参数解析 |
| `init.cpp` / `.hpp` | VM 初始化 |
| `vm_version.cpp` / `.hpp` | VM 版本信息 |

##### vm/services/ — 诊断与管理服务

| 关键文件 | 说明 |
|----------|------|
| `diagnosticCommand.cpp` / `.hpp` | 诊断命令（DCmd）框架 |
| `heapDumper.cpp` / `.hpp` | 堆转储（HPROF 格式） |
| `memoryService.cpp` / `.hpp` | 内存管理服务（JMX） |
| `memoryManager.cpp` / `.hpp` | 内存管理器 |
| `memoryPool.cpp` / `.hpp` | 内存池 |
| `classLoadingService.cpp` / `.hpp` | 类加载服务 |
| `threadService.cpp` / `.hpp` | 线程服务 |
| `runtimeService.cpp` / `.hpp` | 运行时服务 |
| `gcNotifier.cpp` / `.hpp` | GC 通知 |
| `management.cpp` / `.hpp` | 管理接口 |
| `attachListener.cpp` / `.hpp` | 附加监听器 |
| `writeableFlags.cpp` / `.hpp` | 可写标志管理 |

##### vm/shark/ — Shark/LLVM JIT 后端

Shark 是基于 LLVM 的 JIT 编译器后端（已弃用）。

| 关键文件 | 说明 |
|----------|------|
| `sharkCompiler.cpp` / `.hpp` | Shark 编译器主入口 |
| `sharkRuntime.cpp` / `.hpp` | Shark 运行时支持 |
| `sharkBuilder.cpp` / `.hpp` | LLVM IR 构建器 |
| `sharkBlock.cpp` / `.hpp` | 基本块 |
| `sharkFunction.cpp` / `.hpp` | 函数编译 |
| `sharkInliner.cpp` / `.hpp` | 内联器 |
| `sharkMemoryManager.cpp` / `.hpp` | 内存管理器 |
| `sharkTopLevelBlock.cpp` / `.hpp` | 顶层基本块 |
| `sharkValue.cpp` / `.hpp` | 值表示 |

##### vm/trace/ — 跟踪事件

| 关键文件 | 说明 |
|----------|------|
| `trace.xml` | 跟踪事件定义（XML 格式） |
| `tracetypes.xml` | 跟踪类型定义 |
| `traceEvent.hpp` | 跟踪事件基类 |
| `traceStream.hpp` | 跟踪流 |
| `traceMacros.hpp` | 跟踪宏 |
| `traceBackend.hpp` | 跟踪后端 |

##### vm/utilities/ — 通用工具

| 关键文件 | 说明 |
|----------|------|
| `hashtable.cpp` / `.hpp` | 哈希表 |
| `bitMap.cpp` / `.hpp` | 位图 |
| `elfFile.cpp` / `.hpp` | ELF 文件解析 |
| `vmError.cpp` / `.hpp` | VM 错误处理 |
| `stringUtils.cpp` / `.hpp` | 字符串工具 |
| `xmlstream.cpp` / `.hpp` | XML 流 |
| `debug.cpp` / `.hpp` | 调试断言 |
| `exceptions.cpp` / `.hpp` | 异常工具 |
| `events.cpp` / `.hpp` | 事件日志 |
| `globalDefinitions.cpp` / `.hpp` | 全局定义 |
| `growableArray.cpp` / `.hpp` | 可增长数组 |
| `linkedlist.cpp` / `.hpp` | 链表 |
| `numberSeq.cpp` / `.hpp` | 数值序列统计 |
| `ostream.cpp` / `.hpp` | 输出流 |
| `preserveException.cpp` / `.hpp` | 异常保存 |
| `taskqueue.cpp` / `.hpp` | 任务队列 |
| `workgroup.cpp` / `.hpp` | 工作组（并行执行） |
| `yieldTooter.cpp` / `.hpp` | 让步工具 |
| `copy.cpp` / `.hpp` | 内存拷贝 |
| `macros.hpp` | 通用宏 |

##### vm/precompiled/ — 预编译头

| 文件 | 说明 |
|------|------|
| `precompiled.hpp` | 预编译头文件，包含最常用的头文件 |

---

## 📁 test/ — 测试套件

HotSpot 使用 jtreg 测试框架，测试按功能域组织。

```
test/
├── Makefile               # 测试构建入口
├── TEST.ROOT              # 测试根标识
├── TEST.groups            # 测试分组定义
├── jprt.config            # JPRT 测试配置
├── jbProblemsList.txt     # 已知问题列表
├── test_env.sh            # 测试环境设置
├── compiler/              # 编译器测试
├── gc/                    # GC 通用测试
├── gc_implementation/     # GC 实现特定测试
├── runtime/               # 运行时测试
├── sanity/                # 基本健全性测试
├── serviceability/         # 可服务性测试
├── stress/                # 压力测试
├── testlibrary/           # 测试库
└── testlibrary_tests/     # 测试库自测
```

### 顶层测试文件

| 文件 | 说明 |
|------|------|
| `Makefile` | 测试执行入口，使用 jtreg 框架，定义 `hotspot_all`、`clienttest`、`servertest` 等目标 |
| `TEST.ROOT` | 测试套件根标识，声明测试键（`cte_test`、`jcmd`、`nmt`、`regression`、`gc`、`stress`）和分组文件 |
| `TEST.groups` | 测试分组定义，映射 `hotspot_compiler`、`hotspot_gc`、`hotspot_runtime`、`hotspot_serviceability`、`hotspot_all`、`hotspot_tier1` 等组到具体测试目录 |
| `jprt.config` | JPRT（Java Putback Release Tool）持续集成配置 |
| `jbProblemsList.txt` | 已知问题与排除列表 |
| `test_env.sh` | 测试环境变量设置 |

### test/compiler/ — 编译器测试

按功能子域组织的编译器回归与正确性测试。

| 子目录 | 说明 |
|--------|------|
| `arguments/` | 编译器参数测试 |
| `c1/` | C1 编译器特定测试 |
| `c2/` | C2 编译器特定测试 |
| `ciReplay/` | CI Replay（编译器回放）测试 |
| `classUnloading/` | 类卸载时的编译器行为测试 |
| `codecache/` | 代码缓存测试 |
| `codegen/` | 代码生成测试 |
| `cpuflags/` | CPU 标志测试 |
| `criticalnatives/` | 关键原生方法测试 |
| `debug/` | 调试信息测试 |
| `dependencies/` | 编译依赖测试 |
| `EliminateAutoBox/` | 自动装箱消除测试 |
| `EscapeAnalysis/` | 逃逸分析测试 |
| `exceptions/` | 异常处理测试 |
| `floatingpoint/` | 浮点运算测试 |
| `gcbarriers/` | GC 屏障测试 |
| `inlining/` | 方法内联测试 |
| `integerArithmetic/` | 整数算术测试 |
| `intrinsics/` | 内置函数（Intrinsic）测试 |
| `jsr292/` | JSR 292（InvokeDynamic）测试 |
| `loopopts/` | 循环优化测试 |
| `macronodes/` | 宏节点测试 |
| `membars/` | 内存屏障测试 |
| `native/` | 原生方法编译测试 |
| `osr/` | 栈上替换（OSR）测试 |
| `print/` | 编译输出测试 |
| `profiling/` | 方法性能分析测试 |
| `rangechecks/` | 范围检查测试 |
| `reflection/` | 反射编译测试 |
| `regalloc/` | 寄存器分配测试 |
| `relocations/` | 重定位测试 |
| `rtm/` | 受限事务内存（RTM）测试 |
| `stable/` | 稳定性测试 |
| `startup/` | 启动行为测试 |
| `stringopts/` | 字符串优化测试 |
| `testlibrary/` | 编译器测试支撑库 |
| `tiered/` | 分层编译测试 |
| `types/` | 类型系统测试 |
| `uncommontrap/` | 逆优化陷阱测试 |
| `unsafe/` | Unsafe API 测试 |
| `whitebox/` | WhiteBox API 测试 |

### test/gc/ — GC 通用测试

| 子目录 | 说明 |
|--------|------|
| `6581734/` 等 | 按Bug编号的回归测试 |
| `arguments/` | GC 参数测试 |
| `class_unloading/` | 类卸载与GC测试 |
| `concurrentMarkSweep/` | CMS 收集器测试 |
| `ergonomics/` | GC 自适应策略测试 |
| `g1/` | G1 收集器测试 |
| `logging/` | GC 日志测试 |
| `metaspace/` | 元空间GC测试 |
| `parallelScavenge/` | Parallel Scavenge 测试 |
| `startup_warnings/` | 启动警告测试 |
| `stress/` | GC 压力测试 |
| `survivorAlignment/` | Survivor 对齐测试 |
| `whitebox/` | GC WhiteBox 测试 |

另有独立测试文件如：
- `TestVerifyDuringStartup.java` — 启动期间验证
- `TestSystemGC.java` — System.gc() 测试
- `TestSoftReferencesBehaviorOnOOME.java` — OOM 时软引用行为

### test/gc_implementation/ — GC 实现特定测试

| 子目录 | 说明 |
|--------|------|
| `g1/` | G1 实现特定测试，如 `TestNoEagerReclaimOfHumongousRegions.java` |

### test/runtime/ — 运行时测试

最大的测试子集，按Bug编号和功能域组织。

| 子目录 | 说明 |
|--------|------|
| `6294277/` 等 | 按Bug编号的回归测试 |
| `CDSCompressedKPtrs/` | 压缩类指针的 CDS 测试 |
| `CheckEndorsedAndExtDirs/` | 标准扩展目录检查测试 |
| `ClassFile/` | 类文件测试 |
| `classFileParserBug/` | 类文件解析器Bug测试 |
| `ClassUnload/` | 类卸载测试 |
| `CommandLine/` | 命令行参数测试 |
| `CompressedOops/` | 压缩Oop测试 |
| `containers/` | 容器（Docker）感知测试 |
| `contended/` | `@Contended` 注解测试 |
| `EnableTracing/` | 跟踪启用测试 |
| `ErrorHandling/` | 错误处理测试 |
| `execstack/` | 执行栈测试 |
| `Final/` | finalize 测试 |
| `InitialThreadOverflow/` | 初始线程栈溢出测试 |
| `InternalApi/` | 内部API测试 |
| `interned/` | 字符串驻留测试 |
| `invokedynamic/` | InvokeDynamic 测试 |
| `jni/` | JNI 测试 |
| `jsig/` | 信号处理测试 |
| `lambda-features/` | Lambda 特性测试 |
| `LoadClass/` | 类加载测试 |
| `memory/` | 内存管理测试 |
| `Metaspace/` | 元空间测试 |
| `NMT/` | 原生内存跟踪（Native Memory Tracking）测试 |
| `os/` | 操作系统相关测试 |
| `RedefineTests/` | 类重定义测试 |
| `SharedArchiveFile/` | 共享归档文件测试 |
| `StackGap/` | 栈间隙测试 |
| `stackMapCheck/` | 栈图检查测试 |
| `testlibrary/` | 运行时测试支撑库 |
| `Thread/` | 线程相关测试 |
| `verifier/` | 验证器测试 |
| `VtableTests/` | 虚表测试 |
| `XCheckJniJsig/` | JNI 检查与信号测试 |

### test/sanity/ — 基本健全性测试

| 文件 | 说明 |
|------|------|
| `ExecuteInternalVMTests.java` | VM 内部测试执行 |
| `WBApi.java` | WhiteBox API 基本测试 |
| `MismatchedWhiteBox/` | WhiteBox 版本不匹配测试 |

### test/serviceability/ — 可服务性测试

| 子目录 | 说明 |
|--------|------|
| `7170638/` | Bug 回归测试 |
| `attach/` | Attach API 测试 |
| `dcmd/` | 诊断命令测试（如 `ClassLoaderStatsTest`、`DynLibDcmdTest`、`HeapInfoTest`） |
| `jvmti/` | JVMTI 测试（如 `GetObjectSizeOverflow`、`TestRedefineWithUnresolvedClass`） |
| `sa/` | Serviceability Agent 测试（如 `TestRevPtrsForInvokeDynamic`、jmap-hashcode、jmap-hprof） |
| `threads/` | 线程诊断测试 |
| `ParserTest.java` | 解析器测试 |

### test/stress/ — 压力测试

| 子目录 | 说明 |
|--------|------|
| `gc/` | GC 压力测试，如 `TestStressRSetCoarsening.java` |

### test/testlibrary/ — 测试支撑库

| 文件/目录 | 说明 |
|-----------|------|
| `ClassFileInstaller.java` | 类文件安装器 |
| `RedefineClassHelper.java` | 类重定义辅助 |
| `com/oracle/java/testlibrary/` | Oracle 测试库（`OutputAnalyzer`、`ProcessTools`、`JDKToolFinder`、`Platform`、`DockerTestUtils`、`PerfCounter` 等） |
| `ctw/` | CompileTheWorld 工具 |
| `whitebox/` | WhiteBox API 实现 |

### test/testlibrary_tests/ — 测试库自测

| 文件/目录 | 说明 |
|-----------|------|
| `AssertsTest.java` | 断言工具测试 |
| `OutputAnalyzerReportingTest.java` | 输出分析器报告测试 |
| `OutputAnalyzerTest.java` | 输出分析器测试 |
| `RedefineClassTest.java` | 类重定义测试 |
| `TestMutuallyExclusivePlatformPredicates.java` | 平台互斥谓词测试 |
| `whitebox/vm_flags/` | WhiteBox VM 标志测试（`BooleanTest`、`DoubleTest`、`IntxTest`、`StringTest`、`Uint64Test`、`UintxTest`） |

---

## 🏗 架构总结

```
                    ┌─────────────────────┐
                    │   HotSpot JVM (8u)   │
                    └─────────┬───────────┘
                              │
          ┌───────────────────┼───────────────────────┐
          │                   │                       │
    ┌─────┴─────┐     ┌──────┴──────┐          ┌─────┴─────┐
    │  src/cpu/  │     │  src/os/    │          │src/share/ │
    │  架构层    │     │  操作系统层  │          │  共享核心  │
    │ ppc/sparc/ │     │aix/bsd/     │          │  vm/      │
    │ x86/zero  │     │linux/solaris│          │  tools/   │
    └───────────┘     │ /windows    │          └───────────┘
                      └─────────────┘
                              │
                    ┌─────────┴──────────┐
                    │  src/os_cpu/       │
                    │  OS+架构组合层      │
                    │  (10个平台组合)     │
                    └────────────────────┘
```

**核心子系统依赖关系：**

```
prims (JNI/JVMTI)
  └→ runtime (线程/安全点/帧/OS抽象)
       └→ memory (堆/代/元空间)
            └→ oops (对象模型/Klass)
                 └→ classfile (类加载/验证/系统字典)
       └→ interpreter (字节码解释器)
       └→ compiler (编译代理/策略)
            └→ ci (编译器接口)
                 └→ c1 (Client Compiler)
                 └→ opto (Server Compiler/C2)
                      └→ adlc (架构描述)
       └→ code (代码缓存/nmethod)
       └→ services (诊断/管理)
  └→ gc_interface → gc_implementation (CMS/G1/PS/ParNew)
```

---

*本文档基于 OpenJDK 8u HotSpot 源码生成，涵盖了项目所有主要目录和关键文件的详细说明。*