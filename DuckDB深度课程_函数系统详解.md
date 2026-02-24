# DuckDB 深度课程：函数系统详解

> 本课程深入讲解DuckDB的函数系统设计，包括标量函数、聚合函数、表函数的实现机制和向量化执行。

---

## 课程概览

### 学习目标

- 理解函数系统的架构设计
- 掌握标量函数的实现和注册
- 学习聚合函数的状态管理
- 理解表函数的执行模型
- 掌握函数重载和类型推导
- 能够实现自定义函数

### 前置知识

- 向量化执行基础
- C++模板和函数指针
- SQL函数语义
- 类型系统基础

---

## 第一部分：函数系统架构

### 1.1 函数系统层次结构

```
┌─────────────────────────────────────────────────────────┐
│              DuckDB 函数系统架构                       │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  函数分类：                                           │
│  ┌─────────────────────────────────────────────┐          │
│  │ 1. 标量函数 (Scalar Functions)        │          │
│  │    - 数学函数 (ABS, SQRT, POW...)      │          │
│  │    - 字符串函数 (CONCAT, SUBSTR...)    │          │
│  │    - 日期函数 (NOW, DATE_TRUNC...)    │          │
│  │    - 类型转换 (CAST, TRY_CAST...)      │          │
│  └─────────────────────────────────────────────┘          │
│                                                         │
│  ┌─────────────────────────────────────────────┐          │
│  │ 2. 聚合函数 (Aggregate Functions)      │          │
│  │    - COUNT, SUM, AVG, MIN, MAX...       │          │
│  │    - ARRAY_AGG, LIST_AGG...            │          │
│  │    - PERCENTILE_CONT, STDDEV...        │          │
│  └─────────────────────────────────────────────┘          │
│                                                         │
│  ┌─────────────────────────────────────────────┐          │
│  │ 3. 表函数 (Table Functions)            │          │
│  │    - GENERATE_SERIES                   │          │
│  │    - READ_CSV, READ_PARQUET...         │          │
│  │    - UNNEST                            │          │
│  └─────────────────────────────────────────────┘          │
│                                                         │
│  函数注册：                                           │
│  ┌─────────────────────────────────────────────┐          │
│  │ Catalog (函数注册表)                      │          │
│  │    ├── ScalarFunctionCatalog             │          │
│  │    ├── AggregateFunctionCatalog           │          │
│  │    └── TableFunctionCatalog              │          │
│  │                                          │          │
│  │  FunctionSet (函数重载)                 │          │
│  │    ├── ScalarFunctionSet                │          │
│  │    ├── AggregateFunctionSet             │          │
│  │    └── TableFunctionSet                 │          │
│  └─────────────────────────────────────────────┘          │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 1.2 函数查找流程

```cpp
// 函数查找算法

class FunctionBinder {
public:
    // 绑定函数调用
    optional_idx BindFunction(
        const string &name,
        FunctionSet<SimpleFunction> &functions,
        const vector<LogicalType> &arguments,
        ErrorData &error) {

        // 1. 查找所有候选函数
        vector<idx_t> candidates = FindCandidateFunctions(
            name, functions, arguments
        );

        if (candidates.empty()) {
            error.AddError("Function '%s' not found", name);
            return optional_idx();
        }

        // 2. 选择最佳匹配
        idx_t best_match = SelectBestFunction(
            candidates, arguments
        );

        if (candidates.size() > 1 && !HasUniqueBestMatch()) {
            error.AddWarning("Ambiguous function call to '%s'", name);
        }

        return best_match;
    }

private:
    // 查找候选函数
    vector<idx_t> FindCandidateFunctions(
        const string &name,
        FunctionSet<SimpleFunction> &functions,
        const vector<LogicalType> &arguments) {

        vector<idx_t> candidates;

        for (idx_t i = 0; i < functions.functions.size(); i++) {
            auto &func = functions.functions[i];

            // 检查参数数量
            if (!IsParameterCountMatch(func, arguments.size())) {
                continue;
            }

            // 检查参数类型兼容性
            if (AreArgumentsCompatible(func, arguments)) {
                candidates.push_back(i);
            }
        }

        return candidates;
    }

