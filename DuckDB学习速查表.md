# 🚀 DuckDB 学习速查表

快速参考DuckDB核心概念、API和常用代码模式。

---

## 📊 核心数据结构速查

### Vector - 列向量

```cpp
// 创建Vector
Vector vec(LogicalType::INTEGER, 2048);

// 获取数据指针
auto data = FlatVector::GetData<int32_t>(vec);

// 访问/修改数据
data[i] = value;

// 获取Validity Mask
auto &validity = FlatVector::Validity(vec);
validity.SetInvalid(i);  // 设置为NULL
bool is_valid = validity.RowIsValid(i);

// 设置Vector大小
vec.SetVectorSize(count);
```

**常用方法：**
- `vec.GetType()` - 获取逻辑类型
- `vec.GetVectorSize()` - 获取大小
- `vec.Slice(source, sel, count)` - 切片操作
- `vec.Flatten(count)` - 展平为FlatVector

---

### DataChunk - 数据批次

```cpp
// 创建DataChunk
DataChunk chunk;
vector<LogicalType> types = {LogicalType::INTEGER, LogicalType::VARCHAR};
chunk.Initialize(Allocator::DefaultAllocator(), types);

// 访问列
Vector &col0 = chunk.data[0];

// 设置行数
chunk.SetCardinality(count);

// 获取信息
idx_t row_count = chunk.size();
idx_t col_count = chunk.ColumnCount();

// 重置
chunk.Reset();

// 拷贝
DataChunk other;
chunk.Copy(other);

// 访问单个值
Value val = chunk.GetValue(col_idx, row_idx);
chunk.SetValue(col_idx, row_idx, Value::INTEGER(42));
```

---

### SelectionVector - 行选择向量

```cpp
// 创建SelectionVector
SelectionVector sel(STANDARD_VECTOR_SIZE);

// 设置索引
for (idx_t i = 0; i < count; i++) {
    sel.set_index(i, source_index);
}

// 使用SelectionVector进行切片
result_vec.Slice(source_vec, sel, count);
```

---

### Value - 单个值

```cpp
// 创建Value
Value v1 = Value::INTEGER(42);
Value v2 = Value::DOUBLE(3.14);
Value v3 = Value::VARCHAR("hello");
Value null_val = Value(LogicalType::INTEGER);  // NULL

// 获取值
int32_t i = v1.GetValue<int32_t>();
string s = v3.ToString();

// 检查NULL
bool is_null = val.IsNull();

// 类型转换
Value casted = val.DefaultCastAs(LogicalType::BIGINT);
```

---

## 🔧 DuckDB C++ API 速查

### 数据库连接

```cpp
// 创建数据库
DuckDB db(nullptr);           // 内存数据库
DuckDB db("mydb.duckdb");     // 持久化数据库

// 创建连接
Connection con(db);

// 执行查询
auto result = con.Query("SELECT * FROM table");

// 带参数的查询
auto prepared = con.Prepare("SELECT * FROM table WHERE id = ?");
auto result = prepared->Execute(42);

// 插入数据
con.Query("INSERT INTO table VALUES (1, 'test')");

// 错误处理
try {
    con.Query("INVALID SQL");
} catch (Exception &e) {
    std::cerr << "Error: " << e.what() << std::endl;
}
```

---

### 结果处理

```cpp
// 遍历结果
auto result = con.Query("SELECT a, b FROM table");

// 方式1：逐行访问
for (idx_t row = 0; row < result->RowCount(); row++) {
    auto a = result->GetValue(0, row);  // 第一列
    auto b = result->GetValue(1, row);  // 第二列
}

// 方式2：使用Chunk
auto &chunks = result->Collection();
for (auto &chunk : chunks.Chunks()) {
    auto col_a = FlatVector::GetData<int32_t>(chunk.data[0]);
    auto col_b = FlatVector::GetData<double>(chunk.data[1]);

    for (idx_t i = 0; i < chunk.size(); i++) {
        // 处理 col_a[i], col_b[i]
    }
}

// 打印结果
result->Print();
```

---

## 🎯 算子实现模式

### Source算子 (TableScan)

```cpp
class PhysicalTableScan : public PhysicalOperator {
public:
    // 获取一批数据
    OperatorResultType GetData(ExecutionContext &context,
                               DataChunk &result,
                               GlobalSourceState &gstate,
                               LocalSourceState &lstate) const override {
        // 扫描数据到result
        // 返回 HAVE_MORE_OUTPUT 或 FINISHED
    }
};
```

