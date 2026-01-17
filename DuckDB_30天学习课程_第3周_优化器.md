# DuckDB 30天学习课程 - 第三周：优化器与性能

本文档包含第三周（Day 16-21）的详细内容，深入学习查询优化技术。

---

## Day 16: Filter Pushdown优化

**学习目标：** 理解Filter Pushdown的原理和实现

### 16.1 Filter Pushdown基本概念

Filter Pushdown是最重要的优化之一，将过滤条件尽可能下推到数据源附近：

**未优化：**
```
LogicalProjection [name]
└── LogicalFilter [age > 25]
    └── LogicalJoin [students.class_id = classes.id]
        ├── LogicalGet [students]
        └── LogicalGet [classes]
```

**优化后：**
```
LogicalProjection [name]
└── LogicalJoin [students.class_id = classes.id]
    ├── LogicalFilter [age > 25]
    │   └── LogicalGet [students]
    └── LogicalGet [classes]
```

**优势：**
- 减少Join的输入数据量
- 在存储层应用过滤（减少I/O）
- 提前剪枝，减少内存使用

### 16.2 FilterPushdown实现

```cpp
// src/optimizer/filter_pushdown.cpp
class FilterPushdown {
public:
    unique_ptr<LogicalOperator> Rewrite(unique_ptr<LogicalOperator> op) {
        // 后序遍历（先优化子节点）
        for (auto &child : op->children) {
            child = Rewrite(std::move(child));
        }

        // 根据算子类型处理
        switch (op->type) {
        case LogicalOperatorType::LOGICAL_FILTER:
            return PushdownFilter(std::move(op));
        case LogicalOperatorType::LOGICAL_COMPARISON_JOIN:
            return PushdownJoin(std::move(op));
        case LogicalOperatorType::LOGICAL_AGGREGATE_AND_GROUP_BY:
            return PushdownAggregate(std::move(op));
        default:
            return op;
        }
    }

private:
    unique_ptr<LogicalOperator> PushdownFilter(unique_ptr<LogicalOperator> op);
    unique_ptr<LogicalOperator> PushdownJoin(unique_ptr<LogicalOperator> op);
    unique_ptr<LogicalOperator> PushdownAggregate(unique_ptr<LogicalOperator> op);
};
```

### 16.3 过滤条件分解

```cpp
// 将AND连接的过滤条件分解
vector<unique_ptr<Expression>> FilterPushdown::SplitConjunction(Expression &expr) {
    vector<unique_ptr<Expression>> filters;

    if (expr.type == ExpressionType::CONJUNCTION_AND) {
        auto &conj = (BoundConjunctionExpression&)expr;
        for (auto &child : conj.children) {
            auto split = SplitConjunction(*child);
            filters.insert(filters.end(),
                          std::make_move_iterator(split.begin()),
                          std::make_move_iterator(split.end()));
        }
    } else {
        filters.push_back(expr.Copy());
    }

    return filters;
}
```

**示例：**
```sql
WHERE age > 25 AND score > 80 AND name LIKE 'A%'

-- 分解为：
filters = [
    age > 25,
    score > 80,
    name LIKE 'A%'
]
```

### 16.4 下推到TableScan

```cpp
unique_ptr<LogicalOperator> FilterPushdown::PushdownFilter(
    unique_ptr<LogicalOperator> op) {

    auto &filter = (LogicalFilter&)*op;
    auto &child = *filter.children[0];

    // 情况1: 下推到TableScan
    if (child.type == LogicalOperatorType::LOGICAL_GET) {
        auto &get = (LogicalGet&)child;

        // 分解过滤条件
        vector<unique_ptr<Expression>> filters;
        for (auto &expr : filter.expressions) {
            auto split = SplitConjunction(*expr);
            filters.insert(filters.end(),
                          std::make_move_iterator(split.begin()),
                          std::make_move_iterator(split.end()));
        }

        // 尝试将过滤条件转换为TableFilter
        vector<unique_ptr<Expression>> remaining_filters;
        for (auto &filter_expr : filters) {
            if (CanPushdownToStorage(filter_expr, get)) {
                // 转换为TableFilter并添加到Get
                auto table_filter = ConvertToTableFilter(filter_expr);
                get.table_filters->AddFilter(std::move(table_filter));
            } else {
                // 无法下推，保留在Filter算子
                remaining_filters.push_back(std::move(filter_expr));
            }
        }

        // 如果所有过滤条件都下推了，移除Filter算子
        if (remaining_filters.empty()) {
            return std::move(filter.children[0]);
        }

        // 否则更新Filter的条件
        filter.expressions = std::move(remaining_filters);
        return op;
    }

    // 其他情况...
    return op;
}
```

### 16.5 TableFilter类型

