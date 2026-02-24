# DuckDB 深度课程：向量化执行引擎

> 本课程深入讲解DuckDB的向量化执行引擎，这是DuckDB高性能的核心秘密。通过本课程，你将完全理解向量化执行的设计原理和实现细节。

---

## 课程概览

### 学习目标

- 理解向量化执行的理论基础和优势
- 掌握Vector和DataChunk的设计与实现
- 学习不同Vector类型的转换机制
- 理解零拷贝操作和缓存优化
- 掌握向量化算子的实现模式
- 能够实现高性能的向量化操作

### 前置知识

- C++11及以上知识
- 计算机体系结构基础（缓存、SIMD）
- 数据库查询执行基础
- 模板元编程基础

---

## 第一部分：向量化执行理论基础

### 1.1 行式vs向量化执行

#### 行式执行（Volcano模型）

```
传统行式执行流程：
┌─────────┐
│ Operator│  Next() → tuple → Next() → tuple → Next() → tuple
└────┬────┘
     ↓
  每次处理1行
  • 函数调用开销大
  • CPU缓存利用率低
  • 无法利用SIMD
```

**行式执行示例代码：**

```cpp
// Volcano模型：每次返回一行
class RowOperator {
    virtual tuple_t Next() = 0;
};

class ScanOperator : public RowOperator {
    tuple_t Next() override {
        if (current_row < table->RowCount()) {
            return table->GetRow(current_row++);
        }
        return nullptr;
    }
};

// 执行流程：频繁的虚函数调用
while (auto row = scan->Next()) {
    if (row->GetInt(0) > 100) {  // 每行一次比较
        output->Collect(row);
    }
}

// 性能问题：
// 1. 每行一次虚函数调用
// 2. CPU缓存行浪费
// 3. 无法利用SIMD指令
// 4. 分支预测失败率高
```

#### 向量化执行

```
向量化执行流程：
┌─────────┐
│ Operator│  Execute() → DataChunk (2048 rows) → Execute() → DataChunk
└────┬────┘
     ↓
  每次处理2048行
  • 函数调用开销小
  • CPU缓存命中率高
  • 充分利用SIMD
```

**向量化执行示例代码：**

```cpp
// 向量化模型：每次返回2048行
class VectorizedOperator {
    virtual void Execute(DataChunk &input, DataChunk &output) = 0;
};

class VectorizedFilter : public VectorizedOperator {
    void Execute(DataChunk &input, DataChunk &output) override {
        // 批量处理2048行
        SelectionVector sel;
        idx_t count = 0;

        auto &input_vec = input.data[0];
        auto input_data = FlatVector::GetData<int32_t>(input_vec);

        // SIMD优化的循环
        for (idx_t i = 0; i < input.size(); i++) {
            if (input_data[i] > 100) {
                sel.set_index(count++, i);
            }
        }

        // 零拷贝：只复制选择的行
        output.Slice(input, sel, count);
    }
};

// 执行流程：大幅减少函数调用
while (true) {
    DataChunk chunk;
    scan->Execute(chunk);  // 一次获取2048行
    if (chunk.size() == 0) break;

    filter->Execute(chunk, output);  // 批量处理
}

// 性能优势：
// 1. 函数调用减少2048倍
// 2. CPU缓存行高效利用
// 3. 编译器自动向量化（SIMD）
// 4. 分支预测友好
```

### 1.2 性能对比分析

#### CPU缓存利用

```
L1 Cache: 32KB - 64KB
L2 Cache: 256KB - 512KB
L3 Cache: 8MB - 32MB
RAM:      GB级

行式执行：
• 每次访问1行（如100字节）
• 2048行 = 204KB数据
• 频繁的L1/L2 Cache Miss

向量化执行：
• 连续访问2048行（204KB）
• 列式存储：2048 * 4字节 = 8KB（整数列）
• 全部在L1 Cache内！
```

**理论加速比计算：**

```cpp
// 内存访问对比

// 行式：缓存未命中主导
// 每行访问假设：
// - 列1: 缓存未命中
// - 列2: 缓存未命中
// - 列3: 缓存未命中
// 总计：3次缓存未命中/行

// 向量化：缓存命中主导
// 每2048行访问：
// - 列1: 2048次缓存命中
// - 列2: 2048次缓存命中
// - 列3: 2048次缓存命中
// 总计：0次缓存未命中/2048行

// 理论加速：
// 行式：2048行 * 3次未命中 = 6144次未命中
// 向量：2048行 / 2048 * 3次未命中 = 3次未命中
// 加速比 = 6144 / 3 = 2048倍
```

#### SIMD指令利用

```
AVX2指令集：256位寄存器
• int32: 8个元素/指令
• int16: 16个元素/指令
• int8:  32个元素/指令

向量化执行：
2048行 / 8 = 256次AVX2指令
vs
2048次标量指令

理论加速：8倍（int32）
```

```cpp
// 编译器自动向量化示例
void VectorAdd(int32_t *a, int32_t *b, int32_t *c, idx_t count) {
    for (idx_t i = 0; i < count; i++) {
        c[i] = a[i] + b[i];
    }
}

// GCC -O3 -march=haswell 编译后：
// vpaddd  ymm0, ymm1, ymm2  // 一次加8个int32
// vmovdqa ymmword [rdx+rax*4], ymm0

// 实测性能：
// 标量版本：2048次循环 = ~2048条指令
// AVX2版本：256次循环 = ~256条指令
// 加速比：8倍
```

