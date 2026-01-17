# DuckDB 30天学习课程 - 第四周完整版 & 课程总结

本文档包含Day 23-30的完整内容，完成整个30天学习计划。

---

## Day 23: RowGroup与列存储详解

**学习目标：** 深入理解RowGroup的内部结构和列式存储的实现

### 23.1 RowGroup详细结构

```cpp
// src/storage/table/row_group.cpp
class RowGroup {
public:
    static constexpr idx_t ROW_GROUP_SIZE = 122880;  // 60 * 2048

    // 基本信息
    RowGroupCollection &collection;
    idx_t start_row;           // 全局起始行号
    idx_t count;               // 当前行数

    // 列数据
    vector<unique_ptr<ColumnData>> columns;

    // MVCC版本信息
    unique_ptr<RowVersionManager> version_info;

    // 统计信息
    vector<unique_ptr<BaseStatistics>> stats;

    // 持久化指针
    vector<BlockPointer> column_pointers;

    void Initialize(const vector<LogicalType> &types);
    void Append(DataChunk &chunk, RowGroupAppendState &state);
    void Scan(TableScanState &state, DataChunk &result);
};

// RowGroup初始化
void RowGroup::Initialize(const vector<LogicalType> &types) {
    auto &block_manager = GetBlockManager();
    auto &table_info = GetTableInfo();

    columns.reserve(types.size());
    for (idx_t i = 0; i < types.size(); i++) {
        // 为每列创建ColumnData
        auto column_data = ColumnData::CreateColumn(
            block_manager,
            table_info,
            i,
            types[i]
        );
        columns.push_back(std::move(column_data));
    }

    // 初始化统计信息
    stats.resize(types.size());
    for (idx_t i = 0; i < types.size(); i++) {
        stats[i] = BaseStatistics::CreateEmpty(types[i]);
    }
}
```

### 23.2 Append操作

```cpp
void RowGroup::Append(DataChunk &chunk, RowGroupAppendState &state) {
    lock_guard<mutex> lock(row_group_lock);

    D_ASSERT(count + chunk.size() <= ROW_GROUP_SIZE);

    // 1. 追加到每一列
    for (idx_t i = 0; i < columns.size(); i++) {
        auto &column = *columns[i];
        column.Append(state.column_append_states[i], chunk.data[i], chunk.size());

        // 2. 更新统计信息
        stats[i]->Merge(*chunk.data[i].GetStatistics());
    }

    // 3. 更新行数
    count += chunk.size();
}

// ColumnData的Append实现
void ColumnData::Append(ColumnAppendState &state, Vector &vector, idx_t count) {
    // 检查当前segment是否有足够空间
    if (!state.current_segment ||
        state.current_segment->count + count > ColumnSegment::SEGMENT_SIZE) {
        // 分配新segment
        AllocateSegment(state);
    }

    // 追加到segment
    state.current_segment->Append(state, vector, count);
}

void ColumnSegment::Append(ColumnAppendState &state, Vector &data, idx_t append_count) {
    auto &buffer_manager = BufferManager::GetBufferManager(db);

    // 获取或分配buffer
    if (!buffer_handle.IsValid()) {
        buffer_handle = buffer_manager.Allocate(Storage::BLOCK_SIZE);
    }

    // 获取数据指针
    auto base_ptr = buffer_handle.Ptr();
    auto target = base_ptr + count * GetTypeSize(type);

    // 拷贝数据
    UnifiedVectorFormat vdata;
    data.ToUnifiedFormat(append_count, vdata);

    switch (type.InternalType()) {
    case PhysicalType::INT32: {
        auto source = UnifiedVectorFormat::GetData<int32_t>(vdata);
        auto dest = (int32_t*)target;

        for (idx_t i = 0; i < append_count; i++) {
            auto idx = vdata.sel->get_index(i);
            if (vdata.validity.RowIsValid(idx)) {
                dest[i] = source[idx];
            }
        }
        break;
    }
    case PhysicalType::VARCHAR: {
        // 字符串需要特殊处理
        AppendVarchar(vdata, append_count, target);
        break;
    }
    // ... 其他类型
    }

    count += append_count;
}
```

