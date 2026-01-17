# DuckDB 30天学习课程 - 第二至四周详细内容

本文档是《DuckDB_30天学习课程.md》的续篇，包含Day 10-30的详细内容。

---

## 第二周（续）：查询处理与算子实现

### Day 10: 物理计划 - PhysicalOperator

**学习目标：** 理解逻辑计划到物理计划的转换过程

#### 10.1 LogicalOperator vs PhysicalOperator

**逻辑计划（What）：**
- 描述"做什么"，不关心"怎么做"
- 与具体执行策略无关
- 例如：LogicalJoin不关心是Hash Join还是Merge Join

**物理计划（How）：**
- 描述"怎么做"，具体的执行算法
- 包含执行细节和优化策略
- 例如：PhysicalHashJoin, PhysicalBlockwiseNLJoin

#### 10.2 PhysicalOperator基类

```cpp
// src/include/duckdb/execution/physical_operator.hpp
class PhysicalOperator {
public:
    PhysicalOperatorType type;
    vector<reference<PhysicalOperator>> children;
    vector<LogicalType> types;  // 输出列类型

    idx_t estimated_cardinality;

    // 获取数据的主要方法
    virtual OperatorResultType Execute(ExecutionContext &context,
                                       DataChunk &input,
                                       DataChunk &chunk,
                                       GlobalOperatorState &state,
                                       LocalOperatorState &lstate);

    // 初始化全局状态
    virtual unique_ptr<GlobalOperatorState> GetGlobalOperatorState(ClientContext &context);

    // 初始化本地状态（每个线程）
    virtual unique_ptr<LocalOperatorState> GetLocalOperatorState(ExecutionContext &context,
                                                                   GlobalOperatorState &gstate);
};

enum class OperatorResultType : uint8_t {
    NEED_MORE_INPUT,    // 需要更多输入
    HAVE_MORE_OUTPUT,   // 还有更多输出
    FINISHED            // 完成
};
```

#### 10.3 Source vs Sink vs Operator

DuckDB根据算子的数据流特性分类：

**Source（源算子）：**
- 产生数据，无需输入
- 例如：TableScan, ConstantScan
```cpp
class PhysicalTableScan : public PhysicalOperator {
    // 从存储读取数据
    OperatorResultType Execute(...) {
        // 扫描表，填充output chunk
        table->Scan(transaction, output);
        return output.size() > 0 ? HAVE_MORE_OUTPUT : FINISHED;
    }
};
```

**Operator（普通算子）：**
- 接收输入，产生输出
- 一般是1:1或1:N的流式处理
- 例如：Filter, Projection
```cpp
class PhysicalFilter : public PhysicalOperator {
    OperatorResultType Execute(DataChunk &input, DataChunk &output, ...) {
        // 应用过滤条件
        SelectionVector sel(STANDARD_VECTOR_SIZE);
        idx_t count = FilterData(input, sel);
        output.Slice(input, sel, count);
        return NEED_MORE_INPUT;
    }
};
```

**Sink（汇聚算子）：**
- 接收所有输入后才产生输出
- 例如：HashJoin（Build侧）, Aggregate, OrderBy
```cpp
class PhysicalHashJoin : public PhysicalOperator {
    // Build阶段：收集右表数据
    SinkResultType Sink(DataChunk &input, ...) {
        // 插入Hash Table
        hash_table->Build(input);
        return NEED_MORE_INPUT;
    }

    // Probe阶段：扫描左表并探测
    OperatorResultType Execute(DataChunk &input, DataChunk &output, ...) {
        hash_table->Probe(input, output);
        return HAVE_MORE_OUTPUT;
    }
};
```

#### 10.4 Pipeline执行模型

DuckDB使用**Push-Based Pipeline**模型：

```
┌─────────────┐
│  TableScan  │ (Source)
└──────┬──────┘
       │ Push DataChunk
       ▼
┌─────────────┐
│   Filter    │ (Operator)
└──────┬──────┘
       │ Push filtered DataChunk
       ▼
┌─────────────┐
│ Projection  │ (Operator)
└──────┬──────┘
       │ Push projected DataChunk
       ▼
┌─────────────┐
│  HashBuild  │ (Sink)
└─────────────┘
```

**Pipeline特点：**
1. 数据由Source主动Push
2. 中间算子流式处理，无需缓存
3. Sink算子终止Pipeline
4. 多个Pipeline串联执行

#### 10.5 LogicalOperator到PhysicalOperator的转换

```cpp
// src/execution/physical_plan_generator.cpp
class PhysicalPlanGenerator {
public:
    unique_ptr<PhysicalOperator> CreatePlan(LogicalOperator &op) {
        switch (op.type) {
        case LogicalOperatorType::LOGICAL_GET:
            return CreatePlan((LogicalGet&)op);
        case LogicalOperatorType::LOGICAL_FILTER:
            return CreatePlan((LogicalFilter&)op);
        case LogicalOperatorType::LOGICAL_PROJECTION:
            return CreatePlan((LogicalProjection&)op);
        case LogicalOperatorType::LOGICAL_COMPARISON_JOIN:
            return CreatePlan((LogicalComparisonJoin&)op);
        case LogicalOperatorType::LOGICAL_AGGREGATE_AND_GROUP_BY:
            return CreatePlan((LogicalAggregate&)op);
        // ... 更多类型
        }
    }

private:
    unique_ptr<PhysicalOperator> CreatePlan(LogicalGet &op) {
        // 创建PhysicalTableScan
        return make_unique<PhysicalTableScan>(
            op.types,
            op.function,
            std::move(op.bind_data),
            op.column_ids,
            op.names,
            std::move(op.table_filters),
            op.estimated_cardinality
        );
    }

    unique_ptr<PhysicalOperator> CreatePlan(LogicalComparisonJoin &op) {
        // 根据Join条件和表大小选择算法
        if (op.join_type == JoinType::INNER && HasEquality(op.conditions)) {
            // Hash Join
            return make_unique<PhysicalHashJoin>(
                op,
                std::move(left_plan),
                std::move(right_plan),
                std::move(op.conditions),
                op.join_type,
                op.estimated_cardinality
            );
        } else {
            // Nested Loop Join
            return make_unique<PhysicalNestedLoopJoin>(
                op,
                std::move(left_plan),
                std::move(right_plan),
                std::move(op.conditions),
                op.join_type,
                op.estimated_cardinality
            );
        }
    }
};
```

#### 10.6 执行示例：完整查询

**SQL：**
```sql
SELECT name, AVG(score)
FROM students
WHERE age > 18
GROUP BY name;
```

**逻辑计划：**
```
LogicalProjection [name, AVG(score)]
└── LogicalAggregate [GROUP BY name, AVG(score)]
    └── LogicalFilter [age > 18]
        └── LogicalGet [students]
```