### 1.3 向量化执行的适用场景

| 操作类型 | 行式执行 | 向量化执行 | 加速比 |
|---------|---------|-----------|-------|
| 简单过滤 | 1x | 10-50x | 高 |
| 聚合计算 | 1x | 5-20x | 中高 |
| JOIN | 1x | 2-10x | 中 |
| 排序 | 1x | 2-5x | 低中 |
| 复杂表达式 | 1x | 5-30x | 高 |

---

## 第二部分：Vector类详解

### 2.1 Vector类结构

```cpp
// src/include/duckdb/common/types/vector.hpp

class Vector {
public:
    // 向量存储类型
    VectorType vector_type;

    // 逻辑类型（用户可见类型）
    LogicalType type;

    // 数据指针（指向实际数据）
    data_ptr_t data;

    // 有效性掩码（NULL值位图）
    ValidityMask validity;

    // 主数据缓冲区（拥有数据内存）
    buffer_ptr<VectorBuffer> buffer;

    // 辅助缓冲区（字典向量的子向量等）
    buffer_ptr<VectorBuffer> auxiliary;

    // 构造函数
    Vector(LogicalType type);
    Vector(LogicalType type, VectorType vector_type);
    Vector(Vector &other);

    // 引用另一个向量（零拷贝）
    void Reference(Vector &other);
    void Slice(Vector &other, SelectionVector &sel, idx_t count);

    // 类型转换
    void Cast(LogicalType target_type, idx_t count);

    // 统一格式访问
    void ToUnifiedFormat(idx_t count, UnifiedVectorFormat &result);

    // 获取向量类型
    inline VectorType GetVectorType() {
        return vector_type;
    }

    inline LogicalType GetType() {
        return type;
    }
};
```

### 2.2 VectorType枚举

```cpp
// src/include/duckdb/common/types/vector.hpp

enum class VectorType : uint8_t {
    // 标准平面向量：直接存储数据
    FLAT_VECTOR = 1,

    // FSST压缩的字符串向量
    FSST_VECTOR = 2,

    // 常量向量：所有元素都是同一个值
    CONSTANT_VECTOR = 3,

    // 字典向量：使用选择向量引用另一个向量
    DICTIONARY_VECTOR = 4,

    // 序列向量：表示算术序列
    SEQUENCE_VECTOR = 5
};
```

#### FLAT_VECTOR（平面向量）

```
FLAT_VECTOR结构：
┌────────────────────────────────────────┐
│ ValidityMask (64位 * 32 = 256字节)   │ ← NULL值位图
├────────────────────────────────────────┤
│ Data: [val0, val1, val2, ..., valN]  │ ← 实际数据
└────────────────────────────────────────┘

示例：INTEGER向量
┌────────────────────────────────────────┐
│ Validity: 0xFFFFFFFFFFFFFFFF         │ ← 全部有效
├────────────────────────────────────────┤
│ Data:    [10, 20, 30, 40, 50, ...]  │
└────────────────────────────────────────┘

内存布局（小端序）：
[0A 00 00 00] [14 00 00 00] [1E 00 00 00] ...
  10           20           30
```

#### CONSTANT_VECTOR（常量向量）

```
CONSTANT_VECTOR结构：
┌────────────────────────────────────────┐
│ ValidityMask (单个位)                │
├────────────────────────────────────────┤
│ Data:    [value]                     │ ← 单个值
└────────────────────────────────────────┘

特点：
• 只存储一个值
• 逻辑上表示2048个相同的值
• 常用于常量表达式优化

示例：
SELECT 42 FROM range(2048);
→ CONSTANT_VECTOR with value = 42
→ 避免存储2048次"42"
```

#### DICTIONARY_VECTOR（字典向量）

```
DICTIONARY_VECTOR结构：
┌────────────────────────────────────────┐
│ SelectionVector (sel_t * 2048)       │ ← 索引数组
│   [idx0, idx1, idx2, ..., idxN]     │
├────────────────────────────────────────┤
│ Auxiliary → Child Vector              │ ← 引用的子向量
└────────────────────────────────────────┘

用途：零拷贝地表示过滤结果

示例：过滤 value > 10
原始数据：[5, 15, 8, 20, 3, 25]
结果索引：[1, 3, 5]  (值为15, 20, 25)

DICTIONARY_VECTOR:
┌────────────────────────────────────────┐
│ SelVector: [1, 3, 5]                │
├────────────────────────────────────────┤
│ Child → [5, 15, 8, 20, 3, 25]      │
└────────────────────────────────────────┘

访问操作：
result[i] = child[sel_vector[i]]
result[0] = child[1] = 15
result[1] = child[3] = 20
result[2] = child[5] = 25
```

### 2.3 ValidityMask详解

