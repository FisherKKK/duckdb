# DuckDB 深度课程：列存储与数据压缩

> 本课程深入讲解列式存储架构、压缩算法实现和存储优化技巧。

---

## 课程概览

### 学习目标

- 理解列式存储的原理和优势
- 掌握主流压缩算法的实现
- 学习编码优化技巧
- 了解RowGroup和Segment设计
- 掌握存储性能调优方法

### 前置知识

- 数据结构基础
- 位运算和二进制表示
- 信息论基础（熵、编码）

---

## 第一部分：列存储架构

### 1.1 行存储 vs 列存储

```
行存储（Row-oriented）：
┌─────────────────────────────────────┐
│ Row 1: [id:1, name:Alice, age:25]   │
│ Row 2: [id:2, name:Bob, age:30]     │
│ Row 3: [id:3, name:Charlie, age:35] │
└─────────────────────────────────────┘
优势：适合INSERT、UPDATE、SELECT *
劣势：分析查询性能低

列存储（Column-oriented）：
┌──────────────┐ ┌───────────────────┐ ┌───────────────┐
│ id: [1,2,3]  │ │ name: [Alice,Bob,  │ │ age: [25,30,  │
│              │ │        Charlie]    │ │       35]     │
└──────────────┘ └───────────────────┘ └───────────────┘
优势：适合分析查询、压缩率高
劣势：INSERT、UPDATE性能较低
```

### 1.2 DuckDB列存储架构

```cpp
// src/storage/table/column_data.hpp

namespace duckdb {

// 列数据片段
class ColumnData {
public:
    // 列类型
    LogicalType type;

    // 数据段列表
    vector<unique_ptr ColumnSegment>> segments;

    // 统计信息
    unique_ptr<SegmentStatistics> stats;

    // 读取数据
    void Scan(Transaction &transaction,
             ColumnScanState &state,
             idx_t scan_count,
             Vector &result) {

        idx_t remaining = scan_count;
        idx_t current_offset = 0;

        while (remaining > 0) {
            if (state.segment_index >= segments.size()) {
                break;
            }

            // 扫描当前segment
            auto &segment = segments[state.segment_index];
            idx_t count = MinValue(remaining,
                                   segment->count - state.row_index);

            segment->Scan(transaction, state.row_index, count, result);
            result.Verify(count, result, "Column scan");

            remaining -= count;
            current_offset += count;
            state.row_index += count;

            if (state.row_index >= segment->count) {
                // 移动到下一个segment
                state.segment_index++;
                state.row_index = 0;
            }
        }
    }

    // 追加数据
    void Append(DataChunk &chunk, idx_t col_idx) {
        Vector &vec = chunk.data[col_idx];

        // 获取最后一个segment
        if (segments.empty() ||
            segments.back()->type == SegmentType::INVALID) {
            // 创建新segment
            segments.push_back(make_unique ColumnSegment>(
                type, SegmentType::INLINE, start_row, 0));
        }

        auto &last_segment = segments.back();

        // 尝试追加到最后一个segment
        idx_t append_count = vec.size();
        idx_t remaining = append_count;
        idx_t offset = 0;

        while (remaining > 0) {
            idx_t available = STANDARD_VECTOR_SIZE -
                             last_segment->count;

            if (available == 0) {
                // 当前segment已满，创建新segment
                segments.push_back(make_unique ColumnSegment>(
                    type, SegmentType::INLINE,
                    start_row + append_count - remaining,
                    0));
                last_segment = segments.back();
                available = STANDARD_VECTOR_SIZE;
            }

            // 追加数据
            idx_t append_size = MinValue(available, remaining);
            last_segment->Append(vec, offset, append_size);

            remaining -= append_size;
            offset += append_size;
        }

        // 更新统计信息
        stats->Update(vec, append_count);
    }

    // 更新数据
    void Update(Transaction &transaction, Vector &updates,
                Vector &row_ids) {
        // 逐行更新
        for (idx_t i = 0; i < row_ids.size(); i++) {
            auto row_id = row_ids.GetValue<row_t>(i);

            // 找到对应的segment
            idx_t segment_idx = row_id / STANDARD_VECTOR_SIZE;
            idx_t segment_row = row_id % STANDARD_VECTOR_SIZE;

            if (segment_idx >= segments.size()) {
                throw InvalidInputException("Invalid row ID");
            }

            // 创建undo entry
            UndoEntry undo_entry;
            segments[segment_idx]->FetchRow(segment_row, undo_entry.old_value);

            // 更新数据
            segments[segment_idx]->Update(segment_row, updates.GetValue(i));

            undo_entry.new_value = updates.GetValue(i);
            undo_entry.segment_idx = segment_idx;
            undo_entry.row_id = segment_row;

            // 记录undo
            transaction.PushUndo(std::move(undo_entry));
        }
    }
};

// 列段
struct ColumnSegment {
    SegmentType type;          // INLINE、TRANSIENT等
    idx_t start;               // 起始行号
    idx_t count;               // 行数
    data_ptr_t data;           // 数据指针

    // 压缩信息
    CompressionType compression_type;
    unique_ptr<SegmentCompressionState> compression_state;

    // 扫描
    void Scan(Transaction &transaction,
             idx_t row_offset,
             idx_t count,
             Vector &result) {
        if (type == SegmentType::INLINE) {
            // 内联数据，直接读取
            ScanInternal(row_offset, count, result);
        } else if (type == SegmentType::TRANSIENT) {
            // 临时数据
            ScanTransient(row_offset, count, result);
        }
    }

    // 追加
    void Append(Vector &vec, idx_t offset, idx_t count) {
        // 解压数据（如果需要）
        if (compression_type != CompressionType::COMPRESSION_NONE) {
            DecompressData();
        }

        // 复制数据
        auto target_ptr = data + (this->count * GetTypeIdSize(type));

        for (idx_t i = 0; i < count; i++) {
            target_ptr[i] = vec.GetValue(offset + i);
        }

        this->count += count;

        // 重新压缩
        if (compression_type != CompressionType::COMPRESSION_NONE) {
            CompressData();
        }
    }

private:
    void ScanInternal(idx_t row_offset,
                     idx_t count,
                     Vector &result) {
        if (compression_type == CompressionType::COMPRESSION_CONSTANT) {
            // 常量压缩，所有值相同
            auto constant = GetConstantValue();
            for (idx_t i = 0; i < count; i++) {
                result.SetValue(i, constant);
            }
        } else if (compression_type == CompressionType::COMPRESSION_RLE) {
            // RLE解压
            DecompressRLE(row_offset, count, result);
        } else {
            // 无压缩，直接复制
            auto source_ptr = data + row_offset * GetTypeIdSize(type);
            result.Scan(source_ptr, count);
        }
    }
};

} // namespace duckdb
```