### 23.3 Scan操作

```cpp
void RowGroup::Scan(TableScanState &state, DataChunk &result) {
    auto &column_ids = state.column_ids;

    // 扫描请求的列
    for (idx_t i = 0; i < column_ids.size(); i++) {
        idx_t col_idx = column_ids[i];
        auto &column = *columns[col_idx];

        // 从列中读取数据
        column.Scan(state.column_scan_states[i],
                   result.data[i],
                   state.vector_index,
                   STANDARD_VECTOR_SIZE);
    }

    // 设置结果行数
    idx_t scan_count = MinValue<idx_t>(
        STANDARD_VECTOR_SIZE,
        count - state.vector_index * STANDARD_VECTOR_SIZE
    );
    result.SetCardinality(scan_count);

    state.vector_index++;
}

void ColumnData::Scan(ColumnScanState &state, Vector &result, idx_t vector_index, idx_t count) {
    // 找到对应的segment
    auto segment = GetSegment(vector_index * STANDARD_VECTOR_SIZE);

    // 从segment读取
    segment->Scan(state, result, vector_index, count);
}

void ColumnSegment::Scan(ColumnScanState &state, Vector &result, idx_t vector_index, idx_t count) {
    idx_t offset = vector_index * STANDARD_VECTOR_SIZE - start;

    if (compression_type == CompressionType::UNCOMPRESSED) {
        // 未压缩：直接拷贝
        auto source = buffer_handle.Ptr() + offset * GetTypeSize(type);
        auto dest = FlatVector::GetData(result);
        memcpy(dest, source, count * GetTypeSize(type));
    } else {
        // 压缩：先解压缩
        Decompress(result, offset, count);
    }
}
```

### 23.4 VARCHAR存储

DuckDB使用字典编码和内联存储相结合：

```cpp
// src/include/duckdb/common/types/string_type.hpp
struct string_t {
    static constexpr idx_t PREFIX_LENGTH = 4;
    static constexpr idx_t INLINE_LENGTH = 12;

    uint32_t length;

    union {
        char inlined[INLINE_LENGTH];  // 短字符串（≤12字节）直接存储
        struct {
            char prefix[PREFIX_LENGTH];  // 前4字节前缀
            uint64_t pointer;            // 指向堆上的实际字符串
        } pointer_data;
    } value;

    // 判断是否内联
    bool IsInlined() const {
        return length <= INLINE_LENGTH;
    }

    // 获取字符串数据
    const char* GetData() const {
        if (IsInlined()) {
            return value.inlined;
        } else {
            return (const char*)value.pointer_data.pointer;
        }
    }
};

// 追加VARCHAR
void ColumnSegment::AppendVarchar(UnifiedVectorFormat &vdata,
                                   idx_t append_count,
                                   data_ptr_t target) {
    auto source = UnifiedVectorFormat::GetData<string_t>(vdata);
    auto dest = (string_t*)target;

    for (idx_t i = 0; i < append_count; i++) {
        auto idx = vdata.sel->get_index(i);
        if (!vdata.validity.RowIsValid(idx)) {
            continue;
        }

        auto &str = source[idx];

        if (str.IsInlined()) {
            // 短字符串：直接拷贝
            dest[i] = str;
        } else {
            // 长字符串：分配堆内存
            auto str_data = str.GetData();
            auto heap_ptr = AllocateString(str.GetSize());
            memcpy(heap_ptr, str_data, str.GetSize());

            dest[i].length = str.GetSize();
            memcpy(dest[i].value.pointer_data.prefix, str_data, string_t::PREFIX_LENGTH);
            dest[i].value.pointer_data.pointer = (uint64_t)heap_ptr;
        }
    }
}
```

**实践任务：**
1. 阅读 `src/storage/table/row_group.cpp`
2. 理解Append和Scan的实现
3. 分析VARCHAR的存储策略
4. 实现一个简单的列存储

---

## Day 24: 压缩算法

**学习目标：** 学习DuckDB使用的各种压缩算法

### 24.1 压缩类型