```cpp
// src/include/duckdb/common/types validity_mask.hpp

class ValidityMask {
public:
    // 每个validity_t是64位，存储64行的有效性
    static constexpr const idx_t BITS_PER_VALUE = 64;

    // 有效位指针
    validity_t *validity_mask;

    // 获取行的有效性
    inline bool RowIsValid(idx_t row_idx) const {
        if (!validity_mask) {
            return true;  // 无掩码 = 全部有效
        }
        idx_t entry_idx, idx_in_entry;
        GetEntryIndex(row_idx, entry_idx, idx_in_entry);
        auto entry = GetValidityEntryUnsafe(entry_idx);
        return entry & (validity_t(1) << idx_in_entry);
    }

    // 设置行无效
    inline void SetInvalid(idx_t row_idx) {
        D_ASSERT(validity_mask);
        idx_t entry_idx, idx_in_entry;
        GetEntryIndex(row_idx, entry_idx, idx_in_entry);
        auto &entry = GetValidityEntryUnsafe(entry_idx);
        entry &= ~(validity_t(1) << idx_in_entry);
    }

    // 设置行有效
    inline void SetValid(idx_t row_idx) {
        D_ASSERT(validity_mask);
        idx_t entry_idx, idx_in_entry;
        GetEntryIndex(row_idx, entry_idx, idx_in_entry);
        auto &entry = GetValidityEntryUnsafe(entry_idx);
        entry |= (validity_t(1) << idx_in_entry);
    }

    // 检查是否全部有效
    inline bool AllValid() const {
        return !validity_mask;
    }

    // 设置全部有效
    inline void SetAllValid(idx_t count) {
        if (validity_mask) {
            for (idx_t i = 0; i < ValidityMask::EntryCount(count); i++) {
                validity_mask[i] = ~validity_t(0);
            }
        }
    }

private:
    // 计算位图中的位置
    static inline void GetEntryIndex(idx_t row_idx,
                                     idx_t &entry_idx,
                                     idx_t &idx_in_entry) {
        entry_idx = row_idx / BITS_PER_VALUE;
        idx_in_entry = row_idx % BITS_PER_VALUE;
    }

    inline validity_t &GetValidityEntryUnsafe(idx_t entry_idx) {
        return validity_mask[entry_idx];
    }
};
```

**ValidityMask布局示例：**

```
2048行的ValidityMask：
┌────────┬────────┬────────┬───────┬────────┐
│ Entry0 │ Entry1 │ Entry2 │ ...   │ Entry31│
│ 64位   │ 64位   │ 64位   │       │ 64位   │
└────────┴────────┴────────┴───────┴────────┘

每个Entry存储64行的有效性状态：
Entry0:
┌─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┬─┐
│0│1│2│3│4│5│6│7│8│...                                │63│
└─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┴─┘
 ↑               ↑                           ↑
行0有效        行2无效                     行63有效

内存需求：2048行 / 64位每entry = 32个entry
32 * 8字节 = 256字节

空间效率：256字节 / 2048行 = 0.125字节/行
```

### 2.4 SelectionVector详解

```cpp
// src/include/duckdb/common/types selection_vector.hpp

struct SelectionVector {
    // 选择数据类型（16位无符号整数）
    using sel_t = uint16_t;
    static constexpr const idx_t MAX_VECTOR_SIZE = 65536;

    // 选择数组指针
    sel_t *sel_vector;

    // 拥有的数据（用于管理生命周期）
    buffer_ptr<SelectionData> selection_data;

    // 构造函数
    SelectionVector();
    explicit SelectionVector(idx_t count);
    SelectionVector(sel_t *sel_vector);

    // 设置索引
    inline void set_index(idx_t idx, sel_t value) {
        sel_vector[idx] = value;
    }

    // 获取索引
    inline sel_t get_index(idx_t idx) const {
        return sel_vector[idx];
    }

    // 初始化为增量序列
    void Initialize(idx_t count);

    // 从另一个SelectionVector切片
    void Slice(const SelectionVector &other, idx_t count);
};
```

**SelectionVector使用场景：**

```
场景1：过滤结果
原始数据（INTEGER）: [10, 20, 30, 40, 50, 60, 70, 80]
过滤条件: value > 35
满足条件的索引: [3, 4, 5, 6, 7]

SelectionVector:
[3, 4, 5, 6, 7]

结果数据:
通过sel_vector引用原始数据
result[0] = data[sel_vector[0]] = data[3] = 40
result[1] = data[sel_vector[1]] = data[4] = 50
...

场景2：Join结果
左表索引: [0, 1, 2, 3]
右表索引: [5, 5, 8, 10]

两个SelectionVector表示Join对
result[0] = (left[0], right[5])
result[1] = (left[1], right[5])
result[2] = (left[2], right[8])
result[3] = (left[3], right[10])
```

### 2.5 Vector操作示例

#### 创建和初始化Vector

```cpp
// 1. 创建空Vector
Vector vec(LogicalType::INTEGER);

// 2. 创建并初始化为常量
Vector const_vec(LogicalType::INTEGER, VectorType::CONSTANT_VECTOR);
ConstantVector::SetNull(const_vec, false);
auto const_data = ConstantVector::GetData<int32_t>(const_vec);
const_data[0] = 42;

// 3. 创建平面Vector并初始化数据
Vector flat_vec(LogicalType::INTEGER);
flat_vec.Initialize(100);  // 分配100个元素空间

auto flat_data = FlatVector::GetData<int32_t>(flat_vec);
for (idx_t i = 0; i < 100; i++) {
    flat_data[i] = i * 2;
}

// 4. 设置NULL值
flat_vec.Flatten(100);
FlatVector::SetNull(flat_vec, 50, true);  // 第50行设为NULL
```