**物理计划：**
```
PhysicalProjection
└── PhysicalHashAggregate
    └── PhysicalFilter
        └── PhysicalTableScan
```

**Pipeline分解：**
```
Pipeline 1:
  PhysicalTableScan → PhysicalFilter → PhysicalHashAggregate (Sink)

Pipeline 2:
  PhysicalHashAggregate (Source) → PhysicalProjection
```

**实践任务：**
1. 阅读 `src/execution/physical_plan_generator.cpp`
2. 使用 `EXPLAIN` 查看查询的物理计划
3. 理解Source、Operator、Sink的区别

---

### Day 11: 算子实现 - TableScan和Filter

**学习目标：** 深入理解TableScan和Filter的实现细节

#### 11.1 PhysicalTableScan实现

```cpp
// src/execution/operator/scan/physical_table_scan.cpp

class PhysicalTableScan : public PhysicalOperator {
public:
    TableFunction function;           // 表函数（读取接口）
    unique_ptr<FunctionData> bind_data;  // 绑定数据
    vector<ColumnIndex> column_ids;   // 要读取的列
    unique_ptr<TableFilterSet> table_filters;  // 下推的过滤器

    unique_ptr<GlobalSourceState> GetGlobalSourceState(ClientContext &context) override;
    unique_ptr<LocalSourceState> GetLocalSourceState(ExecutionContext &context,
                                                       GlobalSourceState &gstate) override;

    SourceResultType GetData(ExecutionContext &context,
                             DataChunk &chunk,
                             GlobalSourceState &gstate,
                             LocalSourceState &lstate) override;
};
```

#### 11.2 TableFunction接口

```cpp
// src/include/duckdb/function/table_function.hpp
struct TableFunction {
    string name;

    // 初始化全局状态
    table_function_init_global_t init_global;

    // 初始化本地状态
    table_function_init_local_t init_local;

    // 获取数据
    table_function_t function;

    // 支持并行度
    table_function_max_threads_t max_threads;
};

// 函数签名
typedef void (*table_function_t)(ClientContext &context,
                                  TableFunctionInput &input,
                                  DataChunk &output);
```

#### 11.3 TableScan执行流程

```cpp
// TableScan::GetData 简化版本
SourceResultType PhysicalTableScan::GetData(
    ExecutionContext &context,
    DataChunk &output,
    GlobalSourceState &gstate,
    LocalSourceState &lstate) {

    auto &global_state = gstate.Cast<TableScanGlobalSourceState>();
    auto &local_state = lstate.Cast<TableScanLocalSourceState>();

    // 调用TableFunction获取数据
    TableFunctionInput input(bind_data.get(),
                             local_state.local_state.get(),
                             global_state.global_state.get());

    function.function(context.client, input, output);

    if (output.size() == 0) {
        return SourceResultType::FINISHED;
    }

    // 应用表过滤器（如果有）
    if (table_filters) {
        ApplyFilters(output);
    }

    return SourceResultType::HAVE_MORE_OUTPUT;
}
```

#### 11.4 并行TableScan

```cpp
// 全局状态：跨线程共享
class TableScanGlobalSourceState : public GlobalSourceState {
public:
    unique_ptr<GlobalTableFunctionState> global_state;
    atomic<idx_t> max_threads;

    idx_t MaxThreads() override {
        return max_threads;
    }
};

// 本地状态：每个线程独立
class TableScanLocalSourceState : public LocalSourceState {
public:
    unique_ptr<LocalTableFunctionState> local_state;
};

// 例如：并行扫描RowGroup
class ParquetGlobalState : public GlobalTableFunctionState {
    atomic<idx_t> next_row_group{0};  // 下一个要扫描的RowGroup
    idx_t total_row_groups;

    idx_t GetNextRowGroup() {
        return next_row_group.fetch_add(1);
    }
};

class ParquetLocalState : public LocalTableFunctionState {
    idx_t current_row_group;
    RowGroupReader reader;
};

void ParquetScanFunction(ClientContext &context,
                         TableFunctionInput &input,
                         DataChunk &output) {
    auto &global_state = input.global_state->Cast<ParquetGlobalState>();
    auto &local_state = input.local_state->Cast<ParquetLocalState>();

    // 获取下一个row group
    if (!local_state.reader.IsValid()) {
        local_state.current_row_group = global_state.GetNextRowGroup();
        if (local_state.current_row_group >= global_state.total_row_groups) {
            return;  // 没有更多数据
        }
        local_state.reader.Initialize(local_state.current_row_group);
    }

    // 读取数据
    local_state.reader.Scan(output);
}
```

#### 11.5 PhysicalFilter实现

```cpp
// src/execution/operator/filter/physical_filter.cpp

class PhysicalFilter : public PhysicalOperator {
public:
    vector<unique_ptr<Expression>> expressions;  // 过滤条件

    OperatorResultType Execute(ExecutionContext &context,
                               DataChunk &input,
                               DataChunk &chunk,
                               GlobalOperatorState &gstate,
                               OperatorState &state) override {
        // 1. 执行所有过滤条件
        SelectionVector sel(STANDARD_VECTOR_SIZE);
        idx_t approved_tuple_count = input.size();

        for (auto &expr : expressions) {
            // 执行表达式，得到布尔向量
            Vector result(LogicalType::BOOLEAN);
            ExpressionExecutor executor(expr);
            executor.Execute(input, result);

            // 应用过滤
            approved_tuple_count = SelectTrue(result, sel, approved_tuple_count);

            if (approved_tuple_count == 0) {
                break;  // 所有行都被过滤掉
            }
        }

        // 2. 使用SelectionVector切片数据
        if (approved_tuple_count == input.size()) {
            // 所有行都通过，直接引用
            chunk.Reference(input);
        } else {
            // 部分行通过，切片
            chunk.Slice(input, sel, approved_tuple_count);
        }

        chunk.SetCardinality(approved_tuple_count);
        return OperatorResultType::NEED_MORE_INPUT;
    }
};
```

#### 11.6 SelectionVector优化

```cpp
// src/common/types/selection_vector.hpp
class SelectionVector {
    sel_t *sel_vector;  // 索引数组

public:
    // 设置索引
    void set_index(idx_t idx, sel_t value) {
        sel_vector[idx] = value;
    }

    // 获取索引
    sel_t get_index(idx_t idx) const {
        return sel_vector[idx];
    }
};

// 选择为true的行
idx_t SelectTrue(Vector &input, SelectionVector &sel, idx_t count) {
    auto data = FlatVector::GetData<bool>(input);
    auto &validity = FlatVector::Validity(input);

    idx_t approved_count = 0;
    if (validity.AllValid()) {
        // 无NULL值，简化路径
        for (idx_t i = 0; i < count; i++) {
            if (data[i]) {
                sel.set_index(approved_count++, i);
            }
        }
    } else {
        // 有NULL值
        for (idx_t i = 0; i < count; i++) {
            if (validity.RowIsValid(i) && data[i]) {
                sel.set_index(approved_count++, i);
            }
        }
    }
    return approved_count;
}
```

