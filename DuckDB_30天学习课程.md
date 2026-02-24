# DuckDB 30天深度学习课程

## 课程概述

本课程将带你深入学习DuckDB的核心架构、性能优化技巧和数据库底层知识。通过30天的学习，你将理解如何构建一个高性能的分析型数据库系统。

**学习目标：**
- 掌握向量化执行引擎的设计与实现
- 理解查询优化器的工作原理
- 学习列式存储和压缩技术
- 掌握MVCC事务管理
- 能够设计并实现一个简化版的分析型数据库

---

## 第一周：核心架构与数据表示

### Day 1: DuckDB总体架构概览

**学习目标：** 理解DuckDB的整体架构和各模块的职责

#### 1.1 架构层次

DuckDB采用经典的分层数据库架构：

```
┌─────────────────────────────────────────┐
│         SQL 查询字符串                   │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│  Parser (src/parser/)                   │
│  - 使用libpg_query解析SQL               │
│  - 生成AST (ParsedStatement)            │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│  Planner + Binder (src/planner/)        │
│  - 符号解析和类型检查                   │
│  - 生成LogicalOperator树                │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│  Optimizer (src/optimizer/)             │
│  - 40+个优化Pass                        │
│  - Filter Pushdown, Join Ordering等     │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│  Physical Planner (src/execution/)      │
│  - LogicalOp → PhysicalOp转换           │
│  - 生成执行计划                         │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│  Executor + Pipeline (src/execution/)   │
│  - 向量化执行                           │
│  - 并行执行Pipeline                     │
└─────────────────────────────────────────┘
              ↓
┌─────────────────────────────────────────┐
│  Storage (src/storage/)                 │
│  - 列式存储，压缩                       │
│  - BufferManager, WAL                   │
└─────────────────────────────────────────┘
```

#### 1.2 关键设计理念

**向量化执行（Vectorization）**
- 数据以列向量（Vector）批量处理，而非逐行处理
- 典型批大小：2048行（STANDARD_VECTOR_SIZE）
- 利用CPU缓存和SIMD指令

**列式存储（Columnar Storage）**
- 按列组织数据，提高分析查询性能
- 更好的压缩率
- 更少的I/O操作

**零拷贝架构（Zero-Copy）**
- 尽可能避免数据拷贝
- 使用引用和切片机制

#### 1.3 核心模块职责

查看 `src/README.md` 中的模块说明：

```cpp
// src/parser/ - 解析SQL生成AST
// src/planner/ - 符号解析，生成逻辑计划
// src/optimizer/ - 查询优化
// src/execution/ - 执行引擎
// src/catalog/ - 元数据管理
// src/storage/ - 物理存储
// src/transaction/ - 事务管理
// src/common/ - 公共组件（类型系统、向量等）
```

**实践任务：**
1. 克隆DuckDB仓库，编译debug版本：`make debug`
2. 运行一个简单查询并观察执行流程
3. 阅读 `src/README.md` 和 `CLAUDE.md`

---

### Day 2: 类型系统与向量化执行基础

**学习目标：** 理解DuckDB的类型系统和向量化执行的核心概念

#### 2.1 类型系统

DuckDB区分逻辑类型（LogicalType）和物理类型（PhysicalType）：

**逻辑类型** (`src/include/duckdb/common/types.hpp`)
```cpp
// 用户可见的类型：INTEGER, VARCHAR, DECIMAL, STRUCT, LIST等
enum class LogicalTypeId : uint8_t {
    BOOLEAN,
    TINYINT,
    SMALLINT,
    INTEGER,
    BIGINT,
    HUGEINT,
    FLOAT,
    DOUBLE,
    DECIMAL,
    VARCHAR,
    BLOB,
    TIMESTAMP,
    DATE,
    TIME,
    INTERVAL,
    STRUCT,
    LIST,
    MAP,
    // ... 更多类型
};
```

**物理类型** - 内存中的实际存储格式
```cpp
enum class PhysicalType : uint8_t {
    BOOL,
    INT8,
    INT16,
    INT32,
    INT64,
    UINT8,
    UINT16,
    UINT32,
    UINT64,
    INT128,
    FLOAT,
    DOUBLE,
    VARCHAR,
    STRUCT,
    LIST,
    INVALID
};
```

**类型转换示例：**
- `DECIMAL(10,2)` (逻辑类型) → `INT64` (物理类型，存储为定点数)
- `VARCHAR` (逻辑类型) → `VARCHAR` (物理类型，变长字符串)

#### 2.2 向量化执行的优势

**传统行式执行：**
```cpp
// 伪代码：逐行处理
for (int i = 0; i < num_rows; i++) {
    result[i] = process_row(input[i]);  // 每次一行
}
```

**向量化执行：**
```cpp
// 伪代码：批量处理
for (int chunk = 0; chunk < num_chunks; chunk++) {
    process_vector(input_chunk, result_chunk, VECTOR_SIZE);  // 一次2048行
}
```

**性能优势：**
1. **减少函数调用开销** - 一次处理多行，减少循环和分支
2. **利用CPU缓存** - 连续内存访问
3. **SIMD加速** - 一条指令处理多个数据
4. **降低解释开销** - 减少虚函数调用

#### 2.3 idx_t类型

DuckDB使用 `idx_t` 作为索引/计数类型：

```cpp
// src/include/duckdb/common/types.hpp
using idx_t = uint64_t;

// 使用示例
idx_t row_count = 1000;
idx_t column_index = 5;

// 特殊值
constexpr idx_t INVALID_INDEX = (idx_t)-1;  // DConstants::INVALID_INDEX
```

**为什么不用size_t？**
- 跨平台一致性（size_t在32位和64位系统不同）
- 更清晰的语义（专用于DuckDB的索引）

**实践任务：**
1. 阅读 `src/include/duckdb/common/types.hpp`
2. 理解LogicalType和PhysicalType的映射关系
3. 思考：为什么DECIMAL用整数存储？

---

### Day 3: Vector和DataChunk - 数据流动的载体

**学习目标：** 深入理解Vector和DataChunk的设计与实现

#### 3.1 Vector结构

**Vector定义** (`src/include/duckdb/common/types/vector.hpp`)

```cpp
class Vector {
    friend struct ConstantVector;
    friend struct FlatVector;
    friend class DataChunk;

public:
    // 构造函数
    Vector(LogicalType type, idx_t capacity = STANDARD_VECTOR_SIZE);
    Vector(Vector &other);  // 引用构造
    Vector(const Value &value);  // 常量向量

    LogicalType type;          // 逻辑类型
    VectorType vector_type;    // 向量类型
    data_ptr_t data;           // 实际数据指针
    ValidityMask validity;     // NULL标记位图
    shared_ptr<VectorBuffer> buffer;  // 数据缓冲区
    shared_ptr<VectorBuffer> auxiliary;  // 辅助数据（如字符串）
};
```

#### 3.2 Vector类型

DuckDB支持多种Vector类型：

**1. FLAT_VECTOR（平坦向量）**
```cpp
// 最常见的向量类型
// 数据连续存储，可能包含NULL值
Vector flat_vec(LogicalType::INTEGER, 2048);
int32_t *data = FlatVector::GetData<int32_t>(flat_vec);
data[0] = 42;
data[1] = 100;
```

**2. CONSTANT_VECTOR（常量向量）**
```cpp
// 所有元素值相同
// 例如：SELECT 42 FROM table
Vector const_vec(Value::INTEGER(42));
```

**3. DICTIONARY_VECTOR（字典向量）**
```cpp
// 使用SelectionVector索引另一个向量
// 用于过滤、去重等操作，避免数据拷贝
```

**4. SEQUENCE_VECTOR（序列向量）**
```cpp
// 表示算术序列，不实际存储数据
// 例如：1, 2, 3, 4, ...
```

#### 3.3 ValidityMask - NULL值处理

```cpp
// src/include/duckdb/common/types/validity_mask.hpp
class ValidityMask {
    validity_t *validity_mask;  // 位图

public:
    // 检查某行是否有效（非NULL）
    bool RowIsValid(idx_t row_idx) const;

    // 设置某行为NULL
    void SetInvalid(idx_t row_idx);

    // 设置某行为有效
    void SetValid(idx_t row_idx);
};
```

**位图编码：**
- 每个bit代表一行
- 1 = 有效值，0 = NULL
- 64行使用8字节（uint64_t）

#### 3.4 DataChunk结构

**DataChunk定义** (`src/include/duckdb/common/types/data_chunk.hpp`)

```cpp
class DataChunk {
public:
    vector<Vector> data;  // 多个列向量
    idx_t count;          // 当前行数（≤ capacity）
    idx_t capacity;       // 最大容量

    // 初始化
    void Initialize(Allocator &allocator,
                    const vector<LogicalType> &types,
                    idx_t capacity = STANDARD_VECTOR_SIZE);

    // 重置为空
    void Reset();

    // 设置行数
    void SetCardinality(idx_t count);

    // 获取某个值
    Value GetValue(idx_t col_idx, idx_t row_idx) const;
};
```

**DataChunk示例：**

```
DataChunk with 3 columns, 1024 rows:
┌──────────────┬──────────────┬──────────────┐
│   Column 0   │   Column 1   │   Column 2   │
│   INTEGER    │   VARCHAR    │   DOUBLE     │
├──────────────┼──────────────┼──────────────┤
│      1       │    "Alice"   │    95.5      │
│      2       │    "Bob"     │    87.3      │
│      3       │    NULL      │    92.1      │
│     ...      │     ...      │     ...      │
│    1024      │   "Zara"     │    88.9      │
└──────────────┴──────────────┴──────────────┘
```

#### 3.5 代码实例：创建和操作DataChunk

```cpp
// 位置: 可以在test/unittest.cpp中添加测试

#include "duckdb/common/types/data_chunk.hpp"

void ExampleDataChunk() {
    // 1. 定义schema
    vector<LogicalType> types = {
        LogicalType::INTEGER,
        LogicalType::VARCHAR,
        LogicalType::DOUBLE
    };

    // 2. 创建DataChunk
    DataChunk chunk;
    chunk.Initialize(Allocator::DefaultAllocator(), types);

    // 3. 填充数据
    auto &col0 = chunk.data[0];  // INTEGER列
    auto &col1 = chunk.data[1];  // VARCHAR列
    auto &col2 = chunk.data[2];  // DOUBLE列

    auto col0_data = FlatVector::GetData<int32_t>(col0);
    col0_data[0] = 1;
    col0_data[1] = 2;
    col0_data[2] = 3;

    // 设置字符串
    col1.SetValue(0, Value("Alice"));
    col1.SetValue(1, Value("Bob"));
    col1.SetValue(2, Value::STRING(nullptr));  // NULL值

    auto col2_data = FlatVector::GetData<double>(col2);
    col2_data[0] = 95.5;
    col2_data[1] = 87.3;
    col2_data[2] = 92.1;

    chunk.SetCardinality(3);

    // 4. 读取数据
    for (idx_t row = 0; row < chunk.size(); row++) {
        Value v0 = chunk.GetValue(0, row);
        Value v1 = chunk.GetValue(1, row);
        Value v2 = chunk.GetValue(2, row);
        // 处理...
    }
}
```

**实践任务：**
1. 阅读 `src/common/types/vector.cpp` 的前200行
2. 阅读 `src/common/types/data_chunk.cpp` 的Initialize函数
3. 在 `build/debug/test/unittest` 中添加测试代码，创建DataChunk并打印

---

### Day 4: 内存管理与BufferManager

**学习目标：** 理解DuckDB的内存管理机制和缓冲池设计

#### 4.1 内存管理架构