### 1.3 RowGroup管理

```cpp
// src/storage/table/table_statistics.cpp

// RowGroup：一组行的集合
class RowGroup {
public:
    RowGroupCollection &collection;
    idx_t start_row;              // 起始行号
    idx_t count;                  // 行数
    vector<unique_ptr<ColumnData>> columns;
    AtomicCounter refs;           // 引用计数

    RowGroup(RowGroupCollection &collection_p,
           idx_t start_row_p,
           idx_t count_p)
        : collection(collection_p),
          start_row(start_row_p),
          count(count_p),
          refs(0) {

        // 为每列创建ColumnData
        auto &table = collection.GetTable();
        auto &table_info = table.GetSchema();

        for (auto &col : table_info.GetColumns()) {
            columns.push_back(make_unique<ColumnData>(
                col.Type(),
                start_row,
                count
            ));
        }
    }

    // 扫描
    void Scan(Transaction &transaction,
             TableScanState &state,
             DataChunk &result) {
        // 初始化输出chunk
        if (result.ColumnCount() == 0) {
            result.Initialize(columns[0]->type);
        }

        // 逐列扫描
        for (size_t col_idx = 0; col_idx < columns.size(); col_idx++) {
            auto &col_data = columns[col_idx];
            auto &column_state = state.column_states[col_idx];

            col_data->Scan(transaction, column_state,
                          STANDARD_VECTOR_SIZE, result.data[col_idx]);
        }

        result.SetCardinality(STANDARD_VECTOR_SIZE);
    }

    // 追加
    void Append(DataChunk &chunk) {
        if (chunk.size() != STANDARD_VECTOR_SIZE) {
            throw InvalidInputException(
                "Chunk size must be STANDARD_VECTOR_SIZE");
        }

        // 逐列追加
        for (size_t col_idx = 0; col_idx < columns.size(); col_idx++) {
            columns[col_idx]->Append(chunk, col_idx);
        }

        count += chunk.size();
    }

    // 提交追加
    void CommitAppend(transaction_t transaction_id,
                      idx_t row_group_idx) {
        for (auto &col : columns) {
            col->CommitAppend(transaction_id, row_group_idx);
        }
    }

    // 获取统计信息
    TableStatistics GetStatistics() {
        TableStatistics stats;
        stats.column_stats.reserve(columns.size());

        for (size_t col_idx = 0; col_idx < columns.size(); col_idx++) {
            stats.column_stats.push_back(
                columns[col_idx]->GetStatistics()
            );
        }

        return stats;
    }
};
```

---

## 第二部分：压缩算法

### 2.1 压缩算法分类