**SelectionVector的优势：**
- **零拷贝过滤** - 不移动数据，只记录索引
- **延迟物化** - 推迟实际的数据拷贝
- **链式过滤** - 多个过滤条件可以组合

#### 11.7 Adaptive Filter（自适应过滤）

DuckDB使用自适应过滤来优化过滤性能：

```cpp
// src/execution/adaptive_filter.hpp
class AdaptiveFilter {
    idx_t total_count = 0;
    idx_t passed_count = 0;
    double selectivity;

public:
    // 更新选择率
    void Update(idx_t input_count, idx_t output_count) {
        total_count += input_count;
        passed_count += output_count;
        selectivity = (double)passed_count / total_count;
    }

    // 根据选择率选择策略
    FilterStrategy GetStrategy() {
        if (selectivity > 0.8) {
            return FilterStrategy::NO_SELECTION_VECTOR;  // 大部分通过，不用sel
        } else if (selectivity < 0.2) {
            return FilterStrategy::EARLY_OUT;  // 大部分不通过，尽早退出
        } else {
            return FilterStrategy::STANDARD;  // 标准路径
        }
    }
};
```

**实践任务：**
1. 阅读 `src/execution/operator/scan/physical_table_scan.cpp`
2. 阅读 `src/execution/operator/filter/physical_filter.cpp`
3. 实现一个简单的Filter算子，支持多个AND条件

---

### Day 12: 算子实现 - Hash Join

**学习目标：** 深入理解Hash Join的实现原理

#### 12.1 Hash Join基本原理

Hash Join分为两个阶段：

**Build阶段（构建哈希表）：**
```
右表（较小表）
  ↓
计算Hash(join_key)
  ↓
插入Hash Table
```

**Probe阶段（探测）：**
```
左表（较大表）
  ↓
计算Hash(join_key)
  ↓
在Hash Table中查找
  ↓
输出匹配的行
```

#### 12.2 PhysicalHashJoin结构

```cpp
// src/execution/operator/join/physical_hash_join.cpp
class PhysicalHashJoin : public PhysicalOperator {
public:
    vector<JoinCondition> conditions;  // Join条件
    JoinType join_type;                // INNER, LEFT, RIGHT, FULL, SEMI, ANTI

    // Build侧的列
    vector<LogicalType> condition_types;  // Join key类型
    vector<LogicalType> build_types;      // Payload类型

    // Sink接口：Build阶段
    SinkResultType Sink(ExecutionContext &context,
                        DataChunk &chunk,
                        OperatorSinkInput &input) override;

    // Source接口：Probe阶段
    SourceResultType GetData(ExecutionContext &context,
                             DataChunk &chunk,
                             OperatorSourceInput &input) override;
};
```

#### 12.3 Hash Table结构

```cpp
// src/execution/join_hashtable.hpp
class JoinHashTable {
public:
    // Hash表大小（2的幂）
    idx_t capacity;

    // Hash槽
    Vector hash_map;  // 指向链表头的指针数组

    // 数据块
    vector<unique_ptr<DataChunk>> blocks;  // 存储实际数据

    // 构建Hash表
    void Build(DataChunk &keys, DataChunk &payload);

    // 探测Hash表
    idx_t Probe(DataChunk &keys, DataChunk &result);
};
```

**Hash Table布局：**
```
hash_map (Array of Pointers):
┌────┬────┬────┬────┬────┐
│ 0  │ 1  │ 2  │ 3  │ 4  │  ... (capacity entries)
└─┬──┴──┬─┴────┴─┬──┴────┘
  │     │        │
  ▼     ▼        ▼
链表  链表      链表
```

#### 12.4 Build阶段实现

```cpp
void JoinHashTable::Build(DataChunk &keys, DataChunk &payload) {
    // 1. 计算Hash值
    Vector hashes(LogicalType::HASH);
    Hash(keys, hashes);

    auto hash_data = FlatVector::GetData<hash_t>(hashes);

    // 2. 为每行分配slot
    for (idx_t i = 0; i < keys.size(); i++) {
        hash_t hash = hash_data[i];
        idx_t bucket = hash & (capacity - 1);  // hash % capacity

        // 3. 插入链表
        auto entry = AllocateEntry();
        entry->hash = hash;
        entry->next = hash_map[bucket];
        hash_map[bucket] = entry;

        // 4. 存储数据
        StoreKeys(entry, keys, i);
        StorePayload(entry, payload, i);
    }
}

// 计算Hash值
void JoinHashTable::Hash(DataChunk &keys, Vector &result) {
    result.SetVectorType(VectorType::FLAT_VECTOR);
    auto result_data = FlatVector::GetData<hash_t>(result);

    // 初始化为0
    for (idx_t i = 0; i < keys.size(); i++) {
        result_data[i] = 0;
    }

    // 组合所有key列的hash
    for (idx_t col = 0; col < keys.ColumnCount(); col++) {
        auto &key_col = keys.data[col];
        VectorOperations::Hash(key_col, result, keys.size());
    }
}
```

#### 12.5 Probe阶段实现

```cpp
idx_t JoinHashTable::Probe(DataChunk &keys, DataChunk &result) {
    // 1. 计算左表的Hash值
    Vector hashes(LogicalType::HASH);
    Hash(keys, hashes);

    auto hash_data = FlatVector::GetData<hash_t>(hashes);

    idx_t match_count = 0;
    SelectionVector left_sel(STANDARD_VECTOR_SIZE);
    SelectionVector right_sel(STANDARD_VECTOR_SIZE);

    // 2. 探测Hash表
    for (idx_t i = 0; i < keys.size(); i++) {
        hash_t hash = hash_data[i];
        idx_t bucket = hash & (capacity - 1);

        // 遍历链表
        auto entry = hash_map[bucket];
        while (entry) {
            if (entry->hash == hash) {
                // Hash匹配，检查实际值
                if (KeysEqual(keys, i, entry)) {
                    // 记录匹配
                    left_sel.set_index(match_count, i);
                    right_sel.set_index(match_count, entry->row_id);
                    match_count++;

                    if (join_type == JoinType::SEMI) {
                        break;  // SEMI JOIN只需要一个匹配
                    }
                }
            }
            entry = entry->next;
        }
    }

    // 3. 构建结果
    // 左表列
    for (idx_t col = 0; col < left_columns.size(); col++) {
        result.data[col].Slice(keys.data[left_columns[col]], left_sel, match_count);
    }

    // 右表列
    for (idx_t col = 0; col < right_columns.size(); col++) {
        result.data[left_columns.size() + col].Slice(
            payload.data[right_columns[col]], right_sel, match_count);
    }

    result.SetCardinality(match_count);
    return match_count;
}

// 比较Key是否相等
bool JoinHashTable::KeysEqual(DataChunk &probe_keys, idx_t probe_idx, Entry *build_entry) {
    for (idx_t col = 0; col < probe_keys.ColumnCount(); col++) {
        auto &probe_vector = probe_keys.data[col];
        auto probe_value = probe_vector.GetValue(probe_idx);

        auto build_value = GetKeyValue(build_entry, col);

        if (probe_value != build_value) {
            return false;
        }
    }
    return true;
}
```