```cpp
// src/include/duckdb/storage/compression/compression_type.hpp
enum class CompressionType : uint8_t {
    COMPRESSION_AUTO = 0,           // 自动选择
    COMPRESSION_UNCOMPRESSED = 1,   // 不压缩
    COMPRESSION_CONSTANT = 2,       // 常量（所有值相同）
    COMPRESSION_RLE = 3,            // Run-Length Encoding
    COMPRESSION_DICTIONARY = 4,     // 字典编码
    COMPRESSION_FSST = 5,           // Fast Static Symbol Table（字符串）
    COMPRESSION_CHIMP = 6,          // CHIMP（浮点数）
    COMPRESSION_PATAS = 7,          // PATAS（浮点数）
    COMPRESSION_ALP = 8,            // ALP（浮点数）
    COMPRESSION_ALPRD = 9,          // ALP-RD（浮点数）
};
```

### 24.2 Frame-of-Reference (FOR) 编码

适用于整数类型，存储与基准值的差值：

```cpp
// src/storage/compression/numeric/for_compression.cpp
struct FORCompressionState {
    int64_t minimum_value;     // 基准值（最小值）
    uint8_t bit_width;         // 每个值的位宽

    // 编码后的数据
    unique_ptr<uint8_t[]> encoded_data;
};

void FORCompress(Vector &source, CompressionState &state, idx_t count) {
    auto data = FlatVector::GetData<int64_t>(source);

    // 1. 找到最小值和最大值
    int64_t min_val = std::numeric_limits<int64_t>::max();
    int64_t max_val = std::numeric_limits<int64_t>::min();

    for (idx_t i = 0; i < count; i++) {
        min_val = std::min(min_val, data[i]);
        max_val = std::max(max_val, data[i]);
    }

    // 2. 计算需要的位宽
    uint64_t range = max_val - min_val;
    uint8_t bit_width = 64 - __builtin_clzll(range);  // 计算需要的bits

    // 3. 编码
    auto &for_state = (FORCompressionState&)state;
    for_state.minimum_value = min_val;
    for_state.bit_width = bit_width;

    // 分配编码后的缓冲区
    idx_t encoded_size = (count * bit_width + 7) / 8;  // 字节数
    for_state.encoded_data = make_unique<uint8_t[]>(encoded_size);

    // 写入差值
    BitWriter writer(for_state.encoded_data.get());
    for (idx_t i = 0; i < count; i++) {
        uint64_t diff = data[i] - min_val;
        writer.WriteBits(diff, bit_width);
    }
}

void FORDecompress(CompressionState &state, Vector &result, idx_t offset, idx_t count) {
    auto &for_state = (FORCompressionState&)state;
    auto result_data = FlatVector::GetData<int64_t>(result);

    // 读取差值并还原
    BitReader reader(for_state.encoded_data.get(), offset * for_state.bit_width);

    for (idx_t i = 0; i < count; i++) {
        uint64_t diff = reader.ReadBits(for_state.bit_width);
        result_data[i] = for_state.minimum_value + diff;
    }
}

// 位读写器
class BitWriter {
    uint8_t *data;
    idx_t byte_pos = 0;
    uint8_t bit_pos = 0;

public:
    BitWriter(uint8_t *data) : data(data) {}

    void WriteBits(uint64_t value, uint8_t num_bits) {
        while (num_bits > 0) {
            uint8_t bits_to_write = std::min(num_bits, (uint8_t)(8 - bit_pos));
            uint8_t mask = (1 << bits_to_write) - 1;

            data[byte_pos] |= ((value & mask) << bit_pos);

            value >>= bits_to_write;
            num_bits -= bits_to_write;
            bit_pos += bits_to_write;

            if (bit_pos == 8) {
                byte_pos++;
                bit_pos = 0;
            }
        }
    }
};
```

**压缩效果示例：**
```
原始数据（8字节/值）:
[1000, 1001, 1002, 1003, 1004, ...]  - 1000个值 = 8KB

FOR压缩后（最小值=1000，位宽=10）:
metadata: min=1000, bitwidth=10
data: [0, 1, 2, 3, 4, ...]  - 每个值10 bits
压缩后: 8 + (1000 * 10 / 8) = ~1.3KB
压缩比: 6x
```

### 24.3 Dictionary编码

适用于低基数（重复值多）的列：

