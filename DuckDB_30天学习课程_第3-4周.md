# DuckDB 30天学习课程 - 第三周完结 & 第四周：存储引擎

本文档包含第三周剩余内容（Day 20-21）和第四周完整内容（Day 22-28）。

---

## Day 20: 向量化执行深度解析

**学习目标：** 深入理解向量化执行的底层优化技术

### 20.1 SIMD指令简介

SIMD（Single Instruction Multiple Data）允许一条指令处理多个数据：

```cpp
// 标量加法（每次处理1个数）
void scalar_add(int32_t *a, int32_t *b, int32_t *result, size_t n) {
    for (size_t i = 0; i < n; i++) {
        result[i] = a[i] + b[i];
    }
}

// SIMD加法（每次处理8个数，AVX2）
void simd_add(int32_t *a, int32_t *b, int32_t *result, size_t n) {
    size_t i = 0;

    // 每次处理8个int32
    for (; i + 8 <= n; i += 8) {
        __m256i va = _mm256_loadu_si256((__m256i*)&a[i]);
        __m256i vb = _mm256_loadu_si256((__m256i*)&b[i]);
        __m256i vresult = _mm256_add_epi32(va, vb);
        _mm256_storeu_si256((__m256i*)&result[i], vresult);
    }

    // 处理剩余元素
    for (; i < n; i++) {
        result[i] = a[i] + b[i];
    }
}
```

### 20.2 DuckDB中的SIMD使用

```cpp
// src/common/vector_operations/numeric_operations.cpp
void VectorOperations::Add(Vector &left, Vector &right, Vector &result, idx_t count) {
    switch (left.GetType().InternalType()) {
    case PhysicalType::INT32:
        AddOperation<int32_t>(left, right, result, count);
        break;
    case PhysicalType::INT64:
        AddOperation<int64_t>(left, right, result, count);
        break;
    // ... 其他类型
    }
}

template <class T>
void AddOperation(Vector &left, Vector &right, Vector &result, idx_t count) {
    auto ldata = FlatVector::GetData<T>(left);
    auto rdata = FlatVector::GetData<T>(right);
    auto result_data = FlatVector::GetData<T>(result);

    // 检查是否有NULL值
    if (left.GetVectorType() == VectorType::CONSTANT_VECTOR &&
        right.GetVectorType() == VectorType::CONSTANT_VECTOR) {
        // 两个都是常量，直接计算
        result_data[0] = ldata[0] + rdata[0];
        result.SetVectorType(VectorType::CONSTANT_VECTOR);
        return;
    }

    auto &lvalidity = FlatVector::Validity(left);
    auto &rvalidity = FlatVector::Validity(right);
    auto &result_validity = FlatVector::Validity(result);

    if (lvalidity.AllValid() && rvalidity.AllValid()) {
        // 无NULL值，使用SIMD快速路径
        VectorOperations::ExecuteSIMD<T, T, T, AddOperator>(
            ldata, rdata, result_data, count);
        result_validity.SetAllValid(count);
    } else {
        // 有NULL值，需要处理ValidityMask
        for (idx_t i = 0; i < count; i++) {
            if (lvalidity.RowIsValid(i) && rvalidity.RowIsValid(i)) {
                result_data[i] = ldata[i] + rdata[i];
                result_validity.SetValid(i);
            } else {
                result_validity.SetInvalid(i);
            }
        }
    }
}

// SIMD执行模板
template <class OP>
void ExecuteSIMD(const int32_t *left, const int32_t *right,
                 int32_t *result, idx_t count) {
#ifdef __AVX2__
    idx_t i = 0;
    for (; i + 8 <= count; i += 8) {
        __m256i l = _mm256_loadu_si256((__m256i*)&left[i]);
        __m256i r = _mm256_loadu_si256((__m256i*)&right[i]);
        __m256i res = OP::template Operation<__m256i>(l, r);
        _mm256_storeu_si256((__m256i*)&result[i], res);
    }

    // 处理剩余元素
    for (; i < count; i++) {
        result[i] = OP::Operation(left[i], right[i]);
    }
#else
    // 无SIMD支持，fallback到标量实现
    for (idx_t i = 0; i < count; i++) {
        result[i] = OP::Operation(left[i], right[i]);
    }
#endif
}
```

### 20.3 分支预测优化