```cpp
// src/include/duckdb/planner/table_filter.hpp
enum class TableFilterType : uint8_t {
    CONSTANT_COMPARISON,  // column OP constant (如 age > 25)
    IS_NULL,              // column IS NULL
    IS_NOT_NULL,          // column IS NOT NULL
    CONJUNCTION_OR,       // OR组合
    CONJUNCTION_AND       // AND组合
};

class TableFilter {
public:
    TableFilterType filter_type;

    virtual bool CheckStatistics(BaseStatistics &stats) = 0;
    virtual bool ApplyFilter(Vector &input) = 0;
};

// 常量比较过滤器
class ConstantFilter : public TableFilter {
public:
    ExpressionType comparison_type;  // =, <, >, <=, >=, !=
    Value constant;

    bool CheckStatistics(BaseStatistics &stats) override {
        // 使用统计信息剪枝
        if (comparison_type == ExpressionType::COMPARE_GREATERTHAN) {
            // age > 25
            // 如果max(age) <= 25，整个segment可以跳过
            if (NumericStats::Max(stats) <= constant) {
                return false;  // 整个segment不满足
            }
        }
        return true;  // 可能有满足的行
    }

    bool ApplyFilter(Vector &input) override {
        // 在Vector上应用过滤
        // 返回SelectionVector或修改ValidityMask
        // ...
    }
};
```

### 16.6 下推到Join

```cpp
unique_ptr<LogicalOperator> FilterPushdown::PushdownJoin(
    unique_ptr<LogicalOperator> op) {

    auto &join = (LogicalComparisonJoin&)*op;

    // 收集可下推的过滤条件
    vector<unique_ptr<Expression>> left_filters;
    vector<unique_ptr<Expression>> right_filters;
    vector<unique_ptr<Expression>> join_filters;

    // 分析每个过滤条件涉及的列
    for (auto &filter : filters) {
        auto bindings = ExtractBindings(*filter);

        bool references_left = false;
        bool references_right = false;

        for (auto &binding : bindings) {
            if (IsLeftBinding(binding, join)) {
                references_left = true;
            }
            if (IsRightBinding(binding, join)) {
                references_right = true;
            }
        }

        if (references_left && !references_right) {
            // 只引用左表 -> 下推到左侧
            left_filters.push_back(std::move(filter));
        } else if (references_right && !references_left) {
            // 只引用右表 -> 下推到右侧
            right_filters.push_back(std::move(filter));
        } else {
            // 引用两表 -> 保留在Join之上
            join_filters.push_back(std::move(filter));
        }
    }

    // 将过滤条件下推到子节点
    if (!left_filters.empty()) {
        auto filter = make_unique<LogicalFilter>();
        filter->expressions = std::move(left_filters);
        filter->children.push_back(std::move(join.children[0]));
        join.children[0] = std::move(filter);
    }

    if (!right_filters.empty()) {
        auto filter = make_unique<LogicalFilter>();
        filter->expressions = std::move(right_filters);
        filter->children.push_back(std::move(join.children[1]));
        join.children[1] = std::move(filter);
    }

    return op;
}
```

**示例：**
```sql
SELECT * FROM students s JOIN classes c ON s.class_id = c.id
WHERE s.age > 25 AND c.capacity > 30 AND s.score + c.avg_score > 150

-- 优化后：
-- left_filters: [s.age > 25]  -> 下推到students
-- right_filters: [c.capacity > 30]  -> 下推到classes
-- join_filters: [s.score + c.avg_score > 150]  -> 保留在JOIN之上
```

### 16.7 下推到Aggregate

```cpp
unique_ptr<LogicalOperator> FilterPushdown::PushdownAggregate(
    unique_ptr<LogicalOperator> op) {

    auto &aggr = (LogicalAggregate&)*op;

    vector<unique_ptr<Expression>> pushdown_filters;
    vector<unique_ptr<Expression>> remaining_filters;

    for (auto &filter : filters) {
        // 检查过滤条件是否只涉及GROUP BY列
        if (OnlyReferencesGroups(*filter, aggr)) {
            // 可以下推到Aggregate之前
            pushdown_filters.push_back(std::move(filter));
        } else {
            // 涉及聚合函数，必须在Aggregate之后执行
            remaining_filters.push_back(std::move(filter));
        }
    }

    // 下推过滤条件
    if (!pushdown_filters.empty()) {
        auto filter = make_unique<LogicalFilter>();
        filter->expressions = std::move(pushdown_filters);
        filter->children.push_back(std::move(aggr.children[0]));
        aggr.children[0] = std::move(filter);
    }

    return op;
}

bool FilterPushdown::OnlyReferencesGroups(Expression &expr, LogicalAggregate &aggr) {
    // 检查表达式是否只引用GROUP BY列
    unordered_set<idx_t> group_bindings;
    for (auto &group : aggr.groups) {
        if (group->type == ExpressionType::BOUND_COLUMN_REF) {
            auto &colref = (BoundColumnRefExpression&)*group;
            group_bindings.insert(colref.binding.column_index);
        }
    }

    bool only_groups = true;
    ExpressionIterator::EnumerateExpression(expr, [&](Expression &child) {
        if (child.type == ExpressionType::BOUND_COLUMN_REF) {
            auto &colref = (BoundColumnRefExpression&)child;
            if (group_bindings.find(colref.binding.column_index) == group_bindings.end()) {
                only_groups = false;
            }
        }
    });

    return only_groups;
}
```

**示例：**
```sql
SELECT department, AVG(salary) as avg_sal
FROM employees
GROUP BY department
HAVING department IN ('IT', 'HR') AND AVG(salary) > 50000

-- WHERE优化：
-- department IN ('IT', 'HR')  -> 下推到employees扫描之前
-- AVG(salary) > 50000  -> 保留在HAVING（Aggregate之后）
```

### 16.8 实践：实现简单的Filter Pushdown