DuckDB采用分层内存管理：

```
┌─────────────────────────────────────┐
│  Allocator (分配器接口)             │
│  - DefaultAllocator                 │
│  - DebugAllocator                   │
└─────────────────────────────────────┘
           ↓
┌─────────────────────────────────────┐
│  BufferManager (缓冲管理器)         │
│  - 管理内存块（Block）              │
│  - LRU淘汰策略                      │
│  - 内存限制enforcement              │
└─────────────────────────────────────┘
           ↓
┌─────────────────────────────────────┐
│  BlockManager (块管理器)            │
│  - 持久化块到磁盘                   │
│  - 临时块管理                       │
└─────────────────────────────────────┘
```

#### 4.2 Allocator接口

```cpp
// src/include/duckdb/common/allocator.hpp
class Allocator {
public:
    // 分配内存
    virtual data_ptr_t Allocate(idx_t size);

    // 释放内存
    virtual void Free(data_ptr_t pointer, idx_t size);

    // 重新分配
    virtual data_ptr_t Reallocate(data_ptr_t pointer,
                                   idx_t old_size,
                                   idx_t new_size);
};
```

**为什么需要自定义Allocator？**
1. 追踪内存使用量
2. 执行内存限制
3. Debug模式检测内存泄漏
4. 优化分配性能（Arena分配）

#### 4.3 BufferManager核心概念

**BufferPool** - 内存池
```cpp
// src/include/duckdb/storage/buffer_manager.hpp
class BufferManager {
    idx_t maximum_memory;        // 最大内存限制
    idx_t current_memory;        // 当前使用内存

    // 分配缓冲区
    shared_ptr<BlockHandle> Allocate(idx_t block_size);

    // Pin（固定）缓冲区，防止被淘汰
    BufferHandle Pin(shared_ptr<BlockHandle> handle);

    // Unpin，允许淘汰
    void Unpin(shared_ptr<BlockHandle> handle);
};
```

**Block** - 固定大小的内存块
```cpp
// 默认块大小: 256KB (Storage::BLOCK_SIZE)
constexpr idx_t BLOCK_SIZE = 262144;

struct BlockHandle {
    idx_t block_id;           // 块ID
    data_ptr_t buffer;        // 内存指针
    bool is_pinned;           // 是否被固定
    uint32_t readers;         // 读者数量
};
```

#### 4.4 BufferHandle - RAII内存管理

```cpp
// src/include/duckdb/storage/buffer/buffer_handle.hpp
class BufferHandle {
    shared_ptr<BlockHandle> handle;

public:
    // 构造时Pin
    BufferHandle(BufferManager &manager, shared_ptr<BlockHandle> handle);

    // 析构时Unpin
    ~BufferHandle();

    // 获取数据指针
    data_ptr_t Ptr() const { return handle->buffer; }
};
```

**RAII模式示例：**
```cpp
void ProcessData(BufferManager &manager) {
    // 自动Pin
    BufferHandle handle = manager.Pin(block_handle);

    // 使用数据
    data_ptr_t data = handle.Ptr();
    // ... 处理 ...

    // 离开作用域时自动Unpin
}
```

#### 4.5 LRU淘汰策略

当内存不足时，BufferManager使用LRU（Least Recently Used）淘汰unpinned的块：

```cpp
// 简化的淘汰逻辑
void BufferManager::Evict(idx_t required_memory) {
    while (current_memory + required_memory > maximum_memory) {
        // 找到最久未使用的unpinned块
        auto victim = lru_queue.back();

        if (victim->is_dirty) {
            // 写回磁盘
            WriteBlockToDisk(victim);
        }

        // 释放内存
        FreeBlock(victim);
        lru_queue.pop_back();
    }
}
```

#### 4.6 Arena分配器

对于查询执行期间的临时对象，DuckDB使用Arena分配器：

```cpp
// src/include/duckdb/common/allocator.hpp
class ArenaAllocator : public Allocator {
    vector<data_ptr_t> chunks;  // 大块内存
    idx_t current_offset;

public:
    data_ptr_t Allocate(idx_t size) override {
        // 在当前chunk中分配
        if (current_offset + size > chunk_size) {
            // 分配新chunk
            AllocateNewChunk();
        }
        auto ptr = chunks.back() + current_offset;
        current_offset += size;
        return ptr;
    }

    // 一次性释放所有内存
    void Reset() {
        for (auto chunk : chunks) {
            free(chunk);
        }
        chunks.clear();
    }
};
```

**Arena优势：**
- 快速分配（无需维护free list）
- 统一释放（查询结束时）
- 减少内存碎片

**实践任务：**
1. 阅读 `src/storage/buffer_manager.cpp` 了解Pin/Unpin机制
2. 理解BufferHandle的RAII模式
3. 思考：为什么需要Pin/Unpin？直接分配释放不行吗？

---

### Day 5: SQL解析器架构

**学习目标：** 理解SQL解析流程和AST结构

#### 5.1 Parser架构

DuckDB使用PostgreSQL的libpg_query作为底层解析器：

```
SQL String
    ↓
┌─────────────────────────────────┐
│  libpg_query                    │
│  (PostgreSQL parser)            │
│  - 词法分析 (Lexer)             │
│  - 语法分析 (Parser)            │
└─────────────────────────────────┘
    ↓ Postgres AST
┌─────────────────────────────────┐
│  Transformer                    │
│  (src/parser/transform/)        │
│  - 转换为DuckDB AST             │
└─────────────────────────────────┘
    ↓ DuckDB AST
┌─────────────────────────────────┐
│  SQLStatement                   │
│  - SelectStatement              │
│  - InsertStatement              │
│  - CreateTableStatement         │
│  - ...                          │
└─────────────────────────────────┘
```

#### 5.2 SQLStatement层次结构

```cpp
// src/include/duckdb/parser/sql_statement.hpp
class SQLStatement {
public:
    StatementType type;  // SELECT, INSERT, CREATE, etc.

    virtual string ToString() const;
    virtual void Serialize(Serializer &serializer) const;
};

// 主要语句类型
enum class StatementType : uint8_t {
    SELECT_STATEMENT,
    INSERT_STATEMENT,
    UPDATE_STATEMENT,
    DELETE_STATEMENT,
    CREATE_STATEMENT,
    DROP_STATEMENT,
    ALTER_STATEMENT,
    TRANSACTION_STATEMENT,
    // ...
};
```

#### 5.3 SelectStatement结构

```cpp
// src/include/duckdb/parser/statement/select_statement.hpp
class SelectStatement : public SQLStatement {
public:
    unique_ptr<QueryNode> node;  // 查询节点（SelectNode或SetOperationNode）
    vector<OrderByNode> orders;  // ORDER BY
    vector<unique_ptr<ParsedExpression>> limit;  // LIMIT/OFFSET
};

class SelectNode : public QueryNode {
public:
    vector<unique_ptr<ParsedExpression>> select_list;  // SELECT列表
    unique_ptr<TableRef> from_table;                   // FROM子句
    unique_ptr<ParsedExpression> where_clause;         // WHERE
    vector<unique_ptr<ParsedExpression>> groups;       // GROUP BY
    unique_ptr<ParsedExpression> having;               // HAVING
};
```

**示例SQL：**
```sql
SELECT name, AVG(score)
FROM students
WHERE age > 18
GROUP BY name
HAVING AVG(score) > 80
ORDER BY name
LIMIT 10;
```

**对应的AST：**
```
SelectStatement
├── SelectNode
│   ├── select_list
│   │   ├── ColumnRefExpression("name")
│   │   └── FunctionExpression("AVG", [ColumnRefExpression("score")])
│   ├── from_table
│   │   └── BaseTableRef("students")
│   ├── where_clause
│   │   └── ComparisonExpression(">", ColumnRef("age"), Constant(18))
│   ├── groups
│   │   └── ColumnRefExpression("name")
│   └── having
│       └── ComparisonExpression(">", FunctionExpression("AVG", [...]), Constant(80))
├── orders
│   └── OrderByNode(ColumnRef("name"), ASC)
└── limit
    └── ConstantExpression(10)
```

#### 5.4 Expression类型

```cpp
// src/include/duckdb/parser/expression.hpp
class ParsedExpression {
public:
    ExpressionClass type;
    ExpressionType expression_type;

    virtual string ToString() const = 0;
};

enum class ExpressionClass : uint8_t {
    COLUMN_REF,           // 列引用: table.column
    CONSTANT,             // 常量: 42, 'hello'
    FUNCTION,             // 函数调用: SUM(x)
    OPERATOR,             // 运算符: a + b
    COMPARISON,           // 比较: a > b
    CONJUNCTION,          // AND/OR
    SUBQUERY,             // 子查询
    CASE,                 // CASE WHEN
    CAST,                 // CAST(x AS INT)
    // ...
};
```

**常用Expression子类：**

1. **ColumnRefExpression** - 列引用
```cpp
class ColumnRefExpression : public ParsedExpression {
public:
    string column_name;
    string table_name;  // 可选
};
```

2. **ConstantExpression** - 常量
```cpp
class ConstantExpression : public ParsedExpression {
public:
    Value value;  // 实际值
};
```

3. **FunctionExpression** - 函数调用
```cpp
class FunctionExpression : public ParsedExpression {
public:
    string function_name;
    vector<unique_ptr<ParsedExpression>> children;  // 参数
    bool is_operator;  // 是否是运算符形式（a+b vs ADD(a,b)）
};
```

4. **ComparisonExpression** - 比较表达式
```cpp
class ComparisonExpression : public ParsedExpression {
public:
    ExpressionType type;  // =, <, >, <=, >=, !=
    unique_ptr<ParsedExpression> left;
    unique_ptr<ParsedExpression> right;
};
```

#### 5.5 TableRef类型

```cpp
// src/include/duckdb/parser/tableref.hpp
class TableRef {
public:
    TableReferenceType type;
    string alias;
};

enum class TableReferenceType : uint8_t {
    BASE_TABLE,          // 基表: FROM users
    JOIN,                // JOIN: FROM a JOIN b ON ...
    SUBQUERY,            // 子查询: FROM (SELECT ...)
    TABLE_FUNCTION,      // 表函数: FROM read_csv(...)
    CROSS_PRODUCT,       // 笛卡尔积: FROM a, b
    // ...
};
```

#### 5.6 Transformer工作流程

```cpp
// src/parser/transform/transform.cpp
// 简化版本

class Transformer {
public:
    unique_ptr<SQLStatement> TransformStatement(Node *stmt) {
        switch (stmt->type) {
        case T_SelectStmt:
            return TransformSelect(stmt);
        case T_InsertStmt:
            return TransformInsert(stmt);
        // ... 其他语句类型
        }
    }

private:
    unique_ptr<SelectStatement> TransformSelect(Node *node) {
        auto result = make_unique<SelectStatement>();

        // 转换SELECT列表
        result->node->select_list = TransformSelectList(node->targetList);

        // 转换FROM子句
        result->node->from_table = TransformFrom(node->fromClause);

        // 转换WHERE子句
        if (node->whereClause) {
            result->node->where_clause = TransformExpression(node->whereClause);
        }

        // ... 其他子句
        return result;
    }
};
```

**实践任务：**
1. 阅读 `src/parser/transform/statement/transform_select.cpp`
2. 使用DuckDB CLI解析一个复杂SQL，观察AST：
   ```sql
   .explain SELECT * FROM t1 JOIN t2 ON t1.id = t2.id WHERE t1.value > 100;
   ```
3. 理解ParsedExpression和TableRef的继承关系

---

### Day 6: 表达式系统

**学习目标：** 深入理解表达式的绑定和求值机制

