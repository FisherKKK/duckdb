# DuckDB 高级课程：CMake编译系统与优化技巧

> 本课程是DuckDB 30天学习课程的补充，深入讲解CMake构建系统、编译优化技巧和性能调优方法。

---

## 第一部分：CMake构建系统详解

### 1.1 构建系统架构概览

DuckDB使用CMake作为主要构建系统，通过Makefile提供便捷的编译接口：

```
Makefile (用户接口)
    ↓
CMake (构建生成器)
    ↓
    ├── src/CMakeLists.txt (主库)
    ├── extension/*/CMakeLists.txt (扩展)
    ├── test/CMakeLists.txt (测试)
    └── third_party/*/CMakeLists.txt (第三方库)
    ↓
构建产物
    ├── libduckdb.so / duckdb.dll
    ├── libduckdb_static.a
    ├── duckdb (CLI工具)
    └── test/unittest (测试)
```

### 1.2 核心CMakeLists文件解析

#### 顶层 CMakeLists.txt

**文件位置:** `/home/dev/duckdb/CMakeLists.txt`

```cmake
# CMake最小版本要求
cmake_minimum_required(VERSION 3.5...3.29)

# 项目定义
project(DuckDB)

# C++标准设置
set(CMAKE_CXX_STANDARD "11" CACHE STRING "C++ standard to enforce")
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# 编译器缓存器支持 (ccache/sccache)
if(NOT DEFINED CMAKE_CXX_COMPILER_LAUNCHER)
    find_program(COMPILER_LAUNCHER NAMES ccache sccache)
    if(COMPILER_LAUNCHER)
        set(CMAKE_CXX_COMPILER_LAUNCHER "${COMPILER_LAUNCHER}")
    endif()
endif()
```

**关键配置选项：**

| 选项 | 默认值 | 说明 |
|------|--------|------|
| `CMAKE_CXX_STANDARD` | "11" | C++标准版本 |
| `CMAKE_CXX_STANDARD_REQUIRED` | ON | 强制使用指定标准 |
| `CMAKE_CXX_EXTENSIONS` | OFF | 禁用编译器扩展 |
| `CMAKE_EXPORT_COMPILE_COMMANDS` | ON | 导出compile_commands.json |

#### 源码目录 CMakeLists.txt

**文件位置:** `/home/dev/duckdb/src/CMakeLists.txt`

```cmake
# 定义DuckDB主程序宏
add_definitions(-DDUCKDB)
add_definitions(-DDUCKDB_MAIN_LIBRARY)

# 添加子目录（模块化构建）
add_subdirectory(optimizer)        # 优化器
add_subdirectory(planner)          # 查询规划器
add_subdirectory(parser)           # SQL解析器
add_subdirectory(function)         # 函数系统
add_subdirectory(catalog)          # 元数据目录
add_subdirectory(common)           # 公共组件
add_subdirectory(logging)          # 日志系统
add_subdirectory(execution)        # 执行引擎
add_subdirectory(main)             # 主入口
add_subdirectory(parallel)         # 并行执行
add_subdirectory(storage)          # 存储引擎
add_subdirectory(transaction)      # 事务管理
add_subdirectory(verification)     # 验证模块

# 第三方依赖库
set(DUCKDB_LINK_LIBS
    duckdb_fsst          # 字符串压缩
    duckdb_fmt           # 格式化库
    duckdb_pg_query      # PostgreSQL解析器
    duckdb_re2           # 正则表达式
    duckdb_miniz         # 压缩库
    duckdb_utf8proc      # UTF-8处理
    duckdb_hyperloglog   # HyperLogLog基数估计
    duckdb_fastpforlib   # FastPFor压缩
    duckdb_skiplistlib   # 跳表
    duckdb_mbedtls       # 加密库
    duckdb_yyjson        # JSON解析
    duckdb_zstd          # Zstandard压缩
)

# 构建静态库
add_library(duckdb_static STATIC ${ALL_OBJECT_FILES} ${LINK_OBJECTS})
target_link_libraries(duckdb_static PUBLIC ${DUCKDB_SYSTEM_LIBS})
link_threads(duckdb_static PUBLIC)

# 构建动态库
add_library(duckdb SHARED ${ALL_OBJECT_FILES} ${LINK_OBJECTS})
target_link_libraries(duckdb PUBLIC ${DUCKDB_SYSTEM_LIBS})
link_extension_libraries(duckdb PRIVATE)
```

### 1.3 Unity Build系统

DuckDB使用自定义的`add_library_unity`函数实现Unity Build，加速编译：

```cmake
# src/common/CMakeLists.txt
add_library_unity(
  duckdb_common
  OBJECT
  allocator.cpp
  assert.cpp
  file_system.cpp
  # ... 更多源文件
)
```

**Unity Build原理：**
- 将多个`.cpp`文件`#include`到一个统一文件中编译
- 减少编译器调用次数
- 降低模板实例化开销
- 加快编译速度2-5倍

**Unity Build缺点：**
- 可能导致符号冲突
- 代码必须严格遵守标准
- 禁用方式：`DISABLE_UNITY=1 make`

### 1.4 Makefile接口

**文件位置:** `/home/dev/duckdb/Makefile`

