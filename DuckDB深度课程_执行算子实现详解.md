# DuckDB 深度课程：执行算子实现详解

> 本课程深入讲解DuckDB各种执行算子的实现细节，包括数据扫描、过滤、投影、连接、聚合、排序等核心算子。

---

## 课程概览

### 学习目标

- 掌握PhysicalOperator基类的设计模式
- 理解各类算子的执行流程和状态管理
- 学习Push-based vs Pull-based执行模型
- 掌握并行执行机制
- 理解Pipeline的构建和调度
- 能够实现自定义执行算子

### 前置知识

- 向量化执行基础
- C++模板和智能指针
- 多线程编程基础
- 哈希表和排序算法

---

## 第一部分：PhysicalOperator基类设计

### 1.1 基类结构

```cpp
// src/include/duckdb/execution/physical_operator.hpp

class PhysicalOperator {
public:
    // 子算子列表（使用Arena链表管理）
    ArenaLinkedList<reference<PhysicalOperator>> children;

    // 算子类型
    PhysicalOperatorType type;

    // 输出类型
    vector<LogicalType> types;

    // 估算基数
    idx_t estimated_cardinality;

    // 算子名称（用于调试和EXPLAIN）
    string GetName() const;

    // ==================== 状态管理接口 ====================

    // 创建全局状态（跨线程共享）
    virtual unique_ptr<GlobalOperatorState> GetGlobalState(ClientContext &context);

    // 创建局部状态（线程私有）
    virtual unique_ptr<OperatorState> GetOperatorState(ExecutionContext &context);

    // ==================== 执行接口 ====================

    // 标准执行接口（Operator角色）
    virtual OperatorResultType Execute(ExecutionContext &context,
                                       DataChunk &input,
                                       DataChunk &output,
                                       GlobalOperatorState &gstate,
                                       OperatorState &state);

    // ==================== Source接口 ====================

    // 是否为数据源算子
    virtual bool IsSource() const { return false; }

    // 获取全局源状态
    virtual unique_ptr<GlobalSourceState> GetGlobalSourceState(ClientContext &context);

    // 获取局部源状态
    virtual unique_ptr<LocalSourceState> GetLocalSourceState(ExecutionContext &context);

    // 从数据源获取数据
    virtual SourceResultType GetData(ExecutionContext &context,
                                     DataChunk &chunk,
                                     OperatorSourceInput &input);

    // ==================== Sink接口 ====================

    // 是否为数据汇算子
    virtual bool IsSink() const { return false; }

    // 获取全局Sink状态
    virtual unique_ptr<GlobalSinkState> GetGlobalSinkState(ClientContext &context);

    // 获取局部Sink状态
    virtual unique_ptr<LocalSinkState> GetLocalSinkState(ExecutionContext &context);

    // 接收数据
    virtual SinkResultType Sink(ExecutionContext &context,
                                DataChunk &chunk,
                                OperatorSinkInput &input);

    // 完成Sink操作
    virtual SinkFinalizeResultType Finalize(ExecutionContext &context,
                                            OperatorSinkFinalizeInput &input);

    // ==================== 并行接口 ====================

    // 并行Source
    virtual ParallelSourceState ParallelSource();

    // 并行Sink
    virtual ParallelSinkState ParallelSink();

    // 并行Operator
    virtual ParallelOperatorState ParallelOperator();

    // ==================== 辅助接口 ====================

    // 估算并行度
    virtual idx_t MaxThreads(ClientContext &context);

    // 打印算子信息
    virtual string ToString() const;

    // 验证算子
    virtual void Verify();
};
```

### 1.2 算子类型枚举

```cpp
// src/include/duckdb/common/enums/operator_type.hpp

enum class PhysicalOperatorType : uint8_t {
    // ==================== 数据源算子 ====================
    PHYSICAL_TABLE_SCAN,           // 表扫描
    PHYSICAL_COLUMN_DATA_GET,       // 列数据获取
    PHYSICAL_EMPTY_RESULT,          // 空结果

    // ==================== 过滤和投影 ====================
    PHYSICAL_FILTER,               // 过滤
    PHYSICAL_PROJECTION,           // 投影
    PHYSICAL_LIMIT,                // 限制
    PHYSICAL_ORDER_BY,             // 排序
    PHYSICAL_TOP_N,                // Top-N

    // ==================== 连接算子 ====================
    PHYSICAL_HASH_JOIN,            // 哈希连接
    PHYSICAL_ASOF_JOIN,            // ASOF连接
    PHYSICAL_PIECEWISE_MERGE_JOIN,// 分段归并连接
    PHYSICAL_CROSS_PRODUCT,        // 笛卡尔积
    PHYSICAL_POSITIONAL_JOIN,       // 位置连接

    // ==================== 聚合算子 ====================
    PHYSICAL_HASH_AGGREGATE,       // 哈希聚合
    PHYSICAL_PERMUT_AGGREGATE,     // 排列聚合
    PHYSICAL_STREAMING_AGGREGATE,   // 流式聚合
    PHYSICAL_WINDOW,               // 窗口函数

    // ==================== 集合算子 ====================
    PHYSICAL_UNION,                // 并集
    PHYSICAL_EXCEPT,               // 差集
    PHYSICAL_INTERSECT,            // 交集

    // ==================== 修改算子 ====================
    PHYSICAL_INSERT,               // 插入
    PHYSICAL_DELETE,               // 删除
    PHYSICAL_UPDATE,               // 更新
    PHYSICAL_CREATE_TABLE,          // 创建表
    PHYSICAL_CREATE_INDEX,          // 创建索引

    // ==================== 其他算子 ====================
    PHYSICAL_DISTINCT,             // 去重
    PHYSICAL_SAMPLE,               // 采样
    PHYSICAL_UNGROUP,              // 解除分组
    PHYSICAL_EXPLAIN,              // 解释
    PHYSICAL_VACUUM,               // 清理
    PHYSICAL_DELIM_JOIN,           // Deliminate Join
    PHYSICAL_RECURSIVE_CTE,        // 递归CTE
};
```

### 1.3 三重接口设计模式

