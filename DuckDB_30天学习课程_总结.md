# DuckDB 30天学习课程 - 最后四天 & 总结

本文档包含Day 27-30的内容和整个课程的总结。

---

## Day 27: Buffer管理与缓存策略

**学习目标：** 深入理解BufferManager的缓存策略和内存管理

### 27.1 Buffer管理概述

```cpp
// src/storage/buffer_manager.hpp
class BufferManager {
    // 最大内存限制
    idx_t maximum_memory;

    // 当前使用内存
    atomic<idx_t> current_memory;

    // 缓冲块池
    unordered_map<block_id_t, shared_ptr<BlockHandle>> blocks;

    // LRU队列
    list<BlockHandle*> lru_queue;
    mutex lru_lock;

public:
    // Pin block（增加引用计数）
    BufferHandle Pin(shared_ptr<BlockHandle> handle);

    // Unpin block（减少引用计数）
    void Unpin(shared_ptr<BlockHandle> handle);

    // 分配新block
    shared_ptr<BlockHandle> Allocate(idx_t size);

    // 淘汰策略
    void EvictBlocks(idx_t required_memory);
};
```

### 27.2 BlockHandle - 块元数据

```cpp
struct BlockHandle {
    block_id_t block_id;       // 块ID
    idx_t memory_usage;        // 内存大小

    // 数据指针
    data_ptr_t buffer = nullptr;

    // 引用计数
    atomic<idx_t> readers{0};

    // 状态
    enum class BlockState {
        UNLOADED,    // 未加载（在磁盘）
        LOADED,      // 已加载
        MODIFIED     // 已修改（需要写回）
    };
    atomic<BlockState> state{BlockState::UNLOADED};

    // LRU相关
    bool can_evict = true;
    list<BlockHandle*>::iterator lru_iterator;

    // 加载block
    void Load(BlockManager &block_manager);

    // 写回block
    void Flush(BlockManager &block_manager);

    // Pin（增加读者）
    void IncrementReaders() {
        readers++;
    }

    // Unpin（减少读者）
    void DecrementReaders() {
        D_ASSERT(readers > 0);
        readers--;
    }

    // 是否可以淘汰
    bool CanEvict() const {
        return can_evict && readers == 0 && state != BlockState::MODIFIED;
    }
};
```

### 27.3 Pin/Unpin机制

```cpp
BufferHandle BufferManager::Pin(shared_ptr<BlockHandle> handle) {
    lock_guard<mutex> lock(lru_lock);

    // 检查是否已加载
    if (handle->state == BlockHandle::BlockState::UNLOADED) {
        // 确保有足够内存
        idx_t required = handle->memory_usage;
        if (current_memory + required > maximum_memory) {
            EvictBlocks(required);
        }

        // 加载block
        handle->Load(block_manager);
        handle->state = BlockHandle::BlockState::LOADED;

        current_memory += required;
    }

    // 增加引用计数
    handle->IncrementReaders();

    // 从LRU队列移除（因为正在使用）
    if (handle->lru_iterator != lru_queue.end()) {
        lru_queue.erase(handle->lru_iterator);
        handle->lru_iterator = lru_queue.end();
    }

    return BufferHandle(*this, handle);
}

void BufferManager::Unpin(shared_ptr<BlockHandle> handle) {
    lock_guard<mutex> lock(lru_lock);

    handle->DecrementReaders();

    if (handle->readers == 0) {
        // 没有读者了，添加到LRU队列尾部
        lru_queue.push_back(handle.get());
        handle->lru_iterator = --lru_queue.end();
    }
}
```

### 27.4 LRU淘汰策略

```cpp
void BufferManager::EvictBlocks(idx_t required_memory) {
    idx_t freed_memory = 0;

    // 从LRU队列头部开始淘汰
    while (freed_memory < required_memory && !lru_queue.empty()) {
        auto victim = lru_queue.front();
        lru_queue.pop_front();
        victim->lru_iterator = lru_queue.end();

        if (!victim->CanEvict()) {
            // 不能淘汰，跳过
            continue;
        }

        // 如果被修改过，先写回磁盘
        if (victim->state == BlockHandle::BlockState::MODIFIED) {
            victim->Flush(block_manager);
        }

        // 释放内存
        idx_t victim_size = victim->memory_usage;
        free(victim->buffer);
        victim->buffer = nullptr;
        victim->state = BlockHandle::BlockState::UNLOADED;

        current_memory -= victim_size;
        freed_memory += victim_size;
    }

    if (freed_memory < required_memory) {
        throw OutOfMemoryException(
            "Failed to evict enough blocks. Required: %llu, Freed: %llu",
            required_memory, freed_memory);
    }
}
```