    // 选择最佳函数
    idx_t SelectBestFunction(
        vector<idx_t> &candidates,
        const vector<LogicalType> &arguments) {

        idx_t best_idx = DConstants::INVALID_INDEX;
        int64_t lowest_cost = NumericLimits<int64_t>::Maximum();

        for (auto idx : candidates) {
            auto &func = functions.functions[idx];

            // 计算类型转换成本
            int64_t cost = CalculateConversionCost(
                func, arguments
            );

            if (cost < lowest_cost) {
                lowest_cost = cost;
                best_idx = idx;
            }
        }

        return best_idx;
    }

    // 计算转换成本
    int64_t CalculateConversionCost(
        SimpleFunction &func,
        const vector<LogicalType> &arguments) {

        int64_t total_cost = 0;

        for (idx_t i = 0; i < arguments.size(); i++) {
            if (i >= func.parameters.size()) {
                // 可变参数
                break;
            }

            auto &param_type = func.parameters[i];
            auto &arg_type = arguments[i];

            // 计算类型转换成本
            total_cost += GetCastCost(arg_type, param_type);
        }

        return total_cost;
    }

    // 类型转换成本表
    int64_t GetCastCost(LogicalType from, LogicalType to) {
        // 相同类型：0
        if (from == to) {
            return 0;
        }

        // 类型提升：1
        if (IsNumericPromotion(from, to)) {
            return 1;
        }

        // 隐式转换：10
        if (CanImplicitCast(from, to)) {
            return 10;
        }

        // 需要显式转换：100
        return 100;
    }
};
```

---

## 第二部分：标量函数

### 2.1 ScalarFunction结构

```cpp
// src/include/duckdb/function/scalar_function.hpp

class ScalarFunction : public SimpleFunction {
public:
    // 函数名
    string name;

    // 参数类型列表
    vector<LogicalType> arguments;

    // 返回类型
    LogicalType return_type;

    // 可变参数类型
    LogicalType varargs;

    // ==================== 函数指针 ====================

    // 主执行函数
    scalar_function_t function;

    // 绑定函数（类型检查和参数验证）
    bind_scalar_function_t bind;

    // 扩展绑定函数
    bind_scalar_function_extended_t bind_extended;

    // 初始化本地状态
    init_local_state_t init_local_state;

    // 统计信息传播
    function_statistics_t statistics;

    // Lambda函数绑定
    bind_lambda_function_t bind_lambda;

    // ==================== 函数属性 ====================

    // 函数稳定性
    FunctionStability stability;

    // NULL处理
    FunctionNullHandling null_handling;

    // 构造函数
    ScalarFunction(string name,
                   vector<LogicalType> arguments,
                   LogicalType return_type,
                   scalar_function_t function,
                   bind_scalar_function_t bind = nullptr,
                   FunctionStability stability = FunctionStability::CONSISTENT,
                   FunctionNullHandling null_handling = FunctionNullHandling::DEFAULT_NULL_HANDLING)
        : name(std::move(name)),
          arguments(std::move(arguments)),
          return_type(std::move(return_type)),
          function(std::move(function)),
          bind(bind),
          stability(stability),
          null_handling(null_handling) {}
};
```

### 2.2 向量化标量函数实现

```cpp
// 标量函数执行示例

template <class OP>
static void UnaryFunction(DataChunk &args,
                          ExpressionState &state,
                          Vector &result) {

    // 1. 获取输入向量
    auto &input = args.data[0];

    // 2. 统一格式化（处理不同向量类型）
    UnifiedVectorFormat input_data;
    input.ToUnifiedFormat(args.size(), input_data);

    // 3. 获取结果数据指针
    result.SetVectorType(VectorType::FLAT_VECTOR);
    auto result_data = FlatVector::GetData<double>(result);

    // 4. 批量处理
    for (idx_t i = 0; i < args.size(); i++) {
        // 获取实际索引
        idx_t idx = input_data.sel->get_index(i);

        // 检查NULL
        if (!input_data.validity.RowIsValid(idx)) {
            result.validity.SetInvalid(i);
            continue;
        }

        // 应用操作
        result_data[i] = OP::Apply(
            ((double *)input_data.data)[idx]
        );
    }

    result.Verify(args.size());
}

// 操作符定义
struct AbsOperator {
    static inline double Apply(double value) {
        return std::abs(value);
    }
};