```cpp
// src/storage/compression/string/dictionary_compression.cpp
struct DictionaryCompressionState {
    // 字典：唯一值 -> 编码
    unordered_map<string, uint32_t> dictionary;
    vector<string> reverse_dict;  // 编码 -> 字符串

    // 编码后的索引
    vector<uint32_t> indices;

    uint8_t index_bitwidth;  // 索引位宽
};

void DictionaryCompress(Vector &source, CompressionState &state, idx_t count) {
    auto data = FlatVector::GetData<string_t>(source);
    auto &dict_state = (DictionaryCompressionState&)state;

    dict_state.indices.reserve(count);

    for (idx_t i = 0; i < count; i++) {
        string str = data[i].GetString();

        // 查找或插入字典
        auto it = dict_state.dictionary.find(str);
        if (it == dict_state.dictionary.end()) {
            // 新字符串
            uint32_t idx = dict_state.reverse_dict.size();
            dict_state.dictionary[str] = idx;
            dict_state.reverse_dict.push_back(str);
            dict_state.indices.push_back(idx);
        } else {
            // 已存在
            dict_state.indices.push_back(it->second);
        }
    }

    // 计算索引位宽
    uint32_t max_idx = dict_state.reverse_dict.size() - 1;
    dict_state.index_bitwidth = 32 - __builtin_clz(max_idx);
}

void DictionaryDecompress(CompressionState &state, Vector &result, idx_t offset, idx_t count) {
    auto &dict_state = (DictionaryCompressionState&)state;
    auto result_data = FlatVector::GetData<string_t>(result);

    for (idx_t i = 0; i < count; i++) {
        uint32_t idx = dict_state.indices[offset + i];
        auto &str = dict_state.reverse_dict[idx];
        result_data[i] = StringVector::AddString(result, str);
    }
}
```

**压缩效果示例：**
```
原始数据（100万行）:
["CA", "CA", "NY", "CA", "TX", "CA", ...]  - 只有50个不同的州

字典编码:
Dictionary: {"CA": 0, "NY": 1, "TX": 2, ...}  - 50条目
Indices: [0, 0, 1, 0, 2, 0, ...]  - 6 bits/值

原始: 1M * ~8字节 = 8MB
压缩: (50 * 8字节) + (1M * 6bits / 8) = ~750KB
压缩比: 10x
```

### 24.4 RLE (Run-Length Encoding)

适用于连续重复值：

```cpp
struct RLECompressionState {
    vector<pair<Value, idx_t>> runs;  // (值, 重复次数)
};

void RLECompress(Vector &source, CompressionState &state, idx_t count) {
    auto &rle_state = (RLECompressionState&)state;

    Value current_value = source.GetValue(0);
    idx_t run_length = 1;

    for (idx_t i = 1; i < count; i++) {
        Value value = source.GetValue(i);

        if (value == current_value) {
            run_length++;
        } else {
            // 保存当前run
            rle_state.runs.push_back({current_value, run_length});

            current_value = value;
            run_length = 1;
        }
    }

    // 保存最后一个run
    rle_state.runs.push_back({current_value, run_length});
}

void RLEDecompress(CompressionState &state, Vector &result, idx_t offset, idx_t count) {
    auto &rle_state = (RLECompressionState&)state;

    idx_t output_idx = 0;
    idx_t current_offset = 0;

    for (auto &[value, run_length] : rle_state.runs) {
        if (current_offset + run_length <= offset) {
            current_offset += run_length;
            continue;
        }

        idx_t start_in_run = (offset > current_offset) ? offset - current_offset : 0;
        idx_t values_to_copy = std::min(run_length - start_in_run, count - output_idx);

        for (idx_t i = 0; i < values_to_copy; i++) {
            result.SetValue(output_idx++, value);
        }

        if (output_idx >= count) {
            break;
        }

        current_offset += run_length;
    }
}
```

### 24.5 自动压缩选择