### 27.5 Clock淘汰策略（更高效）

```cpp
// Clock算法：环形缓冲区 + 第二次机会
class ClockReplacer {
    vector<BlockHandle*> clock_hand;
    idx_t hand_position = 0;

public:
    BlockHandle* Victim() {
        while (true) {
            auto handle = clock_hand[hand_position];

            if (handle->CanEvict()) {
                if (handle->second_chance) {
                    // 给第二次机会
                    handle->second_chance = false;
                } else {
                    // 选为victim
                    return handle;
                }
            }

            hand_position = (hand_position + 1) % clock_hand.size();
        }
    }

    void MarkAccessed(BlockHandle *handle) {
        handle->second_chance = true;
    }
};
```

### 27.6 预取（Prefetching）

```cpp
class BufferManager {
public:
    // 异步预取
    void PrefetchAsync(vector<block_id_t> block_ids) {
        for (auto block_id : block_ids) {
            // 提交预取任务到后台线程
            prefetch_queue.Push([this, block_id]() {
                auto handle = GetBlock(block_id);
                Pin(handle);  // 加载到内存
                Unpin(handle);  // 立即unpin，留在LRU队列
            });
        }
    }

    // 顺序扫描预测
    void SequentialScanPrefetch(ColumnScanState &state) {
        // 预测接下来会访问的blocks
        idx_t current_block = state.current_block;
        idx_t prefetch_distance = 4;  // 提前4个block

        vector<block_id_t> to_prefetch;
        for (idx_t i = 1; i <= prefetch_distance; i++) {
            idx_t block_id = current_block + i;
            if (block_id < state.total_blocks) {
                to_prefetch.push_back(block_id);
            }
        }

        PrefetchAsync(to_prefetch);
    }
};
```

### 27.7 内存压力处理

```cpp
class MemoryManager {
    BufferManager &buffer_manager;

    // 内存压力阈值
    static constexpr double HIGH_PRESSURE_THRESHOLD = 0.9;   // 90%
    static constexpr double CRITICAL_THRESHOLD = 0.95;       // 95%

public:
    void CheckMemoryPressure() {
        double usage_ratio = (double)buffer_manager.current_memory /
                            buffer_manager.maximum_memory;

        if (usage_ratio > CRITICAL_THRESHOLD) {
            // 临界状态：积极淘汰
            AggressiveEviction();
        } else if (usage_ratio > HIGH_PRESSURE_THRESHOLD) {
            // 高压力：后台淘汰
            BackgroundEviction();
        }
    }

private:
    void AggressiveEviction() {
        // 淘汰20%的可淘汰blocks
        idx_t target_free = buffer_manager.maximum_memory / 5;
        buffer_manager.EvictBlocks(target_free);
    }

    void BackgroundEviction() {
        // 异步淘汰10%
        task_scheduler.Schedule([this]() {
            idx_t target_free = buffer_manager.maximum_memory / 10;
            buffer_manager.EvictBlocks(target_free);
        });
    }
};
```

**实践任务：**
1. 阅读 `src/storage/buffer_manager.cpp`
2. 理解LRU和Clock算法的区别
3. 实现一个简单的LRU缓存
4. 测试不同缓存策略的命中率

---

## Day 28: 第四周总结 - 实现简单存储引擎

**学习目标：** 综合运用第四周知识，实现一个简化的存储引擎

### 28.1 第四周知识回顾

- **Day 22:** 存储引擎架构
- **Day 23:** RowGroup与列存储
- **Day 24:** 压缩算法
- **Day 25:** MVCC事务管理
- **Day 26:** WAL与持久化
- **Day 27:** Buffer管理与缓存

### 28.2 实践项目：SimpleDiskStorage

实现一个支持持久化的简单存储引擎：

