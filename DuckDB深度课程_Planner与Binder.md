# DuckDB 深度课程：Planner与Binder

> 本课程深入讲解DuckDB如何将SQL查询转换为逻辑执行计划，包括解析、绑定、符号解析、类型检查等核心流程。

---

## 课程概览

### 学习目标

- 理解SQL查询的完整处理流程
- 掌握Binder的符号解析机制
- 学习LogicalOperator的层次结构
- 理解子查询和CTE的处理
- 掌握聚合和GROUP BY的绑定逻辑
- 能够实现自定义SQL语句绑定

### 前置知识

- SQL语法基础
- 编译原理（抽象语法树、符号表）
- C++模板和继承
- 关系代数基础

---

## 第一部分：SQL处理流程总览

### 1.1 完整查询处理流程

```
┌─────────────────────────────────────────────────────────┐
│         DuckDB 查询处理完整流程                       │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. SQL字符串                                        │
│     "SELECT * FROM users WHERE age > 18"               │
│          ↓                                             │
│  2. Parser（解析器）                                  │
│     libpg_query → PostgreSQL AST                       │
│          ↓                                             │
│  3. Transformer（转换器）                              │
│     PostgreSQL AST → DuckDB SQLStatement               │
│          ↓                                             │
│  4. Binder（绑定器）                                  │
│     符号解析、类型检查 → BoundStatement                │
│          ↓                                             │
│  5. Planner（规划器）                                  │
│     BoundStatement → LogicalOperator Tree              │
│          ↓                                             │
│  6. Optimizer（优化器）                                │
│     LogicalOperator → 优化的LogicalOperator           │
│          ↓                                             │
│  7. Physical Planner（物理规划器）                        │
│     LogicalOperator → PhysicalOperator Tree            │
│          ↓                                             │
│  8. Executor（执行器）                                  │
│     执行查询 → 结果集                                  │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 1.2 各阶段详细职责

#### Parser阶段

```cpp
// src/parser/parser.cpp

class Parser {
public:
    // 解析SQL字符串为AST
    vector<unique_ptr<SQLStatement>> ParseQuery(const string &query) {
        // 1. 调用PostgreSQL的libpg_query
        auto pg_result = duckdb_libpgquery::parse_sql(query.c_str());

        // 2. 检查解析错误
        if (pg_result->parse_tree.empty()) {
            // 有错误
            throw ParserException(pg_result->error_message);
        }

        // 3. 转换为DuckDB的SQLStatement
        vector<unique_ptr<SQLStatement>> result;
        Transformer transformer;
        for (auto &pg_node : pg_result->parse_tree) {
            result.push_back(
                transformer.TransformParseTree(pg_node)
            );
        }

        return result;
    }
};
```

#### Binder阶段

```cpp
// src/planner/binder.cpp

class Binder {
public:
    // 绑定SQLStatement为BoundStatement
    unique_ptr<BoundStatement> Bind(SQLStatement &statement) {
        switch (statement.type) {
        case StatementType::SELECT_STATEMENT:
            return BindSelect((SelectStatement &)statement);

        case StatementType::INSERT_STATEMENT:
            return BindInsert((InsertStatement &)statement);

        case StatementType::DELETE_STATEMENT:
            return BindDelete((DeleteStatement &)statement);

        case StatementType::UPDATE_STATEMENT:
            return BindUpdate((UpdateStatement &)statement);

        case StatementType::CREATE_STATEMENT:
            return BindCreate((CreateStatement &)statement);

        default:
            throw InternalException("Unsupported statement type");
        }
    }

private:
    // 绑定上下文
    BindContext bind_context;

    // CTE（Common Table Expression）绑定
    vector<unique_ptr<CommonTableExpression>> cte_bindings;
};
```

#### Planner阶段

```cpp
// src/planner/planner.cpp

class Planner {
public:
    // 创建逻辑计划
    unique_ptr<LogicalOperator> CreatePlan(
        BoundStatement &statement) {

        switch (statement.type) {
        case StatementType::SELECT_STATEMENT:
            return PlanSelect((BoundSelectStatement &)statement);

        case StatementType::INSERT_STATEMENT:
            return PlanInsert((BoundInsertStatement &)statement);

        case StatementType::DELETE_STATEMENT:
            return PlanDelete((BoundDeleteStatement &)statement);

        // ... 其他语句类型
        default:
            throw InternalException("Unsupported statement type");
        }
    }
};
```

---

## 第二部分：Binder详解

### 2.1 BindContext结构

```cpp
// src/include/duckdb/planner/binder_context.hpp

class BindContext {
public:
    // 绑定列表（作用域栈）
    vector<unique_ptr<Binding>> bindings_list;

    // USING列（JOIN ... USING ...）
    case_insensitive_map_t<reference_set_t<UsingColumnSet>>
        using_columns;