```
压缩算法分类：

1. 通用压缩
   ├── BitPacking      // 位打包
   ├── RLE            // 行程编码
   ├── Dictionary     // 字典编码
   ├── FOR            // Frame-Of-Reference
   └── ZSTD/LZ4       // 通用压缩算法

2. 特定类型压缩
   ├── BOOL:          BitPacking
   ├── INTEGER:       FOR, BitPacking, RLE
   ├── DOUBLE:        Gorillas, Chimp
   └── STRING:        Dictionary, RLE, FST

3. 组合压缩
   ├── MultiCompression: 多种算法组合
   └── Adaptive:       根据数据特征自动选择
```

### 2.2 BitPacking（位打包）

```cpp
// src/storage/compression/bitpacking.cpp

// BitPacking：将数值压缩到位级别
// 例如：32位整数，如果最大值是1000，只需要10位存储

class Bitpacking {
public:
    // 压缩
    static void Pack(const uint32_t *input, uint32_t count,
                    uint8_t *output, uint8_t bit_width) {
        // bit_width: 每个值占用的位数
        // 输入：[0, 1, 2, 3, ...]
        // 输出：位打包后的数据

        uint64_t current_word = 0;
        uint8_t current_bit = 0;

        for (uint32_t i = 0; i < count; i++) {
            // 将下一个值打包到current_word中
            current_word |= ((uint64_t)input[i]) << current_bit;
            current_bit += bit_width;

            // 如果current_word满了（64位），写入输出
            if (current_bit >= 64) {
                memcpy(output, &current_word, sizeof(uint64_t));
                output += sizeof(uint64_t);

                // 处理溢出的位
                if (current_bit > 64) {
                    uint8_t overflow = current_bit - 64;
                    current_word = ((uint64_t)input[i]) >> (bit_width - overflow);
                    current_bit = overflow;
                } else {
                    current_word = 0;
                    current_bit = 0;
                }
            }
        }

        // 写入最后一个word
        if (current_bit > 0) {
            memcpy(output, &current_word, sizeof(uint64_t));
        }
    }

    // 解压
    static void Unpack(const uint8_t *input, uint32_t count,
                      uint32_t *output, uint8_t bit_width) {
        uint64_t current_word = 0;
        uint8_t current_bit = 0;
        uint32_t mask = (1ULL << bit_width) - 1;

        for (uint32_t i = 0; i < count; i++) {
            if (current_bit + bit_width > 64) {
                // 需要从下一个word读取
                memcpy(&current_word, input, sizeof(uint64_t));
                input += sizeof(uint64_t);

                uint8_t remaining = 64 - current_bit;
                output[i] = (current_word >> current_bit) & mask;

                // 继续读取下一个word
                memcpy(&current_word, input, sizeof(uint64_t));
                input += sizeof(uint64_t);

                output[i] |= ((current_word << remaining) & mask);
                current_bit = bit_width - remaining;
            } else {
                output[i] = (current_word >> current_bit) & mask;
                current_bit += bit_width;

                if (current_bit >= 64) {
                    memcpy(&current_word, input, sizeof(uint64_t));
                    input += sizeof(uint64_t);
                    current_bit -= 64;
                }
            }
        }
    }

    // 计算需要的bit宽度
    static uint8_t GetBitWidth(uint32_t max_value) {
        if (max_value == 0) {
            return 1;
        }

        uint8_t width = 0;
        while (max_value > 0) {
            max_value >>= 1;
            width++;
        }

        return width;
    }

    // 计算压缩率
    static double CalculateCompressionRatio(uint32_t count,
                                           uint8_t bit_width) {
        // 原始大小：count * 4 bytes (32位整数)
        // 压缩后大小：count * bit_width bits
        size_t original_size = count * sizeof(uint32_t);
        size_t compressed_size = (count * bit_width + 7) / 8;

        return (double)original_size / compressed_size;
    }
};
```

### 2.3 FOR编码（Frame-Of-Reference）