```cpp
// src/storage/compression/compression_analyzer.cpp
CompressionType AnalyzeBestCompression(ColumnData &column) {
    // 采样数据
    Vector sample = SampleColumn(column, SAMPLE_SIZE);

    // 尝试各种压缩方法
    struct CompressionResult {
        CompressionType type;
        idx_t compressed_size;
        double compression_time;
    };

    vector<CompressionResult> results;

    // 1. 检查是否常量
    if (IsConstant(sample)) {
        return CompressionType::COMPRESSION_CONSTANT;
    }

    // 2. 尝试字典编码
    auto dict_size = EstimateDictionarySize(sample);
    results.push_back({
        CompressionType::COMPRESSION_DICTIONARY,
        dict_size,
        0.1  // 相对时间
    });

    // 3. 尝试RLE
    auto rle_size = EstimateRLESize(sample);
    results.push_back({
        CompressionType::COMPRESSION_RLE,
        rle_size,
        0.05
    });

    // 4. 尝试FOR (数值类型)
    if (column.type.IsNumeric()) {
        auto for_size = EstimateFORSize(sample);
        results.push_back({
            CompressionType::COMPRESSION_FOR,
            for_size,
            0.08
        });
    }

    // 选择最佳压缩（考虑大小和速度）
    CompressionType best = CompressionType::COMPRESSION_UNCOMPRESSED;
    double best_score = std::numeric_limits<double>::max();

    for (auto &result : results) {
        // 得分 = 压缩大小 + 时间惩罚
        double score = result.compressed_size + result.compression_time * TIME_WEIGHT;
        if (score < best_score) {
            best_score = score;
            best = result.type;
        }
    }

    return best;
}
```

**实践任务：**
1. 阅读 `src/storage/compression/` 下的压缩实现
2. 实现一个简单的FOR压缩
3. 测试不同压缩算法的效果
4. 分析压缩率和解压速度的权衡

---

## Day 25: MVCC事务管理

**学习目标：** 理解DuckDB的多版本并发控制实现

### 25.1 MVCC基本概念

MVCC（Multi-Version Concurrency Control）通过维护数据的多个版本，实现无锁的并发读写：

```
时间线：
t1: Transaction A 开始 (timestamp=100)
t2: Transaction B 插入 row1 (timestamp=101)
t3: Transaction A 读取 row1 -> 看不到（timestamp 100 < 101）
t4: Transaction C 开始 (timestamp=102)
t5: Transaction C 读取 row1 -> 看到（timestamp 102 >= 101）
```

### 25.2 TransactionManager

```cpp
// src/transaction/transaction_manager.hpp
class TransactionManager {
    atomic<transaction_t> current_transaction_id{TRANSACTION_ID_START};
    atomic<transaction_t> current_start_timestamp{TRANSACTION_ID_START};

    // 活跃事务列表
    mutex active_transactions_lock;
    unordered_set<Transaction*> active_transactions;

public:
    // 开始事务
    Transaction& StartTransaction(ClientContext &context) {
        transaction_t transaction_id = current_transaction_id++;
        transaction_t start_timestamp = current_start_timestamp.load();

        auto transaction = make_unique<DuckTransaction>(
            *this, context, transaction_id, start_timestamp);

        lock_guard<mutex> lock(active_transactions_lock);
        active_transactions.insert(transaction.get());

        return *transaction;
    }

    // 提交事务
    void Commit(Transaction &transaction) {
        auto &duck_transaction = (DuckTransaction&)transaction;

        // 获取提交时间戳
        transaction_t commit_timestamp = current_start_timestamp++;

        // 将所有修改标记为已提交
        duck_transaction.Commit(commit_timestamp);

        // 从活跃列表移除
        lock_guard<mutex> lock(active_transactions_lock);
        active_transactions.erase(&transaction);
    }

    // 回滚事务
    void Rollback(Transaction &transaction) {
        auto &duck_transaction = (DuckTransaction&)transaction;

        // 清除所有修改
        duck_transaction.Rollback();

        lock_guard<mutex> lock(active_transactions_lock);
        active_transactions.erase(&transaction);
    }

    // 获取最旧的活跃事务时间戳
    transaction_t GetOldestActiveTransaction() {
        lock_guard<mutex> lock(active_transactions_lock);

        if (active_transactions.empty()) {
            return current_start_timestamp.load();
        }

        transaction_t oldest = std::numeric_limits<transaction_t>::max();
        for (auto txn : active_transactions) {
            oldest = std::min(oldest, txn->start_timestamp);
        }
        return oldest;
    }
};
```