    // CTE绑定
    vector<unique_ptr<CTEBinding>> cte_bindings;

    // 父上下文（用于子查询）
    BindContext *parent;

    // 别名
    case_insensitive_map_t<string> aliases;

    // ==================== 绑定操作 ====================

    // 添加表绑定
    void AddBinding(unique_ptr<Binding> binding) {
        bindings_list.push_back(std::move(binding));
    }

    // 添加CTE绑定
    void AddCTEBinding(unique_ptr<CTEBinding> cte_binding) {
        cte_bindings.push_back(std::move(cte_binding));
    }

    // ==================== 查找操作 ====================

    // 查找表绑定
    Binding *GetBinding(const string &name) {
        // 1. 首先查找CTE
        for (auto &cte : cte_bindings) {
            if (cte->alias == name) {
                return cte.get();
            }
        }

        // 2. 然后查找表别名
        for (auto &binding : bindings_list) {
            if (binding->alias == name) {
                return binding.get();
            }
        }

        // 3. 最后查找父上下文
        if (parent) {
            return parent->GetBinding(name);
        }

        return nullptr;
    }

    // 查找列绑定
    BindResult GetColumnBinding(const string &column_name,
                                const string &table_name) {
        // 1. 检查USING列
        auto using_entry = using_columns.find(column_name);
        if (using_entry != using_columns.end()) {
            return BindResult(using_entry->second);
        }

        // 2. 在当前上下文查找
        for (auto &binding : bindings_list) {
            if (table_name.empty() || binding->alias == table_name) {
                auto column_idx = binding->GetColumnIndex(column_name);
                if (column_idx != DConstants::INVALID_INDEX) {
                    return BindResult(binding.get(), column_idx);
                }
            }
        }

        // 3. 在父上下文查找（相关子查询）
        if (parent) {
            return parent->GetColumnBinding(column_name, table_name);
        }

        // 4. 未找到
        return BindResult(BindResultType::BINDING_FAILED);
    }
};
```

### 2.2 Binding结构

```cpp
// src/include/duckdb/planner/expression_binder/binding.hpp

class Binding {
public:
    // 绑定别名
    BindingAlias alias;

    // 列名列表
    vector<string> names;

    // 列类型列表
    vector<LogicalType> types;

    // 绑定索引
    idx_t index;

    Binding(BindingAlias alias, vector<string> names,
             vector<LogicalType> types, idx_t index)
        : alias(alias), names(std::move(names)),
          types(std::move(types)), index(index) {}

    virtual ~Binding() = default;

    // 获取列索引
    virtual idx_t GetColumnIndex(const string &name) const {
        for (idx_t i = 0; i < names.size(); i++) {
            if (names[i] == name) {
                return i;
            }
        }
        return DConstants::INVALID_INDEX;
    }

    // 是否为CTE绑定
    virtual bool IsCTEBinding() const {
        return false;
    }
};

// 表绑定
class TableBinding : public Binding {
public:
    TableBinding(const string &alias, TableCatalogEntry &table)
        : Binding(BindingAlias(alias),
                  table.GetColumns(), table.GetTypes(), 0) {}
};

// CTE绑定
class CTEBinding : public Binding {
public:
    CTEBinding(const string &alias, BoundQueryNode &cte)
        : Binding(BindingAlias(alias),
                  cte.names, cte.types, 0) {}

    bool IsCTEBinding() const override {
        return true;
    }
};
```

### 2.3 符号解析流程

```
┌─────────────────────────────────────────────────────────┐
│              列名符号解析流程                           │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  示例：SELECT t1.name, t2.age                         │
│         FROM users t1                                    │
│         JOIN orders t2 ON t1.id = t2.user_id            │
│                                                         │
│  解析 "t1.name"：                                      │
│  ┌─────────────────────────────────────────────┐          │
│  │ 1. 检查列名 = "name"                    │          │
│  │ 2. 检查表限定符 = "t1"                 │          │
│  │ 3. 在BindContext中查找表"t1"           │          │
│  │ 4. 在t1的Binding中查找"name"列          │          │
│  │ 5. 返回Binding + 列索引                 │          │
│  └─────────────────────────────────────────────┘          │
│                                                         │
│  解析 "t2.age"：                                       │
│  ┌─────────────────────────────────────────────┐          │
│  │ 1. 检查列名 = "age"                     │          │
│  │ 2. 检查表限定符 = "t2"                 │          │
│  │ 3. 在BindContext中查找表"t2"           │          │
│  │ 4. 在t2的Binding中查找"age"列          │          │
│  │ 5. 返回Binding + 列索引                 │          │
│  └─────────────────────────────────────────────┘          │
│                                                         │
│  歧义列处理：                                          │
│  示例：SELECT id FROM t1 JOIN t2 ON t1.id = t2.id      │
│                                                         │
│  解析"id"：                                            │
│  ┌─────────────────────────────────────────────┐          │
│  │ 1. 未指定表限定符                       │          │
│  │ 2. 查找列名"id"                        │          │
│  │ 3. 发现t1和t2都有"id"列                  │          │
│  │ 4. 抛出歧义错误：                         │          │
│  │    "Ambiguous column reference 'id'"      │          │
│  └─────────────────────────────────────────────┘          │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 第三部分：表达式绑定