```cpp
// src/storage/compression/for.cpp

// FOR编码：存储每个值与基准值的差值
// 适用于单调递增或相对集中的数据

class FORCompression {
public:
    struct FORState {
        uint32_t frame_of_reference;  // 基准值
        uint8_t bit_width;           // 差值所需的位数
        unique_ptr<uint8_t[]> packed_data;
    };

    // 压缩
    static unique_ptr<FORState> Compress(const uint32_t *input,
                                         uint32_t count) {
        auto state = make_unique<FORState>();

        // 1. 找到最小值作为基准
        uint32_t min_val = UINT32_MAX;
        uint32_t max_val = 0;

        for (uint32_t i = 0; i < count; i++) {
            if (input[i] < min_val) {
                min_val = input[i];
            }
            if (input[i] > max_val) {
                max_val = input[i];
            }
        }

        state->frame_of_reference = min_val;

        // 2. 计算最大差值
        uint32_t max_diff = max_val - min_val;
        state->bit_width = Bitpacking::GetBitWidth(max_diff);

        // 3. 计算差值
        vector<uint32_t> diffs(count);
        for (uint32_t i = 0; i < count; i++) {
            diffs[i] = input[i] - min_val;
        }

        // 4. BitPack差值
        state->packed_data = unique_ptr<uint8_t[]>(
            new uint8_t[(count * state->bit_width + 7) / 8]
        );

        Bitpacking::Pack(diffs.data(), count,
                        state->packed_data.get(),
                        state->bit_width);

        return state;
    }

    // 解压
    static void Decompress(const FORState &state,
                          uint32_t *output,
                          uint32_t count) {
        // 1. Unpack差值
        vector<uint32_t> diffs(count);
        Bitpacking::Unpack(state.packed_data.get(), count,
                          diffs.data(), state.bit_width);

        // 2. 加上基准值
        for (uint32_t i = 0; i < count; i++) {
            output[i] = diffs[i] + state.frame_of_reference;
        }
    }

    // 估算压缩率
    static double EstimateCompressionRatio(const uint32_t *input,
                                          uint32_t count) {
        // 原始大小
        size_t original_size = count * sizeof(uint32_t);

        // 找最小值
        uint32_t min_val = UINT32_MAX;
        uint32_t max_val = 0;

        for (uint32_t i = 0; i < count; i++) {
            if (input[i] < min_val) min_val = input[i];
            if (input[i] > max_val) max_val = input[i];
        }

        // 计算差值范围
        uint32_t max_diff = max_val - min_val;
        uint8_t bit_width = Bitpacking::GetBitWidth(max_diff);

        // 压缩后大小
        size_t compressed_size = sizeof(uint32_t) +  // 基准值
                                sizeof(uint8_t) +     // bit宽度
                                (count * bit_width + 7) / 8;  // 数据

        return (double)original_size / compressed_size;
    }
};
```

### 2.4 RLE（Run-Length Encoding）

```cpp
// src/storage/compression/rle.cpp

// RLE：连续相同的值只存储一次和重复次数

class RLECompression {
public:
    struct RLESegment {
        uint32_t value;
        uint32_t count;
    };

    // 压缩
    static vector<RLESegment> Compress(const uint32_t *input,
                                      uint32_t count) {
        vector<RLESegment> segments;

        if (count == 0) {
            return segments;
        }

        RLESegment current;
        current.value = input[0];
        current.count = 1;

        for (uint32_t i = 1; i < count; i++) {
            if (input[i] == current.value) {
                // 继续当前run
                current.count++;
            } else {
                // 结束当前run，开始新run
                segments.push_back(current);

                current.value = input[i];
                current.count = 1;
            }
        }

        // 添加最后一个segment
        segments.push_back(current);

        return segments;
    }

    // 解压
    static void Decompress(const vector<RLESegment> &segments,
                          uint32_t *output) {
        uint32_t offset = 0;

        for (const auto &segment : segments) {
            for (uint32_t i = 0; i < segment.count; i++) {
                output[offset++] = segment.value;
            }
        }
    }

    // 压缩率估算
    static double EstimateCompressionRatio(const uint32_t *input,
                                          uint32_t count) {
        auto segments = Compress(input, count);

        // 原始大小
        size_t original_size = count * sizeof(uint32_t);

        // 压缩后大小
        size_t compressed_size = segments.size() *
                                (sizeof(uint32_t) * 2);  // value + count

        return (double)original_size / compressed_size;
    }

    // 判断是否适合RLE
    static bool IsSuitable(const uint32_t *input, uint32_t count) {
        if (count < 16) {
            return false;  // 数据量太少
        }

        // 计算连续相同值的平均长度
        auto segments = Compress(input, count);
        double avg_run_length = (double)count / segments.size();

        // 平均run长度 > 4，适合RLE
        return avg_run_length > 4.0;
    }
};
```

### 2.5 Dictionary编码