---

### Operator算子 (Filter)

```cpp
class PhysicalFilter : public PhysicalOperator {
public:
    // 处理输入，产生输出
    OperatorResultType Execute(ExecutionContext &context,
                               DataChunk &input,
                               DataChunk &output,
                               GlobalOperatorState &gstate,
                               OperatorState &state) const override {
        // 应用过滤条件
        SelectionVector sel(STANDARD_VECTOR_SIZE);
        idx_t approved_count = 0;

        for (idx_t i = 0; i < input.size(); i++) {
            if (predicate_matches(i)) {
                sel.set_index(approved_count++, i);
            }
        }

        // 切片输出
        output.Slice(input, sel, approved_count);
        output.SetCardinality(approved_count);

        return OperatorResultType::NEED_MORE_INPUT;
    }
};
```

---

### Sink算子 (HashJoin Build)

```cpp
class PhysicalHashJoin : public PhysicalSink {
public:
    // Sink: 构建哈希表
    SinkResultType Sink(ExecutionContext &context,
                        DataChunk &input,
                        OperatorSinkState &state) const override {
        auto &ht_state = (HashTableState &)state;

        // 将input添加到哈希表
        ht_state.hash_table.Build(input);

        return SinkResultType::NEED_MORE_INPUT;
    }

    // Finalize: 完成哈希表构建
    SinkFinalizeType Finalize(ExecutionContext &context,
                              OperatorSinkState &state) const override {
        auto &ht_state = (HashTableState &)state;
        ht_state.hash_table.Finalize();
        return SinkFinalizeType::READY;
    }

    // Source: 执行探测
    OperatorResultType GetData(ExecutionContext &context,
                               DataChunk &result,
                               GlobalSourceState &gstate,
                               LocalSourceState &lstate) const override {
        // 使用哈希表进行探测
        // 产生join结果
    }
};
```

---

## 🔍 表达式系统速查

### 表达式类型

```cpp
// 常量表达式
auto const_expr = make_uniq<BoundConstantExpression>(Value::INTEGER(42));

// 列引用
auto col_ref = make_uniq<BoundColumnRefExpression>(
    LogicalType::INTEGER,
    ColumnBinding(table_index, column_index)
);

// 函数调用
vector<unique_ptr<Expression>> args;
args.push_back(std::move(left_expr));
args.push_back(std::move(right_expr));

auto func_expr = make_uniq<BoundFunctionExpression>(
    LogicalType::INTEGER,
    add_function,
    std::move(args)
);

// 比较表达式
auto compare_expr = make_uniq<BoundComparisonExpression>(
    ExpressionType::COMPARE_EQUAL,
    std::move(left),
    std::move(right)
);
```

---

### 表达式执行

```cpp
// 使用ExpressionExecutor
ExpressionExecutor executor(Allocator::DefaultAllocator());

// 添加表达式
executor.AddExpression(*expression);

// 执行
DataChunk input, output;
executor.Execute(input, output);
```

---

## 📈 优化器模式

### Filter Pushdown

```cpp
// 在Optimizer中注册规则
optimizer.AddRule<FilterPushdownRule>();

// FilterPushdownRule实现
class FilterPushdownRule : public Rule {
    unique_ptr<LogicalOperator> Apply(LogicalOperator *op) {
        // 检查是否是Filter -> 其他算子
        if (op->type == LogicalOperatorType::FILTER) {
            auto filter = (LogicalFilter *)op;
            auto child = filter->children[0].get();

            // 尝试下推
            if (CanPushdown(child)) {
                return PushdownFilter(filter, child);
            }
        }
        return nullptr;
    }
};
```

---

### Join Order优化

```cpp
// 使用动态规划
unordered_map<set<idx_t>, PlanNode> dp;

// Base case: 单表
for (auto table : tables) {
    set<idx_t> s = {table};
    dp[s] = CreateScanPlan(table);
}

// 递归构建
for (size_t size = 2; size <= n; size++) {
    for (auto &subset : AllSubsets(size)) {
        // 尝试所有分割
        for (auto &left_subset : AllProperSubsets(subset)) {
            auto right_subset = subset - left_subset;

            double cost = dp[left_subset].cost + dp[right_subset].cost
                        + JoinCost(left_subset, right_subset);

            if (cost < dp[subset].cost) {
                dp[subset] = CreateJoinPlan(left_subset, right_subset, cost);
            }
        }
    }
}
```