```makefile
# 主要构建目标
all: release              # 默认构建release版本
opt: release              # 同上
unit: unittest            # 运行单元测试

# 生成器选择
ifeq ($(GEN),ninja)
    GENERATOR=-G "Ninja"
    FORCE_COLOR=-DFORCE_COLORED_OUTPUT=1
endif

# 构建扩展
ifeq (${BUILD_TPCH}, 1)
    BUILD_EXTENSIONS:=${BUILD_EXTENSIONS};tpch
endif

# 编译配置
debug:
    cmake -S . -B build/debug \
          -DCMAKE_BUILD_TYPE=Debug \
          -DENABLE_SANITIZER=TRUE
    cmake --build build/debug -j$(nproc)

release:
    cmake -S . -B build/release \
          -DCMAKE_BUILD_TYPE=Release
    cmake --build build/release -j$(nproc)

reldebug:
    cmake -S . -B build/reldebug \
          -DCMAKE_BUILD_TYPE=RelWithDebInfo
    cmake --build build/reldebug -j$(nproc)
```

### 1.5 常用构建命令

#### 基础构建

```bash
# Debug构建（带符号和sanitizer）
make debug

# Release构建（优化）
make release

# RelWithDebInfo构建（优化+调试信息）
make reldebug

# 使用Ninja加速构建
GEN=ninja make

# 并行构建限制CPU核心数
CMAKE_BUILD_PARALLEL_LEVEL=4 GEN=ninja make
```

#### 扩展构建

```bash
# 构建特定扩展
BUILD_EXTENSIONS=parquet make
BUILD_EXTENSIONS=parquet;json;icu make

# 跳过特定扩展
SKIP_EXTENSIONS=parquet make

# 构建TPC-H基准测试扩展
BUILD_TPCH=1 make

# 构建所有扩展
BUILD_COMPLETE_EXTENSION_SET=1 make
```

#### 测试构建

```bash
# 构建并运行单元测试
make unittest

# 构建并运行所有测试
make allunit

# 运行特定测试
./build/debug/test/unittest "[sqlitelogic]"

# 逐个运行测试（用于调试）
python3 scripts/run_tests_one_by_one.py build/debug/test/unittest --time_execution
```

#### 优化构建选项

```bash
# 禁用sanitizer（加快编译）
DISABLE_SANITIZER=1 make debug

# 禁用unity build（增量编译）
DISABLE_UNITY=1 make

# 使用ccache加速重新编译
# ccache会自动被检测和启用

# 32位构建
FORCE_32_BIT=1 make

# 静态链接libcpp（macOS）
STATIC_LIBCPP=1 make
```

### 1.6 编译器标志详解

#### Debug模式标志

```cmake
# GCC/Clang通用警告
set(CMAKE_CXX_FLAGS_DEBUG
    "${CMAKE_CXX_FLAGS_DEBUG} -Wextra -Wno-unused-parameter -Wno-redundant-move"
)

# GCC特定
if(CMAKE_COMPILER_IS_GNUCC)
    set(CMAKE_CXX_FLAGS_DEBUG
        "${CMAKE_CXX_FLAGS_DEBUG} -Wimplicit-fallthrough"
    )
endif()

# Clang特定
if("${CMAKE_CXX_COMPILER_ID}" MATCHES "Clang")
    set(CMAKE_CXX_FLAGS_DEBUG
        "${CMAKE_CXX_FLAGS_DEBUG} -Wexit-time-destructors \
         -Wimplicit-int-conversion -Wshorten-64-to-32 \
         -Wnarrowing -Wsign-conversion -Wsign-compare \
         -Wconversion -Wtype-limits"
    )
endif()
```

#### Sanitizer配置

```cmake
# Address Sanitizer (ASan)
option(ENABLE_SANITIZER "Enable address sanitizer." TRUE)
if(${ENABLE_SANITIZER})
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=address")
endif()

# Thread Sanitizer (TSan)
option(ENABLE_THREAD_SANITIZER "Enable thread sanitizer." FALSE)
if(${ENABLE_THREAD_SANITIZER})
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=thread")
endif()

# Undefined Behavior Sanitizer (UBSan)
option(ENABLE_UBSAN "Enable undefined behavior sanitizer." TRUE)
if(${ENABLE_UBSAN})
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=undefined -fno-sanitize-recover=all")
endif()

# 禁用vptr sanitizer（M1 Mac兼容性）
option(DISABLE_VPTR_SANITIZER "Disable vptr sanitizer" FALSE)
if(${DISABLE_VPTR_SANITIZER})
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fno-sanitize=vptr")
endif()
```

**Sanitizer使用建议：**

| Sanitizer | 检测问题 | 性能影响 | 推荐场景 |
|-----------|----------|----------|----------|
| ASan | 内存错误（越界、use-after-free） | 2-3x | 日常开发 |
| TSan | 数据竞争 | 5-10x | 多线程调试 |
| UBSan | 未定义行为 | 1.5-2x | 发布前检查 |

#### Release优化标志

```bash
# Release模式使用-O3优化
CMAKE_BUILD_TYPE=Release

# 手动添加优化标志
CMAKE_CXX_FLAGS_RELEASE="-O3 -march=native -mtune=native"

# 链接时优化 (LTO)
CMAKE_INTERPROCEDURAL_OPTIMIZATION=TRUE

# 函数和数据段分离（用于扩展链接）
CMAKE_CXX_FLAGS_RELEASE="-ffunction-sections -fdata-sections"
```