```cpp
// 避免分支的技巧

// 不好的做法：频繁分支
void filter_naive(int32_t *data, bool *filter, int32_t *output,
                  idx_t count, idx_t &output_count) {
    output_count = 0;
    for (idx_t i = 0; i < count; i++) {
        if (filter[i]) {  // 分支，性能杀手
            output[output_count++] = data[i];
        }
    }
}

// 更好的做法：使用SelectionVector，延迟分支
void filter_optimized(int32_t *data, bool *filter, SelectionVector &sel,
                      idx_t count, idx_t &output_count) {
    output_count = 0;
    for (idx_t i = 0; i < count; i++) {
        // 使用算术操作避免分支
        sel.set_index(output_count, i);
        output_count += filter[i];  // 0或1
    }
}

// 最优做法：SIMD + 无分支
void filter_simd(int32_t *data, bool *filter, SelectionVector &sel,
                 idx_t count, idx_t &output_count) {
#ifdef __AVX2__
    output_count = 0;
    idx_t i = 0;

    for (; i + 8 <= count; i += 8) {
        // 加载8个bool值
        __m128i mask = _mm_loadu_si128((__m128i*)&filter[i]);

        // 转换为32位mask
        __m256i mask32 = _mm256_cvtepi8_epi32(mask);

        // 生成索引
        __m256i indices = _mm256_set_epi32(i+7, i+6, i+5, i+4, i+3, i+2, i+1, i);

        // 压缩非零索引
        // 这里使用AVX-512的compress指令或者手动实现
        // ...

        // 更新output_count
        int popcount = _mm_popcnt_u32(_mm256_movemask_ps(_mm256_castsi256_ps(mask32)));
        output_count += popcount;
    }

    // 处理剩余元素
    for (; i < count; i++) {
        sel.set_index(output_count, i);
        output_count += filter[i];
    }
#else
    filter_optimized(data, filter, sel, count, output_count);
#endif
}
```

### 20.4 缓存友好的数据布局

**列式存储的优势：**
```cpp
// 行式存储（不友好）
struct RowStore {
    struct Row {
        int32_t id;
        int32_t age;
        double salary;
        char name[32];
    };
    Row rows[10000];

    // 计算age的总和
    int64_t sum_age() {
        int64_t sum = 0;
        for (int i = 0; i < 10000; i++) {
            // 每次访问都要跳过其他字段，缓存不友好
            sum += rows[i].age;
        }
        return sum;
    }
};

// 列式存储（友好）
struct ColumnStore {
    int32_t id[10000];
    int32_t age[10000];      // 连续存储
    double salary[10000];
    char names[10000][32];

    // 计算age的总和
    int64_t sum_age() {
        int64_t sum = 0;
        for (int i = 0; i < 10000; i++) {
            // 连续访问，缓存命中率高
            sum += age[i];
        }
        return sum;
    }
};
```

**性能对比：**
- 行式：每次访问需要加载整行（~48字节），只用其中4字节
- 列式：连续访问age列，充分利用缓存行（64字节）

### 20.5 预取（Prefetching）

```cpp
// 手动预取数据到缓存
void hash_probe_with_prefetch(HashTable &ht, Vector &keys, Vector &result, idx_t count) {
    auto key_data = FlatVector::GetData<int64_t>(keys);

    // 预取距离（根据缓存延迟调整）
    constexpr idx_t PREFETCH_DISTANCE = 16;

    for (idx_t i = 0; i < count; i++) {
        // 预取未来的键
        if (i + PREFETCH_DISTANCE < count) {
            idx_t prefetch_key = key_data[i + PREFETCH_DISTANCE];
            idx_t bucket = prefetch_key % ht.capacity;
            __builtin_prefetch(&ht.buckets[bucket], 0, 1);  // 预取到L2缓存
        }

        // 处理当前键
        idx_t key = key_data[i];
        idx_t bucket = key % ht.capacity;
        auto entry = ht.buckets[bucket];  // 可能已在缓存中

        // 探测链表
        while (entry) {
            if (entry->key == key) {
                // 找到匹配
                result.SetValue(i, entry->value);
                break;
            }
            entry = entry->next;
        }
    }
}
```

### 20.6 向量化Hash聚合