```cpp
// simple_filter_pushdown.hpp
#pragma once

#include "simple_query_engine.hpp"

namespace duckdb {

class SimpleFilterPushdown {
public:
    static unique_ptr<SimpleOperator> Optimize(unique_ptr<SimpleOperator> plan) {
        // 简化版：只处理Filter -> TableScan的情况

        // 检查是否是Filter算子
        auto filter = dynamic_cast<SimpleFilter*>(plan.get());
        if (!filter) {
            return plan;
        }

        // 检查子节点是否是TableScan
        auto scan = dynamic_cast<SimpleTableScan*>(filter->GetChild());
        if (!scan) {
            return plan;
        }

        // 将过滤条件下推到TableScan
        // 实际应该修改TableScan的实现以支持过滤
        // 这里简化为：如果能下推，直接在扫描时应用过滤

        // 创建新的TableScan with Filter
        auto new_scan = make_unique<SimpleTableScanWithFilter>(
            scan->GetTable(),
            filter->GetPredicate()
        );

        return new_scan;
    }
};

// 支持过滤的TableScan
class SimpleTableScanWithFilter : public SimpleOperator {
    SimpleTable &table;
    std::function<bool(DataChunk&, idx_t)> predicate;
    idx_t current_chunk_idx = 0;

public:
    SimpleTableScanWithFilter(SimpleTable &table,
                             std::function<bool(DataChunk&, idx_t)> pred)
        : table(table), predicate(pred) {}

    bool GetNext(DataChunk &result) override {
        while (current_chunk_idx < table.GetChunkCount()) {
            auto &chunk = table.GetChunk(current_chunk_idx++);

            // 应用过滤
            SelectionVector sel(STANDARD_VECTOR_SIZE);
            idx_t approved_count = 0;

            for (idx_t i = 0; i < chunk.size(); i++) {
                if (predicate(chunk, i)) {
                    sel.set_index(approved_count++, i);
                }
            }

            if (approved_count > 0) {
                result.Slice(chunk, sel, approved_count);
                result.SetCardinality(approved_count);
                return true;
            }
        }
        return false;
    }

    void Reset() override {
        current_chunk_idx = 0;
    }
};

} // namespace duckdb
```

**实践任务：**
1. 阅读 `src/optimizer/filter_pushdown.cpp`
2. 理解过滤条件的分解和重组
3. 实现简单的Filter Pushdown，支持下推到TableScan和Join

---

## Day 17: Join Order优化

**学习目标：** 理解Join Order优化的算法和成本模型

### 17.1 Join Order问题

对于多表Join，不同的Join顺序性能差异巨大：

```sql
SELECT * FROM A JOIN B ON A.id = B.id JOIN C ON B.id = C.id

-- 可能的Join顺序：
-- 1. (A JOIN B) JOIN C
-- 2. (A JOIN C) JOIN B
-- 3. (B JOIN C) JOIN A
-- 4. A JOIN (B JOIN C)
-- 5. B JOIN (A JOIN C)
-- 6. C JOIN (A JOIN B)
```

**问题规模：**
- N个表的Join：有 (2N-2)! / (N-1)! 种可能的顺序
- 3表：12种
- 4表：120种
- 10表：17,297,280种（穷举不可行）

### 17.2 JoinOrderOptimizer架构

```cpp
// src/optimizer/join_order_optimizer.hpp
class JoinOrderOptimizer {
public:
    unique_ptr<LogicalOperator> Optimize(unique_ptr<LogicalOperator> plan);

private:
    // 提取所有Join节点
    void ExtractJoins(LogicalOperator &op, vector<JoinNode> &join_nodes);

    // 动态规划算法
    unique_ptr<LogicalOperator> SolveJoinOrder(vector<JoinNode> &nodes);

    // 成本估算
    double EstimateCost(JoinNode &left, JoinNode &right, JoinCondition &condition);
};

struct JoinNode {
    unique_ptr<LogicalOperator> node;  // 基表或子Join
    idx_t cardinality;                 // 估计的行数
    unordered_set<idx_t> table_indices;  // 涉及的表
    unique_ptr<BaseStatistics> stats;  // 统计信息
};
```

### 17.3 动态规划算法（DP）

DuckDB使用改进的动态规划算法：

```cpp
// 简化版DP算法
unique_ptr<LogicalOperator> JoinOrderOptimizer::SolveJoinOrder(
    vector<JoinNode> &nodes) {

    idx_t n = nodes.size();

    // dp[set] = 访问set中表的最优计划
    unordered_map<idx_t, JoinPlan> dp;

    // 1. 初始化：单表访问
    for (idx_t i = 0; i < n; i++) {
        idx_t set = 1ULL << i;  // 位集合表示
        dp[set] = JoinPlan{
            .plan = nodes[i].node->Copy(),
            .cardinality = nodes[i].cardinality,
            .cost = 0  // 扫描成本
        };
    }

    // 2. 枚举所有子集大小（2到n）
    for (idx_t size = 2; size <= n; size++) {
        // 枚举所有size大小的子集
        for (idx_t set = 0; set < (1ULL << n); set++) {
            if (PopCount(set) != size) continue;

            // 枚举分割点
            for (idx_t subset = set; subset > 0; subset = (subset - 1) & set) {
                if (subset == set) continue;

                idx_t complement = set ^ subset;

                // 检查是否有Join条件连接subset和complement
                auto join_condition = FindJoinCondition(subset, complement);
                if (!join_condition) continue;

                // 计算成本
                auto &left_plan = dp[subset];
                auto &right_plan = dp[complement];

                double cost = left_plan.cost + right_plan.cost +
                             EstimateJoinCost(left_plan, right_plan, *join_condition);

                // 更新最优计划
                if (!dp.count(set) || cost < dp[set].cost) {
                    dp[set] = CreateJoinPlan(left_plan, right_plan,
                                            *join_condition, cost);
                }
            }
        }
    }

    // 3. 返回访问所有表的最优计划
    idx_t full_set = (1ULL << n) - 1;
    return std::move(dp[full_set].plan);
}

struct JoinPlan {
    unique_ptr<LogicalOperator> plan;
    idx_t cardinality;
    double cost;
};
```