#### 12.6 处理不同Join类型

```cpp
// INNER JOIN
if (join_type == JoinType::INNER) {
    // 只输出匹配的行
    return ProbeInner(keys, result);
}

// LEFT OUTER JOIN
if (join_type == JoinType::LEFT) {
    idx_t match_count = ProbeInner(keys, result);

    // 找出未匹配的左表行
    SelectionVector unmatched_sel(STANDARD_VECTOR_SIZE);
    idx_t unmatched_count = 0;

    for (idx_t i = 0; i < keys.size(); i++) {
        if (!IsMatched(i)) {
            unmatched_sel.set_index(unmatched_count++, i);
        }
    }

    // 为未匹配行填充NULL
    AppendUnmatched(keys, result, unmatched_sel, unmatched_count);

    return match_count + unmatched_count;
}

// SEMI JOIN
if (join_type == JoinType::SEMI) {
    // 只要有匹配就输出左表行（不输出右表）
    SelectionVector sel(STANDARD_VECTOR_SIZE);
    idx_t match_count = 0;

    for (idx_t i = 0; i < keys.size(); i++) {
        if (HasMatch(keys, i)) {
            sel.set_index(match_count++, i);
        }
    }

    result.Slice(keys, sel, match_count);
    return match_count;
}

// ANTI JOIN
if (join_type == JoinType::ANTI) {
    // 只输出没有匹配的左表行
    SelectionVector sel(STANDARD_VECTOR_SIZE);
    idx_t match_count = 0;

    for (idx_t i = 0; i < keys.size(); i++) {
        if (!HasMatch(keys, i)) {
            sel.set_index(match_count++, i);
        }
    }

    result.Slice(keys, sel, match_count);
    return match_count;
}
```

#### 12.7 分区Hash Join（处理大数据）

当Hash表太大无法放入内存时，使用分区策略：

```cpp
// Radix Partitioning
class RadixPartitionedHashTable {
    static constexpr idx_t RADIX_BITS = 8;
    static constexpr idx_t NUM_PARTITIONS = 1 << RADIX_BITS;  // 256

    vector<unique_ptr<JoinHashTable>> partitions;

public:
    void Build(DataChunk &keys, DataChunk &payload) {
        // 1. 计算Hash值
        Vector hashes(LogicalType::HASH);
        Hash(keys, hashes);

        // 2. 根据高位bits分区
        auto hash_data = FlatVector::GetData<hash_t>(hashes);

        for (idx_t i = 0; i < keys.size(); i++) {
            hash_t hash = hash_data[i];
            idx_t partition_idx = hash >> (64 - RADIX_BITS);

            // 插入对应分区
            partitions[partition_idx]->BuildRow(keys, payload, i);
        }
    }

    idx_t Probe(DataChunk &keys, DataChunk &result) {
        Vector hashes(LogicalType::HASH);
        Hash(keys, hashes);

        // 对每个分区分别探测
        idx_t total_matches = 0;
        for (idx_t part = 0; part < NUM_PARTITIONS; part++) {
            auto matches = partitions[part]->Probe(keys, result);
            total_matches += matches;
        }

        return total_matches;
    }
};
```

**实践任务：**
1. 阅读 `src/execution/operator/join/physical_hash_join.cpp`
2. 理解Build和Probe两个阶段
3. 实现一个简单的Hash Join（只支持INNER JOIN和单个等值条件）

---

### Day 13: 算子实现 - Aggregation

**学习目标：** 理解聚合算子的实现，包括Hash聚合和排序聚合

#### 13.1 聚合类型

DuckDB支持多种聚合类型：

```cpp
enum class AggregateType {
    NON_DISTINCT,   // 普通聚合: SUM(x)
    DISTINCT,       // 去重聚合: SUM(DISTINCT x)
    SORTED          // 排序聚合: ARRAY_AGG(x ORDER BY y)
};
```

#### 13.2 AggregateFunction接口

```cpp
// src/include/duckdb/function/aggregate_function.hpp
struct AggregateFunction {
    string name;
    vector<LogicalType> arguments;
    LogicalType return_type;

    // State大小
    aggregate_size_t state_size;

    // 初始化State
    aggregate_initialize_t initialize;

    // 更新State（单行）
    aggregate_update_t update;

    // 合并State（并行）
    aggregate_combine_t combine;

    // 最终化（产生结果）
    aggregate_finalize_t finalize;
};
```

**聚合函数生命周期：**
```
Initialize → Update* → Combine* → Finalize
```

#### 13.3 实现SUM聚合函数

```cpp
// 以SUM(INTEGER)为例

struct SumState {
    int64_t sum;
    bool is_set;
};

// 初始化
void SumInitialize(data_ptr_t state) {
    auto sum_state = (SumState*)state;
    sum_state->sum = 0;
    sum_state->is_set = false;
}

// 更新（单行）
void SumUpdate(Vector inputs[], AggregateInputData &aggr_input_data,
               idx_t input_count, data_ptr_t state, idx_t count) {
    auto sum_state = (SumState*)state;
    auto &input = inputs[0];

    auto input_data = FlatVector::GetData<int32_t>(input);
    auto &validity = FlatVector::Validity(input);

    for (idx_t i = 0; i < count; i++) {
        if (validity.RowIsValid(i)) {
            sum_state->sum += input_data[i];
            sum_state->is_set = true;
        }
    }
}

// 合并（并行聚合）
void SumCombine(Vector &source, Vector &target, AggregateInputData &aggr_input_data,
                idx_t count) {
    auto source_data = (SumState*)FlatVector::GetData(source);
    auto target_data = (SumState*)FlatVector::GetData(target);

    for (idx_t i = 0; i < count; i++) {
        if (source_data[i].is_set) {
            target_data[i].sum += source_data[i].sum;
            target_data[i].is_set = true;
        }
    }
}

// 最终化
void SumFinalize(Vector &result, AggregateInputData &aggr_input_data,
                 Vector &states, idx_t count) {
    auto states_data = (SumState*)FlatVector::GetData(states);
    auto result_data = FlatVector::GetData<int64_t>(result);
    auto &result_validity = FlatVector::Validity(result);

    for (idx_t i = 0; i < count; i++) {
        if (states_data[i].is_set) {
            result_data[i] = states_data[i].sum;
        } else {
            result_validity.SetInvalid(i);  // NULL
        }
    }
}

// 注册函数
AggregateFunction sum_function(
    "sum",
    {LogicalType::INTEGER},
    LogicalType::BIGINT,
    AggregateFunction::StateSize<SumState>,
    AggregateFunction::StateInitialize<SumState, SumInitialize>,
    SumUpdate,
    SumCombine,
    SumFinalize
);
```