#### Vector引用和切片

```cpp
// 原始Vector
Vector original(LogicalType::INTEGER);
auto orig_data = FlatVector::GetData<int32_t>(original);
for (idx_t i = 0; i < 2048; i++) {
    orig_data[i] = i;
}

// 1. 引用Vector（零拷贝）
Vector ref(LogicalType::INTEGER);
ref.Reference(original);
// ref.data == original.data
// ref.validity == original.validity

// 2. 创建SelectionVector
SelectionVector sel(10);  // 选择10个元素
for (idx_t i = 0; i < 10; i++) {
    sel.set_index(i, i * 100);  // 选择索引0, 100, 200, ...
}

// 3. 切片Vector
Vector sliced(LogicalType::INTEGER);
sliced.Slice(original, sel, 10);
// sliced.vector_type == DICTIONARY_VECTOR
// sliced.auxiliary → original
// sliced.buffer → sel
```

#### Vector类型转换

```cpp
// INTEGER → BIGINT
Vector int_vec(LogicalType::INTEGER);
// ... 初始化数据

Vector bigint_vec(LogicalType::BIGINT);
int_vec.Cast(bigint_vec, 2048);

// VARCHAR → INTEGER（解析）
Vector varchar_vec(LogicalType::VARCHAR);
// ... 初始化字符串数据

Vector int_vec(LogicalType::INTEGER);
varchar_vec.Cast(int_vec, 2048);
// 字符串 "123" → 整数 123
// 无效字符串 → NULL
```

---

## 第三部分：DataChunk类详解

### 3.1 DataChunk结构

```cpp
// src/include/duckdb/common/types data_chunk.hpp

class DataChunk {
public:
    // 向量数组（列）
    vector<Vector> data;

    // 当前行数
    idx_t count;

    // 容量
    idx_t capacity;

    // 初始容量
    idx_t initial_capacity;

    // 向量缓存（用于重用分配的内存）
    vector<VectorCache> vector_caches;

    // 构造函数
    DataChunk();
    explicit DataChunk(idx_t capacity);

    // 初始化列
    void Initialize(vector<LogicalType> types);
    void InitializeEmpty(vector<LogicalType> types);
    void Initialize(vector<LogicalType> types, idx_t capacity);

    // 获取列数
    idx_t ColumnCount() {
        return data.size();
    }

    // 获取大小
    idx_t Size() const {
        return count;
    }

    // 重置数据块
    void Reset();
    void Destroy();

    // 设置容量
    void SetCapacity(idx_t capacity);

    // 设置基数
    void SetCardinality(idx_t count);

    // 引用另一个数据块
    void Reference(DataChunk &other);

    // 切片操作
    void Slice(DataChunk &other, SelectionVector &sel, idx_t count);
    void Slice(const SelectionVector &sel, idx_t count);

    // 验证数据块
    void Verify();

    // 获取列向量
    Vector &GetData(idx_t col_idx);
    Vector *GetVector(idx_t col_idx);
};
```

### 3.2 DataChunk内存布局

```
DataChunk内存布局（3列，2048行）：

┌─────────────────────────────────────────────────────────┐
│                      DataChunk                        │
├─────────────────────────────────────────────────────────┤
│  count = 2048                                        │
│  capacity = 2048                                     │
│  data.size() = 3                                     │
├─────────────────────────────────────────────────────────┤
│                                                        │
│  data[0] (INTEGER)                                  │
│  ┌───────────────────────────────────────┐          │
│  │ ValidityMask (256 bytes)              │          │
│  ├───────────────────────────────────────┤          │
│  │ Data: [10, 20, 30, ..., 20480]     │          │
│  │       (8192 bytes)                  │          │
│  └───────────────────────────────────────┘          │
│                                                        │
│  data[1] (VARCHAR)                                  │
│  ┌───────────────────────────────────────┐          │
│  │ ValidityMask (256 bytes)              │          │
│  ├───────────────────────────────────────┤          │
│  │ Data: [string_t, string_t, ...]     │          │
│  │       (16 bytes * 2048)              │          │
│  ├───────────────────────────────────────┤          │
│  │ Auxiliary (heap strings)              │          │
│  └───────────────────────────────────────┘          │
│                                                        │
│  data[2] (DOUBLE)                                   │
│  ┌───────────────────────────────────────┐          │
│  │ ValidityMask (256 bytes)              │          │
│  ├───────────────────────────────────────┤          │
│  │ Data: [1.5, 2.5, 3.5, ..., ...]    │          │
│  │       (16384 bytes)                  │          │
│  └───────────────────────────────────────┘          │
│                                                        │
└─────────────────────────────────────────────────────────┘

总内存：~ 30KB（不含字符串堆）
全部可放入L1 Cache！
```

### 3.3 DataChunk操作示例

#### 初始化DataChunk