### 1.7 跨平台编译配置

#### 平台检测

```cmake
# 操作系统检测
if(APPLE)
    set(OS_NAME "osx")
elseif(WIN32)
    set(OS_NAME "windows")
elseif(UNIX)
    set(OS_NAME "linux")
endif()

# 架构检测
string(REGEX MATCH "(arm64|aarch64)" IS_ARM "${CMAKE_SYSTEM_PROCESSOR}")
if(IS_ARM)
    set(OS_ARCH "arm64")
elseif(FORCE_32_BIT)
    set(OS_ARCH "i386")
else()
    set(OS_ARCH "amd64")
endif()
```

#### macOS特定配置

```bash
# 构建通用二进制（x86_64 + arm64）
OSX_BUILD_UNIVERSAL=1 make

# 构建特定架构
OSX_BUILD_ARCH=arm64 make

# 设置部署目标
CMAKE_OSX_DEPLOYMENT_TARGET=11.0 make
```

#### Windows特定配置

```cmake
# MSVC运行时库
set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")

# 大对象支持（Windows）
if(MSVC)
    add_compile_options("/bigobj")
endif()

# 系统库
if(MSVC OR MINGW)
    set(DUCKDB_SYSTEM_LIBS ${DUCKDB_SYSTEM_LIBS} ws2_32 rstrtmgr)
endif()
```

---

## 第二部分：代码架构实现细节

### 2.1 模块化架构

DuckDB采用高度模块化的设计，每个子目录对应一个独立模块：

```
src/
├── common/           # 公共基础组件
├── parser/           # SQL解析
├── binder/           # 符号绑定
├── planner/          # 逻辑计划
├── optimizer/        # 查询优化
├── execution/        # 物理执行
├── storage/          # 存储引擎
├── transaction/      # 事务管理
├── catalog/          # 元数据管理
├── function/         # 函数注册
├── main/             # 数据库连接
├── parallel/         # 并行执行
└── verification/     # 验证工具
```

#### 模块依赖关系

```
                 ┌─────────────┐
                 │    main     │  (数据库连接)
                 └──────┬──────┘
                        │
        ┌───────────────┼───────────────┐
        ↓               ↓               ↓
┌──────────────┐ ┌──────────┐ ┌──────────────┐
│   parser     │ │  binder  │ │   catalog    │
└──────┬───────┘ └────┬─────┘ └──────────────┘
       │              │
       ↓              ↓
┌──────────────┐ ┌──────────┐
│   planner    │ │optimizer │
└──────┬───────┘ └────┬─────┘
       │              │
       └──────┬───────┘
              ↓
     ┌──────────────┐
     │  execution   │
     └──────┬───────┘
            │
      ┌─────┴─────┐
      ↓           ↓
┌──────────┐ ┌──────────┐
│ storage  │ │transaction│
└──────────┘ └──────────┘
      ↑           ↑
      └─────┬─────┘
            │
      ┌──────────┐
      │  common  │  (所有模块依赖)
      └──────────┘
```

### 2.2 类型系统架构

#### 类型层次结构

```cpp
// src/include/duckdb/common/types.hpp

// 物理类型：实际存储格式
enum class PhysicalType : uint8_t {
    BOOL,       // 1 bit
    INT8,       // 1 byte
    INT16,      // 2 bytes
    INT32,      // 4 bytes
    INT64,      // 8 bytes
    UINT8,
    UINT16,
    UINT32,
    UINT64,
    INT128,     // 16 bytes
    FLOAT,      // 4 bytes
    DOUBLE,     // 8 bytes
    VARCHAR,    // 变长
    STRUCT,     // 复合类型
    LIST,       // 数组类型
    INVALID
};

// 逻辑类型：用户可见类型
class LogicalType {
public:
    LogicalTypeId id_;                    // 类型ID
    optional_idx width_;                  // VARCHAR宽度
    uint8_t scale_ = 0;                   // DECIMAL小数位数
    uint8_t type_scale_ = 0;
    LogicalType *collation_ = nullptr;    // 排序规则
    TypeInfo *type_info_ = nullptr;       // 扩展信息

    PhysicalType GetInternalType() const; // 转换为物理类型
};
```

#### 类型映射表

| 逻辑类型 | 物理类型 | 说明 |
|----------|----------|------|
| `BOOLEAN` | `BOOL` | 1位存储 |
| `TINYINT` | `INT8` | 有符号1字节 |
| `SMALLINT` | `INT16` | 有符号2字节 |
| `INTEGER` | `INT32` | 有符号4字节 |
| `BIGINT` | `INT64` | 有符号8字节 |
| `HUGEINT` | `INT128` | 有符号16字节 |
| `FLOAT` | `FLOAT` | 4字节浮点 |
| `DOUBLE` | `DOUBLE` | 8字节浮点 |
| `DECIMAL(s,p)` | `INT16/INT32/INT64` | 定点数存储 |
| `VARCHAR` | `VARCHAR` | 变长字符串 |
| `BLOB` | `VARCHAR` | 二进制数据 |
| `DATE` | `INT32` | 天数（距离1970-01-01） |
| `TIMESTAMP` | `INT64` | 微秒（距离1970-01-01） |
| `INTERVAL` | `INT64` | 微秒间隔 |
| `STRUCT` | `STRUCT` | 结构体 |
| `LIST` | `LIST` | 列表/数组 |