#### 6.1 表达式的生命周期

表达式在DuckDB中经历三个阶段：

```
ParsedExpression (Parser)
    ↓ Binding
BoundExpression (Planner)
    ↓ Execution
ExpressionExecutor
```

**1. ParsedExpression** - 解析阶段
- 只包含字符串引用（列名、函数名）
- 类型未知
- 未验证正确性

**2. BoundExpression** - 绑定阶段
- 列引用已解析到具体表和列
- 类型已推导
- 函数已绑定到具体实现

**3. ExpressionExecutor** - 执行阶段
- 在Vector上执行表达式
- 向量化计算

#### 6.2 Binder - 符号解析

```cpp
// src/planner/binder/binder.cpp
class Binder {
    Catalog &catalog;
    ClientContext &context;

public:
    // 绑定表达式
    unique_ptr<Expression> Bind(unique_ptr<ParsedExpression> expr) {
        switch (expr->type) {
        case ExpressionClass::COLUMN_REF:
            return BindColumnRef(expr);
        case ExpressionClass::FUNCTION:
            return BindFunction(expr);
        case ExpressionClass::COMPARISON:
            return BindComparison(expr);
        // ...
        }
    }

private:
    unique_ptr<Expression> BindColumnRef(ParsedExpression *expr) {
        auto &colref = (ColumnRefExpression&)*expr;

        // 在当前作用域查找列
        auto binding = bind_context.GetColumn(colref.column_name);

        // 创建BoundColumnRefExpression
        return make_unique<BoundColumnRefExpression>(
            colref.column_name,
            binding.type,
            binding.table_index,
            binding.column_index
        );
    }
};
```

#### 6.3 BoundExpression层次

```cpp
// src/include/duckdb/planner/expression.hpp
class Expression {
public:
    ExpressionClass expression_class;
    ExpressionType type;
    LogicalType return_type;  // 返回类型（已推导）

    virtual bool HasSideEffects() const;
    virtual bool IsScalar() const;
    virtual bool IsAggregate() const;
};
```

**主要BoundExpression类型：**

1. **BoundColumnRefExpression** - 绑定的列引用
```cpp
class BoundColumnRefExpression : public Expression {
public:
    idx_t binding;       // 表绑定索引
    idx_t column_index;  // 列索引
    LogicalType return_type;
};
```

2. **BoundConstantExpression** - 常量
```cpp
class BoundConstantExpression : public Expression {
public:
    Value value;
};
```

3. **BoundFunctionExpression** - 标量函数
```cpp
class BoundFunctionExpression : public Expression {
public:
    ScalarFunction function;  // 绑定的函数实现
    vector<unique_ptr<Expression>> children;  // 参数
    unique_ptr<FunctionData> bind_info;  // 绑定时的额外信息
};
```

4. **BoundAggregateExpression** - 聚合函数
```cpp
class BoundAggregateExpression : public Expression {
public:
    AggregateFunction function;
    vector<unique_ptr<Expression>> children;
    unique_ptr<FunctionData> bind_info;
    AggregateType aggr_type;  // DISTINCT, ORDER BY等
};
```

5. **BoundComparisonExpression** - 比较表达式
```cpp
class BoundComparisonExpression : public Expression {
public:
    ExpressionType type;  // =, <, >, ...
    unique_ptr<Expression> left;
    unique_ptr<Expression> right;
};
```

#### 6.4 函数绑定过程

```cpp
// src/planner/binder/expression/bind_function_expression.cpp

unique_ptr<Expression> Binder::BindFunction(FunctionExpression *expr) {
    // 1. 绑定参数
    vector<unique_ptr<Expression>> children;
    vector<LogicalType> arguments;
    for (auto &child : expr->children) {
        auto bound_child = Bind(child);
        arguments.push_back(bound_child->return_type);
        children.push_back(std::move(bound_child));
    }

    // 2. 在Catalog中查找函数
    auto &func_catalog = catalog.GetEntry<ScalarFunctionCatalogEntry>(
        context, expr->function_name);

    // 3. 根据参数类型选择最佳重载
    auto function = func_catalog.functions.GetFunctionByArguments(arguments);

    // 4. 调用bind函数（如果有）
    unique_ptr<FunctionData> bind_data;
    if (function.bind) {
        bind_data = function.bind(context, function, children);
    }

    // 5. 创建BoundFunctionExpression
    return make_unique<BoundFunctionExpression>(
        function.return_type,
        std::move(function),
        std::move(children),
        std::move(bind_data)
    );
}
```

#### 6.5 ExpressionExecutor - 向量化求值

```cpp
// src/execution/expression_executor.cpp
class ExpressionExecutor {
    Expression &expr;

public:
    // 在DataChunk上执行表达式
    void Execute(DataChunk &input, Vector &result) {
        ExecuteExpression(expr, input, result);
    }

private:
    void ExecuteExpression(Expression &expr, DataChunk &input, Vector &result) {
        switch (expr.expression_class) {
        case ExpressionClass::BOUND_COLUMN_REF:
            ExecuteColumnRef(expr, input, result);
            break;
        case ExpressionClass::BOUND_FUNCTION:
            ExecuteFunction(expr, input, result);
            break;
        // ...
        }
    }

    void ExecuteColumnRef(BoundColumnRefExpression &expr,
                          DataChunk &input,
                          Vector &result) {
        // 直接引用输入列
        result.Reference(input.data[expr.column_index]);
    }

    void ExecuteFunction(BoundFunctionExpression &expr,
                         DataChunk &input,
                         Vector &result) {
        // 1. 执行所有参数
        vector<Vector> arguments;
        for (auto &child : expr.children) {
            Vector arg(child->return_type);
            Execute(child, input, arg);
            arguments.push_back(std::move(arg));
        }

        // 2. 调用函数
        expr.function.function(arguments, result, input.size());
    }
};
```

#### 6.6 ScalarFunction接口

```cpp
// src/include/duckdb/function/scalar_function.hpp
struct ScalarFunction {
    string name;
    vector<LogicalType> arguments;
    LogicalType return_type;

    // 函数实现（向量化）
    scalar_function_t function;

    // 可选：绑定时调用
    bind_scalar_function_t bind;

    // 可选：依赖信息
    function_statistics_t statistics;
};

// 函数签名
typedef void (*scalar_function_t)(
    DataChunk &args,       // 输入参数
    ExpressionState &state,
    Vector &result         // 输出结果
);
```

**函数实现示例：**
```cpp
// 实现 ABS(x) 函数
void AbsFunction(DataChunk &args, ExpressionState &state, Vector &result) {
    auto &input = args.data[0];
    auto input_data = FlatVector::GetData<int32_t>(input);
    auto result_data = FlatVector::GetData<int32_t>(result);

    // 向量化处理
    for (idx_t i = 0; i < args.size(); i++) {
        result_data[i] = std::abs(input_data[i]);
    }

    // 处理NULL值
    if (input.validity.AllValid()) {
        result.validity.SetAllValid(args.size());
    } else {
        for (idx_t i = 0; i < args.size(); i++) {
            if (!input.validity.RowIsValid(i)) {
                result.validity.SetInvalid(i);
            }
        }
    }
}
```

**实践任务：**
1. 阅读 `src/planner/binder/expression/bind_columnref_expression.cpp`
2. 阅读 `src/execution/expression_executor.cpp` 的Execute函数
3. 实现一个简单的标量函数（如SQUARE(x) = x*x）

---

### Day 7: 第一周总结 - 实现简单内存表

**学习目标：** 综合运用第一周知识，实现一个简单的内存表和查询

#### 7.1 回顾第一周内容

- **Day 1:** DuckDB整体架构
- **Day 2:** 类型系统和向量化执行
- **Day 3:** Vector和DataChunk数据结构
- **Day 4:** 内存管理和BufferManager
- **Day 5:** SQL解析和AST
- **Day 6:** 表达式系统

#### 7.2 实践项目：SimpleTable

我们将实现一个简化版的内存表，支持：
- 插入数据
- 全表扫描
- 简单过滤（WHERE子句）

```cpp
// simple_table.hpp
#pragma once

#include "duckdb/common/types/data_chunk.hpp"
#include "duckdb/common/types/value.hpp"

namespace duckdb {

class SimpleTable {
public:
    SimpleTable(vector<LogicalType> types, vector<string> names);

    // 插入一行数据
    void Insert(vector<Value> values);

    // 扫描所有数据
    void Scan(DataChunk &result);

    // 带过滤的扫描
    void ScanWithFilter(DataChunk &result,
                        std::function<bool(DataChunk&, idx_t)> filter);

private:
    vector<LogicalType> types;
    vector<string> column_names;
    vector<DataChunk> chunks;  // 存储数据的chunks
    idx_t current_row_count;
};

} // namespace duckdb
```

```cpp
// simple_table.cpp
#include "simple_table.hpp"

namespace duckdb {

SimpleTable::SimpleTable(vector<LogicalType> types_p, vector<string> names_p)
    : types(std::move(types_p)), column_names(std::move(names_p)),
      current_row_count(0) {
    // 创建第一个chunk
    DataChunk chunk;
    chunk.Initialize(Allocator::DefaultAllocator(), types);
    chunks.push_back(std::move(chunk));
}

void SimpleTable::Insert(vector<Value> values) {
    D_ASSERT(values.size() == types.size());

    auto &current_chunk = chunks.back();

    // 如果当前chunk已满，创建新chunk
    if (current_chunk.size() >= STANDARD_VECTOR_SIZE) {
        DataChunk new_chunk;
        new_chunk.Initialize(Allocator::DefaultAllocator(), types);
        chunks.push_back(std::move(new_chunk));
        current_chunk = chunks.back();
    }

    // 插入值
    idx_t row_idx = current_chunk.size();
    for (idx_t col = 0; col < values.size(); col++) {
        current_chunk.SetValue(col, row_idx, values[col]);
    }

    current_chunk.SetCardinality(current_chunk.size() + 1);
    current_row_count++;
}

void SimpleTable::Scan(DataChunk &result) {
    result.Reset();

    // 简化版：返回第一个chunk的数据
    if (!chunks.empty()) {
        result.Reference(chunks[0]);
    }
}

void SimpleTable::ScanWithFilter(
    DataChunk &result,
    std::function<bool(DataChunk&, idx_t)> filter) {

    result.Initialize(Allocator::DefaultAllocator(), types);

    SelectionVector sel(STANDARD_VECTOR_SIZE);
    idx_t approved_count = 0;

    for (auto &chunk : chunks) {
        // 应用过滤器
        for (idx_t i = 0; i < chunk.size(); i++) {
            if (filter(chunk, i)) {
                sel.set_index(approved_count++, i);
            }
        }

        // 使用SelectionVector切片数据
        if (approved_count > 0) {
            for (idx_t col = 0; col < types.size(); col++) {
                result.data[col].Slice(chunk.data[col], sel, approved_count);
            }
            result.SetCardinality(approved_count);
            break;  // 简化版：只处理第一个chunk
        }
    }
}

} // namespace duckdb
```

#### 7.3 使用示例