### 17.4 基数估计

Join输出的行数估计：

```cpp
// 估计Join后的行数
idx_t EstimateJoinCardinality(JoinNode &left, JoinNode &right,
                              JoinCondition &condition) {
    idx_t left_card = left.cardinality;
    idx_t right_card = right.cardinality;

    if (condition.comparison == ExpressionType::COMPARE_EQUAL) {
        // 等值Join: R ⋈ S
        // 估计公式: |R| * |S| / max(V(R,a), V(S,b))
        // V(R,a) = R.a列的不同值数量（NDV）

        idx_t left_ndv = GetDistinctCount(left, condition.left);
        idx_t right_ndv = GetDistinctCount(right, condition.right);

        if (left_ndv == 0 || right_ndv == 0) {
            // 没有统计信息，使用启发式
            return left_card * right_card / 10;
        }

        idx_t max_ndv = std::max(left_ndv, right_ndv);
        return (left_card * right_card) / max_ndv;
    } else {
        // 非等值Join（如 <, >）
        // 使用默认选择率
        double selectivity = 0.1;  // 10%
        return (idx_t)(left_card * right_card * selectivity);
    }
}

// 获取列的不同值数量
idx_t GetDistinctCount(JoinNode &node, Expression &expr) {
    if (expr.type != ExpressionType::BOUND_COLUMN_REF) {
        return 0;  // 无法估计
    }

    auto &colref = (BoundColumnRefExpression&)expr;

    // 从统计信息获取NDV
    if (node.stats && node.stats->GetDistinctCount()) {
        return node.stats->GetDistinctCount()->GetIndex();
    }

    // 默认估计：行数的平方根
    return (idx_t)std::sqrt(node.cardinality);
}
```

### 17.5 成本模型

```cpp
// 估计Join的执行成本
double EstimateJoinCost(JoinPlan &left, JoinPlan &right, JoinCondition &condition) {
    // 简化的成本模型：
    // Cost = 读取成本 + 处理成本 + 写入成本

    double read_cost = left.cardinality + right.cardinality;

    // Hash Join成本
    // Build: 构建Hash表
    double build_cost = right.cardinality * HASH_BUILD_COST;

    // Probe: 探测Hash表
    double probe_cost = left.cardinality * HASH_PROBE_COST;

    // 输出成本
    idx_t output_card = EstimateJoinCardinality(left.node, right.node, condition);
    double output_cost = output_card * OUTPUT_COST;

    return read_cost + build_cost + probe_cost + output_cost;
}

// 成本常数（可调参数）
constexpr double HASH_BUILD_COST = 1.2;
constexpr double HASH_PROBE_COST = 1.0;
constexpr double OUTPUT_COST = 0.1;
```

### 17.6 剪枝策略

为了加速搜索，使用多种剪枝策略：

**1. 成本剪枝（Cost-based Pruning）**
```cpp
// 如果当前部分计划的成本已经超过已知的最优解，则剪枝
if (current_cost >= best_known_cost) {
    continue;  // 剪枝
}
```

**2. 统计剪枝（Statistics-based Pruning）**
```cpp
// 使用统计信息提前判断Join结果为空
if (CanPruneByStatistics(left.stats, right.stats, condition)) {
    continue;  // 剪枝
}

bool CanPruneByStatistics(BaseStatistics &left_stats,
                          BaseStatistics &right_stats,
                          JoinCondition &condition) {
    // 例如：如果left.max < right.min，则等值Join结果为空
    if (condition.comparison == ExpressionType::COMPARE_EQUAL) {
        if (NumericStats::Max(left_stats) < NumericStats::Min(right_stats)) {
            return true;  // 可以剪枝
        }
    }
    return false;
}
```

**3. Bushy vs Left-Deep Trees**

```cpp
// Left-Deep Tree（左深树）- 简化搜索空间
//     ⋈
//    / \
//   ⋈   D
//  / \
// ⋈   C
/// \
//A   B

// Bushy Tree（茂密树）- 搜索空间更大但可能找到更优计划
//     ⋈
//    / \
//   ⋈   ⋈
//  / \ / \
// A  B C  D

// 配置选项
enum class JoinOrderStrategy {
    LEFT_DEEP_ONLY,   // 只考虑左深树（快但可能不是最优）
    BUSHY_TREES,      // 考虑所有可能的树形（慢但更优）
    ADAPTIVE          // 根据表数量自适应选择
};
```