### 2.3 向量化执行架构

#### Vector类型系统

```cpp
// src/include/duckdb/common/types/vector.hpp

enum class VectorType : uint8_t {
    FLAT_VECTOR,           // 标准扁平向量
    CONSTANT_VECTOR,       // 常量向量（所有值相同）
    DICTIONARY_VECTOR,     // 字典向量（使用SelectionVector引用）
    SEQUENCE_VECTOR        // 序列向量（算术序列）
};

class Vector {
public:
    VectorType vector_type;           // 向量类型
    LogicalType type;                 // 数据类型
    data_ptr_t data;                  // 数据指针
    ValidityMask validity;            // NULL标记
    shared_ptr<VectorBuffer> buffer;  // 拥有的缓冲区
    shared_ptr<VectorBuffer> auxiliary; // 辅助数据（字符串字典等）

    // 切片操作（零拷贝）
    void Slice(Vector &other, SelectionVector &sel, idx_t count);
    void Slice(Vector &other, idx_t offset, idx_t count);

    // 引用操作（零拷贝）
    void Reference(Vector &other);
};
```

#### Dictionary Vector详解

```cpp
// DICTIONARY_VECTOR示例
// 避免过滤操作的数据拷贝

// 原始数据
Vector original(LogicalType::INTEGER);
int32_t *orig_data = FlatVector::GetData<int32_t>(original);
orig_data[0] = 10;
orig_data[1] = 20;
orig_data[2] = 30;
orig_data[3] = 40;

// 过滤条件: value > 25
SelectionVector sel(2);  // 结果有2行
sel.set_index(0, 2);     // 第一个结果来自原始索引2
sel.set_index(1, 3);     // 第二个结果来自原始索引3

// 创建字典向量（零拷贝）
Vector filtered(LogicalType::INTEGER);
filtered.Reference(original);
filtered.Slice(original, sel, 2);

// filtered.data = original.data (共享同一块内存)
// filtered.vector_type = DICTIONARY_VECTOR
// filtered.auxiliary = sel
```

#### Vector缓存友好设计

```cpp
// STANDARD_VECTOR_SIZE选择原因
constexpr idx_t STANDARD_VECTOR_SIZE = 2048;

// 考虑因素：
// 1. L1缓存大小（通常32-64KB）
//    - INTEGER列: 2048 * 4 bytes = 8KB
//    - 3列JOIN: 3 * 8KB = 24KB（适合L1）
//
// 2. SIMD指令宽度
//    - AVX-512: 16个32位整数
//    - 2048 / 16 = 128次SIMD迭代
//
// 3. 函数调用开销
//    - 每次处理2048行，减少虚函数调用
//
// 4. ValidityMask对齐
//    - 2048位 = 256字节，缓存行对齐
```

### 2.4 执行算子架构

#### PhysicalOperator基类

```cpp
// src/include/duckdb/execution/physical_operator.hpp

class PhysicalOperator {
public:
    PhysicalOperatorType type;           // 算子类型
    vector<unique_ptr<PhysicalOperator>> children;  // 子算子
    idx_t estimated_cardinality;         // 估计基数
    vector<LogicalType> types;           // 输出类型

    // 执行接口
    unique_ptr<OperatorState> GetOperatorState(ExecutionContext &context);
    unique_ptr<GlobalOperatorState> GetGlobalState(ClientContext &context);

    // Push-based执行
    bool Execute(ExecutionContext &context, DataChunk &input, DataChunk &output);
    OperatorResultType Execute(ExecutionContext &context,
                               DataChunk &input,
                               GlobalOperatorState &gstate,
                               OperatorState &state);

    // 源/汇接口
    bool IsSource() const;      // 数据源算子
    bool IsSink() const;        // 数据汇算子
    bool IsParallel() const;    // 并行算子
};
```

#### Pipeline执行模型

```cpp
// Push-based执行流程

// 1. 创建Pipeline
struct Pipeline {
    PhysicalOperator *sink;          // 终止算子
    vector<PhysicalOperator*> operators;  // 中间算子
    PhysicalOperator *source;        // 源算子

    // 执行Pipeline
    void Execute(ClientContext &context) {
        auto state = InitializeState();
        DataChunk input, output;

        // 从Source拉取数据
        while (source->Execute(context, input)) {
            // 依次通过中间算子
            for (auto &op : operators) {
                op->Execute(context, input, output, state);
                input = std::move(output);
            }

            // 推送到Sink
            sink->Execute(context, input, state);
        }
    }
};

// 示例: SELECT * FROM t WHERE x > 10
//
// Pipeline结构:
// ┌──────────────────────────────────┐
// │ Source: PhysicalTableScan        │ ← 拉取数据
// └────────────┬─────────────────────┘
//              ↓
// ┌──────────────────────────────────┐
// │ Filter: PhysicalFilter (x > 10) │ ← 过滤数据
// └────────────┬─────────────────────┘
//              ↓
// ┌──────────────────────────────────┐
// │ Sink: PhysicalMaterialize        │ ← 物化结果
// └──────────────────────────────────┘
```

