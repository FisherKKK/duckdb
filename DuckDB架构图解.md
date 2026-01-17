# 🎨 DuckDB 架构图解

使用ASCII艺术和可视化图表展示DuckDB的核心架构。

---

## 📋 目录

1. [整体架构](#整体架构)
2. [查询处理流程](#查询处理流程)
3. [数据结构层次](#数据结构层次)
4. [Pipeline执行模型](#pipeline执行模型)
5. [存储引擎架构](#存储引擎架构)
6. [内存管理](#内存管理)
7. [事务处理](#事务处理)
8. [优化器架构](#优化器架构)

---

## 🏗️ 整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                         DuckDB System                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌───────────────┐        ┌──────────────┐                      │
│  │  Client API   │◄──────►│   Catalog    │                      │
│  └───────┬───────┘        └──────────────┘                      │
│          │                                                        │
│          ▼                                                        │
│  ┌───────────────────────────────────────────────┐              │
│  │         Query Processing Layer                 │              │
│  │  ┌────────┐  ┌────────┐  ┌─────────┐         │              │
│  │  │ Parser │─►│ Binder │─►│Optimizer│         │              │
│  │  └────────┘  └────────┘  └────┬────┘         │              │
│  │                                 ▼              │              │
│  │                          ┌────────────┐       │              │
│  │                          │  Planner   │       │              │
│  │                          └─────┬──────┘       │              │
│  └────────────────────────────────┼──────────────┘              │
│                                    ▼                              │
│  ┌───────────────────────────────────────────────┐              │
│  │         Execution Engine                       │              │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐   │              │
│  │  │Pipeline 1│  │Pipeline 2│  │Pipeline 3│   │              │
│  │  └─────┬────┘  └─────┬────┘  └─────┬────┘   │              │
│  │        │             │             │         │              │
│  │        └─────────────┴─────────────┘         │              │
│  │                      │                        │              │
│  └──────────────────────┼────────────────────────┘              │
│                         ▼                                        │
│  ┌───────────────────────────────────────────────┐              │
│  │         Storage Engine                         │              │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐   │              │
│  │  │RowGroup 1│  │RowGroup 2│  │RowGroup 3│   │              │
│  │  └──────────┘  └──────────┘  └──────────┘   │              │
│  │                                               │              │
│  │  ┌────────────────────────────────────────┐ │              │
│  │  │      Buffer Manager (LRU Cache)        │ │              │
│  │  └────────────────────────────────────────┘ │              │
│  │                                               │              │
│  │  ┌────────────────────────────────────────┐ │              │
│  │  │      WAL (Write-Ahead Log)             │ │              │
│  │  └────────────────────────────────────────┘ │              │
│  └───────────────────────────────────────────────┘              │
│                         │                                        │
│                         ▼                                        │
│  ┌───────────────────────────────────────────────┐              │
│  │            Disk Storage                        │              │
│  │  [Data Blocks] [Index Blocks] [WAL Blocks]   │              │
│  └───────────────────────────────────────────────┘              │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔄 查询处理流程

### 端到端流程

```
┌─────────────────────────────────────────────────────────────────┐
│                       Query Execution Flow                       │
└─────────────────────────────────────────────────────────────────┘

SQL Query: "SELECT name FROM users WHERE age > 18"
    │
    │ 1. PARSING
    ▼
┌──────────────────────┐
│   Abstract Syntax    │  ┌─────────────────────────────┐
│      Tree (AST)      │  │ SelectStatement             │
└──────────┬───────────┘  │  ├─ target: [name]          │
           │              │  ├─ from: users             │
           │              │  └─ where: age > 18         │
           │              └─────────────────────────────┘
           │ 2. BINDING
           ▼
┌──────────────────────┐
│   Logical Plan       │  ┌─────────────────────────────┐
│   (Unoptimized)      │  │ LogicalProjection [name]    │
└──────────┬───────────┘  │    └─ LogicalFilter [age>18]│
           │              │        └─ LogicalGet [users] │
           │              └─────────────────────────────┘
           │ 3. OPTIMIZATION
           ▼
┌──────────────────────┐
│   Logical Plan       │  ┌─────────────────────────────┐
│   (Optimized)        │  │ LogicalProjection [name]    │
└──────────┬───────────┘  │    └─ LogicalGet [users]    │
           │              │        filter: age > 18     │
           │              └─────────────────────────────┘
           │                (Filter pushed down)
           │ 4. PHYSICAL PLANNING
           ▼
┌──────────────────────┐
│   Physical Plan      │  ┌─────────────────────────────┐
└──────────┬───────────┘  │ PhysicalProjection [name]   │
           │              │    └─ PhysicalTableScan     │
           │              │        (users, age > 18)    │
           │              └─────────────────────────────┘
           │ 5. EXECUTION
           ▼
┌──────────────────────┐
│     Pipeline         │  ┌─────────────────────────────┐
│     Execution        │  │ Scan → Project              │
└──────────┬───────────┘  │  ↓      ↓                   │
           │              │ Chunk  Chunk → Result       │
           │              └─────────────────────────────┘
           │ 6. RESULT
           ▼
┌──────────────────────┐
│  Query Result        │  ┌─────────────────────────────┐
│  (DataChunk)         │  │ name                        │
└──────────────────────┘  │ ──────                      │
                          │ Alice                       │
                          │ Bob                         │
                          │ Charlie                     │
                          └─────────────────────────────┘
```

---

## 📦 数据结构层次

### Value → Vector → DataChunk

```
┌──────────────────────────────────────────────────────────┐
│                     Data Hierarchy                        │
└──────────────────────────────────────────────────────────┘

┌────────────┐
│   Value    │  单个标量值
└────────────┘
     │         Value::INTEGER(42)
     │         Value::VARCHAR("hello")
     │
     ▼ (2048个Value组成一个Vector)
┌────────────────────────────────────────────────┐
│               Vector (Column)                   │
│  ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐   │
│  │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │ 8 │...│2048│  │  Data Array
│  └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘   │
│  ┌─────────────────────────────────────────┐  │
│  │  ValidityMask (NULL bitmap)             │  │
│  │  [1][1][0][1][1][1][1][1]...[1]        │  │  0=NULL, 1=Valid
│  └─────────────────────────────────────────┘  │
│  type: LogicalType::INTEGER                   │
│  count: 2048                                   │
└────────────────────────────────────────────────┘
     │
     ▼ (多个Vector组成DataChunk)
┌─────────────────────────────────────────────────────────┐
│              DataChunk (Batch of Rows)                   │
│                                                           │
│  Column 0 (id)     Column 1 (name)    Column 2 (age)    │
│  ┌────────────┐   ┌────────────┐    ┌────────────┐     │
│  │ Vector     │   │ Vector     │    │ Vector     │     │
│  │ [1,2,3,..] │   │ [A,B,C,..] │    │ [25,30,..] │     │
│  └────────────┘   └────────────┘    └────────────┘     │
│                                                           │
│  count: 2048 rows                                        │
│  ColumnCount(): 3                                        │
└─────────────────────────────────────────────────────────┘
     │
     ▼ (多个DataChunk流过Pipeline)
┌─────────────────────────────────────────────────────────┐
│                    Pipeline                              │
│   Chunk 1 → Chunk 2 → Chunk 3 → ... → Result           │
└─────────────────────────────────────────────────────────┘
```

---

### Vector类型详解

```
┌──────────────────────────────────────────────────────────┐
│                   VectorType Variants                     │
└──────────────────────────────────────────────────────────┘

1. FLAT_VECTOR (最常见)
   ┌─────────────────────────────────────────┐
   │ Data: [10, 20, 30, 40, 50, ...]        │  连续数组
   │ Validity: [1, 1, 0, 1, 1, ...]         │
   └─────────────────────────────────────────┘

2. CONSTANT_VECTOR (所有值相同)
   ┌─────────────────────────────────────────┐
   │ Value: 42                               │  只存储一个值
   │ Count: 2048                             │  逻辑上有2048个
   └─────────────────────────────────────────┘

3. DICTIONARY_VECTOR (字典编码)
   ┌─────────────────────────────────────────┐
   │ Dictionary: ["apple", "banana", "cherry"]│
   │ Codes: [0, 1, 0, 2, 1, 0, ...]         │  索引数组
   └─────────────────────────────────────────┘

4. SEQUENCE_VECTOR (序列)
   ┌─────────────────────────────────────────┐
   │ Start: 100                              │  起始值
   │ Increment: 1                            │  增量
   │ Count: 2048                             │  → [100,101,102,...]
   └─────────────────────────────────────────┘
```

---

## ⚡ Pipeline执行模型

### Pipeline结构

```
┌──────────────────────────────────────────────────────────┐
│                    Pipeline Anatomy                       │
└──────────────────────────────────────────────────────────┘

┌─────────────┐
│   Source    │  产生数据 (e.g., TableScan)
│  Operator   │
└──────┬──────┘
       │ DataChunk
       ▼
┌─────────────┐
│  Operator   │  转换数据 (e.g., Filter, Project)
│  (Filter)   │
└──────┬──────┘
       │ DataChunk (filtered)
       ▼
┌─────────────┐
│  Operator   │  转换数据
│ (Project)   │
└──────┬──────┘
       │ DataChunk (projected)
       ▼
┌─────────────┐
│    Sink     │  消费数据 (e.g., HashJoin Build, Aggregate)
│  Operator   │
└─────────────┘
```

---

### 完整查询示例

```
Query: SELECT name FROM users WHERE age > 18 JOIN orders ON users.id = orders.user_id

Pipeline Breakdown:

Pipeline 1: Build Hash Table for Join
┌──────────────────┐
│  TableScan       │  扫描 orders 表
│  (orders)        │
└────────┬─────────┘
         │ DataChunk
         ▼
┌──────────────────┐
│  HashJoin        │  Build Phase: 构建哈希表
│  (Build)         │  Key: user_id
└──────────────────┘

Pipeline 2: Probe and Output
┌──────────────────┐
│  TableScan       │  扫描 users 表
│  (users)         │
└────────┬─────────┘
         │ DataChunk
         ▼
┌──────────────────┐
│  Filter          │  age > 18
│  (age > 18)      │
└────────┬─────────┘
         │ DataChunk (filtered)
         ▼
┌──────────────────┐
│  HashJoin        │  Probe Phase: 探测哈希表
│  (Probe)         │  Output: joined rows
└────────┬─────────┘
         │ DataChunk (joined)
         ▼
┌──────────────────┐
│  Projection      │  投影 name 列
│  (name)          │
└────────┬─────────┘
         │ DataChunk (final)
         ▼
┌──────────────────┐
│  Result          │
└──────────────────┘
```

---

### Push-Based执行

```
┌──────────────────────────────────────────────────────────┐
│              Push-Based Execution Model                   │
└──────────────────────────────────────────────────────────┘

Source算子 "推送" 数据到下游:

   ┌──────────┐
   │ TableScan│ <── Pull from storage
   └─────┬────┘
         │ Push(Chunk 1)
         ▼
   ┌──────────┐
   │  Filter  │ <── Process and push filtered data
   └─────┬────┘
         │ Push(Chunk 1, filtered)
         ▼
   ┌──────────┐
   │ Project  │ <── Process and push projected data
   └─────┬────┘
         │ Push(Chunk 1, final)
         ▼
   ┌──────────┐
   │  Result  │ <── Collect results
   └──────────┘

每个算子:
1. 接收上游的DataChunk
2. 处理DataChunk
3. 推送结果到下游
```

---

## 💾 存储引擎架构

### 存储层次结构

```
┌──────────────────────────────────────────────────────────┐
│                  Storage Hierarchy                        │
└──────────────────────────────────────────────────────────┘

Database
    │
    └── DataTable (e.g., "users")
           │
           ├── RowGroup 1 (0 - 122,880 rows)
           │      │
           │      ├── ColumnData (column 0: id)
           │      │      │
           │      │      ├── ColumnSegment 1 (rows 0-2047, compressed)
           │      │      ├── ColumnSegment 2 (rows 2048-4095, compressed)
           │      │      └── ...
           │      │
           │      ├── ColumnData (column 1: name)
           │      │      │
           │      │      └── ColumnSegment(s)
           │      │
           │      └── ColumnData (column 2: age)
           │             └── ColumnSegment(s)
           │
           ├── RowGroup 2 (122,881 - 245,760 rows)
           │      └── ... (similar structure)
           │
           └── RowGroup N
                  └── ...
```

---

### RowGroup详细结构

```
┌──────────────────────────────────────────────────────────┐
│              RowGroup Internal Structure                  │
└──────────────────────────────────────────────────────────┘

RowGroup (最多 ~122,880 rows)
    │
    │ row_count: 100,000
    │ start_row: 0
    │
    ├── Column 0 (id: INTEGER)
    │      │
    │      ├── Statistics
    │      │    ├─ Min: 1
    │      │    ├─ Max: 100000
    │      │    ├─ NDV: 100000
    │      │    └─ NULL count: 0
    │      │
    │      └── Segments
    │           ├─ Segment 0: [rows 0-2047]
    │           │    ├─ Compression: FOR (base=1, offsets)
    │           │    ├─ Compressed Size: 512 bytes
    │           │    └─ Zone Map: Min=1, Max=2048
    │           │
    │           ├─ Segment 1: [rows 2048-4095]
    │           │    └─ ...
    │           └─ ...
    │
    ├── Column 1 (name: VARCHAR)
    │      │
    │      ├── Statistics
    │      │    ├─ NDV: 5000 (5000个不同名字)
    │      │    └─ NULL count: 100
    │      │
    │      └── Segments
    │           ├─ Segment 0
    │           │    ├─ Compression: Dictionary
    │           │    │   Dictionary: ["Alice", "Bob", ...]
    │           │    │   Codes: [0, 1, 0, 2, ...]
    │           │    └─ Compressed Size: 2KB
    │           └─ ...
    │
    └── Column 2 (age: INTEGER)
           │
           ├── Statistics
           └── Segments
                └─ ...
```

---

### 压缩示例

```
┌──────────────────────────────────────────────────────────┐
│                 Compression Techniques                    │
└──────────────────────────────────────────────────────────┘

1. Frame-of-Reference (FOR) - 适用于范围较小的整数
   原始数据:  [1000, 1001, 1002, 1003, 1004, ...]
                ↓
   压缩:      base = 1000
              offsets = [0, 1, 2, 3, 4, ...] (每个只需2 bits)
   节省: 32 bits → 2 bits (16x压缩)

2. Dictionary Encoding - 适用于重复字符串
   原始数据:  ["apple", "banana", "apple", "cherry", "banana", ...]
                ↓
   字典:      0:"apple", 1:"banana", 2:"cherry"
   编码:      [0, 1, 0, 2, 1, ...] (codes)
   节省: ~20 bytes/string → 1-2 bytes/code (10x+)

3. Run-Length Encoding (RLE) - 适用于连续相同值
   原始数据:  [A, A, A, A, B, B, C, C, C, C, C, ...]
                ↓
   压缩:      [(A, 4), (B, 2), (C, 5), ...]
   节省: 高度取决于数据重复度

4. BitPacking - 适用于小整数
   原始数据:  [1, 2, 3, 4, 5, 6, 7, 8] (每个用8 bits存储)
                ↓
   压缩:      每个用3 bits (max=8需要3 bits)
              [001][010][011][100][101][110][111][000]
   节省: 8 bits → 3 bits (2.67x)
```

---

## 🧠 内存管理

### BufferManager架构

```
┌──────────────────────────────────────────────────────────┐
│                   Buffer Manager                          │
└──────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                  Memory Pool                             │
│                                                           │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐   │
│  │ Block 1 │  │ Block 2 │  │ Block 3 │  │ Block 4 │   │
│  │ (Pinned)│  │ (Free)  │  │ (Pinned)│  │ (Free)  │   │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘   │
│       ▲            │             ▲            │          │
│       │ Pin        │ Evict       │ Pin        │ Evict   │
│       │            ▼             │            ▼          │
│  ┌─────────────────────────────────────────────────┐   │
│  │          LRU Eviction Policy                    │   │
│  │  [Block 2] → [Block 4] → [Block 7] → ...       │   │
│  │  (Least Recently Used)                          │   │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
                           │
                           │ If memory pressure
                           ▼
┌─────────────────────────────────────────────────────────┐
│                  Disk Storage                            │
│  [Block 2 swapped] [Block 4 swapped] ...               │
└─────────────────────────────────────────────────────────┘

Pin/Unpin机制:
1. Pin(block_id) → 加载到内存，引用计数+1，不可驱逐
2. Unpin(block_id) → 引用计数-1，可驱逐
3. Evict → 选择LRU且未pinned的block写回磁盘
```

---

### Clock算法 (LRU近似)

```
┌──────────────────────────────────────────────────────────┐
│                    Clock Algorithm                        │
└──────────────────────────────────────────────────────────┘

循环链表中的Buffer:

    ┌─────────┐      ┌─────────┐      ┌─────────┐
    │ Block 1 │──────│ Block 2 │──────│ Block 3 │
    │ ref=1   │      │ ref=0   │      │ ref=1   │
    └────┬────┘      └────┬────┘      └────┬────┘
         │                │                │
         └────────────────┴────────────────┘
                          │
                      Clock Hand (指针)

驱逐算法:
1. 从Clock Hand开始扫描
2. 如果 ref=1: 设置ref=0, 继续
3. 如果 ref=0: 驱逐该block
4. 移动Clock Hand到下一个

示例:
Hand at Block 2, ref=0 → Evict Block 2
Hand moves to Block 3
```

---

## 🔐 事务处理

### MVCC版本链

```
┌──────────────────────────────────────────────────────────┐
│              MVCC Version Chain                           │
└──────────────────────────────────────────────────────────┘

Table: users (id=1的行)

Version Chain (从新到旧):

┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  Version 3      │ ──► │  Version 2      │ ──► │  Version 1      │
│  txn_id: 30     │     │  txn_id: 20     │     │  txn_id: 10     │
│  name: "Carol"  │     │  name: "Bob"    │     │  name: "Alice"  │
│  age: 35        │     │  age: 30        │     │  age: 25        │
│  deleted: false │     │  deleted: false │     │  deleted: false │
└─────────────────┘     └─────────────────┘     └─────────────────┘
   (最新版本)              (旧版本)                (最老版本)

事务可见性:
- Transaction 15: 看到 Version 1 (txn_id=10 < 15)
- Transaction 25: 看到 Version 2 (txn_id=20 < 25)
- Transaction 35: 看到 Version 3 (txn_id=30 < 35)

垃圾回收:
- 当所有活跃事务都 > 10时, Version 1可以被删除
```

---

### WAL (Write-Ahead Log)

```
┌──────────────────────────────────────────────────────────┐
│                 WAL Processing Flow                       │
└──────────────────────────────────────────────────────────┘

Transaction Execution:

1. 开始事务
   BEGIN TRANSACTION txn_id=100

2. 执行操作 (先写WAL)
   INSERT INTO users VALUES (1, 'Alice', 25)
     ↓
   ┌─────────────────────────────────────┐
   │         WAL File                     │
   │  [txn=100, op=INSERT, table=users,  │ ← 先写入WAL
   │   data=(1, 'Alice', 25)]            │
   └─────────────────────────────────────┘
     ↓ fsync (确保写入磁盘)
     ↓
   ┌─────────────────────────────────────┐
   │       Memory (DataTable)             │
   │  users: [..., (1, 'Alice', 25)]     │ ← 再更新内存
   └─────────────────────────────────────┘

3. 提交事务
   COMMIT txn_id=100
     ↓
   ┌─────────────────────────────────────┐
   │         WAL File                     │
   │  [..., COMMIT txn=100]              │ ← 写入COMMIT记录
   └─────────────────────────────────────┘

4. 后台Checkpoint
   定期将内存中的数据刷新到磁盘
     ↓
   ┌─────────────────────────────────────┐
   │      Disk (Data Files)               │
   │  users_rowgroup_1.bin: [...]        │ ← 持久化数据
   └─────────────────────────────────────┘
     ↓
   截断WAL (已checkpoint的部分可以删除)

恢复流程 (Crash Recovery):
1. 从最近的Checkpoint加载数据
2. 重放WAL中的操作
3. 恢复到崩溃前的状态
```

---

## 🚀 优化器架构

### 优化器Pipeline

```
┌──────────────────────────────────────────────────────────┐
│                Optimizer Architecture                     │
└──────────────────────────────────────────────────────────┘

Logical Plan (from Binder)
    │
    ├─ LogicalFilter (age > 18)
    │   └─ LogicalProjection (name, age)
    │       └─ LogicalJoin (users.id = orders.user_id)
    │           ├─ LogicalGet (users)
    │           └─ LogicalGet (orders)
    │
    ▼
┌─────────────────────────────────────────────────────────┐
│              Optimization Passes                         │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  Pass 1: Expression Rewriter                             │
│    ├─ Constant Folding (2+3 → 5)                        │
│    ├─ Common Subexpression Elimination                  │
│    └─ Expression Simplification (x*1 → x)               │
│                                                           │
│  Pass 2: Filter Pushdown                                 │
│    └─ Move filters closer to data sources               │
│                                                           │
│  Pass 3: Join Order Optimization                         │
│    └─ Reorder joins using Dynamic Programming           │
│                                                           │
│  Pass 4: Column Pruning                                  │
│    └─ Remove unused columns from scans                  │
│                                                           │
│  Pass 5: Predicate Pushdown                              │
│    └─ Push predicates into scans                        │
│                                                           │
│  ... (40+ passes in total)                               │
│                                                           │
└─────────────────────────────────────────────────────────┘
    │
    ▼
Optimized Logical Plan
    │
    ├─ LogicalProjection (name)
    │   └─ LogicalJoin (users.id = orders.user_id)
    │       ├─ LogicalGet (users, filter: age>18, columns: [id, name, age])
    │       └─ LogicalGet (orders, columns: [user_id])
    │
    ▼
Physical Plan
```

---

### Filter Pushdown示例

```
┌──────────────────────────────────────────────────────────┐
│              Filter Pushdown Optimization                 │
└──────────────────────────────────────────────────────────┘

Before Optimization:
┌─────────────────────┐
│  LogicalProjection  │  SELECT name
└──────────┬──────────┘
           │
┌──────────▼──────────┐
│   LogicalFilter     │  WHERE age > 18
└──────────┬──────────┘
           │
┌──────────▼──────────┐
│    LogicalJoin      │  users JOIN orders
├──────────┬──────────┤
│          │          │
▼          ▼          ▼
LogicalGet  LogicalGet
(users)     (orders)

▼▼▼ Optimization ▼▼▼

After Optimization:
┌─────────────────────┐
│  LogicalProjection  │  SELECT name
└──────────┬──────────┘
           │
┌──────────▼──────────┐
│    LogicalJoin      │  users JOIN orders
├──────────┬──────────┤
│          │          │
▼          ▼          ▼
LogicalGet  LogicalGet
(users)     (orders)
filter:age>18   <── Filter pushed down!

Benefits:
- Scan fewer rows from users table
- Less data transferred through pipeline
- Smaller hash table for join
```

---

## 📊 性能监控可视化

### 查询执行时间分解

```
┌──────────────────────────────────────────────────────────┐
│              Query Execution Timeline                     │
└──────────────────────────────────────────────────────────┘

Total Time: 1000ms

┌────────────────────────────────────────────────────────┐
│ Parsing              ▓░░░░░░░░░░░░░░░░░░░░░░░░░░ 10ms  │
├────────────────────────────────────────────────────────┤
│ Binding              ▓▓░░░░░░░░░░░░░░░░░░░░░░░░ 20ms  │
├────────────────────────────────────────────────────────┤
│ Optimization         ▓▓▓░░░░░░░░░░░░░░░░░░░░░░░ 30ms  │
├────────────────────────────────────────────────────────┤
│ Execution            ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓ 940ms  │
│  ├─ TableScan (200ms)                                  │
│  ├─ HashJoin Build (300ms)                             │
│  ├─ HashJoin Probe (400ms)                             │
│  └─ Projection (40ms)                                  │
└────────────────────────────────────────────────────────┘

Execution占主导 (94%) → 优化执行阶段最重要
```

---

## 🎯 关键数字速查

```
┌──────────────────────────────────────────────────────────┐
│                  DuckDB Key Constants                     │
└──────────────────────────────────────────────────────────┘

STANDARD_VECTOR_SIZE = 2048       Vector中的元素数量
BLOCK_SIZE = 262,144 (256 KB)     磁盘块大小
ROW_GROUP_SIZE ≈ 122,880          RowGroup的最大行数

Memory Limits (默认):
- Buffer Pool: 80% of system memory
- Max memory per query: Buffer Pool size

Performance Targets:
- Vectorized operations: 2-8x speedup vs scalar
- Compression ratio: 5-20x (取决于数据)
- Filter Pushdown: 2-10x speedup
- Join Order Optimization: 10-100x+ speedup
```

---

## 📚 学习路径可视化

```
┌──────────────────────────────────────────────────────────┐
│              DuckDB Learning Roadmap                      │
└──────────────────────────────────────────────────────────┘

Week 1: Foundations
    ├─ Day 1-2: Architecture & Type System
    ├─ Day 3-4: Vector & DataChunk
    └─ Day 5-7: Expressions & Memory

Week 2: Query Execution
    ├─ Day 8-9: Binder & Logical Plan
    ├─ Day 10-11: Physical Operators & Scan
    └─ Day 12-14: Join & Aggregation

Week 3: Optimization
    ├─ Day 15-16: Optimizer & Filter Pushdown
    ├─ Day 17-18: Join Order & Statistics
    └─ Day 19-21: Expression Opt & Vectorization

Week 4: Storage & Advanced
    ├─ Day 22-24: Storage & Compression
    ├─ Day 25-27: MVCC & WAL & Buffer
    └─ Day 28-30: Projects & Integration

每周项目 ▼
    │
    ├─ Week 1: SimpleTable
    ├─ Week 2: SimpleQueryEngine
    ├─ Week 3: SimpleOptimizer
    ├─ Week 4: SimpleDiskStorage
    └─ Final: Mini-DuckDB
```

---

## 🔗 组件交互图

```
┌──────────────────────────────────────────────────────────┐
│              Component Interaction                        │
└──────────────────────────────────────────────────────────┘

┌──────────┐
│  Client  │
└────┬─────┘
     │ SQL Query
     ▼
┌────────────────┐
│    Parser      │────────────┐
└────┬───────────┘            │
     │ AST                    │
     ▼                        │
┌────────────────┐            │
│    Binder      │◄───────────┤ Catalog
└────┬───────────┘            │ (Tables, Types, ...)
     │ Logical Plan           │
     ▼                        │
┌────────────────┐            │
│   Optimizer    │◄───────────┘ Statistics
└────┬───────────┘              (NDV, Min/Max, ...)
     │ Optimized Plan
     ▼
┌────────────────┐
│ Physical Plan  │
│    Generator   │
└────┬───────────┘
     │ Physical Plan
     ▼
┌────────────────────────────┐
│  Execution Engine          │
│  ┌──────────────────────┐ │
│  │  Pipeline Executor   │ │
│  └──────────┬───────────┘ │
│             │ Data Access │
│             ▼             │
│  ┌──────────────────────┐ │
│  │  Storage Engine      │ │
│  │  ├─ RowGroups        │ │
│  │  ├─ BufferManager    │ │
│  │  └─ WAL              │ │
│  └──────────────────────┘ │
└────────────────────────────┘
```

---

**提示:** 将这些图保存下来，在学习过程中经常回顾，有助于建立整体的架构理解！

---

最后更新：2026-01-17