#### 13.4 PhysicalHashAggregate

```cpp
// src/execution/operator/aggregate/physical_hash_aggregate.cpp
class PhysicalHashAggregate : public PhysicalOperator {
public:
    vector<unique_ptr<Expression>> groups;       // GROUP BY表达式
    vector<unique_ptr<Expression>> aggregates;   // 聚合函数

    // Sink阶段：收集数据
    SinkResultType Sink(ExecutionContext &context,
                        DataChunk &chunk,
                        OperatorSinkInput &input) override;

    // Source阶段：输出结果
    SourceResultType GetData(ExecutionContext &context,
                             DataChunk &chunk,
                             OperatorSourceInput &input) override;
};
```

#### 13.5 GroupedAggregateHashTable

```cpp
// src/execution/aggregate_hashtable.hpp
class GroupedAggregateHashTable {
public:
    // Group keys
    vector<LogicalType> group_types;

    // Aggregate states
    vector<AggregateObject> aggregates;

    // Hash表
    Vector hash_map;
    vector<unique_ptr<DataChunk>> data_blocks;

    // 添加一批数据
    void AddChunk(DataChunk &groups, DataChunk &payload);

    // 扫描结果
    void Scan(DataChunk &result);
};

struct AggregateObject {
    AggregateFunction function;
    vector<unique_ptr<Expression>> children;
    unique_ptr<FunctionData> bind_data;
};
```

#### 13.6 Hash聚合执行流程

**Sink阶段：**
```cpp
SinkResultType PhysicalHashAggregate::Sink(
    ExecutionContext &context,
    DataChunk &chunk,
    OperatorSinkInput &input) {

    auto &gstate = input.global_state.Cast<HashAggregateGlobalState>();
    auto &lstate = input.local_state.Cast<HashAggregateLocalState>();

    // 1. 计算GROUP BY列
    DataChunk group_chunk;
    group_chunk.InitializeEmpty(group_types);

    for (idx_t i = 0; i < groups.size(); i++) {
        ExpressionExecutor::Execute(groups[i], chunk, group_chunk.data[i]);
    }
    group_chunk.SetCardinality(chunk.size());

    // 2. 计算聚合函数的输入
    DataChunk aggregate_input_chunk;
    aggregate_input_chunk.InitializeEmpty(aggregate_input_types);

    for (idx_t i = 0; i < aggregates.size(); i++) {
        auto &aggr = aggregates[i];
        for (auto &child : aggr.children) {
            Vector result(child->return_type);
            ExpressionExecutor::Execute(child, chunk, result);
            aggregate_input_chunk.data.push_back(std::move(result));
        }
    }

    // 3. 插入Hash表并更新聚合状态
    lstate.ht->AddChunk(group_chunk, aggregate_input_chunk);

    return SinkResultType::NEED_MORE_INPUT;
}
```

**AddChunk实现：**
```cpp
void GroupedAggregateHashTable::AddChunk(DataChunk &groups, DataChunk &payload) {
    // 1. 计算group keys的hash
    Vector hashes(LogicalType::HASH);
    Hash(groups, hashes);

    auto hash_data = FlatVector::GetData<hash_t>(hashes);

    for (idx_t i = 0; i < groups.size(); i++) {
        hash_t hash = hash_data[i];
        idx_t bucket = hash & (capacity - 1);

        // 2. 查找或创建group entry
        auto entry = FindOrCreateGroup(groups, i, hash, bucket);

        // 3. 更新聚合状态
        for (idx_t aggr_idx = 0; aggr_idx < aggregates.size(); aggr_idx++) {
            auto &aggr = aggregates[aggr_idx];
            auto state_ptr = GetAggregateState(entry, aggr_idx);

            // 调用update函数
            Vector aggr_input;
            aggr_input.Reference(payload.data[aggr_idx]);

            aggr.function.update(&aggr_input, aggr.bind_data.get(),
                                1, state_ptr, 1);
        }
    }
}

Entry* GroupedAggregateHashTable::FindOrCreateGroup(
    DataChunk &groups, idx_t row_idx, hash_t hash, idx_t bucket) {

    // 在链表中查找
    auto entry = hash_map[bucket];
    while (entry) {
        if (entry->hash == hash && GroupKeysEqual(groups, row_idx, entry)) {
            return entry;  // 找到现有group
        }
        entry = entry->next;
    }

    // 创建新group
    auto new_entry = AllocateEntry();
    new_entry->hash = hash;
    new_entry->next = hash_map[bucket];
    hash_map[bucket] = new_entry;

    // 初始化group keys
    StoreGroupKeys(new_entry, groups, row_idx);

    // 初始化聚合states
    for (idx_t i = 0; i < aggregates.size(); i++) {
        auto state_ptr = GetAggregateState(new_entry, i);
        aggregates[i].function.initialize(state_ptr);
    }

    return new_entry;
}
```

**Source阶段：**
```cpp
SourceResultType PhysicalHashAggregate::GetData(
    ExecutionContext &context,
    DataChunk &chunk,
    OperatorSourceInput &input) {

    auto &gstate = input.global_state.Cast<HashAggregateGlobalState>();

    // 扫描Hash表
    gstate.ht->Scan(chunk);

    if (chunk.size() == 0) {
        return SourceResultType::FINISHED;
    }

    return SourceResultType::HAVE_MORE_OUTPUT;
}

void GroupedAggregateHashTable::Scan(DataChunk &result) {
    idx_t scan_count = 0;

    // 遍历所有entries
    while (scan_count < STANDARD_VECTOR_SIZE && HasMoreEntries()) {
        auto entry = GetNextEntry();

        // 输出group keys
        for (idx_t col = 0; col < group_types.size(); col++) {
            result.data[col].SetValue(scan_count, GetGroupKey(entry, col));
        }

        // 输出聚合结果
        for (idx_t aggr_idx = 0; aggr_idx < aggregates.size(); aggr_idx++) {
            auto &aggr = aggregates[aggr_idx];
            auto state_ptr = GetAggregateState(entry, aggr_idx);

            // 调用finalize
            Vector temp_result(aggr.function.return_type);
            aggr.function.finalize(temp_result, aggr.bind_data.get(),
                                  state_ptr, 1);

            idx_t col_idx = group_types.size() + aggr_idx;
            result.data[col_idx].SetValue(scan_count, temp_result.GetValue(0));
        }

        scan_count++;
    }

    result.SetCardinality(scan_count);
}
```