```cpp
#include "simple_table.hpp"

void TestSimpleTable() {
    using namespace duckdb;

    // 1. 创建表 (id INT, name VARCHAR, score DOUBLE)
    vector<LogicalType> types = {
        LogicalType::INTEGER,
        LogicalType::VARCHAR,
        LogicalType::DOUBLE
    };
    vector<string> names = {"id", "name", "score"};

    SimpleTable table(types, names);

    // 2. 插入数据
    table.Insert({Value::INTEGER(1), Value("Alice"), Value::DOUBLE(95.5)});
    table.Insert({Value::INTEGER(2), Value("Bob"), Value::DOUBLE(87.3)});
    table.Insert({Value::INTEGER(3), Value("Charlie"), Value::DOUBLE(92.1)});
    table.Insert({Value::INTEGER(4), Value("Diana"), Value::DOUBLE(78.5)});

    // 3. 全表扫描
    DataChunk result;
    table.Scan(result);

    printf("Full scan:\n");
    for (idx_t row = 0; row < result.size(); row++) {
        auto id = result.GetValue(0, row);
        auto name = result.GetValue(1, row);
        auto score = result.GetValue(2, row);
        printf("  %s, %s, %s\n",
               id.ToString().c_str(),
               name.ToString().c_str(),
               score.ToString().c_str());
    }

    // 4. 带过滤的扫描 (score > 90)
    DataChunk filtered_result;
    table.ScanWithFilter(filtered_result, [](DataChunk &chunk, idx_t row) {
        auto score = chunk.GetValue(2, row);
        return score.GetValue<double>() > 90.0;
    });

    printf("\nFiltered scan (score > 90):\n");
    for (idx_t row = 0; row < filtered_result.size(); row++) {
        auto id = filtered_result.GetValue(0, row);
        auto name = filtered_result.GetValue(1, row);
        auto score = filtered_result.GetValue(2, row);
        printf("  %s, %s, %s\n",
               id.ToString().c_str(),
               name.ToString().c_str(),
               score.ToString().c_str());
    }
}
```

#### 7.4 扩展练习

在理解上述代码后，尝试以下扩展：

1. **支持多个chunk的扫描**
   - 当前实现只扫描第一个chunk
   - 修改为扫描所有chunks并合并结果

2. **实现投影（Projection）**
   - 只返回选定的列
   - `ScanWithProjection(result, {0, 2})` 只返回id和score

3. **实现简单聚合**
   - 计算COUNT、SUM、AVG
   - `Aggregate("score", AggregateType::AVG)`

4. **支持多个过滤条件**
   - AND组合多个条件
   - `score > 90 AND id < 3`

#### 7.5 第一周知识点总结

| 主题 | 核心概念 | 关键文件 |
|------|----------|----------|
| 架构 | Parser→Planner→Optimizer→Executor | `src/README.md` |
| 类型 | LogicalType, PhysicalType | `src/common/types.hpp` |
| 数据 | Vector, DataChunk, ValidityMask | `src/common/types/vector.hpp` |
| 内存 | BufferManager, Allocator, Arena | `src/storage/buffer_manager.hpp` |
| 解析 | AST, SQLStatement, TableRef | `src/parser/` |
| 表达式 | ParsedExpression, BoundExpression | `src/planner/expression/` |

**下周预告：**
第二周我们将深入查询处理流程，学习LogicalOperator和PhysicalOperator的设计，以及如何实现Scan、Filter、Join、Aggregation等核心算子。

---

## 第二周：查询处理与算子实现

### Day 8: Binder与符号解析

**学习目标：** 深入理解Binder如何将ParsedExpression转换为BoundExpression

#### 8.1 Binder的职责

Binder是连接Parser和Planner的桥梁：

```
ParsedExpression (符号引用)
        ↓ Binder.Bind()
BoundExpression (完全解析)
```

**Binder的主要任务：**
1. **符号解析** - 将列名解析为具体的表和列索引
2. **类型推导** - 确定表达式的返回类型
3. **函数绑定** - 选择合适的函数重载
4. **语义检查** - 验证查询的合法性
5. **作用域管理** - 处理子查询、CTE等作用域

#### 8.2 BindContext - 绑定上下文

```cpp
// src/planner/binder/bind_context.hpp
class BindContext {
public:
    // 当前作用域的表绑定
    case_insensitive_map_t<unique_ptr<Binding>> bindings;

    // 父作用域（用于子查询）
    BindContext *parent;

    // 添加表绑定
    void AddTable(const string &alias, unique_ptr<Binding> binding);

    // 查找列
    BindResult GetColumn(const string &column_name, const string &table_name = "");
};
```

**Binding** - 表绑定信息
```cpp
class Binding {
public:
    string alias;                    // 表别名
    vector<string> column_names;     // 列名
    vector<LogicalType> column_types;// 列类型
    idx_t index;                     // 表索引

    virtual BindResult Bind(const string &column_name);
};

class TableBinding : public Binding {
    // 绑定到实际表
};

class SubqueryBinding : public Binding {
    // 绑定到子查询结果
};
```

#### 8.3 绑定SELECT语句

```cpp
// src/planner/binder/statement/bind_select.cpp

BoundStatement Binder::Bind(SelectStatement &stmt) {
    // 1. 绑定FROM子句
    auto from_binding = BindFrom(stmt.node->from_table);

    // 2. 绑定WHERE子句
    if (stmt.node->where_clause) {
        stmt.node->where_clause = Bind(stmt.node->where_clause);
    }

    // 3. 绑定SELECT列表
    vector<unique_ptr<Expression>> select_list;
    for (auto &expr : stmt.node->select_list) {
        select_list.push_back(Bind(expr));
    }

    // 4. 绑定GROUP BY
    vector<unique_ptr<Expression>> groups;
    for (auto &group : stmt.node->groups) {
        groups.push_back(Bind(group));
    }

    // 5. 绑定HAVING
    if (stmt.node->having) {
        stmt.node->having = Bind(stmt.node->having);
    }

    // 6. 创建LogicalOperator树
    return CreatePlan(stmt);
}
```

#### 8.4 列引用解析

```cpp
// src/planner/binder/expression/bind_columnref_expression.cpp

BindResult Binder::BindColumnRef(ColumnRefExpression &colref) {
    // 情况1: 单个列名 "column"
    if (colref.IsQualified() == false) {
        // 在当前BindContext查找
        auto result = bind_context.GetColumn(colref.column_names.back());
        if (result.HasError()) {
            // 在父作用域查找（correlated subquery）
            if (parent_binder) {
                return parent_binder->BindColumnRef(colref);
            }
            throw BinderException("Column \"%s\" not found", colref.column_names.back());
        }
        return result;
    }

    // 情况2: 限定列名 "table.column"
    string table_name = colref.column_names[0];
    string column_name = colref.column_names[1];

    // 查找表绑定
    auto table_binding = bind_context.GetBinding(table_name);
    if (!table_binding) {
        throw BinderException("Table \"%s\" not found", table_name);
    }

    // 在表中查找列
    return table_binding->Bind(column_name);
}

BindResult Binding::Bind(const string &column_name) {
    // 在列列表中查找
    for (idx_t i = 0; i < column_names.size(); i++) {
        if (StringUtil::CIEquals(column_names[i], column_name)) {
            // 创建BoundColumnRefExpression
            auto result = make_unique<BoundColumnRefExpression>(
                column_name,
                column_types[i],
                ColumnBinding(index, i)  // (table_index, column_index)
            );
            return BindResult(std::move(result));
        }
    }

    return BindResult("Column \"" + column_name + "\" not found in table");
}
```

#### 8.5 函数绑定和重载解析

```cpp
// src/planner/binder/expression/bind_function_expression.cpp

BindResult Binder::BindFunction(FunctionExpression &function) {
    // 1. 绑定所有参数
    vector<unique_ptr<Expression>> children;
    vector<LogicalType> arguments;

    for (auto &child : function.children) {
        auto bound_child = Bind(child);
        arguments.push_back(bound_child->return_type);
        children.push_back(std::move(bound_child));
    }

    // 2. 查找函数
    auto &catalog = Catalog::GetSystemCatalog(context);
    auto func_entry = catalog.GetEntry<ScalarFunctionCatalogEntry>(
        context, DEFAULT_SCHEMA, function.function_name);

    if (!func_entry) {
        throw BinderException("Function \"%s\" not found", function.function_name);
    }

    // 3. 选择最佳重载
    string error;
    auto best_function = func_entry->functions.GetFunctionByArguments(
        arguments, error);

    if (!best_function) {
        throw BinderException("No matching function for %s(%s): %s",
            function.function_name,
            StringUtil::Join(arguments, ", "),
            error);
    }

    // 4. 插入类型转换（如果需要）
    for (idx_t i = 0; i < children.size(); i++) {
        if (children[i]->return_type != best_function->arguments[i]) {
            children[i] = BoundCastExpression::AddCastToType(
                std::move(children[i]),
                best_function->arguments[i]
            );
        }
    }

    // 5. 调用bind函数
    unique_ptr<FunctionData> bind_data;
    if (best_function->bind) {
        bind_data = best_function->bind(context, *best_function, children);
    }

    // 6. 创建BoundFunctionExpression
    return BindResult(make_unique<BoundFunctionExpression>(
        best_function->return_type,
        *best_function,
        std::move(children),
        std::move(bind_data)
    ));
}
```

**函数重载选择算法：**
```cpp
// 简化版本
Function* GetFunctionByArguments(vector<LogicalType> &arguments) {
    Function *best_match = nullptr;
    int best_score = -1;

    for (auto &candidate : overloads) {
        if (candidate.arguments.size() != arguments.size()) {
            continue;  // 参数数量不匹配
        }

        int score = 0;
        bool can_cast = true;

        for (idx_t i = 0; i < arguments.size(); i++) {
            if (arguments[i] == candidate.arguments[i]) {
                score += 100;  // 精确匹配
            } else if (CanCast(arguments[i], candidate.arguments[i])) {
                score += 50;   // 需要隐式转换
            } else {
                can_cast = false;
                break;
            }
        }

        if (can_cast && score > best_score) {
            best_score = score;
            best_match = &candidate;
        }
    }

    return best_match;
}
```

#### 8.6 聚合函数绑定

```cpp
// src/planner/binder/expression/bind_aggregate_expression.cpp

BindResult Binder::BindAggregate(FunctionExpression &aggr_function) {
    // 聚合函数不能嵌套
    if (inside_aggregate) {
        throw BinderException("Aggregate function calls cannot be nested");
    }

    // 标记进入聚合上下文
    inside_aggregate = true;

    // 绑定参数
    vector<unique_ptr<Expression>> children;
    for (auto &child : aggr_function.children) {
        children.push_back(Bind(child));
    }

    inside_aggregate = false;

    // 查找聚合函数
    auto &catalog = Catalog::GetSystemCatalog(context);
    auto aggr_entry = catalog.GetEntry<AggregateFunctionCatalogEntry>(
        context, DEFAULT_SCHEMA, aggr_function.function_name);

    // ... 类似标量函数的绑定过程

    return BindResult(make_unique<BoundAggregateExpression>(
        best_function->return_type,
        *best_function,
        std::move(children),
        std::move(bind_data),
        aggr_function.distinct ? AggregateType::DISTINCT : AggregateType::NON_DISTINCT
    ));
}
```

**实践任务：**
1. 阅读 `src/planner/binder/bind_context.cpp`
2. 跟踪一个SELECT语句的绑定过程
3. 理解函数重载选择算法

---

### Day 9: 逻辑计划 - LogicalOperator

**学习目标：** 理解逻辑计划的结构和各种LogicalOperator的作用

#### 9.1 LogicalOperator基类