### 17.7 查询图（Query Graph）

```cpp
// 将Join转换为查询图
struct QueryGraph {
    vector<JoinNode> nodes;           // 顶点（表）
    vector<JoinEdge> edges;           // 边（Join条件）

    // 构建查询图
    static QueryGraph Build(LogicalOperator &op);
};

struct JoinEdge {
    idx_t left_table;
    idx_t right_table;
    unique_ptr<Expression> condition;
    double selectivity;
};

// 示例：
// SELECT * FROM A JOIN B ON A.id = B.id JOIN C ON B.id = C.id
//
// 查询图：
//   A --- B --- C
//
// nodes = [A, B, C]
// edges = [
//   {A, B, A.id = B.id},
//   {B, C, B.id = C.id}
// ]
```

### 17.8 实践：简单的Join Order优化

```cpp
// simple_join_order_optimizer.hpp
#pragma once

namespace duckdb {

class SimpleJoinOrderOptimizer {
public:
    // 简化版：只支持3表的穷举搜索
    static vector<SimpleOperator*> OptimizeJoinOrder(
        vector<SimpleTableScan*> tables,
        vector<JoinCondition> conditions) {

        D_ASSERT(tables.size() == 3);  // 只支持3表

        // 穷举所有可能的Join顺序
        double best_cost = std::numeric_limits<double>::max();
        vector<SimpleOperator*> best_plan;

        // 6种可能的顺序
        vector<vector<int>> orders = {
            {0, 1, 2},  // (A JOIN B) JOIN C
            {0, 2, 1},  // (A JOIN C) JOIN B
            {1, 0, 2},  // (B JOIN A) JOIN C
            {1, 2, 0},  // (B JOIN C) JOIN A
            {2, 0, 1},  // (C JOIN A) JOIN B
            {2, 1, 0}   // (C JOIN B) JOIN A
        };

        for (auto &order : orders) {
            auto plan = BuildJoinPlan(tables, conditions, order);
            double cost = EstimatePlanCost(plan);

            if (cost < best_cost) {
                best_cost = cost;
                best_plan = plan;
            }
        }

        return best_plan;
    }

private:
    static vector<SimpleOperator*> BuildJoinPlan(
        vector<SimpleTableScan*> tables,
        vector<JoinCondition> conditions,
        vector<int> order) {

        // 根据order构建Join计划
        // (tables[order[0]] JOIN tables[order[1]]) JOIN tables[order[2]]

        auto join1 = new SimpleHashJoin(
            tables[order[0]],
            tables[order[1]],
            /* join keys */
        );

        auto join2 = new SimpleHashJoin(
            join1,
            tables[order[2]],
            /* join keys */
        );

        return {join1, join2};
    }

    static double EstimatePlanCost(vector<SimpleOperator*> plan) {
        // 简化的成本估计
        double cost = 0;
        // ... 估计逻辑
        return cost;
    }
};

} // namespace duckdb
```

**实践任务：**
1. 阅读 `src/optimizer/join_order/join_order_optimizer.cpp`
2. 理解动态规划算法的实现
3. 实现一个简单的3表Join Order优化器（穷举法）
4. 测试不同Join顺序的性能差异

---

## Day 18: 统计信息与基数估计

**学习目标：** 理解统计信息的收集和使用

### 18.1 统计信息类型

DuckDB收集多种统计信息：

```cpp
// src/include/duckdb/storage/statistics/base_statistics.hpp
class BaseStatistics {
public:
    LogicalType type;
    unique_ptr<BaseStatistics> min;  // 最小值
    unique_ptr<BaseStatistics> max;  // 最大值
    unique_ptr<DistinctStatistics> distinct_stats;  // 不同值统计
    bool has_null = false;           // 是否有NULL值
};

// 数值统计
class NumericStatistics : public BaseStatistics {
public:
    Value min_value;
    Value max_value;

    static Value Min(BaseStatistics &stats) {
        return ((NumericStatistics&)stats).min_value;
    }

    static Value Max(BaseStatistics &stats) {
        return ((NumericStatistics&)stats).max_value;
    }
};

// 字符串统计
class StringStatistics : public BaseStatistics {
public:
    string min_value;
    string max_value;
    idx_t max_string_length;  // 最长字符串长度
};
```

### 18.2 HyperLogLog - 估计不同值数量

DuckDB使用HyperLogLog算法估计NDV（Number of Distinct Values）：

