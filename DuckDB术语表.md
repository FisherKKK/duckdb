# 📖 DuckDB 术语表 (Glossary)

DuckDB核心概念和数据库术语的完整参考。

---

## 目录

- [A](#a) | [B](#b) | [C](#c) | [D](#d) | [E](#e) | [F](#f) | [G](#g) | [H](#h)
- [I](#i) | [J](#j) | [K](#k) | [L](#l) | [M](#m) | [N](#n) | [O](#o) | [P](#p)
- [Q](#q) | [R](#r) | [S](#s) | [T](#t) | [U](#u) | [V](#v) | [W](#w) | [X-Z](#x-z)

---

## A

### Allocator (分配器)
内存分配器，负责管理DuckDB中的内存分配和释放。常见的有`Allocator::DefaultAllocator()`。

**示例:**
```cpp
DataChunk chunk;
chunk.Initialize(Allocator::DefaultAllocator(), types);
```

**相关:** BufferManager, Arena Allocator

---

### Aggregation (聚合)
将多行数据合并成单个结果的操作，如SUM、COUNT、AVG等。

**SQL示例:**
```sql
SELECT department, SUM(salary) FROM employees GROUP BY department;
```

**相关:** Hash Aggregation, Group By, Aggregate Function

---

### AST (Abstract Syntax Tree, 抽象语法树)
SQL查询解析后的树形表示，是查询处理的第一个中间表示。

**示例:**
```
SELECT a FROM t WHERE b > 10
    ↓
  SELECT
   ├── Projection: [a]
   ├── FROM: t
   └── WHERE: b > 10
```

**相关:** Parser, Binder

---

### Arena Allocator (竞技场分配器)
一种内存分配策略，批量分配大块内存，然后分发小块，最后统一释放，避免频繁的malloc/free。

**相关:** Allocator, Memory Management

---

## B

### Binder (绑定器)
将AST转换为逻辑计划的组件，负责：
- 符号解析（表名、列名）
- 类型检查
- 语义验证

**示例:**
```
AST: SELECT * FROM users WHERE id = 1
    ↓ Binder
LogicalPlan: Filter(id=1) -> Scan(users)
```

**相关:** AST, LogicalOperator

---

### Block (块)
存储系统的基本单位，DuckDB中默认256KB。

**定义:**
```cpp
constexpr idx_t BLOCK_SIZE = 262144;  // 256KB
```

**相关:** BufferManager, Storage

---

### Buffer (缓冲区)
内存中的数据块，由BufferManager管理。可以被pin（固定）在内存中或evict（驱逐）到磁盘。

**相关:** BufferManager, Pin/Unpin

---

### BufferManager (缓冲区管理器)
管理内存和磁盘之间的数据交换，实现LRU驱逐策略。

**核心功能:**
- 分配和释放Buffer
- Pin/Unpin机制
- 内存压力时驱逐数据到磁盘

**示例:**
```cpp
auto &buffer_manager = BufferManager::GetBufferManager(db);
auto handle = buffer_manager.Allocate(size);
data_ptr_t data = handle.Ptr();
```

**相关:** Block, LRU, Pin/Unpin

---

## C

### Cardinality (基数)
数据集中的行数。

**示例:**
```cpp
chunk.SetCardinality(100);  // 设置chunk包含100行
idx_t count = chunk.size(); // 获取行数
```

**相关:** Cardinality Estimation

---

### Cardinality Estimation (基数估计)
查询优化器估计查询中间结果的行数，用于选择最优执行计划。

**技术:**
- 直方图 (Histogram)
- HyperLogLog (NDV估计)
- 采样 (Sampling)

**相关:** Statistics, Query Optimizer

---

### Catalog (目录)
存储数据库元数据（表、列、索引等）的系统组件。

**包含:**
- 表定义
- 列类型
- 索引信息
- 函数注册

**示例:**
```cpp
auto &catalog = Catalog::GetCatalog(db);
auto table = catalog.GetEntry<TableCatalogEntry>(context, "my_table");
```

**相关:** Schema, Metadata

---

### Checkpoint (检查点)
将内存中的数据持久化到磁盘的过程，保证数据持久性。

**过程:**
1. 暂停写入
2. 将所有RowGroup写入磁盘
3. 更新元数据
4. 截断WAL

**相关:** WAL, Persistence

---

### Chunk (见 DataChunk)

---

### Clock Algorithm (时钟算法)
BufferManager中的页面替换算法，LRU的近似实现。

**原理:**
- 维护一个循环链表
- 每个页面有一个"引用位"
- 时钟指针扫描，清除引用位
- 引用位为0的页面被驱逐

**相关:** BufferManager, LRU

---

### Columnar Storage (列式存储)
按列而不是按行存储数据，适合分析型查询。

**优势:**
- 更好的压缩率
- 减少I/O（只读需要的列）
- 向量化友好

**对比:**
```
行式: [row1: a1,b1,c1] [row2: a2,b2,c2] ...
列式: [col_a: a1,a2,...] [col_b: b1,b2,...] [col_c: c1,c2,...]
```

**相关:** RowGroup, ColumnSegment

---

### ColumnSegment (列段)
ColumnData的一个压缩片段，通常包含一个或多个Vector的数据。

**结构:**
```
ColumnData
├── ColumnSegment 1 (压缩)
├── ColumnSegment 2 (压缩)
└── ...
```

**相关:** ColumnData, Compression

---

### Compression (压缩)
减少数据存储空间的技术。

**DuckDB使用的压缩算法:**
- **FOR (Frame-of-Reference)**: 存储相对于基准值的偏移
- **Dictionary Encoding**: 字符串去重
- **RLE (Run-Length Encoding)**: 连续相同值压缩
- **BitPacking**: 使用最少位数存储整数

**示例:**
```
原始: [100, 101, 102, 103, 104]
FOR: base=100, offsets=[0,1,2,3,4]
```

**相关:** ColumnSegment, Storage

---

### Constant Folding (常量折叠)
在编译期计算常量表达式的优化。

**示例:**
```sql
-- 优化前
SELECT * FROM table WHERE 2 + 3 > x;

-- 优化后
SELECT * FROM table WHERE 5 > x;
```

**相关:** Expression Optimization, Optimizer

---

### CSE (Common Subexpression Elimination, 公共子表达式消除)
消除重复计算的优化技术。

**示例:**
```sql
-- 优化前
SELECT a + b, (a + b) * 2 FROM table;

-- 优化后
temp = a + b
SELECT temp, temp * 2 FROM table;
```

**相关:** Expression Optimization

---

## D

### DataChunk (数据块)
DuckDB的核心数据结构，表示一批列式数据（通常2048行）。

**结构:**
```cpp
class DataChunk {
    vector<Vector> data;      // 列向量数组
    idx_t count;              // 行数
};
```

**用途:**
- 算子之间传递数据
- 批量处理减少开销

**示例:**
```cpp
DataChunk chunk;
chunk.Initialize(alloc, types);
chunk.SetCardinality(2048);
```

**相关:** Vector, Pipeline

---

### Dictionary Encoding (字典编码)
字符串压缩技术，将重复的字符串映射到整数ID。

**示例:**
```
原始: ["apple", "banana", "apple", "cherry", "banana"]
字典: {0: "apple", 1: "banana", 2: "cherry"}
编码: [0, 1, 0, 2, 1]
```

**优势:**
- 节省存储空间
- 加速字符串比较

**相关:** Compression, ColumnSegment

---

### Dynamic Programming (DP, 动态规划)
Join Order优化使用的算法，找到最优的多表连接顺序。

**时间复杂度:** O(3^n) - n为表的数量

**相关:** Join Order Optimization, Cost-based Optimization

---

## E

### Execution Context (执行上下文)
包含查询执行时的运行时信息。

**包含:**
```cpp
struct ExecutionContext {
    ClientContext &client;
    ThreadContext thread;
    PhysicalOperator *operator;
};
```

**相关:** Pipeline, Operator

---

### Expression (表达式)
SQL中的计算单元，如列引用、常量、函数调用等。

**类型:**
- **BoundColumnRefExpression**: 列引用 (e.g., `table.column`)
- **BoundConstantExpression**: 常量 (e.g., `42`, `'hello'`)
- **BoundFunctionExpression**: 函数调用 (e.g., `SUM(x)`)
- **BoundComparisonExpression**: 比较 (e.g., `a > b`)

**示例:**
```sql
SELECT a + 10 FROM t WHERE b > 5
        ↑            ↑
   Function(+)   Comparison(>)
```

**相关:** ExpressionExecutor

---

### ExpressionExecutor (表达式执行器)
计算表达式的组件。

**示例:**
```cpp
ExpressionExecutor executor(allocator);
executor.AddExpression(*expr);

DataChunk input, result;
executor.Execute(input, result);
```

**相关:** Expression, Vector

---

## F

### Filter (过滤器)
根据谓词条件筛选行的操作。

**物理算子:**
```cpp
class PhysicalFilter : public PhysicalOperator {
    unique_ptr<Expression> condition;
};
```

**示例:**
```sql
SELECT * FROM table WHERE age > 18;
```

**相关:** SelectionVector, Predicate

---

### Filter Pushdown (过滤下推)
将过滤条件尽可能向数据源移动的优化技术。

**优势:**
- 减少数据传输
- 提前过滤减少后续处理

**示例:**
```
优化前: Scan -> Project -> Filter
优化后: Scan(with filter) -> Project
```

**相关:** Optimizer, Predicate Pushdown

---

### FlatVector (扁平向量)
最常见的Vector类型，数据连续存储在内存中。

**对比:**
- **FlatVector**: 连续数组
- **ConstantVector**: 所有值相同
- **DictionaryVector**: 通过字典映射

**相关:** Vector, VectorType

---

### FOR (Frame-of-Reference) Encoding
整数压缩算法，存储相对于基准值的偏移量。

**示例:**
```
原始: [1000, 1001, 1002, 1003]
FOR: base=1000, offsets=[0,1,2,3]  (每个offset只需2 bits)
```

**相关:** Compression

---

## G

### Global State (全局状态)
算子在整个查询执行过程中共享的状态。

**用途:**
- 哈希表 (Hash Join)
- 聚合结果 (Aggregation)
- 统计信息

**相关:** Local State, Pipeline

---

### GROUP BY (分组)
按指定列对数据进行分组，通常与聚合函数配合使用。

**示例:**
```sql
SELECT department, COUNT(*) FROM employees GROUP BY department;
```

**实现:** Hash Aggregation

**相关:** Aggregation, Hash Table

---

## H

### Hash Aggregation (哈希聚合)
使用哈希表实现的分组聚合。

**步骤:**
1. Build: 扫描数据，构建哈希表
2. Finalize: 完成聚合计算
3. Scan: 输出结果

**相关:** Hash Table, Aggregation

---

### Hash Join (哈希连接)
使用哈希表实现的等值连接。

**步骤:**
1. **Build**: 扫描右表，构建哈希表 (key -> rows)
2. **Probe**: 扫描左表，探测哈希表匹配

**优势:**
- O(n+m) 时间复杂度
- 适合大数据集

**相关:** Join, Hash Table

---

### Hash Table (哈希表)
用于Hash Join和Hash Aggregation的数据结构。

**实现:**
```cpp
class HashTable {
    vector<vector<idx_t>> buckets;  // 桶数组
    DataChunk data;                  // 存储数据
};
```

**相关:** Hash Join, Hash Aggregation

---

### HyperLogLog (HLL)
估计集合中不同元素数量(NDV)的概率算法。

**特点:**
- 空间复杂度: O(1) - 只需几KB
- 误差率: ~2%
- 可合并

**用途:**
- 估计 `COUNT(DISTINCT column)`
- 查询优化中的基数估计

**相关:** Statistics, Cardinality Estimation

---

## I

### Index (索引)
加速数据查找的数据结构。

**DuckDB支持:**
- ART (Adaptive Radix Tree) Index

**示例:**
```sql
CREATE INDEX idx_users_id ON users(id);
```

**相关:** ART, B-Tree

---

## J

### Join (连接)
合并两个表的操作。

**类型:**
- **INNER JOIN**: 返回匹配的行
- **LEFT JOIN**: 返回左表所有行
- **RIGHT JOIN**: 返回右表所有行
- **FULL OUTER JOIN**: 返回所有行

**实现:**
- Hash Join (等值连接)
- Nested Loop Join (非等值连接)
- Merge Join (排序数据)

**相关:** Hash Join, Join Order Optimization

---

### Join Order Optimization (连接顺序优化)
选择最优的多表连接顺序以最小化执行成本。

**算法:** 动态规划 (Dynamic Programming)

**示例:**
```sql
-- 3个表: A (1000行), B (100行), C (10行)
SELECT * FROM A JOIN B JOIN C
-- 最优顺序: C JOIN B JOIN A (从小到大)
```

**相关:** Cost-based Optimization, Dynamic Programming

---

## L

### Local State (本地状态)
每个线程独有的算子状态。

**用途:**
- 临时缓冲区
- 线程私有数据
- 避免锁竞争

**相关:** Global State, Pipeline

---

### Logical Operator (逻辑算子)
查询计划中的逻辑节点，描述"做什么"。

**常见类型:**
- LogicalGet (表扫描)
- LogicalFilter (过滤)
- LogicalJoin (连接)
- LogicalAggregate (聚合)
- LogicalProjection (投影)

**示例:**
```
LogicalFilter (age > 18)
    └── LogicalGet (users)
```

**相关:** Physical Operator, Optimizer

---

### LogicalType (逻辑类型)
DuckDB中的数据类型。

**常见类型:**
```cpp
LogicalType::INTEGER
LogicalType::BIGINT
LogicalType::DOUBLE
LogicalType::VARCHAR
LogicalType::DATE
LogicalType::TIMESTAMP
```

**相关:** PhysicalType, Type System

---

### LRU (Least Recently Used, 最近最少使用)
BufferManager使用的页面替换策略。

**原理:**
- 驱逐最久未使用的页面
- DuckDB使用Clock算法近似LRU

**相关:** BufferManager, Clock Algorithm

---

## M

### Materialization (物化)
将查询结果完整计算并存储。

**对比:**
- **Materialized**: 计算全部结果
- **Streaming**: 按需计算，逐批返回

**相关:** Pipeline, Execution Model

---

### Metadata (元数据)
描述数据的数据，如表结构、列类型、统计信息等。

**存储位置:**
- Catalog (目录)
- Statistics (统计)

**相关:** Catalog, Statistics

---

### Morsel-Driven Parallelism (块驱动并行)
一种并行执行模型，将数据分成小块(morsel)分配给线程。

**特点:**
- 动态负载均衡
- 细粒度并行
- 适合NUMA架构

**相关:** Pipeline, Parallelism

---

### MVCC (Multi-Version Concurrency Control, 多版本并发控制)
通过维护数据的多个版本实现事务隔离。

**核心概念:**
- 每个事务看到一个一致的快照
- 读不阻塞写，写不阻塞读
- 版本链管理

**示例:**
```
Row 1:  v1(txn=10) -> v2(txn=15) -> v3(txn=20)
Txn 12: 看到 v1
Txn 18: 看到 v2
Txn 25: 看到 v3
```

**相关:** Transaction, Isolation Level

---

## N

### NDV (Number of Distinct Values, 不同值数量)
列中唯一值的数量。

**用途:**
- 基数估计
- Join Order优化
- 选择索引

**估计方法:**
- HyperLogLog
- 采样

**相关:** HyperLogLog, Statistics

---

### NULL (空值)
表示缺失或未知的值。

**实现:** ValidityMask

**示例:**
```cpp
auto &validity = FlatVector::Validity(vec);
validity.SetInvalid(3);  // 设置第3行为NULL
bool is_null = !validity.RowIsValid(3);
```

**相关:** ValidityMask

---

## O

### Operator (算子)
查询执行计划中的节点，执行特定的操作。

**分类:**
- **Source**: 产生数据 (TableScan)
- **Operator**: 转换数据 (Filter, Project)
- **Sink**: 消费数据 (HashJoin Build, Aggregation)

**相关:** Physical Operator, Pipeline

---

### Optimizer (优化器)
将逻辑计划转换为高效物理计划的组件。

**优化技术:**
- Filter Pushdown
- Join Order Optimization
- Constant Folding
- Common Subexpression Elimination
- Column Pruning

**示例:**
```
Logical Plan → Optimizer → Optimized Logical Plan → Physical Plan
```

**相关:** Logical Operator, Physical Operator

---

## P

### Parser (解析器)
将SQL文本转换为AST的组件。

**过程:**
```
SQL String → Lexer → Tokens → Parser → AST
```

**示例:**
```sql
SELECT * FROM users WHERE id = 1
    ↓
AST {
  type: SELECT,
  columns: [*],
  from: "users",
  where: { left: "id", op: "=", right: 1 }
}
```

**相关:** AST, Binder

---

### Physical Operator (物理算子)
执行计划中的实际执行节点，描述"怎么做"。

**常见类型:**
- PhysicalTableScan
- PhysicalFilter
- PhysicalHashJoin
- PhysicalHashAggregate

**接口:**
```cpp
class PhysicalOperator {
    virtual OperatorResultType GetData(DataChunk &chunk) = 0;
};
```

**相关:** Logical Operator, Pipeline

---

### Pin/Unpin (固定/取消固定)
BufferManager中的操作，控制页面是否可以被驱逐。

**用法:**
```cpp
auto handle = buffer_manager.Pin(block);
data_ptr_t data = handle.Ptr();
// 使用数据...
// handle析构时自动Unpin
```

**相关:** BufferManager, Buffer

---

### Pipeline (管道)
一系列可以流式执行的算子链。

**结构:**
```
Source → Operator → Operator → ... → Sink
```

**特点:**
- 数据以DataChunk为单位流动
- 减少物化开销
- 支持并行执行

**示例:**
```
TableScan → Filter → Project → HashJoin(Build)
```

**相关:** Operator, Morsel-Driven Parallelism

---

### Predicate (谓词)
布尔表达式，用于过滤数据。

**示例:**
```sql
WHERE age > 18 AND city = 'NYC'
```

**相关:** Filter, Expression

---

### Projection (投影)
选择需要的列。

**示例:**
```sql
SELECT id, name FROM users;  -- 只投影id和name列
```

**相关:** Column Pruning

---

## Q

### Query Plan (查询计划)
查询的执行蓝图。

**层次:**
1. **AST** - 语法树
2. **Logical Plan** - 逻辑计划
3. **Optimized Logical Plan** - 优化后的逻辑计划
4. **Physical Plan** - 物理执行计划

**查看:**
```sql
EXPLAIN SELECT ...;
```

**相关:** Optimizer, Physical Operator

---

## R

### RLE (Run-Length Encoding, 游程编码)
压缩连续相同值的算法。

**示例:**
```
原始: [A, A, A, A, B, B, C, C, C]
RLE:  [(A,4), (B,2), (C,3)]
```

**适用场景:**
- 排序列
- 低基数列

**相关:** Compression

---

### RowGroup (行组)
存储引擎的基本单位，包含~120,000行数据。

**结构:**
```
RowGroup
├── ColumnData 1
│   ├── ColumnSegment 1
│   └── ColumnSegment 2
├── ColumnData 2
└── ...
```

**相关:** DataTable, ColumnData

---

## S

### Scan (扫描)
读取表数据的操作。

**类型:**
- **Sequential Scan (顺序扫描)**: 逐行读取
- **Index Scan (索引扫描)**: 通过索引定位

**相关:** TableScan, Physical Operator

---

### SelectionVector (选择向量)
存储行索引的数组，实现零拷贝过滤。

**用法:**
```cpp
SelectionVector sel(STANDARD_VECTOR_SIZE);
idx_t approved_count = 0;

for (idx_t i = 0; i < count; i++) {
    if (predicate(i)) {
        sel.set_index(approved_count++, i);
    }
}

result.Slice(input, sel, approved_count);
```

**相关:** Filter, Vector::Slice

---

### SIMD (Single Instruction Multiple Data, 单指令多数据)
利用CPU的向量指令并行处理多个数据。

**示例 (AVX2):**
```cpp
// 一次处理8个int32
__m256i a = _mm256_load_si256((__m256i*)&data_a[i]);
__m256i b = _mm256_load_si256((__m256i*)&data_b[i]);
__m256i result = _mm256_add_epi32(a, b);
_mm256_store_si256((__m256i*)&result_data[i], result);
```

**相关:** Vectorization, Performance

---

### Sink (汇算子)
消费数据的算子，如HashJoin的Build阶段、Aggregation。

**接口:**
```cpp
virtual SinkResultType Sink(DataChunk &input);
virtual SinkFinalizeType Finalize();
```

**相关:** Source, Operator

---

### Source (源算子)
产生数据的算子，如TableScan。

**接口:**
```cpp
virtual OperatorResultType GetData(DataChunk &output);
```

**相关:** Sink, Operator

---

### Statistics (统计信息)
关于数据分布的元数据。

**包含:**
- Min/Max值
- NDV (不同值数量)
- NULL计数
- 直方图

**用途:**
- 基数估计
- Join Order优化
- Filter Pushdown

**相关:** Cardinality Estimation, Optimizer

---

## T

### TableScan (表扫描)
读取表数据的Source算子。

**优化:**
- Filter Pushdown
- Column Pruning
- Projection Pushdown

**相关:** Scan, Physical Operator

---

### Transaction (事务)
一组数据库操作的逻辑单元，满足ACID特性。

**ACID:**
- **Atomicity (原子性)**: 全部成功或全部失败
- **Consistency (一致性)**: 数据完整性
- **Isolation (隔离性)**: 并发事务互不影响
- **Durability (持久性)**: 提交后永久保存

**相关:** MVCC, WAL

---

### Type System (类型系统)
定义数据类型及其转换规则。

**类型:**
- 数值型: INTEGER, BIGINT, DOUBLE
- 字符串: VARCHAR
- 时间: DATE, TIMESTAMP
- 复杂类型: STRUCT, LIST, MAP

**相关:** LogicalType, PhysicalType

---

## U

### Unified Vector Format (统一向量格式)
将不同VectorType转换为统一的访问接口。

**用途:**
- 简化向量操作的实现
- 支持多种VectorType

**示例:**
```cpp
UnifiedVectorFormat vdata;
vec.ToUnifiedFormat(count, vdata);

// 访问数据
auto data = (int32_t*)vdata.data;
for (idx_t i = 0; i < count; i++) {
    idx_t idx = vdata.sel->get_index(i);
    if (vdata.validity.RowIsValid(idx)) {
        int32_t value = data[idx];
    }
}
```

**相关:** Vector, VectorType

---

## V

### ValidityMask (有效性掩码)
使用位向量标记NULL值。

**实现:**
```cpp
class ValidityMask {
    uint64_t *bits;  // 每个bit表示一行是否有效
};
```

**用法:**
```cpp
ValidityMask mask;
mask.SetInvalid(3);              // 设置第3行为NULL
bool is_valid = mask.RowIsValid(3);  // false
```

**相关:** NULL, Vector

---

### Value (值)
单个标量值的容器。

**示例:**
```cpp
Value v1 = Value::INTEGER(42);
Value v2 = Value::VARCHAR("hello");
Value null_val = Value(LogicalType::INTEGER);  // NULL

int32_t i = v1.GetValue<int32_t>();
```

**相关:** Vector, LogicalType

---

### Vector (向量)
列数据的容器，DuckDB的核心数据结构。

**类型 (VectorType):**
- **FLAT_VECTOR**: 连续数组
- **CONSTANT_VECTOR**: 所有值相同
- **DICTIONARY_VECTOR**: 字典编码

**大小:** 通常2048个元素 (STANDARD_VECTOR_SIZE)

**示例:**
```cpp
Vector vec(LogicalType::INTEGER, 2048);
auto data = FlatVector::GetData<int32_t>(vec);
data[0] = 42;
```

**相关:** DataChunk, FlatVector

---

### Vectorization (向量化)
批量处理数据而不是逐行处理。

**优势:**
- 减少函数调用开销
- 利用SIMD
- 更好的缓存局部性

**示例:**
```cpp
// ❌ 标量处理
for (idx_t i = 0; i < count; i++) {
    result[i] = input[i] * 2;
}

// ✅ 向量化
VectorOperations::Multiply(input, constant_2, result, count);
```

**相关:** SIMD, Vector

---

### VectorType (向量类型)
Vector的存储方式。

**类型:**
- **FLAT_VECTOR**: 扁平数组
- **CONSTANT_VECTOR**: 常量向量
- **DICTIONARY_VECTOR**: 字典向量
- **SEQUENCE_VECTOR**: 序列向量 (如递增序列)

**相关:** Vector, Unified Vector Format

---

## W

### WAL (Write-Ahead Log, 预写日志)
在修改数据前先写入日志，保证持久性和可恢复性。

**过程:**
1. 写入WAL
2. 应用修改到内存
3. 后台刷新到磁盘

**恢复:**
```
Crash → 重放WAL → 恢复数据
```

**相关:** Checkpoint, Transaction, Durability

---

## X-Z

### Zone Map (区域映射)
存储ColumnSegment的Min/Max值，用于快速跳过不相关的数据。

**示例:**
```
Segment 1: [1, 2, 3, 4, 5]     → Zone Map: Min=1, Max=5
Segment 2: [100, 101, 102]     → Zone Map: Min=100, Max=102

Query: WHERE x > 50
→ 跳过 Segment 1 (Max=5 < 50)
→ 扫描 Segment 2
```

**相关:** Statistics, Column Pruning

---

## 🔗 关系图

### 数据流层次
```
Value (单个值)
    ↓
Vector (列向量, ~2048个值)
    ↓
DataChunk (多个Vector, 一批数据)
    ↓
Pipeline (多个DataChunk流动)
    ↓
Query Result (完整结果)
```

### 查询处理流程
```
SQL String
    ↓ Parser
AST
    ↓ Binder
Logical Plan
    ↓ Optimizer
Optimized Logical Plan
    ↓ Physical Planner
Physical Plan
    ↓ Executor
Result
```

### 存储层次
```
Database
    ↓
DataTable
    ↓
RowGroup (~120k rows)
    ↓
ColumnData (单列)
    ↓
ColumnSegment (压缩段)
    ↓
Block (256KB)
```

---

## 📚 学习建议

1. **基础概念先行**: 先理解Vector、DataChunk、Value
2. **跟随数据流**: 理解数据如何在Pipeline中流动
3. **查询执行路径**: 从SQL到结果的完整流程
4. **存储结构**: RowGroup和列式存储
5. **优化技术**: Filter Pushdown、Join Order等

---

**提示:** 遇到不熟悉的术语时，回到此术语表查找！

---

最后更新：2026-01-17
