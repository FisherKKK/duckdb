# DuckDB 高级课程：查询优化器深度解析

> 本课程深入讲解DuckDB查询优化器的架构、优化规则、代价模型和Join Order算法。

---

## 课程概览

### 学习目标

- 理解查询优化器的整体架构
- 掌握表达式优化技术
- 学习逻辑优化规则
- 深入理解Join Order优化算法
- 掌握代价模型和基数估计
- 能够实现自定义优化规则

### 前置知识

- 熟悉C++基础
- 了解SQL查询执行流程
- 理解数据结构（树、图、动态规划）
- 了解关系代数基础

---

## 第一部分：优化器架构

### 1.1 优化器执行流程

DuckDB的查询优化器采用基于规则的优化（RBO）和基于代价的优化（CBO）相结合的方式。

```cpp
// src/optimizer/optimizer.cpp

// 优化器执行流程
void Optimizer::RunBuiltInOptimizers() {
    // 1. 表达式重写（不改变逻辑计划结构）
    RunOptimizer(OptimizerType::EXPRESSION_REWRITER, [&]() {
        rewriter.VisitOperator(*plan);
    });

    // 2. CTE内联
    RunOptimizer(OptimizerType::CTE_INLINING, [&]() {
        CTEInlining cte_inlining(*this);
        plan = cte_inlining.Optimize(std::move(plan));
    });

    // 3. SUM重写
    RunOptimizer(OptimizerType::SUM_REWRITER, [&]() {
        SumRewriterOptimizer optimizer(*this);
        optimizer.Optimize(plan);
    });

    // 4. Filter Pullup（将过滤器上提）
    RunOptimizer(OptimizerType::FILTER_PULLUP, [&]() {
        FilterPullup filter_pullup;
        plan = filter_pullup.Rewrite(std::move(plan));
    });

    // 5. Filter Pushdown（将过滤器下推）
    RunOptimizer(OptimizerType::FILTER_PUSHDOWN, [&]() {
        FilterPushdown filter_pushdown(*this);
        plan = filter_pushdown.Rewrite(std::move(plan));
    });

    // 6. 正则表达式范围优化
    RunOptimizer(OptimizerType::REGEX_RANGE, [&]() {
        RegexRangeFilter regex_opt;
        plan = regex_opt.Rewrite(std::move(plan));
    });

    // 7. IN子句重写
    RunOptimizer(OptimizerType::IN_CLAUSE, [&]() {
        InClauseRewriter ic_rewriter(context, *this);
        plan = ic_rewriter.Rewrite(std::move(plan));
    });

    // 8. Deliminator优化
    RunOptimizer(OptimizerType::DELIMINATOR, [&]() {
        Deliminator deliminator;
        plan = deliminator.Optimize(std::move(plan));
    });

    // 9. 空结果上提
    RunOptimizer(OptimizerType::EMPTY_RESULT_PULLUP, [&]() {
        EmptyResultPullup empty_result_pullup;
        plan = empty_result_pullup.Optimize(std::move(plan));
    });

    // 10. Window Self Join优化
    RunOptimizer(OptimizerType::WINDOW_SELF_JOIN, [&]() {
        WindowSelfJoinOptimizer window_self_join_optimizer(*this);
        plan = window_self_join_optimizer.Optimize(std::move(plan));
    });

    // 11. 去除重复分组
    RunOptimizer(OptimizerType::REMOVE_DUPLICATE_GROUPS, [&]() {
        RemoveDuplicateGroups remove_dup_groups;
        plan = remove_dup_groups.Optimize(std::move(plan));
    });

    // 12. Join消除
    RunOptimizer(OptimizerType::JOIN_ELIMINATION, [&]() {
        JoinElimination join_elimination(*this);
        plan = join_elimination.Optimize(std::move(plan));
    });

    // 13. 统计信息传播
    RunOptimizer(OptimizerType::STATISTICS_PROPAGATION, [&]() {
        StatisticsPropagator statistics_propagator(*this);
        plan = statistics_propagator.Optimize(std::move(plan));
    });

    // 14. 列裁剪
    RunOptimizer(OptimizerType::UNUSED_COLUMNS, [&]() {
        RemoveUnusedColumns unused_columns(true);
        plan = unused_columns.Optimize(std::move(plan));
    });

    // 15. 常量折叠
    RunOptimizer(OptimizerType::CONSTANT_FOLDING, [&]() {
        ExpressionHeuristics expression_heuristics(*this);
        plan = expression_heuristics.Optimize(std::move(plan));
    });

    // 16. CSE（公共子表达式消除）
    RunOptimizer(OptimizerType::COMMON_SUBEXPRESSIONS, [&]() {
        CommonSubExpressionOptimizer cse_optimizer;
        plan = cse_optimizer.Optimize(std::move(plan));
    });

    // 17. Limit下推
    RunOptimizer(OptimizerType::LIMIT_PUSHDOWN, [&]() {
        LimitPushdown limit_pushdown;
        plan = limit_pushdown.Optimize(std::move(plan));
    });

    // 18. Join Order优化（代价模型）
    RunOptimizer(OptimizerType::JOIN_ORDER, [&]() {
        JoinOrderOptimizer join_order_optimizer(context);
        plan = join_order_optimizer.Optimize(std::move(plan));
    });

    // 19. 列生命周期分析
    RunOptimizer(OptimizerType::COLUMN_LIFETIME, [&]() {
        ColumnLifetimeAnalyzer column_lifetime_analyzer;
        plan = column_lifetime_analyzer.Optimize(std::move(plan));
    });
}
```