#### 13.7 并行聚合

```cpp
// 每个线程维护本地Hash表
class HashAggregateLocalState : public LocalSinkState {
public:
    unique_ptr<GroupedAggregateHashTable> ht;
};

// 全局状态负责合并
class HashAggregateGlobalState : public GlobalSinkState {
public:
    unique_ptr<GroupedAggregateHashTable> ht;

    void Combine(LocalSinkState &lstate) {
        auto &local = lstate.Cast<HashAggregateLocalState>();

        // 合并本地Hash表到全局Hash表
        local.ht->Combine(*ht);
    }
};

void GroupedAggregateHashTable::Combine(GroupedAggregateHashTable &other) {
    // 遍历other的所有entries
    for (auto entry : other.Entries()) {
        // 在this中查找或创建对应的group
        auto this_entry = FindOrCreateGroup(entry.keys);

        // 合并聚合状态
        for (idx_t i = 0; i < aggregates.size(); i++) {
            auto this_state = GetAggregateState(this_entry, i);
            auto other_state = other.GetAggregateState(entry, i);

            // 调用combine函数
            aggregates[i].function.combine(other_state, this_state,
                                          aggregates[i].bind_data.get(), 1);
        }
    }
}
```

**实践任务：**
1. 阅读 `src/function/aggregate/distributive/sum.cpp` (SUM实现)
2. 阅读 `src/execution/operator/aggregate/physical_hash_aggregate.cpp`
3. 实现一个简单的聚合函数（如COUNT或AVG）

---

### Day 14: 第二周总结 - 实现基本查询执行

**学习目标：** 综合运用第二周知识，实现一个简单的查询执行引擎

#### 14.1 第二周知识回顾

- **Day 8:** Binder - 符号解析和类型推导
- **Day 9:** LogicalOperator - 逻辑计划表示
- **Day 10:** PhysicalOperator - 物理执行算子
- **Day 11:** TableScan和Filter实现
- **Day 12:** Hash Join实现
- **Day 13:** Aggregation实现

#### 14.2 实践项目：SimpleQueryEngine

实现一个简化的查询执行引擎，支持：
- FROM（表扫描）
- WHERE（过滤）
- JOIN（Hash Join）
- GROUP BY + Aggregation
- SELECT（投影）

```cpp
// simple_query_engine.hpp
#pragma once

#include "duckdb/common/types/data_chunk.hpp"
#include "simple_table.hpp"  // Day 7实现

namespace duckdb {

// 简化的物理算子基类
class SimpleOperator {
public:
    virtual ~SimpleOperator() = default;

    // 获取下一批数据
    virtual bool GetNext(DataChunk &result) = 0;

    // 重置算子
    virtual void Reset() = 0;
};

// TableScan算子
class SimpleTableScan : public SimpleOperator {
    SimpleTable &table;
    idx_t current_chunk_idx = 0;

public:
    SimpleTableScan(SimpleTable &table) : table(table) {}

    bool GetNext(DataChunk &result) override {
        if (current_chunk_idx >= table.GetChunkCount()) {
            return false;
        }
        result.Reference(table.GetChunk(current_chunk_idx++));
        return true;
    }

    void Reset() override {
        current_chunk_idx = 0;
    }
};

// Filter算子
class SimpleFilter : public SimpleOperator {
    unique_ptr<SimpleOperator> child;
    std::function<bool(DataChunk&, idx_t)> predicate;

public:
    SimpleFilter(unique_ptr<SimpleOperator> child,
                 std::function<bool(DataChunk&, idx_t)> pred)
        : child(std::move(child)), predicate(pred) {}

    bool GetNext(DataChunk &result) override {
        DataChunk input;
        while (child->GetNext(input)) {
            SelectionVector sel(STANDARD_VECTOR_SIZE);
            idx_t approved_count = 0;

            // 应用谓词
            for (idx_t i = 0; i < input.size(); i++) {
                if (predicate(input, i)) {
                    sel.set_index(approved_count++, i);
                }
            }

            if (approved_count > 0) {
                result.Slice(input, sel, approved_count);
                result.SetCardinality(approved_count);
                return true;
            }
        }
        return false;
    }

    void Reset() override {
        child->Reset();
    }
};

// Projection算子
class SimpleProjection : public SimpleOperator {
    unique_ptr<SimpleOperator> child;
    vector<idx_t> column_indices;  // 要投影的列

public:
    SimpleProjection(unique_ptr<SimpleOperator> child, vector<idx_t> cols)
        : child(std::move(child)), column_indices(std::move(cols)) {}

    bool GetNext(DataChunk &result) override {
        DataChunk input;
        if (!child->GetNext(input)) {
            return false;
        }

        // 只选择指定的列
        vector<LogicalType> types;
        for (auto col_idx : column_indices) {
            types.push_back(input.data[col_idx].GetType());
        }

        result.InitializeEmpty(types);
        for (idx_t i = 0; i < column_indices.size(); i++) {
            result.data[i].Reference(input.data[column_indices[i]]);
        }
        result.SetCardinality(input.size());

        return true;
    }

    void Reset() override {
        child->Reset();
    }
};

// HashJoin算子（简化版，只支持INNER JOIN）
class SimpleHashJoin : public SimpleOperator {
    unique_ptr<SimpleOperator> left_child;
    unique_ptr<SimpleOperator> right_child;
    idx_t left_key_col;
    idx_t right_key_col;

    // Hash表
    unordered_map<Value, vector<idx_t>> hash_table;
    vector<DataChunk> right_chunks;

    bool build_done = false;
    idx_t current_left_chunk_idx = 0;
    vector<DataChunk> left_chunks;

public:
    SimpleHashJoin(unique_ptr<SimpleOperator> left,
                   unique_ptr<SimpleOperator> right,
                   idx_t left_key, idx_t right_key)
        : left_child(std::move(left)), right_child(std::move(right)),
          left_key_col(left_key), right_key_col(right_key) {}

    bool GetNext(DataChunk &result) override {
        // Build阶段：构建右表的Hash表
        if (!build_done) {
            BuildHashTable();
            build_done = true;

            // 缓存左表数据
            DataChunk chunk;
            while (left_child->GetNext(chunk)) {
                left_chunks.push_back(chunk);
            }
        }

        // Probe阶段
        if (current_left_chunk_idx >= left_chunks.size()) {
            return false;
        }

        auto &left_chunk = left_chunks[current_left_chunk_idx++];
        ProbeHashTable(left_chunk, result);

        return result.size() > 0 || GetNext(result);  // 递归直到有结果
    }

    void Reset() override {
        left_child->Reset();
        right_child->Reset();
        hash_table.clear();
        right_chunks.clear();
        left_chunks.clear();
        build_done = false;
        current_left_chunk_idx = 0;
    }

private:
    void BuildHashTable() {
        DataChunk chunk;
        while (right_child->GetNext(chunk)) {
            // 存储chunk
            idx_t chunk_idx = right_chunks.size();
            right_chunks.push_back(chunk);

            // 构建hash表
            for (idx_t i = 0; i < chunk.size(); i++) {
                Value key = chunk.GetValue(right_key_col, i);
                hash_table[key].push_back(chunk_idx * STANDARD_VECTOR_SIZE + i);
            }
        }
    }

    void ProbeHashTable(DataChunk &left_chunk, DataChunk &result) {
        vector<idx_t> left_matches;
        vector<idx_t> right_matches;

        // 探测
        for (idx_t i = 0; i < left_chunk.size(); i++) {
            Value key = left_chunk.GetValue(left_key_col, i);

            auto it = hash_table.find(key);
            if (it != hash_table.end()) {
                for (auto right_row : it->second) {
                    left_matches.push_back(i);
                    right_matches.push_back(right_row);
                }
            }
        }

        // 构建结果
        if (left_matches.empty()) {
            result.SetCardinality(0);
            return;
        }

        // 简化：只返回左表和右表的所有列
        // 实际应该根据投影列表选择
        idx_t match_count = left_matches.size();

        // ... 构建result (留作练习)

        result.SetCardinality(match_count);
    }
};

// HashAggregate算子（简化版）
class SimpleHashAggregate : public SimpleOperator {
    unique_ptr<SimpleOperator> child;
    vector<idx_t> group_cols;  // GROUP BY列
    idx_t aggr_col;            // 要聚合的列

    struct GroupState {
        int64_t sum = 0;
        idx_t count = 0;
    };

    unordered_map<Value, GroupState> groups;
    bool scan_done = false;

public:
    SimpleHashAggregate(unique_ptr<SimpleOperator> child,
                        vector<idx_t> groups, idx_t aggr)
        : child(std::move(child)), group_cols(std::move(groups)),
          aggr_col(aggr) {}

    bool GetNext(DataChunk &result) override {
        if (!scan_done) {
            // Sink阶段：收集所有数据
            DataChunk chunk;
            while (child->GetNext(chunk)) {
                for (idx_t i = 0; i < chunk.size(); i++) {
                    // 简化：只支持单个group key
                    Value group_key = chunk.GetValue(group_cols[0], i);
                    Value aggr_value = chunk.GetValue(aggr_col, i);

                    auto &state = groups[group_key];
                    state.sum += aggr_value.GetValue<int64_t>();
                    state.count++;
                }
            }
            scan_done = true;
        }

        // Source阶段：输出结果
        // ... 输出聚合结果（留作练习）

        return false;
    }

    void Reset() override {
        child->Reset();
        groups.clear();
        scan_done = false;
    }
};

} // namespace duckdb
```