### 3.1 ExpressionBinder

```cpp
// src/planner/expression_binder/expression_binder.cpp

class ExpressionBinder {
public:
    // 绑定表达式
    unique_ptr<BoundExpression> BindExpression(
        ParsedExpression &expr) {

        switch (expr.type) {
        case ExpressionType::COLUMN_REF:
            return BindColumn((ColumnRefExpression &)expr);

        case ExpressionType::FUNCTION:
            return BindFunction((FunctionExpression &)expr);

        case ExpressionType::OPERATOR:
            return BindOperator((OperatorExpression &)expr);

        case ExpressionType::SUBQUERY:
            return BindSubquery((SubqueryExpression &)expr);

        case ExpressionType::STAR:
            return BindStar((StarExpression &)expr);

        case ExpressionType::CONSTANT:
            return BindConstant((ConstantExpression &)expr);

        default:
            throw InternalException("Unsupported expression type");
        }
    }

protected:
    // 绑定上下文
    Binder &binder;

    // 当前聚合深度
    idx_t aggregate_depth;

    // ==================== 列绑定 ====================

    unique_ptr<BoundExpression> BindColumn(ColumnRefExpression &expr) {
        // 1. 获取列名
        string column_name = expr.GetColumnName();
        string table_name = expr.GetTableName();

        // 2. 在BindContext中查找
        auto binding_result = bind_context.GetColumnBinding(
            column_name, table_name
        );

        if (binding_result.result_type == BindResultType::BINDING_FAILED) {
            // 3. 未找到列
            throw BinderException("Column '%s' not found",
                                 column_name);
        }

        // 4. 创建BoundColumnRefExpression
        return make_unique<BoundColumnRefExpression>(
            binding_result.types[binding_result.index],
            binding_result.names[binding_result.index],
            binding_result.binding,
            binding_result.index
        );
    }

    // ==================== 函数绑定 ====================

    unique_ptr<BoundExpression> BindFunction(FunctionExpression &expr) {
        // 1. 获取函数名
        string function_name = expr.function_name;

        // 2. 绑定函数参数
        vector<unique_ptr<BoundExpression>> children;
        for (auto &child : expr.children) {
            children.push_back(BindExpression(*child));
        }

        // 3. 查找函数
        auto function = Catalog::GetFunction(context, function_name,
                                           children);

        if (!function) {
            throw BinderException("Function '%s' not found",
                                 function_name);
        }

        // 4. 函数类型检查
        function->CheckArguments(children);

        // 5. 创建BoundFunctionExpression
        return make_unique<BoundFunctionExpression>(
            function->return_type,
            std::move(function),
            std::move(children)
        );
    }

    // ==================== 操作符绑定 ====================

    unique_ptr<BoundExpression> BindOperator(OperatorExpression &expr) {
        // 1. 绑定操作数
        vector<unique_ptr<BoundExpression>> children;
        for (auto &child : expr.children) {
            children.push_back(BindExpression(*child));
        }

        // 2. 类型检查和类型提升
        LogicalType result_type = ResolveOperatorType(expr.type, children);

        // 3. 创建BoundOperatorExpression
        return make_unique<BoundOperatorExpression>(
            expr.type,
            result_type,
            std::move(children)
        );
    }

    // ==================== 子查询绑定 ====================

    unique_ptr<BoundExpression> BindSubquery(SubqueryExpression &expr) {
        // 1. 创建子查询Binder
        auto sub_binder = make_unique<Binder>(context,
                                              bind_context.parent);

        // 2. 绑定子查询
        auto sub_statement = sub_binder->Bind(*expr.subquery);

        // 3. 提取子查询信息
        if (sub_statement->types.size() != 1) {
            throw BinderException(
                "Subquery must return exactly one column"
            );
        }

        // 4. 创建BoundSubqueryExpression
        return make_unique<BoundSubqueryExpression>(
            sub_statement->types[0],
            std::move(sub_statement->plan),
            SubqueryType::SCALAR
        );
    }
};
```

### 3.2 类型推导和转换