#### 并行执行模型

```cpp
// src/execution/operator/scan/physical_table_scan.hpp

class PhysicalTableScan : public PhysicalOperator {
public:
    // 最大并行度
    idx_t MaxThreads(ClientContext &context) override {
        return table->GetTotalRows() / STANDARD_VECTOR_SIZE;
    }

    // 并行执行接口
    unique_ptr<GlobalSourceState> GetGlobalSourceState(ClientContext &context) override;
    unique_ptr<LocalSourceState> GetLocalSourceState(ExecutionContext &context,
                                                     GlobalSourceState &gstate) override;

    // 从特定分区读取
    SourceResultType GetData(ExecutionContext &context,
                            DataChunk &chunk,
                            OperatorState &state) override;
};

// 并行扫描示例
// 4个线程扫描100万行
// Thread 1: rows 0 - 250000
// Thread 2: rows 250000 - 500000
// Thread 3: rows 500000 - 750000
// Thread 4: rows 750000 - 1000000
```

---

## 第三部分：编译优化技巧

### 3.1 编译时优化

#### 优化标志配置

```bash
# 基础优化
CMAKE_BUILD_TYPE=Release           # -O3优化
CMAKE_CXX_FLAGS_RELEASE="-O3"      # 显式设置

# CPU特定优化
CMAKE_CXX_FLAGS="-march=native"    # 针对当前CPU优化
CMAKE_CXX_FLAGS="-mtune=native"    # 调优指令调度

# 链接时优化 (LTO)
CMAKE_INTERPROCEDURAL_OPTIMIZATION=TRUE

# Profile-Guided Optimization (PGO)
# 第一步：生成profile
make profgen
# 运行代表性 workload
./build/profile/duckdb < workload.sql
# 第二步：使用profile优化
make profuse
```

#### 优化级别对比

| 级别 | 编译时间 | 运行时性能 | 代码大小 | 调试友好度 |
|------|----------|------------|----------|------------|
| `-O0` | 最快 | 最慢 | 最大 | 最友好 |
| `-O1` | 较快 | 较慢 | 较大 | 友好 |
| `-O2` | 较慢 | 较快 | 较小 | 一般 |
| `-O3` | 最慢 | 最快 | 最小 | 不友好 |
| `-Os` | 较慢 | 中等 | 最小 | 一般 |
| `-Og` | 较慢 | 中等 | 较大 | 友好 |

### 3.2 模板元编程优化

#### 编译时多态

```cpp
// src/common/types/vector.hpp

// 模板函数在编译时为每种类型生成特化版本
template <class T>
static inline T *GetData(Vector &vector) {
    return (T *)vector.data;
}

// 使用示例
Vector int_vec(LogicalType::INTEGER);
auto int_data = FlatVector::GetData<int32_t>(int_vec);  // 编译时类型检查

Vector double_vec(LogicalType::DOUBLE);
auto double_data = FlatVector::GetData<double>(double_vec);
```

#### constexpr函数

```cpp
// src/include/duckdb/common/constants.hpp

// 编译时计算常量
constexpr idx_t STANDARD_VECTOR_SIZE = 2048;
constexpr idx_t BUFFER_BLOCK_SIZE = 262144;  // 256KB
constexpr idx_t ROW_GROUP_SIZE = 122880;     // 120,000行

// 编译时断言
static_assert(STANDARD_VECTOR_SIZE % 64 == 0,
              "Vector size must be multiple of 64 for alignment");

// 编译时类型检查
template <class T>
constexpr bool IsIntegralType() {
    return std::is_integral<T>::value;
}
```

#### 类型推导

```cpp
// 编译时选择最优实现
template <class T>
struct TypeTraits {
    static inline bool Equals(T a, T b) {
        if constexpr(std::is_floating_point<T>::value) {
            // 浮点数使用近似比较
            return std::abs(a - b) < 1e-9;
        } else {
            // 整数精确比较
            return a == b;
        }
    }
};
```

### 3.3 内存优化技巧

#### Arena分配器

```cpp
// src/include/duckdb/common/arena_allocator.hpp

class ArenaAllocator {
    vector<data_ptr_t> arenas;      // 大块内存
    idx_t current_position = 0;
    idx_t arena_size = 256 * 1024;  // 256KB per arena

public:
    // 快速分配（无需查找空闲块）
    template <class T, class... ARGS>
    T *Allocate(ARGS &&... args) {
        auto size = sizeof(T);
        auto aligned_size = AlignValue<idx_t>(size, 8);

        if (current_position + aligned_size > arena_size) {
            // 分配新arena
            arenas.push_back(AllocateArena(arena_size));
            current_position = 0;
        }

        auto ptr = arenas.back() + current_position;
        current_position += aligned_size;

        // 原位构造
        return new (ptr) T(std::forward<ARGS>(args)...);
    }

    // 析构时不释放单个对象
    // 一次性释放所有arena
    void Reset() {
        for (auto arena : arenas) {
            free(arena);
        }
        arenas.clear();
        current_position = 0;
    }
};

// 使用场景：查询执行期间的临时对象
class QueryContext {
    ArenaAllocator allocator;

public:
    // 一次分配，查询结束时统一释放
    DataChunk *AllocateChunk() {
        return allocator.Allocate<DataChunk>();
    }
};
```