```cpp
// 1. 创建空的DataChunk
DataChunk chunk;

// 2. 初始化列类型
vector<LogicalType> types = {
    LogicalType::INTEGER,
    LogicalType::VARCHAR,
    LogicalType::DOUBLE
};
chunk.Initialize(types);

// 3. 设置行数
chunk.SetCardinality(100);

// 4. 访问列数据
auto &int_col = chunk.data[0];
auto int_data = FlatVector::GetData<int32_t>(int_col);

auto &varchar_col = chunk.data[1];
auto varchar_data = FlatVector::GetData<string_t>(varchar_col);

auto &double_col = chunk.data[2];
auto double_data = FlatVector::GetData<double>(double_col);

// 5. 填充数据
for (idx_t i = 0; i < 100; i++) {
    int_data[i] = i;
    varchar_data[i] = StringVector::AddString(varchar_col,
                                              "value_" + to_string(i));
    double_data[i] = i * 1.5;
}
```

#### DataChunk引用和切片

```cpp
// 原始DataChunk
DataChunk original;
original.Initialize({LogicalType::INTEGER, LogicalType::VARCHAR});
// ... 填充数据

// 1. 引用DataChunk（零拷贝）
DataChunk ref_chunk;
ref_chunk.Reference(original);
// ref_chunk.data[0] 引用 original.data[0]
// ref_chunk.data[1] 引用 original.data[1]

// 2. 切片DataChunk
SelectionVector sel(50);
// ... 设置选择索引

DataChunk sliced;
sliced.Reference(original);
sliced.Slice(sel, 50);
// sliced只包含选中的50行
```

#### DataChunk合并

```cpp
// 合并多个DataChunk
vector<DataChunk> chunks;
// ... 初始化多个chunk

DataChunk merged;
merged.Initialize(chunks[0].GetTypes());

idx_t total_size = 0;
for (auto &chunk : chunks) {
    merged.Append(chunk);
    total_size += chunk.size();
}

// 或者一次性合并
merged.Append(chunks);
```

---

## 第四部分：向量化算子实现

### 4.1 Filter算子

```cpp
// src/execution/operator/filter/physical_filter.cpp

class PhysicalFilter : public PhysicalOperator {
public:
    PhysicalFilter(vector<unique_ptr<Expression>> select_expressions,
                  idx_t estimated_cardinality)
        : PhysicalOperator(PhysicalOperatorType::FILTER, {}, estimated_cardinality),
          expressions(std::move(select_expressions)) {}

    unique_ptr<OperatorState> GetOperatorState(ExecutionContext &context) override {
        return make_unique<FilterOperatorState>();
    }

    // 执行过滤
    OperatorResultType Execute(ExecutionContext &context,
                               DataChunk &input,
                               DataChunk &output,
                               GlobalOperatorState &gstate,
                               OperatorState &state) override {
        auto &fstate = (FilterOperatorState &)state;

        // 1. 执行过滤表达式
        SelectionVector sel;
        idx_t sel_count;

        // 计算过滤条件
        ExecuteExpressions(input, fstate.executor, expressions);

        // 2. 获取过滤结果
        auto &result_vector = expressions[0];
        D_ASSERT(result_vector.type == LogicalType::BOOLEAN);

        // 3. 从布尔向量提取SelectionVector
        UnifiedVectorFormat vdata;
        result_vector.ToUnifiedFormat(input.size(), vdata);

        // 执行过滤
        sel_count = FilterSelectionsel(vdata, input.size(), sel);

        // 4. 如果没有行通过过滤
        if (sel_count == 0) {
            output.SetCardinality(0);
            return OperatorResultType::NEED_MORE_INPUT;
        }

        // 5. 如果所有行都通过过滤
        if (sel_count == input.size()) {
            output.Reference(input);
            return OperatorResultType::HAVE_MORE_OUTPUT;
        }

        // 6. 部分行通过过滤：切片
        output.Slice(input, sel, sel_count);
        return OperatorResultType::HAVE_MORE_OUTPUT;
    }

private:
    vector<unique_ptr<Expression>> expressions;
};

// 过滤选择实现
static idx_t FilterSelectionSel(UnifiedVectorFormat &vdata, idx_t count,
                                SelectionVector &sel) {
    idx_t sel_count = 0;

    // 处理NULL
    if (vdata.validity.AllValid()) {
        // 没有NULL值
        for (idx_t i = 0; i < count; i++) {
            idx_t idx = vdata.sel->get_index(i);
            if (((bool *)vdata.data)[idx]) {
                sel.set_index(sel_count++, i);
            }
        }
    } else {
        // 有NULL值
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

### 4.2 Projection算子

```cpp
// src/execution/operator/projection/physical_projection.cpp

class PhysicalProjection : public PhysicalOperator {
public:
    PhysicalProjection(vector<unique_ptr<Expression>> select_expressions,
                     idx_t estimated_cardinality)
        : PhysicalOperator(PhysicalOperatorType::PROJECTION, {},
                          estimated_cardinality),
          expressions(std::move(select_expressions)) {}

    unique_ptr<OperatorState> GetOperatorState(ExecutionContext &context) override {
        return make_unique<ProjectionOperatorState>();
    }