```cpp
// 类型推导

class TypeResolver {
public:
    // 推导表达式类型
    static LogicalType ResolveExpressionType(ParsedExpression &expr) {
        switch (expr.type) {
        case ExpressionType::COLUMN_REF:
            // 列引用：从绑定获取类型
            return GetColumnType(expr);

        case ExpressionType::CONSTANT:
            // 常量：从值获取类型
            return GetConstantType(expr);

        case ExpressionType::OPERATOR_ADD:
        case ExpressionType::OPERATOR_MULTIPLY:
            // 算术操作：数值类型提升
            return NumericTypePromotion(expr);

        case ExpressionType::OPERATOR_CONCAT:
            // 字符串连接：VARCHAR
            return LogicalType::VARCHAR;

        case ExpressionType::COMPARE_EQUAL:
        case ExpressionType::COMPARE_GREATERTHAN:
            // 比较：BOOLEAN
            return LogicalType::BOOLEAN;

        default:
            return LogicalType::INVALID;
        }
    }

    // 数值类型提升
    static LogicalType NumericTypePromotion(vector<LogicalType> &types) {
        // 类型层次：TINYINT < SMALLINT < INTEGER < BIGINT < DOUBLE
        // 选择最大类型

        LogicalType max_type = LogicalType::TINYINT;
        for (auto &type : types) {
            if (GetNumericTypeId(type) > GetNumericTypeId(max_type)) {
                max_type = type;
            }
        }

        return max_type;
    }
};

// 隐式类型转换

class ImplicitCast {
public:
    // 检查是否需要隐式转换
    static bool NeedCast(LogicalType from, LogicalType to) {
        // 1. 相同类型：不需要转换
        if (from == to) {
            return false;
        }

        // 2. NULL可以转换为任何类型
        if (from == LogicalType::SQLNULL) {
            return false;
        }

        // 3. VARCHAR可以转换为数值类型（如果格式正确）
        if (from == LogicalType::VARCHAR && IsNumericType(to)) {
            return true;
        }

        // 4. 数值类型可以转换（向上转换）
        if (IsNumericType(from) && IsNumericType(to)) {
            return GetNumericTypeId(from) != GetNumericTypeId(to);
        }

        return false;
    }

    // 插入隐式转换
    static unique_ptr<BoundExpression> InsertCast(
        unique_ptr<BoundExpression> expr,
        LogicalType target_type) {

        LogicalType from_type = expr->return_type;

        if (!NeedCast(from_type, target_type)) {
            return expr;
        }

        // 创建转换表达式
        return make_unique<BoundCastExpression>(
            target_type,
            std::move(expr)
        );
    }
};
```

---

## 第四部分：LogicalOperator详解

### 4.1 LogicalOperator基类

```cpp
// src/include/duckdb/planner/logical_operator.hpp

class LogicalOperator {
public:
    // 算子类型
    LogicalOperatorType type;

    // 子算子列表
    vector<unique_ptr<LogicalOperator>> children;

    // 表达式列表
    vector<unique_ptr<Expression>> expressions;

    // 输出类型
    vector<LogicalType> types;

    // 估算基数
    idx_t estimated_cardinality;

    // 算子名称
    string name;

    LogicalOperator(LogicalOperatorType type)
        : type(type), estimated_cardinality(0) {}

    virtual ~LogicalOperator() = default;

    // ==================== 辅助方法 ====================

    // 获取列名
    virtual vector<string> GetName() const;

    // 获取所有表引用
    virtual void GetTableReferences(
        case_insensitive_set_t<string> &table_set) const;

    // 序列化
    virtual string ToString() const;

    // 验证计划
    virtual void Verify();

    // ==================== 算子属性 ====================

    // 是否为聚合
    bool IsAggregate() const {
        return type == LogicalOperatorType::LOGICAL_AGGREGATE_AND_GROUP_BY;
    }

    // 是否为窗口函数
    bool IsWindow() const {
        return type == LogicalOperatorType::LOGICAL_WINDOW;
    }

    // 是否为投影
    bool IsProjection() const {
        return type == LogicalOperatorType::LOGICAL_PROJECTION;
    }
};
```

### 4.2 LogicalOperator类型

```cpp
// src/include/duckdb/common/enums/operator_type.hpp

enum class LogicalOperatorType : uint8_t {
    // ==================== 数据源 ====================
    LOGICAL_GET,
    LOGICAL_CHUNK_GET,
    LOGICAL_EXPRESSION_GET,
    LOGICAL_DUMMY_SCAN,
    LOGICAL_CTE_REF,

    // ==================== 过滤和投影 ====================
    LOGICAL_PROJECTION,
    LOGICAL_FILTER,
    LOGICAL_AGGREGATE_AND_GROUP_BY,
    LOGICAL_WINDOW,
    LOGICAL_DISTINCT,
    LOGICAL_SAMPLE,

    // ==================== 连接 ====================
    LOGICAL_JOIN,
    LOGICAL_COMPARISON_JOIN,
    LOGICAL_CROSS_PRODUCT,
    LOGICAL_ASOF_JOIN,
    LOGICAL_DEPENDENT_JOIN,

    // ==================== 集合操作 ====================
    LOGICAL_UNION,
    LOGICAL_EXCEPT,
    LOGICAL_INTERSECT,
    LOGICAL_RECURSIVE_CTE,

    // ==================== 排序和限制 ====================
    LOGICAL_ORDER_BY,
    LOGICAL_LIMIT,
    LOGICAL_TOP_N,

    // ==================== 数据修改 ====================
    LOGICAL_INSERT,
    LOGICAL_DELETE,
    LOGICAL_UPDATE,
    LOGICAL_CREATE_TABLE,
    LOGICAL_CREATE_INDEX,

    // ==================== 其他 ====================
    LOGICAL_PREPARE,
    LOGICAL_EXECUTE,
    LOGICAL_EXPLAIN,
    LOGICAL_VACUUM
};
```

