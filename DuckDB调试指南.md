# 🐛 DuckDB 调试与故障排除指南

完整的DuckDB开发调试技巧、常见问题解决方案和性能分析方法。

---

## 📋 目录

1. [编译调试版本](#编译调试版本)
2. [使用GDB调试](#使用gdb调试)
3. [使用LLDB调试](#使用lldb调试)
4. [日志和打印调试](#日志和打印调试)
5. [内存调试](#内存调试)
6. [性能分析](#性能分析)
7. [常见错误和解决方案](#常见错误和解决方案)
8. [单元测试调试](#单元测试调试)
9. [查询执行调试](#查询执行调试)
10. [存储引擎调试](#存储引擎调试)

---

## 🔨 编译调试版本

### Debug构建

```bash
# 基础debug构建
make debug

# 使用Ninja加速构建
GEN=ninja make debug

# 清理后重新构建
make clean
make debug

# 只构建特定目标
cd build/debug
ninja duckdb        # 构建CLI
ninja unittest      # 构建单元测试
ninja duckdb_test   # 构建特定测试
```

---

### 调试选项

在`build/debug/CMakeCache.txt`中查看/修改构建选项：

```cmake
# 关键调试选项
CMAKE_BUILD_TYPE:STRING=Debug
ENABLE_SANITIZER:BOOL=ON          # 启用地址/UB sanitizer
DISABLE_ASSERTIONS:BOOL=OFF       # 保持断言开启
BUILD_UNITTESTS:BOOL=ON           # 构建单元测试
```

---

### Sanitizers

**Address Sanitizer (检测内存错误):**
```bash
# 自动启用在debug构建中
make debug

# 运行时
./build/debug/duckdb
# 如果有内存错误，会打印详细堆栈
```

**常见ASan错误:**
- `heap-use-after-free`: 使用已释放的内存
- `heap-buffer-overflow`: 数组越界
- `stack-use-after-scope`: 使用超出作用域的栈变量
- `memory leak`: 内存泄漏

---

**Undefined Behavior Sanitizer:**
```bash
# 在CMake中启用
cmake -DENABLE_UBSAN=1 ..

# 检测未定义行为
# - 整数溢出
# - 空指针解引用
# - 违反对齐要求
```

---

## 🔍 使用GDB调试

### 启动GDB

```bash
# 调试DuckDB CLI
gdb ./build/debug/duckdb

# 调试单元测试
gdb ./build/debug/test/unittest

# 带参数启动
gdb --args ./build/debug/duckdb mydb.duckdb

# 附加到运行中的进程
gdb -p <pid>
```

---

### 基本GDB命令

```gdb
# 设置断点
break main                          # 在main函数设置断点
break vector.cpp:123                # 在文件特定行设置断点
break Vector::Slice                 # 在成员函数设置断点
break duckdb::PhysicalHashJoin::Execute  # 完整命名空间

# 条件断点
break vector.cpp:123 if count > 100

# 运行
run                                 # 开始执行
run arg1 arg2                       # 带参数运行

# 执行控制
continue (c)                        # 继续执行
step (s)                            # 单步进入
next (n)                            # 单步跳过
finish                              # 执行到当前函数返回
until 150                           # 执行到第150行

# 查看信息
backtrace (bt)                      # 查看调用栈
frame 3                             # 切换到第3帧
info locals                         # 查看局部变量
info args                           # 查看函数参数
info breakpoints                    # 查看所有断点

# 打印变量
print vec                           # 打印变量
print vec.count                     # 打印成员
print *ptr                          # 解引用指针
print vec.data[0]@10                # 打印数组前10个元素

# 监视变量
watch count                         # 当count变化时中断
watch *0x12345678                   # 监视内存地址

# 删除断点
delete 1                            # 删除断点1
clear vector.cpp:123                # 清除特定位置的断点
disable 2                           # 禁用断点2
enable 2                            # 启用断点2
```

---

### DuckDB特定调试技巧

**调试Vector操作:**

```gdb
# 在Vector::Slice设置断点
break Vector::Slice

# 运行到断点
run

# 查看Vector内容
print this->type               # LogicalType
print this->count              # 元素数量
print this->vector_type        # VectorType (FLAT, CONSTANT, etc.)

# 查看FlatVector数据
print ((int32_t*)this->data)[0]@10   # 打印前10个int32元素

# 查看ValidityMask
print this->validity
print this->validity.validity_mask
```

---

**调试DataChunk:**

```gdb
break DataChunk::Append

# 查看DataChunk信息
print this->count               # 行数
print this->data.size()         # 列数

# 遍历所有列
set $i = 0
while $i < this->data.size()
    print this->data[$i]
    set $i = $i + 1
end

# 打印特定列的数据
print ((int32_t*)this->data[0].data)[0]@this->count
```

---

**调试物理算子:**

```gdb
# 在算子执行时中断
break PhysicalHashJoin::GetData

# 查看算子状态
print this->type
print this->children.size()

# 查看执行上下文
print context.client.interrupted
print context.thread.profiler
```

---

### GDB Python扩展

创建`.gdbinit`文件以自定义调试命令：

```python
# .gdbinit
define print_vector
    set $vec = (duckdb::Vector*)$arg0
    printf "Vector: type=%d, count=%lld\n", $vec->type, $vec->count
    if $vec->vector_type == 0
        # FLAT vector
        set $i = 0
        while $i < $vec->count
            printf "[%d] = %d\n", $i, ((int32_t*)$vec->data)[$i]
            set $i = $i + 1
        end
    end
end

define print_chunk
    set $chunk = (duckdb::DataChunk*)$arg0
    printf "DataChunk: %lld rows, %lld columns\n", $chunk->count, $chunk->data.size()
end
```

使用：
```gdb
print_vector &my_vector
print_chunk &my_chunk
```

---

## 🍎 使用LLDB调试 (macOS)

### 基本LLDB命令

```lldb
# 启动
lldb ./build/debug/duckdb

# 设置断点
breakpoint set --name main
breakpoint set --file vector.cpp --line 123
breakpoint set --method Slice
b Vector::Slice                     # 简写

# 运行
run
r                                   # 简写

# 执行控制
continue (c)
step (s)
next (n)
finish

# 查看信息
bt                                  # 调用栈
frame select 3                      # 选择帧
frame variable                      # 查看变量
p vec                               # 打印变量

# 监视
watchpoint set variable count
watchpoint set expression -- &vec->data

# 断点管理
breakpoint list
breakpoint delete 1
breakpoint disable 2
```

---

## 📝 日志和打印调试

### 基础打印调试

```cpp
// 包含iostream
#include <iostream>

// 打印基本类型
std::cout << "count = " << count << std::endl;

// 打印Vector
void DebugPrintVector(Vector &vec, idx_t count) {
    std::cout << "Vector [type=" << vec.GetType().ToString()
              << ", count=" << count << "]" << std::endl;

    if (vec.GetVectorType() == VectorType::FLAT_VECTOR) {
        auto data = FlatVector::GetData<int32_t>(vec);
        auto &validity = FlatVector::Validity(vec);

        for (idx_t i = 0; i < count; i++) {
            if (validity.RowIsValid(i)) {
                std::cout << "  [" << i << "] = " << data[i] << std::endl;
            } else {
                std::cout << "  [" << i << "] = NULL" << std::endl;
            }
        }
    }
}

// 使用DuckDB内置打印
vec.Print(count);           // 打印Vector
chunk.Print();              // 打印DataChunk
val.Print();                // 打印Value
```

---

### 使用D_ASSERT调试

```cpp
// DuckDB断言（只在Debug构建中生效）
D_ASSERT(count <= STANDARD_VECTOR_SIZE);
D_ASSERT(chunk.ColumnCount() == expected_cols);
D_ASSERT(!vec.GetType().IsInvalid());

// 带消息的断言
#define D_ASSERT_MSG(condition, message) \
    D_ASSERT((condition) || (std::cerr << message << std::endl, false))

D_ASSERT_MSG(idx < count, "Index out of bounds: " << idx << " >= " << count);
```

---

### 条件编译调试代码

```cpp
// 只在Debug构建中编译
#ifndef NDEBUG
    std::cout << "Debug: processing " << count << " rows" << std::endl;
    chunk.Print();
#endif

// 使用自定义宏
#ifdef DUCKDB_DEBUG_VECTOR
    DebugPrintVector(vec, count);
#endif
```

---

### 日志到文件

```cpp
#include <fstream>

class DebugLogger {
    std::ofstream log_file;
public:
    DebugLogger() : log_file("debug.log", std::ios::app) {}

    template<typename T>
    void Log(const std::string &msg, const T &value) {
        log_file << msg << ": " << value << std::endl;
    }

    void LogChunk(const DataChunk &chunk) {
        log_file << "DataChunk: " << chunk.size() << " rows, "
                 << chunk.ColumnCount() << " cols" << std::endl;
    }
};

// 使用
DebugLogger logger;
logger.Log("Processing chunk", chunk_id);
logger.LogChunk(chunk);
```

---

## 🧠 内存调试

### 使用Valgrind

```bash
# 安装Valgrind
sudo apt-get install valgrind   # Linux
brew install valgrind            # macOS (需要特殊版本)

# 内存泄漏检测
valgrind --leak-check=full \
         --show-leak-kinds=all \
         --track-origins=yes \
         --verbose \
         ./build/debug/duckdb

# 保存日志
valgrind --leak-check=full \
         --log-file=valgrind.log \
         ./build/debug/duckdb

# 针对特定测试
valgrind --leak-check=full ./build/debug/test/unittest "[vector]"
```

---

### 内存泄漏检测

**常见内存泄漏模式:**

```cpp
// ❌ 错误：忘记释放
Vector *vec = new Vector(LogicalType::INTEGER, 100);
// ... 使用vec
// 忘记 delete vec;

// ✅ 正确：使用智能指针
auto vec = make_uniq<Vector>(LogicalType::INTEGER, 100);
// 自动释放

// ❌ 错误：原始指针管理
data_ptr_t ptr = (data_ptr_t)malloc(size);
// ... 可能在某些路径忘记 free(ptr)

// ✅ 正确：使用BufferManager
auto handle = buffer_manager.Allocate(size);
data_ptr_t ptr = handle.Ptr();
// handle析构时自动释放
```

---

### Address Sanitizer输出解读

```
=================================================================
==12345==ERROR: AddressSanitizer: heap-use-after-free on address 0x12345678
READ of size 4 at 0x12345678 thread T0
    #0 Vector::GetData() vector.cpp:123
    #1 PhysicalFilter::Execute() physical_filter.cpp:45
    #2 Pipeline::Execute() pipeline.cpp:78

0x12345678 is located 8 bytes inside of 2048-byte region freed by thread T0
    #0 operator delete
    #1 Vector::~Vector() vector.cpp:89
    #2 DataChunk::Reset() data_chunk.cpp:56
```

**解读:**
- 在`vector.cpp:123`尝试读取已释放的内存
- 该内存在`data_chunk.cpp:56`的`Reset()`中被释放
- 需要检查Vector的生命周期管理

---

## ⚡ 性能分析

### 使用perf (Linux)

```bash
# 安装perf
sudo apt-get install linux-tools-common linux-tools-generic

# 记录性能数据
perf record -g ./build/release/duckdb < query.sql

# 查看报告
perf report

# 针对特定函数
perf record -g -e cpu-clock -F 999 ./build/release/duckdb

# 生成火焰图
perf script | stackcollapse-perf.pl | flamegraph.pl > flamegraph.svg
```

---

### 使用Instruments (macOS)

```bash
# 使用Xcode的Instruments
instruments -t "Time Profiler" ./build/release/duckdb

# 或者在Xcode中：
# Product -> Profile (Cmd+I)
```

---

### DuckDB内置性能分析

```sql
-- 查看执行计划
EXPLAIN SELECT * FROM table WHERE x > 10;

-- 查看执行统计
EXPLAIN ANALYZE SELECT * FROM table WHERE x > 10;

-- 启用Profiler
PRAGMA enable_profiling;
PRAGMA profiling_output='profile.json';

SELECT * FROM large_table WHERE ...;

-- 查看profile结果
-- 在profile.json中查看详细时间分布
```

---

### C++ Profiler集成

```cpp
#include "duckdb/main/profiler.hpp"

// 在代码中手动计时
auto start = std::chrono::high_resolution_clock::now();

// ... 需要测量的代码

auto end = std::chrono::high_resolution_clock::now();
auto duration = std::chrono::duration_cast<std::chrono::microseconds>(end - start);

std::cout << "Elapsed: " << duration.count() << " us" << std::endl;
```

---

### 性能热点识别

```cpp
// 使用Profiler API
class TimedSection {
    std::string name;
    std::chrono::time_point<std::chrono::high_resolution_clock> start;
public:
    TimedSection(const std::string &name) : name(name) {
        start = std::chrono::high_resolution_clock::now();
    }

    ~TimedSection() {
        auto end = std::chrono::high_resolution_clock::now();
        auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
        std::cout << name << ": " << duration.count() << " ms" << std::endl;
    }
};

// 使用
void PhysicalHashJoin::Execute() {
    TimedSection timer("HashJoin::Execute");
    // ... 函数体
}
```

---

## ❗ 常见错误和解决方案

### 1. Segmentation Fault

**症状:**
```
Segmentation fault (core dumped)
```

**调试步骤:**
```bash
# 启用core dump
ulimit -c unlimited

# 运行程序触发错误
./build/debug/duckdb

# 用GDB查看core dump
gdb ./build/debug/duckdb core

# 在GDB中
(gdb) bt              # 查看崩溃位置
(gdb) frame 0         # 查看崩溃帧
(gdb) info locals    # 查看局部变量
```

**常见原因:**
- 空指针解引用
- 数组越界
- 使用已释放的内存
- 栈溢出

---

### 2. Assertion Failed

**症状:**
```
Assertion failed: (count <= STANDARD_VECTOR_SIZE), function Append, file vector.cpp, line 123
```

**解决方法:**
1. 查看断言条件：`count <= STANDARD_VECTOR_SIZE`
2. 检查`count`的值是否超出范围
3. 回溯调用栈找到`count`被设置的位置
4. 修复逻辑错误

**在GDB中:**
```gdb
# 在断言处中断
break __assert_fail

# 运行到断言
run

# 查看变量
print count
print STANDARD_VECTOR_SIZE

# 回溯调用
bt
```

---

### 3. Invalid Memory Access

**症状 (Address Sanitizer):**
```
==12345==ERROR: AddressSanitizer: heap-buffer-overflow
```

**调试:**
```bash
# 用ASan运行
./build/debug/duckdb

# ASan会打印:
# - 错误类型 (heap-buffer-overflow)
# - 访问位置 (读/写, 地址)
# - 分配位置
# - 调用栈
```

**修复方向:**
- 检查数组索引是否正确
- 验证循环边界
- 确保分配了足够的内存

---

### 4. Type Mismatch

**症状:**
```
Error: Cannot compare values of type INTEGER and VARCHAR
```

**调试:**
```cpp
// 打印类型信息
std::cout << "Left type: " << left_type.ToString() << std::endl;
std::cout << "Right type: " << right_type.ToString() << std::endl;

// 检查类型兼容性
if (left_type != right_type) {
    // 需要类型转换
    Value casted = left_val.DefaultCastAs(right_type);
}
```

---

### 5. Memory Leak

**症状 (Valgrind):**
```
==12345== 2048 bytes in 1 blocks are definitely lost
```

**定位泄漏:**
```bash
valgrind --leak-check=full --show-leak-kinds=all ./build/debug/duckdb
```

**常见泄漏模式:**
```cpp
// ❌ new without delete
Vector *vec = new Vector(...);

// ❌ malloc without free
data_ptr_t ptr = (data_ptr_t)malloc(size);

// ❌ 循环引用 (智能指针)
shared_ptr<Node> a = make_shared<Node>();
shared_ptr<Node> b = make_shared<Node>();
a->next = b;
b->prev = a;  // 循环引用

// ✅ 使用unique_ptr
auto vec = make_uniq<Vector>(...);
```

---

## 🧪 单元测试调试

### 运行特定测试

```bash
# 运行所有测试
./build/debug/test/unittest

# 运行特定标签的测试
./build/debug/test/unittest "[vector]"

# 运行特定测试名称
./build/debug/test/unittest "Vector Operations"

# 列出所有测试
./build/debug/test/unittest --list-tests
```

---

### 在GDB中调试测试

```bash
# 启动GDB
gdb ./build/debug/test/unittest

# 设置断点
break vector_operations.test.cpp:45

# 运行特定测试
run "[vector]"

# 或者运行所有测试
run
```

---

### 编写调试友好的测试

```cpp
TEST_CASE("Vector slicing", "[vector]") {
    Vector input(LogicalType::INTEGER, 10);
    auto data = FlatVector::GetData<int32_t>(input);

    // 填充测试数据
    for (idx_t i = 0; i < 10; i++) {
        data[i] = i;
    }

    // 调试打印
    #ifndef NDEBUG
        std::cout << "Input vector:" << std::endl;
        input.Print(10);
    #endif

    // 执行切片
    SelectionVector sel(4);
    for (idx_t i = 0; i < 4; i++) {
        sel.set_index(i, 3 + i);
    }

    Vector result(LogicalType::INTEGER, 4);
    result.Slice(input, sel, 4);

    // 调试打印
    #ifndef NDEBUG
        std::cout << "Result vector:" << std::endl;
        result.Print(4);
    #endif

    // 验证
    auto result_data = FlatVector::GetData<int32_t>(result);
    REQUIRE(result_data[0] == 3);
    REQUIRE(result_data[1] == 4);
    REQUIRE(result_data[2] == 5);
    REQUIRE(result_data[3] == 6);
}
```

---

## 🔎 查询执行调试

### 查看执行计划

```sql
-- 逻辑计划
EXPLAIN SELECT * FROM table1 JOIN table2 ON table1.id = table2.id;

-- 物理计划
EXPLAIN SELECT * FROM table1 JOIN table2 ON table1.id = table2.id;

-- 带执行统计
EXPLAIN ANALYZE SELECT * FROM table1 JOIN table2 ON table1.id = table2.id;
```

---

### 调试算子执行

```cpp
// 在PhysicalOperator中添加调试代码
OperatorResultType PhysicalFilter::Execute(
    ExecutionContext &context,
    DataChunk &input,
    DataChunk &chunk,
    GlobalOperatorState &gstate,
    OperatorState &state) const {

    #ifndef NDEBUG
        std::cout << "PhysicalFilter::Execute - input rows: "
                  << input.size() << std::endl;
    #endif

    // 执行过滤逻辑
    SelectionVector sel(STANDARD_VECTOR_SIZE);
    idx_t approved_count = executor.SelectExpression(input, sel);

    #ifndef NDEBUG
        std::cout << "PhysicalFilter::Execute - output rows: "
                  << approved_count << std::endl;
    #endif

    chunk.Slice(input, sel, approved_count);
    chunk.SetCardinality(approved_count);

    return OperatorResultType::NEED_MORE_INPUT;
}
```

---

### 使用Profiler

```cpp
// 启用profiler
Connection con(db);
con.Query("PRAGMA enable_profiling='json'");
con.Query("PRAGMA profiling_output='profile.json'");

// 执行查询
auto result = con.Query("SELECT ...");

// 查看profile.json
```

---

## 💾 存储引擎调试

### 调试数据写入

```cpp
void RowGroup::Append(DataChunk &chunk) {
    #ifndef NDEBUG
        std::cout << "RowGroup::Append - chunk size: " << chunk.size()
                  << ", current row_count: " << row_count << std::endl;
    #endif

    D_ASSERT(chunk.size() > 0);
    D_ASSERT(row_count + chunk.size() <= Storage::ROW_GROUP_SIZE);

    for (idx_t col = 0; col < chunk.ColumnCount(); col++) {
        #ifndef NDEBUG
            std::cout << "  Appending column " << col << std::endl;
        #endif

        columns[col]->Append(chunk.data[col], chunk.size());
    }

    row_count += chunk.size();

    #ifndef NDEBUG
        std::cout << "RowGroup::Append - new row_count: " << row_count << std::endl;
    #endif
}
```

---

### 验证存储正确性

```cpp
// 写入后立即读取验证
void TestStorageCorrectness() {
    // 创建并写入数据
    DataChunk write_chunk;
    // ... 填充 write_chunk

    row_group.Append(write_chunk);

    // 立即读取
    DataChunk read_chunk;
    row_group.Scan(read_chunk, 0, write_chunk.size());

    // 逐列验证
    for (idx_t col = 0; col < write_chunk.ColumnCount(); col++) {
        auto write_data = FlatVector::GetData<int32_t>(write_chunk.data[col]);
        auto read_data = FlatVector::GetData<int32_t>(read_chunk.data[col]);

        for (idx_t row = 0; row < write_chunk.size(); row++) {
            if (write_data[row] != read_data[row]) {
                std::cerr << "Mismatch at col=" << col << ", row=" << row
                          << ": wrote " << write_data[row]
                          << ", read " << read_data[row] << std::endl;
                D_ASSERT(false);
            }
        }
    }

    std::cout << "Storage correctness test passed!" << std::endl;
}
```

---

## 📊 性能问题诊断

### Checklist

1. **查询计划是否优化?**
   ```sql
   EXPLAIN ANALYZE SELECT ...;
   ```
   检查是否有：
   - Filter Pushdown
   - Join Order优化
   - 索引使用

2. **是否有全表扫描?**
   ```sql
   -- 应该避免
   SELECT * FROM huge_table WHERE rarely_true_condition;

   -- 创建索引或重构查询
   ```

3. **向量化是否有效?**
   ```cpp
   // ❌ 逐行处理
   for (idx_t i = 0; i < count; i++) {
       result_data[i] = input_data[i] * 2;
   }

   // ✅ 向量化
   VectorOperations::Multiply(input, constant_2, result, count);
   ```

4. **内存使用是否合理?**
   ```bash
   # 监控内存使用
   /usr/bin/time -v ./build/release/duckdb < query.sql
   ```

5. **是否有不必要的数据拷贝?**
   ```cpp
   // ❌ 不必要的拷贝
   DataChunk temp;
   input.Copy(temp);
   ProcessChunk(temp);

   // ✅ 直接处理
   ProcessChunk(input);

   // 或使用引用
   ProcessChunk(input);  // input作为const引用
   ```

---

## 💡 调试技巧总结

### Do's ✅

- ✅ 使用Debug构建进行开发
- ✅ 启用Address Sanitizer
- ✅ 编写单元测试验证每个函数
- ✅ 使用断言检查前置条件和后置条件
- ✅ 在复杂逻辑中添加调试打印
- ✅ 使用GDB/LLDB单步调试
- ✅ 定期使用Valgrind检查内存泄漏
- ✅ 使用perf/Instruments进行性能分析
- ✅ 查看EXPLAIN ANALYZE了解查询执行

### Don'ts ❌

- ❌ 在Release构建中调试（断言被禁用）
- ❌ 忽略编译器警告
- ❌ 使用printf调试后忘记删除
- ❌ 不写测试就修改代码
- ❌ 盲目优化（先测量再优化）
- ❌ 忽略ASan/Valgrind的报告
- ❌ 提交包含调试代码的PR

---

## 📚 参考资源

- **GDB教程:** https://sourceware.org/gdb/documentation/
- **LLDB教程:** https://lldb.llvm.org/use/tutorial.html
- **Valgrind手册:** https://valgrind.org/docs/manual/manual.html
- **Perf Wiki:** https://perf.wiki.kernel.org/
- **ASan文档:** https://github.com/google/sanitizers/wiki/AddressSanitizer

---

**祝调试顺利！Remember: 调试是一门艺术，需要耐心和经验。🐛**

---

最后更新：2026-01-17