### 1.2 优化器类型分类

```
优化规则分类：

1. 表达式优化
   ├── 常量折叠
   ├── 表达式简化
   ├── CSE消除
   └── 类型转换优化

2. 逻辑优化
   ├── Filter Pushdown/Pullup
   ├── Join Elimination
   ├── Limit Pushdown
   ├── CTE Inlining
   └── Projection Elimination

3. 物理优化
   ├── Join Order优化
   ├── Join算法选择
   ├── Aggregate优化
   └── 排序优化

4. 统计信息优化
   ├── 统计信息收集
   ├── 基数估计
   └── 代价计算
```

### 1.3 优化规则执行顺序的重要性

```cpp
// 示例：优化顺序的影响

// 初始查询：
SELECT department.name, COUNT(*)
FROM employee
JOIN department ON employee.dept_id = department.id
WHERE employee.salary > 50000
GROUP BY department.name;

// 错误的优化顺序：
// 1. 先执行Join Order → 可能产生较大的中间结果
// 2. 再执行Filter Pushdown → 无法利用过滤条件缩小Join输入

// 正确的优化顺序：
// 1. 先执行Filter Pushdown → 过滤条件下推到扫描
// 2. 再执行Join Order → 基于过滤后的基数估计选择最优Join顺序
```

---

## 第二部分：表达式优化

### 2.1 常量折叠（Constant Folding）

```cpp
// src/optimizer/rule/constant_folding.cpp

class ConstantFoldingRule : public Rule {
public:
    // 应用常量折叠
    unique_ptr<Expression> Apply(LogicalOperator &op, Expression &expr) override {
        // 示例：1 + 2 * 3
        // 规则：
        // 1. 先计算优先级高的操作符
        // 2. 2 * 3 = 6
        // 3. 1 + 6 = 7

        if (expr.type == ExpressionType::FUNCTION) {
            auto &func_expr = (FunctionExpression &)expr;

            // 检查所有参数是否都是常量
            bool all_const = true;
            for (auto &child : func_expr.children) {
                if (child->type != ExpressionType::VALUE_CONSTANT) {
                    all_const = false;
                    break;
                }
            }

            if (all_const) {
                // 执行函数并返回常量结果
                Value result = ExecuteFunction(func_expr);
                return make_unique<ConstantExpression>(result);
            }
        }

        return nullptr;
    }

private:
    Value ExecuteFunction(FunctionExpression &expr) {
        // 实际执行函数
        // 例如：1 + 2 → 执行加法 → 返回 3
        // 注意：这里需要处理各种函数类型
        return Value();
    }
};

// 示例优化
// 优化前：
SELECT salary * 1.1 + 1000 FROM employees;

// 优化后（如果salary是常量列）：
SELECT 11000 FROM employees;  -- 假设salary = 10000
```

### 2.2 表达式简化

```cpp
// src/optimizer/rule/arithmetic_simplification.cpp

class ArithmeticSimplificationRule : public Rule {
public:
    unique_ptr<Expression> Apply(LogicalOperator &op, Expression &expr) override {
        // 规则1：x + 0 → x
        if (expr.type == ExpressionType::OPERATOR_ADD) {
            auto &add_expr = (OperatorExpression &)expr;
            if (IsZero(add_expr.children[1])) {
                return std::move(add_expr.children[0]);
            }
        }

        // 规则2：x * 1 → x
        if (expr.type == ExpressionType::OPERATOR_MULTIPLY) {
            auto &mul_expr = (OperatorExpression &)expr;
            if (IsOne(mul_expr.children[1])) {
                return std::move(mul_expr.children[0]);
            }
        }

        // 规则3：x * 0 → 0
        if (expr.type == ExpressionType::OPERATOR_MULTIPLY) {
            auto &mul_expr = (OperatorExpression &)expr;
            if (IsZero(mul_expr.children[1])) {
                return make_unique<ConstantExpression>(Value::INTEGER(0));
            }
        }

        // 规则4：x / 1 → x
        if (expr.type == ExpressionType::OPERATOR_DIVIDE) {
            auto &div_expr = (OperatorExpression &)expr;
            if (IsOne(div_expr.children[1])) {
                return std::move(div_expr.children[0]);
            }
        }

        // 规则5：!!x → x
        if (expr.type == ExpressionType::OPERATOR_NOT) {
            auto &not_expr = (OperatorExpression &)expr;
            if (not_expr.children[0]->type == ExpressionType::OPERATOR_NOT) {
                auto &inner_not = (OperatorExpression &)*not_expr.children[0];
                return std::move(inner_not.children[0]);
            }
        }

        return nullptr;
    }
};

// 示例优化
// 优化前：
SELECT (salary + 0) * 1 FROM employees;

// 优化后：
SELECT salary FROM employees;
```

### 2.3 公共子表达式消除（CSE）