### 4.3 常用LogicalOperator详解

#### LogicalGet（表扫描）

```cpp
// src/planner/operator/logical_get.hpp

class LogicalGet : public LogicalOperator {
public:
    // 表函数
    TableFunction function;

    // 绑定数据
    unique_ptr<FunctionData> bind_data;

    // 表名
    string table_name;

    // 列ID（用于投影下推）
    vector<ColumnIndex> column_ids;

    // 表过滤器（用于过滤下推）
    unique_ptr<TableFilterSet> table_filters;

    LogicalGet(TableCatalogEntry &table)
        : LogicalOperator(LogicalOperatorType::LOGICAL_GET),
          table_name(table.name) {

        // 初始化列ID
        for (idx_t i = 0; i < table.GetColumns().size(); i++) {
            column_ids.push_back(i);
        }

        // 设置类型
        types = table.GetTypes();
        names = table.GetColumns();
    }

    // 序列化
    string ToString() const override {
        return string("SCAN ") + table_name;
    }
};
```

#### LogicalFilter（过滤）

```cpp
// src/planner/operator/logical_filter.hpp

class LogicalFilter : public LogicalOperator {
public:
    // 过滤条件表达式
    unique_ptr<Expression> filter;

    LogicalFilter(unique_ptr<Expression> filter)
        : LogicalOperator(LogicalOperatorType::LOGICAL_FILTER),
          filter(std::move(filter)) {}

    void ResolveTypes() {
        // Filter不改变类型
        types = children[0]->types;
    }

    string ToString() const override {
        return "FILTER [" + filter->ToString() + "]";
    }
};
```

#### LogicalProjection（投影）

```cpp
// src/planner/operator/logical_projection.hpp

class LogicalProjection : public LogicalOperator {
public:
    // 投影表达式列表
    vector<unique_ptr<Expression>> select_list;

    LogicalProjection(vector<unique_ptr<Expression>> select_list)
        : LogicalOperator(LogicalOperatorType::LOGICAL_PROJECTION),
          select_list(std::move(select_list)) {}

    void ResolveTypes() {
        // 从表达式推导类型
        types.clear();
        for (auto &expr : select_list) {
            types.push_back(expr->return_type);
        }
    }

    string ToString() const override {
        string result = "PROJECTION [";
        for (size_t i = 0; i < select_list.size(); i++) {
            if (i > 0) result += ", ";
            result += select_list[i]->ToString();
        }
        result += "]";
        return result;
    }
};
```

#### LogicalAggregate（聚合）

```cpp
// src/planner/operator/logical_aggregate.hpp

class LogicalAggregate : public LogicalOperator {
public:
    // 分组列
    vector<unique_ptr<Expression>> groups;

    // 聚合表达式
    vector<unique_ptr<Expression>> aggregates;

    // 分组集合（GROUPING SETS）
    vector<GroupingSet> grouping_sets;

    LogicalAggregate(vector<unique_ptr<Expression>> groups,
                     vector<unique_ptr<Expression>> aggregates)
        : LogicalOperator(LogicalOperatorType::LOGICAL_AGGREGATE_AND_GROUP_BY),
          groups(std::move(groups)),
          aggregates(std::move(aggregates)) {}

    void ResolveTypes() {
        types.clear();

        // 分组列类型
        for (auto &group : groups) {
            types.push_back(group->return_type);
        }

        // 聚合表达式类型
        for (auto &agg : aggregates) {
            types.push_back(agg->return_type);
        }
    }

    string ToString() const override {
        string result = "AGGREGATE\n  GROUPS: [";
        for (size_t i = 0; i < groups.size(); i++) {
            if (i > 0) result += ", ";
            result += groups[i]->ToString();
        }
        result += "]\n  AGGREGATES: [";
        for (size_t i = 0; i < aggregates.size(); i++) {
            if (i > 0) result += ", ";
            result += aggregates[i]->ToString();
        }
        result += "]";
        return result;
    }
};
```

---

## 第五部分：子查询和CTE处理

### 5.1 子查询绑定