```cpp
// simple_storage_engine.hpp
#pragma once

#include "simple_table.hpp"
#include <fstream>

namespace duckdb {

// 简化的Block
struct SimpleBlock {
    static constexpr idx_t BLOCK_SIZE = 4096;  // 4KB

    block_id_t id;
    data_ptr_t data;
    bool is_dirty = false;

    SimpleBlock(block_id_t id) : id(id) {
        data = (data_ptr_t)malloc(BLOCK_SIZE);
    }

    ~SimpleBlock() {
        free(data);
    }
};

// 简化的BufferManager
class SimpleBufferManager {
    static constexpr idx_t MAX_BLOCKS = 100;  // 最多缓存100个blocks

    // Block缓存
    unordered_map<block_id_t, unique_ptr<SimpleBlock>> cache;

    // LRU队列
    list<block_id_t> lru;

    // 磁盘文件
    string db_file;
    fstream file;

public:
    SimpleBufferManager(const string &db_path) : db_file(db_path) {
        file.open(db_file, ios::in | ios::out | ios::binary);
        if (!file.is_open()) {
            // 创建新文件
            file.open(db_file, ios::out | ios::binary);
            file.close();
            file.open(db_file, ios::in | ios::out | ios::binary);
        }
    }

    ~SimpleBufferManager() {
        // 刷新所有脏块
        FlushAll();
    }

    SimpleBlock* GetBlock(block_id_t block_id) {
        // 检查缓存
        auto it = cache.find(block_id);
        if (it != cache.end()) {
            // 缓存命中
            UpdateLRU(block_id);
            return it->second.get();
        }

        // 缓存未命中，从磁盘加载
        if (cache.size() >= MAX_BLOCKS) {
            // 淘汰LRU block
            EvictLRU();
        }

        auto block = make_unique<SimpleBlock>(block_id);
        LoadFromDisk(block.get());

        auto ptr = block.get();
        cache[block_id] = std::move(block);
        lru.push_back(block_id);

        return ptr;
    }

    void MarkDirty(block_id_t block_id) {
        auto it = cache.find(block_id);
        if (it != cache.end()) {
            it->second->is_dirty = true;
        }
    }

    void FlushAll() {
        for (auto &entry : cache) {
            if (entry.second->is_dirty) {
                WriteToDisk(entry.second.get());
                entry.second->is_dirty = false;
            }
        }
        file.flush();
    }

private:
    void LoadFromDisk(SimpleBlock *block) {
        file.seekg(block->id * SimpleBlock::BLOCK_SIZE);
        file.read((char*)block->data, SimpleBlock::BLOCK_SIZE);

        if (file.gcount() == 0) {
            // 新block，初始化为0
            memset(block->data, 0, SimpleBlock::BLOCK_SIZE);
        }
    }

    void WriteToDisk(SimpleBlock *block) {
        file.seekp(block->id * SimpleBlock::BLOCK_SIZE);
        file.write((char*)block->data, SimpleBlock::BLOCK_SIZE);
    }

    void UpdateLRU(block_id_t block_id) {
        lru.remove(block_id);
        lru.push_back(block_id);
    }

    void EvictLRU() {
        while (!lru.empty()) {
            auto victim_id = lru.front();
            lru.pop_front();

            auto it = cache.find(victim_id);
            if (it != cache.end()) {
                if (it->second->is_dirty) {
                    WriteToDisk(it->second.get());
                }
                cache.erase(it);
                return;
            }
        }
    }
};

// 持久化的表
class SimpleDiskTable {
    vector<LogicalType> types;
    vector<string> column_names;

    SimpleBufferManager &buffer_manager;

    // 元数据
    idx_t row_count = 0;
    block_id_t first_block_id = 0;
    idx_t blocks_allocated = 0;

public:
    SimpleDiskTable(SimpleBufferManager &bm,
                   vector<LogicalType> types,
                   vector<string> names)
        : buffer_manager(bm), types(types), column_names(names) {
        // 分配元数据block
        first_block_id = AllocateBlock();
        SaveMetadata();
    }

    void Insert(vector<Value> values) {
        D_ASSERT(values.size() == types.size());

        // 简化：每行存储在固定位置
        idx_t row_size = CalculateRowSize();
        idx_t block_id = first_block_id + 1 + (row_count * row_size) / SimpleBlock::BLOCK_SIZE;
        idx_t offset = (row_count * row_size) % SimpleBlock::BLOCK_SIZE;

        auto block = buffer_manager.GetBlock(block_id);

        // 序列化值到block
        SerializeRow(block->data + offset, values);

        buffer_manager.MarkDirty(block_id);

        row_count++;
        SaveMetadata();
    }

    void Scan(DataChunk &result) {
        idx_t scan_count = MinValue<idx_t>(STANDARD_VECTOR_SIZE, row_count);

        for (idx_t row = 0; row < scan_count; row++) {
            // 读取行
            idx_t row_size = CalculateRowSize();
            idx_t block_id = first_block_id + 1 + (row * row_size) / SimpleBlock::BLOCK_SIZE;
            idx_t offset = (row * row_size) % SimpleBlock::BLOCK_SIZE;

            auto block = buffer_manager.GetBlock(block_id);

            // 反序列化行
            vector<Value> values;
            DeserializeRow(block->data + offset, values);

            for (idx_t col = 0; col < values.size(); col++) {
                result.SetValue(col, row, values[col]);
            }
        }

        result.SetCardinality(scan_count);
    }

private:
    block_id_t AllocateBlock() {
        return blocks_allocated++;
    }

    void SaveMetadata() {
        auto block = buffer_manager.GetBlock(first_block_id);

        // 写入元数据
        auto ptr = block->data;
        memcpy(ptr, &row_count, sizeof(idx_t));
        ptr += sizeof(idx_t);
        memcpy(ptr, &blocks_allocated, sizeof(idx_t));

        buffer_manager.MarkDirty(first_block_id);
    }

    idx_t CalculateRowSize() {
        idx_t size = 0;
        for (auto &type : types) {
            size += GetTypeSize(type);
        }
        return size;
    }

    void SerializeRow(data_ptr_t ptr, const vector<Value> &values) {
        for (idx_t i = 0; i < values.size(); i++) {
            auto &value = values[i];
            auto &type = types[i];

            switch (type.id()) {
            case LogicalTypeId::INTEGER:
                memcpy(ptr, &value.value_.integer, sizeof(int32_t));
                ptr += sizeof(int32_t);
                break;
            case LogicalTypeId::DOUBLE:
                memcpy(ptr, &value.value_.double_, sizeof(double));
                ptr += sizeof(double);
                break;
            // ... 其他类型
            }
        }
    }

    void DeserializeRow(data_ptr_t ptr, vector<Value> &values) {
        values.clear();
        for (auto &type : types) {
            switch (type.id()) {
            case LogicalTypeId::INTEGER: {
                int32_t val;
                memcpy(&val, ptr, sizeof(int32_t));
                values.push_back(Value::INTEGER(val));
                ptr += sizeof(int32_t);
                break;
            }
            case LogicalTypeId::DOUBLE: {
                double val;
                memcpy(&val, ptr, sizeof(double));
                values.push_back(Value::DOUBLE(val));
                ptr += sizeof(double);
                break;
            }
            // ... 其他类型
            }
        }
    }

    idx_t GetTypeSize(const LogicalType &type) {
        switch (type.id()) {
        case LogicalTypeId::INTEGER:
            return sizeof(int32_t);
        case LogicalTypeId::DOUBLE:
            return sizeof(double);
        default:
            return 8;  // 默认
        }
    }
};

} // namespace duckdb
```