```cpp
// src/execution/aggregate_hashtable.cpp
void GroupedAggregateHashTable::AddChunk(DataChunk &groups, DataChunk &payload) {
    // 1. 批量计算Hash值
    Vector hashes(LogicalType::HASH);
    VectorOperations::Hash(groups.data[0], hashes, groups.size());

    for (idx_t col = 1; col < groups.ColumnCount(); col++) {
        VectorOperations::CombineHash(hashes, groups.data[col], groups.size());
    }

    auto hash_data = FlatVector::GetData<hash_t>(hashes);

    // 2. 批量查找/插入
    Vector addresses(LogicalType::POINTER);  // 聚合状态地址
    auto addr_data = FlatVector::GetData<data_ptr_t>(addresses);

    for (idx_t i = 0; i < groups.size(); i++) {
        hash_t hash = hash_data[i];
        idx_t bucket = hash & (capacity - 1);

        // 查找或创建entry
        auto entry = FindOrCreateGroup(groups, i, hash, bucket);
        addr_data[i] = GetAggregateStateAddress(entry);
    }

    // 3. 向量化更新聚合状态
    for (idx_t aggr_idx = 0; aggr_idx < aggregates.size(); aggr_idx++) {
        auto &aggr = aggregates[aggr_idx];

        // 调用向量化update函数
        aggr.function.update(
            payload.data,          // 输入列
            aggr.bind_data.get(),
            groups.size(),
            addresses,             // 聚合状态地址向量
            payload.size()
        );
    }
}

// 向量化的SUM更新
void SumUpdateVectorized(Vector inputs[], AggregateInputData &aggr_input_data,
                        idx_t input_count, Vector &states, idx_t count) {
    auto input_data = FlatVector::GetData<int32_t>(inputs[0]);
    auto state_ptrs = FlatVector::GetData<data_ptr_t>(states);
    auto &validity = FlatVector::Validity(inputs[0]);

    // 向量化更新所有状态
    for (idx_t i = 0; i < count; i++) {
        if (validity.RowIsValid(i)) {
            auto state = (SumState*)state_ptrs[i];
            state->sum += input_data[i];
            state->is_set = true;
        }
    }
}
```

### 20.7 编译器优化Hints

```cpp
// 使用编译器hints优化性能

// 1. 标记热点函数
__attribute__((hot))
void CriticalPath(Vector &input, Vector &output, idx_t count) {
    // 编译器会更积极地内联和优化
}

// 2. 分支预测提示
if (__builtin_expect(count > 0, 1)) {  // likely
    ProcessData();
} else {
    HandleEmpty();  // unlikely
}

// 3. 循环展开提示
#pragma GCC unroll 4
for (idx_t i = 0; i < count; i++) {
    result[i] = input[i] * 2;
}

// 4. 对齐提示
alignas(64) int32_t data[VECTOR_SIZE];  // 缓存行对齐

// 5. restrict指针（无别名）
void add(int32_t * __restrict__ a,
         int32_t * __restrict__ b,
         int32_t * __restrict__ result,
         size_t n) {
    // 编译器知道a、b、result不重叠，可以向量化
    for (size_t i = 0; i < n; i++) {
        result[i] = a[i] + b[i];
    }
}
```

**实践任务：**
1. 阅读 `src/common/vector_operations/` 下的向量化实现
2. 比较标量和SIMD版本的性能
3. 实现一个向量化的过滤操作
4. 测量缓存命中率和性能差异

---

## Day 21: 第三周总结 - 实现优化规则

**学习目标：** 综合运用第三周知识，实现完整的查询优化器

### 21.1 第三周知识回顾

- **Day 15:** 优化器架构 - 规则驱动的设计
- **Day 16:** Filter Pushdown - 下推过滤条件
- **Day 17:** Join Order优化 - 动态规划算法
- **Day 18:** 统计信息 - HyperLogLog、基数估计
- **Day 19:** 表达式优化 - 常量折叠、CSE
- **Day 20:** 向量化执行 - SIMD、缓存优化

### 21.2 实践项目：SimpleOptimizer

实现一个简化版查询优化器：