```cpp
// 子查询绑定流程

class SubqueryBinder {
public:
    unique_ptr<BoundExpression> BindSubquery(
        SubqueryExpression &expr) {

        // 1. 创建子查询Binder
        // 注意：不继承父Binder的BindContext
        auto sub_binder = make_unique<Binder>(
            context,
            bind_context  // 使用父上下文，用于相关子查询
        );

        // 2. 绑定子查询
        auto bound_subquery = sub_binder->Bind(*expr.subquery);

        // 3. 确定子查询类型
        SubqueryType subquery_type;

        if (expr.subquery_type == SubqueryType::SCALAR) {
            // 标量子查询：返回单行单列
            if (bound_subquery->types.size() != 1) {
                throw BinderException(
                    "Scalar subquery must return exactly one column"
                );
            }
            subquery_type = SubqueryType::SCALAR;
        }
        else if (expr.subquery_type == SubqueryType::EXISTS) {
            // EXISTS子查询
            subquery_type = SubqueryType::EXISTS;
        }
        else if (expr.subquery_type == SubqueryType::ANY) {
            // ANY子查询
            subquery_type = SubqueryType::ANY;
        }

        // 4. 处理相关列
        vector<CorrelatedColumnInfo> correlated_columns;
        FindCorrelatedColumns(bound_subquery->plan,
                            correlated_columns);

        // 5. 创建BoundSubqueryExpression
        return make_unique<BoundSubqueryExpression>(
            bound_subquery->types,
            std::move(bound_subquery->plan),
            subquery_type,
            std::move(correlated_columns)
        );
    }

private:
    // 查找相关列
    void FindCorrelatedColumns(
        LogicalOperator &op,
        vector<CorrelatedColumnInfo> &correlated_columns) {

        // 遍历逻辑计划树
        if (op.type == LogicalOperatorType::LOGICAL_GET) {
            auto &get = (LogicalGet &)op;
            // 检查是否为外部表引用
            if (IsExternalReference(get)) {
                correlated_columns.push_back(
                    CorrelatedColumnInfo(get.table_name)
                );
            }
        }

        for (auto &child : op.children) {
            FindCorrelatedColumns(*child, correlated_columns);
        }
    }
};
```

### 5.2 CTE处理

```cpp
// CTE（Common Table Expression）绑定

class CTEBinder {
public:
    // 绑定CTE
    void BindCTE(CommonTableExpression &expr) {
        // 1. 创建CTE Binder
        auto cte_binder = make_unique<Binder>(context, bind_context);

        // 2. 绑定CTE定义
        auto bound_cte = cte_binder->Bind(*expr.query);

        // 3. 创建CTE绑定信息
        auto cte_binding = make_unique<CTEBinding>(
            expr.alias,
            bound_cte->names,
            bound_cte->types
        );

        // 4. 添加到当前Binder的CTE列表
        bind_context.AddCTEBinding(std::move(cte_binding));

        // 5. 保存绑定计划
        cte_map[expr.alias] = std::move(bound_cte->plan);
    }

    // 引用CTE
    unique_ptr<LogicalOperator> GetCTEPlan(const string &cte_name) {
        auto it = cte_map.find(cte_name);
        if (it == cte_map.end()) {
            throw BinderException("CTE '%s' not found", cte_name);
        }

        // 返回CTE的引用计划
        auto cte_ref = make_unique<LogicalCTERef>(
            it->second  // CTE计划
        );

        return cte_ref;
    }
};
```

#### CTE处理示例

```
示例SQL：
WITH cte1 AS (
    SELECT id, name FROM users WHERE age > 18
),
cte2 AS (
    SELECT user_id, amount FROM orders WHERE amount > 100
)
SELECT c1.name, c2.amount
FROM cte1 c1
JOIN cte2 c2 ON c1.id = c2.user_id;

处理流程：
1. Parser解析
   → CommonTableExpression列表
   → 主查询

2. Binder处理CTE
   ┌─────────────────────────────────────────┐
   │  PreBindCTE("cte1")               │
   │  → 创建cte1的绑定信息                │
   │                                     │
   │  PreBindCTE("cte2")               │
   │  → 创建cte2的绑定信息                │
   └─────────────────────────────────────────┘

3. 绑定主查询
   ┌─────────────────────────────────────────┐
   │  FROM cte1 c1                      │
   │  → 在cte_bindings中查找"cte1"      │
   │  → 返回CTEBinding                   │
   │                                     │
   │  JOIN cte2 c2                       │
   │  → 在cte_bindings中查找"cte2"      │
   │  → 返回CTEBinding                   │
   └─────────────────────────────────────────┘

4. 生成的逻辑计划
   LogicalJoin
   ├── LogicalGet(cte1)
   │   └── LogicalFilter(age > 18)
   │       └── LogicalGet(users)
   └── LogicalGet(cte2)
       └── LogicalFilter(amount > 100)
           └── LogicalGet(orders)
```

---

## 第六部分：聚合和GROUP BY绑定