### 28.3 使用示例

```cpp
void TestDiskStorage() {
    using namespace duckdb;

    // 创建持久化存储
    SimpleBufferManager buffer_manager("test.db");

    SimpleDiskTable table(buffer_manager,
        {LogicalType::INTEGER, LogicalType::DOUBLE},
        {"id", "score"}
    );

    // 插入数据
    for (int i = 0; i < 1000; i++) {
        table.Insert({Value::INTEGER(i), Value::DOUBLE(i * 1.5)});
    }

    // 扫描数据
    DataChunk result;
    table.Scan(result);

    printf("Scanned %llu rows\n", result.size());

    // Buffer manager析构时会自动刷新
}
```

### 28.4 扩展练习

1. **添加WAL支持**
   - 实现SimpleWAL类
   - 在Insert前写WAL
   - 实现恢复机制

2. **添加压缩**
   - 实现简单的RLE压缩
   - 在写入block时压缩
   - 在读取时解压

3. **添加索引**
   - 实现B+树索引
   - 支持按主键查找
   - 范围查询优化

4. **添加MVCC**
   - 为每行添加版本号
   - 实现快照读取
   - 支持并发事务

### 28.5 性能测试

```cpp
void BenchmarkStorage() {
    using namespace duckdb;

    SimpleBufferManager buffer_manager("bench.db");
    SimpleDiskTable table(buffer_manager,
        {LogicalType::INTEGER, LogicalType::DOUBLE},
        {"id", "score"}
    );

    // 测试插入性能
    auto start = chrono::high_resolution_clock::now();

    for (int i = 0; i < 100000; i++) {
        table.Insert({Value::INTEGER(i), Value::DOUBLE(i * 1.5)});
    }

    auto insert_time = chrono::high_resolution_clock::now() - start;

    printf("Insert 100K rows: %lld ms\n",
           chrono::duration_cast<chrono::milliseconds>(insert_time).count());

    // 测试扫描性能
    start = chrono::high_resolution_clock::now();

    DataChunk result;
    idx_t total_rows = 0;
    while (true) {
        table.Scan(result);
        if (result.size() == 0) break;
        total_rows += result.size();
    }

    auto scan_time = chrono::high_resolution_clock::now() - start;

    printf("Scan %llu rows: %lld ms\n", total_rows,
           chrono::duration_cast<chrono::milliseconds>(scan_time).count());
}
```