```cpp
// simple_optimizer.hpp
#pragma once

#include "simple_query_engine.hpp"
#include <memory>

namespace duckdb {

// 优化规则基类
class OptimizationRule {
public:
    virtual ~OptimizationRule() = default;
    virtual unique_ptr<SimpleOperator> Apply(unique_ptr<SimpleOperator> plan) = 0;
    virtual string Name() const = 0;
};

// 主优化器
class SimpleOptimizer {
    vector<unique_ptr<OptimizationRule>> rules;

public:
    SimpleOptimizer() {
        // 注册优化规则（按顺序执行）
        RegisterRule(make_unique<FilterPushdownRule>());
        RegisterRule(make_unique<ProjectionPushdownRule>());
        RegisterRule(make_unique<RemoveRedundantProjectionRule>());
        RegisterRule(make_unique<JoinOrderRule>());
    }

    void RegisterRule(unique_ptr<OptimizationRule> rule) {
        rules.push_back(std::move(rule));
    }

    unique_ptr<SimpleOperator> Optimize(unique_ptr<SimpleOperator> plan) {
        printf("=== 开始优化 ===\n");
        PrintPlan(plan.get(), 0);

        // 依次应用所有规则
        for (auto &rule : rules) {
            printf("\n应用规则: %s\n", rule->Name().c_str());
            plan = rule->Apply(std::move(plan));
            PrintPlan(plan.get(), 0);
        }

        printf("\n=== 优化完成 ===\n");
        return plan;
    }

private:
    void PrintPlan(SimpleOperator *op, int indent) {
        // 打印执行计划（用于调试）
        for (int i = 0; i < indent; i++) printf("  ");
        printf("%s\n", op->ToString().c_str());

        for (auto child : op->GetChildren()) {
            PrintPlan(child, indent + 1);
        }
    }
};

} // namespace duckdb
```

### 21.3 Filter Pushdown规则

```cpp
// filter_pushdown_rule.hpp
class FilterPushdownRule : public OptimizationRule {
public:
    string Name() const override { return "Filter Pushdown"; }

    unique_ptr<SimpleOperator> Apply(unique_ptr<SimpleOperator> plan) override {
        return PushdownRecursive(std::move(plan));
    }

private:
    unique_ptr<SimpleOperator> PushdownRecursive(unique_ptr<SimpleOperator> op) {
        // 先递归优化子节点
        op->TransformChildren([this](unique_ptr<SimpleOperator> child) {
            return PushdownRecursive(std::move(child));
        });

        // 检查是否是Filter算子
        auto filter = dynamic_cast<SimpleFilter*>(op.get());
        if (!filter) {
            return op;
        }

        auto child = filter->GetChild();

        // 情况1: Filter -> TableScan
        if (auto scan = dynamic_cast<SimpleTableScan*>(child)) {
            // 下推到TableScan
            return make_unique<SimpleTableScanWithFilter>(
                scan->GetTable(),
                filter->GetPredicate()
            );
        }

        // 情况2: Filter -> Projection
        if (auto proj = dynamic_cast<SimpleProjection*>(child)) {
            // 交换Filter和Projection的顺序
            // Filter -> Projection -> Child
            // 变为: Projection -> Filter -> Child

            auto filter_child = proj->GetChild();

            // 重写Filter中的列引用（因为Projection改变了列索引）
            auto rewritten_predicate = RewriteFilterPredicate(
                filter->GetPredicate(),
                proj->GetColumnIndices()
            );

            // 创建新计划
            auto new_filter = make_unique<SimpleFilter>(
                proj->TakeChild(),
                rewritten_predicate
            );

            proj->SetChild(std::move(new_filter));
            return proj->TakeOwnership();
        }

        // 情况3: Filter -> Join
        if (auto join = dynamic_cast<SimpleHashJoin*>(child)) {
            // 分析过滤条件涉及的表
            auto filter_bindings = AnalyzeFilterBindings(filter->GetPredicate());

            if (filter_bindings.only_left) {
                // 只涉及左表，下推到左侧
                auto new_filter = make_unique<SimpleFilter>(
                    join->TakeLeftChild(),
                    filter->GetPredicate()
                );
                join->SetLeftChild(std::move(new_filter));
                return join->TakeOwnership();
            } else if (filter_bindings.only_right) {
                // 只涉及右表，下推到右侧
                auto new_filter = make_unique<SimpleFilter>(
                    join->TakeRightChild(),
                    filter->GetPredicate()
                );
                join->SetRightChild(std::move(new_filter));
                return join->TakeOwnership();
            }
            // else: 涉及两表，保持在Join之上
        }

        return op;
    }

    struct FilterBindings {
        bool only_left;
        bool only_right;
        bool both;
    };

    FilterBindings AnalyzeFilterBindings(
        const std::function<bool(DataChunk&, idx_t)> &predicate) {
        // 简化：通过启发式判断
        // 实际应该分析表达式的列引用
        return {false, false, true};
    }

    std::function<bool(DataChunk&, idx_t)> RewriteFilterPredicate(
        const std::function<bool(DataChunk&, idx_t)> &predicate,
        const vector<idx_t> &column_mapping) {
        // 重写谓词中的列索引
        // 简化：返回原谓词
        return predicate;
    }
};
```