```
┌─────────────────────────────────────────────────────────┐
│              PhysicalOperator三重角色                   │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌──────────────┐   ┌──────────────┐   ┌───────────┐ │
│  │   Source     │   │   Operator   │   │   Sink    │ │
│  │  (生产者)    │   │  (转换者)    │   │  (消费者)  │ │
│  └──────┬───────┘   └──────┬───────┘   └─────┬─────┘ │
│         │                   │                  │         │
│         ↓                   ↓                  ↓         │
│  GetData()           Execute()          Sink()        │
│  获取数据            转换数据           接收数据        │
│                                                         │
└─────────────────────────────────────────────────────────┘

Source示例: Table Scan, CSV Scan
Operator示例: Filter, Projection
Sink示例: Insert, Create Index
```

### 1.4 状态管理层次

```cpp
// ==================== 全局状态（跨线程共享） ====================

class GlobalOperatorState {
public:
    virtual ~GlobalOperatorState() = default;

    // 返回最大线程数
    virtual idx_t MaxThreads(idx_t source_max_threads) {
        return source_max_threads;
    }

    // 合并局部状态
    virtual void Merge(LocalSinkState &state);

    // 重置状态
    virtual void Reset();
};

// ==================== 局部状态（线程私有） ====================

class OperatorState {
public:
    virtual ~OperatorState() = default;

    // Finalize状态
    virtual void Finalize(const PhysicalOperator &op,
                         ExecutionContext &context) {}

    // 析构时清理
    virtual void Destroy();
};

// ==================== Source状态 ====================

class GlobalSourceState {
public:
    virtual ~GlobalSourceState() = default;

    // 获取下一个分区
    virtual idx_t MaxThreads();
};

class LocalSourceState {
public:
    virtual ~LocalSourceState() = default;

    // 初始化
    virtual void Initialize(SourceStateType type);
};

// ==================== Sink状态 ====================

class GlobalSinkState {
public:
    virtual ~GlobalSinkState() = default;

    // 合并局部状态
    virtual void Merge(LocalSinkState &state);

    // Finalize
    virtual void Finalize(ClientContext &context);
};

class LocalSinkState {
public:
    virtual ~LocalSinkState() = default;

    // Sink数据
    virtual void Sink(DataChunk &chunk);
};
```

---

## 第二部分：扫描算子

### 2.1 Table Scan算子

```cpp
// src/execution/operator/scan/physical_table_scan.hpp

class PhysicalTableScan : public PhysicalOperator {
public:
    // 表函数
    TableFunction function;

    // 绑定数据
    unique_ptr<FunctionData> bind_data;

    // 返回类型
    vector<LogicalType> returned_types;

    // 列ID（投影下推）
    vector<ColumnIndex> column_ids;

    // 表过滤器（过滤下推）
    unique_ptr<TableFilterSet> table_filters;

    // 构造函数
    PhysicalTableScan(vector<LogicalType> types,
                      vector<ColumnIndex> column_ids,
                      unique_ptr<TableFilterSet> table_filters,
                      TableFunction function,
                      unique_ptr<FunctionData> bind_data,
                      idx_t estimated_cardinality);

    // ==================== Source接口实现 ====================

    bool IsSource() const override {
        return true;
    }

    unique_ptr<GlobalSourceState> GetGlobalSourceState(
        ClientContext &context) override {

        auto state = make_unique<TableScanGlobalState>();

        // 初始化表扫描器
        state->initialize = true;

        return state;
    }

    unique_ptr<LocalSourceState> GetLocalSourceState(
        ExecutionContext &context) override {

        return make_unique<TableScanLocalState>();
    }

    SourceResultType GetData(ExecutionContext &context,
                             DataChunk &output,
                             OperatorSourceInput &input) override;

    // ==================== 并行接口 ====================

    idx_t MaxThreads(ClientContext &context) override {
        // 计算可并行扫描的分区数
        idx_t row_group_count = GetTableRowGroupCount();

        // 每个线程至少扫描两个Row Group
        return MaxValue<idx_t>(row_group_count / 2, 1);
    }

private:
    // 表扫描全局状态
    struct TableScanGlobalState : public GlobalSourceState {
        idx_t row_group_index;
        idx_t max_row_group;
        bool initialize;
    };

    // 表扫描局部状态
    struct TableScanLocalState : public LocalSourceState {
        TableScanState scan_state;
        DataChunk chunk;
    };
};
```

#### Table Scan执行流程

```
┌─────────────────────────────────────────────────┐
│         Table Scan 执行流程                    │
├─────────────────────────────────────────────────┤
│                                                 │
│  1. 初始化阶段                                  │
│     ├── 创建GlobalSourceState                    │
│     ├── 确定扫描范围（Row Group列表）            │
│     └── 分配给各线程                            │
│                                                 │
│  2. 扫描阶段                                    │
│     ├── 获取当前Row Group                       │
│     ├── 应用table_filters（过滤下推）              │
│     ├── 读取投影列（列裁剪）                     │
│     ├── 解压数据（如果需要）                      │
│     └── 填充DataChunk                          │
│                                                 │
│  3. 完成                                       │
│     └── 所有Row Group扫描完成                    │
│                                                 │
└─────────────────────────────────────────────────┘

并行扫描示例：

表有100个Row Groups，4个线程：

Thread 1: Row Groups 0-24
Thread 2: Row Groups 25-49
Thread 3: Row Groups 50-74
Thread 4: Row Groups 75-99

每个线程独立扫描自己的Row Group，无锁竞争
```

### 2.2 CSV扫描算子

```cpp
// src/execution/operator/scan/physical_csv_scan.hpp

class PhysicalCSVScan : public PhysicalTableScan {
public:
    // CSV文件路径
    string file_path;

    // CSV选项
    CSVReaderOptions options;

    SourceResultType GetData(ExecutionContext &context,
                             DataChunk &output,
                             OperatorSourceInput &input) override {

        auto &gstate = input.global_state.Cast<CSVGlobalState>();
        auto &lstate = input.local_state.Cast<CSVLocalState>();

        // 1. 初始化CSV读取器
        if (!lstate.initialized) {
            lstate.csv_reader = make_unique<CSVReader>(
                file_path, options, gstate.column_ids
            );
            lstate.initialized = true;
        }

        // 2. 读取一批数据
        lstate.csv_reader->Read(output);

        // 3. 检查是否完成
        if (output.size() == 0) {
            return SourceResultType::FINISHED;
        }

        return SourceResultType::HAVE_MORE_DATA;
    }

private:
    struct CSVGlobalState : public GlobalSourceState {
        // CSV文件的字节偏移分配
        vector<idx_t> byte_offsets;
    };

    struct CSVLocalState : public LocalSourceState {
        unique_ptr<CSVReader> csv_reader;
        bool initialized = false;
    };
};
```

#### CSV扫描优化