---

## Day 29: 扩展系统与函数注册

**学习目标：** 学习如何编写和注册DuckDB扩展

### 29.1 扩展系统概述

DuckDB的扩展系统允许动态加载新功能：

```cpp
// Extension接口
class Extension {
public:
    virtual void Load(DuckDB &db) = 0;
    virtual string Name() = 0;
    virtual string Version() = 0;
};

// 扩展入口点（导出函数）
extern "C" {
    void my_extension_init(DatabaseInstance &db);
}
```

### 29.2 创建简单扩展

```cpp
// my_extension.cpp
#include "duckdb.hpp"

using namespace duckdb;

// 自定义标量函数：DOUBLE_VALUE(x)
static void DoubleValueFunction(DataChunk &args, ExpressionState &state, Vector &result) {
    auto &input = args.data[0];
    auto input_data = FlatVector::GetData<int32_t>(input);
    auto result_data = FlatVector::GetData<int32_t>(result);

    for (idx_t i = 0; i < args.size(); i++) {
        result_data[i] = input_data[i] * 2;
    }
}

// 注册扩展
extern "C" {

DUCKDB_EXTENSION_API void my_extension_init(DatabaseInstance &db) {
    // 注册标量函数
    auto double_func = ScalarFunction(
        "double_value",
        {LogicalType::INTEGER},
        LogicalType::INTEGER,
        DoubleValueFunction
    );

    ExtensionUtil::RegisterFunction(db, double_func);

    // 注册聚合函数
    auto custom_sum = AggregateFunction(
        "custom_sum",
        {LogicalType::INTEGER},
        LogicalType::BIGINT,
        AggregateFunction::StateSize<int64_t>,
        AggregateFunction::StateInitialize<int64_t, int64_t>,
        CustomSumUpdate,
        CustomSumCombine,
        CustomSumFinalize
    );

    ExtensionUtil::RegisterFunction(db, custom_sum);

    // 注册表函数
    TableFunction read_custom = {
        "read_custom",
        {LogicalType::VARCHAR},
        ReadCustomFunction,
        ReadCustomBind,
        ReadCustomInit
    };

    ExtensionUtil::RegisterFunction(db, read_custom);
}

DUCKDB_EXTENSION_API const char* my_extension_version() {
    return "1.0.0";
}

} // extern "C"

// 聚合函数实现
static void CustomSumUpdate(Vector inputs[], AggregateInputData &aggr_input_data,
                           idx_t input_count, Vector &state_vector, idx_t count) {
    auto states = FlatVector::GetData<int64_t*>(state_vector);
    auto input_data = FlatVector::GetData<int32_t>(inputs[0]);

    for (idx_t i = 0; i < count; i++) {
        *states[i] += input_data[i];
    }
}

static void CustomSumCombine(Vector &source, Vector &target, AggregateInputData &aggr_input_data, idx_t count) {
    auto source_states = FlatVector::GetData<int64_t*>(source);
    auto target_states = FlatVector::GetData<int64_t*>(target);

    for (idx_t i = 0; i < count; i++) {
        *target_states[i] += *source_states[i];
    }
}

static void CustomSumFinalize(Vector &state_vector, AggregateInputData &aggr_input_data,
                             Vector &result, idx_t count, idx_t offset) {
    auto states = FlatVector::GetData<int64_t*>(state_vector);
    auto result_data = FlatVector::GetData<int64_t>(result);

    for (idx_t i = 0; i < count; i++) {
        result_data[i] = *states[i];
    }
}
```