```cpp
// src/storage/compression/dictionary.cpp

// Dictionary编码：将字符串映射为整数ID

class DictionaryCompression {
public:
    struct Dictionary {
        vector<string> strings;        // 字符串表
        unordered_map<string, uint32_t> index;  // 字符串 -> ID
    };

    // 压缩
    static unique_ptr<Dictionary> Build(const string *input,
                                        uint32_t count) {
        auto dict = make_unique<Dictionary>();

        for (uint32_t i = 0; i < count; i++) {
            const string &str = input[i];

            // 查找或创建
            auto it = dict->index.find(str);
            if (it == dict->index.end()) {
                // 新字符串
                uint32_t id = dict->strings.size();
                dict->strings.push_back(str);
                dict->index[str] = id;
            }
        }

        return dict;
    }

    // 编码：字符串 -> ID
    static vector<uint32_t> Encode(const Dictionary &dict,
                                   const string *input,
                                   uint32_t count) {
        vector<uint32_t> result(count);

        for (uint32_t i = 0; i < count; i++) {
            auto it = dict.index.find(input[i]);
            if (it != dict.index.end()) {
                result[i] = it->second;
            } else {
                // 未找到，需要更新字典
                result[i] = UINT32_MAX;  // 错误标记
            }
        }

        return result;
    }

    // 解码：ID -> 字符串
    static void Decode(const Dictionary &dict,
                      const uint32_t *input,
                      string *output,
                      uint32_t count) {
        for (uint32_t i = 0; i < count; i++) {
            uint32_t id = input[i];

            if (id < dict.strings.size()) {
                output[i] = dict.strings[id];
            } else {
                output[i] = "";  // 无效ID
            }
        }
    }

    // 计算压缩率
    static double CalculateCompressionRatio(const Dictionary &dict,
                                           uint32_t count) {
        // 原始大小：假设平均字符串长度20
        size_t original_size = count * 20;

        // 压缩后大小
        size_t dict_size = 0;
        for (const auto &str : dict.strings) {
            dict_size += str.size();
        }

        size_t data_size = count * sizeof(uint32_t);
        size_t compressed_size = dict_size + data_size;

        return (double)original_size / compressed_size;
    }

    // 基数（Cardinality）
    static double GetCardinality(const Dictionary &dict) {
        return static_cast<double>(dict.strings.size());
    }

    // 编码效率
    static double GetEncodingEfficiency(const Dictionary &dict,
                                        uint32_t count) {
        // 编码效率 = 不同值数量 / 总值数量
        return dict.strings.size() / static_cast<double>(count);
    }
};
```

---

## 第三部分：高级压缩算法

### 3.1 Gorillas算法（浮点数压缩）

```cpp
// Gorillas算法：用于压缩时间序列浮点数

class GorillasCompression {
public:
    struct GorillasState {
        double previous_value;
        uint64_t previous_leading_zeros;
        uint64_t previous_trailing_zeros;
    };

    // 压缩单个值
    static void CompressValue(double value,
                             GorillasState &state,
                             vector<uint8_t> &output) {
        // XOR with previous value
        uint64_t XORed = EncodeDouble(value) ^
                       EncodeDouble(state.previous_value);

        // 计算前导零
        uint64_t leading_zeros = CountLeadingZeros(XORed);
        uint64_t trailing_zeros = CountTrailingZeros(XORed);

        if (XORed == 0) {
            // 与前一个值相同，写入'0'
            output.push_back(0);
        } else {
            // 计算有效位
            uint64_t meaningful_bits = 64 - leading_zeros - trailing_zeros;

            if (leading_zeros >= state.previous_leading_zeros &&
                trailing_zeros >= state.previous_trailing_zeros) {
                // 使用前一个leading zero count
                WriteBit(1, output);  // 标志位

                // 写入有效位
                WriteBits(XORed >> state.previous_trailing_zeros,
                         meaningful_bits, output);
            } else {
                // 需要写入leading zero count
                WriteBit(1, output);  // 标志位

                // 写入leading zero count (5 bits)
                WriteBits(leading_zeros, 5, output);

                // 写入有效位
                WriteBits(XORed >> trailing_zeros, meaningful_bits, output);

                state.previous_leading_zeros = leading_zeros;
                state.previous_trailing_zeros = trailing_zeros;
            }
        }

        state.previous_value = value;
    }

    // 解压缩
    static double DecompressValue(vector<uint8_t> &input,
                                 GorillasState &state,
                                 size_t &bit_offset) {
        uint8_t first_bit = ReadBit(input, bit_offset);

        if (first_bit == 0) {
            // 与前一个值相同
            return state.previous_value;
        }

        uint8_t leading_zero_bits = ReadBits(input, bit_offset, 5);
        uint64_t leading_zeros = leading_zero_bits;

        // 计算有效位
        uint64_t meaningful_bits = 64 - leading_zeros;

        // 读取XORed值
        uint64_t XORed = ReadBits(input, bit_offset, meaningful_bits);

        // 恢复原值
        uint64_t decoded = XORed ^ EncodeDouble(state.previous_value);

        state.previous_value = DecodeDouble(decoded);
        state.previous_leading_zeros = leading_zeros;

        return state.previous_value;
    }

private:
    static uint64_t EncodeDouble(double value) {
        uint64_t result;
        memcpy(&result, &value, sizeof(double));
        return result;
    }

    static double DecodeDouble(uint64_t value) {
        double result;
        memcpy(&result, &value, sizeof(double));
        return result;
    }

    static uint64_t CountLeadingZeros(uint64_t value) {
        if (value == 0) return 64;
        return __builtin_clzll(value);
    }

    static uint64_t CountTrailingZeros(uint64_t value) {
        if (value == 0) return 64;
        return __builtin_ctzll(value);
    }

    static void WriteBit(uint8_t bit, vector<uint8_t> &output) {
        if (output.empty() || output.back() >= 128) {
            output.push_back(0);
        }

        output.back() = (output.back() << 1) | bit;
    }

    static void WriteBits(uint64_t value, uint8_t count,
                         vector<uint8_t> &output) {
        for (int i = count - 1; i >= 0; i--) {
            WriteBit((value >> i) & 1, output);
        }
    }

    static uint8_t ReadBit(const vector<uint8_t> &input, size_t &offset) {
        size_t byte_offset = offset / 8;
        size_t bit_offset = 7 - (offset % 8);

        uint8_t bit = (input[byte_offset] >> bit_offset) & 1;
        offset++;
        return bit;
    }

    static uint64_t ReadBits(const vector<uint8_t> &input,
                            size_t &offset,
                            uint8_t count) {
        uint64_t result = 0;

        for (uint8_t i = 0; i < count; i++) {
            result = (result << 1) | ReadBit(input, offset);
        }

        return result;
    }
};
```