    // 执行投影
    OperatorResultType Execute(ExecutionContext &context,
                               DataChunk &input,
                               DataChunk &output,
                               GlobalOperatorState &gstate,
                               OperatorState &state) override {
        auto &pstate = (ProjectionOperatorState &)state;

        // 1. 确保输出DataChunk已初始化
        if (output.data.empty()) {
            output.Initialize(types);
        }

        // 2. 执行投影表达式
        ExecuteExpressions(input, pstate.executor, expressions);

        // 3. 设置输出列
        for (idx_t i = 0; i < expressions.size(); i++) {
            output.data[i].Reference(expressions[i]);
        }

        // 4. 设置基数
        output.SetCardinality(input.size());

        return OperatorResultType::HAVE_MORE_OUTPUT;
    }

private:
    vector<unique_ptr<Expression>> expressions;
    vector<LogicalType> types;
};
```

### 4.3 Aggregate算子

```cpp
// 向量化聚合实现

class PhysicalAggregate : public PhysicalOperator {
public:
    struct AggregateState {
        vector<unique_ptr<AggregateStateData>> states;
        vector<Vector> group_vectors;
        idx_t group_count;
    };

    // 执行聚合
    OperatorResultType Execute(ExecutionContext &context,
                               DataChunk &input,
                               DataChunk &output,
                               GlobalOperatorState &gstate,
                               OperatorState &state) override {
        auto &astate = (AggregateOperatorState &)state;

        // 1. 分组Key计算
        DataChunk group_chunk;
        ComputeGroupKeys(input, group_chunk);

        // 2. 聚合状态更新
        UpdateAggregates(input, group_chunk, astate);

        // 3. 收集结果
        if (Finished(astate)) {
            FinalizeAggregates(astate, output);
            return OperatorResultType::FINISHED;
        }

        return OperatorResultType::NEED_MORE_INPUT;
    }

private:
    void UpdateAggregates(DataChunk &input, DataChunk &group_chunk,
                          AggregateOperatorState &astate) {
        // 1. 哈希分组Key
        auto hashes = ComputeHashes(group_chunk);

        // 2. 查找/创建分组
        for (idx_t i = 0; i < input.size(); i++) {
            idx_t group_idx = FindOrCreateGroup(hashes[i], group_chunk, i);

            // 3. 更新聚合状态
            for (idx_t agg_idx = 0; agg_idx < aggregates.size(); agg_idx++) {
                aggregates[agg_idx]->Update(
                    astate.states[agg_idx].get(),
                    group_idx,
                    input,
                    i
                );
            }
        }
    }

    void FinalizeAggregates(AggregateOperatorState &astate, DataChunk &output) {
        output.InitializeEmpty(output_types);

        // 设置分组列
        for (idx_t i = 0; i < group_keys.size(); i++) {
            output.data[i].Reference(astate.group_vectors[i]);
        }

        // Finalize聚合
        for (idx_t i = 0; i < aggregates.size(); i++) {
            aggregates[i]->Finalize(
                astate.states[i].get(),
                output.data[group_keys.size() + i],
                astate.group_count
            );
        }

        output.SetCardinality(astate.group_count);
    }
};
```

### 4.4 Hash Join算子

```cpp
// 简化的Hash Join实现

class PhysicalHashJoin : public PhysicalOperator {
public:
    struct HashJoinState {
        // 构建阶段：右表的哈希表
        unique_ptr<HashTable> hash_table;

        // 探针阶段：左表探针
        unique_ptr<ProbeState> probe_state;
    };

    unique_ptr<GlobalOperatorState> GetGlobalState(ClientContext &context) override {
        auto state = make_unique<HashJoinGlobalState>();
        state->hash_table = make_unique<HashTable>(condition);
        return state;
    }

    unique_ptr<OperatorState> GetOperatorState(ExecutionContext &context) override {
        auto state = make_unique<HashJoinOperatorState>();
        state->probe_state = make_unique<ProbeState>();
        return state;
    }

    // 执行Join
    OperatorResultType Execute(ExecutionContext &context,
                               DataChunk &input,
                               DataChunk &output,
                               GlobalOperatorState &gstate,
                               OperatorState &state) override {
        auto &hstate = (HashJoinGlobalState &)gstate;
        auto &jstate = (HashJoinOperatorState &)state;

        // 构建阶段：处理右表
        if (!hstate->initialized) {
            BuildHashTable(hstate);
            hstate->initialized = true;
        }

        // 探针阶段：左表查找
        ProbeHashTable(input, output, jstate, hstate);

        return OperatorResultType::HAVE_MORE_OUTPUT;
    }

private:
    void BuildHashTable(HashJoinGlobalState &state) {
        // 扫描右表
        DataChunk right_chunk;
        right_chunk.Initialize(right_types);

        while (true) {
            children[1]->Execute(context, right_chunk, *state.probe_state);
            if (right_chunk.size() == 0) break;

            // 插入哈希表
            state.hash_table->Insert(right_chunk);
        }

        // Finalize哈希表
        state.hash_table->Finalize();
    }