```cpp
// 并行CSV扫描实现

void ParallelCSVScan::Initialize() {
    // 1. 计算文件大小
    auto file_size = GetFileSize(file_path);

    // 2. 按线程数划分字节范围
    idx_t chunk_size = file_size / thread_count;

    for (idx_t i = 0; i < thread_count; i++) {
        idx_t start = i * chunk_size;
        idx_t end = (i == thread_count - 1) ? file_size : (i + 1) * chunk_size;

        // 3. 调整到行边界
        // 向后搜索换行符
        if (i > 0) {
            start = FindNextNewline(file_path, start);
        }

        byte_offsets.push_back({start, end});
    }
}

// 每个线程独立扫描自己的字节范围
// 避免锁竞争，充分利用并行I/O
```

---

## 第三部分：过滤和投影算子

### 3.1 Filter算子

```cpp
// src/execution/operator/filter/physical_filter.hpp

class PhysicalFilter : public CachingPhysicalOperator {
public:
    // 过滤表达式
    unique_ptr<Expression> expression;

    PhysicalFilter(vector<unique_ptr<Expression>> select_expressions,
                   idx_t estimated_cardinality)
        : CachingPhysicalOperator(PhysicalOperatorType::FILTER),
          expressions(std::move(select_expressions)) {

        D_ASSERT(expressions.size() == 1);
        expression = expressions[0]->Copy();
    }

    // ==================== 执行接口 ====================

    unique_ptr<OperatorState> GetOperatorState(
        ExecutionContext &context) override {

        auto state = make_unique<FilterOperatorState>();

        // 初始化表达式执行器
        state->executor.Initialize(expression);

        return state;
    }

    OperatorResultType ExecuteInternal(ExecutionContext &context,
                                      DataChunk &input,
                                      DataChunk &output,
                                      GlobalOperatorState &gstate,
                                      OperatorState &state_p) const override;

private:
    vector<unique_ptr<Expression>> expressions;

    struct FilterOperatorState : public OperatorState {
        ExpressionExecutor executor;
        SelectionVector sel;
        DataChunk expr_chunk;
    };
};
```

#### Filter执行实现

```cpp
// src/execution/operator/filter/physical_filter.cpp

OperatorResultType PhysicalFilter::ExecuteInternal(
    ExecutionContext &context,
    DataChunk &input,
    DataChunk &output,
    GlobalOperatorState &gstate,
    OperatorState &state_p) const {

    auto &state = state_p.Cast<FilterOperatorState>();

    // 1. 执行过滤表达式
    // 结果存储在expr_chunk中，类型为BOOLEAN
    state.exprector.ExecuteExpression(input, state.expr_chunk);

    // 2. 从布尔向量提取SelectionVector
    auto &result_vector = state.expr_chunk.data[0];
    D_ASSERT(result_vector.type == LogicalType::BOOLEAN);

    // 3. 执行过滤选择
    idx_t result_count = FilterSelection(
        result_vector,
        input.size(),
        state.sel
    );

    // 4. 处理结果
    if (result_count == 0) {
        // 没有行通过过滤
        output.SetCardinality(0);
        return OperatorResultType::NEED_MORE_INPUT;

    } else if (result_count == input.size()) {
        // 所有行都通过过滤
        output.Reference(input);
        return OperatorResultType::HAVE_MORE_OUTPUT;

    } else {
        // 部分行通过过滤：切片
        output.Slice(input, state.sel, result_count);
        return OperatorResultType::HAVE_MORE_OUTPUT;
    }
}

// 过滤选择实现
static idx_t FilterSelection(Vector &vector,
                             idx_t count,
                             SelectionVector &sel) {

    // 统一格式化
    UnifiedVectorFormat vdata;
    vector.ToUnifiedFormat(count, vdata);

    idx_t sel_count = 0;

    // 检查NULL值
    if (vdata.validity.AllValid()) {
        // 无NULL：直接过滤
        for (idx_t i = 0; i < count; i++) {
            idx_t idx = vdata.sel->get_index(i);
            if (((bool *)vdata.data)[idx]) {
                sel.set_index(sel_count++, i);
            }
        }
    } else {
        // 有NULL：跳过NULL
        for (idx_t i = 0; i < count; i++) {
            idx_t idx = vdata.sel->get_index(i);
            if (vdata.validity.RowIsValid(idx) &&
                ((bool *)vdata.data)[idx]) {
                sel.set_index(sel_count++, i);
            }
        }
    }

    return sel_count;
}
```

### 3.2 Projection算子

```cpp
// src/execution/operator/projection/physical_projection.hpp

class PhysicalProjection : public PhysicalOperator {
public:
    // 选择列表（投影表达式）
    vector<unique_ptr<Expression>> select_list;

    PhysicalProjection(vector<unique_ptr<Expression>> select_list,
                      vector<LogicalType> types,
                      idx_t estimated_cardinality)
        : PhysicalOperator(PhysicalOperatorType::PHYSICAL_PROJECTION,
                          std::move(types), estimated_cardinality),
          select_list(std::move(select_list)) {

        // 设置投影哈希（用于优化）
        projection_hash = ComputeProjectionHash();
    }

    unique_ptr<OperatorState> GetOperatorState(
        ExecutionContext &context) override {

        auto state = make_unique<ProjectionOperatorState>();

        // 初始化所有表达式执行器
        for (auto &expr : select_list) {
            state->executor.AddExpression(*expr);
        }

        return state;
    }

    OperatorResultType Execute(ExecutionContext &context,
                               DataChunk &input,
                               DataChunk &output,
                               GlobalOperatorState &gstate,
                               OperatorState &state_p) override {

        auto &state = state_p.Cast<ProjectionOperatorState>();

        // 1. 确保输出已初始化
        if (output.data.empty()) {
            output.Initialize(types);
        }

        // 2. 执行所有投影表达式
        state.executor.Execute(input, output);

        // 3. 设置基数
        output.SetCardinality(input.size());

        return OperatorResultType::HAVE_MORE_OUTPUT;
    }

private:
    vector<unique_ptr<Expression>> select_list;
    idx_t projection_hash;
};
```

---

## 第四部分：连接算子

### 4.1 Hash Join算子