struct SqrtOperator {
    static inline double Apply(double value) {
        return std::sqrt(value);
    }
};

struct CeilOperator {
    static inline double Apply(double value) {
        return std::ceil(value);
    }
};

// 注册函数
ScalarFunctionSet GetAbsFunction() {
    ScalarFunctionSet set("abs");

    set.AddFunction(ScalarFunction(
        "abs",
        {LogicalType::DOUBLE},         // 参数：DOUBLE
        LogicalType::DOUBLE,            // 返回：DOUBLE
        &UnaryFunction<AbsOperator>    // 执行函数
    ));

    return set;
}
```

### 2.3 二元函数实现

```cpp
// 二元函数实现

template <class OP>
static void BinaryFunction(DataChunk &args,
                           ExpressionState &state,
                           Vector &result) {

    D_ASSERT(args.ColumnCount() == 2);

    // 1. 获取输入向量
    auto &left = args.data[0];
    auto &right = args.data[1];

    // 2. 统一格式化
    UnifiedVectorFormat left_data, right_data;
    left.ToUnifiedFormat(args.size(), left_data);
    right.ToUnifiedFormat(args.size(), right_data);

    // 3. 准备结果
    result.SetVectorType(VectorType::FLAT_VECTOR);
    auto result_data = FlatVector::GetData<double>(result);
    result.validity.SetAllValid(args.size());

    // 4. 批量处理
    for (idx_t i = 0; i < args.size(); i++) {
        idx_t lidx = left_data.sel->get_index(i);
        idx_t ridx = right_data.sel->get_index(i);

        // 检查NULL
        if (!left_data.validity.RowIsValid(lidx) ||
            !right_data.validity.RowIsValid(ridx)) {
            result.validity.SetInvalid(i);
            continue;
        }

        // 应用二元操作
        result_data[i] = OP::Apply(
            ((double *)left_data.data)[lidx],
            ((double *)right_data.data)[ridx]
        );
    }

    result.Verify(args.size());
}

// 二元操作符
struct AddOperator {
    static inline double Apply(double left, double right) {
        return left + right;
    }
};

struct MultiplyOperator {
    static inline double Apply(double left, double right) {
        return left * right;
    }
};

struct PowerOperator {
    static inline double Apply(double left, double right) {
        return std::pow(left, right);
    }
};

// 注册二元函数
ScalarFunctionSet GetArithmeticFunctions() {
    ScalarFunctionSet set("add");

    set.AddFunction(ScalarFunction(
        "add",
        {LogicalType::DOUBLE, LogicalType::DOUBLE},
        LogicalType::DOUBLE,
        &BinaryFunction<AddOperator>
    ));

    return set;
}
```

### 2.4 字符串函数实现

```cpp
// 字符串函数：CONCAT示例

static void ConcatFunction(DataChunk &args,
                          ExpressionState &state,
                          Vector &result) {

    // 1. 参数验证
    if (args.ColumnCount() < 2) {
        throw InvalidInputException(
            "CONCAT requires at least 2 arguments"
        );
    }

    // 2. 统一格式化所有参数
    vector<UnifiedVectorFormat> inputs_data;
    for (idx_t col = 0; col < args.ColumnCount(); col++) {
        UnifiedVectorFormat vdata;
        args.data[col].ToUnifiedFormat(args.size(), vdata);
        inputs_data.push_back(std::move(vdata));
    }

    // 3. 准备结果
    result.SetVectorType(VectorType::FLAT_VECTOR);
    auto result_data = FlatVector::GetData<string_t>(result);

    // 4. 批量处理
    for (idx_t i = 0; i < args.size(); i++) {
        // 计算总长度
        idx_t total_length = 0;
        bool has_null = false;

        for (idx_t col = 0; col < args.ColumnCount(); col++) {
            idx_t idx = inputs_data[col].sel->get_index(i);

            if (!inputs_data[col].validity.RowIsValid(idx)) {
                has_null = true;
                break;
            }

            auto &str = ((string_t *)inputs_data[col].data)[idx];
            total_length += str.GetSize();
        }

        // 处理NULL
        if (has_null) {
            result.validity.SetInvalid(i);
            continue;
        }

        // 分配结果字符串
        result_data[i] = StringVector::EmptyString(
            result,
            total_length
        );

        // 拼接字符串
        idx_t offset = 0;
        for (idx_t col = 0; col < args.ColumnCount(); col++) {
            idx_t idx = inputs_data[col].sel->get_index(i);
            auto &str = ((string_t *)inputs_data[col].data)[idx];

            memcpy(
                result_data[i].GetDataWriteable() + offset,
                str.GetData(),
                str.GetSize()
            );
            offset += str.GetSize();
        }
    }

    result.Verify(args.size());
}
```

---

## 第三部分：聚合函数

### 3.1 AggregateFunction结构

```cpp
// src/include/duckdb/function/aggregate_function.hpp