### 21.4 Projection下推规则

```cpp
class ProjectionPushdownRule : public OptimizationRule {
public:
    string Name() const override { return "Projection Pushdown"; }

    unique_ptr<SimpleOperator> Apply(unique_ptr<SimpleOperator> plan) override {
        // 收集所有被使用的列
        unordered_set<idx_t> used_columns;
        CollectUsedColumns(plan.get(), used_columns);

        // 在TableScan处添加列裁剪
        return PushdownColumns(std::move(plan), used_columns);
    }

private:
    void CollectUsedColumns(SimpleOperator *op, unordered_set<idx_t> &used_columns) {
        // 收集此算子使用的列
        for (auto col : op->GetReferencedColumns()) {
            used_columns.insert(col);
        }

        // 递归收集子节点
        for (auto child : op->GetChildren()) {
            CollectUsedColumns(child, used_columns);
        }
    }

    unique_ptr<SimpleOperator> PushdownColumns(
        unique_ptr<SimpleOperator> op,
        const unordered_set<idx_t> &used_columns) {

        // 递归处理子节点
        op->TransformChildren([&](unique_ptr<SimpleOperator> child) {
            return PushdownColumns(std::move(child), used_columns);
        });

        // 如果是TableScan，裁剪不需要的列
        if (auto scan = dynamic_cast<SimpleTableScan*>(op.get())) {
            scan->SetProjectedColumns(
                vector<idx_t>(used_columns.begin(), used_columns.end())
            );
        }

        return op;
    }
};
```

### 21.5 移除冗余Projection

```cpp
class RemoveRedundantProjectionRule : public OptimizationRule {
public:
    string Name() const override { return "Remove Redundant Projection"; }

    unique_ptr<SimpleOperator> Apply(unique_ptr<SimpleOperator> plan) override {
        return RemoveRecursive(std::move(plan));
    }

private:
    unique_ptr<SimpleOperator> RemoveRecursive(unique_ptr<SimpleOperator> op) {
        // 递归处理子节点
        op->TransformChildren([this](unique_ptr<SimpleOperator> child) {
            return RemoveRecursive(std::move(child));
        });

        // 检查是否是Projection
        auto proj = dynamic_cast<SimpleProjection*>(op.get());
        if (!proj) {
            return op;
        }

        // 检查是否是恒等投影（所有列按顺序）
        auto &cols = proj->GetColumnIndices();
        bool is_identity = true;
        for (idx_t i = 0; i < cols.size(); i++) {
            if (cols[i] != i) {
                is_identity = false;
                break;
            }
        }

        if (is_identity) {
            // 移除冗余的Projection
            return proj->TakeChild();
        }

        return op;
    }
};
```

### 21.6 简单的Join Order优化

```cpp
class JoinOrderRule : public OptimizationRule {
public:
    string Name() const override { return "Join Order"; }

    unique_ptr<SimpleOperator> Apply(unique_ptr<SimpleOperator> plan) override {
        return ReorderJoins(std::move(plan));
    }

private:
    unique_ptr<SimpleOperator> ReorderJoins(unique_ptr<SimpleOperator> op) {
        // 简化版：只处理两个Join的情况
        // (A JOIN B) JOIN C

        auto top_join = dynamic_cast<SimpleHashJoin*>(op.get());
        if (!top_join) {
            return op;
        }

        auto left_join = dynamic_cast<SimpleHashJoin*>(top_join->GetLeftChild());
        if (!left_join) {
            return op;
        }

        // 现在有: ((A JOIN B) JOIN C)
        // 三个表: A, B, C
        // 估计不同Join顺序的成本

        auto table_a = left_join->GetLeftChild();
        auto table_b = left_join->GetRightChild();
        auto table_c = top_join->GetRightChild();

        idx_t card_a = EstimateCardinality(table_a);
        idx_t card_b = EstimateCardinality(table_b);
        idx_t card_c = EstimateCardinality(table_c);

        // 启发式：先Join小表
        // 尝试 (A JOIN C) JOIN B
        if (card_c < card_b) {
            // 重新组织Join
            auto new_left_join = make_unique<SimpleHashJoin>(
                table_a->Copy(),
                table_c->Copy(),
                /* join keys */
            );

            auto new_top_join = make_unique<SimpleHashJoin>(
                std::move(new_left_join),
                table_b->Copy(),
                /* join keys */
            );

            return new_top_join;
        }

        return op;
    }

    idx_t EstimateCardinality(SimpleOperator *op) {
        // 简化：返回固定值或者从表统计信息获取
        if (auto scan = dynamic_cast<SimpleTableScan*>(op)) {
            return scan->GetTable().GetRowCount();
        }
        return 1000;  // 默认值
    }
};
```