```cpp
// src/execution/operator/join/physical_hash_join.hpp

class PhysicalHashJoin : public PhysicalComparisonJoin {
public:
    // 连接条件类型
    vector<LogicalType> condition_types;

    // 载荷列
    JoinProjectionColumns payload_columns;

    PhysicalHashJoin(LogicalOperator &op,
                     unique_ptr<PhysicalOperator> left,
                     unique_ptr<PhysicalOperator> right,
                     vector<JoinCondition> conditions,
                     JoinType join_type,
                     vector<unique_ptr<Expression>> projections)
        : PhysicalComparisonJoin(PhysicalOperatorType::PHYSICAL_HASH_JOIN,
                                op.type, std::move(left), std::move(right),
                                std::move(conditions), join_type) {}

    // ==================== Sink接口（构建阶段） ====================

    bool IsSink() const override {
        return true;
    }

    unique_ptr<GlobalSinkState> GetGlobalSinkState(
        ClientContext &context) override {

        auto state = make_unique<HashJoinGlobalState>();

        // 初始化哈希表
        state->hash_table = InitializeHashTable(context);

        // 初始化分区
        state->partitioned_hash_table =
            make_unique<RadixPartitionedHashTable>(*state->hash_table);

        return state;
    }

    SinkResultType Sink(ExecutionContext &context,
                         DataChunk &chunk,
                         OperatorSinkInput &input) override {

        auto &gstate = input.global_state.Cast<HashJoinGlobalState>();
        auto &lstate = input.local_state.Cast<HashJoinLocalState>();

        // 1. 确定是左表还是右表
        // Hash Join通常对右表构建哈希表
        bool is_build = (input.intermediate_state.chunk_index == 1);

        if (is_build) {
            // 2. 构建哈希表
            gstate.partitioned_hash_table->Build(chunk, lstate);

            // 3. 检查内存使用
            if (gstate.partitioned_hash_table->MemoryExceeded()) {
                // 执行分区溢出到磁盘
                gstate.partitioned_hash_table->Partition();
            }
        }

        return SinkResultType::NEED_MORE_INPUT;
    }

    // ==================== Source接口（探测阶段） ====================

    bool IsSource() const override {
        return true;
    }

    unique_ptr<GlobalSourceState> GetGlobalSourceState(
        ClientContext &context) override {

        auto &gstate = sink_state->Cast<HashJoinGlobalState>();
        return gstate.hash_table->GetGlobalSourceState(context);
    }

    SourceResultType GetData(ExecutionContext &context,
                             DataChunk &chunk,
                             OperatorSourceInput &input) override;

private:
    struct HashJoinGlobalState : public GlobalSinkState {
        unique_ptr<JoinHashTable> hash_table;
        unique_ptr<RadixPartitionedHashTable> partitioned_hash_table;
    };

    struct HashJoinLocalState : public LocalSinkState {
        unique_ptr<JoinHashTable::LocalState> local_hash_table;
    };
};
```

#### Hash Join执行流程

```
┌─────────────────────────────────────────────────────────┐
│          Hash Join 双阶段执行流程                      │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  阶段1：Sink（构建哈希表）                            │
│  ┌───────────────┐                                   │
│  │   Right Table │                                   │
│  │   Build Side  │                                   │
│  └───────┬───────┘                                   │
│          ↓                                            │
│  ┌───────────────────────────────────────────────┐     │
│  │  1. 计算Hash Values                          │     │
│  │  2. Radix Partitioning（可选）                │     │
│  │  3. Build Hash Table                         │     │
│  │     - Key: Join Column Hash                  │     │
│  │     - Value: Row ID + Payload               │     │
│  └───────────────────────────────────────────────┘     │
│                                                         │
│  阶段2：Source（探测哈希表）                           │
│  ┌───────────────┐                                   │
│  │   Left Table   │                                   │
│  │  Probe Side    │                                   │
│  └───────┬───────┘                                   │
│          ↓                                            │
│  ┌───────────────────────────────────────────────┐     │
│  │  1. 计算Left Table的Hash Values              │     │
│  │  2. Probe Hash Table                         │     │
│  │  3. 处理匹配结果                            │     │
│  │     - Inner Join: 只输出匹配                 │     │
│  │     - Left Join: 输出所有左表行             │     │
│  │     - Right/Full Join: 输出未匹配右表行      │     │
│  └───────────────────────────────────────────────┘     │
│                                                         │
└─────────────────────────────────────────────────────────┘

示例：SELECT * FROM orders JOIN customers ON
  orders.customer_id = customers.id

Sink阶段（构建）：
customers表：
┌────┬───────┐
│ id │  name  │
├────┼───────┤
│ 1  │ Alice  │
│ 2  │ Bob    │
│ 3  │ Carol  │
└────┴───────┘

哈希表（Hash(customer_id))：
hash(1) → [Row1, name="Alice"]
hash(2) → [Row2, name="Bob"]
hash(3) → [Row3, name="Carol"]

Source阶段（探测）：
orders表：
┌──────────┬─────────────┐
│ order_id │ customer_id │
├──────────┼─────────────┤
│ 100      │ 2           │
│ 101      │ 1           │
│ 102      │ 5           │
└──────────┴─────────────┘

探测：
Row 100: hash(2) → Bob → 输出 (100, 2, Bob)
Row 101: hash(1) → Alice → 输出 (101, 1, Alice)
Row 102: hash(5) → 未找到 → (Inner Join跳过, Left Join输出NULL)
```

### 4.2 Sort Merge Join算子

```cpp
// src/execution/operator/join/physical_piecewise_merge_join.hpp

class PhysicalPiecewiseMergeJoin : public PhysicalRangeJoin {
public:
    PhysicalPiecewiseMergeJoin(LogicalOperator &op,
                               unique_ptr<PhysicalOperator> left,
                               unique_ptr<PhysicalOperator> right,
                               vector<JoinCondition> conditions,
                               JoinType join_type)
        : PhysicalRangeJoin(PhysicalOperatorType::PHYSICAL_PIECEWISE_MERGE_JOIN,
                            op.type, std::move(left), std::move(right),
                            std::move(conditions), join_type) {}

    // ==================== Sink接口（排序阶段） ====================

    SinkResultType Sink(ExecutionContext &context,
                         DataChunk &chunk,
                         OperatorSinkInput &input) override;

    SinkFinalizeResultType Finalize(ExecutionContext &context,
                                     OperatorSinkFinalizeInput &input) override;

    // ==================== Source接口（合并阶段） ====================

    SourceResultType GetData(ExecutionContext &context,
                             DataChunk &chunk,
                             OperatorSourceInput &input) override;

private:
    struct PiecewiseMergeJoinGlobalState : public GlobalSinkState {
        // 排序后的左表
        unique_ptr<SortedDataContainer> left_data;
        // 排序后的右表
        unique_ptr<SortedDataContainer> right_data;
    };
};
```