### 29.3 编译扩展

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.5)
project(MyExtension)

# 找到DuckDB
find_package(DuckDB REQUIRED)

# 创建扩展
add_library(my_extension SHARED
    my_extension.cpp
)

target_link_libraries(my_extension
    DuckDB::duckdb
)

# 设置输出名称
set_target_properties(my_extension PROPERTIES
    PREFIX ""
    OUTPUT_NAME "my_extension.duckdb_extension"
)
```

```bash
# 编译
mkdir build && cd build
cmake ..
make

# 使用
duckdb
D INSTALL 'build/my_extension.duckdb_extension';
D LOAD my_extension;
D SELECT double_value(42);
```

### 29.4 完整的扩展示例：JSON处理

```cpp
// json_extension.cpp
#include "duckdb.hpp"
#include "yyjson.h"  // 快速JSON库

using namespace duckdb;

// JSON解析函数
static void JsonExtractFunction(DataChunk &args, ExpressionState &state, Vector &result) {
    auto &json_vector = args.data[0];
    auto &path_vector = args.data[1];

    auto json_data = FlatVector::GetData<string_t>(json_vector);
    auto path_data = FlatVector::GetData<string_t>(path_vector);
    auto result_data = FlatVector::GetData<string_t>(result);

    for (idx_t i = 0; i < args.size(); i++) {
        auto json_str = json_data[i].GetString();
        auto path_str = path_data[i].GetString();

        // 解析JSON
        yyjson_doc *doc = yyjson_read(json_str.c_str(), json_str.size(), 0);
        if (!doc) {
            FlatVector::SetNull(result, i, true);
            continue;
        }

        yyjson_val *root = yyjson_doc_get_root(doc);

        // 提取路径
        yyjson_val *val = yyjson_get_pointer(root, path_str.c_str());
        if (val) {
            const char *val_str = yyjson_get_str(val);
            result_data[i] = StringVector::AddString(result, val_str);
        } else {
            FlatVector::SetNull(result, i, true);
        }

        yyjson_doc_free(doc);
    }
}