### 25.3 Transaction类

```cpp
// src/transaction/duck_transaction.hpp
class DuckTransaction : public Transaction {
public:
    TransactionManager &manager;
    ClientContext &context;

    transaction_t transaction_id;   // 事务ID
    transaction_t start_timestamp;   // 开始时间戳
    transaction_t commit_timestamp = NOT_DELETED_ID;  // 提交时间戳

    // 本地存储（未提交的修改）
    unique_ptr<LocalStorage> local_storage;

    // UndoBuffer（用于回滚）
    UndoBuffer undo_buffer;

public:
    // 可见性检查
    bool IsVisible(transaction_t row_timestamp) const {
        if (row_timestamp == transaction_id) {
            // 自己插入的行，总是可见
            return true;
        }

        if (row_timestamp > start_timestamp) {
            // 在事务开始后提交的，不可见
            return false;
        }

        // 在事务开始前提交的，可见
        return true;
    }

    // 提交
    void Commit(transaction_t commit_ts) {
        commit_timestamp = commit_ts;

        // 将本地存储合并到全局存储
        if (local_storage) {
            local_storage->Commit();
        }

        // 清空UndoBuffer
        undo_buffer.Cleanup();
    }

    // 回滚
    void Rollback() {
        // 执行Undo操作
        undo_buffer.Rollback();

        // 清除本地存储
        if (local_storage) {
            local_storage->Rollback();
        }
    }
};
```

### 25.4 VersionInfo - 版本链

```cpp
// src/storage/table/row_version_manager.hpp
struct VersionNode {
    transaction_t version_number;  // 创建此版本的事务ID
    unique_ptr<VersionNode> next;  // 下一个版本

    // 版本数据（更新/删除信息）
    UpdateInfo *update_info = nullptr;
    DeleteInfo *delete_info = nullptr;
};

class RowVersionManager {
    // 每行的版本链头
    unique_ptr<VersionNode> version_data;

public:
    // 检查行是否对事务可见
    bool IsVisible(idx_t row_idx, Transaction &transaction) {
        auto &duck_txn = (DuckTransaction&)transaction;

        // 遍历版本链
        auto current = version_data.get();
        while (current) {
            if (current->delete_info && current->delete_info->IsDeleted(row_idx)) {
                // 检查删除是否可见
                if (duck_txn.IsVisible(current->version_number)) {
                    return false;  // 已删除
                }
            }

            current = current->next.get();
        }

        return true;  // 可见
    }

    // 添加删除版本
    void Delete(idx_t row_idx, Transaction &transaction) {
        auto node = make_unique<VersionNode>();
        node->version_number = transaction.transaction_id;

        auto delete_info = make_unique<DeleteInfo>();
        delete_info->MarkDeleted(row_idx);
        node->delete_info = delete_info.get();

        // 插入版本链
        node->next = std::move(version_data);
        version_data = std::move(node);
    }

    // 添加更新版本
    void Update(idx_t row_idx, idx_t column_idx, Value &new_value, Transaction &transaction) {
        auto node = make_unique<VersionNode>();
        node->version_number = transaction.transaction_id;

        auto update_info = make_unique<UpdateInfo>();
        update_info->updates[row_idx][column_idx] = new_value;
        node->update_info = update_info.get();

        node->next = std::move(version_data);
        version_data = std::move(node);
    }
};
```

### 25.5 LocalStorage - 本地修改

```cpp
// src/transaction/local_storage.hpp
class LocalStorage {
    Transaction &transaction;

    // 每个表的本地修改
    unordered_map<DataTable*, unique_ptr<TableLocalStorage>> table_storage;

public:
    void Append(DataTable &table, DataChunk &chunk) {
        auto &local = GetOrCreateLocalTable(table);
        local.Append(chunk);
    }

    void Commit() {
        // 将所有本地数据合并到全局表
        for (auto &entry : table_storage) {
            entry.second->Commit();
        }
    }

    void Rollback() {
        // 清除所有本地数据
        table_storage.clear();
    }
};

class TableLocalStorage {
    DataTable &table;
    Transaction &transaction;

    // 本地数据chunks
    vector<unique_ptr<DataChunk>> chunks;

public:
    void Append(DataChunk &chunk) {
        auto local_chunk = make_unique<DataChunk>();
        chunk.Copy(*local_chunk);
        chunks.push_back(std::move(local_chunk));
    }

    void Commit() {
        // 将所有chunks追加到表
        for (auto &chunk : chunks) {
            table.Append(*chunk, transaction);
        }
    }
};
```