### 3.2 Chimp算法（改进的浮点数压缩）

```cpp
// Chimp算法：改进的Gorillas，使用上下文建模

class ChimpCompression {
public:
    struct ChimpState {
        double previous_value;
        double stored_values[256];  // 上下文
        uint8_t next_index;
    };

    static void CompressValue(double value,
                             ChimpState &state,
                             vector<uint8_t> &output) {
        // 检查是否匹配之前的值
        for (int i = 0; i < 256; i++) {
            if (state.stored_values[i] == value) {
                // 匹配！写入索引
                output.push_back(0);  // 标志位
                output.push_back(i);
                return;
            }
        }

        // 不匹配，使用Gorillas算法
        GorillasCompression::CompressValue(value, state, output);

        // 存储值到上下文
        state.stored_values[state.next_index] = value;
        state.next_index++;
    }

    static double DecompressValue(vector<uint8_t> &input,
                                  ChimpState &state,
                                  size_t &bit_offset,
                                  size_t &byte_offset) {
        uint8_t flag = input[byte_offset++];

        if (flag == 0) {
            // 从上下文读取
            uint8_t index = input[byte_offset++];
            return state.stored_values[index];
        }

        // 使用Gorillas解压
        return GorillasCompression::DecompressValue(input, state,
                                                   bit_offset);
    }
};
```

---

## 第四部分：存储优化

### 4.1 自适应压缩选择

```cpp
// src/storage/compression/adaptive.cpp

// 自动选择最佳压缩算法

class AdaptiveCompression {
public:
    struct CompressionResult {
        CompressionType type;
        unique_ptr<uint8_t[]> data;
        size_t compressed_size;
        double compression_ratio;
    };

    // 尝试所有压缩算法，选择最佳
    static CompressionResult Compress(const Vector &input) {
        vector<CompressionResult> results;

        // 尝试各种压缩算法
        results.push_back(TryConstantCompression(input));
        results.push_back(TryRLECompression(input));
        results.push_back(TryBitPackingCompression(input));
        results.push_back(TryFORCompression(input));
        results.push_back(TryDictionaryCompression(input));
        results.push_back(TryZSTDCompression(input));

        // 选择压缩率最高的
        auto best = *std::max_element(results.begin(), results.end(),
            [](const CompressionResult &a, const CompressionResult &b) {
                return a.compression_ratio < b.compression_ratio;
            });

        return best;
    }

private:
    static CompressionResult TryConstantCompression(const Vector &input) {
        CompressionResult result;
        result.type = CompressionType::COMPRESSION_CONSTANT;

        // 检查是否所有值相同
        bool all_same = true;
        Value first_value;

        for (idx_t i = 0; i < input.size(); i++) {
            auto value = input.GetValue(i);

            if (i == 0) {
                first_value = value;
            } else if (value != first_value) {
                all_same = false;
                break;
            }
        }

        if (all_same) {
            // 存储单个值
            result.compressed_size = sizeof(Value);
            result.data = unique_ptr<uint8_t[]>(
                new uint8_t[sizeof(Value)]);
            memcpy(result.data.get(), &first_value, sizeof(Value));

            result.compression_ratio =
                (double)(input.size() * sizeof(Value)) /
                result.compressed_size;
        } else {
            // 不适合常量压缩
            result.compression_ratio = 1.0;
            result.compressed_size = SIZE_MAX;
        }

        return result;
    }

    static CompressionResult TryRLECompression(const Vector &input) {
        // ... 实现RLE压缩
    }

    static CompressionResult TryBitPackingCompression(const Vector &input) {
        // ... 实现BitPacking压缩
    }

    static CompressionResult TryFORCompression(const Vector &input) {
        // ... 实现FOR压缩
    }

    static CompressionResult TryDictionaryCompression(const Vector &input) {
        // ... 实现Dictionary压缩
    }

    static CompressionResult TryZSTDCompression(const Vector &input) {
        // ... 实现ZSTD压缩
    }
};
```

### 4.2 分区裁剪（Partition Pruning）