#### Sort Merge Join执行流程

```
┌─────────────────────────────────────────────────────────┐
│       Sort Merge Join 执行流程                          │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  阶段1：Sink（收集和排序）                            │
│  ┌───────────────┐     ┌───────────────┐              │
│  │   Left Table   │     │   Right Table  │              │
│  └───────┬───────┘     └───────┬───────┘              │
│          │                      │                       │
│          ↓                      ↓                       │
│  ┌───────────────┐     ┌───────────────┐              │
│  │  Sort by Key  │     │  Sort by Key  │              │
│  └───────┬───────┘     └───────┬───────┘              │
│          │                      │                       │
│          └──────────┬───────────┘                       │
│                     ↓                                    │
│            ┌─────────────────┐                           │
│            │  Sorted Data    │                           │
│            └─────────────────┘                           │
│                                                         │
│  阶段2：Source（合并）                                 │
│  ┌─────────────────────────────────────────────┐          │
│  │  双指针合并                                │          │
│  │                                            │          │
│  │  Left:  [1, 3, 5, 7, 9]                  │          │
│  │  Right: [2, 3, 5, 8]                      │          │
│  │                                            │          │
│  │  i=0, j=0: Left[0]=1 < Right[0]=2         │          │
│  │           i++                              │          │
│  │                                            │          │
│  │  i=1, j=0: Left[1]=3 > Right[0]=2         │          │
│  │           j++                              │          │
│  │                                            │          │
│  │  i=1, j=1: Left[1]=3 == Right[1]=3        │          │
│  │           输出 (3,3), i++, j++            │          │
│  │                                            │          │
│  └─────────────────────────────────────────────┘          │
│                                                         │
└─────────────────────────────────────────────────────────┘

优势：
• 适用于有序数据（避免排序开销）
• 内存效率高（无需哈希表）
• 支持范围连接

劣势：
• 排序开销大
• 不适合一次性格式
```

---

## 第五部分：聚合算子

### 5.1 Hash Aggregate算子

```cpp
// src/execution/operator/aggregate/physical_hash_aggregate.hpp

class PhysicalHashAggregate : public PhysicalOperator {
public:
    // 分组聚合数据
    GroupedAggregateData grouped_aggregate_data;

    // 分组集合
    vector<GroupingSet> grouping_sets;

    PhysicalHashAggregate(vector<unique_ptr<Expression>> select_list,
                         vector<unique_ptr<Expression>> groups,
                         vector<unique_ptr<Expression>> group_by_names,
                         vector<GroupingSet> grouping_sets,
                         idx_t estimated_cardinality)
        : PhysicalOperator(PhysicalOperatorType::PHYSICAL_HASH_AGGREGATE,
                          types, estimated_cardinality),
          grouping_sets(std::move(grouping_sets)) {}

    // ==================== Sink接口 ====================

    bool IsSink() const override {
        return true;
    }

    unique_ptr<GlobalSinkState> GetGlobalSinkState(
        ClientContext &context) override {

        auto state = make_unique<HashAggregateGlobalState>();

        // 初始化分组哈希表
        state->grouping_sets.resize(grouping_sets.size());

        for (size_t i = 0; i < grouping_sets.size(); i++) {
            state->grouping_sets[i] =
                make_unique<HashAggregateGroupingData>(grouping_sets[i]);
        }

        return state;
    }

    SinkResultType Sink(ExecutionContext &context,
                         DataChunk &chunk,
                         OperatorSinkInput &input) override {

        auto &gstate = input.global_state.Cast<HashAggregateGlobalState>();
        auto &lstate = input.local_state.Cast<HashAggregateLocalState>();

        // 1. 计算分组键
        DataChunk group_chunk;
        ComputeGroupKeys(chunk, group_chunk, lstate);

        // 2. 计算哈希值
        Vector hashes(LogicalType::HASH);
        ComputeHashes(group_chunk, hashes);

        // 3. 查找/创建分组
        for (idx_t i = 0; i < chunk.size(); i++) {
            idx_t group_idx = FindOrCreateGroup(
                gstate, lstate, group_chunk, hashes, i
            );

            // 4. 更新聚合状态
            for (idx_t agg_idx = 0; agg_idx < aggregates.size(); agg_idx++) {
                aggregates[agg_idx]->Update(
                    *lstate.aggregate_states[agg_idx],
                    group_idx,
                    chunk,
                    i
                );
            }
        }

        return SinkResultType::NEED_MORE_INPUT;
    }

    // ==================== Source接口 ====================

    bool IsSource() const override {
        return true;
    }

    SourceResultType GetData(ExecutionContext &context,
                             DataChunk &chunk,
                             OperatorSourceInput &input) override;

private:
    struct HashAggregateGlobalState : public GlobalSinkState {
        // 多个分组集合（GROUPING SETS）
        vector<unique_ptr<HashAggregateGroupingData>> grouping_sets;

        // 总分组数
        idx_t total_groups = 0;
    };

    struct HashAggregateLocalState : public LocalSinkState {
        // 聚合状态（每个聚合一个）
        vector<unique_ptr<AggregateStateData>> aggregate_states;

        // 分组键缓冲区
        DataChunk group_chunk;
    };
};
```

#### Hash Aggregate执行流程