class AggregateFunction : public SimpleFunction {
public:
    // 函数名
    string name;

    // 参数类型
    vector<LogicalType> arguments;

    // 返回类型
    LogicalType return_type;

    // ==================== 状态管理 ====================

    // 状态大小
    aggregate_size_t state_size;

    // 状态初始化
    aggregate_initialize_t initialize;

    // 状态更新（单批次）
    aggregate_update_t update;

    // 状态合并（并行聚合）
    aggregate_combine_t combine;

    // 状态最终化
    aggregate_finalize_t finalize;

    // 简单更新（优化路径）
    aggregate_simple_update_t simple_update;

    // 绑定函数
    bind_aggregate_function_t bind;

    // 析构函数
    aggregate_destructor_t destructor;

    // 统计信息
    aggregate_statistics_t statistics;

    // 窗口函数支持
    aggregate_window_t window;

    // ==================== 聚合属性 ====================

    // 是否为ORDER BY聚合
    bool order_dependent;

    // 是否为DISTINCT聚合
    bool distinct_dependent;

    // 构造函数
    AggregateFunction(string name,
                      vector<LogicalType> arguments,
                      LogicalType return_type,
                      aggregate_size_t state_size,
                      aggregate_initialize_t initialize,
                      aggregate_update_t update,
                      aggregate_combine_t combine,
                      aggregate_finalize_t finalize)
        : name(std::move(name)),
          arguments(std::move(arguments)),
          return_type(std::move(return_type)),
          state_size(state_size),
          initialize(std::move(initialize)),
          update(std::move(update)),
          combine(std::move(combine)),
          finalize(std::move(finalize)) {}
};
```

### 3.2 SUM聚合实现

```cpp
// SUM聚合函数

// 聚合状态
struct SumState {
    double sum;
    idx_t count;
};

// 状态初始化
static void SumInitialize(data_ptr_t state_ptr) {
    auto &state = *(SumState *)state_ptr;
    state.sum = 0.0;
    state.count = 0;
}

// 状态更新
static void SumUpdate(Vector inputs[], aggregate_parameter_t &parameters,
                     idx_t input_count, Vector &state_vector,
                     idx_t count) {

    auto &states = (SumState **)state_vector.GetData();
    auto &input = inputs[0];

    // 统一格式化
    UnifiedVectorFormat input_data;
    input.ToUnifiedFormat(input_count, input_data);

    // 批量更新
    for (idx_t i = 0; i < count; i++) {
        auto &state = *states[i];
        idx_t idx = input_data.sel->get_index(i);

        // 检查NULL
        if (!input_data.validity.RowIsValid(idx)) {
            continue;
        }

        // 更新状态
        double value = ((double *)input_data.data)[idx];
        state.sum += value;
        state.count++;
    }
}

// 状态合并（并行聚合）
static void SumCombine(Vector &state_vector,
                       Vector &combined_state,
                       idx_t count) {

    auto &states = (SumState **)state_vector.GetData();
    auto &combined = *(SumState **)combined_state.GetData()[0];

    for (idx_t i = 0; i < count; i++) {
        combined->sum += states[i]->sum;
        combined->count += states[i]->count;
    }
}

// 状态最终化
static void SumFinalize(Vector &state_vector,
                       Vector &result,
                       AggregateInputData &aggr_input_data,
                       idx_t count) {

    auto &states = (SumState **)state_vector.GetData();
    auto result_data = FlatVector::GetData<double>(result);

    for (idx_t i = 0; i < count; i++) {
        auto &state = *states[i];

        if (state.count == 0) {
            // 没有非NULL值
            result.validity.SetInvalid(i);
        } else {
            result_data[i] = state.sum;
        }
    }
}