```cpp
// src/include/duckdb/planner/logical_operator.hpp
class LogicalOperator {
public:
    LogicalOperatorType type;
    vector<unique_ptr<LogicalOperator>> children;
    vector<unique_ptr<Expression>> expressions;

    idx_t estimated_cardinality;

    virtual vector<ColumnBinding> GetColumnBindings() = 0;
    virtual string ToString() const;
};

enum class LogicalOperatorType : uint8_t {
    LOGICAL_GET,              // 表扫描
    LOGICAL_PROJECTION,       // 投影
    LOGICAL_FILTER,           // 过滤
    LOGICAL_AGGREGATE_AND_GROUP_BY,  // 聚合
    LOGICAL_COMPARISON_JOIN,  // JOIN
    LOGICAL_UNION,            // UNION
    LOGICAL_ORDER_BY,         // ORDER BY
    LOGICAL_LIMIT,            // LIMIT
    LOGICAL_INSERT,           // INSERT
    // ... 更多类型
};
```

#### 9.2 主要LogicalOperator类型

**1. LogicalGet** - 表扫描
```cpp
class LogicalGet : public LogicalOperator {
public:
    TableCatalogEntry *table;       // 扫描的表
    idx_t table_index;              // 表索引
    vector<idx_t> column_ids;       // 要读取的列
    vector<LogicalType> returned_types;

    // 可选的表过滤器（pushdown的谓词）
    unique_ptr<TableFilterSet> table_filters;
};
```

**逻辑计划示例：** `SELECT * FROM students`
```
LogicalGet
├── table: students
├── table_index: 0
└── column_ids: [0, 1, 2]  (id, name, score)
```

**2. LogicalFilter** - 过滤
```cpp
class LogicalFilter : public LogicalOperator {
public:
    vector<unique_ptr<Expression>> expressions;  // WHERE条件

    // 一个child: 被过滤的operator
};
```

**逻辑计划示例：** `SELECT * FROM students WHERE age > 18`
```
LogicalFilter
├── expressions: [age > 18]
└── child:
    └── LogicalGet(students)
```

**3. LogicalProjection** - 投影
```cpp
class LogicalProjection : public LogicalOperator {
public:
    vector<unique_ptr<Expression>> expressions;  // SELECT列表
    idx_t table_index;
};
```

**逻辑计划示例：** `SELECT name, score * 2 FROM students`
```
LogicalProjection
├── expressions: [name, score * 2]
└── child:
    └── LogicalGet(students)
```

**4. LogicalComparisonJoin** - JOIN
```cpp
class LogicalComparisonJoin : public LogicalOperator {
public:
    JoinType join_type;  // INNER, LEFT, RIGHT, FULL, SEMI, ANTI
    vector<JoinCondition> conditions;

    // 两个children: left和right
};

struct JoinCondition {
    unique_ptr<Expression> left;
    unique_ptr<Expression> right;
    ExpressionType comparison;  // =, <, >, ...
};
```

**逻辑计划示例：** `SELECT * FROM a JOIN b ON a.id = b.id`
```
LogicalComparisonJoin (INNER)
├── conditions: [a.id = b.id]
├── left:
│   └── LogicalGet(a)
└── right:
    └── LogicalGet(b)
```

**5. LogicalAggregate** - 聚合
```cpp
class LogicalAggregate : public LogicalOperator {
public:
    vector<unique_ptr<Expression>> groups;       // GROUP BY
    idx_t group_index;
    vector<unique_ptr<Expression>> expressions;  // 聚合函数
    idx_t aggregate_index;
};
```

**逻辑计划示例：** `SELECT name, AVG(score) FROM students GROUP BY name`
```
LogicalAggregate
├── groups: [name]
├── expressions: [AVG(score)]
└── child:
    └── LogicalGet(students)
```

#### 9.3 逻辑计划生成过程

```cpp
// src/planner/planner.cpp

unique_ptr<LogicalOperator> Planner::CreatePlan(SelectStatement &stmt) {
    // 1. 绑定语句
    auto bound_stmt = binder.Bind(stmt);

    // 2. 从底向上构建逻辑计划
    unique_ptr<LogicalOperator> root;

    // 2.1 FROM子句 -> LogicalGet或LogicalJoin
    root = CreatePlanForTableRef(stmt.node->from_table);

    // 2.2 WHERE子句 -> LogicalFilter
    if (stmt.node->where_clause) {
        auto filter = make_unique<LogicalFilter>();
        filter->expressions.push_back(std::move(stmt.node->where_clause));
        filter->children.push_back(std::move(root));
        root = std::move(filter);
    }

    // 2.3 GROUP BY -> LogicalAggregate
    if (!stmt.node->groups.empty()) {
        auto aggregate = make_unique<LogicalAggregate>();
        aggregate->groups = std::move(stmt.node->groups);
        // 从SELECT列表中提取聚合函数
        ExtractAggregates(stmt.node->select_list, aggregate->expressions);
        aggregate->children.push_back(std::move(root));
        root = std::move(aggregate);
    }

    // 2.4 HAVING -> LogicalFilter
    if (stmt.node->having) {
        auto filter = make_unique<LogicalFilter>();
        filter->expressions.push_back(std::move(stmt.node->having));
        filter->children.push_back(std::move(root));
        root = std::move(filter);
    }

    // 2.5 SELECT -> LogicalProjection
    auto projection = make_unique<LogicalProjection>();
    projection->expressions = std::move(stmt.node->select_list);
    projection->children.push_back(std::move(root));
    root = std::move(projection);

    // 2.6 ORDER BY -> LogicalOrder
    if (!stmt.orders.empty()) {
        auto order = make_unique<LogicalOrder>();
        order->orders = std::move(stmt.orders);
        order->children.push_back(std::move(root));
        root = std::move(order);
    }

    // 2.7 LIMIT -> LogicalLimit
    if (stmt.limit) {
        auto limit = make_unique<LogicalLimit>();
        limit->limit_val = stmt.limit;
        limit->offset_val = stmt.offset;
        limit->children.push_back(std::move(root));
        root = std::move(limit);
    }

    return root;
}
```

#### 9.4 复杂查询的逻辑计划

**示例SQL：**
```sql
SELECT department, AVG(salary) as avg_salary
FROM employees
WHERE age > 25
GROUP BY department
HAVING AVG(salary) > 50000
ORDER BY avg_salary DESC
LIMIT 10;
```

**对应的逻辑计划：**
```
LogicalLimit
├── limit: 10
└── child:
    LogicalOrder
    ├── orders: [avg_salary DESC]
    └── child:
        LogicalProjection
        ├── expressions: [department, avg_salary]
        └── child:
            LogicalFilter (HAVING)
            ├── expressions: [AVG(salary) > 50000]
            └── child:
                LogicalAggregate
                ├── groups: [department]
                ├── expressions: [AVG(salary)]
                └── child:
                    LogicalFilter (WHERE)
                    ├── expressions: [age > 25]
                    └── child:
                        LogicalGet
                        └── table: employees
```

**实践任务：**
1. 阅读 `src/planner/logical_operator.hpp` 了解所有LogicalOperator类型
2. 使用DuckDB的EXPLAIN查看查询的逻辑计划
3. 手动构建一个简单的逻辑计划树

---

## Day 10: 物理计划 - PhysicalOperator

**学习目标：** 理解逻辑计划如何转换为物理执行计划

### 10.1 PhysicalOperator基类

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

    // 算子类型判断
    bool IsSource() const;      // 数据源算子
    bool IsSink() const;        // 数据汇算子
    bool IsParallel() const;    // 并行算子
};

enum class PhysicalOperatorType : uint8_t {
    PHYSICAL_TABLE_SCAN,           // 表扫描
    PHYSICAL_FILTER,               // 过滤
    PHYSICAL_PROJECTION,           // 投影
    PHYSICAL_HASH_JOIN,            // Hash Join
    PHYSICAL_ORDER_BY,             // 排序
    PHYSICAL_AGGREGATE,            // 聚合
    PHYSICAL_LIMIT,                // Limit
    // ... 更多类型
};
```

### 10.2 逻辑到物理的转换

#### 转换示例：SELECT * FROM t WHERE x > 10

```
逻辑计划：
LogicalFilter
└── LogicalGet

物理计划（经过优化器）：
PhysicalTableScan(t, filter: x > 10)
```

Filter被下推到TableScan中，避免额外的Filter算子。

### 10.3 PhysicalPlanGenerator

```cpp
// src/execution/physical_plan_generator.cpp

class PhysicalPlanGenerator {
public:
    unique_ptr<PhysicalOperator> CreatePlan(LogicalOperator &op) {
        switch (op.type) {
        case LogicalOperatorType::LOGICAL_GET:
            return CreatePlan((LogicalGet &)op);
        case LogicalOperatorType::LOGICAL_FILTER:
            return CreatePlan((LogicalFilter &)op);
        case LogicalOperatorType::LOGICAL_PROJECTION:
            return CreatePlan((LogicalProjection &)op);
        case LogicalOperatorType::LOGICAL_AGGREGATE:
            return CreatePlan((LogicalAggregate &)op);
        case LogicalOperatorType::LOGICAL_COMPARISON_JOIN:
            return CreatePlan((LogicalComparisonJoin &)op);
        // ...
        }
    }

private:
    unique_ptr<PhysicalOperator> CreatePlan(LogicalGet &op) {
        // 创建物理表扫描
        auto table_scan = make_unique<PhysicalTableScan>(
            op.table,
            op.column_ids,
            op.return_types
        );

        // 如果有表过滤器，可以下推
        if (op.table_filters) {
            table_scan->table_filters = std::move(op.table_filters);
        }

        return table_scan;
    }

    unique_ptr<PhysicalOperator> CreatePlan(LogicalFilter &op) {
        // 先创建子节点的物理计划
        auto child_plan = CreatePlan(*op.children[0]);

        // 尝试下推Filter
        if (CanPushdown(op, *child_plan)) {
            // Filter下推成功
            return child_plan;
        }

        // 创建物理Filter算子
        return make_unique<PhysicalFilter>(
            std::move(op.expressions),
            std::move(child_plan)
        );
    }
};
```

### 10.4 执行状态管理

```cpp
// 算子状态（每个线程一份）
class OperatorState {
public:
    virtual ~OperatorState() = default;
};

// 全局状态（所有线程共享）
class GlobalOperatorState {
public:
    virtual ~GlobalOperatorState() = default;
};

// 表扫描状态
class TableScanState : public OperatorState {
public:
    idx_t current_row_group;       // 当前RowGroup
    idx_t current_row;             // 当前行
    vector<column_t> scanned_columns;  // 扫描的列
};

// 聚合全局状态
class HashAggregateGlobalState : public GlobalOperatorState {
public:
    unique_ptr<HashAggregateHashTable> hash_table;  // 全局哈希表
    mutex lock;  // 保护哈希表
};
```

### 10.5 Pipeline执行模型

```cpp
// Pipeline结构
struct Pipeline {
    PhysicalOperator *source;           // 源算子
    PhysicalOperator *sink;             // 汇算子
    vector<PhysicalOperator*> operators;  // 中间算子

    void Execute(ClientContext &context) {
        // 初始化状态
        auto global_state = sink->GetGlobalState(context);
        auto state = sink->GetOperatorState(context);

        DataChunk input, output;

        // 从Source拉取数据，通过Pipeline推送到Sink
        while (source->Execute(context, input, output)) {
            // 依次通过中间算子
            for (auto op : operators) {
                op->Execute(context, input, output, *state);
                input = std::move(output);
            }

            // 推送到Sink
            sink->Execute(context, input, *global_state, *state);
        }
    }
};
```

**实践任务：**
1. 阅读 `src/execution/physical_operator.hpp`
2. 使用EXPLAIN查看查询的物理计划
3. 跟踪一个简单查询的物理计划生成过程

---

## Day 11: 算子实现 - TableScan和Filter

**学习目标：** 深入理解表扫描和过滤算子的实现

### 11.1 PhysicalTableScan实现

```cpp
// src/execution/operator/scan/physical_table_scan.cpp