#### 14.3 使用示例

```cpp
#include "simple_query_engine.hpp"

void TestQueryEngine() {
    using namespace duckdb;

    // 创建表1: students (id, name, class_id, score)
    SimpleTable students({
        LogicalType::INTEGER,
        LogicalType::VARCHAR,
        LogicalType::INTEGER,
        LogicalType::DOUBLE
    }, {"id", "name", "class_id", "score"});

    students.Insert({Value::INTEGER(1), Value("Alice"), Value::INTEGER(1), Value::DOUBLE(95.5)});
    students.Insert({Value::INTEGER(2), Value("Bob"), Value::INTEGER(2), Value::DOUBLE(87.3)});
    students.Insert({Value::INTEGER(3), Value("Charlie"), Value::INTEGER(1), Value::DOUBLE(92.1)});

    // 创建表2: classes (id, name)
    SimpleTable classes({
        LogicalType::INTEGER,
        LogicalType::VARCHAR
    }, {"id", "name"});

    classes.Insert({Value::INTEGER(1), Value("Math")});
    classes.Insert({Value::INTEGER(2), Value("English")});

    // 查询: SELECT s.name, c.name, s.score
    //       FROM students s JOIN classes c ON s.class_id = c.id
    //       WHERE s.score > 90

    // 构建查询计划
    auto left = make_unique<SimpleTableScan>(students);
    auto right = make_unique<SimpleTableScan>(classes);

    // JOIN
    auto join = make_unique<SimpleHashJoin>(
        std::move(left), std::move(right),
        2,  // students.class_id
        0   // classes.id
    );

    // WHERE score > 90
    auto filter = make_unique<SimpleFilter>(
        std::move(join),
        [](DataChunk &chunk, idx_t row) {
            auto score = chunk.GetValue(3, row);  // score列
            return score.GetValue<double>() > 90.0;
        }
    );

    // SELECT s.name, c.name, s.score
    auto projection = make_unique<SimpleProjection>(
        std::move(filter),
        {1, 5, 3}  // name, class_name, score
    );

    // 执行查询
    DataChunk result;
    while (projection->GetNext(result)) {
        printf("Results:\n");
        for (idx_t row = 0; row < result.size(); row++) {
            printf("  %s, %s, %s\n",
                   result.GetValue(0, row).ToString().c_str(),
                   result.GetValue(1, row).ToString().c_str(),
                   result.GetValue(2, row).ToString().c_str());
        }
    }
}
```

#### 14.4 扩展练习

1. **完善HashJoin**
   - 支持多列Join条件
   - 支持LEFT/RIGHT/FULL OUTER JOIN

2. **完善HashAggregate**
   - 支持多个GROUP BY列
   - 支持COUNT、AVG、MIN、MAX等聚合函数
   - 实现并行聚合（多个本地hash表合并）

3. **添加ORDER BY**
   - 实现排序算子
   - 支持多列排序和ASC/DESC

4. **添加LIMIT**
   - 实现限制输出行数的算子

5. **查询优化**
   - 实现简单的Filter Pushdown
   - 实现Join重排序（根据表大小）

#### 14.5 第二周总结

| 主题 | 核心概念 | 关键文件 |
|------|----------|----------|
| Binder | 符号解析、类型推导 | `src/planner/binder/` |
| LogicalOperator | 逻辑计划表示 | `src/planner/logical_operator.hpp` |
| PhysicalOperator | 物理执行算子 | `src/execution/physical_operator.hpp` |
| TableScan | 表扫描、并行扫描 | `src/execution/operator/scan/` |
| Filter | SelectionVector、零拷贝 | `src/execution/operator/filter/` |
| Hash Join | Build/Probe、分区 | `src/execution/operator/join/` |
| Aggregation | State管理、并行合并 | `src/execution/operator/aggregate/` |

**下周预告：**
第三周将深入优化器，学习Filter Pushdown、Join Order优化、统计信息收集等查询优化技术。

---

## 第三周：优化器与性能

### Day 15: 优化器架构与规则系统

**学习目标：** 理解DuckDB优化器的整体架构和规则系统