    void ProbeHashTable(DataChunk &input, DataChunk &output,
                       HashJoinOperatorState &jstate,
                       HashJoinGlobalState &hstate) {
        // 1. 计算左表哈希值
        auto hashes = ComputeHashes(input);

        // 2. 探针哈希表
        SelectionVector sel;
        idx_t match_count;

        match_count = hstate.hash_table->Probe(
            input,
            hashes,
            sel,
            jstate.probe_state.get()
        );

        // 3. 构建结果
        if (match_count > 0) {
            output.Initialize(output_types);
            ConstructResult(input, sel, match_count, output);
            output.SetCardinality(match_count);
        } else {
            output.SetCardinality(0);
        }
    }

    void ConstructResult(DataChunk &input, SelectionVector &sel,
                         idx_t count, DataChunk &output) {
        // 左表列
        for (idx_t i = 0; i < input.ColumnCount(); i++) {
            output.data[i].Reference(input.data[i]);
            output.data[i].Slice(input.data[i], sel, count);
        }

        // 右表列（从哈希表获取）
        for (idx_t i = 0; i < right_types.size(); i++) {
            hash_table->GetColumn(i, output.data[input.ColumnCount() + i]);
        }
    }
};
```

---

## 第五部分：性能优化实践

### 5.1 SIMD优化

```cpp
// 使用SIMD intrinsics优化

#include <immintrin.h>

// 标量版本
void ScalarAdd(int32_t *a, int32_t *b, int32_t *c, idx_t count) {
    for (idx_t i = 0; i < count; i++) {
        c[i] = a[i] + b[i];
    }
}

// AVX2版本（8个int32/指令）
void SIMDAdd(int32_t *a, int32_t *b, int32_t *c, idx_t count) {
    idx_t i = 0;

    // AVX2主循环
    for (; i + 8 <= count; i += 8) {
        __m256i va = _mm256_loadu_si256((__m256i*)&a[i]);
        __m256i vb = _mm256_loadu_si256((__m256i*)&b[i]);
        __m256i vc = _mm256_add_epi32(va, vb);
        _mm256_storeu_si256((__m256i*)&c[i], vc);
    }

    // 处理剩余元素
    for (; i < count; i++) {
        c[i] = a[i] + b[i];
    }
}

// AVX-512版本（16个int32/指令）
void AVX512Add(int32_t *a, int32_t *b, int32_t *c, idx_t count) {
    idx_t i = 0;

    // AVX-512主循环
    for (; i + 16 <= count; i += 16) {
        __m512i va = _mm512_loadu_si512((__m512i*)&a[i]);
        __m512i vb = _mm512_loadu_si512((__m512i*)&b[i]);
        __m512i vc = _mm512_add_epi32(va, vb);
        _mm512_storeu_si512((__m512i*)&c[i], vc);
    }

    // 处理剩余元素
    for (; i < count; i++) {
        c[i] = a[i] + b[i];
    }
}

// 性能对比（Intel Core i7，count = 1,000,000）：
// Scalar:   ~1.5ms
// AVX2:     ~0.3ms (5x加速)
// AVX-512:  ~0.2ms (7.5x加速)
```

### 5.2 缓存优化

```cpp
// 缓存友好的数据访问

// ❌ 差：缓存未友好
void CacheUnfriendlySum(int32_t **matrix, idx_t rows, idx_t cols) {
    int64_t sum = 0;

    // 列主序访问（跳跃访问）
    for (idx_t j = 0; j < cols; j++) {
        for (idx_t i = 0; i < rows; i++) {
            sum += matrix[i][j];  // 每次访问跳过一行
        }
    }
    // 假设1024x1024矩阵
    // 每行8KB，每次跳跃8KB
    // 大量缓存未命中
}

// ✅ 好：缓存友好
void CacheFriendlySum(int32_t **matrix, idx_t rows, idx_t cols) {
    int64_t sum = 0;

    // 行主序访问（连续访问）
    for (idx_t i = 0; i < rows; i++) {
        for (idx_t j = 0; j < cols; j++) {
            sum += matrix[i][j];  // 连续访问
        }
    }
    // 数据连续加载到缓存
    // 大量缓存命中
}

// 性能对比（1024x1024矩阵）：
// CacheUnfriendly: ~15ms
// CacheFriendly:     ~3ms (5x加速)
```

### 5.3 预取优化

```cpp
// 软件预取

void PrefetchOptimizedSum(int32_t *a, int32_t *b, idx_t count) {
    const idx_t PREFETCH_DISTANCE = 8;  // 预取距离

    int64_t sum = 0;
    idx_t i = 0;

    // 主循环：带预取
    for (; i + PREFETCH_DISTANCE < count; i++) {
        // 预取未来的数据
        __builtin_prefetch(&a[i + PREFETCH_DISTANCE], 0, 3);
        __builtin_prefetch(&b[i + PREFETCH_DISTANCE], 0, 3);

        // 处理当前数据
        sum += a[i] + b[i];
    }

    // 处理剩余元素
    for (; i < count; i++) {
        sum += a[i] + b[i];
    }
}

// 性能对比（大数组 > 1MB）：
// 无预取:  ~10ms
// 有预取:   ~6ms (1.7x加速)