```
┌─────────────────────────────────────────────────────────┐
│        Hash Aggregate 执行流程                          │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  示例：SELECT department, SUM(salary)                   │
│         FROM employees                                   │
│         GROUP BY department                              │
│                                                         │
│  Sink阶段：                                            │
│  ┌─────────────────────────────────────────────┐          │
│  │  Input DataChunk:                         │          │
│  │  ┌─────────────────────────────────────┐    │          │
│  │  │ name │ dept │   salary            │    │          │
│  │  ├─────────────────────────────────────┤    │          │
│  │  │ Alice│ Sales│ 50000              │    │          │
│  │  │ Bob  │ IT   │ 60000              │    │          │
│  │  │ Carol│ Sales│ 55000              │    │          │
│  │  └─────────────────────────────────────┘    │          │
│  └─────────────────────────────────────────────┘          │
│          │                                              │
│          ↓                                              │
│  ┌─────────────────────────────────────────────┐          │
│  │  1. 提取分组键 (department)                │          │
│  │  2. 计算Hash(department)                   │          │
│  │  3. 查找哈希表                            │          │
│  │  4. 如果存在：更新聚合                      │          │
│  │     如果不存在：创建新分组                    │          │
│  └─────────────────────────────────────────────┘          │
│                                                         │
│  哈希表状态：                                          │
│  ┌─────────────────────────────────────┐                   │
│  │ Hash("Sales") → {count: 2, sum: 105000} │          │
│  │ Hash("IT")    → {count: 1, sum: 60000}   │          │
│  │ Hash("HR")    → {count: 0, sum: 0}       │          │
│  └─────────────────────────────────────┘                   │
│                                                         │
│  Source阶段：                                          │
│  ┌─────────────────────────────────────────────┐          │
│  │  遍历哈希表输出结果                      │          │
│  │                                            │          │
│  │  department │ SUM(salary)                   │          │
│  │  ──────────────────────────────             │          │
│  │  Sales      │ 105000                       │          │
│  │  IT         │ 60000                        │          │
│  │  HR         │ 0                            │          │
│  └─────────────────────────────────────────────┘          │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 5.2 Streaming Aggregate（流式聚合）

```cpp
// src/execution/operator/aggregate/physical_streaming_aggregate.hpp

class PhysicalStreamingAggregate : public PhysicalOperator {
public:
    // 流式聚合：适用于已排序数据

    StreamingAggregateFunction aggregate;

    SinkResultType Sink(ExecutionContext &context,
                         DataChunk &chunk,
                         OperatorSinkInput &input) override {

        auto &lstate = input.local_state.Cast<StreamingAggregateLocalState>();

        // 1. 检查是否需要输出
        if (IsNewGroup(lstate, chunk)) {
            // 输出当前分组结果
            FlushCurrentGroup(lstate);
        }

        // 2. 更新当前分组
        UpdateAggregate(lstate, chunk);

        return SinkResultType::NEED_MORE_INPUT;
    }

private:
    bool IsNewGroup(StreamingAggregateLocalState &state,
                    DataChunk &chunk) {
        // 检查分组键是否变化
        // 如果变化，说明新的分组开始
        return state.current_group_key != chunk.group_key;
    }
};
```

---

## 第六部分：排序和限制算子

### 6.1 Order By算子

```cpp
// src/execution/operator/order/physical_order.hpp

class PhysicalOrder : public PhysicalOperator {
public:
    // 排序条件
    vector<BoundOrderByNode> orders;

    // 投影列
    vector<idx_t> projections;

    PhysicalOrder(vector<LogicalType> types,
                   vector<BoundOrderByNode> orders,
                   idx_t estimated_cardinality,
                   vector<idx_t> projections)
        : PhysicalOperator(PhysicalOperatorType::PHYSICAL_ORDER_BY,
                          std::move(types), estimated_cardinality),
          orders(std::move(orders)),
          projections(std::move(projections)) {}

    // ==================== Sink接口（收集阶段） ====================

    SinkResultType Sink(ExecutionContext &context,
                         DataChunk &chunk,
                         OperatorSinkInput &input) override {

        auto &gstate = input.global_state.Cast<OrderGlobalState>();

        // 1. 应用投影
        DataChunk projected_chunk;
        ProjectColumns(chunk, projections, projected_chunk);

        // 2. 添加到排序缓冲区
        gstate.payload->Append(projected_chunk);

        // 3. 检查内存使用
        if (gstate.payload->SizeInBytes() > context.config.max_memory) {
            // 执行外部排序
            gstate.sort_state->ExecuteSort();
            SpillToDisk(gstate);
        }

        return SinkResultType::NEED_MORE_INPUT;
    }

    SinkFinalizeResultType Finalize(ExecutionContext &context,
                                     OperatorSinkFinalizeInput &input) override {

        auto &gstate = input.global_state.Cast<OrderGlobalState>();

        // 1. 执行最终排序
        gstate.sort_state->ExecuteSort();

        // 2. 如果有溢出，归并排序文件
        if (gstate.sort_state->external) {
            MergeSortedRuns(gstate);
        }

        return SinkFinalizeResultType::FINISHED;
    }

    // ==================== Source接口（输出阶段） ====================

    SourceResultType GetData(ExecutionContext &context,
                             DataChunk &chunk,
                             OperatorSourceInput &input) override {

        auto &gstate = input.global_state.Cast<OrderGlobalState>();
        auto &lstate = input.local_state.Cast<OrderLocalState>();

        // 从排序后的数据中读取
        idx_t count = gstate.sorted_data->Read(lstate.position, chunk);

        if (count == 0) {
            return SourceResultType::FINISHED;
        }

        lstate.position += count;
        return SourceResultType::HAVE_MORE_DATA;
    }

private:
    struct OrderGlobalState : public GlobalSinkState {
        // 排序状态
        unique_ptr<SortedData> sorted_data;

        // 有效载荷（实际数据）
        unique_ptr<RowDataCollection> payload;

        // 排序器
        unique_ptr<LocalSortState> sort_state;

        // 外部排序文件
        vector<unique_ptr<BufferedData>> spill_runs;
    };
};
```

#### 排序算法选择

```
┌─────────────────────────────────────────────────────────┐
│            DuckDB 排序算法选择                         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  数据大小决策树：                                       │
│                                                         │
│                        ┌───────┐                      │
│                        │数据量 │                        │
│                        └───┬───┘                      │
│                            │                            │
│               ┌────────────┴────────────┐               │
│               │                         │                │
│          小于内存                   超过内存              │
│               │                         │                │
│               ↓                         ↓                │
│      ┌─────────────┐         ┌───────────────┐        │
│      │ 内存排序    │         │ 外部排序      │        │
│      │ (In-Memory) │         │ (External)    │        │
│      └──────┬──────┘         └───────┬───────┘        │
│             │                         │                │
│             ↓                         ↓                │
│      ┌─────────────┐         ┌───────────────┐        │
│      │ 快速排序    │         │ 归并排序      │        │
│      │ (QuickSort) │         │ (Merge Sort)  │        │
│      └─────────────┘         └───────────────┘        │
│                                                         │
│  复杂度分析：                                          │
│  • 内存排序：O(n log n)                               │
│  • 外部排序：O(n log n) + I/O开销                     │
│                                                         │
└─────────────────────────────────────────────────────────┘

外部排序流程：