```cpp
// src/common/hll.hpp
class HyperLogLog {
    static constexpr idx_t HLL_BITS = 12;
    static constexpr idx_t HLL_BUCKETS = 1 << HLL_BITS;  // 4096

    uint8_t buckets[HLL_BUCKETS];

public:
    // 添加一个值
    void Add(hash_t hash) {
        // 使用前HLL_BITS位作为bucket索引
        idx_t bucket_idx = hash & (HLL_BUCKETS - 1);

        // 使用剩余位计算leading zeros
        hash_t value = hash >> HLL_BITS;
        uint8_t leading_zeros = CountLeadingZeros(value) + 1;

        // 保留最大的leading zeros
        buckets[bucket_idx] = std::max(buckets[bucket_idx], leading_zeros);
    }

    // 估计不同值数量
    idx_t Count() const {
        double sum = 0;
        idx_t zero_buckets = 0;

        for (idx_t i = 0; i < HLL_BUCKETS; i++) {
            if (buckets[i] == 0) {
                zero_buckets++;
            }
            sum += std::pow(2.0, -buckets[i]);
        }

        double alpha = GetAlpha();
        double estimate = alpha * HLL_BUCKETS * HLL_BUCKETS / sum;

        // 小范围修正
        if (estimate <= 2.5 * HLL_BUCKETS && zero_buckets > 0) {
            estimate = HLL_BUCKETS * std::log((double)HLL_BUCKETS / zero_buckets);
        }

        return (idx_t)estimate;
    }

private:
    static double GetAlpha() {
        // HyperLogLog的修正因子
        return 0.7213 / (1 + 1.079 / HLL_BUCKETS);
    }

    static uint8_t CountLeadingZeros(hash_t value) {
        if (value == 0) return 64;
        return __builtin_clzll(value);  // GCC/Clang内建函数
    }
};
```

**使用示例：**
```cpp
// 估计列的不同值数量
HyperLogLog hll;
Vector column_data = ...;
auto data = FlatVector::GetData<int64_t>(column_data);

for (idx_t i = 0; i < count; i++) {
    hash_t hash = Hash(data[i]);
    hll.Add(hash);
}

idx_t estimated_ndv = hll.Count();
printf("Estimated distinct values: %llu\n", estimated_ndv);
```

### 18.3 统计信息收集

```cpp
// src/storage/statistics/statistics_manager.cpp
class StatisticsManager {
public:
    // 为列收集统计信息
    unique_ptr<BaseStatistics> CollectColumnStatistics(ColumnData &column) {
        auto stats = make_unique<NumericStatistics>(column.type);

        // 扫描所有segments
        for (auto &segment : column.data.segments) {
            auto segment_stats = segment->GetStatistics();
            stats->Merge(*segment_stats);
        }

        return stats;
    }
};

// Segment级别统计
class ColumnSegment {
    unique_ptr<BaseStatistics> stats;

public:
    unique_ptr<BaseStatistics> GetStatistics() {
        if (!stats) {
            stats = ComputeStatistics();
        }
        return stats->Copy();
    }

private:
    unique_ptr<BaseStatistics> ComputeStatistics() {
        auto stats = make_unique<NumericStatistics>(type);

        // 扫描segment数据
        Vector data_vector(type);
        Scan(data_vector, count);

        auto data = FlatVector::GetData<int64_t>(data_vector);
        auto &validity = FlatVector::Validity(data_vector);

        for (idx_t i = 0; i < count; i++) {
            if (!validity.RowIsValid(i)) {
                stats->has_null = true;
                continue;
            }

            int64_t value = data[i];
            stats->UpdateMin(value);
            stats->UpdateMax(value);
        }

        return stats;
    }
};
```

### 18.4 选择率估计

```cpp
// 估计谓词的选择率
double EstimateSelectivity(Expression &expr, BaseStatistics &stats) {
    if (expr.type == ExpressionType::COMPARE_EQUAL) {
        // column = constant
        auto &comp = (BoundComparisonExpression&)expr;
        if (comp.right->IsFoldable()) {
            Value constant = ExpressionExecutor::EvaluateScalar(*comp.right);

            // 使用NDV估计
            if (stats.distinct_stats) {
                idx_t ndv = stats.distinct_stats->GetCount();
                return 1.0 / ndv;  // 1/不同值数量
            }

            // 默认选择率
            return 0.1;  // 10%
        }
    }

    if (expr.type == ExpressionType::COMPARE_GREATERTHAN) {
        // column > constant
        auto &comp = (BoundComparisonExpression&)expr;
        if (comp.right->IsFoldable()) {
            Value constant = ExpressionExecutor::EvaluateScalar(*comp.right);

            // 使用min/max估计
            if (stats.min && stats.max) {
                Value min_val = NumericStats::Min(stats);
                Value max_val = NumericStats::Max(stats);

                if (constant <= min_val) {
                    return 1.0;  // 所有值都满足
                }
                if (constant >= max_val) {
                    return 0.0;  // 没有值满足
                }

                // 线性插值
                double range = max_val.GetValue<double>() - min_val.GetValue<double>();
                double offset = constant.GetValue<double>() - min_val.GetValue<double>();
                return 1.0 - (offset / range);
            }

            // 默认选择率
            return 0.5;  // 50%
        }
    }

    // AND组合
    if (expr.type == ExpressionType::CONJUNCTION_AND) {
        auto &conj = (BoundConjunctionExpression&)expr;
        double selectivity = 1.0;
        for (auto &child : conj.children) {
            selectivity *= EstimateSelectivity(*child, stats);
        }
        return selectivity;
    }

    // OR组合
    if (expr.type == ExpressionType::CONJUNCTION_OR) {
        auto &conj = (BoundConjunctionExpression&)expr;
        double selectivity = 0.0;
        for (auto &child : conj.children) {
            double child_sel = EstimateSelectivity(*child, stats);
            selectivity = selectivity + child_sel - selectivity * child_sel;
        }
        return selectivity;
    }

    // 未知谓词，默认选择率
    return 0.5;
}
```

### 18.5 统计信息传播