### 21.7 完整示例

```cpp
void TestOptimizer() {
    using namespace duckdb;

    // 创建表
    SimpleTable students(...);
    SimpleTable classes(...);

    // 构建未优化的计划
    // SELECT name FROM students WHERE age > 18
    auto scan = make_unique<SimpleTableScan>(students);

    auto projection = make_unique<SimpleProjection>(
        std::move(scan),
        vector<idx_t>{0, 1, 2, 3}  // 所有列
    );

    auto filter = make_unique<SimpleFilter>(
        std::move(projection),
        [](DataChunk &chunk, idx_t row) {
            return chunk.GetValue(2, row).GetValue<int32_t>() > 18;  // age > 18
        }
    );

    auto final_projection = make_unique<SimpleProjection>(
        std::move(filter),
        vector<idx_t>{1}  // 只要name列
    );

    printf("=== 原始计划 ===\n");
    printf("Projection [name]\n");
    printf("  Filter [age > 18]\n");
    printf("    Projection [*]\n");
    printf("      TableScan [students]\n");

    // 应用优化
    SimpleOptimizer optimizer;
    auto optimized = optimizer.Optimize(std::move(final_projection));

    printf("\n=== 优化后计划 ===\n");
    printf("Projection [name]\n");
    printf("  TableScanWithFilter [students, age > 18]\n");
    printf("    (Filter已下推，冗余Projection已移除)\n");

    // 执行优化后的计划
    DataChunk result;
    while (optimized->GetNext(result)) {
        // 处理结果
    }
}
```

### 21.8 优化效果对比

```cpp
// 性能测试
void BenchmarkOptimization() {
    // 准备大数据集
    SimpleTable large_table = CreateLargeTable(1000000);  // 100万行

    // 未优化的计划
    auto unoptimized = BuildUnoptimizedPlan(large_table);

    // 优化后的计划
    SimpleOptimizer optimizer;
    auto optimized = optimizer.Optimize(unoptimized->Copy());

    // 测试未优化版本
    auto start = std::chrono::high_resolution_clock::now();
    idx_t unopt_count = ExecutePlan(unoptimized.get());
    auto unopt_time = std::chrono::high_resolution_clock::now() - start;

    // 测试优化版本
    start = std::chrono::high_resolution_clock::now();
    idx_t opt_count = ExecutePlan(optimized.get());
    auto opt_time = std::chrono::high_resolution_clock::now() - start;

    printf("未优化: %llu ms, %llu rows\n",
           std::chrono::duration_cast<std::chrono::milliseconds>(unopt_time).count(),
           unopt_count);
    printf("优化后: %llu ms, %llu rows\n",
           std::chrono::duration_cast<std::chrono::milliseconds>(opt_time).count(),
           opt_count);
    printf("加速比: %.2fx\n",
           (double)unopt_time.count() / opt_time.count());
}
```

**实践任务：**
1. 实现完整的SimpleOptimizer
2. 添加更多优化规则（表达式简化、常量折叠等）
3. 测试优化效果
4. 分析不同优化规则的性能影响

**第三周总结：**

| 优化技术 | 典型加速 | 适用场景 |
|----------|----------|----------|
| Filter Pushdown | 2-10x | 过滤率高的查询 |
| Join Order | 10-100x | 多表Join |
| 列裁剪 | 1.5-3x | 宽表查询 |
| 常量折叠 | 1.2-2x | 复杂表达式 |
| SIMD向量化 | 2-8x | 数值计算密集 |

---

## 第四周：存储引擎

### Day 22: 存储引擎架构

**学习目标：** 理解DuckDB存储引擎的整体架构

### 22.1 存储层次结构

```
Database
├── StorageManager
│   ├── BufferManager (内存管理)
│   ├── BlockManager (块管理)
│   └── WAL (Write-Ahead Log)
├── Catalog
│   └── TableCatalogEntry
│       └── DataTable
│           └── RowGroupCollection
│               ├── RowGroup 1
│               ├── RowGroup 2
│               └── ...
└── TransactionManager
```