// 预取参数说明：
// __builtin_prefetch(addr, rw, locality)
// - addr: 预取地址
// - rw: 0=读取, 1=写入
// - locality: 0-3，3=数据留在L1缓存
```

### 5.4 零拷贝优化

```cpp
// 避免不必要的数据拷贝

// ❌ 差：多次拷贝
DataChunk BadFilter(Vector &input) {
    // 1. 拷贝输入
    DataChunk result;
    result.Reference(input);

    // 2. 执行过滤
    SelectionVector sel;
    idx_t count = ApplyFilter(input, sel);

    // 3. 拷贝结果（深拷贝！）
    DataChunk final_result;
    final_result.Initialize(result.GetTypes());
    for (idx_t col = 0; col < result.ColumnCount(); col++) {
        final_result.data[col].Copy(result.data[col], sel, count);
    }

    return final_result;
    // 总计：3次拷贝
}

// ✅ 好：零拷贝
DataChunk GoodFilter(Vector &input) {
    DataChunk result;
    result.Reference(input);

    SelectionVector sel;
    idx_t count = ApplyFilter(input, sel);

    // 零拷贝切片
    result.Slice(sel, count);
    return result;
    // 总计：0次拷贝
}

// 性能对比（100万行）：
// BadFilter:   ~50ms
// GoodFilter:  ~5ms (10x加速)
```

---

## 第六部分：实战项目

### 项目：实现向量化字符串长度函数

```cpp
// string_length.hpp

#include "duckdb/common/types/vector.hpp"
#include "duckdb/common/vector_operations/vector_operations.hpp"

namespace duckdb {

// 字符串长度函数（向量化实现）
static void StringLengthFunction(DataChunk &args,
                                 ExpressionState &state,
                                 Vector &result) {
    // 输入向量
    auto &input_vector = args.data[0];
    D_ASSERT(input_vector.type == LogicalType::VARCHAR);

    // 统一格式化（处理不同向量类型）
    UnifiedVectorFormat input_data;
    input_vector.ToUnifiedFormat(args.size(), input_data);

    // 输出向量
    result.Initialize(LogicalType::INTEGER);
    auto result_data = FlatVector::GetData<int32_t>(result);

    // 批量处理
    for (idx_t i = 0; i < args.size(); i++) {
        // 获取实际索引
        idx_t idx = input_data.sel->get_index(i);

        // 检查NULL
        if (!input_data.validity.RowIsValid(idx)) {
            result.validity.SetInvalid(i);
            continue;
        }

        // 获取字符串
        auto str = ((string_t *)input_data.data)[idx];

        // 计算长度
        result_data[i] = str.GetSize();
    }

    result.SetVectorType(VectorType::FLAT_VECTOR);
    result.Verify(args.size());
}

// 注册函数
void RegisterStringLengthFunction() {
    ScalarFunctionSet func_set("string_length");

    ScalarFunction func(
        {LogicalType::VARCHAR},      // 输入：字符串
        LogicalType::INTEGER,         // 输出：整数
        StringLengthFunction
    );

    func_set.AddFunction(func);

    // 注册到内置函数
    auto &catalog = DatabaseInstance::GetBuiltinFunctions();
    catalog.AddFunction("string_length", func_set);
}

} // namespace duckdb
```

### 测试代码

```sql
-- 测试字符串长度函数
CREATE TABLE strings AS
    SELECT 'hello' as s
    UNION ALL
    SELECT 'world'
    UNION ALL
    SELECT 'DuckDB'
    UNION ALL
    SELECT NULL;

SELECT s, string_length(s) as len
FROM strings;

-- 结果：
-- s      | len
-- -------|-----
-- hello  | 5
-- world  | 5
-- DuckDB | 6
-- NULL   | NULL

-- 性能测试
SELECT string_length(repeat('a', 1000))
FROM range(1000000);
-- 应该在毫秒级完成
```

---

## 学习总结

### 向量化执行关键要点

1. **批量处理**：一次处理2048行，减少函数调用开销
2. **列式存储**：连续内存布局，缓存友好
3. **SIMD利用**：编译器自动向量化，充分利用CPU向量指令
4. **零拷贝**：通过引用和切片避免数据拷贝
5. **统一接口**：UnifiedVectorFormat提供一致的访问方式

### 性能优化技巧

| 优化技术 | 适用场景 | 预期加速 |
|---------|---------|---------|
| 向量化执行 | 所有算子 | 2-10x |
| SIMD指令 | 简单操作 | 4-8x |
| 缓存优化 | 大数据集 | 2-5x |
| 预取 | 超过L1缓存 | 1.5-2x |
| 零拷贝 | 中间结果 | 5-20x |

### 推荐阅读

**论文：**
- "MonetDB/X100: Hyper-Pipelining Query Execution" (CIDR 2005)
- "Column-Oriented Database Systems" (VLDB 2005)

**代码位置：**
- `src/common/types/vector.hpp` - Vector类定义
- `src/common/types data_chunk.hpp` - DataChunk类定义
- `src/execution/operator/` - 各类算子实现

**相关课程：**
- [DuckDB高级课程_查询优化器深度解析](./DuckDB高级课程_查询优化器深度解析.md)
- [DuckDB深度课程_执行算子实现详解](./DuckDB深度课程_执行算子实现详解.md)

---

**最后更新：2026-01-23**