```cpp
// src/optimizer/statistics_propagator.cpp
class StatisticsPropagator {
public:
    void PropagateStatistics(LogicalOperator &op) {
        // 后序遍历
        for (auto &child : op.children) {
            PropagateStatistics(*child);
        }

        // 根据算子类型传播统计信息
        switch (op.type) {
        case LogicalOperatorType::LOGICAL_FILTER:
            PropagateFilter((LogicalFilter&)op);
            break;
        case LogicalOperatorType::LOGICAL_PROJECTION:
            PropagateProjection((LogicalProjection&)op);
            break;
        case LogicalOperatorType::LOGICAL_COMPARISON_JOIN:
            PropagateJoin((LogicalComparisonJoin&)op);
            break;
        case LogicalOperatorType::LOGICAL_AGGREGATE_AND_GROUP_BY:
            PropagateAggregate((LogicalAggregate&)op);
            break;
        default:
            break;
        }
    }

private:
    void PropagateFilter(LogicalFilter &filter) {
        auto &child_stats = filter.children[0]->statistics;

        // 计算过滤后的基数
        double selectivity = 1.0;
        for (auto &expr : filter.expressions) {
            selectivity *= EstimateSelectivity(*expr, *child_stats);
        }

        filter.estimated_cardinality = (idx_t)(child_stats->cardinality * selectivity);

        // 更新统计信息（min/max可能变化）
        filter.statistics = UpdateStatistics(*child_stats, filter.expressions);
    }

    void PropagateJoin(LogicalComparisonJoin &join) {
        auto &left_stats = join.children[0]->statistics;
        auto &right_stats = join.children[1]->statistics;

        // 估计Join后的基数
        idx_t join_card = EstimateJoinCardinality(
            left_stats->cardinality,
            right_stats->cardinality,
            join.conditions
        );

        join.estimated_cardinality = join_card;
    }
};
```

**实践任务：**
1. 阅读 `src/storage/statistics/base_statistics.cpp`
2. 了解HyperLogLog算法原理
3. 实现一个简单的统计信息收集器
4. 实现选择率估计函数

---

## Day 19: 表达式优化与常量折叠

**学习目标：** 学习表达式级别的优化技术

### 19.1 常量折叠（Constant Folding）

在编译时计算常量表达式：

```cpp
// src/optimizer/expression_rewriter/constant_folding.cpp
class ConstantFolder : public ExpressionRewriter {
public:
    unique_ptr<Expression> Rewrite(unique_ptr<Expression> expr) override {
        // 递归重写子表达式
        ExpressionIterator::EnumerateChildren(*expr, [&](unique_ptr<Expression> &child) {
            child = Rewrite(std::move(child));
        });

        // 如果表达式可折叠，则计算值
        if (expr->IsFoldable()) {
            Value result = ExpressionExecutor::EvaluateScalar(*expr);
            return make_unique<BoundConstantExpression>(result);
        }

        return expr;
    }
};

// 判断表达式是否可折叠
bool Expression::IsFoldable() const {
    if (HasParameter()) {
        return false;  // 包含参数，不能折叠
    }

    if (HasSubquery()) {
        return false;  // 包含子查询，不能折叠
    }

    if (IsVolatile()) {
        return false;  // 易变函数（如random()），不能折叠
    }

    // 所有子表达式都是常量
    return AllChildrenAreFoldable();
}
```

**示例：**
```sql
-- 原始查询
SELECT * FROM t WHERE age > 18 + 7 AND price * 2 < 100

-- 常量折叠后
SELECT * FROM t WHERE age > 25 AND price < 50
```

### 19.2 公共子表达式消除（CSE）

识别并消除重复的子表达式：

```cpp
// src/optimizer/common_subexpression_elimination.cpp
class CommonSubexpressionElimination {
    // 表达式 -> 列引用的映射
    expression_map_t<unique_ptr<Expression>> expression_map;

public:
    unique_ptr<Expression> VisitReplace(Expression &expr) override {
        // 检查是否已经计算过此表达式
        auto entry = expression_map.find(&expr);
        if (entry != expression_map.end()) {
            // 找到重复表达式，返回列引用
            return entry->second->Copy();
        }

        // 新表达式，添加到映射
        if (IsComplexExpression(expr)) {
            auto colref = CreateColumnReference(expr);
            expression_map[&expr] = colref->Copy();
            return colref;
        }

        return nullptr;  // 不修改
    }

private:
    bool IsComplexExpression(Expression &expr) {
        // 只对复杂表达式做CSE（避免过度优化）
        return expr.type == ExpressionType::FUNCTION ||
               expr.type == ExpressionType::OPERATOR;
    }
};
```

**示例：**
```sql
-- 原始查询
SELECT price * 1.1, price * 1.1 * quantity
FROM orders

-- CSE后（概念上）
WITH temp AS (
    SELECT price * 1.1 AS price_with_tax, quantity
    FROM orders
)
SELECT price_with_tax, price_with_tax * quantity FROM temp
```

### 19.3 谓词简化