---

## 💾 存储系统速查

### RowGroup操作

```cpp
// Append数据
void RowGroup::Append(DataChunk &chunk) {
    for (idx_t col = 0; col < chunk.ColumnCount(); col++) {
        columns[col]->Append(chunk.data[col], chunk.size());
    }
    row_count += chunk.size();
}

// Scan数据
void RowGroup::Scan(DataChunk &result, idx_t start_row, idx_t count) {
    for (idx_t col = 0; col < result.ColumnCount(); col++) {
        columns[col]->Scan(result.data[col], start_row, count);
    }
    result.SetCardinality(count);
}

// Checkpoint: 持久化到磁盘
void RowGroup::Checkpoint(Serializer &serializer) {
    serializer.Write<idx_t>(row_count);
    for (auto &column : columns) {
        column->Checkpoint(serializer);
    }
}
```

---

### 压缩算法

**Frame-of-Reference (FOR):**
```cpp
// 压缩
int32_t base = *std::min_element(data, data + count);
for (idx_t i = 0; i < count; i++) {
    compressed[i] = data[i] - base;
}

// 解压
for (idx_t i = 0; i < count; i++) {
    data[i] = compressed[i] + base;
}
```

**Dictionary编码:**
```cpp
// 构建字典
unordered_map<string, uint32_t> dictionary;
vector<uint32_t> codes;

for (auto &str : strings) {
    if (dictionary.find(str) == dictionary.end()) {
        dictionary[str] = dict_size++;
    }
    codes.push_back(dictionary[str]);
}
```

**RLE (Run-Length Encoding):**
```cpp
// 压缩
vector<pair<int32_t, idx_t>> runs;
int32_t current = data[0];
idx_t run_length = 1;

for (idx_t i = 1; i < count; i++) {
    if (data[i] == current) {
        run_length++;
    } else {
        runs.push_back({current, run_length});
        current = data[i];
        run_length = 1;
    }
}
runs.push_back({current, run_length});
```

---

## 🔐 事务系统速查

### MVCC基础

```cpp
// 版本链
struct VersionInfo {
    transaction_t version_number;
    VersionInfo *next;
    bool deleted;
    // 数据...
};

// 可见性检查
bool IsVisible(VersionInfo *version, transaction_t my_txn) {
    if (version->version_number > my_txn) {
        return false;  // 未来版本
    }
    if (version->deleted && version->version_number <= my_txn) {
        return false;  // 已删除
    }
    return true;
}

// 获取可见版本
VersionInfo *GetVisibleVersion(VersionInfo *head, transaction_t txn) {
    for (auto v = head; v != nullptr; v = v->next) {
        if (IsVisible(v, txn)) {
            return v;
        }
    }
    return nullptr;
}
```

---

## 🛠️ 常用工具函数

### 类型转换

```cpp
// Value转换
Value int_val = Value::INTEGER(42);
Value bigint_val = int_val.DefaultCastAs(LogicalType::BIGINT);

// Vector类型转换
Vector result_vec(target_type, count);
VectorOperations::Cast(source_vec, result_vec, count);
```

---

### 字符串操作

```cpp
// 创建string_t
string_t str = string_t("hello");

// 获取C风格字符串
const char *cstr = str.GetDataUnsafe();

// 字符串长度
uint32_t len = str.GetSize();

// 拷贝字符串
string result(str.GetDataUnsafe(), str.GetSize());
```

---

### 内存分配

```cpp
// 使用Allocator
Allocator &allocator = Allocator::DefaultAllocator();

// 分配内存
data_ptr_t ptr = allocator.Allocate(size);

// 释放内存（通常由BufferHandle管理）
// allocator.Free(ptr, size);

// 使用BufferManager
auto &buffer_manager = BufferManager::GetBufferManager(db);
auto handle = buffer_manager.Allocate(size);

// 访问数据
data_ptr_t data = handle.Ptr();
```

---

## 🎨 向量化操作

### 向量化算术