// 创建SUM函数
AggregateFunction GetSumFunction() {
    return AggregateFunction("sum",
        {LogicalType::DOUBLE},           // 参数类型
        LogicalType::DOUBLE,             // 返回类型
        sizeof(SumState),               // 状态大小
        &SumInitialize,                 // 初始化
        &SumUpdate,                     // 更新
        &SumCombine,                    // 合并
        &SumFinalize);                  // 最终化
}
```

### 3.3 AVG聚合实现

```cpp
// AVG聚合函数

// 聚合状态
struct AverageState {
    double sum;
    idx_t count;
};

// 状态更新
static void AverageUpdate(Vector inputs[],
                          aggregate_parameter_t &parameters,
                          idx_t input_count,
                          Vector &state_vector,
                          idx_t count) {

    auto &states = (AverageState **)state_vector.GetData();
    auto &input = inputs[0];

    UnifiedVectorFormat input_data;
    input.ToUnifiedFormat(input_count, input_data);

    for (idx_t i = 0; i < count; i++) {
        auto &state = *states[i];
        idx_t idx = input_data.sel->get_index(i);

        if (!input_data.validity.RowIsValid(idx)) {
            continue;
        }

        double value = ((double *)input_data.data)[idx];
        state.sum += value;
        state.count++;
    }
}

// 状态最终化
static void AverageFinalize(Vector &state_vector,
                           Vector &result,
                           AggregateInputData &aggr_input_data,
                           idx_t count) {

    auto &states = (AverageState **)state_vector.GetData();
    auto result_data = FlatVector::GetData<double>(result);

    for (idx_t i = 0; i < count; i++) {
        auto &state = *states[i];

        if (state.count == 0) {
            result.validity.SetInvalid(i);
        } else {
            result_data[i] = state.sum / state.count;
        }
    }
}

// 创建AVG函数
AggregateFunction GetAvgFunction() {
    return AggregateFunction("avg",
        {LogicalType::DOUBLE},
        LogicalType::DOUBLE,
        sizeof(AverageState),
        &SumInitialize,                  // 复用SUM的初始化
        &AverageUpdate,
        &SumCombine,                     // 复用SUM的合并
        &AverageFinalize);
}
```

### 3.4 COUNT聚合实现

```cpp
// COUNT聚合函数

// COUNT(*)状态（简化版）
struct CountStarState {
    int64_t count;
};

// COUNT(column)状态
struct CountState {
    int64_t count;
    bool has_null;  // 是否遇到过NULL
};

// COUNT更新
static void CountUpdate(Vector inputs[],
                        aggregate_parameter_t &parameters,
                        idx_t input_count,
                        Vector &state_vector,
                        idx_t count) {

    auto &states = (CountState **)state_vector.GetData();
    auto &input = inputs[0];

    UnifiedVectorFormat input_data;
    input.ToUnifiedFormat(input_count, input_data);

    for (idx_t i = 0; i < count; i++) {
        auto &state = *states[i];
        idx_t idx = input_data.sel->get_index(i);

        // 计数所有行（包括NULL）
        state.count++;

        // 标记是否遇到过NULL
        if (!input_data.validity.RowIsValid(idx)) {
            state.has_null = true;
        }
    }
}

// COUNT最终化
static void CountFinalize(Vector &state_vector,
                         Vector &result,
                         AggregateInputData &aggr_input_data,
                         idx_t count) {

    auto &states = (CountState **)state_vector.GetData();
    auto result_data = FlatVector::GetData<int64_t>(result);

    for (idx_t i = 0; i < count; i++) {
        auto &state = *states[i];
        result_data[i] = state.count;
    }
}

// 创建COUNT函数
AggregateFunction GetCountFunction() {
    return AggregateFunction("count",
        {LogicalType::ANY},                // 接受任何类型
        LogicalType::BIGINT,               // 返回BIGINT
        sizeof(CountState),
        &CountInitialize,                 // 初始化
        &CountUpdate,                     // 更新
        &CountCombine,                    // 合并
        &CountFinalize);                  // 最终化
}
```

---

## 第四部分：表函数

### 4.1 TableFunction结构

```cpp
// src/include/duckdb/function/table_function.hpp