```cpp
// src/optimizer/cse_optimizer.cpp

class CommonSubExpressionOptimizer {
public:
    // 表达式等价性检查
    struct ExpressionHash {
        size_t operator()(const Expression *expr) const {
            // 基于表达式类型和子节点计算hash
            size_t hash = std::hash<int>()((int)expr->type);
            for (auto &child : expr->children) {
                hash ^= ExpressionHash()(child.get());
            }
            return hash;
        }
    };

    struct ExpressionEqual {
        bool operator()(const Expression *a, const Expression *b) const {
            if (a->type != b->type) {
                return false;
            }
            if (a->children.size() != b->children.size()) {
                return false;
            }
            for (size_t i = 0; i < a->children.size(); i++) {
                if (!ExpressionEqual()(a->children[i].get(),
                                       b->children[i].get())) {
                    return false;
                }
            }
            return true;
        }
    };

    unique_ptr<LogicalOperator> Optimize(unique_ptr<LogicalOperator> plan) {
        // 遍历逻辑计划，查找公共子表达式
        unordered_map<Expression*, idx_t, ExpressionHash, ExpressionEqual> expr_map;

        // 收集所有表达式
        vector<Expression*> all_expressions;
        CollectExpressions(*plan, all_expressions);

        // 找出重复的表达式
        for (auto &expr : all_expressions) {
            auto it = expr_map.find(expr);
            if (it != expr_map.end()) {
                // 发现公共子表达式！
                // 用引用替换重复表达式
                ReplaceExpression(expr, it->second);
            } else {
                expr_map[expr] = expr_map.size();
            }
        }

        return plan;
    }

private:
    void CollectExpressions(LogicalOperator &op,
                          vector<Expression*> &result) {
        // 收集算子中的所有表达式
        for (auto &expr : op.expressions) {
            result.push_back(expr.get());
            CollectExpressionsRecursive(*expr, result);
        }

        // 递归处理子节点
        for (auto &child : op.children) {
            CollectExpressions(*child, result);
        }
    }

    void CollectExpressionsRecursive(Expression &expr,
                                     vector<Expression*> &result) {
        for (auto &child : expr.children) {
            result.push_back(child.get());
            CollectExpressionsRecursive(*child, result);
        }
    }

    void ReplaceExpression(Expression *expr, idx_t replacement_idx) {
        // 用引用替换表达式
        // 实现细节...
    }
};

// 示例优化
// 优化前：
SELECT (salary + bonus) * 0.1, (salary + bonus) * 0.9
FROM employees;

// 优化后：
-- temp = salary + bonus
SELECT temp * 0.1, temp * 0.9
FROM employees;
```

---

## 第三部分：逻辑优化

### 3.1 Filter Pushdown（过滤下推）

```cpp
// src/optimizer/filter_pushdown.cpp

class FilterPushdown {
public:
    unique_ptr<LogicalOperator> Rewrite(unique_ptr<LogicalOperator> plan) {
        // 递归下推过滤器
        return RewriteOp(plan.get());
    }

private:
    unique_ptr<LogicalOperator> RewriteOp(LogicalOperator *op) {
        // 处理不同类型的算子
        switch (op->type) {
        case LogicalOperatorType::LOGICAL_FILTER:
            return RewriteFilter((LogicalFilter &)*op);
        case LogicalOperatorType::LOGICAL_PROJECTION:
            return RewriteProjection((LogicalProjection &)*op);
        case LogicalOperatorType::LOGICAL_JOIN:
            return RewriteJoin((LogicalJoin &)*op);
        case LogicalOperatorType::LOGICAL_AGGREGATE_AND_GROUP_BY:
            return RewriteAggregate((LogicalAggregate &)*op);
        case LogicalOperatorType::LOGICAL_CROSS_PRODUCT:
            return RewriteCrossProduct((LogicalCrossProduct &)*op);
        default:
            return op->Clone();
        }
    }

    unique_ptr<LogicalOperator> RewriteFilter(LogicalFilter &op) {
        // 递归处理子节点
        auto child = RewriteOp(op.children[0].get());

        // 获取过滤条件
        auto &filter_expr = op.expressions[0];

        // 尝试将过滤条件下推到子节点
        if (child->type == LogicalOperatorType::LOGICAL_GET) {
            // 可以下推到表扫描
            auto &get_op = (LogicalGet &)*child;
            get_op.filters.push_back(filter_expr->Copy());

            // 返回没有过滤器的GET
            return std::move(child);
        }

        // 如果不能下推，保留过滤器
        auto new_filter = make_unique<LogicalFilter>();
        new_filter->children.push_back(std::move(child));
        new_filter->expressions.push_back(std::move(filter_expr));
        return new_filter;
    }

    unique_ptr<LogicalOperator> RewriteJoin(LogicalJoin &op) {
        // 处理Join的过滤条件下推

        // 先处理子节点
        auto left = RewriteOp(op.children[0].get());
        auto right = RewriteOp(op.children[1].get());

        // 分离过滤条件
        vector<unique_ptr<Expression>> left_filters;
        vector<unique_ptr<Expression>> right_filters;
        vector<unique_ptr<Expression>> join_filters;

        SplitFilters(op.expressions, left_filters, right_filters, join_filters);

        // 创建带过滤条件的子节点
        if (!left_filters.empty()) {
            left = AddFilterToNode(std::move(left), left_filters);
        }

        if (!right_filters.empty()) {
            right = AddFilterToNode(std::move(right), right_filters);
        }

        // 创建Join节点
        auto new_join = op.Clone();
        new_join->children[0] = std::move(left);
        new_join->children[1] = std::move(right);
        new_join->expressions = std::move(join_filters);

        return new_join;
    }

    void SplitFilters(vector<unique_ptr<Expression>> &filters,
                     vector<unique_ptr<Expression>> &left_filters,
                     vector<unique_ptr<Expression>> &right_filters,
                     vector<unique_ptr<Expression>> &join_filters) {
        // 根据表达式中引用的列，将过滤条件分类
        // - 只引用左表列 → left_filters
        // - 只引用右表列 → right_filters
        // - 引用两表列 → join_filters
        for (auto &filter : filters) {
            auto referenced_columns = GetReferencedColumns(*filter);

            bool uses_left = UsesLeftTable(referenced_columns);
            bool uses_right = UsesRightTable(referenced_columns);

            if (uses_left && !uses_right) {
                left_filters.push_back(filter->Copy());
            } else if (!uses_left && uses_right) {
                right_filters.push_back(filter->Copy());
            } else {
                join_filters.push_back(filter->Copy());
            }
        }
    }
};

// 示例优化
// 查询：
SELECT *
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.amount > 1000 AND c.country = 'USA';

// 优化前计划：
┌─────────────────────────┐
│   Filter               │
│   (o.amount > 1000 AND │
│    c.country = 'USA')   │
└──────────┬──────────────┘
           ↓
┌─────────────────────────┐
│   Join                 │
│   o.id = c.id          │
└──────────┬──────────────┘
           ↓
┌──────┐ ┌──────┐
│orders│ │customers│
└──────┘ └──────┘

// 优化后计划：
┌─────────────────────────┐
│   Join                 │
│   o.id = c.id          │
└──────────┬──────────────┘
           ↓
     ┌─────┴─────┐
     ↓           ↓
┌─────────┐ ┌─────────────┐
│ Filter  │ │   Filter    │
│ o.amount│ │ c.country = │
│  > 1000 │ │   'USA'     │
└────┬────┘ └──────┬──────┘
     ↓             ↓
┌─────────┐  ┌─────────┐
│ orders  │  │customers│
└─────────┘  └─────────┘
```