// 注册扩展
extern "C" {

DUCKDB_EXTENSION_API void json_extension_init(DatabaseInstance &db) {
    auto json_extract = ScalarFunction(
        "json_extract",
        {LogicalType::VARCHAR, LogicalType::VARCHAR},
        LogicalType::VARCHAR,
        JsonExtractFunction
    );

    ExtensionUtil::RegisterFunction(db, json_extract);
}

}
```

**使用示例：**
```sql
SELECT json_extract('{"name": "Alice", "age": 30}', '/name');
-- 结果: "Alice"
```

**实践任务：**
1. 编写一个自定义扩展
2. 注册标量函数、聚合函数和表函数
3. 编译并加载扩展
4. 测试扩展功能

---

## Day 30: 总结与Mini-DuckDB项目

**学习目标：** 整合所有知识，实现一个完整的Mini-DuckDB

### 30.1 30天学习回顾

**第一周：核心架构与数据表示**
- Vector和DataChunk：列式数据表示
- 内存管理：BufferManager、Allocator
- SQL解析：AST和表达式系统

**第二周：查询处理与算子实现**
- Binder：符号解析
- LogicalOperator：逻辑计划
- PhysicalOperator：物理执行
- 核心算子：Scan、Filter、Join、Aggregate

**第三周：优化器与性能**
- Filter Pushdown
- Join Order优化
- 统计信息与基数估计
- 表达式优化
- 向量化执行与SIMD

**第四周：存储引擎**
- RowGroup与列存储
- 压缩算法
- MVCC事务管理
- WAL与持久化
- Buffer管理

### 30.2 Mini-DuckDB架构

```
┌─────────────────────────────────────┐
│       Mini-DuckDB Database          │
├─────────────────────────────────────┤
│  SQL Parser (简化版)                │
│  ├── SELECT/INSERT/CREATE TABLE    │
│  └── WHERE/JOIN/GROUP BY           │
├─────────────────────────────────────┤
│  Query Optimizer                    │
│  ├── Filter Pushdown                │
│  └── Simple Join Reorder            │
├─────────────────────────────────────┤
│  Execution Engine                   │
│  ├── TableScan                      │
│  ├── Filter                         │
│  ├── HashJoin                       │
│  ├── HashAggregate                  │
│  └── Projection                     │
├─────────────────────────────────────┤
│  Storage Engine                     │
│  ├── Column Storage                 │
│  ├── Simple Compression             │
│  ├── Buffer Manager (LRU)          │
│  └── WAL                            │
└─────────────────────────────────────┘
```

### 30.3 功能清单

**已实现：**
- ✅ 内存表（SimpleTable）
- ✅ 基本算子（Scan、Filter、Join、Aggregate）
- ✅ 简单优化器（Filter Pushdown）
- ✅ 列式存储（SimpleDiskTable）
- ✅ LRU缓存（SimpleBufferManager）

**建议扩展：**
1. **SQL解析器**
   - 使用现成库（如sqlparser-rs绑定）
   - 或手写简单的递归下降parser

2. **更多算子**
   - ORDER BY（排序）
   - LIMIT/OFFSET
   - DISTINCT

3. **更完善的优化**
   - 常量折叠
   - Join Order（DP算法）
   - 统计信息收集

4. **持久化**
   - Checkpoint机制
   - WAL恢复
   - 元数据管理

5. **事务支持**
   - 简单的锁机制
   - 或MVCC实现

### 30.4 完整示例：构建Mini-DuckDB

```cpp
// mini_duckdb.hpp
#pragma once

namespace miniduck {

class MiniDuckDB {
    // 组件
    unique_ptr<SimpleBufferManager> buffer_manager;
    unordered_map<string, unique_ptr<SimpleDiskTable>> tables;
    unique_ptr<SimpleOptimizer> optimizer;

public:
    MiniDuckDB(const string &db_path) {
        buffer_manager = make_unique<SimpleBufferManager>(db_path);
        optimizer = make_unique<SimpleOptimizer>();
    }

    // CREATE TABLE
    void CreateTable(const string &name, vector<LogicalType> types, vector<string> columns) {
        auto table = make_unique<SimpleDiskTable>(*buffer_manager, types, columns);
        tables[name] = std::move(table);
    }

    // INSERT
    void Insert(const string &table_name, vector<Value> values) {
        auto it = tables.find(table_name);
        if (it == tables.end()) {
            throw Exception("Table not found: " + table_name);
        }
        it->second->Insert(values);
    }

    // SELECT (简化版)
    unique_ptr<DataChunk> Query(const string &sql) {
        // 1. 解析SQL（简化：假设已解析）
        auto plan = ParseSQL(sql);

        // 2. 优化
        plan = optimizer->Optimize(std::move(plan));

        // 3. 执行
        auto result = make_unique<DataChunk>();
        plan->GetNext(*result);

        return result;
    }

    // 刷新到磁盘
    void Checkpoint() {
        buffer_manager->FlushAll();
    }

private:
    unique_ptr<SimpleOperator> ParseSQL(const string &sql) {
        // 简化：硬编码一个查询计划
        // 实际应该实现真正的SQL parser

        // SELECT * FROM table WHERE age > 25
        auto scan = make_unique<SimpleTableScan>(*tables["users"]);
        auto filter = make_unique<SimpleFilter>(
            std::move(scan),
            [](DataChunk &chunk, idx_t row) {
                return chunk.GetValue(1, row).GetValue<int>() > 25;
            }
        );
        return filter;
    }
};

} // namespace miniduck
```

### 30.5 使用Mini-DuckDB

```cpp
#include "mini_duckdb.hpp"