```cpp
// src/storage/table/part_manager.cpp

// 分区裁剪：只扫描需要的分区

class PartitionManager {
public:
    struct Partition {
        idx_t start;
        idx_t end;
        Vector min_values;  // 每列的最小值
        Vector max_values;  // 每列的最大值
    };

    // 查找需要扫描的分区
    static vector<idx_t> FindRelevantPartitions(
            const vector<Partition> &partitions,
            const vector<unique_ptr<Expression>> &filters) {

        vector<idx_t> relevant_partitions;

        for (idx_t i = 0; i < partitions.size(); i++) {
            auto &partition = partitions[i];

            // 检查分区是否可能与过滤条件匹配
            bool is_relevant = true;

            for (const auto &filter : filters) {
                if (!CouldMatchFilter(partition, filter)) {
                    is_relevant = false;
                    break;
                }
            }

            if (is_relevant) {
                relevant_partitions.push_back(i);
            }
        }

        return relevant_partitions;
    }

private:
    static bool CouldMatchFilter(const Partition &partition,
                                const unique_ptr<Expression> &filter) {
        // 简化示例：只处理 x > constant 形式
        if (filter->type == ExpressionType::COMPARE_GREATER_THAN) {
            auto &compare = (ComparisonExpression &)*filter;
            auto &constant = (BoundConstantExpression &)*compare.right;

            // 检查分区最大值
            idx_t col_idx = GetColumnIndex(compare.left.get());

            if (constant.value >
                partition.max_values.GetValue(col_idx)) {
                return false;  // 所有值都小于阈值
            }
        }

        return true;
    }

    static idx_t GetColumnIndex(Expression *expr) {
        // 从表达式提取列索引
        // ... 实现细节
        return 0;
    }
};
```

### 4.3 延迟物化（Late Materialization）

```cpp
// 延迟物化：只在最后时刻才提取实际数据

class LateMaterialization {
public:
    // 第一阶段：只处理行号
    static vector<row_t> ScanRowIDs(Table &table,
                                   Expression &filter) {
        vector<row_t> result;

        // 扫描RowGroup
        for (auto &row_group : table.row_groups) {
            // 只扫描必要的列（过滤条件中用到的）
            DataChunk filter_chunk;
            row_group->ScanColumns({filter_column_index}, filter_chunk);

            // 应用过滤
            SelectionVector sel;
            idx_t count = ApplyFilter(filter_chunk, filter, sel);

            // 收集匹配的行号
            for (idx_t i = 0; i < count; i++) {
                result.push_back(
                    row_group->start_row + sel.get_index(i)
                );
            }
        }

        return result;
    }

    // 第二阶段：根据行号提取需要的列
    static DataChunk MaterializeColumns(Table &table,
                                       vector<row_t> &row_ids,
                                       vector<idx_t> &column_indices) {
        DataChunk result;
        result.Initialize(column_indices);

        // 按RowGroup分组（提高效率）
        sort(row_ids.begin(), row_ids.end(),
            [](row_t a, row_t b) {
                return (a / STANDARD_VECTOR_SIZE) <
                       (b / STANDARD_VECTOR_SIZE);
            });

        // 逐RowGroup提取
        idx_t current_row_group = -1;
        idx_t output_offset = 0;

        for (row_t row_id : row_ids) {
            idx_t row_group_idx = row_id / STANDARD_VECTOR_SIZE;
            idx_t row_offset = row_id % STANDARD_VECTOR_SIZE;

            if (row_group_idx != current_row_group) {
                // 切换RowGroup
                current_row_group = row_group_idx;
            }

            // 提取所需的列
            for (size_t col_idx = 0; col_idx < column_indices.size(); col_idx++) {
                auto &row_group = table.row_groups[current_row_group];
                auto &column_data = row_group->columns[column_indices[col_idx]];

                result.data[col_idx].SetValue(output_offset,
                    column_data->GetValue(row_offset));
            }

            output_offset++;
        }

        result.SetCardinality(row_ids.size());
        return result;
    }
};
```

---

## 第五部分：实践项目

### 项目：实现压缩算法测试