### 3.2 Join Elimination（Join消除）

```cpp
// src/optimizer/join_elimination.cpp

class JoinElimination {
public:
    unique_ptr<LogicalOperator> Optimize(unique_ptr<LogicalOperator> plan) {
        EliminateJoins(*plan);
        return plan;
    }

private:
    void EliminateJoins(LogicalOperator &op) {
        // 递归处理子节点
        for (auto &child : op.children) {
            EliminateJoins(*child);
        }

        // 检查是否是Join节点
        if (op.type == LogicalOperatorType::LOGICAL_JOIN ||
            op.type == LogicalOperatorType::LOGICAL_CROSS_PRODUCT) {

            auto &join = (LogicalJoin &)op;

            // 检查Join条件
            if (CanEliminateJoin(join)) {
                // 消除Join
                EliminateJoin(join);
            }
        }
    }

    bool CanEliminateJoin(LogicalJoin &join) {
        // 规则1：Inner Join on Foreign Key = Primary Key
        // 如果Join条件是外键=主键，且没有选择外键表的列
        if (IsForeignKeyPrimaryKeyJoin(join)) {
            return !UsesTableColumns(join, /*foreign_key_table*/ 1);
        }

        // 规则2：恒真Join条件（如1=1）
        if (IsTrivialJoin(join)) {
            return true;
        }

        // 规则3：空表消除
        if (IsEmptyTable(join.children[1])) {
            return true;
        }

        return false;
    }

    bool IsForeignKeyPrimaryKeyJoin(LogicalJoin &join) {
        // 检查Join条件是否是外键-主键关系
        // 实现需要Catalog信息
        return false;
    }

    bool IsTrivialJoin(LogicalJoin &join) {
        // 检查Join条件是否是恒真（如1=1）
        if (join.conditions.empty()) {
            return false;
        }

        for (auto &cond : join.conditions) {
            if (!IsAlwaysTrue(*cond)) {
                return false;
            }
        }

        return true;
    }

    void EliminateJoin(LogicalJoin &join) {
        // 将Join替换为其左子节点
        // 假设左子节点是primary key表
        join.type = LogicalOperatorType::LOGICAL_GET;
        join.children = {std::move(join.children[0])};
        join.conditions.clear();
    }
};

// 示例优化
// 查询：
SELECT c.name
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.amount > 1000;

// 假设：orders.customer_id是外键，指向customers.id（主键）
// 且：查询只使用customers表的列

// 优化前：
┌─────────────────────┐
│ Projection         │
│ (c.name)           │
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│ Filter             │
│ (o.amount > 1000)  │
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│ Join               │
│ o.customer_id =    │
│ c.id               │
└──────────┬──────────┘
     ┌──────┴──────┐
     ↓             ↓
┌─────────┐  ┌─────────┐
│ orders  │  │customers│
└─────────┘  └─────────┘

// 优化后（消除customers表）：
┌─────────────────────┐
│ Projection         │
│ (o.customer_name)  │ ← 需要非规范化数据
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│ Filter             │
│ (o.amount > 1000)  │
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│ Scan               │
│ orders             │
└─────────────────────┘
```

### 3.3 Limit Pushdown