#### 对象复用

```cpp
// 对象池模式
template <class T>
class ObjectPool {
    vector<unique_ptr<T>> pool;
    vector<T*> free_list;

public:
    T *Get() {
        if (!free_list.empty()) {
            auto obj = free_list.back();
            free_list.pop_back();
            return obj;
        }
        // 创建新对象
        auto obj = make_unique<T>();
        auto ptr = obj.get();
        pool.push_back(std::move(obj));
        return ptr;
    }

    void Return(T *obj) {
        obj->Reset();  // 重置状态
        free_list.push_back(obj);
    }
};

// 使用：Vector对象复用
class VectorCache {
    ObjectPool<Vector> pool;

public:
    Vector *GetVector(LogicalType type) {
        auto vec = pool.Get();
        vec->type = type;
        return vec;
    }

    void ReturnVector(Vector *vec) {
        pool.Return(vec);
    }
};
```

### 3.4 SIMD优化

#### 向量化操作

```cpp
// src/common/vector_operations/vector_operations.cpp

// 自动SIMD加速
void VectorOperations::Add(Vector &left, Vector &right, Vector &result, idx_t count) {
    // 编译器会自动向量化这个循环
    auto ldata = FlatVector::GetData<int32_t>(left);
    auto rdata = FlatVector::GetData<int32_t>(right);
    auto res_data = FlatVector::GetData<int32_t>(result);

    // 现代编译器会生成AVX2指令：
    // vpaddd ymm0, ymm1, ymm2  (一次处理8个int32)
    for (idx_t i = 0; i < count; i++) {
        res_data[i] = ldata[i] + rdata[i];
    }
}
```

#### 显式SIMD指令

```cpp
// 使用AVX2指令集
#include <immintrin.h>

void SIMDAdd(int32_t *a, int32_t *b, int32_t *c, idx_t count) {
    idx_t i = 0;

    // AVX2：一次处理8个int32
    for (; i + 8 <= count; i += 8) {
        __m256i va = _mm256_loadu_si256((__m256i*)&a[i]);
        __m256i vb = _mm256_loadu_si256((__m256i*)&b[i]);
        __m256i vc = _mm256_add_epi32(va, vb);
        _mm256_storeu_si256((__m256i*)&c[i], vc);
    }

    // 处理剩余元素
    for (; i < count; i++) {
        c[i] = a[i] + b[i];
    }
}

// 性能对比：
// - 标量循环: 8次迭代 + 8次加法 = 16条指令
// - AVX2循环: 1次迭代 + 1次向量加法 = 3条指令 (4.5x加速)
```

### 3.5 缓存优化

#### 数据布局优化

```cpp
// 缓存友好的数据结构

// ❌ 差：结构体数组
struct PersonAoD {
    vector<string> names;      // 缓存行1-3
    vector<int32_t> ages;      // 缓存行4
    vector<double> salaries;   // 缓存行5-6
};

// 访问模式：
// for (i = 0; i < n; i++) {
//     process(names[i], ages[i], salaries[i]);  // 3次缓存未命中
// }

// ✅ 好：数组结构体
struct PersonSoA {
    struct Person {
        string name;
        int32_t age;
        double salary;
    };
    vector<Person> persons;  // 连续内存
};

// 访问模式：
// for (i = 0; i < n; i++) {
//     process(persons[i]);  // 1次缓存未命中，后续访问命中L1
// }
```

#### 预取优化

```cpp
// 软件预取
void PrefetchOptimized(int32_t *a, int32_t *b, int32_t *c, idx_t count) {
    const idx_t PREFETCH_DISTANCE = 8;  // 预取距离

    for (idx_t i = 0; i < count; i++) {
        // 预取未来的数据
        if (i + PREFETCH_DISTANCE < count) {
            __builtin_prefetch(&a[i + PREFETCH_DISTANCE], 0, 3);
            __builtin_prefetch(&b[i + PREFETCH_DISTANCE], 0, 3);
        }

        // 处理当前数据（此时预取的数据正在加载）
        c[i] = a[i] + b[i];
    }
}

// 预取参数说明：
// - 第一个参数：地址
// - 第二个参数：0=读取, 1=写入
// - 第三个参数：临时性(0-3)，3=数据会被重用，留在L1缓存
```

#### 对齐优化

```cpp
// 缓存行对齐
struct alignas(64) CacheLineAligned {
    int32_t data[16];  // 64字节，一个缓存行
};

// 避免伪共享
class Counter {
    alignas(64) atomic<uint64_t> local_count;  // 每个线程独立缓存行

public:
    void Increment() {
        local_count++;  // 不会与其他线程的缓存行冲突
    }
};
```

### 3.6 查询优化技巧

#### Filter Pushdown