```cpp
// 简化逻辑表达式
unique_ptr<Expression> SimplifyConjunction(BoundConjunctionExpression &conj) {
    if (conj.type == ExpressionType::CONJUNCTION_AND) {
        // 移除恒真条件
        vector<unique_ptr<Expression>> new_children;
        for (auto &child : conj.children) {
            if (IsAlwaysTrue(*child)) {
                continue;  // 跳过恒真条件
            }
            if (IsAlwaysFalse(*child)) {
                // 整个AND为FALSE
                return make_unique<BoundConstantExpression>(Value::BOOLEAN(false));
            }
            new_children.push_back(std::move(child));
        }

        if (new_children.empty()) {
            // 所有条件都是TRUE
            return make_unique<BoundConstantExpression>(Value::BOOLEAN(true));
        }

        if (new_children.size() == 1) {
            // 只剩一个条件
            return std::move(new_children[0]);
        }

        conj.children = std::move(new_children);
        return nullptr;  // 不修改
    }

    if (conj.type == ExpressionType::CONJUNCTION_OR) {
        // 移除恒假条件
        vector<unique_ptr<Expression>> new_children;
        for (auto &child : conj.children) {
            if (IsAlwaysFalse(*child)) {
                continue;  // 跳过恒假条件
            }
            if (IsAlwaysTrue(*child)) {
                // 整个OR为TRUE
                return make_unique<BoundConstantExpression>(Value::BOOLEAN(true));
            }
            new_children.push_back(std::move(child));
        }

        if (new_children.empty()) {
            // 所有条件都是FALSE
            return make_unique<BoundConstantExpression>(Value::BOOLEAN(false));
        }

        if (new_children.size() == 1) {
            // 只剩一个条件
            return std::move(new_children[0]);
        }

        conj.children = std::move(new_children);
        return nullptr;
    }

    return nullptr;
}
```

**示例：**
```sql
-- 原始
WHERE age > 18 AND true AND department = 'IT'

-- 简化后
WHERE age > 18 AND department = 'IT'

-- 原始
WHERE age > 18 OR false OR name LIKE 'A%'

-- 简化后
WHERE age > 18 OR name LIKE 'A%'
```

### 19.4 范围合并

```cpp
// 合并重叠的范围条件
unique_ptr<Expression> MergeRangeFilters(vector<unique_ptr<Expression>> &filters) {
    // 收集所有涉及同一列的范围条件
    unordered_map<idx_t, vector<RangeFilter>> column_ranges;

    for (auto &filter : filters) {
        if (auto range = ExtractRangeFilter(*filter)) {
            column_ranges[range->column_index].push_back(*range);
        }
    }

    // 对每列合并范围
    for (auto &entry : column_ranges) {
        auto &ranges = entry.second;

        // 计算交集
        Value min_value = Value::MinValue(ranges[0].type);
        Value max_value = Value::MaxValue(ranges[0].type);

        for (auto &range : ranges) {
            if (range.min_value > min_value) {
                min_value = range.min_value;
            }
            if (range.max_value < max_value) {
                max_value = range.max_value;
            }
        }

        // 检查范围是否为空
        if (min_value > max_value) {
            // 矛盾条件，永远为FALSE
            return make_unique<BoundConstantExpression>(Value::BOOLEAN(false));
        }

        // 用合并后的范围替换原条件
        // ...
    }

    return nullptr;
}

struct RangeFilter {
    idx_t column_index;
    Value min_value;
    Value max_value;
    LogicalType type;
};
```

**示例：**
```sql
-- 原始
WHERE age >= 18 AND age > 20 AND age < 65

-- 合并后
WHERE age > 20 AND age < 65

-- 矛盾条件
WHERE age > 50 AND age < 30
-- 优化为：WHERE FALSE (永不满足)
```

### 19.5 IN列表优化

```cpp
// 优化IN列表
unique_ptr<Expression> OptimizeInList(BoundOperatorExpression &in_expr) {
    auto &list = in_expr.children;

    // 1. 移除重复值
    unordered_set<Value> unique_values;
    vector<unique_ptr<Expression>> new_list;

    for (auto &value_expr : list) {
        if (value_expr->IsFoldable()) {
            Value val = ExpressionExecutor::EvaluateScalar(*value_expr);
            if (unique_values.insert(val).second) {
                new_list.push_back(std::move(value_expr));
            }
        }
    }

    // 2. 如果列表为空
    if (new_list.empty()) {
        return make_unique<BoundConstantExpression>(Value::BOOLEAN(false));
    }

    // 3. 如果只有一个值，转换为等值比较
    if (new_list.size() == 1) {
        return make_unique<BoundComparisonExpression>(
            ExpressionType::COMPARE_EQUAL,
            in_expr.children[0]->Copy(),
            std::move(new_list[0])
        );
    }

    // 4. 如果列表很大，考虑转换为Hash Set查找
    if (new_list.size() > IN_LIST_HASH_THRESHOLD) {
        return ConvertToHashSetLookup(in_expr.children[0], new_list);
    }

    in_expr.children = std::move(new_list);
    return nullptr;
}

constexpr idx_t IN_LIST_HASH_THRESHOLD = 20;
```

**示例：**
```sql
-- 原始
WHERE id IN (1, 2, 3, 2, 1, 4)

-- 去重后
WHERE id IN (1, 2, 3, 4)

-- 如果只有一个值
WHERE id IN (42)
-- 转换为
WHERE id = 42
```

**实践任务：**
1. 阅读 `src/optimizer/expression_rewriter/` 下的各种重写器
2. 实现常量折叠优化
3. 实现简单的CSE（公共子表达式消除）
4. 测试优化前后的性能差异

---

**(继续Day 20-21...)**