class TableFunction : public SimpleNamedParameterFunction {
public:
    // 函数名
    string name;

    // 参数类型
    vector<LogicalType> arguments;

    // 返回类型（列）
    vector<LogicalType> return_types;

    // 列名
    vector<string> column_names;

    // ==================== 函数指针 ====================

    // 主执行函数
    table_function_t function;

    // 绑定函数
    table_function_bind_t bind;

    // 全局状态初始化
    table_function_init_global_t init_global;

    // 本地状态初始化
    table_function_init_local_t init_local;

    // ==================== 优化特性 ====================

    // 投影下推支持
    bool projection_pushdown;

    // 过滤器下推支持
    bool filter_pushdown;

    // 过滤器剪枝
    bool filter_prune;

    // 采样下推
    bool sampling_pushdown;

    // 延迟物化
    bool late_materialization;

    // 构造函数
    TableFunction(string name,
                  vector<LogicalType> arguments,
                  vector<LogicalType> return_types,
                  vector<string> column_names,
                  table_function_t function)
        : name(std::move(name)),
          arguments(std::move(arguments)),
          return_types(std::move(return_types)),
          column_names(std::move(column_names)),
          function(std::move(function)) {}
};
```

### 4.2 表函数执行流程

```
┌─────────────────────────────────────────────────────────┐
│              表函数执行流程                           │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. 绑定阶段 (Bind)                                  │
│     ┌─────────────────────────────────────┐              │
│     │ - 验证参数类型和数量                  │              │
│     │ - 确定返回类型                          │              │
│     │ - 创建绑定数据 (FunctionData)         │              │
│     └─────────────────────────────────────┘              │
│          ↓                                              │
│  2. 初始化阶段 (Init)                                  │
│     ┌─────────────────────────────────────┐              │
│     │ - 初始化全局状态 (GlobalState)        │              │
│     │ - 初始化本地状态 (LocalState)          │              │
│     │ - 准备执行环境                         │              │
│     └─────────────────────────────────────┘              │
│          ↓                                              │
│  3. 执行阶段 (Execute)                                  │
│     ┌─────────────────────────────────────┐              │
│     │ while (true) {                         │              │
│     │   生成一个 DataChunk (2048 rows)   │              │
│     │   如果没有数据，break                  │              │
│     │   返回 DataChunk 给下游              │              │
│     │ }                                    │              │
│     └─────────────────────────────────────┘              │
│          ↓                                              │
│  4. 清理阶段 (Cleanup)                                  │
│     ┌─────────────────────────────────────┐              │
│     │ - 释放全局状态                         │              │
│     │ - 释放本地状态                         │              │
│     │ - 清理临时数据                         │              │
│     └─────────────────────────────────────┘              │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 4.3 GENERATE_SERIES表函数

```cpp
// GENERATE_SERIES表函数实现

// 函数数据
struct GenerateSeriesData : public TableFunctionData {
    int64_t start;
    int64_t end;
    int64_t increment;

    int64_t current;
};

// 绑定函数
static unique_ptr<FunctionData> GenerateSeriesBind(
    ClientContext &context,
    TableFunctionBindInput &input,
    vector<LogicalType> &return_types,
    vector<string> &names) {

    // 1. 验证参数
    if (input.inputs.size() < 1 || input.inputs.size() > 3) {
        throw InvalidInputException(
            "GENERATE_SERIES requires 1 to 3 arguments"
        );
    }

    // 2. 创建函数数据
    auto bind_data = make_unique<GenerateSeriesData>();

    // 提取参数
    Value start_val, end_val, incr_val;
    // ... 参数提取逻辑

    bind_data->start = start_val.GetValue<int64_t>();
    bind_data->end = end_val.GetValue<int64_t>();
    bind_data->increment = incr_val ?
                             incr_val.GetValue<int64_t>() : 1;

    // 3. 设置返回类型
    return_types = {LogicalType::BIGINT};
    names = {"value"};

    return bind_data;
}

// 执行函数
static void GenerateSeriesExecute(ClientContext &context,
                                TableFunctionInput &data_p,
                                DataChunk &output) {

    auto &data = (GenerateSeriesData &)*data_p.bind_data;

    // 1. 计算本次要生成的行数
    idx_t count = 0;
    const idx_t max_count = STANDARD_VECTOR_SIZE;

    while (count < max_count &&
           ((data.increment > 0 && data.current <= data.end) ||
            (data.increment < 0 && data.current >= data.end) ||
            (data.increment == 0 && data.current == data.end))) {

        // 2. 填充结果
        auto result_data = FlatVector::GetData<int64_t>(output.data[0]);
        result_data[count] = data.current;

        data.current += data.increment;
        count++;
    }

    // 3. 设置基数
    output.SetCardinality(count);

    // 4. 检查是否完成
    if (count == 0) {
        data_p.finished = true;
    }
}

// 注册表函数
TableFunction GetGenerateSeriesFunction() {
    return TableFunction("generate_series",
        {LogicalType::BIGINT, LogicalType::BIGINT, LogicalType::BIGINT},
        {LogicalType::BIGINT},
        {"value"},
        &GenerateSeriesExecute);
}
```