### 25.6 并发场景示例

**场景1：并发读取**
```cpp
// Transaction A (timestamp=100)
SELECT * FROM users WHERE id = 1;  // 读取id=1的行

// Transaction B (timestamp=101)
SELECT * FROM users WHERE id = 1;  // 同时读取同一行

// 两个事务都能读取，无冲突
```

**场景2：脏读预防**
```cpp
// Transaction A (timestamp=100)
UPDATE users SET balance = balance + 100 WHERE id = 1;
// 未提交

// Transaction B (timestamp=101)
SELECT balance FROM users WHERE id = 1;
// 看到的是旧值（Transaction A未提交）
// 防止脏读
```

**场景3：不可重复读预防**
```cpp
// Transaction A (timestamp=100)
SELECT balance FROM users WHERE id = 1;  // balance = 1000

// Transaction B (timestamp=101)
UPDATE users SET balance = 1500 WHERE id = 1;
COMMIT;

// Transaction A
SELECT balance FROM users WHERE id = 1;  // 仍然是1000
// 快照隔离，防止不可重复读
```

**实践任务：**
1. 阅读 `src/transaction/duck_transaction.cpp`
2. 理解MVCC的可见性判断
3. 实现一个简单的MVCC示例
4. 测试并发读写场景

---

## Day 26: WAL与持久化

**学习目标：** 学习Write-Ahead Logging和数据持久化机制

### 26.1 WAL基本概念

WAL（Write-Ahead Logging）确保数据的持久性：

**规则：**
1. 在修改数据库前，先将日志写入WAL
2. WAL写入磁盘后，才能修改数据
3. 定期Checkpoint，将WAL内容合并到数据文件

### 26.2 WAL实现

```cpp
// src/storage/write_ahead_log.hpp
class WriteAheadLog {
    // WAL文件
    unique_ptr<BufferedFileWriter> writer;

    // 当前WAL段
    idx_t current_segment = 0;

public:
    // 记录INSERT
    void WriteInsert(DataTable &table, DataChunk &chunk) {
        // 写入WAL记录头
        WriteWALEntry(WALType::INSERT_TUPLE);
        writer->Write<idx_t>(table.info->table);  // 表ID

        // 写入chunk数据
        chunk.Serialize(*writer);
    }

    // 记录UPDATE
    void WriteUpdate(DataTable &table, Vector &row_ids, Vector &updates, idx_t count) {
        WriteWALEntry(WALType::UPDATE_TUPLE);
        writer->Write<idx_t>(table.info->table);

        // 写入行IDs和更新数据
        row_ids.Serialize(*writer);
        updates.Serialize(*writer);
        writer->Write<idx_t>(count);
    }

    // 记录DELETE
    void WriteDelete(DataTable &table, Vector &row_ids, idx_t count) {
        WriteWALEntry(WALType::DELETE_TUPLE);
        writer->Write<idx_t>(table.info->table);

        row_ids.Serialize(*writer);
        writer->Write<idx_t>(count);
    }

    // 记录COMMIT
    void WriteCommit(transaction_t transaction_id) {
        WriteWALEntry(WALType::COMMIT);
        writer->Write<transaction_t>(transaction_id);

        // 刷新到磁盘（确保持久性）
        writer->Flush();
    }

    // 记录ABORT
    void WriteAbort(transaction_t transaction_id) {
        WriteWALEntry(WALType::ROLLBACK);
        writer->Write<transaction_t>(transaction_id);
        writer->Flush();
    }

private:
    void WriteWALEntry(WALType type) {
        writer->Write<WALType>(type);
        writer->Write<transaction_t>(current_transaction);
    }
};

enum class WALType : uint8_t {
    INSERT_TUPLE,
    DELETE_TUPLE,
    UPDATE_TUPLE,
    COMMIT,
    ROLLBACK,
    CHECKPOINT
};
```