### 22.2 关键组件

**1. DataTable** - 表的逻辑表示
```cpp
// src/storage/data_table.hpp
class DataTable {
public:
    // 表元数据
    TableCatalogEntry &table_entry;
    vector<LogicalType> types;

    // 数据存储
    unique_ptr<RowGroupCollection> row_groups;

    // 持久化指针
    shared_ptr<DataTableInfo> info;

    // 操作接口
    void Append(DataChunk &chunk, Transaction &transaction);
    void Scan(TableScanState &state, DataChunk &result);
    void Update(UpdateInfo &info, Transaction &transaction);
    void Delete(DeleteInfo &info, Transaction &transaction);
};
```

**2. RowGroupCollection** - RowGroup的集合
```cpp
class RowGroupCollection {
    vector<unique_ptr<RowGroup>> row_groups;
    idx_t total_rows;

public:
    void Append(DataChunk &chunk);
    void Scan(TableScanState &state, DataChunk &result);

    // 获取统计信息
    unique_ptr<BaseStatistics> GetStatistics(idx_t column_idx);
};
```

**3. RowGroup** - 固定行数的数据块
```cpp
// src/storage/table/row_group.hpp
class RowGroup {
    static constexpr idx_t ROW_GROUP_SIZE = 122880;  // ~120K行

    RowGroupCollection &collection;
    idx_t start;         // 起始行号
    idx_t count;         // 当前行数

    // 每列的数据
    vector<unique_ptr<ColumnData>> columns;

    // 版本信息（MVCC）
    unique_ptr<VersionInfo> version_info;

public:
    void Append(DataChunk &chunk);
    void Scan(ColumnScanState &state, Vector &result);

    // 更新/删除
    void Update(TransactionData &transaction, idx_t column_idx,
                Vector &update_vector, row_t *ids, idx_t count);
    void Delete(TransactionData &transaction, row_t *ids, idx_t count);
};
```

### 22.3 列式存储布局

```cpp
// ColumnData - 单列的数据
class ColumnData {
    LogicalType type;
    idx_t start;

    // Segments链表
    unique_ptr<ColumnSegment> data;

public:
    void Append(ColumnAppendState &state, Vector &vector, idx_t count);
    void Scan(ColumnScanState &state, Vector &result);

    // 统计信息
    unique_ptr<BaseStatistics> GetStatistics();
};

// ColumnSegment - 列的一段数据
class ColumnSegment {
    static constexpr idx_t SEGMENT_SIZE = 2048 * STANDARD_VECTOR_SIZE;  // ~4M values

    ColumnData &column;
    idx_t start;
    idx_t count;

    // 实际数据存储
    unique_ptr<ColumnSegmentState> segment_state;
    BufferHandle buffer_handle;  // 指向BufferManager中的块

    // 压缩信息
    CompressionType compression_type;

public:
    void Scan(ColumnScanState &state, Vector &result, idx_t count);
    void Append(ColumnAppendState &state, Vector &data, idx_t offset, idx_t count);

    // 压缩/解压缩
    void Compress();
    void Decompress(Vector &result, idx_t offset, idx_t count);
};
```

### 22.4 数据布局示例

```
Table: students (100万行)
├── RowGroup 0 [0-122879]
│   ├── Column 0 (id: INTEGER)
│   │   ├── Segment 0 [0-524287]      - 压缩: FOR
│   │   └── Segment 1 [524288-...]    - 压缩: FOR
│   ├── Column 1 (name: VARCHAR)
│   │   ├── Segment 0 [0-524287]      - 压缩: Dictionary
│   │   └── Segment 1 [524288-...]    - 压缩: Dictionary
│   └── Column 2 (score: DOUBLE)
│       ├── Segment 0 [0-524287]      - 压缩: Uncompressed
│       └── Segment 1 [524288-...]
├── RowGroup 1 [122880-245759]
│   └── ...
└── RowGroup 8 [983040-999999]
    └── ...
```

**实践任务：**
1. 阅读 `src/storage/data_table.hpp`
2. 理解RowGroup和ColumnData的关系
3. 绘制存储层次结构图

---

**(继续Day 23-28的详细内容...)**

本文档提供了第三周完整内容和第四周开篇，剩余Day 23-30的详细内容将在下一个文件中继续。