---

## 第五部分：函数重载

### 5.1 函数重载机制

```cpp
// 函数集合（支持重载）

template <class T>
class FunctionSet {
public:
    string name;
    vector<T> functions;

    void AddFunction(T function) {
        functions.push_back(std::move(function));
    }

    idx_t Size() {
        return functions.size();
    }

    T GetFunctionByOffset(idx_t offset) {
        D_ASSERT(offset < functions.size());
        return functions[offset];
    }
};

// 重载示例：ABS函数
ScalarFunctionSet GetAbsFunctions() {
    ScalarFunctionSet set("abs");

    // ABS(INTEGER) -> INTEGER
    set.AddFunction(ScalarFunction(
        "abs",
        {LogicalType::INTEGER},
        LogicalType::INTEGER,
        &UnaryAbsFunction<int32_t>
    ));

    // ABS(BIGINT) -> BIGINT
    set.AddFunction(ScalarFunction(
        "abs",
        {LogicalType::BIGINT},
        LogicalType::BIGINT,
        &UnaryAbsFunction<int64_t>
    ));

    // ABS(DOUBLE) -> DOUBLE
    set.AddFunction(ScalarFunction(
        "abs",
        {LogicalType::DOUBLE},
        LogicalType::DOUBLE,
        &UnaryAbsFunction<double>
    ));

    return set;
}
```

### 5.2 类型推导和隐式转换

```cpp
// 类型推导

class TypeDerivation {
public:
    // 推导函数返回类型
    static LogicalType DeriveFunctionType(
        SimpleFunction &function,
        vector<LogicalType> &arguments) {

        // 1. 检查是否有显式返回类型
        if (function.HasExplicitReturnType()) {
            return function.return_type;
        }

        // 2. 根据参数类型推导
        switch (function.function_type) {
        case FunctionType::ARITHMETIC:
            return DeriveArithmeticType(arguments);

        case FunctionType::COMPARISON:
            return LogicalType::BOOLEAN;

        case FunctionType::STRING_CONCAT:
            return LogicalType::VARCHAR;

        default:
            return LogicalType::INVALID;
        }
    }

private:
    // 推导算术运算类型
    static LogicalType DeriveArithmeticType(
        vector<LogicalType> &arguments) {

        // 类型提升规则：
        // TINYINT < SMALLINT < INTEGER < BIGINT < HUGEINT < DOUBLE

        LogicalType max_type = LogicalType::TINYINT;

        for (auto &arg_type : arguments) {
            if (GetNumericPriority(arg_type) >
                GetNumericPriority(max_type)) {
                max_type = arg_type;
            }
        }

        return max_type;
    }
};
```

---

## 第六部分：函数注册

### 6.1 内置函数注册