```cpp
// src/optimizer/limit_pushdown.cpp

class LimitPushdown {
public:
    unique_ptr<LogicalOperator> Optimize(unique_ptr<LogicalOperator> plan) {
        return RewriteOp(plan.get());
    }

private:
    unique_ptr<LogicalOperator> RewriteOp(LogicalOperator *op) {
        switch (op->type) {
        case LogicalOperatorType::LOGICAL_LIMIT:
            return RewriteLimit((LogicalLimit &)*op);
        case LogicalOperatorType::LOGICAL_ORDER_BY:
            return RewriteOrderBy((LogicalOrder &)*op);
        case LogicalOperatorType::LOGICAL_PROJECTION:
            return RewriteProjection((LogicalProjection &)*op);
        case LogicalOperatorType::LOGICAL_FILTER:
            return RewriteFilter((LogicalFilter &)*op);
        default:
            return op->Clone();
        }
    }

    unique_ptr<LogicalOperator> RewriteLimit(LogicalLimit &op) {
        // 递归处理子节点
        auto child = RewriteOp(op.children[0].get());

        // 尝试下推Limit
        if (CanPushDownLimit(*child)) {
            // 在子节点上添加Limit
            auto new_limit = make_unique<LogicalLimit>();
            new_limit->limit_val = op.limit_val;
            new_limit->offset_val = op.offset_val;
            new_limit->children.push_back(std::move(child));

            // 保留原Limit（处理offset）
            op.children[0] = std::move(new_limit);
            return op.Clone();
        }

        // 不能下推，保留原Limit
        op.children[0] = std::move(child);
        return op.Clone();
    }

    bool CanPushDownLimit(LogicalOperator &op) {
        // 规则：Limit可以跨越Projection、Filter、OrderBy
        // 但不能跨越Join、Aggregate、Distinct、SetOperation

        switch (op.type) {
        case LogicalOperatorType::LOGICAL_PROJECTION:
        case LogicalOperatorType::LOGICAL_FILTER:
        case LogicalOperatorType::LOGICAL_ORDER_BY:
            return true;
        default:
            return false;
        }
    }
};

// 示例优化
// 查询：
SELECT name, salary
FROM employees
WHERE salary > 50000
ORDER BY salary DESC
LIMIT 10;

// 优化前计划：
┌─────────────────────┐
│ Limit 10           │ ← 在最后才限制
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│ Order By           │ ← 对所有结果排序
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│ Filter             │
│ salary > 50000     │
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│ Scan employees     │
└─────────────────────┘

// 优化后计划：
┌─────────────────────┐
│ Limit 10           │
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│ Order By           │ ← 只对前10个最大值排序
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│ Limit 10           │ ← 提前下推，减少处理
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│ Filter             │
│ salary > 50000     │
└──────────┬──────────┘
           ↓
┌─────────────────────┐
│ Scan employees     │
└─────────────────────┘
```

---

## 第四部分：Join Order优化

### 4.1 Join Order问题

```sql
-- 示例查询
SELECT *
FROM orders o
JOIN customers c ON o.customer_id = c.id
JOIN products p ON o.product_id = p.id
JOIN suppliers s ON p.supplier_id = s.id
WHERE o.order_date > '2024-01-01';

-- 问题：有3种可能的Join顺序
-- 1. ((orders × customers) × products) × suppliers
-- 2. (orders × customers) × (products × suppliers)
-- 3. (orders × (customers × products)) × suppliers
-- ... 更多组合

-- 目标：选择中间结果最小的Join顺序
```

### 4.2 动态规划算法

```cpp
// src/optimizer/join_order/plan_enumerator.cpp

class PlanEnumerator {
public:
    // 使用动态规划枚举所有Join顺序
    void SolveJoinOrder() {
        auto &relations = query_graph_manager.relation_manager.GetRelations();

        // 初始化单关系计划
        InitLeafPlans();

        // 动态规划：逐步合并关系
        // dp[S] = 关系集合S的最优Join计划
        map<JoinRelationSet, unique_ptr<JoinNode>> dp;

        // 初始状态：每个关系单独作为一个集合
        for (auto &rel : relations) {
            auto single_set = JoinRelationSet::Singleton(rel->set.index);
            dp[single_set] = CreateLeafPlan(rel);
        }

        // 逐步合并：从大小2到n
        for (size_t size = 2; size <= relations.size(); size++) {
            // 枚举所有大小为size的子集
            for (auto &entry : dp) {
                auto &left_set = entry.first;

                // 跳过不符合大小的集合
                if (left_set.count != size) {
                    continue;
                }

                // 尝试将left_set与其他集合合并
                for (auto &other_entry : dp) {
                    auto &right_set = other_entry.first;

                    // 确保不重叠
                    if (!left_set.IsDisjoint(right_set)) {
                        continue;
                    }

                    // 合并集合
                    auto combined_set = left_set.Union(right_set);

                    // 计算Join代价
                    auto join_node = CreateJoinNode(
                        dp[left_set],
                        dp[right_set],
                        combined_set
                    );

                    // 更新最优计划
                    if (dp.find(combined_set) == dp.end() ||
                        join_node->cost < dp[combined_set]->cost) {
                        dp[combined_set] = std::move(join_node);
                    }
                }
            }
        }

        // 最终结果：包含所有关系的集合
        auto final_set = JoinRelationSet::All(relations.size());
        best_plan = std::move(dp[final_set]);
    }

private:
    unique_ptr<JoinNode> CreateJoinNode(
        unique_ptr<JoinNode> &left,
        unique_ptr<JoinNode> &right,
        JoinRelationSet &combined_set) {

        auto join_node = make_unique<JoinNode>();

        // 1. 估计Join基数
        double join_cardinality = cost_model.EstimateCardinality(
            *left, *right
        );

        // 2. 计算Join代价
        double cost = cost_model.ComputeCost(
            *left, *right, join_cardinality
        );

        // 3. 设置Join信息
        join_node->left = std::move(left);
        join_node->right = std::move(right);
        join_node->set = combined_set.Copy();
        join_node->cardinality = join_cardinality;
        join_node->cost = cost;

        return join_node;
    }

    CostModel &cost_model;
    map<JoinRelationSet, unique_ptr<JoinNode>> plans;
    unique_ptr<JoinNode> best_plan;
};
```