```cpp
// 加法
VectorOperations::Add(left, right, result, count);

// 乘法
VectorOperations::Multiply(left, right, result, count);

// 聚合
int64_t sum = VectorOperations::Sum(vec, count);
double avg = VectorOperations::Average(vec, count);
```

---

### 向量化比较

```cpp
// 创建SelectionVector存储匹配的行
SelectionVector sel(STANDARD_VECTOR_SIZE);
idx_t approved_count = 0;

// 使用VectorOperations进行比较
VectorOperations::Equals(left, right, sel, approved_count, count);

// 手动比较
auto left_data = FlatVector::GetData<int32_t>(left);
auto right_data = FlatVector::GetData<int32_t>(right);

for (idx_t i = 0; i < count; i++) {
    if (left_data[i] == right_data[i]) {
        sel.set_index(approved_count++, i);
    }
}
```

---

## 🐛 调试技巧

### 打印调试

```cpp
// 打印Vector
vec.Print(count);

// 打印DataChunk
chunk.Print();

// 打印Value
std::cout << val.ToString() << std::endl;

// 打印查询计划
auto plan = con.Query("EXPLAIN SELECT ...");
plan->Print();

// 打印执行统计
auto result = con.Query("EXPLAIN ANALYZE SELECT ...");
result->Print();
```

---

### 断言和验证

```cpp
// DuckDB断言
D_ASSERT(condition);

// 验证Vector
D_ASSERT(vec.GetVectorSize() <= STANDARD_VECTOR_SIZE);

// 验证DataChunk
D_ASSERT(chunk.size() <= STANDARD_VECTOR_SIZE);
D_ASSERT(chunk.ColumnCount() == expected_columns);
```

---

## 📚 常量速查

```cpp
// 标准Vector大小
STANDARD_VECTOR_SIZE = 2048

// 无效索引
DConstants::INVALID_INDEX = (idx_t)-1

// 存储常量
Storage::BLOCK_SIZE = 262144  // 256KB
Storage::ROW_GROUP_SIZE = 122880  // 行
```

---

## 🔗 常用头文件

```cpp
// 核心类型
#include "duckdb/common/types/vector.hpp"
#include "duckdb/common/types/data_chunk.hpp"
#include "duckdb/common/types/value.hpp"

// 算子
#include "duckdb/execution/physical_operator.hpp"
#include "duckdb/execution/operator/scan/physical_table_scan.hpp"
#include "duckdb/execution/operator/filter/physical_filter.hpp"

// 表达式
#include "duckdb/planner/expression.hpp"
#include "duckdb/execution/expression_executor.hpp"

// 存储
#include "duckdb/storage/data_table.hpp"
#include "duckdb/storage/table/row_group.hpp"

// C++ API
#include "duckdb.hpp"
#include "duckdb/main/connection.hpp"
```

---

## 🚀 性能优化Checklist

- ✅ 使用向量化操作而不是逐行处理
- ✅ 避免不必要的数据拷贝（使用Slice和SelectionVector）
- ✅ 合理使用SelectionVector进行零拷贝过滤
- ✅ 预分配内存减少动态分配
- ✅ 使用SIMD指令加速计算
- ✅ 利用缓存局部性（列式访问）
- ✅ 实现Filter Pushdown减少数据量
- ✅ 优化Join Order降低中间结果
- ✅ 使用压缩减少I/O和内存占用
- ✅ 启用并行执行（Pipeline并行）

---

## 📖 快速参考

| 操作 | 代码片段 |
|------|----------|
| 创建Vector | `Vector vec(LogicalType::INTEGER, 2048);` |
| 获取数据指针 | `auto data = FlatVector::GetData<int32_t>(vec);` |
| 设置NULL | `FlatVector::Validity(vec).SetInvalid(i);` |
| 创建DataChunk | `chunk.Initialize(alloc, types);` |
| 执行SQL | `con.Query("SELECT ...");` |
| 打印结果 | `result->Print();` |
| 查看执行计划 | `con.Query("EXPLAIN ANALYZE ...");` |
| 创建数据库 | `DuckDB db("mydb.duckdb");` |
| 向量化加法 | `VectorOperations::Add(a, b, result, count);` |
| Filter操作 | `result.Slice(input, sel, approved_count);` |

---

**提示：** 将此速查表打印或保存在手边，在编写DuckDB代码时随时参考！

---

最后更新：2026-01-17