```cpp
// 优化前：全表扫描后过滤
SELECT name, score
FROM students
WHERE score > 90;

LogicalPlan:
┌─────────────────────┐
│  LogicalProjection  │  处理所有行
│  (name, score)      │
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│   LogicalFilter     │  处理所有行
│   (score > 90)      │
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│   LogicalGet        │  读取所有列
│   (students)        │
└─────────────────────┘

// 优化后：过滤下推到扫描
LogicalPlan:
┌─────────────────────┐
│  LogicalProjection  │  只处理过滤后的行
│  (name, score)      │
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│   LogicalGet        │  只读取需要的列，应用过滤
│   (name, score)     │  + TableFilterSet
└─────────────────────┘

// 性能提升：
// - 减少I/O：只读取name和score列
// - 减少计算：Filter在C++层执行，避免表达式开销
// - 提前终止：满足LIMIT即可停止扫描
```

#### Join Order优化

```cpp
// 动态规划选择最优Join顺序

// 小表驱动大表
SELECT *
FROM orders (1M行) o
JOIN customers (100K行) c ON o.customer_id = c.id
JOIN products (10K行) p ON o.product_id = p.id;

// ❌ 差的顺序：
// orders × customers × products
// 中间结果: 1M × 100K = 100B行

// ✅ 好的顺序：
// products × customers × orders
// 1. products × customers = 10K × 100K = 1B行（但很多不匹配，实际很少）
// 2. 结果 × orders ≈ 1M行

// Join Order Optimizer使用代价模型：
// cost(Join A, B) = cardinality(A) × cardinality(B) × selectivity
// 最小化总代价
```

#### 列裁剪

```cpp
// 只读取需要的列
SELECT name FROM students;

// 解析阶段识别只需要name列
LogicalGet:
├── table: students
└── column_ids: [1]  // 只读取第1列(name)

// 存储层读取：
for (auto &row_group : table.row_groups) {
    // 只解压name列
    auto name_column = row_group.columns[1];
    name_column.Read(chunk.data[0]);
}
```

---

## 第四部分：性能分析与调优

### 4.1 性能分析工具

#### perf（Linux）

```bash
# CPU性能分析
perf record -F 99 -g ./build/release/duckdb < workload.sql
perf report

# 火焰图生成
perf script | stackcollapse-perf.pl | flamegraph.pl > flamegraph.svg

# 热点函数分析
perf top -p $(pidof duckdb)

# 缓存未命中分析
perf stat -e cache-misses,cache-references ./build/release/duckdb < workload.sql
```

#### Instruments（macOS）

```bash
# Time Profiler
instruments -t "Time Profiler" ./build/release/duckdb < workload.sql

# Allocations（内存分配）
instruments -t "Allocations" ./build/release/duckdb < workload.sql

# System Trace（系统调用）
instruments -t "System Trace" ./build/release/duckdb < workload.sql
```

#### Valgrind（内存分析）

```bash
# 缓存模拟
valgrind --tool=cachegrind ./build/debug/duckdb < workload.sql
cg_annotate cachegrind.out.<pid>

# 内存泄漏检测
valgrind --leak-check=full ./build/debug/duckdb < workload.sql

# 堆分析
valgrind --tool=massif ./build/debug/duckdb < workload.sql
ms_print massif.out.<pid>
```

### 4.2 查询性能分析

#### EXPLAIN ANALYZE

```sql
-- 查看查询执行计划和实际性能
EXPLAIN ANALYZE
SELECT customer_id, SUM(amount)
FROM orders
WHERE order_date > '2024-01-01'
GROUP BY customer_id;

-- 输出示例：
┌────────────────────────────────────┐
││ Query Plan                        │
├────────────────────────────────────┤
││  AGGREGATE                         │
││  • Group By: customer_id           │
││  • Aggregates: sum(amount)         │
││  • Execution Time: 152.3ms        │
││  →  FILTER                         │
││     • Filter: order_date > ...    │
││     • Execution Time: 89.1ms      │
││     →  SEQ_SCAN                    │
││        • Table: orders            │
││        • Rows Scanned: 1,000,000  │
││        • Rows Filtered: 50,000    │
││        • Execution Time: 12.4ms   │
└────────────────────────────────────┘

-- 总执行时间：152.3ms
-- 各阶段耗时分布
-- 滤波器选择性：5% (50K/1M)
```

#### PRAGMA优化选项

```sql
-- 启用查询profiling
PRAGMA enable_profiling;
PRAGMA profiling_output = 'query_profile.json';

-- 运行查询后查看详细profile
SELECT * FROM large_table WHERE condition;

-- 查看profile
cat query_profile.json | jq '.timing'

-- 内存使用分析
PRAGMA memory_limit = '2GB';
PRAGMA enable_object_cache = true;

-- 并行度设置
PRAGMA threads = 8;
PRAGMA max_execution_time = 60000;  -- 60秒超时
```

### 4.3 常见性能问题与解决

#### 问题1：查询缓慢

**诊断步骤：**

```sql
-- 1. 检查执行计划
EXPLAIN SELECT ...

-- 2. 分析各阶段耗时
EXPLAIN ANALYZE SELECT ...

-- 3. 查看统计信息
PRAGMA database_size;
SELECT * FROM duckdb_tables();

-- 4. 检查是否有合适的索引
-- DuckDB主要使用列存储，较少使用传统索引
```

**解决方案：**

```sql
-- 方案1：创建列存储表（自动）
CREATE TABLE orders AS
SELECT * FROM read_csv('orders.csv');

-- 方案2：使用Hive分区
CREATE TABLE orders_partitioned (
    order_date DATE,
    ...
) PARTITION BY (year, month);

-- 方案3：调整内存限制
PRAGMA memory_limit = '4GB';

-- 方案4：增加并行度
PRAGMA threads = 16;
```