int main() {
    using namespace miniduck;

    // 创建数据库
    MiniDuckDB db("mydb.duck");

    // 创建表
    db.CreateTable("users",
        {LogicalType::INTEGER, LogicalType::INTEGER, LogicalType::VARCHAR},
        {"id", "age", "name"}
    );

    // 插入数据
    for (int i = 0; i < 1000; i++) {
        db.Insert("users", {
            Value::INTEGER(i),
            Value::INTEGER(18 + i % 50),
            Value::VARCHAR("User" + std::to_string(i))
        });
    }

    // 查询
    auto result = db.Query("SELECT * FROM users WHERE age > 25");

    printf("Query returned %llu rows\n", result->size());

    // Checkpoint
    db.Checkpoint();

    return 0;
}
```

### 30.6 学习成果总结

通过30天的学习，你应该掌握了：

**理论知识：**
- ✅ 向量化执行原理
- ✅ 列式存储优势
- ✅ 查询优化技术
- ✅ MVCC事务管理
- ✅ 压缩算法
- ✅ 缓存策略

**实践能力：**
- ✅ 阅读DuckDB源码
- ✅ 实现基本查询算子
- ✅ 编写优化规则
- ✅ 设计存储引擎
- ✅ 构建简单数据库

**下一步学习方向：**

1. **深入DuckDB高级特性**
   - Adaptive Filter
   - Parallel Execution
   - Extension System

2. **阅读相关论文**
   - "MonetDB/X100: Hyper-Pipelining Query Execution"
   - "Morsel-Driven Parallelism"
   - "Everything You Always Wanted to Know About Compiled and Vectorized Queries But Were Afraid to Ask"

3. **实践项目**
   - 为Mini-DuckDB添加更多功能
   - 贡献到DuckDB开源项目
   - 构建自己的专用数据库

4. **性能调优**
   - Profiling和性能分析
   - SIMD优化
   - 缓存优化

### 30.7 课程资源汇总

**文档：**
- `DuckDB_30天学习课程.md` - Day 1-7
- `DuckDB_30天学习课程_第2-4周.md` - Day 10-14
- `DuckDB_30天学习课程_第3周_优化器.md` - Day 16-19
- `DuckDB_30天学习课程_第3-4周.md` - Day 20-22
- `DuckDB_30天学习课程_第4周完整版.md` - Day 23-26
- 本文档 - Day 27-30

**关键文件索引：**
```
src/
├── common/types/           # Vector, DataChunk
├── parser/                 # SQL解析
├── planner/               # 逻辑计划
├── optimizer/             # 查询优化
├── execution/             # 执行引擎
│   ├── operator/          # 算子实现
│   └── aggregate_hashtable.cpp
├── storage/               # 存储引擎
│   ├── table/             # 表管理
│   ├── compression/       # 压缩
│   └── buffer_manager.cpp # 缓存
└── transaction/           # 事务管理
```

## 结语

恭喜你完成了DuckDB 30天深度学习课程！

你已经从零开始，系统学习了现代分析型数据库的核心技术。通过理论学习和动手实践，你不仅理解了DuckDB的设计原理，也掌握了如何构建自己的数据库系统。

数据库系统是计算机科学中最复杂的软件之一，需要综合运用算法、数据结构、操作系统、编译器等多方面知识。DuckDB作为现代列式数据库的优秀代表，展示了向量化执行、智能优化、高效存储等前沿技术的最佳实践。

继续深入，持续学习，你将在数据库领域走得更远！

**保持学习，保持好奇！🚀**

---

## 附录：常见问题

**Q1: 如何调试DuckDB源码？**
```bash
# 使用debug build
make debug
gdb ./build/debug/duckdb

# 或使用IDE（VSCode/CLion）
# 配置launch.json指向debug二进制
```

**Q2: 如何贡献到DuckDB项目？**
1. Fork仓库
2. 创建feature分支
3. 编写代码和测试
4. 运行 `make format-fix`
5. 提交PR

**Q3: 性能优化最佳实践？**
- 使用SIMD
- 减少分支
- 优化内存布局
- 预取数据
- 批量处理

**Q4: 推荐学习资源？**
- CMU 15-445/645 Database Systems
- DuckDB Blog: https://duckdb.org/blog
- DuckDB Discord: https://discord.gg/tcvwpjfnZx