#### 15.1 优化器架构

DuckDB的优化器采用**规则驱动**的设计：

```
LogicalOperator Tree (未优化)
        ↓
┌───────────────────────┐
│    Optimizer          │
│  ┌─────────────────┐  │
│  │ Rule 1: Filter  │  │
│  │    Pushdown     │  │
│  └─────────────────┘  │
│  ┌─────────────────┐  │
│  │ Rule 2: Join    │  │
│  │    Ordering     │  │
│  └─────────────────┘  │
│  ┌─────────────────┐  │
│  │ Rule 3: ...     │  │
│  └─────────────────┘  │
│  ... 40+ rules        │
└───────────────────────┘
        ↓
LogicalOperator Tree (优化后)
```

#### 15.2 Optimizer类

```cpp
// src/optimizer/optimizer.hpp
class Optimizer {
public:
    // 优化逻辑计划
    unique_ptr<LogicalOperator> Optimize(unique_ptr<LogicalOperator> plan);

private:
    ClientContext &context;
};
```

**优化Pass列表**（按执行顺序）：

```cpp
// src/optimizer/optimizer.cpp
unique_ptr<LogicalOperator> Optimizer::Optimize(unique_ptr<LogicalOperator> plan) {
    // 1. Expression重写
    ExpressionRewriter expr_rewriter;
    expr_rewriter.VisitOperator(*plan);

    // 2. Filter Pushdown
    FilterPushdown filter_pushdown;
    plan = filter_pushdown.Rewrite(std::move(plan));

    // 3. 删除未使用的列
    ColumnLifetimeAnalyzer column_lifetime;
    plan = column_lifetime.Optimize(std::move(plan));

    // 4. 公共子表达式消除
    CommonSubexpressionElimination cse;
    plan = cse.Optimize(std::move(plan));

    // 5. Join Order优化
    JoinOrderOptimizer join_order;
    plan = join_order.Optimize(std::move(plan));

    // 6. Statistics Propagation
    StatisticsPropagator stats_propagator;
    stats_propagator.PropagateStatistics(*plan);

    // 7. 常量折叠
    ConstantFoldingOptimizer constant_folder;
    plan = constant_folder.Optimize(std::move(plan));

    // 8. 移除不必要的Projection
    RemoveUnusedColumns unused_columns;
    plan = unused_columns.Optimize(std::move(plan));

    // ... 更多优化

    return plan;
}
```

#### 15.3 优化器基类

```cpp
// src/optimizer/rule.hpp
class Rule {
public:
    // 重写逻辑计划
    virtual unique_ptr<LogicalOperator> Rewrite(unique_ptr<LogicalOperator> op) = 0;

protected:
    // 递归访问
    void VisitOperator(LogicalOperator &op);

    // 重写特定算子类型
    virtual unique_ptr<Expression> VisitReplace(Expression &expr) {
        return nullptr;  // 不修改
    }
};
```

#### 15.4 表达式重写器

```cpp
// src/optimizer/expression_rewriter.cpp
class ExpressionRewriter : public LogicalOperatorVisitor {
public:
    void VisitOperator(LogicalOperator &op) override {
        // 访问所有表达式
        for (auto &expr : op.expressions) {
            expr = Rewrite(std::move(expr));
        }

        // 递归访问子算子
        for (auto &child : op.children) {
            VisitOperator(*child);
        }
    }

private:
    unique_ptr<Expression> Rewrite(unique_ptr<Expression> expr) {
        // 应用各种表达式重写规则
        expr = RewriteArithmetic(std::move(expr));
        expr = RewriteComparison(std::move(expr));
        expr = RewriteConjunction(std::move(expr));
        return expr;
    }

    // 算术表达式重写
    // 例如: x + 0 → x, x * 1 → x, x * 0 → 0
    unique_ptr<Expression> RewriteArithmetic(unique_ptr<Expression> expr) {
        if (expr->type != ExpressionType::OPERATOR_ADD &&
            expr->type != ExpressionType::OPERATOR_MULTIPLY) {
            return expr;
        }

        auto &op = (BoundOperatorExpression&)*expr;

        // x + 0 → x
        if (op.type == ExpressionType::OPERATOR_ADD) {
            if (IsConstantZero(op.children[1].get())) {
                return std::move(op.children[0]);
            }
            if (IsConstantZero(op.children[0].get())) {
                return std::move(op.children[1]);
            }
        }

        // x * 0 → 0
        if (op.type == ExpressionType::OPERATOR_MULTIPLY) {
            if (IsConstantZero(op.children[0].get()) ||
                IsConstantZero(op.children[1].get())) {
                return make_unique<BoundConstantExpression>(Value::INTEGER(0));
            }
        }

        // x * 1 → x
        if (op.type == ExpressionType::OPERATOR_MULTIPLY) {
            if (IsConstantOne(op.children[1].get())) {
                return std::move(op.children[0]);
            }
            if (IsConstantOne(op.children[0].get())) {
                return std::move(op.children[1]);
            }
        }

        return expr;
    }
};
```

**实践任务：**
1. 阅读 `src/optimizer/optimizer.cpp` 了解优化Pass顺序
2. 阅读 `src/optimizer/expression_rewriter.cpp`
3. 理解规则驱动的优化器架构

---

**(Day 16-30的详细内容将继续...)**

## 后续课程大纲

### Day 16: Filter Pushdown优化
- 下推谓词到数据源
- 处理JOIN和聚合
- 实现示例

### Day 17: Join Order优化
- 基数估计
- 动态规划算法
- 成本模型

### Day 18: 统计信息与基数估计
- 收集统计信息
- 直方图和HyperLogLog
- 选择率估算

### Day 19: 表达式优化与常量折叠
- 常量折叠
- 公共子表达式消除
- 表达式简化

### Day 20: 向量化执行深度解析
- SIMD指令
- 分支预测
- 缓存友好

### Day 21: 第三周总结
- 实现优化规则
- 性能测试

### Day 22-28: 存储引擎（第四周）
- RowGroup结构
- 压缩算法
- MVCC实现
- WAL和恢复
- Buffer管理

### Day 29-30: 扩展系统与总结
- 函数注册
- 扩展加载
- Mini-DuckDB项目

---

## 学习资源补充

1. **必读论文**:
   - "MonetDB/X100: Hyper-Pipelining Query Execution" - 向量化执行
   - "Morsel-Driven Parallelism" - 并行执行
   - "Push-Based Execution in DuckDB" - DuckDB论文

2. **参考项目**:
   - SQLite - 简单的嵌入式数据库
   - MonetDB - 列式数据库先驱
   - Velox - Facebook的向量化引擎

3. **在线课程**:
   - CMU 15-445 Database Systems
   - CMU 15-721 Advanced Database Systems

祝学习愉快！有问题可以查看DuckDB源码或访问官方Discord。
