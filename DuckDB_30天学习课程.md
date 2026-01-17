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

**(由于篇幅限制，课程内容将继续，包含Day 10-30的详细内容...)**

---

## 课程大纲（Day 10-30）

### 第二周（续）

**Day 10:** 物理计划 - PhysicalOperator
**Day 11:** 算子实现 - TableScan和Filter
**Day 12:** 算子实现 - Hash Join
**Day 13:** 算子实现 - Aggregation
**Day 14:** 第二周总结 - 实现基本查询执行

### 第三周：优化器与性能

**Day 15:** 优化器架构与规则系统
**Day 16:** Filter Pushdown优化
**Day 17:** Join Order优化
**Day 18:** 统计信息与基数估计
**Day 19:** 表达式优化与常量折叠
**Day 20:** 向量化执行深度解析
**Day 21:** 第三周总结 - 实现优化规则

### 第四周：存储与事务

**Day 22:** 存储引擎架构
**Day 23:** RowGroup与列存储
**Day 24:** 压缩算法
**Day 25:** MVCC事务管理
**Day 26:** WAL与持久化
**Day 27:** Buffer管理与缓存策略
**Day 28:** 第四周总结 - 实现简单存储引擎

### 最后两天

**Day 29:** 扩展系统与函数注册
**Day 30:** 总结与构建Mini-DuckDB项目

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