1. 分段排序：
   Input: [5, 2, 8, 1, 9, 3, 7, 4, 6]

   Memory Limit: 3 elements

   Run 1: [5, 2, 8] → Sort → [2, 5, 8] → Disk
   Run 2: [1, 9, 3] → Sort → [1, 3, 9] → Disk
   Run 3: [7, 4, 6] → Sort → [4, 6, 7] → Disk

2. 归并：
   Disk Files:
   Run 1: [2, 5, 8]
   Run 2: [1, 3, 9]
   Run 3: [4, 6, 7]

   Merge: [1, 2, 3, 4, 5, 6, 7, 8, 9]
```

### 6.2 Limit算子

```cpp
// src/execution/operator/limit/physical_limit.hpp

class PhysicalLimit : public PhysicalOperator {
public:
    // 限制值
    BoundLimitNode limit_val;

    // 偏移值
    BoundLimitNode offset_val;

    PhysicalLimit(BoundLimitNode limit_val,
                   BoundLimitNode offset_val,
                   idx_t estimated_cardinality)
        : PhysicalOperator(PhysicalOperatorType::PHYSICAL_LIMIT,
                          {}, estimated_cardinality),
          limit_val(std::move(limit_val)),
          offset_val(std::move(offset_val)) {}

    // ==================== Sink接口 ====================

    SinkResultType Sink(ExecutionContext &context,
                         DataChunk &chunk,
                         OperatorSinkInput &input) override {

        auto &gstate = input.global_state.Cast<LimitGlobalState>();
        auto &lstate = input.local_state.Cast<LimitLocalState>();

        // 1. 计算有效行数
        idx_t available = gstate.current_count - lstate.skip_count;
        idx_t remaining = gstate.limit - lstate.output_count;
        idx_t copy_count = MinValue(available, remaining);

        if (copy_count > 0) {
            // 2. 复制有效行
            DataChunk slice;
            chunk.Slice(lstate.skip_count, copy_count);
            gstate.data->Append(slice);

            lstate.output_count += copy_count;
        }

        gstate.current_count += chunk.size();

        // 3. 检查是否达到限制
        if (lstate.output_count >= gstate.limit) {
            return SinkResultType::FINISHED;
        }

        return SinkResultType::NEED_MORE_INPUT;
    }

    // ==================== Source接口 ====================

    SourceResultType GetData(ExecutionContext &context,
                             DataChunk &chunk,
                             OperatorSourceInput &input) override {

        auto &gstate = input.global_state.Cast<LimitGlobalState>();
        auto &lstate = input.local_state.Cast<LimitLocalState>();

        // 从收集的数据中读取
        idx_t count = gstate.data->Read(lstate.position, chunk);
        lstate.position += count;

        if (lstate.position >= gstate.limit) {
            return SourceResultType::FINISHED;
        }

        return SourceResultType::HAVE_MORE_DATA;
    }

private:
    struct LimitGlobalState : public GlobalSinkState {
        // 收集的数据
        unique_ptr<RowDataCollection> data;

        // 当前计数
        idx_t current_count = 0;

        // 限制值
        idx_t limit;
    };

    struct LimitLocalState : public LocalSinkState {
        // 跳过的行数（offset）
        idx_t skip_count = 0;

        // 输出的行数
        idx_t output_count = 0;

        // 读取位置
        idx_t position = 0;
    };
};
```

---

## 第七部分：Pipeline执行模型

### 7.1 Pipeline结构

```cpp
// src/execution/physical_plan/pipeline.hpp

class Pipeline : public enable_shared_from_this<Pipeline> {
public:
    // 关联的执行器
    Executor &executor;

    // 算子列表（从source到sink）
    vector<reference<PhysicalOperator>> operators;

    // 源算子
    PhysicalOperator *source;

    // 汇算子
    PhysicalOperator *sink;

    // 依赖的Pipeline
    vector<shared_ptr<Pipeline>> dependencies;

    // 构建Pipeline
    void BuildPipeline(PhysicalOperator &op);

    // 获取Pipeline的线程数
    idx_t GetMaxThreads();

    // 执行Pipeline
    void Execute();
};
```

### 7.2 PipelineExecutor

```cpp
// src/execution/physical_plan/pipeline_execute.hpp

class PipelineExecutor {
public:
    PipelineExecutor(Pipeline &pipeline,
                     ExecutionContext &context)
        : pipeline(pipeline), context(context) {

        Initialize();
    }

    // 执行Pipeline
    PipelineExecuteResult Execute() {
        // 1. 初始化
        InitializeStates();

        // 2. 主循环
        while (true) {
            auto result = ExecuteStep();

            if (result == PipelineExecuteResult::FINISHED) {
                break;
            }
        }

        // 3. Finalize
        FinalizeStates();

        return PipelineExecuteResult::FINISHED;
    }

private:
    PipelineExecuteResult ExecuteStep() {
        // 1. 从源获取数据
        auto source_result = FetchFromSource();

        if (source_result == SourceResultType::FINISHED) {
            // 2. 推送剩余数据
            FinalizeSink();
            return PipelineExecuteResult::FINISHED;
        }

        // 3. 推送数据到汇
        PushToSink();

        return PipelineExecuteResult::HAVE_MORE_DATA;
    }

    SourceResultType FetchFromSource() {
        DataChunk chunk;

        auto result = source->GetData(
            context,
            chunk,
            source_input
        );

        if (result == SourceResultType::HAVE_MORE_DATA) {
            current_chunk = std::move(chunk);
        }

        return result;
    }

    void PushToSink() {
        // 执行中间算子
        DataChunk result_chunk = current_chunk;

        for (auto &op_ref : pipeline.operators) {
            auto &op = op_ref.get();

            op.Execute(
                context,
                result_chunk,
                result_chunk,
                *global_state,
                *local_state
            );
        }

        // 发送到汇
        sink->Sink(
            context,
            result_chunk,
            sink_input
        );
    }

    Pipeline &pipeline;
    ExecutionContext &context;

    unique_ptr<GlobalSourceState> global_source_state;
    unique_ptr<LocalSourceState> local_source_state;

    unique_ptr<GlobalSinkState> global_sink_state;
    unique_ptr<LocalSinkState> local_sink_state;

    unique_ptr<GlobalOperatorState> global_state;
    unique_ptr<OperatorState> local_state;