class PhysicalTableScan : public PhysicalOperator {
public:
    DataTable &table;               // 要扫描的表
    vector<idx_t> column_ids;       // 要读取的列
    vector<LogicalType> types;      // 输出类型

    TableFilterSet *table_filters;  // 表过滤器（下推的WHERE）

public:
    // 初始化扫描状态
    unique_ptr<OperatorState> GetOperatorState(ExecutionContext &context) override {
        return make_unique<TableScanState>();
    }

    // 执行扫描
    OperatorResultType Execute(ExecutionContext &context,
                               DataChunk &input,
                               GlobalOperatorState &gstate,
                               OperatorState &state) override {
        auto &scan_state = (TableScanState &)state;

        // 初始化输出chunk
        DataChunk output;
        output.Initialize(Allocator::DefaultAllocator(), types);

        // 扫描RowGroups
        while (output.size() < STANDARD_VECTOR_SIZE) {
            // 检查是否需要切换到下一个RowGroup
            if (scan_state.current_row >= scan_state.row_group_count) {
                if (!MoveToNextRowGroup(scan_state)) {
                    break;  // 没有更多数据
                }
            }

            // 读取当前RowGroup
            idx_t remaining = STANDARD_VECTOR_SIZE - output.size();
            idx_t to_scan = MinValue(remaining,
                                     scan_state.row_group_count - scan_state.current_row);

            // 扫描数据
            ScanRowGroup(scan_state, output, to_scan);

            scan_state.current_row += to_scan;
        }

        // 如果没有任何数据，返回NEED_MORE_INPUT
        if (output.size() == 0) {
            return OperatorResultType::NEED_MORE_INPUT;
        }

        return OperatorResultType::HAVE_MORE_OUTPUT;
    }

private:
    bool MoveToNextRowGroup(TableScanState &state) {
        state.current_row_group++;
        if (state.current_row_group >= table.row_groups.size()) {
            return false;
        }

        // 获取RowGroup
        auto &row_group = table.row_groups[state.current_row_group];
        state.row_group_count = row_group->start + row_group->count;
        state.current_row = row_group->start;

        return true;
    }

    void ScanRowGroup(TableScanState &state, DataChunk &output, idx_t count) {
        auto &row_group = table.row_groups[state.current_row_group];

        // 逐列读取
        for (size_t col_idx = 0; col_idx < column_ids.size(); col_idx++) {
            idx_t col_id = column_ids[col_idx];
            auto &column = row_group->columns[col_id];

            // 读取数据到输出向量
            Vector &out_vec = output.data[col_idx];
            column->Scan(state.current_row, count, out_vec);
        }

        output.SetCardinality(output.size() + count);
    }
};
```

### 11.2 PhysicalFilter实现

```cpp
// src/execution/operator/filter/physical_filter.cpp

class PhysicalFilter : public PhysicalOperator {
public:
    vector<unique_ptr<Expression>> expressions;  // WHERE条件
    unique_ptr<PhysicalOperator> child;          // 子算子

public:
    unique_ptr<OperatorState> GetOperatorState(ExecutionContext &context) override {
        auto child_state = child->GetOperatorState(context);
        return make_unique<FilterState>(std::move(child_state), expressions);
    }

    OperatorResultType Execute(ExecutionContext &context,
                               DataChunk &input,
                               GlobalOperatorState &gstate,
                               OperatorState &state) override {
        auto &filter_state = (FilterState &)state;

        // 从子节点获取数据
        DataChunk child_chunk;
        auto result = child->Execute(context, input, gstate, *filter_state.child_state);

        if (result == OperatorResultType::NEED_MORE_INPUT) {
            return OperatorResultType::NEED_MORE_INPUT;
        }

        // 执行过滤条件
        SelectionVector sel(STANDARD_VECTOR_SIZE);
        idx_t approved_count = 0;

        for (auto &expr : expressions) {
            // 在chunk上执行表达式
            Vector result_vec(LogicalType::BOOLEAN);
            ExpressionExecutor::Execute(expr, child_chunk, result_vec);

            // 应用选择向量
            auto result_data = FlatVector::GetData<bool>(result_vec);
            auto &validity = result_vec.validity;

            for (idx_t i = 0; i < child_chunk.size(); i++) {
                if (validity.RowIsValid(i) && result_data[i]) {
                    sel.set_index(approved_count++, i);
                }
            }
        }

        // 构建输出chunk（使用SelectionVector切片）
        input.Reference(child_chunk);
        input.Slice(input, sel, approved_count);
        input.SetCardinality(approved_count);

        return approved_count > 0 ?
            OperatorResultType::HAVE_MORE_OUTPUT :
            OperatorResultType::NEED_MORE_INPUT;
    }
};
```

### 11.3 自适应过滤器

```cpp
// src/execution/adaptive_filter.cpp

// 多个条件时，动态调整过滤顺序以提高性能

class AdaptiveFilter {
private:
    vector<unique_ptr<Expression>> filters;
    vector<idx_t> order;  // 过滤器执行顺序
    vector<idx_t> execution_count;  // 执行次数统计
    vector<idx_t> selectivity;  // 选择性统计

public:
    void UpdateStatistics(idx_t filter_idx, idx_t input_count, idx_t output_count) {
        execution_count[filter_idx]++;
        double sel = (double)output_count / input_count;
        // 更新平均选择性
        selectivity[filter_idx] =
            (selectivity[filter_idx] * (execution_count[filter_idx] - 1) + sel) /
            execution_count[filter_idx];
    }

    void ReorderFilters() {
        // 按选择性排序（最严格的过滤器先执行）
        sort(order.begin(), order.end(),
             [this](idx_t a, idx_t b) {
                 return selectivity[a] < selectivity[b];
             });
    }
};
```

**实践任务：**
1. 阅读 `src/execution/operator/scan/physical_table_scan.cpp`
2. 阅读 `src/execution/operator/filter/physical_filter.cpp`
3. 实现一个简单的TableScan算子
4. 实现一个Filter算子，支持AND和OR条件

---

## Day 12: 算子实现 - Hash Join

**学习目标：** 理解Hash Join的实现和优化

### 12.1 Hash Join概述

Hash Join是处理等值Join最常用的算法，特别是当数据量较大无法放入内存时。

```
Hash Join三个阶段：
1. Build阶段：构建probe表（通常是较小的表）
2. Probe阶段：探测匹配
3. 收集阶段：输出结果
```

### 12.2 PhysicalHashJoin实现

```cpp
// src/execution/operator/join/physical_hash_join.cpp

class PhysicalHashJoin : public PhysicalOperator {
public:
    JoinType join_type;              // INNER, LEFT, RIGHT, FULL
    vector<JoinCondition> conditions;  // Join条件
    unique_ptr<PhysicalOperator> left;
    unique_ptr<PhysicalOperator> right;

public:
    unique_ptr<OperatorState> GetOperatorState(ExecutionContext &context) override {
        auto left_state = left->GetOperatorState(context);
        auto right_state = right->GetOperatorState(context);
        return make_unique<HashJoinState>(
            std::move(left_state),
            std::move(right_state),
            conditions
        );
    }

    OperatorResultType Execute(ExecutionContext &context,
                               DataChunk &input,
                               GlobalOperatorState &gstate,
                               OperatorState &state) override {
        auto &hash_join_state = (HashJoinState &)state;

        // 阶段1: Build哈希表
        if (!hash_join_state.build_complete) {
            BuildHashTable(context, hash_join_state);
            hash_join_state.build_complete = true;
        }

        // 阶段2: Probe哈希表
        DataChunk probe_chunk;
        while (right->Execute(context, input, gstate, *hash_join_state.right_state)) {
            // 探测哈希表
            ProbeHashTable(input, hash_join_state, probe_chunk);

            if (probe_chunk.size() > 0) {
                return OperatorResultType::HAVE_MORE_OUTPUT;
            }
        }

        return OperatorResultType::NEED_MORE_INPUT;
    }

private:
    void BuildHashTable(ExecutionContext &context, HashJoinState &state) {
        DataChunk build_chunk;
        auto &global_state = (HashJoinGlobalState &)gstate;

        // 从左表读取所有数据
        while (left->Execute(context, build_chunk, gstate, *state.left_state)) {
            // 计算Join key的哈希
            Vector hashes(LogicalType::HASH);
            VectorOperations::Hash(state.build_keys, hashes);

            // 插入哈希表
            for (idx_t i = 0; i < build_chunk.size(); i++) {
                auto hash = hashes.GetValue<hash_t>(i);
                global_state.hash_table->Insert(hash, build_chunk, i);
            }
        }
    }

    void ProbeHashTable(DataChunk &probe, HashJoinState &state, DataChunk &result) {
        // 计算probe key的哈希
        Vector hashes(LogicalType::HASH);
        VectorOperations::Hash(state.probe_keys, hashes);

        // 在哈希表中查找匹配
        for (idx_t i = 0; i < probe.size(); i++) {
            auto hash = hashes.GetValue<hash_t>(i);

            // 查找哈希表
            auto entries = state.hash_table->Find(hash);

            // 检查Join条件
            for (auto entry : entries) {
                if (CheckJoinCondition(probe, i, entry)) {
                    // 找到匹配，构造输出
                    ConstructOutput(probe, i, entry, result);
                }
            }
        }
    }
};
```

### 12.3 完善HashJoin - 多线程支持

```cpp
// 并行Hash Join

class ParallelHashJoinState : public GlobalOperatorState {
public:
    // 每个线程的本地哈希表
    vector<unique_ptr<HashAggregateHashTable>> local_hash_tables;

    // 全局哈希表（合并后）
    unique_ptr<HashAggregateHashTable> global_hash_table;

    // Radix分区（用于超大数据集）
    vector<unique_ptr<RadixPartition>> partitions;
};

void ParallelHashJoin::Execute(ExecutionContext &context,
                                DataChunk &input,
                                GlobalOperatorState &gstate,
                                OperatorState &state) {
    auto &parallel_state = (ParallelHashJoinState &)gstate;

    // 第一步：多线程构建本地哈希表
    if (!parallel_state.local_build_complete) {
        ParallelBuild(context, parallel_state);
        parallel_state.local_build_complete = true;
    }

    // 第二步：合并本地哈希表
    if (!parallel_state.global_build_complete) {
        MergeLocalHashTables(parallel_state);
        parallel_state.global_build_complete = true;
    }

    // 第三步：Probe全局哈希表
    ProbeHashTable(input, parallel_state);
}
```

### 12.4 Join类型实现

```cpp
// INNER JOIN: 只输出匹配的行
// LEFT JOIN: 左表所有行 + 匹配的右表行，不匹配用NULL填充
// RIGHT JOIN: 右表所有行 + 匹配的左表行
// FULL OUTER JOIN: 两表所有行

void HandleLeftJoin(DataChunk &left, idx_t left_idx,
                   DataChunk &result) {
    // 左表的行没有找到匹配，用NULL填充右表
    for (size_t col_idx = 0; col_idx < left.ColumnCount(); col_idx++) {
        result.data[col_idx].SetValue(result.size(),
                                       left.GetValue(col_idx, left_idx));
    }

    // 右表列设为NULL
    for (size_t col_idx = left.ColumnCount(); col_idx < result.ColumnCount(); col_idx++) {
        result.data[col_idx].SetNull(result.size());
    }

    result.SetCardinality(result.size() + 1);
}