#### 问题2：内存占用过高

**诊断：**

```bash
# 运行时监控内存
watch -n 1 'ps aux | grep duckdb'

# 或使用DuckDB内置监控
PRAGMA enable_progress_bar;
```

**解决方案：**

```sql
-- 方案1：限制内存使用
PRAGMA memory_limit = '1GB';

-- 方案2：使用流式处理
-- 避免一次性加载所有数据
COPY (SELECT * FROM large_table) TO 'output.csv'
    (FORMAT CSV, DELIMITER ',');

-- 方案3：分批处理
CREATE OR REPLACE MACRO batch_process(batch_size) AS (
    SELECT *
    FROM large_table
    LIMIT batch_size
    OFFSET (SELECT batch_number * batch_size FROM batch_state)
);

-- 方案4：释放未使用的缓存
PRAGMA force_checkpoint;  -- 写入WAL
-- 重启连接
```

#### 问题3：JOIN性能差

**诊断：**

```sql
-- 检查Join顺序
EXPLAIN SELECT * FROM a JOIN b ON a.id = b.id;

-- 查看表大小
SELECT
    schema_name,
    table_name,
    estimated_size
FROM duckdb_tables();
```

**解决方案：**

```sql
-- 方案1：确保统计信息准确
ANALYZE table_name;

-- 方案2：显式指定Join顺序
SELECT /*+ HASH_JOIN(a, b) */
    *
FROM a
JOIN b ON a.id = b.id;

-- 方案3：创建合适的列存储
-- DuckDB会自动选择最优Join算法（Hash Join vs Sort-Merge Join）
```

### 4.4 高级性能调优

#### 自定义优化规则

```cpp
// 添加自定义优化规则

// src/optimizer/optimizer.cpp
void Optimizer::AddCustomRule(unique_ptr<OptimizerRule> rule) {
    rules.push_back(std::move(rule));
}

// 示例：常量表达式预计算
class ConstantFoldingRule : public OptimRule {
public:
    unique_ptr<LogicalOperator> Apply(LogicalOperator *op) override {
        // 遍历表达式树
        for (auto &expr : op->expressions) {
            if (TryFoldConstant(expr)) {
                // 替换为常量
                expr = make_unique<BoundConstantExpression>(
                    EvaluateConstant(expr)
                );
            }
        }
        return op->ToString();
    }
};
```

#### JIT编译

```cpp
// DuckDB使用LLVM进行JIT编译（实验性功能）

// 启用JIT
PRAGMA enable_jit = true;

// JIT会热编译高频表达式
-- 示例：复杂的用户定义函数
CREATE FUNCTION my_complex_func(x DOUBLE) AS (
    x * x + 2 * x + 1
);

-- JIT会将此函数编译为机器码
SELECT my_complex_func(value) FROM large_table;
```

---

## 第五部分：最佳实践总结

### 5.1 编译最佳实践

```bash
# 日常开发
make debug

# 性能测试
GEN=ninja make release

# CI/CD
DISABLE_SANITIZER=1 GEN=ninja make release

# 生产部署
CMAKE_BUILD_TYPE=Release \
CMAKE_CXX_FLAGS_RELEASE="-O3 -march=x86-64-v3" \
make
```

### 5.2 代码优化Checklist

- [ ] **内存访问模式**
  - 连续内存访问
  - 避免随机访问
  - 使用预取

- [ ] **向量化**
  - 批量处理数据
  - 使用Vector/DataChunk
  - 避免逐行处理

- [ ] **零拷贝**
  - 使用引用而非深拷贝
  - Slice操作
  - Dictionary Vector

- [ ] **编译时优化**
  - constexpr函数
  - 模板元编程
  - 内联小函数

- [ ] **并行化**
  - 多线程Pipeline执行
  - 并行表扫描
  - 并行聚合

### 5.3 性能优化流程

```
1. 基准测试
   ↓
2. 性能分析
   ↓
3. 瓶颈识别
   ↓
4. 优化实现
   ↓
5. 验证效果
   ↓
6. 回到步骤2（持续迭代）
```

### 5.4 学习资源

**推荐阅读：**

1. **论文**
   - "MonetDB/X100: Hyper-Pipelining Query Execution"
   - "Push-Based Execution in DuckDB"
   - "Morsel-Driven Parallelism"

2. **代码位置**
   - CMakeLists.txt：构建系统
   - src/optimizer/：优化器
   - src/execution/：执行引擎
   - src/common/vector_operations/：向量化操作

3. **工具**
   - perf、Instruments、Valgrind
   - DuckDB的EXPLAIN ANALYZE
   - PRAGMA enable_profiling

---

**下一步学习：**

1. 阅读DuckDB源码中的优化器实现
2. 实现一个自定义优化规则
3. 使用perf分析查询性能
4. 尝试编写SIMD优化的向量操作

**相关文档：**
- [DuckDB 30天学习课程](./DuckDB_30天学习课程.md)
- [DuckDB调试指南](./DuckDB调试指南.md)
- [DuckDB学习速查表](./DuckDB学习速查表.md)

---

最后更新：2026-01-23