```cpp
// extension/core_functions/scalar_functions/numeric.cpp

void RegisterNumericFunctions(BuiltinFunctions &set) {
    // ==================== 数学函数 ====================

    // ABS函数
    set.AddFunction(GetAbsFunctions());

    // ROUND函数
    set.AddFunction(GetRoundFunctions());

    // CEIL/FLOOR函数
    set.AddFunction(GetCeilFloorFunctions());

    // SQRT函数
    set.AddFunction("sqrt", GetSqrtFunction());

    // POW函数
    set.AddFunction("pow", GetPowFunction());

    // ==================== 三角函数 ====================

    set.AddFunction("sin", GetSinFunction());
    set.AddFunction("cos", GetCosFunction());
    set.AddFunction("tan", GetTanFunction());

    // ==================== 算术函数 ====================

    set.AddFunction(GetAddFunctions());
    set.AddFunction(GetSubtractFunctions());
    set.AddFunction(GetMultiplyFunctions());
    set.AddFunction(GetDivideFunctions());
    set.AddFunction(GetModFunctions());
}

// extension/core_functions/aggregate_functions/aggregate_functions.cpp

void RegisterAggregateFunctions(BuiltinFunctions &set) {
    // ==================== 基础聚合 ====================

    set.AddFunction(AggregateFunction::CountFunction());
    set.AddFunction(AggregateFunction::SumFunction());
    set.AddFunction(AggregateFunction::AvgFunction());
    set.AddFunction(AggregateFunction::MinFunction());
    set.AddFunction(AggregateFunction::MaxFunction());

    // ==================== 统计聚合 ====================

    set.AddFunction("stddev", GetStdDevFunction());
    set.AddFunction("variance", GetVarianceFunction());

    // ==================== 数组聚合 ====================

    set.AddFunction("array_agg", GetArrayAggFunction());
    set.AddFunction("list_agg", GetListAggFunction());
}
```

### 6.2 自定义函数注册

```cpp
// 注册自定义标量函数

void RegisterMyScalarFunction(DatabaseInstance &db) {
    // 1. 创建函数
    ScalarFunction my_func(
        "my_func",
        {LogicalType::INTEGER},
        LogicalType::INTEGER,
        &MyScalarFunctionImpl
    );

    // 2. 注册到Catalog
    auto &catalog = db.GetCatalog();
    catalog.AddFunction("my_func", my_func);
}

// 注册自定义聚合函数

void RegisterMyAggregateFunction(DatabaseInstance &db) {
    // 1. 创建聚合函数
    AggregateFunction my_agg(
        "my_agg",
        {LogicalType::DOUBLE},
        LogicalType::DOUBLE,
        sizeof(MyAggregateState),
        &MyAggregateInitialize,
        &MyAggregateUpdate,
        &MyAggregateCombine,
        &MyAggregateFinalize
    );

    // 2. 注册到Catalog
    auto &catalog = db.GetCatalog();
    catalog.AddFunction("my_agg", my_agg);
}

// 注册自定义表函数

void RegisterMyTableFunction(DatabaseInstance &db) {
    // 1. 创建表函数
    TableFunction my_table_func(
        "my_table_func",
        {LogicalType::INTEGER},
        {LogicalType::INTEGER, LogicalType::VARCHAR},
        {"id", "name"},
        &MyTableFunctionImpl
    );

    // 2. 注册到Catalog
    auto &catalog = db.GetCatalog();
    catalog.AddFunction("my_table_func", my_table_func);
}
```

---

## 学习总结

### 函数系统关键要点

1. **向量化优先**：所有函数基于DataChunk批量处理
2. **类型安全**：严格的类型检查和转换机制
3. **重载支持**：函数重载和最佳匹配算法
4. **状态管理**：聚合函数支持复杂的状态生命周期
5. **扩展性强**：支持自定义函数注册

### 函数性能对比

| 函数类型 | 实现复杂度 | 向量化支持 | 性能优化 |
|---------|-----------|-----------|---------|
| 标量函数 | 低 | ✅ | 高 |
| 简单聚合 | 中 | ✅ | 中高 |
| 复杂聚合 | 高 | ✅ | 中 |
| 表函数 | 中高 | ✅ | 中高 |

### 推荐阅读

**代码位置：**
- `src/function/scalar_function/` - 标量函数
- `src/function/aggregate_function/` - 聚合函数
- `src/function/table_function/` - 表函数
- `extension/core_functions/` - 内置函数实现

**相关课程：**
- [DuckDB深度课程_向量化执行引擎](./DuckDB深度课程_向量化执行引擎.md)
- [DuckDB深度课程_类型系统实现](./DuckDB深度课程_类型系统实现.md)
- [DuckDB高级课程_扩展系统开发](./DuckDB高级课程_扩展系统开发.md)

---

**最后更新：2026-01-23**