void HandleFullJoin(DataChunk &left, DataChunk &right,
                   HashSet<idx_t> &matched_left,
                   HashSet<idx_t> &matched_right,
                   DataChunk &result) {
    // 处理左表中未匹配的行
    for (idx_t i = 0; i < left.size(); i++) {
        if (matched_left.find(i) == matched_left.end()) {
            HandleLeftJoin(left, i, result);
        }
    }

    // 处理右表中未匹配的行
    for (idx_t i = 0; i < right.size(); i++) {
        if (matched_right.find(i) == matched_right.end()) {
            HandleRightJoin(right, i, result);
        }
    }
}
```

**实践任务：**
1. 阅读 `src/execution/operator/join/physical_hash_join.cpp`
2. 实现一个简单的Hash Join（仅支持INNER JOIN）
3. 添加LEFT JOIN支持
4. 思考：如何处理超大数据集（无法放入内存）？

---

## Day 13: 算子实现 - Aggregation

**学习目标：** 理解聚合算子的实现，包括GROUP BY和聚合函数

### 13.1 聚合类型

```sql
-- 无GROUP BY（全局聚合）
SELECT COUNT(*), SUM(amount) FROM orders;

-- 有GROUP BY（分组聚合）
SELECT customer_id, COUNT(*), SUM(amount)
FROM orders
GROUP BY customer_id;

-- 多个GROUP BY列
SELECT customer_id, product_id, SUM(quantity)
FROM orders
GROUP BY customer_id, product_id;
```

### 13.2 PhysicalHashAggregate实现

```cpp
// src/execution/operator/aggregate/physical_hash_aggregate.cpp

class PhysicalHashAggregate : public PhysicalOperator {
public:
    vector<unique_ptr<Expression>> groups;       // GROUP BY列
    vector<unique_ptr<Expression>> aggregates;   // 聚合函数
    vector<LogicalType> group_types;             // GROUP BY列类型
    vector<LogicalType> aggregate_types;         // 聚合函数类型

public:
    unique_ptr<OperatorState> GetOperatorState(ExecutionContext &context) override {
        return make_unique<HashAggregateState>(groups, aggregates);
    }

    OperatorResultType Execute(ExecutionContext &context,
                               DataChunk &input,
                               GlobalOperatorState &gstate,
                               OperatorState &state) override {
        auto &agg_state = (HashAggregateState &)state;
        auto &global_state = (HashAggregateGlobalState &)gstate;

        // 第一阶段：从输入聚合
        DataChunk input_chunk;
        while (child->Execute(context, input_chunk, gstate, *agg_state.child_state)) {
            AggregateInput(input_chunk, agg_state);
        }

        // 第二阶段：扫描哈希表，输出结果
        DataChunk result;
        result.Initialize(Allocator::DefaultAllocator(),
                        group_types + aggregate_types);

        idx_t output_count = 0;
        ScanHashTable(global_state.hash_table, result, output_count);

        if (output_count == 0) {
            return OperatorResultType::NEED_MORE_INPUT;
        }

        result.SetCardinality(output_count);
        return OperatorResultType::HAVE_MORE_OUTPUT;
    }

private:
    void AggregateInput(DataChunk &input, HashAggregateState &state) {
        // 计算GROUP BY列的哈希
        Vector group_hashes(LogicalType::HASH);
        VectorOperations::Hash(input.data, group_hashes, input.size());

        // 逐行聚合
        for (idx_t i = 0; i < input.size(); i++) {
            auto hash = group_hashes.GetValue<hash_t>(i);

            // 在哈希表中查找/创建组
            auto *group_state = state.hash_table->FindOrCreateGroup(hash);

            // 更新聚合函数
            for (size_t agg_idx = 0; agg_idx < aggregates.size(); agg_idx++) {
                auto &aggregate = aggregates[agg_idx];
                auto &agg_func = aggregate->function;

                // 更新聚合状态
                agg_func.update(input, i, group_state->aggregates[agg_idx]);
            }
        }
    }

    void ScanHashTable(HashAggregateHashTable *ht,
                       DataChunk &result,
                       idx_t &output_count) {
        idx_t current_idx = 0;

        for (auto &entry : *ht) {
            if (current_idx >= STANDARD_VECTOR_SIZE) {
                break;
            }

            // 输出GROUP BY列
            for (size_t group_idx = 0; group_idx < groups.size(); group_idx++) {
                result.data[group_idx].SetValue(current_idx,
                                                  entry.group_values[group_idx]);
            }

            // 最终化聚合函数
            for (size_t agg_idx = 0; agg_idx < aggregates.size(); agg_idx++) {
                auto &aggregate = aggregates[agg_idx];
                auto &agg_func = aggregate->function;

                // 执行finalize
                agg_func.finalize(entry.aggregates[agg_idx],
                                  result.data[groups.size() + agg_idx],
                                  current_idx);
            }

            current_idx++;
        }
    }
};
```

### 13.3 常用聚合函数实现

```cpp
// COUNT函数
struct CountAggregateState {
    idx_t count;
};

void CountUpdate(Vector &input, idx_t count, CountAggregateState &state) {
    // 计算非NULL值数量
    idx_t non_null_count = 0;
    auto &validity = input.validity;

    for (idx_t i = 0; i < count; i++) {
        if (validity.RowIsValid(i)) {
            non_null_count++;
        }
    }

    state.count += non_null_count;
}

void CountFinalize(CountAggregateState &state, Vector &result, idx_t index) {
    result.SetValue(index, Value::BIGINT(state.count));
}

// SUM函数
struct SumAggregateState {
    double sum;
    bool is_empty;
};

void SumUpdate(Vector &input, idx_t count, SumAggregateState &state) {
    auto data = FlatVector::GetData<double>(input);
    auto &validity = input.validity;

    for (idx_t i = 0; i < count; i++) {
        if (validity.RowIsValid(i)) {
            state.sum += data[i];
            state.is_empty = false;
        }
    }
}

// AVG函数
struct AvgAggregateState {
    double sum;
    idx_t count;
};

void AvgUpdate(Vector &input, idx_t count, AvgAggregateState &state) {
    auto data = FlatVector::GetData<double>(input);
    auto &validity = input.validity;

    for (idx_t i = 0; i < count; i++) {
        if (validity.RowIsValid(i)) {
            state.sum += data[i];
            state.count++;
        }
    }
}

void AvgFinalize(AvgAggregateState &state, Vector &result, idx_t index) {
    if (state.count == 0) {
        result.SetNull(index);
    } else {
        result.SetValue(index, Value::DOUBLE(state.sum / state.count));
    }
}

// MIN/MAX函数
template <class T>
struct MinMaxAggregateState {
    T value;
    bool is_empty;
};

template <class T>
void MinUpdate(Vector &input, idx_t count, MinMaxAggregateState<T> &state) {
    auto data = FlatVector::GetData<T>(input);
    auto &validity = input.validity;

    for (idx_t i = 0; i < count; i++) {
        if (validity.RowIsValid(i)) {
            if (state.is_empty || data[i] < state.value) {
                state.value = data[i];
                state.is_empty = false;
            }
        }
    }
}
```

### 13.4 两阶段聚合（并行优化）

```cpp
// 两阶段聚合：本地聚合 + 全局聚合
// 适用于并行执行场景

// 第一阶段：每个线程本地聚合
class LocalAggregateState {
public:
    HashAggregateHashTable local_hash_table;
};

void ThreadLocalAggregation(DataChunk &input, LocalAggregateState &state) {
    // 在本地哈希表中聚合
    AggregateInput(input, state.local_hash_table);
}

// 第二阶段：合并本地聚合结果
void MergeLocalAggregates(vector<LocalAggregateState*> &local_states,
                          GlobalAggregateState &global_state) {
    for (auto local_state : local_states) {
        // 遍历本地哈希表
        for (auto &entry : *local_state->local_hash_table) {
            // 在全局哈希表中查找/创建组
            auto *global_group = global_state.hash_table->FindOrCreateGroup(
                entry.group_hash);

            // 合并聚合状态
            for (size_t agg_idx = 0; agg_idx < aggregates.size(); agg_idx++) {
                aggregates[agg_idx]->function.combine(
                    entry.aggregates[agg_idx],
                    global_group->aggregates[agg_idx]
                );
            }
        }
    }
}
```

**实践任务：**
1. 阅读 `src/execution/operator/aggregate/physical_hash_aggregate.cpp`
2. 实现COUNT和SUM聚合函数
3. 添加对GROUP BY多个列的支持
4. 实现两阶段聚合优化

---

## Day 14: 第二周总结 - 实现基本查询执行

**学习目标：** 综合运用第二周知识，实现一个简单的查询执行引擎

### 14.1 项目目标

实现一个支持以下功能的查询引擎：
- 表扫描（TableScan）
- 过滤（Filter）
- 投影（Projection）
- 简单Join（Hash Join）
- 基本聚合（COUNT, SUM）

### 14.2 完整示例：实现简单查询引擎

```cpp
// simple_query_engine.hpp

#pragma once
#include "duckdb/common/types/data_chunk.hpp"
#include "duckdb/common/types/value.hpp"
#include "duckdb/planner/expression.hpp"
#include <memory>
#include <vector>
#include <functional>

namespace duckdb {

// 简单的内存表
class SimpleTable {
public:
    SimpleTable(vector<LogicalType> types_p, vector<string> names_p)
        : types(std::move(types_p)), column_names(std::move(names_p)) {
        // 创建第一个chunk
        DataChunk chunk;
        chunk.Initialize(Allocator::DefaultAllocator(), types);
        chunks.push_back(std::move(chunk));
    }

    // 插入数据
    void Insert(vector<Value> values) {
        D_ASSERT(values.size() == types.size());

        auto &current_chunk = chunks.back();

        // 如果当前chunk已满，创建新chunk
        if (current_chunk.size() >= STANDARD_VECTOR_SIZE) {
            DataChunk new_chunk;
            new_chunk.Initialize(Allocator::DefaultAllocator(), types);
            chunks.push_back(std::move(new_chunk));
        }

        // 插入值
        idx_t row_idx = current_chunk.size();
        for (idx_t col = 0; col < values.size(); col++) {
            current_chunk.SetValue(col, row_idx, values[col]);
        }

        current_chunk.SetCardinality(current_chunk.size() + 1);
    }

    // 全表扫描
    void Scan(DataChunk &result) {
        result.Reset();

        if (!chunks.empty()) {
            result.Reference(chunks[0]);
        }
    }

    vector<LogicalType> types;
    vector<string> column_names;
    vector<DataChunk> chunks;
};

// 物理算子基类
class SimpleOperator {
public:
    virtual ~SimpleOperator() = default;

    virtual bool GetNext(DataChunk &result) = 0;
    virtual void Reset() = 0;
};

// 表扫描算子
class SimpleTableScan : public SimpleOperator {
private:
    SimpleTable &table;
    idx_t current_chunk;
    bool initialized;

public:
    SimpleTableScan(SimpleTable &t) : table(t), current_chunk(0), initialized(false) {
        result.Initialize(Allocator::DefaultAllocator(), table.types);
    }

    bool GetNext(DataChunk &result) override {
        if (!initialized) {
            current_chunk = 0;
            initialized = true;
        }

        if (current_chunk >= table.chunks.size()) {
            return false;
        }

        result.Reference(table.chunks[current_chunk]);
        current_chunk++;
        return true;
    }

    void Reset() override {
        current_chunk = 0;
        initialized = false;
    }

private:
    DataChunk result;
};

// 过滤算子
class SimpleFilter : public SimpleOperator {
private:
    unique_ptr<SimpleOperator> child;
    std::function<bool(DataChunk&, idx_t)> filter_fn;
    DataChunk result;

public:
    SimpleFilter(unique_ptr<SimpleOperator> child_p,
                std::function<bool(DataChunk&, idx_t)> fn)
        : child(std::move(child_p)), filter_fn(fn) {}