### 4.3 代价模型

```cpp
// src/optimizer/join_order/cost_model.cpp

class CostModel {
public:
    // 计算Join代价
    double ComputeCost(JoinNode &left, JoinNode &right,
                       double join_cardinality) {
        // 代价 = I/O代价 + CPU代价

        // 1. I/O代价：读取右表的代价
        double io_cost = right.cardinality;

        // 2. CPU代价：Join比较代价
        double cpu_cost = left.cardinality * right.cardinality;

        // 3. 如果右表已排序，使用Sort-Merge Join（更便宜）
        if (right.HasSortedOrder()) {
            cpu_cost = left.cardinality + right.cardinality;
        }

        // 4. 如果可以建立哈希表，使用Hash Join
        // 哈希表建立代价 = 右表基数
        // 探针代价 = 左表基数
        double hash_cost = right.cardinality + left.cardinality;

        // 选择最小的代价
        return io_cost + std::min(cpu_cost, hash_cost);
    }

    // 估计Join基数
    double EstimateCardinality(JoinNode &left, JoinNode &right) {
        // 简单估计：left.cardinality * right.cardinality
        // 如果有Join条件，使用选择性

        double selectivity = 1.0;

        // 从Join条件估计选择性
        // 假设等值Join的选择性 = 1 / max(distinct_left, distinct_right)
        if (HasEqualityCondition(left, right)) {
            double distinct_left = GetDistinctCount(left);
            double distinct_right = GetDistinctCount(right);
            selectivity = 1.0 / std::max(distinct_left, distinct_right);
        }

        return left.cardinality * right.cardinality * selectivity;
    }

private:
    bool HasEqualityCondition(JoinNode &left, JoinNode &right) {
        // 检查是否有等值Join条件
        // 实现细节...
        return true;
    }

    double GetDistinctCount(JoinNode &node) {
        // 获取不同值的数量
        // 可以从统计信息获取
        return node.cardinality * 0.1;  // 默认假设10%不同
    }
};
```

### 4.4 Query Graph

```cpp
// src/optimizer/join_order/query_graph.cpp

class QueryGraph {
public:
    // Query Graph节点：表示一个关系（表）
    struct RelationNode {
        idx_t table_index;
        string table_name;
        double cardinality;
        vector<idx_t> filter_ids;  // 该表上的过滤条件
    };

    // Query Graph边：表示Join条件
    struct Edge {
        idx_t left_relation;
        idx_t right_relation;
        vector<unique_ptr<Expression>> conditions;
        double selectivity;  // Join选择性
    };

    // 构建Query Graph
    void BuildGraph(LogicalOperator &plan) {
        // 遍历逻辑计划，提取关系和Join条件
        ExtractRelationsAndJoins(plan);

        // 计算每条边的选择性
        for (auto &edge : edges) {
            edge.selectivity = ComputeSelectivity(edge);
        }
    }

    // 获取邻居节点（用于Join Order枚举）
    vector<idx_t> GetNeighbors(idx_t relation_idx) {
        vector<idx_t> neighbors;
        for (auto &edge : edges) {
            if (edge.left_relation == relation_idx) {
                neighbors.push_back(edge.right_relation);
            } else if (edge.right_relation == relation_idx) {
                neighbors.push_back(edge.left_relation);
            }
        }
        return neighbors;
    }

private:
    vector<RelationNode> relations;
    vector<Edge> edges;
};
```

---

## 第五部分：统计信息与基数估计

### 5.1 统计信息收集