### 6.1 GROUP BY绑定

```cpp
// GROUP BY绑定器

class GroupBinder : public ExpressionBinder {
public:
    GroupBinder(Binder &binder,
                vector<unique_ptr<Expression>> &groups)
        : ExpressionBinder(binder),
          group_expressions(groups) {}

    // 绑定GROUP BY子句
    void BindGroupBy(vector<ParsedExpression *> &group_expressions) {
        for (auto &expr : group_expressions) {
            auto bound_expr = BindExpression(*expr);

            // 验证：GROUP BY表达式不能包含聚合函数
            if (bound_expr->HasSubquery()) {
                throw BinderException(
                    "GROUP BY cannot contain subqueries"
                );
            }

            group_expressions.push_back(std::move(bound_expr));
        }
    }

    // 处理GROUP BY索引（SELECT列表索引）
    void BindGroupByIndex(vector<idx_t> &group_indexes,
                          vector<unique_ptr<Expression>> &select_list) {
        for (auto idx : group_indexes) {
            if (idx >= select_list.size()) {
                throw BinderException(
                    "GROUP BY column index out of range"
                );
            }

            // 复制SELECT列表中的表达式
            group_expressions.push_back(
                select_list[idx]->Copy()
            );
        }
    }

private:
    vector<unique_ptr<Expression>> &group_expressions;
};
```

### 6.2 聚合函数绑定

```cpp
// 聚合函数绑定器

class AggregateBinder : public ExpressionBinder {
public:
    unique_ptr<BoundExpression> BindAggregate(
        FunctionExpression &expr) {

        // 1. 验证聚合上下文
        if (aggregate_depth == 0) {
            throw BinderException(
                "Aggregate functions are not allowed in WHERE clause"
            );
        }

        // 2. 绑定聚合函数参数
        vector<unique_ptr<BoundExpression>> children;
        for (auto &child : expr.children) {
            // 嵌套聚合：增加深度
            aggregate_depth++;
            children.push_back(BindExpression(*child));
            aggregate_depth--;
        }

        // 3. 查找聚合函数
        auto function = Catalog::GetAggregateFunction(
            context,
            expr.function_name,
            children
        );

        if (!function) {
            throw BinderException(
                "Aggregate function '%s' not found",
                expr.function_name
            );
        }

        // 4. 处理DISTINCT
        bool is_distinct = expr.distinct;

        // 5. 创建BoundAggregateExpression
        return make_unique<BoundAggregateExpression>(
            function->return_type,
            std::move(function),
            std::move(children),
            is_distinct
        );
    }
};
```

### 6.3 HAVING绑定

```cpp
// HAVING子句绑定

class HavingBinder : public ExpressionBinder {
public:
    unique_ptr<BoundExpression> BindHaving(
        ParsedExpression &expr,
        vector<unique_ptr<Expression>> &groups) {

        // 1. 创建HAVING Binder
        // 允许使用聚合函数
        auto having_binder = make_unique<HavingBinder>(
            binder, groups
        );

        // 2. 绑定HAVING表达式
        auto bound_expr = having_binder->BindExpression(expr);

        // 3. 验证：HAVING必须包含聚合或分组列
        if (!bound_expr->HasAggregate() &&
            !bound_expr->HasGroupRef()) {
            throw BinderException(
                "HAVING clause must contain aggregate or GROUP BY columns"
            );
        }

        return bound_expr;
    }
};
```

---

## 第七部分：完整示例

### 7.1 SELECT语句完整绑定过程

```sql
-- 示例查询
SELECT department, COUNT(*) as emp_count, AVG(salary) as avg_salary
FROM employees
WHERE salary > 50000
GROUP BY department
HAVING AVG(salary) > 60000
ORDER BY emp_count DESC
LIMIT 10;
```