    DataChunk current_chunk;
};
```

### 7.3 Push-based vs Pull-based执行

```
┌─────────────────────────────────────────────────────────┐
│       Push-based vs Pull-based 执行模型                   │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Pull-based（传统Volcano模型）:                       │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐        │
│  │  Scan   │───→│ Filter  │───→│ Project │         │
│  └─────────┘    └─────────┘    └─────────┘         │
│       ↓              ↓                ↓                 │
│   调用Next()    调用Next()      调用Next()             │
│   返回1行       返回1行         返回1行                │
│                                                         │
│  Push-based（DuckDB模型）:                           │
│  ┌─────────┐    ┌─────────┐    ┌─────────┐        │
│  │  Scan   │───→│ Filter  │───→│ Project │         │
│  └─────────┘    └─────────┘    └─────────┘         │
│       ↑              ↑                ↑                 │
│   Execute()     Execute()       Execute()              │
│   输出2048行    输出2048行      输出2048行            │
│                                                         │
│  对比：                                                │
│  ┌───────────────────┬───────────────────┐              │
│  │ Pull-based       │ Push-based       │              │
│  ├───────────────────┼───────────────────┤              │
│  │ 每次返回1行      │ 每次返回2048行    │              │
│  │ 函数调用频繁      │ 函数调用少        │              │
│  │ 虚函数开销大      │ 内联优化          │              │
│  │ 控制流复杂       │ 数据流简单        │              │
│  │ 难以并行化       │ 易于并行化        │              │
│  └───────────────────┴───────────────────┘              │
│                                                         │
└─────────────────────────────────────────────────────────┘

DuckDB Push-based执行流程：

1. Source扫描2048行 → DataChunk
2. 推送到Filter → 过滤后的DataChunk
3. 推送到Project → 投影后的DataChunk
4. 推送到Sink → 输出或存储

优势：
• 批处理减少函数调用开销
• 缓存友好（连续内存访问）
• 易于SIMD优化
• 编译器优化空间大
```

---

## 第八部分：并行执行

### 8.1 并行度控制

```cpp
// 估算并行度

idx_t PhysicalOperator::EstimatedThreadCount() const {
    // 1. 基于数据大小估算
    idx_t data_size = estimated_cardinality;

    // 2. 每个线程处理至少2个Row Group
    idx_t min_rows_per_thread = STANDARD_VECTOR_SIZE * 2;

    idx_t thread_count = data_size / min_rows_per_thread;

    // 3. 限制在合理范围
    idx_t max_threads = context.config.max_threads;
    idx_t optimal_threads = MinValue(thread_count, max_threads);

    return MaxValue<idx_t>(optimal_threads, 1);
}
```

### 8.2 并行扫描

```cpp
// 并行Table扫描

class ParallelTableScan {
public:
    void ExecuteParallel() {
        // 1. 获取Row Group数量
        idx_t num_row_groups = table->GetRowGroupCount();

        // 2. 计算线程数
        idx_t num_threads = MinValue(num_row_groups / 2, max_threads);

        // 3. 分配Row Group给线程
        vector<vector<idx_t>> thread_assignments(num_threads);

        for (idx_t rg = 0; rg < num_row_groups; rg++) {
            idx_t thread = rg % num_threads;
            thread_assignments[thread].push_back(rg);
        }

        // 4. 启动并行任务
        vector<thread> threads;
        vector<DataChunk> results(num_threads);

        for (idx_t t = 0; t < num_threads; t++) {
            threads.emplace_back([this, t, &thread_assignments, &results]() {
                // 每个线程独立扫描分配的Row Group
                for (idx_t rg : thread_assignments[t]) {
                    ScanRowGroup(rg, results[t]);
                }
            });
        }

        // 5. 等待所有线程完成
        for (auto &t : threads) {
            t.join();
        }

        // 6. 合并结果
        for (idx_t t = 0; t < num_threads; t++) {
            final_result.Append(results[t]);
        }
    }
};
```

### 8.3 并行哈希构建

```cpp
// 并行哈希表构建

class ParallelHashBuild {
public:
    void BuildParallel(DataChunk &chunk, idx_t num_threads) {
        // 1. 每个线程构建局部哈希表
        vector<unique_ptr<HashTable>> local_tables(num_threads);

        for (idx_t t = 0; t < num_threads; t++) {
            local_tables[t] = make_unique<HashTable>();
        }

        // 2. 分区数据
        idx_t rows_per_thread = chunk.size() / num_threads;

        vector<thread> threads;
        for (idx_t t = 0; t < num_threads; t++) {
            idx_t start = t * rows_per_thread;
            idx_t end = (t == num_threads - 1) ?
                        chunk.size() : (t + 1) * rows_per_thread;

            threads.emplace_back([this, &chunk, start, end, &local_tables, t]() {
                for (idx_t i = start; i < end; i++) {
                    local_tables[t]->Insert(chunk, i);
                }
            });
        }

        for (auto &t : threads) {
            t.join();
        }

        // 3. 合并局部哈希表
        for (idx_t t = 0; t < num_threads; t++) {
            global_table->Merge(*local_tables[t]);
        }
    }
};
```

---

## 学习总结

### 执行算子关键要点

1. **三重接口设计**：Source、Operator、Sink
2. **状态管理**：GlobalState（共享）+ LocalState（私有）
3. **Push模型**：批量数据推送，减少函数调用
4. **向量化执行**：DataChunk（2048行）批量处理
5. **并行执行**：Pipeline级和Operator级并行

### 算子性能对比

| 算子类型 | 时间复杂度 | 空间复杂度 | 并行化 |
|---------|-----------|-----------|--------|
| Table Scan | O(n) | O(1) | ✅ |
| Filter | O(n) | O(n) | ✅ |
| Projection | O(n) | O(n) | ✅ |
| Hash Join | O(n+m) | O(min(n,m)) | ✅ |
| Sort Merge Join | O(n log n + m log m) | O(n+m) | ⚠️ |
| Hash Aggregate | O(n) | O(k) | ✅ |
| Order By | O(n log n) | O(n) | ⚠️ |
| Limit | O(n) | O(limit) | ✅ |

### 推荐阅读

**论文：**
- "Volcano - An Extensible and Parallel Query Execution System"
- "The Design of the BUST Traversal System"
- "Morsel-Driven Parallelism: A FPGA-Aware Query Execution Model"

**代码位置：**
- `src/execution/operator/` - 所有算子实现
- `src/execution/physical_plan/pipeline.hpp` - Pipeline实现

**相关课程：**
- [DuckDB深度课程_向量化执行引擎](./DuckDB深度课程_向量化执行引擎.md)
- [DuckDB高级课程_查询优化器深度解析](./DuckDB高级课程_查询优化器深度解析.md)

---

**最后更新：2026-01-23**