```cpp
// src/optimizer/statistics_propagator.cpp

class StatisticsPropagator {
public:
    unique_ptr<LogicalOperator> Optimize(unique_ptr<LogicalOperator> plan) {
        // 遍历逻辑计划，收集和传播统计信息
        PropagateStatistics(*plan);
        return plan;
    }

private:
    void PropagateStatistics(LogicalOperator &op) {
        // 从叶子节点向上传播
        for (auto &child : op.children) {
            PropagateStatistics(*child);
        }

        // 为当前算子计算统计信息
        switch (op.type) {
        case LogicalOperatorType::LOGICAL_GET:
            ComputeGetStatistics((LogicalGet &)op);
            break;
        case LogicalOperatorType::LOGICAL_FILTER:
            ComputeFilterStatistics((LogicalFilter &)op);
            break;
        case LogicalOperatorType::LOGICAL_JOIN:
            ComputeJoinStatistics((LogicalJoin &)op);
            break;
        case LogicalOperatorType::LOGICAL_AGGREGATE_AND_GROUP_BY:
            ComputeAggregateStatistics((LogicalAggregate &)op);
            break;
        default:
            break;
        }
    }

    void ComputeGetStatistics(LogicalGet &op) {
        // 从表统计信息获取基数
        auto &table_stats = GetTableStatistics(op.table);

        op.estimated_cardinality = table_stats.row_count;

        // 获取每列的统计信息
        for (auto &col : op.columns) {
            ColumnStats col_stats;
            col_stats.min = table_stats.GetMin(col);
            col_stats.max = table_stats.GetMax(col);
            col_stats.distinct_count = table_stats.GetDistinctCount(col);
            op.column_stats[col] = col_stats;
        }
    }

    void ComputeFilterStatistics(LogicalFilter &op) {
        auto &child_stats = op.children[0]->estimated_cardinality;

        // 计算过滤条件的选择性
        double selectivity = EstimateSelectivity(op.expressions[0]);

        op.estimated_cardinality = child_stats * selectivity;
    }

    void ComputeJoinStatistics(LogicalJoin &op) {
        auto &left_stats = op.children[0]->estimated_cardinality;
        auto &right_stats = op.children[1]->estimated_cardinality;

        // 基本估计：笛卡尔积
        double join_card = left_stats * right_stats;

        // 如果有Join条件，应用选择性
        if (!op.conditions.empty()) {
            double selectivity = EstimateJoinSelectivity(op);
            join_card *= selectivity;
        }

        op.estimated_cardinality = join_card;
    }

    void ComputeAggregateStatistics(LogicalAggregate &op) {
        // Group By的基数估计
        // 估计：MIN(表基数, 所有Group By列的distinct数之积)

        double max_card = op.children[0]->estimated_cardinality;

        double product_distinct = 1.0;
        for (auto &group_col : op.groups) {
            auto distinct = GetDistinctCount(*op.children[0], group_col);
            product_distinct *= distinct;
        }

        op.estimated_cardinality = std::min(max_card, product_distinct);
    }

    double EstimateSelectivity(Expression &expr) {
        // 根据表达式类型估计选择性

        switch (expr.type) {
        case ExpressionType::COMPARE_EQUAL:
            // 等值条件：假设1 / distinct_count
            return 1.0 / EstimateDistinctCount(expr);

        case ExpressionType::COMPARE_GREATERTHAN:
        case ExpressionType::COMPARE_LESSTHAN:
            // 范围条件：假设0.3（经验值）
            return 0.3;

        case ExpressionType::CONJUNCTION_AND:
            // AND：假设独立，相乘
            return EstimateConjunctionSelectivity((ConjunctionExpression &)expr);

        case ExpressionType::CONJUNCTION_OR:
            // OR：假设独立
            return EstimateDisjunctionSelectivity(expr);

        default:
            return 0.1;  // 默认选择性
        }
    }

    double EstimateConjunctionSelectivity(ConjunctionExpression &expr) {
        double selectivity = 1.0;
        for (auto &child : expr.children) {
            selectivity *= EstimateSelectivity(*child);
        }
        return selectivity;
    }
};
```

### 5.2 HyperLogLog基数估计

```cpp
// HyperLogLog算法实现

class HyperLogLog {
public:
    // 使用HyperLogLog估计不同值数量
    static idx_t EstimateDistinct(const vector<Value> &values) {
        // HyperLogLog参数
        constexpr uint8_t PRECISION = 12;  // 使用2^12 = 4096个寄存器
        constexpr uint32_t NUM_REGISTERS = 1 << PRECISION;

        // 初始化寄存器
        vector<uint8_t> registers(NUM_REGISTERS, 0);

        // 计算每个值的hash
        for (const auto &value : values) {
            uint64_t hash = ComputeHash(value);

            // 提取寄存器索引（前PRECISION位）
            uint32_t register_idx = hash & (NUM_REGISTERS - 1);

            // 计算前导零数量（剩余位）
            uint8_t leading_zeros = CountLeadingZeros(hash >> PRECISION);

            // 更新寄存器：保留最大值
            registers[register_idx] = std::max(registers[register_idx],
                                                 leading_zeros);
        }

        // 计算基数估计
        double sum = 0.0;
        for (uint8_t reg : registers) {
            sum += pow(2.0, -reg);
        }

        double alpha = 0.7213 / (1.0 + 1.079 / NUM_REGISTERS);
        double estimate = alpha * NUM_REGISTERS * NUM_REGISTERS / sum;

        return (idx_t)estimate;
    }

private:
    static uint64_t ComputeHash(const Value &value) {
        // 实际实现应该使用好的哈希函数
        // 这里简化
        return std::hash<string>()(value.ToString());
    }

    static uint8_t CountLeadingZeros(uint64_t value) {
        if (value == 0) {
            return 64;  // 全部为0
        }

        uint8_t count = 0;
        while ((value & 0x8000000000000000ULL) == 0) {
            count++;
            value <<= 1;
        }
        return count;
    }
};

// 使用示例
vector<Value> values = {/* 大量值 */};
idx_t distinct_estimate = HyperLogLog::EstimateDistinct(values);
```

---

## 第六部分：实践项目

### 项目1：实现简单的Filter Pushdown

```cpp
// simple_filter_pushdown.hpp

class SimpleFilterPushdown {
public:
    struct LogicalOperator {
        enum Type { SCAN, FILTER, JOIN, PROJECT };

        Type type;
        vector<unique_ptr<LogicalOperator>> children;
        vector<string> columns;
        string filter_condition;  // 如果是FILTER
    };

    static unique_ptr<LogicalOperator> Optimize(unique_ptr<LogicalOperator> plan) {
        if (!plan || plan->type != LogicalOperator::FILTER) {
            return plan;
        }

        auto &filter_op = *plan;
        auto filter_cond = filter_op.filter_condition;

        // 尝试下推到子节点
        auto &child = filter_op.children[0];

        if (child->type == LogicalOperator::SCAN) {
            // 可以下推到扫描
            child->filter_condition = filter_cond;
            // 返回没有过滤器的扫描
            return std::move(child);
        }

        if (child->type == LogicalOperator::JOIN) {
            // 检查过滤条件是否只引用左表或右表
            auto left_cols = GetColumns(*child->children[0]);
            auto right_cols = GetColumns(*child->children[1]);

            auto used_cols = GetUsedColumns(filter_cond);

            bool only_left = IsSubset(used_cols, left_cols);
            bool only_right = IsSubset(used_cols, right_cols);

            if (only_left || only_right) {
                // 下推到对应的一侧
                size_t target_idx = only_left ? 0 : 1;
                auto new_filter = make_unique<LogicalOperator>();
                new_filter->type = LogicalOperator::FILTER;
                new_filter->filter_condition = filter_cond;
                new_filter->children.push_back(std::move(child->children[target_idx]));

                child->children[target_idx] = std::move(new_filter);
                return plan;
            }
        }

        return plan;
    }
};
```