```cpp
// compression_benchmark.cpp

#include <vector>
#include <random>
#include <chrono>
#include <iostream>

using namespace std;

// 生成测试数据
vector<uint32_t> GenerateData(size_t count,
                               uint32_t min_val,
                               uint32_t max_val) {
    random_device rd;
    mt19937 gen(rd());
    uniform_int_distribution<> dis(min_val, max_val);

    vector<uint32_t> data(count);
    for (size_t i = 0; i < count; i++) {
        data[i] = dis(gen);
    }

    return data;
}

// 测试压缩率
void TestCompressionRatio() {
    cout << "=== Compression Ratio Test ===" << endl;

    // 测试不同数据集
    struct TestCase {
        string name;
        vector<uint32_t> data;
    };

    vector<TestCase> test_cases;

    // 1. 紧凑整数（1-100）
    test_cases.push_back({
        "Compact Integers (1-100)",
        GenerateData(10000, 1, 100)
    });

    // 2. 大范围整数
    test_cases.push_back({
        "Large Range (0-1M)",
        GenerateData(10000, 0, 1000000)
    });

    // 3. 重复数据
    vector<uint32_t> repeated_data(10000, 42);
    test_cases.push_back({
        "Repeated Data",
        repeated_data
    });

    // 4. 单调递增
    vector<uint32_t> increasing_data(10000);
    for (size_t i = 0; i < increasing_data.size(); i++) {
        increasing_data[i] = i;
    }
    test_cases.push_back({
        "Monotonically Increasing",
        increasing_data
    });

    // 测试各种压缩算法
    for (const auto &test_case : test_cases) {
        cout << "\nTest: " << test_case.name << endl;

        size_t original_size = test_case.data.size() * sizeof(uint32_t);

        // BitPacking
        uint8_t bit_width = Bitpacking::GetBitWidth(
            *max_element(test_case.data.begin(),
                        test_case.data.end())
        );
        vector<uint8_t> packed_data(
            (test_case.data.size() * bit_width + 7) / 8
        );
        Bitpacking::Pack(test_case.data.data(),
                        test_case.data.size(),
                        packed_data.data(),
                        bit_width);

        double bp_ratio = (double)original_size / packed_data.size();
        cout << "  BitPacking: " << bp_ratio << "x" << endl;

        // RLE
        auto rle_segments = RLECompression::Compress(
            test_case.data.data(),
            test_case.data.size()
        );
        size_t rle_size = rle_segments.size() * (sizeof(uint32_t) * 2);
        double rle_ratio = (double)original_size / rle_size;
        cout << "  RLE: " << rle_ratio << "x" << endl;

        // FOR
        auto for_state = FORCompression::Compress(
            test_case.data.data(),
            test_case.data.size()
        );
        size_t for_size = sizeof(uint32_t) + sizeof(uint8_t) +
                          (test_case.data.size() * for_state->bit_width + 7) / 8;
        double for_ratio = (double)original_size / for_size;
        cout << "  FOR: " << for_ratio << "x" << endl;
    }
}

// 测试压缩/解压速度
void TestCompressionSpeed() {
    cout << "\n=== Compression Speed Test ===" << endl;

    auto data = GenerateData(1000000, 0, 10000);

    // BitPacking
    auto start = chrono::high_resolution_clock::now();

    uint8_t bit_width = Bitpacking::GetBitWidth(10000);
    vector<uint8_t> packed_data(
        (data.size() * bit_width + 7) / 8
    );
    Bitpacking::Pack(data.data(), data.size(),
                    packed_data.data(), bit_width);

    auto end = chrono::high_resolution_clock::now();
    auto pack_time = chrono::duration_cast<chrono::milliseconds>(
        end - start
    ).count();

    cout << "BitPacking: " << pack_time << "ms" << endl;

    // 解压
    start = chrono::high_resolution_clock::now();

    vector<uint32_t> unpacked_data(data.size());
    Bitpacking::Unpack(packed_data.data(), data.size(),
                      unpacked_data.data(), bit_width);

    end = chrono::high_resolution_clock::now();
    auto unpack_time = chrono::duration_cast<chrono::milliseconds>(
        end - start
    ).count();

    cout << "BitUnpacking: " << unpack_time << "ms" << endl;

    // 验证正确性
    bool correct = (data == unpacked_data);
    cout << "Correctness: " << (correct ? "PASS" : "FAIL") << endl;
}

int main() {
    TestCompressionRatio();
    TestCompressionSpeed();

    return 0;
}
```

---

## 学习总结

### 压缩算法对比

| 算法 | 适用场景 | 压缩率 | CPU开销 |
|------|----------|--------|---------|
| BitPacking | 紧凑整数 | 2-4x | 低 |
| RLE | 重复数据 | 10-100x | 极低 |
| FOR | 相对集中数据 | 2-8x | 低 |
| Dictionary | 低基数字符串 | 5-20x | 中 |
| Gorillas | 时间序列浮点数 | 5-10x | 中 |
| ZSTD | 通用数据 | 3-5x | 高 |

### 存储优化技巧

1. **列裁剪**：只读取需要的列
2. **分区裁剪**：跳过不相关的分区
3. **延迟物化**：推迟提取实际数据
4. **自适应压缩**：根据数据特征选择算法
5. **统计信息**：利用min/max加速过滤

### 推荐资源

**论文：**
- "Compression of Time-Stamped Data in Gorilla"
- "PForDelta: Better Compression through Integer Compression"
- "Adaptive Column Compression for Analytical Workloads"

**代码：**
- DuckDB: `src/storage/compression/`
- ClickHouse: `src/Compression/`
- Parquet: Apache Arrow实现

---

**最后更新：2026-01-23**