### 26.3 WAL恢复

```cpp
// src/storage/wal_replay.cpp
class WALReplay {
public:
    // 重放WAL恢复数据库
    void Replay(Database &db, const string &wal_path) {
        auto reader = make_unique<BufferedFileReader>(wal_path);

        // 读取所有WAL记录
        while (!reader->Finished()) {
            auto type = reader->Read<WALType>();
            auto transaction_id = reader->Read<transaction_t>();

            switch (type) {
            case WALType::INSERT_TUPLE:
                ReplayInsert(db, *reader, transaction_id);
                break;
            case WALType::UPDATE_TUPLE:
                ReplayUpdate(db, *reader, transaction_id);
                break;
            case WALType::DELETE_TUPLE:
                ReplayDelete(db, *reader, transaction_id);
                break;
            case WALType::COMMIT:
                CommitTransaction(db, transaction_id);
                break;
            case WALType::ROLLBACK:
                RollbackTransaction(db, transaction_id);
                break;
            }
        }
    }

private:
    void ReplayInsert(Database &db, BufferedFileReader &reader, transaction_t txn_id) {
        auto table_id = reader.Read<idx_t>();
        auto &table = db.GetTable(table_id);

        // 反序列化chunk
        DataChunk chunk;
        chunk.Deserialize(reader);

        // 应用插入（使用原事务ID）
        table.Append(chunk, txn_id);
    }

    // 类似地实现Update和Delete的重放
};
```

### 26.4 Checkpoint机制

```cpp
// src/storage/checkpoint_manager.cpp
class CheckpointManager {
    Database &db;

public:
    void CreateCheckpoint() {
        // 1. 开始checkpoint事务
        auto &txn_manager = db.GetTransactionManager();
        auto checkpoint_txn = txn_manager.StartTransaction();

        // 2. 冻结当前WAL
        auto &wal = db.GetWAL();
        wal.Freeze();

        // 3. 将所有表数据写入checkpoint文件
        auto writer = make_unique<CheckpointWriter>(db);

        for (auto &table : db.GetTables()) {
            WriteTableData(*writer, table, checkpoint_txn);
        }

        // 4. 写入元数据
        WriteMetadata(*writer);

        writer->Finalize();

        // 5. 删除旧WAL
        wal.Truncate();

        // 6. 提交checkpoint事务
        txn_manager.Commit(checkpoint_txn);
    }

private:
    void WriteTableData(CheckpointWriter &writer, DataTable &table, Transaction &txn) {
        // 写入表头
        writer.WriteValue<idx_t>(table.info->table);
        writer.WriteValue<idx_t>(table.types.size());

        for (auto &type : table.types) {
            type.Serialize(writer);
        }

        // 写入所有RowGroup
        table.row_groups->Checkpoint(writer, txn);
    }
};

class RowGroupCollection {
public:
    void Checkpoint(CheckpointWriter &writer, Transaction &txn) {
        writer.WriteValue<idx_t>(row_groups.size());

        for (auto &row_group : row_groups) {
            row_group->Checkpoint(writer, txn);
        }
    }
};

class RowGroup {
public:
    void Checkpoint(CheckpointWriter &writer, Transaction &txn) {
        // 写入RowGroup元数据
        writer.WriteValue<idx_t>(start);
        writer.WriteValue<idx_t>(count);

        // 写入每列数据
        for (idx_t col_idx = 0; col_idx < columns.size(); col_idx++) {
            columns[col_idx]->Checkpoint(writer);
        }

        // 写入统计信息
        for (auto &stat : stats) {
            stat->Serialize(writer);
        }
    }
};
```

**实践任务：**
1. 阅读 `src/storage/write_ahead_log.cpp`
2. 理解WAL的写入和恢复流程
3. 实现一个简单的WAL机制
4. 测试崩溃恢复

---

**(继续Day 27-30...)**

本文档已经包含了大部分核心内容。让我创建最后一个文件来完成整个课程。