### 项目2：实现Join Order优化器

```cpp
// join_order_optimizer.hpp

class JoinOrderOptimizer {
public:
    struct TableInfo {
        string name;
        double cardinality;
        vector<string> columns;
    };

    struct JoinInfo {
        size_t left_table;
        size_t right_table;
        string condition;
        double selectivity;
    };

    struct JoinPlan {
        vector<size_t> table_order;
        double cost;
    };

    // 使用动态规划找最优Join顺序
    static JoinPlan Optimize(const vector<TableInfo> &tables,
                           const vector<JoinInfo> &joins) {

        // dp[mask] = 使用的表集合为mask时的最优计划
        map<size_t, JoinPlan> dp;

        // 初始化：单表计划
        for (size_t i = 0; i < tables.size(); i++) {
            size_t mask = 1ULL << i;
            dp[mask].table_order = {i};
            dp[mask].cost = 0.0;
        }

        // 动态规划：逐步合并
        for (size_t mask = 1; mask < (1ULL << tables.size()); mask++) {
            // 跳过不可达的mask
            if (dp.find(mask) == dp.end()) {
                continue;
            }

            // 尝试添加一个新表
            for (size_t i = 0; i < tables.size(); i++) {
                if (mask & (1ULL << i)) {
                    continue;  // 已经在mask中
                }

                size_t new_mask = mask | (1ULL << i);

                // 计算新代价
                double new_cost = dp[mask].cost +
                                ComputeJoinCost(mask, i, tables, joins);

                // 更新最优解
                if (dp.find(new_mask) == dp.end() ||
                    new_cost < dp[new_mask].cost) {
                    dp[new_mask].table_order = dp[mask].table_order;
                    dp[new_mask].table_order.push_back(i);
                    dp[new_mask].cost = new_cost;
                }
            }
        }

        // 返回包含所有表的计划
        size_t full_mask = (1ULL << tables.size()) - 1;
        return dp[full_mask];
    }

private:
    static double ComputeJoinCost(size_t left_mask, size_t right_table,
                                 const vector<TableInfo> &tables,
                                 const vector<JoinInfo> &joins) {
        // 计算左侧集合的基数
        double left_card = 1.0;
        for (size_t i = 0; i < tables.size(); i++) {
            if (left_mask & (1ULL << i)) {
                left_card *= tables[i].cardinality;
            }
        }

        // 右表基数
        double right_card = tables[right_table].cardinality;

        // 查找Join条件和选择性
        double selectivity = 0.1;  // 默认选择性
        for (const auto &join : joins) {
            bool left_in_mask = (left_mask & (1ULL << join.left_table)) != 0;
            if (left_in_mask && join.right_table == right_table) {
                selectivity = join.selectivity;
                break;
            }
        }

        // Hash Join代价：构建哈希表 + 探针
        return right_card + left_card * selectivity;
    }
};

// 使用示例
vector<TableInfo> tables = {
    {"orders", 1000000, {}},
    {"customers", 100000, {}},
    {"products", 10000, {}}
};

vector<JoinInfo> joins = {
    {0, 1, "orders.customer_id = customers.id", 0.01},
    {0, 2, "orders.product_id = products.id", 0.1}
};

auto plan = JoinOrderOptimizer::Optimize(tables, joins);
cout << "Optimal join order: ";
for (size_t idx : plan.table_order) {
    cout << tables[idx].name << " ";
}
cout << "\nCost: " << plan.cost << endl;
```

---

## 学习总结

### 优化规则对比

| 优化规则 | 类型 | 效果 | 适用场景 |
|---------|------|------|----------|
| Filter Pushdown | 逻辑 | 2-100x | 所有查询 |
| Join Elimination | 逻辑 | 2-10x | 外键Join |
| Limit Pushdown | 逻辑 | 10-1000x | Top-N查询 |
| CSE | 表达式 | 1.1-2x | 重复表达式 |
| Constant Folding | 表达式 | 1.1-5x | 常量表达式 |
| Join Order | 物理 | 2-100x | 多表Join |

### 优化器设计原则

1. **规则顺序很重要**：先做不变换结构的优化，后做改变结构的优化
2. **迭代应用**：某些规则需要多次应用才能达到最优
3. **代价估计**：统计信息质量直接影响优化效果
4. **避免过度优化**：优化本身也有开销，需要权衡

### 推荐资源

**论文：**
- "Access Path Selection in a Relational Database Management System"
- "Query Optimization in Database Systems"
- "Dynamic Programming Strikes Back"

**代码：**
- DuckDB: `src/optimizer/`
- PostgreSQL: `src/backend/optimizer/`
- MySQL: `sql/sql_optimizer.cc`

**书籍：**
- "Database System Concepts" - Chapter 15: Query Optimization
- "Practical Issues in Database Management"

---

**最后更新：2026-01-23**