```cpp
// 完整绑定流程

unique_ptr<BoundStatement> BindSelect(SelectStatement &stmt) {
    auto bound_statement = make_unique<BoundSelectStatement>();

    // ==================== 步骤1：绑定CTE ====================
    if (stmt.cte_map) {
        for (auto &cte : stmt.cte_map->ctes) {
            BindCTE(cte);
        }
    }

    // ==================== 步骤2：绑定FROM子句 ====================
    BindFrom(stmt.from);

    // ==================== 步骤3：绑定WHERE子句 ====================
    if (stmt.where) {
        auto where_expr = BindExpression(*stmt.where);
        bound_statement->where = std::move(where_expr);
    }

    // ==================== 步骤4：绑定GROUP BY子句 ====================
    if (!stmt.group_by.empty()) {
        GroupBinder group_binder(*this, bound_statement->group_expressions);
        group_binder.BindGroupBy(stmt.group_by);
    }

    // ==================== 步骤5：绑定SELECT列表 ====================
    SelectBinder select_binder(*this);
    for (auto &select_elem : stmt.select_list) {
        auto bound_expr = select_binder.BindExpression(*select_elem);
        bound_statement->select_list.push_back(std::move(bound_expr));
    }

    // ==================== 步骤6：绑定HAVING子句 ====================
    if (stmt.having) {
        HavingBinder having_binder(*this,
                                     bound_statement->group_expressions);
        auto having_expr = having_binder.BindExpression(*stmt.having);
        bound_statement->having = std::move(having_expr);
    }

    // ==================== 步骤7：绑定ORDER BY子句 ====================
    if (!stmt.order_by.empty()) {
        OrderBinder order_binder(*this);
        for (auto &order_elem : stmt.order_by) {
            auto bound_order = order_binder.BindOrder(order_elem);
            bound_statement->order_by.push_back(std::move(bound_order));
        }
    }

    // ==================== 步骤8：绑定LIMIT子句 ====================
    if (stmt.limit) {
        bound_statement->limit = BindExpression(*stmt.limit);
    }

    if (stmt.offset) {
        bound_statement->offset = BindExpression(*stmt.offset);
    }

    // ==================== 步骤9：创建逻辑计划 ====================
    bound_statement->plan = CreateLogicalPlan(*bound_statement);

    return bound_statement;
}

// 创建逻辑计划
unique_ptr<LogicalOperator> CreateLogicalPlan(
    BoundSelectStatement &stmt) {

    // 1. 从FROM创建Get算子
    unique_ptr<LogicalOperator> plan = CreateGet(stmt.from);

    // 2. 添加WHERE过滤
    if (stmt.where) {
        auto filter = make_unique<LogicalFilter>(
            stmt.where->Copy()
        );
        filter->children.push_back(std::move(plan));
        plan = std::move(filter);
    }

    // 3. 添加聚合
    if (!stmt.group_expressions.empty() || HasAggregates(stmt.select_list)) {
        auto aggregate = make_unique<LogicalAggregate>(
            stmt.group_expressions,
            stmt.select_list  // 包含聚合函数
        );
        aggregate->children.push_back(std::move(plan));
        plan = std::move(aggregate);
    }

    // 4. 添加HAVING过滤
    if (stmt.having) {
        auto filter = make_unique<LogicalFilter>(
            stmt.having->Copy()
        );
        filter->children.push_back(std::move(plan));
        plan = std::move(filter);
    }
    else if (!stmt.group_expressions.empty() && HasNonGroupedColumns()) {
        // 特殊情况：有GROUP BY但SELECT中有非分组列
        // 需要添加投影
        auto projection = CreateProjection(stmt.select_list);
        projection->children.push_back(std::move(plan));
        plan = std::move(projection);
    }

    // 5. 添加ORDER BY
    if (!stmt.order_by.empty()) {
        auto order_by = make_unique<LogicalOrder>(
            stmt.order_by
        );
        order_by->children.push_back(std::move(plan));
        plan = std::move(order_by);
    }

    // 6. 添加LIMIT
    if (stmt.limit || stmt.offset) {
        auto limit = make_unique<LogicalLimit>(
            stmt.limit ? stmt.limit->Copy() : nullptr,
            stmt.offset ? stmt.offset->Copy() : nullptr
        );
        limit->children.push_back(std::move(plan));
        plan = std::move(limit);
    }

    return plan;
}
```

---

## 学习总结

### Planner与Binder关键要点

1. **分层处理**：Parser → Transformer → Binder → Planner
2. **符号解析**：BindContext维护作用域层次
3. **类型推导**：自动推断表达式类型
4. **逻辑计划**：LogicalOperator树表示查询结构
5. **子查询处理**：相关子查询和CTE的特殊处理

### SQL语句处理对照

| SQL语句 | 主要LogicalOperator | 特殊处理 |
|---------|------------------|---------|
| SELECT | Get + Filter + Projection | 聚合和GROUP BY |
| INSERT | Insert + Get(子查询) | RETURNING子句 |
| UPDATE | Update + Get | 多表更新 |
| DELETE | Delete + Get | USING子句 |
| CTE | CTE Ref + 重复计划 | 递归CTE |
| UNION | Union + Projection | ALL vs DISTINCT |

### 推荐阅读

**论文：**
- "System R: Relational Database Management System"
- "The Architecture of a Database System"
- "How to Design a Database System"

**代码位置：**
- `src/parser/` - Parser实现
- `src/planner/binder.cpp` - Binder主逻辑
- `src/planner/expression_binder/` - 表达式绑定
- `src/planner/operator/` - LogicalOperator实现

**相关课程：**
- [DuckDB高级课程_查询优化器深度解析](./DuckDB高级课程_查询优化器深度解析.md)
- [DuckDB深度课程_执行算子实现详解](./DuckDB深度课程_执行算子实现详解.md)

---

**最后更新：2026-01-23**