    bool GetNext(DataChunk &result) override {
        result.Reset();

        SelectionVector sel(STANDARD_VECTOR_SIZE);
        idx_t approved_count = 0;

        while (approved_count == 0) {
            if (!child->GetNext(temp_chunk)) {
                if (approved_count == 0) {
                    return false;
                }
                break;
            }

            // 应用过滤器
            for (idx_t i = 0; i < temp_chunk.size(); i++) {
                if (filter_fn(temp_chunk, i)) {
                    sel.set_index(approved_count++, i);
                }
            }
        }

        // 使用SelectionVector切片
        result.Initialize(Allocator::DefaultAllocator(), temp_chunk.ColumnCount());
        for (idx_t col = 0; col < temp_chunk.ColumnCount(); col++) {
            result.data[col].Slice(temp_chunk.data[col], sel, approved_count);
        }
        result.SetCardinality(approved_count);

        return true;
    }

    void Reset() override {
        child->Reset();
    }

private:
    DataChunk temp_chunk;
};

// 投影算子
class SimpleProjection : public SimpleOperator {
private:
    unique_ptr<SimpleOperator> child;
    vector<idx_t> projection_list;
    DataChunk result;

public:
    SimpleProjection(unique_ptr<SimpleOperator> child_p,
                    vector<idx_t> proj_list)
        : child(std::move(child_p)),
          projection_list(std::move(proj_list)) {}

    bool GetNext(DataChunk &result) override {
        if (!child->GetNext(temp_chunk)) {
            return false;
        }

        // 初始化result（仅首次）
        if (result.ColumnCount() == 0) {
            vector<LogicalType> types;
            for (auto col_idx : projection_list) {
                types.push_back(temp_chunk.data[col_idx].type);
            }
            result.Initialize(Allocator::DefaultAllocator(), types);
        }

        // 复制选定的列
        for (size_t i = 0; i < projection_list.size(); i++) {
            auto col_idx = projection_list[i];
            result.data[i].Reference(temp_chunk.data[col_idx]);
        }

        result.SetCardinality(temp_chunk.size());
        return true;
    }

    void Reset() override {
        child->Reset();
    }

private:
    DataChunk temp_chunk;
};

// Hash Join算子
class SimpleHashJoin : public SimpleOperator {
private:
    unique_ptr<SimpleOperator> left;
    unique_ptr<SimpleOperator> right;
    idx_t left_join_col;
    idx_t right_join_col;
    DataChunk result;

    // 哈希表
    struct HashEntry {
        Value key;
        DataChunk data;
        idx_t row_idx;
    };
    vector<vector<HashEntry>> hash_table;

    bool build_complete;

public:
    SimpleHashJoin(unique_ptr<SimpleOperator> left_p,
                   unique_ptr<SimpleOperator> right_p,
                   idx_t left_col,
                   idx_t right_col)
        : left(std::move(left_p)),
          right(std::move(right_p)),
          left_join_col(left_col),
          right_join_col(right_col),
          build_complete(false) {
        hash_table.resize(1024);  // 简化：固定大小
    }

    bool GetNext(DataChunk &result) override {
        // Phase 1: Build哈希表
        if (!build_complete) {
            BuildHashTable();
            build_complete = true;
        }

        // Phase 2: Probe哈希表
        DataChunk probe_chunk;
        if (!right->GetNext(probe_chunk)) {
            return false;
        }

        // 初始化result
        if (result.ColumnCount() == 0) {
            vector<LogicalType> types;
            // 从left添加列
            for (idx_t col = 0; col < left_types.size(); col++) {
                types.push_back(left_types[col]);
            }
            // 从right添加列（排除join列）
            for (idx_t col = 0; col < right_types.size(); col++) {
                if (col != right_join_col) {
                    types.push_back(right_types[col]);
                }
            }
            result.Initialize(Allocator::DefaultAllocator(), types);
        }

        // Probe并构造输出
        idx_t output_count = 0;
        for (idx_t i = 0; i < probe_chunk.size(); i++) {
            auto join_key = probe_chunk.GetValue(right_join_col, i);
            auto hash = std::hash<Value>()(join_key) % hash_table.size();

            // 在哈希表中查找匹配
            for (auto &entry : hash_table[hash]) {
                if (entry.key == join_key) {
                    // 构造输出行
                    for (idx_t col = 0; col < left_types.size(); col++) {
                        result.data[col].SetValue(output_count,
                                                   entry.data.GetValue(col, entry.row_idx));
                    }

                    idx_t result_col = left_types.size();
                    for (idx_t col = 0; col < right_types.size(); col++) {
                        if (col != right_join_col) {
                            result.data[result_col++].SetValue(output_count,
                                                               probe_chunk.GetValue(col, i));
                        }
                    }

                    output_count++;
                }
            }
        }

        if (output_count == 0) {
            return GetNext(result);  // 递归查找下一个匹配的chunk
        }

        result.SetCardinality(output_count);
        return true;
    }

    void Reset() override {
        left->Reset();
        right->Reset();
        build_complete = false;
    }

private:
    void BuildHashTable() {
        DataChunk build_chunk;
        while (left->GetNext(build_chunk)) {
            for (idx_t i = 0; i < build_chunk.size(); i++) {
                auto key = build_chunk.GetValue(left_join_col, i);
                auto hash = std::hash<Value>()(key) % hash_table.size();

                hash_table[hash].push_back({key, build_chunk, i});
            }
        }
    }

    vector<LogicalType> left_types;
    vector<LogicalType> right_types;
};

// 聚合算子
class SimpleHashAggregate : public SimpleOperator {
private:
    unique_ptr<SimpleOperator> child;
    vector<idx_t> group_by_cols;
    bool is_count_agg;

    // 聚合状态
    struct AggregateGroup {
        vector<Value> group_values;
        idx_t count;
    };
    unordered_map<string, AggregateGroup> groups;

    bool scan_complete;
    idx_t current_group;

public:
    SimpleHashAggregate(unique_ptr<SimpleOperator> child_p,
                       vector<idx_t> group_cols,
                       bool count = true)
        : child(std::move(child_p)),
          group_by_cols(std::move(group_cols)),
          is_count_agg(count),
          scan_complete(false),
          current_group(0) {}

    bool GetNext(DataChunk &result) override {
        // Phase 1: 构建聚合
        if (!scan_complete) {
            BuildAggregation();
            scan_complete = true;
        }

        // Phase 2: 输出结果
        if (current_group >= groups.size()) {
            return false;
        }

        // 初始化result（仅首次）
        if (result.ColumnCount() == 0) {
            vector<LogicalType> types;
            for (auto col_idx : group_by_cols) {
                types.push_back(child_types[col_idx]);
            }
            if (is_count_agg) {
                types.push_back(LogicalType::BIGINT);
            }
            result.Initialize(Allocator::DefaultAllocator(), types);
        }

        // 输出当前组
        auto it = groups.begin();
        std::advance(it, current_group);

        for (size_t i = 0; i < group_by_cols.size(); i++) {
            result.data[i].SetValue(0, it->second.group_values[i]);
        }

        if (is_count_agg) {
            result.data[group_by_cols.size()].SetValue(
                0, Value::BIGINT(it->second.count));
        }

        result.SetCardinality(1);
        current_group++;

        return true;
    }

    void Reset() override {
        child->Reset();
        groups.clear();
        scan_complete = false;
        current_group = 0;
    }

private:
    void BuildAggregation() {
        DataChunk input;
        while (child->GetNext(input)) {
            for (idx_t i = 0; i < input.size(); i++) {
                // 构造group key
                string group_key;
                for (auto col_idx : group_by_cols) {
                    group_key += input.GetValue(col_idx, i).ToString();
                    group_key += "|";
                }

                // 查找或创建组
                if (groups.find(group_key) == groups.end()) {
                    AggregateGroup group;
                    for (auto col_idx : group_by_cols) {
                        group.group_values.push_back(input.GetValue(col_idx, i));
                    }
                    group.count = 0;
                    groups[group_key] = std::move(group);
                }

                // 更新聚合
                if (is_count_agg) {
                    groups[group_key].count++;
                }
            }
        }
    }

    vector<LogicalType> child_types;
};

} // namespace duckdb
```

### 14.3 使用示例

```cpp
// 创建测试表
SimpleTable students({
    LogicalType::INTEGER,    // id
    LogicalType::VARCHAR,    // name
    LogicalType::INTEGER,    // class_id
    LogicalType::DOUBLE      // score
}, {"id", "name", "class_id", "score"});

// 插入数据
students.Insert({Value::INTEGER(1), Value("Alice"), Value::INTEGER(1), Value::DOUBLE(95.5)});
students.Insert({Value::INTEGER(2), Value("Bob"), Value::INTEGER(2), Value::DOUBLE(87.3)});
students.Insert({Value::INTEGER(3), Value("Charlie"), Value::INTEGER(1), Value::DOUBLE(92.1)});

// 构建查询计划
// SELECT class_id, COUNT(*) FROM students WHERE score > 90 GROUP BY class_id

auto scan = make_unique<SimpleTableScan>(students);

auto filter = make_unique<SimpleFilter>(
    std::move(scan),
    [](DataChunk &chunk, idx_t row) {
        auto score = chunk.GetValue(3, row);
        return !score.IsNull() && score.GetValue<double>() > 90.0;
    }
);

auto aggregate = make_unique<SimpleHashAggregate>(
    std::move(filter),
    {2},  // GROUP BY class_id (column index 2)
    true  // COUNT(*)
);

// 执行查询
DataChunk result;
while (aggregate->GetNext(result)) {
    printf("class_id: %s, count: %s\n",
           result.GetValue(0, 0).ToString().c_str(),
           result.GetValue(1, 0).ToString().c_str());
}
```

### 14.4 扩展练习

1. **完善Filter算子**
   - 支持AND/OR组合条件
   - 支持比较运算符（<, >, =等）

2. **完善Join算子**
   - 支持多列Join条件
   - 支持LEFT OUTER JOIN
   - 使用更高效的哈希表实现

3. **添加更多聚合函数**
   - SUM, AVG, MIN, MAX
   - 支持多列GROUP BY

4. **添加投影优化**
   - 只读取需要的列
   - 延迟计算表达式

**第二周总结：**

| 主题 | 核心概念 | 关键文件 |
|------|----------|----------|
| 物理计划 | PhysicalOperator, Pipeline | `src/execution/physical_operator.hpp` |
| TableScan | RowGroup, Column扫描 | `src/execution/operator/scan/` |
| Filter | SelectionVector, 向量化过滤 | `src/execution/operator/filter/` |
| Hash Join | Build/Probe阶段, 哈希表 | `src/execution/operator/join/` |
| Aggregation | 哈希聚合, 分组, 聚合函数 | `src/execution/operator/aggregate/` |

**下周预告：**
第三周我们将学习查询优化器，了解如何通过Filter Pushdown、Join Order优化等技术提升查询性能。

---

## 学习资源

1. **DuckDB官方文档**: https://duckdb.org/docs/
2. **源代码**: https://github.com/duckdb/duckdb
3. **相关论文**: "Push-Based Execution in DuckDB"
4. **参考书籍**: "Database Internals" by Alex Petrov

## 学习建议

1. **理论与实践结合** - 每天学习后立即编写代码验证
2. **阅读源码** - 不仅要看示例，还要深入DuckDB源码
3. **循序渐进** - 不要跳过基础内容
4. **做笔记** - 记录关键概念和难点
5. **提问讨论** - 加入DuckDB社区或Discord讨论

祝学习愉快！
